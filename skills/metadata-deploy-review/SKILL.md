---
name: metadata-deploy-review
description: "Use when reviewing a Salesforce metadata change (a pull request, change set or package.xml) before it reaches production. Checks what changed and why, dependency gaps, destructive changes, profile and permission set side effects, test level, deploy order and rollback, then returns a go or no-go call with the reasons."
license: MIT
metadata:
  title: "Metadata Deployment Review"
  category: "devops-release"
  author: "flowsprite"
  summary: "Review a Salesforce metadata change before production: dependencies, destructive changes, access side effects, tests, deploy order, rollback and a go/no-go."
  tags: "salesforce, metadata api, deployment, release management, change review"
  version: "1.0"
  updated: "2026-11-10"
---
# Metadata Deployment Review

This skill reviews a Salesforce metadata change the way a careful release manager would before it touches production. It reads the change, finds what will break or surprise people, picks a test strategy, sets the deploy order, writes down how to back out, and ends with a clear go or no-go.

## When to use

- A pull request with Salesforce metadata is about to merge into the branch that deploys to production.
- A change set or package.xml is ready to deploy and nobody has read it line by line.
- A validation failed in production and you need to know whether the fix is safe.
- A release bundles work from several people and you need one view of the risk.

For building the change itself, or for data loads, use a different skill. This one reviews; it does not author.

## What you need

Ask for these before reviewing. If something is missing, review anyway and list the gap in the output.

1. **The change:** the diff, the package.xml, or the list of components in the change set. File contents beat a list of names.
2. **Why it exists:** the ticket or request, so you can check the change does what was asked and nothing more.
3. **Target org facts:** whether the target is production, which Apex tests exist, and whether the components already exist there.
4. **Destructive manifests:** any destructiveChanges.xml, destructiveChangesPre.xml or destructiveChangesPost.xml.
5. **Release window:** when it ships, who is on call, and whether users will be working during the deploy.

## Process

### 1. Summarize what changed and why

List every component by type: custom objects and fields, flows, Apex classes and triggers, layouts, Lightning pages, profiles, permission sets, validation rules, record types. Mark each as new, modified or deleted. Then compare the list with the stated reason. A ticket about one field that also edits three unrelated flows needs an explanation before it ships.

### 2. Check dependencies

Everything a component references must exist in the target org or travel in the same deploy. Look hardest at:

- Fields referenced by flows, formulas, validation rules, layouts, Lightning pages, reports and Apex.
- Picklist values and record types that a flow decision or an assignment depends on.
- Permission sets and profiles that grant access to fields or objects not yet in the target.
- Apex that calls a class, custom metadata type or custom label missing from the package.
- A flow that calls a subflow or an invocable Apex action.

A validate-only deploy catches most missing references. It will not catch logic that silently depends on data, such as a flow that expects a record type to have a specific developer name.

### 3. Review destructive changes

Deleting a field deletes its data for every record. Confirm for each deletion:

- Nothing still references it. Remove references in the same release and delete afterwards, using a post-destructive manifest so the references go first.
- The data is backed up or genuinely unwanted.
- Reports, integrations and external systems that read the field have been told.

Deleted custom fields sit in the recycle bin for a limited time and can be restored there, but treat every production deletion as permanent in your plan.

### 4. Check access side effects

Profiles and permission sets are where deploys change more than the diff suggests.

- **New fields without access.** A field deployed with no permission set or profile entry is usually invisible to users. Confirm who should see and edit it.
- **Profile files.** A retrieved profile only contains the permissions for components that were in the retrieve, and deploying it can grant or remove access the author never meant to touch. Read the actual profile diff. Prefer permission sets for new access.
- **Broad grants.** Flag any new View All, Modify All, View All Data, Modify All Data or Author Apex permission, and any object permission upgraded to delete.
- **Sharing changes.** Org-wide default changes trigger sharing recalculation and can affect every user.

### 5. Choose the test level

| Test level | When it fits | Coverage rule |
|---|---|---|
| `RunSpecifiedTests` | Small, well-understood Apex changes with targeted tests | Each class and trigger in the deploy needs 75% coverage from the named tests |
| `RunLocalTests` | Larger releases, or when you cannot name the right tests with confidence | 75% coverage across the org's non-managed code |
| `RunAllTestsInOrg` | Rarely; when installed package tests matter too | Same as local, plus managed package tests run |
| `NoTestRun` | Changes with no Apex, such as layouts or permission sets | Not allowed in production when the deploy contains Apex |

In production, a deploy that contains Apex runs local tests by default if no level is given. Name the tests explicitly with `RunSpecifiedTests` when you can, because it is faster, but check that the named tests actually exercise the changed code.

### 6. Validate first, then quick deploy

Run a validate-only deploy against production with the chosen test level. It runs the tests and checks every reference without saving anything. If it succeeds, Salesforce keeps the validation eligible for quick deploy for 10 days, so the real deploy skips the test run.

```bash
sf project deploy validate --manifest package.xml --target-org prod \
  --test-level RunSpecifiedTests --tests AccountServiceTest --tests LeadRoutingTest
sf project deploy quick --job-id VALIDATION_JOB_ID --target-org prod
```

If other changes land in production between validation and quick deploy, revalidate.

### 7. Set the deploy order

A single deploy is applied as a unit in production: if any component fails, nothing is saved. When the release needs several deploys, order them:

1. Objects, fields, record types and picklist values.
2. Apex, custom metadata and labels.
3. Flows, validation rules and Lightning pages that use them.
4. Permission sets and profile changes.
5. Destructive changes, after every reference is gone.

Check the org's setting for whether flows deploy as active. Plan who activates them and when.

### 8. Write the rollback plan

Metadata deploys have no undo button. Before deploying, retrieve the current production version of every component in the package and keep it with the release. Rolling back means deploying that snapshot. Note what the snapshot cannot restore: deleted field data, records changed by a new flow or trigger while it ran, and emails already sent. For flows, the previous version can be reactivated directly.

## Output format

```
Release: [name or ticket]
Verdict: GO / GO WITH CONDITIONS / NO-GO
Reason: [one sentence]

What changed
- [type] [API name]: new / modified / deleted, [one-line purpose]

Blocking issues
- [issue, component, fix]

Warnings
- [issue, component, recommended action]

Destructive changes: [list, references removed, backup status]
Access changes: [permission sets and profiles, net effect]
Test level: [level, named tests, why]
Deploy order: [numbered steps]
Rollback: [snapshot location, what cannot be restored]
Open questions: [missing inputs]
```

## Quality checklist

- Every component in the change is listed and matched to the stated reason.
- Every field, record type and picklist value referenced by the change exists in the target or ships with it.
- Each deletion has a reference check, a backup decision and a post-destructive step.
- The profile diff was read, not assumed.
- The test level is named with a reason, and the named tests cover the changed Apex.
- A validate-only deploy ran against the real target before the verdict.
- The rollback snapshot exists before the deploy starts.
- The verdict names its blocking issues, or says there are none.

## Common mistakes

- **Reviewing names, not contents.** A package.xml says a flow changed. Only the flow file says it now updates every open opportunity.
- **Trusting a sandbox success.** Production has different data, users and installed packages. Validate against production itself.
- **Deploying a whole profile.** It drags in access changes nobody reviewed.
- **Deleting and removing references in the wrong order.** The deletion fails, or worse, succeeds while an integration still reads the field.
- **Naming tests that do not cover the change.** The coverage rule fails late, or passes while the new logic stays untested.
- **No snapshot.** Without the previous versions, rollback becomes a rebuild under pressure.

## Example prompts

- "Review this pull request before we deploy it to production. It adds two fields to Opportunity and updates the stage flow."
- "Here is our package.xml and destructiveChangesPost.xml. What could go wrong, and in what order should we deploy?"
- "Which test level should I use for this release? It changes one Apex trigger and three permission sets."
- "Give me a go or no-go on this change set and a rollback plan."
- "Our validation failed with a missing field error. Read the error and the diff and tell me what is missing."
