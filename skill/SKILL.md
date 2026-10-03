---
name: codex-with-chatgpt
description: >
  Use ChatGPT (web) for planning and revision strategy when an authorized coding
  task needs decomposition or replanning and the applicable model-routing policy
  selects C2C. Also use for explicit Codex with ChatGPT requests, workspace
  connection/disconnection, and the ChatGPT planning loop. Skip routine Q&A and
  simple fully specified work that does not need planning. Codex owns execution.
---

# Codex with ChatGPT

ChatGPT thinks. Codex works.

You (Codex) own execution: editing, shell, git, tests, recovery.
ChatGPT owns high-level reasoning: understanding, planning, review, debug strategy.
The C2C Bridge gives ChatGPT read-only MCP access to the current workspace, so
control messages between you and ChatGPT stay tiny (< 1 KB) — ChatGPT pulls
whatever data it needs by itself.

Codex execution ownership follows `~/.agents/core/model-routing.md` when applicable.
Under that routing policy, general planning and revision strategy prefer C2C;
explicit C2C requests also select ChatGPT collaboration. Simple fully specified
work does not need an extra planning round. Preserve useful plans and return to
ChatGPT for material gaps, changed assumptions, or replanning, not every small fix.

If C2C is confirmed unavailable, preserve its checkpoint and report the limitation;
the controller may use the policy's GPT-6.1 Sol medium planning fallback. Browser,
login, consent, authorization, and repair gates below still apply; fallback does
not permit bypassing them. An explicit ChatGPT-only requirement is not silently
replaced. Record ChatGPT's observed selected model/setting separately; do not label
it as a Codex model/effort or invent effective provider identity.

Same-chat `REVIEW` or `DONE` replies cannot substitute for required fresh GPT-6.1
Sol/Astra reviews. A protocol or checkpoint `DONE` cannot mark unaccepted work as
delivered. Moving planning to ChatGPT reduces repeated Codex control-message input,
but is not evidence of lower total tokens across channels.

## Golden Rules

1. **No Data Leakage**: NEVER paste file contents, diffs, or logs into ChatGPT.
   ChatGPT reads them through MCP. Never record shell history, `.env`, keys, or secrets.
2. **Hide Technical Internals**: NEVER show technical internals (MCP, OAuth, PKCE,
   tunnels, ports, localhost) to the user. Speak in terms of "连接 ChatGPT / 安全连接 / 配对".
   The only exception is guided manual setup: expose only exact settings labels/values.
3. **Credentials**: The pairing code is the ONLY credential you may ever type into
   a browser. Never touch OAuth tokens, cookies, or session storage.
4. **Silent Doctor & Machine-Wide Preferences**: For local-only tasks or after IAB
   is confirmed available for UI flows, `c2c doctor` repairs silently. For UI-dependent
   workflows, browser availability discovery MUST precede any doctor auto-repair or
   mutations. Only involve the user for logins, CAPTCHA, 2FA, explicit consent, or
   guided manual setup — and give ONE action at a time.
   Machine-wide preferences (`c2c prefs --json`):
   - Missing `setupMode`: present `setupChoicePrompt`, wait for「1」or「2」, then save
     via `c2c prefs set --setup-mode auto|manual --json`. Do not start ChatGPT setup until answered.
   - `setupMode: "manual"`: skip automatic browser settings; use guided manual from the start (chosen).
   - `setupMode: "auto"`: automatic browser configuration. Two genuine failures of the same
     step after repair fall back to guided manual. (Timeouts, loading, login/2FA do not count as failures).
     Do not mutate saved `setupMode` upon fallback.
   - `developerModeEnabled: true`: skip `#settings/Security` until connector creation requires it.
5. **In-App Browser (IAB) Gate**: ALWAYS use the built-in in-app browser (`iab`) for
   ChatGPT interactions. Read [references/browser.md](references/browser.md) before any UI action.
   Discover `iab` FIRST (`await cua.getState()` then select `iab`); if unavailable or
   unknown, STOP UI-dependent workflows immediately with ZERO C2C commands or mutations
   (no doctor auto-repair, no tunnel creation, no pairing codes, no sandbox writes).
   Preserve checkpoint and report environment limitation. Never automatically fall back
   to external Chrome. Local-only token revocation (`c2c unpair`) does not require IAB
   and remains accessible.
6. **Conversation & Connector Isolation**:
   - `long-chat`: ONE ChatGPT conversation per workspace. Never silently start a new chat.
   - `project`: ONE ChatGPT Project per workspace. Current Codex thread reuses its saved chat URL;
     new thread opens a new chat from the Project collection page.
   - Exactly ONE connector per workspace. Never rename, edit, or delete another workspace's connector.
7. **Sandbox Permissions & Runtime Authorization**: After first-time setup, run
   `c2c sandbox-allow --json` (idempotent). Actual user authorization and current runtime
   permission policy are authoritative: if denied with EPERM and the runtime supports
   escalation for an authorized operation, use the supported approval mechanism to retry once;
   if escalation is prohibited or unsupported (e.g. approval policy is never), do not prompt,
   report the limitation, and retain state. Silent repair cannot expand authorization or bypass
   applicable confirmation policies. Do not add ritual prompts on authorized reversible actions.
8. **Navigation**: Use the known destinations in [references/browser.md](references/browser.md)
   and their visible controls. For connector creation, follow that reference's observed
   Plugins hub route; do not guess replacement deep links.
9. **Doctor Gate**: After `c2c doctor --json`, do NOT open ChatGPT or send `[C2C]`
   until local health is green (except reconnect settings pages when `chatgptRepair.needed` is true).
   Local is not green if:
   - `report.bridge.ok` is not true;
   - `report.mcp.ok` is not true (unauthenticated local `/mcp` must be 401);
   - sandbox or state-dir write failed (EPERM);
   - workspace had a public URL and the tunnel is down;
   - `chatgptRepair.needed` is true (fix connector first, then doctor again);
   - `namedRepair.needed` is true (user logs into Cloudflare, then doctor again; do not delete connector);
   - `report.bridge` says 状态无法确认: bridge may still be running; do NOT run `c2c start`, do NOT
     delete connector, do NOT treat as `chatgptRepair` — wait and run doctor again.
   If doctor is green and `chatgptRepair.needed` is false, do not restart, do not start a second tunnel,
   and do not delete connector. Unknown bridge status must not cause churn.

## Locations

- The codex-with-chatgpt checkout lives at: `<ACTUAL_CHECKOUT_PATH>`
  (installer/update MUST replace this line in the installed Skill with the user's actual checkout path.)
- CLI: let `<checkout>` mean the path on the previous line; run
  `node "<checkout>/bin/c2c.js" <command>` (or `c2c <command>` if globally linked).
  Use `--json` only where the command supports it; check that subcommand's `--help`
  when uncertain. `session get` supports it, while `session set`, `restart`, and
  `record` do not in the currently installed CLI.
- If the checkout has no `node_modules` or no `dist/`, first run
  `corepack pnpm install && corepack pnpm build` inside it.
- For commands that act on the user's project (`setup`, `doctor`, `session`,
  `restart`, `start`, `stop`, `status`, `pair`, `unpair`, `logs`, `workspace`,
  `record`, `tunnel status`, `tunnel choose`), pass `-w <workspace root>`
  (the project the user is working on, NOT the c2c repo).
- Do not add `-w` to machine-wide commands: `update-check`, `sandbox-allow`,
  `prefs`, `tunnel login`. They still accept and ignore `-w`, so a leftover
  flag must not fail the command.

## Operating Modes and Reference Routing

Do not read all references by default. Load only the reference required for the active mode:

- **First-Time Setup & Connection Choice**:
  When the user requests initial configuration, pairing, or tunnel choice, read
  [references/setup.md](references/setup.md).
- **Coding Task & Planning Loop**:
  When executing a user task through the planning loop, managing checkpoints,
  recording iterations, or handling resume/completion, read
  [references/task-loop.md](references/task-loop.md).
- **Conversation & Project Management**:
  When inspecting sessions, switching chats, opening chats in threads, or binding
  a Project, read [references/conversations.md](references/conversations.md).
- **Maintenance, Updates, and Repair**:
  For daily update checks, updating the skill, disconnecting, reconnecting after
  address reclaim, doctor diagnostics, or recovery actions, read
  [references/maintenance.md](references/maintenance.md).
- **Browser UI Interactions**:
  Before performing any browser action, tab creation, navigation, or reply wait,
  read [references/browser.md](references/browser.md).
