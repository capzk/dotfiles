# Traefik 生产部署模板

这是一个可复制到不同服务器的 Traefik 边缘网关模板。默认使用 Docker 配置提供者、文件配置提供者、Cloudflare DNS-01 验证，以及 Let's Encrypt 泛域名证书。

模板同时覆盖两种常见部署架构：

- **同机应用自动发现**：Traefik 和业务容器运行在同一台服务器上，业务容器加入共享 Docker 网络，并通过 Docker labels 自动生成路由。
- **跨服务器远程后端**：Traefik 运行在 Server A，业务应用运行在 Server B；Traefik 继续管理入口和证书，通过 File Provider 的动态配置把流量转发到 Server B 的内网 IP 和端口。

## 目录内容

- `compose.yml`：Traefik 服务定义
- `config/traefik.yml`：Traefik 静态配置
- `config/dynamic/routes.yml`：通用中间件和 TLS 配置，也可在同目录增加远程后端路由文件
- `.env.example`：环境变量模板
- `.env`：当前服务器配置，不要提交到代码仓库
- `data/letsencrypt/acme.json`：Let's Encrypt 证书存储文件，不要复制给其他服务器
- `examples/memos`：同机 Docker Provider 接入示例
- `examples/remote-webapp`：跨服务器 File Provider 接入示例

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

- `CLOUDFLARE_DNS_API_TOKEN`：Cloudflare API 令牌，至少需要目标 Zone 的 `Zone / Zone / Read` 和 `Zone / DNS / Edit` 权限。
- `APP_PROXY_NETWORK`
- `ACME_EMAIL`
- `ACME_CA_SERVER`
- `DOMAIN`
- `TRAEFIK_DASHBOARD_HOST`
- `TRAEFIK_DASHBOARD_AUTH`

Traefik 服务通过 `.env` 读取 Cloudflare 令牌、公共业务网络名、ACME 邮箱、证书主域名、控制台域名和控制台凭据；`compose.yml` 使用必填变量校验，漏填时会在启动前直接报错。
模板会在固定存在的控制台路由上显式声明 `DOMAIN` 和 `*.DOMAIN`，启动后由 Traefik 通过 DNS-01 申请根域名和通配符证书。

`ACME_CA_SERVER` 默认使用 Let's Encrypt staging 接口：

```env
ACME_CA_SERVER=https://acme-staging-v02.api.letsencrypt.org/directory
```

staging 适合首次部署、调试 Cloudflare DNS-01、反复重建路由和验证配置，避免过早触发 Let's Encrypt 正式环境限流。staging 签发的证书不被浏览器信任；确认 Traefik 日志里 DNS-01 和证书申请流程都正常后，再改成正式接口：

```env
ACME_CA_SERVER=https://acme-v02.api.letsencrypt.org/directory
```

从 staging 切到正式环境时，ACME 账号和证书环境不同。如果 `data/letsencrypt/acme.json` 里只有测试证书，可以停止 Traefik、备份后重置该文件为 `{}`，再启动 Traefik 重新申请正式证书。

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

Traefik 通过只读挂载的 `/var/run/docker.sock` 读取本机容器 labels，用于 Docker Provider 自动发现。这个 socket 不会发布成 TCP 端口；不要额外暴露 Docker API 的 `2375` 或 `2376` 端口。

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

如果只签发了控制台单域名证书，没有签发通配符证书，优先检查：

- `.env` 是否已经设置 `DOMAIN`，例如 `DOMAIN=example.com`，不要带 `https://` 或通配符前缀。
- `.env` 的 `ACME_CA_SERVER` 当前是 staging 还是生产接口；staging 证书用于测试，不会被浏览器信任。
- Traefik 是否已经按新模板重建：`docker compose up -d --force-recreate traefik`。
- `docker compose config` 展开后，控制台路由是否包含 `tls.domains[0].main` 和 `tls.domains[0].sans`。
- Traefik 日志里是否有 Cloudflare API 权限、DNS TXT 传播、Let's Encrypt 限流或 `acme.json` JSON 损坏相关错误。

Traefik 官方 ACME 文档说明，证书解析器会优先根据路由上的 `tls.domains` 申请证书；如果没有显式设置，则会从 `Host()` 规则推导域名，这种情况下通常只会申请当前路由的单域名证书。

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

## 业务服务接入方式

Traefik 主模板同时启用了 Docker Provider 和 File Provider。选择哪种方式取决于业务应用和 Traefik 是否在同一台 Docker 主机上。

### 方式一：同机 Docker 自动发现

适用场景：

- Traefik 和业务容器在同一台服务器上。
- 业务容器可以加入 `.env` 中 `APP_PROXY_NETWORK` 指向的共享 Docker 网络。
- 希望通过 labels 维护路由，避免手写动态后端地址。

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

### 方式二：跨服务器 File Provider 后端

适用场景：

- Traefik 在 Server A，业务应用在 Server B。
- 两台服务器之间有 VPC、内网、专线、WireGuard、Tailscale 或其他私有网络。
- 业务应用不能加入 Traefik 所在主机的 Docker 网络。

这种架构下，不建议让 Traefik 直接连接远程 Docker socket 做自动发现。更稳妥的做法是：

- Server A 的 Traefik 继续暴露公网 `80/443`，负责 TLS、路由、中间件和访问日志。
- Server B 的业务容器把服务端口发布到宿主机内网地址，或使用 `network_mode: host`。
- Server B 防火墙只允许 Server A 的内网 IP 访问业务端口。
- Server A 在 `config/dynamic/` 下为远程应用新增一个动态配置文件，手动写入 `http://Server-B-内网IP:端口`。

Server B 应用示例：

```yaml
services:
  webapp:
    image: your-web-image:latest
    restart: unless-stopped
    ports:
      - "10.0.2.20:8080:80"
```

如果不能绑定到固定内网 IP，可以使用 `"8080:80"`，但必须用安全组或防火墙限制只允许 Server A 访问。

Server A 动态配置示例：

```yaml
http:
  routers:
    remote-app-router:
      rule: Host(`app.example.com`)
      entryPoints:
        - websecure
      middlewares:
        - default-security-headers@file
        - compress@file
      service: remote-app-service
      tls:
        certResolver: letsencrypt

  services:
    remote-app-service:
      loadBalancer:
        passHostHeader: true
        servers:
          - url: http://10.0.2.20:8080
        healthCheck:
          path: /
          interval: 30s
          timeout: 5s
```

完整跨服务器示例见 [examples/remote-webapp](examples/remote-webapp)。复制其中的 `dynamic/remote-app.yml` 到主目录 `config/dynamic/` 后，按真实域名、内网 IP、端口和健康检查路径修改即可。

远程后端使用 HTTPS 时，可以在动态配置中增加 `serversTransports`，为内网 CA、自签证书或后端 SNI 设置专用传输配置。优先配置可信 CA，不要把跳过证书校验作为默认方案。

跨服务器排查顺序：

- 在 Server A 上执行 `curl -fsS http://Server-B-内网IP:端口/`，先确认内网连通性；如果应用有 `/health`，优先检查专用健康检查路径。
- 检查 Server B 的容器端口是否发布到宿主机，应用是否监听正确地址。
- 检查 Server B 防火墙、安全组或云 ACL 是否允许 Server A 的内网 IP。
- 查看 `docker compose logs -f traefik`，确认 File Provider 动态配置加载成功。
- 访问仍异常时，再检查域名解析、TLS 签发、健康检查路径和中间件。

## 生产注意事项

- 不要复用其他服务器的 `data/letsencrypt/acme.json`。
- 不要把 `.env`、`acme.json`、Cloudflare 令牌提交到仓库。
- `.env` 里的 `TRAEFIK_DASHBOARD_AUTH` 是控制台凭据哈希，也不要提交。
- Cloudflare 令牌建议只作用于目标 Zone，并授予 `Zone / Zone / Read` 和 `Zone / DNS / Edit` 权限。
- 控制台必须使用独立域名，并建议在 Cloudflare 侧配置访问限制。
- `.env` 里的 `shared_services_net`、`admin@example.com`、`example.com`、`traefik.example.com` 是示例值，部署前要按真实环境确认或替换。
- Traefik 镜像版本已经在 `compose.yml` 中固定，升级前先在测试机验证。
- `default-security-headers@file` 启用了 HSTS preload，需要确认所有子域名都长期支持 HTTPS；公开站点或不确定的服务可使用 `public-security-headers@file`。

## 官方参考

- https://doc.traefik.io/traefik/getting-started/docker/
- https://doc.traefik.io/traefik/reference/install-configuration/providers/docker/
- https://doc.traefik.io/traefik/reference/install-configuration/providers/others/file/
- https://doc.traefik.io/traefik/reference/install-configuration/entrypoints/
- https://doc.traefik.io/traefik/reference/install-configuration/api-dashboard/
- https://doc.traefik.io/traefik/reference/install-configuration/tls/certificate-resolvers/acme/
- https://doc.traefik.io/traefik/reference/routing-configuration/http/load-balancing/service/
- https://doc.traefik.io/traefik/reference/routing-configuration/http/load-balancing/serverstransport/
- https://go-acme.github.io/lego/dns/cloudflare/
