# Memos 服务部署示例

这是一个业务服务接入 Traefik 的标准示例。服务本身不暴露宿主机端口，只加入 Traefik 的共享网络，通过 Docker labels 交给 Traefik 自动路由。

## 部署步骤

1. 修改示例域名：

`compose.yml` 里的 `memos.example.com` 和 `https://memos.example.com` 是示例值，部署前改成真实域名。

2. 确认共享网络已存在：

```bash
docker network inspect shared_services_net
```

`shared_services_net` 必须和 Traefik 主配置中的外部网络名一致。如果你在 Traefik 主配置里改了网络名，这个示例的 `compose.yml` 也要同步修改。

3. 创建数据目录并设置权限：

```bash
mkdir -p data
chmod 755 data
chmod 644 compose.yml README.md
```

4. 启动：

```bash
docker compose up -d
docker compose logs -f memos
```

## 关键点

- 不要配置 `ports: "5230:5230"`，否则会绕过 Traefik 直接暴露服务端口。
- 示例没有使用 `container_name`，方便同一台服务器上用 Compose 项目名隔离不同服务。
- `expose` 只声明容器内部端口，不会发布到宿主机公网。
- `traefik.docker.network=shared_services_net` 必须和服务加入的外部网络名一致。
- `default-security-headers@file` 和 `compress@file` 来自 Traefik 主配置的 File Provider。
- Memos 数据保存在当前目录的 `data/`，迁移服务时需要一起备份。
