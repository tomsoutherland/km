## Install arch to a ZFS root  

You will need a custom ISO with zfs support for this install. I have included instructions in this *km* repo to build such an ISO.  Once booted using the ISO, follow these instructions.

1. Wipe the installation disk(s)  
   ```
   wipefs -a /dev/sda
   sgdisk --zap-all /dev/sda
   ```
2. Partition Disk(s). I suggest an EFI, swap, and ZFS data partitions  
   a. Create EFI partition  
   ```
   sgdisk -n "1:1m:+512m" -t "1:ef00" /dev/sda
   ```  
   b. I suggest a swap partition large enough to hold the contents of RAM if you intend to use hibernate (my system has 16g)
   ```
   sgdisk -n "2:0:+16g" -t "2:8200" /dev/sda
   ```
   c. Allocate the remainder to the ZFS pool  
   ```
   sgdisk -n "3:0:-10m" -t "3:bf00" /dev/sda
   ```
3. Create the ZFS pool  
    a. Get the partition UUID for the ZFS data  
      ```
      ls -l /dev/disk/by-partuuid|grep sda3
      lrwxrwxrwx 1 root root 10 Sep  2 10:21 1aea6cc8-8333-4d5c-a9e6-e0a51296391d -> ../../sda3
      ```
    b. This install will be using [zfsbootmenu](https://zfsbootmenu.org) which is currently using OpenZFS 2.3
      ```
      zgenhostid -f 0x00bab10c
      zpool create -f -o ashift=12 \
       -O compression=lz4 \
       -O acltype=posixacl \
       -O xattr=sa \
       -O relatime=on \
       -o autotrim=on \
       -o compatibility=openzfs-2.3-linux \
       -m none zroot /dev/disk/by-partuuid/1aea6cc8-8333-4d5c-a9e6-e0a51296391d
      ```
4. Export / Import the ZFS pool changing altroot to /mnt
   ```
   zpool export zroot && zpool import -NR /mnt zroot
   ```
5. Create the ZFS file systems for the initial install  
   ```
   zfs create -o mountpoint=none zroot/ROOT
   zfs create -o mountpoint=/ -o canmount=noauto zroot/ROOT/arch
   zfs mount zroot/ROOT/arch
   ```
6. Set up EFI and swap partitions  
   ```
   mkfs.vfat -F32 /dev/sda1
   mkswap /dev/sda2
   ```
7. Install the base system. Include the CPU specific ucode (amd / intel)  
   ```
   pacstrap /mnt linux linux-headers linux-firmware base vim amd-ucode networkmanager sudo openssh rsync
   ```
8. Populate the fstab and hostid  
   ```
   echo "$(blkid /dev/sda1|cut -d\  -f2) /boot/efi vfat defaults 0 2" >> /mnt/etc/fstab
   echo "$(blkid /dev/sda2|cut -d\  -f2) none swap defaults 0 0" >> /mnt/etc/fstab
   cp /etc/hostid /mnt/etc
   ```
9. Move into the install  
   ```
   arch-chroot /mnt
   ```
10. Set up arch system  
   a. Time  
      ```
      ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime
      hwclock --systohc
      ```
    b. Locale  
       ```
       sed -i 's/#\(en_US\.UTF-8\)/\1/' /etc/locale.gen
       locale-gen
       echo "LANG=en_US.UTF-8" > /etc/locale.conf
       ```
    c. vconsole  
       ```
       echo -e "KEYMAP=us\n#FONT=latarcyrheb-sun32" > /etc/vconsole.conf
       ```
    d. hostname  
       ```
       echo YOUR_HOSTNAME > /etc/hostname
       ```
    e. root password  
       ```
       passwd root
       ```
11. Create a normal user and add to sudoers by uncommenting one of the *wheel* stanzas.  
    ```
    useradd -G wheel some_user
    passwd some_user
    mkdir -p ~some_user && chown some_user ~some_user
    EDITOR=vim visudo
    ```
12. Change user to some_user and install aur helper and ZFS packages
    ```
    su - some_user
    sudo pacman -S --needed base-devel git
    git clone https://aur.archlinux.org/paru.git
    cd paru
    makepkg -si
    paru -S zfs-utils
    paru -S zfs-dkms
    exit
    ```
13. Edit /etc/mkinitcpio.conf and modify the HOOKS line to include *resume* and *zfs*  
    ```
    vim /etc/mkinitcpio.conf
    grep ^HOOKS /etc/mkinitcpio.conf
    HOOKS=(base udev autodetect microcode modconf kms keyboard keymap zfs consolefont block filesystems resume)
    ```
    Provided the *grep* shows the above, rebuild the boot loader  
    ```
    mkinitcpio -P
    ```
14. We will be making home directories for *root* and *some_user* once the system is rebooted. Clean up for this step.  
    ```
    rm -rf ~some_user
    mv /root /_root
    ```
15. Set up systemd services  
    ```
    systemctl disable zfs-import-cache.service
    systemctl mask zfs-import-cache.service
    systemctl enable zfs-import-scan.service zfs-mount.service zfs.target zfs-volumes.target zfs-import.target zfs-zed.service NetworkManager
    ```
16. Set up the EFI and install zfsbootmenu  
    ```
    zfs set org.zfsbootmenu:commandline="rw quiet" zroot/ROOT
    mkdir /boot/efi
    mount /boot/efi
    mkdir -p /boot/efi/EFI/ZBM
    curl -o /boot/efi/EFI/ZBM/VMLINUZ.EFI -L https://get.zfsbootmenu.org/efi
    cp /boot/efi/EFI/ZBM/VMLINUZ.EFI /boot/efi/EFI/ZBM/VMLINUZ-BACKUP.EFI
    pacman -S refind
    refind-install
    rm /boot/refind_linux.conf
    cat << EOF >> /boot/efi/EFI/refind/refind.conf

    menuentry "zbm boot" {
        loader /EFI/ZBM/VMLINUZ.EFI
        options "quiet loglevel=0 zbm.skip"
    }
    menuentry "zbm menu" {
        loader /EFI/ZBM/VMLINUZ.EFI
        options "quiet loglevel=0 zbm.show"
    }
    menuentry "zbm backup menu" {
        loader /EFI/ZBM/VMLINUZ-BACKUP.EFI
        options "quiet loglevel=0 zbm.show"
    }
    EOF
    ```
17. Exit and reboot  
    ```
    exit
    umount -n -R /mnt
    zpool export zroot
    reboot
    ```
18. Log in as root and finish up  
    ```
    zfs create -o mountpoint=none zroot/homes
    cd /
    rm -rf /root
    zfs create -o mountpoint=/root zroot/homes/root
    rsync -pa /_root/ /root
    rm -rf /_root
    zfs create -o mountpoint=/home/some_user zroot/homes/some_user
    cp -r /etc/skel/. ~some_user
    chown -R some_user:some_user ~some_user
    chmod 750 ~some_user
    ```
19. I suggest taking a snapshot in the event you want to back out any changes made after this point.  
    ```
    zfs snapshot zroot/ROOT/arch@initial
    ```
10. Log out and log in as *some_user* and install your preferred DE
    



