# apple-oss-scaffold

A Claude Code skill for scaffolding, building, and releasing open-source Apple platform apps (macOS and iOS) using XcodeGen.

## What It Does

Claude automatically uses this skill when you're working on Apple open-source projects. It covers:

- Setting up a new macOS or iOS project with XcodeGen (no committed `.xcodeproj`)
- Generating AppIcon sizes from a single 1024×1024 PNG using `sips`
- Creating a polished DMG installer for macOS apps with a custom background and volume icon
- Working around the macOS 15 DMG background rendering bug
- Cutting GitHub Releases with the `gh` CLI
- Setting up GitHub Actions CI for both platforms

This skill covers the **unsigned pre-1.0 / open-source path**. For notarized releases with Sparkle auto-updates, use the `releasing-macos-apps` skill instead.

## Usage

Once installed, Claude activates this skill automatically when you:

- Ask to set up a new macOS or iOS project for open-source distribution
- Ask about `project.yml`, XcodeGen, or replacing `.xcodeproj`
- Ask to create a DMG, cut a release, or set up GitHub Actions CI
- Ask about `create-dmg`, `VolumeIcon.icns`, DMG backgrounds, or AppIcon generation
- Mention unsigned or pre-1.0 Apple app releases

## Examples

```
"Set up a new macOS app with XcodeGen"
"Create a DMG installer for my app"
"Cut a GitHub Release for version 0.9.2"
"Generate AppIcon sizes from my 1024px PNG"
"Set up CI for my iOS open-source project"
```

## Installation

### Via claude-plugins-official

Install from the Claude Code plugin registry.

### Manual

Copy the `skills/` directory into your `.claude/` folder:

```bash
cp -r skills/apple-oss-scaffold ~/.claude/skills/
```

## Skill Contents

```
skills/
└── apple-oss-scaffold/
    ├── SKILL.md                    — main skill (loaded by Claude)
    └── references/
        ├── macos-setup.md          — project.yml template, sips icon generation,
        │                             DMG background, release.sh template, CI workflow
        ├── macos-release.md        — version bump checklist, macOS 15 DMG workaround,
        │                             GitHub Release steps, CHANGELOG format
        └── ios-setup.md            — iOS project.yml template, AppIcon generation,
                                      CI workflow, release options
```

## License

MIT
