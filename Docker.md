1. ### 1. Docker 网络配置

   - **创建自定义网络**：可以使用 `docker network create` 命令创建自定义网络，使容器之间可以直接通过名称互相访问。
     
     ```bash
     docker network create --driver bridge my_custom_network
     ```
   - **连接和断开网络**：在创建容器时使用 `--network` 选项连接到指定网络，也可以用 `docker network connect/disconnect` 动态更改容器网络。
     
     ```bash
     docker network connect my_custom_network my_container
     docker network disconnect my_custom_network my_container
     ```
   - **多主机网络**：通过 Docker Swarm 或者 Kubernetes 配置多主机网络，提供跨主机的容器通信。

   ---

   ### 2. 数据管理

   - **卷（Volume）管理**：卷是容器中最可靠的持久化存储方式。
     
     - 创建卷：`docker volume create my_volume`
     - 挂载卷：使用 `-v` 标志，将卷挂载到容器中的特定路径。
     - 清理卷：`docker volume prune` 可以清理未使用的卷。
   - **绑定挂载**：允许将主机文件夹直接映射到容器内。
     
     ```bash
     docker run -v /host/path:/container/path my_image
     ```

   ---

   ### 3. Docker Compose 高级应用

   - **多环境配置**：利用 `.env` 文件来管理不同环境的配置。可以在 `docker-compose.yml` 文件中使用环境变量。
   - **动态扩展服务**：在开发、测试和生产环境中使用不同的 Compose 文件。
     
     ```bash
     docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
     ```
   - **服务扩容与缩容**：利用 `docker-compose up --scale` 命令增加或减少服务实例数量。
     
     ```bash
     docker-compose up --scale web=3
     ```

   ---

   ### 4. 多阶段构建（Multi-Stage Build）

   - 多阶段构建允许你在一个 Dockerfile 中使用多个 `FROM`，这有助于减少最终镜像大小。
     ```Dockerfile
     FROM golang:1.16 AS builder
     WORKDIR /app
     COPY . .
     RUN go build -o myapp
     
     FROM alpine:latest
     WORKDIR /root/
     COPY --from=builder /app/myapp .
     CMD ["./myapp"]
     ```

   ---

   ### 5. Docker 安全

   - **权限控制**：Docker 提供了不同的用户权限（比如 rootless 模式），可以让容器运行在非 root 用户下，减少安全风险。
     
     ```bash
     docker run --user $(id -u):$(id -g) my_image
     ```
   - **Seccomp 与 AppArmor 配置**：可以为容器配置 seccomp profiles 和 AppArmor profiles，限制容器的系统调用和权限。
   - **镜像签名与扫描**：定期扫描镜像漏洞，并通过 Docker Content Trust (DCT) 签名镜像，确保其来源可信。

   ---

   ### 6. Docker 日志管理

   - **日志驱动**：Docker 支持多种日志驱动（如 `json-file`、`syslog`、`journald` 等）。
     
     ```bash
     docker run --log-driver=syslog my_image
     ```
   - **日志文件管理**：通过配置 `max-size` 和 `max-file` 参数限制日志文件大小和数量，避免日志无限增长。
     
     ```bash
     docker run --log-opt max-size=10m --log-opt max-file=3 my_image
     ```

   ---

   ### 7. Docker Swarm 和集群管理

   - **Swarm 初始化**：使用 `docker swarm init` 将 Docker 守护进程初始化为 Swarm 模式。
   - **服务和任务管理**：可以使用 `docker service` 管理 Swarm 集群中的服务。
     
     ```bash
     docker service create --name my_service --replicas 3 my_image
     ```
   - **Overlay 网络**：Swarm 可以自动创建跨节点的 Overlay 网络，实现多主机容器通信。

   ---

   ### 8. 容器优化

   - **分层优化**：在 Dockerfile 中尽量减少层数，合并多个命令，避免创建不必要的缓存层。
     
     ```Dockerfile
     RUN apt-get update && apt-get install -y \
         curl \
         vim \
         && rm -rf /var/lib/apt/lists/*
     ```
   - **使用轻量级基础镜像**：如 `alpine`，减少镜像大小。

   ---

   ### 9. 清理未使用的资源

   - **清理容器**：清理退出状态的容器。
     
     ```bash
     docker container prune
     ```
   - **清理镜像**：删除未被任何容器使用的镜像。
     
     ```bash
     docker image prune -a
     ```
   - **清理卷**：删除未使用的卷。
     
     ```bash
     docker volume prune
     ```
   - **自动化清理**：可以将清理命令加入到 CI/CD 流程中，以确保环境整洁。