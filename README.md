# ClaudeCodeProxy 部署说明

## 准备

- 一台 Linux 服务器（x86_64 / amd64），建议 2 核 4G 以上，已安装 Docker 和 Docker Compose 插件
  - 安装：`curl -fsSL https://get.docker.com | sh`

## 第一次部署

```bash
mkdir -p /opt/cc2api && cd /opt/cc2api
curl -fsSLO https://raw.githubusercontent.com/2530185073/ccproxy/main/docker-compose.yml
curl -fsSLO https://raw.githubusercontent.com/2530185073/ccproxy/main/.env.example

# 1. 生成配置
cp .env.example .env
# 用下面的命令生成随机密码，分别填到 ADMIN_PASSWORD / POSTGRES_PASSWORD / REDIS_PASSWORD
openssl rand -hex 16
# ADMIN_PATH 也改成不容易猜到的路径
vi .env

# 2. 启动
docker compose up -d
docker compose ps        # app / postgres / redis 都是 healthy 即正常
```

打开 `http://<服务器IP>:<APP_PORT>/<ADMIN_PATH>/`，用 `ADMIN_PASSWORD` 登录后台。

## 升级

```bash
cd /opt/cc2api
# 把 .env 里的 CC2API_VERSION 改成新版本号（用 latest 可跳过这步）
docker compose pull
docker compose up -d
```

数据保存在 Docker 数据卷里，升级不会丢失。

## 备份

```bash
cd /opt/cc2api
docker compose exec -T postgres pg_dump -U ccproxy -d claude_code_gateway | gzip > backup-$(date +%F).sql.gz
```

## 常见问题

- 启动时报「请在 .env 中设置 …」：`.env` 里对应的密码没有填。
- 建议在前面加 Nginx / Caddy 配 HTTPS，并把 `.env` 里的 `APP_BIND` 改为 `127.0.0.1`，只让反向代理访问。
