# Kubernetes (k8s) 

## 1. **Kubernetes 概述**
   - **定义**：由 Google 开源的容器编排平台，用于自动化部署、扩展和管理容器化应用程序。
   - **简称**：Kubernetes（K8s），源于单词中每个字母之间的8个字母。

## 2. **Kubernetes 架构**
   - **控制平面**：负责整个集群的管理和决策，包括 API Server、Scheduler、Controller Manager 和 etcd。
   - **工作节点（Node）**：运行容器化应用程序的服务器。

## 3. **控制平面内部组件**
| 组件                   | 功能描述                                                   | 手动操作（以前）                                        |
| ---------------------- | ---------------------------------------------------------- | ------------------------------------------------------- |
| **API Server**         | 集群的前端，处理所有 RESTful 请求和更新状态。              | 登录到每台服务器，手动执行命令                          |
| **Scheduler**          | 负责决定在哪个节点上运行 Pod，基于资源需求、亲和性规则等。 | 查看每台服务器的 CPU 和内存资源，手动选择服务器部署应用 |
| **Controller Manager** | 负责集群中各种控制器的运行，如节点控制器、副本控制器等。   | 找到服务器后，手动创建或关闭服务                        |
| **etcd**               | 分布式键值存储，用于保存集群的所有数据，包括配置、状态等。 | /                                                       |

## 4. **Node 内部组件**
   - **Container Runtime**：负责运行容器，如 Docker、containerd 等。
   - **Pod**：Kubernetes 基本部署单元，可以包含一个或多个容器。
   - **Kubelet**：在每个节点上运行，确保容器运行在 Pod 中。
   - **Kube Proxy**：管理节点上的网络请求，实现服务发现和负载均衡。

## 5. **Cluster（集群）**
   - **定义**：由一组节点组成，共同提供应用程序的运行环境。
   - **环境**：通常包括开发、测试和生产环境。

## 6. **kubectl**
   - **功能**：Kubernetes 的命令行工具，用于与集群交互和管理资源。
   - **使用**：通过 YAML 文件定义资源，然后使用 `kubectl` 命令部署和管理。

## 7. **部署服务**
   - **YAML 文件**：定义 Pod、服务、部署等资源的配置。
   - **部署命令**：`kubectl apply -f xx.yaml`，将配置应用到集群。

## 8. **服务调用**
   - **Ingress 控制器**：管理外部访问集群内部服务的入口。
   - **请求流程**：外部请求 → Ingress 控制器 → Kube Proxy → Pod → 容器服务。

## 9. **Docker 与 Kubernetes 的关系**
   - **Docker**：容器化平台，用于创建和运行容器。
   - **Kubernetes**：容器编排平台，用于管理 Docker 容器的部署、扩展和运行。



# MacOS本地体验K8s

## 安装(minikube)

- 二进制下载

  ```shell
  curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-arm64
  sudo install minikube-darwin-arm64 /usr/local/bin/minikube
  ```

- Homebrew下载

  ```shell
  brew install minikube
  ```

## 启动

```shell
minikube start （如果需要多映射几个端口 增加--port参数）
```

⚠️ 这里踩坑一次（启动失败 Docker版本 4.34.0），更新成最新版就行了

## 部署程序

启用 ingress 附加组件

```shell
minikube addons enable ingress
```

以下示例创建简单的 echo-server 服务和一个 Ingress 对象以路由到这些服务。

```shell
kind: Pod
apiVersion: v1
metadata:
  name: foo-app
  labels:
    app: foo
spec:
  containers:
    - name: foo-app
      image: 'kicbase/echo-server:1.0'
---
kind: Service
apiVersion: v1
metadata:
  name: foo-service
spec:
  selector:
    app: foo
  ports:
    - port: 8080
---
kind: Pod
apiVersion: v1
metadata:
  name: bar-app
  labels:
    app: bar
spec:
  containers:
    - name: bar-app
      image: 'kicbase/echo-server:1.0'
---
kind: Service
apiVersion: v1
metadata:
  name: bar-service
spec:
  selector:
    app: bar
  ports:
    - port: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
spec:
  rules:
    - http:
        paths:
          - pathType: Prefix
            path: /foo
            backend:
              service:
                name: foo-service
                port:
                  number: 8080
          - pathType: Prefix
            path: /bar
            backend:
              service:
                name: bar-service
                port:
                  number: 8080
---
```

应用内容

```shell
kubectl apply -f https://storage.googleapis.com/minikube-site-examples/ingress-example.yaml
```

等待入口地址

```shell
kubectl get ingress
NAME              CLASS   HOSTS   ADDRESS          PORTS   AGE
example-ingress   nginx   *       <your_ip_here>   80      5m45s
```

**Docker Desktop 用户注意事项**

要使入口正常工作，您需要打开一个新的终端窗口并运行 `minikube tunnel`，然后在以下步骤中使用 `127.0.0.1` 替换 `<ip_from_above>`。

现在验证入口是否正常工作

```shell
$ curl <ip_from_above>/foo
Request served by foo-app
...

$ curl <ip_from_above>/bar
Request served by bar-app
...
```

## 其他操作

暂停 Kubernetes，而不影响已部署的应用程序：

```shell
minikube pause
```

取消暂停已暂停的实例：

```shell
minikube unpause
```

停止集群：

```shell
minikube stop
```

更改默认内存限制（需要重新启动）：

```shell
minikube config set memory 9001
```

浏览易于安装的 Kubernetes 服务目录：

```shell
minikube addons list
```

创建运行较旧 Kubernetes 版本的第二个集群：

```shell
minikube start -p aged --kubernetes-version=v1.16.1
```

删除所有 minikube 集群：

```shell
minikube delete --all
```





# 部署项目

### 1. 创建 MySQL 部署和服务

首先，你需要为 MySQL 创建一个部署和服务。可以使用以下 YAML 文件（`mysql-deployment.yaml`）：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: your_root_password
        - name: MYSQL_DATABASE
          value: your_database_name
        ports:
        - containerPort: 3306
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
spec:
  type: ClusterIP
  ports:
  - port: 3306
    targetPort: 3306
  selector:
    app: mysql
```

### 2. 创建 Redis 部署和服务

同样地，为 Redis 创建一个部署和服务（`redis-deployment.yaml`）：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:latest
        ports:
        - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
spec:
  type: ClusterIP
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    app: redis
```

### 3. 部署你的 ASP.NET Core 应用

确保你的应用能够连接到 MySQL 和 Redis。创建一个应用的部署和服务文件（`app-deployment.yaml`）：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: your-app-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: your-app
  template:
    metadata:
      labels:
        app: your-app
    spec:
      containers:
      - name: your-app
        image: your-app-image:latest
        env:
        - name: ConnectionStrings__DefaultConnection
          value: "Server=mysql-service;Database=your_database_name;User=root;Password=your_root_password;"
        - name: Redis__Configuration
          value: "redis-service:6379"
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: your-app-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: your-app
```

### 4. 应用配置

使用以下命令应用配置：

```shell
kubectl apply -f mysql-deployment.yaml
kubectl apply -f redis-deployment.yaml
kubectl apply -f app-deployment.yaml
```

### 5. 检查部署状态

可以使用以下命令检查部署状态：

```shell
kubectl get deployments
kubectl get services
```

### 6. 访问应用

如果你使用的是 LoadBalancer 类型的服务，可以通过外部 IP 访问应用。如果是 ClusterIP，可以使用 `kubectl port-forward` 命令进行访问。

```bash
kubectl port-forward svc/mysql-service 3306:3306
```

### 注意事项

- 根据需要修改 MySQL 和 Redis 的配置。
- 确保你的应用中使用的连接字符串与服务名称一致。
- 如果需要持久化数据，考虑使用 StatefulSet 和 Persistent Volume。
