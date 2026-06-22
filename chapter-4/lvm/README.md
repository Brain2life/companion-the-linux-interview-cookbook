# LVM: Extending Disk Space for a Database Server

## Scenario

The database team reports that their PostgreSQL data directory (`/pgdata`) is running low on disk space and applications are at risk of failing. As the DevOps engineer on call, you SSH into the server, analyse the current LVM setup, and extend the storage without any downtime.

The server already has one LVM-backed volume in production:

| Device | Role | Volume Group | Logical Volume | Mount |
|--------|------|--------------|----------------|-------|
| `/dev/sdb` | existing data disk | `vg_data` | `lv_pgdata` (4 GB) | `/pgdata` |
| `/dev/sdc` | new disk (raw) | — | — | — |
| `/dev/sdd` | new disk (raw) | — | — | — |

---

## Prerequisites

| Tool | Tested version |
|------|---------------|
| [Vagrant](https://www.vagrantup.com/) | ≥ 2.3 |
| [VirtualBox](https://www.virtualbox.org/) | ≥ 7.0 |

---

## Quick Start

```bash
# Clone the companion repo (if you haven't already)
git clone https://github.com/your-org/companion-the-linux-interview-cookbook.git
cd companion-the-linux-interview-cookbook/chapter-3/lvm

# Bring up the VM (~3–5 minutes first run)
vagrant up

# SSH into the server
vagrant ssh
```

Three `.vdi` disk image files (`extra_disk_sdb.vdi`, `extra_disk_sdc.vdi`, `extra_disk_sdd.vdi`) will be created in the same directory as the `Vagrantfile`. These are ignored by `.gitignore` — do not commit them.

---

## Exercises

### Exercise 1 – Analyse the Current State

Before touching anything, understand what you're working with.

```bash
# Overall block device layout
lsblk

# Disk space usage
df -h

# LVM layers
sudo pvs          # physical volumes
sudo vgs          # volume groups
sudo lvs          # logical volumes

# Detailed views
pvdisplay
vgdisplay vg_data
lvdisplay /dev/vg_data/lv_pgdata
```

**Questions to answer before proceeding:**
- How much free space remains in `vg_data`?
- What percentage of `lv_pgdata` is currently used?
- Which raw disks are available (`lsblk` with no MOUNTPOINTS)?

---

### Exercise 2 – Extend the Existing Logical Volume

The database team needs `/pgdata` to be larger. Add `/dev/sdc` to the existing `vg_data` volume group, then grow the logical volume and its filesystem — all while the volume remains mounted.

**Step 1 — Create a Physical Volume on the new disk**

```bash
sudo pvcreate /dev/sdc
```

Verify:

```bash
sudo pvs
# /dev/sdc should now appear with all space in the PFree column
```

**Step 2 — Add the Physical Volume to the Volume Group**

```bash
sudo vgextend vg_data /dev/sdc
```

Verify that `VFree` in the VG has grown:

```bash
sudo vgdisplay vg_data | grep -E "VG Size|Alloc|Free"
```

**Step 3 — Extend the Logical Volume**

Extend to use all newly available free space in the VG:

```bash
sudo lvextend -l +100%FREE /dev/vg_data/lv_pgdata
```

Or extend by a specific size (e.g., 4 GB):

```bash
sudo lvextend -L +4G /dev/vg_data/lv_pgdata
```

**Step 4 — Resize the Filesystem**

The logical volume is now larger, but the filesystem does not know yet.

For **ext4** (used in this lab):

```bash
sudo resize2fs /dev/vg_data/lv_pgdata
```

For **XFS** (resize while mounted, requires `-d` flag):

```bash
sudo xfs_growfs /pgdata
```

**Step 5 — Verify**

```bash
df -h /pgdata
sudo lvdisplay /dev/vg_data/lv_pgdata
```

The filesystem size shown by `df` should now reflect the additional space.

---

### Exercise 3 – Create a Dedicated Log Volume (Bonus)

The database team also wants a separate volume for WAL logs to prevent log growth from starving the data directory.

```bash
# Create PV on the third raw disk
sudo pvcreate /dev/sdd

# Add it to the same VG (or create a new one — your choice)
sudo vgextend vg_data /dev/sdd

# Create a 4 GB logical volume for WAL logs
sudo lvcreate -L 4G -n lv_pglogs vg_data

# Format and mount
sudo mkfs.ext4 /dev/vg_data/lv_pglogs
sudo mkdir -p /pgdata/pg_wal_dedicated
sudo mount /dev/vg_data/lv_pglogs /pgdata/pg_wal_dedicated

# Make the mount persistent
echo "/dev/vg_data/lv_pglogs  /pgdata/pg_wal_dedicated  ext4  defaults  0 2" \
  | sudo tee -a /etc/fstab
```

Verify the full picture:

```bash
lsblk
df -h
pvs && vgs && lvs
```

You should see:
```bash
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0                       7:0    0 91.7M  1 loop /snap/lxd/38800
loop1                       7:1    0   87M  1 loop /snap/lxd/29351
loop2                       7:2    0 63.9M  1 loop /snap/core20/2318
loop3                       7:3    0 38.8M  1 loop /snap/snapd/21759
loop4                       7:4    0 49.3M  1 loop /snap/snapd/26865
sda                         8:0    0   64G  0 disk 
├─sda1                      8:1    0    1M  0 part 
├─sda2                      8:2    0    2G  0 part /boot
└─sda3                      8:3    0   62G  0 part 
  └─ubuntu--vg-ubuntu--lv 253:1    0   31G  0 lvm  /
sdb                         8:16   0    5G  0 disk 
└─vg_data-lv_pgdata       253:0    0   10G  0 lvm  /pgdata
sdc                         8:32   0    5G  0 disk 
└─vg_data-lv_pgdata       253:0    0   10G  0 lvm  /pgdata
sdd                         8:48   0    5G  0 disk 
└─vg_data-lv_pglogs       253:2    0    4G  0 lvm  /pgdata/pg_wal_dedicated
```

---

### Exercise 4 – Make All Mounts Persistent

Confirm `/etc/fstab` is correct and survives a reboot.

```bash
cat /etc/fstab

# Test fstab without rebooting
sudo umount /pgdata/pg_wal_dedicated
sudo mount -a          # mounts everything listed in fstab
df -h /pgdata/pg_wal_dedicated
```

Reboot and re-verify:

```bash
# From your host machine
vagrant reload
vagrant ssh
df -h
```

You should see:
```bash
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                               96M  1.1M   95M   2% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   31G  5.0G   24G  18% /
tmpfs                              479M     0  479M   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
/dev/sda2                          2.0G  131M  1.7G   8% /boot
/dev/mapper/vg_data-lv_pgdata      9.8G  563M  8.8G   6% /pgdata
/dev/mapper/vg_data-lv_pglogs      3.9G   24K  3.7G   1% /pgdata/pg_wal_dedicated
vagrant                            915G  149G  767G  17% /vagrant
tmpfs                               96M  8.0K   96M   1% /run/user/1000
```

---

## LVM Command Reference

| Goal | Command |
|------|---------|
| Initialise a disk as a PV | `pvcreate /dev/sdX` |
| List physical volumes | `pvs` / `pvdisplay` |
| Create a volume group | `vgcreate vg_name /dev/sdX` |
| Add a PV to an existing VG | `vgextend vg_name /dev/sdX` |
| List volume groups | `vgs` / `vgdisplay` |
| Create a logical volume (fixed size) | `lvcreate -L 4G -n lv_name vg_name` |
| Create a logical volume (all free space) | `lvcreate -l 100%FREE -n lv_name vg_name` |
| Extend an LV by size | `lvextend -L +2G /dev/vg_name/lv_name` |
| Extend an LV using all VG free space | `lvextend -l +100%FREE /dev/vg_name/lv_name` |
| List logical volumes | `lvs` / `lvdisplay` |
| Resize ext4 filesystem | `resize2fs /dev/vg_name/lv_name` |
| Resize XFS filesystem | `xfs_growfs /mount/point` |
| Scan for all LVM metadata | `vgscan` / `lvscan` |

---

## What Interviewers Look For

- **Order of operations**: `pvcreate` --> `vgcreate`/`vgextend` --> `lvcreate`/`lvextend` --> filesystem resize. Skipping or reversing steps is a common mistake.
- **Live resize**: Knowing that `lvextend` + `resize2fs` can be done on a mounted ext4 filesystem (online resize), while XFS uses `xfs_growfs`.
- **`-l` vs `-L`**: `-L` takes an absolute size (`4G`), `-l` takes extents or percentages (`+100%FREE`).
- **fstab persistence**: Candidates often forget to update `/etc/fstab`, leaving the mount missing after the next reboot.
- **Safety**: Always run `pvdisplay`/`vgdisplay` before and after each step to confirm the change took effect.

---

## Cleanup

```bash
# From your host machine — destroys the VM and all disk images
vagrant destroy -f
rm -f extra_disk_sdb.vdi extra_disk_sdc.vdi extra_disk_sdd.vdi
```

---

## Support 

If you find this exercise valuable, consider supporting it on Ko-fi:

<a href='https://ko-fi.com/B0B61Z2ERC' target='_blank'><img height='36' style='border:0px;height:36px;' src='https://storage.ko-fi.com/cdn/kofi6.png?v=6' border='0' alt='Buy Me a Coffee at ko-fi.com' /></a>