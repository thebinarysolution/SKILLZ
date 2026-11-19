# iOS: App Store Connect, signing, TestFlight, App Review

## Prerequisites the user must provide

- macOS with Xcode (`xcodebuild -version`).
- An **App Store Connect API key** (`.p8`) with the **App Manager** or **Admin** role: App Store
  Connect → Users and Access → Integrations → App Store Connect API. Apple lets them download the
  `.p8` once. `key_id` and `issuer_id` go in `buildline.yml`; the `.p8` is stored as a secret
  reference (see [config.md](config.md)).
- The **app record** created once in App Store Connect (Apps → ＋). The API cannot create it.
- A **store passphrase**, stored as a secret reference.

## Signing store (`sign setup`)

`buildline sign setup` creates (or reuses) a distribution certificate and an App Store profile
named `buildline <bundle_id> appstore` at Apple, stores the certificate, private key, and profile
AES-256-GCM encrypted, and pushes the store. It is idempotent; a second run with everything valid
only reads. `build`/`ship` then sign manually from an ephemeral keychain; the Xcode project can
stay on Automatic signing.

Store layout: `buildline-store.json` (plaintext index), `check.enc` (passphrase canary),
`certs/<type>/<CERT_ID>/{cert.cer,key.pem}.enc`, `profiles/<type>/<bundle_id>.mobileprovision.enc`.

Rules that keep it healthy:
- Prefer a `git:` store in a private repo shared by every machine and CI.
- A `path:` store must be **committed**, all of it. A `.gitignore` entry such as `ios/.buildline/`
  that covers the store loses `certs/` on a clone, branch switch, or `git clean`, while the index
  and profile may already be committed. Apple still holds the certificate, but its key is gone.
  BuildLine ≥ 0.5.0 stops with "its private key is missing" in this state.
- Recovery order: restore the `.enc` files from a backup or the machine that created them →
  `buildline sign import --p12 <file> --password keychain:<svc>` with an exported identity →
  only then `sign setup --revoke-existing` (see the SKILL's "Revoking certificates").
- Apple allows only a few active distribution certificates per team, so creating certificates
  per machine is not an option; share one store.

`buildline sign status` compares the store with Apple (read-only; non-zero exit on mismatch,
for example a revoked certificate).

## `build`

`xcodebuild archive` then `-exportArchive` → `.buildline/export/<App>.ipa`. Without a
`signing:` section Xcode signs automatically (with `-allowProvisioningUpdates`); with one, manual
signing from the store. Logs: `.buildline/xcodebuild-archive.log`,
`.buildline/xcodebuild-exportArchive.log`.

## `ship`

Order: test → resolve the next build number from App Store Connect → prepare signing → archive →
export → upload with `altool` → poll until processing finishes → declare export compliance and
wait for it to clear → add to `distribute.beta_groups`.

- Without a TTY the version prompt is skipped: the marketing version is `ios.marketing_version`
  or the project's `MARKETING_VERSION`; the build number auto-increments.
- `--reuse-archive` reuses `.buildline/<Scheme>.xcarchive` and takes the version baked into it.
  Use it only to retry after a failure past the archive step; otherwise `--new-archive`.
- Export compliance: `--exempt` (standard HTTPS/TLS only, most apps), `--non-exempt`, or
  `distribute.uses_nonexempt_encryption`. Best: `ITSAppUsesNonExemptEncryption` = `false` in the
  Info.plist. A hand-maintained Info.plist (common in React Native) does not get Xcode's build
  setting merged in; check the built app with
  `/usr/libexec/PlistBuddy -c "Print :ITSAppUsesNonExemptEncryption" <App>.app/Info.plist`.
- Upload log: `.buildline/altool-upload.log`.

## `submit`

Pushes app and version metadata and screenshots from `metadata/`, prepares the App Store version,
attaches the build (highest VALID build of the marketing version, or `submit.build_number`), sets
review details and declarations. Without `--confirm` it stops there: everything is staged in App
Store Connect, nothing is sent to review. `--confirm` submits for review. `--cancel` withdraws an
in-progress submission.

Since 0.5.0 it first checks the build's signing certificate is still active. A build uploaded
before every active distribution certificate existed was signed with one that has been revoked;
App Review would reject it with ITMS-90035, so `submit` refuses and asks for a new `ship`.

The age rating questionnaire is not filled by `submit`; the user sets it once in the App Store
Connect UI.

### Metadata layout (fastlane `deliver` style)

```
metadata/
  categories.yml                 # primary: UTILITIES   (required for a first submission)
  en-US/
    name.txt  subtitle.txt  description.txt  keywords.txt
    support_url.txt  privacy_url.txt  (marketing_url.txt, promotional_text.txt, release_notes.txt)
  screenshots/
    en-US/APP_IPHONE_67/01_home.png     # display-type token; NN_ prefix sets the order
```

Display-type tokens lag the marketing names: iPhone 6.9" is `APP_IPHONE_67`, iPad 13" is
`APP_IPAD_PRO_3GEN_129`. `submit` validates the token and lists valid ones on a typo.
