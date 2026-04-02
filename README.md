# n8n-desk-skills-hub

Agent Skills for [n8n-desk](https://github.com/geckse/n8n-desk) — the conversational desktop app for [n8n](https://n8n.io) workflow automation using the official n8n MCP server.

## Plugins

| Plugin | Description |
|--------|-------------|
| [building-n8n-workflows](plugins/building-n8n-workflows/) | Builds, validates, and manages n8n workflows programmatically using the `@n8n/workflow-sdk` and the n8n MCP server. Covers workflow creation, node discovery, AI agent workflows, expression syntax, and the validate-fix loop. |
| [writing-n8n-code](plugins/writing-n8n-code/) | Writes JavaScript and Python code for n8n Code nodes. Covers data access patterns, return format, built-in helpers, webhook data structure, and common error prevention. |

## Install via Claude Code Plugin Marketplace

Add the marketplace and install the plugins:

```
/plugin marketplace add geckse/n8n-desk-skills-hub
/plugin install building-n8n-workflows@n8n-desk-skills
/plugin install writing-n8n-code@n8n-desk-skills
```

Once installed, Claude will automatically activate the appropriate skill — `building-n8n-workflows` when you ask to create, update, or fix an n8n workflow, and `writing-n8n-code` when you need to write JavaScript or Python inside n8n Code nodes.

## Manual Usage

Copy or symlink a skill directory from `plugins/<plugin-name>/` into your agent's skills folder. The agent will discover it automatically via the `SKILL.md` frontmatter.

## Format

Skills follow the [Agent Skills](https://agentskills.io) open specification and are packaged as a [Claude Code plugin](https://code.claude.com/docs/en/plugins).
