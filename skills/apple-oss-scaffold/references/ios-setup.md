# iOS Setup — Full Reference

## project.yml Template

```yaml
name: AppName
options:
  bundleIdPrefix: com.yourname
  deploymentTarget:
    iOS: "17.0"
settings:
  base:
    SWIFT_VERSION: "5.0"
    IPHONEOS_DEPLOYMENT_TARGET: "17.0"
targets:
  AppName:
    type: application
    platform: iOS
    sources:
      - path: AppName
        excludes: ["**/.DS_Store"]
    settings:
      base:
        PRODUCT_BUNDLE_IDENTIFIER: com.yourname.AppName
        MARKETING_VERSION: "0.9.0"
        CURRENT_PROJECT_VERSION: "1"
        CODE_SIGN_STYLE: Automatic
        DEVELOPMENT_TEAM: YOUR_TEAM_ID
        ASSETCATALOG_COMPILER_APPICON_NAME: AppIcon
        GENERATE_INFOPLIST_FILE: YES
        INFOPLIST_KEY_CFBundleDisplayName: "App Name"
        INFOPLIST_KEY_UILaunchScreen_Generation: YES
        INFOPLIST_KEY_UISupportedInterfaceOrientations_iPhone: >-
          UIInterfaceOrientationPortrait
        INFOPLIST_KEY_UISupportedInterfaceOrientations_iPad: >-
          UIInterfaceOrientationPortrait
          UIInterfaceOrientationPortraitUpsideDown
          UIInterfaceOrientationLandscapeLeft
          UIInterfaceOrientationLandscapeRight
```

**Key differences from macOS:**
- `platform: iOS` instead of `macOS`
- `IPHONEOS_DEPLOYMENT_TARGET` instead of `MACOSX_DEPLOYMENT_TARGET`
- No `INFOPLIST_KEY_LSUIElement` (that's macOS-only)
- Launch screen and orientation keys instead of menu bar keys
- No `create-dmg` or DMG tooling needed

After writing `project.yml`:
```bash
xcodegen generate
open AppName.xcodeproj
```

---

## AppIcon Generation

iOS requires more icon sizes than macOS. Generate from a **1024×1024 PNG**:

```bash
ICON_SRC="AppIcon-1024.png"
ICONSET_DIR="AppName/Assets.xcassets/AppIcon.appiconset"
mkdir -p "$ICONSET_DIR"

# iPhone
sips -z 40 40    "$ICON_SRC" --out "$ICONSET_DIR/Icon-20@2x.png"
sips -z 60 60    "$ICON_SRC" --out "$ICONSET_DIR/Icon-20@3x.png"
sips -z 58 58    "$ICON_SRC" --out "$ICONSET_DIR/Icon-29@2x.png"
sips -z 87 87    "$ICON_SRC" --out "$ICONSET_DIR/Icon-29@3x.png"
sips -z 76 76    "$ICON_SRC" --out "$ICONSET_DIR/Icon-38@2x.png"
sips -z 114 114  "$ICON_SRC" --out "$ICONSET_DIR/Icon-38@3x.png"
sips -z 80 80    "$ICON_SRC" --out "$ICONSET_DIR/Icon-40@2x.png"
sips -z 120 120  "$ICON_SRC" --out "$ICONSET_DIR/Icon-40@3x.png"
sips -z 120 120  "$ICON_SRC" --out "$ICONSET_DIR/Icon-60@2x.png"
sips -z 180 180  "$ICON_SRC" --out "$ICONSET_DIR/Icon-60@3x.png"
# iPad
sips -z 20 20    "$ICON_SRC" --out "$ICONSET_DIR/Icon-20.png"
sips -z 40 40    "$ICON_SRC" --out "$ICONSET_DIR/Icon-20@2x-ipad.png"
sips -z 29 29    "$ICON_SRC" --out "$ICONSET_DIR/Icon-29.png"
sips -z 58 58    "$ICON_SRC" --out "$ICONSET_DIR/Icon-29@2x-ipad.png"
sips -z 40 40    "$ICON_SRC" --out "$ICONSET_DIR/Icon-40.png"
sips -z 80 80    "$ICON_SRC" --out "$ICONSET_DIR/Icon-40@2x-ipad.png"
sips -z 76 76    "$ICON_SRC" --out "$ICONSET_DIR/Icon-76.png"
sips -z 152 152  "$ICON_SRC" --out "$ICONSET_DIR/Icon-76@2x.png"
sips -z 167 167  "$ICON_SRC" --out "$ICONSET_DIR/Icon-83.5@2x.png"
# App Store
cp "$ICON_SRC"       "$ICONSET_DIR/Icon-1024.png"
```

Then update `Contents.json` in the iconset to reference these filenames, or use Xcode to drag them into the AppIcon slots.

**Simpler alternative:** Use [makeappicon.com](https://makeappicon.com) or [IconKitchen](https://icon.kitchen) to generate the full set from your 1024×1024 PNG.

---

## CI Workflow

`.github/workflows/ci.yml`:

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
            -destination "generic/platform=iOS Simulator" \
            CODE_SIGNING_ALLOWED=NO \
            build
```

**Note:** iOS builds use `generic/platform=iOS Simulator` as the destination for CI. You cannot build for a real device without signing on CI without additional setup.

---

## iOS Release Options

### Source-Only (Open Source Default)

Tag the release on GitHub. Contributors clone and build themselves:

```bash
VERSION="0.9.0"
APP_NAME="AppName"

gh release create "v${VERSION}" \
  --title "${APP_NAME} ${VERSION}" \
  --notes "$(cat CHANGELOG.md)"
```

Write clear build instructions in README:
```markdown
## Building from Source

1. Clone the repo
2. Install XcodeGen: `brew install xcodegen`
3. Generate the project: `xcodegen generate`
4. Open `AppName.xcodeproj` in Xcode
5. Select your team in Signing & Capabilities
6. Build and run (⌘R)
```

### TestFlight

Archive manually in Xcode (Product → Archive → Distribute App → TestFlight) or use Fastlane for automation. Requires Apple Developer Program membership.

### App Store

Out of scope for this skill — use Fastlane or the manual Xcode workflow. Consider the `releasing-macos-apps` skill for patterns that also apply to iOS App Store distribution.

---

## Version Bump (iOS)

Same pattern as macOS — only `MARKETING_VERSION` in `project.yml` needs to change:

1. Edit `MARKETING_VERSION` in `project.yml`
2. Update `CHANGELOG.md`
3. Commit + tag
4. Create GitHub Release (source-only) or Archive for TestFlight/App Store

---

## Common iOS-Specific project.yml Settings

```yaml
# SwiftData / Core Data
INFOPLIST_KEY_NSPersistentStoreUsesNSHistoryTrackingOption: YES

# Camera access
INFOPLIST_KEY_NSCameraUsageDescription: "Used to scan QR codes."

# Location (always-on, not needed for most apps)
INFOPLIST_KEY_NSLocationWhenInUseUsageDescription: "Used to show nearby results."

# Face ID
INFOPLIST_KEY_NSFaceIDUsageDescription: "Used to authenticate."

# Background modes (add only what you need)
INFOPLIST_KEY_UIBackgroundModes: fetch remote-notification

# Hide status bar
INFOPLIST_KEY_UIStatusBarHidden: YES

# Require full screen on iPad
INFOPLIST_KEY_UIRequiresFullScreen: YES
```

---

## Docs scaffold (same as macOS)

```
README.md       — build instructions, screenshots, requirements
CHANGELOG.md    — Keep a Changelog format
LICENSE         — MIT
SECURITY.md     — vulnerability disclosure contact
```

No Gatekeeper note needed for iOS (TestFlight and App Store handle trust).
