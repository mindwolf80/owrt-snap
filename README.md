# owrt-snap: router snapshots in one command

`owrt-snap` saves the complete state of the router to a single file and can later put the router
back exactly as it was in that file. The router is a GL.iNet Flint 2 (GL-MT6000) running
OpenWrt 25.12.6.

A snapshot covers everything you've changed since the firmware was first flashed:

- every setting (Wi-Fi, network, DNS/stubby, adblock, SQM/CAKE, firewall)
- every installed package, including the custom MediaTek Wi-Fi driver and the patched CAKE module
- passwords, SSH keys and your own scripts

After a restore, nothing needs to be reinstalled or reconfigured.

Use it before a risky change, such as a new Wi-Fi driver build, a package upgrade or a config
experiment. If the change goes wrong, one command puts the router back.

> Technical details (file format, how the restore works internally, design choices) are in
> [DESIGN.md](DESIGN.md).

---

## Quick start

The commands below run on the router. Log in first:

```sh
ssh root@192.168.1.1
```

### 1. Take a snapshot

```sh
owrt-snap create -n before-driver-test
```

The name is optional. Without one, it uses the date and time. It takes about 12 seconds, and the
file is about 13 MB.

To also bundle the firmware image, so the snapshot can be restored without any other file, add
`-i`. That makes it about 23 MB.

```sh
owrt-snap create -n before-driver-test -i /tmp/openwrt-25.12.6-mediatek-filogic-glinet_gl-mt6000-squashfs-sysupgrade.bin
```

### 2. See your snapshots

```sh
owrt-snap list
```

```
NAME                DATE                  SIZE_KB  BUILD               IMG  SUM
pre-test-20261116   2026-11-16T13:48:22Z  23192    r33311-981c978eff   y    ok
```

`IMG y` means the firmware image is inside the snapshot. `SUM ok` means the file is undamaged.

### 3. Copy it to your PC (do this every time)

Run this on the PC, not on the router:

```sh
scp -O root@192.168.1.1:/etc/owrt-snap/store/before-driver-test.tar.gz ~/Downloads/
```

A copy on the router alone is not a backup. If the router dies, the copy dies with it.

### 4. Restore

Connect over a **LAN cable**, not Wi-Fi, because the Wi-Fi goes down during the restore.

```sh
ssh -t root@192.168.1.1 owrt-snap restore before-driver-test
```

Before it changes anything, it shows what it is about to do and asks for confirmation:

```
Snapshot:   before-driver-test (created 2026-11-16T13:48:22Z)
Firmware:   25.12.6 r33311-981c978eff, vermagic f0a7…
Image:      sha256 a25a0597…
Restore:    restore.tgz 36040 KB, unpacks to 56374 KB of 505016 KB boot /tmp
Store:      carried forward (1 snapshots: before-driver-test)
Flash and restore? [y/N]
```

Type `y` and press Enter. Any other answer cancels with nothing changed.

The router then reflashes and reboots. It is back in about **1.5 minutes**. The 5 GHz Wi-Fi takes
about another minute, because it runs a radar (DFS) check first.

If the snapshot has no bundled image, give it the firmware file:
`owrt-snap restore NAME -i /tmp/openwrt-…-sysupgrade.bin`.

### 5. Check the result

```sh
owrt-snap verify-restore
```

It should end with `verify-restore: OK`. Then you can remove its leftover notes:

```sh
rm -rf /etc/owrt-snap/last-restore
```

---

## All commands

| Command | What it does |
|---|---|
| `owrt-snap create [-n NAME] [-i IMAGE]` | Take a snapshot. `-i` bundles the firmware image. |
| `owrt-snap list` | Show the snapshots in the store. |
| `owrt-snap verify NAME` | Check a snapshot file is complete and undamaged. |
| `owrt-snap delete NAME` | Delete a snapshot from the store. |
| `owrt-snap restore NAME [-i IMAGE]` | Put the router back to that snapshot (asks first). |
| `owrt-snap verify-restore [NAME]` | After a restore, confirm everything came back. |

Restore options:

| Option | Effect |
|---|---|
| `--yes` | Skip the `[y/N]` question (for scripts). |
| `--no-carry-store` | Don't bring the other snapshots along through the restore. |
| `--force` | Accept a firmware image the tool can't verify. Avoid unless you know why. |

`NAME` can also be a path to a snapshot file, for example one copied back from the PC into `/tmp`.

---

## Where things live

| What | Where |
|---|---|
| The tool | `/usr/sbin/owrt-snap` on the router; source in this folder |
| Settings | `/etc/config/owrt-snap` (`store` folder, `keep` = how many snapshots to keep, default 5) |
| Snapshots on the router | `/etc/owrt-snap/store/` |
| Off-router copy of the tested snapshot | `~/Downloads/owrt-snap-test-2026-11-16/pre-test-20261116.tar.gz` |
| Firmware image | the same folder, `openwrt-25.12.6-…-sysupgrade.bin` (official image, sha256 `a25a0597…`) |

When `create` runs and there are already more than `keep` snapshots, it deletes the oldest ones.

---

## Installing on a router

Run these on the PC, from this folder:

```sh
ssh root@192.168.1.1 'cat > /usr/sbin/owrt-snap && chmod 755 /usr/sbin/owrt-snap' < owrt-snap
ssh root@192.168.1.1 'cat > /etc/config/owrt-snap' < owrt-snap.config
```

It needs no extra packages. It uses only what OpenWrt already includes.

---

## Good to know

- **Restore always reflashes the firmware.** That is intended: re-writing the same firmware is
  harmless and uses OpenWrt's normal, well-tested upgrade path. It's also why a firmware image
  is needed.
- **Snapshots only restore onto the same model** (GL-MT6000). The tool refuses any other board,
  and `--force` does not override this.
- **Snapshots survive a restore** if there is room in memory. The tool checks this and warns you,
  naming the snapshots that would be lost.
- **5 GHz may come back on channel 40 instead of 36.** The config still says 36. The radio picks
  40 inside the same 160 MHz block during its startup radar check. Nothing is wrong.
- **Don't use `apk upgrade` or Attended Sysupgrade** on this router. They replace the custom
  MediaTek Wi-Fi packages with stock ones.

---

## If something goes badly wrong

If the router doesn't come back after about 10 minutes, use the GL.iNet U-Boot recovery page. It
is built into the router and works even when OpenWrt is broken.

1. Unplug the router's power. Connect the PC by cable to a LAN port.
2. Set the PC's wired network to the static address **192.168.1.2**, mask 255.255.255.0,
   gateway 192.168.1.1.
3. Hold the **reset** button, plug the power back in, and keep holding for about 10 seconds,
   until the LED blinks and then stays solid.
4. Open **http://192.168.1.1** in a browser, upload the `openwrt-25.12.6-…-sysupgrade.bin` file
   and start the update. Don't unplug it until it has rebooted.
5. Set the PC back to automatic (DHCP). Then reinstall `owrt-snap` (see "Installing on a router"),
   copy your snapshot to the router's `/tmp` and run
   `owrt-snap restore /tmp/NAME.tar.gz`.

---

## Tested

These tests ran on 2026-11-16. The router restored `pre-test-20261116` and was reachable again
93 seconds after the flash started. Afterwards:

- all 243 packages were identical
- CAKE/SQM, stubby DNS-over-TLS, adblock and the network were running as before
- the MediaTek driver version was unchanged
- the test change (a system description) was reverted

The logs are in `~/Downloads/owrt-snap-test-2026-11-16/`.
