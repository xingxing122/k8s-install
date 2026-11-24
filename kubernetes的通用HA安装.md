+++
date = '2026-11-22T23:04:04+08:00'
draft = false 
title = 'Kubernetes的通用安装'
categories = ["kubernetes"]
tags = ["kubernetes"]

+++

Kubernetes的通用安装
<!--more-->

#### 环境准备

##### 系统准备

###### 1. 关闭swap 分区

```bash

swapoff -a && sysctl -w vm.swappiness=0
sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab
```

###### 2. 设置limit  

```bash
# vim /etc/security/limits.conf
* soft nofile 655360
* hard nofile 131072
* soft nproc 655350
* hard nproc 655350
* soft memlock unlimited
* hard memlock unlimited
```

###### 4. 内核配置

```bash
cat > /etc/sysctl.d/kubernetes.conf <<EOF 
#将桥接的IPv4流量传递到iptables 的链
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward                = 1
EOF
sysctl --system # 生效

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
# 应用 sysctl 参数而不重新启动
sudo sysctl --system
#安装nftables kube-proxy 代理模式 
dnf install -y nftables
```

##### 安装containerd

```bash
yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
                  
yum install -y yum-utils
yum-config-manager    --add-repo     https://download.docker.com/linux/centos/docker-ce.repo
yum install -y containerd.io
```

###### 1 containerd 配置修改

```bash
containerd config default > /etc/containerd/config.toml


#vim /etc/containerd/config.toml  修改镜像或者存储目录
sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.6"

sed -i "s#sandbox = 'registry.k8s.io/pause:3.10.2'#sandbox = 'registry.aliyuncs.com/google_containers/pause:3.10.2'#" /etc/containerd/config.toml

按照k8s版本号https://v1-36.docs.kubernetes.io/zh-cn/docs/setup/production-environment/container-runtimes/#containerd 对应的参数
[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc]
  ...
  [plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options]
    SystemdCgroup = true  # 改为true 

systemctl daemon-reload
systemctl restart containerd
```

##### kubeadm 安装

###### 配置源

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.36/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```

###### 安装

```bash
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

##### k8s安装

```bash
##生成初始化文件
kubeadm config print  init-defaults  >>  kubeadm-init.yaml 

#修改内容如下
kind: InitConfiguration
  # local 如果是单网卡的可以自动寻找匹配，
  # localAPIEndpoint:
  # advertiseAddress: 1.2.3.4
  # bindPort: 6443
nodeRegistration:

# 证书时间改为100年
caCertificateValidityPeriod: 876000h0m0s
certificateValidityPeriod: 87600h0m0s

clusterName: kubernetes
# 所有节点和管理机都应将此域名解析到负载均衡 VIP。
controlPlaneEndpoint: api.k8s.io:8443

imageRepository: registry.aliyuncs.com/google_containers

scheduler: {}
---
#添加kubelet配置参数
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
---
###指定kube-proxy 默认代理模式，默认是iptables 
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: nftables
```

###### 安装nginx 

```bash
yum install nginx nginx-mod-stream -y

# 修改配置文件，添加如下
vim  /etc/nginx/nginx.conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log notice;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;
events {
    worker_connections 10240;
}
stream {
    upstream api.k8s.io {
        least_conn;
        # 配置master IP 高可用节点
        server 10.37.129.3:6443 max_fails=3 fail_timeout=30s;
        #server 10.0.0.12:6443 max_fails=3 fail_timeout=30s;
        #server 10.0.0.13:6443 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 8443;
        proxy_pass api.k8s.io;
        proxy_connect_timeout 5s;
        proxy_timeout 300s;
    }
}
http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    keepalive_timeout   65;
    types_hash_max_size 4096;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    include /etc/nginx/conf.d/*.conf;
}
# 启动nginx 
systemctl  restart nginx
```



##### 初始化安装k8s

```bash
#先看看需要拉取的镜像  
kubeadm config images list --config kubeadm-init.yaml
#拉取镜像
kubeadm config images pull --config kubeadm-init.yaml 
#初始化
kubeadm init --config kubeadm-init.yaml --upload-certs
```

##### 安装calico 

```bash
#参考https://docs.tigera.io/calico/latest/getting-started/kubernetes/nftables 
#因为kube-proxy
安装
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/v3_projectcalico_org.yaml  
#安装tigera operator 
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/tigera-operator.yaml 

#安装自定义calico 
cat > custom-resources.yaml <<EOF
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    linuxDataplane: Nftables
    ipPools:
    - name: default-ipv4-ippool
      blockSize: 26
      cidr: 10.244.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
EOF
#安装
kubectl create -f custom-resources.yaml
```

 ##### 验证

```bash
#确认所有pod 是否正正常运行
watch kubectl get pods -n calico-system
#清除master 上的污点，允许调度pod 
kubectl taint nodes --all node-role.kubernetes.io/control-plane-

#查看集群中已有的节点
kubectl get nodes -o wide
#命令补全 
dnf install -y bash-completion

# kubectl
kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl > /dev/null
# kubeadm
kubeadm completion bash | sudo tee /etc/bash_completion.d/kubeadm > /dev/null
# crictl
crictl completion bash | sudo tee /etc/bash_completion.d/crictl > /dev/null


```

