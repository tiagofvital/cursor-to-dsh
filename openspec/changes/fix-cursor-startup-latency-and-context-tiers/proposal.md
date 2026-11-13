# Proposal

## Why

Users report two symptoms with the `cursor` route: agents sometimes appear frozen for a long time before any output, and the route cannot reach the 1M context window that the Cursor app offers on the same models. Both are real defects, not user error.

Measured against the live API with the pinned `@cursor/sdk` 1.0.37:

- Nothing is emitted to DSH until the first Cursor model chunk arrives. A cold turn is silent for 4.5–5.5 s; on a 4.1 GB repository every turn is silent for ~3 s; a 234k-token transcript is silent for ~7.3 s. The plugin emits no frame while `Agent.create`, `send()` or time-to-first-token are pending.
- That window has no upper bound. The SDK's own transport retries (`enableAgentRetries` defaults to on) happen inside `send()`/`wait()` and are invisible to DSH, and neither DSH nor this plugin applies an inactivity timeout to the route. Repeated `fresh` session restarts each repay create + send + full-transcript TTFT, so minutes of silence are reachable by compounding rather than by one slow step.
- The eager prewarm at plugin load runs a throwaway agent in `tmpdir()/cursor-to-dsh-warmup`. The SDK caches executors per working directory, so it warms a different executor than the session uses and pays its workspace scan for nothing — and on a large repository it competes with the first real turn.
- The context tier defaults to `min`, so the smallest offered tier is requested, and `contextWindow` silently falls back to `200_000` for every model without a readable `context` param. The 1M tier is never requested by default.
- When a tier is rejected, the plugin steps down in memory for the life of the process only, and only on a first-call, output-free run.
- `context=1m` is rejected outright for `claude-sonnet-5-5` and `claude-haiku-5-5` on SDK 1.0.37. Those models' 1M variants require Cursor Max Mode; the SDK learned to detect and send that in **1.0.38** (`isMaxMode` on variants, `maxMode` on the selection). With 1.0.38+ the same params succeed. This is the whole reason the user sees 1M in the Cursor app but not through this route.

## What Changes

- **Context tier default becomes `max`** — the largest tier a model offers — with a per-model override honored ahead of it. The hidden `200_000` fallback is removed: a model with no readable tier reports its tier or none, not a fabricated number.
- **A rejected tier steps down to the next smaller offered tier and is remembered per model** across process restarts, instead of living in a process-global `Map`.
- **A visible progress block is emitted before the first Cursor chunk**, so a starting turn is never silent. It is replaced by real content as soon as it arrives.
- **Prewarm runs eagerly at plugin load against the session working directory**, through the SDK's own `prewarmLocalWorkspace`, whose release handle is held for the process lifetime instead of being dropped.
- **`@cursor/sdk` is bumped from 1.0.37 to 1.3.1** so Max Mode variants are accepted, and `zod` becomes a peer + dev dependency as the new SDK requires.
- **BREAKING (behavioral)**: the route now requests the largest context tier by default, which changes cost and latency characteristics for every user who relied on the implicit smallest tier. `DSH_CURSOR_CONTEXT`/`context` still select a smaller tier explicitly.
- Cleanup shipped alongside, because the touched code is where the duplication lives: `src/session.ts` loses its mixed responsibilities around run lifecycle vs. handoff, the role/block mapping duplicated across `transcript.ts`/`prompt.ts`/`reply.ts` gains one owner, and `models.ts` stops reading `process.env` from inside a pure resolver.

## Capabilities

### New Capabilities

- `cursor-model-selection`: how the route turns a Cursor catalog entry into the parameters Cursor receives and the context window DSH is told about — tier resolution and its default, per-model overrides, learning from rejections, and reasoning/effort mapping.
- `cursor-turn-startup`: what a starting turn makes observable and how the warm path is prepared — progress before the first model chunk, and the prewarm lifecycle keyed to the real working directory.

### Modified Capabilities

None. The project has no existing specs (`openspec list --specs` is empty); this is the first OpenSpec change here.

## Impact

- `src/index.ts` — `context` default, prewarm wiring, eager prewarm at load, cwd plumbing for prewarm.
- `src/adapter.ts` — `resolveModel` context reporting, prewarm deps, catalog-load side effects.
- `src/models.ts` — `preferredContext` default and env coupling, `contextWindow` fallback, `workingContext` persistence, `modelParams` Max Mode awareness.
- `src/sdk.ts` — replace the throwaway warmup agent with `prewarmLocalWorkspace`, hold and release the handle.
- `src/session.ts` — step-down path, persistence hook, startup progress emission, responsibility split.
- `src/reply.ts` — progress frame emission and its replacement semantics.
- `package.json` / `package-lock.json` — `@cursor/sdk` bump, `zod` peer + dev entry.
- `cordis.patch.yml` and `README.md` — documented `context` default changes from `min` to `max`.
- Tests: `test/models.test.ts` asserts the `200k`/`200_000` behavior and must change; `test/session.test.ts` needs startup-progress and step-down-persistence coverage through `CursorAdapter.stream()`.
- Compatibility: route id `cursor`, model ids, the `llm-cursor` row id and `DSH_CURSOR_*` variables stay stable. The `context` config keeps accepting `min`/`max`/explicit tiers.
- Dependency risk: the SDK bump to 1.3.1 moves `zod` from `dependencies` to `peerDependencies`; DSH supplies zod 4.4.3, inside the declared `^3.25.0 || ^4.0.0` range.
