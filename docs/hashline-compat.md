# pi-hashline-edit-pro × pi-plan-build compatibility conflict report

Investigated 2026-09-13 by read-only source inspection plus session-state forensics. No runtime instrumentation was used.

**Conclusion: the conflicts are real. There are four, one of them high severity (Plan mode's read-only guarantee can be bypassed). Syncing to upstream 0.1.94 does not fix any of them — the released 0.1.94 commit is byte-identical to the version inspected here, so every defect below lives in upstream code.**

## 1. Scope and versions

| Component | Version / commit | Location |
| --- | --- | --- |
| pi-plan-build (running) | 0.1.94 `f9a42e2` | `/home/k/.pi/agent/npm/node_modules/@janvitos/pi-plan-build/index.ts` (1264 lines) |
| pi-plan-build (this repo, `main`) | 0.1.94 `f9a42e2` | this repo's `index.ts`; `git show HEAD:index.ts` is **diff-identical** to the installed package file above |
| pi-hashline-edit-pro | 4.2.6 | `/home/k/.pi/agent/npm/node_modules/pi-hashline-edit-pro/` |
| pi `@earendil-works/pi-coding-agent` | 0.85.1 | `dist/core/extensions/types.d.ts` |

Extension load order comes from the `packages` array in `/home/k/.pi/agent/settings.json`: `pi-hashline-edit-pro` is line 12, `@janvitos/pi-plan-build` is line 16, so **hashline loads first**. Pi guarantees that event handlers run in extension load order (`docs/extensions.md`; the `tool_call` section also states "Later `tool_call` handlers see mutations made by earlier handlers"). The observations in this report match that order (see §7).

Pi currently exposes **no tool semantics** that would let a mode-style extension decide whether a given tool writes files. `ToolDefinition` carries only `name/label/description/promptSnippet/promptGuidelines/parameters/constrainedSampling/renderShell/prepareArguments/executionMode/execute/renderCall/renderResult` (`types.d.ts:344`), and a whole-file match for `readOnly|mutating|isMutation` returns **0 hits**. The `ExtensionEvent` union (`types.d.ts:813`) has no "active tools changed" event either. All that is available is `getActiveTools()/getAllTools()/setActiveTools()` (`types.d.ts:995/997/999`).

## 2. Conflict 1: the Plan-mode read-only guard misses hashline's mutating tools (high severity)

Evidence (0.1.94 `index.ts`):

- `:1076` `if (effectiveMode !== "plan" || (event.toolName !== "edit" && event.toolName !== "write")) return;` — in Plan mode only `edit`/`write` reach the "plan file only" decision; every other tool returns before it, i.e. before the `:1081` message `Plan mode only permits edit/write access to the plan file`.
- The same hard-coded enumeration appears at `:1054` (mutations blocked while plan state is unavailable), `:1055` (`plan_task` must be alone in its batch before dependent operations), `:1061` (current and historical plan files are read-only in Build mode), and `:1070` (project mutations blocked while step-by-step execution awaits an instruction).
- hashline's mutating tools: `src/replace.ts:268` (`replace`), `src/insert.ts:165` (`insert`), `src/replace-undo.ts:96` (`undo_last_change`). Its only `tool_call` hook targets `write` (`src/write-hook.ts:37-39`, returning `[E_WRITE_HASH_ECHO]` at `:34`) and carries no mode semantics.

Reproduction: in Plan mode, call `replace` / `insert` / `undo_last_change` with a target path outside the plan file. None of the four `includes(...)` guards match those tool names.

Impact:

1. Plan mode's "read-only, do not mutate the system" promise does not hold — any workspace text file can be rewritten.
2. The Build-mode rule "current and historical plan files are read-only" (`:1061`) can be bypassed with `replace`/`insert`, so plan Markdown (step markers, completion state) can be silently rewritten.
3. The step-by-step gate (`:1070`) can be bypassed the same way: files can be changed while no step is approved.
4. The batch constraint at `:1055` (`plan_task` must precede dependent work in a separate batch) can also be bypassed.

## 3. Conflict 2: Plan mode re-adds the built-in `edit`, defeating hashline's removal

Evidence:

- hashline removes the built-in `edit` on `session_start`: `index.ts:61` (handler) and `:64` `pi.setActiveTools(active.filter((t) => t !== "edit"));`
- pi-plan-build force-adds it back in Plan mode: `index.ts:258` (`function applyTools`) and `:262` `pi.setActiveTools(unique([...base, "edit", "write", "question", "plan_exit", "plan_task"]));`
- `base` comes from `toolsBeforeModes`, and `MODE_ADDED_TOOLS` (`:81`) deliberately excludes `edit`/`write` from that snapshot — i.e. pi-plan-build treats both as mode-added tools it re-injects on every `applyTools`.

Impact: in Plan mode the built-in `edit` and hashline's `replace`/`insert` are active at the same time (§7 shows this live). That defeats hashline's reason for removing `edit` (steering the model onto anchored tools, and keeping anchor ownership plus undo state consistent). The two editing paths are not equivalent: hashline's `undo_last_change` and anchor registry only cover hashline's own tools.

## 4. Conflict 3: the persisted `toolsBeforeModes` snapshot overrides hashline's runtime toggles

Evidence:

- pi-plan-build prefers the session-state snapshot when restoring the tool set: `index.ts:1219-1222`

  ```ts
  toolsBeforeModes = Array.isArray(raw?.toolsBeforeModes)
      ? raw.toolsBeforeModes.filter((name): name is string => typeof name === "string" && !MANAGED_TOOLS.has(name))
      : pi.getActiveTools().filter((name) => !MANAGED_TOOLS.has(name));
  ```

  The snapshot is written to the session branch via `pi.appendEntry(STATE_TYPE, …)` (near `:196`), so it survives restarts.
- hashline switches between `grep` and `anchor_grep` from its config: `index.ts:63-64` (records `grepWasActive`, removes `edit`), `:82-86` (`config.anchorGrepEnabled ? t !== "grep" : t !== "anchor_grep"`), `:111` (recomputes the active set when `/hashline-config` toggles it).
- pi-plan-build calls `applyTools()` (`:258`) from many places: mode switches, `plan_task`, `before_agent_start` (`:1101`), `agent_settled` (`:1123`), `session_compact` (`:1151`).

Reproduction: once `anchor_grep` is in the snapshot, turn anchor grep off with `/hashline-config` (or on, in the other direction), then trigger any `applyTools` call.

Impact: hashline's toggle is rolled back by the snapshot (`grep` and `anchor_grep` can end up both active, or the disabled one comes back), and it reproduces after a restart because the snapshot comes from the session branch record. **It partially self-heals but not enough:** `applyTools` first calls `discoverUnmanagedTools()` (`:253`), which merges tools that are currently active and not in `MANAGED_TOOLS` into `base` — so tools hashline *adds* are picked up, but tools hashline *removes* are re-added from the stale `base`.

## 5. Conflict 4: plan-file change detection and reconciliation only recognize `edit`/`write`

Evidence: both checks inside `pi.on("tool_result")` (`index.ts:1042`) match built-in tool names only — `:1044` (calls `armReconciliation` when a non-plan file was changed) and `:1045` (refreshes the title and recomputes the tool set after the plan file changes).

Impact: after editing the plan file with hashline's `replace`/`insert`, pi-plan-build neither refreshes `savedPlanState`/the panel title nor recomputes the tool set. Changing project files with those tools never arms the "out-of-plan change → reconcile" reminder. In other words, both of pi-plan-build's self-reflection paths are blind to hashline.

## 6. Non-conflicts (verified, no action needed)

- `read`: registered by hashline (`src/read.ts:170`). pi-plan-build never touches `read` and has no read semantics of its own.
- Tool namespace: hashline registers `read`/`replace`/`insert`/`anchor_grep`/`undo_last_change` (`index.ts:39-45`); pi-plan-build registers `question`/`plan_task`/`plan_enter`/`plan_exit`/`plan_step_control`/`plan_step_complete`/`plan_complete`/`plan_finish`. No name collisions.
- Command namespace: `/hashline-config` and `/clear-anchors` do not overlap with `/plan`, `/build`, `/build-fresh`, `/plan-settings`.
- Rendering: pi-plan-build uses `registerEntryRenderer`/`registerMessageRenderer` plus `UserMessageComponent`; hashline uses its own overlay (`src/config-ui.ts`) and per-tool `renderCall`/`renderResult`. Neither overrides the other.
- `tool_call` blocking composes: hashline's write echo guard (`src/write-hook.ts`) and pi-plan-build's write guards both apply; the first-loaded one decides first, and the outcome is never incorrect (both only ever add strictness).
- Batch planning in `message_end`/`turn_end` (`src/batch.ts`) rewrites arguments only for `replace`/`insert` calls, while pi-plan-build only reads `plan_task` call arguments. They do not interfere.

## 7. Runtime evidence (this session)

- Session file: `/home/k/.pi/agent/sessions/--home-k-repos-pi-plan-build--/2026-09-13T06-39-38-232Z_01a0997e-2cf8-7758-ac82-eeaa6f3425e0.jsonl`
- The persisted pi-plan-build state record (`STATE_TYPE`) contains `toolsBeforeModes` = `["read","bash","write","ask_user_question",…,"replace","insert","anchor_grep","undo_last_change",…]` — **without `edit`**, proving hashline's `edit` removal ran first and took effect (load order in §1).
- In the same session, Plan mode (`/home/k/.pi/agent/pi-plan-build.json` → `{"defaultMode":"plan"}`) exposes **both** the built-in `edit` and hashline's `replace`/`insert`/`undo_last_change` — i.e. §3 is visible at runtime, and `edit` can only come from pi-plan-build's injection at `:262` (it is not in the snapshot).
- `grep` is absent from the tool list while `anchor_grep` is present — consistent with hashline's default `anchorGrepEnabled: true` (`src/config.ts`). This also shows that the toggle state is decided by hashline but pinned long-term by pi-plan-build's snapshot (§4).
- Live confirmation of Conflict 1 while writing this report: in Build mode with no step running, `pi-plan-build`'s own `tool_call` guard (`:1070`) refused a `write` tool call with "Step-by-step execution is waiting for an explicit natural-language instruction from the user; no step is approved for project mutations." The same restriction does not apply to hashline's `replace`/`insert`, which is exactly the asymmetry described in §2.

## 8. Current repository state (for cross-checking)

- `main` = `upstream/main` = `origin/main` = `f9a42e2` (Release 0.1.94). The `feat/default-mode` branch was left untouched (its `a7d8c28` is not the same commit as the merged upstream `466594e`).
- This report is an **untracked file**: it was not `git add`ed and not committed, so `main` still matches upstream and can keep fast-forwarding.
- Environment note (unrelated to hashline): the local `pi-lens` extension runs format/autofix over files that get rewritten, so uncommitted reflow diffs can appear in the working tree at any time. On the synced `main` this was observed for `index.ts`, `plan-lifecycle.test.ts`, `prompts.ts`, `shortcut-config.ts`, and `ui-compat.test.ts`; all of it was force-discarded, so the pushed commit still matches upstream.
- Recorded in passing (pre-existing upstream code, deliberately not changed here): `index.ts:196` `pi.appendEntry(STATE_TYPE, JSON.parse(snapshot));` is not wrapped in try/catch (pi-lens flags it as a blocker).

## Recommended fixes (recommendation only; not implemented in this change)

- **Target repository for a PR: `janvitos/pi-plan-build`, not `pi-hashline-edit-pro`.** All four defects are inside upstream 0.1.94 code (this repo's synced commit is byte-identical to the installed package). The invariants "Plan is read-only" and "plan files are read-only" are declared by pi-plan-build, so pi-plan-build has to fail closed on them. Pi exposes no tool semantics (`types.d.ts:344` has no readOnly/mutating field) and no active-tools change event (`types.d.ts:813`), so third-party mutating tools cannot be detected automatically; and hashline's removal of `edit` and its config-driven `grep`/`anchor_grep` swap are deliberate, self-consistent behaviour, not defects. The fix belongs with the side that declares the invariant, rather than coupling a general-purpose editing plugin to one specific mode plugin.
- **Three suggested fixes (pi-plan-build side)**
  1. Make Plan mode deny-by-default: replace the early return at `:1076` with "allow only a read-only allowlist (`read`/`grep`/`anchor_grep`/search-style tools, …), block unknown tools by default, and only pass when the path points at the current plan file". Convert the `includes(...)` checks at `:1054`/`:1055`/`:1061`/`:1070` to the same unified "mutating tool" predicate. `replace`/`insert`/`undo_last_change` then fall under the guard automatically, as would any future third-party mutating tool.
  2. Stop unconditionally re-adding `edit`/`write` in `applyTools("plan")` (`:262`); keep the host's existing tool set (`base`) instead. On stock pi both are already active, and on a hashline host the plan file can be written through `replace`/`insert`.
  3. Make `toolsBeforeModes` (`:1219-1222`) defer to live `pi.getActiveTools()`, using the persisted snapshot only to restore the self-managed `MANAGED_TOOLS`; and re-assert once in `before_agent_start` (`:1101`) so runtime toggles such as `/hashline-config` are no longer rolled back by the snapshot.
  Together these also close the Build-mode "plan file is read-only" and step-by-step gate bypasses via `replace`. If implemented, the activeTools assertions in the tests must be updated as well (`ui-compat.test.ts:45/69-70`, `plan-lifecycle.test.ts:41-42`).
- **For pi-hashline-edit-pro, an issue at most (not a PR):** suggest exporting a stable list of mutating tools (e.g. `["replace","insert","undo_last_change"]`) and documenting that the built-in `edit` is removed on `session_start`, so mode-style extensions can consume it. If pi-plan-build adopts deny-by-default, this is low priority.
- **Scope of this change:** this report contains recommendations only. No PR was opened and no plugin source was modified (explicit user requirement, confirmed 2026-09-13).
