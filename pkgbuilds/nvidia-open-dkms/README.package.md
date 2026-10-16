# NVIDIA open kernel modules with pending fixes

Edge package of NVIDIA's open kernel modules 615.78.08 for aarch64 and x86_64.
Both architectures carry NVIDIA fixes that are still pending upstream
(`0008`, `0010`-`0012`); aarch64 also carries the N1x display fixes
(`0003`-`0007`, `0009`), which only that build applies. Based on
[Arch's DKMS recipe](https://gitlab.archlinux.org/archlinux/packaging/packages/nvidia-utils/-/commit/f9ae10b379f8b1d0832ec92bca1c12072aa123e9).
It used to carry Martin Stark's
[NVIDIA PR #1359](https://github.com/NVIDIA/open-gpu-kernel-modules/pull/1359)
as `0001`/`0002`; 615.78.08 has NVIDIA's own version of that DisplayPort
detach fix, so they are gone.

It carries `0003-set-oled-edp-brightness-over-aux.patch`: the GPU
firmware sets an internal panel's brightness as PWM or a VESA eDP level, and
the Dell XPS 16 (N1x)'s eDP 1.5 OLED panel ignores both. nvkms also sets it as
a target luminance, as the kernel's `drm_edp_backlight` helpers do for i915
and amdgpu, but only on eDP 1.5 panels without a PWM input that support
luminance control; everything else stays with the firmware. Keep it until
NVIDIA's driver sets these panels itself.

And `0004-drive-both-displays-of-a-dock-on-one-n1x-usb4-port.patch`: on the
N1x a USB4 port's two DisplayPort IN adapters are two USB-C connectors on one
DP pad-link, each with its own SOR, but the GPU firmware's head routing map
lets a pad-link drive one display only. A dock's second display never lit.
When the firmware rejects a set of displays, nvkms asks again with one USB-C DP
connector per shared pad-link and accepts the set if that passes. Keep it until
NVIDIA's firmware routes both adapters itself.

And `0005-retrain-the-link-after-the-sink-was-unplugged.patch`: when a
DisplayPort monitor came back after an unplug while its head was still
attached, nvkms skipped link training if the link status still read as
trained. Behind a USB4 dock that is every time the dock's DisplayPort tunnel
is rebuilt (the dock is replugged, or the monitor drops out as it wakes), so
the monitor stayed black while the desktop kept it. nvkms now trains the link
again after an unplug. Keep it until NVIDIA's driver does so itself.

And `0006-set-a-mode-afresh-on-a-sink-that-comes-back-through-a-usb4-tunnel.patch`:
the DP library revives the head of a monitor that comes back before the
client noticed it was unplugged. Behind a USB4 dock the DisplayPort tunnel is
new by then and the revived head leaves the monitor black, which happened to
a dock's second monitor whenever the dock was replugged within a few seconds.
On tunnelled links nvkms now reports such a monitor as unplugged until its old
head is shut down, so the client always sets a mode on it afresh.

And `0007-train-a-tunnelled-link-again-when-the-sink-cannot-decode-it.patch`:
behind a USB4 dock, a monitor could come out of standby black at a link that
read as fully trained, while it counted symbol errors on every lane. After
each modeset on a tunnelled link nvkms now reads the monitor's symbol error
counters and, if every lane keeps counting errors, trains the link again.

And `0008-free-the-semaphore-surface-callback-when-the-waiter-is-already-signalled.patch`,
JD Daniels'
[NVIDIA PR #1368](https://github.com/NVIDIA/open-gpu-kernel-modules/pull/1368):
when nvidia-drm asked nvkms to wait on a semaphore surface fence that had
already signalled, nvkms returned without freeing the callback record it had
allocated. Under explicit sync that is most composited frames, so the kernel
leaked a small slab object per frame while anything on screen moved, gigabytes
of unreclaimable memory over a few days of uptime. nvkms now frees the record
on that path. Keep it until NVIDIA merges the fix.

And, on both architectures, Lars Nieradzik's
[NVIDIA PR #1299](https://github.com/NVIDIA/open-gpu-kernel-modules/pull/1299)
as `0010` and `0011` (both commits, authorship kept): on hybrid laptops the
firmware sends an NVPCF notification through ACPI on every battery percent,
and each one woke a runtime-suspended GPU from D3cold for 20-25 s at +6-9 W.
The notifications are coalesced while the GPU sleeps and replayed after it
resumes. On an ASUS ROG Zephyrus G16 (GA605KM, RTX 5060 Laptop) on battery
the stock driver woke for 13 of 13 battery steps, the patched one for 0 of 3.
Keep it until NVIDIA merges the fix.

And Hans van Essen's
[NVIDIA PR #1412](https://github.com/NVIDIA/open-gpu-kernel-modules/pull/1412)
as `0012`: after the hibernation image is written, thaw is a no-op, so
`nv_pci_shutdown()` tore down a GPU that was still suspended and power-off
hung. It now only clears bus mastering in that state. On the same G16 the
stock driver never powered off after writing the image; patched, 4 of 4
hibernations powered off and resumed with no driver errors. Keep it until
NVIDIA merges the fix.

On x86_64 the package replaces Arch's `nvidia-open-dkms` of the same
version; it does not touch `nvidia-utils`, which still comes from Arch.
Machines with the stock `linux` kernel and Arch's prebuilt `nvidia-open`
are not affected: this package conflicts with it and is only pulled in where
`nvidia-open-dkms` is installed, as Omarchy's NVIDIA setup does.

Requires `[omarchy]` before `[extra]` and matching `nvidia-utils=615.78.08`.
Update both NVIDIA packages together; automatic version tracking is disabled.

On x86_64, drop `0010`-`0012` once NVIDIA ships them. This package pins
`nvidia-utils=615.78.08`, and on x86_64 `nvidia-utils` comes from Arch: once a
channel's Arch snapshot carries a newer `nvidia-utils`, `pacman -Syu` stops
with a broken dependency until this recipe is bumped to the same version.
Bump it before, or together with, the mirror snapshot that moves the driver.

Remove the aarch64 part of this recipe and the published package/database
entry once a fixed Arch Linux ARM driver is validated. A stale package in the earlier repository
can block driver updates.
