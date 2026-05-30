# ECS China QuickStart
## AWS 中国区 ECS 动手实验集

基于 ECS Fargate Linux · 宁夏区（cn-northwest-1）

---

## 与 Global QuickStart 的差异和运行前检查

本 QuickStart 面向 AWS 中国区，不能完全照搬 Global 区域的 ARN、endpoint、镜像源和端口习惯。开始前建议确认以下事项，避免 Demo 执行到一半被分区差异或网络限制打断：

1. 使用 `cn-northwest-1`，IAM ARN 使用 `arn:aws-cn:`，服务 endpoint 使用 `.amazonaws.com.cn`；ECR registry 格式为 `<account>.dkr.ecr.cn-northwest-1.amazonaws.com.cn`。
2. `public.ecr.aws` 在中国区不稳定；Dockerfile FROM 和 Fargate task definition 中的基础镜像（nginx、busybox）统一改用共享镜像仓库 `048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn`，`docker build` 前需额外登录该仓库。
3. `aws ecr start-image-scan` 在中国区返回 `ValidationException: This feature is disabled`；Demo02 仅依赖 `scanOnPush=true` 触发扫描，扫描状态固定为 `ACTIVE`，直接查询 `describe-image-scan-findings` 获取漏洞统计。
4. 公网 ALB Listener 使用 8080，Demo13 蓝绿发布额外 Listener 使用 8081，避免 80/443 备案限制影响验证；Fargate + ALB Target Group 必须指定 `--target-type ip`。
5. 创建 S3 bucket 必须携带 `--create-bucket-configuration LocationConstraint=cn-northwest-1`，否则报错；CodeCommit clone URL 格式为 `https://git-codecommit.cn-northwest-1.amazonaws.com.cn/v1/repos/<repo>`，推送前需配置 credential helper。
6. 创建 IAM Role 后需 `sleep 10` 再使用，避免 IAM 传播延迟导致权限未生效；CodeDeploy 蓝绿发布的 Blue/Green Target Group 必须在 ECS Service 创建前就已存在。
7. Demo08 创建 6 个 Interface VPC Endpoint 按小时计费，实验结束后及时删除；Demo09（ECS Exec）前需在操作机安装 `session-manager-plugin`；Demo11 清理时必须先删 Mount Target 再删 EFS File System。

---

## Demo 列表

建议先完成 **基础**，再按需要选择 **进阶**。基础覆盖 ECS Fargate 从环境准备、镜像发布、服务部署、发布策略、权限安全、弹性伸缩到 CI/CD 的最小完整闭环；进阶用于补充存储、服务通信、蓝绿发布和成本治理能力。

## 基础

### 1. 基础资源、镜像与第一个服务

* [Demo01 — 准备实验环境与基础资源](docs/demo01-environment-setup.md)
* [Demo02 — ECR 镜像构建与发布](docs/demo02-ecr-build-push.md)
* [Demo03 — 部署第一个 Fargate 服务](docs/demo03-first-fargate-service.md)
* [Demo04 — 使用 ALB 暴露服务](docs/demo04-alb-expose-service.md)

### 2. 发布、权限与弹性

* [Demo05 — 滚动发布、回滚与故障排查](docs/demo05-rolling-deploy-rollback.md)
* [Demo06 — 任务配置：Secrets 与 Task Role](docs/demo06-secrets-task-role.md)
* [Demo07 — Service Auto Scaling](docs/demo07-service-autoscaling.md)

### 3. 私有网络、可观测性与 CI/CD

* [Demo08 — 私有子网与 VPC Endpoint](docs/demo08-private-subnet-vpc-endpoint.md)
* [Demo09 — CloudWatch 日志、指标与 ECS Exec](docs/demo09-cloudwatch-logs-ecs-exec.md)
* [Demo10 — CodePipeline 自动部署 ECS](docs/demo10-codepipeline-cicd.md)

## 进阶

### 4. 存储与服务通信

* [Demo11 — EFS 持久化存储](docs/demo11-efs-persistent-storage.md)
* [Demo12 — Service Discovery 与 Service Connect](docs/demo12-service-discovery.md)

### 5. 发布治理与成本治理

* [Demo13 — CodeDeploy 蓝绿发布（可选）](docs/demo13-codedeploy-blue-green.md)
* [Demo14 — 成本治理与清理审计（可选）](docs/demo14-cost-audit-cleanup.md)

## 使用方式

1. 在此目录下打开 Claude Code，`CLAUDE.md` 自动加载中国区 ECS 配置
2. 按顺序打开 `docs/demoXX-*.md`，将内容粘贴到对话框，由 AI 自主执行
3. 每个 Demo 末尾均有**清理**步骤，实验结束后执行以避免持续计费

## 环境要求

| 工具 | 版本 |
|------|------|
| AWS CLI | 2.x |
| Docker | 24.x+ |
| jq | latest |
| git | latest |
| session-manager-plugin | Demo09（ECS Exec）前安装 |

操作机建议使用 Amazon Linux 2023 EC2，绑定具备 ECS、EC2、IAM、ECR、Logs、ELBv2、Application Auto Scaling、Secrets Manager、SSM、CodeBuild、CodePipeline、CodeCommit、EFS、Service Discovery、CodeDeploy 操作权限的 IAM Role。

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 免责声明

本 Workshop 仅供学习和测试用途。执行过程中会创建 AWS 资源并产生费用，请在实验完成后及时清理资源。作者不对因使用本 Workshop 产生的任何费用或损失承担责任。所有命令和配置仅作为示例参考，生产环境使用前请根据实际需求进行安全评估和调整。
