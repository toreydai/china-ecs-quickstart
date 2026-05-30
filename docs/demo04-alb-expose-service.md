# Demo04 — 使用 ALB 暴露服务

## 实验简介

本实验创建应用负载均衡器（ALB）和 ip 类型目标组，将 ECS Service 注册到 ALB，实现通过公网 HTTP 访问容器服务。同时演练修改健康检查路径导致 target 变为 unhealthy 的故障场景，掌握 ALB + ECS 的联动排查方法。

**实验目标：**
- 掌握 Fargate + ALB 必须使用 `target-type ip` 的配置要求和原因
- 理解中国区 ALB 使用 8080 端口的必要性（避免 80/443 在备案场景下受限）
- 能够演练并恢复健康检查失败导致的 target unhealthy 故障

**实验流程：**
1. 创建 ALB（跨双 AZ 公有子网）和 ip 类型 Target Group
2. 创建 8080 端口 Listener 转发到 Target Group
3. 将 ECS Service 接入 ALB，设置健康检查宽限期
4. 轮询等待 target 变为 healthy
5. 通过 curl 验证公网 HTTP 访问
6. 演练健康检查失败（改路径为 `/missing`）并观察 ECS 替换 Task
7. 恢复健康检查路径，验证服务恢复

**预计 AI 执行时长：** 7-10 分钟

---

## 前提条件

- **工具**：AWS CLI v2、curl
- **权限**：ELBv2、ECS 权限
- **前提**：Demo01-Demo03 已完成
- **中国区注意**：实验 listener 使用 8080，避免未备案场景下 80/443 访问受限；Fargate + ALB target group 必须使用 `target-type ip`

---

## 步骤

### 1. 设置环境变量

```bash
source /tmp/demo-ecs.env
export SERVICE_NAME=${SERVICE_NAME:-demo-ecs-web}
export CONTAINER_NAME=${CONTAINER_NAME:-web}
export ALB_NAME=demo-ecs-alb
export TG_NAME=demo-ecs-tg
```

### 2. 创建 ALB 与 Target Group

```bash
export ALB_ARN=$(aws elbv2 create-load-balancer \
  --name ${ALB_NAME} \
  --subnets ${PUBLIC_SUBNET_1} ${PUBLIC_SUBNET_2} \
  --security-groups ${ALB_SG_ID} \
  --tags Key=Project,Value=${PROJECT_TAG} Key=Demo,Value=Demo04 Key=Owner,Value=${OWNER_TAG} \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)

export TG_ARN=$(aws elbv2 create-target-group \
  --name ${TG_NAME} \
  --protocol HTTP \
  --port 80 \
  --target-type ip \
  --vpc-id ${VPC_ID} \
  --health-check-protocol HTTP \
  --health-check-path / \
  --health-check-interval-seconds 15 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 2 \
  --matcher HttpCode=200 \
  --tags Key=Project,Value=${PROJECT_TAG} Key=Demo,Value=Demo04 Key=Owner,Value=${OWNER_TAG} \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

export LISTENER_ARN=$(aws elbv2 create-listener \
  --load-balancer-arn ${ALB_ARN} \
  --protocol HTTP \
  --port 8080 \
  --default-actions Type=forward,TargetGroupArn=${TG_ARN} \
  --query 'Listeners[0].ListenerArn' --output text)
```

**预期输出**：ALB ARN、Target Group ARN、Listener ARN 均有值

### 3. 将 Service 接入 ALB

```bash
aws ecs update-service \
  --cluster ${CLUSTER_NAME} \
  --service ${SERVICE_NAME} \
  --load-balancers targetGroupArn=${TG_ARN},containerName=${CONTAINER_NAME},containerPort=80 \
  --health-check-grace-period-seconds 60

aws ecs wait services-stable --cluster ${CLUSTER_NAME} --services ${SERVICE_NAME}
```

**预期输出**：Service 稳定，running count 等于 desired count

### 4. 轮询 Target Health

ALB 创建后 target 变为 healthy 通常需要 1-3 分钟。

```bash
for i in $(seq 1 20); do
  aws elbv2 describe-target-health \
    --target-group-arn ${TG_ARN} \
    --query 'TargetHealthDescriptions[].{target:Target.Id,port:Target.Port,state:TargetHealth.State,reason:TargetHealth.Reason}'
  HEALTHY=$(aws elbv2 describe-target-health \
    --target-group-arn ${TG_ARN} \
    --query "length(TargetHealthDescriptions[?TargetHealth.State=='healthy'])" \
    --output text)
  [ "${HEALTHY}" != "0" ] && break
  sleep 15
done
```

**预期输出**：至少一个 target state=healthy

### 5. 验证公网访问

```bash
export ALB_DNS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns ${ALB_ARN} \
  --query 'LoadBalancers[0].DNSName' --output text)

curl -m5 http://${ALB_DNS}:8080
```

保存 ALB 变量：

```bash
cat >> /tmp/demo-ecs.env <<EOF2
export ALB_ARN=${ALB_ARN}
export TG_ARN=${TG_ARN}
export LISTENER_ARN=${LISTENER_ARN}
export ALB_DNS=${ALB_DNS}
EOF2
```

**预期输出**：返回 nginx HTML 页面，包含"ECS China QuickStart"

### 6. 健康检查失败演练

将健康检查路径临时改为不存在的 `/missing`，观察 target 变为 unhealthy。

```bash
aws elbv2 modify-target-group \
  --target-group-arn ${TG_ARN} \
  --health-check-path /missing

for i in $(seq 1 12); do
  aws elbv2 describe-target-health \
    --target-group-arn ${TG_ARN} \
    --query 'TargetHealthDescriptions[].{target:Target.Id,state:TargetHealth.State,reason:TargetHealth.Reason}'
  UNHEALTHY=$(aws elbv2 describe-target-health \
    --target-group-arn ${TG_ARN} \
    --query "length(TargetHealthDescriptions[?TargetHealth.State=='unhealthy'])" \
    --output text)
  [ "${UNHEALTHY}" != "0" ] && break
  sleep 15
done

aws ecs describe-services \
  --cluster ${CLUSTER_NAME} \
  --services ${SERVICE_NAME} \
  --query 'services[0].events[:5]'
```

**预期输出**：target state 变为 unhealthy，service events 中有替换任务的记录

### 7. 恢复健康检查

```bash
aws elbv2 modify-target-group \
  --target-group-arn ${TG_ARN} \
  --health-check-path /

for i in $(seq 1 20); do
  HEALTHY=$(aws elbv2 describe-target-health \
    --target-group-arn ${TG_ARN} \
    --query "length(TargetHealthDescriptions[?TargetHealth.State=='healthy'])" \
    --output text)
  [ "${HEALTHY}" != "0" ] && break
  sleep 15
done

curl -m5 http://${ALB_DNS}:8080
```

**预期输出**：target 恢复 healthy，curl 返回正常页面

---

## 验收标准

完成本实验后，你应当能够：
- [ ] ALB `demo-ecs-alb` 状态为 active
- [ ] Target Group `demo-ecs-tg` 类型为 ip，至少 1 个 target 状态为 healthy
- [ ] Listener 监听 8080 端口并转发到 Target Group
- [ ] `curl http://<ALB_DNS>:8080` 返回包含"ECS China QuickStart"的 HTML 页面
- [ ] 演练中观察到 target 变为 unhealthy，恢复后返回 healthy

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws elbv2 describe-load-balancers --names demo-ecs-alb --region cn-northwest-1 --query 'LoadBalancers[0].State.Code' --output text` | `active` |
| 2 | `aws elbv2 describe-target-groups --names demo-ecs-tg --region cn-northwest-1 --query 'TargetGroups[0].TargetType' --output text` | `ip` |
| 3 | `aws elbv2 describe-target-health --target-group-arn $(aws elbv2 describe-target-groups --names demo-ecs-tg --region cn-northwest-1 --query 'TargetGroups[0].TargetGroupArn' --output text) --region cn-northwest-1 --query "length(TargetHealthDescriptions[?TargetHealth.State=='healthy'])" --output text` | 大于 `0` |
| 4 | `aws elbv2 describe-listeners --load-balancer-arn $(aws elbv2 describe-load-balancers --names demo-ecs-alb --region cn-northwest-1 --query 'LoadBalancers[0].LoadBalancerArn' --output text) --region cn-northwest-1 --query 'Listeners[0].Port' --output text` | `8080` |

---

## 实验总结

本实验实现了 ECS Fargate 服务通过 ALB 的公网访问，ip 类型 Target Group 是 Fargate 的必选配置（awsvpc 模式下 Task 有独立 IP，不经过主机端口）。健康检查路径配置错误是 target 变为 unhealthy 最常见的原因之一，Service Events 是排查此类问题的第一入口。下一个 Demo（Demo05）将基于此环境演示滚动发布、故障版本发布和手动回滚的完整流程。

---

## 清理

后续 Demo05-Demo10 复用 ALB，不建议立即清理。

```bash
source /tmp/demo-ecs.env
aws ecs update-service --cluster ${CLUSTER_NAME} --service ${SERVICE_NAME} --load-balancers '[]'
aws elbv2 delete-listener --listener-arn ${LISTENER_ARN}
aws elbv2 delete-load-balancer --load-balancer-arn ${ALB_ARN}
aws elbv2 delete-target-group --target-group-arn ${TG_ARN}
```
