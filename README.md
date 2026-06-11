# K8S SRE智能运维助手
## 描述：
  基于GitOps+LLM的k8s集群自愈agent，自动感知Pod崩溃告警，通过大模型决策并执行重启，实现一个简易运维自动化
## 核心特性
- ✅ 自动感知：自动检测Prometheus的PodCrashLoopBackOff告警，无需人工介入
- ✅ 智能决策：基于Ollama部署的Qwen2.5大模型，动态判断是否需要重启Pod
- ✅ GitOps全流程：ArgoCD接管所有组件部署，实现声明式配置、状态收敛，减少人工出错可能性
- ✅ 云原生架构：基于K8s 1.32+Containerd+Calico，符合生产级部署规范
- ✅ 可观测性：Prometheus+Grafana覆盖集群监控、告警、可视化全链路
## 技术栈
### 开发：
  - 语言：Golang（k8s client-go，容器化适配）
  - 大模型：Ollama（Qwen2.5，本地化部署）
### 基础设施
  - 容器编排：一个版本为 v1.32 的1主2从的k8s集群
  - 网络：Calico CNI 网络插件
  - 容器运行时：Containerd（Systemd Cgroup配置）
### 运维工具
  - 集群部署：Ansible（基于roles部署，保障幂等性）
  - GitOps：ArgoCD（应用声明式管理，自愈/自动同步）
  - 监控告警：Promethus（自定义告警规则）、Grafana（可视化）
  - 包管理：Helm（Promethus栈部署）
# 整体流程
  - 监控层：Promethus采集Pod状态，触发CrashLoopBackOff告警
  - 决策层：Go Agent调用Ollam API，基于告警上下文生成重启或忽略决策
  - 执行层：Agent通过k8s client-go调用k8s API，删除Pod实现重启
  - 部署层：ArgoCD接管Agent/Ollama/Promethus部署，保障配置一致性
# 快速开始
## 快速开始
### 前置条件
- 3台CentOS节点（192.168.30.11/12/13），已配置SSH免密登录
- Ansible 2.15+、Docker、Golang 1.21+

### 1. 部署K8s集群（Ansible）
```bash
# 克隆项目
git clone https://你的仓库/sre-agent-gitops.git
cd sre-agent-gitops/ansible-k8s-deployment

# 安装Ansible依赖集合
ansible-galaxy install -r requirements.yml

# 执行Playbook部署集群
ansible-playbook deploy-k8s.yml
```
### 2.部署核心组件
```bash
# 部署ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 部署Prometheus（Helm）
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring --create-namespace --version 57.2.0

# 部署Ollama（GitOps）
kubectl apply -f apps/ollama/ollama.yaml
```
### 3. 运行SRE Agent
```bash
# 本地运行（开发阶段）
cd sre-agent
go run main.go

# 容器化部署（生产阶段）
docker build -t sre-agent:v1.0 .
ctr -n k8s.io image import sre-agent-v1.0.tar
kubectl apply -f apps/sre-agent/deployment.yaml
```
# 可能出现的问题及解决
  #### 再使用Ansible部署集群时，出现无法连接Master API情况
  原因： inventory清单文件中Masters组名写错；节点防火墙未关闭；
  解决： 修改inventory清单并严格检查；执行 systemctl stop firewalld && systemctl disable firewalld
  #### Agent容器无法访问k8s API
  原因：容器内无~/.kube/config，没有进行集群配置
  解决：修改getKubernetesClient函数，优先使用rest.InClusterConfig()；
