# FlowSprite Salesforce Skills

Free, tool-agnostic skills for Salesforce admins, RevOps teams and AI agents that work on Salesforce orgs: Flow reviews, validation rules, permission audits, field cleanup, SOQL, Apex tests and safe deployments.

Each skill is a single `SKILL.md` file that teaches an AI assistant how to do one Salesforce job well: what to ask for, the steps to follow, the output format and a quality checklist. No software, integrations or API keys are required. They work in Claude, ChatGPT, Gemini, Copilot, Cursor or any assistant that accepts instructions.

Browse, search and copy every skill at **[flowsprite.com/skills](https://flowsprite.com/skills/)**.

## Install

**With the skills CLI** (Claude Code, Cursor, Copilot and other agents that support Agent Skills):

```bash
# all skills
npx skills add noahatcheckpoint/flowsprite-skills

# one skill
npx skills add noahatcheckpoint/flowsprite-skills --skill flow-review
```

**Any other assistant:** open a skill's `SKILL.md`, copy it and paste it into a project's instructions, a custom GPT, a Gem or the start of a chat.

## Skills

The tables below list the starter library. New skills are added to [`skills/`](skills/) over time; the full, current catalog is always at [flowsprite.com/skills](https://flowsprite.com/skills/).

### Admin & Configuration

| Skill | What it does |
|---|---|
| [Validation Rule Writer](skills/validation-rule-writer/SKILL.md) | Write a Salesforce validation rule that blocks bad data, explains itself to users, lets integrations through and comes with tests. [View](https://flowsprite.com/skills/validation-rule-writer/) |

### Flows & Automation

| Skill | What it does |
|---|---|
| [Salesforce Flow Review](skills/flow-review/SKILL.md) | Review a Salesforce Flow before release: entry conditions, bulk safety, recursion, fault paths, hard-coded IDs and a test plan, ranked by risk. [View](https://flowsprite.com/skills/flow-review/) |

### Data & SOQL

| Skill | What it does |
|---|---|
| [SOQL Query Builder](skills/soql-query-builder/SKILL.md) | Turn a plain-English data question into correct, selective SOQL, with the right relationships, filters and aggregates, plus the caveats of the result. [View](https://flowsprite.com/skills/soql-query-builder/) |
| [Unused Field Cleanup](skills/field-usage-cleanup/SKILL.md) | Find custom fields nobody uses, prove they are safe to remove, and retire them in stages without breaking automation or integrations. [View](https://flowsprite.com/skills/field-usage-cleanup/) |

### Security & Access

| Skill | What it does |
|---|---|
| [Permission Set Audit](skills/permission-set-audit/SKILL.md) | Audit Salesforce access across profiles, permission sets and sharing, rank the risky grants by severity and plan a safe cleanup. [View](https://flowsprite.com/skills/permission-set-audit/) |

### DevOps & Release

| Skill | What it does |
|---|---|
| [Metadata Deployment Review](skills/metadata-deploy-review/SKILL.md) | Review a Salesforce metadata change before production: dependencies, destructive changes, access side effects, tests, deploy order, rollback and a go/no-go. [View](https://flowsprite.com/skills/metadata-deploy-review/) |

### Apex & Testing

| Skill | What it does |
|---|---|
| [Apex Test Plan](skills/apex-test-plan/SKILL.md) | Plan and write Apex tests that prove behavior with isolated data, bulk and negative cases, runAs checks and mocked callouts, not just coverage. [View](https://flowsprite.com/skills/apex-test-plan/) |

### AI Agents & MCP

| Skill | What it does |
|---|---|
| [AI Agent Org Access Plan](skills/ai-agent-org-access/SKILL.md) | Plan safe Salesforce org access for an AI agent: sandbox first, a least-privilege integration user, reviewed production changes and a revoke path. [View](https://flowsprite.com/skills/ai-agent-org-access/) |

## Format

Every skill follows the Agent Skills format: YAML frontmatter (`name`, `description`, `license`, `metadata`) and a markdown body with seven sections: When to use, What you need, Process, Output format, Quality checklist, Common mistakes and Example prompts. The site validates every file on each sync and refuses one that breaks the format.

## About

Maintained by the team behind [FlowSprite](https://flowsprite.com/), which turns what your AI proposes into a pull request in your own GitHub repo before anything reaches Salesforce. FlowSprite is a product of Checkpoint GTM and is not affiliated with Salesforce, Inc.

## License

MIT. See [LICENSE](LICENSE).
