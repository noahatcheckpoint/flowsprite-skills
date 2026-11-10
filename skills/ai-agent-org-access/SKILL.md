---
name: ai-agent-org-access
description: "Use when an AI assistant or agent is about to connect to a Salesforce org through an MCP server, the Salesforce CLI or an API integration. Produces a written access plan: sandbox first, a dedicated least-privilege integration user, separate read and write scopes, production changes behind version control and human review, a list of forbidden actions, logging and a revocation procedure."
license: MIT
metadata:
  title: "AI Agent Org Access Plan"
  category: "ai-agents"
  author: "flowsprite"
  summary: "Plan safe Salesforce org access for an AI agent: sandbox first, a least-privilege integration user, reviewed production changes and a revoke path."
  tags: "ai agents, mcp, integration user, least privilege, salesforce security"
  version: "1.0"
  updated: "2026-11-10"
---
# AI Agent Org Access Plan

This skill produces a written plan for giving an AI agent access to a Salesforce org. An agent acts with exactly the permissions of the user it signs in as, at machine speed, and it can misread an instruction. The plan limits what it can reach, keeps production changes behind a human, and makes every action traceable and reversible.

## When to use

- Connecting an AI assistant to an org for the first time, through an MCP server, the Salesforce CLI or a custom integration.
- An admin has already connected an agent with their own login and wants to fix that.
- Moving an agent from a sandbox pilot to production.
- A security review, audit or customer questionnaire asks how AI tools access the org.

## What you need

Ask for these. If the user cannot answer one, write the plan with that item marked as an open risk.

1. **The agent and its connection method:** which assistant, and whether it uses an MCP server, the CLI, a connected app or another integration.
2. **The jobs it should do:** for example answer questions about records, draft Flows, write Apex, clean data. Be specific.
3. **Orgs in scope:** production, and which sandboxes exist (Developer, Developer Pro, Partial Copy, Full).
4. **Data sensitivity:** objects and fields holding personal, financial or health data, and any regulatory obligations.
5. **Current change process:** whether metadata lives in version control, who approves deployments, how releases reach production.
6. **Owner:** the named person accountable for the agent's access.

## Process

### 1. Start in a sandbox

Connect the agent to a Developer or Developer Pro sandbox first, where a mistake costs nothing. Partial Copy and Full sandboxes contain copies of production records, so treat their personal data as production data, or mask it before the agent connects. Move to production only after someone has reviewed the agent's sandbox work.

### 2. Create a dedicated integration user

Never connect an agent with a person's login, and never with an admin. Create one user per agent and per environment, named so it is obvious in logs (for example "AI Agent Integration, Sandbox"). Where your edition offers it, use the API-only Salesforce Integration user license with the Minimum Access, API Only Integrations profile; otherwise use a minimum-access profile and grant the API Only User permission so the account cannot log in to the UI. Use a team-monitored email address.

### 3. Grant least privilege with permission sets

Keep the profile empty and grant access through permission sets or a permission set group, one per job. Start from the jobs list: for each job, list the objects, fields and actions it needs and grant only those. Sharing still applies, so set the user's role and sharing rules to the records the agent should see.

Permissions the agent user should not hold:

| Permission | Why |
|---|---|
| Modify All Data, View All Data | Bypass sharing across the whole org |
| Customize Application, Modify Metadata Through Metadata API Functions | Change org configuration directly |
| Manage Users, Assign Permission Sets, Manage Profiles and Permission Sets | Lets the agent grant itself more access |
| Author Apex | Allows anonymous Apex, which runs arbitrary code |
| Bulk API Hard Delete | Deletes records without the recycle bin |
| Export Reports, Data Export | Moves data out of the org in bulk |

### 4. Split read and write

Give the agent a read-only permission set by default. Add write access per object only when a job needs it, and prefer a separate, narrower user for write work so read access can stay on while write access is switched off. If the connection uses OAuth, request the narrowest scopes the tool supports (usually `api` and `refresh_token`) and avoid `full`. In the connected app settings, set "Admin approved users are pre-authorized", restrict it to the integration user's profile or permission set, and set a refresh token expiry.

### 5. Keep production changes behind review

The agent may build and test changes in a sandbox. It does not deploy metadata to production on its own. The path to production is:

1. The agent commits its metadata changes to a branch in version control.
2. It opens a pull request describing what changed and why.
3. A human reviews the diff and approves the pull request.
4. A validate-only deployment runs against production with tests (for example `sf project deploy validate`), and the result is attached to the pull request.
5. A human starts the real deployment, either a quick deploy of the validated job or a release through the normal pipeline.

The production integration user should hold no metadata deploy permissions, so this path is enforced by access, not by the agent's good behavior.

### 6. Write down what the agent may never do

Put these in the plan and in the agent's instructions, and back each one with a missing permission where possible:

- Delete records in bulk, or hard delete anything.
- Change users, profiles, permission sets, roles or sharing settings.
- Export personal data from the org, or paste it into prompts sent to outside services beyond what the job requires.
- Run anonymous Apex in production.
- Deploy destructive changes.
- Deactivate Flows, validation rules or triggers in production.

### 7. Log and review what it ran

Decide where the record of the agent's actions lives. Sources include Setup Audit Trail for configuration changes, Login History for the integration user, Field History Tracking on sensitive fields, deployment history, and Event Monitoring if the org has it. Keep the agent tool's own log of every command or API call it ran. Name a reviewer and a schedule (for example weekly during the pilot) and a list of things that trigger an immediate review, such as a failed login spike or any write to a sensitive object.

### 8. Plan revocation before go-live

Write the exact steps and test them once in the sandbox:

1. Freeze the integration user (immediate) or deactivate it.
2. Revoke its OAuth tokens from the user record or the connected app's usage page.
3. Rotate the connected app's consumer secret if the tool stored it.
4. Remove the CLI or MCP server's stored org authorization on every machine that holds one (for example `sf org logout`).
5. Record who revoked access, when and why.

## Output format

```
AI Agent Org Access Plan: [agent name]
Owner: [name] · Review date: [date]

1. Scope: jobs, orgs, data classification
2. Environments: sandbox pilot plan, production go-live criteria
3. Integration users: name, license, profile, per environment
4. Permission sets: name, objects/fields, read or write, job served
5. OAuth and connection: scopes, connected app policies, token expiry
6. Production change path: branch, pull request, approver, validate-only deploy, who deploys
7. Forbidden actions: list, and the missing permission that enforces each
8. Logging and review: sources, reviewer, cadence, alert triggers
9. Revocation: steps, tested on [date]
Open risks: [items the user could not answer]
```

## Quality checklist

- No agent signs in as a person or as an admin.
- Each integration user holds only permission sets tied to a named job.
- No permission from the "should not hold" table is granted, or each exception has a written reason and an owner.
- Production write access, if any, is narrower than sandbox access.
- No path exists for the agent to deploy to production without a human approval.
- Revocation steps were tested, not just written.
- The plan names one accountable owner.

## Common mistakes

- **Borrowing the admin's login.** The agent inherits every permission, and the audit trail shows the admin doing things they did not do.
- **One user for every environment.** A sandbox credential that also works in production removes the sandbox's protection.
- **Instructions instead of permissions.** Telling the agent not to delete records is not a control; removing the permission is.
- **Forgetting copied data.** A Full sandbox holds production personal data.
- **No exit plan.** Teams that never tested revocation find stored tokens on laptops months after the pilot ended.

## Example prompts

- "We want Claude to read our Salesforce org through an MCP server. Write the access plan."
- "Our admin connected an AI assistant with their own login. How do we move it to a proper integration user?"
- "Which permissions should an AI agent's integration user never have? Give me a table for our security review."
- "Design the review process so an agent can build Flows in a sandbox but only humans deploy to production."
- "Write the revocation runbook for removing an AI agent's access to our org."
