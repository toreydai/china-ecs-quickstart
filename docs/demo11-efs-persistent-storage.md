# Demo11 — EFS 持久化存储

## 实验简介

本实验为 Fargate Task 挂载 Amazon EFS 共享文件系统，通过运行两次 Task 验证数据在容器销毁和重新创建后的持久保留。EFS 是 ECS Fargate 中实现有状态工作负载（文件共享、数据库、缓存持久化）的核心存储方案。

**实验目标：**
- 掌握 EFS 文件系统、Mount Target 和安全组（允许 Task SG 访问 NFS 2049 端口）的配置链路
- 理解 Task Definition 中 `volumes` 字段（efsVolumeConfiguration）和 `mountPoints` 的配置关系
- 能够通过运行两次 Task 验证跨 Task 的数据持久性（hello.txt 累积时间戳）

**实验流程：**
1. 创建 EFS 安全组（允许 TCP 2049）并创建加密 EFS 文件系统
2. 在 2 个公有子网创建 Mount Target
3. 等待 Mount Target 状态变为 available
4. 注册带 EFS Volume 的 Task Definition
5. 运行第一次 Task，验证 hello.txt 有 1 行时间戳
6. 运行第二次 Task，验证 hello.txt 有 2 行时间戳（数据持久保留）

**预计 AI 执行时长：** 10-13 分钟

---

## 前提条件

- **工具**：AWS CLI v2
- **权限**：EFS、ECS、EC2 权限
- **前提**：Demo01 已完成，VPC 中已有公有子网
- **注意**：EFS 按存储量计费，实验结束后及时清理 mount target 和 file system

---

## 步骤

### 1. 设置环境变量

```bash
source /tmp/demo-ecs.env
export EFS_NAME=demo-ecs-efs
export EFS_SG_NAME=demo-ecs-efs-sg
```

### 2. 创建 EFS 与安全组

```bash
export EFS_SG_ID=$(aws ec2 create-security-group \
  --group-name ${EFS_SG_NAME} \
  --description "EFS security group for ECS demo" \
  --vpc-id ${VPC_ID} \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress \
  --group-id ${EFS_SG_ID} \
  --protocol tcp \
  --port 2049 \
  --source-group ${TASK_SG_ID}

export FILE_SYSTEM_ID=$(aws efs create-file-system \
  --creation-token ${EFS_NAME} \
  --encrypted \
  --tags Key=Name,Value=${EFS_NAME} Key=Project,Value=${PROJECT_TAG} Key=Demo,Value=Demo11 Key=Owner,Value=${OWNER_TAG} \
  --query 'FileSystemId' --output text)

aws efs create-mount-target \
  --file-system-id ${FILE_SYSTEM_ID} \
  --subnet-id ${PUBLIC_SUBNET_1} \
  --security-groups ${EFS_SG_ID}

aws efs create-mount-target \
  --file-system-id ${FILE_SYSTEM_ID} \
  --subnet-id ${PUBLIC_SUBNET_2} \
  --security-groups ${EFS_SG_ID}
```

**预期输出**：EFS File System ID 输出，2 个 Mount Target 创建

### 3. 等待 Mount Target 变为 available

```bash
until [ "$(aws efs describe-mount-targets \
  --file-system-id ${FILE_SYSTEM_ID} \
  --query 'MountTargets[*].LifeCycleState' --output text \
  | tr '\t' '\n' | grep -v available | wc -l)" -eq 0 ]; do
  echo "waiting for mount targets..."; sleep 10
done

aws efs describe-mount-targets \
  --file-system-id ${FILE_SYSTEM_ID} \
  --query 'MountTargets[].{Subnet:SubnetId,State:LifeCycleState}' --output table
```

**预期输出**：两个 Mount Target 状态均为 available

### 4. 注册带 EFS Volume 的 Task Definition

```bash
cat > /tmp/demo-efs-task.json <<EOF
{
  "family": "demo-ecs-efs",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws-cn:iam::${ACCOUNT_ID}:role/demo-ecs-task-execution-role",
  "taskRoleArn": "arn:aws-cn:iam::${ACCOUNT_ID}:role/demo-ecs-task-role",
  "volumes": [{
    "name": "data",
    "efsVolumeConfiguration": {
      "fileSystemId": "${FILE_SYSTEM_ID}",
      "transitEncryption": "ENABLED"
    }
  }],
  "containerDefinitions": [{
    "name": "app",
    "image": "048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/busybox:1.36",
    "command": ["sh","-c","date >> /data/hello.txt && cat /data/hello.txt"],
    "mountPoints": [{"sourceVolume":"data","containerPath":"/data"}],
    "essential": true,
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "${LOG_GROUP}",
        "awslogs-region": "${AWS_REGION}",
        "awslogs-stream-prefix": "efs"
      }
    }
  }]
}
EOF

aws ecs register-task-definition --cli-input-json file:///tmp/demo-efs-task.json
```

**预期输出**：Task definition `demo-ecs-efs:1` 注册成功

### 5. 运行第一次 Task 并验证

```bash
TASK1_ARN=$(aws ecs run-task \
  --cluster ${CLUSTER_NAME} \
  --launch-type FARGATE \
  --task-definition demo-ecs-efs \
  --network-configuration "awsvpcConfiguration={subnets=[${PUBLIC_SUBNET_1}],securityGroups=[${TASK_SG_ID}],assignPublicIp=ENABLED}" \
  --query 'tasks[0].taskArn' --output text)

aws ecs wait tasks-stopped --cluster ${CLUSTER_NAME} --tasks ${TASK1_ARN}

echo "=== 第一次 task 日志 ==="
aws logs tail ${LOG_GROUP} --log-stream-name-prefix efs --since 5m
```

**预期输出**：日志中输出 `/data/hello.txt` 内容（第一次只有一行时间戳）

### 6. 运行第二次 Task 验证数据持久化

```bash
TASK2_ARN=$(aws ecs run-task \
  --cluster ${CLUSTER_NAME} \
  --launch-type FARGATE \
  --task-definition demo-ecs-efs \
  --network-configuration "awsvpcConfiguration={subnets=[${PUBLIC_SUBNET_1}],securityGroups=[${TASK_SG_ID}],assignPublicIp=ENABLED}" \
  --query 'tasks[0].taskArn' --output text)

aws ecs wait tasks-stopped --cluster ${CLUSTER_NAME} --tasks ${TASK2_ARN}

echo "=== 第二次 task 日志（应包含两行时间戳）==="
aws logs tail ${LOG_GROUP} --log-stream-name-prefix efs --since 5m
```

**预期输出**：日志中 `/data/hello.txt` 包含两行时间戳，证明数据跨 task 持久保留

---

## 验收标准

完成本实验后，你应当能够：
- [ ] EFS 文件系统状态为 available，启用了加密
- [ ] 2 个 Mount Target 均处于 available 状态，分别位于 2 个公有子网
- [ ] Task Definition `demo-ecs-efs` 包含 `data` volume（efsVolumeConfiguration 配置）
- [ ] 第二次 Task 日志中 `/data/hello.txt` 包含 2 行时间戳，证明数据跨 Task 持久保留

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws efs describe-file-systems --creation-token demo-ecs-efs --region cn-northwest-1 --query 'FileSystems[0].LifeCycleState' --output text` | `available` |
| 2 | `aws efs describe-mount-targets --file-system-id $(aws efs describe-file-systems --creation-token demo-ecs-efs --region cn-northwest-1 --query 'FileSystems[0].FileSystemId' --output text) --region cn-northwest-1 --query 'length(MountTargets)' --output text` | `2` |
| 3 | `aws ecs describe-task-definition --task-definition demo-ecs-efs --region cn-northwest-1 --query 'taskDefinition.volumes[0].name' --output text` | `data` |

---

## 实验总结

本实验验证了 Fargate 容器通过 EFS 实现跨 Task 数据持久化的完整流程，EFS transitEncryption=ENABLED 确保传输层加密（生产环境推荐）。EFS 按实际存储量计费，Mount Target 也有小时费用，实验后及时清理。相较于 EBS（单实例挂载），EFS 支持多个 Task 同时挂载同一文件系统，是文件共享型有状态服务的首选。下一个 Demo（Demo12）将配置 Cloud Map 服务发现，实现 Frontend 和 Backend 服务之间的 DNS 名称通信。

---

## 清理

```bash
source /tmp/demo-ecs.env

FILE_SYSTEM_ID=$(aws efs describe-file-systems \
  --creation-token demo-ecs-efs \
  --query 'FileSystems[0].FileSystemId' --output text)

MOUNT_TARGET_IDS=$(aws efs describe-mount-targets \
  --file-system-id ${FILE_SYSTEM_ID} \
  --query 'MountTargets[].MountTargetId' --output text)

for MT_ID in ${MOUNT_TARGET_IDS}; do
  aws efs delete-mount-target --mount-target-id ${MT_ID}
  echo "已删除 mount target: ${MT_ID}"
done

echo "等待 mount target 删除..."
sleep 30

aws efs delete-file-system --file-system-id ${FILE_SYSTEM_ID}

EFS_SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=demo-ecs-efs-sg" \
  --query 'SecurityGroups[0].GroupId' --output text)
aws ec2 delete-security-group --group-id ${EFS_SG_ID}
```
