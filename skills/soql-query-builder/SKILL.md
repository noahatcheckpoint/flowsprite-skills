---
name: soql-query-builder
description: "Use when turning a plain-English question about Salesforce data into a SOQL query. Picks the right objects and relationship names, writes selective filters and date literals, handles aggregates, nulls and polymorphic fields, keeps governor limits in mind, and explains what the result means and where it can mislead."
license: MIT
metadata:
  title: "SOQL Query Builder"
  category: "data"
  author: "flowsprite"
  summary: "Turn a plain-English data question into correct, selective SOQL, with the right relationships, filters and aggregates, plus the caveats of the result."
  tags: "salesforce, soql, data, reporting, queries"
  version: "1.0"
  updated: "2026-11-10"
---
# SOQL Query Builder

This skill turns a business question ("how many open opportunities closed late last quarter, by owner?") into a SOQL query that returns the right rows, runs fast on a large org, and reads no more than it needs. Every query comes with a short explanation of what it counts and what it leaves out.

## When to use

- Someone asks a data question and the answer lives in Salesforce records.
- A report cannot express the question, such as a filter on a child record or a cross-object count.
- An existing query is slow, times out or hits a non-selective query error.
- You need a query for Apex, a flow, an integration or a data export and want it correct the first time.

For changing data, use a data load or cleanup skill. SOQL only reads, and this skill keeps it that way.

## What you need

Ask for these before writing. If the user does not know the field names, write the query with the likely standard names and mark every guessed name as one to confirm.

1. **The question in plain words**, including the time window and the unit being counted (records, accounts, amount).
2. **Objects and fields involved**, with API names if known. Custom objects and fields end in `__c`.
3. **Org specifics:** record types, custom fiscal year, multi-currency, and any field that means something special in this org.
4. **Where the query runs:** the developer console, a CLI, Apex, a flow or an integration. Limits and security behave differently in each.
5. **Rough data volume** of the main object, so you know whether selectivity matters.

## Process

### 1. Restate the question as a query plan

Write one sentence each for the object you are counting, the filters, the grouping and the fields to show. Ambiguity hides here. "Customers" might mean Accounts with a closed-won opportunity or Accounts with Type set to Customer. Pick one, state it, and offer the other.

### 2. Choose the object and relationship names

- **Child to parent:** use the relationship name with dot notation. Standard lookups drop `Id` (`Account.Name` on Contact). Custom lookups swap `__c` for `__r` (`Region__r.Name`). Parent traversal goes up to five levels.
- **Parent to child:** use a subquery on the child relationship name, which is usually plural (`Contacts`, `Opportunities`, `Line_Items__r`). Check the exact name on the lookup field definition, because it is set by whoever created the field.
- **Filtering on related records:** use a semi-join (`WHERE Id IN (SELECT AccountId FROM Opportunity WHERE ...)`) or an anti-join with `NOT IN`. It is clearer and cheaper than pulling child rows and filtering them yourself.

```sql
SELECT Id, Name, Owner.Name,
       (SELECT Id, Name, StageName, Amount
        FROM Opportunities
        WHERE IsClosed = false)
FROM Account
WHERE Id IN (SELECT AccountId FROM Opportunity
             WHERE IsClosed = false AND Amount > 50000)
```

### 3. Write selective filters

On large objects, the query optimizer needs at least one filter it can use an index for. Fields that are indexed by default include `Id`, `Name`, `OwnerId`, `CreatedDate`, `SystemModstamp`, `RecordTypeId`, lookup and master-detail fields, and fields marked External ID or Unique.

- Lead with an indexed filter, then narrow with the rest.
- Avoid filters the optimizer cannot use an index for: `!=`, `NOT IN`, `NOT LIKE`, a `LIKE` with a leading wildcard, and formula fields that are not indexed.
- Use `SystemModstamp` instead of `LastModifiedDate` for "changed recently" filters on big objects, since `SystemModstamp` is indexed.
- When a query is still slow, check it with the Query Plan tool in the developer console before guessing.

### 4. Use date literals for time windows

Date literals are clearer than hard-coded dates and follow the org's settings.

| Need | Literal |
|---|---|
| Rolling window that includes today | `LAST_N_DAYS:30` |
| Current calendar quarter | `THIS_QUARTER` |
| Previous calendar quarter | `LAST_QUARTER` |
| Current fiscal quarter | `THIS_FISCAL_QUARTER` |
| The next n days, starting tomorrow | `NEXT_N_DAYS:7` |

Fiscal literals follow the org's fiscal year setup, calendar literals do not. Check which one the asker means. Literals are evaluated in the running user's time zone, while datetime values written out by hand must be in UTC, such as `2026-01-01T00:00:00Z`. Date fields like `CloseDate` take plain dates; datetime fields like `CreatedDate` take literals or full datetimes.

### 5. Aggregate when the question is a number

Use `COUNT()`, `COUNT(field)`, `COUNT_DISTINCT(field)`, `SUM`, `AVG`, `MIN` and `MAX` with `GROUP BY`. Filter groups with `HAVING`, not `WHERE`. Give aggregates an alias so the result is readable.

```sql
SELECT Owner.Name owner, COUNT(Id) deals, SUM(Amount) total
FROM Opportunity
WHERE IsClosed = true AND IsWon = true AND CloseDate = LAST_QUARTER
GROUP BY Owner.Name
HAVING COUNT(Id) > 2
ORDER BY SUM(Amount) DESC
```

In a multi-currency org, `SUM(Amount)` adds raw values across currencies. Use `convertCurrency(Amount)` or group by `CurrencyIsoCode`.

### 6. Handle nulls on purpose

- `Field = null` and `Field != null` are the only null tests.
- Unlike SQL, `!=` and `NOT IN` return rows where the field is null. Add `AND Field != null` when the asker means "has a different value".
- `COUNT(field)`, `SUM` and `AVG` skip nulls; `COUNT()` and `COUNT(Id)` do not. Say which one the number uses.
- `GROUP BY` puts null values in their own group. Label it instead of hiding it.
- Checkbox fields are never null, only true or false.

### 7. Treat polymorphic fields carefully

`WhoId` on Task and Event points to a Lead or a Contact. `WhatId` points to many objects. `OwnerId` on many objects can be a User or a Queue. Filter by type with `Who.Type = 'Contact'` or `What.Type = 'Account'`. Only fields shared by every possible target, such as `Name`, can be read directly through the relationship. For type-specific fields, use a `TYPEOF` clause or run a second query on the target object.

### 8. Respect governor limits

In synchronous Apex, a transaction allows 100 SOQL queries and 50,000 returned rows in total. Never put a query inside a loop. Query once with `IN :idSet` and map the results. In Apex, rows scanned by an aggregate query can count toward the row limit too. Run user-facing queries with `WITH USER_MODE` so field-level security and sharing apply.

### 9. Read only what you need

- List the fields the answer uses. Do not use `FIELDS(ALL)`; it also requires a `LIMIT` of 200 or fewer.
- Add `LIMIT` when sampling or previewing.
- Leave out `FOR UPDATE`, `FOR VIEW` and `FOR REFERENCE` unless you mean them. The first locks records; the other two update recently viewed and referenced tracking, so they change org state.

## Output format

```
Question: [restated in one sentence]
Assumptions: [definitions chosen, guessed field names to confirm]

Query:
[SOQL]

What it returns: [one row per ..., columns]
Selectivity: [indexed filter used, or the risk]
Caveats: [nulls, currency, fiscal vs calendar, sharing, records excluded]
Check it: [a quick sanity query, such as a total count to compare against]
```

## Quality checklist

- The object being counted matches the unit in the question.
- Every relationship name is confirmed or flagged as a guess.
- At least one filter on a large object uses an indexed field.
- The time window uses the right calendar or fiscal literal.
- Null behavior is stated wherever a filter or aggregate is affected.
- Multi-currency sums are converted or grouped by currency.
- Only the needed fields are selected, with no locking or tracking clauses.
- The explanation names what the query leaves out.

## Common mistakes

- **Counting the wrong thing.** Counting opportunities when the question was about accounts with opportunities. A semi-join or `COUNT_DISTINCT(AccountId)` fixes it.
- **Guessing child relationship names.** `Invoice__cs` and `Invoices__r` look plausible and fail. Look it up.
- **Calendar versus fiscal.** `THIS_QUARTER` and `THIS_FISCAL_QUARTER` differ in any org with a non-January fiscal year.
- **Forgetting that `!=` includes nulls.** The "not in Retail" count quietly includes accounts with no industry.
- **Querying in a loop.** Works in a sandbox with ten records, fails in production.
- **Ignoring sharing.** A query run as an admin sees more than the person who asked will see in a report.

## Example prompts

- "Write a SOQL query for all open opportunities over 50k whose close date already passed, with the account name and owner."
- "How many leads did each source create last quarter? Our fiscal year starts in April."
- "Give me accounts that have no activity in the last 90 days but do have an open opportunity."
- "This query times out on our Case object. Make it selective."
- "List the tasks logged against contacts at accounts in Germany this month."
