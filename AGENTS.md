# Repository Guidelines

This is the primary working guide for AI coding agents and contributors. It summarizes the repo-specific rules from `CLAUDE.md` and adds contributor workflow guidance. Prefer it over generic Next.js assumptions.

## Project Structure & First Reads

InfiPlot is a Next.js 16 / TypeScript app for AI-driven interactive visual novels. The server is intentionally stateless: the client carries the full `Session` and sends it to API routes whenever new generation is needed.

- `app/`: App Router pages and API routes. Start here for request/response behavior.
- `components/`: Client UI, especially `PlayCanvas.tsx`, forms, cards, and analytics.
- `lib/types/index.ts`: Shared domain contracts. Read this before changing payload shapes.
- `lib/engine/`: Core story engine. `director.ts` orchestrates scene generation.
- `lib/engine/agents/`: Architect, Writer, CharacterDesigner, Cinematographer, Painter.
- `lib/engine/prompts.ts`: Agent prompts and prompt-cache-sensitive message builders.
- `lib/ai-client/`: Text, image, vision, and retry wrappers.
- `lib/tts-client/`: TTS integration.
- `scripts/`: Asset and preset generation helpers.
- `public/`, `docs/`: Static assets and documentation imagery.

For engine work, read `lib/types/index.ts`, the target agent/orchestrator file, and the API route exposing the behavior. For UI work, inspect the component and the owning page.

## Core Architecture

The engine behaves like `Session + EngineConfig -> SceneResult`. The client appends returned scenes to `session.history`, replaces `session.characters` and `session.storyState`, and sends the updated `Session` back later. Do not introduce server-side session storage, hidden global game state, or persistence unless explicitly requested.

The core pipeline is `directScene()` in `lib/engine/director.ts`:

1. Writer runs serially and produces `beats[]`, `sceneSummary`, `sceneKey`, and `storyStatePatch`.
2. CharacterDesigner and Cinematographer run in parallel.
3. Entry-beat portraits may block Painter because they become references.
4. Non-entry portraits and all voice provisioning should overlap with painting.
5. Painter generates the scene background from `integratedPrompt` plus `referenceImages`.

Do not add blocking calls between Writer completion and Painter start. Anything that can overlap with painting should.

## Domain Model Invariants

`Scene` is an image plus a graph of `Beat` nodes. `Beat.next` is either `continue` or `choice`. A scene should have at least one meaningful exit toward a new scene.

`StoryState` has stable and volatile zones. Stable fields are set by Architect and must not be patched by Writer: `logline`, `genreTags`, `protagonist`, `castNotes`. Volatile fields may be rewritten every scene: `synopsis`, `openThreads`, `relationships`, `nextHook`. If adding a field, classify it and update `applyStoryStatePatch()` plus Writer coercion.

Characters are identified by `name`. `mergeCharacters()` preserves existing portrait and voice fields when a later design omits them. Do not casually change character matching without checking Writer, Director, and Painter reference handling.

The player POV is hardcoded as second-person Chinese `"你"`. The player should not appear in `activeCharacters`, images, portraits, or TTS. Preserve normalization in Writer and InsertBeat flows.

## Agent Output & Error Handling

Agent outputs should follow the existing pattern:

1. Raw LLM type accepts optional and variant fields.
2. Coercion normalizes names, defaults, and malformed values.
3. Repair fixes structural issues.
4. Fallback returns a safe value instead of throwing at the agent boundary.

Never use direct `JSON.parse()` on LLM output. Use `parseJsonLoose()` from `lib/engine/jsonParser.ts`, which attempts direct parse, fenced JSON extraction, object slicing, and `jsonrepair`.

Maintain graceful degradation. Existing flows tolerate malformed AI JSON, failed character cards, failed portraits, failed TTS, failed image references, optional analytics, and provider timeouts. Do not convert optional provider failures into hard crashes.

## Visual Continuity & Prompt Caching

`sceneKey` identifies a physical space such as `"classroom-dusk"`. If a new scene shares a key with prior history, the prior scene image should be reused as a reference. Character portraits are also references.

Runware allows at most 4 references. Preserve the priority: style reference image, prior scene, speaker portrait, then other NPCs. Prefer image URLs for `referenceImages` when needed because Runware can fail to recognize UUIDs.

Writer prompt caching depends on `buildWriterUserMessage()` keeping a stable prefix: world, style, story spine, archived history, and character list. The dynamic suffix contains current state, last beat, and exit hint. Do not reorder or reformat the stable prefix casually; it can destroy cache hit rates.

## API Flow

Common routes live under `app/api/`:

- `POST /api/start`: starts a session via Architect then `directScene()`.
- `POST /api/scene`: generates the next scene from an existing session.
- `POST /api/vision`: interprets scene-image clicks.
- `POST /api/insert-beat`: creates a transient beat without image generation.
- `POST /api/beat-audio`: lazy TTS for a displayed beat.
- `POST /api/parse-style-image`: extracts a style prompt from uploaded reference art.

When changing public types or route payloads, update all route callers and client consumers in the same change.

## Build, Test, and Development Commands

Use pnpm with Node >=22. `pnpm-lock.yaml` is the source of truth; `package-lock.json` is legacy and should not be updated unless requested.

- `pnpm dev`: local Next.js dev server.
- `pnpm build`: production build for Vercel/default target.
- `pnpm start`: run production server after building.
- `pnpm lint`: Next.js built-in lint.
- `pnpm typecheck`: `tsc --noEmit`.
- `pnpm build:cf`: Cloudflare Workers build through OpenNext.
- `pnpm preview:cf`: local Cloudflare preview.
- `pnpm deploy:cf`: Cloudflare deploy.

There is no dedicated test framework, no Prettier config, and no standalone ESLint config. Before handing off code changes, run `pnpm typecheck` and `pnpm lint`; run `pnpm build` for routing, deployment, or provider initialization changes.

## Coding Style & Imports

Write TypeScript with 2-space indentation, double quotes, semicolons, and ESM imports. Prefer named exports for shared helpers and components when practical.

Use aliases from `tsconfig.json`: `@/*`, `@infiplot/engine`, `@infiplot/ai-client`, `@infiplot/tts-client`, and `@infiplot/types`. Avoid deep relative import chains when an alias exists.

React components use PascalCase. Hooks, helpers, variables, and functions use camelCase. Types and interfaces use PascalCase. Route folders follow Next.js App Router conventions. UI work should follow the existing Tailwind-heavy visual language.

Comment only non-obvious sequencing, provider quirks, fallback behavior, or architectural invariants.

## Configuration & Providers

Use `.env.example` as the source of truth. Never commit `.env.local`, API keys, uploaded user content, or generated secrets.

- Text and Vision use OpenAI-compatible endpoints: `TEXT_*`, `VISION_*`.
- Image uses Runware task-array protocol, not OpenAI-compatible images: `IMAGE_*`.
- TTS uses Xiaomi MiMo protocol and is optional: blank config means silent mode.
- `MOCK_IMAGE=true` skips image generation and returns a placeholder for cheap local iteration.
- `NEXT_PUBLIC_*` values are inlined at build time.

## File Dependency Map

If modifying Writer, also check `director.ts`, `prompts.ts`, and Cinematographer consumers. If modifying CharacterDesigner, check Director scheduling/merge logic, portrait prompts, and Painter reference collection. If modifying Cinematographer or Painter, check Director and prompt builders. If modifying Architect, check `orchestrator.ts`, `prompts.ts`, and StoryState patch rules. If modifying `lib/types/index.ts`, check all agents, Director, Orchestrator, and client consumers.

## Commit & Pull Request Guidelines

Follow observed Conventional Commit style: `feat(web): ...`, `fix(play): ...`, `perf(engine): ...`, `chore(engine): ...`.

PRs should include a short behavior summary, validation commands run, linked issues when relevant, screenshots or recordings for UI changes, and notes for environment, provider, deployment, or payload-shape changes.

## What Not To Do

- Do not make the server stateful.
- Do not generate images, portraits, or TTS for `"你"`.
- Do not let Writer patch stable `StoryState` fields.
- Do not reorder the Writer stable prompt prefix without a clear cache-aware reason.
- Do not assume Runware UUID references always work.
- Do not remove fallbacks, timeout handling, analytics privacy constraints, or reference priority rules.
- Do not regenerate large assets in `public/` unless the user requested asset work.
- Do not mix prompt refactors, provider-client rewrites, UI restyling, and deployment changes in one narrow task.
