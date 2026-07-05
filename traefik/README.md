# Traefik 部署说明

这是一个独立的 Traefik 生产部署目录，基于官方 Docker 文档思路整理，适配当前仓库的 `core_net` 共享网络。

## 目录内容

- `compose.yml`：Traefik 服务定义
- `config/traefik.yml`：静态配置
- `config/dynamic/routes.yml`：动态路由配置
- `.env.example`：环境变量模板
- `data/letsencrypt/acme.json`：Let's Encrypt 证书存储文件


3. 确保已创建外部网络：

```bash
docker network create core_net
```



8. 创建证书存储文件并限制权限：

```bash
touch data/letsencrypt/acme.json
chmod 600 data/letsencrypt/acme.json
```


9. 启动：

```bash
docker compose up -d
```



## 官方参考

- https://doc.traefik.io/traefik/getting-started/docker/
- https://doc.traefik.io/traefik/reference/install-configuration/providers/docker/
- https://doc.traefik.io/traefik/reference/install-configuration/entrypoints/
- https://doc.traefik.io/traefik/reference/install-configuration/api-dashboard/
- https://doc.traefik.io/traefik/reference/install-configuration/tls/certificate-resolvers/acme/
- https://go-acme.github.io/lego/dns/cloudflare/
