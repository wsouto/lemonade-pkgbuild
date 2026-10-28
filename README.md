# lemonade-pkgbuild

Arch Linux PKGBUILD for [Lemonade](https://github.com/lemonade-sdk/lemonade),
derived from the official Arch packaging with local changes.

> **Unofficial.** This packaging is not affiliated with or endorsed by the
> Lemonade project or Arch Linux. It is not published to the AUR.

- Arch packaging baseline: `https://gitlab.archlinux.org/archlinux/packaging/packages/lemonade`
- Upstream: `https://github.com/lemonade-sdk/lemonade`
- Split packages: `lemonade-server` (daemon + CLI + web launcher) and
  `lemonade-desktop` (native Tauri app)

## Current package target

The authoritative build settings are in [PKGBUILD](PKGBUILD); generated package
metadata is in [.SRCINFO](.SRCINFO). This table describes the target, not proof
that an existing local archive was built with the current settings.

| Setting | Value |
| --- | --- |
| Version | `2026.43.3-2` |
| Upstream ref | `v2026.43.3` (commit `b48d190a`) |
| Artifacts | `lemonade-server-2026.43.3-2-x86_64.pkg.tar.zst`, `lemonade-desktop-2026.43.3-2-x86_64.pkg.tar.zst` |

This build:

- Installs `lemond` (daemon), `lemonade` (CLI) and `lemonade-app` (native GUI).
  The old `lemonade-server` binary name no longer exists.
- systemd service is `lemond.service`. Server settings are persisted in
  `config.json`; `/etc/default/lemond` is an optional environment file, not the
  server settings file. See [Configuration](#configuration).
- Both menu entries are shipped as upstream intends: "Lemonade App"
  (`lemonade-desktop`) and "Lemonade Web App" (`lemonade-server`).

## Local packaging settings

Re-check these settings after any upstream packaging update:

1. **Version / tag** — builds a specific upstream ref, which may be a
   *candidate* prerelease:

   ```bash
   pkgver=2026.43.3
   pkgrel=2
   _tag=v${pkgver}
   ```

2. **`.version` file** (in `prepare()`) — upstream uses calendar versioning
   (`YYYY.WW.N`). `tools/version.py` recognizes release tags matching
   `vYYYY.WW.N`, but not `candidate-v…`. Without an override it can generate a
   development version instead; an unrecognized candidate tag does not itself
   make CMake abort. Writing `.version` (highest priority) makes the binaries
   report exactly `${pkgver}`:

   ```bash
   printf '%s\n' "${pkgver}" > .version
   ```

3. **Debug packages** — `options=(!debug)` disables separate debug packages.
4. **Build scope** — `BUILD_TESTING=OFF` avoids compiling upstream test binaries
   that this package does not run or install. Python is required by CMake for
   version detection. `libdrm` is also a server runtime dependency.
5. **Web launcher dependencies** — `jq` is required for port discovery;
   `xdg-utils` is optional for the browser fallback.

Both desktop entries remain enabled. The web-app patch avoids a shared-icon
file conflict between the split packages.

## Updating to a new release

Follow the
[Arch Package Guidelines](https://wiki.archlinux.org/title/Arch_package_guidelines);
the [AUR Submission Guidelines](https://wiki.archlinux.org/title/AUR_submission_guidelines)
are a useful reference even though this package is not published there.

### 0. Refresh the packaging repo

Compare the local PKGBUILD and patches with the Arch packaging baseline:

```bash
git remote -v
# One-time setup, if this remote is not already configured:
git remote add arch https://gitlab.archlinux.org/archlinux/packaging/packages/lemonade.git
git fetch arch
git diff HEAD arch/main -- PKGBUILD lemonade-sysusers.patch lemonade-web-app.patch
```

Review and integrate relevant Arch changes while preserving the local version,
`.version` override, dependency fixes, build options and icon patch.
Do not blindly rebase onto `arch/main`; this repository's history is separate.

Build artifacts are ignored by the AUR-style `.gitignore`; package archives are
not intended for version control. Edits to tracked packaging files still appear
in `git status`. Run the following commands from the packaging repository root.

### 1. Find the ref to build

```bash
curl -s 'https://api.github.com/repos/lemonade-sdk/lemonade/releases?per_page=10' \
  | jq -r '.[] | "\(.tag_name)\tprerelease=\(.prerelease)"'
```

- Stable tag `v2026.43.3` → `pkgver=2026.43.3`, `_tag=v${pkgver}`
- Candidate prerelease `candidate-v2026.43.3` → `pkgver=2026.43.3`,
  `_tag=candidate-v${pkgver}`

> Select a ref that contains fixes required by your installed dependencies.
> For example, a release predating mbedTLS 4 support may fail with mbedTLS 4.
> Check `pacman -Q mbedtls` against the release notes; do not assume a candidate
> is necessary when a stable release already includes the fix.

To pin a commit instead: drop `_tag`, use
`git+${url}.git#commit=<sha>` in `source`.

### 2. Edit the PKGBUILD

- `pkgver=<new>`, `pkgrel=1`
- `_tag=v${pkgver}` (stable) or `_tag=candidate-v${pkgver}` (candidate)
- Increment `pkgrel` for packaging-only changes or a different ref with the
  same `pkgver`; otherwise pacman cannot distinguish the builds by version.
- Leave the `.version` line as-is; it follows `pkgver` automatically.
- Also diff against the upstream `PKGBUILD` (from step 0) in case
  `makedepends`, patches or the `package_*()` functions changed.

### 3. Regenerate checksums and metadata

`updpkgsums` is provided by `pacman-contrib`. Install `base-devel`, `git` and
`pacman-contrib` before starting; the release-query example also needs `curl`
and `jq`. Run `makepkg` as a regular user, not root.

```bash
updpkgsums                             # recomputes all source checksums
makepkg --printsrcinfo > .SRCINFO       # refresh tracked metadata
```

Check that metadata is synchronized and the PKGBUILD parses:

```bash
bash -n PKGBUILD
makepkg --printsrcinfo | diff -u .SRCINFO -
git diff --check
```

Review checksum changes before trusting them. If you edit either patch in the
next step, repeat both commands afterward.

### 4. Re-check the patches

The sysusers patch locks the service account. The web-app patch gives the
browser launcher its own icon so the split packages can coexist.
Both patches are applied in `prepare()`. Check both against the selected ref
before the long build; do not assume either patch still applies:

```bash
(
  set -e -o pipefail
  REPO=$PWD                       # run from this packaging directory
  ref=v2026.43.3                   # replace with the selected tag or commit
  checkdir=$(mktemp -d)
  trap 'rm -rf -- "$checkdir"' EXIT
  git --git-dir="$REPO/lemonade" archive --format=tar "$ref" |
    tar -x -C "$checkdir"
  cd "$checkdir"
  patch -Np1 --fuzz=0 --dry-run -i "$REPO/lemonade-sysusers.patch"
  patch -Np1 --fuzz=0 --dry-run -i "$REPO/lemonade-web-app.patch"
)
```

`./lemonade` is the git source cache left by `updpkgsums`/`makepkg` with the
default `SRCDEST`. If you override `SRCDEST`, adjust the cache path above.
The subshell leaves your working directory unchanged and removes only the
temporary directory it created.

An offset or fuzz is a reason to inspect where the hunk landed, not proof that
the patch is still correct. Update failed or misplaced hunks and repeat the
checks. Keep the web-app patch while both packages would otherwise install
`/usr/share/pixmaps/lemonade-app.svg`: it renames the server package's icon to
`lemonade-web-app`, allowing both packages to be installed side by side.

### 5. Preflight dependencies and Node

The PKGBUILD declares `nodejs-lts-krypton` and `npm`; it does not require
vite-plus or an exact Node `v26.10.0`. A custom Node installation on `PATH`
does not satisfy makepkg's package dependency checks.

```bash
command -v node npm
node --version
npm --version
```

If using a custom toolchain, install missing dependencies separately and
re-check `PATH` afterward. Avoid relying on a shell-local Node override
surviving dependency installation.

### 6. Build, install, verify

Build both split packages from the packaging repository root:

```bash
makepkg --syncdeps --log                   # install missing dependencies, then build
makepkg --packagelist                      # locate the resulting archives
sudo pacman -U /path/to/lemonade-server-<ver>-<rel>-x86_64.pkg.tar.zst \
               /path/to/lemonade-desktop-<ver>-<rel>-x86_64.pkg.tar.zst

lemond --version && lemonade --version    # both → <ver>
sudo systemctl daemon-reload
sudo systemctl enable --now lemond
systemctl status lemond
```

Replace the install paths with the output of `makepkg --packagelist`; this also
handles a custom `PKGDEST`. If all dependencies are already installed, plain
`makepkg` is sufficient. Use `makepkg --force` only when intentionally replacing
existing archives with the same version.

`options=(!debug)` means this PKGBUILD does **not** produce a separate
`lemonade-debug` package; an old debug archive may remain from an earlier build.
Enabling an already-running service does not restart it after an upgrade:
run `sudo systemctl restart lemond` when upgrading a running daemon, then
check `systemctl status lemond` again.

### Quick re-package after packaging-only changes

For packaging-function or runtime-dependency changes, no rebuild is required
if the successful build's source/build directories are
still present. Run from the packaging repository root:

```bash
makepkg -R -f --noconfirm
```

This reruns packaging, not `prepare()` or `build()`. Changes to patches, source,
dependencies used at build time, or build flags need a normal build instead.
Reinstall the regenerated archives afterward.

## Configuration

For the system service, persistent server settings such as `port`, `host` and
`models_dir` live in `/var/lib/lemonade/config.json` by default. The service
sets `StateDirectory=lemonade`, which takes precedence over the service user's
home directory. For a manually launched daemon, the default is
`${XDG_CONFIG_HOME:-$HOME/.config}/lemonade/config.json`; `lemond --help`
documents the positional `config_dir` override.

A user service is also installed. For a daemon running as your own user, use
`systemctl --user enable --now lemond`; its optional environment file is
`~/.config/lemonade/lemond.conf` (or under `$XDG_CONFIG_HOME`). Choose either the
system service or the user service for the default port to avoid a port conflict.

Use `lemonade config` to inspect the running server's settings and
`lemonade config set --help` for the update syntax.

`/etc/default/lemond` is for supported environment variables such as `HF_TOKEN`,
`LEMONADE_API_KEY` and `LEMONADE_ADMIN_API_KEY`, not port/model settings.
`backup=(etc/default/lemond)` tells pacman to preserve changes to that file on
upgrade. Restart the service after changing its environment.

## Web-app menu entry

Two launchers exist, one per split package:

| Menu entry | Package | Exec |
| --- | --- | --- |
| "Lemonade App" (native Tauri) | `lemonade-desktop` | `lemonade-app` |
| "Lemonade Web App" (browser) | `lemonade-server` | `lemonade-web-app` |

The web launcher requires `jq` to discover the server port before browser
selection. It prefers available Chromium-based browsers with `--app`; if none
is available (or Chromium preference is disabled), it uses `xdg-open`, provided
by the optional `xdg-utils` package.

Both apps connect to a running server; opening a menu entry does not enable the
systemd service. The native app can also connect to a remote server and does not
require the local server package. For local use, install `lemonade-server` and
start one of the services described above.

Both entries are shipped by default. If you ever want only one:

### Keep only the native app (drop the web-app entry)

Add to `package_lemonade-server()` (after the `cmake --install` line):

```bash
rm -f "${pkgdir}/usr/share/applications/lemonade-web-app.desktop"
```

then `makepkg -R -f --noconfirm` and reinstall `lemonade-server`.

### Keep only the web app

```bash
sudo pacman -Rns lemonade-desktop
```

### Hide the web-app entry but keep it functional

Add this to `package_lemonade-server()` after `cmake --install`:

```bash
printf '\nNoDisplay=true\n' >> "${pkgdir}/usr/share/applications/lemonade-web-app.desktop"
```

Then repackage and reinstall `lemonade-server` as above. `${pkgdir}` is supplied
by makepkg; do not run these packaging snippets directly in your shell.

## Gotchas

- Keep candidate refs in `_tag`, not `pkgver` (which cannot contain hyphens).
  The `.version` override keeps the reported binary version aligned with
  `pkgver`, but does not mark candidate builds as prereleases.
- `makepkg` (7.1) has no per-split-package build; `build()` always compiles the
  Tauri app.
- First build downloads Rust crates and may fetch some CMake `FetchContent`
  deps (brotli); allow network.
- If `extra`'s `mbedtls` is v4 and the target release predates the mbedTLS 4
  fix, the build fails — pick a newer ref.
- The optional environment file and persistent server settings are distinct;
  see [Configuration](#configuration).

## License

The packaging files in this repository (PKGBUILD, patches, README and
metadata) are licensed under [0BSD](LICENSE). Lemonade itself keeps its own
license (Apache-2.0); see the upstream repository.
