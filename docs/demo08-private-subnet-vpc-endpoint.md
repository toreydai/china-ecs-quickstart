# Demo08 — 私有子网与 VPC Endpoint

## 实验简介

本实验为 VPC 创建 ECR、Logs、Secrets Manager、SSM 等服务的 Interface VPC Endpoint 和 S3 Gateway Endpoint，将 ECS Service 从公有子网迁移到私有子网，实现无公网 IP 的安全运行。VPC Endpoint 是生产环境 ECS 私有化部署的必要基础设施。

**实验目标：**
- 掌握为 ECS Fargate 配置所需 Interface VPC Endpoint 的完整清单（ecr.api、ecr.dkr、logs、secretsmanager、ssmmessages、kms）
- 理解 Interface Endpoint（按小时计费）与 Gateway Endpoint（免费）的差异和适用场景
- 能够将 ECS Service 从公有子网迁移到私有子网（assignPublicIp=DISABLED）

**实验流程：**
1. 创建 VPCE 安全组（允许 Task SG 访问 443 端口）
2. 创建 6 个 Interface Endpoint（ecr.api/ecr.dkr/logs/secretsmanager/ssmmessages/kms）
3. 创建 S3 Gateway Endpoint（绑定私有路由表）
4. 等待所有 Endpoint 变为 available 状态
5. 更新 ECS Service 到私有子网（assignPublicIp=DISABLED）
6. 验证服务通过 ALB 仍可正常访问

**预计 AI 执行时长：** 10-13 分钟（含 Endpoint 创建等待时间）

---

## 前提条件

- **工具**：AWS CLI v2
- **权限**：EC2（VPC Endpoint 创建）、ECS 权限
- **前提**：Demo01-Demo06 已完成
- **中国区注意**：Interface endpoint 按小时和流量计费；私有子网 ECS Exec 需要 `ssmmessages` endpoint；实验结束后及时删除 endpoint

---

## 步骤

### 1. 设置环境变量

```bash
source /tmp/demo-ecs.env
export SERVICE_NAME=demo-ecs-web
```

### 2. 创建 Endpoint 安全组

```bash
export VPCE_SG_ID=$(aws ec2 create-security-group \
  --group-name demo-ecs-vpce-sg \
  --description "VPC endpoint security group for ECS demo" \
  --vpc-id ${VPC_ID} \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress \
  --group-id ${VPCE_SG_ID} \
  --protocol tcp \
  --port 443 \
  --source-group ${TASK_SG_ID}

echo "VPCE 安全组已创建：${VPCE_SG_ID}"

cat >> /tmp/demo-ecs.env <<EOF2
export VPCE_SG_ID=${VPCE_SG_ID}
EOF2
```

**预期输出**：安全组 ID 输出，变量已写入 `/tmp/demo-ecs.env`

### 3. 创建 VPC Interface Endpoint

ECR 拉镜像需要 `ecr.api`、`ecr.dkr`；写日志需要 `logs`；读取 Secrets 需要 `secretsmanager`；ECS Exec 需要 `ssmmessages`；使用 KMS 加密时需要 `kms`。

> ⚠️ **中国区限制**：ECR 的 Interface Endpoint 服务名前缀为 `cn.com.amazonaws`，与其他服务的 `com.amazonaws` 不同。混用前缀会导致 `InvalidServiceName` 报错。

```bash
# ECR endpoints 使用 cn.com.amazonaws 前缀
for service in ecr.api ecr.dkr; do
  aws ec2 create-vpc-endpoint \
    --vpc-id ${VPC_ID} \
    --vpc-endpoint-type Interface \
    --service-name cn.com.amazonaws.${AWS_REGION}.${service} \
    --subnet-ids ${PRIVATE_SUBNET_1} ${PRIVATE_SUBNET_2} \
    --security-group-ids ${VPCE_SG_ID} \
    --private-dns-enabled \
    --tag-specifications "ResourceType=vpc-endpoint,Tags=[{Key=Project,Value=${PROJECT_TAG}},{Key=Demo,Value=Demo08},{Key=Owner,Value=${OWNER_TAG}}]"
  echo "Created endpoint: ${service}"
done

# 其他 endpoints 使用 com.amazonaws 前缀
for service in logs secretsmanager ssmmessages kms; do
  aws ec2 create-vpc-endpoint \
    --vpc-id ${VPC_ID} \
    --vpc-endpoint-type Interface \
    --service-name com.amazonaws.${AWS_REGION}.${service} \
    --subnet-ids ${PRIVATE_SUBNET_1} ${PRIVATE_SUBNET_2} \
    --security-group-ids ${VPCE_SG_ID} \
    --private-dns-enabled \
    --tag-specifications "ResourceType=vpc-endpoint,Tags=[{Key=Project,Value=${PROJECT_TAG}},{Key=Demo,Value=Demo08},{Key=Owner,Value=${OWNER_TAG}}]"
  echo "Created endpoint: ${service}"
done
```

创建 S3 Gateway Endpoint（绑定私有路由表，无额外费用）：

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id ${VPC_ID} \
  --vpc-endpoint-type Gateway \
  --service-name com.amazonaws.${AWS_REGION}.s3 \
  --route-table-ids ${PRIVATE_RT_ID} \
  --tag-specifications "ResourceType=vpc-endpoint,Tags=[{Key=Project,Value=${PROJECT_TAG}},{Key=Demo,Value=Demo08},{Key=Owner,Value=${OWNER_TAG}}]"
```

**预期输出**：6 个 Interface Endpoint + 1 个 Gateway Endpoint 创建成功

### 4. 等待 Interface Endpoint 变为 available

```bash
until aws ec2 describe-vpc-endpoints \
  --filters Name=vpc-id,Values=${VPC_ID} Name=state,Values=available \
  --query 'length(VpcEndpoints)' --output text | grep -qE '^[1-9]'; do
  echo "waiting for endpoints..."; sleep 15
done

aws ec2 describe-vpc-endpoints \
  --filters Name=vpc-id,Values=${VPC_ID} \
  --query 'VpcEndpoints[].{Service:ServiceName,State:State}' --output table
```

**预期输出**：所有 Interface endpoint 状态为 available

### 5. 迁移 Service 到私有子网

```bash
aws ecs update-service \
  --cluster ${CLUSTER_NAME} \
  --service ${SERVICE_NAME} \
  --network-configuration "awsvpcConfiguration={subnets=[${PRIVATE_SUBNET_1},${PRIVATE_SUBNET_2}],securityGroups=[${TASK_SG_ID}],assignPublicIp=DISABLED}" \
  --force-new-deployment

aws ecs wait services-stable --cluster ${CLUSTER_NAME} --services ${SERVICE_NAME}
```

**预期输出**：Service 稳定，task 运行在私有子网，没有公网 IP

### 6. 验证

```bash
aws ecs describe-services \
  --cluster ${CLUSTER_NAME} \
  --services ${SERVICE_NAME} \
  --query 'services[0].{desired:desiredCount,running:runningCount,events:events[:3]}'

aws logs describe-log-streams \
  --log-group-name ${LOG_GROUP} \
  --order-by LastEventTime \
  --descending \
  --max-items 3

curl -m5 http://${ALB_DNS}:8080
```

**预期输出**：running=desired；日志正常；ALB 通过私有子网访问 task

---

## 验收标准

完成本实验后，你应当能够：
- [ ] VPC 中有 6 个 Interface Endpoint 状态为 available
- [ ] S3 Gateway Endpoint 已绑定到私有路由表
- [ ] ECS Service running count ≥ 2，Task 运行在私有子网（无公网 IP）
- [ ] `curl http://<ALB_DNS>:8080` 通过 ALB 仍能访问到服务

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=$(aws ec2 describe-vpcs --filters Name=tag:Name,Values=demo-ecs-vpc --region cn-northwest-1 --query 'Vpcs[0].VpcId' --output text)" "Name=vpc-endpoint-type,Values=Interface" "Name=state,Values=available" --region cn-northwest-1 --query 'length(VpcEndpoints)' --output text` | `6` |
| 2 | `aws ecs describe-services --cluster demo-ecs --services demo-ecs-web --region cn-northwest-1 --query 'services[0].runningCount' --output text` | `2` 或以上 |
| 3 | `aws ec2 describe-security-groups --group-names demo-ecs-vpce-sg --region cn-northwest-1 --query 'SecurityGroups[0].GroupName' --output text` | `demo-ecs-vpce-sg` |

---

## 实验总结

本实验完成了 ECS 私有化部署的关键基础设施配置，Fargate 容器在私有子网中运行，通过 VPC Endpoint 访问 ECR、CloudWatch、Secrets Manager 等 AWS 服务，所有流量不经过公网。Interface Endpoint 按小时计费，实验结束后及时删除是控制成本的重要习惯。`ssmmessages` Endpoint 为下一个 Demo（Demo09）的 ECS Exec 功能提供了网络基础。

---

## 清理

Interface endpoint 按小时计费，实验结束后及时删除。

```bash
source /tmp/demo-ecs.env

ENDPOINT_IDS=$(aws ec2 describe-vpc-endpoints \
  --filters Name=vpc-id,Values=${VPC_ID} \
  --query 'VpcEndpoints[].VpcEndpointId' --output text)

echo "将要删除的 Endpoint IDs："
echo "${ENDPOINT_IDS}"

# 确认后执行删除
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids ${ENDPOINT_IDS}

aws ec2 delete-security-group --group-id ${VPCE_SG_ID}
```
