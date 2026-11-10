---
name: field-usage-cleanup
description: "Use when finding custom fields in a Salesforce org that nobody fills in or reads, and retiring them safely. Measures fill rates with SOQL, checks dependencies in automation, code, reports and integrations, then runs a staged deprecation before anything is deleted."
license: MIT
metadata:
  title: "Unused Field Cleanup"
  category: "data"
  author: "flowsprite"
  summary: "Find custom fields nobody uses, prove they are safe to remove, and retire them in stages without breaking automation or integrations."
  tags: "salesforce, custom fields, technical debt, field usage, data cleanup"
  version: "1.0"
  updated: "2026-11-10"
---
# Unused Field Cleanup

Orgs collect custom fields the way garages collect boxes. This skill finds the fields nobody fills in or reads, proves each one is safe to retire, and removes it in stages so that a forgotten integration or report surfaces as a warning instead of an outage. The output is a decision for every candidate field and a dated retirement plan.

## When to use

- An object has so many fields that page layouts, reports and the field picker are hard to use.
- The org is approaching its custom field limit on a core object.
- Before a migration, a data model redesign or a new integration, so dead fields are not carried forward.
- After a project was cancelled and its fields were left behind.

## What you need

Ask for these. Without dependency access, produce the fill-rate list and mark every field "needs dependency check".

1. **Objects in scope:** start with one or two (Account, Opportunity, Lead or a heavily used custom object).
2. **Query access:** SOQL against the data, plus Setup or Tooling API access for field metadata.
3. **Integration inventory:** every system that reads or writes the org (middleware, ETL, data warehouse, marketing and billing tools) and who owns each.
4. **Field owners:** the admin or team that requested each field, if known.
5. **A sandbox** that mirrors production metadata, for testing removals.

## Process

### 1. Build the candidate list

Export the custom fields on each object with label, API name, type, created date, description and help text. Exclude formula and roll-up summary fields from fill-rate math, since they store no entered data, but keep them in the list: they are dependencies of other fields. Fields from managed packages cannot be deleted individually, so list them separately.

### 2. Measure fill rate

Count populated records against the total:

```sql
SELECT COUNT() FROM Opportunity
SELECT COUNT() FROM Opportunity WHERE Legacy_Region__c != null
```

Adjust for the field type:

- **Checkbox:** never null, so count `= true` instead.
- **Long text area and rich text:** cannot be filtered in a `WHERE` clause. Use a report or a data export.
- **Picklist:** group by value to see whether only a default value is ever set.

```sql
SELECT Legacy_Region__c, COUNT(Id) FROM Opportunity GROUP BY Legacy_Region__c
```

On very large objects, run counts through a report or the Bulk API to avoid query timeouts. A field filled only on old records is a different case from one never filled at all, so split the count by year using `CreatedDate`.

### 3. Check recent activity

Fill rate says the field holds data. It does not say anyone still writes to it. Find the newest record that has a value:

```sql
SELECT MAX(LastModifiedDate) FROM Opportunity
WHERE Legacy_Region__c != null
```

This is a weak signal, because `LastModifiedDate` changes when any field changes. If field history tracking is on for the field, query the history object (for example `OpportunityFieldHistory` filtered on `Field`) for the last real edit. History retention is limited unless the org has Field Audit Trail, so absence of history is not proof of absence of use.

### 4. Check dependencies

Start with the **Where is this used?** button on the field's detail page, then cover what it does not see. Check each of these:

- Flows (including inactive versions you may reactivate), process automation and legacy workflow rules.
- Validation rules, formula fields, roll-up summaries, lookup filters and default values.
- Page layouts, Lightning record pages with Dynamic Forms or visibility rules, compact layouts and list views.
- Reports, report types and dashboards.
- Apex classes and triggers, including dynamic SOQL that builds field names as strings. Search the source for the API name.
- Email templates, duplicate and matching rules, assignment and escalation rules, approval processes.
- Integrations: middleware mappings, ETL jobs, warehouse queries and any external system that writes to the field.

The Tooling API object `MetadataComponentDependency` can list many references in one query, but treat it as a supplement: it is not exhaustive and does not see anything outside the org.

### 5. Talk to the owners

For every field that looks dead, ask the requesting team or the field's listed owner one question: what breaks if this disappears? Record the answer and the date.

### 6. Deprecate in stages

Never go straight from candidate to delete.

1. **Label it.** Prefix the description with `DEPRECATED`, the date, the ticket and the contact person. Optionally prefix the label so it sorts to the bottom of the field picker.
2. **Back up the data.** Export the record IDs and the field's values before changing anything.
3. **Hide it.** Remove it from page layouts and Lightning pages. Users stop seeing it, and nothing breaks.
4. **Remove field-level security.** Take read and edit away from every profile and permission set, including integration users. This is the real scream test: integrations and reports that still depend on the field will now fail or show it missing, and someone will speak up.
5. **Wait.** Hold for at least one full business cycle, long enough to cover month-end or quarter-end jobs that only run occasionally.
6. **Delete.** Remove the remaining references, then delete the field in a sandbox, run the tests, and repeat in production. If the org deploys from source control, remove the field from the repository and deploy with a destructive changes manifest so environments stay aligned.

### 7. Know what deletion does

Salesforce blocks deleting a field that Apex, formulas and some other metadata still reference, which catches part of what step 4 missed. A deleted custom field goes to the **Deleted Fields** list on the object, where it can be restored with its data for 15 days. Until that list is purged, it still counts against the object's custom field limit. After restoring, recheck layouts, field-level security and reports, since they do not reliably return to their previous state. Erasing from Deleted Fields is permanent.

### 8. Decide per field

| Situation | Decision |
|---|---|
| Never filled, no dependencies, no owner | Deprecate and delete |
| Filled only on old records, no current writes | Archive values, then deprecate |
| Low fill but used in automation or integration | Keep, document the owner and purpose |
| Duplicates another field | Migrate data to the surviving field, repoint dependencies, then deprecate |
| Formula or roll-up nothing reads | Delete after checking reports and layouts |
| Managed package field | Leave, or raise with the package owner |
| Owner says it is needed but data says otherwise | Hide from layouts, review again next quarter |

## Output format

```
Object: [name]   Records: [count]   Custom fields: [count]   Date of data pull: [date]

| Field (API name) | Type | Fill rate | Last value written | Dependencies found | Owner | Decision | Stage |

Retirement schedule
| Field | Labeled | Hidden | FLS removed | Delete after | Backup location |

Open questions: [fields waiting on an owner or an integration check]
```

## Quality checklist

- Fill rate accounts for field type (checkboxes, long text, picklist defaults).
- Every delete candidate has a dependency check that went beyond "Where is this used?".
- Integration owners confirmed, in writing, that their systems do not read or write the field.
- Data was exported with record IDs before field-level security was removed.
- The wait period covers at least one month-end or quarter-end cycle.
- Deletion was rehearsed in a sandbox with tests run.
- Source control and other environments were updated in the same change.

## Common mistakes

- **Deleting on fill rate alone.** A field used by one nightly integration can be nearly empty and still critical.
- **Trusting "Where is this used?" as complete.** It misses dynamic Apex, external systems and anything outside the org.
- **Skipping the field-level security stage.** Hiding a field from layouts tests nothing; removing access is what flushes out hidden consumers.
- **Forgetting that deleted fields still count.** Deleting to free up capacity does not free the slot until the field is erased.
- **Deleting in production first.** A sandbox run catches blocked deletions and failing tests before users feel them.
- **Leaving the field in source control.** The next deploy quietly recreates it.

## Example prompts

- "Find the custom fields on Opportunity that are barely used and tell me which ones are safe to retire."
- "Write the SOQL to check fill rate and last activity for these 40 Account fields."
- "We are near the custom field limit on Case. Build a cleanup plan with a staged deprecation schedule."
- "This field shows no references in Where is this used. What else should I check before deleting it?"
- "I deleted a field last week and a report broke. Can I get it back and what will I need to fix?"
