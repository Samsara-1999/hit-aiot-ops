# HIT-AIOT-OPS

轻量 GPU 集群运维平台：保留 SSH 使用习惯，后台完成监控、计费、配额控制、账号映射与管理。

## 功能概览

- **节点 Agent（Go）**：每分钟采集 GPU/CPU 进程并上报控制器
- **控制器（Go + Gin + PostgreSQL）**：落库、计费、限制动作下发、管理 API
- **Web 管理端（Vue3）**：管理员与普通用户分角色界面
- **用户能力**：注册、登录、找回密码、修改密码、查询个人余额/用量、管理个人服务器账号映射
- **管理员能力**：运营看板、节点状态、价格配置、注册审核、账号映射管理、SSH 白名单、邮件配置与测试发送

---

## 🚀 快速开始（本机）

### 🧰 0) 安装依赖（Ubuntu 22.04，清华源优先，固定版本）

> 建议先配置 apt 为清华源，再安装基础依赖；以下版本为本项目推荐固定版本。

```bash
cd /home/baojh/hit-aiot-ops
bash scripts/install_deps_ubuntu2204.sh
```

可选参数（示例）：

```bash
# 跳过 Docker
INSTALL_DOCKER=0 bash scripts/install_deps_ubuntu2204.sh

# 指定版本
GO_VERSION=1.22.5 NODE_MAJOR=20 PNPM_VERSION=10.28.2 bash scripts/install_deps_ubuntu2204.sh
```

脚本等价于下方手动步骤，若你想逐条执行可继续参考：

```bash
# 0.1 切换 apt 清华源（Ubuntu 22.04 / jammy）
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak.$(date +%s)
sudo tee /etc/apt/sources.list >/dev/null <<'EOF'
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-security main restricted universe multiverse
EOF

# 0.2 基础依赖
sudo apt-get update
sudo apt-get install -y --no-install-recommends \
  ca-certificates curl wget git jq build-essential docker.io docker-compose-plugin

# 0.3 安装 Go 1.22.5（清华失败自动切阿里/腾讯）
cd /tmp
rm -f go.tgz
wget -O go.tgz https://mirrors.tuna.tsinghua.edu.cn/golang/go1.22.5.linux-amd64.tar.gz \
|| wget -O go.tgz https://mirrors.aliyun.com/golang/go1.22.5.linux-amd64.tar.gz \
|| wget -O go.tgz https://mirrors.cloud.tencent.com/golang/go1.22.5.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf /tmp/go.tgz
echo 'export PATH=/usr/local/go/bin:$PATH' | sudo tee /etc/profile.d/go.sh >/dev/null
export PATH=/usr/local/go/bin:$PATH
grep -q '/usr/local/go/bin' ~/.bashrc || echo 'export PATH=/usr/local/go/bin:$PATH' >> ~/.bashrc
hash -r
go version   # 期望：go1.22.5

# 0.4 安装 Node 20 + pnpm 10.28.2
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
sudo corepack enable
corepack prepare pnpm@10.28.2 --activate
node -v      # 期望：v20.x
pnpm -v      # 期望：10.28.2

# 0.5 Go 网络建议（国内）
go env -w GOPROXY=https://goproxy.cn,direct
go env -w GOSUMDB=off
```

### 🗄️ 1) 启动 PostgreSQL

```bash
cd /home/baojh/hit-aiot-ops
docker-compose up -d

docker-compose ps -a
docker-compose logs --tail=200 postgres
```

默认数据库：`gpuops`，账号密码：`gpuops/gpuops`，端口：`5432`。

### 🧠 2) 启动控制器

```bash
cd /home/baojh/hit-aiot-ops/controller
go run . --config ../config/controller.yaml
```

健康检查：

```bash
curl -s http://127.0.0.1:8000/healthz
```

### 🖥️ 3) 构建前端（首次或前端改动后）

```bash
cd /home/baojh/hit-aiot-ops/web
pnpm install
pnpm build
```

说明：控制器只托管 `web/dist`，前端改动后需重新 `pnpm build`，然后重启控制器。

### 🔐 4) 初始化管理员账号（仅首次）

```bash
# admin_token 请从 config/controller.yaml 读取
curl -fsS -H "Authorization: Bearer <admin_token>" \
  -H "Content-Type: application/json" \
  -X POST http://127.0.0.1:8000/api/admin/bootstrap \
  -d '{"username":"admin","password":"ChangeMe_123456"}'
```

登录地址：`http://127.0.0.1:8000/login`

### 🤖 5) 本机模拟启动 Agent

```bash
cd /home/baojh/hit-aiot-ops/node-agent
NODE_ID=60000 \
CONTROLLER_URL=http://192.168.1.244:8000 \
AGENT_TOKEN=<agent_token> \
go run .
```

`AGENT_TOKEN` 必须与 `config/controller.yaml` 的 `agent_token` 一致。

计算节点本地安装时，推荐用“一条命令”启动（避免代理导致下载超时）：

```bash
cd /home/<用户名>/hit-aiot-ops/node-agent
env -u http_proxy -u https_proxy -u HTTP_PROXY -u HTTPS_PROXY -u ALL_PROXY -u all_proxy \
GOPROXY=https://goproxy.cn,direct \
GOSUMDB=sum.golang.google.cn \
GO111MODULE=on \
NODE_ID=60001 \
CONTROLLER_URL=http://192.168.1.244:8000 \
AGENT_TOKEN=<agent_token> \
go run .
```

---

## 🔁 日常启动（开发环境）

```bash
# 1) 控制端：数据库
cd /home/baojh/hit-aiot-ops
docker-compose up -d

# 2) 控制端：控制器
cd /home/baojh/hit-aiot-ops/controller
go run . --config ../config/controller.yaml

# 3) 节点端：Agent（如未用 systemd 托管）
cd /home/baojh/hit-aiot-ops/node-agent
NODE_ID=60000 CONTROLLER_URL=http://127.0.0.1:8000 AGENT_TOKEN=<agent_token> go run .
```

> `pnpm build` 不需要每次开机执行，只有前端代码变更后需要。

---

## 🧩 计算节点部署（Ubuntu 22.04，支持 sudo）

### 🎯 单点部署方案（推荐给手工安装场景）

适用场景：你已经 SSH 到某一台计算节点，想本地安装并启动，不依赖批量远程脚本。

```bash
cd /home/<用户名>/hit-aiot-ops
NODE_ID=60001 \
CONTROLLER_URL=http://<控制器IP>:8000 \
AGENT_TOKEN=<config/controller.yaml里的agent_token> \
bash scripts/install_agent_local.sh
```

说明：
- 该脚本会自动处理：依赖安装、Go 源配置、代理清理、本地编译、systemd 安装与启动。
- 默认会先做控制器健康检查（`/healthz`），失败会阻止安装并给出错误。

如需跳过依赖安装或健康检查：

```bash
INSTALL_DEPS=0 SKIP_CONTROLLER_HEALTHCHECK=1 \
NODE_ID=60001 CONTROLLER_URL=http://<控制器IP>:8000 AGENT_TOKEN=<agent_token> \
bash scripts/install_agent_local.sh
```

### 🛠️ 1) 在控制器机编译 agent

```bash
cd /home/baojh/hit-aiot-ops/node-agent
go build -o node-agent .
```

### 🚚 2) 批量部署

```bash
cd /home/baojh/hit-aiot-ops

AGENT_BIN=./node-agent/node-agent \
AGENT_TOKEN='<agent_token>' \
CONTROLLER_URL='http://<controller-ip>:8000' \
SSH_USER=ubuntu \
INSTALL_PREREQS=1 \
INSTALL_GO=0 \
NODES='60000:192.168.1.104 60001:192.168.1.220' \
bash scripts/deploy_agent.sh
```

### 🛡️ 3) 可选：启用 SSH Guard（未登记限制登录）

```bash
ENABLE_SSH_GUARD=1 \
SSH_GUARD_EXCLUDE_USERS='root ubuntu' \
SSH_GUARD_FAIL_OPEN=1 \
AGENT_BIN=./node-agent/node-agent \
AGENT_TOKEN='<agent_token>' \
CONTROLLER_URL='http://<controller-ip>:8000' \
SSH_USER=ubuntu \
NODES='60000:192.168.1.104' \
bash scripts/deploy_agent.sh
```

### ✅ 4) 节点状态检查

```bash
sudo systemctl status gpu-node-agent
sudo journalctl -u gpu-node-agent -n 100 --no-pager
```

### 🚨 常见错误：`connect: connection refused`

示例：

```text
[node-agent] tick 异常：Post "http://192.168.1.244:8000/api/metrics": dial tcp 192.168.1.244:8000: connect: connection refused
```

这通常表示“控制器地址可达，但 8000 端口没有监听”。

按顺序检查：

1. 控制器进程是否在跑：

```bash
ps -ef | grep '[c]ontroller'
```

2. 控制器是否监听 8000 端口：

```bash
ss -lntp | grep 8000
```

3. `config/controller.yaml` 的监听地址是否对外开放。  
如果是 `listen_addr: "127.0.0.1:8000"`，外部节点无法连接，应改为：

```yaml
listen_addr: "0.0.0.0:8000"
```

4. 在节点上测试健康检查：

```bash
curl -sS http://192.168.1.244:8000/healthz
```

5. 检查防火墙/安全组是否放行 `8000/tcp`。

---

## 🧭 主要页面

- 登录：`/login`
- 用户注册：`/register`
- 找回密码：`/forgot-password`
- 管理员运营看板：`/admin/board`
- 节点状态：`/admin/nodes`
- 账号映射管理：`/admin/accounts`、`/user/accounts`
- SSH 白名单：`/admin/whitelist`
- 邮件设置与测试发送：`/admin/mail`

---

## 🔌 API 速查

- Agent 上报：`POST /api/metrics`
- 用户自助：`POST /api/auth/register`、`POST /api/auth/forgot-password`、`POST /api/auth/reset-password`
- 登录会话：`POST /api/auth/login`、`GET /api/auth/me`、`POST /api/auth/change-password`
- 用户查询：`GET /api/user/me/balance`、`GET /api/user/me/usage`
- 账号映射：
  - 用户：`GET/POST/PUT/DELETE /api/user/accounts`
  - 管理员：`GET/POST/PUT/DELETE /api/admin/accounts`
- 白名单：`GET/POST/DELETE /api/admin/whitelist`
- 运营统计：`GET /api/admin/stats/users`、`GET /api/admin/stats/monthly`、`GET /api/admin/stats/recharges`
- 邮件：`GET/POST /api/admin/mail/settings`、`POST /api/admin/mail/test`

完整字段说明见：`docs/api-reference.md`

---

## 🧪 测试与构建

```bash
# Go 测试（多模块）
go test ./controller/... ./node-agent/...

# 前端构建
cd web && pnpm build
```

---

## 📚 文档导航

- `docs/plan.md`：总体方案
- `docs/runbook.md`：上线运行手册
- `docs/admin-guide.md`：管理员手册
- `docs/user-guide.md`：用户手册
- `docs/go-live-checklist.md`：上线检查项

---

## 🗂️ 目录结构

```text
hit-aiot-ops/
├── controller/      # 控制器
├── node-agent/      # 节点 Agent
├── web/             # 前端
├── database/        # schema + migrations
├── scripts/         # 部署/运维脚本
├── tools/           # 用户侧工具
├── config/          # 配置
├── systemd/         # service 示例
└── docs/            # 文档
```
