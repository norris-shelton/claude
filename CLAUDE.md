# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This repository stores custom Claude Code slash commands and skills for reuse across projects. Commands are `.md` files that describe multi-step automated tasks Claude Code executes when invoked. Skills are model-invoked capabilities packaged as a directory containing a `SKILL.md`.

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

## Custom Skills

Skills live in `.claude/skills/<skill-name>/SKILL.md` and are auto-discovered by Claude Code when working in this repository. To make them available globally (in all projects), deploy them to `~/.claude/skills/<skill-name>/SKILL.md`.

## Deploying Skills

Run this to sync all skills to your user-level skills directory:
```bash
for d in .claude/skills/*/; do
  name=$(basename "$d")
  mkdir -p ~/.claude/skills/"$name"
  cp "$d"SKILL.md ~/.claude/skills/"$name"/SKILL.md
done
```

Or to deploy a single skill:
```bash
mkdir -p ~/.claude/skills/<name> && cp .claude/skills/<name>/SKILL.md ~/.claude/skills/<name>/SKILL.md
```

After adding or updating a skill in this repo, re-run the deploy step so the latest version is picked up globally.

## Available Commands

- `.claude/commands/check-jmx-config.md` — Audits and auto-fixes JMX configuration in Spring Boot Java projects at `com.twinspires.*`. Checks DataSource wrapper class naming, `@ManagedResource` annotation, package placement, duplicate bean registrations, `application.properties` JMX properties, and the `JmxTest` integration test.

## Available Skills

- `.claude/skills/java-check-jmx-config/` — Check JMX configuration and DataSource setup in a Spring Boot Java service, auto-fixing missing properties and conventions.
- `.claude/skills/java-pooled-restclient/` — Configure outbound HTTP for a Spring service: managed Apache HttpClient 5 connection pool with cookie storage disabled (prevents LB stickiness pinning). Supersedes the former `java-fix-restclient-config` and `java-stop-sticky-sessions` skills.
- `.claude/skills/java-setup-opentelemetry/` — Set up OpenTelemetry observability for a Spring Boot 3 Java service (Micrometer Tracing + OTLP → DataDog Agent or any OTel collector).
