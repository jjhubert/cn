## Preparation
- [virtualbox6.1](https://www.virtualbox.org/)
- [centos iso](https://mirrors.aliyun.com/centos/7/isos/x86_64/)
## Install

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

### Golang
- install epel repo
    ```shell
    yum install -y epel-release 
    ```

- update repo
    ```shell
    yum update
    ```

- install golang
    ```shell
    yum install -y golang
    ```

- set GOPROXY and GO111MODULE env variable
    ```shell
    export GOPROXY="https://goproxy.cn,direct"
    export GO111MODULE="on"
    ```

### Kubectl
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

- install
  ``` shell
  yum install -y kubectl --disableexcludes=kubernetes
  ```

### Kind
- install
    ```shell
    go get sigs.k8s.io/kind@v0.11.1 && export PATH=$PATH:$(go env GOPATH)/bin && kind create cluster
    ```

- create a simple cluster
    ```shell
    [root@localhost ~]# kind create cluster
    Creating cluster "kind" ...
     ✓ Ensuring node image (kindest/node:v1.21.1) 🖼 
     ✓ Preparing nodes 📦  
     ✓ Writing configuration 📜 
     ✓ Starting control-plane 🕹️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️️ 
     ✓ Installing CNI 🔌 
     ✓ Installing StorageClass 💾 
    Set kubectl context to "kind-kind"
    You can now use your cluster with:
    
    kubectl cluster-info --context kind-kind
    
    Have a nice day! 👋
    
    ```