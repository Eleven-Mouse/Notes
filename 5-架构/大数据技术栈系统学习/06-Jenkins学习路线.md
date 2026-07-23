# Jenkins 学习路线：从小白到工程化思维

## 结论

Jenkins 是自动化服务器，核心价值是：**把拉代码、编译、测试、打包、部署这些重复操作变成稳定流水线。**

它不是业务框架，也不是大数据组件。它负责工程化自动化。

## 它是什么

Jenkins 可以理解为自动化任务平台：

```text
代码提交
  |
  v
触发 Jenkins
  |
  v
拉代码 -> 编译 -> 测试 -> 打包 -> 部署
```

核心概念：

- Job
- Pipeline
- Jenkinsfile
- Agent
- Workspace
- Plugin
- Credential
- Artifact

## 什么时候用

适合用 Jenkins：

- 项目需要自动构建。
- 项目需要自动测试。
- 项目需要自动部署。
- 数据任务需要定时触发。
- 多环境发布流程复杂。
- 团队需要统一发布入口。

不适合过早复杂化：

- 只有本地练习项目。
- 构建步骤非常简单。
- 没有测试、部署、环境隔离要求。

架构判断：

```text
Jenkins 解决“流程自动化”，不是解决“代码本身怎么写”。
```

## 能做什么

Jenkins 可以：

- 拉取 Git 代码。
- 编译 Java / Scala 项目。
- 执行单元测试。
- 打包 Jar。
- 构建 Docker 镜像。
- 推送镜像仓库。
- 部署到服务器或 K8s。
- 定时触发 Spark 任务。
- 记录构建日志。
- 管理不同环境发布。

典型链路：

```text
Git
  |
  v
Jenkins
  |
  |-- Maven 编译
  |-- 单元测试
  |-- 打包 Jar
  |-- 构建镜像
  |-- 部署 K8s
```

## 架构位置

Jenkins 位于 CI/CD 层：

```text
开发提交代码
  |
  v
CI/CD：Jenkins
  |
  v
运行环境：服务器 / K8s / Yarn
```

在大数据项目中：

```text
Spark 代码 -> Jenkins 构建 -> 上传 Jar -> 调度执行
```

## 入门阶段

目标：能创建任务，跑通构建。

重点：

- 安装 Jenkins
- 创建 Job
- 配置 Git 仓库
- 配置 JDK
- 配置 Maven
- 查看构建日志
- 手动触发构建

最小流程：

```text
拉代码 -> mvn test -> mvn package
```

必须理解：

```text
Jenkins 本质是在一台或多台机器上按流程执行命令。
```

## 进阶阶段

目标：能写 Jenkinsfile，把流程代码化。

重点：

- Declarative Pipeline
- Stage
- Step
- Environment
- Parameters
- Credentials
- Post actions
- Artifact archive
- 定时触发
- Webhook 触发

Jenkinsfile 示例结构：

```groovy
pipeline {
  agent any

  stages {
    stage('Build') {
      steps {
        sh 'mvn clean package'
      }
    }

    stage('Test') {
      steps {
        sh 'mvn test'
      }
    }
  }
}
```

重点：

```text
流水线要可重复、可追踪、失败可定位。
```

## 高级阶段

目标：能设计团队级 CI/CD 流程。

重点：

- 多分支流水线
- 共享库
- Agent 节点管理
- Docker Agent
- K8s Agent
- 制品管理
- 权限控制
- 凭据安全
- 灰度发布
- 回滚
- 环境隔离
- 流水线模板化

高级问题：

- 构建慢怎么优化？
- 凭据泄漏怎么避免？
- 多项目重复 Jenkinsfile 怎么治理？
- 测试失败时如何阻断发布？
- 怎么保证发布可回滚？

## 重点难点

### Pipeline

Pipeline 是 Jenkins 的核心。

好处：

- 构建流程写进代码。
- 版本可追踪。
- 团队可复用。
- 出问题能查历史。

架构判断：

```text
发布流程越复杂，越应该 Pipeline as Code。
```

### 凭据管理

不要把密码、Token、私钥写在 Jenkinsfile 或代码里。

应该使用：

- Jenkins Credentials
- 环境变量注入
- 最小权限账号

风险：

```text
CI/CD 系统一旦泄漏凭据，影响通常比单个业务服务更大。
```

### 构建环境一致性

常见问题：

- 本地能跑，Jenkins 不能跑。
- Jenkins 节点 JDK 版本不一致。
- Maven 配置不一致。
- 环境变量缺失。

解决思路：

- 固定 JDK 和 Maven 版本。
- 使用 Docker 构建环境。
- 把构建命令写进脚本。
- 减少手工配置。

## 实战项目

做一个 Spark 项目的 Jenkins 流水线：

```text
Git 提交
  |
  v
Jenkins 拉代码
  |
  v
编译 Scala / Java
  |
  v
运行测试
  |
  v
打包 Spark Jar
  |
  v
上传到服务器
  |
  v
触发 Spark Submit
```

验收标准：

- Jenkins 能自动拉代码。
- 构建失败时能看到日志。
- 测试失败会阻断后续步骤。
- 构建产物可下载。
- 能定时触发任务。
- 敏感信息没有写死在代码里。

## 学习路线

```text
Job
  -> Git 集成
  -> Maven 构建
  -> Jenkinsfile
  -> 参数化构建
  -> 凭据管理
  -> Docker 镜像构建
  -> K8s 部署
  -> 团队级流水线治理
```

## 面试和工作重点

必须能讲清楚：

- Jenkins 解决什么问题。
- Job 和 Pipeline 的区别。
- Jenkinsfile 的价值。
- 凭据怎么安全管理。
- CI 和 CD 的区别。
- 构建失败怎么排查。
- Jenkins 怎么和 Docker、K8s 配合。

## 一句话总结

Jenkins 的核心是：**把靠人手点按钮的构建发布流程，变成可重复、可追踪、可治理的自动化流水线。**

