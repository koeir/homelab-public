error after running pacman -S linux linux-firmware --noconfirm
==> Creating zstd-compressed initcpio image: '/boot/initramfs-linux.img'
==> WARNING: errors were encountered during the build. The image may not be complete.
error: command failed to execute correctly

error after running `mkinitcpio -P`
==> ERROR: file not found: '/etc/vconsole.conf'

solution:
echo "KEYMAP=us" > /etc/vconsole.conf
