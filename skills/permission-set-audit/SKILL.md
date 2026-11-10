---
name: permission-set-audit
description: "Use when auditing who can see and change what in a Salesforce org: profiles, permission sets, permission set groups, high-risk system permissions, object and field access, and sharing. Produces a findings table with severity and a staged remediation plan, including a move from profile-based to permission-set-based access."
license: MIT
metadata:
  title: "Permission Set Audit"
  category: "security-access"
  author: "flowsprite"
  summary: "Audit Salesforce access across profiles, permission sets and sharing, rank the risky grants by severity and plan a safe cleanup."
  tags: "salesforce, permission sets, profiles, sharing, security audit"
  version: "1.0"
  updated: "2026-11-10"
---
# Permission Set Audit

This skill produces an evidence-based answer to the question every Salesforce security review starts with: who can see and change what, and why. It traces each risky grant back to the profile, permission set or group that gives it, rates it, and turns the findings into a remediation plan you can roll out without locking anyone out of their work.

## When to use

- Before an internal security review, a SOC 2 or ISO audit, or a customer security questionnaire.
- After a merger, a reorg or a long period where admins granted access ticket by ticket.
- When an integration user, a contractor or a former admin turns out to have more access than anyone expected.
- When planning a move from profile-based access to permission sets and permission set groups.

## What you need

Ask for these first. If the user can only share exports, run the audit on exports and say which checks could not be done.

1. **Read access to setup metadata:** the ability to run SOQL against `PermissionSet`, `PermissionSetAssignment`, `ObjectPermissions`, `FieldPermissions` and `User`, or exports of the same.
2. **A list of sensitive data:** objects and fields that hold personal, financial or contract data (for example salary, tax ID, discount or health fields).
3. **The org's user population:** active users by profile and role, plus which accounts are integration or service users.
4. **Business owners:** who can confirm whether a team actually needs a permission.
5. **Constraints:** any compliance framework in scope and change windows for production.

## Process

### 1. Inventory the access model

List every profile, permission set and permission set group with its assignment count. Note that each profile has a hidden permission set behind it (`IsOwnedByProfile = true`), so one query over `PermissionSet` covers both. Flag unassigned permission sets and profiles with no active users as cleanup candidates, not risks.

### 2. Find who holds high-risk system permissions

Query assignments for each dangerous permission. This catches grants from profiles, permission sets and groups in one pass:

```sql
SELECT Assignee.Name, Assignee.Username, Assignee.Profile.Name,
       PermissionSet.Name, PermissionSet.IsOwnedByProfile
FROM PermissionSetAssignment
WHERE Assignee.IsActive = true
  AND PermissionSet.PermissionsModifyAllData = true
```

Repeat with `PermissionsViewAllData`, `PermissionsAuthorApex`, `PermissionsManageUsers`, `PermissionsCustomizeApplication` and `PermissionsApiEnabled`. Why each matters:

- **Modify All Data** and **View All Data** bypass sharing for every object. Treat each holder as an admin.
- **Author Apex** lets a user write code that runs in system context, so it is effectively full data access.
- **Manage Users** lets a user assign profiles and permission sets, including to themselves. It is a privilege escalation path.
- **Customize Application** allows changes to objects, fields, layouts and automation.
- **API Enabled** is the gate for data loaders, scripts and bulk export. Ask who needs it rather than assuming everyone does.

Also check `Assign Permission Sets`, `Reset User Passwords`, `Export Reports` and `Modify Metadata Through Metadata API Functions` for the same pattern.

### 3. Check object and field access

For each sensitive object, query `ObjectPermissions` for `PermissionsViewAllRecords` and `PermissionsModifyAllRecords`, which override sharing on that object. Then check `PermissionsDelete` on core objects. For sensitive fields, query `FieldPermissions` filtered on `SobjectType` and `Field`, and list every permission set that grants read or edit. Compare that list against the people who need it.

### 4. Review sharing

Record the org-wide default for each sensitive object, both internal and external. A Public Read/Write default on an object holding contract or personal data is a finding by itself. Then review sharing rules (owner-based and criteria-based), the role hierarchy depth, whether **Grant Access Using Hierarchies** is on for custom objects, and any public groups that include "All Internal Users". Sharing only narrows record access; it never limits what View All or Modify All grants.

### 5. Check permission set groups and muting

For each group, list its component permission sets and any muting permission set. A muting permission set only removes permissions granted inside that same group. It does not remove anything a user gets from their profile or from a permission set assigned outside the group. Confirm the user does not get the muted permission another way.

### 6. Rate each finding

| Severity | Meaning |
|---|---|
| Critical | Bypasses sharing or enables escalation for users who are not admins, or for any inactive or shared account |
| High | Sensitive object or field readable or editable beyond the people who need it |
| Medium | Broad but non-sensitive access, API access without a named use, unclear ownership |
| Low | Hygiene: unused permission sets, empty profiles, duplicate grants |

### 7. Plan the remediation

Order the plan by severity, then by blast radius. For a move off profiles:

1. Build permission sets by job function, then bundle them into permission set groups per persona.
2. Assign the groups alongside the existing profiles, so nothing is lost.
3. Strip permissions from the profiles in stages, moving users toward a minimal profile such as Minimum Access - Salesforce.
4. Test each stage in a full or partial sandbox with real user personas before production.
5. Use assignment expiration dates for temporary access instead of permanent grants.

## Output format

```
Scope: [org, objects in scope, date of data pull]
Access model summary: [profiles, permission sets, groups, active users]

Findings
| # | Severity | Permission or access | Granted by | Held by | Why it matters | Fix |

High-risk holders
| Permission | Holders (count) | Named users | Expected? |

Sharing summary
| Object | Internal OWD | External OWD | Sharing rules | Note |

Remediation plan
Phase 1 (this week): [critical fixes, owner, rollback]
Phase 2: [high and medium fixes]
Phase 3: [profile to permission set migration steps]

Checks not run: [and why]
```

## Quality checklist

- Every finding names the exact profile, permission set or group that grants the access.
- High-risk permissions were queried through assignments, not only by reading profiles.
- Integration, service and inactive users were checked separately.
- Muting permission sets were checked against grants from outside their group.
- Each fix has an owner and a rollback step.
- Sensitive fields were checked at field level, not inferred from object access.
- Checks that could not run are listed, so the report does not overstate coverage.

## Common mistakes

- **Auditing profiles only.** Most excess access now arrives through permission sets and groups. A clean profile list proves little.
- **Trusting sharing to protect data.** View All Data, View All Records and Modify All Records ignore sharing entirely.
- **Treating API Enabled as harmless.** It is the route for bulk extraction. Every holder should map to a named use.
- **Removing access in one big change.** Stripping a profile before the replacement groups are assigned breaks people's work on Monday morning.
- **Assuming muting is global.** It only applies inside its group.
- **Ignoring the integration user.** Integration accounts often carry Modify All Data because it was the quickest fix during setup.

## Example prompts

- "Audit who has Modify All Data, View All Data and Author Apex in our org and tell me which grants look wrong."
- "Here are exports of our PermissionSetAssignment and FieldPermissions tables. Build a findings table with severity."
- "Plan our move from profile-based access to permission set groups for the sales and service teams."
- "Our integration user can see everything. Work out the minimum access it needs and how to cut it back safely."
- "Review our sharing model for Opportunity and Contract and flag anything too open."
