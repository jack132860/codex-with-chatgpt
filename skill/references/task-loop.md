# Task Loop

This reference defines the C2C planning and execution loop between Codex and ChatGPT, checkpoint states, resume logic, and delivery boundaries.

## States Overview

- **Protocol States (Sent to ChatGPT)**:
  `INIT` → `PLAN` → `EXECUTING` → `EXECUTED` → `REVIEW` → (`PLAN` | `DONE` | `BLOCKED`).
- **Local Checkpoint States (Session only, never sent to ChatGPT)**:
  `INIT`, `PLAN_RECEIVED`, `EXECUTING`, `EXECUTED_LOCAL`, `EXECUTED_SENT`, `DONE`, `BLOCKED`.
- **Invariants**:
  - Never invent `STATE: RESUME`. If a conversation is lost, send `HANDOFF` (<checkout>/docs/protocol.md §HANDOFF).
  - All control messages begin with `[C2C]` and stay under 1 KB.
  - Full protocol details are documented in `<checkout>/docs/protocol.md`.

---

## Step 0: Pre-Flight, IAB Verification, and Doctor Gate

1. **IAB Availability Gate (MUST Precede Any Mutation)**:
   - Check browser availability following [browser.md](browser.md) §1: execute `await cua.getState()`, then check `cua.getBrowser({ id: "iab" })`.
   - **If `iab` is absent, unavailable, or unknown**: **STOP immediately**.
   - Do **NOT** execute any C2C commands or mutations: do not run `c2c doctor`, do not start tunnels, do not restart bridges.
   - Preserve the active checkpoint and report the capability limitation to the user. Never fall back to Chrome automatically.
2. **Tunnel & Connection Choice**: Check `c2c tunnel status -w <workspace> --json`. If `needsChoice` is true, resolve connection choice first (see [setup.md](setup.md)).
3. **Doctor Gate**: Run `c2c doctor -w <workspace> --json` (performs auto-repair).
   - Local health must be green before opening ChatGPT or sending `[C2C]`.
   - If `namedRepair.needed` is true: tell the user `namedRepair.userMessage`, run `c2c tunnel login --json` (external browser exception), then run doctor again. Do not delete the connector.
   - If `chatgptRepair.needed` is true: tell the user `chatgptRepair.userMessage`, execute the reconnect workflow in [maintenance.md](maintenance.md), run doctor again, and proceed only when green.
   - If bridge status cannot be confirmed: bridge may still be running; wait and rerun doctor rather than churning `c2c start` or deleting connectors.
4. **Task ID**: Generate a new task ID as `c2c_` + 4 random hex characters, unless an active checkpoint already defines one (reuse the existing task ID).

---

## Step 1: Session Initialization and Checkpoint Resume

1. Inspect session: `c2c session -w <workspace> --json`.
2. Open ChatGPT on the single IAB tab per [conversations.md](conversations.md) and [browser.md](browser.md).
3. If initiating a new conversation: confirm Chat mode ([browser.md](browser.md) §7), send the boot prompt from `<checkout>/docs/protocol.md` §Boot Prompt, and send the `workspace_info` check referencing the exact `connectorName`. Verify the workspace name matches before saving the session URL.
4. **Resume from `session.checkpoint` before sending any INIT**:
   - `EXECUTED_SENT` with `waitingFor=GPT_REVIEW`: Do NOT send `INIT`, do NOT re-run execution, and do NOT resend `EXECUTED`. Maintain standby on the saved chat and wait for review. If the chat 404s, send `HANDOFF` from checkpoint fields (no logs) in a replacement chat.
   - `EXECUTED_LOCAL`: Local execution is already finished. Do not re-run. Record if unrecorded, then send `EXECUTED`.
   - `EXECUTING`: Execution incomplete. Continue the existing `PLAN` if in hand; otherwise send `HANDOFF` requesting ChatGPT to restate the last `PLAN`. Do not mark done and do not `INIT` a new task.
   - `PLAN_RECEIVED`: Execute the existing plan. Do not `INIT`.
   - `INIT` with `waitingFor=GPT_PLAN`: Claim the tab and wait for the plan. Do not resend `INIT`.
   - `DONE`:
     - **Canonical completion condition**: Clear checkpoint (`--clear-checkpoint`) ONLY when the objective is complete AND (independent acceptance is NOT required for this task, OR ALL applicable required fresh Sol/Astra checks have passed on the unchanged candidate).
     - If any applicable required check is pending, FAIL, INCONCLUSIVE, or unavailable: **retain existing checkpoint and evidence** in a not-delivered state; do NOT clear checkpoint.
     - When the canonical completion condition is met:
       `c2c session set -w <ws> --state DONE --clear-checkpoint`
   - `BLOCKED`: Surface ChatGPT's blocking reason to the user; do not `INIT`.
   *Never re-pair, recreate the connector, or rewrite Project instructions simply to resume an interrupted task.*

---

## Step 2: Send INIT

Send the `INIT` message containing the user's goal (skip if resuming from a non-INIT checkpoint):

```
[C2C]
STATE: INIT
TASK_ID: <task_id>
ITERATION: 0

GOAL:
<user's goal, one paragraph>

INSTRUCTION:
Inspect the connected workspace through the Codex with ChatGPT MCP connector.
Produce a C2C PLAN message.
```

- Confirm the message is visibly rendered in the ChatGPT conversation (one cheap DOM check). If Retry-only appears, follow recovery in [browser.md](browser.md) §7.
- Do not write the waiting checkpoint until the message is confirmed visible.
- Record the checkpoint:
  `c2c session set -w <ws> --task <id> --iteration 0 --state INIT --protocol-state INIT --waiting-for GPT_PLAN --goal "<short goal>" --next-step "wait for PLAN"`

---

## Step 3: Wait for PLAN

1. Wait for ChatGPT's `STATE: PLAN` reply following [browser.md](browser.md) §8.
2. Inspect `GOAL`, `ACTIONS`, `TESTS`, and `SUCCESS_CRITERIA`.
3. Require substantive plans: A proper plan must contain rationale and concrete per-file edit suggestions (which file, what to change, and why). If the reply is a bare one-liner, ask once:
   "Please expand the plan with rationale and concrete per-file suggestions."
4. Update the checkpoint:
  `c2c session set -w <ws> --protocol-state PLAN_RECEIVED --waiting-for none --next-step "execute PLAN"`

### Read Coverage During Planning and Review

- `read_file.truncated` means lines remain after the returned `endLine` in the **whole file**. It can be `true` even when an explicit requested range was returned in full; do not infer missing content or blame long lines from that flag alone.
- Check returned `startLine`/`endLine` against the requested range and `totalLines`. For further coverage, continue at `nextStartLine` without gaps or overlap. If `endLine` falls short of the requested end (bounded by EOF), read the remaining range; when only a bounded scope is needed, do not read unrelated lines merely to reach `truncated=false`.
- Tool pagination and UI/combined-response truncation are separate. If displayed content is actually omitted, request that range separately in smaller pages and state any remaining visibility limit. Complete retrieval does not by itself establish semantic review or acceptance.

---

## Step 4: Execute Plan and Ownership Boundaries

1. Execute the plan locally using Codex's own harness (tools, editing, and judgment; ChatGPT does not micromanage tool calls).
2. **Multi-model routing and acceptance contract**:
   - Codex execution ownership follows `~/.agents/core/model-routing.md` when applicable.
   - Same-chat ChatGPT `REVIEW` or `DONE` replies are advisory collaboration only and **cannot substitute for required independent fresh Sol/Astra reviews**.
3. Update checkpoint before execution:
  `c2c session set -w <ws> --protocol-state EXECUTING --waiting-for none --next-step "finish PLAN then record"`

---

## Step 5: Record Execution

Record iteration metadata locally so ChatGPT can inspect it via MCP:

```bash
c2c record -w <ws> --task <task_id> --iteration <n> --changed-files "src/a.ts,src/b.ts" --tests "27 passed" --exit-status ok
```

- If this iteration ran a test, build, lint, or typecheck command, capture stdout/stderr to a temporary file and pass it:
  `c2c record -w <ws> --task <id> --iteration <n> ... --command "pnpm test" --output-file <temp> --exit-code <n>`
- **Sanitization**: Never record shell history, `.env` files, API keys, credentials, or unrelated system dumps. Never paste output bodies into ChatGPT messages.
- If the CLI reports output was withheld or restricted, proceed to send `EXECUTED`; ChatGPT will review changes from git diff.
- Update checkpoint:
  `c2c session set -w <ws> --iteration <n> --state EXECUTED --protocol-state EXECUTED_LOCAL --waiting-for none --next-step "send EXECUTED"`

---

## Step 6: Send EXECUTED

Send the `EXECUTED` control message (never include diffs or logs):

```
[C2C]
STATE: EXECUTED
TASK_ID: <task_id>
ITERATION: <n>

RESULT:
Execution finished.

CHANGED_FILES:
<count>

TESTS:
<summary>

Please independently inspect the workspace and current git diff through MCP.
If execution_output lists a readable item for this iteration, list then read it.
If status is restricted, ignore it and review from git_diff.
```

Update checkpoint:
`c2c session set -w <ws> --protocol-state EXECUTED_SENT --waiting-for GPT_REVIEW --next-step "wait for PLAN or DONE"`

---

## Step 7: Review, Iteration, and Loop Bounds

1. ChatGPT reviews changes independently via MCP (`git_diff`, `read_file`, `test_status`, `execution_output`) and replies with `DONE`, `PLAN` (next iteration), or `BLOCKED`.
2. **Loop limits**:
   - Bounded by `maxIterations` in `.c2c.json` (default 12) AND core time and correction bounds.
   - When either threshold is reached, pause and inform the user with the actual numbers:
     "已达到界限（实际完成 <actual_iterations> 轮协作，耗时 <actual_time>，触发原因：<trigger_reason>），是否继续？"
3. **On DONE**:
   - Summarize the result in plain language.
   - **Acceptance Boundary**: A ChatGPT `DONE` protocol state cannot mark unaccepted work as delivered when independent acceptance (e.g. fresh-context review) is mandated by project core.
   - **Canonical completion condition**: Clear checkpoint (`--clear-checkpoint`) ONLY when the objective is complete AND (independent acceptance is NOT required for this task, OR ALL applicable required fresh Sol/Astra checks have passed on the unchanged candidate).
   - If any applicable required check is pending, FAIL, INCONCLUSIVE, or unavailable: **retain existing checkpoint and evidence** in a not-delivered state; do NOT call `--clear-checkpoint`.
   - When the canonical completion condition is met:
     `c2c session set -w <ws> --state DONE --clear-checkpoint`
4. **On BLOCKED**:
   - Inspect the blocking reason provided by ChatGPT. Resolve if possible within authorization, or present the single decision required to the user.
   - Update checkpoint:
     `c2c session set -w <ws> --protocol-state BLOCKED --waiting-for USER --known-issues "<short reason>"`

See also [../SKILL.md](../SKILL.md) for global invariants.
