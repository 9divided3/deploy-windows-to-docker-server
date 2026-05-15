---
name: deploy-windows-to-docker-server
description: 在 Windows 无法安装 Docker、服务器无法访问 GitHub 的情况下，将本地项目部署到 Docker 服务器的通用流程。
---

# Windows 项目部署到 Docker 服务器

## 适用场景

- 本地 Windows 无法安装 Docker Desktop
- 服务器网络受限，无法访问 GitHub（拉取代码）
- 项目已具备 Dockerfile 和 docker-compose.yml

## 部署流程

### 第一步：本地打包项目

排除无需上传的目录（node_modules、.git、构建产物等），打包为 tar.gz：

如果项目有 `.gitignore` 且配置完善，推荐：

```powershell
cd 项目目录
git archive --format=tar.gz --output=project.tar.gz HEAD
```

如果没有 `.gitignore` 或未纳入 git 管理，手动指定排除项：

```powershell
tar -czf project.tar.gz --exclude=node_modules --exclude=.git --exclude=dist --exclude=build .
```

### 第二步：上传到服务器

```powershell
scp project.tar.gz 用户名@服务器IP:~/
```

### 第三步：服务器解压与配置

SSH 登录服务器后：

```bash
mkdir 项目名 && cd 项目名
tar -xzf ~/project.tar.gz
```

检查项目是否有 `.env.example`，复制并填写环境变量：

```bash
cp .env.example .env
# 编辑 .env，修改密钥等敏感配置
```

如果项目没有 `.env.example`，根据 docker-compose.yml 中引用的变量手动创建：

```bash
echo "VAR_NAME=value" > .env
```

### 第四步：启动容器

```bash
sudo docker compose up -d
```

### 第五步：验证

```bash
# 检查容器状态（状态应为 Up，不是 Restarting）
sudo docker ps | grep 容器名

# 测试服务是否响应
curl http://localhost:端口/health  # 或服务根路径
```

外网访问需要在云控制台安全组中放行对应端口。

## 常见问题排查

| 现象 | 原因 | 解决 |
|------|------|------|
| 容器状态 `Restarting` | 应用内部报错 | `sudo docker logs 容器名` 查看日志 |
| `EACCES: permission denied, mkdir` | 挂载目录属于宿主机用户，容器内无权限 | `sudo chown -R 容器uid:容器gid 挂载目录` |
| `Connection refused` | 端口未放行或服务未启动 | 先 `curl localhost:端口` 确认本地通，再检查安全组 |
| `Connection timed out` 访问外部 API | 服务器网络受限 | 配置代理或使用宿主机工具转发 |
| 证书错误 `unable to verify certificate` | 代理/加速工具使用自签证书 | 设环境变量 `NODE_TLS_REJECT_UNAUTHORIZED=0` |
| `docker: command not found` | 服务器未安装 Docker | `curl -fsSL https://get.docker.com \| sudo sh` |

## 后续更新流程

本地修改代码后，重新打包上传并重建容器：

```powershell
# 本地
git archive --format=tar.gz --output=project.tar.gz HEAD
scp project.tar.gz 用户名@服务器IP:~/
```

```bash
# 服务器
cd ~/项目名
tar -xzf ~/project.tar.gz            # 覆盖旧文件
sudo docker compose down             # 停止旧容器
sudo docker compose up -d --build    # 重新构建并启动
```

## Dockerfile 检查清单

部署前确认 Dockerfile 覆盖以下要点，避免上传后反复调试：

- [ ] 多阶段构建（前端构建 + 生产镜像分离），减小体积
- [ ] 使用非 root 用户运行（`RUN adduser && USER`）
- [ ] 持久化目录（数据库、上传文件）通过 `VOLUME` 或 `docker-compose volumes` 挂载
- [ ] 健康检查端点（`/health`）+ `HEALTHCHECK` 指令
- [ ] 必要的环境变量在 `docker-compose.yml` 中声明，敏感值通过 `.env` 注入
- [ ] `.dockerignore` 排除 node_modules、.git、构建产物
- [ ] CORS 生产环境关闭或限制来源
- [ ] 请求体大小限制（`express.json({ limit })` 或 nginx `client_max_body_size`）
