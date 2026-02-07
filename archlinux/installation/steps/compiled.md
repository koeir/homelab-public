Arch Linux _minimal installation_ guide with LUKS encrypted
disk (not including `boot` and `efi`) + TPM2 auto-unlock.

> **[DISCLAIMER]**
> This guide assumes that your machine is compatible with TPM2.
> This guide includes commands to follow and explains them, but it is recommended to actually research about each command if you don't already recognize/understand them.
> This installation does NOT include desktop environments and is designed for home server use.
>
> I suggest reading the _ArchWiki_ documentation while you follow this guide.

# ISO Flashing

1. Download latest official Arch Linux ISO from the official website.
   - It is recommended to confirm the hash and signature after downloading.
2. Flash a USB drive or other installation media with the ISO.
   - Recommended tool:
     `Rufus`

# Installation

1. Attach the installation media to the machine and enter the boot menu.
   - The boot menu can be opened by (repeatedly) pressing <Esc> while the machine starts up.
2. Boot the installation media.
   - Try to find the device name of the installation media.
   - If you can't find it initially, try to restart the machine a couple of times.

## Connecting to Wi-Fi

1. Connect to the network with `iwctl` or `mmcli`.
   - If you're having trouble with connecting to networks with special chars in SSIDs, check out `archlinux/installation/troubleshooting/iwctl_ssid_trouble.md`
2. It is recommended to upgrade `openssh` if planning to continue using `SSH`.
   - `pacman -Sy openssh --noconfirm`

## Setting up SSH (optional)

1. Make some security configurations to `SSH` before proceeding.

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
   - `root@archiso ~ # systemctl restart sshd && systemctl status sshd`
   - The logs should say something like `Server listening on 0.0.0.0 port {port}.`
3. Generate key-pairs on local machine (if haven't already) transfer the public key.

> [!TIP]
> `netcat` can be used to quickly transfer _public_ keys. Don't use netcat to transfer sensitive information.
> A safer way to transfer your pubkey would be with `ssh-copy-id`.

```
root@archiso ~ # netcat -lvnp {port} > ~/.ssh/authorized_keys

local@machine ~ # ssh-keygen -t ed25519 -C "archlinux"
local@machine ~ # cat ~/.ssh/archlinux.pub | netcat {arch_machine_ip} {port} -q 0
```

4. `ssh` into the remote machine.
   - `local@machine ~ # ssh {arch_machine_ip} -p {port} -i {path_to_privkey}`
   - No need to specify the `-i` flag if you used password authentication.

> [!NOTE]
> You could also use password authentication instead of pubkey authentication. This, of course, is less secure,
> but is more convenient (especially if you keep messing up and restarting the installation).
> The key pair generated on the local machine (that is to say **not** the machine you are currently installing Arch on) can be reused later after Arch is installed.

# Disk Partitioning

1. Make partitions for boot, efi, and root filesystem using any partitioning tool.
   - Set the type of your root partition to `Linux LVM`. (optional but useful)

> [!TIP]
> Look up how to use `lsblk`.

```
root@archiso ~ # fdisk /dev/sda

# This is just a representation of the interactive fdisk-cli,
# it does not look like this exactly.
[fdisk-interactive]
# efi
Command: g
Command: n
    Partition Number: [default (1)]
    First sector: [default]
    Last sector: +1G

# boot
Command: n
    Partition Number: [default (2)]
    First sector: [default]
    Last sector: +1G

# root partition
Command: n
    Partition Number: [default (3)]
    First sector: [default]
    Last sector: [default]

# Set root partition type to LVM
Command: t
    Partition Number: [default (3)]
    Type: 44
```

> [!WARNING]
> Using `default` for the last sector uses up all remaining space.
>
> `G` and `GB` are different.

2. Run `fdisk -l` to verify everything. The device names might vary, but the `Size`s (of the `boot` and `efi` partitions) and `Type`s and should follow.

```
Device       Start        End    Sectors   Size Type
/dev/sda1     2048    2099199    2097152     1G Linux filesystem
/dev/sda2  2099200    4196351    2097152     1G Linux filesystem
/dev/sda3  4196352 1953523711 1949327360 929.5G Linux LVM
```

> [!NOTE]
> The `Type` configuration isn't functionally necessary and are just metadata.
>
> `ESP` stands for EFI System Partition. `ESP` and `efi` might interchanged a couple time in this guide, but note that they are not necessarily the same thing. `ESP` refers to the partition where the `efi` or `uefi` resides.
>
> The _ArchWiki_ docs only use one partition for `boot` and `ESP`, but this guide separates them for security and compatibility.
> Having a separate `boot` partition also allows you to later encrypt it if you wish.

# Disk Formatting (Boot and ESP)

3. Format the `boot` and `efi system partitions`.

```
root@archiso ~ # mkfs.fat -F32 /dev/sda1
root@archiso ~ # mkfs.ext4 /dev/sda2
```

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

> [!NOTE]
> `lvm` (found in `cryptsetup open` line), `volgroup0` (found in `vgcreate` line), and volumes starting with `lv_` are _conventional names_. That is to say that they can be replaced with whatever you want. The `lvm` name is also just a temporary name used for the `cryptsetup open` instance.

> [!TIP]
> It is also possible to encrypt the `boot` partition, but that requires extra configuration. It is possible to encrypt the `boot` partition after installation.

# Mounting

1. Mount the respective partitions and swap
   - `swap` should be mounted with `swapon {swap_partition}`

```
root@archiso ~ # mount /dev/volgroup0/lv_root /mnt
root@archiso ~ # mount /dev/sda2 /mnt/boot

# If you have swap
root@archiso ~ # swapon /dev/volgroup0/lv_swap
```

# Installing Essential Packages

1. Install base packages and the `linux` kernel (and firmware).
   - `pacstrap -K /mnt base linux linux-firmware`
   - An error stating that `/etc/vconsole.conf` is not found might appear after installing the kernel.
   - This is fine, but has to be fixed later.

# Verifying information

1. Generate `fstab` and verify results.

```
genfstab -U /mnt >> /mnt/etc/fstab
root@archiso ~ # cat /mnt/etc/fstab
```

> [!NOTE]
> The `fstab` (pronounced _ef-es-tab_) file contains information about the mounted partitions under a specified mountpoint.
> The `-U` flag makes the `genfstab` command generates UUIDs for each device, which will be useful later.

# Arch-Chroot

> chroot can be entered by entering the command `arch-chroot /mnt`

## Installing Packages

1. Install _essential_ packages
   - Packages can be installed with `pacman -S package1 package2`
   - Essential list:
     - grub, efibootmgr, lvm2, sudo
   - Personally essential:
     - git, vim, networkmanager, man, fd, dosfstools, openssh, zsh, gpg, fzf

## Configuring Initramfs

2. Add necessary hooks to the `/etc/mkinitcpio.conf` file
   - Find the uncommented `HOOKS=` line.
   - Add `sd-encrypt lvm2`

> [!WARNING]
> `sd-encrypt lvm2` should **ALWAYS** go after `block` and before `filesystems`, **in that order**.
> `sd-encrypt` and `encrypt` are different hooks. For this setup, it is essential to only have `sd-encrypt` from the two.

3. Update the `initramfs`
   - `mkinitcpio -P` or `mkinitcpio -p linux`
   - If it is looking for file `/etc/vconsole.conf`, it is likely empty or does
     not exist.
   - If it is either of the conditions above, just run `echo "KEYMAP=us" > /etc/vconsole.conf`
   - or whatever `KEYMAP` you would like.

## Customizations

4. Set time and locale
   - `ln -sf /usr/share/zoneinfo/{Area}/{Location} /etc/localtime`
   - locale can be set in `/etc/locale.gen` by uncommenting a locale (e.g. `en_us.UTF-8 UTF-8`) and running `locale-gen`
   - Update the `LANG` variable in `locale.conf` accordingly
     - Use a text editor like `vim` or `nano`
   - (e.g. using `en_US.UTF-8`)

   ```
   /etc/locale.conf

   LANG=en_US.UTF-8
   ```

5. Edit `/etc/hostname` accordingly.
   - Just append any name. This will be your device name.

## Installing and Configuring GRUB

6. Configure GRUB
   - Add the `root` values to the `LINUX` line.

   ```
   /etc/default/grub

   GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 root=/dev/volgroup0/lv_root quiet"
   ```

> [!NOTE]
> Specifying `root=` is actually unnecessary. `grub-mkconfig` is smart enough to determine the correct root filesystem.
> If you have swap, consider setting `resume={swap_UUID}`.

7. Make a directory for the `ESP` and mount the first partition.
   - `systemctl daemon-reload` might need to be called first; this command must be ran in the installer environment, as it cannot be ran in `chroot`.

```
# If it prompts you to run `systemctl daemon-reload`
[root@archiso /] exit
root@archiso ~ # systemctl daemon-reload
root@archiso ~ # arch-chroot /mnt

[root@archiso /] mkdir /boot/EFI
[root@archiso /] mount /dev/sda1 /boot/EFI
```

8. Install `grub/efi`
9. Copy locale to grub
10. Make grub config file

```
[root@archiso /] grub-install --target=x86_64-efi --bootloader-id=grub_uefi --recheck
[root@archiso /] cp /usr/share/locale/en\@quot/LC_MESSAGES/grub.mo /boot/grub/en.mo
[root@archiso /] grub-mkconfig -o /boot/grub/grub.cfg
```

## Final Touches

11. Add a `passwd` to the root user.
    - Just run `passwd`

> [!WARNING]
> Make sure this password is secure! The `root` user has access to every single file in the machine.
> Consider _disabling_ the `root` user later and solely rely on `sudo`.

12. Create your user

- `useradd -m -g users -G wheel {username}`
- The `-m` flag adds a home directory for your user, and the `-g` and `-G`
  flags sets groups and seconday groups for your user respectively.

> [!NOTE]
> Add the `wheel` group to the sudoers list. This can be done later after reboot.

13. Exit the chroot environment.
14. Unmount everything

- `umount -R /mnt`

> [!TIP]
> Re-check if anything is mounted under `/mnt` with `mount`.

15. Reboot.

# Setting Up TPM2

> For this section, I recommend reading the `Trusted Platform Module` and `dm_mod/System_configuration` documentation on ArchWiki.
> This should be done _after_ booting into the actual installed Arch Linux operating system, not _during_ installation as `root@archiso`.
> Setting up the TPM2 keys in the installation media causes TPM to read the installation media's PCRs instead of the intended drive.

1. Verify TPM2 support for your device.
   - Refer to the `Trusted Platform Module` documentation on _ArchWiki_.
2. Enroll a `TPM2` key for the `LUKS` encrypted partition.
   - `systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=<pcr_parameters> <luks_crypt partition>`

> [!NOTE]
> The `--tpm2-pcrs` technically aren't necessary, but should be used for security.
> PCRs are different values measured during and after boot. TPM checks whether the measured PCRs at the time of enrolling the keys are the same as the PCR values of the current boot.
> If TPM measures a change in one of the locked PCRs, it won't automatically unlock the partition.
> Without the PCR checking, the encryption is pretty much useless.
>
> Any external/removable drives should be removed before enrolling PCRs. Anything that might not normally be plugged into the machine during/before boot, as these affect the PCR values.
>
> The more PCRs you have the slower the boot time may be. Since I'm using my machine as a server, I don't necessarily need a fast boot time, and so I prefer strong security.
>
> You should read `TPM2` documentation on what the `--tpm2-pcrs` parameter and specified values are for.
>
> I personally recommend 5, 7, 11, 15:
>
> `5` detects any addition of, removal of, and any modification to the partitions. Ensures that if a malicious actor is messing around with the partitions, the partition isn't automatically unlocked. This PCR measures **all** partitions detected, not just the root filesystem's, therefore it changes if any removable drives that were/weren't connected during enrollment unplugs/plugs. Among other things, this PCR-lock prevents unauthorized access from vulnerable bootable devices (recall mounting the rootfs during installation).
>
> `7` detects if the `Secure Boot` state changes. It tracks what the `Secure Boot` state was when the TPM was enrolled. It is recommended to set up `Secure Boot` at a later date. Note that the TPM key has to be wiped and re-enrolled if you change the `Secure Boot` state. If `Secure Boot` is enabled, this ensures that if a malicious actor disables `Secure Boot`, the partition isn't automatically unlocked.
>
> `11` detects if the kernel and boot phases. It ensures that the partition isn't automatically unlocked if a malicious actor tampers with the kernel or succeeding boot phases.
>
> `15` detects a lot of things, and is not strictly defined, so it may differ on different OSes. On Arch, it detects the machine ID, mount points, and partition and filesystem stuffs. I don't strictly recommend this one, but is good for extra security.

3. Set the `LUKS` encrypted partition to be automatically unlocked by `TPM2` in
   `/etc/crypttab.initramfs`
   > It is recommended read the _ArchWiki_ documentation on `dm-crypt/System_configuration` for further details.

```
/etc/crypttab.initramfs

root    UUID={luks_encrypted_partition_UUID}    none    tpm2-device=auto
```

> [!TIP]
> It is recommended read the _ArchWiki_ documentation on `dm-crypt/System_configuration` for further details.
>
> On a separate note, `root` can be replaced with any name you would like the volume to open as here.
> Make sure you add the encrypted **partition** (`/dev/sda3` or the 3rd partition made in `./disk_partitioning.md`), not the logical volumes.
> The UUID of the encrypted `LUKS` partition can be found by running `lsblk -lf`. It should have an `FSTYPE` of `crypto_LUKS` or something similar.

4. Run `mkinitcpio -P` or `mkinitcpio -p linux` again, and you are all set.
   - This is to make sure that `initramd` knows about `/etc/crypttab.initramfs`

> [!NOTE]
> You may also need to update `initramfs` (by running `mkinitcpio`) everytime you change your TPM key.
