# Demo05 — 滚动发布、回滚与故障排查

## 实验简介

本实验演示 ECS 滚动发布的完整生命周期：发布 v2 镜像、模拟使用不存在镜像 tag 的故障发布、通过 stopped task 排查 `CannotPullContainerError`，最后手动回滚到已知健康的 revision。掌握这个排查和回滚闭环是生产环境 ECS 运维的核心技能。

**实验目标：**
- 掌握通过注册新 task definition revision 并更新 service 完成滚动发布的流程
- 理解 deployment circuit breaker 在发布失败时的自动回滚机制
- 能够通过 describe-tasks 和 stopped task 定位 CannotPullContainerError 等发布失败原因

**实验流程：**
1. 基于当前 task definition 创建 v2 版本 revision 并滚动发布
2. 观察 deployments 列表中的发布状态
3. 注册使用不存在镜像 tag 的故障 task definition 并触发发布
4. 查看 service events 和 stopped task 排查失败原因
5. 手动更新 service 回滚到 revision 2，验证 ALB 恢复正常

**预计 AI 执行时长：** 7-10 分钟

---

## 前提条件

- **工具**：AWS CLI v2、jq
- **权限**：ECS 权限
- **前提**：Demo01-Demo04 已完成

---

## 步骤

### 1. 设置环境变量

```bash
source /tmp/demo-ecs.env
export ECR_REPO=demo-ecs-web
export SERVICE_NAME=demo-ecs-web
export TASK_FAMILY=demo-ecs-web
export IMAGE_V2=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com.cn/${ECR_REPO}:v2
```

### 2. 发布 v2

基于当前 task definition 创建新 revision，只更换镜像 URI。

```bash
aws ecs describe-task-definition \
  --task-definition ${TASK_FAMILY} \
  --query 'taskDefinition' > /tmp/taskdef-current.json

jq --arg IMAGE "${IMAGE_V2}" '
  del(.taskDefinitionArn,.revision,.status,.requiresAttributes,.compatibilities,.registeredAt,.registeredBy)
  | .containerDefinitions[0].image = $IMAGE
' /tmp/taskdef-current.json > /tmp/taskdef-v2.json

aws ecs register-task-definition --cli-input-json file:///tmp/taskdef-v2.json

aws ecs update-service \
  --cluster ${CLUSTER_NAME} \
  --service ${SERVICE_NAME} \
  --task-definition ${TASK_FAMILY} \
  --deployment-configuration '{"minimumHealthyPercent":100,"maximumPercent":200}'

aws ecs wait services-stable --cluster ${CLUSTER_NAME} --services ${SERVICE_NAME}
```

**预期输出**：Service 稳定，新 task 使用 v2 镜像，ALB 可访问

### 3. 观察发布过程

```bash
aws ecs describe-services \
  --cluster ${CLUSTER_NAME} \
  --services ${SERVICE_NAME} \
  --query 'services[0].deployments'
```

**预期输出**：deployments 列表中有一个 PRIMARY 状态 deployment

### 4. 发布故障版本

使用不存在的镜像 tag 模拟发布失败。

```bash
export BAD_IMAGE=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com.cn/${ECR_REPO}:missing

jq --arg IMAGE "${BAD_IMAGE}" '
  del(.taskDefinitionArn,.revision,.status,.requiresAttributes,.compatibilities,.registeredAt,.registeredBy)
  | .containerDefinitions[0].image = $IMAGE
' /tmp/taskdef-v2.json > /tmp/taskdef-bad.json
aws ecs register-task-definition --cli-input-json file:///tmp/taskdef-bad.json

aws ecs update-service \
  --cluster ${CLUSTER_NAME} \
  --service ${SERVICE_NAME} \
  --task-definition ${TASK_FAMILY}
```

排查 service events 和 stopped task：

```bash
sleep 30

aws ecs describe-services \
  --cluster ${CLUSTER_NAME} \
  --services ${SERVICE_NAME} \
  --query 'services[0].events[:10]'

STOPPED_TASK=$(aws ecs list-tasks --cluster ${CLUSTER_NAME} --desired-status STOPPED --query 'taskArns[0]' --output text)
[ -n "${STOPPED_TASK}" ] && aws ecs describe-tasks \
  --cluster ${CLUSTER_NAME} \
  --tasks ${STOPPED_TASK} \
  --query 'tasks[0].{lastStatus:lastStatus,stopCode:stopCode,stoppedReason:stoppedReason,containers:containers[].{name:name,reason:reason,exitCode:exitCode}}'
```

**预期输出**：service events 显示任务启动失败，stopped task 有 CannotPullContainerError 或 ImagePullBackOff 信息

### 5. 回滚

将 service 更新回已知健康的 task definition revision（v2 为 revision 2）：

```bash
aws ecs update-service \
  --cluster ${CLUSTER_NAME} \
  --service ${SERVICE_NAME} \
  --task-definition demo-ecs-web:2

aws ecs wait services-stable --cluster ${CLUSTER_NAME} --services ${SERVICE_NAME}

curl -m5 http://${ALB_DNS}:8080
```

**预期输出**：Service 稳定，ALB 恢复返回 v2 页面

---

## 验收标准

完成本实验后，你应当能够：
- [ ] Service `demo-ecs-web` 最终状态为 ACTIVE，running count = 2
- [ ] Task Definition `demo-ecs-web` 已有 3 个以上 revision（正常、v2、故障版本）
- [ ] 观察到故障发布时 stopped task 有 `CannotPullContainerError` 相关信息
- [ ] 手动回滚后 ALB 通过 `curl` 验证返回 v2 页面

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws ecs describe-services --cluster demo-ecs --services demo-ecs-web --region cn-northwest-1 --query 'services[0].status' --output text` | `ACTIVE` |
| 2 | `aws ecs describe-services --cluster demo-ecs --services demo-ecs-web --region cn-northwest-1 --query 'services[0].runningCount' --output text` | `2` |
| 3 | `aws ecs list-task-definitions --family-prefix demo-ecs-web --region cn-northwest-1 --query 'length(taskDefinitionArns)' --output text` | `3` 或以上 |

---

## 实验总结

本实验完成了 ECS 滚动发布的排查与回滚闭环，核心经验是：发布失败时先看 Service Events（快速定位），再看 stopped task（获取详细报错），最后通过指定已知健康的 revision 手动回滚。每次 `register-task-definition` 会生成新 revision，这是 ECS 发布历史的完整审计链。下一个 Demo（Demo06）将演示如何通过 Secrets Manager 安全注入敏感配置，并为 Task Role 配置最小权限。

---

## 清理

本 Demo 不创建独立 AWS 资源，无需额外清理。
