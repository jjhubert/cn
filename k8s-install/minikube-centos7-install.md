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
    - 8G
- cpu
    - number:4
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
  > br_netfilter
  > overlay
  > EOF
  br_netfilter
  [root@k8s-00 ~]# modprobe  br_netfilter
  [root@k8s-00 ~]# modprobe overlay
  [root@k8s-00 ~]# cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
  > net.bridge.bridge-nf-call-ip6tables = 1
  > net.bridge.bridge-nf-call-iptables = 1
  > net.ipv4.ip_forward                 = 1
  > EOF
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
  yum install docker-ce docker-ce-cli containerd.io 
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

### MiniKube
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

- install kubelet kubectl
  ``` shell
  yum install -y kubelet kubectl --disableexcludes=kubernetes
  systemctl enable --now kubelet
  ```

- install minikube
  ```shell
  curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-latest.x86_64.rpm
  sudo rpm -Uvh minikube-latest.x86_64.rpm 
  ```

- create common user
  ```shell 
  adduser ops
  usermod -aG root ops
  usermod -aG docker ops
  passwd ops
  ```

- switch user
  ```shell 
  su - ops
  ```

- minikube create cluster`From a terminal with administrator access (but not logged in as root), run`
  ```shell 
  minikube start --image-mirror-country=cn --image-repository='registry.cn-hangzhou.aliyuncs.com/google_containers'  --driver=docker --kubernetes-version v1.22.2
  ```
