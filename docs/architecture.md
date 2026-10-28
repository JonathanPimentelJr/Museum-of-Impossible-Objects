# Architecture: Museum of Impossible Objects

## Overview

```
 Desktop    Mobile    Catalog view    VR
     \         |           |          /
      React + Three.js app (shared scene, swappable input)
                         |
      FastAPI backend (rate limit, moderate, validate)
            /                              \
   Claude API (prompt → JSON spec)    Postgres (objects + generation logs)
```

## Key decision: the LLM outputs data, not code

The model never writes Three.js code. It fills in a JSON spec from a fixed vocabulary of shapes, and the frontend's `ObjectRenderer` turns that spec into meshes.

- **Safety:** no model-generated code ever runs in a visitor's browser.
- **Reliability:** specs can be validated, clamped, and retried.
- **Testability:** the same spec always renders the same object, so it can be unit-tested and evaluated.

### Example spec

```json
{
  "title": "The Teapot That Pours Time",
  "parts": [
    { "shape": "sphere", "scale": [1, 0.8, 1], "position": [0, 0.5, 0],
      "material": { "color": "#c9a227", "metalness": 0.8, "roughness": 0.3 } },
    { "shape": "cylinder", "scale": [0.15, 0.6, 0.15], "position": [0.9, 0.7, 0],
      "rotation": [0, 0, -0.8], "material": { "color": "#c9a227" } }
  ],
  "animation": { "type": "float", "speed": 0.4 },
  "effects": ["sparkles"],
  "plaque": { "era": "c. 1780", "origin": "...", "description": "..." }
}
```

### Schema rules (enforced in `backend/app/schemas/object_spec.py`)

- 1 to 12 parts per object
- `shape` is one of a fixed list: box, sphere, cylinder, cone, torus, (extendable)
- scale, position, and rotation values are clamped to sane ranges
- colors must be valid hex
- `animation.type` and `effects` come from fixed lists
- plaque text has length limits

## Generation flow

1. Visitor submits a description → `POST /api/objects`
2. **Rate limiter:** per-visitor limit and a daily spend cap
3. **Moderation:** reject inappropriate prompts (the gallery is public)
4. **Claude:** returns the spec via structured output (tool use)
5. **Validation:** Pydantic schema check. On failure, retry once with the validation error fed back; on a second failure, return a friendly error
6. **Save:** store the object, assign a room and pedestal slot, log the generation
7. **Render:** frontend places the object on its pedestal and shows the plaque

v1 is a synchronous request with a loading state. A later version can stream progress over SSE ("Sketching… Sculpting… Writing plaque…").

## Data model

**objects**

| Column | Type | Notes |
|---|---|---|
| id | uuid | primary key |
| prompt | text | visitor's description |
| spec | jsonb | validated object spec |
| title | text | from spec, for listing |
| room | int | which gallery room |
| slot | int | pedestal 0–11 within the room |
| status | text | approved / rejected |
| visitor_hash | text | hashed identifier for rate limiting |
| created_at | timestamptz | |

**generations**

| Column | Type | Notes |
|---|---|---|
| id | uuid | primary key |
| object_id | uuid, nullable | null if generation failed |
| model | text | model used |
| prompt_version | text | e.g. `generate_object_v1` |
| input_tokens / output_tokens | int | for cost tracking |
| latency_ms | int | |
| retries | int | |
| outcome | text | success / invalid / moderated / error |
| error | text, nullable | |
| created_at | timestamptz | |

## API routes

| Route | Purpose |
|---|---|
| `POST /api/objects` | Generate and save a new object |
| `GET /api/rooms/{n}` | All objects in room n (pagination by room) |
| `GET /api/objects/{id}` | Single object, used for share links |
| `GET /api/objects/{id}/debug` | Prompt version, tokens, latency, retries |
| `GET /api/health` | Health check |

Each room holds 12 pedestals. The frontend loads only the current room's objects.

## Input modes

The scene and objects never know which mode is active. Only the `controls/` component changes, chosen by device detection.

- **Desktop:** WASD + mouse look (pointer lock); click a pedestal to read its plaque
- **Mobile:** on-screen joystick + drag to look; tap a pedestal to glide to it
- **Catalog view:** 2D grid of all objects; click one to open a rotatable single-object viewer. Fast path for recruiters, low-end devices, and accessibility
- **VR:** "Enter VR" appears only if the browser supports `immersive-vr` (`@react-three/xr`); teleport movement + controller ray to select

## Folder structure

```
museum-of-impossible-objects/
├── frontend/              Vite + React + TypeScript
│   └── src/
│       ├── scene/         Gallery, Room, Pedestal
│       ├── objects/       ObjectRenderer, primitives, materials
│       ├── controls/      DesktopControls, MobileControls, VRControls
│       ├── ui/            CreatePanel, Plaque, UnderTheHood, CatalogView
│       ├── api/           client + TanStack Query hooks
│       └── types/         generated from backend OpenAPI
├── backend/               FastAPI
│   ├── app/
│   │   ├── routes/        objects.py, rooms.py
│   │   ├── services/      llm.py, moderation.py, rate_limit.py, placement.py
│   │   ├── schemas/       object_spec.py
│   │   ├── prompts/       generate_object_v1.md
│   │   └── db/            models.py, session.py
│   ├── evals/             test prompts + run_evals.py
│   └── tests/
├── docs/architecture.md
├── CLAUDE.md
└── README.md
```

## Hosting

- **Frontend:** Cloudflare
- **Backend:** Python host (Render, Fly, or Railway)
- **Database:** managed Postgres (Neon or Supabase)
- **Secrets:** Anthropic API key only in backend environment variables

## Evals

`backend/evals/` holds a set of test prompts (normal, weird, adversarial). `run_evals.py` generates specs for each and reports:

- schema-valid rate on first try
- retry rate
- moderation catch rate on adversarial prompts
- average latency and token cost

Run before and after every prompt version change, and record results in the README.
