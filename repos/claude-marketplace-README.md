# claude-marketplace

Shared Claude Code skills marketplace for the Ditto team.

## Prerequisites

Some plugins depend on skills and MCP servers from other sources. Install them first.

- [Exa MCP](https://exa.ai/) requires an Exa account (OAuth on first use)
- [GitHub plugin](https://github.com/anthropics/claude-plugins-official) requires `gh` CLI installed and authenticated

```bash
claude plugin marketplace add anthropics/claude-plugins-official
claude plugin install superpowers
claude plugin install github
claude mcp add --transport http exa "https://mcp.exa.ai/mcp?tools=get_code_context_exa"
```

## Install

```bash
# Add the marketplace
claude plugin marketplace add dodo-world/claude-marketplace

# Install a plugin
claude plugin install claude-skills
```

## Available Plugins

### claude-skills

Shared Claude Code skills for the Ditto team.

| Name | Type | Description |
|------|------|-------------|
| `/pr-comments` | Command | Fetch, address, and reply to PR review comments |
| `responding-to-pr-comments` | Skill | Auto-triggered when handling PR review feedback |
| `get-code-context-exa` | Skill | Find real code snippets and docs via Exa (requires Exa MCP) |

### commit-commands

Git workflow commands for committing, pushing, and creating PRs.

| Command | Description |
|---------|-------------|
| `/commit` | Create a git commit with gitmoji |
| `/commit-push-pr` | Commit, push, and open a PR |
| `/clean_gone` | Clean up local branches deleted on remote |

## Adding a New Plugin

1. Create a new directory under `plugins/` with its own `.claude-plugin/plugin.json`
2. Add an entry to `.claude-plugin/marketplace.json`
