# Octopus

## 基础介绍

1. **Octopus Deploy**：Octopus Deploy 提供了免费试用，可以轻松开始探索平台。Octopus Cloud 是使用 Octopus Deploy 的最简便方式，它提供完全托管的服务。如果需要自行管理的持续交付 (CD) 解决方案，可以下载 Octopus Server 并在本地运行。

2. **Octopus 在软件交付流程中的作用**：Octopus Deploy 专注于软件的发布和部署，并不涉及整个软件交付流程。它与主要的持续集成 (CI) 系统兼容，假设用户已有 CI 系统，并提供深度集成。

3. **项目、环境和发布**：Octopus 的仪表板展示了三个主要构建块：项目（要部署的应用程序）、环境（应用程序部署的地点，如开发、测试和生产环境）和发布（部署所需内容的集合）。

4. **部署过程**：每个项目中都可以配置部署过程，这类似于部署项目的配方，定义了每次部署时要执行的步骤。

5. **变量**：Octopus 支持高级变量和作用域管理，允许根据部署范围调整应用程序的配置文件。

6. **基础设施**：Octopus 将部署目标（如机器和服务）组织成环境组，典型的环境包括开发、测试和生产。

7. **生命周期**：定义项目时需要选择一个生命周期，生命周期定义了项目在不同环境之间部署的规则。

8. **Runbook 自动化**：Octopus Runbooks 可用于自动化常规维护和紧急操作任务，如基础设施配置、数据库管理和网站故障恢复。

9. **租户**：Octopus 中的租户允许用户创建特定于客户的部署管道，而无需复制项目配置。

10. **空间**：对于拥有多个团队的大型组织，可以使用 Spaces 功能为每个团队提供单独的空间，以便管理各自的项目、环境和基础设施，同时保持资产的独立性。

    

## 入门教程

1. **设置 Octopus Deploy 实例**：
   - 使用 Octopus Cloud，由 Octopus 托管实例并连接到用户的服务器。
   - 在 Windows Server 上自托管，下载 MSI 并安装在带 SQL Server 后端的 Windows Server 上。
   - 作为 Docker 容器自托管，同样需要免费许可证。

2. **登录和开始**：登录 Octopus 实例后，系统会引导用户开始部署第一个应用程序。

3. **添加项目**：创建项目以管理软件应用和服务。为项目命名并保存。

4. **添加环境**：添加环境用于将基础设施组织成代表不同部署阶段的组，如开发、测试和生产环境。

5. **项目问卷**：填写简短的调查问卷，帮助 Octopus 团队了解客户使用的技术，从而改进平台。

6. **创建部署流程**：定义部署软件的步骤。例如，对于简单的 Hello World 部署，可以配置一个步骤来打印 "Hello World"。

7. **发布和部署**：创建一个可以部署到环境的版本，版本是部署流程和相关资产（如包、脚本、变量）的快照。

8. **部署成功**：完成第一次部署后，系统会提示成功。



## Kubernetes资源管理

在Kubernetes集群中，应用的核心是**Deployments**、**Services**等资源，它们负责管理应用的容器、负载均衡、扩展性等。Octopus的Kubernetes资源配置步骤可以帮助你定义、部署和管理这些资源，具体步骤包括：

1. **定义资源模板**：通过YAML文件定义Kubernetes的核心资源，如Deployments、Services等。模板可以根据环境动态调整。

2. **资源部署与更新**：Octopus会在部署过程中将配置文件应用到Kubernetes集群中，创建或更新相应资源。

3. **蓝绿部署与健康检查**：支持蓝绿部署策略，结合Kubernetes探针（Probes）进行健康检查，确保新版本部署不会影响现有用户。

4. **ConfigMap与Secrets**：通过Octopus，你可以轻松管理敏感配置，如API密钥或数据库连接信息，并安全地传递给应用。

5. **安全与版本管理**：Kubernetes资源的安全上下文和版本控制确保应用的安全性和部署的一致性。

   

## Kubernetes Ingress资源管理

在完成了应用程序的部署之后，管理外部流量的路由就变得尤为重要。**Ingress资源**可以用来控制外部HTTP和HTTPS流量的进入方式，将流量精准地引导到内部的Kubernetes服务。配置Ingress资源时，Octopus主要负责以下几个方面：

1. **流量路由与负载均衡**：通过Ingress规则，可以根据请求的URL、路径或主机名将外部流量定向到不同的Kubernetes服务，实现负载均衡。

2. **SSL终止与安全性**：Ingress能够处理SSL/TLS证书，保证服务的通信安全性。

3. **简化流量管理**：通过一个Ingress资源，用户可以集中管理多个服务的流量规则，而不需要单独为每个服务编写复杂的配置。

4. **集成监控与日志**：Ingress集中的流量管理方式还方便了与监控和日志工具的集成，提升了可观察性。



# Kubernetes 回滚操作

### 1. 现有部署过程
- **部署 MySQL 容器**：首先部署数据库容器以支持应用的后端需求。
- **部署 PetClinic Web**：接着部署应用的前端容器。
- **运行 Flyway 数据库迁移作业**：用于更新数据库架构和数据。
- **验证部署**：确保所有服务正常运行。
- **通知利益相关者**：部署完成后，通知团队成员和相关人员。

### 2. 零配置回滚
- **回滚至先前版本**：通过查找想要回滚的版本，并点击环境旁的 **REDEPLOY** 按钮实现。
- **快照机制**：每次创建版本时，都会生成一个包含部署流程、项目变量、引用变量集和包版本的快照。
- **重新部署**：通过重新部署先前的版本，可以恢复到该版本创建时的部署流程状态。

### 3. 简单回滚过程
- **计算部署模式**：使用 Octopus Deploy 提供的步骤模板，比较当前环境的版本号和正在部署的版本号，以确定是部署还是回滚。
- **跳过数据库步骤**：在回滚过程中，跳过 MySQL 容器的部署和 Flyway 数据库迁移作业。
- **阻止发布进度**：在回滚时，使用 Octopus Deploy 的 API 阻止回滚版本的进一步部署。

### 4. 复杂回滚过程
- **计算部署模式**：与简单回滚过程相同，确定当前操作是部署还是回滚。
- **回滚原因**：在回滚过程中，通过手动干预步骤要求用户提供回滚的原因。
- **部署 PetClinic Web**：在 Kubernetes 中部署应用的 Web 前端。
- **运行 Flyway 作业**（在回滚时跳过）：在回滚过程中跳过数据库迁移作业。
- **验证部署**：确保应用在回滚后仍然按预期运行。
- **通知利益相关者**：在回滚操作完成后通知相关人员。
- **回滚到 PetClinic Web 的上一个版本**：使用 Kubernetes 的修订历史功能，找到并回滚到 PetClinic Web 的上一个版本。
- **阻止发布进度**：在回滚时，使用 Octopus Deploy 的 API 阻止回滚版本的进一步部署。

### 5. 回滚到上一个版本的具体步骤
- **使用 Kubernetes 修订历史**：通过 `kubectl rollout history deployment.v1.apps/<deployment-name>` 命令查看所有部署修订。
- **确定要回滚的版本**：在修订历史中找到需要回滚到的版本。
- **执行回滚命令**：使用 `kubectl rollout undo` 命令回滚到指定的修订版本。

### 6. 选择回滚策略
- **简单回滚策略**：适用于大多数情况，不需要复杂的配置。
- **复杂回滚策略**：如果简单回滚策略不满足需求，可以采用更复杂的回滚策略。
