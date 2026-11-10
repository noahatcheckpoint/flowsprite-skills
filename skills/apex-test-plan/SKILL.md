---
name: apex-test-plan
description: "Use when planning or writing Apex unit tests for a trigger, class, batch job or callout, or when a deployment is blocked on code coverage. Builds a test plan that proves behavior with isolated data, bulk and negative cases, permission checks and mocked callouts, then maps the remaining coverage gaps."
license: MIT
metadata:
  title: "Apex Test Plan"
  category: "apex-code"
  author: "flowsprite"
  summary: "Plan and write Apex tests that prove behavior with isolated data, bulk and negative cases, runAs checks and mocked callouts, not just coverage."
  tags: "apex, unit testing, code coverage, test data, deployment"
  version: "1.0"
  updated: "2026-11-10"
---
# Apex Test Plan

This skill turns a piece of Apex into a test plan and then into test classes that would fail if the code broke. Coverage is a side effect of good tests, not the goal. A test that runs every line and asserts nothing passes the deploy gate and proves nothing.

## When to use

- Writing tests for a new trigger, handler, service class, batch, queueable or scheduled job.
- A production deployment fails on coverage or on a failing test.
- Reviewing an existing test class that has high coverage but few assertions.
- Adding a callout to an external system and needing a repeatable test for it.

## What you need

Ask for these before planning. If the code is missing, plan from the description and mark every assumption.

1. **The code under test:** the class or trigger source, plus any handler or helper it calls.
2. **The intended behavior:** what should happen for a normal record, an edge case and a bad input, in plain language.
3. **Entry points:** trigger events (before insert, after update and so on), invocable methods, controllers, REST resources, async jobs.
4. **Sharing and security model:** `with sharing` or `without sharing`, and whether the code runs queries or DML in user mode.
5. **External dependencies:** callouts, platform events, custom metadata, custom settings.
6. **Current coverage report:** per class, if one exists, and the test level the deployment will use.

## Process

### 1. Know the deployment gate

Salesforce requires that, for a production deployment, at least 75% of Apex code is covered by unit tests, every trigger has some coverage, and all tests pass. Test classes and test methods themselves do not count toward the total. When a deployment runs specified tests rather than all local tests, each class and trigger in the deployment must reach 75% on its own. Treat 75% as the floor the platform enforces, and set your own target per class from the behavior list, not from the number.

### 2. List behaviors before writing code

Write one line per behavior: "When an Opportunity closes as won, a renewal Opportunity is created with the same Account." Each behavior becomes at least one test method. Group them into four columns: happy path, bulk, negative, permission. A branch with no behavior is dead code or a missing requirement; ask which.

### 3. Build isolated test data

- Use `@isTest` on the class and never `SeeAllData=true`. By default tests cannot see org records (users, profiles and some setup objects are the exception), and tests that depend on live data break when someone edits a record.
- Put shared records in a `@TestSetup` method. It runs once per class, and each test method starts from a fresh copy of that data, so one test's changes do not leak into the next.
- Create records through a test data factory class with methods like `createAccounts(Integer count, Boolean doInsert)`. Required fields and validation rule changes then get fixed in one place.
- For price books use `Test.getStandardPricebookId()` instead of querying for the standard price book.
- Query records back inside each test method instead of holding them in static variables.

### 4. Control limits and async with startTest and stopTest

Wrap only the action under test in `Test.startTest()` and `Test.stopTest()`. Code between them gets a fresh set of governor limits, so setup DML does not eat into the budget you are measuring. Queueable, future, batch and scheduled work enqueued inside the block runs when `Test.stopTest()` is called, so assert on async results after it. Each test method can call the pair only once.

### 5. Test in bulk

Insert or update 200 records in one DML statement for every trigger and handler. This catches SOQL and DML inside loops. Add a mixed batch where some records qualify and some do not, and assert each record's outcome individually.

### 6. Test the negative paths

- Invalid input should raise the expected error. Use a try/catch, call `Assert.fail('Expected a DmlException')` after the statement that should throw, and assert on the exception message or type in the catch block.
- Records that should be ignored are left unchanged.
- Validation rules or `addError` calls block the save with the right message.

### 7. Test permissions with System.runAs

`System.runAs(user)` enforces the target user's record sharing. It does not enforce object or field permissions on its own; those apply when the code runs queries or DML in user mode (for example `WITH USER_MODE` or `AccessLevel.USER_MODE`) or strips fields with `Security.stripInaccessible`. Create a test user with a minimal profile, assign the permission set the real user would have, and assert both that a permitted user succeeds and that a user without access is blocked or sees fewer records. `runAs` also avoids mixed DML errors when a test inserts setup and non-setup objects.

### 8. Assert with messages

Use the `Assert` class: `Assert.areEqual(expected, actual, message)`, `Assert.isTrue`, `Assert.isNotNull`, `Assert.fail`. Every assertion carries a message that names the behavior, so a failed deploy log says what broke. Assert on the data the user cares about (field values, related records, counts), not on the fact that no exception occurred.

### 9. Mock callouts

Unit tests cannot make real HTTP callouts. Implement `HttpCalloutMock`, return a crafted `HttpResponse` from `respond(HttpRequest req)`, and register it with `Test.setMock(HttpCalloutMock.class, new MyMock())` before the callout runs. Write one mock per scenario: success, an error status code, a malformed body. Inside the mock, assert on the request too (endpoint, method, headers) so the test checks what your code sends. If the test inserts data first, do the DML before `Test.startTest()` and the callout after it to avoid the uncommitted work error.

```apex
@isTest
static void syncMarksAccountOnSuccess() {
    Account acct = [SELECT Id FROM Account LIMIT 1];
    Test.setMock(HttpCalloutMock.class, new ErpMock(200, '{"status":"ok"}'));
    Test.startTest();
    ErpSync.syncAccount(acct.Id);
    Test.stopTest();
    acct = [SELECT Sync_Status__c FROM Account WHERE Id = :acct.Id];
    Assert.areEqual('Synced', acct.Sync_Status__c, 'A 200 response should mark the account as synced');
}
```

### 10. Map the gaps

Run the tests with coverage (for example `sf apex run test --test-level RunLocalTests --code-coverage --result-format human`). For every uncovered block, decide: add a test, delete the code, or accept it with a reason.

## Output format

```
Apex Test Plan: [class or trigger name]

Behaviors
| # | Behavior | Type (happy/bulk/negative/permission) | Test method |

Test data
- Factory methods needed: ...
- @TestSetup records: ...

Mocks
- [Mock class]: scenarios covered, request assertions

Coverage gap table
| Class/trigger | Current % | Uncovered lines or branch | Action (test / delete / accept) | Reason |

Test class code
[full test class, then factory and mock classes]

Open questions
- [behavior or requirement that needs a decision]
```

## Quality checklist

- Every test method asserts at least one business outcome, with a message.
- No `SeeAllData=true`, no hardcoded record Ids, no queries for live org data.
- Each trigger has a 200-record bulk test.
- At least one negative test per entry point.
- Permission tests exist wherever the code enforces sharing or runs in user mode.
- Async work is asserted after `Test.stopTest()`.
- Every callout path has a mock, including an error response.
- The gap table explains every uncovered block that stays uncovered.

## Common mistakes

- **Coverage farming.** Calling methods without asserting anything hits 75% and catches no regressions.
- **Branching on `Test.isRunningTest()`.** Production code that behaves differently under test means the test proves the wrong path.
- **One giant test method.** A single method that tests ten behaviors hides which one failed and makes the gap table useless.
- **Testing with one record.** Code that passes with one record and fails at 200 is the most common trigger defect in production.
- **Assuming runAs checks field access.** It enforces sharing; object and field permissions need user mode or explicit checks in the code.
- **Brittle data.** Tests that insert records missing a field a new validation rule requires break every deployment. The factory fixes that once.

## Example prompts

- "Write a test class for this Opportunity trigger handler, including a 200-record bulk test."
- "Our deployment fails at 71% coverage. Here is the coverage report; tell me which tests to write first."
- "This class calls an external REST API. Build an HttpCalloutMock with success and error cases and the tests that use it."
- "Review this test class: it has 95% coverage but I don't trust it."
- "Plan tests for a queueable that updates contacts when an account owner changes, including a user who lacks access."
