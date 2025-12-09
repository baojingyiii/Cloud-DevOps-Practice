# Cloud DevOps Practice Platform
## 📋 项目概述
一个完整的云原生DevOps实战平台，涵盖了从Kubernetes集群搭建到CI/CD流水线、监控告警、微服务部署的全流程实践。具体操作详见manual.md
## 🎯 项目目标
- 掌握企业级K8s集群的部署与运维
- 构建完整的CI/CD自动化流水线
- 建立完善的监控告警体系
- 实践微服务在K8s上的部署与管理
## 🏗️ 技术架构
```
用户请求 → Nginx Ingress → Spring Boot微服务 → MySQL/Elasticsearch
    ↑                              ↑
监控平台(Prometheus+Grafana)      CI/CD流水线(GitLab CI+Argo CD)
```

## 🛠️ 技术栈
- **容器编排**: Kubernetes 1.25.5 + containerd
- **CI/CD**: GitLab CI + Argo CD (GitOps)
- **监控告警**: Prometheus + Grafana + Alertmanager
- **微服务**: Spring Boot + MySQL + Redis
- **自动化运维**: Ansible + Shell/Python脚本
- **云平台**: 阿里云ECS
## 📁 项目结构
（上面提到的目录结构）

## 🚀 快速开始
### 1. 环境准备
# 克隆项目
```
git clone https://github.com/baojingyiii/Cloud-DevOps-Practice.git
cd Cloud-DevOps-Practice
```
# 执行自动化安装脚本
```
chmod +x scripts/install-k8s.sh
./scripts/install-k8s.sh
```
2. 部署Kubernetes集群
详细步骤见 docs/02-k8s-cluster-deployment.md

3. 配置CI/CD流水线
详细步骤见 docs/03-gitlab-ci-cd-setup.md

📸 项目成果展示

🔧 故障排查
常见问题及解决方案

📚 学习资料
Kubernetes官方文档
GitLab CI/CD文档
Prometheus监控指南
🤝 贡献指南
欢迎提交Issue和Pull Request！

📄 许可证
MIT License

