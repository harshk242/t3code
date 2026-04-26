# Fork Upstream Merge Guide

This repo is a fork of [pingdotgg/t3code](https://github.com/pingdotgg/t3code) with Linux desktop (`.deb`) customizations. This document guides Claude (or a human) through merging upstream changes without losing fork-specific work.

## Remotes

```
origin    git@github.com:harshk242/t3code.git    (fork)
upstream  https://github.com/pingdotgg/t3code.git (source)
```

## Fork-Specific Changes

These are the custom changes maintained in this fork. **All exist because upstream only targets macOS (dmg) and Linux AppImage; this fork adds `.deb` packaging and GNOME desktop integration.**

### 1. Multi-size Linux icon generation (`scripts/build-desktop-artifact.ts`)

- **What**: `stageLinuxIcons()` generates 16/32/48/64/128/256/512px PNGs via ImageMagick `convert`, plus a root `icon.png` for Electron's runtime window icon. Build config uses `icon: "icons"` (directory) instead of `icon: "icon.png"` (single file).
- **Why**: electron-builder installs icons into `hicolor/` for the `.desktop` entry. A single high-res PNG only creates one size (e.g. 1024x1024) which GNOME ignores -- it needs standard sizes. AppImage doesn't have this problem because it embeds icons differently.
- **Requires**: `imagemagick` (`convert` command) on the build machine.

### 2. Deb build metadata (`scripts/build-desktop-artifact.ts`)

- **What**: `StagePackageJson` interface and construction include `homepage` and structured `author` (with email). Linux build config includes `maintainer`.
- **Why**: electron-builder's FPM/deb target requires these fields and will error without them. AppImage and dmg targets don't need them, so upstream omits them.

### 3. `dist:desktop:deb` script (`package.json`)

- **What**: `"dist:desktop:deb": "node scripts/build-desktop-artifact.ts --platform linux --target deb --arch x64"`
- **Why**: Convenience script to build the `.deb` package. Upstream only has `dist:desktop:linux` (AppImage).

### 4. GNOME dock icon and pinning (`apps/desktop/src/main.ts`)

- **What**: Sets `process.env.CHROME_DESKTOP = LINUX_DESKTOP_ENTRY_NAME` and uses `app.setName(LINUX_WM_CLASS)` on Linux instead of `APP_DISPLAY_NAME`.
- **Why**: GNOME matches running windows to `.desktop` entries using WM_CLASS and the `CHROME_DESKTOP` env var (Chromium/Electron convention). Without these, the app won't pin to the dock correctly and shows a generic icon in the taskbar.

### 5. Debugging docs (`AGENTS.md`)

- **What**: A "Debugging" section noting that logs are at `~/.t3/userdata/logs/`.
- **Why**: Useful reference for development. Upstream doesn't include this.

### 6. Linux build docs (`docs/linux-build.md`)

- **What**: Prerequisites, build commands, and install instructions for Ubuntu/Linux.
- **Why**: Upstream doesn't document Linux desktop builds.

## Merge Procedure

```bash
git fetch upstream
git merge upstream/main
# resolve conflicts
# rebuild: bun install && bun run dist:desktop:deb --verbose
# install: sudo dpkg -i release/T3-Code-*.deb
# verify: icon shows in dock, app launches, pinning works
```

## Conflict Resolution Decision Framework

When a merge conflict involves a fork-changed file, ask these questions **in order**:

### Step 1: Did upstream delete or replace the file?

If yes: check whether the replacement covers the fork's intent.

| Example | Resolution |
|---|---|
| `fixPath.ts` removed, replaced by `syncShellEnvironment.ts` that handles Linux | **Drop fork change** -- upstream solved it differently |
| `bootstrap.ts` ENXIO catch, but upstream added `isBootstrapFdPathDuplicationError` covering ENXIO | **Drop fork change** -- upstream's solution is more general |

### Step 2: Did upstream rewrite the surrounding code?

If upstream restructured function signatures, moved parameters, or changed architecture:

- **Adapt the fork change to the new code**, don't force the old version back.
- Example: `stageLinuxIcons` changed from resolving icon sources internally to taking a `sourcePng` parameter. Adapt the multi-size generation to accept the parameter rather than reverting upstream's refactor.

### Step 3: Is the fork change deb-specific metadata?

Fields like `homepage`, `author.email`, `maintainer`, `desktopName` -- upstream will drop these because they don't build debs.

- **Always re-add them.** They are required by electron-builder's FPM target.
- If upstream changed the `StagePackageJson` interface or `createBuildConfig`, add the fields to the new structure.

### Step 4: Is it a Linux desktop integration change?

`CHROME_DESKTOP`, `app.setName(LINUX_WM_CLASS)`, `icon: "icons"` -- upstream doesn't test on GNOME/Ubuntu and may inadvertently revert these.

- **Keep fork changes** unless upstream has explicitly added equivalent GNOME integration (check their PRs/commit messages for "GNOME", "dock", "WM_CLASS", "desktop entry").
- These typically auto-merge cleanly since upstream rarely touches the same lines.

### Step 5: Is it documentation?

`AGENTS.md` debugging section, `docs/linux-build.md` -- upstream may change surrounding content but won't add these.

- **Keep fork changes.** They auto-merge in most cases.

## Quick Reference: Conflict Resolution Table

| File / Area | Fork adds | Upstream tendency | Resolution |
|---|---|---|---|
| `scripts/build-desktop-artifact.ts` : `stageLinuxIcons` | Multi-size icon gen + root `icon.png` | Single `icon.png` copy | **Keep fork** (adapt to new signatures) |
| `scripts/build-desktop-artifact.ts` : linux build config | `icon: "icons"`, `maintainer` | `icon: "icon.png"`, no maintainer | **Keep fork** |
| `scripts/build-desktop-artifact.ts` : `StagePackageJson` | `homepage`, `author` with email | No homepage, author as string | **Keep fork** |
| `package.json` : scripts | `dist:desktop:deb` | Will be absent | **Re-add after upstream lines** |
| `apps/desktop/src/main.ts` : Linux block | `CHROME_DESKTOP` env var | Won't have it | **Keep fork** (usually auto-merges) |
| `apps/desktop/src/main.ts` : `configureAppIdentity` | `setName(LINUX_WM_CLASS)` on Linux | `setName(APP_DISPLAY_NAME)` | **Keep fork** (usually auto-merges) |
| `AGENTS.md` | Debugging section | Won't have it | **Keep fork** |
| `docs/linux-build.md` | Entire file | Won't have it | **Keep fork** |

## Post-Merge Verification

After every upstream merge:

1. `bun install` -- new packages may have been added
2. `bun run dist:desktop:deb --verbose` -- full build
3. `sudo dpkg -i release/T3-Code-*.deb` -- install
4. `sudo gtk-update-icon-cache -f -t /usr/share/icons/hicolor` -- refresh icon cache
5. Verify: app icon visible in launcher, app launches, dock pinning works
