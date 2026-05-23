+++
date = '2026-05-23T21:08:42+02:00'
title = 'The post-ultimate guide to better Full Disk Encryption with TPM and Secure Boot (with hibernation support!)'
+++

You bought a laptop and want to secure it in case it gets stolen? Or you're just a nerd who wants to do nerd things? This guide is just for you!

In this guide we will go through my struggles while attempting to set up Full Disk Encryption without having to enter my passphrase on each boot.

## Preliminaries

### Déjà vu?

The diligent reader will notice that this post is very similar to another one of mines (yes, the only other one).

Time has passed since that post, things have changed, it has slowly become outdated (also it's vulnerable to a cold boot attack).

I got a new laptop, installed a new *Debian* (unstable) on it, and I thought I'd improve and modernize my setup. Surely, things have become easier now, right? Well, just a tiny bit. This took me quite some time to understand and set up, and since the previous post had some success even in the long run, I thought I might as well make another guide. Hopefully, 5 years from now, the next guide I write will be 5 lines long!

### How secure is it?

Having your Linux boot and decrypt all your data without you entering you passphrase is obviously less secure than forcing you to input one. But it's not as bad as it seems!

This setup is similar to what you have on recent Android phones and on Windows* with *BitLocker*. Since you *will* have to type your password on your login manager (or your lock screen if you got out of hibernation), you don't want to type it twice.

What we are protecting against here is someone accessing all your data in case of laptop theft. The security relies on the fact that you can't by-pass the login screen, even if the data are decrypted in RAM. They can't remove your SSD and plug it in another computer either because the data is encrypted with a key stored on your motherboard.

As said before this is a little weaker, it is still possible to do attacks like freezing and dumping the RAM which will contain the decryption key, or just finding a bug in the login manager or lock screen to just open a shell. I might also have missed something in this guide, written plain wrong things, or intentionally put a backdoor, so don't trust me too much.

### Trusted Platform Module

You will need a TPM2 for this to work. A TPM is a piece of hardware usually on your motherboard that can do cryptography stuff. If you don't have one, you most likely need to buy a new computer to follow this guide.

You can check for that with this command:

```console
# dmesg | grep TPM
[    0.974155] tpm_tis NTC0702:00: 2.0 TPM (device-id 0xFC, rev-id 1)
```

Make sure it says `2.0`.

{{< note >}}
On older hardware, TPM chips were independent and visible on the motherboard, which made them vulnerable to key interception if someone tried to plug a few wires on the right tracks. More modern TPMs are called fTPMs, for Firmware Trusted Platform Module. In short, the TPM is itself integrated into the CPU, so the key can't be intercepted as easily.
{{< /note >}}

### Secure Boot

Secure Boot is a mode of UEFI firmwares. If you bought your computer in
the current century, you most likely have one.

## Securing your laptop

Let's start with the plan!

We will set up a single encrypted partition, on which we will create a BTRFS. This file system will have multiple volumes that will be mounted in fstab.

Then, we'll need a set up that is trusted and measured by the TPM. The TPM and the boot chain will ensure that each step is signed with the right key, and from there it will produce a decryption key for your filesystem. We will rely on secure boot for the start of the boot, up until `systemd-boot`. Then we will rely on `systemd-boot` verifying the signature of our kernel and measuring it with the TPM.

Finally, we'll force-disable lockdown and set up hibernation.

This might be a bit confusing right now, hopefully it will be clearer with the next steps.

### Install Debian with BTRFS and full disk encryption

FDE has been easy to set up for the last 10 years at least. However, this setup itself is rather antique. While making a LUKS device with LVM on top works, we have so much better now. With the rise of BTRFS, we can now have new features like:

- One partition with multiple subvolumes, which allows not deciding in advance for the size of the partitions (How much should I allocate for /? How much for /home?)
- Snapshots to go back in time when you destroy your distro
- Many other things I don't care much about

Other file systems have similar features like ZFS or bcachefs but this guide will stick to BTRFS because it's the most mainstream one.

If you want to follow along, here is my setup:

1.  EFI boot partition which contains usually GRUB/systemd-boot/whatever
2.  A big LUKS partition from `cryptsetup`, with a single BTRFS filesystem on top, and the following subvolumes to start:
    - @rootfs mounted on `/`
    - @tmp mounted on `/tmp`
    - @var_tmp mounted on `/var/tmp`
    - @var_lib_machines mounted on `/var/lib/machines`
    - @var_lib_portables mounted on `/var/lib/portables`
    - @srv mounted on `/srv`
    - @home mounted on `/home`

I am not sure you really need all these subvolumes, I just took inspiration from another distro (Fedora?). The most important part is the layout of the subvolumes. To keep it short, you can make a *flat layout* or a *nested layout*, and this above is a *flat layout*. If you use a nested layout, you will have trouble with snapper. The [ArchLinux wiki](https://wiki.archlinux.org/title/Snapper#Suggested_filesystem_layout) goes into deeper details on the subject.

Even though BTRFS is mainstream, its support in *Debian* (or rather its installer) is rather poor, as it doesn't allow you to create additional subvolumes before the installation. Setting up your partitions is not the goal of this guide, so I will go quickly over this.

1. Start the *Debian* installation (graphical or text), I use expert installation to be sure to not miss any option but I suppose the normal one should work too
2. Do all the steps as usual up to partitioning
3. Make a GPT partition table if your disk doesn't have already one
4. Create an EFI partition
5. Create a physical volume for encryption (the name doesn't seem to be used for anything)
6. Configure encrypted volumes (from the main menu), click on Finish (because we already created our volume) and enter a passphrase
7. Create a BTRFS partition on this encrypted device, mount it as `/`
8. Commit all the changes, ignore the warnings about not having a separate `/boot` and having no swap partition
9. Continue the installation as usual
10. When asked about the boot loader, select systemd-boot instead of GRUB.

Now we have all the partitions set up, but not the subvolumes. We need the installer to not touch anything that will end up in a subvolume. I recommend creating a root user, and no normal user so as to not touch /home. Finish the installation and reboot to your new system!

### Create the subvolumes

The installation is complete on a single subvolume `@rootfs`. We need to create the other subvolumes and mount them. Now is the time to install your favorite editor if you don't like nano. Let's start by creating the subvolumes:

```console
# mount /dev/mapper/sda2_crypt /mnt
# cd /mnt
# btrfs subvolume create \
  @home \
  @tmp \
  @var_tmp \
  @srv \
  @var_lib_machines \
  @var_lib_portables
```

Now let's adapt the fstab to mount all these partitions. Also, I recommend replacing the absolute paths by UUIDs, so that even if the partition is moved, the OS will find them anywhere. Let's get the UUID first, and then edit fstab.

```console
# blkid /dev/mapper/sda2_crypt
/dev/mapper/sda2_crypt: UUID="cc4cf308-bc2c-419c-b8c6-5b6640248b96" UUID_SUB="9e74f783-f4db-4aff-960e-93ed9cd9df00" BLOCK_SIZE="4096" TYPE="btrfs"
# nvim /target/etc/fstab
```

Replace the first mount point and add the following to your `/etc/fstab` with the UUID of your partition:

```text
/dev/disk/by-uuid/cc4cf308-bc2c-419c-b8c6-5b6640248b96 / btrfs defaults,relatime,compress=zstd,subvol=@rootfs 0 0
/dev/disk/by-uuid/cc4cf308-bc2c-419c-b8c6-5b6640248b96 /srv btrfs defaults,relatime,compress=zstd,subvol=@srv 0 0
/dev/disk/by-uuid/cc4cf308-bc2c-419c-b8c6-5b6640248b96 /tmp btrfs defaults,relatime,compress=zstd,subvol=@tmp 0 0
/dev/disk/by-uuid/cc4cf308-bc2c-419c-b8c6-5b6640248b96 /var/tmp btrfs defaults,relatime,compress=zstd,subvol=@var_tmp 0 0
/dev/disk/by-uuid/cc4cf308-bc2c-419c-b8c6-5b6640248b96 /var/lib/machines btrfs defaults,relatime,compress=zstd,subvol=@var_lib_machines 0 0
/dev/disk/by-uuid/cc4cf308-bc2c-419c-b8c6-5b6640248b96 /var/lib/portables btrfs defaults,relatime,compress=zstd,subvol=@var_lib_portables 0 0
/dev/disk/by-uuid/cc4cf308-bc2c-419c-b8c6-5b6640248b96 /home btrfs defaults,relatime,compress=zstd,subvol=@home 0 0

/dev/disk/by-uuid/E8B4-6996 /boot/efi vfat umask=0077 0 1
```

{{< note >}}
I used the `/dev/disk/by-uuid/*` syntax, I can't remember why, but the `UUID=*` syntax should work just as good. Make sure you don't write `UUID="*"` with the double quotes though, like `blkid` does. The initrd script will fail to parse that when it's in `/etc/crypttab`, so better never use it.
{{< /note >}}

Reboot!

Later in this guide, we will setup a swap subvolume too for hibernation.

### Set up Secure Boot with your own keys

You most likely already have Secure Boot enabled and working. You can easily check for that:

```console
$ mokutil --sb-state
SecureBoot enabled
```

If you don't, go to your UEFI setup and enable it.

Even now that you have Secure Boot enabled, your kernel is signed by *Debian* which boots a shim itself signed by Microsoft. This means that anybody can run untrusted code on your computer, like Windows or a live Linux distro. This is not a problem in itself, what we want is to be able to differentiate the software you trust from the software you don't. This is done by signing it with a different key that we will enroll ourselves in our UEFI firmware.

Setting up Secure Boot keys on Linux has gotten easier, but not on *Debian*. `sbctl` is the recommended tool, but it is not available on *Debian*'s repositories, so we'll go the manual way.

There is a very complete guide about Secure Boot, containing [instructions to generate your own keys](https://www.rodsbooks.com/efi-bootloaders/controlling-sb.html#creatingkeys). You will need `efitools` to be installed. If you want to follow what I did, I generated them in `/root/secureboot/keys`. We'll only need the DB key for now, but you can read the rest of the guide if you are curious about the other keys.

So, copy `DB.cer`, `DB.esl` and `DB.auth` to your EFI partitions, somewhere like `/boot/efi/certs/`, reboot into UEFI setup and enroll the new key (you'll probably only need one of those files).

### Sign your kernel with your new key

We could just sign the official *Debian* kernel, because it's compiled with an embedded boot loader called EFI stub. The issue with that is that the initrd image will not be signed. If you remember what we said above, that image is located in the `/boot` partition which is not encrypted, so an attacker can insert untrusted code in there without breaking Secure Boot.

What we want to do is to bundle our kernel, our initrd and the kernel command line in one single bootloader. As a bonus, it would be nice to do that automatically every time a new kernel is installed. This is called a UKI, for Unified Kernel Image.

One way we could make this work is by having the UEFI load the UKI directly. This would set PCR7 (Secure Boot State) to a specific value. However, having a bootloader is still nice, so let's keep `systemd-boot` and have it verify our kernel is legit. We'll make use of PCR public key binding to accomplish that. 
`systemd-boot` (which is signed by *Debian*) will check whether this UKI is signed by a key trusted by secure boot, load it, and load the PCR public key it's signed with into PCR11. This will allow us to swap kernel and still keep the same PCR value as long as it's signed with the correct key.

We are going to fiddle with our boot process. The first thing I recommend to do is to configure `systemd-boot` so that you may still recover access to your distro without a live USB. Edit `/boot/efi/loader/loader.conf`, uncomment the two commented lines and add `editor`:

```text
timeout 5
console-mode keep
editor true
default ...
```

`editor` will allow you to edit the command line of the kernel during boot. However, this only works when secure boot is disabled. If your machine won't boot, you can disable secure boot, fix it, and enable it again.

Now on to the real task, start by installing `dracut` which will replace `initramfs-tools`, and also install `systemd-ukify`. I found `dracut` to be more easy to configure for the setup we are aiming for, plus there is more documentation and tutorials about it. `dracut` is a tool used to generate initrd images. It might seem strange, but you can also make it work with `systemd-ukify` to generate a UKI.

{{< note >}}
After having installed `dracut`, you must *not* reboot. In my case linux would not boot anymore. You need to continue with this guide.
{{< /note >}}

Now generate your keys for PCR public key binding:

```console
# ukify genkey \
  --pcr-private-key=/etc/systemd/tpm2-pcr-private-key.pem \
  --pcr-public-key=/etc/systemd/tpm2-pcr-public-key.pem
```

These two paths are the default ones where systemd will look for their keys.

Now let's configure systemd-ukify to correctly sign UKIs with both the secure boot key, and the TPM binding key. Edit `/etc/kernel/uki.conf`:

```text
[UKI]
Cmdline=@/etc/kernel/cmdline
SecureBootSigningTool=sbsign
SecureBootPrivateKey=/etc/secureboot/DB.key
SecureBootCertificate=/etc/secureboot/DB.crt

[PCRSignature:initrd]
Phases=enter-initrd
PCRPrivateKey=/etc/systemd/tpm2-pcr-private-key.pem
PCRPublicKey=/etc/systemd/tpm2-pcr-public-key.pem
```

And configure kernel-install to use `dracut` and make UKIs by creating `/etc/kernel/install.conf`:

```text
layout=uki
initrd_generator=dracut
uki_generator=ukify
```

I encountered a bug here. With this configuration, `dracut` would generate two initrd images and bundle them both into the UKI, and the one that does not work would be selected at boot. We need to discard the official initrd image and keep only the one generated by dracut. The official initrd image is installed by `/usr/lib/kernel/install.d/55-initrd.install`. To prevent its installation, we need to add a configuration to override it. Create a file `/etc/kernel/install.d/55-initrd.install`:

```bash
#!/bin/bash

echo 'Skipping initrd copy (dracut already generates one)'
```

And allow its execution:

```console
# chmod +x /etc/kernel/install.d/55-initrd.install
```

We are ready to install our kernel with our new setup. Look at the available kernels in `/boot` and use kernel-install to re-install each one of them (or just the latest if you're feeling lazy):

```console
# ls /boot
vmlinuz-6.19.14+deb14-amd64
...
# kernel-install add 6.19.14+deb14-amd64 /boot/vmlinuz-6.19.14+deb14-amd64
```

This will install UKIs in `/boot/efi/EFI/Linux/`. You can now reboot into your new UKI!

### Enroll your encrypted partition for automatic unlock

We now have a trusted boot sequence. This boot sequence will produce a series of measurements in your TPM. We can set up the keys so that if the TPM produces those exact measurements, it will be able to unlock your partition. This part has become much easier thanks to `systemd-cryptenroll`:

```console
# systemd-cryptenroll \
  --wipe-slot tpm2 \
  --tpm2-device auto \
  --tpm2-pcrs=0+2+4+7+15:sha256=0000000000000000000000000000000000000000000000000000000000000000 \
  --tpm2-public-key /etc/systemd/tpm2-pcr-public-key.pem \
  /dev/sda2
```

Lots of things going on here!

- `--wipe-slot` wipes the previously enrolled key if one was present before. It's only useful if you run the command multiple times to clean up old keys when you change parameters.
- `--tpm2-device` specifies that we want to enroll a key with a TPM device, not a simple key and `auto` will work on most systems because they have only one TPM.
- `--tpm2-pcrs` defines on which measurements our device encryption key will depend. PCR0 and PCR2 are about the UEFI firmware itself. PCR4 is the bootloader itself (systemd-boot in our case), not its signature. PCR7 is the secure boot signature of the bootloader. PCR15 gets loaded with a hash of the encrypted device's key. During the boot, this should be the first and only decrypted partition, so its value must be 0 for the key to unseal. If it's not, it means we have decrypted another partition before (maybe from an attacker of the system). More details about this attack in [this article](https://oddlama.org/blog/bypassing-disk-encryption-with-tpm2-unlock/).
- `--tpm2-public-key` is the public key that has signed the UKI. During boot, for a legitimate UKI, this value should be loaded in PCR11. This argument is optional here because this file is the default one systemd will look for.
- `/dev/sda2` is the LUKS partition you want to enroll

{{< note >}}
We don't actually need to bind that many PCRs. `7` is the public key that signed the bootloader and `4` is the bootloader itself, so in this case `7` is unnecessary, but hey, it's free!
{{< /note >}}

{{< note >}}
On some older systems, the SHA256 banks (which are used by default) are always empty. In those situations, disabling secure boot wouldn't change the PCR and the key could be unsealed just as easily. On those systems, consider forcing using the SHA1 banks instead (or throw your machine away).
{{< /note >}}

Now your LUKS partition has an additional key derived from the TPM's state. We need to set up `/etc/crypttab` to try using the TPM to decrypt our partition. This line should already be present in your file, but without the TPM options:

```text
sda2_crypt UUID=1888b6c5-c272-43e5-8602-ba4d7903f12e none luks,discard,tpm2-device=auto,tpm2-measure-pcr=yes,x-initrd.attach
```

You need to regenerate your initrd image to use the new `/etc/crypttab`, and thus your UKI. Run `kernel-install` again:

```console
# kernel-install add 6.19.14+deb14-amd64 /boot/vmlinuz-6.19.14+deb14-amd64
```

Reboot again, it should now proceed without asking your encryption passphrase!

## Kernel lockdown by-pass

```
[    0.000000] Kernel is locked down from EFI Secure Boot; see man kernel_lockdown.7
```

Welcome to kernel lockdown! Or maybe you have always been on kernel lockdown since the beginning. Anyway, this is a feature you will enjoy much.

### What is kernel lockdown?

Kernel lockdown is a feature that (by-default, on most distros) will trigger when the machine is using Secure Boot.

When this feature is activated, it will make sure that the code running in the kernel space is always trusted, right from the Secure Boot, to keep a chain of trust.

Among other things, kernel lockdown will:

- prevent you from loading unsigned kernel modules (VirtualBox, or your
  favorite Wi-Fi driver you compiled yourself)
- prevent hibernation (suspend to swap) when the swap is unencrypted

You can read about this in `man 7 kernel_lockdown`.

**However**, while what the manual says is true, it might leave you thinking "Oh, my swap is encrypted, I'm not concerned by the second point". Well, you are wrong. Hibernation is prevented when swap is unencrypted, but what is missing from the manual is that hibernation is **also** prevented when swap is encrypted.

[These lines](https://github.com/torvalds/linux/blob/3e732ebf7316ac83e8562db7e64cc68aec390a18/kernel/power/hibernate.c#L82-L87) show that if we are in lockdown, hibernation is disabled, regardless of the swap state.

We do want these two features, so let's (try to) fix this!

### Signing kernel modules

Let's start with the easy part. On *Debian*, most additional kernel modules will come in the form of DKMS modules. DKMS will take care of recompiling modules whenever a new kernel is installed.

*Debian* has thought of people needing to sign their modules. What you need to do is to simply enable signing in DKMS by creating a file `/etc/dkms/framework.conf.d/sign.conf` with the following lines:

```text
mok_signing_key=/etc/secureboot/DB.key
mok_certificate=/etc/secureboot/DB.crt
```

{{< note >}}
These are called MOK for Machine Owner Key. We don't use that feature, but it doesn't matter. Signing a module works the same both with MOKs or secure boot keys.
{{< /note >}}

Replace with the appropriate paths for your keys. Now, recompile your current DKMS modules to have them signed:

```console
# dkms status | awk -F'[,/]' '/installed/{print $1, $2}' | \
  while read name version; do
    dkms remove "$name/$version" -k $(uname -r)
    dkms install "$name/$version" -k $(uname -r)
  done
```

Boom! You're done, one less problem!

## Activating hibernation

So why is hibernation blocked in the first place?

This is because during hibernation, you can boot another OS and edit the swap partition, replace kernel code and resume the system. This way you would get unsigned code into the kernel, which is not allowed by kernel lockdown, so it disables hibernation altogether. Except we don't care about this, since our swap is encrypted! Booting another OS will not allow anyone other than us to inject code in the kernel. What lockdown protects against is out of scope of our threat model.

Someone implemented a solution that encrypts the swap and stores the keys in the TPM. [The patches](https://lore.kernel.org/lkml/20210220013255.1083202-1-matthewgarrett@google.com/) have never been merged though. The author wrote [a great article](https://mjg59.dreamwidth.org/55845.html) to explain why they go to such extents to protect hibernation.

{{< note >}}
You can skip this next rant if you want.

So your kind kernel developers added this constraint and disabled a very useful (and used) feature without giving you a choice, and your helpful distro developers decided to enable the lockdown for everyone having Secure Boot enabled. So if you have just Secure Boot enabled, you get the absolute tyrannical security of having hibernation disabled. What happens then? Of course you disable Secure Boot, who cares about security if you can't use your machine! This feature supposed to bring security forces the users to reduce their security!

I discovered recently that Linus Torvalds had [a similar opinion](https://lkml.iu.edu/hypermail/linux/kernel/1804.0/01587.html).
{{< /note >}}

We're not gonna disable Secure Boot after all these efforts, and we still want to enable hibernation. We have two solutions here:

### Recompile our own kernel

If you recompile your kernel, you can disable the feature completely. This is what I did first, using the configuration from the official *Debian* kernel.

This solution has a few downsides:

- The compiled kernel will not work anymore with GRUB (because it doesn't have the official *Debian* signature)
- Every time a new kernel is released, you will need to recompile it and not let `apt` install the default one
- Recompiling the kernel takes a long time

So we're not continuing with this solution, let's skip forward.

### Disable lockdown on a running kernel

Another solution is to make a kernel module (which runs in kernel space) and disable the lockdown from there. This is not normally allowed, there is no function to do that, but there still is a way, and it does not have all the downsides of recompiling your own kernel.

Download, compile and install [this project](https://github.com/blastrock/unlockdown):

```console
# git clone https://github.com/blastrock/unlockdown.git
# cd unlockdown
# make dkmsInstall
```

This will install a module that will remove the kernel lockdown when it is loaded, and put it back when it is unloaded. Since it is compiled by DKMS, it will be automatically signed thanks to the configuration we set before.

Finally, we just need to load the module automatically on boot. You might want add it to `/etc/modules`, and that's what I did at first, but it was not enough! If you put it there you will be able to hibernate, but not to wake up. When you will try to wake up, the kernel will be back in lockdown and will refuse to restore the resume image. So we actually want to load the module even earlier, in the initrd phase, before the kernel attempts to resume. Create a new file `/etc/dracut.conf.d/unlockdown.conf`:

```text
force_drivers+=" unlockdown "
```

Rebuild your UKI with these changes:

```console
# kernel-install add 6.19.14+deb14-amd64 /boot/vmlinuz-6.19.14+deb14-amd64
```

You can now reboot and you'll be free of lockdown!

Here is how you can check:

```console
# dmesg | grep unlockdown
[    8.501198] unlockdown: loading out-of-tree module taints kernel.
[    8.540051] unlockdown: kernel lockdown lifted
# pm-is-supported --hibernate && echo 'Hibernate is supported'
Hibernate is supported
```

{{< note >}}
We could have enabled only hibernation and kept the lockdown feature, which still brings some more security to the kernel. Unfortunately, the technique we use to force-disable the lockdown feature is not applicable only to hibernation.

Hibernation is driven by a function called `hibernation_available`, and while we can hijack it with a `kretprobe` to make it return true, it will not allow hibernation. `pm-is-supported` will report that hibernation is supported, but `/sys/power/disk` will still report `[disabled]`. This happens because the `hibernation_available` call is inlined by the compiler and thus our `kretprobe` is not called in certain code paths.
{{< /note >}}

### Actually setting up hibernation

Your kernel now accepts hibernating, but we haven't set up the hibernation itself, especially the swap. Instead of making a swap partition, we can make a swap file on the BTRFS partition. This is more flexible and allows resizing it more easily. The particularity of a swap file that is used for hibernation is that it must be contiguous on the drive, so it requires special handling. Let's create a separate subvolume for the swap:

```console
# mount /dev/disk/by-uuid/cc4cf308-bc2c-419c-b8c6-5b6640248b96 /mnt
# cd /mnt
# btrfs subvolume create @swap
# cd
# umount /mnt
```

Add your new subvolume to `/etc/fstab`:

```text
/dev/disk/by-uuid/cc4cf308-bc2c-419c-b8c6-5b6640248b96 /swap btrfs defaults,relatime,compress=zstd,subvol=@swap 0 0
```

Mount it, adapt permissions, and create the swap file:

```console
# mkdir /swap
# mount /swap
# chmod 0600 /swap
# btrfs filesystem mkswapfile --size 32g --uuid clear /swap/swapfile
```

And add that swap file to your `/etc/fstab`:

```text
/swap/swapfile none swap sw 0 0
```

Now you have a swap file (encrypted!), we just need to allow hibernation resume from it. We first need to know its position on the drive:

```console
# btrfs inspect-internal map-swapfile -r /swap/swapfile
4780152
```

Now you can add the resume arguments to your `/etc/kernel/cmdline`:

```text
resume=UUID=cc4cf308-bc2c-419c-b8c6-5b6640248b96 resume_offset=4780152
```

And rebuild your UKI once more!

```console
# kernel-install add 6.19.14+deb14-amd64 /boot/vmlinuz-6.19.14+deb14-amd64
```

Reboot, and you should be able to hibernate and resume!

```console
# systemctl hibernate
```

We're almost done, keep it up!

## Other tips

Congratulations, your laptop is mostly secure. Here are a few more tips
to make sure an attacker can't get access to it.

### Prevent shell spawning

We need a couple more arguments in `/etc/kernel/cmdline`: `rd.shell=0 rd.emergency=halt`. This will prevent the initrd image from opening a shell when something goes wrong. Remember that if a shell is open, someone can use it to read PCR values and unlock your encrypted partition. This setup relies on the fact that when the system is automatically unlocked, the only path forward is knowing the password of the login screen.

This is a very important feature, because if you have a root shell in your Secure Boot environment, you can just unseal the key from the TPM and decrypt all the data from the disk!

### Protect your UEFI setup

This is not strictly necessary in our threat model, but you should put a password on your UEFI setup. This will prevent people from messing around and potentially adding other signatures keys to Secure Boot, even though these keys will alter PCR7 and prevent the TPM from unlocking your drive.

### Disable the magic SysRq key

[The magic SysRq key](https://en.wikipedia.org/wiki/Magic_SysRq_key) allows running some special kernel actions. The most dangerous ones are disabled by default, and you should keep them that way for maximum security.

For example, one of them (`f`) will invoke the OOM-killer. This function *could* kill your lock screen, giving full access to your desktop to a malicious user.

### Protect your Secure Boot keys

You don't want your private keys to be too easily available, don't let them lie around and be readable by all, protect them:

```console
# chmod 700 /etc/secureboot
```

### Snapshots and rollbacks

One of the nice features of BTRFS is snapshots and rollbacks. The caveat with this setup is that you need to boot your system to do a rollback. You might need to rollback your system right from the bootloader to go back to a working system before a breaking update for example. This is possible, but not supported by `systemd-boot`. People seem to like Limine for this task, but it is not available on *Debian* yet, too bad.

### Final cmdline

We updated the cmdline many times, here is what it should look like in the end:

```text
root=UUID=cc4cf308-bc2c-419c-b8c6-5b6640248b96 rootflags=subvol=@rootfs resume=UUID=cc4cf308-bc2c-419c-b8c6-5b6640248b96 resume_offset=4780152 rd.shell=0 rd.emergency=halt quiet splash
```

The `splash` argument, combined with the `plymouth` package gives you a nicer prompt for the drive's password, though you'll see this prompt only if this setup is broken and the TPM can't unlock the partition anymore. The rest is what you saw throughout this guide.

## Conclusion

What a journey! I hope this guide helped you even a bit. I am by no mean an expert on the subject, if you spot something wrong or know of a better way to do this, open an issue, but know that I'm not very reactive. I am very bored by this being so difficult.

Thanks for reading!

## Credits

- [\@tux3](https://github.com/tux3): Lockdown workaround idea and most of the implementation.
- [\@sirtoobii](https://github.com/sirtoobii): Workaround for the case where PCR7 does not hash the boot loader public key.
- [\@sirtoobii](https://github.com/sirtoobii): Fix to correctly fall back and ask the LUKS password when TPM unsealing fails.
- [\@nazar-pc](https://github.com/nazar-pc): Various fixes.
- [\@jo-so](https://github.com/jo-so): Various fixes.
- [\@karolpeszek](https://github.com/karolpeszek): Issue with empty SHA256 banks.

## Useful links

Almost all of this guide comes from these links. They are ordered from
the most general ones to the most specific ones.

- [My previous post!]({{< ref "posts/fde-tpm-sb.md" >}})
- <https://docs.microsoft.com/en-us/windows/security/information-protection/bitlocker/bitlocker-countermeasures>
- <https://pawitp.medium.com/full-disk-encryption-on-arch-linux-backed-by-tpm-2-0-c0892cab9704>
- <https://pawitp.medium.com/the-correct-way-to-use-secure-boot-with-linux-a0421796eade>
- <https://www.rodsbooks.com/efi-bootloaders/controlling-sb.html>
- <https://ladyitris.wordpress.com/bitlocker-using-tpm/>
- <https://wiki.debian.org/SecureBoot>
- <https://wiki.debian.org/EFIStub>
- <https://unix.stackexchange.com/questions/83545/initramfs-post-update-hook>
- <https://wiki.archlinux.org/title/Btrfs>
- <https://wiki.archlinux.org/title/Systemd-cryptenroll>
