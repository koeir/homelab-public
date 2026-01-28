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
   - If it is either of the conditions above, just run `echo "KEYMAP=us >/etc/vconsole.conf`
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
> Consider *disabling* the `root` user later and solely rely on `sudo`.

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
