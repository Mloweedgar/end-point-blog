---
title: "How to Mount a BitLocker Drive on Linux at Login"
author: Seth Jensen
github_issue_number: 2169
featured:
  image_url: /blog/2026/02/how-to-mount-a-bitlocker-drive-on-linux-at-login/leaf-in-water.webp
description: Using fstab to mount a BitLocker drive when you log in to linux
date: 2026-02-12
tags:
- linux
- sysadmin
- tips
---

![A single tree shoot with a few leaves pokes out from a lightly rippling water which reflects late afternoon sun](/blog/2026/02/how-to-mount-a-bitlocker-drive-on-linux-at-login/leaf-in-water.webp)

<!-- Photo by Seth Jensen, 2025. -->

I recently reinstalled Linux on my desktop machine alongside Windows on a separate SSD. I also have a 3TB hard drive for backups and slower storage, which I formatted on Windows. I didn't manually enable any encryption, but I got the following error when trying to mount it directly:

```
mount: /mnt/<mydrive>: unknown filesystem type 'BitLocker'
```

Looks encrypted to me! After some googling, I found a common solution is to use dislocker to decrypt the drive.

> I was confused because my drive wasn't encrypted, but the filesystem type was still "BitLocker". My guess is that Windows encrypted the drive by default, though as you'll see soon, it uses a "clear key," meaning it's technically encrypted but the key is stored unencrypted, meaning anyone can decrypt the drive. I suppose they want me to set up encryption somehow, but since this is a home computer, I don't mind it being unencrypted for now.
>
> If your drive is encrypted, there are a few extra steps. There's an excellent guide on std.rocks[^1] for this. I'll also annotate the fstab-specific section so you know where to put the recovery key there.

### Installing dislocker & FUSE

> From this point on, all of the commands need to be run as superuser. Add `sudo` before the command or run them from the `root` user. And, of course, proceed with caution and read your man pages before blindly trusting me :D

You can find dislocker packages in the default repos for most distros, or [build it from source](https://github.com/Aorimn/dislocker/blob/master/INSTALL.md). I'm running Ubuntu right now, so I just had to run:

```
apt-get install dislocker ntfs-3g
```

> I was able to mount the filesystem as read-only with the default `ntfs` driver, but to write I had to install `ntfs-3g`.

### Disabling Fast Startup in Windows

If Fast Startup is enabled in Windows, the drive can get corrupted or be forced into read-only mode in Linux. To prevent this, disable Fast Startup by entering the following in an administrator PowerShell:

```
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Power\" -Name "HiberbootEnabled" -Value "0"
```

You can also disable it from the control panel — just search "Fast Startup".

### Mounting on login with `fstab`

It's well and good to mount a drive one time, but since this is a persistent volume, I want it to be mounted consistently on boot. This is where `fstab` comes in.

To find a device, run `lsblk`, find the partition you're looking for, and note the device name. You should see something like:

```
sda      8:0    0 232.9G  0 disk 
├─sda1   8:1    0     1G  0 part /boot/efi
└─sda2   8:2    0 231.8G  0 part /
sdb      8:16   0   2.7T  0 disk 
├─sdb1   8:17   0    16M  0 part 
└─sdb2   8:18   0   2.7T  0 part 
```

In my case, I want to mount `/dev/sdb2`. First, though, we'll need the device's PARTUUID. Device names like `/dev/sdb2` can change when you add or remove disks, so referencing a UUID is recommended in fstab's man page to make sure you're mounting the same disk every time. To get the PARTUUID, run the following:

```
blkid | sort
```

Here's my output:

```
/dev/sda1: UUID="A1A1-A1A1" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="1a1a1a1a-1a1a-1a1a-1a1a-1a1a1a1a1a1a"
/dev/sda2: UUID="1a1a1a1a-1a1a-1a1a-1a1a-1a1a1a1a1a1a" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="1a1a1a1a-1a1a-1a1a-1a1a-1a1a1a1a1a1a"
/dev/sdb1: PARTLABEL="Microsoft reserved partition" PARTUUID="1a1a1a1a-1a1a-1a1a-1a1a-1a1a1a1a1a1a"
/dev/sdb2: TYPE="BitLocker" PARTLABEL="Basic data partition" PARTUUID="<mydrive-part-uuid>"
```

> You should also be able to see the UUID after running `lsblk -f`, but for some reason, that command didn't show me any disk information for `/dev/sdb`. It also didn't show me the disk's UUID, just the PARTUUID for each partition. That worked well enough for me, but please leave a comment if you know why that's happening.

Open `/etc/fstab` in your favorite editor and add the following at the end:

```
/dev/disk/by-partuuid/<mydrive-part-uuid> /mnt/bitlocker  fuse.dislocker  nofail,clear-key                              0   0
/mnt/bitlocker/dislocker-file             /mnt/<mydrive>  ntfs            nofail,x-gvfs-show,x-gvfs-name=bitlocker-raw  0   0
```

I specified the `clear-key` option which is assumed in the `dislocker` command — the `fuse.dislocker` filesystem type takes dislocker's long options. If your drive is encrypted, you'll want to replace this with `recovery-password=000000-000000-000000-000000-000000-000000-000000-000000` (recovery password is required if TPM is enabled) or `user-password=123456` (a PIN code here works if TPM is disabled, according to std.rocks[^1]).

Now, I recommend trying to mount the drive before rebooting to avoid any startup issues if there are syntax errors or similar. After saving the file, run the following to get your `fstab` changes into systemd:

```
systemctl daemon-reload
```

Now you can mount the drive, then mount the `dislocker-file` just by naming the mount points:

```
mount /mnt/bitlocker
mount /mnt/mydrive
```

If you see files in `/mnt/mydrive`, hooray, it worked! Your drive should be mounted on reboot now. If not, there may be an error in your fstab file. I'd recommend trying the manual mounting method in the next section and noting at what step the failure happens.

### Troubleshooting: manual mounting

I found it easier to diagnose and fix issues when I mounted manually first:

```
mkdir /mnt/bitlocker
dislocker -V /dev/sdb2 -- /mnt/bitlocker
```

If your drive *is* encrypted, dislocker should prompt you for a user or recovery password. Since mine isn't, I instead used the default "clear key" stored on the volume. The guide on std.rocks[^1] has more info for this.

If you check in the `/mnt/bitlocker` folder, you can see that there's now a file called `dislocker-file` there. This file is what we mount:

```
mount -o loop -t ntfs /mnt/bitlocker/dislocker-file /mnt/<mydrive>
```

* `-o loop` says that `dislocker-file` is a normal file you're mounting via the loop interface
* `-t ntfs` specifies that this drive is an NTFS filesystem, the default on modern Windows (post-1993, that is 😉)

After running this, you should see your files in `/mnt/<mydrive>`.

### References

[^1]: https://std.rocks/gnulinux_bitlocker.html
* https://github.com/Aorimn/dislocker
* `man fstab`
