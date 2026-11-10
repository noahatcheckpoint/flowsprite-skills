---
name: flow-review
description: "Use when reviewing a Salesforce Flow (record-triggered, schedule-triggered or screen) before it ships to production. Checks entry conditions, the before-save or after-save choice, bulkification, recursion and order of execution, fault handling, hard-coded IDs, run context and documentation, then returns a ranked list of findings with fixes and a test plan."
license: MIT
metadata:
  title: "Salesforce Flow Review"
  category: "flows-automation"
  author: "flowsprite"
  summary: "Review a Salesforce Flow before release: entry conditions, bulk safety, recursion, fault paths, hard-coded IDs and a test plan, ranked by risk."
  tags: "salesforce flow, flow builder, record-triggered flow, automation, code review"
  version: "1.0"
  updated: "2026-11-10"
---
# Salesforce Flow Review

This skill reviews a Flow the way a senior admin would before approving a release. It looks for the defects that only show up under real load or real data: a flow that fires on every edit, a query inside a loop, an update that retriggers itself, a failure that surfaces to users as an unreadable error. The output is a ranked list of findings, each with a concrete fix, plus the tests that prove the flow works.

## When to use

- A new or changed record-triggered flow is about to move from a sandbox to production.
- A flow works for single-record edits but fails during data loads or integrations.
- Users report "An unhandled fault has occurred in this flow" or unexpected field values.
- You are consolidating several flows on one object and need to check how they interact.

## What you need

Ask for these before reviewing. A review on a description alone is possible, but say clearly which checks could not be done.

1. **The flow itself:** the flow metadata file, a screenshot of the canvas, or a step-by-step description of every element in order.
2. **Trigger setup:** object, trigger (created, updated, created or updated, deleted), entry conditions, and whether it is optimized for fast field updates or actions and related records.
3. **Other automation on the same object:** other flows, Apex triggers, legacy workflow rules or processes, and validation rules.
4. **Expected volume:** how records arrive (user edits, data loader, an integration user, mass updates) and typical batch sizes.
5. **Intent:** one or two sentences on what the flow should do, and what it must never do.

## Process

### 1. Confirm the trigger and entry conditions

- Check that the flow runs only when it needs to. An update-triggered flow without entry conditions runs on every save of every record of that object, including saves made by other automation.
- For update logic, check the "only when a record is updated to meet the condition requirements" option. Without it, the flow repeats its work on every later edit of a record that already meets the criteria.
- Inside the flow, compare `$Record` with `$Record__Prior` when the logic depends on a field changing. Note that `$Record__Prior` is empty on create.
- Prefer entry conditions over a first-step Decision element, which still starts an interview for every record.

### 2. Check the before-save or after-save choice

- **Fast field updates (before-save)** is the right choice when the flow only sets fields on the triggering record. It changes `$Record` directly with an Assignment, needs no Update Records element, and does not cause another save.
- **Actions and related records (after-save)** is needed to create or update other records, send emails, call subflows or invocable actions, or use the record ID on create.
- Flag an after-save flow that only updates the triggering record. It costs an extra DML statement and reruns the save cycle, so it should move to before-save.
- Check whether slow or callout work belongs on an asynchronous path or a scheduled path instead of the main transaction.

### 3. Check bulkification

Record-triggered flows run in batches when records arrive together, and every batch shares one transaction's governor limits.

- No Get Records, Create Records, Update Records or Delete Records element sits inside a Loop. Inside a loop, use Assignment elements to build a collection, then run one data element after the loop.
- Get Records filters on indexed or selective fields where possible, and pulls only the fields the flow uses.
- Subflows and invocable Apex called per record are treated with the same suspicion: confirm the Apex is bulk-safe.

### 4. Check recursion and order of execution

- Before-save flows run before Apex before triggers; after-save flows run late in the save order, after Apex after triggers. List what else fires on the object and in which order.
- An after-save flow that updates its own triggering record starts a new save cycle, which runs before-save flows, triggers and validation rules again. Confirm the entry conditions stop the second pass from doing the work twice.
- Two flows that update each other's objects can loop. Trace each cross-object update to its own automation.
- When several flows share an object and trigger, check that the Trigger Order values are set deliberately, and use Flow Trigger Explorer to see the sequence.

### 5. Check fault handling

- Every data element and action has a fault connector. An unhandled fault rolls back the transaction and shows users a generic error.
- Fault paths do something useful: log to a custom object or platform event, notify an owner, or show a clear message on screen flows. A fault path that ends without recording the error hides the problem.
- Confirm who receives flow error emails in Process Automation Settings, and that someone reads them.

### 6. Check hard-coded values

- Flag any 15 or 18 character record ID in a filter, assignment or formula: record types, queues, users, profiles, pricebooks. IDs differ between sandbox and production, so the flow breaks on deploy or silently matches nothing.
- Replace them with a Get Records on `DeveloperName`, a custom metadata type, or a custom label.

### 7. Check run context and access

- Record-triggered flows run in system context, so they can read and change records the triggering user cannot see. Confirm that is intended, and that the flow does not copy restricted data onto records with wider sharing.
- For screen and autolaunched flows, check the "How to Run the Flow" setting, and whether running with or without sharing matches the intent.
- Screen flows that expose data or start sensitive actions should check the user's permissions, for example with a custom permission in a Decision.

### 8. Check naming and documentation

- The flow has a name that states the object, trigger and purpose, and a description of what it does and why.
- Elements and variables have descriptive API names, not `Decision_1` or `var1`. Each element has a description where the logic is not obvious.

### 9. Define the test plan

- Run Debug on the trigger with real sample records, and use rollback mode so the debug run does not commit data.
- Create Flow tests for record-triggered flows covering the main path, the entry condition not met, and each Decision outcome.
- Test a bulk load of a few hundred records with a data loader in a sandbox, including records that should and should not qualify.
- Test as a representative user and as the integration user.

## Output format

```
Flow: [API name], [object], [trigger], [before-save or after-save]
Verdict: Ready / Ready after fixes / Not ready

Findings (highest risk first)
1. [Severity: blocker, major, minor] [Area]
   What: [the defect, pointing to the element by name]
   Why it matters: [the failure it causes and when]
   Fix: [the specific change]

Interactions with other automation: [list with order]
Test plan: [numbered cases with expected results]
Not reviewed: [checks skipped and the missing input]
```

## Quality checklist

- Every finding names its element and gives a fix.
- Blockers are limited to defects that break data, exceed limits or fail on deploy.
- The before-save or after-save recommendation is stated explicitly.
- Every data element in a loop and every hard-coded ID is listed.
- Every data element and action without a fault connector is listed.
- The test plan includes a bulk case and a case where the entry conditions are not met.
- Checks that could not be run are listed under "Not reviewed".

## Common mistakes

- **Reviewing only the happy path.** Most flow incidents come from bulk loads, blank fields and records that already meet the criteria.
- **Fixing recursion with a checkbox field.** A "processed" flag adds a field and a write. Entry conditions and `$Record__Prior` comparisons usually solve it cleanly.
- **Assuming debug proves bulk safety.** Debug runs one interview. Only a multi-record load shows limit problems.
- **Accepting a fault path that emails a person and nothing else.** Errors need to be recorded somewhere searchable.

## Example prompts

- "Review this record-triggered flow on Opportunity before I deploy it. Here is the flow XML."
- "Our case assignment flow fails when the support team bulk-updates cases. What is wrong with it?"
- "Should this flow be before-save or after-save? It sets the region and creates a follow-up task."
- "We have three flows and an Apex trigger on Account. Check how they interact and whether any can loop."
- "Write a test plan for this schedule-triggered flow that closes stale leads every night."
