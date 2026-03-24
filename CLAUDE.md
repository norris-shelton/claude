# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This repository stores custom Claude Code slash commands for reuse across projects. Commands are `.md` files that describe multi-step automated tasks Claude Code executes when invoked.

## Custom Slash Commands

Commands live in `.claude/commands/` and are auto-discovered by Claude Code when working in this repository. To make them available globally (in all projects), deploy them to `~/.claude/commands/`.

## Deploying Commands

Run this to sync all commands to your user-level commands directory:
```bash
cp .claude/commands/*.md ~/.claude/commands/
```

Or to deploy a single command:
```bash
cp .claude/commands/<name>.md ~/.claude/commands/<name>.md
```

After adding or updating a command in this repo, re-run the deploy step so the latest version is picked up globally.

## Available Commands

- `.claude/commands/check-jmx-config.md` — Audits and auto-fixes JMX configuration in Spring Boot Java projects at `com.twinspires.*`. Checks DataSource wrapper class naming, `@ManagedResource` annotation, package placement, duplicate bean registrations, `application.properties` JMX properties, and the `JmxTest` integration test.
