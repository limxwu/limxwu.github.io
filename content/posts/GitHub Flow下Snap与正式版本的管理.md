---
Title: GitFlow下Snap与正式版本的管理
date: 2025-09-12 12:00:00
---

基于GitHub的GitFlow如何管理java的snapshot与正式版本，这是一个非常经典且重要的问题。将 GitHub Flow 的敏捷性与 Java 项目的 Snapshot/Release 版本管理相结合，可以构建一个高效且可靠的发布流程。

下面我将详细阐述基于 GitHub Flow 的最佳实践，包括分支策略、版本命名约定、自动化流程以及具体操作步骤。

### 核心原则

1.  **GitHub Flow 基础**：
    
    -   主分支（`main` 或 `master`）始终反映生产就绪状态。
    -   新功能在功能分支（`feature/*`）上开发，通过 Pull Request (PR) 合并回主分支。
    -   合并到主分支后应立即部署（或具备部署能力）。
2.  **Maven 版本约定**：
    
    -   **Snapshot 版本（快照版本）**：`-SNAPSHOT` 后缀（例如 `1.2.3-SNAPSHOT`）。用于开发中的、不稳定的版本。同一快照版本可以多次部署到仓库，Maven 会在构建时尝试获取最新的快照。
    -   **Release 版本（正式版本）**： 没有后缀的版本（例如 `1.2.3`）。用于稳定的、生产可用的版本。一旦发布，该版本的工件在仓库中是不可变的。

### 分支策略与版本管理方案

这个方案的核心是：**功能分支和主分支使用 Snapshot 版本，创建发布分支时移除 Snapshot 后缀以生成 Release 版本。**

#### 1\. 日常开发阶段 (Snapshot 流程)

-   **分支**： 所有开发都在功能分支（如 `feature/add-payment`）上进行。
-   **版本号**： 主分支（`main`）的 `pom.xml` 中的版本始终是 **下一个预期版本的 Snapshot**。
    -   例如，你刚刚发布了 `1.2.3`，那么主分支的版本应立即升级为 `1.3.0-SNAPSHOT`（遵循语义化版本控制）。
-   **流程**：
    1.  开发者从 `main` 分支切出功能分支。
    2.  在功能分支上开发、提交代码。**功能分支的版本号与 `main` 分支保持一致**（即 `1.3.0-SNAPSHOT`），无需修改。
    3.  完成功能后，创建 PR 请求合并到 `main`。
    4.  CI/CD 工具（如 GitHub Actions）会检测到 PR 或合并事件，自动构建并运行测试。
    5.  **合并到 `main` 后**，CI/CD 应自动执行 `mvn deploy`，将最新的 `1.3.0-SNAPSHOT` 工件发布到 Maven Snapshot 仓库（如 GitHub Packages 或 Nexus Snapshot Repo）。其他依赖此项目的 Snapshot 版本的项目可以获取到这次最新的构建。

#### 2\. 发布阶段 (Release 流程)

当 `main` 分支的功能积累到一定程度，准备发布一个正式版本时。

-   **分支**： 从 `main` 分支创建一个**发布分支**（`release/v1.3.0`）。
-   **版本号**： **在发布分支上移除 `-SNAPSHOT` 后缀**。
    -   `main` 分支： `1.3.0-SNAPSHOT` -> (保持不变)
    -   `release/v1.3.0` 分支： `1.3.0-SNAPSHOT` -> `1.3.0`
-   **流程**：
    1.  从 `main` 分支切出 `release/v1.3.0` 分支。
    2.  在 `release/v1.3.0` 分支上执行：`mvn versions:set -DnewVersion=1.3.0`。这会移除 `-SNAPSHOT` 后缀。
    3.  提交这次版本变更（例如提交信息为 `"Release version 1.3.0"`）。
    4.  **为这个提交打上 Git Tag**： `git tag -a v1.3.0 -m "Release version 1.3.0"`。Tag 名与版本号对应，方便追溯。
    5.  将发布分支和 Tag 推送到远程仓库：`git push origin release/v1.3.0 && git push origin v1.3.0`。
    6.  CI/CD 工具检测到 `release/**` 分支的推送或 Tag 的创建，自动触发发布流程：`mvn clean deploy`。这次部署会将 **正式版本 `1.3.0`** 的工件发布到 Maven Release 仓库。
    7.  （可选）在发布分支上进行最终的验收测试（UAT）。如果发现任何小问题，直接在 `release/v1.3.0` 分支上修复，并递增版本号（如 `1.3.1`），然后重新触发部署。

#### 3\. 发布后阶段

-   **合并回主分支**： 发布完成后，**必须将发布分支上的版本变更提交（以及任何修复提交）合并回 `main` 分支**。
    -   这通常通过一个 PR 来完成，将 `release/v1.3.0` 合并回 `main`。
    -   由于 `main` 分支还是 `1.3.0-SNAPSHOT`，而发布分支是 `1.3.0`，这会产生一个冲突。**解决冲突时，保留 `main` 分支的 `-SNAPSHOT` 后缀**。
-   **升级主分支版本**： 合并完成后，`main` 分支现在包含了所有发布的内容。此时，**立即将 `main` 分支的版本升级为下一个开发周期的 Snapshot 版本**。
    -   执行 `mvn versions:set -DnewVersion=1.4.0-SNAPSHOT`。
    -   提交并推送 `main` 分支。
-   **归档发布分支**： 发布分支可以保留，也可以在一段时间后删除，因为 Tag 已经标记了发布的确切时点。

### 自动化（CI/CD）的关键作用

整个流程高度依赖自动化，通常使用 **GitHub Actions** 来实现。

1.  **CI 流水线（针对 PR 和 `main` 分支）**：
    
    -   触发器：`push` 到 `main`, `push` 到 `feature/*`，或 `pull_request` 事件。
    -   任务：编译、运行单元测试、集成测试、代码质量检查。
    -   **仅在合并到 `main` 后**：执行 `mvn deploy` 到 Snapshot 仓库。
2.  **CD 流水线（针对发布分支和 Tag）**：
    
    -   触发器：`push` 到 `release/**` 分支，或 `tag` 创建（如 `v*`）。
    -   任务：执行完整的 CI 流程。
    -   执行 `mvn deploy` 到 **Release 仓库**。确保此流水线有访问 Release 仓库的权限。

### 总结与最佳实践

| **开发** | `feature/*` | `1.3.0-SNAPSHOT` | 频繁提交，创建 PR | 不部署 |

| **集成** | `main` | `1.3.0-SNAPSHOT` | 合并 PR，CI 运行测试 | **自动部署到 Snapshot 仓库** |

| **发布** | `release/v1.3.0` | `1.3.0` (去 Snapshot) | 创建分支，改版本，打 Tag `v1.3.0` | **自动部署到 Release 仓库** |

| **迭代** | `main` | `1.4.0-SNAPSHOT` (升级) | 合并发布分支的修改，并升级版本 | - |

**优点**：

-   **清晰**： `main` 分支永远是“下一个版本的 Snapshot”，状态明确。
-   **自动化**： 最大程度减少人工干预，避免错误。
-   **可追溯**： Git Tag 与发布的正式版本一一对应。
-   **符合惯例**： 完全遵循 Maven 和 GitHub Flow 的标准约定。

**工具推荐**：

-   **Maven Versions Plugin**： 用于在命令行中高效地修改 `pom.xml` 版本。
    
    ```bash
    mvn versions:set -DnewVersion=1.3.0-SNAPSHOT
    
    ```
    
-   **GitHub Actions**： 实现整个 CI/CD 流程自动化。
    
-   **Maven Release Plugin**： 虽然功能强大，但其交互式工作流可能与 GitHub Flow 的自动化理念不太契合。上述方案通常更灵活、更现代。
    

通过遵循这个方案，你的 Java 项目可以实现高效、安全且一致的 Snapshot 和 Release 版本管理。