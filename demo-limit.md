# 中国区 ECS QuickStart Demo 限制与卡点

本文汇总在 AWS 中国区运行 ECS QuickStart demos 时常见的限制与卡点。

## 1. public.ecr.aws 镜像访问不稳定

**现象**

- Dockerfile `FROM public.ecr.aws/...` 在 `docker build` 时拉取超时或失败。
- Fargate task 启动时拉取 `public.ecr.aws/docker/library/busybox` 等镜像报 `CannotPullContainerError`。

**影响 Demo**

- Demo02：Dockerfile 使用 nginx 基础镜像
- Demo07：压测 task 使用 busybox
- Demo10：CodeBuild buildspec Dockerfile 使用 nginx
- Demo11：EFS 验证 task 使用 busybox
- Demo12：Frontend task 使用 busybox

**规避方式**

所有 `public.ecr.aws/docker/library/` 镜像统一改用中国区共享镜像仓库（`048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn`）：

```bash
# 对应替换规则：
# public.ecr.aws/docker/library/nginx:1.27-alpine
→ 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/nginx:1.27-alpine

# public.ecr.aws/docker/library/busybox:1.36
→ 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/busybox:1.36
```

`docker build` 前额外登录共享仓库：

```bash
aws ecr get-login-password --region cn-northwest-1 \
  | docker login --username AWS --password-stdin \
    048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn
```

## 2. ECR start-image-scan 不可用

**现象**

- `aws ecr start-image-scan` 返回 `ValidationException: This feature is disabled`。
- 使用轮询等待 `imageScanStatus = COMPLETE` 的脚本会永远阻塞，因为状态只会是 `ACTIVE`。

**影响 Demo**

- Demo02：步骤 7 查看镜像漏洞扫描结果

**规避方式**

- 中国区只用 `scanOnPush=true` 触发自动扫描（创建仓库时设置）。
- 直接查询 `describe-image-scan-findings`，不依赖 `start-image-scan`，不等待 `COMPLETE` 状态。

```bash
aws ecr describe-image-scan-findings \
  --repository-name <repo> \
  --image-id imageTag=<tag> \
  --region cn-northwest-1 \
  --query '{status: imageScanStatus, counts: imageScanFindings.findingSeverityCounts}'
# 输出 status 为 ACTIVE 即正常，可直接看 counts 漏洞统计
```

## 3. AWS 中国区 Partition 与 Endpoint

**现象**

- IAM policy、ARN、ECR、S3、CodeCommit 等命令报 ARN 或 endpoint 错误。

**中国区必须使用的格式**

```text
arn:aws-cn:                                         # ARN 前缀
*.amazonaws.com.cn                                  # 服务 endpoint 域名
<account>.dkr.ecr.cn-northwest-1.amazonaws.com.cn  # ECR Registry
https://git-codecommit.cn-northwest-1.amazonaws.com.cn/v1/repos/<repo>  # CodeCommit
```

**规避方式**

每个 Demo 前设置环境变量：

```bash
export AWS_PARTITION=aws-cn
export AWS_REGION=cn-northwest-1
export ECR_REGISTRY=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com.cn
```

## 4. S3 Bucket 创建方式

**现象**

- 在 `cn-northwest-1` 使用简化版 `aws s3 mb` 或不带 LocationConstraint 的 `create-bucket` 失败。

**影响 Demo**

- Demo06：Task Role S3 权限演示 bucket
- Demo10：CodePipeline artifact bucket

**规避方式**

使用 `s3api` 显式指定 LocationConstraint：

```bash
aws s3api create-bucket \
  --bucket <bucket-name> \
  --region cn-northwest-1 \
  --create-bucket-configuration LocationConstraint=cn-northwest-1
```

## 5. 公网 Load Balancer 端口

**现象**

- 公网 80/443 端口访问失败，中国区未备案域名受 ICP 备案要求影响。

**影响 Demo**

- Demo04：ALB 暴露服务
- Demo13：CodeDeploy 蓝绿发布独立 Listener

**规避方式**

- 公网 HTTP Demo 统一使用 `8080`（Demo04）或 `8081`（Demo13）。
- ALB Listener 创建时指定非标准端口。

## 6. CodeCommit 认证

**现象**

- `git push` 到 CodeCommit 失败，HTTPS 认证反复提示。

**影响 Demo**

- Demo10：CodePipeline CI/CD

**规避方式**

配置 AWS CodeCommit credential helper：

```bash
git config --global credential.helper '!aws codecommit credential-helper $@'
git config --global credential.UseHttpPath true
```

中国区 CodeCommit clone URL 格式：

```text
https://git-codecommit.cn-northwest-1.amazonaws.com.cn/v1/repos/<repo>
```

## 7. ECS Exec 在私有子网中的依赖

**现象**

- 完成 Demo08（私有子网迁移）后，`aws ecs execute-command` 超时或无法连接。

**影响 Demo**

- Demo09：ECS Exec（前提是 Demo08 已迁移到私有子网）

**规避方式**

- Demo08 中已创建 `ssmmessages` Interface VPC Endpoint，这是 ECS Exec 在私有子网中工作的必要条件。
- 确保 `ssmmessages` endpoint 状态为 `available` 后再执行 Demo09。

## 8. VPC Endpoint 费用

**现象**

- Interface VPC Endpoint 按小时计费，Demo08 创建 6 个 Interface Endpoint，如不及时清理会持续产生费用。

**影响 Demo**

- Demo08：私有子网与 VPC Endpoint

**规避方式**

- 完成 Demo08 及依赖私有子网的实验后，立即执行 Demo08 清理步骤删除所有 Endpoint。
- Demo14 成本审计步骤中会检查残留 Endpoint。

## 9. ALB Target Group 类型必须为 ip

**现象**

- Fargate + ALB Target Group 使用默认 `instance` 类型时，target 注册失败或健康检查异常。

**影响 Demo**

- Demo04：ALB 暴露服务
- Demo13：CodeDeploy 蓝绿发布（Blue/Green TG）

**规避方式**

Fargate 使用 `awsvpc` 网络模式，每个 Task 有独立 ENI，不经过主机端口，必须使用 `--target-type ip`：

```bash
aws elbv2 create-target-group \
  --name <name> \
  --target-type ip \   # 不能省略，Fargate 必选
  ...
```

## 10. CodeDeploy 蓝绿发布 Target Group 配置

**现象**

- `aws deploy create-deployment-group` 报错，或蓝绿发布过程中流量切换失败。

**影响 Demo**

- Demo13：CodeDeploy 蓝绿发布

**规避方式**

- ECS Service `--deployment-controller type=CODE_DEPLOY` 中的 `--load-balancers targetGroupArn=<blue_tg>` 必须与 Deployment Group 的 `targetGroupPairInfoList` 中 `targetGroups[0]` 对应的 Blue TG 完全一致。
- 两个 TG（Blue/Green）必须在 ECS Service 创建前就存在，不能在创建 Deployment Group 时才创建。

## 11. IAM Role 传播延迟

**现象**

- 创建 IAM Role 后立即用于 CodeBuild/CodeDeploy 等服务，报 `EntityAlreadyExists`（重跑时）或权限不生效。

**影响 Demo**

- Demo10：CodeBuild/CodePipeline role 创建后立即使用
- Demo13：CodeDeploy role 创建后立即关联 Deployment Group

**规避方式**

- IAM Role 创建后加 `sleep 10` 等待 IAM 传播再使用。
- 如遇 `EntityAlreadyExists`，用 `|| true` 忽略后继续（Role 已存在时直接复用）。

## 12. EFS Mount Target 删除顺序

**现象**

- 直接 `aws efs delete-file-system` 报 `FileSystemInUse`。

**影响 Demo**

- Demo11：EFS 持久化存储清理

**规避方式**

必须先删除所有 Mount Target 并等待删除完成，再删除 File System：

```bash
# 先删 Mount Target
for MT_ID in $(aws efs describe-mount-targets --file-system-id <fs-id> \
  --query 'MountTargets[].MountTargetId' --output text); do
  aws efs delete-mount-target --mount-target-id ${MT_ID}
done
sleep 30   # 等待删除完成
# 再删 File System
aws efs delete-file-system --file-system-id <fs-id>
```

## Demo 前检查清单

```text
[ ] 操作机在中国区，public.ecr.aws 和 048912060910 ECR 均可访问。
[ ] 已执行双 ECR 登录（自己账号 + 048912060910）。
[ ] 所有 IAM ARN 使用 arn:aws-cn:，managed policy 使用 arn:aws-cn:iam::aws:policy/。
[ ] S3 bucket 创建时指定 LocationConstraint=cn-northwest-1。
[ ] 公网 ALB Demo 使用 8080 端口，Demo13 蓝绿额外 Listener 使用 8081。
[ ] Fargate ALB Target Group 必须指定 --target-type ip。
[ ] CodeCommit credential helper 已配置。
[ ] CodeDeploy Blue/Green TG 在 ECS Service 创建前已存在。
[ ] IAM Role 创建后 sleep 10 再使用。
[ ] Demo08 执行后，Interface Endpoint 费用及时清理。
[ ] Demo11 清理时先删 Mount Target，再删 File System。
```
