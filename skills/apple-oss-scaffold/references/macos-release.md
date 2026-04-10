# macOS Release — Full Reference

## Version Bump Checklist

1. Edit `MARKETING_VERSION` in `project.yml` (single source of truth)
2. Add entry to `CHANGELOG.md`
3. Commit both files: `git commit -m "chore: bump version to X.X.X"`
4. Push: `git push`
5. Run `./scripts/release.sh` → produces `release/AppName-X.X.X.dmg`
6. Create GitHub Release (see below)
7. Update any website `CURRENT_VERSION` constant

**That's the entire loop.** Everything else is automated by `release.sh`.

---

## What release.sh Does

1. Calls `xcodegen generate` to ensure the project reflects the latest `project.yml`
2. Reads `MARKETING_VERSION` from xcodebuild settings (no hardcoding)
3. Archives the app unsigned (`CODE_SIGNING_ALLOWED=NO`)
4. Runs `create-dmg` to build the DMG structure with background, volume icon, and icon positions
5. Applies the **macOS 15 background workaround** (see below)
6. Outputs `release/AppName-X.X.X.dmg`

The `release/` directory is git-ignored — DMGs never get committed.

---

## macOS 15 DMG Background Workaround

On macOS 15, `create-dmg`'s internal AppleScript call to set the background image silently fails. The DMG opens but shows a plain white/default background.

**Root cause:** `create-dmg` mounts the DMG read-only when running the AppleScript, so Finder can't persist the window settings.

**Fix (baked into release.sh):**

```
create-dmg runs → produces a DMG without working background
     ↓
hdiutil convert → UDRW (read-write writable image)
     ↓
hdiutil attach → mount at /Volumes/AppNameRW
     ↓
osascript → tell Finder's disk window to set background + icon positions
     ↓
hdiutil detach → unmount
     ↓
hdiutil convert → UDZO (compressed final DMG)
     ↓
remove the intermediate UDRW
```

The AppleScript sets:
- `current view to icon view`
- `background picture` to `.background/dmg-background.png` inside the mounted volume
- Icon positions for the app and the Applications alias
- Window bounds (matches the `--window-size` passed to `create-dmg`)

This is already implemented in the `release.sh` template in `references/macos-setup.md`.

---

## Verifying the DMG

After `release.sh` completes:
```bash
open release/AppName-X.X.X.dmg
```

Check:
- Background image appears (not plain white)
- App icon and Applications alias are in the correct positions
- Window is the right size
- Volume icon (in Finder sidebar) matches the app icon

If the background is missing, the most common cause is the AppleScript `delay` values being too short for a slow machine. Increase `delay 2` and `delay 5` in `release.sh`.

---

## Creating a GitHub Release

**Using `gh` CLI:**
```bash
VERSION="0.9.2"
APP_NAME="AppName"

gh release create "v${VERSION}" \
  "release/${APP_NAME}-${VERSION}.dmg" \
  --title "${APP_NAME} ${VERSION}" \
  --notes-file CHANGELOG_EXCERPT.md
```

Or create interactively:
```bash
gh release create "v${VERSION}" "release/${APP_NAME}-${VERSION}.dmg"
```

**Via GitHub web:**
1. Go to the repo → Releases → Draft a new release
2. Tag: `vX.X.X` (create new tag on publish)
3. Title: `AppName X.X.X`
4. Body: paste from CHANGELOG.md for this version
5. Attach the `.dmg` file
6. Publish release

---

## Download URL Pattern

Once the release is published, the DMG download URL is predictable:

```
https://github.com/{username}/{repo}/releases/download/v{version}/{AppName}-{version}.dmg
```

Example:
```
https://github.com/username/MyApp/releases/download/v0.9.2/MyApp-0.9.2.dmg
```

Use this URL in your website's download button, README badge, etc.

---

## Release Notes Template

Keep a local `RELEASE_NOTES.md` (git-ignored) with project-specific notes. For GitHub Release body, pull from `CHANGELOG.md`:

```markdown
## What's New in X.X.X

### Added
- Feature one
- Feature two

### Fixed
- Bug one

---

**System Requirements:** macOS 15.0 or later

> **First launch:** Right-click the app → Open → Open (Gatekeeper notice for unsigned apps)

[Full changelog](https://github.com/user/repo/blob/main/CHANGELOG.md)
```

---

## CHANGELOG.md Format

Follow [Keep a Changelog](https://keepachangelog.com/en/1.0.0/):

```markdown
# Changelog

## [Unreleased]

## [0.9.2] - 2026-04-10
### Added
- New feature

### Fixed
- Bug fix

## [0.9.1] - 2026-03-15
...
```

---

## Notarization (Post-1.0)

This skill covers **unsigned** releases. For notarized releases with Sparkle auto-updates, use the `releasing-macos-apps` skill instead.

For notarization you'll need:
- Apple Developer Program membership ($99/yr)
- App-specific password for notarytool
- Signed + hardened runtime build
- `xcrun notarytool submit` + `xcrun stapler staple`

Plan for this at 1.0, not before.
