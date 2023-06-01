---
title: 配置Kubernetes实验环境
date: 2023-4-28 15:01:39
tags: ['IT', 'Kubernetes', 'K8S','Manual']
description: '在Virtual Box中配置Kubernetes实验环境'
keywords: ['Kubernetes', 'CKA']
# top_img: img/配置Kubernetes实验环境.webp
category: ['IT', 'Container']
---

# 配置Kubernetes实验环境

## 说明
为了熟悉kubernetes的操作与配置，搭建拥有3个Node的Cluster。
1. 创建3个虚拟机，配置网络为双网卡，NAT+NAT netwok
2. 在3个虚拟机上安装ubuntu系统，并指定电脑名称分别为 master, worker1, worker2
3. 在3个虚拟机上安装kubernetes环境
4. 在master上初始化集群管理服务，worker1和worker2加入master集群

## 配置Virtualbox
为Virtualbox添加NAT network [参考](https://www.cnblogs.com/ddvv/p/6882514.html)
``` text
192.168.10.0/24
DHCP enabled
```

## 下载[ubuntu-20.4.5镜像](https://www.ubuntu.com/download)并创建 3 个虚拟机
虚拟机配置
```
CPU 4核
RAM 4G
硬盘 20G
```
``` SHELL
# 修改电脑名
sudo hostname master # 其他两个虚拟机为 worker1, worker2

# 分别在3个虚拟机上配置hosts DNS
sudo vim /etc/hosts
#192.168.10.101 master
#192.168.10.102 worker1
#192.168.10.103 worker2

# 配置静态IP，以master为例
# 
sudo vim /etc/netplan/00-installer-config.yaml
```
``` YAML
network:
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.10.101/24 # master
  version: 2
```


## 配置Ubuntu系统
1. 修改ubuntu的镜像源: http://mirrors.163.com/ubuntu
1. 配置Ubuntu
``` SHELL
sudo swapoff -a
sudo vim /etc/fstab #注释掉 swap配置,一般在最后一行
# 安装依赖包
sudo apt-get install -y apt-transport-https ca-certificates curl
# 配置IPv4
# Forwarding IPv4 and letting iptables see bridged traffic
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
lsmod | grep br_netfilter
lsmod | grep overlay

# 确认配置结果
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

## 安装containerd [参考](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd)

1. 安装Containerd
``` SHELL
sudo apt-get install -y containerd
sudo mkdir /etc/containerd
sudo containerd config default > config.toml && sudo mv config.toml /etc/containerd/
```
1. 设定Systemd `/etc/containerd/config.toml`
``` INI
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```
1. 设定cri sanbox_image
``` INI
[plugins."io.containerd.grpc.v1.cri"]
  sandbox_image = "registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.6"
```
1. 重启containerd
``` SHELL
sudo service containerd restart
```

## 使用阿里云镜像源，安装Kubernetes
``` SHELL
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add - 
cat <<EOF >kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
sudo mv kubernetes.list /etc/apt/sources.list.d/

sudo apt-get update
sudo apt-get install -y kubelet=1.24.0-00 kubeadm=1.24.0-00 kubectl=1.24.0-00
sudo apt-mark hold kubelet kubeadm kubectl
```

## 在master(192.168.10.101)上创建集群
``` SHELL
kubeadm init --apiserver-advertise-address=192.168.10.101 --node-name=k8s-master --kubernetes-version=1.24.0 --pod-network-cidr=10.32.1.0/24 --image-repository registry.aliyuncs.com/google_containers --v=99
```
## 安装Calico网络组建

``` SHELL
wget https://docs.projectcalico.org/v3.23/manifests/calico.yaml
kubectl apply -f calico.yaml
```
注意需要修改以下`calico.yaml`
``` YAML
- name: CALICO_IPV4POOL_CIDR
              value: "10.32.1.0/24"
```

## 在worker1(192.168.10.102)和worker2(192.168.10.103)上修改kubelet服务IP地址
1. 编辑10-kubeadm.conf
``` SHELL
vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```
2. 修改node-ip
``` YAML
Environment="KUBELET_EXTRA_ARGS=--node-ip=192.168.10.102"
```
3. 重启kubelet
``` SHELL
systemctl daemon-reload
systemctl restart kubelet
```

4. 然后使用mater创建时的输出提示
``` SHELL
kubeadm join
```
