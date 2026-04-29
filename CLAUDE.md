# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Read first — every session, before writing

Before authoring or editing **any** plugin file, skill, or `marketplace.json` entry in this repo, fetch these references with WebFetch. The schemas and best practices change; do not rely on training-data recollection.

**Plugin marketplace schema (this repo's catalog format):**
- https://code.claude.com/docs/en/plugin-marketplaces — full `marketplace.json` schema, plugin sources, version resolution, hosting
- https://code.claude.com/docs/en/plugins — `plugin.json` manifest, hooks, MCP servers, agents
- https://code.claude.com/docs/en/plugins-reference — technical reference: caching, `${CLAUDE_PLUGIN_ROOT}`, file resolution, debugging
- https://code.claude.com/docs/en/discover-plugins — install/update commands users will run against this marketplace

**Agent Skills authoring (every `SKILL.md` here is one):**
- https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview — what skills are, frontmatter fields, invocation model
- https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices — how to write a skill that the model actually picks up correctly

If a task only touches Korean copy in `README.md` or non-schema text, you can skip the fetch. For anything that touches JSON schemas or skill frontmatter, fetch first.

## What this repo is

A Claude Code plugin marketplace named `jljm-skills`, distributed at `git@github.com:justlikejesus/skills.git`. It is a **single merged marketplace** that serves two organizations — Just Like Jesus Ministries and Just Like Jesus School — distinguished only by the `category` (`ministries` | `school`) and `tags` fields on each plugin entry. Do not split into separate marketplaces unless explicitly asked; that decision was deliberate.

There is no build system, package manager, or test runner. The repo is JSON catalogs plus Markdown skill files.

## Validating and installing locally

```bash
claude plugin validate .                              # validate marketplace.json + every plugin/skill
```

Inside an interactive Claude Code session:

```
/plugin marketplace add ./                            # register this checkout as a marketplace
/plugin install <plugin-name>@jljm-skills             # install a plugin from it
/plugin marketplace update jljm-skills                # pull latest after edits
```

After editing any `plugin.json` or `marketplace.json`, run `claude plugin validate .` before committing.

## Architecture

```
.claude-plugin/marketplace.json   # the catalog — every plugin must be listed here
plugins/<plugin>/.claude-plugin/plugin.json    # one per plugin
plugins/<plugin>/skills/<skill>/SKILL.md       # one or more skills per plugin
```

The catalog uses `metadata.pluginRoot: "./plugins"`, so plugin `source` values are relative paths like `./plugins/school-seeders` resolved from the repo root (the directory containing `.claude-plugin/`, **not** the `.claude-plugin/` directory itself).

Plugins are bundled by **shared workflow context**, not by skill count. `school-seeders` groups three distinct skills (`seed-bible-translation`, `seed-course-from-srt`, `seed-quiz-from-srt`) under one plugin because they all target the same `school-monorepo` Neon database and share the two-file (data + executor), single-transaction, idempotent seed pattern. When adding a new skill, prefer extending an existing plugin if the workflow context matches; create a new plugin only when the audience or runtime context genuinely differs.

## Skill source-of-truth

The skills under `plugins/school-seeders/skills/` are **mirrored** from `~/Codes/jljm/school-monorepo/.claude/skills/`. The monorepo is upstream — that is where the skills are actively developed and tested against the real database. This repo redistributes them.

When updating a school-seeder skill:

1. Edit it in `school-monorepo/.claude/skills/<skill>/SKILL.md` first.
2. Copy the updated `SKILL.md` over to `plugins/school-seeders/skills/<skill>/SKILL.md`.
3. Bump `version` in `plugins/school-seeders/.claude-plugin/plugin.json` so installed users actually receive the update — without a version bump, Claude Code's cache treats it as unchanged.

Never edit a school-seeder `SKILL.md` only here; it will drift from the monorepo and be silently overwritten on the next sync.

## Plugin versioning gotcha

`plugin.json` declares `version`, so users only get updates when that string changes. If you change a `SKILL.md` and forget to bump `version`, existing installs keep the cached old copy. Bump on every release. Do not also set `version` in the marketplace entry — `plugin.json` wins silently and the duplication causes confusion.

## Required plugin metadata

Every plugin needs:
- An entry in `marketplace.json` with `name`, `source`, `description`, `category` (`ministries` or `school`), and relevant `tags`.
- A `plugins/<name>/.claude-plugin/plugin.json` with matching `name`, `description`, `version`, and `author`.
- Skill files at `plugins/<name>/skills/<skill>/SKILL.md` with frontmatter (`name`, `description`, optionally `user-invocable: true` and `argument-hint`).

Plugin `name` must be kebab-case — the Claude.ai marketplace sync rejects anything else even though local Claude Code is permissive.

## README is in Korean

The user-facing README.md is intentionally in Korean. When updating it, keep the Korean voice; do not translate to English.
