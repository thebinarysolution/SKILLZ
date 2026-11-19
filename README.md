# SKILLZ

Skills for [Claude Code](https://code.claude.com) by [The Binary Solution](https://github.com/thebinarysolution).
Each skill teaches Claude a workflow end to end, so you can say "ship this to TestFlight" in any
project and have it done the same careful way every time.

## Skills

| Skill | What it does | Docs |
|---|---|---|
| **buildline** | Build, sign, and ship iOS, Android, and npm releases with the [BuildLine](https://github.com/thebinarysolution/homebrew-buildline) CLI: TestFlight, App Store review, Google Play tracks and production, npm dist-tags. | [buildline/](buildline/README.md) |

More are on the way.

## Install

This repo is a Claude Code plugin marketplace. Add it once:

```bash
claude plugin marketplace add thebinarysolution/SKILLZ
```

Then install the skills you want:

```bash
claude plugin install buildline@tbs-skillz
```

Inside a Claude Code session, the same works as `/plugin marketplace add thebinarysolution/SKILLZ`
and `/plugin install buildline@tbs-skillz`. Claude picks a skill up automatically when your
request matches it, or you can invoke it directly, for example `/buildline:buildline`.

**Updates:** `claude plugin marketplace update tbs-skillz`, then restart Claude Code or run
`/reload-plugins`.

### Without the plugin system

Every skill is a plain folder you can copy into your personal skills directory:

```bash
git clone https://github.com/thebinarysolution/SKILLZ.git
cp -R SKILLZ/buildline/plugin/skills/buildline ~/.claude/skills/
```

You won't get updates that way; pull and copy again when you want them.

## Repository layout

```
.claude-plugin/marketplace.json   the catalog: one entry per skill
<skill>/
  README.md                       what the skill does, requirements, examples
  CLAUDE.md                       notes for whoever maintains the skill
  plugin/                         the installable plugin
    .claude-plugin/plugin.json
    skills/<skill>/SKILL.md       the instructions Claude follows (+ references/)
```

## Contributing

Issues and pull requests are welcome. Adding or changing a skill follows the rules in
[CLAUDE.md](CLAUDE.md). Run `claude plugin validate .` before opening a pull request.

## License

[MIT](LICENSE)
