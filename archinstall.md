# Arch Install Guide

|   |   |
|---|---|
|Author|Keith Patton|
|Date|2019-05-24|
|Revision|1.0|


## Configuration

These instructions will walk you through the process of installing Arch Linux in the following format:

* UEFI
* x86_64
* (Full partition details below)
* Single LVM Physical Volume
* Single LVM Volume Group
* LVM Logical Volumes for : root, home
* GRUB2 bootloader
* SWAP file rather than partition
* Dual boot Arch and Windows

## Disk layout

                 ╔═════════════════════╗╔════════════╗
                 ║    /root (100GB)    ║║/home (30GB)║ [LVM Logical Volumes]
                 ╚═════════════════════╝╚════════════╝

                 |---------------vg-00---------------| [LVM Volume Group]

    ┌───────────┐┌───────────────────────────────────┐┌──────────────────────┐
    │EFI (500MB)││          [LVM] PV (130GB)         ││     empty (100GB)    │
    └───────────┘└───────────────────────────────────┘└──────────────────────┘

    ┌────────────────────────────────────────────────────────────────────────┐
    │                           PHYSICAL DISK (256GB)                        │
    └────────────────────────────────────────────────────────────────────────┘

### Reasoning

The above disk layout provides a dual-boot environment with Arch Linux being the mainstay OS and Windows being a less important / destructable OS.

Using LVM allows for realtime snapshot and resizing of the Arch Linux install that may be useful for backups / upgrades.

The remaining free disk space can be used to install Windows once Linux is installed.

## Instructions

1. **Boot up the install media using UEFI.**

2. **Set the keyboard layout for the installation steps**
  ```bash
  loadkeys uk
  ```

3. **Verify we are booting using UEFI**
  ```bash
  ls /sys/firmware/efi/efivars
  ```
  Confirm the **`efivars`** directory exists - if not, you have not booted the machine using UEFI

4. **Connect to internet for install**

  Check your network interface is detected and shows as **`UP`**
  ```bash
  ip link
  ```

  Confirm you have an IP on your interface
  ```bash
  ip a
  ```

  Ping a public IP to confirm network activity
  ```bash
  ping 1.1.1.1
  ```

  Ping a public FQDN to confirm DNS routing
  ```bash
  ping www.google.co.uk
  ```

5. **Update system clock**
  ```bash
  timedatectl set-ntp true
  ```

6. **Partition the disks**
  
  Use **`lsblk`** to identify your disks
  ```bash
  lsblk
  ```

  An example layout:
  The following shows a system with 1x 1TB SATA disk, and 1x 240GB SSD _(NVMe)_ - each with a single partition.

        NAME                  MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
        sda                     8:0    0 931.5G  0 disk
        └─sda1                  8:1    0 931.5G  0 part
        nvme0n1               259:0    0 232.9G  0 disk
        └─nvme0n1p1           259:1    0    10G  0 part

  The following shows 2x unpartitioned disks.

        NAME                  MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
        sda                     8:0    0 931.5G  0 disk
        nvme0n1               259:0    0 232.9G  0 disk

  In the above examples, the disk we will choose to work with would be **`nvme0n1`** as our main OS disk.
  **Please note : your disk may be different. If you only have sda (SSD) disks and not NVMe (nvme01) then use `/dev/sda` as your disk to partition.**

  **a. Partition the main OS disk**

  Connect to the main disk
  Create a new partition table
  Create a new physical partition for EFI (500MB)
  Mark the partition type as EFI
  Label the partition **EFI**
  Create a new physical partition for LVM physical volume (100GB)
  Mark the partition type as LVM
  Label the partition **Linux**

    ```bash
    # NOTE : where it asks for confirmation, choose 'y'
    
    gdisk /dev/nvme0n1
    # (or /dev/sda if you only have SSD disks)

    o

    n
    # (confirm)
    # (accept default partition start point)
    +500M
    ef00
    c
    EFI

    n
    # (confirm)
    # (accept default partition start point)
    +130G
    8e00
    c
    # (choose your partition - likely 2)
    Linux

    w
    ```

  Confirm the partitions have been written to disk.
  If successful, you should now have a partition table similar to the example below.
  ```bash
  lsblk
  ```
          NAME                  MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
          sda                     8:0    0 931.5G  0 disk
          nvme0n1               259:0    0 232.9G  0 disk
          ├─nvme0n1p1           259:1    0   512M  0 part
          └─nvme0n1p2           259:2    0   130G  0 part

  **b. Create LVM Volumes**

  First, load the device mapper.
  ```bash
  modprobe dm_mod
  ```

  Confirm the system is seeing the newly partitioned disk as containing an LVM capable partition on the NVMe drive.
  ```bash
  lvmdiskscan
  ```
       /dev/nvme0n1   [    <232.89 GiB]
       /dev/nvme0n1p1 [     512.00 MiB]
       /dev/nvme0n1p2 [     130.00 GiB]
       0 disks
       2 partitions
       0 LVM physical volume whole disks
       0 LVM physical volume

  Create the `LVM physical volume` on the **second partition** of the main disk.
  Do not create an LVM Physical Volume on the first partition as this will be used by your bootloader which may not recognise LVM partitions _(GRUB does, but that is outside the scope of this quick guide)_.

  ```bash
  pvcreate /dev/nvme0n1p2
  ```

  Confirm the LVM volume got created
  ```bash
  pvdisplay
  ```
       --- Physical volume ---
       PV Name               /dev/nvme0n1p2
       VG Name               
       PV Size               130.00 GiB / not usable 4.00 MiB
       Allocatable           yes
       PE Size               4.00 MiB
       Total PE              33279
       Free PE               33279
       Allocated PE          0
       PV UUID               <this-will-be-unique>

  Create the `LVM volume group` that will hold all of the LVM logical volumes.
  LVM can manage multiple groups (from m=numerous physical volumes), each containing multiple logical volumes.
  For simplicity we will call this first, main, volume group **`vg-00`** on the LVM physcial volume.
  ```bash
  vgcreate vg-00 /dev/nvme0n1p2
  ```

  Confirm the LVM volume group got created
  ```bash
  vgdisplay
  ```
       --- Volume group ---
       VG Name               vg-00
       System ID             
       Format                lvm2
       Metadata Areas        1
       Metadata Sequence No  3
       VG Access             read/write
       VG Status             resizable
       MAX LV                0
       Cur LV                0
       Open LV               0
       Max PV                0
       Cur PV                1
       Act PV                1
       VG Size               <130.00 GiB
       PE Size               4.00 MiB
       Total PE              33279
       Alloc PE / Size       0 / <130.00 GiB
       Free  PE / Size       33279 / 0   
       VG UUID               <this-will-be-unique>

   Create the `LVM logical volumes` that will hold your filesystems.
   As outlined previously we will be creating `/root` (100GB) and `/home` (30GB).

   ```bash
   lvcreate -L 100G vg-00 -n arch-lv-root
   lvcreate -L 100%FREE vg-00 -n arch-lv-home
   ```

   Confirm the LVM logical volumes got created.
   ```bash
   lvdisplay
   ```
       --- Logical volume ---
       LV Path                /dev/vg-00/arch-lv-home
       LV Name                arch-lv-home
       VG Name                vg-00
       LV UUID                <unique>
       LV Write Access        read/write
       LV Creation host, time archiso, <datetime>
       LV Status              available
       # open                 1
       LV Size                30.00 GiB
       Current LE             7680
       Segments               1
       Allocation             inherit
       Read ahead sectors     auto
       - currently set to     256
       Block device           254:0

       --- Logical volume ---
       LV Path                /dev/vg-00/arch-lv-root
       LV Name                arch-lv-root
       VG Name                vg-00
       LV UUID                <unique>
       LV Write Access        read/write
       LV Creation host, time archiso, <dateime>
       LV Status              available
       # open                 1
       LV Size                <100.00 GiB
       Current LE             25599
       Segments               1
       Allocation             inherit
       Read ahead sectors     auto
       - currently set to     256
       Block device           254:1

  Online the devices.
  ```bash
  vgscan
  vgchange -ay
  ```

  You should now have LVM configured in the following way:
       Physical volume /dev/nvme0n1p2 (130GB)
       Volume Group vg-00
       + Logical Volume /dev/vg-00/arch-lv-root (100GB)
       + Logical Volume /dev/vg-00/arch-lv-home (30GB)

  Configure the init creator `/etc/mkinitcpio.conf` to recognise LVM partitions on boot.

  ```bash
  vim /etc/mkinitcpio.conf
  ```

  Add **`systemd`** and **`sd-lvm2`** hooks to the config as follows :

       HOOKS=(base systemd autodetect modconf block sd-lvm2 filesystems keyboard fsck)

7. **Format the partitions**

 Format the `EFI` partition using **FAT32**.
  ```bash
  mkfs.fat -F32 /dev/nvme0n1p1
  ```

  Format `/` and `/home` to **ext4** using the new LVM logical volume mappings.
  ```bash
  mkfs.ext4 /dev/vg-00/arch-lv-root
  mkfs.ext4 /dev/vg-00/arch-lv-home
  ```
8. **Mount the filesystems**

  Importantly, mount the root `/` filesystem first, then mount every other filesystem to the correct mountpoint on top of `/`.

  Mount Arch **root** partition to `/`
  Create the **boot directory**
  Mount **EFI** partition to `/efi`
  Create the **home directory**
  Mount the Arch **home** partition to `/home`
  ```bash
  mount /dev/vg-00/arch-lv-root /mnt

  mkdir /mnt/efi
  mount /dev/nvme0n1p1 /mnt/efi

  mkdir /mnt/home
  mount /dev/vg-00/arch-lv-home /mnt/home
  ```
  **Bug fix**
  There is a current _(2019-05-28)_ bug in lvm2 / systemd which does not like chroot into LVM volumes.
  Correct the issue by mount the LVM runtime details into a mountpoint in chroot
  ```bash
  mkdir /mnt/hostlvm
  mount --bind /run/lvm /mnt/hostlvm
  ```

9. **Install the base packages**

  Firstly, populate the pacman GPG keyring.
  
  ```bash
  pacman-key --populate archlinux
  pacman-key --refresh-keys
  pacman -Syy
  ```
  
  Now install the base Arch Linux packages along with any additional packages. Here I have added packages for an additional **Linux LTS kernel**, `git` and `wget` for installing `yay` (AUR), and some packages for connecting to wifi from TTY.
  
  ```bash
  pacstrap /mnt base base-devel git wget sudo vim lvm2 linux-lts iw wpa_supplicant dialog
  ```

10. **Create the permanent Filesystem Table config**
  ```bash
  genfstab -U /mnt >> /mnt/etc/fstab
  ```

11. **Change the root environment to the new install**
  ```bash
  arch-chroot /mnt
  ln -s /hostlvm /run/lvm
  modprobe dm_mod
  vgscan
  vgchange -ay
  ```

12. **Set the timezone for you new install**
  ```bash
  ln -sf /usr/share/zoneinfo/Europe/Dublin /etc/localtime
  ```

  Create the time offset
  ```bash
  hwclock --systohc
  ```

13. **Set the localization of the new install**

  Tell the system which locales you wish to use in the system.
  Typically English (UK) and English (USA).

  Uncomment `en_GB.UTF-8 UTF-8`, `en_GB ISO-8859-1` and `en_US.UTF-8 UTF-8`, `en_US ISO-8859-1` as below:
  ```bash
  vim /etc/locale.gen
  ```

        ...
        ...
        en_GB.UTF-8 UTF-8
        en_GB ISO-8859-1
        ...
        ...
        en_US.UTF-8 UTF-8
        en_US ISO-8859-1
        ...
        ...

  Generate the locales
  ```bash
  locale-gen
  ```

  Create the system-wide LANG variable.
  Add `LANG=en_GB.UTF-8` to **/etc/locale.conf** _(For International English)_
  ```bash
  vim /etc/locale.conf
  ```

        LANG=en_GB.UTF-8

  Set the terminal keyboard layout.
  ```bash
  vim /etc/vconsole.conf
  ```

        KEYMAP=uk

14. **Configure system hosts**

  Create your system's hostname.
  ```bash
  vim /etc/hostname
  ```

        <set-your-hostname-to-something>

  Add your hostname to the system hosts file
  ```bash
  vim /etc/hosts
  ```

         127.0.0.1		localhost
         ::1			  localhost
         127.0.1.1		<your-system-hostname>.localdomain	<your-system-hostname>

15. **Generate an initial boot disk for the bootloader**
  ```bash
  mkinitcpio -p linux
  ```

16. **Install the CPU microcode**

  Install one of the following packages:
  * If using Intel : `intel-ucode`
  * If using AMD : `amd-ucode`

  Firstly create the pacman-keyring
  ```bash
  pacman-key --populate archlinux
  pacman-key --refresh-keys
  pacman -Syy
  ```

  Install the microcode package (Intel)
  ```bash
  pacman -S intel-ucode
  ```

  **OR**

  Install the microcode package (AMD)
  ```bash
  pacman -S amd-ucode
  ```

16. **Install the bootloader**

  Install GRUB2
  ```bash
  pacman -S grub efibootmgr os-prober
  grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
  grub-mkconfig -o /boot/grub/grub.cfg
  ```

18. **Allow the `wheel` group to use `sudo`**
  ```bash
  chmod o+w /etc/sudoers
  vim /etc/sudoers
  ```
        ## Uncomment to allow members of group wheel to execute any command
        %wheel ALL=(ALL) ALL

  ```bash
  chmod o-w /etc/sudoers
  ```

18. **Create a new user for yourself**
  ```bash
  useradd -c "<some-comment-here-about-you>" -mUG wheel <your-username-here>
  ```
  
  Set a password for the new user
  ```bash
  passwd <your-new-user>
  ```

17. **Change the root password for security**
  ```bash
  passwd
  ```

18. **Leave CHROOT, umount the disks and reboot to your new install**
  ```bash
  exit
  umount -R /mnt
  reboot
  ```

## Final Steps

At this point, if everything went well, your system should boot up to the newly installed Arch distro with only a terminal session.
You should login as your new user and perform any additional installations / configurations.

For example to install the nVidia drivers and Gnome Desktop
```bash
sudo pacman -S nvidia gnome gnome-extra gnome-control-center gdm networkmanager
```

For a full list of final recommendations see : https://wiki.archlinux.org/index.php/General_recommendations