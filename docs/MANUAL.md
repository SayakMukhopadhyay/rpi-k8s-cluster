# SD Card setup
1. Use Raspberry Pi Image to download and write the OS to an SD Card.
2. In the `boot` paertition, change the `usrconfg

# Network setup
1. Change the hostname by `sudo hostnamectl set-hostname clstr-01-cp-01`. This will change the file `/etc/hostname` to add the hostname to it.
The `hostname` file only keeps track of the system hostname and should not be a FQDN.
2. Open `/etc/hosts` and add the following at the bottom:
```
127.0.1.1 clstr-01-cp-01 clstr-01-cp-01.sayakm.me
```
So the complete file looks like
```
127.0.0.1 localhost

# The following lines are desirable for IPV6 capable hosts
::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00:0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts

127.0.1.1 clstr-01-cp-01 clstr-01-cp-01.sayakm.me
```
This will ensure that calling by the hostname or the FQDN from within the system will loopback.

3. The next files to be changed will be done using `netplan`. This tool updates the `resolv.conf` via a symlink so **DON'T** update `resolv.vonf`. `dhcpcd.conf` doesn't exist in Ubuntu so DHCP config needs to be changed from `netplan` which updates `/run/systemd/network` so that it is flushed and recreated at boot. So, `/etc/systemd/network/` doesn't have anything. To configure `netplan`, create a new file `/etc/netplan/60-static-ip.yaml`. This will override the existing file that we don't want to change. Add the following to the file
```yaml
network:
    version: 2
    ethernets:
        eth0:
            dhcp4: no
            addresses: [192.168.0.50/24]
            gateway4: 192.168.0.1
            nameservers:
                addresses: [8.8.8.8,8.8.4.4,1.1.1.1,1.0.0.1]
```
The above turns off DHCP. That means we need to manually configure what gateway and nameservers to use, as this was earlier provided by the DHCP server. The `addresses` clause is not only setting the static IP but also setting the subnet mask. The `gateway` is taken from the router properties. The `nameservers` are from Google and Cloudflare. Finally fo `sudo netplan apply` to commit these changes.

4. Finally we will update the `etc/hosts` file so that the FQDNs are resolved properly when called from cluster to cluster. Add the following to the file in each cluster
```
192.168.0.50 clstr-01-cp-01 clstr-01-cp-01.sayakm.me
192.168.0.51 clstr-01-cp-02 clstr-01-cp-02.sayakm.me
192.168.0.52 clstr-01-cp-03 clstr-01-cp-03.sayakm.me
192.168.0.100 clstr-01-nd-01 clstr-01-cp-01.sayakm.me
192.168.0.101 clstr-01-nd-02 clstr-01-cp-02.sayakm.me
192.168.0.102 clstr-01-nd-03 clstr-01-cp-03.sayakm.me
```
so that the file finally looks like this
```
127.0.0.1 localhost

# The following lines are desirable for IPv6 capable hosts
::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts

127.0.1.1 clstr-01-cp-01 clstr-01-cp-01.sayakm.me

192.168.0.50 clstr-01-cp-01 clstr-01-cp-01.sayakm.me
192.168.0.51 clstr-01-cp-02 clstr-01-cp-02.sayakm.me
192.168.0.52 clstr-01-cp-03 clstr-01-cp-03.sayakm.me
192.168.0.100 clstr-01-nd-01 clstr-01-cp-01.sayakm.me
192.168.0.101 clstr-01-nd-02 clstr-01-cp-02.sayakm.me
192.168.0.102 clstr-01-nd-03 clstr-01-cp-03.sayakm.me
```
