# Building T3 Code on Linux (Ubuntu)

## Prerequisites

### 1. Node.js 24.x

```bash
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt-get install -y nodejs
```

### 2. Bun 1.3.9+

```bash
curl -fsSL https://bun.sh/install | bash
```

### 3. Linux build dependencies

```bash
sudo apt-get install -y \
  build-essential \
  libssl-dev \
  libfuse2 \
  python3 \
  fakeroot \
  imagemagick
```

> `fakeroot` and `imagemagick` are only required when building the desktop `.deb` package.

## Install dependencies

```bash
bun install
```

## Build options

### Web + server only (no desktop)

```bash
bun run build
```

### Desktop — AppImage

```bash
bun run dist:desktop:linux
```

### Desktop — `.deb` package

```bash
bun run dist:desktop:deb
```

Add `--verbose` to see full electron-builder output if the build fails.

If you've already built once and only want to repackage without a full rebuild:

```bash
bun run dist:desktop:deb --skip-build
```

The output artifact is placed in the `release/` directory.

## Installing the `.deb` package

```bash
sudo dpkg -i release/T3-Code-0.0.4-alpha.1-amd64.deb
```

After installing, refresh the icon cache so the app icon appears in your launcher:

```bash
sudo gtk-update-icon-cache -f -t /usr/share/icons/hicolor
sudo update-desktop-database /usr/share/applications/
```

## Notes

- T3 Code requires [Codex CLI](https://github.com/openai/codex) to be installed and authenticated at runtime.
- The version number in the `.deb` filename matches the version in `apps/server/package.json`.
