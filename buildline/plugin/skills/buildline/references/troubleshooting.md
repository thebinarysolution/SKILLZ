# Troubleshooting: error → cause → fix

Find the first real error in the command output (full logs: `.buildline/*.log`). Fix what you
can in config or by running the missing step; hand the user only what needs their account or
their secrets.

## Signing and certificates (iOS)

| Symptom | Cause | Fix |
|---|---|---|
| App Review email: **ITMS-90035 Invalid Signature**, though TestFlight accepted the build | The distribution certificate that signed the build was revoked after upload (often by `sign setup --revoke-existing` after a store lost its key). Apple re-checks the signature at review. | Run `buildline ship` to upload a build signed with the current certificate, then `submit`. Then fix why the store lost its key (below) so it doesn't recur. |
| `submit`: "the certificate that signed it has since been revoked" | Same as above, caught before staging (BuildLine ≥ 0.5.0). | `buildline ship`, then `submit` again. |
| "the signing store lists … certificate … but its private key is missing" | Half-present store: the index is committed, the `certs/` blobs are not (usually gitignored). | Restore the `.enc` files from a backup or the original machine, or `buildline sign import --p12`. Then commit them (they're encrypted) or move to a `git:` store. Revoke only if the key is truly gone. |
| "an Apple distribution certificate already exists … but its private key is not in the signing store" | Apple has a certificate created elsewhere (another machine, Xcode, a lost store). | Prefer `sign import --p12` with that identity. `--revoke-existing` invalidates every unapproved build it signed; ask the user first and plan a re-ship. |
| "no signing store at …" / "signing store at … is empty or unreachable" | `path:` store missing (lost to git), or the `git:` remote is wrong or not accessible. | Recover the store; check the remote and SSH access. Only `sign setup`/`sign import` may create a new store. |
| `wrong store passphrase` | The passphrase reference resolves to a different value than the store was created with. | Fix the reference; the user re-saves the right value (`sign setup --save-passphrase`). |
| "signing.storage.passphrase (…) is not set on this machine" | Reference doesn't resolve, and there's no TTY to prompt. | The user runs `buildline sign setup --save-passphrase` in their own terminal. |
| `No profiles for '<id>' were found` | Manual signing configured but no profile in the store. | `buildline sign setup`, or remove `signing:` to let Xcode sign automatically. |
| `sign status` exits non-zero | Store and Apple disagree (revoked or expired cert, deleted profile). | Read its table; `sign setup` repairs a missing or expired profile; a revoked cert needs the recovery above. |

## App Store Connect

| Symptom | Fix |
|---|---|
| `App Store Connect rejected the API key (401)` | Check `asc_key.key_id`/`issuer_id`, that the `.p8` reference resolves, and the key has App Manager or Admin. |
| `no app record for <bundle id>` | The user creates the app in App Store Connect → Apps → ＋ (the API can't). |
| `this build number is already on App Store Connect` | Let it auto-increment (remove a pinned `ios.build_number`) or bump it. |
| `no VALID build for version X` on submit | `ship` first, or the marketing version in config doesn't match the uploaded build's. |
| `no distribute.beta_groups in buildline.yml` | Ask the user for the TestFlight group name(s) and add them. |
| Build stuck in **Missing Compliance** despite `--exempt` | A hand-maintained Info.plist lacks `ITSAppUsesNonExemptEncryption` = `false` (Xcode's build setting isn't merged into it). Add it as a Boolean and verify on the built app with PlistBuddy. React Native may strip it on `pod install`; re-add in a Podfile `post_install` hook. |
| Screenshot display-type rejected | Use the token BuildLine lists (iPhone 6.9" = `APP_IPHONE_67`, iPad 13" = `APP_IPAD_PRO_3GEN_129`). |
| `multiple schemes found` | Set `ios.scheme` or pass `init --scheme`. |
| `found both an Xcode project and a Gradle project` | Set `platform:`; keep separate configs in `ios/` and `android/`. |
| Transient 500 / 429 from App Store Connect | BuildLine retries with backoff; if it still fails, re-run later. |

## Google Play

| Symptom | Fix |
|---|---|
| `permission denied` (401/403) | Grant the service account the app permissions in Play Console → Users and permissions; confirm the app exists. |
| "signed with the wrong key … expected SHA1 …" | Existing app: `sign import --keystore` the registered upload key (compare SHA1s). |
| `this version code already exists at Play` | Wire `-PversionCode` into `build.gradle` (see android.md) or bump `android.version_code`. |

## npm

| Symptom | Fix |
|---|---|
| `<pkg>@<v> is already published` | Bump `version` in `package.json`. |
| `is not on the registry yet — run buildline ship` | `submit` only promotes a published version; `ship` first. |
| `403 Forbidden` / `ENEEDAUTH` | Token lacks publish rights to the package or scope, or a scoped package needs `access: public`. |
