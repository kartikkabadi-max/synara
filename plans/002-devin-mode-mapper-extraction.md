# Plan 002: Extract DevinModeMapper — testable canonical mode resolution out of the adapter

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. When done, update the status row for this plan
> in `plans/README.md`.
>
> **Drift check (run first)**: `git diff --stat f572445..HEAD -- apps/server/src/provider/Layers/DevinAdapter.ts apps/server/src/provider/acp/`
> If any in-scope file changed since this plan was written, compare the
> "Current state" excerpts against the live code before proceeding; on a
> mismatch, treat it as a STOP condition. NOTE: if Plan 001 already landed,
> the surrounding adapter code will differ slightly (model alias helpers
> removed) — that specific drift is expected and fine; the mode functions
> below must still match.

## Status

- **Priority**: P2
- **Effort**: S
- **Risk**: LOW
- **Depends on**: none (composes with 001; either order works)
- **Category**: tech-debt
- **Planned at**: commit `f572445`, 2026-06-10

## Why this matters

The locked architecture for the Devin PR says: Devin mode selection is
**runtime-discovered** (the ACP session advertises its modes), and Synara applies
a **small canonical mapper** from Synara's `runtimeMode`/`interactionMode` onto
the session-exposed mode list. That mapper currently lives as four private
functions inside the 875-line `DevinAdapter.ts`, where its policy (alias tables,
full-access → code fallback) is only testable through the whole adapter. The
target shape in the approved architecture names `DevinModeMapper` as its own
concern-specific module. Extraction makes the policy explicit, directly testable,
and keeps the adapter to lifecycle + event projection only.

## Current state

All in `apps/server/src/provider/Layers/DevinAdapter.ts`:

```ts
// DevinAdapter.ts:57-59
const DEVIN_PLAN_MODE_ALIASES = ["plan"];
const DEVIN_FULL_ACCESS_MODE_ALIASES = ["bypass", "bypass permissions"];
const DEVIN_CODE_MODE_ALIASES = ["accept-edits", "code", "accept edits"];
```

```ts
// DevinAdapter.ts:167-200
function normalizedModeText(value: string): string {
  return value
    .trim()
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, " ");
}

function findDevinModeByAliases(
  modes: ReadonlyArray<AcpSessionMode>,
  aliases: ReadonlyArray<string>,
): AcpSessionMode | undefined {
  const normalizedAliases = aliases.map(normalizedModeText);
  return modes.find((mode) => {
    const haystack = normalizedModeText(`${mode.id} ${mode.name} ${mode.description ?? ""}`);
    return normalizedAliases.some((alias) => haystack.includes(alias));
  });
}

function resolveDevinModeId(input: {
  readonly modes: ReadonlyArray<AcpSessionMode>;
  readonly runtimeMode: RuntimeMode;
  readonly interactionMode?: ProviderInteractionMode | undefined;
}): string | undefined {
  if (input.interactionMode === "plan") {
    return findDevinModeByAliases(input.modes, DEVIN_PLAN_MODE_ALIASES)?.id;
  }
  if (input.runtimeMode === "full-access") {
    return (
      findDevinModeByAliases(input.modes, DEVIN_FULL_ACCESS_MODE_ALIASES)?.id ??
      findDevinModeByAliases(input.modes, DEVIN_CODE_MODE_ALIASES)?.id
    );
  }
  return findDevinModeByAliases(input.modes, DEVIN_CODE_MODE_ALIASES)?.id;
}
```

```ts
// DevinAdapter.ts:202-225
function applyDevinModeSelection(input: {
  readonly runtime: AcpSessionRuntimeShape;
  readonly threadId: ThreadId;
  readonly runtimeMode: RuntimeMode;
  readonly interactionMode?: ProviderInteractionMode | undefined;
}): Effect.Effect<void, ProviderAdapterError> {
  return Effect.gen(function* () {
    const modeState = yield* input.runtime.getModeState;
    if (!modeState) return;
    const modeId = resolveDevinModeId({
      modes: modeState.availableModes,
      runtimeMode: input.runtimeMode,
      ...(input.interactionMode ? { interactionMode: input.interactionMode } : {}),
    });
    if (!modeId || modeId === modeState.currentModeId) return;
    yield* input.runtime
      .setMode(modeId)
      .pipe(
        Effect.mapError((error) =>
          mapAcpToAdapterError(PROVIDER, input.threadId, "session/set_mode", error),
        ),
      );
  });
}
```

Call sites: `applyDevinModeSelection` at `DevinAdapter.ts:407` (startSession) and
`DevinAdapter.ts:598` (sendTurn). Types involved: `AcpSessionMode` from
`../acp/AcpRuntimeModel.ts`, `AcpSessionRuntimeShape` from `../acp/AcpSessionRuntime.ts`,
`RuntimeMode` / `ProviderInteractionMode` / `ThreadId` from `@t3tools/contracts`,
`mapAcpToAdapterError` from `../acp/AcpAdapterSupport.ts`, `ProviderAdapterError`
from `../Errors.ts`.

Existing behavioral test: `DevinAdapter.test.ts:239-283`
("applies Devin plan mode through ACP session/set_mode") — must keep passing unchanged.

Conventions: provider-specific ACP helpers live in `apps/server/src/provider/acp/`
(see `DevinAcpSupport.ts`, `CursorAcpSupport.ts`) with a `@module` doc comment and a
colocated `*.test.ts`. Relative imports use explicit `.ts` extensions.

## Commands you will need

| Purpose           | Command (repo root `/tmp/synara-pr`)                                   | Expected on success |
| ----------------- | ---------------------------------------------------------------------- | ------------------- |
| Install           | `bun install`                                                          | exit 0              |
| Single test file  | `bunx vitest run apps/server/src/provider/acp/DevinModeMapper.test.ts` | all pass            |
| Adapter tests     | `bunx vitest run apps/server/src/provider/Layers/DevinAdapter.test.ts` | all pass            |
| Final gate (once) | `bun fmt && bun lint && bun typecheck`                                 | all exit 0          |

NEVER run `bun test`; always `bun run test` / `bunx vitest run <file>`.

## Scope

**In scope** (only files to modify/create):

- `apps/server/src/provider/acp/DevinModeMapper.ts` (create)
- `apps/server/src/provider/acp/DevinModeMapper.test.ts` (create)
- `apps/server/src/provider/Layers/DevinAdapter.ts` (delete moved code, add import)

**Out of scope** (do NOT touch):

- `AcpRuntimeModel.ts`, `AcpSessionRuntime.ts` — the mapper consumes their types only.
- Mode behavior itself: this is a pure move. Do not "improve" alias lists, matching
  rules, or fallback order. Any behavior change is out of scope.
- Other adapters (Cursor/Grok also do mode handling — leave them alone).

## Git workflow

- Branch: current PR branch `devin-acp-provider-v2`.
- One commit: `Extract DevinModeMapper from DevinAdapter`.
- Do NOT push unless instructed.

## Steps

### Step 1: Create `apps/server/src/provider/acp/DevinModeMapper.ts`

Move the three alias constants and four functions verbatim (modulo exports). Module
header:

```ts
/**
 * DevinModeMapper - maps Synara runtime/interaction modes onto the mode list
 * the Devin ACP session advertises at runtime.
 *
 * The session's `availableModes` is the source of truth; the alias tables here
 * are matching heuristics only (Devin may rename/describe modes differently
 * across versions), not a mode catalog.
 *
 * @module DevinModeMapper
 */
```

Export `resolveDevinModeId` and `applyDevinModeSelection`; keep
`normalizedModeText` and `findDevinModeByAliases` unexported (test them through
`resolveDevinModeId`). The alias constants may stay unexported.
`applyDevinModeSelection` needs the provider literal for error mapping — add a
`provider: ProviderKind` field to its input (or a `"devin"` literal constant in
the module; prefer the literal constant `const PROVIDER = "devin" as const;` to
keep the call sites unchanged in shape). Imports needed:
`Effect` from `effect`; `type { ProviderInteractionMode, RuntimeMode, ThreadId }`
from `@t3tools/contracts`; `mapAcpToAdapterError` from `./AcpAdapterSupport.ts`;
`type { AcpSessionMode }` from `./AcpRuntimeModel.ts`;
`type { AcpSessionRuntimeShape }` from `./AcpSessionRuntime.ts`;
`type { ProviderAdapterError }` from `../Errors.ts`.

**Verify**: defer compile check to step 3 test run.

### Step 2: Update `DevinAdapter.ts`

Delete lines 57-59 (alias constants) and the four function definitions
(167-225). Add `import { applyDevinModeSelection } from "../acp/DevinModeMapper.ts";`.
The two call sites (407 and 598) keep working unchanged because the function
signature is preserved. Remove now-unused imports if any (`mapAcpToAdapterError`
is still used elsewhere in the adapter at lines 221/399/417/etc. — check with
grep before removing anything).

**Verify**: `grep -n "resolveDevinModeId\|findDevinModeByAliases\|normalizedModeText\|DEVIN_PLAN_MODE_ALIASES" apps/server/src/provider/Layers/DevinAdapter.ts` → no matches.

### Step 3: Write `DevinModeMapper.test.ts`

Plain `describe`/`it`/`assert` from `@effect/vitest` for `resolveDevinModeId`
(pure function); one `it.effect` for `applyDevinModeSelection` using a minimal
mock runtime (copy the `getModeState`/`setMode` parts of `makeMockRuntime` from
`DevinAdapter.test.ts:19-70`). Cases for `resolveDevinModeId`:

- plan interactionMode picks the mode whose name matches "plan" (e.g. modes `[{id:"code",name:"Code"},{id:"planning",name:"Plan"}]` → `"planning"`).
- full-access picks a "bypass permissions" mode when present.
- full-access falls back to the code mode when no bypass mode exists.
- default (approval-required, no interactionMode) picks the code/accept-edits mode.
- alias matching is punctuation/case-insensitive (mode named `"Accept Edits!"` matches).
- returns `undefined` when nothing matches.

Cases for `applyDevinModeSelection`:

- no-op (no `setMode` call) when resolved mode equals `currentModeId`.
- no-op when `getModeState` yields `undefined`.
- calls `setMode("planning")` for plan interactionMode (mirrors the adapter test at `DevinAdapter.test.ts:239-283`).

**Verify**: `bunx vitest run apps/server/src/provider/acp/DevinModeMapper.test.ts` → all pass.

### Step 4: Confirm adapter behavior unchanged, then final gate

`bunx vitest run apps/server/src/provider/Layers/DevinAdapter.test.ts` → all pass
(especially "applies Devin plan mode through ACP session/set_mode").
Then once: `bun fmt && bun lint && bun typecheck`.

**Verify**: all exit 0.

## Test plan

See step 3. Pattern exemplars: `DevinAdapter.test.ts` (mock runtime), any pure-function
test in `apps/server/src/provider/acp/` such as `AcpRuntimeModel.test.ts`.

## Done criteria

- [ ] `apps/server/src/provider/acp/DevinModeMapper.ts` exists; `DevinAdapter.ts` contains no mode-alias constants or mode-resolution functions
- [ ] `bunx vitest run apps/server/src/provider/acp/DevinModeMapper.test.ts apps/server/src/provider/Layers/DevinAdapter.test.ts` → all pass (≥9 new mapper tests)
- [ ] `bun fmt && bun lint && bun typecheck` → exit 0 (single final pass)
- [ ] `git status` clean outside the in-scope list
- [ ] `plans/README.md` status row updated

## STOP conditions

- The mode functions at the cited lines don't match the excerpts (beyond the expected Plan-001 drift described in the drift check).
- You find a second caller of these functions outside `DevinAdapter.ts` (grep first: `grep -rn "resolveDevinModeId\|applyDevinModeSelection" apps/ packages/`).
- Preserving behavior requires changing `AcpSessionRuntime` or contracts types.

## Maintenance notes

- Plan 004 (slash commands) and Plan 003 (user input) also touch `DevinAdapter.ts`; land this small move first or rebase carefully — the diff is mostly deletions.
- If Devin later renames its ACP modes, only the alias tables in this module change; add a regression test capturing the new names when that happens.
- Reviewer should check: pure move, no behavior delta (diff should show relocation, an import, and tests only).
