# SD Card setup
1. Use Raspberry Pi Image to download and write the OS to an SD Card.
2. In the `boot` paertition, change the `usercofg.txt` file to have the following:
    ```
    hdmi_force_hotplug=1
    hdmi_group=2
    hdmi_mode=82
    ```

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
    127.0.1.1 clstr-01-cp-01 clstr-01-cp-01.sayakm.me

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
    127.0.1.1 clstr-01-cp-01 clstr-01-cp-01.sayakm.me

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
5. Now we start installing k8s related stuff. Start with `containerd` CRE. Run
    ```sh
    sudo apt-get install containerd
    ```
    Next, we have to initialise the default config if not present. Check if the folder `/etc/containerd` exists. If not, create it. 
    ```sh
    sudo mkdir -p /etc/containerd
    ```
    Then create the `config.toml` file.
    ```sh
    containerd config default | sudo tee /etc/containerd/config.toml
    ```
    Next, we need to use `systemd` cgroup driver in the config file with `runc`. Update the above `config.toml` file with
    ```
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
    ...
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
        SystemdCgroup = true
    ```
    The restart the `containerd` service.
    ```sh
    sudo systemctl restart containerd
    ```
6. We need to load some networking modules for iptables to work. Load the `br_netfilter` module.
    ```sh
    sudo modprobe br_netfilter
    ```
    We also need to ensure that this module is alwasys loaded on boot. So add it to a file in the `modules-load.d` folder.
    ```sh
    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
    overlay
    br_netfilter
    EOF
    ```
    Then to ensure that the node iptables correctly see bridged traffic, we need to add the following to the `/etc/sysctl.d/k8s.conf`. Also reload sysctl.
    ```sh
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-ip6tables = 1
    net.ipv4.ip_forward                 = 1
    net.bridge.bridge-nf-call-iptables  = 1
    EOF
    sudo sysctl --system
    ```
    Restart the `containerd` server.
    ```sh
    sudo systemctl restart containerd
    ```
7. Now we are ghoing to install kubernetes and related packages. Start by 
updating the packages and  installing `apt-transport-https` and `curl`. Then add the GPG key of the repo.
    ```sh
    sudo apt-get update
    sudo apt-get install -y apt-transport-https ca-certificates curl
    sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
    ```
    Then add the Kubernetes `apt` repository. This repo still doesn't have a folder newer than `xenial`.
    ```sh
    echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
    ```
    Then install `kubelet`, `kubeadm`, and `kubectl` and pin their versions.
    ```sh
    sudo apt-get update
    sudo apt-get install kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl
    ```
    The last command is used to ensure that upgrades don't change their versions.
8. It is also important to have swap disabled permanently. It is already disabled if you have 8GB RAM during installation. If it's not disabled, disable it using the following
    ```
    sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
    sudo swapoff -a
    ```
9. Next we need to ensure if the memory cgroups are enabled. To check, use `cat /proc/cgroups` and see of the value of `enabled` for `memory` is `1` or not. If not, we have to edit the file `/boot/firmware/cmdline.txt` and add the following to the end of the line
    ```
    cgroup_enable=memory
    ```
    Then reboot the system. After the reboot, check again if the value of `enabled` for `memory` is `1` or not.
10. The next steps include setting the system up for a Highly Available Control Plane. This neccesitates the presence of a Load Balancer and in this case we are going to use a software best Load Balancer called `kube-vip`. To install `kube-vip`, first we need to create a config file which will be used to convert it into a manifest which will be use by `kubeadm` while initialising to create a static pod of `kube-vip`. Start by pulling the image of `kube-vip` and creating an alias to run the container.
    ```sh
    sudo ctr images pull  ghcr.io/kube-vip/kube-vip:v0.4.0
    alias kube-vip="sudo ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:v0.4.0 vip /kube-vip
    ```
    You can optionally permanently store the above alias in the `~/.bash_aliases`.
    ```sh
    if [ ! -f ~/.bash_aliases ]; then
        echo kube-vip="sudo ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:v0.4.0 vip /kube-vip" > ~/.bash_aliases
    elif grep -Fq "kube-vip=" ~/.bash_aliases; then
        sed -i "/kube-vip=/s/^\(.*\)$/kube-vip='sudo ctr run --rm --net-host ghcr.io\/kube-vip\/kube-vip:v0.4.0 vip \/kube-vip'/g" ~/.bash_aliases
    else
        echo kube-vip="sudo ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:v0.4.0 vip /kube-vip" >> ~/.bash_aliases
    fi

    . ~/.bashrc
    ```
11. Next generate the manifest for the static pod. We are turning on HA for the control plane, and load balancers for both the control plane and the worker nodes.
    ```sh
    kube-vip manifest pod \
        --interface eth0 \
        --vip 192.168.0.150 \
        --controlplane \
        --services \
        --arp \
        --leaderElection \
        --enableLoadBalancer | sudo tee /etc/kubernetes/manifests/kube-vip.yaml
    ```
12. Finally, we initialise the cluster using `kubeadm init`. We need to ensure that the installed kubernetes version is the same as kubeadm. So, first do
    ```sh
    KUBE_VERSION=$(sudo kubeadm version -o short)
    ```
    Then, initialise the cluster
    ```sh
    sudo kubeadm init --control-plane-endpoint "192.168.0.150:6443" --upload-certs --kubernetes-version=$KUBE_VERSION --pod-network-cidr=10.244.0.0/16
    ```
    `--control-plane-endpoint` is the IP address of the load balancer as set earlier in `kube-vip`.

    `--upload-certs` is used to upload the certificates to the kubernetes cluster automatically without us needing to supply them.

    `--kubernetes-version` is the version of kubernetes that you are using so that newer versions are not automatically used.

    `--pod-network-cidr` is the CIDR block for the pod network. This is necessary for Flannel to work.
13. The output should look something like this
    ```
    [init] Using Kubernetes version: v1.22.2
    [preflight] Running pre-flight checks
            [WARNING SystemVerification]: missing optional cgroups: hugetlb
    [preflight] Pulling images required for setting up a Kubernetes cluster
    [preflight] This might take a minute or two, depending on the speed of your internet connection
    [preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
    [certs] Using certificateDir folder "/etc/kubernetes/pki"
    [certs] Generating "ca" certificate and key
    [certs] Generating "apiserver" certificate and key
    [certs] apiserver serving cert is signed for DNS names [clstr-01-cp-01 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.0.50 192.168.0.150]
    [certs] Generating "apiserver-kubelet-client" certificate and key
    [certs] Generating "front-proxy-ca" certificate and key
    [certs] Generating "front-proxy-client" certificate and key
    [certs] Generating "etcd/ca" certificate and key
    [certs] Generating "etcd/server" certificate and key
    [certs] etcd/server serving cert is signed for DNS names [clstr-01-cp-01 localhost] and IPs [192.168.0.50 127.0.0.1 ::1]
    [certs] Generating "etcd/peer" certificate and key
    [certs] etcd/peer serving cert is signed for DNS names [clstr-01-cp-01 localhost] and IPs [192.168.0.50 127.0.0.1 ::1]
    [certs] Generating "etcd/healthcheck-client" certificate and key
    [certs] Generating "apiserver-etcd-client" certificate and key
    [certs] Generating "sa" key and public key
    [kubeconfig] Using kubeconfig folder "/etc/kubernetes"
    [kubeconfig] Writing "admin.conf" kubeconfig file
    [kubeconfig] Writing "kubelet.conf" kubeconfig file
    [kubeconfig] Writing "controller-manager.conf" kubeconfig file
    [kubeconfig] Writing "scheduler.conf" kubeconfig file
    [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
    [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
    [kubelet-start] Starting the kubelet
    [control-plane] Using manifest folder "/etc/kubernetes/manifests"
    [control-plane] Creating static Pod manifest for "kube-apiserver"
    [control-plane] Creating static Pod manifest for "kube-controller-manager"
    [control-plane] Creating static Pod manifest for "kube-scheduler"
    [etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
    [wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
    [apiclient] All control plane components are healthy after 29.562791 seconds
    [upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
    [kubelet] Creating a ConfigMap "kubelet-config-1.22" in namespace kube-system with the configuration for the kubelets in the cluster
    [upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
    [upload-certs] Using certificate key:
    feb5064c88b7e3a154b5deb1d6fb379036e7a4b76862fcf08c742db7031624d9
    [mark-control-plane] Marking the node clstr-01-cp-01 as control-plane by adding the labels: [node-role.kubernetes.io/master(deprecated) node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
    [mark-control-plane] Marking the node clstr-01-cp-01 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
    [bootstrap-token] Using token: 39w134.5w0n8s3ktz63rv47
    [bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
    [bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
    [bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
    [bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
    [bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
    [bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
    [kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
    [addons] Applied essential addon: CoreDNS
    [addons] Applied essential addon: kube-proxy

    Your Kubernetes control-plane has initialized successfully!

    To start using your cluster, you need to run the following as a regular user:

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

    Alternatively, if you are the root user, you can run:

    export KUBECONFIG=/etc/kubernetes/admin.conf

    You should now deploy a pod network to the cluster.
    Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
    https://kubernetes.io/docs/concepts/cluster-administration/addons/

    You can now join any number of the control-plane node running the following command on each as root:

    kubeadm join 192.168.0.150:6443 --token REDACTED \
            --discovery-token-ca-cert-hash sha256:REDACTED \
            --control-plane --certificate-key REDACTED

    Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
    As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
    "kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

    Then you can join any number of worker nodes by running the following on each as root:

    kubeadm join 192.168.0.150:6443 --token REDACTED \
            --discovery-token-ca-cert-hash sha256:REDACTED
    ```
14. Next follow the instructions in the above output to craete the kubectl config file in the home directory.
    ```sh
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```
15. Next we will deploy the pod network. We are going to use Flannel for this. We are using Flannel 0.15.0 for this. It's always better to anchor the version.
    ```sh
    kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/v0.15.0/Documentation/kube-flannel.yml
    ```