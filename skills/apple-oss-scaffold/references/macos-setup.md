# macOS Setup — Full Reference

## project.yml Template

```yaml
name: AppName
options:
  bundleIdPrefix: com.yourname
  deploymentTarget:
    macOS: "15.6"
settings:
  base:
    SWIFT_VERSION: "5.0"
    MACOSX_DEPLOYMENT_TARGET: "15.6"
targets:
  AppName:
    type: application
    platform: macOS
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
        INFOPLIST_KEY_LSApplicationCategoryType: "public.app-category.utilities"
        INFOPLIST_KEY_LSUIElement: YES          # omit if you want a Dock icon
        LD_RUNPATH_SEARCH_PATHS: "$(inherited) @executable_path/../Frameworks"
```

**Key fields:**
- `INFOPLIST_KEY_LSUIElement: YES` — hides app from Dock (menu bar apps only)
- `DEVELOPMENT_TEAM` — find in Xcode → Signing & Capabilities, or `security find-identity -p codesigning`
- `MARKETING_VERSION` — the only version field you need to bump for a release

After writing `project.yml`, regenerate:
```bash
xcodegen generate
```

---

## AppIcon Generation with `sips`

Start with one **1024×1024 PNG** (e.g. `AppIcon-1024.png`). Generate all required macOS sizes:

```bash
ICON_SRC="AppIcon-1024.png"
ICONSET="AppIcon.iconset"
mkdir -p "$ICONSET"

sips -z 16 16     "$ICON_SRC" --out "$ICONSET/icon_16x16.png"
sips -z 32 32     "$ICON_SRC" --out "$ICONSET/icon_16x16@2x.png"
sips -z 32 32     "$ICON_SRC" --out "$ICONSET/icon_32x32.png"
sips -z 64 64     "$ICON_SRC" --out "$ICONSET/icon_32x32@2x.png"
sips -z 128 128   "$ICON_SRC" --out "$ICONSET/icon_128x128.png"
sips -z 256 256   "$ICON_SRC" --out "$ICONSET/icon_128x128@2x.png"
sips -z 256 256   "$ICON_SRC" --out "$ICONSET/icon_256x256.png"
sips -z 512 512   "$ICON_SRC" --out "$ICONSET/icon_256x256@2x.png"
sips -z 512 512   "$ICON_SRC" --out "$ICONSET/icon_512x512.png"
cp "$ICON_SRC"        "$ICONSET/icon_512x512@2x.png"
```

Then drag each file into the correct slot in `Assets.xcassets/AppIcon.appiconset/` in Xcode, or copy them there directly and update `Contents.json`.

---

## VolumeIcon.icns for DMG

The DMG volume icon (the disk image icon that appears in Finder when mounted) should be generated from the same 1024×1024 source:

```bash
ICON_SRC="AppIcon-1024.png"
ICONSET="VolumeIcon.iconset"
mkdir -p "$ICONSET"

sips -z 16 16     "$ICON_SRC" --out "$ICONSET/icon_16x16.png"
sips -z 32 32     "$ICON_SRC" --out "$ICONSET/icon_16x16@2x.png"
sips -z 32 32     "$ICON_SRC" --out "$ICONSET/icon_32x32.png"
sips -z 64 64     "$ICON_SRC" --out "$ICONSET/icon_32x32@2x.png"
sips -z 128 128   "$ICON_SRC" --out "$ICONSET/icon_128x128.png"
sips -z 256 256   "$ICON_SRC" --out "$ICONSET/icon_128x128@2x.png"
sips -z 256 256   "$ICON_SRC" --out "$ICONSET/icon_256x256.png"
sips -z 512 512   "$ICON_SRC" --out "$ICONSET/icon_256x256@2x.png"
sips -z 512 512   "$ICON_SRC" --out "$ICONSET/icon_512x512.png"
cp "$ICON_SRC"        "$ICONSET/icon_512x512@2x.png"

iconutil -c icns "$ICONSET" -o scripts/VolumeIcon.icns
rm -rf "$ICONSET"
```

Place `VolumeIcon.icns` in `scripts/` — that's where `release.sh` expects it.

---

## DMG Background

Design a **800×500px** PNG in Figma (or any tool). Export:
- `dmg-background.png` (1x, 800×500)
- `dmg-background@2x.png` (2x, 1600×1000)

Place both in `scripts/`. The release script will embed them into the DMG's hidden `.background/` folder.

Guidelines for the background design:
- Put the app icon drop zone on the left (~185px from left, vertically centered)
- Put the Applications alias on the right (~610px from left, vertically centered)
- Arrow or visual cue pointing left→right helps users understand the drag gesture
- Keep it subtle — dark or light gradient, or a simple branded color wash

---

## release.sh Template

Save to `scripts/release.sh` and `chmod +x scripts/release.sh`.

```bash
#!/bin/bash
set -e

APP_NAME="AppName"
SCHEME="AppName"
PROJECT="AppName.xcodeproj"
RELEASE_DIR="release"
SCRIPTS_DIR="scripts"
BACKGROUND_IMAGE="$SCRIPTS_DIR/dmg-background.png"
VOLUME_ICON="$SCRIPTS_DIR/VolumeIcon.icns"

# ── Step 1: Read version ────────────────────────────────────────────────────
echo "→ Reading version from project..."
xcodegen generate --quiet
VERSION=$(xcodebuild -project "$PROJECT" -scheme "$SCHEME" \
  -showBuildSettings 2>/dev/null | grep -w MARKETING_VERSION | awk '{print $3}')
DMG_NAME="${APP_NAME}-${VERSION}.dmg"
VOLUME_NAME="${APP_NAME} ${VERSION}"
echo "  Version: $VERSION"

# ── Step 2: Archive ─────────────────────────────────────────────────────────
ARCHIVE_PATH="$RELEASE_DIR/$APP_NAME.xcarchive"
mkdir -p "$RELEASE_DIR"
echo "→ Archiving..."
xcodebuild archive \
  -project "$PROJECT" \
  -scheme "$SCHEME" \
  -configuration Release \
  -archivePath "$ARCHIVE_PATH" \
  CODE_SIGNING_ALLOWED=NO \
  -quiet

APP_PATH="$ARCHIVE_PATH/Products/Applications/$APP_NAME.app"

# ── Step 3: Create DMG (structure only) ─────────────────────────────────────
echo "→ Creating DMG..."
create-dmg \
  --volname "$VOLUME_NAME" \
  --volicon "$VOLUME_ICON" \
  --background "$BACKGROUND_IMAGE" \
  --window-pos 200 120 \
  --window-size 800 500 \
  --icon-size 128 \
  --icon "$APP_NAME.app" 185 245 \
  --app-drop-link 610 245 \
  --no-internet-enable \
  "$RELEASE_DIR/$DMG_NAME" \
  "$APP_PATH"

# ── Step 4: macOS 15 background workaround ──────────────────────────────────
# create-dmg's AppleScript silently fails to set the background on macOS 15.
# Fix: convert to read-write, mount, run AppleScript, detach, recompress.
echo "→ Applying background (macOS 15 workaround)..."
RW_DMG="$RELEASE_DIR/${APP_NAME}-rw.dmg"
MOUNT_POINT="/Volumes/${APP_NAME}RW"

hdiutil detach "/Volumes/$APP_NAME" -quiet 2>/dev/null || true
hdiutil convert "$RELEASE_DIR/$DMG_NAME" -format UDRW -o "$RW_DMG" -quiet
hdiutil attach "$RW_DMG" -mountpoint "$MOUNT_POINT" -quiet

osascript << APPLESCRIPT
tell application "Finder"
  tell disk "$VOLUME_NAME"
    open
    delay 2
    tell container window
      set current view to icon view
      set toolbar visible to false
      set statusbar visible to false
      set bounds to {200, 120, 1000, 620}
      tell its icon view options
        set arrangement to not arranged
        set icon size to 128
        set background picture to (POSIX file "$MOUNT_POINT/.background/dmg-background.png" as alias)
      end tell
      set position of item "$APP_NAME.app" to {185, 245}
      set position of item "Applications" to {610, 245}
    end tell
    update without registering applications
    delay 5
    close
  end tell
end tell
APPLESCRIPT

hdiutil detach "$MOUNT_POINT" -quiet
rm -f "$RELEASE_DIR/$DMG_NAME"
hdiutil convert "$RW_DMG" -format UDZO -imagekey zlib-level=9 \
  -o "$RELEASE_DIR/$DMG_NAME" -quiet
rm -f "$RW_DMG"

echo ""
echo "✓ Done: $RELEASE_DIR/$DMG_NAME"
```

**Required tools:**
```bash
brew install xcodegen create-dmg
```

---

## docs/ scaffold

```
README.md       — installation, usage, screenshots, Gatekeeper note
CHANGELOG.md    — Keep a Changelog format (https://keepachangelog.com)
LICENSE         — MIT
SECURITY.md     — vulnerability disclosure contact
```

**Gatekeeper note for README** (unsigned apps):

```markdown
> **Note:** This app is not yet notarized. On first launch, right-click the app
> and choose **Open**, then click **Open** in the dialog. You only need to do this once.
```

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
            -destination "platform=macOS" \
            CODE_SIGNING_ALLOWED=NO \
            build
```
