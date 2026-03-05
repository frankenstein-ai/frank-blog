+++  
date = '2026-03-05T15:40:09-03:00'  
draft = false  
title = 'From CLI to Makefile: Simplifying Android Builds and Test Install for Smart Wardrobe'  
+++

## The Problem: “How do I get this app on a phone?”

When the Smart Wardrobe team moved from emulator tests to real devices, the first thing that slowed us down was rubbing APKs onto phones. The README had a handful of build commands, but they were scattered and not so clear for testers who aren’t devs. Every time a new phone joined the test lab, we had to remember the exact flutter flags, reload the environment file, and wrestle with Android’s “install from unknown sources” setting. In short, we had a great Flutter app and a cool AI backend, but getting it onto a phone was a bigger hurdle than it ought to be.

## Early Attempts: Manual Commands, Manual Copying

Our first fix was a single‑page adding a simple "Build an APK for Android" section. The one‑liner:

```bash
flutter build apk --debug --dart-define-from-file=.env.json
```

That was all you needed to pull an APK out of the repo. But the test folder still required a few manual steps:

1. Run the command from the root folder.  
2. Move `build/app/outputs/flutter-apk/app-debug.apk` to the device—via email, Google Drive, or ADB.  
3. Turn on “Install unknown apps” for each phone.  
4. Restart the phone if the install failed due to Play Store protection.

A documented, repeatable process would have saved us half an hour for each device—time that could go straight into testing the AI features.

## Refining the Approach: Makefile + Documentation

We realized the bottleneck was the tooling, not Flutter itself. If we could make the command line feel more familiar, QA users who aren’t command‑line experts would jump in faster.

### The Makefile: One‑Command Build Targets

We added a `Makefile` with clear, named targets:

```makefile
.PHONY: apk apk-debug apk-release install clean

apk: apk-debug

apk-debug:
	flutter build apk --debug --dart-define-from-file=.env.json

apk-release:
	flutter build apk --release --dart-define-from-file=.env.json

install:
	flutter install --debug

clean:
	flutter clean && flutter pub get
```

*Why `make` instead of shell scripts?*  
- It works the same on Linux, macOS, and Windows (via MSYS2).  
- Target names say what they do at a glance.  
- Adding a new target, like `apk-signed`, is a one‑liner.

### Updating the README

We put the most common instructions right next to the description:

```markdown
## Building APK for Testing

Build a debug APK that you can sideload onto a real Android device:

```bash
flutter build apk --debug --dart-define-from-file=.env.json
```

The APK is in `build/app/outputs/flutter-apk/app-debug.apk`.

## Makefile shortcuts

```
make apk          # builds a debug APK  
make apk-release  # builds a release (debug‑signed) APK  
make install      # installs the app on a connected device  
make clean        # cleans the build and refreshes dependencies
```
```

### Enriching the CLI Help (`CLAUDE.md`)

The file that AI models browse to figure out how to run the app now contains direct build commands:

```
+ flutter build apk --debug --dart-define-from-file=.env.json   # debug APK for testing
+ flutter build apk --release --dart-define-from-file=.env.json  # release (debug‑signed) APK
+ flutter install --debug                                         # install on connected device
```

Someone reading the file can see the most important commands right away.

### A New Resource: `INSTALL_APK.md`

The real breakthrough was a stand‑alone guide for Android testers. It walks through each step—from building to enabling “unknown sources”, handling Play Protect, and solving common errors. Highlights:

- Build with `make apk-release` (it’s a debug‑signed release ready for sideloading).  
- Transfer by email, Drive, USB, or a messaging app.  
- Step‑by‑step instructions for enabling “unknown sources” on different Android versions.  
- Troubleshooting tips for “App not installed” and Play Store warnings.

Instead of asking “Where do I find the APK?”, the guide now says “Here’s how to install it”.

## Decision Points and Trade‑offs

| Decision | Reasoning |
|----------|-----------|
| **Makefile over shell scripts** | Reads the same everywhere; no OS quirks. |
| **Add an `install` target** | Removes the need to remember `adb install`. |
| **Provide both debug and release builds** | Debug builds aid quick debugging; release builds look cleaner for testers. |
| **Dedicated `INSTALL_APK.md` doc** | Keeps the README focused while giving testers a single place to look. |
| **Stick to debug‑signed release** | Avoids signing headaches for a non‑Store distribution. |
| **Add a `clean` target** | Makes CI runs predictable and fast. |

We chose the debug‑signed release because the version was only for internal testing and the licensing model (CC BY‑NC‑SA) does not allow commercial Play Store distribution. A store‑ready build would have meant managing a keystore, signing every time, and checking every API call for license compliance.

## The Minor but Meaningful Add‑on: `frank-state.db`

The commit that swapped out a binary `frank-state.db` might look trivial, but it reflects a good practice: keep local state snapshots in sync with code changes so future updates don’t break persisted state. Since the file isn’t part of the public API, we skip the heavy CI job for it, but it reminds us to version synthetic state files.

## Lessons Learned

1. **Build isn’t build, deploy is** – Building an APK is easy; installing it on a phone isn’t. A documented install flow cuts manual errors and speeds QA.  
2. **Makefiles keep the repo tidy** – One command does a lot. Adding new targets (e.g., Play Store signing) is just a couple of lines.  
3. **Separate docs lower friction** – Testers get a single, focused guide instead of hunting the README.  
4. **Be upfront about build types** – `make apk-release` is debug‑signed; that matters for licensing.  
5. **Tiny changes matter** – Updating a one‑liner in the README can turn a “works on my phone” loop into a “works on every phone” loop.  
6. **Unify local and CI workflows** – The same Makefile targets run for developers and in the CI pipeline.  
7. **Show your expectations** – Adding the `install` target and documenting it in `CLAUDE.md` tells people what to run, instead of leaving it to guesswork.

## Conclusion

What began as a handful of flutter commands evolved into a tidy workflow: a clear Makefile, a purpose‑built install guide, and consistent documentation. The team moved from “how do I build?” to “how do I put this app on a phone?”. The difference in time and frustration is measurable. The next step for Smart Wardrobe will be a production‑ready signing process and a continuous‑delivery pipeline that keeps the testing ethos while preparing for public distribution.

If you’re building a Flutter app that needs to land on real devices, question whether the build process is also the source of friction. Often, a single file, a simple command, and a focused guide can make all the difference.
