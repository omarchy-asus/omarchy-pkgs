# omarchy-asus pacman repo

> [!WARNING]
> **EXPERIMENTAL. Built for and only meant for the ASUS ROG Zephyrus G16 GA605 (GA605KM).**
> Nobody has installed these packages yet. The kernel carries untested patches, including an experimental scheduler series that is compiled in but switched off.
> Don't add this repo to any other machine. On a G16, use it only if you can recover from a kernel or driver that won't boot (this repo replaces `linux-omarchy`, so keep a second kernel such as Arch's `linux` installed as a boot fallback).

This branch holds build recipes and tooling for `omarchy-asus`, a signed pacman repo of ASUS
hardware fixes for Omarchy. It was developed on an ROG Zephyrus G16 GA605 (Ryzen AI 300 and an
RTX 50-series laptop GPU).

Every package takes upstream's pkgrel plus `.1` (for example `615.78.08-1.1`). That sorts just
above the stock build it replaces. Local helpers with no upstream build (such as `asus-rog-nkey-winlock`)
use a plain pkgrel instead. When upstream moves past our base version, the package
is rebuilt with the patches rebased, or it is removed from the repo so the stock build flows
through. A stale pin is never kept.

- **Install:** see [LAPTOP-SETUP.md](LAPTOP-SETUP.md).
- **Packages:** the [`x86_64` release](https://github.com/rdoupe-omarchy/omarchy-asus-pkgs/releases/tag/x86_64)
  holds the packages, their signatures and the signed repo database.
- **Signing key:** `keys/omarchy-asus.asc`, fingerprint `0B44FF7CCDD929F61BE4A2E436FE6B28AB24EC88`.

## Packages

| package | version | patches |
|---|---|---|
| nvidia-open-dkms | 615.78.08-1.1 | NVIDIA/open-gpu-kernel-modules [#1412](https://github.com/NVIDIA/open-gpu-kernel-modules/pull/1412) (no hang at power-off after hibernate), [#1299](https://github.com/NVIDIA/open-gpu-kernel-modules/pull/1299) (dGPU no longer wakes from D3cold on every battery change) |

| hyprland, hyprpm | 0.56.2-4.1 | render offset after an HDMI replug (fixed upstream on main, unreleased) |
| linux-omarchy, linux-omarchy-headers | 7.2.8-9.1 | hid-asus ROG N-Key Win-lock pass-through; **plus an EXPERIMENTAL, default-off scheduler series** (Zen 5c-first soft park) for GA605KM battery testing, compile-tested only, nothing enables it; see `recipes/linux-omarchy/NOTES.md` |
| asus-rog-nkey-winlock, asus-zephyrus-g16-control | 0.1.0-1 | local, not an upstream rebuild: `g16-winlock` (Fn+Super Win-key lock OSD / auto-unlock) with its hidraw udev rule and user unit, and the G16 Control bar widget that holds its switch; nothing is enabled by default, and `watch` needs the linux-omarchy 0568 patch; see `recipes/asus-rog-nkey-winlock/README.md` |

## Layout

- `recipes/<pkg>/`: PKGBUILDs and patches.
- `build/pacman.conf`: package sources for the build chroot (the Omarchy stable snapshot plus `[omarchy]`).
- `bin/mkchroot`: creates or updates the clean devtools chroot.
- `bin/build-pkg recipes/X`: `makechrootpkg -c -u` at idle CPU and IO priority.
- `bin/publish FILES`: signs the packages and runs `repo-add --sign --verify --remove`.
- `bin/unpublish PKG`: drops a package so the stock build flows through.
- `bin/sync-release`: mirrors the local repo to the `x86_64` GitHub release.
- `bin/check-upstream [--json]`: compares each package with the version upstream currently ships and the state of the upstream PRs listed in `recipes/*/UPSTREAM_PRS`. Exits 10 when something needs action.
