---
layout: posts
title: Deniable Encryption
classes: wide
date: 2025-12-30
---  
I remember years ago when I first made a bootable Kali thumb drive ('BackTrack' back then) with encrypted peristence there was an option to create a sort of "self destruct" where if you typed a special password the drive would be erased rather than unlocked.  Kali still offers this package: [cryptsetup-nuke-password](https://www.kali.org/tools/cryptsetup-nuke-password/).  
For some reason this ability still fascinates me.  The option to self desctruct your stuff at a moment's notice.  I've always used Luks encryption on Linux devices but this specific "nuke password" setup doesn't seem to exist much outside of Kali.  
I continued reading and doing some experimenting on an older laptop and eventually decided to set my Framework 13 laptop up with *deniable encryption*.  

* TOC
{:toc}  

# What Is "Deniable Encryption"
Ok, but what exactlyl does "deniable encryption" mean, and where's my self descruct button?  [Wikipedia](https://en.wikipedia.org/wiki/Deniable_encryption) has a decent page describing what deniable encryption means.  Basically for my context this means if anyone were to inspect the laptop or the drive itself there would be no evidence that the drive contained any data or even any encrypted data.  
It turns out that with Luks encryption, by default, the first 16MB of a targeted partition for a Luks container is used to store the Luks header and the keyslots for unlocking the Luks container.  The "nuke password" option from Kali operates by deleting the keys from the Luks header.  This is unrecoverable and basically turns all of your encrypted data into pure entropy.  However, the Luks header still exists and by its exxistence implies that there *was* encrypted data there.  
Luks has the option to work with a detached header, whereby you create the header and store it somewhere else.  This could be a file, or another disk partition.  I experimented with putting both the boot partition and the Luks header on a removable USB drive and using the entire internal drive as a Luks container.  With the USB drive removed there is no bootable partition, and the internal drive looks like random data from the first bit to the end.  

# Why Do This?  
Mostly for fun and because I like to "try" things in order to learn them.  My threat model does not include needing to be able to "nuke" my laptop setup and have plausible deniability about whether or not there was ever any data on it.  I'm also going to use this post as a bit of a guide for myself for installing Arch Linux.  After the disk and Luks setup, the rest of the steps are pretty much the same.  

# I use Arch btw...  
I have mostly used Ubuntu since first getting in to Linux and had only really heard of Arch, but never seen it.  It's been over 2 years since I first tried Arch and I've come to really enjoy it as a distro.  It makes you responsible for every aspect of the computer and puts you in control of it all, for better or for worse.  I've learned a lot more about Linux in general and the Arch Wiki has become my go-to resource for pretty much all Linux questions.  
  
# Boot Arch ISO  
The first step is to boot the Arch Linux ISO.  It's always best to go out and get the most current one from Arch as they publish a new version every month.  I've been using [Ventoy](https://www.ventoy.net/en/index.html) for a while to consolidate dozens of different ISOs on to one physical media.  With a new version of the Arch ISO copied over to my Ventoy drive I'll boot my Framework 13 and mash F12 to get the one-time boot menu.  Select my Ventoy drive, and from the menu select the Arch ISO.  
  
# Disk Setup  
Once the Arch installer finishes booting the first step is to identify the disks we'll be working with.  In my case I've got an internal NVME drive and a Framework 250GB expansion card occupying one of the expansion card slots.  This registers as a USB drive and will end up containing the boot partition as well as the detatched Luks header.  
Using the `lsblk` command to list the current disks:  
{% highlight sh %}
root@archiso ~ # lsblk
NAME       MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0        7:0    0 971.6M  1 loop /run/archiso/airootfs
sda          8:0    0 232.9G  0 disk
sdb          8:16   1 119.5G  0 disk
├─sdb1       8:17   1 119.5G  0 part
│ └─ventoy 253:0    0   1.4G  1 dm
└─sdb2       8:18   1    32M  0 part
nvme0n1    259:0    0 465.8G  0 disk
root@archiso ~ #
{% endhighlight%}  
In this example `/dev/sda` is the expansion drive (250GB) and `/dev/nvme0n1` is the internal NVME drive.  `/dev/sdb` is the USB drive for Ventoy that Arch is booted off of.  
## Partition Disks  
Start with the expansion/removable drive.  
{% highlight sh %}
fdisk /dev/sda
{% endhighlight%}  
Then `g` to start an empty GPT style partition table.  
`n` to start a new partition, default number and first sector. For size set it to `+1GB` to make it a 1GB partition for boot.  
`n` again to a new partition, default number (2) and first sector. For size set it to `+17MB`.  This will be where the detached Luks header is stored.
`n` one last time to start a third partition with default first and last sector.  It's a 250GB drive, figure it'd be nice to be able to use that space for something.  
Lastly, need to set the type for each partition with the `t` command, and choose the following values.  
```
Partition 1 = '1' for an EFI partition.
Partition 2 = '23' for a "Linux root (x86-64)" type
Partition 3 = '23' for a "Linux root (x86-64)" type
```  
`w` to write the changes to disk.  At this point you could create a partition on the NVME drive, but I've found that you can omit this step and just target the whole drive with cryptsetup.  In this way, there isn't even a partition listed when looking at the drive.  
Otherwise I'd create an empty GPT partition table on the NVME drive, create 1 partition with default first and last sectors and type `23`.  
## Shred It  
Plausibly deniable encryption wouldn't quite work if we took a blank disk and started a Luks container on it.  The encrypted data only occupies so many blocks on the device and the rest is zeros.  A sufficiently skilled adversary might be able to determine certain things about the computer based presence, size and location of the used and unused blocks.  It's going to take a bit of time but I'm going to run `shred` against the drive first to overwrite every bit position with random data.  
{% highlight sh %}
root@archiso ~ # shred -n 1 /dev/nvme0n1 -v
{% endhighlight%}  
This will make one pass of random data to the entire NVME drive with the `-v` providing verbose output.  
## Secure It  
Now the actual disk encryption can happen.  
{% highlight sh %}
root@archiso ~ # cryptsetup luksFormat --use-random -h sha512 /dev/nvme0n1 --header=/dev/sda2

WARNING!
========
This will overwrite data on /dev/sda2 irrevocably.

Are you sure? (Type 'yes' in capital letters): YES
Enter passphrase for /dev/sda2:
Verify passphrase:
cryptsetup luksFormat --use-random -h sha512 /dev/nvme0n1 --header=/dev/sda2  24.77s user 0.35s system 130% cpu 19.216 total
{% endhighlight%}  
Here the `--use-random` specifies which random number generator to use.
`-h sha512` is the hash algorithm to be used and the iterations is left at default.  
Then the target drive/partition location followed by the location for the detached header with `--header=/dev/sda2`.  

Next we'll open the encrypted container
{% highlight sh %}
root@archiso ~ # cryptsetup luksOpen /dev/nvme0n1 crypt --header=/dev/sda2
Enter passphrase for /dev/nvme0n1:
cryptsetup luksOpen /dev/nvme0n1 crypt --header=/dev/sda2  7.24s user 0.07s system 134% cpu 5.450 total
{% endhighlight%}  
Now `lsblk` will confirm we have a new partition representing our unlocked Luks container.
{% highlight sh %}
root@archiso ~ # lsblk
NAME       MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
loop0        7:0    0 971.6M  1 loop  /run/archiso/airootfs
sda          8:0    0 232.9G  0 disk
├─sda1       8:1    0     1G  0 part
├─sda2       8:2    0    17M  0 part
└─sda3       8:3    0 231.9G  0 part
sdb          8:16   1 119.5G  0 disk
├─sdb1       8:17   1 119.5G  0 part
│ └─ventoy 253:0    0   1.4G  1 dm
└─sdb2       8:18   1    32M  0 part
nvme0n1    259:0    0 465.8G  0 disk
└─crypt    253:1    0 465.8G  0 crypt
root@archiso ~ #
{% endhighlight%}  
## Format The Partitions  
Next up we need file systems on these partitions so we can use them.  
`mkfs.fat -F32 /dev/sda1` to turn the 1GB partition on our external drive in to a FAT32 boot partition.  
`mkfs.ext4 /dev/mapper/crypt` to turn our unlocked Luks container in to an Ext4 partition to hold our Arch Linux install.  
### Mount Partitions  
Run these commands to mount the boot and root partitions within the live Arch ISO environment.  
```shell
mount /dev/mapper/crypt /mnt
mkdir -p /mnt/boot
mount /dev/sda1 /mnt/boot
```  
# Install Arch 
Parts of this will be slightly different due to the detached Luks header encryption setup, but we'll carry on as if this is a normal install.  
## Pacstrap and Chroot  
Use pacstrap to load some initial packages in to the new blank disk and then arch-chroot for further steps.  
```shell
pacstrap /mnt base base-devel linux linux-firmware intel-ucode git vim sudo networkmanager
```  
Generate fstab.  
`genfstab -U /mnt >> /mnt/etc/fstab`  
chroot into system.
` arch-chroot /mnt`  
## Create User  
Create a user account for the new installation.  
```shell
useradd -m greyed
passwd greyed
```
Add the user to the wheel group, which will determine who has access to sudo.
```shell
gpasswd -a greyed wheel
```
Run visudo, which will open a file where you can add wheel users to sudoers. Uncomment this relevant section:
```shell
## Uncomment to allow members of group wheel to execute any command
%wheel ALL=(ALL) ALL
```  
If you want to increase the time period a sudo auth is good for and change the default editor for sudo now is a good time to add these to visudo as well.  
```shell
Defaults    timestamp_timeout=5
Defaults    editor=/usr/bin/vim
```  
## Locale, Time and Hostname Oh My  
Set time locale (choose a relevant locale):

```shell
ln -sf /usr/share/zoneinfo/US/Pacific /etc/localtime
```

Set clock:

```shell
hwclock --systohc
```

Uncomment `en_US.UTF-8 UTF-8` `en_US ISO-8859-1` or whatever localizations you need in `/etc/locale.gen`. Now run:

```shell
locale-gen
```

Create locale config file:

```shell
locale > /etc/locale.conf
```

Set the lang variable in the above file (Choose the language code that is relevant to you):

```shell
LANG=en_US.UTF-8
LANGUAGE=en_US
```

Add an hostname (any hostname of your choice as one line in the file. eg. `myhostname`):

```shell
vim /etc/hostname
```
Update `/etc/hosts` to contain (replace `myhostname` with the host name you used above):

```text
127.0.1.1   myhostname.localdomain  myhostname
```  
## mkinitcpio and boot  
Edit mkinitcpio.conf to add the right kernel hooks for unlocking the disk with a detached header
```
HOOKS=(base systemd autodetect modconf kms keyboard sd-vconsole plymouth sd-encrypt block filesystems fsck)
```
Note this has 'plymouth' included for a no splash boot, but it can be added later. 

Then create the file /etc/crypttab.initramfs to specify that my detached header was on a separate drive.
```
crypt /dev/nvme0n1 none header=/dev/disk/by-partuuid/15f1a4e5-940b-4a14-b48d-c7af015597d6
```
where "crypt" is the name of the mapped decrypted Luks container. The next argument is the location of the Luks container, and then the header is specified using the partition UUID.  The `blkid` command can return can return the details for a specific drive
```
blkid /dev/sda2
/dev/sda2: UUID="f818c840-81ae-4085-a7c1-d44bf62833f1" TYPE="crypto_LUKS" PARTUUID="15f1a4e5-940b-4a14-b48d-c7af015597d6"
```
Then you can use that PARTUUID in the crypttab file

Regenerate the initramfs:

```shell
mkinitcpio -p linux
```

Install a bootloader:

```shell
bootctl --path=/boot/ install
```

Create bootloader. Edit `/boot/loader/loader.conf`. Replace the file's contents with:

```text
default arch
timeout 0
```
Now to make the loader entry to support booting this detached Luks header setup
file at /boot/loader/entries/arch.conf
```
title Arch Linux
version 6.18.2
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
options cryptdevice=/dev/nvme0n1:crypt root=/dev/mapper/crypt rw quiet splash

```

The cryptdevice should normally be specified by partition UUID but because we targeted the entire NVME drive, there is no partition UUID.  This is followed by the name of the opened Luks container, how it will be mapped and then the final `quiet splash` parts are for a Plymouth setup.

A final check of the `/etc/fstab` file to make sure it's using the correct UUIDs
```
# Static information about the filesystems.
# See fstab(5) for details.

# <file system> <dir> <type> <options> <dump> <pass>
# /dev/mapper/crypt
UUID=a4d5cd3a-7edc-454d-8830-990c75df61b1   /           ext4        rw,relatime 0 1

# /dev/sda1
PARTUUID=5fcda863-68ff-4e36-8955-4bf32e8bc667    /boot       vfat        rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro   0 2
```
the first is the UUID, from blkid, of the /dev/mapper/crypt disk and then the boot partition referenced as a partition UUID since it might change otherwise at every boot.  This also allows us to have a clone of the removable drive as a backup.  More on this later.  
## Exit and Reboot  
Exit the arch-chroot
`exit`
and unmount the drives  
`umount -R /mnt`

after first login, setup Network Manager
```shell
systemctl start NetworkManager
systemctl enable NetworkManager
```  
Voila! that's a basic Arch setup.  
# Backup the Luks Header  
As discussed, without the Luks header the data on the encrypted drive is completely unrecoverable.  If the external drive that our detatched Luks header was on failed, went missing, or got destroyed we would have no other option but to start from scratch.  The Luks header can be backed up to a file at any time with a syntax like this.  
```shell
$ sudo cryptsetup luksHeaderBackup /dev/nvme0n1 --header-backup-file headerbackup.img
```  
Then that file can be stored safely somewher else.  
Additionally every time the kernel is updated the boot partition will get a new initramfs.  If we only cloned the Luks header to a new, similar drive this 'backup' drive would eventually end up being unbootable as the kernel version deviated from the rest of the system.  Running `dd` every so often on a 250GB drive would be tedious, error prone, and time consuming.  
I wrote a small bash script that simplifies this for me.  I can plug in my other Framework 250GB expansion drive and run this script to keep a functional backup for myself.  
{% highlight sh %}
#!/bin/bash
#for cloning the current boot disk to a target disk containing the same partitions


# Must be run as root
if [[ $EUID -ne 0 ]]; then
    echo "This script must be run as root (sudo)." 1>&2
    exit 1
fi

boot_dev=$(findmnt -no SOURCE /boot)
boot_disk=$(lsblk -no PKNAME "$boot_dev")
source_disk="/dev/$boot_disk"
echo "Source Disk: $source_disk"
UUID=$(blkid -s UUID -o value "$boot_dev")
duplicate=$(blkid -s UUID | grep "$UUID" | grep -v "^$boot_dev")

if [[ -n $duplicate ]]; then
    echo "Target disk found:"
    echo "    $duplicate"
    clone_dev=$(blkid -s UUID | grep "$UUID" | grep -v "^$boot_dev" | awk -F: '{print $1}')
    clone_disk=$(lsblk -no PKNAME "$clone_dev")
    target_disk="/dev/$clone_disk"
    echo "Source: $source_disk"
    echo "Target: $target_disk"
    echo "---------------------"
    read -p "Check kernel versions before cloning? (yes/no): " answer1

    if [[ "$answer1" = "yes" ]]; then
        mntdir="/mnt/targetboot"
        mkdir $mntdir
        mount "${target_disk}1" $mntdir
        source_ver=$(lsinitcpio /boot/initramfs-linux.img -a | grep 'Kernel' | awk -F: '{print $2}')
        target_ver=$(lsinitcpio "${mntdir}/initramfs-linux.img" -a | grep 'Kernel' | awk -F: '{print $2}')
        umount $mntdir
        rmdir $mntdir
        echo "Source Kernel Version:$source_ver"
        echo "Target Kernel Version:$target_ver"
        echo "---------------------"
    fi

    if [[ "$source_disk" == "$target_disk" ]]; then
        echo "Error: source and target disks are the same!"
        exit 1
    fi

    read -p "This will completely overwrite $target_disk. Continue? (yes/no): " answer2

    if [[ "$answer2" != "yes" ]]; then
        echo "Clone aborted."
        exit 1
    fi

    dd if="${source_disk}1" of="${target_disk}1" bs=4M status=progress
    dd if="${source_disk}2" of="${target_disk}2" bs=4M status=progress
else
    echo "Target disk NOT found."
fi
{% endhighlight %}
