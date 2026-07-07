# Traefik 生产部署模板

这是一个可复制到不同服务器的 Traefik 边缘网关模板。默认使用 Docker 配置提供者、文件配置提供者、Cloudflare DNS-01 验证、Let's Encrypt 泛域名证书，以及 Docker socket proxy。

## 目录内容

- `compose.yml`：Traefik 和 Docker socket proxy 服务定义
- `config/traefik.yml`：Traefik 静态配置
- `config/dynamic/routes.yml`：通用中间件和 TLS 配置
- `.env.example`：环境变量模板
- `.env`：当前服务器配置，不要提交到代码仓库
- `data/letsencrypt/acme.json`：Let's Encrypt 证书存储文件，不要复制给其他服务器

## 配置原则

正常部署 Traefik 主网关时，只需要复制并编辑 `.env`。`compose.yml`、`config/traefik.yml`、`config/dynamic/routes.yml` 已经模板化，普通用户不要修改。

需要修改的文件：

- `.env`：当前服务器的自定义配置，包括 Cloudflare 令牌、公共 Docker 网络名、主域名、ACME 邮箱、控制台域名和控制台登录凭据。
- 业务服务自己的 `compose.yml` 或 `.env`：添加业务服务时，修改业务服务域名、容器端口、镜像、数据卷和要使用的中间件。

不要修改的文件：

- `compose.yml`：Traefik 主服务模板，变量已经从 `.env` 读取。
- `config/traefik.yml`：Traefik 静态配置，包含入口、配置提供者、ACME、真实 IP 信任边界等基础行为。
- `config/dynamic/routes.yml`：通用中间件和 TLS 策略。
- `data/letsencrypt/acme.json`：证书存储文件，除非按故障修复步骤处理空文件或损坏 JSON。

只有在明确要改变模板行为时，才修改主配置文件。例如更换入口端口、调整 TLS 策略、变更 Cloudflare 真实 IP 信任范围、升级镜像版本或新增全局中间件。

## 首次部署

1. 复制环境变量模板：

```bash
cp .env.example .env
```

2. 只编辑 `.env`，不要修改主配置文件。至少修改：

- `CLOUDFLARE_DNS_API_TOKEN`
- `APP_PROXY_NETWORK`
- `ACME_EMAIL`
- `DOMAIN`
- `TRAEFIK_DASHBOARD_HOST`
- `TRAEFIK_DASHBOARD_AUTH`

Traefik 服务通过 `.env` 读取 Cloudflare 令牌、公共业务网络名、ACME 邮箱、证书主域名、控制台域名和控制台凭据；`compose.yml` 使用必填变量校验，漏填时会在启动前直接报错。

控制台使用 Traefik 基础认证（BasicAuth）。可以用本机 `htpasswd` 或临时 Docker 容器生成密码哈希：

```bash
htpasswd -nbB admin 'your-dashboard-password'
# 或
docker run --rm httpd:2.4-alpine htpasswd -nbB admin 'your-dashboard-password'
```

把输出写入 `.env` 的 `TRAEFIK_DASHBOARD_AUTH`，并使用单引号保留哈希里的 `$` 字符：

```env
TRAEFIK_DASHBOARD_AUTH='admin:$2y$05$replace-with-real-bcrypt-hash'
```

3. 按 `.env` 里的 `APP_PROXY_NETWORK` 创建外部网络。默认示例值是：

```bash
docker network create shared_services_net
```

部署时请按 `.env` 里的实际值创建同名公共 Docker 网络，Traefik 和需要被反代的业务服务都加入这个网络。

如果改成其他名字，只需要统一修改各项目 `.env` 里的 `APP_PROXY_NETWORK`；Traefik 主模板和业务服务模板会把外部网络名、Docker provider 默认网络、`traefik.docker.network` label 同步到这个值。不要为了改网络名去改 `compose.yml`。

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

- `.env` 包含 Cloudflare 令牌，只允许当前用户读写。
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

## 控制台访问控制

控制台已启用 Traefik 基础认证（BasicAuth），凭据来自 `.env` 中的 `TRAEFIK_DASHBOARD_AUTH`。
控制台域名来自 `.env` 中的 `TRAEFIK_DASHBOARD_HOST`，必须是 `DOMAIN` 或其子域名，才能复用模板签发的泛域名证书。

如果同时使用 Cloudflare Access、Zero Trust 或 WAF，控制台域名的 DNS 记录必须启用 Cloudflare 代理，也就是橙色云朵；否则访问流量不会经过 Cloudflare 的访问限制。

基础认证是 Traefik 侧的认证，Cloudflare Access/WAF 和源站防火墙仍建议作为额外保护。源站直连流量建议使用 UFW、安全组或云防火墙限制。

## 安全响应头中间件

模板提供两个可复用的响应头中间件：

- `default-security-headers@file`：适合私有服务和管理后台。它会禁止搜索引擎索引、禁止 iframe 嵌入，并启用 HSTS preload。
- `public-security-headers@file`：适合公开站点。它保留基础安全响应头，不设置 `X-Robots-Tag`，不强制 `frameDeny`，也不启用 HSTS preload。

公开网站、需要被搜索引擎收录的页面，或需要被可信页面 iframe 嵌入的服务，优先使用 `public-security-headers@file`。

## 业务服务接入示例

业务容器需要加入 `APP_PROXY_NETWORK` 指向的公共网络，并显式开启 Traefik：

```yaml
labels:
  - traefik.enable=true
  - traefik.docker.network=${APP_PROXY_NETWORK:?请在 .env 设置 APP_PROXY_NETWORK}
  - traefik.http.routers.app.rule=Host(`app.example.com`)
  - traefik.http.routers.app.entrypoints=websecure
  - traefik.http.routers.app.tls.certresolver=letsencrypt
  - traefik.http.routers.app.middlewares=default-security-headers@file,compress@file
  - traefik.http.services.app.loadbalancer.server.port=3000
```

公开站点可以把中间件改成：

```yaml
labels:
  - traefik.http.routers.app.middlewares=public-security-headers@file,compress@file
```

完整业务服务示例见 [examples/memos](examples/memos)。这个示例展示了如何部署 Memos：不暴露宿主机端口，只通过 Traefik 的共享网络和 labels 对外提供 HTTPS 访问。

业务服务接入时，主 Traefik 配置仍然不要改。只在业务服务目录里改该服务自己的域名、端口、镜像、数据目录和 labels；公共网络名继续通过 `APP_PROXY_NETWORK` 保持一致。

## 生产注意事项

- 不要复用其他服务器的 `data/letsencrypt/acme.json`。
- 不要把 `.env`、`acme.json`、Cloudflare 令牌提交到仓库。
- `.env` 里的 `TRAEFIK_DASHBOARD_AUTH` 是控制台凭据哈希，也不要提交。
- Cloudflare 令牌建议只授予目标 Zone 的 DNS 编辑权限。
- 控制台必须使用独立域名，并建议在 Cloudflare 侧配置访问限制。
- `.env` 里的 `shared_services_net`、`admin@example.com`、`example.com`、`traefik.example.com` 是示例值，部署前要按真实环境确认或替换。
- Traefik 和 Docker socket proxy 镜像版本已经在 `compose.yml` 中固定，升级前先在测试机验证。
- `default-security-headers@file` 启用了 HSTS preload，需要确认所有子域名都长期支持 HTTPS；公开站点或不确定的服务可使用 `public-security-headers@file`。

## 官方参考

- https://doc.traefik.io/traefik/getting-started/docker/
- https://doc.traefik.io/traefik/reference/install-configuration/providers/docker/
- https://doc.traefik.io/traefik/reference/install-configuration/entrypoints/
- https://doc.traefik.io/traefik/reference/install-configuration/api-dashboard/
- https://doc.traefik.io/traefik/reference/install-configuration/tls/certificate-resolvers/acme/
- https://go-acme.github.io/lego/dns/cloudflare/
