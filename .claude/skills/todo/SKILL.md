---
name: todo
model: sonnet
effort: low
description: This skill should be used when the user invokes "/todo" or asks to update TODO.md with the changes and improvements made during this session.
version: 0.8.0
metadata:
  author: Twinspires Engineering
  tags: workflow,documentation,audit-trail
  alwaysApply: "false"
allowed-tools: [Read, Edit, Write]
---

# Todo

Maintain a permanent `TODO.md` file at the project root to track all changes and improvements made by Claude Code. When working on this codebase:

- **ALWAYS update TODO.md** to document any changes, improvements, or tasks completed
- Add new entries at the **TOP** of the file with the current date
- Use **checkboxes** to track task completion status:
  - `- [ ]` for pending/in-progress tasks
  - `- [x]` for completed tasks
- Include **detailed descriptions** of what was changed, added, or improved
- Organize entries by date with clear section headers (e.g., "## Spring Boot 3 Observability Implementation - October 22, 2025")
- Provide summary sections explaining the impact and benefits of changes

If `TODO.md` does not exist at the project root, create it with a `# TODO` header before adding entries.

This approach maintains a comprehensive audit trail of all development work and allows for easy tracking of project evolution over time.
