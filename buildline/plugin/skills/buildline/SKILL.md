---
name: buildline
description: Build, sign, and ship mobile apps and npm packages with the BuildLine CLI (`buildline`). Use when the user wants to build an .ipa/.aab, set up or fix code signing, upload to TestFlight or a Google Play testing track, submit for App Store review or release to Play production, publish or promote an npm package, write or fix a buildline.yml, or debug a BuildLine, TestFlight, App Store Connect, Play, or ITMS error such as ITMS-90035 Invalid Signature.
---

# BuildLine

BuildLine is one Go binary that takes an app from source to its store, driven by `buildline.yml`
in the project. The same commands run on every platform, which is auto-detected:

| Command | iOS | Android | npm |
|---|---|---|---|
| `init` | write a starter `buildline.yml` | not supported (use the template) | not supported |
| `test` | `xcodebuild test` on a simulator | — | — |
| `build` | archive + export a signed `.ipa` | `gradlew bundleRelease` → signed `.aab`/`.apk` | install deps + `build` script |
| `sign setup` / `import` / `status` | cert + profile in an encrypted store | upload keystore in the store | — |
| `ship` | upload to **TestFlight** + beta groups | upload to a **Play testing track** | publish under the **`beta`** dist-tag |
| `submit` | stage metadata; `--confirm` sends for **App Store review** | stage production; `--confirm` **releases** | preview; `--confirm` moves **`latest`** |

Your job is to run these more reliably than a person would: settle every input up front, check
the preconditions each command needs, run it non-interactively, read the result, and report it.

## Ground rules

1. **Run from the project directory** that holds the Xcode project, `gradlew`, or `package.json`
   (for React Native/Flutter that is usually `ios/` or `android/`, wherever `buildline.yml` lives).
2. **You have no terminal.** Prompts are skipped or fail without a TTY, so decide every answer
   first and pass it as a flag or write it into `buildline.yml`:
   - `init`: pass `--scheme` and `--name` (and `--force` only if the user agreed to overwrite).
   - `build`/`ship` (iOS): always pass `--new-archive` or `--reuse-archive`.
   - `ship` (Android): always pass `--new-build` or `--reuse-build`.
   - Versions: `ship` keeps the project's marketing version / version name when no TTY. If the user
     wants a new one, set `ios.marketing_version` / `android.version_name` (or the project value).
     Build numbers and version codes auto-increment; only pin them when asked.
   - `ship` (iOS) needs `distribute.beta_groups`; ask the user which group(s) and write them.
3. **Secrets are references, never values.** Never ask the user to paste a key, passphrase, token,
   or password into chat, and never write one into a file or command line. Tell them the exact
   command to run in their own terminal (see [references/config.md](references/config.md)), then
   reference it as `keychain:`, `file:`, or `env:`. A store passphrase that can't be resolved is
   fixed by the user running `buildline sign setup --save-passphrase` in their terminal.
4. **Ask before anything irreversible**, even when the user said "ship it" earlier. Name exactly
   what will happen and wait for a yes:
   - `submit --confirm` (App Store review / Play production release / npm `latest` move)
   - `submit --cancel`
   - `sign setup --revoke-existing` (see "Revoking" below; it is almost never the right fix)
   - npm `ship` (publishes a public, immutable version)
   A request to "ship to TestFlight" or "upload a beta" is consent for that `ship`; nothing more.
5. **Long runs go in the background.** `ship` polls Apple/Play processing and takes 5–20 minutes.
   Start it in the background, then read the output when it finishes. Add `-v` when diagnosing.
6. Use `--json` only when you need to parse events; the human output is easier to summarize.

## Step 1: Check the install

```bash
buildline --version
```

- Missing: the user installs it (Homebrew asks them to trust the tap once):
  `brew tap thebinarysolution/buildline && brew trust thebinarysolution/buildline && brew install buildline`
- Older than 0.5.0: suggest `brew update && brew upgrade buildline` (0.5.0 adds the
  revoked-certificate check before `submit` and the half-present signing store check).
- Toolchain: iOS `xcodebuild -version`; Android `java -version` (JDK 17+); npm `node -v`.

## Step 2: Read the project's config

Read `buildline.yml`. If it is missing:
- iOS: `buildline init --scheme <Scheme> --name "<App Name>"` (list schemes with
  `xcodebuild -list` first). It writes a minimal config plus commented templates.
- Android / npm: write it from the templates in [references/config.md](references/config.md).

Each section unlocks commands: `app`+`ios` (or `android`, `npm`) → `build`; `signing` →
`sign`, manual signing; `distribute` → `ship`; `submit` → `submit`. A missing section is the
usual reason a command refuses; add only what the user's goal needs.

## Step 3: Preflight, then run

Run the checks for the command you are about to use. Each one prevents a failure that otherwise
shows up minutes into a build or, worse, at Apple's review.

**Any command with `signing:` (iOS) or `android.signing:`**
- Signing store health. For a `path:` store, check it is committed and complete:
  `git check-ignore -v <store path>` (ignored = danger) and that every file the store's
  `buildline-store.json` lists under `certificates[].files` exists as `<name>.enc`.
  A store missing its `certs/` blobs is the classic way private keys get lost. Stop and tell the
  user; recovery is restoring the files or `buildline sign import`, not revoking.
- `buildline sign status` (read-only) before the first `ship`/`submit` on a machine. Non-zero
  exit means the store and Apple disagree; resolve that first.

**`ship` (iOS)**: the app record exists in App Store Connect; `distribute.beta_groups` is set;
the export-compliance answer is known: pass `--exempt` for apps using only standard HTTPS/TLS
(most apps) or `--non-exempt`, or confirm `ITSAppUsesNonExemptEncryption` is in the Info.plist.
Run tests unless the user asked for `--skip-tests`.

**`submit` (iOS)**: a VALID build exists for the marketing version (`ship` first); `submit:`
section and `metadata/` tree are present (layout in [references/ios.md](references/ios.md)).
Always run plain `buildline submit` first (stages, sends nothing), show the user what it staged,
then ask before `buildline submit --confirm`. If `submit` reports the build was signed with a
since-revoked certificate, the fix is a fresh `buildline ship`, never forcing the submission.

**Android `ship`/`submit`**: Play service account JSON resolves; the app exists in Play with Play
App Signing; the upload key in the store matches the one Play has registered (an existing app
needs `buildline sign import --keystore`); `build.gradle` reads `-PversionCode`, or the version
code never moves. Details in [references/android.md](references/android.md).

**npm `ship`/`submit`**: `package.json` version is new (versions are immutable); the token has
publish rights; scoped packages need `access: public`. See [references/npm.md](references/npm.md).

## Step 4: Read the result and report

On success, report what the user will look for, taken from the command's output: version and
build number (or version code / npm version), where it went (beta groups, Play track, dist-tag),
the artifact path for `build`, and any warning BuildLine printed (for example a hardcoded
`versionCode` or a profile expiring soon).

On failure, find the first real error (the summary lines; full xcodebuild/altool logs are in
`.buildline/*.log`), look it up in [references/troubleshooting.md](references/troubleshooting.md),
and either fix it (config, missing step) or tell the user the one action only they can take
(create the app record, grant a Play permission, store a secret).

## Revoking certificates

`sign setup` refuses to create a new iOS distribution certificate while Apple holds one whose
private key is not in the store, and offers `--revoke-existing`. Revoking does not affect versions
already live on the App Store, but **every uploaded build signed with that certificate that is not
yet approved becomes invalid**: App Review rejects it with ITMS-90035 "Invalid Signature", even
though TestFlight accepted it. Before revoking, rule out recovering the key (a backup of the store,
the original Mac's keychain, an exported `.p12` → `buildline sign import --p12`). If the user still
chooses to revoke, tell them which builds must be re-shipped and commit the store afterwards.

## References

- [references/config.md](references/config.md): full `buildline.yml` for iOS, Android, npm; secret references and how users store them
- [references/ios.md](references/ios.md): App Store Connect key, signing store, ship and submit details, metadata and screenshots
- [references/android.md](references/android.md): Play service account, upload keystore, version codes, tracks, Play metadata
- [references/npm.md](references/npm.md): token, dist-tags, provenance
- [references/troubleshooting.md](references/troubleshooting.md): error → cause → fix
