# In-app browser (ChatGPT)

All browser UI interactions with ChatGPT must use the built-in in-app browser (`iab`). These rules override defaults that close the tab, hide the window, or stall on settings pages.

## 1. Browser Discovery and First-Call Contract

1. **First-Call Contract**: In a fresh `cua` session, execute exactly **ONE** documented entry call: `await cua.getState()`, with no additional actions in the same invocation.
2. Inspect the runtime documentation and browser inventory returned by `getState()`.
3. **IAB Availability Gate**: Check whether `iab` is available (e.g. via `cua.getBrowser({ id: "iab" })`).
   - If `iab` is absent or returns an error (e.g. `Browser is not available: iab`), **stop UI-dependent workflows immediately**.
   - Preserve the current task checkpoint and report the capability limitation to the user.
   - **When IAB is unavailable or unknown, execute ZERO C2C commands or mutations** (no doctor auto-repair, no tunnel start, no sandbox writes, no pairing code minting).
   - **Never automatically fall back to Chrome or another external browser.**
   - If the user explicitly directs using their external browser, follow user/higher tool instructions directly without requiring scripted magic phrases.
   - Native-app GUI / coordinate-based desktop clicking (e.g. OS-level mouse clicking or screenshot matching) is prohibited where unrequested; `cua` itself is the supported browser interaction layer.
   - Local-only inspection or disconnect operations that do not depend on the browser are exempt from the IAB requirement.

## 2. One Tab Policy

1. Create the ChatGPT tab once (`cua.createBrowserTab("iab", url, { visible: true })`).
2. If a ChatGPT tab already exists, claim it via `cua.getTab(id, { browser: "iab" })` — never open a second ChatGPT tab.
3. Reuse the same tab for direct navigation and the on-page routes below. Do not `goto` the URL you are already on.
4. Visibility: `{ visible: true }` is documented during tab creation; foregrounding an already-open tab has no established generic setter in generic docs, so do not invent ad-hoc setter methods.

## 3. Standby and Lifecycle

1. Call `await tab.markHandoff()` immediately upon opening or claiming the tab, and at the start and end of every turn.
2. After setup succeeds or the C2C chat is open, call `await tab.markDeliverable()`.
3. **Never close this tab.** Whether finished, waiting for the user, or timed out, leave the tab marked in standby. Do not let default turn cleanup close it.

## 4. Navigation and Connector Creation

Open known destinations using `tab.goto(...)`, then use the relevant visible controls:

- **Developer mode**: `https://chatgpt.com/#settings/Security` (skip when `c2c prefs --json` reports `developerModeEnabled: true`).
- **Plugins hub**: `https://chatgpt.com/plugins` (connector management and creation).
- **Add connector**: From the Plugins hub, follow the visible `新增` → `建立 MCP 應用程式` controls to the `新增外掛程式` form (observed 2026-09-29). Inspect current page state between actions; labels may vary by locale or UI version. The former `create-connector=true` deep link redirected to settings without opening the form in this smoke test. Do not keep retrying it or guess replacement URLs; if this route is unavailable, use the existing setup recovery/manual guidance.
- **New chat** (long-chat mode only, and only if no saved chat exists): `https://chatgpt.com/`
- **Saved C2C chat**: `conversation.chatUrl` / `session.url` (long-chat, or the chat bound in this Codex conversation thread).
- **Saved Project collection**: `conversation.projectUrl` (`https://chatgpt.com/g/g-p-…/project`).

**Connector rules:**
- Never click Reconnect / Refresh on an existing connector. The old address is dead and the page hangs on "This site cannot be reached".
- When an address changes: delete THIS workspace's `connectorName` only, then recreate it via the Add connector route above (same name, new Server URL).
- Never put the public or temporary address into Project instructions — write the connector **name** only.

## 5. Connector Setup Confirmation

Do not wait for 8 tools on the settings page. "Connected" / authorize success / pairing code accepted is sufficient. Confirm tool availability in the conversation using `workspace_info`.

## 6. Form Actions and State Inspection

1. Fill known forms using supported AX actions (`paste`, `setValue`, `click`) or DOM evaluation only if the active browser documentation explicitly provides it.
2. After actions, inspect page state via `tab.getAXState()` using fresh element indices rather than stale references. Do not use coordinate-based desktop clicking.

## 7. Conversation Mode and Page Recovery

1. **Chat mode vs. Work mode**: The first ChatGPT chat is the C2C conversation. Chat and Work (聊天 / 工作) are separate: a Work conversation cannot become Chat. On every NEW conversation, if a Chat/Work switcher is visible, confirm **Chat** is selected before sending the boot prompt. If it is Work, do not continue there — switch to a new Chat conversation (send HANDOFF). If no switcher is visible, continue without hunting menus.
2. Send the boot prompt and the `workspace_info` check in that Chat conversation. Confirm the reply names the current workspace **before** saving or replacing the session URL. If validation fails, keep the old saved URL. Do not open a throwaway verify chat and later another C2C chat.
3. **Retry-only recovery**: A collection or chat page showing only `Retry` / `重试` is a navigation error, not a generation or pairing failure. Reuse the same IAB tab:
   - Click Retry once.
   - If it remains Retry-only, `goto` the last working chat URL from this thread (or `session.url`), then click the on-page `Open … project` / `打开“… ”项目` link.
   - On the collection page, require the project chat list and new-chat composer before continuing.
   - Keep the old saved URL/checkpoint until the replacement chat passes `workspace_info`. Do not run `session clear`.

## 8. Waiting for ChatGPT Replies

After sending `INIT`, `EXECUTED`, boot, or the `workspace_info` check:
1. `markHandoff`, keep the tab in standby, and remain in the same task.
2. Use runtime-supported waits without inventing methods; avoid unconditional unchanged-state polling if runtime rules prohibit it.
3. Check state periodically:
   - **Still generating**: continue waiting (do not type, do not resend).
   - **`STATE: PLAN` / `DONE` / `BLOCKED` / workspace name**: read the response and continue the protocol.
   - **Visible error**: enter repair; do not start a new chat.
4. A browser/js timeout is not a failure. Claim the same tab, inspect page state, and remain in standby. If ChatGPT is still generating, continue waiting. Never open a second tab and never resend `INIT` or `EXECUTED` just because a wait timed out.

See also [../SKILL.md](../SKILL.md) for global invariants.
