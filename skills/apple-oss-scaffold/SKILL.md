---
name: apple-oss-scaffold
description: Scaffold, build, and release open-source Apple platform apps (macOS and iOS) using XcodeGen. Use this skill whenever the user wants to set up a new macOS or iOS project for open-source distribution, wire up XcodeGen to replace .xcodeproj, create a DMG installer for a macOS app, cut a GitHub Release, generate AppIcon sizes, generate a DMG volume icon, set up CI with GitHub Actions, or follow an unsigned pre-1.0 release workflow. Also use when the user asks about project.yml, create-dmg, VolumeIcon.icns, DMG backgrounds, or XcodeGen build settings for Apple apps.
version: 1.0.0
license: MIT
---

# Apple Open-Source Scaffold

Covers two phases, two platforms. Jump to what you need:

| | macOS | iOS |
|---|---|---|
| **New project setup** | [macOS Setup](#macos-setup) | [iOS Setup](#ios-setup) |
| **Cut a release** | [macOS Release](#macos-release) | [iOS Release](#ios-release) |

For **notarized** macOS releases with Sparkle auto-updates, use the `releasing-macos-apps` skill instead — this skill covers the unsigned pre-1.0 / open-source path.

---

## Decision Tree

Before starting, answer three questions:

1. **New project or existing?**
   - New → run full setup first, then release
   - Existing → skip to release section

2. **macOS or iOS?**
   - macOS → DMG + GitHub Releases
   - iOS → source-only or TestFlight

3. **XcodeGen already set up?**
   - No → always set it up first — it's the foundation everything else depends on
   - Yes → skip to the relevant release section

---

## XcodeGen Foundation (Both Platforms)

Every project starts here regardless of platform.

**Install:**
```bash
brew install xcodegen create-dmg  # create-dmg only needed for macOS
```

**`.gitignore` — always include:**
```
*.xcodeproj
*.xcworkspace
DerivedData/
xcuserdata/
.build/
build/
release/
.claude/
RELEASE_NOTES.md
```

**Repo scaffold to create at root:**
```
project.yml          ← XcodeGen config (source of truth)
.gitignore
README.md
CHANGELOG.md         ← Keep a Changelog format
LICENSE              ← MIT
SECURITY.md
.github/workflows/ci.yml
scripts/             ← release tooling
```

**Never commit `.xcodeproj`** — regenerate with `xcodegen generate` on every clone.

**CI workflow (`.github/workflows/ci.yml`):**
```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
jobs:
  build:
    runs-on: macos-15
    steps:
      - uses: actions/checkout@v4
      - name: Install XcodeGen
        run: brew install xcodegen
      - name: Generate project
        run: xcodegen generate
      - name: Build
        run: |
          xcodebuild \
            -project AppName.xcodeproj \
            -scheme AppName \
            -configuration Debug \
            -destination "platform=macOS" \
            CODE_SIGNING_ALLOWED=NO \
            build
```

---

## macOS Setup

See full details in `references/macos-setup.md`.

**Quick path:**
1. Write `project.yml` (template in reference file)
2. `xcodegen generate`
3. Generate AppIcon sizes with `sips` from one 1024×1024 PNG
4. Generate `VolumeIcon.icns` for the DMG with `iconutil`
5. Design DMG background (800×500px PNG + @2x)
6. Write `scripts/release.sh` (template in reference file)
7. Scaffold docs and CI

---

## macOS Release

See full details in `references/macos-release.md`.

**Quick path:**
1. Bump `MARKETING_VERSION` in `project.yml`
2. Update `CHANGELOG.md`
3. Commit + push
4. `./scripts/release.sh` → `release/AppName-X.X.X.dmg`
5. GitHub Release: tag `vX.X.X`, attach DMG
6. Update website `CURRENT_VERSION`

**Download URL pattern:**
```
https://github.com/{user}/{repo}/releases/download/v{version}/{AppName}-{version}.dmg
```

---

## iOS Setup

See full details in `references/ios-setup.md`.

**Quick path:**
1. Write `project.yml` with `platform: iOS`
2. `xcodegen generate`
3. Scaffold docs and CI (no DMG tooling needed)

---

## iOS Release

iOS open-source projects typically ship source-only — contributors build themselves. For distribution:

- **Source-only:** Tag the release, write good build instructions in README
- **TestFlight:** Archive → distribute via App Store Connect (manual or Fastlane)
- **App Store:** Out of scope for this skill — use Fastlane or manual Xcode workflow

**GitHub Release for iOS (source-only):**
```bash
gh release create vX.X.X \
  --title "AppName X.X.X" \
  --notes "$(cat CHANGELOG.md | sed -n '/## \[X.X.X\]/,/## \[/p' | head -n -1)"
```

---

## Common Tasks

**Regenerate project after any `project.yml` change:**
```bash
xcodegen generate
```

**Bump version (both platforms):**
Edit `MARKETING_VERSION` in `project.yml` — single source of truth.

**Verify build settings:**
```bash
xcodebuild -project AppName.xcodeproj \
  -scheme AppName \
  -showBuildSettings | grep MARKETING_VERSION
```
