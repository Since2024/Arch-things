# Infrastructure Log: Offloading Heavy Dev Toolchains to Secondary Partition

**Date:** May 16, 2026  
**OS Environment:** Arch Linux (KDE Plasma, Wayland, PipeWire)  
**Objective:** Resolve storage constraints on the primary root SSD partition by offloading heavy Rust (`.cargo`, `.rustup`) and Solana SBF/BPF toolchains to a secondary 185 GiB partition (`New Volume`), while maintaining identical symlinked path availability for local compilers and IDEs (Neovim/VS Code).

---

## Phase 1: Moving Rust Ecosystem & Establishing Symlinks

### 1. Migrating Data
The primary toolchains were moved out of the local `$HOME` directory onto the external mount point `/run/media/npc/New Volume/`.

### 2. Creating Symbolic Links
To prevent breaking system variables and global binaries, persistent symbolic links were mapped from the home directory back to the new partition destination.

```bash
# Link the Cargo directory
ln -s "/run/media/npc/New Volume/.cargo" ~/.cargo

# Link the Rustup directory
ln -s "/run/media/npc/New Volume/.rustup" ~/.rustup

---

Verification of Links

[npc@npc-new ~]$ ls -ld ~/.cargo ~/.rustup
lrwxrwxrwx 1 npc users 32 May 16 03:38 /home/npc/.cargo -> '/run/media/npc/New Volume/.cargo'
lrwxrwxrwx 1 npc users 33 May 16 03:44 /home/npc/.rustup -> '/run/media/npc/New Volume/.rustup'

[npc@npc-new ~]$ du -sh -L ~/.rustup
# Result: ~5.6 GB offloaded directly to secondary block storage
du: cannot access '/home/npc/.rustup/toolchains/solana'
5.6G    /home/npc/.rustup

----
#### The Resolution:
The broken nested pointer was manually dropped and re-linked to target the active system Solana binary release structure cleanly:
# Remove the old broken link inside the toolchains folder
rm "/run/media/npc/New Volume/.rustup/toolchains/solana"

# Link it directly back to the active local installation runtime package
ln -s /home/npc/.local/share/solana/install/releases/1.18.26/solana-release/bin/sdk/sbf/dependencies/platform-tools/rust "/run/media/npc/New Volume/.rustup/toolchains/solana"

----

Phase 3: System Stability via Desktop Environment Auto-Mounts

Because global configurations (cargo --version) now hard-depend on the secondary volume, running compilers before mounting the file-system would trigger kernel/shell runtime panics.
KDE System Settings Strategy:

    Navigated to System Settings > Disks & Cameras > Device Auto-Mount.

    Enabled global configurations for:

        [✓] On Login (Automatically mount storage devices immediately upon entering KDE Plasma session).

        [✓] On Attach (Mount removable blocks instantly when hot-plugged).

        [✓] Automatically mount removable media that have never been mounted before (Ensure smooth device fallback for external media devices).

Impact: Mitigates risk of runtime storage disconnection, keeping system symlinks valid at execution baseline.