# buildline.yml reference

Only the identity section and the platform section are needed to `build`. Each later section
unlocks a command. Optional sections that are absent make their command refuse with a message
naming the missing section.

## Platform detection

- An `.xcodeproj`/`.xcworkspace` ⇒ `ios`; `gradlew`/`settings.gradle` ⇒ `android`; `package.json`
  ⇒ `npm`. Mobile wins over npm (a React Native app is not treated as an npm package).
- Both an Xcode and a Gradle project in one directory is an error: set `platform: ios` or
  `platform: android`. For React Native/Flutter, keep one `buildline.yml` in `ios/` and another
  in `android/`, and run each platform from its own directory.

## iOS

```yaml
app:
  name: My App
  bundle_id: com.example.myapp

ios:
  scheme: MyApp                 # auto-discovered if there is only one
  # workspace: MyApp.xcworkspace  # set for CocoaPods projects; else project: MyApp.xcodeproj
  # configuration: Release      # default
  # export_method: app-store-connect   # default; development | ad-hoc
  # marketing_version: "1.2.0"  # default: the project's MARKETING_VERSION
  # build_number: "42"          # default: auto = latest build at Apple + 1
  # test_destination: "platform=iOS Simulator,name=iPhone 16"   # default: newest iPhone sim

signing:                        # sign setup + manual signing for build/ship
  team_id: ABCDE12345
  cert_type: distribution       # distribution (default) | development
  asc_key:
    key_id: 2X9R4HXF34          # identifier, not a secret
    issuer_id: 57246542-96fe-1a63-e053-0824d011072a
    key: keychain:buildline-asc-key          # SECRET REF to the .p8 contents
  storage:
    git: git@github.com:you/signing-store.git   # a private repo (recommended), or:
    # path: ./signing-store                     # a committed, NOT gitignored directory
    # branch: main
    passphrase: keychain:buildline-store-passphrase   # SECRET REF
  # clean_profiles: false       # true removes the installed profile after each run

distribute:                     # ship → TestFlight
  beta_groups: ["Internal Testers"]
  uses_nonexempt_encryption: false   # answers export compliance (or ship --exempt); unset = rely on Info.plist
  # whats_new: "What to test"   # TestFlight changelog (en-US)
  # poll_interval: 15s          # default
  # poll_timeout: 30m           # default

submit:                         # submit → App Store review
  primary_locale: en-US         # default
  # locales: [en-US, de-DE]     # default: discovered from metadata/
  release_type: manual          # manual | after_approval (default) | scheduled
  # earliest_release_date: "2026-12-01T09:00:00Z"   # required when scheduled
  # phased_release: false
  # metadata_dir: metadata      # default
  # screenshots_dir: metadata/screenshots   # default
  # content_rights: false       # declare third-party content (unset = leave unchanged)
  # uses_idfa: false            # default false
  # auto_submit: false          # true = submit without --confirm (CI only; avoid)
  # build_number: "42"          # pin a build; default = highest VALID build of the version
  review:
    contact_first_name: Jane
    contact_last_name: Doe
    contact_email: jane@example.com
    contact_phone: "+1 555 0100"
    # notes: "Test steps for the reviewer"
    # demo_account_name: reviewer@example.com
    # demo_account_password: keychain:buildline-demo-password   # SECRET REF
```

Optional `test:` section for `buildline test` (used when `ios.test_destination` is unset):

```yaml
test:
  simulator_name: iPhone 16
  # simulator_os: "18.0"
  # configuration: Debug
  # coverage: true
```

## Android

```yaml
app:
  name: My App
  package_name: com.example.myapp   # the applicationId

platform: android                   # optional; auto-detected from gradlew

android:
  module: app                       # default
  variant: release                  # default
  gradlew: ./gradlew                # default
  output: bundle                    # bundle = .aab (default) | apk
  # version_name: "1.2.0"           # default: from the project
  # version_code: "42"              # default: auto = highest code at Play + 1

  signing:
    store: keystore                 # logical keystore name in the encrypted store
    key_alias: upload
    keystore_password: keychain:bl-keystore-password   # SECRET REF
    key_password: keychain:bl-key-password             # SECRET REF; default = keystore_password
    storage:
      git: git@github.com:you/android-signing-store.git   # or path:
      passphrase: keychain:buildline-store-passphrase     # SECRET REF

  play:
    service_account_json: file:~/secrets/play-sa.json  # SECRET REF

  distribute:                       # ship → testing track
    track: internal                 # internal (default) | alpha | beta | <custom>
    release_status: completed       # draft | inProgress | halted | completed
    # user_fraction: 0.1            # required iff inProgress/halted
    release_notes:
      en-US: "Bug fixes."

  submit:                           # submit → production
    track: production
    metadata_dir: play              # default
    release_status: completed
    # user_fraction: 0.2            # staged rollout; required iff inProgress/halted
    changes_not_sent_for_review: false
    # countries: ["US", "GB"]
```

## npm

```yaml
platform: npm                 # optional; auto-detected from package.json

npm:
  package_manager: auto       # auto (from the lockfile) | npm | pnpm | yarn | bun
  build: true                 # run the "build" script before publishing
  registry: https://registry.npmjs.org
  access: public              # public | restricted
  provenance: true            # added automatically when a CI OIDC issuer is present
  token: keychain:buildline-npm-token   # SECRET REF
  ship:   { tag: beta }
  submit: { tag: latest }
```

## Secret references

A secret field holds a reference; BuildLine rejects a literal value there.

| Reference | Resolves to | Use for |
|---|---|---|
| `keychain:SERVICE` | a macOS Keychain generic password | a developer Mac |
| `file:/path` or `file:~/path` | the file's contents (`chmod 600`) | a `.p8` or JSON key kept on disk |
| `env:NAME` | an environment variable | CI, injected by the runner's secret store |

Give the user these commands to run **in their own terminal** (never paste values into chat).
Leaving `-w` last with no value makes `security` prompt for the secret instead of putting it in
shell history:

```bash
# App Store Connect .p8 (read from the downloaded file)
security add-generic-password -U -a buildline -s buildline-asc-key -w "$(cat ~/Downloads/AuthKey_XXXXXXXXXX.p8)"

# Store passphrase, Android keystore passwords, npm token, demo account password (prompts)
security add-generic-password -U -a buildline -s buildline-store-passphrase -w
```

For the store passphrase there is a shortcut: `buildline sign setup --save-passphrase` prompts,
saves it to the Keychain, and rewrites `signing.storage.passphrase` to
`keychain:buildline-store-passphrase`.

CI convention: `.p8` in `BUILDLINE_ASC_KEY_P8`, passphrase in `BUILDLINE_STORE_PASSPHRASE`,
referenced as `env:...`. Every command is non-interactive there; exit code is 0 or 1.
