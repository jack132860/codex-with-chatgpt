# Setup and Connection Choice

This reference covers the one-time connection choice, the automated first-time setup workflow, and the guided manual setup fallback.

## Workflow: First-Time Setup（"使用 Codex with ChatGPT 完成首次配置"）

1. **IAB Availability Gate (MUST Precede Any Mutation)**:
   - Check browser availability following [browser.md](browser.md) §1: execute `await cua.getState()`, then check `cua.getBrowser({ id: "iab" })`.
   - **If `iab` is absent, unavailable, or unknown**: **STOP immediately**.
   - Do **NOT** run any C2C commands or mutations: do not install packages, do not modify sandbox allowlists, do not prompt for connection choice, do not run `c2c setup`, and do not mint pairing codes.
   - Report the environment limitation to the user and explain that C2C UI setup requires an available in-app browser.
   - Never fall back to Chrome or another external browser automatically.
2. **Prerequisites Check**:
   - Check `node --version` (requires Node.js >= 20).
   - Check `cloudflared`. If missing on macOS run `brew install cloudflared`; on Windows run `winget install Cloudflare.cloudflared`.
3. **Build Check**:
   - If the C2C repository checkout lacks `node_modules` or `dist/`, run `corepack pnpm install && corepack pnpm build` inside `<checkout>`.
4. **Local Initialization and Connection Choice**:
   - Run `c2c sandbox-allow --json` (adds state directory to `writable_roots` in `config.toml`). Current runtime permission policy is authoritative: if denied with EPERM and the runtime permits escalation for this authorized operation, use the supported approval mechanism to retry once; if escalation is prohibited or unsupported (e.g. approval policy is never), do not prompt, report the limitation, and retain state.
   - Resolve **Connection Choice** below.
   - Run `c2c setup -w <workspace> --json` → `{ mcpUrl, pairingCode, workspaceName, connectorName, ... }`.
   - Note on pairing code: Pairing codes expire in ~5 minutes. Do not mint one until the ChatGPT Authorize / pairing form is visibly displayed on screen: run `c2c pair --json` at form time.
5. **Setup Preferences** (`c2c prefs --json` — machine-wide setting):
   - If `setupMode` is null: present the user with `setupChoicePrompt` exactly. Wait for「1」or「2」. Save via `c2c prefs set --setup-mode auto|manual --json`. Do not start browser configuration until answered. Do not default to auto.
   - If the user later requests a switch: use `c2c prefs set --setup-mode auto|manual --json`. Do not re-ask on subsequent workspaces or reconnects.
   - If `setupMode` is `"manual"`: skip step 6's automatic browser settings. Proceed directly to **Guided Manual ChatGPT Setup** below (chosen mode). Opening line:
     `接下来用手动教学配置。一次只需要做一个操作。`
   - If `setupMode` is `"auto"`: proceed with step 6. Follow the two-failure fallback policy.
6. **Automatic Browser Configuration**: Open ChatGPT on the single IAB tab (see [browser.md](browser.md)). Set standby and `markHandoff` immediately. Use the known destinations and observed controls in that reference:
   - **Developer mode**: Skip `https://chatgpt.com/#settings/Security` when `developerModeEnabled` is `true`. Otherwise open it, enable 开发人员模式 ("Developer mode") if off, then run `c2c prefs set --developer-mode`. Never record it as off. If connector creation later fails because developer mode is required, navigate to this page, enable it, save `--developer-mode`, and retry creation.
   - **Existing connector**: Navigate to `https://chatgpt.com/plugins`. If `connectorName` already exists: Delete it (never click Reconnect). Then follow the Add connector route below.
   - **Create connector**: Follow **Add connector** in [browser.md](browser.md) §4 from the Plugins hub.
     Operate ONLY on `connectorName` from step 4:
     - If that exact name exists: Delete it, then recreate it. Never Reconnect, never edit-in-place, never open the old Server URL.
     - If it does not exist: create one with that exact name.
     - Never rename, delete, or modify a connector belonging to another workspace.
     - Description: `Securely connect ChatGPT to the current Codex workspace for planning and review.`
     - Server URL: the `mcpUrl` from step 4.
     - Authentication: `OAuth`.
     Fill the form using AX actions. Click Connect / Authorize. Only then run `c2c pair --json` and enter the freshly minted code.
     As soon as the UI displays Connected / authorized / pairing accepted, continue immediately — **do NOT wait for 8 tools to load on this page**.
7. **Initiate First Conversation**:
   - On the same tab, open the first C2C chat per [conversations.md](conversations.md) (Project collection for new workspaces; `https://chatgpt.com/` for long-chat).
   - Confirm Chat mode per [browser.md](browser.md) §7.
   - Send the boot prompt from `<checkout>/docs/protocol.md` §Boot Prompt.
   - In the same chat, send the verification prompt:
     `Use the "<connectorName>" connector: call workspace_info and read hello-style top-level file. Reply with the workspace name.`
   - Confirm the reply matches `workspaceName` (wait per [browser.md](browser.md) §8). Only after the name matches, save the chat URL via `c2c session set` (see [conversations.md](conversations.md)). If validation fails, do not save.
   - Call `await tab.markDeliverable()`.
8. **Report Completion**: Present the user with this exact status card (no technical internals):

```
Codex with ChatGPT

✓ 当前项目已识别
✓ Workspace Bridge 已启动
✓ 安全连接已建立
✓ ChatGPT 已连接
✓ 文件读取测试通过

Ready.
```

If a login wall appears at any point (ChatGPT or Cloudflare): pause, tell the user the ONE action required ("请登录 ChatGPT，完成后告诉我'好了'"), and wait for completion before proceeding.

---

## Connection Choice (Once per Workspace)

Ask this **before** the public address exists (`c2c setup` or the first `doctor --fix` that starts a tunnel). Do not mention technical internals (MCP, OAuth, PKCE, tunnels, wrangler, DNS, ports, or localhost). Speak only in terms of 临时地址 / 固定域名 / 登录 Cloudflare.

1. Run `c2c tunnel status -w <workspace> --json`.
2. If `needsChoice` is `false`: do not ask again.
3. If `needsChoice` is `true`: present the user with `userPrompt` exactly and wait for their choice:
   - **No account / no domain / temporary / 不用**:
     `c2c tunnel choose -w <ws> --mode quick --json`
   - **Custom domain (e.g. example.com)**:
     Tell them `loginPrompt` first, then run `c2c tunnel choose -w <ws> --mode named --zone <domain> --json`.
     This may open the user's external browser (the single Cloudflare login exception). Wait until the command completes.
     If the user indicated they have an account but provided no domain: ask once for the domain. If the command returns `need: "zone"`, ask once and retry.
     If `fallback` is `true`: output `userMessage` and continue on the temporary address. Do not retry named mode unless explicitly requested.
4. **Credential isolation**: Never store connection credentials in the project workspace. The CLI stores them in the C2C state directory.

---

## Guided Manual ChatGPT Setup

Enter this workflow when:
- `setupMode` is `"manual"` (chosen at the start), or
- Automatic ChatGPT browser configuration fails twice at the same explicit setup or reconnect step after `c2c doctor` / repair.

**Do NOT enter failure fallback** for a browser/js timeout without visible error, a page still loading/generating, or while waiting for login / 2FA / CAPTCHA. Chosen manual setup does not wait for failures.

When entering this mode:
- Stop automating ChatGPT settings.
- Preserve the current local C2C state, `mcpUrl`, `pairingCode`, `workspaceName`, and `connectorName`.
- Do not fall back to Codex-only execution and do not permanently disable C2C.
- Do not alter the saved `setupMode` preference when entering due to failure fallback.

**Opening line:**
- If chosen (`setupMode: "manual"`): `接下来用手动教学配置。一次只需要做一个操作。`
- If failure fallback: `自动配置没有成功，我来带你手动完成。一次只需要做一个操作。`

Guide the user ONE action at a time, waiting for them to say「好了」before giving the next step:

1. If `developerModeEnabled` is not true: ask the user to open `https://chatgpt.com/#settings/Security` and enable 开发人员模式. After they say「好了」, run `c2c prefs set --developer-mode`. If already enabled, skip this step.
2. Ask the user to open `https://chatgpt.com/plugins`. If the exact `connectorName` exists, instruct them to delete only that connector. Never touch another workspace's connector.
3. Guide the user through **Add connector** in [browser.md](browser.md) §4, one visible action at a time, then create a connector with:
   - Name: exact `connectorName`
   - Description: `Securely connect ChatGPT to the current Codex workspace for planning and review.`
   - Server URL: current `mcpUrl`
   - Authentication: `OAuth`
4. Ask the user to click Connect / Authorize. Then run `c2c pair --json` and provide only that pairing code. If it expires, run `c2c pair --json` again.
5. When the user confirms Connected / authorized / pairing accepted, resume the verification step in the normal setup or reconnect workflow. If verification then fails twice at the same explicit step, pause and report the failure; do not loop indefinitely.

See also [../SKILL.md](../SKILL.md) for global invariants.
