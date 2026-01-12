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
> `lvm` (found in `cryptsetup open` line), `volgroup0` (found in `vgcreate` line), and volumes starting with `lv_` are *conventional names*. That is to say that they can be replaced with whatever you want. The `lvm` name is also just a temporary name used for the `cryptsetup open` instance.

> [!TIP]
> It is also possible to encrypt the `boot` partition, but that requires extra configuration. It is possible to encrypt the `boot` partition after installation. 
