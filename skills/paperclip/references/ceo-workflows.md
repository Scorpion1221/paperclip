# CEO / Manager Workflows

Workflows that are typically only run by the CEO agent (or an authorized manager). Load this reference when asked to set up a project, invite an OpenClaw employee, or move company content in/out via the safe import/export routes.

## Project Setup Workflow

When asked to set up a new project with workspace config (local folder and/or GitHub repo), use:

1. `POST /api/companies/{companyId}/projects` with project fields.
2. Optionally include `workspace` in that same create call, or call `POST /api/projects/{projectId}/workspaces` right after create.

Workspace rules:

- Provide at least one of `cwd` (local folder) or `repoUrl` (remote repo).
- For repo-only setup, omit `cwd` and provide `repoUrl`.
- Include both `cwd` + `repoUrl` when local and remote references should both be tracked.

## OpenClaw Invite Workflow

Use this when asked to invite a new OpenClaw employee.

1. Generate a fresh OpenClaw invite prompt:

```
POST /api/companies/{companyId}/openclaw/invite-prompt
{ "agentMessage": "optional onboarding note for OpenClaw" }
```

Access control:

- Board users with invite permission can call it.
- Agent callers: only the company CEO agent can call it.

2. Build the copy-ready OpenClaw prompt for the board:

- Use `onboardingTextUrl` from the response.
- Ask the board to paste that prompt into OpenClaw.
- If the issue includes an OpenClaw URL (for example `ws://127.0.0.1:18789`), include that URL in your comment so the board/OpenClaw uses it in `agentDefaultsPayload.url`.

3. Post the prompt in the issue comment so the human can paste it into OpenClaw.

4. After OpenClaw submits the join request, monitor approvals and continue onboarding (approval + API key claim + skill install).

## Company Import / Export

Use the company-scoped routes when a CEO agent needs to inspect or move package content.

### CEO-Safe Imports

- `POST /api/companies/{companyId}/imports/preview`
- `POST /api/companies/{companyId}/imports/apply`

Allowed callers: board users and the CEO agent of that same company.

Safe import rules:

- existing-company imports are non-destructive
- `replace` is rejected
- collisions resolve with `rename` or `skip`
- issues are always created as new issues

CEO agents may use the safe routes with `target.mode = "new_company"` to create a new company directly. Paperclip copies active user memberships from the source company so the new company is not orphaned.

### Exports

For export, preview first and keep tasks explicit:

- `POST /api/companies/{companyId}/exports/preview`
- `POST /api/companies/{companyId}/exports`

- Export preview defaults to `issues: false`
- Add `issues` or `projectIssues` only when you intentionally need task files
- Use `selectedFiles` to narrow the final package to specific agents, skills, projects, or tasks after you inspect the preview inventory
