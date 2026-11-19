# Android: Gradle, upload key, Google Play

BuildLine's Android support is newer than iOS and less exercised against live Play accounts;
suggest trying it on a non-critical app first.

## Prerequisites the user must provide

- JDK 17+ (`keytool`, `jarsigner`); Android SDK build-tools (`apksigner`) only for `output: apk`.
- A Gradle project with `gradlew`.
- The app created in the Play Console with **Play App Signing** enabled (Google holds the app
  signing key; BuildLine manages only the upload key).
- A **Google Play service account** JSON key, stored as a secret reference.
- A **store passphrase** and the keystore passwords, stored as secret references.

## Service account setup (the user does this in their browser)

1. Enable the *Google Play Android Developer API* for a Google Cloud project.
2. Create a service account; on "Grant this service account access to project", leave the role
   **blank** (no Cloud IAM role is needed). Create a JSON key and download it.
3. Play Console → Users and permissions → Invite the service account email → App permissions for
   the app: *Release to testing tracks*, *Manage testing tracks and edit tester lists* (ship);
   *Release to production, exclude devices, and use Play App Signing* and *Manage store presence*
   (submit); *View app information* (read-only). Permissions can take minutes to propagate.
4. Reference the JSON: `service_account_json: file:~/secrets/play-sa.json` (chmod 600) or `env:`.

## Upload key (`sign setup` / `sign import`)

- New app: `buildline sign setup` generates a PKCS12 upload keystore (RSA-2048) once, stores it
  encrypted, and pushes the store. Play registers the upload certificate from the first upload
  (or the user registers it under Test and release → Setup → App signing).
- **Existing app: never generate a new key.** Play rejects a bundle signed with a different key
  ("signed with the wrong key … expected SHA1 …"). Set `android.signing.{key_alias,
  keystore_password,key_password}` to the existing keystore's values, then
  `buildline sign import --keystore /path/to/upload.jks`. Compare the SHA1 it prints with Play
  Console → App integrity → App signing → Upload key certificate before shipping.
- Lost keystore: Play Console's upload key reset (Google reviews it).

## Version codes

BuildLine passes the next version code (highest at Play + 1) and the version name to Gradle as
`-PversionCode` / `-PversionName`. That only works if `build.gradle` reads them; a hardcoded
`versionCode 1` (the React Native template) silently ignores it and every upload collides.
`ship` warns when it detects this. Wire it in `android/app/build.gradle`, computed above
`android {` (the inline `?:` form inside `defaultConfig` breaks the AGP DSL):

```gradle
def resolvedVersionCode = (project.findProperty("versionCode") ?: "1").toString().toInteger()
def resolvedVersionName = (project.findProperty("versionName") ?: "1.0").toString()

android {
  defaultConfig {
    versionCode resolvedVersionCode
    versionName resolvedVersionName
  }
}
```

Kotlin DSL: `val resolvedVersionCode = (findProperty("versionCode") as String? ?: "1").toInt()`.
Or pin `android.version_code` and manage it by hand.

## Commands

| Command | What it does |
|---|---|
| `build` | `./gradlew :<module>:bundleRelease` → signed `.aab` (or `.apk`). |
| `ship` | resolve version code → build → sign → open a Play edit → upload → assign to the testing track → commit. Testing tracks have no review gate, so `ship` always commits. |
| `submit` | stage the production release and store listing; `--confirm` commits it. Without it, the edit is validated then discarded. |

- `ship` reuses an existing `.aab`/`.apk` only with `--reuse-build` (pass `--new-build`
  otherwise). After a successful upload the artifact is deleted unless `--keep-build`.
- A failed run or Ctrl-C discards the open Play edit; nothing is left half-staged.
- `--cancel` is iOS-only. To pull an Android release, halt or replace it in the Play Console.
- Staged rollout: `release_status: inProgress` with `user_fraction`.

## Play metadata layout (fastlane `supply` style, under `android.submit.metadata_dir`, default `play/`)

```
play/
  en-US/
    title.txt               # ≤ 30 chars
    short_description.txt   # ≤ 80 chars
    full_description.txt    # ≤ 4000 chars
    video.txt               # optional
    changelogs/42.txt       # notes for version code 42 (used when release_notes is unset)
    images/
      phoneScreenshots/01.png   # subdir name is the Play image type
      featureGraphic/main.png
      icon/icon.png
```

Valid image types: `phoneScreenshots`, `sevenInchScreenshots`, `tenInchScreenshots`,
`tvScreenshots`, `wearScreenshots`, `icon`, `featureGraphic`, `tvBanner`. Images already
matching at Play are skipped, so re-running `submit` is cheap.
