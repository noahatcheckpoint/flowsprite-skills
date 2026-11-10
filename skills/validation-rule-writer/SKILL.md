---
name: validation-rule-writer
description: "Use when writing or fixing a Salesforce validation rule. Turns a business requirement into a working error condition formula with a clear error message and location, a custom permission bypass for integrations and data loads, correct use of ISCHANGED, ISNEW, PRIORVALUE and picklist functions, safe null handling and a set of test cases that prove the rule."
license: MIT
metadata:
  title: "Validation Rule Writer"
  category: "admin-config"
  author: "flowsprite"
  summary: "Write a Salesforce validation rule that blocks bad data, explains itself to users, lets integrations through and comes with tests."
  tags: "salesforce, validation rules, formulas, data quality, admin"
  version: "1.0"
  updated: "2026-11-10"
---
# Validation Rule Writer

This skill turns a plain business requirement ("a lost deal must have a loss reason") into a validation rule that is correct on the first deploy. It writes the error condition formula, an error message that tells the user what to do, a bypass for automated loads, and the test cases that prove the rule fires when it should and stays quiet when it should not.

## When to use

- A team wants a field required only in certain stages, record types or statuses.
- Users keep saving records that break reporting or downstream processes.
- An existing rule blocks a data load, an integration or a flow, and needs a bypass.
- A rule fires on records it should ignore, or never fires at all.
- You need to lock a field once a record reaches a final state.

## What you need

Ask for these before writing. If something is missing, write the rule anyway and list the assumption you made.

1. **Object and requirement:** the object and the rule in business terms, including the exception cases.
2. **Fields involved:** API names and field types, especially which are picklists, multi-select picklists, checkboxes, lookups or numbers.
3. **When it applies:** on create, on edit, or only when a specific field changes.
4. **Who is exempt:** integration users, data migration users, admins, or nobody.
5. **Record types:** whether the rule applies to all record types or only some, by developer name.
6. **Existing automation:** flows or Apex that set the same fields, since they run before validation rules and can satisfy or trip them.

## Process

### 1. Write the condition in plain words first

A validation rule formula describes the **bad** state. When it returns TRUE, the save is blocked. Write one sentence that starts with "Block the save when", then translate it. "Block the save when the stage is Closed Lost and the loss reason is blank" maps directly to an `AND()` of two checks.

### 2. Start every rule with the bypass

Create a custom permission (for example `Bypass_Validation_Rules`) and assign it through a permission set to integration and migration users. Put the check first in the formula:

```
AND(
  NOT($Permission.Bypass_Validation_Rules),
  ISPICKVAL(StageName, "Closed Lost"),
  ISBLANK(TEXT(Loss_Reason__c))
)
```

A custom permission beats checking a profile name or user ID: it survives sandbox refreshes, needs no hard-coded IDs, and can be granted temporarily for a single load. Some teams add a second, object-specific permission when only some rules should be bypassed.

### 3. Handle picklists correctly

- Use `ISPICKVAL(Field, "Value")` to test a single picklist value. A direct comparison such as `StageName = "Closed Lost"` does not compile.
- Use `TEXT(Field)` when you need the value as text, for example inside `CASE()` or to test for blank: `ISBLANK(TEXT(Loss_Reason__c))`.
- Use the picklist **API value**, not the label. Labels can be translated or renamed.
- Multi-select picklists need `INCLUDES(Field, "Value")`.

### 4. Handle nulls deliberately

- Use `ISBLANK()`, not `ISNULL()`. `ISNULL()` does not treat an empty text field as null, so a rule built on it never fires for text.
- Checkboxes are never blank. Test them as TRUE or FALSE directly.
- Lookups are tested with `ISBLANK(AccountId)`.
- Before referencing a field through a lookup (such as `Account.Industry`), make sure the rule also handles the lookup being empty.

### 5. Use change functions for edit-only logic

- `ISNEW()` is TRUE only on insert. Wrap edit-only logic in `NOT(ISNEW())`.
- `ISCHANGED(Field)` is TRUE when the field's value differs from its value before the edit. It is always FALSE on insert.
- `PRIORVALUE(Field)` returns the value before the edit. For picklists, wrap it: `ISPICKVAL(PRIORVALUE(StageName), "Closed Won")`.

Example, lock the close date once a deal is won:

```
AND(
  NOT($Permission.Bypass_Validation_Rules),
  NOT(ISNEW()),
  ISPICKVAL(PRIORVALUE(StageName), "Closed Won"),
  ISCHANGED(CloseDate)
)
```

### 6. Avoid hard-coded IDs

Reference record types as `RecordType.DeveloperName = "Enterprise"`, never by ID. Use `$Profile.Name`, `$UserRole.Name` or custom permissions instead of user or profile IDs. Values that change often belong in custom metadata or a custom setting the formula can read.

### 7. Write the error message and choose the location

- Say what is wrong and how to fix it: "Select a Loss Reason before closing this opportunity as lost."
- Use field labels the user sees, not API names, and no internal jargon.
- Place the error on the field when one field fixes the problem. Use the top of the page when the fix involves several fields, or when the field may not be on every page layout.
- Name the rule so it describes the block (for example `Opp_Require_Loss_Reason_When_Lost`) and fill in the description with the business owner and the reason.

### 8. Check the side effects

- Validation rules run on every save path: the UI, the API, data loader, flows and Apex. Any automation that updates the object can now fail.
- Before-save flows and Apex before triggers run first, so a field they populate will satisfy the rule.
- Existing records that already break the rule will fail the next time anyone edits them for any reason. Report how many exist before activating.

### 9. Write the test cases

Cover each branch of the formula, plus the bypass:

- A record that should be blocked, with the exact error expected.
- A record that should save, one per exception in the requirement.
- An insert and an edit, when the rule uses change functions.
- A blank value for every field the formula reads.
- The same blocked case run as a user with the bypass permission, which must save.
- A bulk load in a sandbox, to confirm integrations behave as intended.

## Output format

```
Rule name: [Object_Purpose]
Object: [object]   Active: [yes/no]
Description: [business reason, owner]

Error condition formula:
[formula]

Error message: [text]
Error location: [field name or top of page]

Bypass: [custom permission and who gets it]
Assumptions: [field types, API values, exemptions assumed]

Test cases:
| # | Scenario | Field values | User | Expected |

Impact: [automation and integrations affected, existing records that already fail]
```

## Quality checklist

- The formula returns TRUE only for the bad state, and the plain-words sentence matches it.
- The bypass check is the first condition.
- Every picklist uses `ISPICKVAL`, `TEXT` or `INCLUDES`, with API values.
- No `ISNULL()` on text fields and no hard-coded record, profile or user IDs.
- Edit-only logic is wrapped in `NOT(ISNEW())`.
- The error message tells the user what to do and uses visible field labels.
- The test cases cover the bypass and blank values.
- The impact on existing records and automation is stated.

## Common mistakes

- **Writing the formula for the good state.** The rule then blocks every valid save. Re-read the formula as "block when".
- **Bypassing by profile name.** Profiles get renamed and cloned. A custom permission is explicit and auditable.
- **Forgetting existing data.** A rule that many current records already violate turns every routine edit into an error.
- **Vague messages.** "Invalid data" sends users to the admin. Name the field and the fix.
- **One rule doing five jobs.** Split unrelated checks into separate rules so each error message stays specific.

## Example prompts

- "Write a validation rule that requires a Loss Reason when an Opportunity is set to Closed Lost."
- "Our integration user keeps hitting validation errors during the nightly sync. Add a bypass to these three rules."
- "Lock the Amount field once the opportunity is Closed Won, but let finance change it."
- "This rule never fires on the Description field. Here is the formula, what is wrong?"
- "Require a Billing Country on Accounts of the Enterprise record type, only on edit."
