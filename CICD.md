# CICD

- 了解 CI/CD 工作流相关概念

- GitLab工作流 （Verify，Package，Release）

  ![img](assets/5c611adaf92004e6845665ea1e6ed572.png)

  1. Verify
     - 通过持续集成自动构建和测试应用程序
     - 使用GitLab代码质量（GitLab Code Quality）分析源代码质量
     - 通过浏览器性能测试（Browser Performance Testing）确定代码更改对性能影响
     - 执行一系列测试（Junit Test，Container Scanning，Dependency Scaning等）
  2. Packeage
     - Container Registry：存储 Docker 镜像
     - NPM Registry：存储 NPM 包
     - Maven Registry：存储 Maven Artifacts
  3. Release
     - Git Pages：部署静态网站
     - Canary：灰度发布
     - Feature Flags之后部署功能
     - GitLab Releases：发布说明添加到任意Git Tag
     - Deploy Boards：查看在Kubernetes上运行的每个CI环境的当前运行状况和状态
     - Auto Deploy：将应用程序部署到Kubernetes集群中的生产环境

- 创建管线：在gitlab仓库根目录创建一个**.gitlab-ci.yml**文件

  1. Basic结构模板

     ```yaml
     stages:
       - build
       - test
       - deploy
     
     image: alpine
     
     build_a:
       stage: build
       script:
         - echo "This job builds something."
     
     build_b:
       stage: build
       script:
         - echo "This job builds something else."
     
     test_a:
       stage: test
       script:
         - echo "This job tests something. It will only run when all jobs in the"
         - echo "build stage are complete."
     
     test_b:
       stage: test
       script:
         - echo "This job tests something else. It will only run when all jobs in the"
         - echo "build stage are complete too. It will start at about the same time as test_a."
     
     deploy_a:
       stage: deploy
       script:
         - echo "This job deploys something. It will only run when all jobs in the"
         - echo "test stage complete."
     
     deploy_b:
       stage: deploy
       script:
         - echo "This job deploys something else. It will only run when all jobs in the"
         - echo "test stage complete. It will start at about the same time as deploy_a."
     ```

     

  2. DAG结构模板

     ```yaml
     stages:
       - build
       - test
       - deploy
     
     image: alpine
     
     build_a:
       stage: build
       script:
         - echo "This job builds something quickly."
     
     build_b:
       stage: build
       script:
         - echo "This job builds something else slowly."
     
     test_a:
       stage: test
       needs: [build_a]
       script:
         - echo "This test job will start as soon as build_a finishes."
         - echo "It will not wait for build_b, or other jobs in the build stage, to finish."
     
     test_b:
       stage: test
       needs: [build_b]
       script:
         - echo "This test job will start as soon as build_b finishes."
         - echo "It will not wait for other jobs in the build stage to finish."
     
     deploy_a:
       stage: deploy
       needs: [test_a]
       script:
         - echo "Since build_a and test_a run quickly, this deploy job can run much earlier."
         - echo "It does not need to wait for build_b or test_b."
     
     deploy_b:
       stage: deploy
       needs: [test_b]
       script:
         - echo "Since build_b and test_b run slowly, this deploy job will run much later."
     ```

     

  3. Child / Parent模板

     - Parent

       ```yaml
       stages:
         - triggers
       
       trigger_a:
         stage: triggers
         trigger:
           include: a/.gitlab-ci.yml
         rules:
           - changes:
               - a/*
       
       trigger_b:
         stage: triggers
         trigger:
           include: b/.gitlab-ci.yml
         rules:
           - changes:
               - b/*
       ```

       

     - Child

       - **/a/.gitlab-ci.yml**

         ```yaml
         stages:
           - build
           - test
           - deploy
         
         image: alpine
         
         build_a:
           stage: build
           script:
             - echo "This job builds something."
         
         test_a:
           stage: test
           needs: [build_a]
           script:
             - echo "This job tests something."
         
         deploy_a:
           stage: deploy
           needs: [test_a]
           script:
             - echo "This job deploys something."
         ```

         

       - **/b/.gitlab-ci.yml**

         ```yaml
         stages:
           - build
           - test
           - deploy
         
         image: alpine
         
         build_b:
           stage: build
           script:
             - echo "This job builds something else."
         
         test_b:
           stage: test
           needs: [build_b]
           script:
             - echo "This job tests something else."
         
         deploy_b:
           stage: deploy
           needs: [test_b]
           script:
             - echo "This job deploys something else."
         ```

         