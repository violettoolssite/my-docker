# 分布式 Docker 镜像加速站 (Distributed Docker Proxy)

这是一个开箱即用的、支持多级分布式的专属 Docker 镜像拉取加速站方案。

## 架构说明

本项目分为两个核心组件，您可以将它们部署在不同的服务器上：

1. **Gateway (主节点)**：位于 `gateway/` 目录。
   负责公网暴露，通过 Nginx 接受来自您或您团队的所有 Docker Pull 请求，并利用一致性哈希算法智能转发给后端多台 Worker 机器。这样不仅可以分散网络出口压力，还可以提升缓存命中率。

2. **Worker (加速缓存节点)**：位于 `worker/` 目录。
   负责真正的镜像拉取以及缓存存留（Pull-Through Cache）。如果某个请求镜像对应的 layer 还没有在自己机器的缓存中，它会主动向上游拉取存入 `worker/data/` 目录，下次直接向用户极速分发。

---

## 快速使用部署指引

> **前提条件**：您的所有主节点和子节点机器都需要安装了 `docker` 及 `docker-compose`。

### 第一步：部署多个子节点 (Worker)

1. 在子节点机器上，将本项目 clone 代码下来。
2. 进入 `worker/` 文件夹。
3. （重要）你可以通过修改 `worker/config.yml` 里的 `remoteurl` 来指定它的上游地址（默认是 Docker 官方拉取点）。
4. 启动机器服务：
   ```bash
   cd worker
   docker-compose up -d
   ```
5. 记录下这些机器的局域网或公网 IP 地址。

### 第二步：部署主节点网关 (Gateway)

1. 登录作为总入口的主节点服务器，进入 `gateway/` 文件夹。
2. 编辑 `gateway/nginx.conf`，将其中 `upstream worker_nodes` 代码块里的 IP 替换为您刚刚部署的真实 Worker 节点的 IP 及端口。
3. 启动网关服务：
   ```bash
   cd gateway
   docker-compose up -d
   ```
4. 如果主节点支持域名绑定，可以去 DNS 解析将自己的域名 A 记录指向该 Gateway 所在服务器。

### 第三步：如何测试与使用

在世界上任意一台需要加速拉取镜像的 Docker 主机上，修改或者创建 `/etc/docker/daemon.json`（如果使用 Windows 或 Mac 则在 Docker Desktop 设置的 Docker Engine 中修改）：

```json
{
  "registry-mirrors": ["http://您的Gateway机器的IP或域名"]
}
```

保存并重启 Docker 引擎：
```bash
sudo systemctl restart docker
```

然后验证您的拉取速度：
```bash
docker pull ubuntu:latest
```

这时候请求就会顺着 `您的机器 -> Gateway主节点 -> Worker 缓存节点 -> 官方源` 行进。当你删除刚才拉取的镜像并再次 `docker pull ubuntu:latest` 或者用另一台机器拉取时，就会体验到内网/多机缓存秒回带来的极速体验。
