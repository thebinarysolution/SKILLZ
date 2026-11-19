# buildline

A Claude Code skill for [BuildLine](https://github.com/thebinarysolution/homebrew-buildline), the
CLI that takes an app from source to its store with one `buildline.yml`:

- **iOS:** signed `.ipa`, reproducible code signing, TestFlight, App Store review
- **Android:** signed `.aab`/`.apk`, upload key management, Play testing tracks and production
- **npm:** build, publish under `beta`, promote to `latest`

With the skill installed, you can ask in plain words and Claude runs the right BuildLine commands,
with the checks a careful release engineer would make first.

## What the skill adds over running the commands yourself

- **Settles every input first.** Claude has no terminal, so it never relies on prompts: it
  picks the scheme, versions, beta groups, and archive reuse up front and passes them explicitly.
- **Preflight checks** before each command: config sections present, signing store complete and
  not gitignored, `sign status` agreeing with Apple, export compliance answered, Gradle wired for
  version codes, npm version not already published.
- **Stops before anything irreversible.** App Review submission, Play production release, npm
  publish, and certificate revocation always need your explicit yes, even mid-flow.
- **Secrets stay yours.** Claude never asks you to paste a key or password into chat; it gives
  you the command to store it in your Keychain and references it from `buildline.yml`.
- **Knows the failure modes**, including the costly ones: a signing store that silently loses its
  private key, and builds rejected at App Review with ITMS-90035 after a certificate was revoked.

## Requirements

- [Claude Code](https://code.claude.com)
- BuildLine 0.5.0 or later (macOS):
  ```bash
  brew tap thebinarysolution/buildline
  brew trust thebinarysolution/buildline
  brew install buildline
  ```
- Per platform: Xcode and an App Store Connect API key (iOS); JDK 17+ and a Play service account
  (Android); Node.js and an npm token (npm). The skill walks you through setting each one up.

## Install

```bash
claude plugin marketplace add thebinarysolution/SKILLZ
claude plugin install buildline@tbs-skillz
```

Or copy the skill folder by hand: `cp -R buildline/plugin/skills/buildline ~/.claude/skills/`.

## Try it

From your app's directory, ask things like:

- "Set up BuildLine for this app."
- "Ship this build to TestFlight for the Internal Testers group."
- "Bump the version to 2.3.0 and send it to the internal Play track."
- "Stage the App Store submission for 2.3.0 and show me what's in it."
- "Apple rejected my build with ITMS-90035. What happened?"
- "Why does `buildline ship` say my signing store's private key is missing?"

Or invoke it directly with `/buildline:buildline`.

## Files

- [`plugin/skills/buildline/SKILL.md`](plugin/skills/buildline/SKILL.md): the workflow Claude follows
- [`plugin/skills/buildline/references/`](plugin/skills/buildline/references/): configuration, per-platform guides, troubleshooting
