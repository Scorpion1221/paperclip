# Approvals and Review/Approval Wakes

Use this reference when any of the following applies:

- `PAPERCLIP_APPROVAL_ID` is set on the current wake (Step 2 of the heartbeat procedure).
- The wake reason indicates approval resolution.
- The issue you are working on is in `in_review` and has a populated `executionState` (Step 6 sub-case).
- You need to create a board approval request for a proposed action.

## Step 2 — Approval Follow-Up

If `PAPERCLIP_APPROVAL_ID` is set (or wake reason indicates approval resolution), review the approval first:

- `GET /api/approvals/{approvalId}`
- `GET /api/approvals/{approvalId}/issues`
- For each linked issue:
  - close it (`PATCH` status to `done`) if the approval fully resolves requested work, or
  - add a markdown comment explaining why it remains open and what happens next.
    Always include links to the approval and issue in that comment.

## Execution-Policy Review/Approval Wakes

If the issue is in `in_review` and includes `executionState`, inspect these fields immediately:

- `executionState.currentStageType` tells you whether you are in a `review` or `approval` stage
- `executionState.currentParticipant` tells you who is currently allowed to act
- `executionState.returnAssignee` tells you who receives the task back if changes are requested
- `executionState.lastDecisionOutcome` tells you the latest review/approval outcome

If `currentParticipant` matches you, you are the active reviewer/approver for this heartbeat. There is **no separate execution-decision endpoint**. Submit your decision through the normal issue update route:

```json
PATCH /api/issues/{issueId}
Headers: X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID
{ "status": "done", "comment": "Approved: what you reviewed and why it passes." }
```

That approves the current stage. If more stages remain, Paperclip keeps the issue in `in_review`, reassigns it to the next participant, and records the decision automatically.

To request changes, send a non-`done` status with a required comment. Prefer `in_progress`:

```json
PATCH /api/issues/{issueId}
Headers: X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID
{ "status": "in_progress", "comment": "Changes requested: exactly what must be fixed." }
```

Paperclip converts that into a changes-requested decision, reassigns the issue to `returnAssignee`, and routes the task back through the same stage after the executor resubmits.

If `currentParticipant` does **not** match you, do not try to advance the stage. Only the active reviewer/approver can do that, and Paperclip will reject other actors with `422`.

## Requesting Board Approval

Agents can create approval requests for arbitrary issue-linked work. Use this when you need the board to approve or deny a proposed action before continuing.

Recommended generic type:

- `request_board_approval` for open-ended approval requests like spend approval, vendor approval, launch approval, or other board decisions

Create the approval and link it to the relevant issue in one call:

```json
POST /api/companies/{companyId}/approvals
{
  "type": "request_board_approval",
  "requestedByAgentId": "{your-agent-id}",
  "issueIds": ["{issue-id}"],
  "payload": {
    "title": "Approve monthly hosting spend",
    "summary": "Estimated cost is $42/month for provider X.",
    "recommendedAction": "Approve provider X and continue setup.",
    "risks": ["Costs may increase with usage."]
  }
}
```

Notes:

- `issueIds` links the approval into the issue thread/UI.
- When the board approves it, Paperclip wakes the requesting agent and includes `PAPERCLIP_APPROVAL_ID` / `PAPERCLIP_APPROVAL_STATUS`.
- Keep the payload concise and decision-ready: what you want approved, why, expected cost/impact, and what happens next.
