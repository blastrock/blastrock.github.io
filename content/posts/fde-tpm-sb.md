+++
date = '2022-04-06T12:00:00+00:00'
title = 'The ultimate guide to Full Disk Encryption with TPM and Secure Boot (with hibernation support!)'
aliases = ["/fde-tpm-sb.html"]
+++

*Difficulty*: way harder than it should be!

**IMPORTANT**: This guide has a security flaw as nicely explained in
[this
article](https://oddlama.org/blog/bypassing-disk-encryption-with-tpm2-unlock/).
I have not taken the time to update it yet, so keep that in mind.

You bought a laptop and want to secure it in case it gets stolen? Or
you\'re just a nerd who wants to do nerd things? This guide is just for
you!

In this guide we will go through my struggles while attempting to set up
Full Disk Encryption without having to enter my passphrase on each boot.

{{< note >}}
[\@chrysn](https://github.com/chrysn) suggested a different method from
the one described in this guide which involves keeping GRUB and not
signing the kernel with your own key. The pro is that it\'s way simpler
than what\'s described here. The cons are that you\'d need to input your
password after each update\... and that there is no guide written about
this method yet. You can look at [this
issue](https://github.com/blastrock/blastrock.github.io/issues/9) for
more information.
{{< /note >}}

## Preliminaries

### How secure is it?

Having your Linux boot and decrypt all your data without you entering
you passphrase is obviously less secure than forcing you to input one.
But it\'s not as bad as it seems!

This setup is similar to what you have on recent Android phones and on
*Windows* with *BitLocker*. Since you *will* have to type your password
on your login manager (or your lock screen if you got out of
hibernation), you don\'t want to type it twice.

What we are protecting against here is someone accessing all your data
in case of laptop theft. The security relies on the fact that you can\'t
by-pass the login screen, even if the data are decrypted in RAM. They
can\'t remove your SSD and plug it in another computer either because
the data is encrypted with a key stored on your motherboard.

As said before this is a little weaker, it is still possible to do
attacks like freezing and dumping the RAM which will contain the
decryption key, or just finding a bug in the login manager or lock
screen to just open a shell. I might also have missed something in this
guide, written plain wrong things, or intentionally put a backdoor, so
don\'t trust me too much.

### Full disk encryption

FDE is easy to setup nowadays, on the *Debian* installer for example,
you just have to select \"Guided Partitioning (encrypted disk + LVM)\"
or something like that and it does everything for you. If you don\'t
have it set up yet, you can find a ton of guides for that over the
Internet. Basically, it sets up these partitions:

1.  EFI boot partition which contains usually GRUB
2.  /boot which contains your kernels and `initrd`s
3.  A big LUKS partition from `cryptsetup`
    1.  A swap partition
    2.  A / partition
    3.  A /home partition

### Trusted Platform Module

You will need a TPM2 for this to work. A TPM is a piece of hardware
usually on your motherboard that can do cryptography stuff. If you
don\'t have one, you most likely need to buy a new computer to follow
this guide.

You can check for that with this command:

```console
# dmesg | grep TPM
[    0.974155] tpm_tis NTC0702:00: 2.0 TPM (device-id 0xFC, rev-id 1)
```

Make sure it says `2.0`

### Secure Boot

Secure Boot is a mode of UEFI firmwares. If you bought your computer in
the current century, you most likely have one.

## Securing your laptop

Now that you have everything needed, here is my plan.

What we want to do is to store the key to decrypt the partition in the
TPM. And we want that TPM to only give back that key if a trusted
software is running, guaranteed by Secure Boot.

Since your kernels and GRUB are stored on an unencrypted partition, you
can\'t trust them. So we\'ll need to sign our kernel and initrd and have
the TPM give back the key only if a kernel with the right signature is
booted. This ensures that we have a chain of trust from the EFI firmware
(which we will check too).

We are going to do this on *Debian unstable*, but it should work with
other *Debian*-based distros like *Ubuntu*. If you are on *ArchLinux*,
it looks like there is almost nothing to do as everything is handled by
[systemd-cryptenroll](https://wiki.archlinux.org/title/Systemd-cryptenroll#Trusted_Platform_Module).
While `systemd-cryptenroll` probably works on *Debian*, it does not work
with an encrypted root partition.

### Set up Secure Boot with your own keys

You most likely already have Secure Boot enabled and working. You can
easily check for that:

```console
$ mokutil --sb-state
SecureBoot enabled
```

If you don\'t, go to your UEFI setup and enable it.

Even now that you have Secure Boot enabled, your kernel is signed by
*Debian* which boots a *shim* itself signed by Microsoft. This means
that anybody can run untrusted code on your computer, like *Windows* or
a live Linux distro. This is not a problem, what we want is to be able
to differentiate the software you trust from the software you don\'t.
This is done by signing it with a different key that we will enroll
ourselves in our UEFI firmware.

There is a very complete guide about Secure Boot, containing
[instructions to generate your own
keys](https://www.rodsbooks.com/efi-bootloaders/controlling-sb.html#creatingkeys).
If you want to follow what I did, I generated them in
`/root/secureboot/keys`. We\'ll only need the DB key for now, but you
can read the rest of the guide if you are curious about the other keys.
The ArchWiki also has [a complete
guide](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot#Using_your_own_keys)
to enroll your keys.

So, copy `DB.cer`, `DB.esl` and `DB.auth` to your EFI partitions,
somewhere like `/boot/efi/certs/`, reboot into UEFI setup and enroll the
new key (you\'ll probably only need one of those files).

### Sign your kernel with your new key

We will skip GRUB in the boot process. GRUB is signed by Debian and the
*shim* that boots it is signed by Microsoft. We don\'t want to re-sign
grub and then add our signature keys to it. This is probably feasible,
but the chain of trust is simpler if we just skip GRUB and have our UEFI
firmware directly load our *Linux* kernel.

We could just sign the official *Debian* kernel, because it\'s compiled
with an embedded boot loader called EFI stub. The issue with that is
that the initrd image will not be signed. If you remember what we said
above, that image is located in the `/boot` partition which is not
encrypted, so an attacker can insert untrusted code in there without
breaking Secure Boot.

What we want to do is to bundle our kernel, our initrd and the kernel
command line in one single bootloader. As a bonus, it would be nice to
do that automatically every time a new kernel is installed.

*EDIT 2023-08-18*: This next part has been updated. The objcopy method
does not work anymore (the offsets probably changed), so it has been
replaced by systemd\'s `ukify` from `systemd-boot`.

You will need to install `systemd-ukify` and `systemd-boot-efi`. Then,
create a file named `/etc/kernel/postinst.d/zz-update-efistub` (the `zz`
part is important so that this is the last script run).

```bash
#!/bin/sh

set -e

/usr/lib/systemd/ukify build \
    --os-release @/usr/lib/os-release \
    --cmdline @/cmdline \
    --linux /vmlinuz \
    --initrd /initrd.img \
    --output "/boot/efi/EFI/debian-direct/Linux.efi"

sbsign \
    --key /root/secureboot/keys/DB.key \
    --cert /root/secureboot/keys/DB.crt \
    --output /boot/efi/EFI/debian-direct/Linux.efi \
    /boot/efi/EFI/debian-direct/Linux.efi
```

Replace the paths to your Secure Boot keys in the script. The output
file will go into `/boot/efi/EFI/debian-direct`, I called it like that
because it skips GRUB and goes directly to linux, but you are free to
rename it however you want.

The kernel command line must be stored in `/cmdline` for the script to
pick it up.

```text
root=/dev/mapper/machine--vg-root panic=0 ro quiet
```

Replace the path to your root file system and the partition type. If you
encounter problems, you might want to remove the `quiet` part to have a
more verbose output.

That `panic=0` part is weird, yet important, we will explain it later.

We also need to run this script whenever the initramfs is updated, set
the execution permission and run it.

```console
# mkdir -p /etc/initramfs/post-update.d
# ln -s ../../kernel/postinst.d/zz-update-efistub /etc/initramfs/post-update.d/zz-update-efistub
# chmod +x /etc/kernel/postinst.d/zz-update-efistub
# mkdir /boot/efi/EFI/debian-direct
# /etc/kernel/postinst.d/zz-update-efistub
```

Now you can add an entry in your EFI setup with that new bootloader:

```console
# efibootmgr --disk /dev/nvme0n1 --create --label "debian-direct" --loader '\EFI\debian-direct\Linux.efi'
```

Try to reboot and run this bundle. If this is not your default entry,
you might want to reorder the entries with `efibootmgr` or in the UEFI
setup. If don\'t want it to be your default entry, you should have a way
to pop the boot selection menu during boot. On my laptop, I need to spam
the `F12` key because the UEFI boots really fast.

If you managed to boot, congratulations! Only two more days are needed
to complete the setup.

### Unlock the LUKS partition with the TPM

I said that we will use the TPM to store the LUKS key and give it back
only when the chain of trust is unbroken. The TPM can store a key
encrypted with hash values coming from what are called PCRs. You can
find a complete list of PCRs
[here](https://ladyitris.wordpress.com/bitlocker-using-tpm/). In this
guide we will use just the following ones, but you are free to do as you
like:

- PCR0: Core System Firmware executable code
- PCR2: extended or pluggable executable code
- PCR7: Secure Boot State

The first two contains a hash of the UEFI firmware and the loaded
modules. PCR7 contains a hash of a state that depends on which key was
used to verify the signature of the EFI bootloader. When booting another
bootloader (like a live distro or *Windows*), PCR7 will hold a different
value which will not be able to unlock the LUKS key stored in the TPM.

{{< note >}}
It looks like some UEFI firmwares do not change PCR7 depending on which
signature was found in the bootloader. If you want to check before
continuing to read this guide, you need to boot once through the
`debian-direct` bundle we just created, and and once through the usual
GRUB boot. After each of these boots, run `tpm2_pcrread` and look at the
PCR7 value (SHA1 or SHA256, it doesn\'t matter here). They should be
different for your setup to be secure.

If they are equal, you can either give up on this guide and use a
passphrase, or you can seal the TPM key using PCR4 (in addition to the
other PCRs). PCR4 has a hash of the boot loader. In our case, this is
the `debian-direct` bundle. The downside to this is that you will need
to re-seal the key in your TPM each time you upgrade your kernel.
{{< /note >}}

Ok, still with us? Make sure that for the following part you didn\'t
boot through GRUB and you are running the new `debian-direct` bundle. We
will store the LUKS key with the current state of the TPM\'s PCRs.

First, we need to create a new key for the LUKS partition:

```console
# dd if=/dev/random bs=64 count=1 | xxd -p -c999 | tr -d '\n' > /root/luks_key
1+0 records in
1+0 records out
64 bytes copied, 5.6501e-05 s, 1.1 MB/s
# cryptsetup luksAddKey /dev/nvme0n1p3 /root/luks_key --pbkdf-force-iterations=4 --pbkdf-parallel=1  --pbkdf-memory=32
Enter any existing passphrase: <enter your existing passphrase here>
```

{{< note >}}
For people wondering why we use a hexadecimal key instead of just a
binary one, it\'s because the tool we\'ll use does not support binary
keys.
{{< /note >}}

{{< note >}}
For people wondering why we are specifying these PBKDF parameters, it\'s
because PBKDFs are used to derive long keys from short, low entropy
passphrases through a costly computation. In our case, the key already
has 512 bits of entropy which is what is needed by the AES algorithm
used. The simple thing to do would be to completely disable the PBKDF,
but cryptsetup does not allow that.
{{< /note >}}

Let\'s register that new key into the TPM:

```console
# tpm2-initramfs-tool seal --data $(cat /root/luks_key) --pcrs 0,2,7
```

You can tweak the PCRs to use here.

*EDIT 2026-05-09*: [\@karolpeszek](https://github.com/karolpeszek) noticed that on some older systems, the SHA256 banks (which are used by default) are always empty. In those situations, disabling secure boot wouldn't change the PCR and the key would be unsealable just as easily. On those systems, consider forcing using the SHA1 banks instead (or throw your machine away).

Now that the key is registered, we need to use it to unlock the
partition during boot. The only place this can be done is in the initrd
image, where the usual passphrase-based unlock occurs.

The basic `tpm2-initramfs-tool` will only try to unlock with the TPM.
Some times this will fail. Depending on what PCRs you have used, the TPM
may be in a different state than now. When this happens, you want to be
able to input your usual passphrase, boot your Linux, and fix the
situation by resealing the key with the above command.

*EDIT 2023-10-08*: The previous implementation of the fallback script
was a bit flaky, `askpass` is more reliable.

To still be able to fallback to the passphrase method, let\'s add a new
script in `/etc/initramfs-tools/tpm2-cryptsetup`:

```bash
#!/bin/sh

[ "$CRYPTTAB_TRIED" -lt "1" ] && exec tpm2-initramfs-tool unseal --pcrs 0,2,7

/usr/bin/askpass "Passphrase for $CRYPTTAB_SOURCE ($CRYPTTAB_NAME): "
```

Make it executable, even if we will never run it on our distro,
otherwise `update-initramfs` will complain.

```console
# chmod +x /etc/initramfs-tools/tpm2-cryptsetup
```

Add a hook to copy all this stuff into the initrd image by creating a
file named `/etc/initramfs-tools/hooks/tpm2-initramfs-tool`:

```bash
#!/bin/sh
PREREQ=""
prereqs()
{
   echo "$PREREQ"
}

case $1 in
prereqs)
   prereqs
   exit 0
   ;;
esac

. /usr/share/initramfs-tools/hook-functions

copy_exec /usr/lib/x86_64-linux-gnu/libtss2-tcti-device.so.0
copy_exec /usr/bin/tpm2-initramfs-tool
copy_exec /usr/lib/cryptsetup/askpass /usr/bin

copy_exec /etc/initramfs-tools/tpm2-cryptsetup
```

The long and obscure prologue to this script is required according to
[the initramfs-tool\'s
manual](https://manpages.debian.org/buster/initramfs-tools-core/initramfs-tools.7.en.html#Header).
Make the script executable too:

```console
# chmod +x /etc/initramfs-tools/hooks/tpm2-initramfs-tool
```

Update your `/etc/crypttab` file:

```text
nvme0n1p3_crypt UUID=375027ee-9720-42ce-bd57-aec0e76f8b4a none luks,discard,keyscript=/etc/initramfs-tools/tpm2-cryptsetup
```

You just need to edit the line you already have in there, just add the
`keyscript` option.

Recreate the initrd image:

```console
# update-initramfs -u -k all
update-initramfs: Generating /boot/initrd.img-5.16.0-6-amd64
warning: data remaining[91961712 vs 91972527]: gaps between PE/COFF sections?
warning: data remaining[91961712 vs 91972528]: gaps between PE/COFF sections?
Signing Unsigned original image
```

You can now reboot and your encrypted partition will be unlocked
straight away without passphrase! If you try and reboot through GRUB
however, the TPM will fail and our script above will ask for the
passphrase to unlock the partition.

## Kernel lockdown by-pass

```
[    0.000000] Kernel is locked down from EFI Secure Boot; see man kernel_lockdown.7
```

Welcome to kernel lockdown! Or maybe you have always been on kernel
lockdown since the beginning. Anyway, this is a feature you will enjoy
much.

### What is kernel lockdown?

Kernel lockdown is a feature that (by-default, on most distros) will
trigger when the machine is using Secure Boot.

When this feature is activated, it will make sure that the code running
in the kernel space is always trusted, right from the Secure Boot, to
keep a chain of trust.

Among other things, kernel lockdown will:

- prevent you from loading unsigned kernel modules (VirtualBox, or your
  favorite Wi-Fi driver you compiled yourself)
- prevent hibernation (suspend to swap) when the swap is unencrypted

You can read about this in `man 5 kernel_lockdown`.

**However**, while what the manual says is true, it might leave you
thinking \"Oh, my swap is encrypted, I\'m not concerned by the second
point\". Well, you are wrong. Hibernation is prevented when swap is
unencrypted, but what is missing from the manual is that hibernation is
**also** prevented when swap is encrypted.

[These
lines](https://github.com/torvalds/linux/blob/3e732ebf7316ac83e8562db7e64cc68aec390a18/kernel/power/hibernate.c#L82-L87)
show that if we are in lockdown, hibernation is disabled, regardless of
the swap state.

We do want these two features, so let\'s (try to) fix this!

### Signing kernel modules

Let\'s start with the easy part. On *Debian*, most additional kernel
modules will come in the form of DKMS modules. DKMS will take care of
recompiling modules whenever a new kernel is installed.

*EDIT 2023-08-18*: This has changed too, but long ago and I didn\'t
update this guide, sorry! The DKMS script was broken and fixed and
changed the configuration format.

*Debian* has thought of people needing to sign their modules. What you
need to do is to simply enable siging in DKMS by editing
`/etc/dkms/sign_helper.sh` and adding the following lines:

```text
mok_signing_key=/root/secureboot/keys/dkms.key
mok_certificate=/root/secureboot/keys/dkms.der
```

Replace with the appropriate paths for your keys. Now, recompile your
current DKMS modules to have them signed:

```console
# /usr/lib/dkms/dkms_autoinstaller start `uname -r`
```

Boom! You\'re done, one less problem!

### Activating hibernation

So why is hibernation blocked in the first place?

This is because during hibernation, you can boot another OS and edit the
swap partition, replace kernel code and resume the system. This way you
would get unsigned code into the kernel, which is not allowed by kernel
lockdown, so it disables hibernation altogether. Except we don\'t care
about this, since our swap is encrypted! Booting another OS will not
allow anyone to inject code in the kernel.

*EDIT 2023-08-18*: Someone implemented a solution that encrypts the swap
and stores the keys in the TPM. [The
patches](https://lore.kernel.org/lkml/20210220013255.1083202-1-matthewgarrett@google.com/)
have never been merged though. The author wrote [a great
article](https://mjg59.dreamwidth.org/55845.html) to explain why they go
to such extents to protect hibernation.

*EDIT 2024-10-18*: [As \@weiss kindly
noted](https://github.com/blastrock/blastrock.github.io/issues/10), the
more general goal of kernel lockdown is to prevent the kernel space from
being modified *at all*, even by root, not just prevent unauthorized
users from tampering with the kernel. In other words, if you boot an
Ubuntu with secure boot, you won\'t be able to reach kernel space at
all, you\'d need to have the Ubuntu signature key for that. The
suggested solutions in this guide give up on that tiny security feature
because it\'s USELESS! (yeah, that\'s just my opinion)

{{< note >}}
You can skip this next rant if you want.

So your kind kernel developers added this constraint and disabled a very
useful (and used) feature without giving you a choice, and your helpful
distro developers decided to enable the lockdown for everyone having
Secure Boot enabled. So if you have just Secure Boot enabled, you get
the absolute tyrannical security of having hibernation disabled. What
happens then? Of course you disable Secure Boot, who cares about
security if you can\'t use your machine! This feature supposed to bring
security forces the users to reduce their security!
{{< /note >}}

We\'re not gonna disable Secure Boot after all these efforts, and we
still want to enable hibernation. We have two solutions here:

#### Recompile your own kernel

If you recompile your kernel, you can disable the feature completely.
This is what I did first, using the configuration from the official
*Debian* kernel.

This solution has a few downsides:

- The compiled kernel will not work anymore with GRUB (because it
  doesn\'t have the official *Debian* signature)
- Every time a new kernel is released, you will need to recompile it and
  not let `apt` install the default one
- Recompiling the kernel takes more than 30 minutes on a decent CPU

So we\'re not continuing with this solution, let\'s skip forward.

#### Disable lockdown on a running kernel

Another solution is to make a kernel module (which runs in kernel space)
and disable the lockdown from there. This is not normally allowed, there
is no function to do that, but there still is a way, and it does not
have all the downsides of recompiling your own kernel.

Download, compile and install [this
project](https://github.com/blastrock/unlockdown):

```console
$ git clone https://github.com/blastrock/unlockdown.git
$ cd unlockdown
$ make dkmsInstall
```

This will install a module that will remove the kernel lockdown when it
is loaded, and put it back when it is unloaded. Since it is compiled by
DKMS, it will be automatically signed thanks to the configuration we set
before.

Finally, we just need to load the module automatically on boot. You
might want add it to `/etc/modules`, and that\'s what I first did, but
it was not enough! If you put it there you will be able to hibernate,
but not to wake up. When you will try to wake up, the kernel will be
back in lockdown and will refuse to restore the resume image. So we
actually want to load the module even earlier, in the initrd phase,
before the kernel attempts to resume. Add the module name to
`/etc/initramfs-tools/modules`:

```console
# echo unlockdown >> /etc/initramfs-tools/modules
# update-initramfs -u -k all
update-initramfs: Generating /boot/initrd.img-5.16.0-6-amd64
warning: data remaining[91961712 vs 91972527]: gaps between PE/COFF sections?
warning: data remaining[91961712 vs 91972528]: gaps between PE/COFF sections?
Signing Unsigned original image
```

*EDIT 2023-08-18*: Actually, we now need one final step. unlockdown uses
an API that\'s now disabled by default (from Linux 6.3 or something). To
enable it back, you need to add the `ibt=off` to the kernel command
line. Note that this disables a vulnerability mitigation. The
`/etc/cmdline` file becomes:

```text
root=/dev/mapper/machine--vg-root panic=0 ibt=off ro quiet
```

You can now reboot and you\'ll be able to hibernate!

In case of trouble, here is a couple checks you can do:

```console
# dmesg | grep unlockdown
[    8.501198] unlockdown: loading out-of-tree module taints kernel.
[    8.540051] unlockdown: kernel lockdown lifted
# pm-is-supported --hibernate && echo 'Hibernate is supported'
Hibernate is supported
```

{{< note >}}
We could have enabled only hibernation and kept the lockdown feature,
which still brings some more security to the kernel. Unfortunately, the
technique we use to force-disable the lockdown feature is not applicable
to hibernation.

Hibernation is driven by a function called `hibernation_available`, and
while we can hijack it with a `kretprobe` to make it return true, it
will not allow hibernation. `pm-is-supported` will report that
hibernation is supported, but `/sys/power/disk` will still report
`[disabled]`. This happens because the `hibernation_available` call is
inlined by the compiler and thus our `kretprobe` is not called in
certain code paths.
{{< /note >}}

## Other security tips

Congratulations, your laptop is mostly secure. Here are a few more tips
to make sure an attacker can\'t get access to it.

### `panic=0`

Do you remember about that `panic=0` part in the kernel command line we
put earlier? It means that when the kernel panics, it will not reboot
automatically. This is completely unrelated to what we are doing\...
except that the initrd scripts will interpret that as \"do not spawn a
root shell if anything goes wrong\".

This is a very important feature, because if you have a root shell in
your Secure Boot environment, you can just unseal the key from the TPM
and decrypt all the data from the disk!

### Protect your UEFI setup

This is not strictly necessary in our threat model, but you should put a
password on your UEFI setup. This will prevent people from messing
around and potentially adding other signatures keys to Secure Boot, even
though these keys will alter PCR7 and prevent the TPM from unlocking
your drive.

### Disable the magic SysRq key

[The magic SysRq key](https://en.wikipedia.org/wiki/Magic_SysRq_key)
allows running some special kernel actions. The most dangerous ones are
disabled by default, and you should keep them that way for maximum
security.

For example, one of them (`f`) will invoke the OOM-killer. This function
*could* kill your lockscreen, giving full access to your desktop to a
malicious user.

### Protect your Secure Boot keys

You don\'t want your private keys to be too easily available, don\'t let
them lie around in your home directory. You should keep them in a
protected folder (like `/root`) so that only the necessary scripts can
access them. Make sure that `/root`\'s permissions are `700`.

## Conclusion

What a journey! I hope this guide helped you even a bit. I am by no mean
an expert on the subject, if you spot something wrong or know of a
better way to do this, [please leave a comment]{.strike}. This is a
simple HTML page, there are no comments, so idk, send me a fax maybe or
open an issue on
[GitHub](https://github.com/blastrock/blastrock.github.io).

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

- <https://docs.microsoft.com/en-us/windows/security/information-protection/bitlocker/bitlocker-countermeasures>
- <https://pawitp.medium.com/full-disk-encryption-on-arch-linux-backed-by-tpm-2-0-c0892cab9704>
- <https://pawitp.medium.com/the-correct-way-to-use-secure-boot-with-linux-a0421796eade>
- <https://www.rodsbooks.com/efi-bootloaders/controlling-sb.html>
- <https://ladyitris.wordpress.com/bitlocker-using-tpm/>
- <https://wiki.debian.org/SecureBoot>
- <https://wiki.debian.org/EFIStub>
- <https://unix.stackexchange.com/questions/83545/initramfs-post-update-hook>
