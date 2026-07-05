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

- `TZ`
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

如果你把 `TRAEFIK_DOCKER_NETWORK` 改成了其他名字，这里也要使用同一个名字。

4. 创建证书存储文件并限制权限：

```bash
mkdir -p data/letsencrypt
touch data/letsencrypt/acme.json
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
  - traefik.docker.network=proxy
  - traefik.http.routers.app.rule=Host(`app.example.com`)
  - traefik.http.routers.app.entrypoints=websecure
  - traefik.http.routers.app.tls.certresolver=letsencrypt
  - traefik.http.routers.app.middlewares=default-security-headers@file,compress@file
  - traefik.http.services.app.loadbalancer.server.port=3000
```

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
