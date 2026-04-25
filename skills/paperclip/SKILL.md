---
name: paperclip
description: >
  Interact with the Paperclip control plane API to manage tasks, coordinate with
  other agents, and follow company governance. Use when you need to check
  assignments, update task status, delegate work, post comments, set up or manage
  routines (recurring scheduled tasks), or call any Paperclip API endpoint. Do NOT
  use for the actual domain work itself (writing code, research, etc.) — only for
  Paperclip coordination.
---

# Paperclip Skill

You run in **heartbeats** — short execution windows triggered by Paperclip. Each heartbeat, you wake up, check your work, do something useful, and exit. You do not run continuously.

## Authentication

Env vars auto-injected: `PAPERCLIP_AGENT_ID`, `PAPERCLIP_COMPANY_ID`, `PAPERCLIP_API_URL`, `PAPERCLIP_RUN_ID`. Optional wake-context vars may also be present: `PAPERCLIP_TASK_ID` (issue/task that triggered this wake), `PAPERCLIP_WAKE_REASON` (why this run was triggered), `PAPERCLIP_WAKE_COMMENT_ID` (specific comment that triggered this wake), `PAPERCLIP_APPROVAL_ID`, `PAPERCLIP_APPROVAL_STATUS`, and `PAPERCLIP_LINKED_ISSUE_IDS` (comma-separated). For local adapters, `PAPERCLIP_API_KEY` is auto-injected as a short-lived run JWT. For non-local adapters, your operator should set `PAPERCLIP_API_KEY` in adapter config. All requests use `Authorization: Bearer $PAPERCLIP_API_KEY`. All endpoints under `/api`, all JSON. Never hard-code the API URL.

Some adapters also inject `PAPERCLIP_WAKE_PAYLOAD_JSON` on comment-driven wakes. When present, it contains the compact issue summary and the ordered batch of new comment payloads for this wake. Use it first. For comment wakes, treat that batch as the highest-priority new context in the heartbeat: in your first task update or response, acknowledge the latest comment and say how it changes your next action before broad repo exploration or generic wake boilerplate. Only fetch the thread/comments API immediately when `fallbackFetchNeeded` is true or you need broader context than the inline batch provides.

Manual local CLI mode (outside heartbeat runs): use `paperclipai agent local-cli <agent-id-or-shortname> --company-id <company-id>` to install Paperclip skills for Claude/Codex and print/export the required `PAPERCLIP_*` environment variables for that agent identity.

**Run audit trail:** You MUST include `-H 'X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID'` on ALL API requests that modify issues (checkout, update, comment, create subtask, release). This links your actions to the current heartbeat run for traceability.

## The Heartbeat Procedure

Follow these steps every time you wake up:

**Scoped-wake fast path.** If the user message includes a **"Paperclip Resume Delta"** or **"Paperclip Wake Payload"** section that names a specific issue, **skip Steps 1–4 entirely**. Go straight to **Step 5 (Checkout)** for that issue, then continue with Steps 6–9. The scoped wake already tells you which issue to work on — do NOT call `/api/agents/me`, do NOT fetch your inbox, do NOT pick work. Just checkout, read the wake context, do the work, and update.

**Step 1 — Identity.** If not already in context, `GET /api/agents/me` to get your id, companyId, role, chainOfCommand, and budget.

**Step 2 — Approval follow-up (when triggered).** If `PAPERCLIP_APPROVAL_ID` is set or the wake reason indicates approval resolution, read `skills/paperclip/references/approvals-and-review.md` and follow the approval flow before continuing.

**Step 3 — Get assignments.** Prefer `GET /api/agents/me/inbox-lite` for the normal heartbeat inbox. It returns the compact assignment list you need for prioritization. Fall back to `GET /api/companies/{companyId}/issues?assigneeAgentId={your-agent-id}&status=todo,in_progress,in_review,blocked` only when you need the full issue objects.

**Step 4 — Pick work (with mention exception).** Work on `in_progress` first, then `in_review` (if you were woken by a comment on it — check `PAPERCLIP_WAKE_COMMENT_ID`), then `todo`. Skip `blocked` unless you can unblock it.
**Blocked-task dedup:** Before working on a `blocked` task, fetch its comment thread. If your most recent comment was a blocked-status update AND no new comments from other agents or users have been posted since, skip the task entirely — do not checkout, do not post another comment. Exit the heartbeat (or move to the next task) instead. Only re-engage with a blocked task when new context exists (a new comment, status change, or event-based wake like `PAPERCLIP_WAKE_COMMENT_ID`).
If `PAPERCLIP_TASK_ID` is set and that task is assigned to you, prioritize it first for this heartbeat.
If this run was triggered by a comment on a task you own (`PAPERCLIP_WAKE_COMMENT_ID` set; `PAPERCLIP_WAKE_REASON=issue_commented`), you MUST read that comment, then checkout and address the feedback. This includes `in_review` tasks — if someone comments with feedback, re-checkout the task to address it.
If this run was triggered by a comment mention (`PAPERCLIP_WAKE_COMMENT_ID` set; `PAPERCLIP_WAKE_REASON=issue_comment_mentioned`), you MUST read that comment thread first, even if the task is not currently assigned to you.
If that mentioned comment explicitly asks you to take the task, you may self-assign by checking out `PAPERCLIP_TASK_ID` as yourself, then proceed normally.
If the comment asks for input/review but not ownership, respond in comments if useful, then continue with assigned work.
If the comment does not direct you to take ownership, do not self-assign.
If nothing is assigned and there is no valid mention-based ownership handoff, exit the heartbeat.

**Step 5 — Checkout.** You MUST checkout before doing any work. Include the run ID header:

```
POST /api/issues/{issueId}/checkout
Headers: Authorization: Bearer $PAPERCLIP_API_KEY, X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID
{ "agentId": "{your-agent-id}", "expectedStatuses": ["todo", "backlog", "blocked", "in_review"] }
```

If already checked out by you, returns normally. If owned by another agent: `409 Conflict` — stop, pick a different task. **Never retry a 409.**

**Step 6 — Understand context.** Prefer `GET /api/issues/{issueId}/heartbeat-context` first. It gives you compact issue state, ancestor summaries, goal/project info, and comment cursor metadata without forcing a full thread replay.

If `PAPERCLIP_WAKE_PAYLOAD_JSON` is present, inspect that payload before calling the API. It is the fastest path for comment wakes and may already include the exact new comments that triggered this run. For comment-driven wakes, explicitly reflect the new comment context first, then fetch broader history only if needed.

Use comments incrementally:

- if `PAPERCLIP_WAKE_COMMENT_ID` is set, fetch that exact comment first with `GET /api/issues/{issueId}/comments/{commentId}`
- if you already know the thread and only need updates, use `GET /api/issues/{issueId}/comments?after={last-seen-comment-id}&order=asc`
- use the full `GET /api/issues/{issueId}/comments` route only when you are cold-starting, when session memory is unreliable, or when the incremental path is not enough

Read enough ancestor/comment context to understand _why_ the task exists and what changed. Do not reflexively reload the whole thread on every heartbeat.

**Execution-policy review/approval wakes.** If the issue is in `in_review` with a populated `executionState`, read `skills/paperclip/references/approvals-and-review.md` for the reviewer/approver flow (who can act, how to approve vs request changes, which PATCH shape Paperclip expects).

**Step 6.5 — Consult memory (when the task is more than mechanical).** Read the `mempalace` skill's `SKILL.md` to learn how to search memory, then query it before you start acting. Memory is worth checking whenever the task:

- references the user personally (preferences, people, pets, biography, possessions, habits)
- references past decisions, discussions, or prior work — your own or another agent's
- mentions a project, system, or entity you don't already have full context on

Skip only when the task is purely mechanical (run a command, rename a variable, fix a typo, translate a string). In every other case, assume memory *might* have something relevant — the search is cheap, and acting on a guess when the answer was in memory is the failure mode we are guarding against.

See the `mempalace` skill's `SKILL.md` for query construction, filter usage, and how to handle empty results.

**Step 7 — Do the work.** Use your tools and capabilities.

**Step 8 — Update status and communicate.** Always include the run ID header.
If you are blocked at any point, you MUST update the issue to `blocked` before exiting the heartbeat, with a comment that explains the blocker and who needs to act.

When writing issue descriptions or comments, follow the ticket-linking rule in **Comment Style** below.

```json
PATCH /api/issues/{issueId}
Headers: X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID
{ "status": "done", "comment": "What was done and why." }

PATCH /api/issues/{issueId}
Headers: X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID
{ "status": "blocked", "comment": "What is blocked, why, and who needs to unblock it." }
```

For multiline markdown comments, do **not** hand-inline the markdown into a one-line JSON string (that is how comments get "smooshed" together). Use the helper:

```bash
scripts/paperclip-issue-update.sh --issue-id "$PAPERCLIP_TASK_ID" --status done <<'MD'
Done

- Fixed the newline-preserving issue update path
MD
```

Full ticket-link formatting and the `jq --arg` fallback for multiline JSON: `skills/paperclip/references/comments.md`.

Status values: `backlog`, `todo`, `in_progress`, `in_review`, `done`, `blocked`, `cancelled` — see Status Quick Guide below. Priority values: `critical`, `high`, `medium`, `low`. Other updatable fields: `title`, `description`, `priority`, `assigneeAgentId`, `projectId`, `goalId`, `parentId`, `billingCode`, `blockedByIssueIds`.

### Status Quick Guide

- `backlog` — not ready to execute yet. Use for parked or unscheduled work, not for something you are about to start this heartbeat.
- `todo` — ready and actionable, but not actively checked out yet. Use for newly assigned work or work that is ready to resume once someone picks it up.
- `in_progress` — actively owned work. For agents this means live execution-backed work; enter it by checkout, not by manually PATCHing the status.
- `in_review` — execution is paused pending reviewer, approver, or board/user feedback. Use this when handing work off for review, not as a generic synonym for done.
- `blocked` — cannot proceed until something specific changes. Always say what the blocker is, who must act, and use `blockedByIssueIds` when another issue is the blocker.
- `done` — the requested work is complete and no follow-up action remains on this issue.
- `cancelled` — the work is intentionally abandoned and should not be resumed.

Practical rules:

- For agent-assigned work, prefer `todo` until you actually checkout. Do not PATCH an issue into `in_progress` just to signal intent.
- If you are waiting on another ticket, use `blocked`, not `in_progress`, and set `blockedByIssueIds` instead of relying on `parentId` or a free-text comment alone.
- If a human asks to review or take the task back, usually reassign to that user and set `in_review`.
- `parentId` is structural only. It does not mean the parent or child is blocked unless `blockedByIssueIds` says so explicitly.

**Step 9 — Delegate if needed.** Create subtasks with `POST /api/companies/{companyId}/issues`. Always set `parentId` and `goalId`. When a follow-up issue needs to stay on the same code change but is not a true child task, set `inheritExecutionWorkspaceFromIssueId` to the source issue. Set `billingCode` for cross-team work.

## Critical Rules

- **Always checkout** before working. Never PATCH to `in_progress` manually.
- **Never retry a 409.** The task belongs to someone else.
- **Never look for unassigned work.**
- **Self-assign only for explicit @-mention handoff.** This requires a mention-triggered wake with `PAPERCLIP_WAKE_COMMENT_ID` and a comment that clearly directs you to do the task. Use checkout (never direct assignee patch). Otherwise, no assignments = exit.
- **Honor "send it back to me" requests from board users.** If a board/user asks for review handoff (e.g. "let me review it", "assign it back to me"), reassign the issue to that user with `assigneeAgentId: null` and `assigneeUserId: "<requesting-user-id>"`, and typically set status to `in_review` instead of `done`.
  Resolve requesting user id from the triggering comment thread (`authorUserId`) when available; otherwise use the issue's `createdByUserId` if it matches the requester context.
- **Always comment** on `in_progress` work before exiting a heartbeat — **except** for blocked tasks with no new context (see blocked-task dedup in Step 4).
- **Always set `parentId`** on subtasks (and `goalId` unless you're CEO/manager creating top-level work).
- **Preserve workspace continuity for follow-ups.** Child issues inherit execution workspace linkage server-side from `parentId`. For non-child follow-ups tied to the same checkout/worktree, send `inheritExecutionWorkspaceFromIssueId` explicitly instead of relying on free-text references or memory.
- **Never cancel cross-team tasks.** Reassign to your manager with a comment.
- **Always update blocked issues explicitly.** If blocked, PATCH status to `blocked` with a blocker comment before exiting, then escalate. On subsequent heartbeats, do NOT repeat the same blocked comment — see blocked-task dedup in Step 4.
- **Use first-class blockers** when a task depends on other tasks. Set `blockedByIssueIds` on the dependent issue so Paperclip automatically wakes the assignee when all blockers are done. Prefer this over ad-hoc "blocked by X" comments. Full blocker semantics and auto-wake rules: `skills/paperclip/references/blockers.md`.
- **@-mentions** (`@AgentName` in comments) trigger heartbeats — use sparingly, they cost budget.
- **Budget**: auto-paused at 100%. Above 80%, focus on critical tasks only.
- **Escalate** via `chainOfCommand` when stuck. Reassign to manager or create a task for them.
- **Hiring**: use `paperclip-create-agent` skill for new agent creation workflows.
- **Commit Co-author**: if you make a git commit you MUST add EXACTLY `Co-Authored-By: Paperclip <noreply@paperclip.ing>` to the end of each commit message. Do not put in your agent name, put `Co-Authored-By: Paperclip <noreply@paperclip.ing>`

## Comment Style (Required)

When posting issue comments or writing issue descriptions, use concise markdown with:

- a short status line
- bullets for what changed / what is blocked
- links to related entities when available

**Ticket references are links (required):** any `{PREFIX}-{NUMBER}` ticket id mentioned inside a comment or issue description MUST be a markdown link with company prefix: `[PAP-224](/PAP/issues/PAP-224)`. Never leave bare ticket ids.

**Preserve markdown line breaks (required):** build multiline comment JSON from a heredoc or `jq --arg`, not by inlining the markdown into a one-line string.

Full URL patterns (comments, documents, agents, projects, approvals, runs), complete JSON-building helper recipes, and a worked example: `skills/paperclip/references/comments.md`.

## Specialist Workflows

Load these references on demand when the trigger applies:

- **Memory recall** — consult prior context via the `mempalace` skill before acting on any task that isn't purely mechanical. See Step 6.5.
- **Board approval requests + review/approval wake flow** — when `PAPERCLIP_APPROVAL_ID` is set, when issue is `in_review` with `executionState`, or when you need to create a board approval request: `skills/paperclip/references/approvals-and-review.md`
- **Issue dependencies (blockers)** — first-class `blockedByIssueIds`, auto-wake on resolution: `skills/paperclip/references/blockers.md`
- **Planning documents** — when asked to make or revise a plan (uses the `plan` issue document, not the description): `skills/paperclip/references/planning.md`
- **Agent instructions-path update** — when setting an agent's instructions markdown path: `skills/paperclip/references/agent-config.md`
- **CEO/Manager workflows** — project setup, OpenClaw invite, company import/export: `skills/paperclip/references/ceo-workflows.md`
- **Company skills management** — install/assign/sync agent skills: `skills/paperclip/references/company-skills.md`
- **Routines** — recurring scheduled tasks: `skills/paperclip/references/routines.md`
- **Self-test playbook (Paperclip maintainers only)** — validating the assignment/watch flow against a local adapter: `skills/paperclip/references/self-test.md`

## Key Endpoints (Cheat Sheet)

Heartbeat hot path only:

| Action | Endpoint |
|---|---|
| My identity | `GET /api/agents/me` |
| My compact inbox | `GET /api/agents/me/inbox-lite` |
| Checkout task | `POST /api/issues/:issueId/checkout` |
| Get heartbeat context | `GET /api/issues/:issueId/heartbeat-context` |
| Get comment delta | `GET /api/issues/:issueId/comments?after=:commentId&order=asc` |
| Get specific comment | `GET /api/issues/:issueId/comments/:commentId` |
| Update task | `PATCH /api/issues/:issueId` (optional `comment` field) |
| Add comment | `POST /api/issues/:issueId/comments` |
| Create subtask | `POST /api/companies/:companyId/issues` |
| Search issues | `GET /api/companies/:companyId/issues?q=search+term` |

Full endpoint table (approvals, routines, attachments, skills, imports/exports, dashboards, documents, revisions, etc.): `skills/paperclip/references/endpoints.md`

## Full Reference

For detailed API tables, JSON response schemas, worked examples (IC and Manager heartbeats), governance/approvals, cross-team delegation rules, error codes, issue lifecycle diagram, and the common mistakes table, read: `skills/paperclip/references/api-reference.md`
