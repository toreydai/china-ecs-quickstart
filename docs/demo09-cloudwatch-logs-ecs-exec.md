# Demo09 — CloudWatch 日志、指标与 ECS Exec

## 实验简介

本实验演示 ECS 的可观测性工具链：通过 CloudWatch Logs 查看容器日志、通过 CloudWatch 指标监控 CPU/Memory，以及通过 ECS Exec（基于 SSM Session Manager）进入运行中的 Fargate 容器执行诊断命令。ECS Exec 是生产环境容器实时诊断的关键工具。

**实验目标：**
- 掌握 CloudWatch Logs tail 和 ECS 指标查询的基础用法
- 理解 ECS Exec 依赖 Task Role 的 `ssmmessages` 权限和本地 Session Manager plugin
- 能够通过 `aws ecs execute-command` 进入 Fargate 容器执行交互式命令

**实验流程：**
1. 通过 `aws logs tail` 和 `cloudwatch get-metric-statistics` 查看日志和指标
2. 为 Task Role 添加 ssmmessages 相关权限
3. 启用 Service 的 `execute-command` 并强制新部署
4. 获取运行中 Task ARN，执行 `ecs execute-command` 进入容器
5. 在容器内执行 hostname、env、wget 等命令验证

**预计 AI 执行时长：** 7-10 分钟

---

## 前提条件

- **工具**：AWS CLI v2、Session Manager plugin（`session-manager-plugin`）
- **权限**：ECS、IAM、CloudWatch、SSM 权限
- **前提**：Demo01-Demo08 已完成；操作机已安装 Session Manager plugin
- **中国区注意**：私有子网中 ECS Exec 需要 `ssmmessages` interface endpoint（Demo08 已创建）；启用 KMS 加密时还需要 KMS endpoint 和额外 IAM 权限

---

## 步骤

### 1. 设置环境变量

```bash
source /tmp/demo-ecs.env
export SERVICE_NAME=demo-ecs-web
export TASK_FAMILY=demo-ecs-web
```

### 2. 查看日志和指标

```bash
aws logs tail ${LOG_GROUP} --since 10m

aws cloudwatch get-metric-statistics \
  --namespace AWS/ECS \
  --metric-name CPUUtilization \
  --dimensions Name=ClusterName,Value=${CLUSTER_NAME} Name=ServiceName,Value=${SERVICE_NAME} \
  --statistics Average \
  --period 60 \
  --start-time $(date -u -d '15 minutes ago' +%FT%TZ 2>/dev/null || date -u -v-15M +%FT%TZ) \
  --end-time $(date -u +%FT%TZ)
```

**预期输出**：日志内容正常；CloudWatch 指标有 CPU 利用率数据点

### 3. 为 ECS Exec 授权 Task Role

Task role 需要 SSM Messages 权限才能建立 ECS Exec 会话。

```bash
cat > /tmp/demo-ecs-exec-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "ssmmessages:CreateControlChannel",
      "ssmmessages:CreateDataChannel",
      "ssmmessages:OpenControlChannel",
      "ssmmessages:OpenDataChannel"
    ],
    "Resource": "*"
  }]
}
EOF

aws iam put-role-policy \
  --role-name demo-ecs-task-role \
  --policy-name demo-ecs-exec \
  --policy-document file:///tmp/demo-ecs-exec-policy.json
```

**预期输出**：策略添加成功

### 4. 启用 Execute Command

```bash
aws ecs update-service \
  --cluster ${CLUSTER_NAME} \
  --service ${SERVICE_NAME} \
  --enable-execute-command \
  --force-new-deployment

aws ecs wait services-stable --cluster ${CLUSTER_NAME} --services ${SERVICE_NAME}

echo "ECS Exec 已启用，等待新 task 启动..."
sleep 10
```

**预期输出**：Service 稳定，新 task 已启动并支持 execute-command

### 5. 进入容器

```bash
TASK_ARN=$(aws ecs list-tasks \
  --cluster ${CLUSTER_NAME} \
  --service-name ${SERVICE_NAME} \
  --query 'taskArns[0]' --output text)

echo "连接 task: ${TASK_ARN}"

aws ecs execute-command \
  --cluster ${CLUSTER_NAME} \
  --task ${TASK_ARN} \
  --container web \
  --interactive \
  --command "/bin/sh"
```

在容器内可执行：
- `hostname` — 查看容器 hostname
- `env` — 确认环境变量（包含 Demo06 注入的变量）
- `wget -qO- http://127.0.0.1/ && exit` — 验证 nginx 运行正常

**预期输出**：进入容器 shell，可执行命令

### 6. 查看 ECS Exec 执行历史

```bash
aws ecs list-tasks \
  --cluster ${CLUSTER_NAME} \
  --service-name ${SERVICE_NAME} \
  --query 'taskArns'

aws ecs describe-tasks \
  --cluster ${CLUSTER_NAME} \
  --tasks ${TASK_ARN} \
  --query 'tasks[0].containers[0].managedAgents'
```

**预期输出**：managedAgents 中有 `ExecuteCommandAgent`，status 为 RUNNING

---

## 验收标准

完成本实验后，你应当能够：
- [ ] Task Role `demo-ecs-task-role` 有 `demo-ecs-exec` inline policy（包含 ssmmessages 权限）
- [ ] Service `enableExecuteCommand` 属性为 `True`
- [ ] CloudWatch 日志组有日志流，包含 nginx 访问日志
- [ ] 成功通过 `ecs execute-command` 进入容器并执行命令

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws iam get-role-policy --role-name demo-ecs-task-role --policy-name demo-ecs-exec --query 'PolicyName' --output text` | `demo-ecs-exec` |
| 2 | `aws ecs describe-services --cluster demo-ecs --services demo-ecs-web --region cn-northwest-1 --query 'services[0].enableExecuteCommand' --output text` | `True` |
| 3 | `aws logs describe-log-streams --log-group-name /ecs/demo-ecs --region cn-northwest-1 --order-by LastEventTime --descending --max-items 1 --query 'logStreams[0].logStreamName' --output text` | 非空（有日志流名称） |

---

## 实验总结

本实验建立了 ECS 服务的完整可观测性体系：CloudWatch Logs 提供容器日志集中存储，CloudWatch Metrics 提供 CPU/Memory 利用率趋势，ECS Exec 提供实时容器内部诊断能力。私有子网环境中 ECS Exec 依赖 `ssmmessages` Interface VPC Endpoint（Demo08 已创建），缺少此 Endpoint 会导致 execute-command 超时。下一个 Demo（Demo10）将构建 CodeCommit + CodeBuild + CodePipeline 的完整 CI/CD 流水线，实现代码推送自动触发 ECS 部署。

---

## 清理

```bash
source /tmp/demo-ecs.env

aws iam delete-role-policy --role-name demo-ecs-task-role --policy-name demo-ecs-exec
```
