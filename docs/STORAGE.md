# SD Card setup
1. Use Raspberry Pi Image to download and write the OS to an SD Card.
2. If you want to connect a monitor directily, then in the `boot` partition, change the `usercofg.txt` file to have the following:
    ```
    hdmi_force_hotplug=1
    hdmi_group=2
    hdmi_mode=82
    ```

# Network setup
1. Change the hostname by `sudo hostnamectl set-hostname server-pi-01`. This will change the file `/etc/hostname` to add the hostname to it.
The `hostname` file only keeps track of the system hostname and should not be a FQDN.
2. Open `/etc/hosts` and add the following:
    ```
    127.0.1.1 server-pi-01 server-pi-01.sayakm.me
    ```
    So the complete file looks like
    ```
    127.0.0.1 localhost
    127.0.1.1 server-pi-01 server-pi-01.sayakm.me

    # The following lines are desirable for IPV6 capable hosts
    ::1 ip6-localhost ip6-loopback
    fe00::0 ip6-localnet
    ff00:0 ip6-mcastprefix
    ff02::1 ip6-allnodes
    ff02::2 ip6-allrouters
    ff02::3 ip6-allhosts
    ```
    This will ensure that calling by the hostname or the FQDN from within the system will loopback.

3. The next files to be changed will be done using `netplan`. This tool updates the `resolv.conf` via a symlink so **DON'T** update `resolv.conf`. Moreover, `dhcpcd.conf` doesn't exist in Ubuntu so DHCP config needs to be changed from `netplan` which updates `/run/systemd/network` so that it is flushed and recreated at boot. So, `/etc/systemd/network/` doesn't have anything. To configure `netplan`, create a new file `/etc/netplan/60-static-ip.yaml`. This will override the existing file that we don't want to change. Add the following to the file
    ```yaml
    network:
        version: 2
        ethernets:
            eth0:
                dhcp4: no
                addresses: [192.168.0.151/24]
                gateway4: 192.168.0.1
                nameservers:
                    addresses: [8.8.8.8,8.8.4.4,1.1.1.1,1.0.0.1]
    ```
    The above turns off DHCP. That means we need to manually configure what gateway and nameservers to use, as this was earlier provided by the DHCP server. The `addresses` clause is not only setting the static IP but also setting the subnet mask. The `gateway` is taken from the router properties. The `nameservers` are from Google and Cloudflare. Finally do `sudo netplan apply` to commit these changes.

4. Finally if we need to access anything else like a local k8s cluster using FQDN, we will need to update the `etc/hosts` file so that the FQDNs are resolved properly when called. Add lines like the following to the file
    ```
    192.168.0.50 clstr-01-cp-01 clstr-01-cp-01.sayakm.me
    ```
    so that the file finally looks like this
    ```
    127.0.0.1 localhost
    127.0.1.1 server-pi-01 server-pi-01.sayakm.me

    # The following lines are desirable for IPv6 capable hosts
    ::1 ip6-localhost ip6-loopback
    fe00::0 ip6-localnet
    ff00::0 ip6-mcastprefix
    ff02::1 ip6-allnodes
    ff02::2 ip6-allrouters
    ff02::3 ip6-allhosts

    192.168.0.50 clstr-01-cp-01 clstr-01-cp-01.sayakm.me
    192.168.0.51 clstr-01-cp-02 clstr-01-cp-02.sayakm.me
    192.168.0.52 clstr-01-cp-03 clstr-01-cp-03.sayakm.me
    192.168.0.100 clstr-01-nd-01 clstr-01-cp-01.sayakm.me
    192.168.0.101 clstr-01-nd-02 clstr-01-cp-02.sayakm.me
    192.168.0.102 clstr-01-nd-03 clstr-01-cp-03.sayakm.me
    ```
# Storage setup
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
# Install Container Runtimes
1. Now, we are going to install all the components required to run containers. Start by updating the package repository and installing required packages. Then add the GPG key of the repo.
    ```sh
    sudo apt-get update
    sudo apt-get install ca-certificates curl gnupg lsb-release
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
    ```
    Then add the docker `apt` repository.
    ```sh
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list
    ```
    The install `docker-ce`, `docker-ce-cli` and `containerd.io`.
    ```sh
    sudo apt-get update
    sudo apt-get install docker-ce docker-ce-cli containerd.io
    ```
2. Next, to have proper security we need to ensure that docker does not use the `root` user. First, check if the `docker` group is already created.
    ```sh
    getent group | grep docker
    ```
    If not, we create a `docker` group.
    ```sh
    sudo groupadd docker
    ```
    Then add the current user to it.
    ```sh
    sudo usermod -aG docker $USER
    ```
    Relog to ensure that the changes are applied.
3. Next, we will install `docker-compose` as a plugin to `docker`. First, make a directory to store the plugin.
    ```sh
    mkdir -p ~/.docker/cli-plugins
    ```
    Next, download and place the compose binary.
    ```sh
    curl -SL https://github.com/docker/compose/releases/download/v2.2.2/docker-compose-linux-aarch64 -o ~/.docker/cli-plugins/docker-compose
    ```
    Next, make the binary executable.
    ```sh
    chmod +x ~/.docker/cli-plugins/docker-compose
    ```
    Check the installation with
    ```sh
    docker compose version
    ```