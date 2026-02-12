# Twill Automation Templates

Ready-to-use automation templates for [Twill](https://twill.ai). Browse, customize, and deploy recurring coding automations powered by AI agents.

## How It Works

Each template is a Markdown file in the `templates/` directory with YAML frontmatter for metadata and a body containing the agent instructions.

Templates are loaded dynamically by Twill from this repository — add a new file here and it appears in the app automatically.

## Template Format

```markdown
---
title: "My Automation"
description: "A short description shown in the template picker"
integrations:
  - github
  - linear
schedule: "0 9 * * 1-5"
schedule_description: "Every weekday at 9 AM"
---

Your agent instructions go here. This is what the automation agent
receives on each scheduled run.

The agent can:

- Query integrations via MCP (GitHub, Linear, Notion, Slack)
- Spawn coding tasks using [[TASK]] blocks
- Take direct integration actions (assign issues, etc.)
```

### Frontmatter Fields

| Field                  | Required | Description                                                               |
| ---------------------- | -------- | ------------------------------------------------------------------------- |
| `title`                | Yes      | Display name for the template                                             |
| `description`          | Yes      | One-line summary shown in the picker                                      |
| `integrations`         | Yes      | Array of integrations used: `github`, `linear`, `notion`, `sentry`, `gcp` |
| `schedule`             | Yes      | Default cron expression (5-field)                                         |
| `schedule_description` | Yes      | Human-readable schedule description                                       |

### Body (Agent Instructions)

The Markdown body becomes the automation's `message` field — the instructions the AI agent receives on every scheduled run. Write clear, specific instructions about what the agent should do.

## Contributing

1. Fork this repository
2. Create a new `.md` file in `templates/`
3. Follow the format above
4. Open a pull request

We welcome templates for any recurring development workflow.

## Templates Index

The `templates/index.json` file lists all available templates and is used by Twill to discover them without crawling the directory.

## License

MIT
