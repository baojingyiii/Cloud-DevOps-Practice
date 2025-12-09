## 项目：企业级DevOps平台实战

### 项目架构图

text

```
用户请求 → Nginx Ingress → Spring Boot微服务 → MySQL/Elasticsearch
    ↑                              ↑
监控平台(Prometheus+Grafana)      CI/CD流水线(GitLab CI+Argo CD)
```

------

## 第一阶段：环境规划与准备

### 1.1 节点规划（使用虚拟机）

| 节点类型 | IP地址        | 主机名     | 配置  | 用途        |
| -------- | ------------- | ---------- | ----- | ----------- |
| Master   | 192.168.56.10 | k8s-master | 4核8G | K8s控制平面 |
| Node1    | 192.168.56.11 | k8s-node1  | 4核8G | 工作节点    |
| Node2    | 192.168.56.12 | k8s-node2  | 4核8G | 工作节点    |

**最低要求**：2台服务器（1Master+1Node），每台至少2核4G，20GB磁盘

### 1.2 系统准备（所有节点）

bash

```
# 1. 设置主机名
hostnamectl set-hostname k8s-master  # 在master节点执行
hostnamectl set-hostname k8s-node1   # 在node1节点执行
hostnamectl set-hostname k8s-node2   # 在node2节点执行

# 2. 修改hosts文件
cat >> /etc/hosts << EOF
192.168.56.10 k8s-master
192.168.56.11 k8s-node1
192.168.56.12 k8s-node2
EOF

# 3. 关闭防火墙和SELinux
systemctl stop firewalld
systemctl disable firewalld
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# 4. 关闭swap
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 5. 配置内核参数
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

------

## 第二阶段：Docker安装

### 2.1 所有节点安装Docker 20.10

bash

```
# 卸载旧版本
yum remove -y docker

# 安装依赖
yum install -y yum-utils device-mapper-persistent-data lvm2

# 添加阿里云Docker仓库
#yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# 添加Docker仓库
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# 安装指定版本Docker
yum install -y docker-ce-20.10.23 docker-ce-cli-20.10.23 containerd.io

# 配置Docker
mkdir -p /etc/docker
cat > /etc/docker/daemon.json << EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "registry-mirrors": [
    "https://registry.docker-cn.com",
    "https://docker.mirrors.ustc.edu.cn"
  ]
}
EOF

# 启动Docker
systemctl daemon-reload
systemctl enable --now docker
