# Memos 服务部署示例

这是一个业务服务接入 Traefik 的标准示例。服务本身不暴露宿主机端口，只加入 Traefik 的共享网络，通过 Docker labels 交给 Traefik 自动路由。

## 配置原则

接入业务服务时，不要修改 Traefik 主目录的 `compose.yml`、`config/traefik.yml` 或 `config/dynamic/routes.yml`。

这个示例需要修改的是当前目录的 `compose.yml`，以及当前服务自己的 `.env`。常见修改项是服务域名、实例 URL、数据目录、容器端口、镜像版本和要使用的 Traefik 中间件。

## 部署步骤

1. 修改示例域名：

`compose.yml` 里的 `memos.example.com` 和 `https://memos.example.com` 是示例值，部署前改成真实域名。

2. 确认共享网络已存在：

```bash
docker network inspect shared_services_net
```

`shared_services_net` 是根目录 `.env.example` 中 `APP_PROXY_NETWORK` 的默认示例值。实际部署时，这个值必须和 Traefik 主配置使用的 `APP_PROXY_NETWORK` 一致。

3. 创建数据目录并设置权限：

```bash
mkdir -p data
chmod 755 data
chmod 644 compose.yml README.md
```

4. 启动：

```bash
docker compose --env-file ../../.env up -d
docker compose --env-file ../../.env logs -f memos
```

如果把这个示例复制到独立服务目录，建议在该服务目录的 `.env` 中也写入同一个 `APP_PROXY_NETWORK` 值，然后直接运行 `docker compose up -d`。

## 关键点

- 不要配置 `ports: "5230:5230"`，否则会绕过 Traefik 直接暴露服务端口。
- 示例没有使用 `container_name`，方便同一台服务器上用 Compose 项目名隔离不同服务。
- `expose` 只声明容器内部端口，不会发布到宿主机公网。
- `traefik.docker.network` 必须和服务加入的外部网络名一致，模板会从 `APP_PROXY_NETWORK` 读取。
- `default-security-headers@file` 和 `compress@file` 来自 Traefik 主配置的文件配置提供者。
- Memos 数据保存在当前目录的 `data/`，迁移服务时需要一起备份。
