# 远程 Web 应用接入示例

这个示例适用于 Traefik 和业务应用不在同一台服务器上的部署方式：

- Server A 运行 Traefik，负责公网 80/443、TLS、路由和通用中间件。
- Server B 运行 Web 应用，应用端口只允许 Server A 通过内网访问。
- Traefik 继续使用 Docker Provider 管理本机服务，同时使用 File Provider 手动声明远程后端。

## Server A：添加动态路由

把 `dynamic/remote-app.yml` 复制到 Traefik 主目录的 `config/dynamic/` 下，并按真实环境修改：

- `remote.example.com`：业务域名。
- `10.0.2.20:8080`：Server B 的内网 IP 和发布端口。
- `healthCheck.path`：示例用 `/` 兼容通用 Web 服务，真实应用建议改成专用健康检查路径，例如 `/health`。
- `remote-app-router`、`remote-app-service`：如果有多个远程服务，建议改成每个服务唯一的名称。

示例默认使用 HTTP 后端：

```yaml
servers:
  - url: http://10.0.2.20:8080
```

如果 Server B 后端已经提供 HTTPS，把服务地址改成 `https://...`，并按注释启用 `serversTransport`。使用自签或内网 CA 证书时，优先挂载内网 CA 文件并配置 `rootCAs`，不要直接跳过证书校验。

修改后 Traefik 会通过 File Provider 自动热加载；也可以主动查看日志确认配置已生效：

```bash
docker compose logs -f traefik
```

## Server B：部署应用

在 Server B 上参考 `compose.server-b.yml` 部署应用。关键点是把容器端口发布到宿主机内网地址或宿主机端口，让 Server A 能访问：

```yaml
ports:
  - "10.0.2.20:8080:80"
```

如果 Docker 不支持绑定到该内网地址，或应用需要监听所有地址，可以临时使用：

```yaml
ports:
  - "8080:80"
```

这种写法必须配合云安全组、UFW、iptables 或防火墙，只允许 Server A 的内网 IP 访问 `8080`，不要对公网开放。

## 连通性检查

在 Server A 上检查能否访问 Server B：

```bash
curl -fsS http://10.0.2.20:8080/
```

如果应用有 `/health`，优先检查应用自己的健康检查路径。健康检查失败时，优先排查内网 IP、端口发布、防火墙和应用监听地址。

## 多远程服务

推荐每个远程服务一个动态配置文件，例如：

```text
config/dynamic/
  routes.yml
  remote-memos.yml
  remote-outline.yml
  remote-vaultwarden.yml
```

这样每个服务可以独立维护路由、后端地址、健康检查和中间件。
