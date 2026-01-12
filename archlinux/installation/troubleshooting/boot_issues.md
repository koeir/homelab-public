# [ TIME ]

## 2026-01-06 22:00
Upon reboot, it says
"[ TIME ] Timed out waiting for device /dev/mapper/volgroup0/lv_root"

And at the end, it says cannot open access to console, the root account is locked.
It is also stuck at a "Please Enter to continue" loop. Pressing power button does not do anything.

I thought it had something to do with the luks/tpm setup.
I went back into the ISO, mounted everything again and went back to arch-chroot.
I repeated all the arch-chroot steps in homelab/installation/arch_linux_installation.

However, upon installing grub/efi, it notified that I had to uncomment the ENCRYPT_DISK thing, which I did, and promptly added it to the steps.

It still erred the same, but this time, I had to unlock the partition before entering GRUB, which was a change.

After some consultation with ChatGPT about luks + tpm, I realized that making the tpm key *before* running mkinitcpio might have been a mistake.
Testing my theory tomorrow.

## 2026-01-07 18:10
Made new TPM key, still doesn't boot. Perhaps it could be a misconfiguration in `/boot/default/grub` 
Full error is:
```
[ TIME ] Timed out waiting for device /dev/mapper/volgroup0-lv_root
[DEPEND] Dependency failed for Initrd Root Device
[DEPEND] Dependency failed for /sysroot
[DEPEND] Dependency failed for  Initrd Root File System.
[DEPEND] Dependency failed for File System .k on /dev/mapper/volgroup0-lv_root
```

Last lines of `journalctl -xb`
```
Jan 07 18:18:08 archiso kernel: e820: remove [mem 0xe0000000-0xefffffff] reserved
Jan 07 18:18:08 archiso kernel: efi: Not removing mem37: MMIO range=[0xfe042000-0xfe042fff] (4KB) from e820 map
Jan 07 18:18:08 archiso kernel: efi: Not removing mem38: MMIO range=[0xfe043000-0xfe043fff] (4KB) from e820 map
Jan 07 18:18:08 archiso kernel: efi: Not removing mem39: MMIO range=[0xfe044000-0xfe044fff] (4KB) from e820 map
Jan 07 18:18:08 archiso kernel: efi: Not removing mem40: MMIO range=[0xfe900000-0xfe902fff] (12KB) from e820 map
Jan 07 18:18:08 archiso kernel: efi: Not removing mem41: MMIO range=[0xfec00000-0xfec00fff] (4KB) from e820 map
Jan 07 18:18:08 archiso kernel: efi: Not removing mem42: MMIO range=[0xfed01000-0xfed01fff] (4KB) from e820 map
```
idk wtf that means

Changed the `GRUB_CMD_LINE_LINUX_DEFAULT` line
```
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 cryptdevice=UUID={UUID}:cryptroot root=/dev/volgroup0/lv_root quiet"
```

## 2026-01-07 22:00
I ditched ChatGPT and actually read the documentation and finally got it to work.

Credits to @sudopluto on Youtube on his video about "Arch Linux Full Disk Encryption Using TPM2".
The video wasn't a direct step by step fix, but it did give me enough knowledge to troubleshoot.
The steps are in the current (as of commit 912f83e) installation steps.

> [!NOTE]
> My mistake was using `cryptdevice` in the `GRUB_CMD_LINE_LINUX_DEFAULT` line.
> #1, `cryptdevice` is for the `encrypt` hook, *not* `sd-encrypt`. The `encrypt` is the legacy version, I believe.
> #2, It won't even work with TPM, since that has to be set up in `/etc/crypttab.initramfs`.
