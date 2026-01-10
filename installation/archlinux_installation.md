Arch Linux installation *guide* for a minimal home server with LUKS encrypted disk (not including `boot` and `efi`) + TPM2 auto-unlock.

> **[DISCLAIMER]**
> This guide assumes that your machine is compatible with TPM2. 
>   For checking compatibility with TPM2, read the documentation on *ArchWiki*.
> This guide includes commands to follow and explains them, but it is recommended to actually research about each command if you don't already recognize/understand them.
> This installation does NOT include desktop environments and is designed for balancing minimalism with security and functionality.  

# ISO Flashing
1. Download latest official Arch Linux ISO from the official website. 
    - It is recommended to confirm the hash and signature after downloading.
2. Flash a USB drive or other installation media with the ISO.
    - Recommended tool: `Rufus`

# Installation
1. Attach the installation media to the machine and enter the boot menu. 
2. Boot the installation media.

## Connecting to Wi-Fi
1. Connect to the network with `iwctl` or `mmcli`. 
    - If you're having trouble with `iwctl` regarding SSIDs and special characters, check out `installation/troubleshooting/iwctl_ssid_trouble.md`
2. It is recommended to upgrade `openssh` if planning to continue using it.
    - `pacman -Sy openssh --noconfirm`

## Setting up SSH (optional)
1. Make some security configurations to SSH before proceeding.
```
/etc/ssh/sshd_config

# Make these changes:
Port {port}
MaxAuthTries            3
MaxSessions             1
PasswordAuthentication  no
```
> [!NOTE]
> `{port}` is a placeholder for a port number!!
2. Restart `sshd.service` until the listening port updates.
```
# Repeat until the listening port updates:
root@archiso ~ # systemctl restart sshd && systemctl status sshd
```
3. Generate key-pairs on local machine (if haven't already) transfer the public key.
> [!TIP]
> `netcat` can be used to quickly transfer *public* keys. Don't use netcat to transfer sensitive information.
```
root@archiso ~ # netcat -lvnp {port} > ~/.ssh/authorized_keys

local@machine ~ # ssh-keygen -t ed25519 -C "archlinux"
local@machine ~ # cat ~/.ssh/archlinux.pub | netcat {machine_ip} {port} -q 0
```
4. `ssh` into the remote machine.

> [!NOTE]
> You could also use password authentication instead of pubkey authentication. This, of course, is less secure,
> but is more convenient (especially if you keep messing up and restarting the installation). 
> The key pair generated on the local machine _(that is to say **not** the machine you are currently installing Arch on)_ can be reused later when Arch is installed.

## Disk Partitioning and Formatting
1. Make partitions for boot, efi, and main filesystem using any partitioning tool.
    - Partition 1 (efi): 1G
    - Partition 2 (boot): 1G 
    - Partition 3 (main filesystem): remaining (929.5G in my case)
2. Change partition type of main filesystem to `44` or `Linux LVM`.
> The disk should look like this:
```
sda       8:0    0 931.5G  0 disk
├─sda1    8:1    0     1G  0 part
├─sda2    8:2    0     1G  0 part
└─sda3    8:3    0 929.5G  0 part
```

3. Format the partitions except for the main filesystem (for now)
```
root@archiso ~ # mkfs.fat -F32 /dev/sda1
root@archiso ~ # mkfs.ext4 /dev/sda2
```
> [!NOTE]
> It is also possible to encrypt the `/boot` partition, but that requires extra configuration.

## Encrypted LVM Formatting
1. Format the LVM partition with LUKS.
2. Open the LUKS partition.
3. Configure the volumes.
4. Probe device mapper module.
    - Scan for the volume to confirm; then
    - activate the volume.
5. Format the logical volume.
6. (Optionally add swap)

```
root@archiso ~ # cryptsetup luksFormat /dev/sda3
root@archiso ~ # cryptsetup open --type luks /dev/sda3 lvm
root@archiso ~ # pvcreate /dev/mapper/lvm
root@archiso ~ # vgcreate volgroup0 /dev/mapper/lvm 
root@archiso ~ # lvcreate -L 900G -n lv_root volgroup0
root@archiso ~ # modprobe dm_mod
root@archiso ~ # vgchange -ay
root@archiso ~ # mkfs.ext4 /dev/volgroup0/lv_root

# Optional
root@archiso ~ # lvcreate -L 4G -n lv_swap volgroup0
root@archiso ~ # mkswap /dev/volgroup0/lv_swap
```

## Mounting & Configuration
1. Mount the respective partitions and swap
    - `lv_root` should be mounted at `/mnt`;
    - `sda2` (or whatever boot partition) should be mounted at `/mnt/boot`;
    - `swap` (or in this case, `lv_swap`) should be mounted with `swapon {swap_partition}`

> [!NOTE]
> `lvm` (found in `cryptsetup open` line), `volgroup0` (found in `vgcreate` line), and volumes starting with `lv_` are *conventional names*. That is to say that they can be replaced with whatever you want.   

2. Run `pacstrap -K base /mnt linux linux-firmware` to install base packages.
    - An error stating that `/etc/vconsole.conf` is not found might appear after installing the kernel.
    - This is fine but has to be fixed later.
3. Generate fstab and verify results. 
```
root@archiso ~ # mount /dev/volgroup0/lv_root /mnt
root@archiso ~ # mount /dev/sda2 /mnt/boot

# If you have swap
root@archiso ~ # swapon /dev/volgroup0/lv_swap

root@archiso ~ # pacstrap -K base /mnt linux linux-firmware

root@archiso ~ # genfstab -U /mnt >> /mnt/etc/fstab

# Verifying information
root@archiso ~ # cat /mnt/etc/fstab
```
> [!NOTE]
> The `fstab` file contains information about the mounted partitions in your machine's `hard disk`. 
> The `-U` flag generates UUIDs for each device, which will be useful later.

## Arch-Chroot
1. Make a `passwd` for the root user.

### Installing essential packages
2. Install essential packages
    - Packages can be installed with `pacman -S package1 package2`
    - Essential list:
        - grub, efibootmgr, lvm2, sudo 
    - Personally essential:
        - git, vim, networkmanager, man, fd, dosfstools, openssh, zsh, gpg, fzf

### Preparing Kernel and Initramfs
3. Add necessary hooks
    - `sd-encrypt lvm2` ALWAYS after `block`, and before `filesystems`, **in that order**. 
    - `sd-encrypt` is **necessary** for `TPM2` to work, among other things.
4. Run `mkinitcpio -P` or `mkinitcpio -p linux`. 
    - If it is looking for file `/etc/vconsole.conf`, it is likely empty or does not exist.
    - If it is either of the conditions above, just run `echo "KEYMAP=us > /etc/vconsole.conf`
        - or whatever keymap you would like.

### Customizations
5. Set time and locale.
    - Timezone can be set with `ln -sf /usr/share/zoneinfo/{Area}/{Location} /etc/localtime`
    - locale can be set in `/etc/locale.gen` by uncommenting a locale (e.g. `en_us.UTF-8 UTF-8`) and running `locale-gen`
    - Update the `LANG` variable in `locale.conf` accordingly
    - (e.g. using `en_US.UTF-8`)
    ```
    /etc/locale.conf
    
    LANG=en_US.UTF-8
    ```
6. Edit `/etc/hostname` accordingly.

### Installing and Configuring GRUB
7. Configure GRUB
    - Add the `root` values to the `LINUX` line.
    ```
    /etc/default/grub
    
    GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 root=/dev/volgroup0/lv_root resume={lv_swap_UUID} quiet"
    ```
    - If you have swap, you may also set `resume={swap_UUID}`.
8. Mount the first partition (efi)
    - `systemctl daemon-reload` might need to be called first; this command must be ran in the installer environment
9. Install grub/efi
    - `grub-install --target=x86_64-efi --bootloader-id=grub_uefi --recheck`
12. Copy locale to grub
    - `cp /usr/share/locale/en\@quot/LC_MESSAGES/grub.mo /boot/grub/en.mo`
13. Make grub config file
    - `grub-mkconfig -o /boot/grub/grub.cfg`
      
### Setting Up TPM2
1. Verify TPM2 support for your device.
   - Refer to the `Trusted Platform Module` documentation on *ArchWiki*.
3. Enroll a `TPM2` key for the `LUKS` encrypted partition.
    - `systemd-cryptenroll --tpm2-device=auto <parameters> /dev/sda3`
    - Research for the optimal parameters. If I'm correct, `--tpm2-device=auto` + default paremeters is sufficient for functionality.
    - Refer to `Trusted Platform Module` documentation.
5. Set the `LUKS` encrypted partition to be automatically unlocked by `TPM2` in `/etc/crypttab.initramfs`
```
/etc/crypttab.initramfs

root    UUID={luks_encrypted_partition_UUID}    none    tpm2-device=auto
```
> [!NOTE]
> It is recommended read the *ArchWiki* documentation on `dm-crypt/System_configuration` for further details.
>
> On a separate note, `root` can be replaced with any name you would like the volume to open as here.
> The UUID of the encrypted `LUKS` partition can be found by running `lsblk -lf`.

4. Run `mkinitcpio -P` or `mkinitcpio -p linux` again.

## Final Touches
1. Create your user
    - `useradd -m -g users -G wheel {username}`
    - The `-m` flag adds a home directory for your user, and the `-g` and `-G` flags sets groups and seconday groups for your user respectively.
> [!NOTE]
> Add the `wheel` group to the sudoers list.
> You can do this now or later after you reboot.
2. Exit the chroot environment.
3. Unmount everything
    - `umount -R /mnt`
4. Reboot. 
