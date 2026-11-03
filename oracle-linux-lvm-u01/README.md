# Adding a 30 GB `/u01` Mount Point with LVM on Oracle Linux

Hands-on lab: attaching a new virtual disk to an Oracle Linux VM and presenting it as a persistent `/u01` filesystem using GPT, LVM and ext4. `/u01` is the conventional location for Oracle software installations, so this is a typical first step when preparing a server for an Oracle Database install.

---

## Environment

| Item | Value |
|---|---|
| Hypervisor | Oracle VirtualBox |
| Guest OS | Oracle Linux |
| CPU / RAM | 3 vCPUs / 4 GB |
| OS disk | 20 GB (`/dev/sda`, LVM volume group `ol`) |
| New disk | 30 GB VDI (`/dev/sdb`) |
| Network | 1 × bridged adapter |

## Objective

- Add a dedicated 30 GB disk for `/u01`, kept separate from the OS disk
- Build it on LVM so it can be grown later without downtime
- Make the mount persistent across reboots and verify it

## Storage layout

```
/dev/sdb                      30 GB virtual disk
└── /dev/sdb1                 GPT partition, lvm flag
    └── PV  /dev/sdb1         LVM physical volume
        └── VG  vg_u01        <30.00 GiB volume group
            └── LV  lv_u01    100% of the VG
                └── ext4  →   mounted on /u01 (via /etc/fstab)
```

---

## Steps

### 0. Attach the disk (VirtualBox)
With the VM powered off: **Settings → Storage → SATA controller → Add Hard Disk → Create**, VDI, dynamically allocated, 30 GB.

### 1. Identify the new disk
```bash
lsblk
```
The new disk appears as `sdb` (30G, no partitions, no mount point). Confirming the device name first matters, because the next steps erase whatever disk they are pointed at.

### 2. Create a GPT partition
```bash
parted /dev/sdb --script mklabel gpt mkpart primary 0% 100% set 1 lvm on
echo $?          # 0 = success
lsblk /dev/sdb   # sdb1 now present
```
- `mklabel gpt` writes a new GPT partition table
- `mkpart primary 0% 100%` creates one aligned partition spanning the disk
- `set 1 lvm on` flags partition 1 for LVM use

### 3. Create the LVM physical volume
```bash
pvcreate /dev/sdb1
pvs
```

### 4. Create a dedicated volume group
```bash
vgcreate vg_u01 /dev/sdb1
vgs
```

### 5. Create the logical volume
```bash
lvcreate -n lv_u01 -l 100%FREE vg_u01
lvs
```
`-l 100%FREE` allocates all free extents in the VG to the LV.

### 6. Create the filesystem
```bash
mkfs.ext4 /dev/vg_u01/lv_u01
```

### 7. Mount it
```bash
mkdir -p /u01
mount /dev/vg_u01/lv_u01 /u01
echo $?
df -h
```

### 8. Make it persistent
```bash
cp /etc/fstab /etc/fstab.bak_19-10-26
echo "/dev/mapper/vg_u01-lv_u01   /u01   ext4   defaults   0 0" >> /etc/fstab
```
fstab is backed up first, because a broken entry can stop the system from booting. The LVM mapper path is used instead of `/dev/sdb1` since device names like `sdb` can change between boots.

### 9. Test the fstab entry before rebooting
```bash
umount /u01
systemctl daemon-reload   # systemd caches fstab; reload it after editing
mount -a                  # mounts everything in fstab; errors here = fix before reboot
df -h
```

### 10. Reboot test
```bash
reboot
df -h
```
`/u01` came back automatically after the reboot, confirming the fstab entry.

---

## Verification

Full output: [`output.txt`](output.txt)

```
sdb                 8:16   0   30G  0 disk
└─sdb1              8:17   0   30G  0 part
  └─vg_u01-lv_u01 252:2    0   30G  0 lvm  /u01

  PV         VG     Fmt  Attr PSize   PFree
  /dev/sdb1  vg_u01 lvm2 a--  <30.00g    0

  LV     VG     Attr       LSize
  lv_u01 vg_u01 -wi-ao---- <30.00g

Filesystem                Type  Size  Used Avail Use% Mounted on
/dev/mapper/vg_u01-lv_u01 ext4   30G   28K   28G   1% /u01
```

Note: `Size` is 30G but `Avail` is 28G. ext4 reserves 5% of blocks for root by default (about 1.5 GB here). See *Next steps* for reclaiming it.

---

## Design decisions

| Decision | Reason |
|---|---|
| **Separate disk and VG** (`vg_u01`) instead of extending the OS VG `ol` | Keeps application data independent of the OS disk; `/u01` can be grown, moved or recovered without touching the root filesystem |
| **LVM** rather than a plain partition | Allows online growth later by adding disks to the VG |
| **GPT** rather than MBR | Modern default; no 2 TiB or 4-partition limits; backup table at end of disk |
| **ext4** | Mature and well supported for Oracle software; unlike XFS, it can be shrunk if ever needed |
| **Mapper path in fstab** | Stable across reboots, unlike `/dev/sdX` names |

## Issues encountered

| Issue | Cause | Fix |
|---|---|---|
| `parted` printed its usage text and made no change | Typo: `mklable` instead of `mklabel` | Re-ran with the correct keyword; confirmed with `echo $?` and `lsblk` |
| `lvcreate` rejected the command | Typed `-1` (digit one) instead of `-l` (extents option) | Re-ran with `-l 100%FREE` |
| `ststemctl: command not found` | Typo | Re-ran as `systemctl daemon-reload` |
| Saving output to `oracle-linux-lvm-u01/output.txt` failed | Relative path to a directory that didn't exist in the current working directory | Used an absolute path (`/u01/output.txt`) |

**Lessons:** check exit codes (`echo $?`) after critical steps, back up `/etc/fstab` before editing, and test with `umount` + `mount -a` before rebooting.

---

## Next steps

```bash
# Reclaim most of the ext4 root reserve (5% → 1%) on a data filesystem
tune2fs -m 1 /dev/mapper/vg_u01-lv_u01

# Hand /u01 to the Oracle software owner (once the oracle user and oinstall group exist)
chown -R oracle:oinstall /u01

# Grow /u01 later: attach another disk (e.g. /dev/sdc), then
parted /dev/sdc --script mklabel gpt mkpart primary 0% 100% set 1 lvm on
pvcreate /dev/sdc1
vgextend vg_u01 /dev/sdc1
lvextend -r -l +100%FREE /dev/vg_u01/lv_u01   # -r resizes the filesystem online
```

## Skills demonstrated

- Virtual disk provisioning in VirtualBox
- Disk partitioning with `parted` (GPT)
- LVM administration: `pvcreate`, `vgcreate`, `lvcreate`, `pvs` / `vgs` / `lvs`
- Filesystem creation and mounting (ext4)
- Persistent mounts with `/etc/fstab` and systemd
- Safe change practice: backups, exit-code checks, pre-reboot validation, reboot test
