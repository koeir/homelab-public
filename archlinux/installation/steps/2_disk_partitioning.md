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
