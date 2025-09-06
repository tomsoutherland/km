## Create a custom ISO with ZFS support
I ran into issues getting a customer ISO built using the instructions on the arch wiki but I found the following procedure to work.

1. Install archiso  
```pacman -S archiso```

 2. Create build tree in /var/tmp  
 ``cd /var/tmp; mkdir -p archiso-zfs/{work,isobuild}``

 3. Copy in proto ISO  
 ``cp -r /usr/share/archiso/configs/releng/. archiso-zfs``

 4. Included required packages in archiso-zfs/packages.x86_64  
    ```
    cat << EOF >> archiso-zfs/packages.x86_64
    linux-headers
    libunwind
    zfs-utils
    zfs-dkms
    EOF
    ```
5. Add custom repo to archiso-zfs/pacman.conf  
   ```
   cat << EOF >> archiso-zfs/pacman.conf
   [archzfs]
   SigLevel = TrustAll Optional
   Server = file:///var/tmp/archzfs
   EOF
   ```
6. Create working archzfs directory and populate  
   ```
   mkdir archzfs
   cd archzfs
   git clone https://aur.archlinux.org/zfs-dkms.git
   git clone https://aur.archlinux.org/zfs-utils.git
   ```
7. Build packages and make the custom repo  
   ```
   cd zfs-dkms
   makepkg -s
   cd ../zfs-utils
   makepkg -s
   cd ..
   for f in */*zst; do ln $f .; done
   repo-add /var/tmp/archzfs/archzfs.db.tar.zst /var/tmp/archzfs/*/*zst
   ```
8. Move into archiso-zfs and build the ISO  
   ```
   cd ../archiso-zfs/
   sudo mkarchiso -v -w work -o isobuild /var/tmp/archiso-zfs
   ```
9. When this completes, you should have an ISO in the *isobuild* directory


