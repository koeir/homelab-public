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
   - `pacstrap -K base /mnt linux linux-firmware`
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
