# Traefik 生产部署模板

这是一个可复制到不同服务器的 Traefik 边缘网关模板。默认使用 Docker Provider、File Provider、Cloudflare DNS Challenge、Let's Encrypt 泛域名证书，以及 Docker socket proxy。

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

- `CLOUDFLARE_DNS_API_TOKEN`
- `TRAEFIK_DASHBOARD_AUTH`

Traefik 服务通过 `env_file: .env` 读取 Cloudflare Token，用于 DNS-01 签发证书。

Dashboard 使用 Traefik BasicAuth。可以用本机 `htpasswd` 或临时 Docker 容器生成密码哈希：

```bash
htpasswd -nbB admin 'your-dashboard-password'
# 或
docker run --rm httpd:2.4-alpine htpasswd -nbB admin 'your-dashboard-password'
```

把输出写入 `.env`，并使用单引号保留哈希里的 `$` 字符：

```env
TRAEFIK_DASHBOARD_AUTH='admin:$2y$05$replace-with-real-bcrypt-hash'
```

邮箱和域名直接写在 `compose.yml` 里。示例值是 `admin@example.com`、`example.com`、`*.example.com`、`traefik.example.com`，部署前按实际域名修改。

3. 创建外部网络：

```bash
docker network create shared_services_net
```

`shared_services_net` 是用户手动创建的公共 Docker 网络，Traefik 和需要被反代的业务服务都加入这个网络。如果你要换成其他名字，需要同时修改 Traefik 和业务服务 compose 里的外部网络名，以及 `traefik.docker.network` label。

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

Traefik 会等待 `docker-socket-proxy` 健康检查通过后再启动。这个健康检查通过 Docker API 的 `/_ping` 端点确认 socket proxy 已经可以响应。

6. 检查：

```bash
docker compose ps
docker compose logs -f traefik
```

Traefik 容器会挂载宿主机的 `/etc/localtime`，日志时间跟随 Linux 服务器系统时区。

## 访客真实 IP

模板默认按“Cloudflare 在 Traefik 前面”的架构配置真实 IP：

- `config/traefik.yml` 在 `web` 和 `websecure` 两个入口上配置了 `forwardedHeaders.trustedIPs`，只信任 Cloudflare 边缘节点传入的 `X-Forwarded-*` 请求头。
- 访问日志使用 JSON 格式，并保留 `CF-Connecting-IP`、`X-Forwarded-For`、`X-Real-IP`、`CF-Ray` 等排查真实访客 IP 所需的请求头。
- 日志中优先查看对应的 `request_*` 头字段，例如 `CF-Connecting-IP` 或 `X-Forwarded-For`；`CF-Connecting-IP` 是 Cloudflare 传给源站的单一访客 IP。

如果源站没有放在 Cloudflare 后面，不要直接开启 `forwardedHeaders.insecure=true`。应把 `trustedIPs` 改成你自己的上游反向代理、负载均衡器或 CDN 的出口网段。

Cloudflare IP 段可能变化，迁移或长期运行前建议按官方列表核对：

- https://www.cloudflare.com/ips-v4/
- https://www.cloudflare.com/ips-v6/

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

## Dashboard 访问控制

Dashboard 已启用 Traefik BasicAuth，凭据来自 `.env` 中的 `TRAEFIK_DASHBOARD_AUTH`。

如果同时使用 Cloudflare Access、Zero Trust 或 WAF，Dashboard 域名的 DNS 记录必须启用 Cloudflare 代理，也就是橙色云朵；否则访问流量不会经过 Cloudflare 的访问限制。

BasicAuth 是 Traefik 侧的认证，Cloudflare Access/WAF 和源站防火墙仍建议作为额外保护。源站直连流量建议使用 UFW、安全组或云防火墙限制。

## 业务服务接入示例

业务容器需要加入 `shared_services_net` 网络，并显式开启 Traefik：

```yaml
labels:
  - traefik.enable=true
  - traefik.docker.network=shared_services_net
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
- `.env` 里的 `TRAEFIK_DASHBOARD_AUTH` 是 Dashboard 凭据哈希，也不要提交。
- Cloudflare Token 建议只授予目标 Zone 的 DNS edit 权限。
- Dashboard 必须使用独立域名，并建议在 Cloudflare 侧配置访问限制。
- `compose.yml` 里的 `admin@example.com`、`example.com`、`*.example.com`、`traefik.example.com` 是示例值，部署前要替换成真实值。
- Traefik 和 Docker socket proxy 镜像版本已经在 `compose.yml` 中固定，升级前先在测试机验证。
- HSTS preload 需要确认所有子域名都长期支持 HTTPS；否则把 `stsPreload` 改为 `false`。

## 官方参考

- https://doc.traefik.io/traefik/getting-started/docker/
- https://doc.traefik.io/traefik/reference/install-configuration/providers/docker/
- https://doc.traefik.io/traefik/reference/install-configuration/entrypoints/
- https://doc.traefik.io/traefik/reference/install-configuration/api-dashboard/
- https://doc.traefik.io/traefik/reference/install-configuration/tls/certificate-resolvers/acme/
- https://go-acme.github.io/lego/dns/cloudflare/
