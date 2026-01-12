# Setting Up TPM2

> For this section, I recommend reading the `Trusted Platform Module` and `dm_mod/System_configuration` documentation on ArchWiki.

1. Verify TPM2 support for your device.
   - Refer to the `Trusted Platform Module` documentation on _ArchWiki_.
2. Enroll a `TPM2` key for the `LUKS` encrypted partition.
   - `systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=5+7+11+15 /dev/sda3`

> [!NOTE]
> You can read `TPM2` documentation on what the `--tpm2-pcrs` parameter and specified values are for.
>
> Here are short descriptions of 5, 7, 11, 15:
>
> `5` detects any modification to the partition table. Ensures that if a malicious actor is messing around with the partitions, the partition isn't automatically unlocked.
>
> `7` detects if the `Secure Boot` state changes. It tracks what the `Secure Boot` state was when the TPM was enrolled. It is recommended to set up `Secure Boot` at a later date. Note that the TPM key has to be wiped and re-enrolled if you change the `Secure Boot` state. If `Secure Boot` is enabled, this ensures that if a malicious actor disables `Secure Boot`, the partition isn't automatically unlocked.
>
> `11` detects if the kernel and boot phases. It ensures that the partition isn't automatically unlocked if a malicious actor tampers with the kernel or succeeding boot phases.
>
> `15` detects alot of things, but in general, it detects from where the system is booted. This one is highly recommended, maybe even necessary for security. For an in-depth explanation, read the docs, but the main reason why this `PCR` is highly recommended is because it ensures that a malicious attacker can't bypass authorization by booting the machine from a different storage device. I write more about this (somewhere in this repo).

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
