Arch Linux _minimal installation_ guide with LUKS encrypted
disk (not including `boot` and `efi`) + TPM2 auto-unlock.

> **[DISCLAIMER]**
> This guide assumes that your machine is compatible with TPM2.
> This guide includes commands to follow and explains them, but it is recommended to actually research about each command if you don't already recognize/understand them.
> This installation does NOT include desktop environments and is designed for home server use.

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
