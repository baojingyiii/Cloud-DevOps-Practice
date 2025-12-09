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
### 卸载旧版本
`yum remove -y docker`

### 安装依赖
`yum install -y yum-utils device-mapper-persistent-data lvm2`

### 添加阿里云Docker仓库
`yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo`
### 添加Docker仓库
#`yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo`

### 安装指定版本Docker
`yum install -y docker-ce-20.10.23 docker-ce-cli-20.10.23 containerd.io`

### 配置Docker
```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "registry-mirrors": ["镜像加速器"]   
}
EOF
```

### 启动Docker
`systemctl daemon-reload`
`systemctl enable --now docker`




# 1. 添加Kubernetes仓库

### 2. 安装指定版本
yum install -y kubelet-1.23.15-0 kubeadm-1.23.15-0 kubectl-1.23.15-0 --disableexcludes=kubernetes

### 3. 配置kubelet使用Docker
cat > /etc/sysconfig/kubelet << EOF
KUBELET_EXTRA_ARGS=--cgroup-driver=systemd --container-runtime=docker
EOF

### 4. 启动kubelet
systemctl enable kubelet
systemctl start kubelet






### 5.1初始化Master节点
```
# 在master节点执行
kubeadm init \
  --apiserver-advertise-address=8.154.29.68 \   #修改ip
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.23.15 \
  --service-cidr=10.96.0.0/12 \
  --pod-network-cidr=10.244.0.0/16 \

# 初始化成功后，按照提示执行以下命令
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 保存join命令（用于worker节点加入）
# 会显示类似下面的命令，保存好：
# kubeadm join 192.168.56.10:6443 --token xxx --discovery-token-ca-cert-hash xxx
```
### 5.2 安装网络插件（Calico）

bash

```
# 在master节点执行
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# 等待所有pod运行
kubectl get pods -n kube-system -w
```
### 5.3 Worker节点加入集群
```
# 在每个worker节点执行（使用上面保存的join命令）
kubeadm join 192.168.56.10:6443 --token xxx --discovery-token-ca-cert-hash xxx

# 在master节点查看节点状态
kubectl get nodes
```
### 5.4 验证集群
```
# 查看集群信息
kubectl cluster-info

# 查看所有节点
kubectl get nodes -o wide

# 查看所有pod
kubectl get pods --all-namespaces

# 测试集群
kubectl create deployment nginx --image=nginx
kubectl get pods
```
------
## 第四阶段：GitLab CI/CD环境搭建

### 4.1 安装GitLab Runner
```bash
# 创建GitLab Runner命名空间
kubectl create namespace gitlab-runner

# 创建GitLab Runner配置文件
cat > gitlab-runner-values.yaml << EOF
## GitLab Runner Image
gitlabUrl: https://gitlab.com/  # 如果是自建GitLab，修改为你的地址
runnerRegistrationToken: "YOUR_RUNNER_TOKEN"  # 从GitLab获取

## 配置Runner执行器为Kubernetes
runners:
  config: |
    [[runners]]
      [runners.kubernetes]
        namespace = "{{.Release.Namespace}}"
        image = "centos:7.9"
  tags: "k8s-runner"
  executor: kubernetes
  
## 资源限制
resources:
  limits:
    memory: 512Mi
    cpu: 500m
  requests:
    memory: 256Mi
    cpu: 200m
EOF

# 使用Helm安装GitLab Runner
helm repo add gitlab https://charts.gitlab.io
helm install gitlab-runner -f gitlab-runner-values.yaml gitlab/gitlab-runner \
  --namespace gitlab-runner

# 验证安装
kubectl get pods -n gitlab-runner
```

### 4.2 编写CI/CD流水线示例

创建 `.gitlab-ci.yml` 文件：
```yaml
stages:
  - build
  - test
  - scan
  - deploy

variables:
  DOCKER_IMAGE: registry.example.com/myapp
  K8S_NAMESPACE: production

# 构建阶段
build-job:
  stage: build
  image: docker:20.10
  services:
    - docker:20.10-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $DOCKER_IMAGE:$CI_COMMIT_SHA .
    - docker push $DOCKER_IMAGE:$CI_COMMIT_SHA
  only:
    - main
    - develop

# 测试阶段
test-job:
  stage: test
  image: maven:3.8-openjdk-11
  script:
    - mvn clean test
    - mvn jacoco:report
  artifacts:
    when: always  # 即使任务失败也保存制品
    paths:
      - target/site/jacoco/  # 保存jacoco测试覆盖率报告
    expire_in: 1 week  # 保存jacoco测试覆盖率报告

# 安全扫描
scan-job:
  stage: scan
  image: aquasec/trivy:latest
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $DOCKER_IMAGE:$CI_COMMIT_SHA
    - trivy filesystem --exit-code 0 --severity HIGH,CRITICAL --format table ./

# 部署到开发环境
deploy-dev:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - |
      cat > deployment.yaml << EOF
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: myapp
        namespace: $K8S_NAMESPACE
      spec:
        replicas: 2
        selector:
          matchLabels:
            app: myapp
        template:
          metadata:
            labels:
              app: myapp
          spec:
            containers:
            - name: myapp
              image: $DOCKER_IMAGE:$CI_COMMIT_SHA
              ports:
              - containerPort: 8080
              resources:
                requests:
                  memory: "256Mi"
                  cpu: "250m"
                limits:
                  memory: "512Mi"
                  cpu: "500m"
      EOF
    - kubectl apply -f deployment.yaml
    - kubectl rollout status deployment/myapp -n $K8S_NAMESPACE --timeout=300s
  environment:
    name: development
  only:
    - develop
```

------

## 第五阶段：监控告警体系

### 5.1 安装Prometheus Stack


```bash
# 添加Helm仓库
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# 创建monitoring命名空间
kubectl create namespace monitoring

# 安装kube-prometheus-stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword='admin123' \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set alertmanager.enabled=true
```



```bash
# 1. 手动下载chart
wget https://github.com/prometheus-community/helm-charts/releases/download/kube-prometheus-stack-45.0.0/kube-prometheus-stack-45.0.0.tgz

# 2. 本地安装
helm install prometheus ./kube-prometheus-stack-45.0.0.tgz \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=admin123
```



### 5.2 配置业务应用监控

创建Spring Boot应用的监控配置：


```yaml
# spring-boot-service-monitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: spring-boot-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: spring-boot-app
  endpoints:
  - port: web
    path: /actuator/prometheus
    interval: 30s
```

### 5.3 配置告警规则



```yaml
# custom-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: custom-alerts
  namespace: monitoring
spec:
  groups:
  - name: application-alerts
    rules:
    - alert: HighErrorRate
      expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "High error rate on {{ $labels.instance }}"
        description: "Error rate is {{ $value }}"
    
    - alert: PodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total[5m]) * 60 * 5 > 0
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} is crash looping"
        
    - alert: HighCPUUsage
      expr: (100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)) > 80
      for: 5m
      labels:
        severity: warning
```

------

## 第六阶段：微服务部署实战

### 6.1 部署MySQL数据库



```yaml
# mysql-deployment.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
  namespace: default
data:
  my.cnf: |
    [mysqld]
    default-authentication-plugin=mysql_native_password
    character-set-server=utf8mb4
    collation-server=utf8mb4_unicode_ci
---
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: default
type: Opaque
data:
  password: MTIzNDU2  # echo -n "123456" | base64
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: default
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        - name: MYSQL_DATABASE
          value: "ecommerce"
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
        - name: mysql-config
          mountPath: /etc/mysql/conf.d
      volumes:
      - name: mysql-config
        configMap:
          name: mysql-config
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: default
spec:
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: mysql
```

### 6.2 部署Spring Boot应用

yaml

```
# spring-boot-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: default
  labels:
    app: user-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      containers:
      - name: user-service
        image: springboot/user-service:1.0.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          value: "mysql"
        - name: DB_PORT
          value: "3306"
        - name: DB_NAME
          value: "ecommerce"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: default
spec:
  selector:
    app: user-service
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: user-service-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: user.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 80
```

------

## 第七阶段：运维平台建设

### 7.1 创建运维自动化脚本



```bash
#!/bin/bash
# check-k8s-health.sh
echo "=== Kubernetes集群健康检查 ==="

# 检查节点状态
echo "1. 检查节点状态:"
kubectl get nodes

# 检查所有Pod状态
echo -e "\n2. 检查Pod状态:"
kubectl get pods --all-namespaces | grep -v Running

# 检查资源使用情况
echo -e "\n3. 检查资源使用:"
kubectl top nodes
kubectl top pods --all-namespaces

# 检查事件
echo -e "\n4. 最近错误事件:"
kubectl get events --sort-by='.lastTimestamp' | grep -i error | tail -5

# 检查服务
echo -e "\n5. 服务状态:"
kubectl get svc --all-namespaces
```

### 7.2 创建日志收集脚本

```python
#!/usr/bin/env python3
# log-collector.py
import subprocess
import json
from datetime import datetime, timedelta
import smtplib
from email.mime.text import MIMEText

def collect_error_logs():
    """收集过去1小时的错误日志"""
    cmd = "kubectl logs --since=1h --all-containers=true --all-namespaces | grep -i error"
    result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
    
    if result.stdout:
        send_alert(result.stdout)
        save_to_file(result.stdout)

def send_alert(log_content):
    """发送告警邮件"""
    msg = MIMEText(f"发现错误日志:\n\n{log_content}")
    msg['Subject'] = 'K8s集群错误日志告警'
    msg['From'] = 'monitor@example.com'
    msg['To'] = 'admin@example.com'
    
    with smtplib.SMTP('smtp.example.com', 587) as server:
        server.starttls()
        server.login('user', 'password')
        server.send_message(msg)

def save_to_file(content):
    """保存到文件"""
    timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
    filename = f"error_logs_{timestamp}.txt"
    with open(filename, 'w') as f:
        f.write(content)
    print(f"日志已保存到: {filename}")

if __name__ == "__main__":
    collect_error_logs()
```

### 7.3 创建Dashboard展示

```html
<!-- dashboard.html -->
<!DOCTYPE html>
<html>
<head>
    <title>K8s运维监控面板</title>
    <style>
        .dashboard { display: grid; grid-template-columns: repeat(3, 1fr); gap: 20px; }
        .card { background: white; padding: 20px; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
        .status { padding: 5px 10px; border-radius: 4px; }
        .healthy { background: #d4edda; color: #155724; }
        .warning { background: #fff3cd; color: #856404; }
        .error { background: #f8d7da; color: #721c24; }
    </style>
</head>
<body>
    <h1>K8s运维监控面板</h1>
    <div class="dashboard">
        <div class="card">
            <h3>集群状态</h3>
            <div id="cluster-status" class="status healthy">健康</div>
        </div>
        <div class="card">
            <h3>节点信息</h3>
            <div id="node-count">3个节点</div>
        </div>
        <div class="card">
            <h3>运行Pod</h3>
            <div id="pod-count">24个Pod</div>
        </div>
    </div>
    
    <script>
        // 可以通过API获取真实数据
        async function fetchClusterData() {
            const response = await fetch('/api/cluster-status');
            const data = await response.json();
            updateDashboard(data);
        }
        
        function updateDashboard(data) {
            document.getElementById('cluster-status').textContent = data.status;
            document.getElementById('node-count').textContent = `${data.nodes}个节点`;
            document.getElementById('pod-count').textContent = `${data.pods}个Pod`;
        }
        
        // 每30秒刷新一次
        setInterval(fetchClusterData, 30000);
    </script>
</body>
</html>
```

------
