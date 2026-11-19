# SKILLZ design (approved 2026-11-19)

## Intent
One public home for The Binary Solution's Claude Code skills, so any project (ours or anyone's)
can install them and get updates. The first skill is BuildLine: Claude runs the whole BuildLine
release workflow at least as reliably as a person typing the commands. More skills (ASO,
CleanMyTV, …) follow the same pattern.

## Goals
- Installable by anyone: `/plugin marketplace add thebinarysolution/SKILLZ`, then per-skill installs.
- Each skill also usable by hand (copy its skill folder into `~/.claude/skills/`).
- A root README and CLAUDE.md for the whole repo, plus a README and CLAUDE.md per skill.
- The BuildLine skill covers every command (init, test, build, sign setup/import/status, ship,
  submit) on iOS, Android, and npm, with preflight checks and the known failure modes.

## Non-goals / anti-goals
- No user-, client-, or team-specific content (team IDs, emails, internal tools) in any skill.
- No secrets in skills, and skills never ask users to paste secrets into chat.
- A skill must never take an irreversible store action (App Review submission, production
  release, certificate revocation, npm publish) without an explicit yes from the user.

## Approaches considered
1. **Plugin marketplace repo (chosen).** Updates reach users; per-skill install; standard format.
2. Plain skill folders to copy into `~/.claude/skills/`. Simple, but no updates or discovery.
   Kept as a manual fallback that the README documents.
3. One plugin holding every skill. Fewer entries, but users could not pick skills individually.

## Layout
```
SKILLZ/
  .claude-plugin/marketplace.json   name: tbs-skillz; one entry per skill
  README.md  CLAUDE.md  LICENSE (MIT)
  <skill>/
    README.md  CLAUDE.md            human docs; maintainer notes
    plugin/                         the installable plugin (marketplace source: ./<skill>/plugin)
      .claude-plugin/plugin.json
      skills/<skill>/SKILL.md (+ references/)
```
The plugin lives in `plugin/` because Claude Code does not load a `CLAUDE.md` at a plugin root
and `claude plugin validate` warns about one; the per-skill CLAUDE.md sits one level up instead.

## Calls made by the builder
- Marketplace name `tbs-skillz` (marketplace names must be unique per user; `skillz` alone is
  likely to collide). Skill folders are lowercase kebab-case because plugin names must be.
- MIT license.
