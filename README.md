# QEMU (jclab-oss patched build)

This repository mirrors [upstream QEMU](https://gitlab.com/qemu-project/qemu) and publishes
builds with **WHPX (Windows Hypervisor Platform) patches** that are not yet in an upstream
release.

- Upstream code is never modified in place; patches are added on a separate branch per release tag.
- Builds and releases are fully automated with GitHub Actions.
- Nothing here is intended for upstream contribution. When reporting to the upstream issue
  tracker or mailing list, please reproduce with an upstream build, not this one.

## Branches and tags

| Name | Contents |
|---|---|
| `ci` (default) | This README and the GitHub Actions workflows only |
| `master`, `stable-*`, upstream tags | Upstream mirror, synced daily. Do not commit to these |
| `patched/<tag>` | Upstream `<tag>` with the patches applied (e.g. `patched/v11.1.2`) |
| `<tag>+patched` | Tag on the tip of that branch (e.g. `v11.1.2+patched`) |

## Patches

The authoritative list is `PATCHES` in [`patched-release.yml`](.github/workflows/patched-release.yml).
Each entry is a patchew mbox applied in order with `git am -3`; patches already contained in
the tag are skipped automatically by git. `hw/i386: fw_cfg: do not set VMX feature control on WHPX`
was applied initially but has been dropped since it is upstream (included from v11.1.1).

| Patch | Description | Upstream status (as of 2026-10-21) |
|---|---|---|
| [whpx: i386, arm: do partition reset on boot CPU reset (v2)](https://patchew.org/QEMU/20260421205749.55060-1-mohamed@unpredictable.fr/) | Calls `WHvResetPartition` on system reset to reset partition state. Also fixes the Arm reset call failing while other vCPUs were still running | **Not merged** (not in master) |

The patch applies cleanly to v11.0.5 and v11.1.2, but not to v10.0.x.

## Related upstream issues

The patch does not reference any issue (no `Resolves:` tag), so the links below are
**inferred from the symptoms**. Please verify whether this build fixes your case.

| Issue | State | Symptom |
|---|---|---|
| [#4410](https://gitlab.com/qemu-project/qemu/-/work_items/4410) | open | Guest hangs at SeaBIOS boot after `system_reset` or a guest-initiated reboot (11.1.1) |
| [#2402](https://gitlab.com/qemu-project/qemu/-/work_items/2402) | open | edk2 (UEFI) guest freezes on reboot or fails with `Unexpected VP exit code 4` |
| [#2042](https://gitlab.com/qemu-project/qemu/-/work_items/2042) | open | Linux guest with `-smp` > 1 fails on reboot with `Unexpected VP exit code 4` |
| [#858](https://gitlab.com/qemu-project/qemu/-/work_items/858) | open | `whpx: injection failed, MSI ...` followed by `Unexpected VP exit code 4` while booting OVMF |

`WHvResetPartition` is not available on Windows 10. QEMU skips the call when the function is
missing, so **this patch has no effect on Windows 10 hosts.**

The reporter of [#4123](https://gitlab.com/qemu-project/qemu/-/work_items/4123)
(`Unexpected VP exit code 4` at startup with `-cpu host`) tried this patch and reported that it
did not help. Failures right at startup, rather than on reboot, are not what this patch addresses.

## Automation

### 1. Upstream sync — [`sync-upstream.yml`](.github/workflows/sync-upstream.yml)

- **When**: daily at 03:17 UTC, or manually from the Actions tab.
- **What**: pushes `master`, `stable-*` and all tags from `https://gitlab.com/qemu-project/qemu.git`
  to this repository. A mirrored branch that has diverged makes the job fail instead of being
  force-overwritten.
- **New tags**: for tags that appeared in this sync, a patched release is started if:
  - `AUTO_TAG_PATTERN`: the tag is `vX.Y.Z` (no `-rcN`)
  - `AUTO_TAG_MIN_VERSION`: the version is at least `11.0.0`
  - the repository already had tags before the sync (the very first sync starts no releases).

### 2. Patched release — [`patched-release.yml`](.github/workflows/patched-release.yml)

Called automatically by the sync, or run manually from Actions → **Patched release** → *Run workflow*.

| Input | Default | Description |
|---|---|---|
| `tag` | (required) | Upstream tag (e.g. `v11.1.2`) |
| `force` | `false` | Re-create the `patched/<tag>` branch and `<tag>+patched` tag. With `false`, an existing tag is reused and only the build is redone |
| `targets` | `x86_64-softmmu,aarch64-softmmu` | `configure --target-list` |

Steps:

1. Create `patched/<tag>` from `<tag>` and apply `PATCHES`
2. Create the `<tag>+patched` tag and push the branch and tag
3. Build for four platforms in parallel
4. Create the GitHub release `<tag>+patched` (if it exists, only the files are replaced)

## Release assets

| File | Built on | Notes |
|---|---|---|
| `qemu-<tag>-patched-linux-amd64.tar.xz` | Ubuntu 24.04 | Install tree for `/opt/qemu`, dynamically linked against Ubuntu 24.04 system libraries |
| `qemu-<tag>-patched-linux-arm64.tar.xz` | Ubuntu 24.04 (arm64) | Same as above |
| `qemu-<tag>-patched-windows-amd64.zip` | MSYS2 UCRT64 | WHPX enabled, required DLLs bundled |
| `qemu-<tag>-patched-windows-arm64.zip` | MSYS2 CLANGARM64 | WHPX enabled, required DLLs bundled |
| `SHA256SUMS` | | Checksums of the files above |

File names avoid `+`; the tag itself is still named `<tag>+patched`.

## Setup

- **Default branch**: must be `ci`. Scheduled runs and the manual run button only use workflows
  on the default branch, and `master` is overwritten from upstream on every sync.
- **Secrets**: pushes and releases use a short-lived GitHub App token.
  - `QEMU_BOT_CLIENT_ID`: the GitHub App's Client ID
  - `QEMU_BOT_PRIVATE_KEY`: the GitHub App's private key (PEM)
- **GitHub App permissions**: grant *Read & write* for Contents and Workflows under Repository
  permissions, and install the app on this repository. Workflows permission is needed because
  the upstream history being pushed may contain changes to `.github/workflows/`.

## Adding or removing patches

1. Add or remove patchew mbox URLs in `PATCHES` in `patched-release.yml` on the `ci` branch.
   - Whole series: `https://patchew.org/QEMU/<series Message-ID>/mbox`
   - Single patch from a series: `https://patchew.org/QEMU/<patch Message-ID>/mbox`
2. To update an existing release, re-run **Patched release** for that tag with `force: true`.
3. Once a patch is in an upstream release, `git am` skips it automatically, so it can stay in
   the list until every supported series (currently v11.0.x and later) contains it.

## License

QEMU and the patches keep their original licenses (mostly GPL-2.0-or-later).
See the upstream [LICENSE](https://gitlab.com/qemu-project/qemu/-/blob/master/LICENSE) for details.
