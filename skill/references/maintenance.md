# Maintenance, Updates, and Recovery

This reference covers update checks, safe updating, disconnection, reconnection after address expiration, doctor diagnostics, and the recovery map.

## Doctor Behavior and Side-Effect Boundaries

- Running `c2c doctor -w <workspace> --json` executes **default auto-repair**: it can modify Codex sandbox allowlists, start bridges, and spawn Cloudflare tunnels.
- Passing `--no-fix` (`c2c doctor --no-fix --json`) skips active repair branches, but when a healthy public endpoint is detected, it still persists that endpoint to disk (`persistWorkspaceEndpoint`) and probes public `/health`. **Neither flag is a zero-write audit.**
- Routine operational workflows must NOT invoke config writes, pull from git, or alter repository working state solely for inspection. Version checks and maintenance operations must only occur when relevant and authorized.

---

## Maintenance Version Check

When maintenance is authorized or version verification is relevant:
1. Run `c2c update-check --json` (do not pass `-w`). Note: this may write a local cache file, but must **never** automatically trigger git pull, stashing, installation, or upgrades during ordinary tasks.
2. If `{ "updateAvailable": true }`, inform the user that an update is available and proceed with the update workflow below only if authorized.
3. Ordinary coding tasks do not auto-pull, auto-stash, or auto-upgrade.

---

## Workflow: Update（"更新 Codex with ChatGPT"）

Execute inside `<checkout>` (see Locations in `SKILL.md`):

1. **Working tree inspection**:
   - Check `git status --porcelain`.
   - If local changes or patches are present, **preserve them**. Do not run destructive `git stash` or assume automatic patch reapplication. Inspect differences and integrate from a known clean base in a separate worktree or branch as appropriate.
   - If clean, pull latest changes: `git pull --ff-only`.
2. **Build artifacts**: `corepack pnpm install && corepack pnpm build`.
3. **Re-install the skill** (resolve `<codex-home>` from Locations in `SKILL.md`):
   - Copy the entire `skill/` directory (including `references/`) to `<codex-home>/skills/codex-with-chatgpt/`:
     ```bash
     mkdir -p "<codex-home>/skills/codex-with-chatgpt"
     cp -R skill/. "<codex-home>/skills/codex-with-chatgpt/"
     ```
   - **Safety rules**:
     - Handle paths with spaces safely by quoting all paths.
     - Never run `rm -rf ~/.codex/skills` or `rsync --delete` against the parent skills directory, which would destroy unrelated skills.
   - Replace `<ACTUAL_CHECKOUT_PATH>` in the installed `<codex-home>/skills/codex-with-chatgpt/SKILL.md` with the absolute path of `<checkout>`.
4. **Refresh permissions and runtime**:
   - `c2c sandbox-allow --json`
   - `c2c restart -w <workspace>` (restarts the bridge process with new code)
   - `c2c update-check --force --json` (refreshes the version cache)
5. Report to the user: "✓ 已更新到最新版本".

---

## Workflow: Disconnect（"断开 ChatGPT"）

1. **Local token revocation**:
   - Run `c2c unpair -w <workspace>`.
   - This command does NOT require IAB or browser access.
   - Inspect the command output: `unpair` may fail if the CLI or environment is corrupted. **Only report revocation to the user after confirming the command executed successfully.**
2. **Optional UI cleanup**:
   - If IAB is available per [browser.md](browser.md), navigate to `https://chatgpt.com/plugins` and delete this workspace's connector card (`connectorName`).
   - If IAB is unavailable or fails, inform the user that local access was successfully revoked and that the ChatGPT connector card may be removed manually if desired.
3. Once successful local revocation is confirmed, inform the user: "已断开 ChatGPT 对该项目的访问。"

---

## Workflow: Reconnect After Address Reclaim（全关掉以后地址失效）

When the machine reboots, Codex quits, or the previous tunnel expires:

1. **IAB Availability Gate (MUST Precede UI Reconnect Mutations)**:
   - Discover browser availability following [browser.md](browser.md) §1 (`await cua.getState()`, `cua.getBrowser({ id: "iab" })`).
   - If `iab` is absent or unknown: **STOP immediately**. Do NOT run doctor auto-repair, do NOT spawn tunnels, and do NOT delete connectors. Preserve checkpoint and report limitation to user.
2. Run `c2c doctor -w <workspace> --json`.
   - Output will contain: `{ "chatgptRepair": { "needed": true, "connectorAction": "update", "connectorName": "...", "userMessage": "...", "mcpUrl": "...", "pages": { ... } } }`.
   - `connectorAction: "update"` means **Delete and recreate** — never Reconnect.
3. Tell the user `chatgptRepair.userMessage` exactly. Do not ask them to navigate ChatGPT unless a login wall appears. Do not open the C2C chat or send `[C2C]` until repair finishes and doctor is green.
   - Check `c2c prefs --json`. If `setupMode` is `manual`, follow **Guided Manual ChatGPT Setup** in [setup.md](setup.md).
4. On the single IAB tab (standby + `markHandoff`), use these destinations and the observed creation route:
   - **Developer mode**: Skip `https://chatgpt.com/#settings/Security` if `developerModeEnabled` is `true`. Otherwise enable it and run `c2c prefs set --developer-mode`.
   - **Plugins hub**: `https://chatgpt.com/plugins`
   - **Add connector**: Follow [browser.md](browser.md) §4 from the Plugins hub.
5. Operate ONLY on `chatgptRepair.connectorName`:
   - If that name exists on the plugins hub: **Delete it**.
   - **Never click Reconnect, Refresh, Connect, or Edit** on the old card — the old server URL is unreachable and the page will hang.
   - Follow that Add connector route and create the connector with the same `connectorName`:
     - Description: `Securely connect ChatGPT to the current Codex workspace for planning and review.`
     - Server URL: `chatgptRepair.mcpUrl`
     - Authentication: `OAuth`
   - Click Connect / Authorize, run `c2c pair --json`, and enter the pairing code.
   - Proceed as soon as Connected is displayed (do not wait for 8 tools).
6. Run `c2c doctor -w <workspace> --json` again.
   - Only after doctor is green, return to the chat URL used earlier in this thread.
   - In that chat, send the `workspace_info` check referencing `connectorName`.
   - If the reply names the current workspace: resume work.
   - If `workspace_info` fails, times out, or cannot read the workspace: in Project mode, open the collection page, create a new chat in the Project, send boot + `HANDOFF` (from checkpoint, no logs) + `workspace_info`, and update the session URL after the name matches. In long-chat mode, follow the conversation switch workflow in [conversations.md](conversations.md).
7. If the conversation was lost entirely, follow step 6's replacement path. Never re-upload repository files to ChatGPT.

---

## Workflow: Repair（Anything Looks Broken）

1. If the repair involves browser UI operations, verify IAB availability per [browser.md](browser.md) first.
2. Run `c2c doctor -w <workspace> --json`.
   - **Doctor gate**: Do not navigate to ChatGPT or send `[C2C]` until local health is green (except when accessing reconnect settings pages).
3. If `namedRepair.needed` is true: output `namedRepair.userMessage`, run `c2c tunnel login --json` (external browser exception), then run doctor again. Do NOT delete the connector.
4. If `chatgptRepair.needed` is true: follow **Reconnect After Address Reclaim** above, then run doctor again.
5. Otherwise, apply the recovery map below. Prompt the user only for login, CAPTCHA, or 2FA — one action at a time.

---

## Recovery Map

| Symptom | Action |
| --- | --- |
| Bridge not running | `c2c start` (performed automatically by doctor) |
| Tunnel dead / URL unreachable / 全关掉后连接失效 | First verify IAB availability. Then run `c2c doctor`. If `namedRepair.needed`, log in to Cloudflare and run doctor again (do not delete connector). If `chatgptRepair.needed`, inform the user, **Delete** this workspace's connector only (`connectorName`), and create it anew. Never click Reconnect. Verify with `workspace_info`; if failed, open a new chat in the same Project (or long-chat switch) + `HANDOFF`. |
| Collection page shows only Retry | On the same IAB tab: click Retry once. If still Retry, navigate to the last working chat and click its on-page Project link. Do not write waiting checkpoints until the message is confirmed visible. |
| ChatGPT reports tool call failed / 401 | Token expired or revoked → re-pair (generate new code with `c2c pair --json` and authorize). |
| Pairing code rejected / expired | Run `c2c pair --json` to generate a fresh pairing code at form time. |
| Same explicit ChatGPT setup/reconnect browser step fails twice after repair | Stop automating browser settings and switch to **Guided manual ChatGPT setup** in [setup.md](setup.md). Do not treat timeouts, loading states, or login prompts as failures. |
| Port conflict | Handled automatically by CLI; never surface to the user. |
| Every new chat “repairs” / cannot write log or settings directory | Run `c2c sandbox-allow --json` (once). If denied with EPERM, use supported runtime elevation only if permitted by current policy; if elevation is unsupported or prohibited, report the limitation and retain state without redundant prompts. |
| cloudflared missing | Automatically install via `brew` (macOS) or `winget` (Windows), then retry. |
| Sidebar has no「项目」 | Instruct the user to hover over「聊天」, click the `…`, and select「按项目整理」. |
| Collection page shows wrong Project | Instruct the user to open the named project collection and report「已找到」, or offer long-chat fallback. |

See also [../SKILL.md](../SKILL.md) for global invariants.
