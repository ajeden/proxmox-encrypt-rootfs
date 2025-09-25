# **Goal:** 
Encrypt `/dev/pve/root` in place with LUKS2, boot via systemd-boot (`proxmox-boot-tool`), unlock in **initramfs** (passphrase or keyfile).

## Credits
Partially based on:
https://blog.alex.balgavy.eu/encrypting-an-existing-linux-partition-with-luks/
and a lot of troubleshooting on the go with Proxmox 9.

# 0) On the running (unencrypted) PVE (optional but recommended)

```sh
apt update apt install -y cryptsetup cryptsetup-bin cryptsetup-initramfs
```

Backup anything important.
# 1) Boot SystemRescue **in UEFI mode**

List layout:
```sh
lsblk -o NAME,LABEL,SIZE,TYPE,FSTYPE,UUID,MOUNTPOINTS`
```
You should see LV `pve-root` with `ext4`, an ESP (FAT/vfat), and possibly `pve-swap`.

Disable swap (SystemRescue may auto-enable it):
```sh
swapoff -a
```

# 2) Check & shrink ext4 to make room for the LUKS header

```sh
e2fsck -f /dev/pve/root resize2fs -M /dev/pve/root
```

# 3) In-place convert LV → LUKS2 (reserve 16 MiB header)

```sh
cryptsetup-reencrypt --encrypt --type luks2 --reduce-device-size 16M /dev/pve/root
```
Set a strong passphrase. This runs for a while.
# 4) Open the new LUKS container and mount the system

```sh
cryptsetup luksOpen /dev/pve/root rootfs
```

This creates `/dev/mapper/rootfs`
If the LV inside is plain ext4 (typical): mount /dev/mapper/rootfs /mnt  
If you actually have LVM-inside-LUKS (rare in this flow): 
```sh
vgchange -ay
mount /dev/pve/root /mnt
```

Mount the ESP (adjust device, e.g. /dev/sda2 or /dev/nvme0n1p2):

```sh
mkdir -p /mnt/boot/efi mount /dev/sda2 /mnt/boot/efi
```

Bind & chroot:

```sh
for d in /dev /dev/pts /proc /sys /run; do mount --rbind "$d" "/mnt$d"; done 
chroot /mnt
```

# 5) Configure unlocking (choose ONE)

## A) **Passphrase prompt** (no keyfile)

Get the **LUKS** UUID:

```sh
LUKS_UUID=$(cryptsetup luksUUID /dev/pve/root)
```

In `/etc/crypttab`:

```sh
printf "rootfs  UUID=%s  none  luks,discard\n" "$LUKS_UUID" >> /etc/crypttab
```
## B) **Auto-unlock with keyfile** (key embedded in initramfs)

> [!warning] This is unsafe option, as key is unencrypted in initramfs.

Create a key and add to LUKS:

```sh
mkdir -p /etc/cryptsetup-keys.d 
chmod 700 /etc/cryptsetup-keys.d 
dd if=/dev/urandom of=/etc/cryptsetup-keys.d/boot_os.key bs=4096 count=1
chmod 400 /etc/cryptsetup-keys.d/boot_os.key 
cryptsetup luksAddKey /dev/pve/root /etc/cryptsetup-keys.d/boot_os.key LUKS_UUID=$(cryptsetup luksUUID /dev/pve/root)
```

Tell initramfs to include keys (either one is fine; using both is safest):

1) Explicit path pattern 
```sh 
echo 'KEYFILE_PATTERN="/etc/cryptsetup-keys.d/*.key"' > /etc/cryptsetup-initramfs/conf-hook 
```
2) And mark the mapping as needing the key in initramfs: 
```sh
printf "rootfs  UUID=%s  /etc/cryptsetup-keys.d/boot_os.key  luks,discard,initramfs\n" "$LUKS_UUID" >> /etc/crypttab
```
# 6) fstab for the **unlocked** root + ESP

Get the ext4 UUID of the unlocked root:

```sh
ROOTFS_UUID=$(blkid -s UUID -o value /dev/mapper/rootfs)
```

`/etc/fstab`:

```sh
# <file system>           <mount point>  <type> <options>                 <dump> <pass> 
UUID=${ROOTFS_UUID} / ext4 errors=remount-ro 0 1 
UUID=5AD3-D05D /boot/efi vfat defaults 0 1 
/dev/pve/swap none swap sw 0 0 
proc /proc proc defaults 0 0
```

(Replace the ESP UUID with yours if different.)

# 7) Initramfs hardening + kernel cmdline for systemd-boot

Make UMASK 0077 so embedded keys aren’t world-readable 
```sh 
sed -i 's/^UMASK=.*/UMASK=0077/' /etc/initramfs-tools/initramfs.conf || echo 'UMASK=0077' >> /etc/initramfs-tools/initramfs.conf  
```

Make sure systemd-boot entries point to the new root:
```sh
echo root=/dev/mapper/rootfs ro' > /etc/kernel/cmdline
```

# 8) Initialize/refresh Proxmox boot (systemd-boot path)

`proxmox-boot-tool` wants to own the ESP mount; unmount it first **inside the chroot**:

```sh
umount /boot/efi proxmox-boot-tool init /dev/sda2      
```
or your ESP (e.g. /dev/nvme0n1p2) proxmox-boot-tool refresh
# 9) Build initramfs (and let PVE copy it to ESP)

```sh
update-initramfs -u -k all proxmox-boot-tool refresh
```

> [!info] You do **not** need `GRUB_ENABLE_CRYPTODISK=y` when booting via systemd-boot.

# 10) Verify before reboot

Unpack initramfs and check:

```sh
tmp=$(mktemp -d)
cd "$tmp" 
unmkinitramfs /boot/initrd.img-$(uname -r) .  
```
Mapping present? 
```sh
cat main/conf/conf.d/cryptroot  
```
If keyfile path used, key should be copied: 
```sh
find main/cryptroot -maxdepth 2 -type f -ls || true
```

Check PVE boot status:

```sh
proxmox-boot-tool status 
cat /etc/kernel/proxmox-boot-uuids
```
# 11) Cleanup & reboot
Exit chroot and unmount in reverse:

```sh
exit 
for d in /run /sys /proc /dev/pts /dev; do umount -R "/mnt$d" 2>/dev/null || true; done 
umount -R /mnt/boot/efi 2>/dev/null || true umount -R /mnt 
cryptsetup luksClose rootfs 
reboot
```

---

# Troubleshooting notes (kept short)

- **Booted without passphrase?** You embedded a keyfile (expected). To require a prompt: in `/etc/crypttab` use `none` (no key path), remove any forced `KEYFILE_PATTERN`, rebuild initramfs, `proxmox-boot-tool refresh`.
    
- **GRUB “invalid passphrase”:** Ignore GRUB for unlocking; systemd-boot + initramfs supports LUKS2/Argon2. If you insist on GRUB unlocking, add a **PBKDF2** keyslot and preload `cryptodisk,luks2,lvm` in GRUB.
    
- **UMASK warning from `update-initramfs`:** set `UMASK=0077` (done above).
    
- **Wrong UUID errors:**
    
    - `crypttab` → **LUKS** UUID (`cryptsetup luksUUID /dev/pve/root`)
        
    - `fstab` → **ext4** UUID from `/dev/mapper/rootfs` (or use `/dev/mapper/rootfs` directly)

## Handy GRUB commands:
`cryptomount -a` tries to unlock all crypto paritions GRUB sees. `set pager=1` gives `| less` output `set debug=loader,linux,fs,crypt` gives you verbose output on console `normal` tries to boot normally
