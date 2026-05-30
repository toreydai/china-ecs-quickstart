# Demo12 — Service Discovery 与 Service Connect

## 实验简介

本实验使用 Cloud Map 为 ECS 服务配置私有 DNS 服务发现，部署 Backend（nginx）和 Frontend（busybox）两个服务，Frontend 通过 `backend.demo.local` DNS 名称访问 Backend，验证容器间服务发现的完整工作流程。

**实验目标：**
- 掌握 Cloud Map 私有 DNS Namespace 和 Discovery Service 的创建流程
- 理解 ECS Service `serviceRegistries` 配置使 Task IP 自动注册到 Cloud Map 的机制
- 能够通过 CloudWatch 日志验证 Frontend 使用 DNS 名称成功访问 Backend

**实验流程：**
1. 创建 Cloud Map 私有 DNS Namespace（`demo.local`）
2. 创建 Backend Cloud Map Discovery Service
3. 注册并部署 Backend ECS Service（关联 Cloud Map Service Registry）
4. 注册 Frontend Task Definition（通过 `backend.demo.local` DNS 访问）
5. 部署 Frontend ECS Service
6. 查看 Cloud Map 实例注册，验证 Frontend 日志中的服务间通信
7. 演练 Backend 下线后 Frontend 连接失败和恢复场景

**预计 AI 执行时长：** 10-13 分钟

---

## 前提条件

- **工具**：AWS CLI v2
- **权限**：ECS、Cloud Map（servicediscovery）、Logs 权限
- **前提**：Demo01 已完成；Demo02 的 ECR 仓库可复用
- **中国区注意**：Service Connect 在中国区的可用性需要实测；本 Demo 主线使用 Cloud Map DNS 发现

---

## 步骤

### 1. 设置环境变量

```bash
source /tmp/demo-ecs.env
export NAMESPACE=demo.local
export BACKEND_SERVICE=backend
export FRONTEND_SERVICE=frontend
export BACKEND_FAMILY=demo-ecs-backend
export FRONTEND_FAMILY=demo-ecs-frontend
export ECR_REPO=demo-ecs-web
export IMAGE_URI=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com.cn/${ECR_REPO}:v2
```

### 2. 开放 Task SG 内部通信

Frontend 通过 DNS 名称访问 Backend，两者都在 `TASK_SG_ID` 内，需要允许同 SG 内 TCP 80 互访；Demo01 只配置了 ALB→Task 规则。

```bash
aws ec2 authorize-security-group-ingress \
  --group-id ${TASK_SG_ID} \
  --protocol tcp \
  --port 80 \
  --source-group ${TASK_SG_ID} 2>/dev/null || true
```

**预期输出**：规则添加成功（或已存在）

### 3. 创建 Cloud Map 私有 DNS Namespace

```bash
export OPERATION_ID=$(aws servicediscovery create-private-dns-namespace \
  --name ${NAMESPACE} \
  --vpc ${VPC_ID} \
  --query 'OperationId' --output text)

echo "等待 namespace 创建..."
until [ "$(aws servicediscovery get-operation \
  --operation-id ${OPERATION_ID} \
  --query 'Operation.Status' --output text)" = "SUCCESS" ]; do
  sleep 10
done

export CLOUDMAP_NAMESPACE_ID=$(aws servicediscovery list-namespaces \
  --query "Namespaces[?Name=='${NAMESPACE}'].Id | [0]" \
  --output text)

echo "Namespace ID: ${CLOUDMAP_NAMESPACE_ID}"
```

**预期输出**：操作成功，输出 Namespace ID

### 4. 创建 Backend Service Discovery Service

```bash
export CLOUDMAP_SERVICE_ID=$(aws servicediscovery create-service \
  --name ${BACKEND_SERVICE} \
  --dns-config "NamespaceId=${CLOUDMAP_NAMESPACE_ID},DnsRecords=[{Type=A,TTL=10}],RoutingPolicy=MULTIVALUE" \
  --health-check-custom-config FailureThreshold=1 \
  --query 'Service.Id' --output text)

export CLOUDMAP_SERVICE_ARN=$(aws servicediscovery get-service \
  --id ${CLOUDMAP_SERVICE_ID} \
  --query 'Service.Arn' --output text)

echo "Cloud Map Service ARN: ${CLOUDMAP_SERVICE_ARN}"
```

**预期输出**：Cloud Map Service ARN 输出

### 5. 注册 Backend Task Definition

```bash
cat > /tmp/backend-task.json <<EOF2
{
  "family": "${BACKEND_FAMILY}",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws-cn:iam::${ACCOUNT_ID}:role/demo-ecs-task-execution-role",
  "taskRoleArn": "arn:aws-cn:iam::${ACCOUNT_ID}:role/demo-ecs-task-role",
  "containerDefinitions": [{
    "name": "backend",
    "image": "${IMAGE_URI}",
    "essential": true,
    "portMappings": [{"containerPort":80,"protocol":"tcp"}],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "${LOG_GROUP}",
        "awslogs-region": "${AWS_REGION}",
        "awslogs-stream-prefix": "backend"
      }
    }
  }]
}
EOF2

aws ecs register-task-definition --cli-input-json file:///tmp/backend-task.json
```

**预期输出**：Task definition 注册成功

### 6. 创建 Backend ECS Service（注册到 Cloud Map）

```bash
aws ecs create-service \
  --cluster ${CLUSTER_NAME} \
  --service-name ${BACKEND_SERVICE} \
  --task-definition ${BACKEND_FAMILY} \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[${PUBLIC_SUBNET_1},${PUBLIC_SUBNET_2}],securityGroups=[${TASK_SG_ID}],assignPublicIp=ENABLED}" \
  --service-registries registryArn=${CLOUDMAP_SERVICE_ARN}

aws ecs wait services-stable --cluster ${CLUSTER_NAME} --services ${BACKEND_SERVICE}
```

**预期输出**：Backend service 稳定，Cloud Map 中有实例注册

### 7. 注册 Frontend Task Definition

Frontend 通过 `backend.demo.local` DNS 名称访问 backend，结果写到 CloudWatch Logs 形成可验证的服务间调用。

```bash
cat > /tmp/frontend-task.json <<EOF2
{
  "family": "${FRONTEND_FAMILY}",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws-cn:iam::${ACCOUNT_ID}:role/demo-ecs-task-execution-role",
  "taskRoleArn": "arn:aws-cn:iam::${ACCOUNT_ID}:role/demo-ecs-task-role",
  "containerDefinitions": [{
    "name": "frontend",
    "image": "048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/busybox:1.36",
    "command": ["sh","-c","while true; do date; wget -qO- http://${BACKEND_SERVICE}.${NAMESPACE}; sleep 10; done"],
    "essential": true,
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "${LOG_GROUP}",
        "awslogs-region": "${AWS_REGION}",
        "awslogs-stream-prefix": "frontend"
      }
    }
  }]
}
EOF2

aws ecs register-task-definition --cli-input-json file:///tmp/frontend-task.json
```

### 8. 创建 Frontend ECS Service

```bash
aws ecs create-service \
  --cluster ${CLUSTER_NAME} \
  --service-name ${FRONTEND_SERVICE} \
  --task-definition ${FRONTEND_FAMILY} \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[${PUBLIC_SUBNET_1},${PUBLIC_SUBNET_2}],securityGroups=[${TASK_SG_ID}],assignPublicIp=ENABLED}"

aws ecs wait services-stable --cluster ${CLUSTER_NAME} --services ${FRONTEND_SERVICE}
```

**预期输出**：Frontend service 稳定

### 9. 验证服务发现

```bash
aws servicediscovery list-instances --service-id ${CLOUDMAP_SERVICE_ID}

sleep 30
aws logs tail ${LOG_GROUP} --log-stream-name-prefix frontend --since 5m
```

**预期输出**：Cloud Map 中有 backend 实例；frontend 日志持续出现 backend 返回的 HTML 内容

### 10. 故障验证

临时把 backend 缩到 0，观察 frontend 访问失败；再恢复。

```bash
aws ecs update-service --cluster ${CLUSTER_NAME} --service ${BACKEND_SERVICE} --desired-count 0
sleep 30
aws logs tail ${LOG_GROUP} --log-stream-name-prefix frontend --since 2m

aws ecs update-service --cluster ${CLUSTER_NAME} --service ${BACKEND_SERVICE} --desired-count 1
aws ecs wait services-stable --cluster ${CLUSTER_NAME} --services ${BACKEND_SERVICE}
```

**预期输出**：backend 停止后 frontend 日志出现连接错误；backend 恢复后 frontend 日志恢复正常

---

## 验收标准

完成本实验后，你应当能够：
- [ ] Cloud Map Namespace `demo.local` 已创建
- [ ] Backend ECS Service 状态为 ACTIVE，Cloud Map 中有实例注册
- [ ] Frontend ECS Service 状态为 ACTIVE
- [ ] Frontend 的 CloudWatch 日志持续输出 Backend 返回的 HTML 内容（包含"ECS China QuickStart"）
- [ ] Backend 下线时 Frontend 日志出现连接错误，Backend 恢复后日志恢复正常

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws servicediscovery list-namespaces --region cn-northwest-1 --query "Namespaces[?Name=='demo.local'].Name" --output text` | `demo.local` |
| 2 | `aws ecs describe-services --cluster demo-ecs --services backend --region cn-northwest-1 --query 'services[0].status' --output text` | `ACTIVE` |
| 3 | `aws ecs describe-services --cluster demo-ecs --services frontend --region cn-northwest-1 --query 'services[0].status' --output text` | `ACTIVE` |
| 4 | `aws servicediscovery list-instances --service-id $(aws servicediscovery list-services --region cn-northwest-1 --query "Services[?Name=='backend'].Id | [0]" --output text) --region cn-northwest-1 --query 'length(Instances)' --output text` | 大于 `0` |

---

## 实验总结

本实验建立了基于 Cloud Map DNS 的 ECS 微服务间通信，Task 启动后自动向 Cloud Map 注册 IP，Task 停止后自动注销，Frontend 通过 `backend.demo.local` 实现透明的服务发现。Cloud Map HealthCheckCustomConfig FailureThreshold 决定了 unhealthy 实例被注销前的检查次数，直接影响服务切换延迟。下一个 Demo（Demo13，可选）将通过 CodeDeploy 蓝绿发布实现更安全的零停机部署，适合对发布过程有严格控制要求的生产场景。

---

## 清理

```bash
source /tmp/demo-ecs.env

aws ecs update-service --cluster ${CLUSTER_NAME} --service ${FRONTEND_SERVICE} --desired-count 0
aws ecs delete-service --cluster ${CLUSTER_NAME} --service ${FRONTEND_SERVICE}

aws ecs update-service --cluster ${CLUSTER_NAME} --service ${BACKEND_SERVICE} --desired-count 0
aws ecs delete-service --cluster ${CLUSTER_NAME} --service ${BACKEND_SERVICE}

CLOUDMAP_SERVICE_ID=$(aws servicediscovery list-services \
  --region ${AWS_REGION} \
  --query "Services[?Name=='backend'].Id | [0]" --output text)
[ -n "${CLOUDMAP_SERVICE_ID}" ] && aws servicediscovery delete-service --id ${CLOUDMAP_SERVICE_ID}

CLOUDMAP_NAMESPACE_ID=$(aws servicediscovery list-namespaces \
  --query "Namespaces[?Name=='demo.local'].Id | [0]" --output text)
[ -n "${CLOUDMAP_NAMESPACE_ID}" ] && aws servicediscovery delete-namespace --id ${CLOUDMAP_NAMESPACE_ID}
```
