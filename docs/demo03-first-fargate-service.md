# Demo03 — 部署第一个 Fargate 服务

## 实验简介

本实验将 ECR 镜像部署为 ECS Fargate Service，通过 one-off task 预验证镜像拉取和日志写入，再创建带 deployment circuit breaker 的服务并扩展到 2 副本。掌握 one-off task 预验证方法可将"镜像/权限/日志"问题与"服务调度"问题分开排查。

**实验目标：**
- 掌握 Fargate Task Definition（awsvpc 模式、健康检查、awslogs 日志驱动）的注册流程
- 理解 one-off task 预验证在服务创建前的诊断价值
- 能够创建带 deployment circuit breaker 的 ECS Service 并验证多副本分布

**实验流程：**
1. 注册 Task Definition（Fargate、awsvpc、256CPU/512MEM）
2. 运行 one-off smoke task 验证镜像拉取和健康检查
3. 创建 ECS Service（desired=1，启用 deployment circuit breaker）
4. 等待 Service 稳定，查看 Event 和 Task 状态
5. 查看 CloudWatch 日志确认 nginx 正常启动
6. 扩展到 2 副本，验证跨可用区分布

**预计 AI 执行时长：** 7-10 分钟

---

## 前提条件

- **工具**：AWS CLI v2
- **权限**：ECS、EC2、IAM、Logs 权限
- **前提**：Demo01、Demo02 已完成；主线此阶段使用公有子网和 `assignPublicIp=ENABLED`

---

## 步骤

### 1. 设置环境变量

```bash
source /tmp/demo-ecs.env
export SERVICE_NAME=demo-ecs-web
export TASK_FAMILY=demo-ecs-web
export CONTAINER_NAME=web
export IMAGE_URI=${IMAGE_URI_STABLE:-${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com.cn/${ECR_REPO}:stable}
```

### 2. 注册 Task Definition

Task definition 使用 Fargate、`awsvpc` 网络模式、`awslogs` 日志驱动，并配置容器健康检查。

```bash
cat > /tmp/demo-ecs-task.json <<EOF2
{
  "family": "${TASK_FAMILY}",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws-cn:iam::${ACCOUNT_ID}:role/demo-ecs-task-execution-role",
  "taskRoleArn": "arn:aws-cn:iam::${ACCOUNT_ID}:role/demo-ecs-task-role",
  "containerDefinitions": [{
    "name": "${CONTAINER_NAME}",
    "image": "${IMAGE_URI}",
    "essential": true,
    "portMappings": [{ "containerPort": 80, "protocol": "tcp" }],
    "healthCheck": {
      "command": ["CMD-SHELL", "wget -qO- http://127.0.0.1/ >/dev/null || exit 1"],
      "interval": 30,
      "timeout": 5,
      "retries": 3,
      "startPeriod": 10
    },
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "${LOG_GROUP}",
        "awslogs-region": "${AWS_REGION}",
        "awslogs-stream-prefix": "web"
      }
    }
  }]
}
EOF2

aws ecs register-task-definition \
  --cli-input-json file:///tmp/demo-ecs-task.json \
  --tags key=Project,value=${PROJECT_TAG} key=Demo,value=Demo03 key=Owner,value=${OWNER_TAG}
```

**预期输出**：Task definition 注册成功，revision 为 1

### 3. One-off Task 预验证

先用 `run-task` 验证 task definition 能拉镜像、启动、写日志，再创建 service。可以把"镜像/权限/日志"问题和 service 调度问题分开。

```bash
export SMOKE_TASK_ARN=$(aws ecs run-task \
  --cluster ${CLUSTER_NAME} \
  --launch-type FARGATE \
  --task-definition ${TASK_FAMILY} \
  --network-configuration "awsvpcConfiguration={subnets=[${PUBLIC_SUBNET_1}],securityGroups=[${TASK_SG_ID}],assignPublicIp=ENABLED}" \
  --tags key=Project,value=${PROJECT_TAG} key=Demo,value=Demo03 key=Owner,value=${OWNER_TAG} \
  --query 'tasks[0].taskArn' --output text)

aws ecs wait tasks-running --cluster ${CLUSTER_NAME} --tasks ${SMOKE_TASK_ARN}

aws ecs describe-tasks \
  --cluster ${CLUSTER_NAME} \
  --tasks ${SMOKE_TASK_ARN} \
  --query 'tasks[0].{lastStatus:lastStatus,health:healthStatus,containers:containers[].{name:name,lastStatus:lastStatus,health:healthStatus}}'
```

查看 task ENI 和公网 IP：

```bash
export ENI_ID=$(aws ecs describe-tasks \
  --cluster ${CLUSTER_NAME} \
  --tasks ${SMOKE_TASK_ARN} \
  --query 'tasks[0].attachments[0].details[?name==`networkInterfaceId`].value | [0]' \
  --output text)

aws ec2 describe-network-interfaces \
  --network-interface-ids ${ENI_ID} \
  --query 'NetworkInterfaces[0].{privateIp:PrivateIpAddress,publicIp:Association.PublicIp,subnet:SubnetId}'
```

停止 smoke task：

```bash
aws ecs stop-task --cluster ${CLUSTER_NAME} --task ${SMOKE_TASK_ARN} --reason "smoke test complete"
```

**预期输出**：task lastStatus 为 RUNNING，health 为 HEALTHY，ENI 有公网 IP

### 4. 创建 ECS Service

```bash
aws ecs create-service \
  --cluster ${CLUSTER_NAME} \
  --service-name ${SERVICE_NAME} \
  --task-definition ${TASK_FAMILY} \
  --desired-count 1 \
  --launch-type FARGATE \
  --deployment-configuration '{"minimumHealthyPercent":100,"maximumPercent":200,"deploymentCircuitBreaker":{"enable":true,"rollback":true}}' \
  --network-configuration "awsvpcConfiguration={subnets=[${PUBLIC_SUBNET_1},${PUBLIC_SUBNET_2}],securityGroups=[${TASK_SG_ID}],assignPublicIp=ENABLED}" \
  --tags key=Project,value=${PROJECT_TAG} key=Demo,value=Demo03 key=Owner,value=${OWNER_TAG}

aws ecs wait services-stable --cluster ${CLUSTER_NAME} --services ${SERVICE_NAME}
```

**预期输出**：Service 变为 ACTIVE，running count 等于 desired count

### 5. 观察 Service Event 和 Task

```bash
aws ecs describe-services \
  --cluster ${CLUSTER_NAME} \
  --services ${SERVICE_NAME} \
  --query 'services[0].{status:status,desired:desiredCount,running:runningCount,taskDefinition:taskDefinition,events:events[:5]}'

TASKS=$(aws ecs list-tasks --cluster ${CLUSTER_NAME} --service-name ${SERVICE_NAME} --query 'taskArns' --output text)
aws ecs describe-tasks \
  --cluster ${CLUSTER_NAME} \
  --tasks ${TASKS} \
  --query 'tasks[].{task:taskArn,last:lastStatus,health:healthStatus,az:availabilityZone}'
```

**预期输出**：service status=ACTIVE，running=desired；task lastStatus=RUNNING

### 6. 查看 CloudWatch Logs

```bash
aws logs describe-log-streams \
  --log-group-name ${LOG_GROUP} \
  --log-stream-name-prefix web \
  --order-by LastEventTime \
  --descending \
  --max-items 5

aws logs tail ${LOG_GROUP} --log-stream-name-prefix web --since 10m
```

**预期输出**：有 log stream，日志显示 nginx 启动信息

### 7. 扩展到 2 个副本

```bash
aws ecs update-service \
  --cluster ${CLUSTER_NAME} \
  --service ${SERVICE_NAME} \
  --desired-count 2

aws ecs wait services-stable --cluster ${CLUSTER_NAME} --services ${SERVICE_NAME}

TASKS=$(aws ecs list-tasks --cluster ${CLUSTER_NAME} --service-name ${SERVICE_NAME} --query 'taskArns' --output text)
aws ecs describe-tasks \
  --cluster ${CLUSTER_NAME} \
  --tasks ${TASKS} \
  --query 'tasks[].{task:taskArn,last:lastStatus,health:healthStatus,az:availabilityZone}'
```

**预期输出**：running count=2，两个 task 分布在不同 AZ

### 8. 保存 Service 变量

```bash
cat >> /tmp/demo-ecs.env <<EOF2
export SERVICE_NAME=${SERVICE_NAME}
export TASK_FAMILY=${TASK_FAMILY}
export CONTAINER_NAME=${CONTAINER_NAME}
EOF2
```

---

## 验收标准

完成本实验后，你应当能够：
- [ ] ECS Service `demo-ecs-web` 状态为 ACTIVE，running count = 2
- [ ] Task Definition `demo-ecs-web` 已注册，健康检查配置正确
- [ ] CloudWatch 日志组 `/ecs/demo-ecs` 中有 `web` 前缀的日志流，包含 nginx 启动日志
- [ ] 两个 Task 分布于不同可用区

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws ecs describe-services --cluster demo-ecs --services demo-ecs-web --region cn-northwest-1 --query 'services[0].status' --output text` | `ACTIVE` |
| 2 | `aws ecs describe-services --cluster demo-ecs --services demo-ecs-web --region cn-northwest-1 --query 'services[0].runningCount' --output text` | `2` |
| 3 | `aws ecs list-task-definitions --family-prefix demo-ecs-web --region cn-northwest-1 --query 'length(taskDefinitionArns)' --output text` | `1` 或以上 |
| 4 | `aws logs describe-log-streams --log-group-name /ecs/demo-ecs --log-stream-name-prefix web --region cn-northwest-1 --query 'length(logStreams)' --output text` | 大于 `0` |

---

## 实验总结

本实验部署了运行 2 副本的 Fargate 服务，掌握了 one-off task 预验证方法（先单次运行验证基础能力，再创建长期服务）和 deployment circuit breaker（发布失败自动回滚）两个重要运维实践。awsvpc 网络模式使每个 Task 获得独立 ENI 和私有 IP，是 Fargate 的默认网络模式。下一个 Demo（Demo04）将创建 ALB 将服务暴露到公网，并演练健康检查失败排查流程。

---

## 清理

后续 Demo04-Demo10 复用该 Service，不建议立即清理。

```bash
aws ecs update-service --cluster demo-ecs --service demo-ecs-web --desired-count 0
aws ecs delete-service --cluster demo-ecs --service demo-ecs-web --region cn-northwest-1
```
