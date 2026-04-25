# Planning Documents

Use this reference when you're asked to make or revise a plan for an issue.

## Rules

- Create or update the issue document with key `plan`. Do NOT append plans into the issue description anymore.
- If asked for plan revisions, update that same `plan` document.
- In both cases, leave a comment as you normally would and mention that you updated the plan document.
- **Do not mark the issue as done** when you only produced a plan. Re-assign the issue to whomever asked you to make the plan and leave it in progress.

## Linking to the Plan

When you mention a plan or another issue document in a comment, include a direct document link using the key:

- Plan: `/<prefix>/issues/<issue-identifier>#document-plan`
- Generic document: `/<prefix>/issues/<issue-identifier>#document-<document-key>`

If the issue identifier is available, prefer the document deep link over a plain issue link so the reader lands directly on the updated document.

## Recommended API Flow

```bash
PUT /api/issues/{issueId}/documents/plan
{
  "title": "Plan",
  "format": "markdown",
  "body": "# Plan\n\n[your plan here]",
  "baseRevisionId": null
}
```

If `plan` already exists, fetch the current document first and send its latest `baseRevisionId` when you update it (optimistic concurrency — Paperclip rejects stale writes).
