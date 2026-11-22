# owrt-snap: design and internals

Snapshot and restore the whole writable state of OpenWrt on the GL.iNet Flint 2 (GL-MT6000, eMMC).
A snapshot holds all of `/overlay/upper`: installed packages, the apk database, custom kmods
(MTK Wi-Fi, patched CAKE), configs, scripts and keys. Restoring it needs no reconfiguration.
Written for busybox ash, with no extra packages.

## Install

```sh
ssh root@192.168.1.1 'cat > /usr/sbin/owrt-snap && chmod 755 /usr/sbin/owrt-snap' < owrt-snap
ssh root@192.168.1.1 'cat > /etc/config/owrt-snap' < owrt-snap.config
```

`/etc/config/owrt-snap` sets `store`, the snapshot directory (default `/etc/owrt-snap/store`, on the
root fs so restore can carry it). `create` never captures the store or `/etc/owrt-snap/last-restore`.
It also sets `keep`, the number of snapshots kept after each create
(default 5; 0 keeps all).

## Usage

```
owrt-snap create [-n name] [-i sysupgrade.bin]   # -i bundles the image, only if it matches the running firmware
owrt-snap list                                   # name, date, size, build, image y/n, checksum
owrt-snap verify <name|file>                     # outer + inner sha256, manifest sanity, archive integrity
owrt-snap delete <name>
owrt-snap restore <name|file> [-i sysupgrade.bin] [--force] [--yes] [--no-carry-store]
owrt-snap verify-restore [name|file]             # run after the post-restore reboot
```

Copy snapshots off the router yourself, for example
`scp -O root@192.168.1.1:/etc/owrt-snap/store/NAME.tar.gz .`.

## Snapshot format

`NAME.tar.gz` holds the following, plus a `NAME.tar.gz.sha256` sidecar in the store that `list` uses:

| file | content |
|------|---------|
| `manifest` | board, release, revision, kernel, vermagic, overlay/image sha256, whiteout count, `pkg` lines (`apk info -v`), `world` lines (`/etc/apk/world`) |
| `SHA256SUMS` | hashes of the other members |
| `whiteouts.list` | paths deleted from ROM (overlayfs 0/0 char devices, plus lower entries hidden by opaque dirs) |
| `overlay.tar.gz` | `/overlay/upper` without whiteouts, `tmp`, `dev`, `proc`, `sys`, the store or `last-restore` |
| `image.bin` | optional sysupgrade image, checked byte for byte against the kernel and rootfs partitions at create |

busybox tar can't store xattrs, so opaque dirs are found by their effect: a ROM entry that is
hidden without a whiteout. That entry is recorded as a deletion. None exist on this router.

## Restore flow

1. Copy the snapshot to `/tmp` (refuses if `/tmp` has less than 3× its size + 50 MB free). Check both
   sha256 layers and the manifest.
2. Refuse if the board differs. `--force` can't override this.
3. Choose the image. The bundled image was verified at create. An image given with `-i` must match
   the manifest image hash, or match the running partitions when the snapshot was taken on the
   running build. `--force` accepts an unverified image. With no image at all, restore refuses.
4. Build `restore.tgz`: the overlay contents, plus `/etc/uci-defaults/00-owrt-snap-whiteouts`, plus
   `/etc/owrt-snap/last-restore/{manifest,whiteouts.list,carried.list}`, plus every snapshot in the
   store, so the store comes back after the wipe.
5. The store is carried only when both checks pass. First, the uncompressed restore tar
   (`gzip -cd | wc -c`) plus 50 MB must fit the boot `/tmp`, because fstools unpacks it to
   `/tmp/sysupgrade.tar` in preinit. That tmpfs is unsized, half of RAM, the same as `/tmp` now.
   Second, `/tmp` must have room for image + restore.tgz + the copy sysupgrade makes of it + 50 MB.
   If either check fails, or with `--no-carry-store`, the tarball is rebuilt without the store and
   the prompt names the snapshots that will be lost. Without the store, both checks are mandatory.
   Then `sysupgrade -T -f restore.tgz image` runs, and restore aborts if it fails.
6. Restore prints the snapshot name, build, creation date, image sha256 and store status, then asks
   `Flash and restore? [y/N]`. Any answer other than y or Y aborts with nothing changed. `--yes` skips
   the prompt. Without `--yes`, restore refuses to start when stdin is not a terminal (so use `ssh -t`).
7. Run `sysupgrade -f restore.tgz image`. Stage2 writes the tgz just after the new rootfs. At boot,
   fstools sees the gzip header in the overlay area, unpacks it to `/tmp/sysupgrade.tar` and
   formats the f2fs. Preinit `80_mount_root` then extracts the tar into the new overlay before procd
   starts any service.
8. At boot, `00-owrt-snap-whiteouts` runs first among the uci-defaults (`-` sorts before `_`). It
   deletes every path in the list, logs each one to `/tmp/owrt-snap-restore.log`, deletes itself and
   exits 0. ROM uci-defaults that the snapshot had already consumed are gone before
   `uci_apply_defaults` reaches them.
9. Run `owrt-snap verify-restore`. It checks that every whiteout path is absent, that `apk info -v`
   equals the manifest, that `S25packet_steering` is absent, that no whiteout script is left over,
   and that every carried snapshot is present and passes `verify`. It reports any differences.

Restore always reflashes. Re-writing an identical image changes nothing and uses the standard,
tested sysupgrade path. Staging data under the mounted f2fs and then running firstboot was rejected,
because f2fs can overwrite raw bytes written beneath it before the reboot.

## Limitations

- The store survives a restore only if it is carried (see step 5). Keep off-router copies anyway.
- An image is required for every restore. The running build is the official image
  (`openwrt-25.12.6-mediatek-filogic-glinet_gl-mt6000-squashfs-sysupgrade.bin`, sha256
  `a25a0597…`), so it can be downloaded again.
- `/etc/owrt-snap/last-restore/` stays in the restored overlay. `verify-restore` uses it, and it can
  be deleted afterwards.
- `/etc/rc.d/S25packet_steering` returns from ROM until `00-` deletes it. procd may already have
  queued it, but it does nothing while packet steering is off in the restored config.
- Exclude patterns are globs: a path containing `* ? [` could over-match (none exist).
- A snapshot taken while services write to the overlay captures whatever state those files are in.
- `OWRT_SNAP_UPPER/ROM/SYSINFO/RELEASE/STORE/BOOT_TMP_KB` env overrides exist only for off-router tests.
- Recovery fallback if a flash fails: U-Boot web recovery. Hold reset while powering on, set the PC
  to a static IP in 192.168.1.0/24, then open http://192.168.1.1.
