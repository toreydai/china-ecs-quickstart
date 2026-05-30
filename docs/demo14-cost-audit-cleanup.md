# Demo14 — 成本治理与清理审计（可选）

## 实验简介

本实验对所有 Demo 创建的 ECS 相关 AWS 资源进行系统盘点，识别 ALB、VPC Endpoint、EFS 等持续计费项，设置日志保留策略，并生成按依赖顺序排列的清理检查清单，帮助完整清理实验环境、避免意外计费。

**实验目标：**
- 掌握通过 Resource Groups Tagging API 基于 Tag 查询所有实验资源的方法
- 理解 ALB（按小时）、Interface VPC Endpoint（按小时）、NAT Gateway（按小时）等持续计费资源的识别方法
- 能够按照依赖顺序安全删除所有资源，避免因依赖关系导致删除失败

**实验流程：**
1. 盘点 ECS Cluster、Service 和 Task 运行状态
2. 盘点 ALB、Target Group、VPC Endpoint 等持续计费网络资源
3. 盘点 CloudWatch 日志和 ECR 镜像存储成本
4. 设置日志保留期为 7 天
5. 通过 Tag 查询所有 `Project=ecs-china-quickstart` 资源
6. 生成完整清理检查清单和按依赖顺序的清理命令

**预计 AI 执行时长：** 10-13 分钟

---

## 前提条件

- **工具**：AWS CLI v2
- **权限**：ECS、ELBv2、EC2、ECR、Logs、Resource Groups Tagging 只读权限
- **前提**：至少完成 Demo01-Demo04；完成 Demo08-Demo13 后本 Demo 更有价值
- **预计 AI 执行时长**：10-13 分钟

---

## 步骤

### 1. 设置环境变量

```bash
source /tmp/demo-ecs.env
export COST_TAG_KEY=Project
export COST_TAG_VALUE=ecs-china-quickstart
```

### 2. ECS 资源盘点

```bash
echo "=== ECS Cluster ==="
aws ecs describe-clusters \
  --clusters ${CLUSTER_NAME} \
  --include SETTINGS CONFIGURATIONS STATISTICS \
  --region ${AWS_REGION}

echo ""
echo "=== ECS Services ==="
aws ecs list-services --cluster ${CLUSTER_NAME} --region ${AWS_REGION}

echo ""
echo "=== ECS Tasks (RUNNING) ==="
aws ecs list-tasks --cluster ${CLUSTER_NAME} --region ${AWS_REGION}

echo ""
echo "=== Task Definitions ==="
aws ecs list-task-definitions --family-prefix demo-ecs --sort DESC --region ${AWS_REGION}
```

查看所有 service 的 desired/running 状态：

```bash
SERVICES=$(aws ecs list-services --cluster ${CLUSTER_NAME} --query 'serviceArns' --output text)
[ -n "${SERVICES}" ] && aws ecs describe-services \
  --cluster ${CLUSTER_NAME} \
  --services ${SERVICES} \
  --query 'services[].{name:serviceName,desired:desiredCount,running:runningCount,status:status}'
```

**预期输出**：所有运行中的 ECS 资源列表

### 3. 网络与入口成本盘点

ALB、NAT Gateway、VPC Endpoint 都可能按小时计费。

```bash
echo "=== ALB（持续计费）==="
aws elbv2 describe-load-balancers \
  --query 'LoadBalancers[?contains(LoadBalancerName, `demo-ecs`)].{name:LoadBalancerName,dns:DNSName,state:State.Code,type:Type}' \
  --region ${AWS_REGION}

echo ""
echo "=== Target Groups ==="
aws elbv2 describe-target-groups \
  --query 'TargetGroups[?contains(TargetGroupName, `demo-ecs`)].{name:TargetGroupName,port:Port,targetType:TargetType}' \
  --region ${AWS_REGION}

echo ""
echo "=== VPC Endpoints（Interface 按小时计费）==="
aws ec2 describe-vpc-endpoints \
  --filters Name=vpc-id,Values=${VPC_ID} \
  --query 'VpcEndpoints[].{id:VpcEndpointId,type:VpcEndpointType,service:ServiceName,state:State}' \
  --region ${AWS_REGION}

echo ""
echo "=== NAT Gateway ==="
aws ec2 describe-nat-gateways \
  --filter Name=vpc-id,Values=${VPC_ID} \
  --query 'NatGateways[].{id:NatGatewayId,state:State,subnet:SubnetId}' \
  --region ${AWS_REGION}
```

**预期输出**：列出所有计费网络资源

### 4. 日志与镜像成本盘点

```bash
echo "=== CloudWatch Logs ==="
aws logs describe-log-groups \
  --log-group-name-prefix /ecs/demo-ecs \
  --query 'logGroups[].{name:logGroupName,storedBytes:storedBytes,retention:retentionInDays}' \
  --region ${AWS_REGION}

echo ""
echo "=== ECR Repositories ==="
aws ecr describe-repositories \
  --query 'repositories[?contains(repositoryName, `demo-ecs`)].{name:repositoryName,uri:repositoryUri}' \
  --region ${AWS_REGION}

echo ""
echo "=== ECR Images ==="
aws ecr describe-images \
  --repository-name demo-ecs-web \
  --query 'imageDetails[].{tags:imageTags,size:imageSizeInBytes,pushed:imagePushedAt}' \
  --region ${AWS_REGION} || true
```

**预期输出**：日志组存储量和保留期；ECR 镜像列表

### 5. 设置日志保留期

默认 CloudWatch Logs 永久保留，实验环境建议设置 7 天。

```bash
aws logs put-retention-policy \
  --log-group-name ${LOG_GROUP} \
  --retention-in-days 7 \
  --region ${AWS_REGION}

echo "日志保留期已设置为 7 天"
```

**预期输出**：打印"日志保留期已设置为 7 天"

### 6. 基于 Tag 查询资源

```bash
aws resourcegroupstaggingapi get-resources \
  --tag-filters Key=${COST_TAG_KEY},Values=${COST_TAG_VALUE} \
  --region ${AWS_REGION} \
  --query 'ResourceTagMappingList[].ResourceARN'
```

**预期输出**：所有带 `Project=ecs-china-quickstart` 标签的资源 ARN 列表

### 7. 生成清理检查清单

```bash
cat > /tmp/demo-ecs-cleanup-checklist.txt <<EOF2
==========================================
  ECS China QuickStart 清理检查清单
==========================================

[ ] ECS services deleted: $(aws ecs list-services --cluster ${CLUSTER_NAME} --query 'length(serviceArns)' --output text 2>/dev/null || echo unknown)
[ ] ECS tasks stopped: $(aws ecs list-tasks --cluster ${CLUSTER_NAME} --query 'length(taskArns)' --output text 2>/dev/null || echo unknown)
[ ] ALB deleted: check demo-ecs load balancers
[ ] Target groups deleted: check demo-ecs target groups
[ ] VPC endpoints deleted: check ${VPC_ID}
[ ] EFS deleted: check demo-ecs-efs
[ ] ECR repository deleted or lifecycle policy active
[ ] CloudWatch retention set or log group deleted
[ ] IAM demo roles removed

清理顺序建议（按依赖顺序）：
  1. CodePipeline / CodeBuild / CodeCommit / artifact S3 bucket
  2. ECS services（desired=0 再 delete）
  3. ECS standalone tasks（stop task）
  4. CodeDeploy deployment group / application
  5. ALB listener、ALB、target group
  6. VPC Endpoint、NAT Gateway
  7. EFS mount target、EFS file system
  8. ECR repository
  9. CloudWatch log group
  10. ECS cluster
  11. IAM inline policy、role
  12. Security group、route table、subnet、internet gateway、VPC
==========================================
EOF2

cat /tmp/demo-ecs-cleanup-checklist.txt
```

**预期输出**：打印清理检查清单，输出到 `/tmp/demo-ecs-cleanup-checklist.txt`

### 8. 清理入口命令（逐条确认后执行）

```bash
# 查看所有 ECS services
aws ecs list-services --cluster ${CLUSTER_NAME} --region ${AWS_REGION}

# 查看 ALB / target groups
aws elbv2 describe-load-balancers \
  --query 'LoadBalancers[?contains(LoadBalancerName, `demo-ecs`)].LoadBalancerArn' \
  --region ${AWS_REGION}
aws elbv2 describe-target-groups \
  --query 'TargetGroups[?contains(TargetGroupName, `demo-ecs`)].TargetGroupArn' \
  --region ${AWS_REGION}

# 查看 VPC endpoints
aws ec2 describe-vpc-endpoints \
  --filters Name=vpc-id,Values=${VPC_ID} \
  --query 'VpcEndpoints[].VpcEndpointId' \
  --region ${AWS_REGION}
```

---

## 验收标准

完成本实验后，你应当能够：
- [ ] CloudWatch 日志组 `/ecs/demo-ecs` 保留期已设置为 7 天
- [ ] 已识别所有 Demo 创建的持续计费资源（ALB、VPC Endpoint 等）
- [ ] `/tmp/demo-ecs-cleanup-checklist.txt` 清理检查清单已生成
- [ ] 了解清理资源的正确依赖顺序（先 Pipeline，最后 VPC）

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws logs describe-log-groups --log-group-name-prefix /ecs/demo-ecs --region cn-northwest-1 --query 'logGroups[0].retentionInDays' --output text` | `7` |
| 2 | `aws ecs list-services --cluster demo-ecs --region cn-northwest-1 --query 'length(serviceArns)' --output text` | 数字（了解当前 service 数量） |
| 3 | `aws ecr describe-repositories --region cn-northwest-1 --query "repositories[?repositoryName=='demo-ecs-web'].repositoryName" --output text` | `demo-ecs-web` |

---

## 实验总结

本实验完成了 ECS China QuickStart 的收尾工作，系统盘点了从基础网络到 CI/CD 流水线的全部实验资源。清理顺序遵循"先上层应用后下层基础设施"的原则：Pipeline → ECS Service → ALB → VPC Endpoint → EFS → ECR → 日志组 → ECS Cluster → IAM → VPC。通过本系列 14 个 Demo，你已掌握了 AWS 中国区 ECS Fargate 从零部署到生产化运维的核心技能，包括镜像管理、服务发布、Auto Scaling、安全加固、CI/CD 流水线和蓝绿发布。

---

## 清理

本 Demo 主要是审计和报告，无独立 AWS 资源需清理（除日志保留期已设置外）。

实验结束后，按步骤 8 的清理顺序逐步执行各 Demo 的清理章节命令，最后清理 Demo01 的 VPC 基础资源。
