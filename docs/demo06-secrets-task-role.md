# Demo06 — 任务配置：Secrets 与 Task Role

## 实验简介

本实验演示如何通过 Secrets Manager 将密钥安全注入 ECS 容器，以及如何为 Task Role 配置最小权限 S3 访问策略。理解 Execution Role（基础设施层）和 Task Role（应用层）的职责边界是 ECS 安全配置的关键。

**实验目标：**
- 掌握 Secrets Manager Secret 创建、Execution Role 授权和 Task Definition secrets 字段配置的完整链路
- 理解 Execution Role（拉镜像/注入密钥/写日志）与 Task Role（应用访问 AWS 服务）的职责边界
- 能够为 Task Role 编写最小权限 IAM 策略（仅允许访问特定 S3 bucket）

**实验流程：**
1. 在 Secrets Manager 创建 Secret（JSON 格式密码）
2. 为 Execution Role 添加 `secretsmanager:GetSecretValue` 权限
3. 更新 Task Definition，通过 `secrets` 字段注入密钥为环境变量
4. 创建演示 S3 bucket，为 Task Role 添加最小权限 s3:ListBucket 策略
5. 滚动发布新 Task Definition，验证 Secret 注入生效

**预计 AI 执行时长：** 7-10 分钟

---

## 前提条件

- **工具**：AWS CLI v2、jq
- **权限**：ECS、IAM、Secrets Manager、S3 权限
- **前提**：Demo01-Demo04 已完成

---

## 步骤

### 1. 设置环境变量

```bash
source /tmp/demo-ecs.env
export SERVICE_NAME=demo-ecs-web
export TASK_FAMILY=demo-ecs-web
export SECRET_NAME=demo-ecs/app/password
```

### 2. 创建 Secret

```bash
aws secretsmanager create-secret \
  --name ${SECRET_NAME} \
  --secret-string '{"password":"demo-password"}' \
  --tags Key=Project,Value=${PROJECT_TAG} Key=Demo,Value=Demo06 Key=Owner,Value=${OWNER_TAG} \
  --region ${AWS_REGION} || true

export SECRET_ARN=$(aws secretsmanager describe-secret \
  --secret-id ${SECRET_NAME} \
  --query ARN --output text \
  --region ${AWS_REGION})

echo "Secret ARN: ${SECRET_ARN}"
```

**预期输出**：输出 Secret ARN（以 `arn:aws-cn:` 开头）

### 3. 授权 Execution Role 读取 Secret

Secrets 注入发生在容器启动阶段，由 task execution role 读取，而不是 task role。

```bash
cat > /tmp/demo-ecs-exec-secrets-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["secretsmanager:GetSecretValue"],
    "Resource": "${SECRET_ARN}"
  }]
}
EOF

aws iam put-role-policy \
  --role-name demo-ecs-task-execution-role \
  --policy-name demo-ecs-read-secret \
  --policy-document file:///tmp/demo-ecs-exec-secrets-policy.json
```

**预期输出**：策略添加成功

### 4. 更新 Task Definition，注入 Secret 和环境变量

```bash
# 注意：使用已知好版本的 revision 作为基础（避免拿到 bad image 版本）
# 动态获取最后一个使用正规镜像的 revision
GOOD_REVISION=$(aws ecs list-task-definitions \
  --family-prefix ${TASK_FAMILY} \
  --sort DESC \
  --query 'taskDefinitionArns' --output json | python3 -c "
import sys,json
arns=json.load(sys.stdin)
# 跳过 bad image revision（最新），取次新
print(arns[1].split(':')[-1] if len(arns)>=2 else arns[0].split(':')[-1])
")

aws ecs describe-task-definition \
  --task-definition ${TASK_FAMILY}:${GOOD_REVISION} \
  --query 'taskDefinition' > /tmp/taskdef-current.json

jq --arg SECRET_ARN "${SECRET_ARN}" '
  del(.taskDefinitionArn,.revision,.status,.requiresAttributes,.compatibilities,.registeredAt,.registeredBy)
  | .containerDefinitions[0].environment = [
      {"name":"APP_ENV","value":"demo"},
      {"name":"AWS_REGION","value":"cn-northwest-1"}
    ]
  | .containerDefinitions[0].secrets = [
      {"name":"APP_PASSWORD","valueFrom":$SECRET_ARN}
    ]
' /tmp/taskdef-current.json > /tmp/taskdef-secrets.json

aws ecs register-task-definition --cli-input-json file:///tmp/taskdef-secrets.json

aws ecs update-service \
  --cluster ${CLUSTER_NAME} \
  --service ${SERVICE_NAME} \
  --task-definition ${TASK_FAMILY}

aws ecs wait services-stable --cluster ${CLUSTER_NAME} --services ${SERVICE_NAME}
```

**预期输出**：新 task definition 注册成功，service 稳定

### 5. Task Role 最小权限示例

给应用容器的 task role 授权访问一个指定 S3 bucket，演示最小权限写法。中国区 S3 创建 bucket 需要指定 `LocationConstraint`。

```bash
export DEMO_BUCKET=demo-ecs-${ACCOUNT_ID}-${AWS_REGION}

aws s3api create-bucket \
  --bucket ${DEMO_BUCKET} \
  --region ${AWS_REGION} \
  --create-bucket-configuration LocationConstraint=${AWS_REGION} || true

cat > /tmp/demo-ecs-task-s3-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:ListBucket"],
    "Resource": "arn:aws-cn:s3:::${DEMO_BUCKET}"
  }]
}
EOF

aws iam put-role-policy \
  --role-name demo-ecs-task-role \
  --policy-name demo-ecs-list-one-bucket \
  --policy-document file:///tmp/demo-ecs-task-s3-policy.json
```

**预期输出**：S3 bucket 创建成功（或已存在），task role inline policy 添加成功

### 6. 验证

```bash
aws ecs describe-task-definition \
  --task-definition ${TASK_FAMILY} \
  --query 'taskDefinition.containerDefinitions[0].{env:environment,secrets:secrets}'

aws ecs describe-services \
  --cluster ${CLUSTER_NAME} \
  --services ${SERVICE_NAME} \
  --query 'services[0].{desired:desiredCount,running:runningCount}'
```

**预期输出**：environment 和 secrets 均有配置，running=desired

---

## 验收标准

完成本实验后，你应当能够：
- [ ] Secrets Manager 中有 `demo-ecs/app/password` 密钥
- [ ] `demo-ecs-task-execution-role` 有 `demo-ecs-read-secret` inline policy
- [ ] `demo-ecs-task-role` 有 `demo-ecs-list-one-bucket` inline policy（仅限指定 bucket）
- [ ] Service running count = 2，Task Definition 的 containers 配置包含 `secrets` 字段

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws secretsmanager describe-secret --secret-id demo-ecs/app/password --region cn-northwest-1 --query 'Name' --output text` | `demo-ecs/app/password` |
| 2 | `aws iam get-role-policy --role-name demo-ecs-task-execution-role --policy-name demo-ecs-read-secret --query 'PolicyName' --output text` | `demo-ecs-read-secret` |
| 3 | `aws iam get-role-policy --role-name demo-ecs-task-role --policy-name demo-ecs-list-one-bucket --query 'PolicyName' --output text` | `demo-ecs-list-one-bucket` |
| 4 | `aws ecs describe-services --cluster demo-ecs --services demo-ecs-web --region cn-northwest-1 --query 'services[0].runningCount' --output text` | `2` |

---

## 实验总结

本实验建立了 ECS 密钥安全注入的标准模式：Secrets Manager 存储密钥 → Execution Role 授权读取 → Task Definition `secrets` 字段注入为容器环境变量。Task Role 的最小权限示例（仅 s3:ListBucket 特定 bucket）展示了生产环境权限收敛的最佳实践。下一个 Demo（Demo07）将为服务配置基于 CPU 的 Target Tracking 自动伸缩，演示 ECS 弹性扩缩容能力。

---

## 清理

```bash
source /tmp/demo-ecs.env

aws iam delete-role-policy --role-name demo-ecs-task-execution-role --policy-name demo-ecs-read-secret
aws iam delete-role-policy --role-name demo-ecs-task-role --policy-name demo-ecs-list-one-bucket
aws secretsmanager delete-secret --secret-id ${SECRET_NAME} --force-delete-without-recovery --region ${AWS_REGION}
aws s3 rb s3://${DEMO_BUCKET} --force 2>/dev/null || true
```
