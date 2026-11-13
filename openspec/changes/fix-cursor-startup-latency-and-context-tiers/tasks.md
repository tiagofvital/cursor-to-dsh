# Tasks

## 1. Dependency bump (SDK 1.3.1 + zod peer)

- [ ] 1.1 Pin `@cursor/sdk` to `1.3.1` in `package.json` and declare `zod` as `peerDependencies` (`^3.25.0 || ^4.0.0`) plus `devDependencies`, matching the SDK's own peer range; verify `npm install` succeeds and `npm ls zod` resolves a single version against the DSH host
- [ ] 1.2 Confirm no source change is needed for the bump: run `npm test` and `npm run check` on the pinned version; both pass unchanged (the new SDK keeps `Agent.create`, `customTools`, `prewarmLocalWorkspace` and `settingSources`)
- [ ] 1.3 Prove the 1M rejection is gone on the pinned version with `scripts/live.ts` (key from the DSH credential store): a request carrying `[{id:'context', value:'1m'}]` reports `finished` for `claude-sonnet-5-5` and `claude-haiku-5-5`, not `Invalid parameters for registry model`
- [ ] 1.4 Record the version in `README.md` prerequisites and note that Max Mode tiers require this SDK line; verify the documented install command matches `package.json`

## 2. Single owner for DSH-to-transcript block mapping

- [ ] 2.1 Create `src/content.ts` with the block/role mapping used by `transcript.ts`, `reply.ts` and `prompt.ts`, plus `HOST_NOTE_PREFIX` and `isHostNote(block)`; verify `npm run check` passes with the three modules importing from it and no duplicated switch left
- [ ] 2.2 Make `transcript.ts` drop assistant `reasoning` blocks recognised by `isHostNote`, and verify a unit test asserts a prefixed reasoning block never reaches the transcript while ordinary text/reasoning is preserved
- [ ] 2.3 Verify the refactor is behaviour-neutral: `npm test` passes with the existing transcript/reply/prompt expectations unchanged

## 3. Tier policy as a pure resolver, `max` by default

- [ ] 3.1 Replace the module-level `contextPreference` and the `process.env` read inside `preferredContext` with a resolver taking `(catalogEntry, perModelRecord, globalSelection)` and returning `{ value, tokens, maxMode } | undefined`; verify unit tests drive every precedence case without touching `process.env` or module state
- [ ] 3.2 Default the global selection to `max` and drop the invented `200_000` fallback so an unresolvable tier omits the `context` parameter; verify `test/models.test.ts` asserts the largest tier is chosen by default, a single-tier model gets its only value, and a model without a context parameter gets no `context` param
- [ ] 3.3 Map `min`, `max` and named tiers (`300k`, `1m`) through the resolver with an unoffered value falling back to the largest offered tier; verify unit tests cover all three, including that the unoffered value is never sent
- [ ] 3.4 Make `contextWindow` in `resolveModel` report the resolved tier's token size and omit `context` entirely when nothing resolves; verify a `CursorAdapter.stream()` test in `test/session.test.ts` observes the resolved model info (update the existing `300_000` assertion to the new default) and a no-tier model omits `context`
- [ ] 3.5 Update `README.md` context table and the `context: min` sample in `cordis.patch.yml` for the new default, and verify the documented values (`min`, `max`, `300k`) all resolve through the resolver

## 4. Per-model tier override persisted through settings

- [ ] 4.1 Add `contextByModel` to `Config` as `z.dict(z.string())` marked `volatile()`, and read it in `resolveModel`/model params through a `ContextPolicy` interface (`tierFor`, `learn`) with an in-memory fallback when the settings service or entry id is absent; verify unit tests cover both the settings-backed and the fallback path
- [ ] 4.2 Implement `learn(modelId, tier)` as `settings.update(entryId, { contextByModel: {...} })` with `entryId` from `ctx.fiber.entry?.options.id`, refreshed on `loader/volatile-update`; verify a `CursorAdapter.stream()` test asserting a rejected tier is written to the namespace and survives a fresh adapter instance built from the same settings state
- [ ] 4.3 Delete the process-global `workingContext` map and route the step-down at `session.ts` through the policy; verify the existing step-down test (`test/session.test.ts`, rejects-unless-`1m` fake agent) passes without a module-level map
- [ ] 4.4 Document `contextByModel` in `README.md` (what auto-writes it, how to pin a model, how to clear it) and verify the documented key matches the `Config` field name

## 5. Max Mode selection

- [ ] 5.1 Mark the resolved selection when the requested tier is only reachable through a Max Mode variant (catalog `variants[].isMaxMode`), and send `maxMode` only in that case; verify `npm run check` accepts the new SDK typings and a unit test asserts the flag is absent for ordinary tiers
- [ ] 5.2 Verify end to end through `CursorAdapter.stream()`: a `claude-sonnet-5-5` request at the top tier sends the Max Mode selection and no longer triggers a step-down, using the fake agent to assert the exact `model.params`/`maxMode` shape
- [ ] 5.3 Extend `scripts/live.ts` to assert one Max-Mode-only 1M model and one ordinary-tier model both finish, and record the observed result in the change notes

## 6. Visible progress before the first model chunk

- [ ] 6.1 Add `Reply.progress(line)` emitting a single `reasoning` block whose lines carry `HOST_NOTE_PREFIX`, closed by the first real chunk; verify a `CursorAdapter.stream()` test observes a progress chunk before any model content and exactly one closed block
- [ ] 6.2 Emit it before the first blocking await of the turn (catalog lookup, `Agent.create`, `send` acceptance) so a cold start shows a line immediately; verify a test with a deliberately slow fake `createAgent` observes progress while `stream()` is still being consumed
- [ ] 6.3 Track `sawModelOutput` separately from host notes and use it for the step-down gate and any "model already answered" check; verify a test asserting tier step-down still fires on a rejected first run that already emitted progress
- [ ] 6.4 Verify nothing durable leaks: a failed turn settles as an attempt with no assistant message, and the host note never enters `fingerprint()` or the replayed transcript (extend the tests from 2.2 and 6.1)

## 7. Bounded silent start

- [ ] 7.1 Add a start watchdog armed at turn start and disarmed on the first chunk, cancelling the Cursor run and finishing with `CURSOR_RUN_STALLED` on expiry; verify a test with a fake agent that never emits asserts the cancel call happens and the finish reason is `error` with that code
- [ ] 7.2 Make the timeout configurable (config field defaulting to 120 s plus `DSH_CURSOR_START_TIMEOUT`, `0` disables) and verify tests for a custom timeout and for `0` leaving the turn unbounded
- [ ] 7.3 Emit a second progress line once the turn is past a short threshold with no model output, so SDK-internal retries are visible; verify a test asserts the second line appears and that it still does not count as model output
- [ ] 7.4 Confirm `CURSOR_RUN_STALLED` stays outside DSH's retry list and document the variable in `README.md`; verify `grep` over `src/` shows the code is not in `DEFAULT_RETRYABLE_CODES` and the README row matches the config field

## 8. Prewarm the real working directory

- [ ] 8.1 Add `src/prewarm.ts` exposing `prewarm(cwd, apiKey)` and `releaseAll()` over `createAgentPlatform().prewarmLocalWorkspace({...local options})`, keeping release functions per `cwd|apiKey` in a 4-entry LRU that releases evicted entries; verify unit tests with a stubbed platform assert one call per key, the release on eviction, and `releaseAll()` releasing the rest
- [ ] 8.2 Wire it at plugin load for the resolved working directory and once per newly seen session directory, never awaited by a turn; verify a plugin-level test asserts preparation starts with no turn and with no catalog listing
- [ ] 8.3 Keep `DSH_CURSOR_PREWARM=0|false|off` as the off switch and contain failures; verify tests for disabled-by-config, disabled-by-env, and a rejecting `prewarmLocalWorkspace` all leave turns working
- [ ] 8.4 Retry preparation when a key only becomes available later, release everything in the existing `ctx.effect` teardown, and verify a test asserting no lease survives disposal
- [ ] 8.5 Remove the throwaway-agent warmup in `src/sdk.ts` (`prewarmCursor`, `resetPrewarm`) now that a real one exists; verify `npm test` passes with the old path gone and `README.md` describes the new behaviour

## 9. Split `src/session.ts` along its existing seams

- [ ] 9.1 Move the registry into `src/session/registry.ts` (slots, idle timers, `closeAllCursorSessions`, `cursorSessionCount`), the agent factory into `src/session/agent.ts`, handoff into `src/session/handoff.ts` and the loop into `src/session/turn.ts`, keeping `src/session.ts` as a re-export barrel with unchanged public exports; verify `npm run check` and `npm test` pass and `test/session.test.ts` imports resolve unchanged
- [ ] 9.2 Verify the split is behaviour-neutral by keeping every test from groups 3-8 green without edits to their assertions

## 10. Integration checks

- [ ] 10.1 Bump the package version (minor: the default tier is user-visible) and verify `npm run check:public` reports no secrets, internal links or local absolute paths
- [ ] 10.2 Run the full gate - `npm test`, `npm run check`, `npm run build`, `npm run check:public` - and verify all four succeed on the final tree
- [ ] 10.3 Run `scripts/live.ts` against the real API on the final tree and record cold-start and second-turn timings plus the resolved tier per model in the change notes

## Workflow follow-up

- A control for `contextByModel` on the DSH Models page is a follow-up in the DSH checkout, not this change; plan it there once the settings field ships.
- Archive the change after review requirements are satisfied and verify the archived result with `openspec validate --strict`.
