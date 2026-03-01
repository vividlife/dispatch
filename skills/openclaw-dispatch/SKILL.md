---
name: openclaw-dispatch
description: "Dispatch background Claude Code workers to execute coding tasks in a specified project directory. On worker completion or question, triggers a callback to the coding agent (not the user directly) so the coding agent can evaluate results and decide next steps. The coding agent acts as supervisor: it assesses quality, iterates if needed, and only notifies the user when satisfied. Use when the OpenClaw coding agent receives a development task and needs to delegate actual code changes to a background Claude Code process."
license: MIT
version: "1.0.0"
last_updated: "2026-03-01"
user_invocable: false
---

# OpenClaw Dispatch

You are a **dispatcher** running inside the **OpenClaw coding agent**. Your job is to:
1. Plan coding tasks as checklists
2. Spawn background Claude Code workers in the user's project directory
3. Trigger coding agent callbacks when a worker finishes or asks a question — **workers call back directly; never notify the user directly**
4. As coding agent: evaluate worker results, iterate if needed, and only tell the user when you are satisfied
5. Handle user replies: write IPC answers, inject plan notes, or dispatch follow-up tasks

## Routing

Classify the incoming message first, **in this priority order**:

- **`[dispatch-callback]` prefix** — worker completed; you are the coding agent evaluating results → **Handle Task Callback**
- **`[dispatch-callback-question]` prefix** — worker is asking a question; you are the coding agent deciding the answer → **Handle Worker Question**
- **Config request** — mentions "config", "change gateway", "change token", "change notify channel", "set project", etc. → **Modifying Config**
- **Reply / follow-up** — short message, no task verb, or there are active tasks with unanswered IPC questions → **Handle Incoming Reply**
- **Task request** — anything describing work to do → **Step 0: Read Config**

**Never do the coding work yourself.** Always spawn a background Claude Code worker. You only write plan files, spawn workers, trigger agent callbacks, and write IPC answers.

---

## First-Run Setup

Triggered when `~/.dispatch/config.yaml` does not exist. Run through this, then continue with the original request.

### 1. Detect CLIs

```bash
which claude 2>/dev/null   # Claude Code (required for openclaw-dispatch)
which agent 2>/dev/null    # Cursor CLI (optional fallback)
which codex 2>/dev/null    # Codex CLI (optional fallback)
```

If `claude` is not found, tell the user: "Claude Code CLI not found. Install it with `npm install -g @anthropic-ai/claude-code`. OpenClaw dispatch requires the Claude Code CLI as the primary worker backend."

### 2. Read OpenClaw Gateway Config

Read `~/.openclaw/openclaw.json` to extract:
- `gateway.port` → gateway URL (`http://127.0.0.1:<port>`)
- `gateway.auth.token` → auth token

Also identify the Feishu group where the user talks to the `coding` agent (check `bindings` array for `agentId: "coding"` with channel `feishu`).

### 3. Present findings via AskUserQuestion

Ask the user:
- Confirm the notification channel: "I'll send task notifications to your Feishu group `<group_id>`. Is that correct?"
- Ask for the default project workspace path (e.g., `/Users/xxx/.openclaw/agents/coding/projects/`)

### 4. Generate `~/.dispatch/config.yaml`

```yaml
default: claude

backends:
  claude:
    command: >
      env -u CLAUDE_CODE_ENTRYPOINT -u CLAUDECODE
      claude -p --dangerously-skip-permissions

models:
  claude: { backend: claude }
  opus:   { backend: claude }
  sonnet: { backend: claude }
  haiku:  { backend: claude }

openclaw:
  gateway_url: "http://127.0.0.1:18789"
  gateway_token: "<token from openclaw.json>"
  default_project: "/path/to/default/project"
  notify:
    channel: "feishu"           # or "discord"
    account_id: "default"       # OpenClaw account identifier (usually "default")
    to: "<feishu_group_id>"     # Group/channel where user interacts with coding agent
```

Run `mkdir -p ~/.dispatch` then write the file.

### 5. Continue

Proceed with the original task request. No restart needed.

---

## Modifying Config

1. Read `~/.dispatch/config.yaml`. If it doesn't exist, run **First-Run Setup** first.
2. Apply the requested change (gateway URL, token, notify channel, notify target, default project path).
3. Write updated file to `~/.dispatch/config.yaml`.
4. Confirm what changed.

**Stop here for config requests.**

---

## Handle Incoming Reply

Called when the incoming message is a follow-up (not a new task). Read `~/.dispatch/config.yaml` to find the active project.

### Step 1: Detect Active Tasks

```bash
ls <project_dir>/.dispatch/tasks/*/ipc/*.question 2>/dev/null
```

### Step 2: Route the Reply

**Case A — Unanswered IPC question exists:**

Find the `*.question` file(s) without a matching `*.answer`. Take the user's message as the answer to the earliest unanswered question.

Write the answer atomically:
```bash
echo "<user's reply>" > <task_ipc_dir>/<seq>.answer.tmp
mv <task_ipc_dir>/<seq>.answer.tmp <task_ipc_dir>/<seq>.answer
```

Tell the user: "Answer sent to the worker. It will continue shortly."

The worker is polling for the answer and will resume automatically.

**Case B — Task is running but no pending question:**

Check if `.dispatch/tasks/<task-id>/ipc/.done` exists. If not, the task is still in progress.

Append the user's message as a note to the plan file:
```markdown

> **Note from dispatcher:** <user's message>
```

Tell the user: "Note added to the active task. The worker will read it before its next step."

**Case C — Task is complete (`.done` exists) and user sends a follow-up task:**

Treat the message as a new task request. Go to **Step 0: Read Config** with:
- Same project directory as the completed task
- The new follow-up as the task description

**Case D — No active tasks:**

Ask the user to clarify: "No active tasks found. Would you like to dispatch a new task, or check a specific project?"

---

## Handle Task Callback

Triggered when a worker sends a `[dispatch-callback]` message after completing (or timing out).

### Step 1: Parse the callback

Extract from the message:
- `TASK_ID` — the backtick-wrapped task ID after `任务`
- `plan_content` — everything after `计划状态：`
- `output_content` — everything after `输出结果：`

Also determine `PROJECT_DIR` (read `~/.dispatch/last-project` or infer from context).

### Step 2: Evaluate quality

Read the plan to verify completion:
- Are all items `[x]`?
- Are there any `[?]` (blocked) or `[!]` (error) items?

Assess the output:
- Is the task fully and correctly implemented?
- Is there anything obviously missing or broken?

### Step 3: Decide

**Satisfied:**
Reply to the user (the `--deliver` flag means your reply goes to Feishu automatically):
> ✅ 任务 `<task-id>` 已完成！
> <brief summary from output.md>
>
> 如需继续，请发送新指令。

**Not satisfied / incomplete:**
- Read the iteration counter: `cat <project_dir>/.dispatch/tasks/<task-id>/iterations 2>/dev/null || echo 0`
- If counter < 3: increment it, then re-dispatch a follow-up worker with a targeted prompt describing what's missing. Go to **Step 1** (same project dir, same task ID base, append `-iter<N>` suffix).
- If counter ≥ 3: inform the user that 3 iterations have been attempted and share what was done / what's still missing.

**Needs user judgment:**
Include your evaluation and a specific question in the reply to the user.

**Stop here** — do not loop or poll. The worker for any new iteration will trigger the next callback.

---

## Handle Worker Question

Triggered when a worker sends a `[dispatch-callback-question]` message.

### Step 1: Parse the callback

Extract from the message:
- `TASK_ID` — the backtick-wrapped task ID
- `seq` — the sequence number `#NNN` (e.g. `001`)
- `question_text` — the worker's question
- `IPC_DIR` — the IPC directory path shown in the message

### Step 2: Decide how to answer

**Can answer autonomously** (based on project context, task plan, or common sense):
1. Write the answer atomically to the IPC directory:
   ```bash
   echo '<answer>' > <IPC_DIR>/<seq>.answer.tmp
   mv <IPC_DIR>/<seq>.answer.tmp <IPC_DIR>/<seq>.answer
   ```
2. Reply to the user (via `--deliver`):
   > Worker 对任务 `<task-id>` 的提问已自动回答，继续执行中。
   > 问题：`<question_text>`
   > 我的回答：`<answer>`

**Needs user decision:**
Reply to the user:
> ❓ 任务 `<task-id>` 的 worker 需要您决策（#<seq>）：
>
> <question_text>
>
> 请直接回复您的答案。

The user's reply will arrive as a new message routed to **Handle Incoming Reply → Case A**, which writes the IPC answer.

**Stop here** after replying. The worker continues polling for the `.answer` file.

---

## Step 0: Read Config

### Config file: `~/.dispatch/config.yaml`

Read this file. If it doesn't exist → run **First-Run Setup**, then continue.

### Extract Project Directory

The project directory is determined in this priority order:
1. **Explicitly stated in the message** — "in `/path/to/project`", "项目在 /xxx", etc.
2. **Most recently used project** — check `~/.dispatch/last-project` (written after each dispatch).
3. **Default project** — `openclaw.default_project` from config.
4. **Fallback** — ask the user: "Which project directory should I dispatch this to?"

After determining the project path, write it for next time:
```bash
echo "/path/to/project" > ~/.dispatch/last-project
```

### Read OpenClaw Notify Config

Extract from `openclaw:` section:
- `gateway_url`
- `gateway_token`
- `notify.channel`, `notify.account_id`, `notify.to`

These are injected into the worker prompt in Step 2.

---

**Everything below is for TASK REQUESTS only.**

**CRITICAL RULE: Never do the coding work yourself. You ONLY write plan files, spawn workers, trigger agent callbacks, and write IPC answers.**

---

## Step 1: Create the Plan File

Write the plan file at `<project_dir>/.dispatch/tasks/<task-id>/plan.md`:

```markdown
# <Task Title>

- [ ] First concrete step
- [ ] Second concrete step
- [ ] Third concrete step
- [ ] Write summary of changes to .dispatch/tasks/<task-id>/output.md
```

Rules:
- Each item is a **concrete, verifiable action**.
- Match plan size to task complexity.
- Last item produces output when the task warrants it.
- Use the Write tool. This is the one artifact the user should see.

## Step 2: Set Up and Spawn

### UX Principle

Minimize visible tool calls. All scaffolding (IPC dir, prompt file, worker script) goes in **one Bash call**. Use a clear description.

### Dispatch Procedure

1. **Create all scaffolding in one Bash call:**

   ```bash
   # description: "Set up openclaw-dispatch scaffolding for <task-id>"
   mkdir -p <project_dir>/.dispatch/tasks/<task-id>/ipc

   cat > /tmp/dispatch-<task-id>-prompt.txt << 'PROMPT'
   <worker prompt content>
   PROMPT

   cat > /tmp/worker--<task-id>.sh << 'WORKER'
   #!/bin/bash
   cd <project_dir>
   env -u CLAUDE_CODE_ENTRYPOINT -u CLAUDECODE \
     claude -p --dangerously-skip-permissions \
     "$(cat /tmp/dispatch-<task-id>-prompt.txt)" 2>&1
   WORKER

   chmod +x /tmp/worker--<task-id>.sh
   ```

2. **Spawn worker as a background task:**

   ```bash
   # description: "Run openclaw worker: <task-id>"
   bash /tmp/worker--<task-id>.sh
   ```

   Record the task ID internally. Do NOT report it to the user.

### Worker Prompt Template

```
You have a plan file at .dispatch/tasks/{task-id}/plan.md containing a checklist.
Work through it top to bottom. For each item, do the work, update the plan file ([ ] → [x] with an optional note), and move to the next.

If you need to ask a question:
1. Atomic write: echo '<question>' > .dispatch/tasks/{task-id}/ipc/<NNN>.question.tmp && mv .dispatch/tasks/{task-id}/ipc/<NNN>.question.tmp .dispatch/tasks/{task-id}/ipc/<NNN>.question (sequence from 001)
2. Immediately call (fire-and-forget):
   openclaw agent \
     --agent coding \
     --message "[dispatch-callback-question] 任务 `{task-id}` 的 worker 提问（#<NNN>）：\n\n<question>\n\n请直接回答或向用户提问。回答后写入 IPC answer。" \
     --deliver \
     --reply-channel {notify_channel} \
     --reply-to {notify_to} \
     --json > /dev/null 2>&1 &
3. Poll for .dispatch/tasks/{task-id}/ipc/<NNN>.answer every 5 seconds (max 5 minutes). When received, continue. If timed out, write context to .dispatch/tasks/{task-id}/context.md, mark item [?], and stop.

If you hit an unresolvable error, mark the item [!] with a description and stop.

When all checklist items are done:
1. Write a summary to .dispatch/tasks/{task-id}/output.md
2. Then call back synchronously:
   touch .dispatch/tasks/{task-id}/ipc/.done
   openclaw agent \
     --agent coding \
     --message "[dispatch-callback] 任务 `{task-id}` 已执行完毕。\n\n计划状态：\n$(cat .dispatch/tasks/{task-id}/plan.md)\n\n输出结果：\n$(cat .dispatch/tasks/{task-id}/output.md 2>/dev/null)" \
     --deliver \
     --reply-channel {notify_channel} \
     --reply-to {notify_to} \
     --json > /dev/null 2>&1

Note: {notify_channel} and {notify_to} are literal values injected when the worker prompt is written — no runtime config reading needed.

Context:
<describe the outcome, reference files, constraints — same rules as dispatch SKILL.md>
```

### Task IDs

Short, descriptive, kebab-case: `add-auth`, `fix-login-bug`, `refactor-api`.

## Step 3: Report and Return Control

After dispatching, tell the user:
- Which task was dispatched (task ID + brief plan summary)
- That the worker is running in the background
- That **you (the coding agent) will check the results** when the worker finishes, and will only notify the user once satisfied
- Then **stop and wait** — do not poll or loop

Example:
> 已派发任务 `add-auth`（Claude Code，`/Users/xxx/myapp`）。计划：
> 1. 读取现有认证代码
> 2. 添加 JWT 中间件
> 3. 添加登录/登出接口
> 4. 写测试用例
>
> Worker 执行完成后，我会先检查结果质量，满意后再告知您。如有需要您决策的问题，也会第一时间转告。

**Do NOT** report worker task IDs, script paths, or backend details.

---

## Checking Progress

**When the user asks** ("status", "进展怎么样?", "check"):

```bash
cat <project_dir>/.dispatch/tasks/<task-id>/plan.md
```

Report checklist state. Check for unanswered questions:
```bash
ls <project_dir>/.dispatch/tasks/<task-id>/ipc/*.question 2>/dev/null
```

Interpret markers:
- `[x]` = done
- `[ ]` = pending / in progress
- `[?]` = blocked on a question — surface it to the user
- `[!]` = error — report it

---

## Error Handling

- `[!]` in plan: report the error, ask user to retry or skip.
- Worker exited with unchecked items: report which were done, ask if user wants to re-dispatch remaining items.
- Worker failed to start: check CLI availability (`which claude`). If missing, report and ask user to install.

## Parallel Tasks

Dispatch separate workers with separate plan files. Both run concurrently.

## Sequential Dependencies

Dispatch task A. When the `[dispatch-callback]` arrives and you (coding agent) approve the result, dispatch task B.

## Adding Context to a Running Worker

If the user sends additional context while a task is running (no `.done` yet), append it to the plan file as a note — **do not** write to IPC:

```markdown
> **Note from dispatcher:** <user's additional context>
```

The worker reads the plan file as it proceeds, so it will see the note before its next step.

---

## Example Interactions

### Normal flow

```
User (Feishu): 帮我在 /Users/me/.openclaw/agents/coding/projects/myapp 实现用户登录功能

Dispatcher: [reads config, extracts project path]
Dispatcher: [writes plan.md]
Dispatcher: [single Bash: creates IPC dir, prompt, worker script]
Dispatcher: [spawns worker as background task]
Dispatcher (→ User, Feishu): 已派发任务 `add-login` 到 /projects/myapp，使用 Claude Code。
计划：
1. 读取现有代码结构
2. 实现登录接口
3. 添加 JWT 验证
4. 写单元测试
5. 输出变更摘要

Worker 完成后，我会先评估结果，满意后告知您。

[Worker calls openclaw agent directly: openclaw agent --agent coding --message "[dispatch-callback-question] ..."]

Coding agent (Handle Worker Question):
  [reads question: "现有代码用的是 session 还是 token 认证？"]
  [can answer from context: project uses JWT]
  [writes .dispatch/tasks/add-login/ipc/001.answer atomically]
Coding agent (→ User, Feishu): Worker 提问已自动回答，继续执行中。
  问题：现有代码用的是 session 还是 token 认证？
  回答：JWT token

[Worker calls openclaw agent directly: openclaw agent --agent coding --message "[dispatch-callback] ..."]

Coding agent (Handle Task Callback):
  [reads plan.md: all [x]]
  [reads output.md: JWT 登录接口，POST /api/login 和 POST /api/logout...]
  [evaluates: satisfactory]
Coding agent (→ User, Feishu): ✅ 任务 `add-login` 已完成！
  实现了 JWT 登录接口，新增 POST /api/login 和 POST /api/logout。
  如需继续，请发送新指令。
```

### Follow-up new subtask

```
User (Feishu): 顺便帮我加上注册功能

Dispatcher (Handle Incoming Reply → Case C: task is done, new request):
  [dispatches new task `add-register` in same project dir]
Dispatcher: 已派发新任务 `add-register`，将在 `add-login` 的基础上继续。
```

### Context injection mid-task

```
User (Feishu): 对了，数据库用的是 PostgreSQL，不是 MySQL

Dispatcher (Handle Incoming Reply → Case B: task running, no pending question):
  [appends note to plan.md]
Dispatcher: 已将数据库信息补充到任务计划，worker 执行到下一步时会读取到。
```
