# ClaudeCodeProxy 部署教程

一个镜像 + PostgreSQL + Redis，用 Docker Compose 一键启动。全程大约 10 分钟。

- 镜像：`ghcr.io/2530185073/ccproxy:1.8.32`（公开，无需登录即可拉取）
- 部署文件：本仓库的 `docker-compose.yml` 和 `.env.example`

---

## 一、准备服务器

| 项目 | 要求 |
|---|---|
| 系统 | Linux x86_64（amd64），推荐 Ubuntu 22.04 / 24.04 或 Debian 12 |
| 配置 | 2 核 4G 起步，硬盘 20G 以上 |
| 网络 | 能直接访问 `ghcr.io` 和 `api.anthropic.com`（海外服务器一般没问题） |
| 端口 | 用域名 + HTTPS：放行 80、443；只用 IP 访问：放行 5674 |

> 暂不支持 ARM 服务器（如 AWS Graviton）。如需要请联系我们。

用 root 登录服务器后，先确认架构：

```bash
uname -m        # 输出 x86_64 才能继续
```

## 二、安装 Docker

已经装过的跳过，直接看第 3 步的检查命令。

```bash
# 1. 安装 Docker（官方脚本，包含 Compose 插件）
curl -fsSL https://get.docker.com | sh

# 2. 开机自启
systemctl enable --now docker

# 3. 检查：两条命令都要能输出版本号
docker --version
docker compose version
```

`docker compose version` 报错说明 Compose 插件没装上：Ubuntu / Debian 执行 `apt-get install -y docker-compose-plugin`。

## 三、下载部署文件

```bash
mkdir -p /opt/ccproxy && cd /opt/ccproxy
curl -fsSLO https://raw.githubusercontent.com/2530185073/ccproxy/main/docker-compose.yml
curl -fsSLO https://raw.githubusercontent.com/2530185073/ccproxy/main/.env.example
ls -la          # 应该看到 docker-compose.yml 和 .env.example
```

## 四、填写配置

```bash
cp .env.example .env
```

先生成 3 个随机密码，记下来：

```bash
for i in 1 2 3; do openssl rand -hex 16; done
```

再生成一个后台路径：

```bash
openssl rand -hex 6
```

然后编辑 `.env`（`nano .env` 或 `vi .env`），至少改下面 4 项：

| 配置项 | 填什么 | 说明 |
|---|---|---|
| `ADMIN_PASSWORD` | 第 1 个随机密码 | 后台登录密码 |
| `ADMIN_PATH` | 刚生成的后台路径 | 后台地址是 `http://IP:5674/<ADMIN_PATH>/`，不要用默认值，防止被扫 |
| `POSTGRES_PASSWORD` | 第 2 个随机密码 | 数据库密码 |
| `REDIS_PASSWORD` | 第 3 个随机密码 | 缓存密码 |

> ⚠️ 密码只能用字母、数字、`-`、`_`，不要包含 `@ : / # ? %` 等符号。用上面 `openssl rand -hex` 生成的就没问题。
>
> ⚠️ `POSTGRES_PASSWORD` 只在第一次启动时生效。启动以后再改它，应用会连不上数据库。一定要在第一次启动前填好。

其他配置项一般不用改：

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `CC2API_VERSION` | `1.8.32` | 镜像版本。升级时改成新的版本号，不要用 `latest` |
| `APP_BIND` | `0.0.0.0` | 监听地址。配了 HTTPS 反向代理后改成 `127.0.0.1`（见第七步） |
| `APP_PORT` | `5674` | 对外端口 |
| `TZ` | `Asia/Shanghai` | 时区，影响日志和统计的时间显示 |
| `LOG_LEVEL` | `info` | 日志级别：`debug` / `info` / `warn` / `error` |
| `REQUEST_LOG_MAX_ROWS` | `20000` | 后台保留的请求记录条数 |
| `USAGE_RECOVERY_ENABLED` | `1` | 账号限流到期后自动探测恢复；`0` 关闭 |

改完检查一遍，确认没有残留的 `change-me`：

```bash
grep change-me .env      # 没有任何输出才对
```

## 五、启动

```bash
cd /opt/ccproxy
docker compose up -d
```

第一次启动会拉镜像，大约 1 分钟。然后检查状态：

```bash
docker compose ps
```

三个容器（`app`、`postgres`、`redis`）的 STATUS 都显示 `healthy` 就是启动成功了。如果是 `starting`，等 30 秒再看。

再确认一下服务能访问：

```bash
curl http://127.0.0.1:5674/_health      # 返回 ok 即正常
```

数据库表会在第一次启动时自动创建，不需要手动初始化。

## 六、登录后台，完成初始设置

浏览器打开：

```
http://<服务器IP>:5674/<ADMIN_PATH>/
```

用 `.env` 里的 `ADMIN_PASSWORD` 登录。然后按顺序做三件事。

### 1. 添加 Claude 账号

进入左侧「账号管理」，三种方式任选。

**方式 A：授权登录（推荐）**

1. 点页面右上角「导入」→「授权登录（OAuth / Cookie）」。
2. 选「授权类型」：
   - **完整 OAuth**：支持用量查询，到期自动续期，推荐。
   - **Setup Token**：只有推理权限，有效期一年，不能续期。
3. 选「登录方式」：
   - **链接授权**：生成授权链接 → 在浏览器里登录 Claude 并同意授权 → 把回调页显示的授权码粘贴回来。
   - **sessionKey 登录**：在已登录 claude.ai 的浏览器里打开开发者工具 → Application → Cookies → `https://claude.ai`，复制 `sessionKey` 的值（`sk-ant-sid02-…` 开头）粘贴进来。sessionKey 只用这一次，不会保存。
4. 拿到凭据后，按提示填入新建账号表单，给账号起个名字、填上代理，保存。

**方式 B：手动添加 Setup Token**

在本地电脑执行 `claude setup-token`，得到 `sk-ant-oat01-…`。然后点「添加账号」，认证方式选「Setup Token」，把它粘贴进去，保存。

**方式 C：批量导入**

点「导入」→「批量导入（Token / JSON 备份）」，一次粘贴多个 Setup Token（每行一个），也可以直接拖入 `.txt` / `.csv` 文件。

**建议**：每个账号单独配一个代理，一号一个固定 IP，填在账号的「代理」一栏（支持 `http://` 和 `socks5://`）。

添加后在账号上点「测试连接」，确认账号可用。

### 2. 创建 API 令牌

左侧「API 令牌」→「创建令牌」，填个名字（比如使用人的名字），创建后复制令牌。可以按人分别建令牌，方便分开统计用量、单独停用。

### 3. 检查系统设置（可选）

左侧「系统设置」里可以调整账号调度、请求规整等策略，默认值就能用。

## 七、配置域名和 HTTPS（强烈推荐）

不配 HTTPS，令牌在网络上是明文传输的。下面用 Caddy，它会自动申请和续期证书。

**1. 域名解析**：在域名服务商那里加一条 A 记录，比如 `api.example.com` → 服务器 IP。等几分钟生效，可以用 `ping api.example.com` 检查是否解析到你的服务器。

**2. 安装 Caddy**（Ubuntu / Debian）：

```bash
apt-get install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | tee /etc/apt/sources.list.d/caddy-stable.list
apt-get update && apt-get install -y caddy
```

**3. 写配置**（把 `api.example.com` 换成你的域名）：

```bash
cat > /etc/caddy/Caddyfile <<'EOF'
api.example.com {
    reverse_proxy 127.0.0.1:5674 {
        flush_interval -1
    }
}
EOF
systemctl reload caddy
```

`flush_interval -1` 让流式回复实时推送，不要删。

**4. 只允许通过 Caddy 访问**：把 `.env` 里的 `APP_BIND` 改成 `127.0.0.1`，然后：

```bash
cd /opt/ccproxy && docker compose up -d
```

现在后台地址变成 `https://api.example.com/<ADMIN_PATH>/`，5674 端口不再对外开放。

## 八、客户端接入

### Claude Code

在使用者的电脑上设置两个环境变量：

```bash
export ANTHROPIC_BASE_URL=https://api.example.com     # 没配域名就写 http://<服务器IP>:5674
export ANTHROPIC_AUTH_TOKEN=<第六步创建的 API 令牌>
claude
```

也可以写进 `~/.claude/settings.json`，就不用每次 export：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://api.example.com",
    "ANTHROPIC_AUTH_TOKEN": "<API 令牌>"
  }
}
```

### 其他客户端 / 直接调用 API

兼容 Anthropic Messages API，地址填 `https://api.example.com`，令牌放在 `x-api-key` 头或 `Authorization: Bearer <令牌>` 头里都可以：

```bash
curl https://api.example.com/v1/messages \
  -H "x-api-key: <API 令牌>" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-sonnet-4-6","max_tokens":64,"messages":[{"role":"user","content":"hi"}]}'
```

请求发出后，可以在后台「请求记录」里看到这条记录。

## 九、日常维护

### 升级

```bash
cd /opt/ccproxy
# 先把 .env 里的 CC2API_VERSION 改成新版本号
docker compose pull
docker compose up -d
docker image prune -f      # 清理旧镜像，可选
```

升级不会丢数据，数据都在 Docker 数据卷里。**升级前建议先备份。**

### 备份

```bash
cd /opt/ccproxy
mkdir -p backups
docker compose exec -T postgres pg_dump -U ccproxy -d claude_code_gateway | gzip > backups/ccproxy-$(date +%F-%H%M).sql.gz
chmod 600 backups/*.sql.gz      # 备份里有账号凭据，注意保管
```

每天自动备份一次、保留 14 天（加入 root 的 crontab）：

```bash
(crontab -l 2>/dev/null; echo '0 4 * * * cd /opt/ccproxy && docker compose exec -T postgres pg_dump -U ccproxy -d claude_code_gateway | gzip > backups/ccproxy-$(date +\%F).sql.gz && find backups -name "*.sql.gz" -mtime +14 -delete') | crontab -
```

### 恢复

```bash
cd /opt/ccproxy
docker compose stop app
docker compose exec -T postgres psql -U ccproxy -d postgres -c "DROP DATABASE claude_code_gateway;"
docker compose exec -T postgres psql -U ccproxy -d postgres -c "CREATE DATABASE claude_code_gateway OWNER ccproxy;"
gunzip -c backups/<备份文件>.sql.gz | docker compose exec -T postgres psql -U ccproxy -d claude_code_gateway
docker compose start app
```

### 迁移到新服务器

1. 旧服务器：备份数据库，并把 `.env` 一起拷走（里面的 `POSTGRES_PASSWORD` 等要保持一致）。
2. 新服务器：按第二、三步装好 Docker、下载部署文件，放入旧的 `.env`。
3. 新服务器：`docker compose up -d` 启动，然后按上面的「恢复」步骤导入备份。

### 查看日志

```bash
cd /opt/ccproxy
docker compose logs -f --tail 100 app      # 实时看应用日志，Ctrl+C 退出
docker compose logs --tail 100 postgres
```

### 常用命令

| 操作 | 命令（都在 `/opt/ccproxy` 下执行） |
|---|---|
| 查看状态 | `docker compose ps` |
| 重启应用 | `docker compose restart app` |
| 停止全部 | `docker compose stop` |
| 启动全部 | `docker compose start` |
| 修改 `.env` 后生效 | `docker compose up -d` |

## 十、常见问题

**启动报「请在 .env 中设置 …」**
`.env` 里对应的项没填。按第四步补上后重新 `docker compose up -d`。

**`docker compose ps` 里 app 一直不是 healthy**
看日志 `docker compose logs --tail 50 app`。最常见的原因是数据库密码对不上：第一次启动后改过 `POSTGRES_PASSWORD`。要么改回原来的值；要么在**确认不需要旧数据**的前提下执行 `docker compose down -v` 清空数据重新开始。

**浏览器打不开后台**
- 地址末尾带 `/`：`http://IP:5674/<ADMIN_PATH>/`
- 服务器本机执行 `curl http://127.0.0.1:5674/_health` 能通，但外面打不开：检查云服务商的安全组 / 防火墙有没有放行端口。
- 配了 Caddy 后 `APP_BIND` 改成了 `127.0.0.1`：这时只能用域名访问，IP 加端口访问不了，这是正常的。

**忘了后台密码**
改 `.env` 里的 `ADMIN_PASSWORD`，执行 `docker compose up -d`，用新密码登录。

**端口 5674 被占用**
把 `.env` 里的 `APP_PORT` 改成其他端口，比如 `15674`，执行 `docker compose up -d`。配了 Caddy 的话，Caddyfile 里的 `127.0.0.1:5674` 也要同步改。

**Caddy 申请证书失败**
确认域名已经解析到这台服务器，并且 80、443 端口都已放行。查看原因：`journalctl -u caddy --no-pager | tail -30`。

**客户端返回 401**
令牌填错，或者令牌已在后台被停用。去「API 令牌」页面重新复制一次。

**请求返回 429 / 503**
账号额度用完或被限流了。到「账号管理」看各账号状态，或者多加几个账号。

## 十一、卸载

```bash
cd /opt/ccproxy
docker compose down        # 停止并删除容器，数据保留
docker compose down -v     # ⚠️ 连数据一起删除，无法恢复
```
