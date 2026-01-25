# Setting Up TPM2

> For this section, I recommend reading the `Trusted Platform Module` and `dm_mod/System_configuration` documentation on ArchWiki.
> This should be done *after* booting into the actual installed Arch Linux operating system, not *during* installation as `root@archiso`.
> Setting up the TPM2 keys in the installation media causes TPM to read the installation media's PCRs instead of the intended machine.

1. Verify TPM2 support for your device.
   - Refer to the `Trusted Platform Module` documentation on _ArchWiki_.
2. Enroll a `TPM2` key for the `LUKS` encrypted partition.
   - `systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=<pcr_parameters> <luks_crypt partition>`

> [!NOTE]
> The `--tpm2-pcrs` technically aren't necessary, but should be used for security. 
> Without the PCR checking, the login authorization can be bypassed by mounting the now-decrypted filesystem on a different and otherwise vulnerable boot device.
>
> You can read `TPM2` documentation on what the `--tpm2-pcrs` parameter and specified values are for.
>
> I personally recommend 5, 7, 11, 15:
>
> `5` detects any modification to the partition table. Ensures that if a malicious actor is messing around with the partitions, the partition isn't automatically unlocked.
>
> `7` detects if the `Secure Boot` state changes. It tracks what the `Secure Boot` state was when the TPM was enrolled. It is recommended to set up `Secure Boot` at a later date. Note that the TPM key has to be wiped and re-enrolled if you change the `Secure Boot` state. If `Secure Boot` is enabled, this ensures that if a malicious actor disables `Secure Boot`, the partition isn't automatically unlocked.
>
> `11` detects if the kernel and boot phases. It ensures that the partition isn't automatically unlocked if a malicious actor tampers with the kernel or succeeding boot phases.
>
> `15` detects a lot of things. In general, it detects the machine ID, mount points, and partition and filesystem stuffs. Basically if it's the same booted machine.

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
