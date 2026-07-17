# 架构文档

本仓库包含 14 个 Demo，这里不做全量架构图汇总，只对其中组件交互较复杂、值得可视化的 Demo 提供架构图；其余 Demo 请直接看对应的 `docs/demoXX-*.md`。

选取标准：跨多个 AWS 服务编排、存在中国区特有的网络/身份限制、或有明确的多步骤调用链路。据此选出 3 个 Demo：**Demo08（私有子网与 VPC Endpoint）**、**Demo10（CodePipeline CI/CD）**、**Demo13（CodeDeploy 蓝绿发布）**。

---

## Demo08 — 私有子网与 VPC Endpoint

Demo08 把 `demo-ecs-web` Service 从公有子网迁移到私有子网，为此需要建齐 6 个 Interface VPC Endpoint（`ecr.api`、`ecr.dkr`、`logs`、`secretsmanager`、`ssmmessages`、`kms`）加 1 个 S3 Gateway Endpoint。中国区的坑在于 **ECR 的两个 Endpoint 服务名前缀是 `cn.com.amazonaws`，其余四个是 `com.amazonaws`**，混用会报 `InvalidServiceName`；Endpoint 安全组要放行 Task 安全组到 443 端口，所有 Endpoint 变为 `available` 后才能把 Service 的 `assignPublicIp` 改为 `DISABLED` 并强制重新部署。

```mermaid
flowchart TB
  Op["实验操作者\n(AI / aws CLI)"]

  subgraph AWSCN["AWS cn-northwest-1"]
    subgraph VPC["VPC demo-ecs-vpc"]
      VPCE_SG["VPCE 安全组\n(允许 Task SG → 443)"]

      subgraph IF_ECR["Interface Endpoint\n(前缀 cn.com.amazonaws)"]
        EPEcrApi["ecr.api"]
        EPEcrDkr["ecr.dkr"]
      end

      subgraph IF_Other["Interface Endpoint\n(前缀 com.amazonaws)"]
        EPLogs["logs"]
        EPSM["secretsmanager"]
        EPSSM["ssmmessages"]
        EPKMS["kms"]
      end

      EPS3["S3 Gateway Endpoint\n(绑定私有路由表, 免费)"]

      subgraph PrivSub["私有子网 x2"]
        subgraph Cluster["ECS Cluster: demo-ecs"]
          WebSvc["Service: demo-ecs-web\n(assignPublicIp=DISABLED)"]
        end
      end
    end
    ALB["ALB :8080"]
  end

  Op -->|"1. create-security-group"| VPCE_SG
  Op -->|"2. create-vpc-endpoint x2\n(cn.com.amazonaws)"| EPEcrApi
  Op -->|"2. create-vpc-endpoint x2\n(cn.com.amazonaws)"| EPEcrDkr
  Op -->|"2. create-vpc-endpoint x4\n(com.amazonaws)"| EPLogs
  Op -->|"2. create-vpc-endpoint x4\n(com.amazonaws)"| EPSM
  Op -->|"2. create-vpc-endpoint x4\n(com.amazonaws)"| EPSSM
  Op -->|"2. create-vpc-endpoint x4\n(com.amazonaws)"| EPKMS
  Op -->|"3. create-vpc-endpoint (Gateway)"| EPS3
  Op -->|"4. 轮询直到全部 available"| VPC
  Op -->|"5. update-service\nassignPublicIp=DISABLED"| WebSvc

  VPCE_SG -.保护.-> IF_ECR
  VPCE_SG -.保护.-> IF_Other
  WebSvc -->|"拉镜像"| EPEcrApi
  WebSvc -->|"拉镜像层"| EPEcrDkr
  WebSvc -->|"写日志"| EPLogs
  WebSvc -->|"读密钥"| EPSM
  WebSvc -->|"ECS Exec"| EPSSM
  WebSvc -->|"KMS 解密"| EPKMS
  WebSvc -->|"S3 (若需要)"| EPS3
  ALB -->|"6. 验证仍可访问\n(无公网 IP)"| WebSvc
```

---

## Demo10 — CodePipeline 自动部署 ECS

Demo10 搭建 CodeCommit → CodeBuild → CodePipeline 三段式流水线。中国区差异集中在三处：CodeCommit clone URL 用 `.com.cn` 域名；`buildspec.yml` 的基础镜像来自共享镜像仓库而非 `public.ecr.aws`，CodeBuild 需登录两个 ECR registry；Service Role ARN 统一用 `arn:aws-cn:` 前缀。Demo 最后修改 Dockerfile 再次 `git push`，验证 Pipeline 的自动触发能力。

```mermaid
flowchart LR
  Dev["开发者/操作机"]

  subgraph AWSCN["AWS cn-northwest-1"]
    CC["CodeCommit: demo-ecs-app\n(git-codecommit...amazonaws.com.cn)"]
    S3["S3 Artifact Bucket\n(LocationConstraint=cn-northwest-1)"]
    Mirror["共享镜像仓库\n048912060910...amazonaws.com.cn\n(FROM 基础镜像)"]

    subgraph Pipeline["CodePipeline: demo-ecs-pipeline"]
      Source["Source Stage"]
      Build["Build Stage"]
      Deploy["Deploy Stage (ECS)"]
    end

    CB["CodeBuild: demo-ecs-build\n(buildspec.yml)"]
    ECR["ECR: demo-ecs-web\n(业务镜像仓库)"]
    WebSvc["ECS Service: demo-ecs-web"]
    ALB["ALB :8080"]
  end

  Dev -->|"git push (credential-helper)"| CC
  CC -->|"触发"| Source
  Source --> Build
  Build -->|"StartBuild"| CB
  CB -->|"1. 登录业务 ECR"| ECR
  CB -->|"2. 登录镜像源仓库"| Mirror
  CB -->|"docker build (FROM 镜像源)"| Mirror
  CB -->|"docker push"| ECR
  CB -->|"生成 imagedefinitions.json"| S3
  Build --> Deploy
  Deploy -->|"UpdateService"| WebSvc
  WebSvc --> ALB
```

---

## Demo13 — CodeDeploy 蓝绿发布

Demo13 用独立的 `demo-ecs-bg` Service（`deploymentController=CODE_DEPLOY`）演示蓝绿发布：生产 Listener `:8080` 保持不变，额外开一个 `:8081` 测试 Listener（规避中国区备案限制），分别指向 Blue/Green 两个 Target Group。提交 AppSpec 后 CodeDeploy 把新任务跑到 Green、先在测试端口验证，再把生产 Listener 切到 Green。

```mermaid
flowchart TB
  Op["实验操作者\n(AI / aws CLI)"]

  subgraph AWSCN["AWS cn-northwest-1"]
    subgraph ALB_["ALB demo-ecs-alb"]
      ProdListener["生产 Listener :8080"]
      TestListener["测试 Listener :8081"]
    end
    BlueTG["Target Group: demo-ecs-blue"]
    GreenTG["Target Group: demo-ecs-green"]

    subgraph Cluster["ECS Cluster: demo-ecs"]
      BGSvc["Service: demo-ecs-bg\n(deploymentController=CODE_DEPLOY)"]
      TaskV1["Task (Blue, stable 镜像)"]
      TaskV2["Task (Green, v2 镜像)"]
    end

    subgraph CD["CodeDeploy"]
      CDApp["Application: demo-ecs-codedeploy"]
      CDGroup["Deployment Group: demo-ecs-dg\n(arn:aws-cn IAM Role)"]
    end
  end

  Op -->|"1. create-target-group x2 + create-listener :8081"| BlueTG
  Op -->|"1. create-target-group x2 + create-listener :8081"| GreenTG
  Op -->|"2. create-role\nAWSCodeDeployRoleForECS"| CD
  Op -->|"4. create-service"| BGSvc
  BGSvc -->|"初始挂载"| BlueTG
  BlueTG --> TaskV1
  ProdListener --> BlueTG

  Op -->|"5. create-application +\ncreate-deployment-group\n(targetGroupPairInfoList)"| CDApp
  CDApp --> CDGroup
  CDGroup -.管控.-> BGSvc

  Op -->|"6. 注册 Task Definition v2"| TaskV2
  Op -->|"7. create-deployment (AppSpec)"| CDGroup
  CDGroup -->|"部署新任务到 Green"| GreenTG
  GreenTG --> TaskV2
  TestListener -.验证 Green.-> GreenTG
  CDGroup -->|"验证通过后切流量"| ProdListener
  ProdListener -.切换指向.-> GreenTG
  CDGroup -->|"延迟终止 Blue"| TaskV1
```
