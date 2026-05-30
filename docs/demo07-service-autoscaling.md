# Demo07 — Service Auto Scaling

## 实验简介

本实验为 ECS Service 配置 Application Auto Scaling，以 CPU 利用率 50% 为目标值自动调整副本数量（min=1，max=4），并通过压测 Fargate task 触发扩展，观察 scaling activities 记录。可选配置 Fargate Spot 混合策略以降低非关键工作负载成本。

**实验目标：**
- 掌握 Application Auto Scaling 注册 ECS 可伸缩目标和 Target Tracking 策略的完整配置
- 理解 ScaleOutCooldown 和 ScaleInCooldown 对伸缩稳定性的影响
- 能够通过压测 task 触发 scale out 并观察 scaling activities 验证效果

**实验流程：**
1. 注册 ECS Service 为可伸缩目标（min=1, max=4）
2. 创建 CPU 50% Target Tracking 策略
3. 创建压测 Fargate task（持续向 ALB 发送请求）
4. 等待约 3-5 分钟观察 desired count 增加
5. 停止压测，等待 scale in（约 2-4 分钟）
6. 可选：配置 Fargate Spot 混合容量策略

**预计 AI 执行时长：** 10-13 分钟（含等待伸缩触发时间）

---

## 前提条件

- **工具**：AWS CLI v2
- **权限**：ECS、Application Auto Scaling、CloudWatch 权限
- **前提**：Demo01-Demo04 已完成

---

## 步骤

### 1. 设置环境变量

```bash
source /tmp/demo-ecs.env
export SERVICE_NAME=demo-ecs-web
export RESOURCE_ID=service/${CLUSTER_NAME}/${SERVICE_NAME}
```

### 2. 注册可伸缩目标

```bash
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id ${RESOURCE_ID} \
  --min-capacity 1 \
  --max-capacity 4
```

**预期输出**：注册成功（无输出或返回 ResourceARN）

### 3. 创建 Target Tracking 策略

当 CPU 利用率超过 50% 时自动扩展，降到低于 50% 持续一段时间后收缩。

```bash
cat > /tmp/demo-ecs-scaling-policy.json <<'JSON'
{
  "TargetValue": 50.0,
  "PredefinedMetricSpecification": {
    "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
  },
  "ScaleOutCooldown": 60,
  "ScaleInCooldown": 120
}
JSON

aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id ${RESOURCE_ID} \
  --policy-name demo-ecs-cpu-50 \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration file:///tmp/demo-ecs-scaling-policy.json
```

**预期输出**：策略 ARN 输出

### 4. 触发负载（压测）

用临时 Fargate task 持续访问 ALB，观察 service desired count 是否被拉高。

```bash
export ALB_DNS=$(aws elbv2 describe-load-balancers \
  --names demo-ecs-alb \
  --query 'LoadBalancers[0].DNSName' --output text)

cat > /tmp/load-task.json <<EOF2
{
  "family": "demo-ecs-load",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws-cn:iam::${ACCOUNT_ID}:role/demo-ecs-task-execution-role",
  "containerDefinitions": [{
    "name": "load",
    "image": "048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/busybox:1.36",
    "command": ["sh","-c","while true; do wget -qO- http://${ALB_DNS}:8080 >/dev/null; done"],
    "essential": true
  }]
}
EOF2

aws ecs register-task-definition --cli-input-json file:///tmp/load-task.json

export LOAD_TASK_ARN=$(aws ecs run-task \
  --cluster ${CLUSTER_NAME} \
  --launch-type FARGATE \
  --task-definition demo-ecs-load \
  --network-configuration "awsvpcConfiguration={subnets=[${PUBLIC_SUBNET_1}],securityGroups=[${TASK_SG_ID}],assignPublicIp=ENABLED}" \
  --query 'tasks[0].taskArn' --output text)
```

**预期输出**：压测 task 启动，持续发送请求

### 5. 观察扩缩容

等待约 3-5 分钟后查看 desired count 变化：

```bash
aws ecs describe-services \
  --cluster ${CLUSTER_NAME} \
  --services ${SERVICE_NAME} \
  --query 'services[0].{desired:desiredCount,running:runningCount}'

aws application-autoscaling describe-scaling-activities \
  --service-namespace ecs \
  --resource-id ${RESOURCE_ID} \
  --scalable-dimension ecs:service:DesiredCount \
  --max-results 10
```

停止压测任务后等待 scale in（约 2-4 分钟后）：

```bash
aws ecs stop-task --cluster ${CLUSTER_NAME} --task ${LOAD_TASK_ARN} --reason "demo complete"
```

**预期输出**：scaling activities 中有 ScaleOut 记录；停止压测后有 ScaleIn 记录

### 6. 可选：Fargate Spot 容量策略

```bash
aws ecs put-cluster-capacity-providers \
  --cluster ${CLUSTER_NAME} \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy capacityProvider=FARGATE,weight=1

aws ecs update-service \
  --cluster ${CLUSTER_NAME} \
  --service ${SERVICE_NAME} \
  --capacity-provider-strategy capacityProvider=FARGATE,base=1,weight=1 capacityProvider=FARGATE_SPOT,weight=2 \
  --force-new-deployment

aws ecs wait services-stable --cluster ${CLUSTER_NAME} --services ${SERVICE_NAME}

aws ecs describe-services \
  --cluster ${CLUSTER_NAME} \
  --services ${SERVICE_NAME} \
  --query 'services[0].capacityProviderStrategy'
```

恢复纯 FARGATE：

```bash
aws ecs update-service \
  --cluster ${CLUSTER_NAME} \
  --service ${SERVICE_NAME} \
  --capacity-provider-strategy capacityProvider=FARGATE,weight=1 \
  --force-new-deployment
```

**预期输出**：capacityProviderStrategy 显示 FARGATE 和 FARGATE_SPOT 混合策略

---

## 验收标准

完成本实验后，你应当能够：
- [ ] ECS Service 已注册为可伸缩目标，min=1，max=4
- [ ] `demo-ecs-cpu-50` Target Tracking 策略已创建
- [ ] Scaling activities 中有 ScaleOut 记录（压测期间）
- [ ] 停止压测后 Service 可自动 scale in

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws application-autoscaling describe-scalable-targets --service-namespace ecs --region cn-northwest-1 --query "ScalableTargets[?ResourceId=='service/demo-ecs/demo-ecs-web'].{min:MinCapacity,max:MaxCapacity}" --output text` | `1	4` |
| 2 | `aws application-autoscaling describe-scaling-policies --service-namespace ecs --resource-id service/demo-ecs/demo-ecs-web --region cn-northwest-1 --query 'ScalingPolicies[0].PolicyName' --output text` | `demo-ecs-cpu-50` |

---

## 实验总结

本实验为 ECS Service 添加了 CPU 驱动的弹性伸缩能力，Target Tracking 策略根据指标自动管理副本数，无需人工干预。ScaleOutCooldown（60s）设置较短使扩展响应迅速，ScaleInCooldown（120s）设置较长避免频繁收缩。Fargate Spot 可在可接受中断的工作负载上节省约 70% 成本，适合批处理、压测和非关键后台任务。下一个 Demo（Demo08）将把服务迁移到私有子网，通过 VPC Endpoint 实现完全隔离的安全网络架构。

---

## 清理

```bash
source /tmp/demo-ecs.env

aws application-autoscaling delete-scaling-policy \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id ${RESOURCE_ID} \
  --policy-name demo-ecs-cpu-50

aws application-autoscaling deregister-scalable-target \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id ${RESOURCE_ID}

aws ecs deregister-task-definition \
  --task-definition demo-ecs-load:1 2>/dev/null || true
```
