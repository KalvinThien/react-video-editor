# CLAUDE.md

Operating manual for Claude Code sessions in this repository.

## Project Identity

`remotion-captions` (package name; README title "React Video Editor") is a **browser-only React 18 + TypeScript timeline video editor** (Vite SPA) built on `@designcombo/*` (timeline/state/events/types) and Remotion (`@remotion/player`) for preview. It is derived from `designcombo/react-video-editor`.

**There is no gateway, queue, worker, Python code, browser extension, Dockerfile, or "Dola30" component in this repository** (verified by file listing and `grep -i dola|gateway|chrome\.` over `src/`). The only backend-facing code is a handful of `fetch` calls to endpoints that are **not implemented here** (see "Backend Contract"). If a task refers to a Dola render gateway / Dola30 extension, that code lives elsewhere (or must be created); confirm with the user before inventing it.

## Architecture

- `src/main.tsx` -> `src/app.tsx` (loads fonts into store) -> `src/features/editor/editor.tsx` (layout: menu list, scene, timeline, control panel, navbar).
- State: zustand stores in `src/features/editor/store/` plus `@designcombo/state` (`StateManager`) and `@designcombo/events` (event bus; names in `constants/events.ts`).
- Timeline: `src/features/editor/timeline/` (`@designcombo/timeline` canvas, custom item renderers in `timeline/items/`).
- Preview/playback: `src/features/editor/player/` (Remotion `Player`, `composition.tsx`, `sequence-item.tsx`, animations in `player/animated/`).
- Scene/canvas interaction: `src/features/editor/scene/` (moveable/selection).
- Properties panel: `src/features/editor/control-item/`; asset/menu panels: `src/features/editor/menu-item/`; sample data in `src/features/editor/data/`.
- UI primitives: `src/components/ui/` (shadcn/radix), color picker in `src/components/color-picker/`.
- Path alias `@` -> `src/` (`vite.config.ts`, `tsconfig.app.json`).

## Important Files

| Component | File | Purpose |
|---|---|---|
| Entry | `src/main.tsx`, `src/app.tsx` | Bootstrap, font data load |
| Editor shell | `src/features/editor/editor.tsx` | Main layout and event wiring |
| Export / render client | `src/features/editor/store/use-download-state.ts` | POST `/api/render`, poll status, set output URL |
| Export UI | `src/features/editor/download-progress-modal.tsx`, `navbar.tsx` | Progress modal, export trigger |
| Voice list | `src/components/voice-over/select-options.tsx` | GET `/api/voices?language=` |
| Upload | `src/utils/upload.ts`, `src/components/ui/file-uploader.tsx` | Presigned URL from `transcribe.designcombo.dev`, then upload |
| Download helper | `src/utils/download.ts` | Fetch URL -> browser download |
| File utils | `src/features/editor/utils/file.ts` | Fetch media as blob/text |
| Stores | `src/features/editor/store/use-store.ts`, `use-data-state.ts`, `use-layout-store.ts`, `use-crop-store.ts` | Editor state |
| Event names | `src/features/editor/constants/events.ts` | `@designcombo/events` keys |
| Player | `src/features/editor/player/player.tsx`, `composition.tsx` | Remotion preview |
| Timeline | `src/features/editor/timeline/timeline.tsx` | Timeline canvas and controls |
| Build config | `vite.config.ts`, `tsconfig*.json`, `eslint.config.js`, `tailwind.config.js`, `.prettierrc` | Tooling |

## Request Lifecycle (the only one that exists)

Video export, as implemented in `use-download-state.ts`:

1. `startExport()` sets `exporting`, opens progress modal; requires `payload` (an `IDesign`) in the store.
2. `POST /api/render` with `{ design, options: { fps: 30, size, format: "mp4" } }`; expects `{ video: { id } }`.
3. Polls `GET /api/render?id=<id>&type=VIDEO_RENDERING` every 2.5 s; expects `{ video: { status, progress, url } }`.
4. `status === "COMPLETED"` -> store `output = { url, type }`. `PENDING` -> poll again.

Known client gaps (observed in code, not yet fixed): any other status (e.g. FAILED) stops polling without clearing `exporting`; errors are only `console.error`ed; no timeout.

`/api/*` is relative: there is **no proxy in `vite.config.ts` and no server in this repo**, so these calls 404 under `pnpm dev` unless a backend is supplied. That is the current blocker for export/voices (missing backend, not a policy issue).

## Dola30 Architecture

Not present in this repository. No manifest, content/background scripts, message passing, or gateway client exist. Do not document or assume any until the code is added or its location is given. If asked to build one, treat it as new work: agree the gateway API contract first, keep it within documented/authorized APIs and sessions.

## Development Rules

- Inspect before modifying; prefer minimal, reversible changes.
- Reuse existing architecture (zustand stores, `@designcombo` events, shadcn UI); do not rewrite working components or swap libraries without reason.
- Preserve backward compatibility of the `/api/render` request/response shape unless the backend is changed in tandem.
- Do not invent undocumented APIs or endpoints; document assumptions explicitly.
- Keep error handling (and improve it where touched); never swallow errors silently.
- Validate with typecheck/lint/build; inspect browser console and network tab.

## Agent Workflow

1. Understand 2. Locate 3. Plan 4. Modify 5. Test 6. Debug 7. Verify 8. Report.

## Policy Boundary

Comply with applicable platform, tool and API policies. Do not bypass safety systems, authentication, authorization, rate limits or access controls. If one operation is restricted, state exactly which one, document the boundary, and continue the remaining legitimate work instead of abandoning the task. Never claim something works unless verified.

## Testing

No test framework or test files exist (no vitest/jest, no `test` script, no CI). Validation available:

- `pnpm lint` (`eslint .`)
- `pnpm build` (`tsc -b && vite build`) - typecheck + production build
- Manual: `pnpm dev`, exercise the editor in a browser.

If adding tests, add the framework and script explicitly and document it here.

## Running The Project

```bash
pnpm install     # pnpm-lock.yaml is the lockfile; node_modules is not committed
pnpm dev         # Vite dev server, http://localhost:5173
pnpm build
pnpm preview
pnpm lint
```

## Configuration

- No `.env` files, no `import.meta.env`/`process.env` usage in `src/`.
- Hard-coded: upload presign URL `https://transcribe.designcombo.dev/presigned-url` (`src/utils/upload.ts`); relative `/api/render`, `/api/voices`.
- Dev port: Vite default 5173. No credentials or session config exist. Never put secrets in this file.

## Debugging

- Browser devtools: Network tab for `/api/render` polling and `/api/voices`; Console for `console.error` output from the export store.
- `pnpm build` surfaces type errors; `pnpm lint` surfaces lint errors.
- Event flow: grep `constants/events.ts` names to trace `@designcombo/events` dispatch/subscribe.

## Change Management

Small, single-purpose commits; avoid unrelated reformatting; note any behavior change to the API contract in the commit message.

## Completion Criteria

Done only when: code changed; `pnpm build` (types) and `pnpm lint` checked; relevant runtime behavior verified in browser/network where applicable; errors investigated; changes summarized (What Changed / Why / Testing / Remaining Issues / Next Step).
