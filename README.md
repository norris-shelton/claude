# claude

A version-controlled collection of custom Claude Code slash commands.

## Structure

```
.claude/commands/    # Custom slash commands (auto-discovered by Claude Code in this repo)
```

## Using Commands

Commands in `.claude/commands/` are available as `/command-name` when Claude Code is running in this repository.

To make commands available globally across all projects, deploy them to `~/.claude/commands/`:

```bash
cp .claude/commands/*.md ~/.claude/commands/
```

## Available Commands

| Command | Description |
| --- | --- |
| `/check-jmx-config` | Audits and auto-fixes JMX configuration in Spring Boot projects at `com.twinspires.*` |

## Adding New Commands

Create a `.md` file in `.claude/commands/` describing the steps for Claude to follow, then deploy it with the `cp` command above.
