# Demo01 — 准备实验环境与基础资源

## 实验简介

本实验为整个 ECS China QuickStart 奠定基础，在 AWS 中国区（cn-northwest-1）创建所有后续 Demo 复用的 VPC 网络、IAM 权限和 ECS Cluster。掌握中国区 ARN 前缀（`arn:aws-cn:`）和网络分层是后续实验顺利进行的关键。

**实验目标：**
- 掌握在中国区初始化 ECS Fargate 实验环境的完整流程
- 理解 Task Execution Role（拉镜像/写日志）与 Task Role（业务权限）的职责差异
- 能够独立创建 VPC、子网、安全组、ECS Cluster 和 IAM 角色

**实验流程：**
1. 配置区域环境变量并验证账号身份（ARN 以 `arn:aws-cn:` 开头）
2. 创建 VPC、公有/私有子网和 Internet Gateway
3. 配置路由表并关联子网
4. 创建 ALB 安全组和 Task 安全组
5. 创建 ECS Cluster（启用 Container Insights）和 CloudWatch 日志组
6. 创建 Task Execution Role 和 Task Role
7. 保存环境变量到 `/tmp/demo-ecs.env` 供后续 Demo 复用

**预计 AI 执行时长：** 20-25 分钟

---

## 前提条件

- **工具**：AWS CLI v2、Docker、jq；操作机具备 EC2、ECS、IAM、ECR、Logs、ELB 权限
- **区域**：cn-northwest-1（宁夏）
- **注意**：输出 ARN 必须以 `arn:aws-cn:` 开头

---

## 步骤

### 1. 设置环境变量

```bash
export AWS_PROFILE=cn          # 操作机存在多个 profile，必须显式指定
export AWS_REGION=cn-northwest-1
export AWS_DEFAULT_REGION=cn-northwest-1
export CLUSTER_NAME=demo-ecs
export PROJECT_TAG=ecs-china-quickstart
export OWNER_TAG=${USER:-ecs-lab}
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export LOG_GROUP=/ecs/demo-ecs
```

验证中国区身份（ARN 必须以 `arn:aws-cn:` 开头）：

```bash
aws sts get-caller-identity --query Arn --output text
```

**预期输出**：ARN 以 `arn:aws-cn:iam::` 开头

### 2. 验证工具

```bash
aws --version
docker --version
jq --version
session-manager-plugin --version || echo "Demo09 前需要安装 Session Manager plugin"
```

**预期输出**：各工具版本号正常输出

### 3. 创建 VPC 与子网

主线 Demo03-Demo07 使用公有子网和 `assignPublicIp=ENABLED`；Demo08 切换到私有子网和 VPC Endpoint。

```bash
export VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.42.0.0/16 \
  --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=demo-ecs-vpc},{Key=Project,Value=${PROJECT_TAG}},{Key=Demo,Value=Demo01},{Key=Owner,Value=${OWNER_TAG}}]" \
  --query 'Vpc.VpcId' --output text)

aws ec2 modify-vpc-attribute --vpc-id ${VPC_ID} --enable-dns-hostnames "{\"Value\":true}"
aws ec2 modify-vpc-attribute --vpc-id ${VPC_ID} --enable-dns-support "{\"Value\":true}"

AZ1=$(aws ec2 describe-availability-zones --query 'AvailabilityZones[0].ZoneName' --output text)
AZ2=$(aws ec2 describe-availability-zones --query 'AvailabilityZones[1].ZoneName' --output text)

export PUBLIC_SUBNET_1=$(aws ec2 create-subnet --vpc-id ${VPC_ID} --cidr-block 10.42.1.0/24 --availability-zone ${AZ1} --query 'Subnet.SubnetId' --output text)
export PUBLIC_SUBNET_2=$(aws ec2 create-subnet --vpc-id ${VPC_ID} --cidr-block 10.42.2.0/24 --availability-zone ${AZ2} --query 'Subnet.SubnetId' --output text)
export PRIVATE_SUBNET_1=$(aws ec2 create-subnet --vpc-id ${VPC_ID} --cidr-block 10.42.101.0/24 --availability-zone ${AZ1} --query 'Subnet.SubnetId' --output text)
export PRIVATE_SUBNET_2=$(aws ec2 create-subnet --vpc-id ${VPC_ID} --cidr-block 10.42.102.0/24 --availability-zone ${AZ2} --query 'Subnet.SubnetId' --output text)

# 补充 Project tag（create-subnet 不支持 --tag-specifications，需单独打标签）
aws ec2 create-tags \
  --resources ${PUBLIC_SUBNET_1} ${PUBLIC_SUBNET_2} ${PRIVATE_SUBNET_1} ${PRIVATE_SUBNET_2} \
  --tags Key=Project,Value=${PROJECT_TAG} Key=Demo,Value=Demo01 Key=Owner,Value=${OWNER_TAG}
```

创建互联网网关和路由表：

```bash
export IGW_ID=$(aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --internet-gateway-id ${IGW_ID} --vpc-id ${VPC_ID}

export PUBLIC_RT_ID=$(aws ec2 create-route-table --vpc-id ${VPC_ID} --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id ${PUBLIC_RT_ID} --destination-cidr-block 0.0.0.0/0 --gateway-id ${IGW_ID}
aws ec2 associate-route-table --route-table-id ${PUBLIC_RT_ID} --subnet-id ${PUBLIC_SUBNET_1}
aws ec2 associate-route-table --route-table-id ${PUBLIC_RT_ID} --subnet-id ${PUBLIC_SUBNET_2}

aws ec2 modify-subnet-attribute --subnet-id ${PUBLIC_SUBNET_1} --map-public-ip-on-launch
aws ec2 modify-subnet-attribute --subnet-id ${PUBLIC_SUBNET_2} --map-public-ip-on-launch

export PRIVATE_RT_ID=$(aws ec2 create-route-table --vpc-id ${VPC_ID} --query 'RouteTable.RouteTableId' --output text)
aws ec2 associate-route-table --route-table-id ${PRIVATE_RT_ID} --subnet-id ${PRIVATE_SUBNET_1}
aws ec2 associate-route-table --route-table-id ${PRIVATE_RT_ID} --subnet-id ${PRIVATE_SUBNET_2}
```

**预期输出**：各资源 ID 正常输出，无报错

### 4. 创建安全组

```bash
export ALB_SG_ID=$(aws ec2 create-security-group \
  --group-name demo-ecs-alb-sg \
  --description "ALB security group for ECS demo" \
  --vpc-id ${VPC_ID} \
  --query 'GroupId' --output text)

export TASK_SG_ID=$(aws ec2 create-security-group \
  --group-name demo-ecs-task-sg \
  --description "Task security group for ECS demo" \
  --vpc-id ${VPC_ID} \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id ${ALB_SG_ID} --protocol tcp --port 8080 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id ${TASK_SG_ID} --protocol tcp --port 80 --source-group ${ALB_SG_ID}
```

**预期输出**：两个安全组 ID，ingress 规则添加成功

### 5. 创建 ECS Cluster 与日志组

```bash
aws ecs create-cluster \
  --cluster-name ${CLUSTER_NAME} \
  --settings name=containerInsights,value=enabled \
  --tags key=Project,value=${PROJECT_TAG} key=Demo,value=Demo01 key=Owner,value=${OWNER_TAG}

aws logs create-log-group --log-group-name ${LOG_GROUP} || true
```

**预期输出**：Cluster 状态为 ACTIVE，日志组创建成功

### 6. 创建 IAM Role

Task execution role 用于 ECS Agent 拉镜像、写日志、读取 Secrets；task role 是应用容器自己的业务权限。

```bash
cat > /tmp/ecs-task-trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ecs-tasks.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name demo-ecs-task-execution-role \
  --assume-role-policy-document file:///tmp/ecs-task-trust.json || true

aws iam attach-role-policy \
  --role-name demo-ecs-task-execution-role \
  --policy-arn arn:aws-cn:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

aws iam create-role \
  --role-name demo-ecs-task-role \
  --assume-role-policy-document file:///tmp/ecs-task-trust.json || true
```

**预期输出**：两个 IAM Role 创建成功（或已存在）

### 7. 保存环境变量

```bash
cat > /tmp/demo-ecs.env <<EOF
export AWS_REGION=${AWS_REGION}
export AWS_DEFAULT_REGION=${AWS_DEFAULT_REGION}
export ACCOUNT_ID=${ACCOUNT_ID}
export CLUSTER_NAME=${CLUSTER_NAME}
export PROJECT_TAG=${PROJECT_TAG}
export OWNER_TAG=${OWNER_TAG}
export VPC_ID=${VPC_ID}
export PUBLIC_SUBNET_1=${PUBLIC_SUBNET_1}
export PUBLIC_SUBNET_2=${PUBLIC_SUBNET_2}
export PRIVATE_SUBNET_1=${PRIVATE_SUBNET_1}
export PRIVATE_SUBNET_2=${PRIVATE_SUBNET_2}
export PUBLIC_RT_ID=${PUBLIC_RT_ID}
export PRIVATE_RT_ID=${PRIVATE_RT_ID}
export ALB_SG_ID=${ALB_SG_ID}
export TASK_SG_ID=${TASK_SG_ID}
export LOG_GROUP=${LOG_GROUP}
EOF

echo "/tmp/demo-ecs.env 已保存"
```

**预期输出**：打印"/tmp/demo-ecs.env 已保存"

---

## 验收标准

完成本实验后，你应当能够：
- [ ] ECS Cluster `demo-ecs` 已创建，状态为 ACTIVE，Container Insights 已启用
- [ ] VPC 中有 4 个子网（2 公有 + 2 私有），分布于 2 个可用区
- [ ] IAM Role `demo-ecs-task-execution-role` 已附加 AmazonECSTaskExecutionRolePolicy
- [ ] IAM Role `demo-ecs-task-role` 已创建（供后续 Demo 为容器添加业务权限）
- [ ] `/tmp/demo-ecs.env` 包含所有后续 Demo 所需的环境变量

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws ecs describe-clusters --clusters demo-ecs --region cn-northwest-1 --query 'clusters[0].status' --output text` | `ACTIVE` |
| 2 | `aws ec2 describe-subnets --filters "Name=tag:Project,Values=ecs-china-quickstart" --region cn-northwest-1 --query 'length(Subnets)' --output text` | `4` |
| 3 | `aws iam get-role --role-name demo-ecs-task-execution-role --query 'Role.RoleName' --output text` | `demo-ecs-task-execution-role` |
| 4 | `aws iam get-role --role-name demo-ecs-task-role --query 'Role.RoleName' --output text` | `demo-ecs-task-role` |
| 5 | `aws logs describe-log-groups --log-group-name-prefix /ecs/demo-ecs --region cn-northwest-1 --query 'logGroups[0].logGroupName' --output text` | `/ecs/demo-ecs` |

---

## 实验总结

本实验建立了 ECS China QuickStart 的完整基础环境，包括 VPC 网络架构（公有/私有子网分层）、两个 IAM 角色（Execution Role 负责基础设施权限，Task Role 供应用容器使用）、ECS Cluster 和 CloudWatch 日志组。中国区所有 ARN 必须使用 `arn:aws-cn:` 前缀，这是与全球区最重要的差异之一。下一个 Demo（Demo02）将在此基础上创建 ECR 私有镜像仓库并推送应用镜像，为部署 Fargate 服务做准备。

---

## 清理

建议完成所有 Demo 后统一清理基础资源。若只清理 Demo01，必须先删除后续 Demo 创建的 ECS Service、ALB、Target Group、VPC Endpoint、ECR Repository。

```bash
source /tmp/demo-ecs.env

# IAM Roles
aws iam detach-role-policy --role-name demo-ecs-task-execution-role --policy-arn arn:aws-cn:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
aws iam delete-role --role-name demo-ecs-task-execution-role
aws iam delete-role --role-name demo-ecs-task-role

# ECS Cluster
aws ecs delete-cluster --cluster ${CLUSTER_NAME}

# CloudWatch Log Group
aws logs delete-log-group --log-group-name ${LOG_GROUP}

# 网络资源（在后续 Demo 均清理后执行）
aws ec2 detach-internet-gateway --internet-gateway-id ${IGW_ID} --vpc-id ${VPC_ID}
aws ec2 delete-internet-gateway --internet-gateway-id ${IGW_ID}
aws ec2 delete-subnet --subnet-id ${PUBLIC_SUBNET_1}
aws ec2 delete-subnet --subnet-id ${PUBLIC_SUBNET_2}
aws ec2 delete-subnet --subnet-id ${PRIVATE_SUBNET_1}
aws ec2 delete-subnet --subnet-id ${PRIVATE_SUBNET_2}
aws ec2 delete-route-table --route-table-id ${PUBLIC_RT_ID}
aws ec2 delete-route-table --route-table-id ${PRIVATE_RT_ID}
aws ec2 delete-security-group --group-id ${TASK_SG_ID}
aws ec2 delete-security-group --group-id ${ALB_SG_ID}
aws ec2 delete-vpc --vpc-id ${VPC_ID}
```
