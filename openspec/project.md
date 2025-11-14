# Project Context

## Purpose
Datadog Android SDK - 为 Android 和 Android TV 应用提供客户端监控和可观测性功能的 SDK。主要支持：

- **Real User Monitoring (RUM)**: 用户行为追踪、页面性能监控、错误收集
- **日志收集**: 应用日志的结构化收集和上传
- **APM 追踪**: 应用性能监控，包括网络请求、数据库操作等
- **Session Replay**: 会话回放功能，记录用户交互
- **NDK 崩溃监控**: Native 代码崩溃检测

设计理念：在用户设备上保持**稳定**、**高效**和**透明**的运行。

## Tech Stack
- **主要语言**: Kotlin (优先)、Java (兼容)
- **构建系统**: Gradle (Kotlin DSL)、Android Gradle Plugin 8.9.1
- **最低支持**: Android API 21 (Lollipop, 2014)
- **目标平台**: Android、Android TV、Wear OS、Automotive OS
- **依赖管理**: Gradle Version Catalog (libs.versions.toml)
- **静态分析**: Detekt 1.23.0、KtLint 0.50.0、Android Lint
- **构建优化**: Kotlin 2.0.21、KSP (Kotlin Symbol Processing)

## Project Conventions

### Code Style
- **格式化**: KtLint 默认配置，自动格式化
- **代码结构**: 使用 folding regions 组织方法
  ```kotlin
  // region ClassName
  // region InterfaceName  
  // region Internal
  ```
- **命名约定**:
  - 类名: PascalCase
  - 方法/属性: camelCase
  - 常量: SCREAMING_SNAKE_CASE
  - 内部类: 前缀 `Internal`
- **可见性**: 优先使用最小可见性，避免不必要的 `public`
- **兼容性**: 确保 Kotlin 代码对 Java 调用者友好

### Architecture Patterns
- **模块化设计**: 核心库 + 功能模块 + 集成模块
- **作用域模式**: RumScope 层次结构 (Application -> Session -> View -> Action)
- **面向对象设计**: 遵循 SOLID 原则
- **事件驱动**: RumRawEvent 处理链
- **依赖注入**: 手动 DI，避免框架依赖
- **线程模型**: 
  - 公共 API 快速执行
  - 重操作委托给后台线程
  - `@WorkerThread` 注解标记

### Testing Strategy
- **测试框架**:
  - JUnit 5 Jupiter (主要)
  - Mockito + mockito-kotlin (Mock)
  - AssertJ (断言)
  - Elmyr (属性驱动测试和数据生成)
  - Espresso (UI 集成测试)
- **测试约定**:
  - 类名: `{ClassName}Test`
  - 测试方法: `M expected behavior W method() {context}`
  - Given-When-Then 结构
  - 参数命名: `tested*`, `stub*`, `mock*`, `fake*`
- **测试类型**:
  - 单元测试: 优先黑盒测试
  - 集成测试: reliability/ 模块
  - 性能测试: tools/benchmark
- **覆盖要求**: 所有非琐碎代码必须测试

### Git Workflow
- **分支策略**: develop 为主开发分支
- **PR 要求**:
  - 每个 PR 解决一个问题
  - 必须签名提交
  - Draft 状态直到准备 review
  - 通过 CI 管道和代码质量检查
- **提交约定**: 倾向使用 gitmoji，但非强制
- **代码审查**: 至少一个有 push 权限的成员审批

## Domain Context
- **监控概念**:
  - Session: 用户会话，最多 4 小时或 15 分钟无活动
  - View: 屏幕/页面（Activity/Fragment）
  - Action: 用户交互 (tap, scroll, swipe)
  - Resource: 网络请求、图片加载等
  - Error: 崩溃、异常、自定义错误
  - Long Task: 主线程阻塞 (>100ms)
  
- **数据收集**:
  - 批量上传，避免频繁网络请求
  - 低电量/设备睡眠时暂停上传
  - 本地存储和重试机制
  - 采样率控制

- **隐私和合规性**:
  - 用户数据匿名化
  - 支持数据删除请求
  - 遵循 GDPR/CCPA 要求

## Important Constraints
- **性能约束**:
  - 零崩溃容忍度（除非是开发者可识别错误）
  - 最小化主线程阻塞
  - 库大小优化，避免不必要依赖
  - 电池友好，低网络负载

- **兼容性约束**:
  - 支持 Android API 21+
  - Java/Kotlin 互操作性
  - 向后兼容性（minor 版本更新）
  - ProGuard/R8 兼容

- **安全约束**:
  - 敏感数据加密传输
  - API 密钥保护
  - 防止数据泄露

## External Dependencies
- **Datadog 后端**:
  - RUM 事件端点
  - 日志收集端点  
  - 追踪数据端点
  - Session Replay 端点

- **Android 系统服务**:
  - ActivityManager (应用状态)
  - ConnectivityManager (网络状态)
  - PowerManager (电源状态)
  - WindowManager (显示信息)

- **第三方集成**:
  - OkHttp (网络监控)
  - Apollo GraphQL (查询监控)
  - 图片加载库 (Glide, Coil, Fresco)
  - UI 框架 (Jetpack Compose, Material Design)
  - RxJava, Coroutines (异步操作监控)

- **构建和发布**:
  - Maven Central (发布)
  - Sonatype OSSRH (暂存)
  - GitHub (源码和 CI/CD)
