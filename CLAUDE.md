# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

SKILLZ is a public **Claude Code plugin marketplace** (`thebinarysolution/SKILLZ`, marketplace
name `tbs-skillz`). Each top-level folder is one skill packaged as its own plugin, so people can
install skills individually and receive updates. Anyone can install these, so every skill must be
generic: no personal names, emails, team IDs, client names, internal hosts, or secrets.

## Layout and why

```
.claude-plugin/marketplace.json   catalog; each entry's source is ./<skill>/plugin
<skill>/README.md                 human docs (GitHub renders it)
<skill>/CLAUDE.md                 maintainer notes for that skill
<skill>/plugin/                   the plugin root that gets installed
  .claude-plugin/plugin.json      name must equal the marketplace entry name
  skills/<skill>/SKILL.md         + references/ for detail loaded on demand
```

The plugin root is `plugin/`, one level below the skill folder, because Claude Code does not load
a `CLAUDE.md` at a plugin root and `claude plugin validate` warns about one. Keep `README.md` and
`CLAUDE.md` in `<skill>/`, never inside `plugin/`.

## Adding a skill ("publish this skill")

1. Folder name and plugin name: lowercase kebab-case, the same everywhere (`<skill>/`,
   `plugin.json` `name`, marketplace entry `name`, `skills/<skill>/`). Names must not start with
   `claude-` or `anthropic-`.
2. Create `<skill>/plugin/.claude-plugin/plugin.json` with `name`, `displayName`, `version`
   (start at `1.0.0`), `description`, `author` (`The Binary Solution`, url
   `https://github.com/thebinarysolution`), `homepage`
   (`https://github.com/thebinarysolution/SKILLZ/tree/main/<skill>`), `repository`, `license: MIT`,
   `keywords`. Copy the BuildLine one.
3. Write `skills/<skill>/SKILL.md`: frontmatter `name` and a `description` that says what it does
   and when to use it, key trigger first (the listing truncates at 1,536 chars). Keep the body
   under 500 lines; move detail into `references/*.md` linked from SKILL.md.
4. Write `<skill>/README.md` (what it does, requirements, install, example requests) and
   `<skill>/CLAUDE.md` (sources of truth, how to verify changes, gotchas).
5. Add the marketplace entry (`name`, `source: "./<skill>/plugin"`, `description`) and a row in
   the root README's Skills table.
6. Scrub: `grep -rniE "@[a-z0-9.-]+\.(com|in|io)|password|token|secret" <skill>/` and review
   every hit. Skills must describe secrets as references, never contain one.
7. Validate: `npx -y @anthropic-ai/claude-code plugin validate .` (or `claude plugin validate .`)
   must end in `Validation passed` with no warnings. Fix warnings rather than ignoring them.
8. Commit and push to `main`.

## Changing a skill

Bump `version` in that skill's `plugin.json` on every change people should receive. Installed
copies are pinned to the version, so an unbumped change never reaches users. Use semver: patch
for wording fixes, minor for new coverage, major when the skill's behavior changes.

## Writing skills well

- Write for Claude, not for a reader: what to do, in what order, what to check, when to stop and
  ask. Skip narration and history.
- Claude runs commands without a TTY. Every prompt a tool might show must be answered by a flag
  or config the skill sets first.
- Irreversible or public actions (store submissions, releases, publishing, revoking, deleting)
  need an explicit yes from the user in the skill's instructions, every time.
- Never instruct Claude to ask for secrets in chat; give the user a command to run in their own
  terminal and reference the result.
- Verify facts against the tool's real `--help` and source before writing them down.
