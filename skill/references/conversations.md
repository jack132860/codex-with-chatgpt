# Conversation Management

This reference governs session modes (`long-chat` vs. `project`), thread mapping, conversation recovery, and Project binding.

## Overview

Inspect session configuration via `c2c session -w <ws> --json` → `{ session, conversation }`.

`conversation.mode` is the active switch:
- **`long-chat`**: Retained for existing or legacy sessions (workspaces with an existing chat URL and no Project). Do not force these users to migrate. If they later request a Project, follow **Bind Project** below.
- **`project`**: Default for all new workspaces, or existing workspaces that explicitly opt in.

**Core Invariants:**
- Never match a Project or a chat by display name.
- Never upload the workspace codebase to ChatGPT Project sources / files.
- Never click 分享 / Share.
- Never rename ChatGPT conversations.

---

## 1. Long-Chat Mode

Maintains exactly ONE long-lived ChatGPT conversation per workspace.

- **Locating**: If `conversation.reuseSavedChat` is true and `conversation.chatUrl` is present, navigate to that URL (`tab.goto`) with standby and `markHandoff`, and continue execution there.
- **Saving**: After sending the boot prompt and the `workspace_info` check, and confirming the reply names this workspace:
  `c2c session set -w <ws> --mode long-chat --url <url> --title "C2C <workspace name>"`
  If the name does not match, do not overwrite a previously saved URL.
- **Updating**: After each EXECUTED or DONE event:
  `c2c session set -w <ws> --task <id> --iteration <n> --state <STATE>`
  along with local checkpoint flags (`--protocol-state`, `--waiting-for`, `--goal`, `--next-step`, `--known-issues`).
  On `DONE`: clear checkpoint (`--clear-checkpoint`) ONLY when the canonical completion condition is met: the objective is complete AND (independent acceptance is not required for the task, OR all applicable required fresh Sol/Astra checks have passed on the unchanged candidate; see [task-loop.md](task-loop.md) §Step 7). If any required check is pending, FAIL, INCONCLUSIVE, or unavailable, retain the existing checkpoint and evidence in a not-delivered state without calling `--clear-checkpoint`. Never store logs, diffs, or credentials in these fields.
- **Switching**: Switch conversations ONLY when:
  1. The user explicitly requests a new chat,
  2. The current chat experiences severe visible lag, or
  3. The current conversation is in Work mode instead of Chat mode.
  *Switching procedure:*
  1. On the same IAB tab, navigate to `https://chatgpt.com/`, confirm Chat mode ([browser.md](browser.md) §7), and send the boot prompt from `<checkout>/docs/protocol.md` §Boot Prompt.
  2. Send a `HANDOFF` message (<checkout>/docs/protocol.md §HANDOFF) containing goal, progress, state, known issues, and next expected step (never paste files or logs).
  3. Send the `workspace_info` check; only after confirmation, update the session URL with `c2c session set --url`. If validation fails, preserve the old saved URL.
- **Saved Chat 404s**: Treat as a conversation switch. Reconstruct `HANDOFF` from `session.checkpoint`. If no checkpoint exists, use `task`, `iteration`, `lastState`, and `execution_summary` metadata only. Never paste logs.

---

## 2. Project Mode (New Workspaces)

Maintains exactly ONE ChatGPT Project collection per workspace.

**Thread Mapping Rules:**
1. **Same Codex conversation thread**: Continues using the ChatGPT chat URL bound earlier in THIS thread. Navigate to that chat URL directly; do not open the Project collection first.
2. **New Codex conversation thread in the same workspace**: Opens a new chat from the Project collection page (`conversation.projectUrl`). Do not reuse another thread's chat URL just because `session.url` exists.
3. **Different workspace**: Uses a distinct Project and a distinct connector.

### Opening a Chat in This Thread

- If a ChatGPT chat URL was already saved earlier in THIS Codex conversation: navigate directly to that URL and continue. Do not open a new chat and do not send `HANDOFF`.
- Else if `conversation.projectReady` is true:
  1. Navigate to `conversation.projectUrl`.
  2. On that page, use the on-page composer (「{项目名}中的新聊天」 / "New chat in …"). Do not use the sidebar and do not `goto` `https://chatgpt.com/`.
  3. Confirm Chat mode ([browser.md](browser.md) §7).
  4. Send the boot prompt (<checkout>/docs/protocol.md §Boot Prompt), then verify tools with `workspace_info` referencing the exact `connectorName`.
  5. After the reply confirms this workspace, save the session:
     `c2c session set -w <ws> --mode project --project-url <collection> --url <chat> --connector-name "<connectorName>" --title "C2C <workspace name>"`
  6. If this Codex thread is continuing an ongoing C2C task, send `HANDOFF` immediately after the boot prompt.
- Else: perform **Bind Project** first.

### Recovery in Project Mode

- **Wrong collection**: Do not guess another Project. Tell the user the expected workspace name, ask them to navigate to the correct collection, and tell you「已找到」. Also offer the option「继续用长对话」. If they choose long-chat:
  `c2c session set -w <ws> --mode long-chat`
- **Saved chat 404s** (in this thread): Navigate to `conversation.projectUrl`, open a new chat via the on-page composer, send boot + `HANDOFF` from `session.checkpoint` (no logs) + `workspace_info` check, then save the new chat URL while retaining `--project-url`.

---

## 3. Bind Project (One-Time Setup)

Perform this workflow when initializing a new workspace or when an existing user explicitly requests migrating to Project mode. Do not attempt to create the Project via sidebar automation (Computer Use is forbidden).

1. Present the user with this exact instruction (substituting `<workspaceName>`):

```
请在 ChatGPT 里新建一个项目，名字用「<workspaceName>」，记忆请选「仅限项目记忆」。

如果侧栏里看不到「项目」：把鼠标放在「聊天」上，点右边出现的三个点，选择「按项目整理」。

建好后会打开合集页面。看到页面后跟我说「好了」。
```

2. Wait for the user to confirm「好了」. On the same IAB tab, inspect the URL bar:
   It must match `https://chatgpt.com/g/g-p-…/project`. If not, ask the user to open that project page.
   Save the project URL:
   `c2c session set -w <ws> --mode project --project-url <url> --connector-name "<connectorName>"`

3. On that collection page, open **… → 项目设置** (Project Settings).
   - **记忆 (Memory)**: 仅限项目记忆 (project-only). Keep library access (库访问权限) disabled.
   - **指令 (Instructions)**: Paste the **Project Instructions** template below (substituting `{{workspace_name}}`, `{{project_type}}`, `{{languages}}`, `{{frameworks}}`, and `{{connector_name}}` from `workspace_info` / setup). Never include public or temporary URLs in the instructions — use the connector **name** only.
   Save and close settings.

4. On the collection page, create the first chat using the on-page composer, send the boot prompt and `workspace_info` check, and save the chat URL upon confirmation.

---

## 4. Project Instructions Template

Paste this template into ChatGPT **项目设置 → 指令**:

```
You are the planning and review layer for one local workspace. Codex executes.

This Project is bound only to:
- Workspace name: {{workspace_name}}
- Kind: {{project_type}} ({{languages}} / {{frameworks}})
- Connector (use this one only): {{connector_name}}

When you call tools, use ONLY that connector. Do not use any other
Codex with ChatGPT connector. If workspace_info names a different
workspace, stop. Do not plan. Do not use this Project's memory.

Read code, git, diffs, and any released command output through that
connector. Never ask anyone to paste file bodies, diffs, or logs. After
EXECUTED, call execution_output (list, then read) when a readable item
exists; if status is restricted, review from git instead. Never upload
the repo into this Project's files or sources.

When facts conflict, trust this order:
1. Current code from the connector
2. A HANDOFF in this chat (this task's goal, progress, next step)
3. These instructions
4. This Project's memory (durable architecture only; stale memory loses)

This Project's memory is only for this workspace. On HANDOFF, trust the
brief, re-read code through the connector, and resume at NEXT_EXPECTED_STEP.

Be substantive: why, which file, what to test. No empty one-liners and
no 40-step epics. Use C2C control messages.
```

See also [../SKILL.md](../SKILL.md) for global invariants.
