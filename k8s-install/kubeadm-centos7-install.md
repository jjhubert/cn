## Preparation
- [virtualbox6.1](https://www.virtualbox.org/)
- [centos iso](https://mirrors.aliyun.com/centos/7/isos/x86_64/)
## Install

### VirtualBox
- create a new box
  - name
  - folder path
  - type: Linux
  - version: Other Linux (64-bit)
- memory
  - 4096M
- cpu
  - number:2
  - peak load: 100%
- hard disk
  - create a virtual hard disk now
  - choose VDI
  - Fixed size
  - 100GB
- network
  - adapter1
    - Attached to: Bridge
    - Advanced: default
  - adapter2
    - Attached to: Host-only Adapter
    - advanced:default

- set iso file
  - choose your box, right click it, then choose setting
  - storage
  - choose the logo which is like a CD
    - Attributes => Optical Drive: (click the CD logo and choose your centos iso file)

### Centos
- config hostname
  ``` shell
  hostnamectl set-hostname k8s-00
  ```
- set network adapter
  - adapter1
  ``` shell
  [root@k8s-00 ~]# vi /etc/sysconfig/network-scripts/ifcfg-enp0s3 
  TYPE=Ethernet
  PROXY_METHOD=none
  BROWSER_ONLY=no
  BOOTPROTO=dhcp
  DEFROUTE=yes
  IPV4_FAILURE_FATAL=no
  IPV6INIT=no
  IPV6_AUTOCONF=no
  IPV6_DEFROUTE=no
  IPV6_FAILURE_FATAL=no
  IPV6_ADDR_GEN_MODE=stable-privacy
  NAME=enp0s3
  DEVICE=enp0s3
  ONBOOT=yes
  ```

  - adapter2
  ``` shell
  [root@k8s-00 ~]# vi /etc/sysconfig/network-scripts/ifcfg-enp0s8
  TYPE=Ethernet
  PROXY_METHOD=none
  BROWSER_ONLY=no
  BOOTPROTO=static
  DEFROUTE=yes
  IPV4_FAILURE_FATAL=no
  IPV6INIT=no
  IPV6_AUTOCONF=no
  IPV6_DEFROUTE=no
  IPV6_FAILURE_FATAL=no
  IPV6_ADDR_GEN_MODE=stable-privacy
  NAME=enp0s8
  DEVICE=enp0s8
  ONBOOT=yes
  IPADDR=192.168.100.2
  NETMASK=255.255.255.0
  GATEWAY=192.168.100.1
  DNS1=114.114.114.114
  DNS2=119.29.29.29
  ```

- edit /etc/hosts
  ``` shell
  [root@k8s-00 ~]# vi /etc/hosts
  
  127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4 k8s-00
  ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
  192.168.100.2 k8s-00
  192.168.100.3 k8s-01
  192.168.100.4 k8s-02
  192.168.100.5 k8s-03
  
  ```
  
- disable selinux
  ``` shell
  [root@k8s-00 ~]# vi /etc/selinux/config 
  # This file controls the state of SELinux on the system.
  # SELINUX= can take one of these three values:
  #     enforcing - SELinux security policy is enforced.
  #     permissive - SELinux prints warnings instead of enforcing.
  #     disabled - No SELinux policy is loaded.
  SELINUX=disabled
  # SELINUXTYPE= can take one of three values:
  #     targeted - Targeted processes are protected,
  #     minimum - Modification of targeted policy. Only selected processes are protected.
  #     mls - Multi Level Security protection.
  #SELINUXTYPE=targeted
  
  [root@k8s-00 ~]# setenforce 0
  ```

- disable firewalld
  ``` shell
  systemctl stop firewalld
  systemctl disable firewalld
  systemctl status firewalld
  ```

- disable swap
  ``` shell
  [root@k8s-00 ~]# swapoff -a
  [root@k8s-00 ~]# vi /etc/fstab
  #
  # /etc/fstab
  # Created by anaconda on Sun Aug 22 00:58:48 2021
  #
  # Accessible filesystems, by reference, are maintained under '/dev/disk'
  # See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
  #
  /dev/mapper/centos-root /                       xfs     defaults        0 0
  UUID=be6fcb6b-d425-4cc6-9e97-0bba2b1c7236 /boot                   xfs     defaults        0 0
  /dev/mapper/centos-home /home                   xfs     defaults        0 0
  #/dev/mapper/centos-swap swap                    swap    defaults        0 0  // Comment out the current line
  ```
  
- iptables
  ``` shell
  [root@k8s-00 ~]# yum install -y bridge-utils.x86_64
  [root@k8s-00 ~]# cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
  br_netfilter
  overlay
  EOF
  [root@k8s-00 ~]# modprobe  br_netfilter
  [root@k8s-00 ~]# modprobe overlay
  [root@k8s-00 ~]# cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
  net.bridge.bridge-nf-call-ip6tables = 1
  net.bridge.bridge-nf-call-iptables = 1
  net.ipv4.ip_forward                 = 1
  EOF
  [root@k8s-00 ~]# sudo sysctl --system
  
  ```
  
- set timezone sync
  ``` shell
  yum install -y ntp
  systemctl enable ntpd
  systemctl start ntpd
  timedatectl set-timezone Asia/Shanghai
  timedatectl set-ntp yes
  ntpq -p
  ```
  
### Docker
- clear origin docker
  ``` shell
  yum remove docker docker-client docker-client-latest docker-common  docker-latest docker-latest-logrotate  docker-logrotate  docker-engine
  ```
  
- add utils
  ``` shell
  yum install -y yum-utils 
  ```

- add docker repo
  ``` shell
  yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo 
  ```

- install docker container.d
  ``` shell
  yum install -y docker-ce docker-ce-cli containerd.io 
  ```

- start docker
  ``` shell
  systemctl start docker
  systemctl enable docker
  ```

- modify cgroup driver
  ``` shell
  [root@k8s-00 ~]# vi /etc/docker/daemon.json
  
  {
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
  }


  [root@k8s-00 ~]# systemctl daemon-reload
  [root@k8s-00 ~]# systemctl restart docker
  [root@k8s-00 ~]# systemctl enable docker
  ```

### Containerd
- set containerd config file
  ```shell
  mkdir -p /etc/containerd
  containerd config default | sudo tee /etc/containerd/config.toml
  systemctl restart containerd
  
  vi /etc/containerd/config.toml
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
  ```

- restart containerd
  ```shell
  sudo systemctl restart containerd  
  ```

### Kubeadm
- yum source
  ``` shell
  vi /etc/yum.repos.d/kubernetes.repo
  
  # choose the right baseurl which you feel better in your country.
  [kubernetes]
  name=Kubernetes
  baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
  enabled=1
  gpgcheck=0
  ```

- install kubelet kubeadm
  ``` shell
  yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
  systemctl enable --now kubelet
  ```

- specify kubelet  cgroup driver
  ``` shell
  [root@k8s-00 ~]# vi /etc/sysconfig/kubelet
  
  KUBELET_EXTRA_ARGS=--cgroup-driver=systemd
  ```
- you can make a snapshot to clone more node

- verify the MAC address and product_uuid are unique for every node.


### Cluster
***
#### Master
- kubeadm init
  - by command
  ``` shell
  # --image-repository: Choose a container registry to pull control plane images from (default "k8s.gcr.io")
  # --kubernetes-version:Choose a specific Kubernetes version for the control plane. (default "stable-1")
  # --apiserver-advertise-address: The IP address the API Server will advertise it's listening on. If not set the default network interface will be used
  # --pod-network-cidr: Specify range of IP addresses for the pod network. If set, the control plane will automatically allocate CIDRs for every node.
  # --service-cidr: Use alternative range of IP address for service VIPs. (default "10.96.0.0/12")
  # --cri-socket: Path to the CRI socket to connect. If empty kubeadm will try to auto-detect this value; use this option only if you have more than one CRI installed or if you have non-standard CRI socket.
  kubeadm init --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.22.0 --apiserver-advertise-address=192.168.100.3 --pod-network-cidr=10.244.0.0/16
  ```
  - by kubeadm config
    - generate config file
    ```shell
    kubeadm config print init-defaults --kubeconfig ClusterConfiguration > kubeadm.yml
    ```
    - modify config file
    ```shell
    vi kubeadm.yml
    apiVersion: kubeadm.k8s.io/v1beta3
    bootstrapTokens:
    - groups:
      - system:bootstrappers:kubeadm:default-node-token
      token: abcdef.0123456789abcdef
      ttl: 24h0m0s
      usages:
      - signing
      - authentication
    kind: InitConfiguration
    localAPIEndpoint:
      # set api server
      advertiseAddress: 192.168.100.2
      bindPort: 6443
    nodeRegistration:
      criSocket: /var/run/dockershim.sock
      imagePullPolicy: IfNotPresent
      # set node name
      name: k8s-01
      taints: null
    ---
    apiServer:
      timeoutForControlPlane: 4m0s
    apiVersion: kubeadm.k8s.io/v1beta3
    certificatesDir: /etc/kubernetes/pki
    clusterName: kubernetes
    controllerManager: {}
    dns: {}
    etcd:
      local:
        dataDir: /var/lib/etcd
    # set image repository
    imageRepository: registry.aliyuncs.com/google_containers
    kind: ClusterConfiguration
    kubernetesVersion: 1.22.0
    networking:
      # set pod subnet
      podSubnet: 10.244.0.0/16
      dnsDomain: cluster.local
      serviceSubnet: 10.96.0.0/12
    scheduler: {}
    ---
    kind: KubeletConfiguration
    apiVersion: kubelet.config.k8s.io/v1beta1
    # set cgroup driver
    cgroupDriver: systemd
    ```

    - init cluster
    ```shell
    kubeadm config images list --config kubeadm.yml
    kubeadm config images pull --config kubeadm.yml
    kubeadm init --config=kubeadm.yml --upload-certs | tee kubeadm-init.log 
    ```
    
- init output
    ```
    [init] Using Kubernetes version: v1.22.0
    [preflight] Running pre-flight checks
    [preflight] Pulling images required for setting up a Kubernetes cluster
    [preflight] This might take a minute or two, depending on the speed of your internet connection
    [preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
    [certs] Using certificateDir folder "/etc/kubernetes/pki"
    [certs] Generating "ca" certificate and key
    [certs] Generating "apiserver" certificate and key
    [certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local node] and IPs [10.96.0.1 192.168.100.3]
    [certs] Generating "apiserver-kubelet-client" certificate and key
    [certs] Generating "front-proxy-ca" certificate and key
    [certs] Generating "front-proxy-client" certificate and key
    [certs] Generating "etcd/ca" certificate and key
    [certs] Generating "etcd/server" certificate and key
    [certs] etcd/server serving cert is signed for DNS names [localhost node] and IPs [192.168.100.3 127.0.0.1 ::1]
    [certs] Generating "etcd/peer" certificate and key
    [certs] etcd/peer serving cert is signed for DNS names [localhost node] and IPs [192.168.100.3 127.0.0.1 ::1]
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
    [apiclient] All control plane components are healthy after 5.003151 seconds
    [upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
    [kubelet] Creating a ConfigMap "kubelet-config-1.22" in namespace kube-system with the configuration for the kubelets in the cluster
    [upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
    [upload-certs] Using certificate key:
    ec9250f4974ce8b00f6b58ac1395db39390c6ba297cdffc7f688aecd5162aa11
    [mark-control-plane] Marking the node node as control-plane by adding the labels: [node-role.kubernetes.io/master(deprecated) node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
    [mark-control-plane] Marking the node node as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
    [bootstrap-token] Using token: abcdef.0123456789abcdef
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
    
    Then you can join any number of worker nodes by running the following on each as root:
    
    kubeadm join 192.168.100.3:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:eb92a9bdbf1d09863aa4deab7d3260e12511c8af97489add0f67ed5e70b66fc4 
    
    ```

- copy kubeconfig
  ```shell 
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  
  export KUBECONFIG=/etc/kubernetes/admin.conf
  ```
  
- node join in cluster
  ```shell 
  kubeadm join 192.168.100.3:6443 --token abcdef.0123456789abcdef \
      --discovery-token-ca-cert-hash sha256:eb92a9bdbf1d09863aa4deab7d3260e12511c8af97489add0f67ed5e70b66fc4 
  ```

- download calico cni plugin && set cidr ([calico doc](https://docs.projectcalico.org/getting-started/kubernetes/quickstart))
  
  ```shell
  curl -O https://docs.projectcalico.org/manifests/tigera-operator.yaml
  curl -O https://docs.projectcalico.org/manifests/custom-resources.yaml
  [root@k8s-01 ~]# vi custom-resources.yaml 
    
  apiVersion: operator.tigera.io/v1
  kind: Installation
  metadata:
    name: default
  spec:
    # Configures Calico networking.
    calicoNetwork:
      # Note: The ipPools section cannot be modified post-install.
      ipPools:
      - blockSize: 26
        # set pod subnet
        cidr: 10.244.0.0/16
        encapsulation: VXLANCrossSubnet
        natOutgoing: Enabled
        nodeSelector: all()
  
  ---
  
  # This section configures the Calico API server.
  # For more information, see: https://docs.projectcalico.org/v3.21/reference/installation/api#operator.tigera.io/v1.APIServer
  apiVersion: operator.tigera.io/v1
  kind: APIServer
  metadata:
    name: default
  spec: {}
   ```

- create calico
  ```shell
  kubectl create -f tigera-operator.yaml
  kubectl create -f custom-resources.yaml
  ```
  
- confirm that all of pods are running with following command
  ```shell
  watch kubectl get pods -n calico-system 
  ```

- remove the taint on the master so that you can schedule pods on it.
  ```shell 
  kubectl taint nodes --all node-role.kubernetes.io/master-
  ```


#### worker
- join cluster
  ```shell 
  kubeadm join 192.168.100.3:6443 --token abcdef.0123456789abcdef \
      --discovery-token-ca-cert-hash sha256:eb92a9bdbf1d09863aa4deab7d3260e12511c8af97489add0f67ed5e70b66fc4 
  ```