# SDK Patterns Reference

Detailed decision trees and patterns extracted from all reference SDKs.

## Platform Layout Decision

```
Does the SDK need to run on multiple Ubuntu versions?
├── YES → Multi-base: list each ubuntu@XX.04:arch under platforms
│         Use --platform flag in CI upload.yml
│         Example: node-sdk, rust-sdk, vscode-remote-sdk
│
└── NO → Does it need multiple architectures?
         ├── YES → Single build-base + explicit build-on/build-for
         │         Use --build-for flag (default) in CI
         │         Example: go-sdk (amd64, arm64, riscv64)
         │
         └── NO → Single build-base + simple platform
                   Example: copilot-sdk, opencode-sdk
```

### Multi-base platform format

```yaml
platforms:
  ubuntu@22.04:amd64:
  ubuntu@24.04:amd64:
```

No `build-base` field. CI uses `--platform` flag.

### Single-base with multi-arch

```yaml
build-base: ubuntu@24.04
platforms:
  amd64:
    build-on: [amd64]
    build-for: [amd64]
  arm64:
    build-on: [amd64]
    build-for: [arm64]
```

### Single-base, single-arch

```yaml
build-base: ubuntu@24.04
platforms:
  amd64:
    build-on: [amd64]
    build-for: [amd64]
```

Or the minimal form:

```yaml
build-base: ubuntu@24.04
platforms:
  amd64:
```

## Parts Strategy Decision

```
How is the upstream software distributed?
│
├── Pre-built binary tarball (Go, Ollama, OpenCode)
│   → plugin: nil or plugin: dump
│   → override-pull: curl + tar
│
├── npm package (Copilot CLI, Claude Code, Codex)
│   → plugin: npm
│   → npm-include-node: true
│   → override-pull: curl from registry.npmjs.org + tar
│
├── Rust source (uv, rustup)
│   → plugin: rust
│   → source from git
│
├── C/C++ source (OpenVINO)
│   → plugin: cmake
│   → git clone in override-pull
│
├── Python package — helper/auxiliary (sits alongside a compiled SDK tool)
│   → plugin: nil (install at runtime via pip/uv in hooks)
│   → Just set version in override-pull
│   → Acceptable because the compiled binary is the immutable deliverable
│
├── Python package — primary harness (Python IS the tool; no compiled binary)
│   → plugin: nil + build-packages: [python3-pip]
│   → override-build: pip3 install --prefix "$CRAFT_PART_INSTALL"
│   → prime: [bin/, lib/, VERSION]
│   → setup-base: PATH + PYTHONPATH pointing into $SDK/lib/
│   → Immutable: baked into SDK image, cannot self-update
│   → See "Plugin: nil — Python primary harness" below
│
├── System packages via apt (ROCm, .NET, ROS 2)
│   → plugin: nil
│   → Install in hooks/setup-base via apt
│
└── No upstream source (vscode-remote)
    → plugin: nil
    → Just set version in override-pull
```

### Common override-pull pattern (used by nearly all SDKs)

```bash
VERSION=$(cat "$CRAFT_PROJECT_DIR/VERSION")
craftctl set version="$VERSION"
# Then: curl, git clone, or nothing
```

### Plugin: nil — Version-only part

```yaml
parts:
  version:
    plugin: nil
    override-pull: |
      VERSION=$(cat "$CRAFT_PROJECT_DIR/VERSION")
      craftctl set version="$VERSION"
```

Use when: SDK installs everything via hooks, or has no upstream binary.


### Plugin: nil — Python primary harness

Use when: Python *is* the SDK's main tool (no compiled binary). Bakes the package
and all dependencies into the SDK image so the tool is immutable and version-locked.
Workshop SDKs must not be self-updatable; placing the tool in user-writable space
(e.g. a home-directory venv) breaks that contract.

```yaml
parts:
  myapp:
    plugin: nil
    build-packages: [python3-pip]
    override-pull: |
      VERSION=$(cat "$CRAFT_PROJECT_DIR/VERSION")
      craftctl set version="$VERSION"
    override-build: |
      pip3 install "myapp==$(cat "$CRAFT_PROJECT_DIR/VERSION")" \
        --prefix "$CRAFT_PART_INSTALL" \
        --no-compile
      # Debian/Ubuntu pip routes --prefix output through a local/ subdirectory
      # (scripts → local/bin/, packages → local/lib/). Flatten it so the
      # prime paths are where sdkcraft expects them.
      if [ -d "$CRAFT_PART_INSTALL/local" ]; then
        cp -a "$CRAFT_PART_INSTALL/local/." "$CRAFT_PART_INSTALL/"
        rm -rf "$CRAFT_PART_INSTALL/local"
      fi
      install -m 644 "$CRAFT_PROJECT_DIR/VERSION" "$CRAFT_PART_INSTALL/VERSION"
    prime:
      - bin/
      - lib/
      - VERSION
```

Pair with this `setup-base` to wire `PATH` and `PYTHONPATH` at runtime:

```bash
#!/usr/bin/bash
set -e
# Debian-patched pip may install to dist-packages instead of site-packages,
# and the Python version string varies. Discover the actual directory.
PACKAGES_DIR=$(python3 -c "
import glob
dirs = sorted(glob.glob('$SDK/lib/python*/*-packages'))
print(dirs[0] if dirs else '')
")
[ -n "$PACKAGES_DIR" ] || { echo "ERROR: no packages dir under $SDK/lib/" >&2; exit 1; }
cat <<EOF >/etc/profile.d/myapp.sh
export PATH="$SDK/bin:\$PATH"
export PYTHONPATH="${PACKAGES_DIR}\${PYTHONPATH:+:\$PYTHONPATH}"
EOF
```

No `setup-project` is needed for the tool itself. Any mount plugs are for **user
data only** (credentials, session history) — not for the tool installation.

> **`--no-compile`:** Always pass this flag. Python `.pyc` bytecode files are
> unnecessary in the SDK image (Python regenerates them on first import) and
> `--no-compile` prevents them from being created. Note: `.pyc` files are
> architecture-independent (CPython bytecode targets the VM, not the CPU), so
> they are not a cross-arch correctness risk — but they are bloat.
>
> **Native-extension cross-arch risk:** Python packages commonly depend on
> libraries that ship architecture-specific `.so` files (e.g. pydantic,
> cryptography, numpy). Cross-compiling on amd64 for arm64 causes pip to
> download amd64 wheels for every dependency, producing an SDK image that
> fails at runtime on non-amd64 targets with `ImportError: wrong ELF class`.
> **Default to native builds** — use the bare platform entry form so each
> architecture gets its own build:
>
> ```yaml
> platforms:
>   ubuntu@24.04:amd64:
>   ubuntu@24.04:arm64:
>   ubuntu@24.04:riscv64:
> ```
>
> Only use explicit `build-on`/`build-for` cross-compilation if you have
> confirmed that the package and **all** of its transitive dependencies are
> pure Python (no `.so` files anywhere in the installed tree).

### Plugin: dump — Pre-built binary

```yaml
parts:
  myapp:
    plugin: dump
    source: .
    override-pull: |
      VERSION=$(cat "$CRAFT_PROJECT_DIR/VERSION")
      craftctl set version="$VERSION"
      curl -fLO "https://example.com/releases/v${VERSION}/myapp-linux-amd64.tar.gz"
      tar -xzf "myapp-linux-amd64.tar.gz" -C "$CRAFT_PART_SRC"
      rm "myapp-linux-amd64.tar.gz"
    organize:
      myapp: usr/bin/myapp
```

### Plugin: npm — Node.js CLI tool

```yaml
parts:
  mytool:
    plugin: npm
    source: .
    npm-include-node: true
    npm-node-version: 22.22.1
    override-pull: |
      VERSION=$(cat "$CRAFT_PROJECT_DIR/VERSION")
      craftctl set version="$VERSION"
      curl -fLO "https://registry.npmjs.org/@scope/pkg/-/pkg-${VERSION}.tgz"
      tar -xzf "pkg-${VERSION}.tgz" --strip-components=1 -C "$CRAFT_PART_SRC"
      rm "pkg-${VERSION}.tgz"
```

### Multiple parts — Binary + service file

```yaml
parts:
  runtime:
    plugin: dump
    source: .
    override-pull: |
      # ... download binary
  services:
    plugin: dump
    source: services
    source-type: local
```

### Multiple parts — Binary + workshop-prompt

```yaml
parts:
  agent:
    plugin: npm
    # ... main tool
  workshop-prompt:
    plugin: dump
    source: workshop-prompt.md
    source-type: file
```

## Interface Layout Decision

### Mount plugs — persistent data

Use for any directory that should survive workshop updates:

| What to persist | Target pattern | Examples |
|---|---|---|
| Package cache | `~/.cache/<tool>` | uv, pnpm, yarn |
| Download cache | `~/.npm/_cacache` | npm |
| Config/credentials | `~/.<tool>` | copilot, claude, codex, rustup, cargo |
| Module/dep cache | `~/go/pkg/mod` | Go modules |
| Models | `~/.ollama/models` | Ollama |
| Build artifacts | `~/workspace` | ROS 2 colcon |
| VS Code server | `~/.vscode-server` | vscode-remote |
| Virtual env | `$SDK/venv` | jupyter |

```yaml
plugs:
  my-cache:
    interface: mount
    workshop-target: /home/workshop/.cache/mytool
```

Optional mount attributes: `mode`, `uid`, `gid`, `read-only`.

### GPU plug

```yaml
plugs:
  gpu:
    interface: gpu
```

Use when: ML inference, GPU-accelerated computation, graphics.

### Tunnel slot — expose a network service

```yaml
slots:
  my-server:
    interface: tunnel
    endpoint: 8080
```

Use when: SDK runs a daemon (Ollama, JupyterLab, etc.) that users access
from the host.

### Mount slot — share data with other SDKs

```yaml
slots:
  venv:
    interface: mount
    workshop-source: /home/workshop/my-venv
```

Use when: Other SDKs need to consume a resource this SDK produces (e.g., uv
providing a venv that jupyter consumes).

### Desktop and SSH plugs

```yaml
plugs:
  desktop:
    interface: desktop    # Wayland socket access
  ssh-agent:
    interface: ssh        # SSH agent forwarding (must be named ssh-agent)
```

## Systemd Service Pattern

When the SDK runs a long-lived daemon:

1. Create `services/<name>.service`:

```ini
[Unit]
Description=My Service
After=network.target

[Service]
ExecStart=/bin/bash -lc "myapp serve"
Restart=always
RestartSec=3

[Install]
WantedBy=default.target
```

2. Add a dump part:

```yaml
parts:
  services:
    plugin: dump
    source: services
    source-type: local
```

3. Install in `hooks/setup-project`:

```bash
install -D --mode=644 --target-directory ~/.config/systemd/user "$SDK/<name>.service"
systemctl --user daemon-reload
systemctl --user enable --now <name>
```

## Version Track Decision

```
Does the upstream project have multiple supported major versions?
├── YES → Multiple tracks: branch per major (e.g., 20, 22, 24 for Node.js)
│         Branch pattern: "[0-9]+" or "[0-9]+.[0-9]+"
│         Renovate: baseBranchPatterns lists each, with allowedVersions per branch
│
└── NO → Single "latest" track
          Branch: "latest"
          Renovate: baseBranchPatterns: ["latest"], no allowedVersions needed
```
