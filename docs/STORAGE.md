1. Plug the external hard disk into USB 3 port of a Raspberry Pi
2. Run the commands `lsblk` and `fdisk -l` to see the status of the attached storage mediums. The output will be somewhat like this
    ```sh
    $ lsblk
    NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
    loop0         7:0    0  48.9M  1 loop /snap/core18/2127
    loop1         7:1    0    49M  1 loop /snap/core18/2248
    loop2         7:2    0  28.2M  1 loop /snap/snapd/13643
    loop3         7:3    0  60.4M  1 loop /snap/lxd/21544
    loop4         7:4    0  57.4M  1 loop /snap/core20/1171
    loop5         7:5    0  36.5M  1 loop /snap/snapd/13830
    loop6         7:6    0  60.7M  1 loop /snap/lxd/21843
    sda           8:0    0 931.5G  0 disk
    └─sda1        8:1    0 931.5G  0 part
    mmcblk0     179:0    0  29.7G  0 disk
    ├─mmcblk0p1 179:1    0   256M  0 part /boot/firmware
    └─mmcblk0p2 179:2    0  29.5G  0 part /
    $ sudo fdisk -l
    Disk /dev/loop0: 48.92 MiB, 51277824 bytes, 100152 sectors
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes


    Disk /dev/loop1: 48.98 MiB, 51335168 bytes, 100264 sectors
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes


    Disk /dev/loop2: 28.22 MiB, 29581312 bytes, 57776 sectors
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes


    Disk /dev/loop3: 60.41 MiB, 63336448 bytes, 123704 sectors
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes


    Disk /dev/loop4: 57.42 MiB, 60198912 bytes, 117576 sectors
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes


    Disk /dev/loop5: 36.52 MiB, 38281216 bytes, 74768 sectors
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes


    Disk /dev/loop6: 60.68 MiB, 63610880 bytes, 124240 sectors
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes


    Disk /dev/mmcblk0: 29.74 GiB, 31914983424 bytes, 62333952 sectors
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disklabel type: dos
    Disk identifier: 0xf66f0719

    Device         Boot  Start      End  Sectors  Size Id Type
    /dev/mmcblk0p1 *      2048   526335   524288  256M  c W95 FAT32 (LBA)
    /dev/mmcblk0p2      526336 62333918 61807583 29.5G 83 Linux


    Disk /dev/sda: 931.49 GiB, 1000170586112 bytes, 1953458176 sectors
    Disk model: Elements 2621
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disklabel type: gpt
    Disk identifier: 05CC176F-6CAC-49DB-AA8B-19CF42BD4529

    Device     Start        End    Sectors   Size Type
    /dev/sda1   2048 1953456127 1953454080 931.5G Microsoft basic data
    ```
4. Run the command `sudo mkfs.ext4 /dev/sda` to format the disk. Agree to any prompt about deleting existing partitions. We don't need partitions and we won't partition the disk with `fdisk` as it is not really necessary. 
5. Next, we have to mount this disk to a good place. We will use a folder inside `/mnt` for this. Run the following to create the folder, mount the drive and transfer ownership to the current `ubuntu` user.
    ```sh
    sudo mkdir /mnt/ext-hdd-01
    sudo mount /dev/sda /mnt/ext-hdd-01/
    sudo chown -R ubuntu:ubuntu /mnt/ext-hdd-01/
    ```
6. We have to also auto-mount this on startup. For this, first find out the UUID of the disk. Run the following command to get the UUID of the disk.
    ```sh
    sudo blkid /dev/sda
    ```
    Then we add a line with the above UUID to the `/etc/fstab` file. Change my value of the UUID to the one you got.
    ```sh
    echo UUID="e61c1970-d975-4fdf-a855-4738d2628683" /mnt/ext-hdd-01 ext4 defaults 0 0 | sudo tee -a /etc/fstab
    ```
    Reboot the system and check the status of the mount using
    ```sh
    df -ha /dev/sda
    ```
    It should show `/mnt/ext-hdd-01` under the `Mounted on` column.
    ```
    Filesystem      Size  Used Avail Use% Mounted on
    /dev/sda        916G   77M  870G   1% /mnt/ext-hdd-01
    ```
