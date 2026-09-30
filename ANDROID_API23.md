# Android 6.0 (API 23) support

This fork runs Maestro on Android 6.0 devices and emulators. Upstream Maestro needs Android 7.0 (API 24) or newer.

## What is changed

Three commits on top of upstream, kept separate so each can be sent upstream on its own:

1. **Driver `minSdk` 23.** `maestro-android` declares `minSdk = 24`, so Android 6 refuses to install it. Lint found one API 24 call on the driver path, `AccessibilityNodeInfo.isImportantForAccessibility` in `ViewHierarchy.kt`, which is now guarded. Every dependency (gRPC, Ktor) builds and runs at 23.
2. **`logcat -d` first.** Android 6 `logcat` stops parsing options at the first filter spec. `logcat ... -s ActivityManager:D -d` ignored the `-d` and streamed forever, which hung the CLI after every flow. The flags now come before the filter spec, which works on every Android version.
3. **Legacy adb shell fallback.** dadb runs every command over the `shell,v2` service, which adbd only has since Android 7. On older devices `AndroidDeviceConnection` now detects the missing `shell_v2` feature and:
   - runs commands over the legacy `shell:` service and reads the exit code from an `echo $?` sentinel, normalising the pty's CRLF line endings
   - installs with `push` + `pm install -r`, and uninstalls with `pm uninstall`, because the `cmd` binary does not exist before Android 7
   - runs `am instrument` in the foreground on a held stream. Legacy adbd uses a pty, so a backgrounded instrumentation would get SIGHUP when the shell exits. Readiness is left to the existing port probe in `AndroidDriver.awaitLaunch()`.

   Devices with `shell_v2` take the upstream code path unchanged. Unit tests for the legacy path are in `AndroidDeviceConnectionTest`.

## Verified

On the stock `system-images;android-23;default;x86_64` emulator image: launch, tap, assert, back, scroll, text input with spaces, erase, screenshot and hierarchy. Also a GB BATM2 client buy flow on Android 6.

Not verified on Android 6: permissions, `clearState`, location mocking, airplane and night mode (both use `cmd`), screen recording.

## Installing a release

Releases are on the fork's [Releases page](https://github.com/jancerny/MaestroApi23/releases). Java 17 or newer must be on the `PATH`.

```
curl -fsSL -o /tmp/maestro.zip https://github.com/jancerny/MaestroApi23/releases/latest/download/maestro.zip
rm -rf ~/.maestro-api23 && mkdir -p ~/.maestro-api23 && unzip -q /tmp/maestro.zip -d ~/.maestro-api23
export PATH="$HOME/.maestro-api23/maestro/bin:$PATH"
export MAESTRO_DISABLE_UPDATE_CHECK=true
maestro --version
```

To pin a version, replace `latest/download` with `download/<tag>`, for example `download/2.11.0-api23.1`.

Keep `MAESTRO_DISABLE_UPDATE_CHECK=true` set. Otherwise the CLI checks mobile.dev for new releases and suggests the upstream installer, which replaces this build with one that can't run on Android 6. Installing into `~/.maestro-api23` keeps this build apart from an upstream install in `~/.maestro`.

## Publishing a release

`.github/workflows/fork-release-cli.yaml` exists only in this fork. It runs when you push a tag ending in `-api23.<n>`:

```
git tag 2.11.0-api23.1
git push origin 2.11.0-api23.1
```

The tag becomes the CLI version (`maestro --version`). Use the upstream version you last merged, followed by `-api23.<n>`. The tags have no `v` prefix, so upstream's release workflows (which trigger on `v*` and only run in `mobile-dev-inc/maestro`) don't start.

Before building, the workflow checks that both bundled driver APKs declare `minSdk 23` and runs the `maestro.android.*` tests. Running it manually from the Actions tab does the same checks and build and uploads the zip as a workflow artifact, but creates no release.

## Building

```
./gradlew :maestro-cli:installDist
maestro-cli/build/install/maestro/bin/maestro --device <serial> test flow.yaml
```

The driver APKs bundled in `maestro-client/src/main/resources` (`maestro-app.apk`, `maestro-server.apk`) are already rebuilt with `minSdk 23`. After changing anything in `maestro-android`, rebuild and copy them:

```
./gradlew :maestro-android:assembleDebug :maestro-android:assembleAndroidTest
cp maestro-android/build/outputs/apk/debug/maestro-android-debug.apk maestro-client/src/main/resources/maestro-app.apk
cp maestro-android/build/outputs/apk/androidTest/debug/maestro-android-debug-androidTest.apk maestro-client/src/main/resources/maestro-server.apk
```

`compileSdk` is 37. If Gradle cannot resolve the `android-37` platform, compile against 36 locally without committing that change. The driver does not use any API 37 feature.

## Syncing with upstream

`main` is upstream plus the commits above. Pull upstream changes by merging, not rebasing, so `main` history never gets rewritten:

```
git remote add upstream https://github.com/mobile-dev-inc/Maestro.git   # once
git fetch upstream
git merge upstream/main
```

GitHub's "Sync fork" button does the same merge, but it can't resolve the conflict below, so use the command line when upstream has touched the driver APKs.

Upstream commits new driver APKs into `maestro-client/src/main/resources` with each release, so a merge usually conflicts on `maestro-app.apk` and `maestro-server.apk`. Take either side, rebuild both APKs from the merged source as shown under Building, then finish the merge:

```
git checkout --theirs maestro-client/src/main/resources/maestro-app.apk maestro-client/src/main/resources/maestro-server.apk
# rebuild and copy the two APKs (see Building)
git add maestro-client/src/main/resources/*.apk
git commit
```

Check that the rebuilt APKs still say `sdkVersion:'23'` (`aapt dump badging maestro-client/src/main/resources/maestro-app.apk | grep sdkVersion`). Then run `./gradlew :maestro-client:test --tests 'maestro.android.*'`, which covers the legacy shell path.
