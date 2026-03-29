> **Note:** This repository follows the [agentskills.io](https://agentskills.io) specification and is installable as a Claude Code plugin marketplace.

# QA Skills for Claude Code

Full-stack QA skills that teach Claude to test like a senior QA engineer — UI flows, APIs, databases, and Jira ticket writing. Each QA session builds a project knowledge base that tracks user patterns, friction points, and improvement opportunities.

## Skills

| Skill | What It Does |
|-------|-------------|
| [ui-qa-testing](./skills/ui-qa-testing) | Browser automation with playwright-cli — test user flows, multi-role testing, screenshots as evidence, E2E spec generation |
| [api-qa-testing](./skills/api-qa-testing) | API testing with curl, Hurl (assertions + chaining), hey (load testing), httpie, jq |
| [db-qa-testing](./skills/db-qa-testing) | Query optimization, index analysis, business logic auditing for PostgreSQL, MySQL, MongoDB |
| [jira-story-writing](./skills/jira-story-writing) | QA-friendly Jira stories with Given/When/Then acceptance criteria, test data, playwright-cli commands |

## Commands (Claude Code specific)

| Command | What It Does |
|---------|-------------|
| [/qa](./commands/qa.md) | Single entry point — invoke anytime to test UI, API, DB, or write Jira tickets. Supports parallel multi-role testing. |
| [/qa-report](./commands/qa-report.md) | Generate email-ready .md reports from accumulated QA knowledge base data. |

## Install

### Claude Code Plugin (Recommended)

Register this repo as a plugin marketplace, then install:

```bash
# Add marketplace
/plugin marketplace add deepaktiwari09/claude-qa-skills

# Install via plugin browser
# Select "Browse and install plugins" → "qa-skills" → "Install now"

# Or install directly
/plugin install qa-skills@qa-skills
```

### Install Commands

Commands are Claude Code-specific and need manual copy:

```bash
# Clone the repo
git clone https://github.com/deepaktiwari09/claude-qa-skills.git /tmp/claude-qa-skills

# Copy commands to your global Claude Code config
cp /tmp/claude-qa-skills/commands/*.md ~/.claude/commands/

# Clean up
rm -rf /tmp/claude-qa-skills
```

### Using upskill CLI (Alternative)

If you have the [upskill](https://github.com/ai-ecoverse/gh-upskill) CLI:

```bash
# Install all skills globally
upskill -g deepaktiwari09/claude-qa-skills --all

# Or install specific skills
upskill deepaktiwari09/claude-qa-skills --skill ui-qa-testing --dest-path ~/.claude/skills

# List available skills
upskill deepaktiwari09/claude-qa-skills --list
```

## Prerequisites

Install the tools each skill uses:

```bash
# UI testing (required for ui-qa-testing)
npm install -g @playwright/cli

# API testing (required for api-qa-testing)
brew install hurl        # Assertions + chaining
brew install hey         # Load testing
brew install httpie      # Readable API exploration
# jq is usually pre-installed on macOS

# DB testing — use your existing psql/mysql/mongosh clients
```

## How It Works

### The QA Pipeline

```
/qa [test something]
  → Test with playwright-cli / curl / hurl / psql
  → Take screenshots / capture results
  → Upload evidence to Jira ticket
  → Save as reusable E2E spec (.spec.ts / .hurl / .sql)
  → Snapshot browser state for faster retesting
  → Feed project knowledge base

/qa-report [report type]
  → Read knowledge base → Generate email-ready .md
  → Save to e2e/reports/ → Copy-paste into email
```

### Knowledge Base (Built Automatically)

Every QA session contributes to a per-project knowledge base:

**UI Knowledge Base** (`e2e/knowledge/`)
- Flow maps — steps, time, friction, variants
- Session reports — metrics, errors, UX observations
- Pain points — accumulated by severity
- Improvements — prioritized backlog

**DB Knowledge Base** (`e2e/knowledge/db/`)
- Schema map — tables, relationships, constraints
- Service-DB mapping — which code hits which tables
- Business rules audit — where rules are enforced (DB vs code vs frontend)
- Query cost analysis — expensive queries mapped to infrastructure cost
- Stability risks — race conditions, missing transactions

### Parallel Multi-Role Testing

Test multiple user roles simultaneously using subagents:

```
/qa test login as admin, editor, and viewer in parallel
```

Each role gets its own playwright-cli browser session with isolated state.

## Repo Structure

```
claude-qa-skills/
├── .claude-plugin/
│   └── marketplace.json       ← Plugin marketplace manifest
├── skills/
│   ├── ui-qa-testing/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── user-story-testing.md
│   │       ├── e2e-spec-generation.md
│   │       ├── self-qa-workflow.md
│   │       └── knowledge-base.md
│   ├── api-qa-testing/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── hurl-patterns.md
│   │       └── performance-testing.md
│   ├── db-qa-testing/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── postgresql-optimization.md
│   │       ├── mongodb-patterns.md
│   │       └── business-logic-knowledge-base.md
│   └── jira-story-writing/
│       ├── SKILL.md
│       └── references/
│           ├── story-examples.md
│           └── bug-patterns.md
├── commands/
│   ├── qa.md                  ← /qa command
│   └── qa-report.md           ← /qa-report command
└── README.md
```

## License

Apache-2.0
