# Memos 服务部署示例

这是一个业务服务接入 Traefik 的标准示例。服务本身不暴露宿主机端口，只加入 Traefik 的共享网络，通过 Docker labels 交给 Traefik 自动路由。

## 部署步骤

1. 复制环境变量模板：

```bash
cp .env.example .env
```

2. 编辑 `.env`：

- `TRAEFIK_DOCKER_NETWORK` 必须和 Traefik 主配置中的值一致。
- `MEMOS_HOST` 是访问域名。
- `MEMOS_INSTANCE_URL` 是完整访问地址，通常是 `https://` 加 `MEMOS_HOST`。

3. 确认共享网络已存在：

```bash
docker network inspect proxy
```

如果 `.env` 里使用的是 `TRAEFIK_DOCKER_NETWORK=core_net`，就检查：

```bash
docker network inspect core_net
```

4. 创建数据目录并设置权限：

```bash
mkdir -p data
chmod 755 data
chmod 600 .env
chmod 644 compose.yml .env.example README.md
```

5. 启动：

```bash
docker compose up -d
docker compose logs -f memos
```

## 关键点

- 不要配置 `ports: "5230:5230"`，否则会绕过 Traefik 直接暴露服务端口。
- 示例没有使用 `container_name`，方便同一台服务器上用 Compose 项目名隔离不同服务。
- `expose` 只声明容器内部端口，不会发布到宿主机公网。
- `traefik.docker.network` 必须等于 `.env` 的 `TRAEFIK_DOCKER_NETWORK`。
- `default-security-headers@file` 和 `compress@file` 来自 Traefik 主配置的 File Provider。
- Memos 数据保存在当前目录的 `data/`，迁移服务时需要一起备份。
