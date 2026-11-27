# AGENTS.md

Fork of Telegram for Android ("Fork Client" / porkgram); lineage: DrKLO/Telegram → Forkgram → this repo. Single branch `dev`, no tags, ~14k tracked files. Most of the tree is upstream Telegram code with fork changes layered on top.

## Setup gotchas

- Git submodules are required: `git submodule update --init --recursive`. Without them `TMessagesProj_Modules/media`, `TMessagesProj/lib/jlatexmath`, and all `TMessagesProj/jni/third_party/*` are empty and the build fails. A freshly cloned working copy does not have them initialized.
- Every Gradle build triggers the native prebuild automatically (`preBuild.dependsOn buildNativeDeps` → `TMessagesProj/jni/prepare.py` → `prebuild/build_all.sh`): BoringSSL, FFmpeg, Opus, dav1d, TDLib etc. Requires NDK r27c (27.2.12479018), cmake 3.22.1, rustup with Android targets. First build is very long. The documented path is Linux (BUILDING.md targets Debian 13; CI runs ubuntu-24.04).
- Use JDK 17 (CI and Dockerfile). The F-Droid recipe (`prebuild_fdroid.sh`) sed-patches the build to Java 21 — keep its sed lists in sync if you change build files or dependencies.
- `gradle.properties` is the app config surface: `APP_ID`/`APP_HASH` are placeholders (`0`) — the app builds but cannot log in without real Telegram API credentials. `ADDITIONAL_BUILD_NUMBER` becomes the 4th version component (`utils.gradle`); `F_DROID=1` switches to the FOSS build (applicationId `org.forkgram.messenger`, Play Services stripped).
- Root `build.gradle` `subprojects {}` block (jlatexmath font stripping + media3 namespace fallback) is load-bearing — read its comments before touching.

## Build commands

- APK: `./gradlew :TMessagesProj_App:assembleAfatRelease` (from repo root; `gradlew.bat` on Windows). Output: `TMessagesProj_App/build/outputs/apk/afat/release/app.apk`.
- CI-equivalent unsigned build: add `-PunsignedCiRelease=true`.
- Fast single-ABI build: add `-Parm64-v8a` or `-Parmeabi-v7a` (checked in `TMessagesProj/build.gradle` cmake args).
- Manual native prebuild: `cd TMessagesProj/jni && ./prebuild/build_all.sh` with `ANDROID_NDK_HOME`, `ANDROID_SDK_ROOT`, `ABIS` set (see BUILDING.md).

## Module map

- `TMessagesProj/` — nearly all app logic (library, namespace `org.telegram.messenger`). `src/main/java/org/telegram/**` mirrors upstream Telegram layout.
- `TMessagesProj_App/` — the real application module; applicationId `org.forkclient.messenger` + `.beta` suffix unless `F_DROID=1`. Flavor `afat` = plain APK, `bundleAfat*` = AAB, `forkTest` = testing (arm64 only). All other variants are ignored via `variantFilter`.
- `TMessagesProj_AppHuawei/`, `TMessagesProj_AppStandalone/`, `TMessagesProj_AppFork/` — present on disk but NOT in `settings.gradle`; they do not build here (upstream leftovers). Editing them has no effect on the build. (`TMessagesProj_AppHockeyApp/` was deleted in the telemetry cleanup.)
- `TMessagesProj_Modules/media/` — submodule, patched media3 fork; its `media-lib-*` projects are included via its `core_settings.gradle`.
- `buildSrc/` — custom Gradle plugins: TL-schema codegen (`org.telegram.tlrpc`), generated localization code (`GenerateLocalizationUtilsJavaTask`), and the `test-generator` plugin. Registered as `org.telegram.build-plugin` / `org.telegram.build-app-plugin`.
- `TMessagesProj_AppTests/` — instrumentation tests only (no unit-test dirs anywhere). The `test-generator` plugin generates androidTest Kotlin from `tlscheme/*.json` on every preBuild; tests run on an emulator/device.

## Fork conventions

- Fork features are mostly option toggles wired into `org.telegram.ui.ForkSettingsActivity` (SharedPreferences-backed `prefs()`), committed as "Added X to Fork Settings" / "Added option to Y".
- Telemetry is intentionally stripped: no App Center/HockeyApp, no SafetyNet/Play Integrity (JNI stub `onIntegrityCheckClassic` kept — registered in `TgNetWrapper.cpp`), no FCM, no `help.saveAppLog` events. Push works via UnifiedPush only (`PUSH_TYPE_SIMPLE`); `GooglePushListenerServiceProvider` is gone, `allow_app_hash`/`allow_firebase` are hardcoded `false`. `StatsController` is local-only (powers the "Data Usage" screen).
- Commit title drives CI on `dev` (`.github/workflows/tandroid.yml` parses the head-commit message): `[Release]` → sign + publish to the Telegram update channel + draft GitHub release; `[Optional]` → publish optional APK via Bot API; anything else → verify build only. Every push runs a CI build.
