# Design

## Context

See `proposal.md` - Why for the measured symptoms and their root causes. This section records only the shape of the code and the host constraints the approach has to fit.

**Route shape.** `CursorAdapter extends LlmAdapter`. `stream()` converts the request into a Cursor transcript, creates a `ChunkQueue` + `Reply` pair, fires `runTurn` without awaiting it and yields the queue. `runTurn` keeps one Cursor agent per session key plus parameter signature (`SESSIONS`), and a DSH tool call keeps the Cursor run alive through a custom MCP tool whose `execute` waits for DSH's result. `src/session.ts` owns the session registry, the agent factory, the tool handoff and the turn loop in one 500-line module.

**No progress channel.** DSH's `StreamChunk` union is closed: `block-start`, `text-delta`, `reasoning-delta`, `tool-call-delta`, `block-end`, `usage`, `finish`. `packages/llm/llm/src/assembler.ts` folds exactly these into the durable assistant message, first close wins, and chunks cannot be retracted. There is no custom block type, no ephemeral chunk and no adapter-visible side channel (`session/activity` and friends do not exist). Whatever an adapter streams before its first real token therefore becomes part of the assistant message that DSH stores and sends back on the next call.

**Consequences for "visible progress".** A `text` block would become assistant prose: it reaches the transcript in `prompt.ts` (`renderMessage`), the fingerprint in `prompt.ts` (`fingerprint`), the token estimates in `estimateTranscript`, and Cursor's history on the next turn. A `reasoning` block is visible in the GUI, is ignored by `fingerprint()` and by `renderMessage()` (assistant rendering keeps only `text` and `toolCall`), and is forwarded to Cursor as prior `thinking` only by `transcript.ts`. The GUI already shows a shimmering elapsed-time row for any running turn (`RunningStatus`), so the defect is adapter-side silence, not a missing spinner.

**Tier data.** The catalog entry carries `parameters` (a `context` parameter with values such as `300k` / `1m`) and, from `@cursor/sdk` 1.0.38, `variants` with `isMaxMode`. The pinned 1.0.37 has neither the typing nor the Max Mode handshake, which is why `context=1m` is rejected on the two Claude models whose 1M tier is Max-Mode only (see `proposal.md`). `LlmModelInfo` has no `contextWindow`; only `LlmResolvedModelInfo.context.contextWindow` does, and `compaction-basic` throws `TargetPressureConfigError` when a model reports no context.

**Persistence seam.** A plugin `Config` field marked `volatile()` gives the settings service a namespace for the entry, and `settings.update(entryId, patch)` writes it back through the profile; `loader/volatile-update` reports external edits. `packages/experimental/speech-to-text` is the working reference for both halves. The plugin owns no files.

**Prewarm seam.** `createAgentPlatform().prewarmLocalWorkspace(options)` builds the local executor ahead of the first `send()`. It is keyed on the options that shape that executor (working directory, api key, setting sources, sandbox, MCP servers, subagents) and returns a release function: the executor is reference counted and releasing the last reference tears it down, so the handle must be held for as long as agents may run against that workspace.

**Retry seam.** `packages/llm/llm/src/retry-policy.ts` retries only `EMPTY_RESPONSE`, `RATE_LIMIT`, `SERVER`, `TIMEOUT`, `TRANSPORT`. This route deliberately uses `CURSOR_RUN_FAILED` / `ABORTED` / `MISSING_CREDENTIAL`, and a new terminal failure must stay outside that list because a half-finished Cursor run cannot be replayed.

## Goals / Non-Goals

**Goals:**

- One explicit notion of turn start: at most one host-authored reasoning block, at most two lines, emitted before the first blocking await and before any real model output, closed by the first real chunk.
- Tier resolution as one pure function over explicit inputs (catalog entry, per-model record, global selection), with no `process.env` read inside a resolver and no fabricated fallback number.
- A prewarm lifecycle with explicit acquire and release, keyed to the working directory the turn will actually use.
- Every new behaviour observable through `CursorAdapter.stream()` in the existing fake-agent test harness.

**Non-Goals:**

- No change to the DSH streaming protocol, no request for a new block type or progress channel, no DSH internals wrapped or patched.
- No new DSH client UI work: adapter progress can only appear as assembled content.
- Not addressing Cursor-side cost (workspace resolution, model TTFT beyond our payload). Only our own avoidable work is removed and the remainder is made visible.
- No step-up retry: a tier is only ever lowered, never raised after a rejection.
- No change to tool handoff, permission prompts, or the `tools: ["mcp"]` policy.

## Decisions

### D1 - Progress is a sentinel-tagged reasoning block, stripped when replaying history

**Choice.** `Reply.progress(line)` writes into a single `reasoning` block whose text lines carry `HOST_NOTE_PREFIX` (`[cursor-to-dsh] `). It is emitted before any await that can block (catalog lookup, `Agent.create`, `send` acceptance, first token) and is closed by the first real chunk. `transcript.ts` drops assistant `reasoning` blocks that start with the prefix, so Cursor never sees host text attributed to itself.

**Rationale.** `reasoning` is the only visible block kind that the continuity machinery already ignores: `fingerprint()` skips it, so session reuse is unaffected; `renderMessage()` skips it, so the first-send transcript is unaffected. Filtering it in `transcript.ts` closes the one remaining leak. It also keeps `isVisibleChunk` semantics intact, so the GUI shows the line without any client change.

**Alternatives considered.** (a) A `text` block: visible, but it becomes assistant prose, is fingerprinted, replayed, and counted by estimates - rejected. (b) No synthetic content, rely on `RunningStatus`: the user asked for an early progress block, and the silence also hides SDK-internal retries - rejected. (c) A new block type or activity event: requires changing DSH internals, against the official-seam rule - rejected.

### D2 - Progress is not model output

**Choice.** `Reply` tracks `sawModelOutput` (real `assistant`/`thinking` content) independently from `sawHostNote`. The step-down gate and any "did the model already answer" check read `sawModelOutput`.

**Rationale.** The existing step-down condition (`freshRun && !emitted && modelCalls === 1`) must keep meaning "Cursor produced nothing yet". Emitting a progress line at turn start would otherwise silently disable tier recovery for every turn.

### D3 - Bound the silent start, and keep the failure outside DSH's retry set

**Choice.** Arm a start watchdog when the turn begins and disarm it on the first chunk. On expiry, cancel the Cursor run and finish with `CURSOR_RUN_STALLED` (a code outside `DEFAULT_RETRYABLE_CODES`). Default 120 s, overridable through config and `DSH_CURSOR_START_TIMEOUT` (`0` disables). While the turn is past a short threshold with nothing but the initial line, append a second progress line so the user sees the route is still inside Cursor's own retries.

**Rationale.** The minutes-long waits come from unbounded, invisible waiting rather than one slow step: the SDK retries transport internally, the plugin awaits `send`, and no DSH code path times out this route. A bound converts "frozen forever" into a visible, actionable error, and a non-retryable code avoids replaying a run Cursor may already have accepted.

**Alternatives considered.** (a) Declare a custom retry policy and let DSH retry: replays a possibly-accepted run and duplicates side effects - rejected. (b) Disable `enableAgentRetries` and surface raw transport errors: loses retries that do help with genuine transport blips, and the setting is per executor - rejected. (c) Do nothing and let the GUI's elapsed timer speak: unbounded waits stay unbounded - rejected.

### D4 - Tier resolution is a pure function over explicit inputs, persisted in a volatile dict

**Choice.** A single resolver takes `(catalogEntry, perModelRecord, globalSelection)` and returns `{ value, tokens, maxMode } | undefined`, with precedence: per-model record, then a concrete global selection (`min`, `max`, or a named tier), then `max`. `contextWindow` returns the resolved tier's token size or `undefined`; `resolveModel` omits `context` when none is resolvable. The per-model record lives in `Config.contextByModel` (`z.dict(z.string())`, `volatile()`) and is read and written through a small `ContextPolicy` interface (`tierFor(modelId)`, `learn(modelId, tier)`) implemented over `settings.update(entryId, ...)`, refreshed on `loader/volatile-update`, with an in-memory fallback when the settings service or the entry is absent. A dedicated control on the DSH Models page is deliberately out of scope: it lives in the DSH client, not in this repository, and is recorded as a follow-up rather than a blocker.

**Rationale.** The default flips to `max` with one token of configuration, the record survives restarts, the same field is what a user edits to pin a model, and unit tests can drive the resolver without touching `process.env` or module state. It also removes the last implicit behaviour of the route ("unknown means 200k").

**Alternatives considered.** (a) Keep the module-level `Map`: lost on restart and untestable - rejected. (b) One `Config` field per model: unbounded schema churn and a settings form that changes with the catalog - rejected. (c) A JSON file next to the profile: bypasses the settings/volatile contract and the profile's own config ownership - rejected.

### D5 - Max Mode comes from the SDK bump, not from parameter guessing

**Choice.** Pin `@cursor/sdk` to `1.3.1` - the release verified live end to end, which carries Max Mode detection (`isMaxMode` variants, `maxMode` selection; introduced in 1.0.38). Move `zod` to `peerDependencies` (`^3.25.0 || ^4.0.0`) + `devDependencies`, as the SDK requires from 1.0.38 on. When the resolved tier is only reachable through a Max Mode variant, the selection is marked accordingly; otherwise the explicit `context` value is sent.

**Rationale.** On 1.0.37 no parameter combination makes the 1M tier of `claude-sonnet-5-5` or `claude-haiku-5-5` acceptable - the request shape simply did not exist yet. Adding a `maxMode` flag on the old SDK is rejected server-side, so this is a dependency requirement rather than a code choice.

**Alternatives considered.** (a) Send `1m` and rely on step-down: the account-level defect the user reported would stay - rejected. (b) Pin 1.0.37 and document the limitation: contradicts the reported requirement - rejected.

### D6 - Prewarm is a small owned service with an explicit release

**Choice.** `src/prewarm.ts` exposes `prewarm(cwd, apiKey)` and `releaseAll()`. It calls `prewarmLocalWorkspace` with the same `local` options a real turn uses, keeps the returned release function per `cwd|apiKey` key in a small LRU (4 entries, releasing evicted ones) and releases everything on dispose. The plugin calls it at load for the default working directory and once per newly seen session directory; no turn ever awaits it.

**Rationale.** This is the documented API for exactly this purpose, it warms the executor keyed on the directory the turn will use, and holding the handle is what keeps the warmed executor alive - the current throwaway agent in `tmpdir()` does neither. Keeping it behind an interface also keeps it stubbable for tests.

**Alternatives considered.** (a) Keep the throwaway agent: warms a different executor and competes with the first real turn - rejected. (b) Call `acquireLocalExecutor` directly: same cache, lower-level, undocumented for this purpose - rejected. (c) Prewarm only on catalog load: couples warmup to a UI listing and delays it - rejected.

### D7 - Split the touched modules along seams that already exist

**Choice.** `src/session.ts` becomes a re-export barrel over `src/session/registry.ts` (slots, idle timers, `closeAllCursorSessions`, `cursorSessionCount`), `src/session/agent.ts` (`CursorSession`, `openSession`, tools sync, signature), `src/session/handoff.ts` (`planIncrement`, `continueWithNewTools`) and `src/session/turn.ts` (`runTurn`). A new `src/content.ts` becomes the single owner of block mapping between DSH content and the core transcript shape, plus `HOST_NOTE_PREFIX`; `transcript.ts`, `reply.ts` and `prompt.ts` consume it instead of each carrying its own switch.

**Rationale.** Each extracted seam is already a separate decision with its own tests, so the split is mechanical rather than a redesign, and it keeps the existing public exports (and the test harness) unchanged. The duplicated mapping is the concrete source of the "two block vocabularies drift apart" risk in a file that now also emits synthetic blocks.

**Alternatives considered.** (a) Leave `session.ts` alone and add the new behaviour there: grows a module that already mixes store, factory, handoff and loop - rejected. (b) Introduce a class hierarchy for the turn stages: more structure than the four steps need, and type-strippable source rules out some conveniences - rejected.

## Risks / Trade-offs

- [Synthetic reasoning leaks into Cursor history or into token estimates] -> prefixed lines, stripped in `transcript.ts`, capped at two short lines, and only emitted before the first real chunk.
- [A model with no readable `context` now reports no context window] -> `compaction-basic` throws on that shape, but the shipped profile does not mount it; the resolver logs a debug line naming the model, and `README.md` records the consequence so a future composition can add a configured default instead of a silent one.
- [The start watchdog cancels a run that would have finished] -> generous default, configurable, pre-first-chunk only, and it cancels before failing so no orphan run keeps spending.
- [SDK bump changes transitive behaviour] -> exact pin, `zod` as peer + dev, full suite plus `scripts/live.ts` on the bumped version, and the bump isolated in its own commit so a revert is one commit.
- [Held prewarm leases keep executors alive] -> LRU cap with release on eviction, `releaseAll()` on dispose, and warmup is best-effort so a failure never affects a turn.
- [Progress lines appear in turns that later fail] -> the message is settled as `assistant/attempt` (no message) on failure, so nothing durable is kept.

## Migration Plan

1. Bump the SDK and add the `zod` peer (one commit): `npm test`, `npm run check`, `npm run check:public`, then the live script asserting `context=1m` finishes on `claude-sonnet-5-5` and `claude-haiku-5-5`.
2. Land tier policy + persistence, with tests written against the new default.
3. Land progress + watchdog.
4. Land the prewarm service and the plugin wiring.
5. Land the structural split last, behaviour-neutral, so the tests written in steps 2-4 pin the behaviour first.
6. Update `README.md`, `cordis.patch.yml` and the version (minor bump: the default tier change is user-visible).

Rollback: `context: min`, `DSH_CURSOR_CONTEXT=min`, `DSH_CURSOR_PREWARM=0` and `DSH_CURSOR_START_TIMEOUT=0` restore the previous behaviour without a revert; a stale `contextByModel` entry is inert while the global selection names a concrete tier, and the field can be dropped from the profile. Full revert is the SDK-bump commit plus the defaults.

## Open Questions

None. Both questions this design raised were resolved before the task list was written: the SDK pins to `1.3.1`, and the per-model tier override ships as a settings field here while a control on the DSH Models page becomes a follow-up in the DSH client.
