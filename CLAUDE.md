# Museum of Impossible Objects

Visitors describe an impossible object ("a teapot that pours time"). An LLM turns the description into a structured JSON spec plus a fake museum plaque, and the app renders it as a 3D object on a pedestal in a shared gallery anyone can walk through in the browser, on mobile, or in VR.

This is a portfolio project for landing a SWE I / frontend role by early 2027. Quality of engineering and clarity of explanation matter more than feature count.

See `docs/architecture.md` for the full design.

## How to work with me (Jonathan)

- I'm rebuilding rusty full-stack skills. Teach, don't just do.
- Session flow: short concept explanation → I write the code → you review it and explain what you'd change and why.
- When I'm stuck: hints first, then a small snippet, then full code only if I ask.
- Explain the "why" behind design decisions. I need to be able to defend every choice in an interview.
- Sessions are short: about 1 hour on weekdays, about 3 hours on Wednesdays and Sundays. Scope tasks to fit, and end every session with something I can commit.
- Flag anything that would look bad to a reviewer (no tests, leaked secrets, messy commits, unclear naming).

## Stack

- **Frontend:** Vite + React + TypeScript, React Three Fiber, @react-three/drei, @react-three/xr, TanStack Query, Zustand
- **Backend:** Python, FastAPI, Pydantic, SQLAlchemy (or SQLModel), Anthropic Python SDK
- **Database:** Postgres (managed: Neon or Supabase)
- **Hosting:** frontend on Cloudflare; backend on a Python host (Render, Fly, or Railway)
- **Testing:** Vitest (frontend), pytest (backend), plus an evals script for LLM output quality

## Core rules

- The LLM outputs a **JSON spec**, never executable code. The frontend renders specs from a fixed set of primitives.
- Every spec is validated against the Pydantic schema. Invalid → retry once with the error fed back → otherwise a friendly error.
- The Anthropic API key lives only on the backend. Never in the frontend, never committed. Use `.env` + `.env.example`.
- Every generation is logged (tokens, latency, retries, outcome) in the `generations` table.
- Frontend types for API data are generated from the backend's OpenAPI schema (`openapi-typescript`), not hand-written.
- The scene never depends on the input mode. Only the `controls/` component changes per device.
- VR is a progressive enhancement. Everything must work on desktop and mobile without a headset.

## Conventions

- TypeScript strict mode. No `any` without a comment explaining why.
- Python: type hints everywhere, `ruff` for linting and formatting.
- Small, focused commits with clear messages (e.g. `feat(scene): add pedestal component`).
- Prompts live in `backend/app/prompts/` as versioned files (`generate_object_v1.md`). Bump the version when the prompt changes.

## Build order and status

1. [ ] Desktop walk mode + hardcoded spec rendered on a pedestal (frontend only)
2. [ ] FastAPI + Postgres: save and load objects
3. [ ] LLM generation, schema validation, retries
4. [ ] Catalog view + mobile controls
5. [ ] Moderation, rate limiting, spend cap, "under the hood" panel
6. [ ] Deploy, VR mode, evals, README, demo video

Target: MVP deployed with a demo video by mid-December 2026.
