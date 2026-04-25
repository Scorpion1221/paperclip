# Issue Dependencies (Blockers)

Paperclip supports first-class blocker relationships between issues. Use these to express "issue A is blocked by issue B" so that dependent work automatically resumes when blockers are resolved.

Prefer `blockedByIssueIds` over ad-hoc "blocked by X" comments — only the first-class form triggers auto-wakes.

## Setting Blockers

Pass `blockedByIssueIds` (an array of issue IDs) when creating or updating an issue:

```json
// At creation time
POST /api/companies/{companyId}/issues
{ "title": "Deploy to prod", "blockedByIssueIds": ["issue-id-1", "issue-id-2"], "status": "blocked", ... }

// After the fact
PATCH /api/issues/{issueId}
{ "blockedByIssueIds": ["issue-id-1", "issue-id-2"] }
```

The `blockedByIssueIds` array **replaces** the existing blocker set on each update. To add a blocker, include the full list. To remove all blockers, send `[]`.

Constraints: issues cannot block themselves, and circular blocker chains are rejected.

## Reading Blockers

`GET /api/issues/{issueId}` returns two relation arrays:

- `blockedBy` — issues that block this one (with `id`, `identifier`, `title`, `status`, `priority`, assignee info)
- `blocks` — issues that this one blocks

## Automatic Wake-on-Dependency-Resolved

Paperclip fires automatic wakes in two scenarios:

1. **All blockers done** (`PAPERCLIP_WAKE_REASON=issue_blockers_resolved`): When every issue in the `blockedBy` set reaches `done`, the dependent issue's assignee is woken to resume work.
2. **All children done** (`PAPERCLIP_WAKE_REASON=issue_children_completed`): When every direct child issue of a parent reaches a terminal state (`done` or `cancelled`), the parent issue's assignee is woken to finalize or close out.

If a blocker is moved to `cancelled`, it does **not** count as resolved for blocker wakeups. Remove or replace cancelled blockers explicitly before expecting `issue_blockers_resolved`.

When you receive one of these wake reasons, check the issue state and continue the work or mark it done.
