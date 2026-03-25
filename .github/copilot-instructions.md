# Exegol Images — AI Agent Instructions

## Project Overview

Docker image build system for [Exegol](https://github.com/ThePorgs/Exegol), an offensive security toolkit. Contains Dockerfiles, ~400 tool installation scripts, and configuration assets. All code runs **inside containers only** — never execute install scripts on the host.

Full documentation: <https://docs.exegol.com/>

## Architecture

```
Dockerfile / ad.dockerfile / web.dockerfile / osint.dockerfile / light.dockerfile
  └── COPY sources/ → /root/sources/
       └── entrypoint.sh → sources common.sh + all package_*.sh → dispatches to function name
```

**Image variants** select different subsets of `package_*()` functions. `package_base` is always first (sets up languages, asdf, pyenv, rvm, pipx).

**Key directories inside the container:**

| Path | Purpose |
|------|---------|
| `/opt/tools/` | Cloned/compiled tools |
| `/opt/tools/bin/` | Symlinks to binaries |
| `/.exegol/` | Runtime config, test commands |

## Build & Test

```bash
# Build full image (runs ~6 hours)
docker build -t exegol-full .

# Build a specific variant
docker build -f ad.dockerfile -t exegol-ad .

# The entrypoint dispatches to shell functions:
# ./entrypoint.sh package_base    → foundation setup
# ./entrypoint.sh package_ad      → AD tools
# ./entrypoint.sh post_build      → cleanup & validation
```

**CI** (`entrypoint_pr.yml`): builds on both amd64 and arm64 self-hosted runners. ShellCheck linter runs with ignored codes: SC1071, SC1090, SC1091, SC2148, SC2317.

## Install Function Pattern

Every tool follows this template in `sources/install/package_<category>.sh`:

```bash
function install_toolname() {
    # CODE-CHECK-WHITELIST=add-aliases   # suppress CI check if alias file missing
    colorecho "Installing toolname"

    # Installation (pick one):
    # fapt package-name                                            # APT
    # go install -v github.com/org/tool@${TOOL_VERSION}            # Go
    # pipx install --system-site-packages tool==${TOOL_VERSION}    # PyPI
    # pipx install --system-site-packages git+https://...@${TOOL_VERSION}  # git+pip
    # git -C /opt/tools/ clone --branch "${TOOL_VERSION}" --depth 1 https://...  # git clone

    add-aliases toolname          # loads sources/assets/shells/aliases.d/toolname
    add-history toolname          # loads sources/assets/shells/history.d/toolname
    add-test-command "toolname --help"
    add-to-list "toolname,https://github.com/org/tool,Short description"
}
```

**Required elements** (CI enforces these via code compliance checks):
- `colorecho` announcing the tool name
- `add-test-command` with a working validation command
- `add-to-list` with CSV metadata
- `add-aliases` and `add-history` (or `CODE-CHECK-WHITELIST` annotation to skip)

## Version Pinning System

All tool versions are centralized in `sources/install/tool-versions.env`:

```bash
# renovate: datasource=github-releases depName=org/tool
TOOL_VERSION="v1.2.3"
```

- Loaded via `set_tool_versions()` in `common.sh` (called by `set_env()`)
- **Renovate Bot** parses annotations and opens weekly grouped PRs
- Supported datasources: `github-releases`, `github-tags`, `pypi`, `rubygems`, `npm`, `crate`

**When adding a new tool:** add the version variable + Renovate annotation to `tool-versions.env`, then use `${VAR}` in the install function. Never use `@latest`.

### Version Pinning Coverage

<!-- AGENT-RULE: When you add, remove, or convert a version pin in any
     package_*.sh or tool-versions.env, update the counts below to match.
     Pinned = entries in tool-versions.env referenced via ${..._VERSION}.
     Unpinned = install calls in package_*.sh that lack a version pin.
     Recalculate the total and percentage after every change. -->

**231** / **397** versionable installs pinned (**58%**)

| Unpinned pattern | Count | How to pin |
|------------------|------:|------------|
| `go install …@latest` | 2 | Add var to `tool-versions.env`, use `@${VAR}` |
| `pipx install` (no `==` or `@`) | 31 | Add `==${VAR}` (PyPI) or `@${VAR}` (git+) |
| `git clone` (no `--branch`) | 90 | Add `--branch "${VAR}"` |
| `pip3 install` (no `==`) | 43 | Add `==${VAR}` or pin in requirements.txt |

**Pinned installs** use `${…_VERSION}` variables from `tool-versions.env` via patterns like:
- `go install -v …@${TOOL_VERSION}`
- `pipx install --system-site-packages tool==${TOOL_VERSION}`
- `pipx install --system-site-packages git+https://…@${TOOL_VERSION}`
- `git -C /opt/tools/ clone --branch "${TOOL_VERSION}" --depth 1 https://…`

## Helper Functions (common.sh)

| Function | Purpose |
|----------|---------|
| `colorecho "msg"` | Blue tagged log message |
| `criticalecho "msg"` | Red error + `exit 1` |
| `criticalecho-noexit "msg"` | Red error, continues |
| `fapt pkg1 pkg2` | APT install with auto-update |
| `add-aliases tool` | Source alias file from `assets/shells/aliases.d/` |
| `add-history tool` | Append history from `assets/shells/history.d/` |
| `add-test-command "cmd"` | Register build-time test |
| `add-to-list "t,url,desc"` | Register tool in installed CSV |
| `set_env` | Master environment setup (calls all `set_*_env`) |
| `post_install` | Cleanup caches after each package |
| `post_build` | Final cleanup, kill processes, sort lists |

**Retry mechanism:** `curl`, `git`, `apt-get`, `pip` calls are wrapped with exponential backoff (8s → 600s cap).

## Asset Conventions

| Directory | Contents |
|-----------|----------|
| `sources/assets/shells/aliases.d/<tool>` | Bash/Zsh aliases per tool |
| `sources/assets/shells/history.d/<tool>` | Pre-loaded command history per tool |
| `sources/assets/patches/` | Patch files applied during build |
| `sources/assets/desktop/applications/` | `.desktop` launcher files |
| `sources/assets/desktop/wallpapers/` | Desktop wallpapers |

## Conventions & Pitfalls

- **All shell, no Makefile**: builds are pure Bash dispatched through `entrypoint.sh`
- **Arch awareness**: many functions branch on `uname -m` (`x86_64` vs `aarch64`)
- **asdf for version management**: Go, Node.js, etc. managed via asdf; call `asdf reshim` after `go install`
- **Python envs**: pyenv + rvm coexist; `set_python_env` / `set_ruby_env` must run first
- **Temporary fixes**: use `check_temp_fix_expiry "YYYY-MM-DD"` pattern with a clear date limit
- **GitHub API rate limits**: prefer direct download URLs over API calls to `/releases/latest`
- **PRs target `dev` branch**, not `main`
