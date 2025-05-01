# 构建与 CICD 相结合的最佳实践测试自动化流程：第二部分 —— GitHub 集成与基于基础设施即代码的云部署

作者/译者: [Rob McBryde](https://medium.com/@robert_mcbryde?source=post_page---byline--5ada5c18a4b8---------------------------------------)/溜的一比

文章来源: Mediam

文章地址：https://medium.com/@robert_mcbryde/building-a-best-practice-test-automation-pipeline-with-ci-cd-part-2-github-integration-and-eb6fb5545f73

​![image](assets/image-20250331092740-fgfq5lz.png)​

## 介绍

在我们系列文章的第一部分中，我们通过构建一个具有强大测试能力的 Spring Boot REST API，为我们的测试自动化流程奠定了坚实的基础。我们实现了 /hello 端点，并通过 /health 和 /info 端点启用了执行器支持的健康监测。我们使用 maven-checkstyle-plugin 和 Google 的风格指南进行了静态代码分析，为我们的控制器编写了单元测试，使用 Docker 容器化了我们的应用，并使用 Testcontainers 和 REST Assured 对我们的容器化应用运行 API 测试。

虽然这些功能使我们能够在本地开发和测试我们的应用，但一个生产就绪的应用需要更多：源代码控制、自动化的 CI/CD 流水线以及云环境部署。在本文中，我们将通过使用 Git、GitHub Actions、Terraform 和 Google Cloud Platform (GCP) 实现这些关键实践，将我们的应用提升到一个新的水平。

警告：这是一篇长文，但涵盖了大量内容。

## 先决条件

在深入本教程之前，你需要在你的机器上正确配置 Git、Terraform 和 Google Cloud CLI。这包括安装 Git 并设置你的用户信息以进行提交。我不会在本文中介绍 Git 或 Terraform 的安装和配置过程，因为网上有许多优秀的资源提供了针对不同操作系统的详细说明。我之前曾写过如何在 Mac 上设置 Google Cloud CLI 的文章。

## 我们将完成的任务

在本教程结束时，我们将：

* 为我们的项目添加基于 Git 的源代码控制
* 创建一个 GitHub 仓库并配置它以满足我们的 CI/CD 需求
* 使用 GitHub Actions 实现一个 CI/CD 流水线
* 使用 Terraform 设置基础设施即代码，以配置我们的 GCP 资源
* 将我们的 Spring Boot 应用部署到 Google Cloud Run 的三个独立环境（开发、预生产和生产）中
* 实现冒烟测试以验证我们的部署

让我们开始从本地开发到云部署的旅程吧！

## 使用 Git 设置源代码控制

版本控制对于任何软件项目都是必不可少的。Git 允许我们跟踪更改、有效协作，并维护我们的代码库的历史记录。让我们初始化我们的项目作为一个 Git 仓库。

### 初始化 Git 仓库

首先，导航到我们的项目根目录并初始化一个 Git 仓库：

```bash
cd automation-demo
git init
```

### 创建 .gitignore 文件

接下来，让我们创建一个 .gitignore 文件，以排除不应在我们的仓库中跟踪的文件和目录。在你的项目根目录中创建此文件：

```
HELP.md
target/
!.mvn/wrapper/maven-wrapper.jar
!**/src/main/**/target/
!**/src/test/**/target/

### STS ###
.apt_generated
.classpath
.factorypath
.project
.settings
.springBeans
.sts4-cache

### IntelliJ IDEA ###
.idea
*.iws
*.iml
*.ipr

### NetBeans ###
/nbproject/private/
/nbbuild/
/dist/
/nbdist/
/.nb-gradle/
build/
!**/src/main/**/build/
!**/src/test/**/build/

### VS Code ###
.vscode/

# Terraform
*.tfvars
*.tfstate
*.tfstate.*
.terraform/
.terraform.lock.hcl
!*.tfvars.example

# Specific tfvars exclusion
terraform/service-accounts/terraform.tfvars
```

我们创建的 .gitignore 文件涵盖了不应提交到我们仓库的广泛文件范围，包括构建输出、特定于 IDE 的文件、本地配置和生成的 Terraform 文件。这保持了我们仓库的整洁，防止共享敏感或环境特定的信息。

### 初始提交

现在，让我们添加现有的文件并进行我们的第一次提交：

```bash
git add .
git commit -m "Initial commit: Spring Boot API with testing infrastructure"
```

这在 Git 中捕获了我们项目的当前状态，创建了一个我们可以随时返回的参考点，如果需要的话。

### 设置 GitHub 仓库

现在我们已经设置了本地 Git 仓库，让我们在 GitHub 上创建一个远程仓库。

1. 登录到 GitHub
2. 点击右上角的“+”图标，选择“新建仓库”
3. 命名你的仓库（例如，“automation-demo”）
4. 添加描述（可选）
5. 根据你的需求选择“公开”或“私有”可见性
6. 跳过初始化选项，因为我们已经本地有文件
7. 点击“创建仓库”

创建仓库后，将你的本地仓库连接到远程 GitHub 仓库（注意在 URL 中添加你的 GitHub 用户名）：

```bash
git remote add origin https://github.com/<your_user_name>/automation-demo.git
git branch -M main
git push -u origin main
```

上述命令在你的本地仓库和 GitHub 仓库之间建立了连接，重命名你的主分支（如果它还不叫“main”），并将你的代码推送到 GitHub。现在你的代码已经安全备份，并准备好进行协作开发 🚀。

## 使用 GitHub Actions 实现 CI/CD

现在我们的代码已经在 GitHub 上，让我们使用 GitHub Actions 设置一个 CI/CD 流水线。这将自动化我们的构建、测试和部署过程。

### 常见的开发策略

在我们开始构建 CI/CD 流水线之前，我们需要决定我们的软件开发策略。这些策略旨在有效管理代码更改，减少集成问题，并在稳定性和快速迭代之间取得平衡。正确的策略取决于团队规模、发布节奏和部署过程等因素。在这个系列中，我们将使用基于主干的开发（Trunk-Based Development，TBD）。我在下面总结了我对 TBD 的看法，并解释了为什么我认为它适合我们的项目。

附注：在移动开发中，我倾向于使用 GitFlow，因为苹果和谷歌的应用审查阶段有此需求。我不会在本文中深入探讨，但如果你正在启动一个移动项目，我建议将 GitFlow 作为 TBD 的替代方案进行研究。

#### 基于主干的开发（TBD）

* 开发者直接提交到主分支（主干）或使用短生命周期的功能分支，这些分支快速合并（通常在一到两天内），减少了大型、长期分支导致合并地狱的风险。
* 持续集成（CI）运行自动化测试以确保稳定性。
* 频繁的小型发布最小化风险并启用快速反馈。
* 适用于快节奏、CI/CD 驱动的环境，如 Web 开发、云服务和 DevOps 团队。
* 自动化测试和功能标志允许安全、渐进式的发布。
* 鼓励快速迭代和更快的上市时间。
* 在基于云的和 SaaS 环境中表现良好，这些环境中的部署频繁。

#### 带有部署门的基于主干的开发

我们将实现一个跨越多个环境的 CI/CD 流水线，包括开发、预生产和生产环境。这是一个非常常见的模式，与 DevOps 最佳实践相吻合。

以下是它通常的工作方式：

* **单一源分支**：所有开发都在一个分支中进行，在我们的情况下是主分支。
* **短生命周期的功能分支**：开发者为更改创建短生命周期的功能分支，然后快速合并回主分支。
* **带有门的部署流水线**：一个单一的流水线按顺序部署到所有环境，在环境之间有审批门。

例如，我们对代码库进行更改并在本地运行以确认它符合我们的预期。然后我们将此更改推送到 GitHub 的主分支。这将自动触发我们的 CI/CD 流水线，该流水线将运行我们的静态分析、单元测试、创建我们的应用的 Docker 镜像并创建我们应用的容器化版本。

然后它将在运行中的容器上运行我们的 API 测试，如果所有测试都通过，它将自动将我们的 Spring Boot 应用部署到托管在 Google Cloud 的开发环境，并运行我们的自动化冒烟测试以确保它已成功部署。我们的流水线将在继续部署到预生产环境之前暂停，等待我们的手动批准。

在现实环境中，这允许我们在与同事共享的开发环境中进行进一步的手动测试，以进一步验证我们引入的更改，并确保它们在与其他团队的应用一起运行的更广泛环境中正常工作。

一旦我们批准了网关，流水线将继续将我们的应用部署到 Google Cloud 的预生产环境中，并在完成后再次运行自动化冒烟测试。通常，手动 QA 同事随后可以在预生产环境中进行进一步的测试，以验证我们部署的代码是否符合验收标准和用户需求。

再次，我们的流水线在我们的生产网关处暂停，等待进一步的审查和手动确认，以决定是否继续将部署推进到我们的生产环境，该环境同样托管在 Google Cloud 中。

### 设置 GitHub 环境保护规则

在 GitHub Actions 中实现部署门的关键是使用带有保护规则的环境。以下是设置方法：

1. 在 GitHub 中，进入你的仓库设置
2. 点击侧边栏中的“环境”
3. 创建三个环境：开发、预生产和生产
4. 为每个环境配置适当的保护规则：

    * 开发：最小保护（可选审批）
    * 预生产：必需的审批者（1-2 人）
    * 生产：必需的审批者（2 人或更多）

当工作流运行时：

* 它将自动部署到开发环境
* 然后它将在预生产部署前暂停，需要指定的审批者的批准
* 最后，它将在生产部署前再次暂停，可能需要更高级审批者的批准

### 这种方法的优势

* **简化的分支策略**：所有开发都在一个分支中进行，减少了合并冲突和复杂性
* **持续部署**：更改自动部署到开发环境以进行早期测试
* **手动审批门**：在部署到更高环境之前需要人工判断
* **一致的构建工件**：相同的构建工件穿过所有环境，减少了“在我的机器上可以工作”的问题
* **可审计的部署过程**：审批记录在 GitHub 中，创建了审计跟踪

### 创建 GitHub Actions 工作流文件

GitHub Actions 工作流在存储在 .github/workflows 目录中的 YAML 文件中定义。让我们在项目的根目录中创建此目录和工作流文件：

```bash
mkdir -p .github/workflows
```

现在，让我们创建我们的工作流文件，名为 ci-cd-pipeline.yml，并将其放置在我们刚刚创建的工作流目录中。正是这段代码定义了我们的 CI/CD 流水线中的作业和阶段，所有这些都以代码形式记录下来。我将完整地提供这段代码，然后将其分解为更小的部分来解释它在做什么。

```yaml
name: CI/CD Pipeline with Gates

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:  # Allows manual triggering

jobs:
  build-and-test:
    name: Build and Test
    runs-on: ubuntu-latest

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Set up JDK 17
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'
        cache: maven

    - name: Validate Maven project
      run: mvn validate

    - name: Static Code Analysis
      run: mvn checkstyle:check

    - name: Build with Maven
      run: mvn -B package -DskipTests

    - name: Run Unit Tests
      run: mvn test

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Build Docker image for testing
      run: docker build -t spring-boot-api:test .

    - name: Run API Tests with Testcontainers
      run: mvn verify -Dskip.unit.tests=true

    - name: Upload build artifact
      uses: actions/upload-artifact@v4
      with:
        name: app-jar
        path: target/*.jar
        retention-days: 1

  deploy-to-dev:
    name: Deploy to Development
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && (github.event_name == 'push' || github.event_name == 'workflow_dispatch')
    environment: development 
    env:
      ENVIRONMENT: dev
      PROJECT_ID: ${{ secrets.GCP_PROJECT_DEV }}
      SERVICE_NAME: spring-boot-api-dev
      REGION: us-central1

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Download build artifact
      uses: actions/download-artifact@v4
      with:
        name: app-jar
        path: target

    - name: Setup Cloud SDK
      uses: google-github-actions/setup-gcloud@v1
      with:
        project_id: ${{ env.PROJECT_ID }}

    - name: Authenticate to Google Cloud
      uses: google-github-actions/auth@v1
      with:
        credentials_json: ${{ secrets.GCP_SA_KEY_DEV }}

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Build and Push Docker image
      run: |
        # Configure Docker for Artifact Registry
        gcloud auth configure-docker ${{ env.REGION }}-docker.pkg.dev
    
        # Build with environment-specific Spring profile
        docker build \
          --build-arg SPRING_PROFILES_ACTIVE=${{ env.ENVIRONMENT }} \
          -t ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/spring-boot-api/${{ env.SERVICE_NAME }}:${{ github.sha }} \
          -t ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/spring-boot-api/${{ env.SERVICE_NAME }}:latest .
      
        docker push ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/spring-boot-api/${{ env.SERVICE_NAME }}:${{ github.sha }}
        docker push ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/spring-boot-api/${{ env.SERVICE_NAME }}:latest

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2
      with:
        terraform_version: 1.5.0

    - name: Initialize Terraform
      run: |
        cd terraform/environments/dev
        terraform init

    - name: Apply Terraform
      run: |
        cd terraform/environments/dev
        terraform apply -auto-approve
      env:
        TF_VAR_project_id: ${{ env.PROJECT_ID }}
        TF_VAR_region: ${{ env.REGION }}
        TF_VAR_service_name: ${{ env.SERVICE_NAME }}
        TF_VAR_environment: ${{ env.ENVIRONMENT }}

    - name: Deploy to Cloud Run
      uses: google-github-actions/deploy-cloudrun@v1
      with:
        service: ${{ env.SERVICE_NAME }}
        region: ${{ env.REGION }}
        image: ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/spring-boot-api/${{ env.SERVICE_NAME }}:${{ github.sha }}

    - name: Smoke Test Development
      run: |
        export SERVICE_URL=$(gcloud run services describe ${{ env.SERVICE_NAME }} --region ${{ env.REGION }} --format 'value(status.url)')
        echo "DEV Service URL: $SERVICE_URL"
    
        # Wait for service to be fully deployed
        sleep 30
    
        # Test health endpoint
        HEALTH_STATUS=$(curl -s -o /dev/null -w "%{http_code}" $SERVICE_URL/actuator/health)
        if [ "$HEALTH_STATUS" != "200" ]; then
          echo "Health check failed with status $HEALTH_STATUS"
          exit 1
        fi
    
        # Test hello endpoint
        HELLO_RESPONSE=$(curl -s $SERVICE_URL/api/hello)
        if [[ "$HELLO_RESPONSE" != *"Hello"* ]]; then
          echo "Hello endpoint check failed. Response: $HELLO_RESPONSE"
          exit 1
        fi
        echo "DEV deployment smoke test passed successfully!"

  deploy-to-staging:
    name: Deploy to Staging
    needs: deploy-to-dev
    runs-on: ubuntu-latest
    environment: staging  
    env:
      ENVIRONMENT: staging
      PROJECT_ID: ${{ secrets.GCP_PROJECT_STAGING }}
      SERVICE_NAME: spring-boot-api-staging
      REGION: us-central1

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Download build artifact
      uses: actions/download-artifact@v4
      with:
        name: app-jar
        path: target

    - name: Setup Cloud SDK
      uses: google-github-actions/setup-gcloud@v1
      with:
        project_id: ${{ env.PROJECT_ID }}

    - name: Authenticate to Google Cloud
      uses: google-github-actions/auth@v1
      with:
        credentials_json: ${{ secrets.GCP_SA_KEY_STAGING }}

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Build and Push Docker image
      run: |
        # Configure Docker for Artifact Registry
        gcloud auth configure-docker ${{ env.REGION }}-docker.pkg.dev
    
        # Build with environment-specific Spring profile
        docker build \
          --build-arg SPRING_PROFILES_ACTIVE=${{ env.ENVIRONMENT }} \
          -t ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/spring-boot-api/${{ env.SERVICE_NAME }}:${{ github.sha }} \
          -t ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/spring-boot-api/${{ env.SERVICE_NAME }}:latest .
      
        docker push ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/spring-boot-api/${{ env.SERVICE_NAME }}:${{ github.sha }}
        docker push ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/spring-boot-api/${{ env.SERVICE_NAME }}:latest

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2
      with:
        terraform_version: 1.5.0

    - name: Initialize Terraform
      run: |
        cd terraform/environments/staging
        terraform init

    - name: Apply Terraform
      run: |
        cd terraform/environments/staging
        terraform apply -auto-approve
      env:
        TF_VAR_project_id: ${{ env.PROJECT_ID }}
        TF_VAR_region: ${{ env.REGION }}
        TF_VAR_service_name: ${{ env.SERVICE_NAME }}
        TF_VAR_environment: ${{ env.ENVIRONMENT }}

    - name: Deploy to Cloud Run
      uses: google-github-actions/deploy-cloudrun@v1
      with:
        service: ${{ env.SERVICE_NAME }}
        region: ${{ env.REGION }}
        image: ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/spring-boot-api/${{ env.SERVICE_NAME }}:${{ github.sha }}

    - name: Smoke Test Staging
      run: |
        export SERVICE_URL=$(gcloud run services describe ${{ env.SERVICE_NAME }} --region ${{ env.REGION }} --format 'value(status.url)')
        echo "STAGING Service URL: $SERVICE_URL"
    
        # Wait for service to be fully deployed
        sleep 30
    
        # Test health endpoint
        HEALTH_STATUS=$(curl -s -o /dev/null -w "%{http_code}" $SERVICE_URL/actuator/health)
        if [ "$HEALTH_STATUS" != "200" ]; then
          echo "Health check failed with status $HEALTH_STATUS"
          exit 1
        fi
    
        # Test hello endpoint
        HELLO_RESPONSE=$(curl -s $SERVICE_URL/api/hello)
        if [[ "$HELLO_RESPONSE" != *"Hello"* ]]; then
          echo "Hello endpoint check failed. Response: $HELLO_RESPONSE"
          exit 1
        fi
        echo "STAGING deployment smoke test passed successfully!"

  deploy-to-production:
    name: Deploy to Production
    needs: deploy-to-staging
    runs-on: ubuntu-latest
    environment: production  
    env:
      ENVIRONMENT: prod
      PROJECT_ID: ${{ secrets.GCP_PROJECT_PROD }}
      SERVICE_NAME: spring-boot-api
      REGION: us-central1

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Download build artifact
      uses: actions/download-artifact@v4
      with:
        name: app-jar
        path: target

    - name: Setup Cloud SDK
      uses: google-github-actions/setup-gcloud@v1
      with:
        project_id: ${{ env.PROJECT_ID }}

    - name: Authenticate to Google Cloud
      uses: google-github-actions/auth@v1
      with:
        credentials_json: ${{ secrets.GCP_SA_KEY_PROD }}

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Build and Push Docker image
      run: |
        # Configure Docker for Artifact Registry
        gcloud auth configure-docker ${{ env.REGION }}-docker.pkg.dev
    
        # Build with environment-specific Spring profile
        docker build \
          --build-arg SPRING_PROFILES_ACTIVE=${{ env.ENVIRONMENT }} \
          -t ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/spring-boot-api/${{ env.SERVICE_NAME }}:${{ github.sha }} \
          -t ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/spring-boot-api/${{ env.SERVICE_NAME }}:latest .
      
        docker push ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/spring-boot-api/${{ env.SERVICE_NAME }}:${{ github.sha }}
        docker push ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/spring-boot-api/${{ env.SERVICE_NAME }}:latest

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2
      with:
        terraform_version: 1.5.0

    - name: Initialize Terraform
      run: |
        cd terraform/environments/prod
        terraform init

    - name: Apply Terraform
      run: |
        cd terraform/environments/prod
        terraform apply -auto-approve
      env:
        TF_VAR_project_id: ${{ env.PROJECT_ID }}
        TF_VAR_region: ${{ env.REGION }}
        TF_VAR_service_name: ${{ env.SERVICE_NAME }}
        TF_VAR_environment: ${{ env.ENVIRONMENT }}

    - name: Deploy to Cloud Run
      uses: google-github-actions/deploy-cloudrun@v1
      with:
        service: ${{ env.SERVICE_NAME }}
        region: ${{ env.REGION }}
        image: ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/spring-boot-api/${{ env.SERVICE_NAME }}:${{ github.sha }}

    - name: Smoke Test Production
      run: |
        export SERVICE_URL=$(gcloud run services describe ${{ env.SERVICE_NAME }} --region ${{ env.REGION }} --format 'value(status.url)')
        echo "PRODUCTION Service URL: $SERVICE_URL"
    
        # Wait for service to be fully deployed
        sleep 30
    
        # Test health endpoint
        HEALTH_STATUS=$(curl -s -o /dev/null -w "%{http_code}" $SERVICE_URL/actuator/health)
        if [ "$HEALTH_STATUS" != "200" ]; then
          echo "Health check failed with status $HEALTH_STATUS"
          exit 1
        fi
    
        # Test hello endpoint
        HELLO_RESPONSE=$(curl -s $SERVICE_URL/api/hello)
        if [[ "$HELLO_RESPONSE" != *"Hello"* ]]; then
          echo "Hello endpoint check failed. Response: $HELLO_RESPONSE"
          exit 1
        fi
        echo "PRODUCTION deployment smoke test passed successfully!"
```

我们的流水线代码包含大量代码！让我们分解它到底在做什么。

```yaml
name: CI/CD Pipeline with Gates

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:  # Allows manual triggering
```

我们的流水线首先定义了它应该何时运行。工作流在三个事件上触发：

* 当代码直接推送到主分支时
* 当针对主分支打开拉请求时（允许在合并前进行验证）
* 通过 GitHub 界面手动触发时

通过专注于主分支，我们强制执行基于主干的开发原则，即所有工作都围绕一个单一的、权威的真相来源进行。与基于分支的策略不同，不同的环境生活在不同的分支上，我们的方法将所有内容保持在一个地方，减少了复杂性和合并冲突。

### 作业 1：构建和测试

```yaml
build-and-test:
  name: Build and Test
  runs-on: ubuntu-latest

  steps:
  - name: Checkout repository
    uses: actions/checkout@v4

  - name: Set up JDK 17
    uses: actions/setup-java@v4
    with:
      java-version: '17'
      distribution: 'temurin'
      cache: maven

  - name: Validate Maven project
      run: mvn validate
  
  - name: Static Code Analysis
    run: mvn checkstyle:check

  - name: Build with Maven
    run: mvn -B package -DskipTests

  - name: Run Unit Tests
    run: mvn test

  - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
  
  - name: Build Docker image for testing
    run: docker build -t spring-boot-api:test .

  - name: Run API Tests with Testcontainers
    run: mvn verify -Dskip.unit.tests=true

  - name: Upload build artifact
    uses: actions/upload-artifact@v4
    with:
      name: app-jar
      path: target/*.jar
      retention-days: 1
```

流水线中的第一个作业专注于验证。在将代码部署到任何环境之前，我们需要确认我们的代码是可靠的。此作业在最新的 Ubuntu 运行器上运行，首先检出我们的代码并设置 Java 17。

后续步骤通过多个角度逐步验证我们的代码：

* 首先，我们验证 Maven 项目结构
* 然后，我们运行静态代码分析以捕获风格和潜在质量问题
* 接下来，我们构建应用以验证它是否正确编译
* 最后，我们运行单元测试以验证代码的功能正确性
* 然后我们进入基于容器的测试。这个序列设置 Docker，构建我们应用的测试镜像，并使用 Testcontainers 运行 API 测试。这是验证我们应用在容器化后是否正常工作的关键步骤，正如它在我们的部署环境中一样。
* 最后，我们保存构建工件以供后续作业使用。这一步对于我们的部署策略至关重要。通过在整个环境中保留完全相同的工件，我们减少了不一致性的可能性。这是基于主干部署带有门的关键优势 —— 经过验证的相同构建穿过所有环境。

### 作业 2：部署到开发环境

```yaml
deploy-to-dev:
  name: Deploy to Development
  needs: build-and-test
  runs-on: ubuntu-latest
  if: github.ref == 'refs/heads/main' && (github.event_name == 'push' || github.event_name == 'workflow_dispatch')
  environment: development
  env:
    ENVIRONMENT: dev
    PROJECT_ID: ${{ secrets.GCP_PROJECT_DEV }}
    SERVICE_NAME: spring-boot-api-dev
    REGION: us-central1

  steps:
    - name: Checkout repository
      uses: actions/checkout@v4
  
    - name: Download build artifact
      uses: actions/download-artifact@v4
      with:
        name: app-jar
        path: target
  
    - name: Setup Cloud SDK
      uses: google-github-actions/setup-gcloud@v1
      with:
        project_id: ${{ env.PROJECT_ID }}
  
    - name: Authenticate to Google Cloud
      uses: google-github-actions/auth@v1
      with:
        credentials_json: ${{ secrets.GCP_SA_KEY_DEV }}
  
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
  
    - name: Build and Push Docker image
      run: |
        # Configure Docker for Artifact Registry
        gcloud auth configure-docker ${{ env.REGION }}-docker.pkg.dev
    
        # Build with environment-specific Spring profile
        docker build \
          --build-arg SPRING_PROFILES_ACTIVE=${{ env.ENVIRONMENT }} \
          -t ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/spring-boot-api/${{ env.SERVICE_NAME }}:${{ github.sha }} \
          -t ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/spring-boot-api/${{ env.SERVICE_NAME }}:latest .
      
        docker push ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/spring-boot-api/${{ env.SERVICE_NAME }}:${{ github.sha }}
        docker push ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/spring-boot-api/${{ env.SERVICE_NAME }}:latest

    - name: Setup Terraform
    uses: hashicorp/setup-terraform@v2
    with:
      terraform_version: 1.5.0

    - name: Initialize Terraform
      run: |
        cd terraform/environments/dev
        terraform init
  
    - name: Apply Terraform
      run: |
        cd terraform/environments/dev
        terraform apply -auto-approve
      env:
        TF_VAR_project_id: ${{ env.PROJECT_ID }}
        TF_VAR_region: ${{ env.REGION }}
        TF_VAR_service_name: ${{ env.SERVICE_NAME }}
        TF_VAR_environment: ${{ env.ENVIRONMENT }}

    - name: Deploy to Cloud Run
    uses: google-github-actions/deploy-cloudrun@v1
    with:
      service: ${{ env.SERVICE_NAME }}
      region: ${{ env.REGION }}
      image: ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/spring-boot-api/${{ env.SERVICE_NAME }}:${{ github.sha }}

    - name: Smoke Test Development
      run: |
        export SERVICE_URL=$(gcloud run services describe ${{ env.SERVICE_NAME }} --region ${{ env.REGION }} --format 'value(status.url)')
        echo "DEV Service URL: $SERVICE_URL"
    
        # Wait for service to be fully deployed
        sleep 30
    
        # Test health endpoint
        HEALTH_STATUS=$(curl -s -o /dev/null -w "%{http_code}" $SERVICE_URL/actuator/health)
        if [ "$HEALTH_STATUS" != "200" ]; then
          echo "Health check failed with status $HEALTH_STATUS"
          exit 1
        fi
    
        # Test hello endpoint
        HELLO_RESPONSE=$(curl -s $SERVICE_URL/api/hello)
        if [[ "$HELLO_RESPONSE" != *"Hello"* ]]; then
          echo "Hello endpoint check failed. Response: $HELLO_RESPONSE"
          exit 1
        fi
        echo "DEV deployment smoke test passed successfully!"
```

这个作业开始了我们的部署阶段，从开发环境开始。几个关键元素定义了这个作业的目的：

* ​`needs: build-and-test`​ 确保只有在之前的验证作业成功后，此作业才会运行
* 条件 `if`​ 语句将部署限制在推送到主分支或手动触发时
* ​`environment: development`​ 行是关键 —— 它激活为开发环境配置的任何保护规则
* 环境变量设置此部署的上下文特定值

接下来，我们准备部署。这些步骤检索我们的配置和之前构建的工件。我们然后设置 Google Cloud SDK 并认证到我们的开发 GCP 项目。注意我们使用的是环境特定的凭据 —— 这在云提供者级别强制了环境之间的分离。

接下来是 Docker 镜像创建。这一步为开发环境构建我们的 Docker 镜像。注意使用了 `SPRING_PROFILES_ACTIVE=${{ env.ENVIRONMENT }}`​ —— 这确保我们的应用以开发配置运行。然后我们将此镜像推送到 Google 的 Artifact Registry，带有版本特定的标签（Git 提交 SHA）和最新标签。

下一节处理基础设施配置。使用 Terraform，我们在开发环境中配置或更新所需的基础设施。这种基础设施即代码的方法确保我们的云资源被一致地定义，并且如果需要，可以完全相同地重新创建。我们在 `terraform/environments/dev`​ 中使用了环境特定的 Terraform 配置。

最后，我们部署应用并验证其健康状况。部署步骤使用 Google Cloud Run 操作部署我们的容器。冒烟测试是关键 —— 它通过检查端点验证我们的部署是否实际正常工作。如果任何测试失败，流水线将停止，防止带有故障部署进入预生产环境。

### 作业 3：部署到预生产环境

预生产部署作业遵循与开发相似的模式，但有两个关键区别：

* 它依赖于开发部署的成功（`needs: deploy-to-dev`​）
* 它使用 `environment: staging`​ 规范

这是我们的部署门发挥作用的地方。在 GitHub 中，你可以配置预生产环境需要手动批准。当工作流到达这一点时，它会暂停并等待授权人员审查和批准部署到预生产环境。

剩余步骤镜像了开发部署，但使用了预生产特定的变量和密钥。这保持了环境隔离，同时使用相同的部署过程。

### 作业 4：部署到生产环境

生产部署代表了我们最后的部署门。它依赖于预生产部署的成功，并指定了生产环境，这通常有最严格的批准要求。组织通常需要多个批准者，有时在生产部署之前还需要一个等待期。

生产部署的步骤再次镜像了之前的环境，但使用了生产特定的变量。在许多组织中，生产可能会通过环境特定的 Terraform 配置应用额外的安全措施或扩展配置。

花点时间承认最后一个阶段内容很多，恭喜你坚持到现在！

### 完整的部署流程 —— 总结

退一步看全貌，我们的部署流程经过这些阶段：

* 代码合并或推送到主分支
* 代码被构建并经过彻底测试
* 如果测试通过，它会自动部署到开发环境
* 经过审查和批准后，它被部署到预生产环境
* 经过额外的审查和批准后，它被部署到生产环境

每个环境都使用相同的工件和过程进行部署，但带有环境特定的配置。环境之间的门确保了每个阶段都有适当的监督。

### Terraform：基础设施即代码 —— 它是什么？

本质上，Terraform 允许你将基础设施配置视为软件代码。这意味着你的整个 GCP 设置 —— 包括服务账户、Cloud Run 实例、存储桶和跨项目的权限 —— 都在具有清晰语法的文本文件中定义。这些文件可以：

* 与你的应用代码一起进行版本控制
* 通过与你的应用更改相同的 PR 流程进行审查
* 在应用于环境之前进行测试
* 在环境中一致地复制

### 为什么 Terraform 对现代云部署至关重要

* **一致性和可重复性**：使用 Terraform 的一个最大好处是确保环境的一致性。
* **文档即代码**：我们的 Terraform 文件作为我们基础设施的活文档。
* **变更管理和可审计性**：通过 Terraform 进行基础设施变更时，可以生成执行计划，显示将要进行的变更。
* **可扩展性和可维护性**：随着应用的增长，手动管理基础设施变得越来越容易出错和耗时。使用 Terraform，扩展基础设施 —— 无论是添加更多资源还是在新区域复制设置 —— 变成了修改代码而不是点击控制台界面。

### 设置 Terraform 作为基础设施即代码

Terraform 允许我们将基础设施定义为代码，使其版本化、可重复和自动化。我们的 Terraform 结构支持多环境部署，即开发、预生产和生产。

#### 步骤 1：创建 Terraform 目录结构

首先，我们需要为模块化的 Terraform 设置创建必要的目录。在你的项目根目录的终端中运行以下命令，以创建存储我们的 Terraform 代码的新目录：

```bash
mkdir -p terraform/modules/spring-boot-api
mkdir -p terraform/environments/dev
mkdir -p terraform/environments/staging
mkdir -p terraform/environments/prod
```

#### 步骤 2：创建可重用模块

该模块将包含所有环境共有的核心基础设施组件。让我们创建必要的文件：

##### 模块主文件（`terraform/modules/spring-boot-api/main.tf`​）

```terraform
resource "google_project_service" "run_api" {
  project            = var.project_id
  service            = "run.googleapis.com"
  disable_on_destroy = false
}

resource "google_project_service" "artifactregistry_api" {
  project            = var.project_id
  service            = "artifactregistry.googleapis.com"
  disable_on_destroy = false
}

# Create service account for Cloud Run
resource "google_service_account" "cloud_run_service_account" {
  project      = var.project_id
  account_id   = "${var.service_name}-sa"
  display_name = "Service Account for ${var.service_name}"
}

# Grant necessary permissions to service account
resource "google_project_iam_member" "cloud_run_invoker" {
  project = var.project_id
  role    = "roles/run.invoker"
  member  = "serviceAccount:${google_service_account.cloud_run_service_account.email}"
}

# Cloud Run service
resource "google_cloud_run_service" "spring_boot_api" {
  name     = var.service_name
  location = var.region
  project  = var.project_id

  template {
    spec {
      containers {
        image = "${var.region}-docker.pkg.dev/${var.project_id}/spring-boot-api/${var.service_name}:latest"
    
        resources {
          limits = {
            cpu    = var.cpu
            memory = var.memory
          }
        }
    
        env {
          name  = "SPRING_PROFILES_ACTIVE"
          value = var.environment
        }
    
        # Environment-specific env variables
        dynamic "env" {
          for_each = var.env_variables
          content {
            name  = env.key
            value = env.value
          }
        }
      }
      service_account_name = google_service_account.cloud_run_service_account.email
    }
  }

  traffic {
    percent         = 100
    latest_revision = true
  }

  depends_on = [
    google_project_service.run_api
  ]

  autogenerate_revision_name = true
}

# Make the Cloud Run service publicly accessible
resource "google_cloud_run_service_iam_member" "public_access" {
  project  = var.project_id
  service  = google_cloud_run_service.spring_boot_api.name
  location = google_cloud_run_service.spring_boot_api.location
  role     = "roles/run.invoker"
  member   = "allUsers"
}
```

此文件是一个可重用的 Terraform 模块，定义了将我们的 Spring Boot API 部署到 Google Cloud Run 的核心基础设施。它作为每个环境特定配置（开发/预生产/生产）引用的中央基础设施定义。以下是它所做的工作：

* 启用我们项目所需的 GCP API：Cloud Run API 和 Artifact Registry API。
* 创建服务账户基础设施。为 Cloud Run 创建一个专用服务账户，并授予其必要的“run.invoker”角色。
* 定义 Cloud Run 服务。使用环境特定参数（从环境模块传递）配置服务。设置容器规范，包括从 Artifact Registry 引用的镜像、可以按环境变化的资源限制（CPU 和内存）、环境变量（包括 Spring 配置文件和自定义变量）。将服务与创建的服务账户关联，并将流量路由到最新修订版。
* 通过将“run.invoker”角色授予“allUsers”来配置公共访问。

通过这样做，我们创建了可重用、参数化的模块，这些模块可以使用环境特定的值调用。例如，我们将在 `terraform/environments/dev/main.tf`​ 文件中调用此模块，传递开发环境特定的参数，允许我们在所有环境中保持一致的基础设施，同时根据需要进行适当的更改（开发/预生产/生产）。

##### 模块变量（`terraform/modules/spring-boot-api/variables.tf`​）

此文件定义了 Spring Boot API Terraform 模块接受的所有输入变量，允许每个环境（开发/预生产/生产）在调用模块时传递不同的值：

```terraform
variable "project_id" {
  description = "The ID of the GCP project"
  type        = string
}

variable "region" {
  description = "The region to deploy resources to"
  type        = string
}

variable "service_name" {
  description = "The name of the Cloud Run service"
  type        = string
}

variable "environment" {
  description = "The deployment environment (dev, staging, prod)"
  type        = string
}

variable "cpu" {
  description = "CPU allocation for Cloud Run service"
  type        = string
  default     = "1"
}

variable "memory" {
  description = "Memory allocation for Cloud Run service"
  type        = string
  default     = "512Mi"
}

variable "env_variables" {
  description = "Environment variables to pass to the container"
  type        = map(string)
  default     = {}
}
```

##### 模块输出（`terraform/modules/spring-boot-api/outputs.tf`​）

此文件定义了 Spring Boot API Terraform 模块在部署后公开的值，使关键信息可供调用环境模块（开发/预生产/生产）使用。它导出三个关键信息：部署服务的 URL（在环境特定的输出中引用）、服务名称和服务账户电子邮件。这些输出使我们基础设施的其他部分能够引用和与部署的 Cloud Run 服务交互，通过在所有环境中提供一致的部署结果访问，支持我们的多环境部署流水线：

```terraform
output "service_url" {
  value = google_cloud_run_service.spring_boot_api.status[0].url
}

output "service_name" {
  value = google_cloud_run_service.spring_boot_api.name
}

output "service_account_email" {
  value = google_service_account.cloud_run_service_account.email
}
```

#### 步骤 3：创建环境特定的配置

现在，创建每个环境的配置，从开发开始：

##### 开发环境（`terraform/environments/dev/main.tf`​）

```terraform
module "spring_boot_api" {
  source = "../../modules/spring-boot-api"

  project_id  = var.project_id
  region      = var.region
  service_name = var.service_name
  environment = var.environment
  
  # Dev-specific resource allocations
  cpu    = "1"
  memory = "512Mi"
  
  # Dev-specific environment variables
  env_variables = {
    LOG_LEVEL = "DEBUG"
    # Add other dev-specific variables here
  }
}

output "service_url" {
  value = module.spring_boot_api.service_url
}
```

此文件是第一个环境特定的配置文件，在这种情况下是我们的开发环境。它使用开发特定的参数实例化可重用的 Spring Boot API 模块，包括减少的资源分配（1 个 CPU，512Mi 内存）和开发特定的环境变量，如 DEBUG 日志记录。

##### 开发变量（`terraform/environments/dev/variables.tf`​）

```terraform
variable "project_id" {
  description = "The ID of the GCP project"
  type        = string
}

variable "region" {
  description = "The region to deploy resources to"
  type        = string
  default     = "us-central1"
}

variable "service_name" {
  description = "The name of the Cloud Run service"
  type        = string
}

variable "environment" {
  description = "The deployment environment (dev, staging, prod)"
  type        = string
  default     = "dev"
}
```

此变量文件声明了开发环境部署所需的强制变量，包括 GCP 项目 ID、区域（默认为 us-central1）、服务名称和环境类型（默认值为“dev”）。这些变量在从开发环境的 main.tf 文件调用 Spring Boot API 模块时传递，允许在保持跨三个环境部署流水线的一致基础设施模式的同时进行环境特定的自定义。

##### 开发后端配置（`terraform/environments/dev/backend.tf`​）

```terraform
terraform {
  backend "gcs" {
    bucket = "tf-automation-demo-state-spring-boot-api"
    prefix = "terraform/state/dev"
  }
}
```

此文件定义了 Terraform 为开发环境存储其状态的方式，具体来说是使用名为 tf-automation-demo-state-spring-boot-api 的 Google Cloud Storage (GCS) 桶，带有环境特定的前缀“terraform/state/dev”。

（我们将在本文后面的“为 Terraform 状态创建 Cloud Storage 桶”部分介绍此桶的创建）

#### 步骤 4：为预生产和生产重复此过程

我们为预生产和生产环境创建类似的文件，根据需要调整值。例如，生产环境可能有更高的资源分配或额外的安全措施。

##### 预生产环境（`terraform/environments/staging/main.tf`​）

```terraform
module "spring_boot_api" {
  source = "../../modules/spring-boot-api"

  project_id  = var.project_id
  region      = var.region
  service_name = var.service_name
  environment = var.environment
  
  # Staging-specific resource allocations
  cpu    = "1"
  memory = "768Mi"
  
  # Staging-specific environment variables
  env_variables = {
    LOG_LEVEL = "INFO"
    # Add other staging-specific variables here
  }
}

output "service_url" {
  value = module.spring_boot_api.service_url
}
```

##### （`terraform/environments/staging/variables.tf`​）

```terraform
variable "project_id" {
  description = "The ID of the GCP project"
  type        = string
}

variable "region" {
  description = "The region to deploy resources to"
  type        = string
  default     = "us-central1"
}

variable "service_name" {
  description = "The name of the Cloud Run service"
  type        = string
}

variable "environment" {
  description = "The deployment environment (dev, staging, prod)"
  type        = string
  default     = "staging"
}
```

##### （`terraform/environments/staging/backend.tf`​）

```terraform
terraform {
  backend "gcs" {
    bucket = "tf-automation-demo-state-spring-boot-api"
    prefix = "terraform/state/staging"
  }
}
```

##### 生产环境（`terraform/environments/prod/main.tf`​）

```terraform
module "spring_boot_api" {
  source = "../../modules/spring-boot-api"

  project_id  = var.project_id
  region      = var.region
  service_name = var.service_name
  environment = var.environment
  
  # Production-specific resource allocations
  cpu    = "2"
  memory = "1Gi"
  
  # Production-specific environment variables
  env_variables = {
    LOG_LEVEL = "WARN"
    # Add other production-specific variables here
  }
}

# You might add additional production-specific resources here
# For example, monitoring, alerts, or additional security measures

output "service_url" {
  value = module.spring_boot_api.service_url
}
```

##### （`terraform/environments/prod/variables.tf`​）

```terraform
variable "project_id" {
  description = "The ID of the GCP project"
  type        = string
}

variable "region" {
  description = "The region to deploy resources to"
  type        = string
  default     = "us-central1"
}

variable "service_name" {
  description = "The name of the Cloud Run service"
  type        = string
}

variable "environment" {
  description = "The deployment environment (dev, staging, prod)"
  type        = string
  default     = "prod"
}
```

##### （`terraform/environments/prod/backend.tf`​）

```terraform
terraform {
  backend "gcs" {
    bucket = "tf-automation-demo-state-spring-boot-api"
    prefix = "terraform/state/prod"
  }
}
```

### 设置 GCP 项目以进行部署

在我们能够使用我们的 CI/CD 流水线之前，我们需要在 GCP 中设置一些资源：

#### 1. 创建 3 个 GCP 项目（每个环境一个）

这种方法将为我们提供环境之间的适当隔离，并遵循云最佳实践。

首先，让我们创建我们的三个项目。你可以通过 Google Cloud Console 进行此操作：

1. 进入 Google Cloud Console
2. 点击页面顶部的项目下拉菜单
3. 点击“新建项目”
4. 创建三个具有描述性名称的单独项目：

    * automation-demo-dev（用于开发）
    * automation-demo-staging（用于预生产）
    * automation-demo-prod（用于生产）
5. 创建这些项目时，请注意项目 ID，因为它们可能与你选择的名称略有不同

这为我们提供了我们应用的隔离环境，具有自己的计费、资源和权限。

#### 为 GitHub Actions 创建服务账户

接下来，我们将创建 GitHub Actions 将用于与 GCP 交互的服务账户。我们为每个环境（开发、预生产和生产）创建单独的服务账户。

在多环境设置中，环境之间的隔离至关重要。如果你在一个环境中使用一个具有访问所有环境权限的服务账户，那么在开发部署期间你的流水线中的一个配置错误可能会潜在地影响你的生产环境。

通过为每个环境创建环境特定的服务账户，你建立了强大的安全边界。开发流水线只能影响开发资源，预生产流水线只能影响预生产资源，依此类推。

我们可以通过 Google Cloud Web Console 手动创建这些服务账户，但我们力求将所有配置和基础设施捕获在代码中，并安全地存储在源代码控制中。这将有助于防止在 Web Console 中意外删除/修改我们的服务账户，并使我们能够通过代码完全重新配置整个环境。

让我们首先在 Terraform 中创建一个可重用的模块，然后展示如何将其应用于每个环境。

#### 创建服务账户模块

在我们的 Terraform 模块目录中创建一个新的 main.tf 文件 `terraform/modules/github-actions-sa/`​。这是一个可重用的 Terraform 模块，定义了一个 GCP 项目中的单个 GitHub Actions 服务账户的基础设施。我们将通过一个服务账户根 Terraform 模块调用此模块三次，该模块协调我们每个环境（开发、预生产和生产）的服务账户创建。

```terraform
# Enable required APIs
resource "google_service_account" "github_actions" {
  project      = var.project_id
  account_id   = var.service_account_id
  display_name = "GitHub Actions Service Account for ${var.environment}"
  description  = "Service account used by GitHub Actions to deploy to ${var.environment}"
}

# Enable Service Usage API first using terraform-provider-google-beta
# This is required before we can enable other APIs
resource "google_project_service" "enable_service_usage" {
  project = var.project_id
  service = "serviceusage.googleapis.com"
  
  # Do not disable the API on destroy
  disable_on_destroy = false
  
  # Skip the initial check for API enablement
  # This is necessary since we're enabling the API that would perform the check
  provisioner "local-exec" {
    command = "sleep 60"
  }
}

# First, assign the Service Usage Admin role
resource "google_project_iam_member" "service_usage_admin" {
  depends_on = [google_project_service.enable_service_usage]
  
  project = var.project_id
  role    = "roles/serviceusage.serviceUsageAdmin"
  member  = "serviceAccount:${google_service_account.github_actions.email}"
}

# Then enable required APIs
resource "google_project_service" "required_apis" {
  depends_on = [
    google_project_iam_member.service_usage_admin,
    google_project_service.enable_service_usage
  ]
  
  for_each = toset([
    "artifactregistry.googleapis.com",
    "run.googleapis.com",
    "iam.googleapis.com",
    "cloudresourcemanager.googleapis.com"
  ])
  
  project = var.project_id
  service = each.value
  
  disable_dependent_services = false
  disable_on_destroy        = false
}

# Create Artifact Registry Repository
resource "google_artifact_registry_repository" "app_repository" {
  depends_on = [google_project_service.required_apis]
  
  project       = var.project_id
  location      = var.region
  repository_id = "spring-boot-api"
  description   = "Docker repository for Spring Boot API"
  format        = "DOCKER"
}

# Assign remaining required roles
resource "google_project_iam_member" "service_account_roles" {
  depends_on = [google_project_service.required_apis]
  
  for_each = toset([
    "roles/run.admin",                # Manage Cloud Run services
    "roles/artifactregistry.writer",  # Push to Artifact Registry
    "roles/storage.admin",            # Full access to GCS (for Terraform state)
    "roles/iam.serviceAccountUser",   # Use service accounts
    "roles/iam.serviceAccountAdmin",  # Create and manage service accounts
    "roles/resourcemanager.projectIamAdmin"  # Manage project IAM bindings
  ])
  
  project = var.project_id
  role    = each.value
  member  = "serviceAccount:${google_service_account.github_actions.email}"
}

# Additional environment-specific roles
resource "google_project_iam_member" "environment_specific_roles" {
  depends_on = [google_project_service.required_apis]
  
  for_each = var.environment == "production" ? toset([
    "roles/monitoring.viewer",    # View monitoring in production
    "roles/logging.viewer"        # View logs in production
  ]) : toset([])
  
  project = var.project_id
  role    = each.value
  member  = "serviceAccount:${google_service_account.github_actions.email}"
}

# Create a service account key
resource "google_service_account_key" "github_actions_key" {
  service_account_id = google_service_account.github_actions.name
  public_key_type    = "TYPE_X509_PEM_FILE"
}
```

此 Terraform 模块为我们的三个环境部署流水线（开发/预生产/生产）创建了环境特定的 GitHub Actions 服务账户，并启用了所有必要的权限。它启用了所需的 Google Cloud API（包括 Service Usage、Artifact Registry、Cloud Run、IAM 和 Cloud Resource Manager），为我们的 Docker 镜像创建了 Artifact Registry 仓库，并分配了部署所需的精确 IAM 角色（run.admin、artifactregistry.writer、storage.admin、iam.serviceAccountUser 和 resourcemanager.projectIamAdmin）。

该模块还为 GitHub Secrets 生成了服务账户密钥，并为生产环境应用了额外的监控权限，遵循最小权限原则，同时确保 CI/CD 流水线具有部署 Spring Boot API 到所有环境的所有必要权限。

现在，让我们在新的文件 `terraform/modules/github-actions-sa/variables.tf`​ 中创建此模块的变量文件：

```terraform
variable "project_id" {
  description = "The GCP project ID where the service account will be created"
  type        = string
}

variable "environment" {
  description = "The environment (development, staging, production)"
  type        = string
  
  validation {
    condition     = contains(["development", "staging", "production"], var.environment)
    error_message = "Environment must be one of: development, staging, production"
  }
}

variable "service_account_id" {
  description = "The ID to use for the service account"
  type        = string
  default     = "github-actions-sa"
}

variable "region" {
  description = "The region where resources will be created"
  type        = string
  default     = "us-central1"
}
```

此文件声明了 GitHub Actions 服务账户模块所需的变量，建立了跨三个环境部署流水线的环境特定配置的接口。它定义了四个关键参数：GCP 项目 ID（将在其中创建资源）、部署环境（带有验证以确保它是“development”、“staging”或“production”之一）、服务账户 ID（默认为“github-actions-sa”）和资源创建的区域（默认为“us-central1”）。这些变量使模块能够在我们的开发/预生产/生产环境中重用，同时强制执行一致性和环境特定的自定义。

最后，在 `terraform/modules/github-actions-sa/outputs.tf`​ 中创建输出文件以暴露重要值：

```terraform
output "email" {
  description = "The email address of the service account"
  value       = google_service_account.github_actions.email
}

output "key" {
  description = "The base64 encoded service account key"
  value       = google_service_account_key.github_actions_key.private_key
  sensitive   = true
}

output "artifact_registry_repository" {
  description = "The Artifact Registry repository details"
  value = {
    name     = google_artifact_registry_repository.app_repository.name
    location = google_artifact_registry_repository.app_repository.location
  }
}
```

此文件指定了三个从 GitHub Actions 服务账户模块输出的关键值，这些值对我们的 CI/CD 流水线至关重要：服务账户电子邮件地址（用于 IAM 策略中的引用）、base64 编码的服务账户密钥（标记为敏感以防止在日志中意外暴露），这需要添加到我们的 GitHub 仓库密钥中，以及 Docker 镜像存储的 Artifact Registry 仓库详细信息。这些输出与我们项目中接下来要配置的 GitHub 密钥（GCP\_SA\_KEY\_DEV、GCP\_SA\_KEY\_STAGING、GCP\_SA\_KEY\_PROD）的要求对齐。

### 为 Terraform 状态创建 Cloud Storage 桨

尽管我们已经使用 Terraform 将基础设施定义为代码，但我们仍然需要一个持久的地方来存储 Terraform 的状态信息。这是许多 Terraform 新手容易忽略的一个关键组件。让我们在我们的开发项目中创建一个。

Terraform 状态是一个 JSON 文件，它将你在配置文件中定义的资源映射到你在云提供者中代表的现实世界资源。当你运行 Terraform 命令如 `terraform plan`​ 或 `terraform apply`​ 时，Terraform 会将期望状态（来自你的配置文件）与当前状态（存储在状态文件中）进行比较，以确定需要进行哪些更改。

* **资源跟踪**：它跟踪 Terraform 创建的所有资源，使其知道它负责管理哪些资源。
* **元数据存储**：它存储 API 不提供的资源元数据，例如资源依赖关系。
* **性能优化**：它缓存资源属性以提高大型基础设施的性能。
* **并发控制**：当使用像 Google Cloud Storage 这样的远程后端时，它启用了锁定功能，防止多个用户同时修改基础设施。

让我们使用 GCP 中的 Cloud Shell 创建此桶：

1. 进入 GCP 网页控制台并导航到你的生产项目
2. 点击页面右上角的 Cloud Shell 图标（\>\_），它会在屏幕底部打开一个终端
3. 等待 Cloud Shell 初始化（首次使用可能需要一些时间）
4. 初始化后，运行以下命令创建桶（注意桶必须有一个全局唯一的名称，因此你需要亲自命名一个）：

    ```bash
    gsutil mb -l us-central1 gs://tf-automation-demo-state-spring-boot-api
    ```

Cloud Shell 预安装了所有你需要的 Google Cloud 工具，包括用于与 Cloud Storage 交互的 gsutil。此命令在 us-central1 区域创建一个名为 tf-automation-demo-state-spring-boot-api 的新桶。

如果你需要为你的桶启用版本控制，以防止状态损坏或意外删除，你可以运行：

```bash
gsutil versioning set on gs://tf-automation-demo-state-spring-boot-api
```

这确保了即使状态文件被意外修改或删除，你也可以恢复到之前的版本。

使用 Cloud Shell 很方便，因为：

* 你不需要在本地机器上安装任何软件
* 认证是自动处理的
* 它提供了一个与你的本地设置无关的一致环境
* 它与 GCP 服务有快速的网络连接

现在我们的状态桶已经创建，Terraform 现在可以安全地远程存储和管理其状态，从而实现协作并为我们的基础设施配置提供备份。

我们将在本文后面设置我们环境中服务账户的 Terraform 模块时引用此桶。

### 创建一个单独的 Terraform 配置用于服务账户

为了将服务账户管理与我们的应用基础设施分开，让我们创建一个专用的 Terraform 配置。创建新的目录和文件 `terraform/service-accounts/main.tf`​：

```terraform
provider "google" {
  # No default project specified here, as we'll define different projects for each environment
}

# Service account for development environment
module "github_actions_dev" {
  source = "../modules/github-actions-sa"
  
  project_id        = var.dev_project_id
  environment       = "development"
  service_account_id = "github-actions-sa"
}

# Service account for staging environment
module "github_actions_staging" {
  source = "../modules/github-actions-sa"
  
  project_id        = var.staging_project_id
  environment       = "staging"
  service_account_id = "github-actions-sa"
}

# Service account for production environment
module "github_actions_prod" {
  source = "../modules/github-actions-sa"
  
  project_id        = var.prod_project_id
  environment       = "production"
  service_account_id = "github-actions-sa"
}

# Output the keys (these will be sensitive)
output "dev_service_account_key" {
  value     = module.github_actions_dev.key
  sensitive = true
}

output "staging_service_account_key" {
  value     = module.github_actions_staging.key
  sensitive = true
}

output "prod_service_account_key" {
  value     = module.github_actions_prod.key
  sensitive = true
}
```

​`terraform/service-accounts/main.tf`​ 文件协调在所有三个环境（开发、预生产和生产）中创建 GitHub Actions 服务账户，通过实例化可重用的 github-actions-sa 模块，并将结果服务账户密钥作为敏感输出暴露，这些密钥将用作我们 CI/CD 流水线的 GitHub 密钥。

让我们还创建变量文件 `terraform/service-accounts/variables.tf`​：

```terraform
variable "dev_project_id" {
  description = "The GCP project ID for development environment"
  type        = string
}

variable "staging_project_id" {
  description = "The GCP project ID for staging environment"
  type        = string
}

variable "prod_project_id" {
  description = "The GCP project ID for production environment"
  type        = string
}
```

​`terraform/service-accounts/variables.tf`​ 文件定义了服务账户根模块所需的三个项目 ID 变量（dev\_project\_id、staging\_project\_id 和 prod\_project\_id），以在我们的三个环境部署流水线中创建环境特定的服务账户，作为集中化服务账户管理配置的输入接口。

我们还创建一个 `terraform/service-account/backend.tf`​ 文件：

```terraform
terraform {
  backend "gcs" {
    bucket = "tf-automation-demo-state-spring-boot-api"
    prefix = "terraform/service-accounts"
  }
}
```

​`terraform/service-accounts/backend.tf`​ 文件配置了服务账户根模块的远程状态存储，指定 Terraform 应将其状态存储在我们之前定义的 Google Cloud Storage 桶中，tf-automation-demo-state-spring-boot-api 在前缀 "terraform/service-accounts" 下，确保服务账户基础设施状态与环境特定的状态分开存储，同时在我们的多环境部署流水线中使用相同的集中化桶进行状态管理。

现在，我们在项目中创建一个示例，以提高可读性 `terraform/service-accounts/terraform.tfvars`​ 文件：

```
dev_project_id  = "automation-demo-dev"
staging_project_id  = "automation-demo-staging"
prod_project_id  = "automation-demo-prod"
region            = "us-central1"   
```

在文件的每个实例中，用实际的 GCP 项目 ID 替换值，你可以在 GCP 网页控制台中找到这些 ID。

注意：在本文的前面，我们创建了一个 .gitignore 文件，其中包含了 Terraform 从我们的版本控制中排除的文件，此 \*.tfvars 文件就是这样一个文件。

### 应用 Terraform 配置

现在是时候应用我们的配置，让 Terraform 为每个环境一致地创建我们的服务账户。这是我们为项目只做一次的事情，不会在每次 CI/CD 流水线执行时应用。然而，这意味着如果 GCP 中的服务账户被意外删除或修改，我们可以一致且轻松地重新创建它们：

```bash
cd terraform/service-accounts
terraform init
terraform plan
terraform apply
```

### 添加 GitHub 密钥

现在，我们需要将必要的密钥添加到你的 GitHub 仓库：

1. 进入你的 GitHub 仓库
2. 点击“设置” \> “密钥和变量” \> “操作”
3. 点击“新建仓库密钥”
4. 添加以下密钥：

    * GCP\_PROJECT\_DEV：开发环境的 GCP 项目 ID
    * GCP\_PROJECT\_STAGING：预生产环境的 GCP 项目 ID
    * GCP\_PROJECT\_PROD：生产环境的 GCP 项目 ID
    * GCP\_SA\_KEY\_DEV：开发项目的服务账户密钥。你可以通过在终端中运行以下命令来获取它。它将在你的 terraform/service-accounts/ 目录中生成一个 dev\_key.json 文件

      ```bash
      terraform output -raw dev_service_account_key > dev_key.json
      ```
    * GCP\_SA\_KEY\_STAGING：预生产项目的服务账户密钥。预生产项目的服务账户密钥。你可以通过在终端中运行以下命令来获取它。它将在你的 terraform/service-accounts/ 目录中生成一个 staging\_key.json 文件

      ```bash
      terraform output -raw staging_service_account_key > staging_key.json
      ```
    * GCP\_SA\_KEY\_PROD：生产项目的服务账户密钥。生产项目的服务账户密钥。你可以通过在终端中运行以下命令来获取它。它将在你的 terraform/service-accounts/ 目录中生成一个 prod\_key.json 文件

      ```bash
      terraform output -raw prod_service_account_key > prod_key.json
      ```

GitHub 密钥提供了一种安全的方式来存储敏感信息，如 API 密钥和凭据，使它们在我们的工作流中可用，而不会在我们的代码中暴露。这是一个关键的安全实践，因为它确保敏感凭据不会出现在我们的仓库历史记录或日志中。

添加这些密钥后，确保删除本地密钥文件以保持安全性：

```bash
rm dev_key.json staging_key.json prod_key.json
```

### 使包含 Terraform 状态的桶在项目间可访问

为了遵循云最佳实践，我们创建了三个独立的 GCP 项目，一个用于开发，一个用于预生产，最后一个是用于生产。

当我们的 CI/CD 流水线运行时，我们故意希望在所有环境中重复使用相同的资产，以保持一致性和效率。

这给我们带来了一个需要解决的小问题，因为默认情况下，我们的预生产和生产服务账户无法访问我们在 Dev GCP 项目中存储 Terraform 状态的 Cloud Storage 桶 tf-automation-demo-state-spring-boot-api。

我们可以通过在我们的 terraform/service-accounts 目录中添加以下 cross-project-permissions.tf 文件来解决这个问题：

```terraform
# Grant cross-project permissions to the Terraform state bucket

# Reference to the bucket in the dev project
data "google_storage_bucket" "terraform_state" {
  name    = "tf-automation-demo-state-spring-boot-api"
  project = var.dev_project_id
}

# Grant the staging service account access to the bucket
resource "google_storage_bucket_iam_member" "staging_bucket_access" {
  bucket = data.google_storage_bucket.terraform_state.name
  role   = "roles/storage.admin"
  member = "serviceAccount:github-actions-sa@${var.staging_project_id}.iam.gserviceaccount.com"
}

# Grant the production service account access to the bucket
resource "google_storage_bucket_iam_member" "prod_bucket_access" {
  bucket = data.google_storage_bucket.terraform_state.name
  role   = "roles/storage.admin"
  member = "serviceAccount:github-actions-sa@${var.prod_project_id}.iam.gserviceaccount.com"
}
```

创建此文件后，你需要导航到 terraform/service-accounts 目录并应用 Terraform 中的更改：

```bash
cd terraform/service-accounts
terraform init
terraform apply
```

### 初始环境设置

你需要手动为每个环境运行一次 Terraform 以设置初始基础设施：

```bash
# 初始化并应用开发环境
cd terraform/environments/dev
terraform init
terraform apply

# 重复预生产和生产环境
cd ../staging
terraform init
terraform apply

cd ../prod
terraform init
terraform apply
```

### 更新我们的 Spring Boot 应用以用于生产

让我们进行一些调整，以确保我们的 Spring Boot 应用已准备好在 GCP Cloud Run 中进行生产部署。

首先，让我们更新我们的 application.properties 以支持不同的环境。在你的 Spring 项目中的 src/main/resources/ 目录下创建一个名为 application-prod.properties 的新文件：

```
# 生产环境设置
server.port=${PORT:8080}

# Actuator 端点
management.endpoints.web.exposure.include=health,info
management.endpoint.health.show-details=always

# 应用信息
info.app.name=Spring Boot Test Automation API
info.app.description=Spring Boot REST API for test automation demo
info.app.version=1.0.0
info.app.environment=production

# 日志记录
logging.level.root=INFO
logging.level.org.springframework=INFO
logging.level.com.example=INFO

# 这对于 Cloud Run 来说很重要，以知道你的应用何时准备好
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=20s
```

这个特定于生产的配置文件包含针对我们的 Cloud Run 环境优化的设置。最重要的设置是 server.port\=\${PORT:8080}，它配置我们的应用监听 Cloud Run 通过 PORT 环境变量提供的端口（如果未指定，则默认为 8080）。

我们还配置了 Actuator 仅暴露 health 和 info 端点，这是生产环境的安全最佳实践。详细的健康信息将帮助我们诊断问题，而应用信息提供了关于我们部署应用的有用元数据。

最后，我们设置了适当日志记录级别，以确保我们有足够的信息进行监控，而不会因调试信息而淹没我们的日志。

### 测试 CI/CD 流水线

现在我们已经设置了所有必要的组件，让我们提交更改并推送到 GitHub 以触发我们的 CI/CD 流水线：

```bash
git add .
git commit -m "Add CI/CD pipeline with GitHub Actions and Terraform"
git push origin main
```

推送后，进入你的 GitHub 仓库的“操作”标签以监控你的工作流进度。如果一切设置正确，你应该会看到你的工作流经过各个阶段：

* 构建和测试应用
* 计划 Terraform 基础设施
* 部署到 Google Cloud Run

一旦工作流成功完成，你可以在 Terraform 输出中提供的 URL 访问我们部署的应用。此 URL 将显示在你的 GitHub Actions 工作流日志中，或者你可以在 GCP 控制台的 Cloud Run 中找到它。

一旦我们的流水线成功部署到我们的开发环境，我们到达了第一个门。在 GitHub Actions 中，我们将被提示手动“审查部署”到预生产环境，然后流水线才会继续将我们的 Spring Boot 服务部署到我们的预生产环境。

### 验证我们的部署

让我们验证我们的应用在 Cloud Run 环境中是否正常工作：

1. 进入 Google Cloud 控制台并导航到 Cloud Run
2. 点击你的服务（spring-boot-api）
3. 点击页面顶部显示的 URL
4. 在 URL 后面加上“/hello”以测试我们的 API 端点
5. 在 URL 后面加上“/actuator/health”以检查健康端点

你应该看到两个端点的适当响应，确认我们的应用已部署并正常工作。/hello 端点应返回我们的问候消息，而 /actuator/health 端点应显示我们的应用“UP”并正常运行。

这个验证步骤至关重要 —— 它确认我们的整个流水线如预期般工作，从代码提交到云部署。同样值得注意的是，我们的 CI/CD 流水线包括自动化冒烟测试，这些测试执行类似的检查，即使没有手动验证，也能让我们对部署的成功充满信心。

### 本流水线中使用最佳实践

让我们总结我们在 CI/CD 流水线中实现的最佳实践：

* **基础设施即代码**：我们所有的基础设施都使用 Terraform 定义，使其版本化、可重复和自动化。这确保了环境之间的一致性，并允许我们随时间跟踪基础设施的更改。
* **多阶段流水线**：我们的流水线有构建、测试和部署的不同阶段，确保如果当前阶段失败，则不会进入下一阶段。这在早期捕获问题，节省时间和资源。
* **静态代码分析**：我们使用 Checkstyle 运行静态代码分析以维护代码质量。这帮助我们在问题成为麻烦之前捕获潜在问题，并确保我们团队的代码风格一致。
* **全面测试**：我们在部署前运行单元测试和 API 测试，以早期捕获问题。通过在多个级别进行测试，我们对代码质量的信心增加。
* **冒烟测试**：部署后，我们运行冒烟测试以验证应用在生产环境中正常工作。这为我们提供了部署成功的即时反馈。
* **环境分离**：我们使用 Spring 配置文件为不同环境配置应用。这允许我们针对每个环境优化应用，无需更改代码。
* **安全的密钥管理**：我们将敏感信息（如服务账户密钥）作为 GitHub 密钥存储。这确保我们的凭据永远不会暴露在我们的代码或日志中。
* **有限的权限**：我们的服务账户只有执行其任务所需的权限。这遵循了最小权限原则，减少了安全漏洞的潜在影响。
* **不可变部署**：我们用 Git 提交 SHA 标记 Docker 镜像，启用简单的回滚并确保可重复性。如果我们需要回滚到之前的版本，我们只需重新部署之前的镜像。
* **自动化发布**：Cloud Run 自动管理新版本的发布，无需停机。这为我们的用户提供了无缝的体验，即使在部署期间也是如此。
* **可扩展性**：Cloud Run 根据负载自动扩展我们的应用，从零到多个实例。这意味着我们可以处理流量高峰，而无需过度配置资源。
* **成本效率**：我们只为使用的内容付费，采用 Cloud Run 的无服务器模式。当我们的应用没有接收流量时，它会扩展到零，确保我们不会为闲置资源付费。
* **远程状态管理**：我们将 Terraform 状态存储在 Cloud Storage 桶中，实现协作并为我们的基础设施配置提供备份。这对于团队环境至关重要，确保每个人都在使用相同的最新状态信息。

通过实现这些最佳实践，我们创建了一个强大、安全、高效的 CI/CD 流水线，让我们对部署充满信心，并让我们专注于为用户交付价值。

### 结论

在本文中，我们使用 GitHub Actions、Terraform 和 Google Cloud Platform 为我们的 Spring Boot 应用构建了一个强大的 CI/CD 流水线。我们的流水线在我们向主分支推送更改时自动构建、测试和部署我们的应用，确保我们的代码始终处于可部署状态。

这种方法不仅节省了我们的时间并减少了人为错误的风险，还让我们对开发过程的每一步都充满信心。使用 Terraform 的基础设施即代码确保我们的基础设施是一致的、可版本化的和可重复的，而 Cloud Run 为我们的应用提供了一个可扩展且成本效益高的平台。

恭喜你完成了这篇文章，这是一篇长文，但为我们的自动化 CI/CD 流水线完成了大量工作。

在下一篇文章中，我们将开始构建我们的后端代码，并通过数据库引入持久性，同时确保我们在整个过程中继续引入和维护自动化测试。
