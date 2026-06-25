---
name: qubes-vm-admin
description: "Qubes OS VM lifecycle management — create, clone, remove VMs, bind-dirs persistence, template management, and common administration tasks."
triggers:
  - Qubes create VM
  - Qubes clone VM
  - Qubes remove VM
  - Qubes bind-dirs
  - Qubes persistent storage
  - persist apt packages in AppVM
  - AppVM reboot loses packages
  - Qubes template management
  - Qubes VM resize
  - Qubes qvm-* commands
  - Qubes VM not starting
  - Qubes AppVM lost changes
  - Qubes template update
  - Qubes minimal template
  - Qubes VM administration
  - Qubes persistent storage difference
  - how does Qubes isolation work
  - Qubes dom0 vs domU
  - Qubes VM type differences
related_skills:
  - qubes-qrexec
  - qubes-networking
  - qubes-security
  - qubes-storage
---

# Qubes VM Administration

Covers **persistent storage via bind-dirs** and general VM management.

## VM Types Overview

Qubes has four VM types. Understanding their persistence behavior is critical:

| Type | Root filesystem | Survives reboot? | Typical use |
|------|----------------|-----------------|-------------|
| **AppVM** | Template overlay + COW | `/home` and `/rw` persist; system package changes **lost** | Daily work VM |
| **TemplateVM** | Fully writable | All changes **persist** | Install shared packages |
| **StandaloneVM** | Fully writable | All changes **persist** | Special-purpose OS |
| **DispVM** | Template overlay | **Everything lost** on shutdown | One-off / untrusted tasks |

**Key rule**: `apt install` in an AppVM disappears on reboot. Install in the template or use bind-dirs.

## Persistence Layer

```
AppVM perspective:
  /home/username       ← persistent (user data)
  /usr/local           ← persistent (separate volume; clean spot for persistent non-setuid binaries)
  /rw/config           ← persistent (system config)
  /rw/bind-dirs/       ← persistent (bind-dirs backups)
  --------------------
  /etc/                ← ephemeral (lost on reboot)
  /usr/ (except /usr/local) ← ephemeral (template overlay)
  /                    ← ephemeral (template read-only + overlay)
```

**Practical implications**:
- Install packages → modify template
- Config files (nginx.conf, etc.) → put in `/rw/config/` using bind-dirs
- User data → put in `/home/username/`
- Don't trust anything in `/etc/`
- `/rw` mounts with `nosuid`, so `/usr/bin` cannot be bind-dirs

## Section A: Persistent Storage (bind-dirs)

AppVMs have ephemeral root filesystems. System-level changes are lost on reboot unless persisted via **bind-dirs** (or installed in the template VM).

### ⚠️ Core Rule

**Config directory is `/etc/qubes-bind-dirs.d/`, NOT `/etc/bind-dirs.d/`!**

`/etc/bind-dirs.d/` is NOT read by `bind-dirs.sh`. The script only scans:

```
/usr/lib/qubes-bind-dirs.d/
/etc/qubes-bind-dirs.d/
/rw/config/qubes-bind-dirs.d/
```

### Quick Start

```bash
# 1. Create config (⚠️ must be under /rw, /etc is lost on reboot)
sudo mkdir -p /rw/config/qubes-bind-dirs.d
cat | sudo tee /rw/config/qubes-bind-dirs.d/50_myapp.conf << 'EOF'
binds+=( '/usr/lib/python3' )
EOF

# 2. Initialize (first run snapshots current state to /rw/bind-dirs/)
# Use the init/ path — it exists on every template; the /usr/lib/qubes/bind-dirs.sh
# shortcut is a Debian-only symlink and is absent on Fedora/Arch templates.
sudo /usr/lib/qubes/init/bind-dirs.sh

# 3. Verify (must reboot to confirm)
sudo reboot
# After reboot:
mount | grep bind-dirs
```

⚠️ **Do NOT add `/usr/bin` to binds!** It breaks sudo due to nosuid. See Pitfall #1.

### bind-dirs.sh Behavior

| Command | Effect |
|---------|--------|
| `bind-dirs.sh` (no args) | Read config → if /rw copy doesn't exist, copy → bind mount |
| `bind-dirs.sh umount` | Unmount all bind mounts |

**Key logic**:
1. Scan config files, collect `binds` array
2. For each path `fso_ro`:
   - If `/rw/bind-dirs{fso_ro}` exists → mount directly
   - If not but `fso_ro` exists → `cp --archive` to /rw → mount
   - Neither exists → skip
3. Runs automatically at boot (systemd service)

### bind-dirs Pitfalls

#### 1. 🚨 NEVER bind-dirs `/usr/bin` — breaks sudo!

`/rw` partition mounts with `nosuid` flag. bind-dirs mounting `/usr/bin` inherits `nosuid`, disabling sudo's setuid bit:

```
sudo: effective uid is not 0, is /usr/bin/sudo on a file system with the 'nosuid' option set
```

**Fix**: Remove `/usr/bin` from config, reboot VM. If sudo is already broken, fix from dom0 with `qvm-run -u root`:

```bash
qvm-run -u root <vm-name> 'rm -f /rw/config/qubes-bind-dirs.d/50_myapp.conf'
qvm-shutdown <vm-name> && qvm-start <vm-name>
```

**Alternative**: Install binaries (git, curl, python3-pip) in the template VM, not via bind-dirs.

#### 2. Config must be in persistent location!

`/etc/qubes-bind-dirs.d/` is **temporary**, lost on reboot! Must use:

```bash
# ✅ Persistent (/rw partition, survives reboot)
/rw/config/qubes-bind-dirs.d/50_myapp.conf

# ❌ Temporary (lost on reboot)
/etc/qubes-bind-dirs.d/50_myapp.conf
```

#### 3. Check template first

If the template VM already has the needed packages, no bind-dirs needed:

```bash
dpkg -l <package-name> | grep "^ii"
which <binary>
```

#### 4. Manual run + mount output empty

Running `bind-dirs.sh` via ssh then `mount | grep bind` may show nothing. This is normal — verify after reboot.

#### 5. Template updates may conflict

If template updates packages (e.g., python3 upgrade), old bind-dirs copies override new versions:

```bash
sudo rm -rf /rw/bind-dirs/usr/lib/python3
sudo reboot
```

#### 6. is_fully_persistent check

`bind-dirs.sh` calls `is_fully_persistent` at start. For non-TemplateBasedVM (StandaloneVM), script exits immediately. StandaloneVMs don't need bind-dirs.

### Common Persistent Directories

| Directory | Use | Notes |
|-----------|-----|-------|
| `/usr/lib/python3` | apt/pip Python packages | ✅ Safe |
| `/etc/ssh` | SSH host keys | ✅ Safe |
| `/var/lib/dpkg` | dpkg metadata | ✅ Safe |
| `/usr/share/git-core` | git data files | ✅ Safe |
| `/usr/bin` | ❌ **Don't use!** | nosuid breaks sudo |

**Best practice**: Install base deps (git, curl, python3-pip) in the template VM. Only use bind-dirs for template-unavailable directories.

### Debugging

```bash
mount | grep bind-dirs
sudo find /rw/bind-dirs/ -maxdepth 3
sudo /usr/lib/qubes/init/bind-dirs.sh
sudo rm -rf /rw/bind-dirs/usr/lib/python3  # force re-init
```

## Section B: VM Management Commands

### Create a VM

```bash
# Create an AppVM from a template
qvm-create --template debian-13-xfce <vm-name>

# Create a StandaloneVM (no template dependency)
qvm-create --standalone <vm-name>

# Create a DispVM template
qvm-create --template <template> --property template_for_dispvms=True <dispvm-name>
```

### Clone a VM

```bash
qvm-clone <source-vm> <new-vm-name>
```

### Remove a VM

```bash
qvm-remove <vm-name>
```

### Set VM properties

```bash
qvm-prefs <vm-name> memory 2048
qvm-prefs <vm-name> maxmem 4096
qvm-prefs <vm-name> vcpus 4
qvm-prefs <vm-name> netvm sys-firewall
qvm-prefs <vm-name> provides_network True  # make a ProxyVM
```

### Template management

```bash
# List all templates
qvm-ls --all | grep TemplateVM

# Update template (run in the template VM)
sudo apt update && sudo apt upgrade

# After updating template: shutdown and restart AppVMs to pick up changes
qvm-shutdown <appvm> && qvm-start <appvm>

# Create a custom template from existing
# (cloning a TemplateVM already yields a TemplateVM — no extra step needed)
qvm-clone debian-13-xfce my-custom-template
```

### VM Storage

```bash
# Extend private storage
qvm-volume extend <vm-name>:private 4G

# Resize filesystem inside the VM (Qubes private volume is /dev/xvdb, not /dev/sdaN)
# Often unnecessary — Qubes auto-grows the private volume on boot.
sudo resize2fs /dev/xvdb

# Check actual disk usage (in dom0)
sudo lvs -o +data_percent
```

## Section C: dom0 Operations Reference

Commands that can only run in dom0:

```bash
# List all VMs
qvm-ls

# Start/stop/restart VMs
qvm-start <vm-name>
qvm-shutdown <vm-name>
qvm-kill <vm-name>

# Run a command in a VM (from dom0)
qvm-run <vm-name> "command"
qvm-run --pass-io <vm-name> "command with output"

# Backup and restore
qvm-backup --destination /path/to/backup <vm-name>
qvm-backup-restore /path/to/backup

# Firewall rules (dom0 sets these, VM cannot override)
qvm-firewall <vm-name> list
qvm-firewall <vm-name> add accept proto=tcp dsthost=example.com dstports=443
```

## Common Pitfalls

**⚠️ /etc is ephemeral**

Don't edit files in `/etc/` and expect them to survive a reboot. Use bind-dirs or put configs in `/rw/config/`.

**⚠️ qvm-ls is dom0-only**

You cannot run `qvm-ls` from inside an AppVM. Always ask the user to run it in dom0.

**⚠️ Template updates need AppVM restart**

Installing a package in the template is not enough — affected AppVMs must be restarted to see the change.

**⚠️ Memory overcommit**

Setting `maxmem` too high across many VMs can exhaust dom0 memory. Xen starts swapping and all VMs slow down. Monitor with `xl list` and keep headroom.

## Verification

```bash
# Check VM is running
qvm-run <vm-name> "hostname"

# Verify persistence after reboot
qvm-run --pass-io <vm-name> "cat /rw/config/qubes-bind-dirs.d/50_myapp.conf"
qvm-shutdown <vm-name>
qvm-start <vm-name>
qvm-run --pass-io <vm-name> "mount | grep bind-dirs"

# Check available templates
qvm-ls --all | grep TemplateVM

# Check pool disk usage (dom0)
sudo lvs -o +data_percent | head -10
```

## References

- Official bind-dirs docs: https://www.qubes-os.org/doc/bind-dirs/
- Official qvm-* docs: https://www.qubes-os.org/doc/how-to-use-qvm-run/
- Script source: `/usr/lib/qubes/init/bind-dirs.sh`
- For advanced persistence patterns, see `qubes-qrexec` skill
- For storage management, see `qubes-storage` skill
