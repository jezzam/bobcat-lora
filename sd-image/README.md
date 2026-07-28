# SD-Card SSH-Unlock Image

Prepares SSH access on the Bobcat Miner 300 (hardware "285" — matches this
device's serial `285US915M221301351`) by SD-booting a minimal OS that patches
`/etc/shadow` and `/etc/ssh/sshd_config` on the eMMC. See
[`../Research_beginning_notes.md`](../Research_beginning_notes.md) for full
project context — this file only covers the image itself.

## Contents

| Path | What it is |
|---|---|
| `original/Bobcat_SSH_285.img.xz` | Untouched download, exactly as published |
| `patched/Bobcat_SSH_285_patched.img.xz` | Our sanitized version — **use this one** |
| `CHECKSUMS.sha256` | SHA256 of both files above (verify with `sha256sum -c CHECKSUMS.sha256`) |

## Provenance

- Source: `https://github.com/sicXnull/Bobcat-SSH-Access` (release `1.0`, asset `Bobcat_SSH_285.img.xz`)
- Downloaded: 2026-07-28
- **No checksum was published by the author** for this asset — the SHA256 in
  `CHECKSUMS.sha256` for `original/` is one we computed ourselves on download,
  useful only to detect future tampering/corruption of *our copy*, not as
  independent verification of the upstream file.
- The repo contains only a README — no source code, no build scripts. The
  `.img.xz` is an opaque pre-built binary. We inspected it directly (see
  below) rather than trusting the description alone.

## What the image does (verified by direct inspection, not just the README)

Boots a minimal Debian-based OS from SD. Its `/usr/sbin/init` (a plain shell
script, not real systemd) does, unconditionally and without error-checking:

1. Mounts `/dev/mmcblk0p4` (confirmed eMMC rootfs partition) read-write at `/mnt`
2. Clears the immutable flag on any existing `/etc/shadow`, `/etc/shadow-`, `/etc/ssh/sshd_config`
3. Renames those originals to `.bak` (not deleted — a manual revert path exists once you have any shell access)
4. Copies its staged replacements over them
5. Re-applies the immutable flag (`chattr +i`) so a stock OTA update can't silently revert it
6. Blinks a GPIO LED forever — **this is not a success signal**, it happens even if the mount/copy above failed. There's also no auto-reboot; you must power-cycle yourself.

`sshd_config` sets `PermitRootLogin yes` and `PasswordAuthentication yes` —
**both `admin` and `root` get password login**, not just `admin` as the
upstream README implies. Verified by re-hashing against the known salts:
both passwords are `bobcat`.

## What we changed (`patched/` vs `original/`)

The staged `shadow` and `shadow-` files each contained an **undocumented
third account, `easylinkin`**, with an active (non-locked) password hash —
never mentioned anywhere in the upstream README. It's likely inert in
practice (the script never touches `/etc/passwd`, so the username probably
can't resolve to a UID on stock Bobcat firmware), but we removed it from both
files rather than rely on that assumption. `admin`/`root`/`bobcat` were left
as-is, matching the documented behavior.

Verification performed on the patched result before trusting it:
- `e2fsck -fn` clean on the modified partition (an earlier patch attempt
  corrupted the filesystem via a `debugfs` link-count bug — caught by this
  check, redone correctly)
- Byte-for-byte confirmation that the patched partition merged correctly
  into the full disk image
- GPT partition table unchanged (offsets/sizes match original)
- Re-decompressed the final `.xz` and confirmed it matches the verified raw image

## Flashing

1. Insert the SD card via your USB adapter and identify its exact device
   node with `lsblk` (before/after comparison) — **confirm by size before
   writing anything**, never guess.
2. Recommended: **Balena Etcher**, point it at `patched/Bobcat_SSH_285_patched.img.xz`
   directly (it decompresses on the fly) — GUI shows the target drive size
   before you confirm, harder to hit the wrong disk by accident.
3. Alternative: `dd` — only after the device node is jointly confirmed.
4. Insert the SD card into the Bobcat's TF CARD slot, power on, wait
   (no fixed timing — the LED blink means "finished," not "succeeded"),
   power off, remove the SD card, power back on to boot stock firmware normally.

## After flashing

- SSH in: `ssh admin@<device-ip>` or `ssh root@<device-ip>`, password `bobcat`
- **Change these passwords immediately** — they're now public knowledge (this repo's README, and ours)
- **Never expose port 22 beyond your LAN** with these credentials live
- Worth checking `/etc/passwd` for `easylinkin` yourself once you have shell, to close the loop on whether it was ever actually exploitable on this firmware
