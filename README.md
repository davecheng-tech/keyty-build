# keyty-build

CI harness for building [keytyapp/Keyty](https://github.com/keytyapp/Keyty) from source,
because it can't be compiled on this Mac: Keyty needs Xcode 16 / Swift 6, which don't
run on macOS Monterey. GitHub Actions does the compile; signing and install happen locally.

This repo contains **no Keyty source**. It only checks upstream out at a pinned SHA and builds it.

## Quick start

```sh
gh workflow run build.yml                       # uses the default pinned SHA
gh workflow run build.yml -f ref=<upstream-sha>  # or a specific commit
gh run watch
```

Then download and verify:

```sh
DIR=~/Downloads/keyty-$(date +%Y%m%d-%H%M%S); mkdir -p "$DIR"
gh run download <run-id> --dir "$DIR"
cd "$DIR"/keyty-*
test "$(cat COMMIT.txt)" = "<expected-sha>" && echo "COMMIT OK"
awk '{print $1"  Keyty.zip"}' SHA256.txt | shasum -a 256 -c -
```

**Stop if either check fails.** The artifact contains `Keyty.zip`, `AppStore.entitlements`,
`SHA256.txt` and `COMMIT.txt`.

## Build parameters (don't drift from these)

| What | Value | Why |
|---|---|---|
| Scheme | `Keyty AppStore` | The sandboxed target, with no Sparkle/auto-updater wired in |
| Configuration | `AppStore` | Matches the scheme |
| Project generator | Tuist (`tuist generate --no-open`) | Upstream has no checked-in `.xcodeproj` |
| Code signing | disabled in CI | Signed locally instead; CI has no identity |
| Pinned SHA | `e30e0644df03536a4169936c7e19a049229beb5b` | Known-good build |

Upstream source is never patched. If a build breaks, fix the workflow, not Keyty.

## Building for a different architecture

The `Build` step in `.github/workflows/build.yml` sets the arch explicitly:

```
ARCHS=x86_64 ONLY_ACTIVE_ARCH=NO
```

Change `ARCHS` to retarget:

| Target | `ARCHS` value | Notes |
|---|---|---|
| Intel (current) | `x86_64` | Cross-compiled; `macos-15` runners are Apple Silicon |
| Apple Silicon | `arm64` | Native on `macos-15`, so this is the faster path |
| Universal | `"x86_64 arm64"` | One bundle for both; roughly doubles build time and binary size |

`ONLY_ACTIVE_ARCH=NO` is what makes `ARCHS` authoritative — keep it, or xcodebuild may
quietly build only the runner's native arch and ignore the override.

Confirm what you actually got before trusting it:

```sh
lipo -archs Keyty.app/Contents/MacOS/Keyty
```

Keyty's `LSMinimumSystemVersion` is `11.0`, so every option above is deployable on
Big Sur and later. Nothing else in the workflow is arch-specific.

### Runner images

`runs-on: macos-15` is Apple Silicon. GitHub's label-to-hardware mapping changes over
time (and Intel runners are being retired), so check
[the runner images list](https://github.com/actions/runner-images) before assuming a
label is Intel. Cross-compiling from an arm64 runner works fine and avoids the issue —
that's what the current workflow does.

## Local install (Intel Monterey)

CI ships the app unsigned, so it must be ad-hoc signed before macOS will run it:

```sh
ditto -x -k Keyty.zip .
xattr -dr com.apple.quarantine Keyty.app
codesign --force --deep --sign - Keyty.app
codesign --force --sign - --entitlements AppStore.entitlements Keyty.app
```

Verify, then install:

```sh
codesign -d --entitlements - Keyty.app        # expect app-sandbox = true
codesign --verify --deep --strict Keyty.app   # expect "satisfies its Designated Requirement"
mv Keyty.app ~/Applications/
```

Two ordering details that matter:

- The **deep** sign comes first (it covers nested frameworks and dylibs), then the
  **shallow** re-sign attaches entitlements to the main executable. Reversing these
  drops the entitlements.
- `AppStore.entitlements` from upstream contains only `com.apple.security.app-sandbox`.
  If a future upstream commit adds entitlements, they come along automatically — but
  re-read the file rather than assuming it's still sandbox-only.

## Input Monitoring / TCC

Keyty needs **Input Monitoring** (System Preferences → Security & Privacy → Privacy).
Grant it, then **quit and relaunch** — it only takes effect on a fresh launch.

Ad-hoc signatures have no stable signing identity, so macOS keys the grant largely to
the bundle path. That means the permission is likely to be lost whenever you:

- move the app between `/Applications` and `~/Applications`
- install a rebuild at a different path

Fix: remove the stale `Keyty` entry in that pane (**–**), re-add the app at its current
path (**+**), relaunch. Installing a rebuild *over the same path* usually keeps the grant.

Current install location: `~/Applications/Keyty.app`.

## Rebuilding later

A rebuild from the same SHA should be byte-similar but is **not** guaranteed to be
bit-identical — the toolchain, SDK and Tuist version on the runner all float. So compare
`COMMIT.txt` across runs, not `SHA256.txt`. The checksum's job is verifying the download
didn't corrupt, not proving two runs match.
