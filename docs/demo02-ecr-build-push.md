# Demo02 — ECR 镜像构建与发布

## 实验简介

本实验在中国区 ECR 创建私有镜像仓库，构建并推送多个版本镜像（v1、v2），配置 stable 稳定标签、漏洞扫描和生命周期策略。中国区 ECR Registry URI 格式特殊（以 `.amazonaws.com.cn` 结尾），是实际操作中最容易出错的地方。

**实验目标：**
- 掌握中国区 ECR 镜像仓库的创建和 Docker 登录流程
- 理解镜像 tag 版本管理（v1/v2/stable）和 digest 追踪方式
- 能够配置 scanOnPush 漏洞扫描和生命周期策略控制镜像存储成本

**实验流程：**
1. 创建 ECR 仓库（启用 scanOnPush 和 AES256 加密）
2. 使用 `get-login-password` 登录 ECR（中国区 URI 含 `.cn`）
3. 构建并推送 v1 镜像，记录 digest
4. 构建并推送 v2 镜像
5. 创建 `stable` 标签指向 v1
6. 查看镜像漏洞扫描结果
7. 配置生命周期策略自动清理过期镜像

**预计 AI 执行时长：** 7-10 分钟

---

## 前提条件

- **工具**：AWS CLI v2、Docker
- **权限**：ECR 创建/推送权限
- **前提**：Demo01 已完成，`/tmp/demo-ecs.env` 可用

---

## 步骤

### 1. 设置环境变量

```bash
source /tmp/demo-ecs.env
export ECR_REPO=demo-ecs-web
export IMAGE_TAG=v1
```

### 2. 创建 ECR 仓库

中国区 ECR 仓库 URI 格式为 `${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com.cn/${REPO}`。

```bash
aws ecr create-repository \
  --repository-name ${ECR_REPO} \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=AES256 \
  --tags Key=Project,Value=${PROJECT_TAG} Key=Demo,Value=Demo02 Key=Owner,Value=${OWNER_TAG} \
  --region ${AWS_REGION} || true

aws ecr describe-repositories \
  --repository-names ${ECR_REPO} \
  --region ${AWS_REGION} \
  --query 'repositories[0].{uri:repositoryUri,scan:imageScanningConfiguration.scanOnPush,encryption:encryptionConfiguration.encryptionType}'
```

**预期输出**：仓库 URI、scanOnPush=true、encryption=AES256

### 3. 登录 ECR

同时登录自己账号的 ECR 和共享镜像仓库（`048912060910`），后者提供 `public.ecr.aws` 镜像在中国区的可靠副本，供构建时拉取基础镜像。

```bash
# 登录自己账号的 ECR（推送镜像用）
aws ecr get-login-password --region ${AWS_REGION} \
  | docker login --username AWS --password-stdin \
    ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com.cn

# 登录共享镜像仓库（拉取基础镜像用，避免直接依赖 public.ecr.aws）
aws ecr get-login-password --region ${AWS_REGION} \
  | docker login --username AWS --password-stdin \
    048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn
```

**预期输出**：两次均输出 `Login Succeeded`

### 4. 构建并推送 v1

基础镜像使用中国区共享 ECR（`048912060910`）中的 nginx 镜像副本，避免直接拉取 `public.ecr.aws` 可能的网络抖动。

```bash
mkdir -p /tmp/demo-ecs-web
cat > /tmp/demo-ecs-web/Dockerfile <<'DOCKER'
FROM 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/nginx:1.27-alpine
ARG VERSION=v1
RUN echo "<h1>ECS China QuickStart ${VERSION}</h1>" > /usr/share/nginx/html/index.html
HEALTHCHECK --interval=30s --timeout=3s --retries=3 CMD wget -qO- http://127.0.0.1/ || exit 1
DOCKER

docker build --build-arg VERSION=v1 -t ${ECR_REPO}:v1 /tmp/demo-ecs-web
docker tag ${ECR_REPO}:v1 ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com.cn/${ECR_REPO}:v1
docker push ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com.cn/${ECR_REPO}:v1
```

记录 v1 digest：

```bash
export IMAGE_DIGEST_V1=$(aws ecr describe-images \
  --repository-name ${ECR_REPO} \
  --image-ids imageTag=v1 \
  --region ${AWS_REGION} \
  --query 'imageDetails[0].imageDigest' --output text)

echo "v1 digest: ${IMAGE_DIGEST_V1}"
```

**预期输出**：推送成功，输出 sha256 digest

### 5. 构建并推送 v2

```bash
docker build --build-arg VERSION=v2 -t ${ECR_REPO}:v2 /tmp/demo-ecs-web
docker tag ${ECR_REPO}:v2 ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com.cn/${ECR_REPO}:v2
docker push ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com.cn/${ECR_REPO}:v2

export IMAGE_DIGEST_V2=$(aws ecr describe-images \
  --repository-name ${ECR_REPO} \
  --image-ids imageTag=v2 \
  --region ${AWS_REGION} \
  --query 'imageDetails[0].imageDigest' --output text)
```

**预期输出**：v2 推送成功，输出 sha256 digest

### 6. 创建稳定标签 stable

`stable` 标签模拟生产发布中常见的"已验证版本"指针，先指向 v1。

```bash
aws ecr batch-get-image \
  --repository-name ${ECR_REPO} \
  --image-ids imageTag=v1 \
  --query 'images[0].imageManifest' \
  --output text \
  --region ${AWS_REGION} > /tmp/v1-manifest.json

aws ecr put-image \
  --repository-name ${ECR_REPO} \
  --image-tag stable \
  --image-manifest file:///tmp/v1-manifest.json \
  --region ${AWS_REGION}
```

**预期输出**：stable 标签创建成功

### 7. 查看镜像扫描结果

Push 时已开启 `scanOnPush=true`，扫描结果可能需要约 30 秒生成。

> ⚠️ **中国区限制**：`aws ecr start-image-scan` 在中国区返回 `ValidationException: This feature is disabled`。中国区仅支持 `scanOnPush=true` 自动触发，扫描状态为 `ACTIVE`（不会变为 `COMPLETE`）。直接查询 `describe-image-scan-findings` 即可获取漏洞统计，无需等待状态变更。

```bash
for tag in v1 v2; do
  echo "=== scan result for ${tag} ==="
  aws ecr describe-image-scan-findings \
    --repository-name ${ECR_REPO} \
    --image-id imageTag=${tag} \
    --region ${AWS_REGION} \
    --query '{status: imageScanStatus, counts: imageScanFindings.findingSeverityCounts}' || true
  sleep 15
done
```

**预期输出**：输出各 severity 等级漏洞数量，status 字段显示 `ACTIVE`

### 8. 配置生命周期策略

```bash
aws ecr put-lifecycle-policy \
  --repository-name ${ECR_REPO} \
  --lifecycle-policy-text '{"rules":[{"rulePriority":1,"description":"Remove untagged images after 1 day","selection":{"tagStatus":"untagged","countType":"sinceImagePushed","countUnit":"days","countNumber":1},"action":{"type":"expire"}},{"rulePriority":2,"description":"Keep latest 10 tagged images","selection":{"tagStatus":"any","countType":"imageCountMoreThan","countNumber":10},"action":{"type":"expire"}}]}' \
  --region ${AWS_REGION}
```

**预期输出**：生命周期策略设置成功

### 9. 保存镜像变量

```bash
cat >> /tmp/demo-ecs.env <<EOF2
export ECR_REPO=${ECR_REPO}
export IMAGE_URI_V1=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com.cn/${ECR_REPO}:v1
export IMAGE_URI_V2=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com.cn/${ECR_REPO}:v2
export IMAGE_URI_STABLE=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com.cn/${ECR_REPO}:stable
export IMAGE_DIGEST_V1=${IMAGE_DIGEST_V1}
export IMAGE_DIGEST_V2=${IMAGE_DIGEST_V2}
EOF2

echo "镜像变量已保存"
```

**预期输出**：打印"镜像变量已保存"

---

## 验收标准

完成本实验后，你应当能够：
- [ ] ECR 仓库 `demo-ecs-web` 已创建，scanOnPush 和 AES256 加密已启用
- [ ] 仓库中有 3 个镜像（v1、v2、stable），stable 标签指向 v1
- [ ] 两个镜像的漏洞扫描结果均已生成（约 30 秒后可查看）
- [ ] 生命周期策略已配置（2 条规则：清理 untagged 和保留最近 10 个标签）

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws ecr describe-repositories --repository-names demo-ecs-web --region cn-northwest-1 --query 'repositories[0].repositoryName' --output text` | `demo-ecs-web` |
| 2 | `aws ecr describe-images --repository-name demo-ecs-web --region cn-northwest-1 --query 'length(imageDetails)' --output text` | `3` |
| 3 | `aws ecr describe-images --repository-name demo-ecs-web --image-ids imageTag=stable --region cn-northwest-1 --query 'imageDetails[0].imageTags' --output text` | 包含 `stable` 和 `v1` |
| 4 | `aws ecr get-lifecycle-policy --repository-name demo-ecs-web --region cn-northwest-1 --query 'lifecyclePolicyText' --output text \| python3 -c "import sys,json; p=json.load(sys.stdin); print(len(p['rules']))"` | `2` |

---

## 实验总结

本实验建立了 ECR 私有镜像仓库和版本管理体系，v1/v2 对应不同功能版本，`stable` 标签作为生产部署指针（仅需移动标签即可切换版本）。中国区 ECR Registry URI 格式为 `<account>.dkr.ecr.<region>.amazonaws.com.cn`，多出的 `.cn` 是最常见的配置错误来源。下一个 Demo（Demo03）将使用 `stable` 镜像部署第一个 Fargate 服务，演示完整的容器启动和日志输出流程。

---

## 清理

后续 Demo 依赖该仓库，不建议立即清理。

```bash
aws ecr delete-repository --repository-name demo-ecs-web --force --region cn-northwest-1
```
