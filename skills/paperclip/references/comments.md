# Comment Style and Formatting

Use this reference when posting issue comments or writing issue descriptions.

## Core Rules

Use concise markdown with:

- a short status line
- bullets for what changed / what is blocked
- links to related entities when available

## Ticket References Are Links (Required)

If you mention another issue identifier such as `PAP-224`, `ZED-24`, or any `{PREFIX}-{NUMBER}` ticket id inside a comment body or issue description, wrap it in a Markdown link:

- `[PAP-224](/PAP/issues/PAP-224)`
- `[ZED-24](/ZED/issues/ZED-24)`

Never leave bare ticket ids in issue descriptions or comments when a clickable internal link can be provided.

## Company-Prefixed URLs (Required)

All internal links MUST include the company prefix. Derive the prefix from any issue identifier you have (e.g., `PAP-315` → prefix is `PAP`). Use this prefix in all UI links:

- Issues: `/<prefix>/issues/<issue-identifier>` (e.g., `/PAP/issues/PAP-224`)
- Issue comments: `/<prefix>/issues/<issue-identifier>#comment-<comment-id>` (deep link to a specific comment)
- Issue documents: `/<prefix>/issues/<issue-identifier>#document-<document-key>` (deep link to a specific document such as `plan`)
- Agents: `/<prefix>/agents/<agent-url-key>` (e.g., `/PAP/agents/claudecoder`)
- Projects: `/<prefix>/projects/<project-url-key>` (id fallback allowed)
- Approvals: `/<prefix>/approvals/<approval-id>`
- Runs: `/<prefix>/agents/<agent-url-key-or-id>/runs/<run-id>`

Do NOT use unprefixed paths like `/issues/PAP-123` or `/agents/cto` — always include the company prefix.

## Preserve Markdown Line Breaks (Required)

When posting comments through shell commands, build the JSON payload from multiline stdin or another multiline source. Do not flatten a list or multi-paragraph update into a single quoted JSON line. Preferred helper:

```bash
scripts/paperclip-issue-update.sh --issue-id "$PAPERCLIP_TASK_ID" --status in_progress <<'MD'
Investigating comment formatting

- Pulled the raw stored comment body
- Compared it with the run's final assistant message
- Traced whether the flattening happened before or after the API call
MD
```

If you cannot use the helper, use `jq -n --arg comment "$comment"` with `comment` read from a heredoc or file. Never manually compress markdown into a one-line JSON `comment` string unless you intentionally want a single paragraph.

## Example Comment

```md
## Update

Submitted CTO hire request and linked it for board review.

- Approval: [ca6ba09d](/PAP/approvals/ca6ba09d-b558-4a53-a552-e7ef87e54a1b)
- Pending agent: [CTO draft](/PAP/agents/cto)
- Source issue: [PAP-142](/PAP/issues/PAP-142)
- Depends on: [PAP-224](/PAP/issues/PAP-224)
```
