# Traefik 生产部署模板

这是一个可复制到不同服务器的 Traefik 边缘网关模板。默认使用 Docker Provider、File Provider、Cloudflare DNS Challenge、Let's Encrypt 泛域名证书、Cloudflare 源 IP 白名单，以及 Docker socket proxy。

## 目录内容

- `compose.yml`：Traefik 和 Docker socket proxy 服务定义
- `config/traefik.yml`：Traefik 静态配置
- `config/dynamic/routes.yml`：通用中间件和 TLS 配置
- `.env.example`：环境变量模板
- `.env`：当前服务器配置，不要提交到代码仓库
- `data/letsencrypt/acme.json`：Let's Encrypt 证书存储文件，不要复制给其他服务器

## 首次部署

1. 复制环境变量模板：

```bash
cp .env.example .env
```

2. 编辑 `.env`，至少修改：

- `TRAEFIK_DOCKER_NETWORK`
- `TRAEFIK_ACME_EMAIL`
- `TRAEFIK_ACME_DOMAIN_MAIN`
- `TRAEFIK_ACME_DOMAIN_SANS`
- `TRAEFIK_DASHBOARD_HOST`
- `CLOUDFLARE_DNS_API_TOKEN`

3. 创建外部网络：

```bash
docker network create proxy
```

这里的网络名必须和 `.env` 里的 `TRAEFIK_DOCKER_NETWORK` 一致。`compose.yml` 里的 `app_proxy` 只是 Compose 内部别名，真正的 Docker 网络名由 `TRAEFIK_DOCKER_NETWORK` 决定。

模板默认是 `proxy`；如果你的 `.env` 是 `TRAEFIK_DOCKER_NETWORK=core_net`，就创建：

```bash
docker network create core_net
```

4. 创建证书存储文件并限制权限：

```bash
mkdir -p data/letsencrypt
printf '{}' > data/letsencrypt/acme.json
chmod 755 config config/dynamic data data/letsencrypt
chmod 644 compose.yml config/traefik.yml config/dynamic/routes.yml
chmod 600 .env
chmod 600 data/letsencrypt/acme.json
```

5. 启动：

```bash
docker compose up -d
```

6. 检查：

```bash
docker compose ps
docker compose logs -f traefik
```

Traefik 容器会挂载宿主机的 `/etc/localtime`，日志时间跟随 Linux 服务器系统时区。

## 文件权限

Linux 服务器建议使用以下权限：

```bash
chmod 755 config config/dynamic data data/letsencrypt
chmod 644 compose.yml config/traefik.yml config/dynamic/routes.yml .env.example README.md
chmod 600 .env data/letsencrypt/acme.json
```

权限含义：

- `.env` 包含 Cloudflare Token，只允许当前用户读写。
- `data/letsencrypt/acme.json` 包含 Let's Encrypt 账号和证书私钥，只允许当前用户读写。
- `compose.yml`、`config/traefik.yml`、`config/dynamic/routes.yml` 只需要普通只读权限。
- `config/`、`data/` 目录需要可进入权限，否则 Docker bind mount 可能失败。

## 常见问题

如果日志出现：

```text
unable to get ACME account: unexpected end of JSON input
Router uses a nonexistent certificate resolver
```

说明 `data/letsencrypt/acme.json` 是空文件或损坏 JSON，导致 `letsencrypt` resolver 被 Traefik 跳过。首次部署且还没有签发证书时，可以这样修复：

```bash
docker compose stop traefik
printf '{}' > data/letsencrypt/acme.json
chmod 600 data/letsencrypt/acme.json
docker compose up -d traefik
docker compose logs -f traefik
```

如果这个文件已经有真实证书内容，先备份再处理，不要直接覆盖。

## Cloudflare 访问控制

Dashboard 没有启用 Traefik Basic Auth，访问控制应在 Cloudflare Access、Zero Trust、WAF 或防火墙中完成。

模板同时给 Dashboard 增加了 `cloudflare-only` 中间件，只允许 Cloudflare 官方源 IP 段访问 Traefik Dashboard 路由。这能降低绕过 Cloudflare 直连源站的风险，但前提是 `TRAEFIK_CLOUDFLARE_SOURCE_RANGES` 保持更新。

Dashboard 域名的 DNS 记录必须启用 Cloudflare 代理，也就是橙色云朵；否则访问流量不会经过 Cloudflare 的访问限制。

建议同时在服务器防火墙中只允许 Cloudflare IP 访问 80/443，或至少不要公开真实源站 IP。

## 业务服务接入示例

业务容器需要加入 `.env` 中的 `TRAEFIK_DOCKER_NETWORK` 网络，并显式开启 Traefik：

```yaml
labels:
  - traefik.enable=true
  - traefik.docker.network=${TRAEFIK_DOCKER_NETWORK}
  - traefik.http.routers.app.rule=Host(`app.example.com`)
  - traefik.http.routers.app.entrypoints=websecure
  - traefik.http.routers.app.tls.certresolver=letsencrypt
  - traefik.http.routers.app.middlewares=default-security-headers@file,compress@file
  - traefik.http.services.app.loadbalancer.server.port=3000
```

完整业务服务示例见 [examples/memos](examples/memos)。这个示例展示了如何部署 Memos：不暴露宿主机端口，只通过 Traefik 的共享网络和 labels 对外提供 HTTPS 访问。

## 生产注意事项

- 不要复用其他服务器的 `data/letsencrypt/acme.json`。
- 不要把 `.env`、`acme.json`、Cloudflare Token 提交到仓库。
- Cloudflare Token 建议只授予目标 Zone 的 DNS edit 权限。
- Dashboard 必须使用独立域名，并在 Cloudflare 侧配置访问限制。
- `TRAEFIK_IMAGE_TAG` 和 `DOCKER_SOCKET_PROXY_TAG` 建议固定版本，升级前先在测试机验证。
- HSTS preload 需要确认所有子域名都长期支持 HTTPS；否则把 `stsPreload` 改为 `false`。
- `TRAEFIK_CLOUDFLARE_SOURCE_RANGES` 来自 Cloudflare 官方 IP 列表，迁移或定期维护时应刷新。

## 官方参考

- https://doc.traefik.io/traefik/getting-started/docker/
- https://doc.traefik.io/traefik/reference/install-configuration/providers/docker/
- https://doc.traefik.io/traefik/reference/install-configuration/entrypoints/
- https://doc.traefik.io/traefik/reference/install-configuration/api-dashboard/
- https://doc.traefik.io/traefik/reference/install-configuration/tls/certificate-resolvers/acme/
- https://go-acme.github.io/lego/dns/cloudflare/
- https://www.cloudflare.com/ips/
