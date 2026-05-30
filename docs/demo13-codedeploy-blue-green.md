# Demo13 — CodeDeploy 蓝绿发布（可选）

## 实验简介

本实验使用 AWS CodeDeploy 为 ECS 服务实现蓝绿发布（Blue/Green Deployment），通过 Blue 和 Green 两个 Target Group 切换流量，在新版本（Green）验证通过后将流量从旧版本（Blue）切换过来，实现零停机部署。

**实验目标：**
- 掌握 CODE_DEPLOY controller 类型的 ECS Service 创建和双 Target Group 配置
- 理解 AppSpec 文件格式和 create-deployment API 的调用方式
- 能够观察并验证蓝绿发布的流量切换过程（Blue→Green）

**实验流程：**
1. 创建 Blue/Green Target Group 和独立 8081 端口 Listener
2. 创建 CodeDeploy Service Role（附加 AWSCodeDeployRoleForECS 策略）
3. 注册蓝绿发布专用 Task Definition
4. 创建 CODE_DEPLOY controller 类型的 ECS Service
5. 创建 CodeDeploy Application 和 Deployment Group（配置双 TG 和 Listener）
6. 准备 v2 Task Definition 和 AppSpec.yaml
7. 发起蓝绿发布，轮询等待 Succeeded
8. 验证流量已切换到 Green Target Group

**预计 AI 执行时长：** 13-16 分钟

---

## 前提条件

- **工具**：AWS CLI v2、jq、python3
- **权限**：ECS、ELBv2、CodeDeploy、IAM 权限
- **前提**：Demo01-Demo04 已完成；已有 ALB、ECR 镜像和 ECS cluster
- **说明**：本 Demo 创建独立 ECS service、两个 target group 和 CodeDeploy deployment group，不影响主线 Demo03 的 `demo-ecs-web` service
- **中国区注意**：蓝绿发布独立 Listener 使用 8081 端口（与主线 Demo04 的 8080 并列），避免备案要求导致的 80/443 受限；CodeDeploy service role ARN 使用 `arn:aws-cn:`，managed policy 使用 `arn:aws-cn:iam::aws:policy/AWSCodeDeployRoleForECS`
- **预计 AI 执行时长**：13-16 分钟

---

## 步骤

### 1. 设置环境变量

```bash
source /tmp/demo-ecs.env
export BG_SERVICE=demo-ecs-bg
export BG_TASK_FAMILY=demo-ecs-bg
export BG_CONTAINER=web
export BG_APP=demo-ecs-codedeploy
export BG_DG=demo-ecs-dg
export BLUE_TG_NAME=demo-ecs-blue
export GREEN_TG_NAME=demo-ecs-green
export BG_LISTENER_PORT=8081
export IMAGE_URI=${IMAGE_URI_STABLE:-${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com.cn/${ECR_REPO}:stable}
```

### 2. 创建 Blue / Green Target Group 和 Listener

```bash
export BLUE_TG_ARN=$(aws elbv2 create-target-group \
  --name ${BLUE_TG_NAME} \
  --protocol HTTP \
  --port 80 \
  --target-type ip \
  --vpc-id ${VPC_ID} \
  --health-check-path / \
  --matcher HttpCode=200 \
  --tags Key=Project,Value=${PROJECT_TAG} Key=Demo,Value=Demo13 Key=Owner,Value=${OWNER_TAG} \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

export GREEN_TG_ARN=$(aws elbv2 create-target-group \
  --name ${GREEN_TG_NAME} \
  --protocol HTTP \
  --port 80 \
  --target-type ip \
  --vpc-id ${VPC_ID} \
  --health-check-path / \
  --matcher HttpCode=200 \
  --tags Key=Project,Value=${PROJECT_TAG} Key=Demo,Value=Demo13 Key=Owner,Value=${OWNER_TAG} \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

export BG_LISTENER_ARN=$(aws elbv2 create-listener \
  --load-balancer-arn ${ALB_ARN} \
  --protocol HTTP \
  --port ${BG_LISTENER_PORT} \
  --default-actions Type=forward,TargetGroupArn=${BLUE_TG_ARN} \
  --query 'Listeners[0].ListenerArn' --output text)

# ALB SG 需允许 8081 端口公网入站（Demo01 只开放了 8080）
aws ec2 authorize-security-group-ingress \
  --group-id ${ALB_SG_ID} \
  --protocol tcp \
  --port 8081 \
  --cidr 0.0.0.0/0 2>/dev/null || true
```

**预期输出**：Blue TG ARN、Green TG ARN、Listener ARN 均有值；8081 入站规则添加成功

### 3. 创建 CodeDeploy Service Role

```bash
cat > /tmp/demo13-codedeploy-trust.json <<'JSON'
{
  "Version":"2012-10-17",
  "Statement":[{
    "Effect":"Allow",
    "Principal":{"Service":"codedeploy.amazonaws.com"},
    "Action":"sts:AssumeRole"
  }]
}
JSON

aws iam create-role \
  --role-name demo-ecs-codedeploy-role \
  --assume-role-policy-document file:///tmp/demo13-codedeploy-trust.json || true

aws iam attach-role-policy \
  --role-name demo-ecs-codedeploy-role \
  --policy-arn arn:aws-cn:iam::aws:policy/AWSCodeDeployRoleForECS

sleep 10
```

**预期输出**：Role 创建成功，policy 附加成功

### 4. 注册蓝绿 Task Definition

```bash
cat > /tmp/demo13-bg-task.json <<EOF2
{
  "family": "${BG_TASK_FAMILY}",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws-cn:iam::${ACCOUNT_ID}:role/demo-ecs-task-execution-role",
  "taskRoleArn": "arn:aws-cn:iam::${ACCOUNT_ID}:role/demo-ecs-task-role",
  "containerDefinitions": [{
    "name": "${BG_CONTAINER}",
    "image": "${IMAGE_URI}",
    "essential": true,
    "portMappings": [{"containerPort":80,"protocol":"tcp"}],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "${LOG_GROUP}",
        "awslogs-region": "${AWS_REGION}",
        "awslogs-stream-prefix": "bg"
      }
    }
  }]
}
EOF2

export BG_TASK_DEF_ARN=$(aws ecs register-task-definition \
  --cli-input-json file:///tmp/demo13-bg-task.json \
  --query 'taskDefinition.taskDefinitionArn' --output text)

echo "BG Task Definition ARN: ${BG_TASK_DEF_ARN}"
```

**预期输出**：Task definition ARN 输出

### 5. 创建 CODE_DEPLOY Controller 的 ECS Service

```bash
aws ecs create-service \
  --cluster ${CLUSTER_NAME} \
  --service-name ${BG_SERVICE} \
  --task-definition ${BG_TASK_DEF_ARN} \
  --desired-count 1 \
  --deployment-controller type=CODE_DEPLOY \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[${PUBLIC_SUBNET_1},${PUBLIC_SUBNET_2}],securityGroups=[${TASK_SG_ID}],assignPublicIp=ENABLED}" \
  --load-balancers targetGroupArn=${BLUE_TG_ARN},containerName=${BG_CONTAINER},containerPort=80 \
  --health-check-grace-period-seconds 60

aws ecs wait services-stable --cluster ${CLUSTER_NAME} --services ${BG_SERVICE}
```

**预期输出**：Service 稳定，Blue target group 有健康 target

### 6. 创建 CodeDeploy Application 和 Deployment Group

```bash
aws deploy create-application \
  --application-name ${BG_APP} \
  --compute-platform ECS || true

aws deploy create-deployment-group \
  --application-name ${BG_APP} \
  --deployment-group-name ${BG_DG} \
  --service-role-arn arn:aws-cn:iam::${ACCOUNT_ID}:role/demo-ecs-codedeploy-role \
  --deployment-config-name CodeDeployDefault.ECSAllAtOnce \
  --ecs-services serviceName=${BG_SERVICE},clusterName=${CLUSTER_NAME} \
  --load-balancer-info "targetGroupPairInfoList=[{targetGroups=[{name=${BLUE_TG_NAME}},{name=${GREEN_TG_NAME}}],prodTrafficRoute={listenerArns=[${BG_LISTENER_ARN}]}}]" \
  --blue-green-deployment-configuration "terminateBlueInstancesOnDeploymentSuccess={action=TERMINATE,terminationWaitTimeInMinutes=5},deploymentReadyOption={actionOnTimeout=CONTINUE_DEPLOYMENT}" \
  --deployment-style deploymentType=BLUE_GREEN,deploymentOption=WITH_TRAFFIC_CONTROL
```

**预期输出**：Deployment group 创建成功

### 7. 准备新版本 Task Definition 和 AppSpec

```bash
export IMAGE_URI_V2=${IMAGE_URI_V2:-${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com.cn/${ECR_REPO}:v2}

jq --arg IMAGE "${IMAGE_URI_V2}" \
  'del(.taskDefinitionArn,.revision,.status,.requiresAttributes,.compatibilities,.registeredAt,.registeredBy) | .containerDefinitions[0].image = $IMAGE' \
  /tmp/demo13-bg-task.json > /tmp/demo13-bg-task-v2.json

export BG_TASK_DEF_V2_ARN=$(aws ecs register-task-definition \
  --cli-input-json file:///tmp/demo13-bg-task-v2.json \
  --query 'taskDefinition.taskDefinitionArn' --output text)

cat > /tmp/demo13-appspec.json <<EOF2
{
  "version": 0.0,
  "Resources": [{
    "TargetService": {
      "Type": "AWS::ECS::Service",
      "Properties": {
        "TaskDefinition": "${BG_TASK_DEF_V2_ARN}",
        "LoadBalancerInfo": {
          "ContainerName": "${BG_CONTAINER}",
          "ContainerPort": 80
        }
      }
    }
  }]
}
EOF2
```

> ⚠️ **中国区限制**：AppSpec 使用 YAML 格式并通过 inline string 传入时会报 `INVALID_REVISION`。改用 JSON 格式 + `file://` 路径传入可规避此问题。

**预期输出**：v2 task definition ARN 输出，appspec.json 生成

### 8. 发起蓝绿发布

```bash
export DEPLOYMENT_ID=$(aws deploy create-deployment \
  --application-name ${BG_APP} \
  --deployment-group-name ${BG_DG} \
  --revision "revisionType=AppSpecContent,appSpecContent={content=$(cat /tmp/demo13-appspec.json)}" \
  --query 'deploymentId' --output text)

echo "Deployment ID: ${DEPLOYMENT_ID}"
```

轮询等待成功：

```bash
while true; do
  STATUS=$(aws deploy get-deployment \
    --deployment-id ${DEPLOYMENT_ID} \
    --query 'deploymentInfo.status' --output text)
  echo "$(date -u +%H:%M:%S) status: ${STATUS}"
  [ "${STATUS}" = "Succeeded" ] && break
  [ "${STATUS}" = "Failed" ] && { echo "发布失败，请检查 CodeDeploy 日志"; break; }
  sleep 20
done
```

**预期输出**：deployment status 最终变为 Succeeded（约 5-10 分钟）

### 9. 验证流量切换

```bash
curl -m5 http://${ALB_DNS}:${BG_LISTENER_PORT}

aws elbv2 describe-target-health \
  --target-group-arn ${BLUE_TG_ARN} \
  --query 'TargetHealthDescriptions[].{target:Target.Id,state:TargetHealth.State}'

aws elbv2 describe-target-health \
  --target-group-arn ${GREEN_TG_ARN} \
  --query 'TargetHealthDescriptions[].{target:Target.Id,state:TargetHealth.State}'
```

**预期输出**：返回 v2 页面（"ECS China QuickStart v2"）；流量已切换到 Green target group

---

## 验收标准

完成本实验后，你应当能够：
- [ ] CodeDeploy Application `demo-ecs-codedeploy` 已创建
- [ ] ECS Service `demo-ecs-bg` 的 deploymentController 类型为 `CODE_DEPLOY`
- [ ] 最近一次 Deployment 状态为 `Succeeded`
- [ ] `curl http://<ALB_DNS>:8081` 返回 v2 版本内容（"ECS China QuickStart v2"）
- [ ] Green Target Group 有 healthy target，Blue Target Group 无活跃 target

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws deploy get-application --application-name demo-ecs-codedeploy --region cn-northwest-1 --query 'application.applicationName' --output text` | `demo-ecs-codedeploy` |
| 2 | `aws ecs describe-services --cluster demo-ecs --services demo-ecs-bg --region cn-northwest-1 --query 'services[0].deploymentController.type' --output text` | `CODE_DEPLOY` |
| 3 | `aws deploy get-deployment --deployment-id $(aws deploy list-deployments --application-name demo-ecs-codedeploy --deployment-group-name demo-ecs-dg --region cn-northwest-1 --query 'deployments[0]' --output text) --region cn-northwest-1 --query 'deploymentInfo.status' --output text` | `Succeeded` |

---

## 实验总结

本实验实现了 ECS 蓝绿发布的完整流程，新版本（Green）在独立环境中启动并通过健康检查后，流量才从旧版本（Blue）切换，切换失败可立即回滚到 Blue。`terminateBlueInstancesOnDeploymentSuccess.terminationWaitTimeInMinutes=5` 控制 Blue 环境在流量切换后的保留时间，期间可作为快速回滚的后备。蓝绿发布比滚动发布需要更多资源（同时运行两套环境），适用于对变更风险要求极高的核心服务。下一个 Demo（Demo14，可选）将进行成本审计和资源清理，帮助识别并清除所有实验创建的持续计费资源。

---

## 清理

```bash
source /tmp/demo-ecs.env

aws deploy delete-deployment-group --application-name ${BG_APP} --deployment-group-name ${BG_DG}
aws deploy delete-application --application-name ${BG_APP}

aws ecs update-service --cluster ${CLUSTER_NAME} --service ${BG_SERVICE} --desired-count 0
aws ecs delete-service --cluster ${CLUSTER_NAME} --service ${BG_SERVICE}

aws elbv2 delete-listener --listener-arn ${BG_LISTENER_ARN}
aws elbv2 delete-target-group --target-group-arn ${BLUE_TG_ARN}
aws elbv2 delete-target-group --target-group-arn ${GREEN_TG_ARN}

aws iam detach-role-policy --role-name demo-ecs-codedeploy-role --policy-arn arn:aws-cn:iam::aws:policy/AWSCodeDeployRoleForECS
aws iam delete-role --role-name demo-ecs-codedeploy-role
```
