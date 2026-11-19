# CLAUDE.md: buildline skill

Maintainer notes for the `buildline` skill. Repo-wide rules (layout, naming, versioning, scrub,
validation) are in the root [CLAUDE.md](../CLAUDE.md).

## Sources of truth

Everything in the skill must match the BuildLine version it targets (currently **0.5.0**):

- `buildline <command> --help` for every command and flag (`init`, `test`, `build`, `ship`,
  `submit`, `sign setup|import|status`).
- The BuildLine README published on the tap: https://github.com/thebinarysolution/homebrew-buildline
- `buildline.yml` field names come from BuildLine's config parser; don't add a field to
  `references/config.md` without confirming it exists.

## When BuildLine releases a new version

1. Diff every command's `--help` against the skill; add new flags and commands.
2. Read the release's CHANGELOG entry (on the tap) for behavior changes, new error messages, and
   new failure modes; update `references/troubleshooting.md`.
3. Update the minimum version in `SKILL.md` Step 1 if the skill now relies on the new release.
4. Bump `plugin/.claude-plugin/plugin.json` `version` and validate.

## Behavior the skill depends on

- Prompts only happen with a TTY. Without one: `init` needs `--scheme` when there are several;
  `ship`/`build` default to a fresh archive; the version prompt is skipped; an unset
  `distribute.beta_groups` and an unresolvable store passphrase are errors. The skill therefore
  sets these before running.
- `submit` without `--confirm` stages but never sends; `ship` on iOS/Android always commits to
  TestFlight / the testing track; npm `ship` publishes publicly. The skill's consent rules follow
  exactly this split; keep them aligned if BuildLine changes it.

## Layout

- `plugin/skills/buildline/SKILL.md`: workflow (install check → config → preflight → run → report),
  ground rules, and the revoke warning. Keep it under ~200 lines; detail goes in references.
- `references/config.md`: full `buildline.yml` per platform and secret references.
- `references/ios.md`, `android.md`, `npm.md`: platform setup and command details.
- `references/troubleshooting.md`: error → cause → fix tables.

## Verifying a change

1. `npx -y @anthropic-ai/claude-code plugin validate .` from the repo root passes with no warnings.
2. Install from the local checkout (`claude plugin marketplace add ./` then
   `claude plugin install buildline@tbs-skillz`) and, in a real app project, ask for a
   `buildline build` and a `buildline submit` (without `--confirm`). Check that Claude runs the
   preflight checks and stops before anything irreversible.
