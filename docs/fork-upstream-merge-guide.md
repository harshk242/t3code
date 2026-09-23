# Fork Upstream Merge Guide

This repo is a fork of [pingdotgg/t3code](https://github.com/pingdotgg/t3code) with Linux desktop (`.deb`) customizations. This document guides Claude (or a human) through merging upstream changes without losing fork-specific work.

## Remotes

```
origin    git@github.com:harshk242/t3code.git    (fork)
upstream  https://github.com/pingdotgg/t3code.git (source)
```

## Fork-Specific Changes

These are the custom changes maintained in this fork. Upstream targets Linux with AppImage; this fork additionally targets `.deb` and preserves GNOME-specific desktop integration.

### 1. Linux icon generation (`scripts/build-desktop-artifact.ts`)

- **What**: Upstream now provides `stageLinuxIcons()`, which generates standard-size PNGs via ImageMagick plus a root `icon.png`, and uses the `icons` directory in the Linux build config.
- **Why**: electron-builder installs icons into `hicolor/` for the `.desktop` entry. A single high-res PNG only creates one size (e.g. 1024x1024) which GNOME ignores -- it needs standard sizes. AppImage doesn't have this problem because it embeds icons differently.
- **Merge rule**: Prefer upstream's implementation while verifying that it still stages the root icon and standard sizes and keeps `icon: "icons"`.

### 2. Deb build metadata (`scripts/build-desktop-artifact.ts`)

- **What**: `StagePackageJson` interface and construction include `homepage` and structured `author` (with email). Linux build config includes `maintainer`.
- **Why**: electron-builder's FPM/deb target requires these fields and will error without them. AppImage and dmg targets don't need them, so upstream omits them.
- **Desktop identity**: The deb also stages `desktopName: "t3code.desktop"` and sets `linux.syncDesktopName: true`, so electron-builder's metadata agrees with the installed launcher. These are for GNOME integration, not FPM requirements.

### 3. `dist:desktop:deb` script (`package.json`)

- **What**: `"dist:desktop:deb": "node scripts/build-desktop-artifact.ts --platform linux --target deb --arch x64"`
- **Why**: Convenience script to build the `.deb` package. Upstream only has `dist:desktop:linux` (AppImage).

### 4. GNOME dock icon and pinning

- **What**: The deb installs `t3code.desktop` with `Icon=t3code` and `StartupWMClass=t3code`. Early Electron startup selects `t3code.desktop` for the deb; `DesktopApp.ts` sets `CHROME_DESKTOP` and the Chromium `class` switch, and `DesktopAppIdentity.ts` sets the app name to match. The deb must not generate a second, iconless user-local desktop entry. AppImage and development builds retain their generated entries.
- **Why**: GNOME must associate the running window with the installed launcher. The original 0.0.40 deb renamed the launcher from `t3code.desktop` to `com.t3tools.T3Code.desktop`, and startup created a user-local entry with that same name but no icon. That shadow entry caused the generic gear icon in the dock, even though the application menu icon was correct.

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
# rebuild: vp i && vp run dist:desktop:deb --verbose
# install: sudo dpkg -i release/T3-Code-*.deb
# verify: icon shows in dock, app launches, pinning works
```

## Conflict Resolution Decision Framework

When a merge conflict involves a fork-changed file, ask these questions **in order**:

### Step 1: Did upstream delete or replace the file?

If yes: check whether the replacement covers the fork's intent.

| Example                                                                                           | Resolution                                                  |
| ------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| `fixPath.ts` removed, replaced by `syncShellEnvironment.ts` that handles Linux                    | **Drop fork change** -- upstream solved it differently      |
| `bootstrap.ts` ENXIO catch, but upstream added `isBootstrapFdPathDuplicationError` covering ENXIO | **Drop fork change** -- upstream's solution is more general |

### Step 2: Did upstream rewrite the surrounding code?

If upstream restructured function signatures, moved parameters, or changed architecture:

- **Adapt the fork change to the new code**, don't force the old version back.
- Example: `stageLinuxIcons` changed from resolving icon sources internally to taking a `sourcePng` parameter. Adapt the multi-size generation to accept the parameter rather than reverting upstream's refactor.

### Step 3: Is the fork change deb-specific metadata?

Fields like `homepage`, `author.email`, `maintainer`, and `desktopName` -- upstream may drop these because they don't build debs.

- **Re-add them when missing.** `homepage`, `author.email`, and `maintainer` are required by electron-builder's FPM target. `desktopName` and `linux.syncDesktopName` keep the deb's desktop identity aligned.
- If upstream changed the `StagePackageJson` interface or `createBuildConfig`, add the fields to the new structure.

### Step 4: Is it a Linux desktop integration change?

`CHROME_DESKTOP`, `app.setName(LINUX_WM_CLASS)`, the early desktop name, URL-handler entry behavior, and `icon: "icons"` -- upstream may inadvertently revert these.

- **Keep fork changes** unless upstream has explicitly added equivalent GNOME integration (check their PRs/commit messages for "GNOME", "dock", "WM_CLASS", "desktop entry"). In particular, do not let a generated user-local entry shadow the deb's installed `t3code.desktop`.
- These typically auto-merge cleanly since upstream rarely touches the same lines.

### Step 5: Is it documentation?

`AGENTS.md` debugging section, `docs/linux-build.md` -- upstream may change surrounding content but won't add these.

- **Keep fork changes.** They auto-merge in most cases.

## Quick Reference: Conflict Resolution Table

| File / Area                                                                                                      | Fork adds                                                 | Upstream tendency                                  | Resolution                           |
| ---------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------- | -------------------------------------------------- | ------------------------------------ |
| `scripts/build-desktop-artifact.ts` : `stageLinuxIcons`                                                          | Upstream multi-size icon generation                       | May be refactored                                  | **Prefer upstream; verify behavior** |
| `scripts/build-desktop-artifact.ts` : linux build config                                                         | `maintainer`, deb desktop-name sync                       | No maintainer or deb identity                      | **Keep fork**                        |
| `scripts/build-desktop-artifact.ts` : `StagePackageJson`                                                         | `homepage`, `author` with email, deb `desktopName`        | No deb metadata                                    | **Keep fork**                        |
| `package.json` : scripts                                                                                         | `dist:desktop:deb`                                        | Will be absent                                     | **Re-add after upstream lines**      |
| `apps/desktop/src/app/DesktopApp.ts` : Linux block                                                               | `CHROME_DESKTOP` and `class`                              | Won't have them                                    | **Keep fork** (usually auto-merges)  |
| `apps/desktop/src/app/DesktopAppIdentity.ts`                                                                     | Linux WM class for `setName`                              | Uses display name                                  | **Keep fork** (usually auto-merges)  |
| `apps/desktop/src/app/DesktopEarlyElectronStartup.ts`, `DesktopPreReadyPlatform.ts`, `DesktopLinuxUrlHandler.ts` | Deb uses installed entry; AppImage/dev generate their own | May create a user-local entry for all Linux builds | **Keep fork**                        |
| `AGENTS.md`                                                                                                      | Debugging section                                         | Won't have it                                      | **Keep fork**                        |
| `docs/linux-build.md`                                                                                            | Entire file                                               | Won't have it                                      | **Keep fork**                        |

## Post-Merge Verification

After every upstream merge:

1. `vp i` -- new packages may have been added
2. `vp run dist:desktop:deb --verbose` -- full build
3. `sudo dpkg -i release/T3-Code-*.deb` -- install
4. `sudo gtk-update-icon-cache -f -t /usr/share/icons/hicolor` -- refresh icon cache
5. Verify: app icon visible in launcher, app launches, dock pinning works
