# suibuff-northflank

[freebuff-proxy](https://github.com/trefeon/freebuff-proxy)（Go 编写的 OpenAI 兼容网关，把 Codebuff/FreeBuff CLI 协议转成 OpenAI API）的 **Northflank 适配版**。

上游源码**零改动**，所有适配都在新增文件里。连接 GitHub 仓库即可在 Northflank Sandbox（免费 $0/月，永不休眠）上部署。

- **上游版本**: [`trefeon/freebuff-proxy`](https://github.com/trefeon/freebuff-proxy) @ `e6b2b46` (2026-08-17)
- **平台**: [Northflank](https://northflank.com) Sandbox 免费档 — 2× services + 1× 数据库 + 2× cron，Always-on-compute 不休眠，美国区域可选（us-east/us-central/us-west 等）

---

## 改动点（相对上游）

| 文件 | 类型 | 说明 |
|---|---|---|
| `northflank.json` | 新增 | Northflank 模板蓝图：BuildService（Dockerfile 构建）+ DeploymentService（端口 3457，public HTTP） |
| `.env.northflank` | 新增 | 云上部署的环境变量清单（标了必设项） |
| `README.md`（本文件） | 替换 | 部署指南；上游 README 全文见 [trefeon/freebuff-proxy](https://github.com/trefeon/freebuff-proxy) |
| `Dockerfile` | 原样 | 上游自带，多阶段构建（golang:1.26-alpine → alpine:3.20，监听 3457，非 root 运行） |
| `cmd/` `internal/` `go.mod` 等 | 原样 | 上游代码，零改动 |
| `.gitignore` | 微调 | 放行 `!.env.northflank`（上游规则会忽略所有 `.env.*`） |

**关键环境变量**（部署时必须设置，详见 [.env.northflank](.env.northflank)）：

| 变量 | 值 | 为什么必须 |
|---|---|---|
| `LISTEN_ADDR` | `:3457` | 上游默认 `127.0.0.1:3457` 只监听回环，容器外（Northflank ingress）无法访问 |
| `AUTO_DISCOVER_TOKEN` | `false` | 云上没有 CLI 登录文件，关闭自动发现避免 bridge 模式意外变 pooled |
| `AUTH_TOKENS` | 留空 = bridge 模式 / 填 token = pooled 模式 | 二选一，见上游 README |
| `SAFE_MODE` | `true` | 防封号预设（保持默认） |
| `COST_MODE` | `free` | 保持 `free`，改其他值会走 PAID 路由导致 402 |

---

## 部署（二选一）

### 方式 A：模板导入（推荐，一次到位）

1. 注册/登录 [Northflank](https://app.northflank.com)（Sandbox 档免费，创建项目需绑卡验证，只验证不扣费）
2. Dashboard → **Create Project** → 选择 **Import** / **From template**，上传本仓库的 `northflank.json`
3. 模板会自动创建：构建（Dockerfile）+ 服务（端口 3457，public）+ 部署
4. 在服务页 **Configuration → Environment** 填入 [.env.northflank](.env.northflank) 中的变量（至少 `LISTEN_ADDR=:3457` + `AUTO_DISCOVER_TOKEN=false`）
5. 等部署完成，拿到 `https://<project>-<service>.code.run` 的 HTTPS URL

### 方式 B：UI 手动配置

1. Dashboard → **Create Project** → 命名（如 `suibuff`）→ 创建
2. 项目内 → **+ New Service** → **Connect a Git repository** → 连接 `fskanokano/suibuff-northflank`
3. Build method 选 **Dockerfile**（自动使用仓库根目录的 `Dockerfile`）
4. **Port** 填 `3457`
5. **Environment** 填必设变量（同上）
6. Deploy，等待构建完成（约 1-2 分钟）

---

## 验证

部署完成后：

```bash
# 健康检查（免鉴权）
curl -s https://<your-url>.code.run/healthz
# → {"status":"ok",...}

# 模型列表
curl -s https://<your-url>.code.run/v1/models
# → 200，模型列表

# OpenAI 兼容对话（bridge 模式：客户端自带上游 token）
curl -s https://<your-url>.code.run/v1/chat/completions \
  -H "Authorization: Bearer <client-key-or-upstream-token>" \
  -H "Content-Type: application/json" \
  -d '{"model":"...","messages":[{"role":"user","content":"hi"}],"stream":true}'
```

---

## 云上差异（对比本机运行）

- **不休眠**：Sandbox 是 always-on-compute，内存态会话/run 池**常驻**，无冷启动重置（比 Vercel scale-to-zero 更适合本代理）
- **SSE 流式**：容器平台原生透传，无缓冲、无超时上限
- **uTLS stealth**：原生 Linux 容器，TLS 指纹伪装完整可用（边缘函数平台做不到）
- **美国出口**：部署区域选美国（us-east-1 / us-central1 / us-west 等），出口即该区域
- **无 CLI 登录文件**：必须设 `AUTO_DISCOVER_TOKEN=false`
- **会话持久化**：容器重启会丢内存态会话（上游已支持 `SESSION_PERSIST` + `SESSION_STATE_FILE` 落盘，可选启用；注意 Sandbox 免费档实例重启后文件系统会重置）

## 免费档边界（Sandbox）

- 计算规格：社区实测约 0.2 vCPU / 512MB（nf-compute-20），无自动扩缩容
- 2 个免费 service（本项目占 1 个）+ 1 个免费数据库 + 2 个 cron
- 静态出口 IP 需付费（egress IP），默认动态出口 = 部署区域
- 日志保留 30 天
