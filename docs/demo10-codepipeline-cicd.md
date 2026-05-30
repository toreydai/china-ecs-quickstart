# Demo10 — CodePipeline 自动部署 ECS

## 实验简介

本实验构建基于 CodeCommit + CodeBuild + CodePipeline 的 ECS 自动化发布流水线，实现代码推送即触发镜像构建和 ECS 服务部署。中国区 CodeCommit clone URL 格式（`amazonaws.com.cn`）和 CodeBuild service role ARN 前缀（`arn:aws-cn:`）是配置中需要特别注意的差异。

**实验目标：**
- 掌握三段式 CI/CD 流水线（Source→Build→Deploy）的搭建和 IAM 权限配置
- 理解中国区 CodeCommit URL 格式和 buildspec.yml 中的 ECR 镜像推送写法
- 能够通过修改代码并推送触发 Pipeline 完成端到端自动发布验证

**实验流程：**
1. 创建 CodeCommit 仓库，推送 Dockerfile 和 buildspec.yml
2. 创建 Artifact S3 Bucket（需指定 LocationConstraint）
3. 创建 CodeBuild 项目和 IAM Service Role
4. 创建 CodePipeline IAM Role 和流水线配置
5. 等待首次自动触发发布完成（约 5 分钟）
6. 修改 Dockerfile 推送代码，验证自动触发第二次发布

**预计 AI 执行时长：** 13-16 分钟（含 Pipeline 运行等待时间）

---

## 前提条件

- **工具**：AWS CLI v2、git
- **权限**：CodeCommit、CodeBuild、CodePipeline、IAM、ECR、S3、ECS 权限
- **前提**：Demo01-Demo04 已完成；复用已有 ECS cluster、service、ALB 和 ECR 仓库
- **中国区注意**：CodeCommit clone URL 格式为 `https://git-codecommit.${AWS_REGION}.amazonaws.com.cn/v1/repos/${REPO_NAME}`；S3 创建 bucket 需要 `LocationConstraint`；CodeBuild service role ARN 使用 `arn:aws-cn:`

---

## 步骤

### 1. 设置环境变量

```bash
source /tmp/demo-ecs.env
export ECR_REPO=demo-ecs-web
export SERVICE_NAME=demo-ecs-web
export CONTAINER_NAME=web
export REPO_NAME=demo-ecs-app
export CODEBUILD_PROJECT=demo-ecs-build
export PIPELINE_NAME=demo-ecs-pipeline
export PIPELINE_BUCKET=demo-ecs-pipeline-${ACCOUNT_ID}-${AWS_REGION}
export CODEBUILD_ROLE=demo-ecs-codebuild-role
export CODEPIPELINE_ROLE=demo-ecs-codepipeline-role
```

### 2. 创建 CodeCommit 仓库

```bash
aws codecommit create-repository \
  --repository-name ${REPO_NAME} \
  --repository-description "ECS China QuickStart app" \
  --tags Project=${PROJECT_TAG},Demo=Demo10,Owner=${OWNER_TAG} || true
```

准备应用代码和 `buildspec.yml`：

```bash
mkdir -p /tmp/${REPO_NAME}
cd /tmp/${REPO_NAME}

cat > Dockerfile <<'DOCKER'
FROM 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/ecrpublic/docker/library/nginx:1.27-alpine
ARG VERSION=dev
RUN echo "<h1>ECS pipeline ${VERSION}</h1>" > /usr/share/nginx/html/index.html
DOCKER

cat > buildspec.yml <<EOF2
version: 0.2
phases:
  pre_build:
    commands:
      - aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com.cn
      - aws ecr get-login-password --region cn-northwest-1 | docker login --username AWS --password-stdin 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn
      - IMAGE_TAG=build-\$(date +%Y%m%d%H%M%S)
      - IMAGE_URI=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com.cn/${ECR_REPO}:\${IMAGE_TAG}
  build:
    commands:
      - docker build --build-arg VERSION=\${IMAGE_TAG} -t \${IMAGE_URI} .
      - docker push \${IMAGE_URI}
  post_build:
    commands:
      - printf '[{"name":"${CONTAINER_NAME}","imageUri":"%s"}]' \${IMAGE_URI} > imagedefinitions.json
artifacts:
  files:
    - imagedefinitions.json
EOF2
```

推送到 CodeCommit（中国区 URL 格式）：

```bash
git init
git config user.email "ecs-demo@example.com"
git config user.name "ecs-demo"
git add .
git commit -m "initial ecs app"
git branch -M main

git config --global credential.helper '!aws codecommit credential-helper $@'
git config --global credential.UseHttpPath true

git remote add origin https://git-codecommit.${AWS_REGION}.amazonaws.com.cn/v1/repos/${REPO_NAME}
git push -u origin main
```

**预期输出**：代码推送成功

### 3. 创建 Artifact Bucket

```bash
aws s3api create-bucket \
  --bucket ${PIPELINE_BUCKET} \
  --region ${AWS_REGION} \
  --create-bucket-configuration LocationConstraint=${AWS_REGION} || true
```

**预期输出**：Bucket 创建成功（或已存在）

### 4. 创建 CodeBuild Role 和 Project

```bash
cat > /tmp/codebuild-trust.json <<'JSON'
{
  "Version":"2012-10-17",
  "Statement":[{
    "Effect":"Allow",
    "Principal":{"Service":"codebuild.amazonaws.com"},
    "Action":"sts:AssumeRole"
  }]
}
JSON

aws iam create-role \
  --role-name ${CODEBUILD_ROLE} \
  --assume-role-policy-document file:///tmp/codebuild-trust.json || true

cat > /tmp/codebuild-policy.json <<EOF2
{
  "Version":"2012-10-17",
  "Statement":[{
    "Effect":"Allow",
    "Action":[
      "logs:CreateLogGroup","logs:CreateLogStream","logs:PutLogEvents",
      "codecommit:GitPull",
      "ecr:GetAuthorizationToken","ecr:BatchCheckLayerAvailability",
      "ecr:InitiateLayerUpload","ecr:UploadLayerPart",
      "ecr:CompleteLayerUpload","ecr:PutImage",
      "s3:GetObject","s3:GetObjectVersion","s3:PutObject"
    ],
    "Resource":"*"
  }]
}
EOF2

aws iam put-role-policy \
  --role-name ${CODEBUILD_ROLE} \
  --policy-name demo-ecs-codebuild \
  --policy-document file:///tmp/codebuild-policy.json

sleep 10

aws codebuild create-project \
  --name ${CODEBUILD_PROJECT} \
  --source type=CODECOMMIT,location=https://git-codecommit.${AWS_REGION}.amazonaws.com.cn/v1/repos/${REPO_NAME} \
  --artifacts type=CODEPIPELINE \
  --environment type=LINUX_CONTAINER,image=aws/codebuild/standard:7.0,computeType=BUILD_GENERAL1_SMALL,privilegedMode=true \
  --service-role arn:aws-cn:iam::${ACCOUNT_ID}:role/${CODEBUILD_ROLE} \
  --region ${AWS_REGION} || true
```

**预期输出**：CodeBuild project 创建成功

### 5. 创建 CodePipeline Role

```bash
cat > /tmp/codepipeline-trust.json <<'JSON'
{
  "Version":"2012-10-17",
  "Statement":[{
    "Effect":"Allow",
    "Principal":{"Service":"codepipeline.amazonaws.com"},
    "Action":"sts:AssumeRole"
  }]
}
JSON

aws iam create-role \
  --role-name ${CODEPIPELINE_ROLE} \
  --assume-role-policy-document file:///tmp/codepipeline-trust.json || true

cat > /tmp/codepipeline-policy.json <<EOF2
{
  "Version":"2012-10-17",
  "Statement":[{
    "Effect":"Allow",
    "Action":[
      "codecommit:GetBranch","codecommit:GetCommit",
      "codecommit:UploadArchive","codecommit:GetUploadArchiveStatus","codecommit:CancelUploadArchive",
      "codebuild:BatchGetBuilds","codebuild:StartBuild",
      "ecs:DescribeServices","ecs:DescribeTaskDefinition","ecs:DescribeTasks",
      "ecs:ListTasks","ecs:RegisterTaskDefinition","ecs:UpdateService",
      "iam:PassRole",
      "s3:GetObject","s3:GetObjectVersion","s3:GetBucketVersioning","s3:PutObject"
    ],
    "Resource":"*"
  }]
}
EOF2

aws iam put-role-policy \
  --role-name ${CODEPIPELINE_ROLE} \
  --policy-name demo-ecs-codepipeline \
  --policy-document file:///tmp/codepipeline-policy.json
```

**预期输出**：CodePipeline role 创建成功

### 6. 创建 Pipeline

```bash
cat > /tmp/demo-ecs-pipeline.json <<EOF2
{
  "pipeline": {
    "name": "${PIPELINE_NAME}",
    "roleArn": "arn:aws-cn:iam::${ACCOUNT_ID}:role/${CODEPIPELINE_ROLE}",
    "artifactStore": {
      "type": "S3",
      "location": "${PIPELINE_BUCKET}"
    },
    "stages": [
      {
        "name": "Source",
        "actions": [{
          "name": "Source",
          "actionTypeId": {"category":"Source","owner":"AWS","provider":"CodeCommit","version":"1"},
          "runOrder": 1,
          "configuration": {
            "RepositoryName": "${REPO_NAME}",
            "BranchName": "main",
            "PollForSourceChanges": "true"
          },
          "outputArtifacts": [{"name":"SourceArtifact"}]
        }]
      },
      {
        "name": "Build",
        "actions": [{
          "name": "Build",
          "actionTypeId": {"category":"Build","owner":"AWS","provider":"CodeBuild","version":"1"},
          "runOrder": 1,
          "configuration": {"ProjectName":"${CODEBUILD_PROJECT}"},
          "inputArtifacts": [{"name":"SourceArtifact"}],
          "outputArtifacts": [{"name":"BuildArtifact"}]
        }]
      },
      {
        "name": "Deploy",
        "actions": [{
          "name": "DeployToECS",
          "actionTypeId": {"category":"Deploy","owner":"AWS","provider":"ECS","version":"1"},
          "runOrder": 1,
          "configuration": {
            "ClusterName": "${CLUSTER_NAME}",
            "ServiceName": "${SERVICE_NAME}",
            "FileName": "imagedefinitions.json"
          },
          "inputArtifacts": [{"name":"BuildArtifact"}]
        }]
      }
    ],
    "version": 1
  }
}
EOF2

aws codepipeline create-pipeline \
  --cli-input-json file:///tmp/demo-ecs-pipeline.json \
  --region ${AWS_REGION}
```

**预期输出**：Pipeline 创建成功

### 7. 验证首次发布

等待 Pipeline 首次运行（约 3-5 分钟）：

```bash
for i in $(seq 1 20); do
  STATUS=$(aws codepipeline get-pipeline-state \
    --name ${PIPELINE_NAME} \
    --region ${AWS_REGION} \
    --query 'stageStates[-1].actionStates[0].latestExecution.status' --output text 2>/dev/null)
  echo "Deploy status: ${STATUS}"
  [ "${STATUS}" = "Succeeded" ] && break
  sleep 30
done

export ALB_DNS=$(aws elbv2 describe-load-balancers \
  --names demo-ecs-alb \
  --query 'LoadBalancers[0].DNSName' --output text)

curl -m5 http://${ALB_DNS}:8080
```

**预期输出**：Pipeline 成功；ALB 返回"ECS pipeline"页面

### 8. 触发第二次发布

```bash
cd /tmp/${REPO_NAME}
sed -i 's/ECS pipeline/ECS pipeline updated/' Dockerfile
git add Dockerfile
git commit -m "update app message"
git push origin main

echo "等待 Pipeline 触发..."
sleep 30
aws codepipeline get-pipeline-state --name ${PIPELINE_NAME} --region ${AWS_REGION} \
  --query 'stageStates[].{stage:stageName,status:latestExecution.status}'
```

**预期输出**：Pipeline 再次触发，Deploy 阶段最终 Succeeded

---

## 验收标准

完成本实验后，你应当能够：
- [ ] Pipeline `demo-ecs-pipeline` 已创建，三个阶段（Source/Build/Deploy）均已配置
- [ ] CodeBuild 项目 `demo-ecs-build` 存在
- [ ] CodeCommit 仓库 `demo-ecs-app` 有代码
- [ ] Pipeline 最近一次 Deploy 阶段状态为 Succeeded
- [ ] `curl http://<ALB_DNS>:8080` 返回最新版本构建内容

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws codepipeline get-pipeline --name demo-ecs-pipeline --region cn-northwest-1 --query 'pipeline.name' --output text` | `demo-ecs-pipeline` |
| 2 | `aws codebuild batch-get-projects --names demo-ecs-build --region cn-northwest-1 --query 'projects[0].name' --output text` | `demo-ecs-build` |
| 3 | `aws codecommit get-repository --repository-name demo-ecs-app --region cn-northwest-1 --query 'repositoryMetadata.repositoryName' --output text` | `demo-ecs-app` |
| 4 | `aws codepipeline get-pipeline-state --name demo-ecs-pipeline --region cn-northwest-1 --query 'stageStates[-1].actionStates[0].latestExecution.status' --output text` | `Succeeded` |

---

## 实验总结

本实验建立了"代码推送即部署"的完整 CI/CD 流水线，buildspec.yml 中通过动态 IMAGE_TAG（时间戳格式）避免镜像 tag 冲突，`imagedefinitions.json` 是 CodePipeline ECS 部署 action 的标准接口文件。CodeBuild 需要 ECR 推送权限和 S3 artifact 权限，CodePipeline 需要 `iam:PassRole` 才能代入各 stage 角色。下一个 Demo（Demo11）将探索 EFS 持久化存储，满足有状态容器的数据持久化需求。

---

## 清理

```bash
source /tmp/demo-ecs.env

aws codepipeline delete-pipeline --name ${PIPELINE_NAME} --region ${AWS_REGION}
aws codebuild delete-project --name ${CODEBUILD_PROJECT} --region ${AWS_REGION}
aws codecommit delete-repository --repository-name ${REPO_NAME} --region ${AWS_REGION}

aws s3 rm s3://${PIPELINE_BUCKET} --recursive
aws s3api delete-bucket --bucket ${PIPELINE_BUCKET} --region ${AWS_REGION}

aws iam delete-role-policy --role-name ${CODEBUILD_ROLE} --policy-name demo-ecs-codebuild
aws iam delete-role --role-name ${CODEBUILD_ROLE}
aws iam delete-role-policy --role-name ${CODEPIPELINE_ROLE} --policy-name demo-ecs-codepipeline
aws iam delete-role --role-name ${CODEPIPELINE_ROLE}
```
