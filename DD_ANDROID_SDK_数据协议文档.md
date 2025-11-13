# Datadog Android SDK 数据上报协议文档

本文档详细描述了 Datadog Android SDK 在所有场景下上报的数据协议结构，包括 Trace、Log、RUM 和 Session Replay 四种类型。

---

## 目录

1. [重要问题解答](#重要问题解答)
2. [Log（日志）](#1-log日志)
3. [Trace（追踪）](#2-trace追踪)
4. [RUM（Real User Monitoring）](#3-rumreal-user-monitoring)
   - [3.1 Common Schema（通用字段）](#31-common-schema通用字段)
   - [3.2 View Event（视图事件）](#32-view-event视图事件)
   - [3.3 Action Event（动作事件）](#33-action-event动作事件)
   - [3.4 Error Event（错误事件）](#34-error-event错误事件)
   - [3.5 Resource Event（资源事件）](#35-resource-event资源事件)
   - [3.6 Long Task Event（长任务事件）](#36-long-task-event长任务事件)
   - [3.7 Vital Event（性能指标事件）](#37-vital-event性能指标事件)
5. [Session Replay（会话回放）](#4-session-replay会话回放)
6. [Telemetry（遥测数据）](#5-telemetry遥测数据)
7. [与 Web SDK 对比](#6-与-web-sdk-对比)

---

## 重要问题解答

### 1. Log 与 Web Log 的关系及必要性

**Q: Android SDK 的 Log 和 Web SDK 的 Log 是一回事吗？**

A: 是的，基本是一回事。两者都遵循相同的数据协议和字段定义，主要区别在于：

- **数据来源**：Android SDK 的 Log 来自移动端，包含设备特有信息（CPU 架构、电池状态等）
- **网络信息**：Android 包含 SIM 卡、信号强度等移动网络特有字段
- **线程信息**：Android 包含 `thread_name` 字段，记录创建日志的线程

**Q: RUM 和 Log 是集成在一起的，还是分开的？**

A: 在 Android SDK 中，RUM 和 Log 是**分开的独立模块**：

```kotlin
// 需要分别启用不同的功能
Logs.enable(logsConfiguration, sdkCore)  // 启用日志功能
Rum.enable(rumConfiguration, sdkCore)    // 启用 RUM 功能
```

- Log 模块：`com.datadoghq:dd-sdk-android-logs:x.x.x`
- RUM 模块：`com.datadoghq:dd-sdk-android-rum:x.x.x`

**Q: 对于 Native SDK，Log 服务是否必须？**

A: **不是必须的**。Log 和 RUM 可以独立工作：

- 如果只需要用户行为监控，可以只启用 RUM
- 如果只需要日志收集，可以只启用 Log
- 大部分应用推荐同时启用，以获得完整的可观测性

### 2. Trace Schema 与网络请求的关系

**Q: Trace 为什么会有 Schema？与 Web 应用的异同？**

A: Trace Schema 定义了 Span 事件的数据结构。与 Web 应用的主要异同：

**相同点：**

- 都遵循 OpenTelemetry 协议
- 都会在 HTTP 请求中添加 Tracing Headers
- Span 数据结构基本一致（trace_id、span_id、parent_id 等）

**不同点：**

- **Android SDK**：通过拦截器（如 OkHttp TracingInterceptor）自动添加 headers
- **Web SDK**：直接在 fetch/xhr 请求中注入 headers

**Android 网络请求中的 Tracing Headers：**

```kotlin
// 自动添加的 Headers：
DD-API-KEY: <client-token>
DD-EVP-ORIGIN: android
DD-EVP-ORIGIN-VERSION: <sdk-version>
DD-REQUEST-ID: <request-uuid>

// OpenTelemetry 标准 Headers（如果启用 Tracing）：
traceparent: 00-<trace-id>-<span-id>-01
tracestate: dd=s:1;o:rum
```

### 3. RUM 事件类型

**Q: RUM 事件上报中，主要的事件类型有哪些？**

A: RUM 有 **7 种主要事件类型**：

| 事件类型      | 触发时机                 | 主要用途                             |
| ------------- | ------------------------ | ------------------------------------ |
| **View**      | 页面/屏幕显示            | 跟踪用户访问的页面，性能指标         |
| **Action**    | 用户交互（点击、滑动等） | 用户行为分析                         |
| **Error**     | 发生错误/崩溃            | 错误监控和崩溃分析                   |
| **Resource**  | 网络请求                 | 网络性能监控                         |
| **Long Task** | 长时间任务（>50ms）      | 性能瓶颈识别                         |
| **Vital**     | 性能指标                 | 关键性能指标（应用启动、自定义指标） |
| **Telemetry** | SDK 内部监控             | SDK 运行状态监控                     |

### 4. 数据采集的可靠性

**Q: 操作系统、网络、账号和设备这些数据，一定能确保采集到吗？**

A: **不是所有数据都能 100% 采集到**，具体情况如下：

| 数据类型     | 可靠性  | 说明                                        |
| ------------ | ------- | ------------------------------------------- |
| **OS 信息**  | ✅ 高   | `os.name`、`os.version` 等基本都能获取到    |
| **设备信息** | ✅ 高   | `device.brand`、`device.model` 等通常可获取 |
| **网络信息** | ⚠️ 中   | 需要网络权限，某些信息可能受限              |
| **位置信息** | ❌ 低   | 需要位置权限，用户可能拒绝                  |
| **用户信息** | ❌ 可选 | 完全由应用主动设置                          |
| **账户信息** | ❌ 可选 | 完全由应用主动设置                          |

**采集限制因素：**

- Android 权限系统
- 用户隐私设置
- 设备兼容性
- 系统版本差异

### 5. 通用字段说明

**Q: 通用字段是在所有 RUM 事件中都一定会有的公共属性吗？**

A: **是的**，所有 RUM 事件都继承自 `_common-schema.json`，包含以下**必填**通用字段：

```json
{
  "date": 1234567890, // ✅ 必填：事件时间戳
  "application": { "id": "..." }, // ✅ 必填：应用ID
  "session": {
    // ✅ 必填：Session信息
    "id": "...",
    "type": "user"
  },
  "view": {
    // ✅ 必填：View信息
    "id": "...",
    "url": "..."
  },
  "_dd": {
    // ✅ 必填：内部字段
    "format_version": 2
  }
}
```

**可选通用字段：**

- `usr`（用户信息）
- `device`、`os`（设备和系统信息）
- `connectivity`（网络连接信息）
- `context`（自定义上下文）

### 6. Flutter 支持说明

**Q: 为什么会有 Flutter 事件？Flutter 和 Android 的关系是什么？**

A: Flutter 是跨平台框架，与 Android SDK 的关系：

**Flutter 支持方式：**

1. **Flutter 项目可以直接接入 Android SDK**：通过 Platform Channel 调用原生 Android SDK
2. **专门的 Flutter 插件**：`datadog_flutter_plugin` 封装了 Android/iOS SDK
3. **数据上报**：Flutter 事件最终还是通过 Android SDK 上报，只是增加了 `source: flutter` 标识

**Flutter 特有字段：**

- `view.flutter_build_time`：Flutter Widget 构建时间
- `view.flutter_raster_time`：Flutter 渲染光栅化时间
- `source: flutter`：标识数据来源

**跨平台支持列表：**

- `android`：原生 Android 应用
- `flutter`：Flutter 应用
- `react-native`：React Native 应用
- `kotlin-multiplatform`：Kotlin 多平台应用

### 7. WebView 事件处理

**Q: Android 应用中的 WebView 需要额外引入 Web SDK 吗？**

A: **不需要额外引入 Web SDK**！Android SDK 提供了 WebView 桥接功能：

**集成方式：**

```kotlin
// 1. 添加 WebView 模块
implementation("com.datadoghq:dd-sdk-android-webview:x.x.x")

// 2. 启用 WebView 跟踪
WebViewTracking.enable(
    webView = webView,
    allowedHosts = listOf("example.com", "example.net"),
    logsSampleRate = 100f
)
```

**工作原理：**

1. Android SDK 向 WebView 注入 JavaScript 桥接代码
2. WebView 中的 Web 事件通过 `DatadogEventBridge` 传递给 Android SDK
3. **所有事件归属到同一个移动 Session**，无需单独的 Web Session

**支持的事件类型：**

- RUM 事件（View、Action、Error、Resource）
- Log 事件
- Session Replay 记录

### 8. Vital Event 详解

**Q: Vital Event 具体包含哪些内容？**

A: Vital Event 用于上报**关键性能指标**，包含 3 种子类型：

#### 8.1 Duration Vital（持续时间指标）

```json
{
  "type": "vital",
  "vital": {
    "type": "duration",
    "id": "uuid",
    "name": "custom_metric_name",
    "description": "Custom performance metric",
    "duration": 1500000000 // 纳秒
  }
}
```

**用途：** 自定义性能指标（页面加载时间、API 响应时间等）

#### 8.2 App Launch Vital（应用启动指标）

```json
{
  "type": "vital",
  "vital": {
    "type": "app_launch",
    "app_launch_metric": "ttid", // ttid | ttfd
    "duration": 2000000000,
    "startup_type": "cold_start", // cold_start | warm_start
    "is_prewarmed": false,
    "has_saved_instance_state_bundle": false
  }
}
```

**关键指标：**

- `ttid`：Time To Initial Display（首次显示时间）
- `ttfd`：Time To Full Display（完全显示时间）
- `cold_start`：冷启动
- `warm_start`：温启动

#### 8.3 Feature Operation Vital（功能操作指标）

```json
{
  "type": "vital",
  "vital": {
    "type": "operation_step",
    "operation_key": "photo_upload",
    "step_type": "start", // start | update | retry | end
    "failure_reason": "error" // error | abandoned | other
  }
}
```

**用途：** 跟踪复杂业务操作的各个步骤（如文件上传、支付流程等）

### 9. 跨平台支持详解

**Q: 支持 Flutter，意思是 Flutter 项目可以直接接入 Android SDK 吗？**

A: **可以，但推荐使用专门的 Flutter 插件**：

**方式一：直接使用（复杂）**

```dart
// 通过 MethodChannel 调用原生 Android SDK
const platform = MethodChannel('datadog_sdk');
await platform.invokeMethod('initializeRum', params);
```

**方式二：使用 Flutter 插件（推荐）**

```dart
// 使用官方 Flutter 插件
dependencies:
  datadog_flutter_plugin: ^x.x.x

// 初始化
await DatadogSdk.initialize(
  configuration: DatadogConfiguration(
    clientToken: 'your-token',
    env: 'prod',
    applicationId: 'your-app-id',
  ),
);
```

**支持的跨平台框架：**

- ✅ **Flutter**：官方插件支持
- ✅ **React Native**：官方插件支持
- ✅ **Kotlin Multiplatform**：直接支持
- ✅ **Unity**：官方插件支持
- ⚠️ **Xamarin**：社区支持

---

## 1. Log（日志）

### Schema 文件

`dd-sdk-android-logs/src/main/json/log/log-schema.json`

### 数据结构

```json
{
  "type": "object",
  "title": "LogEvent",
  "required": [
    "message",
    "status",
    "date",
    "service",
    "logger",
    "_dd",
    "ddtags",
    "device",
    "os"
  ]
}
```

### 核心字段说明

| 字段                               | 类型   | 必填 | 说明                                                                         |
| ---------------------------------- | ------ | ---- | ---------------------------------------------------------------------------- |
| **message**                        | string | ✅   | 日志消息内容                                                                 |
| **status**                         | string | ✅   | 日志级别：`critical`, `error`, `warn`, `info`, `debug`, `trace`, `emergency` |
| **date**                           | string | ✅   | ISO-8601 格式的时间戳                                                        |
| **service**                        | string | ✅   | 服务名称                                                                     |
| **logger**                         | object | ✅   | Logger 信息                                                                  |
| **logger.name**                    | string | ✅   | Logger 名称                                                                  |
| **logger.thread_name**             | string | ✅   | 创建日志的线程名                                                             |
| **logger.version**                 | string | ✅   | SDK 版本                                                                     |
| **device**                         | object | ✅   | 设备信息（引用 RUM common schema）                                           |
| **os**                             | object | ✅   | 操作系统信息（引用 RUM common schema）                                       |
| **\_dd**                           | object | ✅   | Datadog 内部信息                                                             |
| **\_dd.device.architecture**       | string | ✅   | CPU 架构                                                                     |
| **usr**                            | object | ❌   | 用户属性                                                                     |
| **usr.anonymous_id**               | string | ❌   | 跨会话的匿名用户标识符                                                       |
| **usr.id**                         | string | ❌   | 用户 ID                                                                      |
| **usr.name**                       | string | ❌   | 用户名                                                                       |
| **usr.email**                      | string | ❌   | 用户邮箱                                                                     |
| **account**                        | object | ❌   | 账户属性                                                                     |
| **account.id**                     | string | ❌   | 账户 ID                                                                      |
| **account.name**                   | string | ❌   | 账户名称                                                                     |
| **network**                        | object | ❌   | 网络信息                                                                     |
| **network.client**                 | object | ✅   | 客户端网络信息                                                               |
| **network.client.connectivity**    | string | ✅   | 活动网络类型                                                                 |
| **network.client.sim_carrier**     | object | ❌   | SIM 卡运营商信息                                                             |
| **network.client.signal_strength** | string | ❌   | 信号强度                                                                     |
| **network.client.downlink_kbps**   | string | ❌   | 下行速度 (kbps)                                                              |
| **network.client.uplink_kbps**     | string | ❌   | 上行速度 (kbps)                                                              |
| **error**                          | object | ❌   | 错误信息（当日志级别为 error 时）                                            |
| **error.kind**                     | string | ❌   | 错误类型（从异常类名解析）                                                   |
| **error.message**                  | string | ❌   | 错误消息                                                                     |
| **error.stack**                    | string | ❌   | 错误堆栈                                                                     |
| **error.source_type**              | string | ❌   | 错误来源类型（如 `android`, `flutter`, `react-native`）                      |
| **error.fingerprint**              | string | ❌   | 自定义错误指纹                                                               |
| **error.threads**                  | array  | ❌   | 线程信息列表                                                                 |
| **build_id**                       | string | ❌   | 应用构建的唯一 ID                                                            |
| **ddtags**                         | string | ✅   | 标签列表（逗号分隔的键值对）                                                 |

### 附加属性

- 支持 `additionalProperties`，可添加自定义日志属性

---

## 2. Trace（追踪）

### Schema 文件

`dd-sdk-android-trace/src/main/json/trace/span-schema.json`

### 数据结构

```json
{
  "type": "object",
  "title": "SpanEvent",
  "required": [
    "trace_id",
    "span_id",
    "parent_id",
    "resource",
    "name",
    "service",
    "duration",
    "start",
    "error",
    "type",
    "meta",
    "metrics"
  ]
}
```

### 核心字段说明

| 字段                         | 类型    | 必填 | 说明                               |
| ---------------------------- | ------- | ---- | ---------------------------------- |
| **trace_id**                 | string  | ✅   | Trace ID                           |
| **span_id**                  | string  | ✅   | Span 的唯一 ID                     |
| **parent_id**                | string  | ✅   | 父 Span 的 ID（根 Span 为 0）      |
| **resource**                 | string  | ✅   | 资源名称                           |
| **name**                     | string  | ✅   | Span 名称                          |
| **service**                  | string  | ✅   | 服务名称                           |
| **duration**                 | integer | ✅   | Span 持续时间（纳秒）              |
| **start**                    | integer | ✅   | Span 开始时间（纳秒）              |
| **error**                    | integer | ✅   | 错误标志（1 表示有错误，默认 0）   |
| **type**                     | string  | ✅   | Span 类型（移动端固定为 `custom`） |
| **metrics**                  | object  | ✅   | 指标数据                           |
| **metrics.\_top_level**      | integer | ❌   | 顶层标志（1 表示根 Span）          |
| **meta**                     | object  | ✅   | 元数据                             |
| **meta.version**             | string  | ✅   | 应用版本                           |
| **meta.\_dd**                | object  | ✅   | Datadog 内部元数据                 |
| **meta.\_dd.source**         | string  | ✅   | 追踪源（默认 `android`）           |
| **meta.\_dd.application.id** | string  | ❌   | RUM Application ID                 |
| **meta.\_dd.session.id**     | string  | ❌   | RUM Session ID                     |
| **meta.\_dd.view.id**        | string  | ❌   | RUM View ID                        |
| **meta.span.kind**           | string  | ❌   | Span 类型（固定为 `client`）       |
| **meta.tracer.version**      | string  | ✅   | Tracer 版本                        |
| **meta.usr**                 | object  | ✅   | 用户属性                           |
| **meta.usr.id**              | string  | ❌   | 用户 ID                            |
| **meta.usr.name**            | string  | ❌   | 用户名                             |
| **meta.usr.email**           | string  | ❌   | 用户邮箱                           |
| **meta.account**             | object  | ❌   | 账户属性                           |
| **meta.network**             | object  | ❌   | 网络信息                           |
| **meta.device**              | object  | ✅   | 设备信息                           |
| **meta.os**                  | object  | ✅   | 操作系统信息                       |

### 附加属性

- `metrics` 支持额外的数值型指标
- `meta` 支持额外的字符串型元数据

---

## 3. RUM（Real User Monitoring）

### 3.1 Common Schema（通用字段）

所有 RUM 事件都继承自 `_common-schema.json`，包含以下通用字段：

#### 核心通用字段

| 字段                           | 类型    | 必填 | 说明                                                                                                      |
| ------------------------------ | ------- | ---- | --------------------------------------------------------------------------------------------------------- |
| **date**                       | integer | ✅   | 事件开始时间（自 epoch 起的毫秒数）                                                                       |
| **application.id**             | string  | ✅   | 应用 UUID                                                                                                 |
| **application.current_locale** | string  | ❌   | 当前语言环境（如 `es-FR`）                                                                                |
| **service**                    | string  | ❌   | 服务名称                                                                                                  |
| **version**                    | string  | ❌   | 应用版本                                                                                                  |
| **build_version**              | string  | ❌   | 构建版本                                                                                                  |
| **build_id**                   | string  | ❌   | 构建唯一 ID                                                                                               |
| **ddtags**                     | string  | ❌   | 标签（格式：`env:prod,version:1.2.3`）                                                                    |
| **session.id**                 | string  | ✅   | Session UUID                                                                                              |
| **session.type**               | string  | ✅   | Session 类型：`user`, `synthetics`, `ci_test`                                                             |
| **session.has_replay**         | boolean | ❌   | 是否有会话回放                                                                                            |
| **source**                     | string  | ❌   | 事件来源：`android`, `ios`, `browser`, `flutter`, `react-native`, `roku`, `unity`, `kotlin-multiplatform` |
| **view.id**                    | string  | ✅   | View UUID                                                                                                 |
| **view.url**                   | string  | ✅   | View URL                                                                                                  |
| **view.name**                  | string  | ❌   | View 自定义名称                                                                                           |
| **view.referrer**              | string  | ❌   | 引用 URL                                                                                                  |

#### 用户信息

| 字段                 | 类型   | 说明           |
| -------------------- | ------ | -------------- |
| **usr.id**           | string | 用户 ID        |
| **usr.name**         | string | 用户名         |
| **usr.email**        | string | 用户邮箱       |
| **usr.anonymous_id** | string | 匿名用户 ID    |
| **usr.\***           | any    | 自定义用户属性 |

#### 账户信息

| 字段             | 类型   | 说明            |
| ---------------- | ------ | --------------- |
| **account.id**   | string | 账户 ID（必填） |
| **account.name** | string | 账户名称        |
| **account.\***   | any    | 自定义账户属性  |

#### 设备信息

| 字段                         | 类型    | 说明                                                                            |
| ---------------------------- | ------- | ------------------------------------------------------------------------------- |
| **device.type**              | string  | 设备类型：`mobile`, `desktop`, `tablet`, `tv`, `gaming_console`, `bot`, `other` |
| **device.name**              | string  | 设备市场名称（如 `Xiaomi Redmi Note 8 Pro`）                                    |
| **device.model**             | string  | 设备型号（如 `Samsung SM-988GN`）                                               |
| **device.brand**             | string  | 设备品牌（如 `Apple`, `OPPO`, `Xiaomi`）                                        |
| **device.architecture**      | string  | CPU 架构                                                                        |
| **device.locale**            | string  | 设备语言环境（如 `en-US`）                                                      |
| **device.locales**           | array   | 用户偏好语言列表                                                                |
| **device.time_zone**         | string  | 时区标识符（如 `Europe/Berlin`）                                                |
| **device.battery_level**     | number  | 电池电量（0.0 - 1.0）                                                           |
| **device.power_saving_mode** | boolean | 是否启用省电模式                                                                |
| **device.brightness_level**  | number  | 屏幕亮度（0.0 - 1.0）                                                           |

#### 操作系统信息

| 字段                 | 类型   | 必填 | 说明                                |
| -------------------- | ------ | ---- | ----------------------------------- |
| **os.name**          | string | ✅   | 操作系统名称（如 `Android`, `iOS`） |
| **os.version**       | string | ✅   | 完整操作系统版本（如 `8.1.1`）      |
| **os.version_major** | string | ✅   | 主版本号（如 `8`）                  |
| **os.build**         | string | ❌   | 构建号（如 `15D21`）                |

#### 连接信息

| 字段                                   | 类型   | 说明                                                                                                    |
| -------------------------------------- | ------ | ------------------------------------------------------------------------------------------------------- |
| **connectivity.status**                | string | 连接状态：`connected`, `not_connected`, `maybe`                                                         |
| **connectivity.interfaces**            | array  | 可用网络接口：`bluetooth`, `cellular`, `ethernet`, `wifi`, `wimax`, `mixed`, `other`, `unknown`, `none` |
| **connectivity.effective_type**        | string | 有效连接类型：`slow-2g`, `2g`, `3g`, `4g`                                                               |
| **connectivity.cellular.technology**   | string | 蜂窝网络技术                                                                                            |
| **connectivity.cellular.carrier_name** | string | 运营商名称                                                                                              |

#### 显示信息

| 字段                        | 类型   | 说明             |
| --------------------------- | ------ | ---------------- |
| **display.viewport.width**  | number | 视口宽度（像素） |
| **display.viewport.height** | number | 视口高度（像素） |

#### Datadog 内部字段

| 字段                                              | 类型    | 必填 | 说明                                         |
| ------------------------------------------------- | ------- | ---- | -------------------------------------------- |
| **\_dd.format_version**                           | integer | ✅   | RUM 事件格式版本（固定为 2）                 |
| **\_dd.session.plan**                             | number  | ❌   | Session 计划：1（无 replay），2（有 replay） |
| **\_dd.session.session_precondition**             | string  | ❌   | Session 创建前提条件                         |
| **\_dd.configuration.session_sample_rate**        | number  | ✅   | Session 采样率（0-100）                      |
| **\_dd.configuration.session_replay_sample_rate** | number  | ❌   | Session Replay 采样率（0-100）               |
| **\_dd.configuration.profiling_sample_rate**      | number  | ❌   | Profiling 采样率（0-100）                    |
| **\_dd.browser_sdk_version**                      | string  | ❌   | Browser SDK 版本                             |
| **\_dd.sdk_name**                                 | string  | ❌   | SDK 名称                                     |

#### 上下文

| 字段        | 类型   | 说明                                   |
| ----------- | ------ | -------------------------------------- |
| **context** | object | 用户提供的自定义上下文（支持任意属性） |

---

### 3.2 View Event（视图事件）

#### Schema 文件

`dd-sdk-android-rum/src/main/json/rum/view-schema.json`

#### 数据结构

```json
{
  "type": "view",
  "allOf": ["_common-schema.json", "_view-container-schema.json"],
  "required": ["type", "view", "_dd"]
}
```

#### 核心字段说明

| 字段                                   | 类型    | 必填 | 说明                                                                                                                                                                                 |
| -------------------------------------- | ------- | ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **type**                               | string  | ✅   | 固定为 `view`                                                                                                                                                                        |
| **view.id**                            | string  | ✅   | View UUID                                                                                                                                                                            |
| **view.url**                           | string  | ✅   | View URL                                                                                                                                                                             |
| **view.time_spent**                    | integer | ✅   | View 停留时间（纳秒）                                                                                                                                                                |
| **view.loading_time**                  | integer | ❌   | View 加载时间（纳秒）                                                                                                                                                                |
| **view.network_settled_time**          | integer | ❌   | 网络请求完成时间（纳秒）                                                                                                                                                             |
| **view.interaction_to_next_view_time** | integer | ❌   | 上一次交互到当前视图显示的时间（纳秒）                                                                                                                                               |
| **view.loading_type**                  | string  | ❌   | 加载类型：`initial_load`, `route_change`, `activity_display`, `activity_redisplay`, `fragment_display`, `fragment_redisplay`, `view_controller_display`, `view_controller_redisplay` |
| **view.custom_timings**                | object  | ❌   | 自定义计时（键值对，值为纳秒）                                                                                                                                                       |
| **view.is_active**                     | boolean | ❌   | View 是否活跃                                                                                                                                                                        |
| **view.is_slow_rendered**              | boolean | ❌   | 是否渲染缓慢                                                                                                                                                                         |

#### 计数统计

| 字段                        | 类型    | 必填 | 说明                     |
| --------------------------- | ------- | ---- | ------------------------ |
| **view.action.count**       | integer | ✅   | View 上的 Action 数量    |
| **view.error.count**        | integer | ✅   | View 上的 Error 数量     |
| **view.crash.count**        | integer | ❌   | View 上的 Crash 数量     |
| **view.long_task.count**    | integer | ❌   | View 上的 Long Task 数量 |
| **view.frozen_frame.count** | integer | ❌   | View 上的冻结帧数量      |
| **view.resource.count**     | integer | ✅   | View 上的 Resource 数量  |
| **view.frustration.count**  | integer | ❌   | View 上的挫败感数量      |

#### 性能指标（移动端特有）

| 字段                          | 类型   | 说明                 |
| ----------------------------- | ------ | -------------------- |
| **view.memory_average**       | number | 平均内存使用（字节） |
| **view.memory_max**           | number | 最大内存使用（字节） |
| **view.cpu_ticks_count**      | number | CPU ticks 总数       |
| **view.cpu_ticks_per_second** | number | 每秒 CPU ticks 数    |
| **view.refresh_rate_average** | number | 平均刷新率（帧/秒）  |
| **view.refresh_rate_min**     | number | 最小刷新率（帧/秒）  |
| **view.slow_frames_rate**     | number | 慢帧率（毫秒/秒）    |
| **view.freeze_rate**          | number | 冻结率（秒/小时）    |

#### 慢帧列表

| 字段                            | 类型    | 说明                                 |
| ------------------------------- | ------- | ------------------------------------ |
| **view.slow_frames**            | array   | 慢帧列表                             |
| **view.slow_frames[].start**    | integer | 慢帧开始时间（相对 view 起始，纳秒） |
| **view.slow_frames[].duration** | integer | 慢帧持续时间（纳秒）                 |

#### 前台时段

| 字段                                      | 类型    | 说明                                 |
| ----------------------------------------- | ------- | ------------------------------------ |
| **view.in_foreground_periods**            | array   | 前台时段列表                         |
| **view.in_foreground_periods[].start**    | integer | 时段开始时间（相对 view 起始，纳秒） |
| **view.in_foreground_periods[].duration** | integer | 时段持续时间（纳秒）                 |

#### Flutter 性能指标

| 字段                         | 类型   | 说明               |
| ---------------------------- | ------ | ------------------ |
| **view.flutter_build_time**  | object | Flutter 构建时间   |
| **view.flutter_raster_time** | object | Flutter 光栅化时间 |

#### Web 性能指标（已弃用）

| 字段                               | 类型    | 说明                         |
| ---------------------------------- | ------- | ---------------------------- |
| **view.first_contentful_paint**    | integer | 首次内容绘制时间（已弃用）   |
| **view.largest_contentful_paint**  | integer | 最大内容绘制时间（已弃用）   |
| **view.first_input_delay**         | integer | 首次输入延迟（已弃用）       |
| **view.interaction_to_next_paint** | integer | 交互到下次绘制时间（已弃用） |
| **view.cumulative_layout_shift**   | number  | 累积布局偏移（已弃用）       |

#### Session 信息

| 字段                           | 类型    | 说明                        |
| ------------------------------ | ------- | --------------------------- |
| **session.is_active**          | boolean | Session 是否活跃            |
| **session.sampled_for_replay** | boolean | Session 是否采样用于 replay |

#### 隐私设置

| 字段                     | 类型   | 必填 | 说明                                                |
| ------------------------ | ------ | ---- | --------------------------------------------------- |
| **privacy.replay_level** | string | ✅   | Replay 隐私级别：`allow`, `mask`, `mask-user-input` |

#### Feature Flags

| 字段              | 类型   | 说明                 |
| ----------------- | ------ | -------------------- |
| **feature_flags** | object | 特性标志（任意属性） |

#### Datadog 内部字段

| 字段                                          | 类型    | 必填 | 说明                                                        |
| --------------------------------------------- | ------- | ---- | ----------------------------------------------------------- |
| **\_dd.document_version**                     | integer | ✅   | View 事件的更新版本号                                       |
| **\_dd.page_states**                          | array   | ❌   | 页面状态列表                                                |
| **\_dd.page_states[].state**                  | string  | ✅   | 状态：`active`, `passive`, `hidden`, `frozen`, `terminated` |
| **\_dd.page_states[].start**                  | integer | ✅   | 状态开始时间（相对 view 起始，纳秒）                        |
| **\_dd.replay_stats.records_count**           | integer | ❌   | Replay 记录数                                               |
| **\_dd.replay_stats.segments_count**          | integer | ❌   | Replay 段数                                                 |
| **\_dd.replay_stats.segments_total_raw_size** | integer | ❌   | Replay 段总大小（字节）                                     |

#### 显示信息

| 字段                                      | 类型   | 说明                               |
| ----------------------------------------- | ------ | ---------------------------------- |
| **display.scroll.max_depth**              | number | 最大滚动深度（像素）               |
| **display.scroll.max_depth_scroll_top**   | number | 到达最大深度时的 scrollTop（像素） |
| **display.scroll.max_scroll_height**      | number | 最大滚动高度（像素）               |
| **display.scroll.max_scroll_height_time** | number | 到达最大滚动高度的时间（纳秒）     |

---

### 3.3 Action Event（动作事件）

#### Schema 文件

`dd-sdk-android-rum/src/main/json/rum/action-schema.json`

#### 数据结构

```json
{
  "type": "action",
  "allOf": ["_common-schema.json", "_view-container-schema.json"],
  "required": ["type", "view", "action"]
}
```

#### 核心字段说明

| 字段                    | 类型    | 必填 | 说明                                                                               |
| ----------------------- | ------- | ---- | ---------------------------------------------------------------------------------- |
| **type**                | string  | ✅   | 固定为 `action`                                                                    |
| **action.type**         | string  | ✅   | 动作类型：`custom`, `click`, `tap`, `scroll`, `swipe`, `application_start`, `back` |
| **action.id**           | string  | ❌   | Action UUID                                                                        |
| **action.loading_time** | integer | ❌   | Action 加载时间（纳秒）                                                            |
| **action.target.name**  | string  | ✅   | 目标名称                                                                           |

#### 计数统计

| 字段                       | 类型    | 说明                       |
| -------------------------- | ------- | -------------------------- |
| **action.error.count**     | integer | Action 上的 Error 数量     |
| **action.crash.count**     | integer | Action 上的 Crash 数量     |
| **action.long_task.count** | integer | Action 上的 Long Task 数量 |
| **action.resource.count**  | integer | Action 上的 Resource 数量  |

#### 挫败感信息

| 字段                        | 类型  | 说明                                                                           |
| --------------------------- | ----- | ------------------------------------------------------------------------------ |
| **action.frustration.type** | array | 挫败感类型：`rage_click`, `dead_click`, `error_click`, `rage_tap`, `error_tap` |

#### View 信息

| 字段                   | 类型    | 说明                  |
| ---------------------- | ------- | --------------------- |
| **view.in_foreground** | boolean | Action 是否在前台发生 |

#### Datadog 内部字段

| 字段                            | 类型    | 说明                                                                                                                          |
| ------------------------------- | ------- | ----------------------------------------------------------------------------------------------------------------------------- |
| **\_dd.action.position.x**      | integer | X 坐标（相对目标元素，像素）                                                                                                  |
| **\_dd.action.position.y**      | integer | Y 坐标（相对目标元素，像素）                                                                                                  |
| **\_dd.action.target.selector** | string  | CSS 选择器路径                                                                                                                |
| **\_dd.action.target.width**    | integer | 目标元素宽度（像素）                                                                                                          |
| **\_dd.action.target.height**   | integer | 目标元素高度（像素）                                                                                                          |
| **\_dd.action.name_source**     | string  | Action 名称来源策略：`custom_attribute`, `mask_placeholder`, `standard_attribute`, `text_content`, `mask_disallowed`, `blank` |

---

### 3.4 Error Event（错误事件）

#### Schema 文件

`dd-sdk-android-rum/src/main/json/rum/error-schema.json`

#### 数据结构

```json
{
  "type": "error",
  "allOf": [
    "_common-schema.json",
    "_action-child-schema.json",
    "_view-container-schema.json"
  ],
  "required": ["type", "view", "error"]
}
```

#### 核心字段说明

| 字段                     | 类型    | 必填 | 说明                                                                                                          |
| ------------------------ | ------- | ---- | ------------------------------------------------------------------------------------------------------------- |
| **type**                 | string  | ✅   | 固定为 `error`                                                                                                |
| **error.id**             | string  | ❌   | Error UUID                                                                                                    |
| **error.message**        | string  | ✅   | 错误消息                                                                                                      |
| **error.source**         | string  | ✅   | 错误来源：`network`, `source`, `console`, `logger`, `agent`, `webview`, `custom`, `report`                    |
| **error.stack**          | string  | ❌   | 错误堆栈                                                                                                      |
| **error.is_crash**       | boolean | ❌   | 是否导致应用崩溃                                                                                              |
| **error.fingerprint**    | string  | ❌   | 自定义分组指纹                                                                                                |
| **error.type**           | string  | ❌   | 错误类型                                                                                                      |
| **error.category**       | string  | ❌   | 错误类别：`ANR`, `App Hang`, `Exception`, `Watchdog Termination`, `Memory Warning`, `Network`                 |
| **error.handling**       | string  | ❌   | 错误处理方式：`handled`, `unhandled`                                                                          |
| **error.handling_stack** | string  | ❌   | 处理调用堆栈                                                                                                  |
| **error.source_type**    | string  | ❌   | 错误源类型：`android`, `browser`, `ios`, `react-native`, `flutter`, `roku`, `ndk`, `ios+il2cpp`, `ndk+il2cpp` |

#### 错误原因链

| 字段                       | 类型   | 说明         |
| -------------------------- | ------ | ------------ |
| **error.causes**           | array  | 错误原因列表 |
| **error.causes[].message** | string | 原因消息     |
| **error.causes[].type**    | string | 原因类型     |
| **error.causes[].stack**   | string | 原因堆栈     |
| **error.causes[].source**  | string | 原因来源     |

#### 网络错误信息

| 字段                               | 类型    | 说明        |
| ---------------------------------- | ------- | ----------- |
| **error.resource.method**          | string  | HTTP 方法   |
| **error.resource.status_code**     | integer | HTTP 状态码 |
| **error.resource.url**             | string  | 资源 URL    |
| **error.resource.provider.domain** | string  | 提供商域名  |
| **error.resource.provider.name**   | string  | 提供商名称  |
| **error.resource.provider.type**   | string  | 提供商类型  |

#### 线程信息

| 字段                        | 类型    | 说明                      |
| --------------------------- | ------- | ------------------------- |
| **error.threads**           | array   | 线程列表                  |
| **error.threads[].name**    | string  | 线程名称（如 `Thread 0`） |
| **error.threads[].crashed** | boolean | 是否崩溃                  |
| **error.threads[].stack**   | string  | 未符号化的堆栈            |
| **error.threads[].state**   | string  | 线程状态                  |

#### 二进制镜像信息

| 字段                                   | 类型    | 说明                       |
| -------------------------------------- | ------- | -------------------------- |
| **error.binary_images**                | array   | 二进制镜像列表（.so 文件） |
| **error.binary_images[].uuid**         | string  | 构建 UUID                  |
| **error.binary_images[].name**         | string  | 库名称                     |
| **error.binary_images[].is_system**    | boolean | 是否系统库                 |
| **error.binary_images[].load_address** | string  | 加载地址（十六进制）       |
| **error.binary_images[].max_address**  | string  | 最大地址（十六进制）       |
| **error.binary_images[].arch**         | string  | CPU 架构                   |

#### 错误元数据

| 字段                               | 类型    | 说明                   |
| ---------------------------------- | ------- | ---------------------- |
| **error.was_truncated**            | boolean | 堆栈是否因混淆而被截断 |
| **error.meta.code_type**           | string  | 崩溃进程的 CPU 架构    |
| **error.meta.parent_process**      | string  | 父进程信息             |
| **error.meta.incident_identifier** | string  | 事件标识符（UUID）     |
| **error.meta.process**             | string  | 崩溃进程名称           |
| **error.meta.exception_type**      | string  | BSD 终止信号名称       |
| **error.meta.exception_codes**     | string  | 异常代码               |
| **error.meta.path**                | string  | 可执行文件路径         |

#### CSP 违规

| 字段                      | 类型   | 说明                              |
| ------------------------- | ------ | --------------------------------- |
| **error.csp.disposition** | string | CSP 处理方式：`enforce`, `report` |

#### 其他

| 字段                           | 类型    | 说明                            |
| ------------------------------ | ------- | ------------------------------- |
| **error.time_since_app_start** | integer | 应用启动后的时间（毫秒）        |
| **freeze.duration**            | integer | ANR/App Hang 的冻结时间（纳秒） |
| **view.in_foreground**         | boolean | 错误是否在前台发生              |
| **feature_flags**              | object  | 特性标志                        |

---

### 3.5 Resource Event（资源事件）

#### Schema 文件

`dd-sdk-android-rum/src/main/json/rum/resource-schema.json`

#### 数据结构

```json
{
  "type": "resource",
  "allOf": [
    "_common-schema.json",
    "_action-child-schema.json",
    "_view-container-schema.json"
  ],
  "required": ["type", "view", "resource"]
}
```

#### 核心字段说明

| 字段                                | 类型    | 必填 | 说明                                                                                                     |
| ----------------------------------- | ------- | ---- | -------------------------------------------------------------------------------------------------------- |
| **type**                            | string  | ✅   | 固定为 `resource`                                                                                        |
| **resource.id**                     | string  | ❌   | Resource UUID                                                                                            |
| **resource.type**                   | string  | ✅   | 资源类型：`document`, `xhr`, `beacon`, `fetch`, `css`, `js`, `image`, `font`, `media`, `other`, `native` |
| **resource.method**                 | string  | ❌   | HTTP 方法                                                                                                |
| **resource.url**                    | string  | ✅   | 资源 URL                                                                                                 |
| **resource.status_code**            | integer | ❌   | HTTP 状态码                                                                                              |
| **resource.duration**               | integer | ❌   | 资源持续时间（纳秒）                                                                                     |
| **resource.size**                   | integer | ❌   | 响应体大小（字节）                                                                                       |
| **resource.encoded_body_size**      | integer | ❌   | 编码前的大小（字节）                                                                                     |
| **resource.decoded_body_size**      | integer | ❌   | 解码后的大小（字节）                                                                                     |
| **resource.transfer_size**          | integer | ❌   | 传输大小（字节）                                                                                         |
| **resource.render_blocking_status** | string  | ❌   | 渲染阻塞状态：`blocking`, `non-blocking`                                                                 |
| **resource.protocol**               | string  | ❌   | 网络协议（如 `http/1.1`, `h2`）                                                                          |
| **resource.delivery_type**          | string  | ❌   | 交付类型：`cache`, `navigational-prefetch`, `other`                                                      |

#### 资源时序

每个阶段包含 `duration` 和 `start` 两个字段（纳秒）：

| 字段                    | 说明         |
| ----------------------- | ------------ |
| **resource.worker**     | Worker 阶段  |
| **resource.redirect**   | 重定向阶段   |
| **resource.dns**        | DNS 解析阶段 |
| **resource.connect**    | 连接阶段     |
| **resource.ssl**        | SSL 握手阶段 |
| **resource.first_byte** | 首字节阶段   |
| **resource.download**   | 下载阶段     |

#### Provider 信息

| 字段                         | 类型   | 说明                                                                                                                                                                            |
| ---------------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **resource.provider.domain** | string | 提供商域名                                                                                                                                                                      |
| **resource.provider.name**   | string | 提供商名称                                                                                                                                                                      |
| **resource.provider.type**   | string | 提供商类型：`ad`, `advertising`, `analytics`, `cdn`, `content`, `customer-success`, `first party`, `hosting`, `marketing`, `other`, `social`, `tag-manager`, `utility`, `video` |

#### GraphQL 信息

| 字段                               | 类型   | 说明                                          |
| ---------------------------------- | ------ | --------------------------------------------- |
| **resource.graphql.operationType** | string | 操作类型：`query`, `mutation`, `subscription` |
| **resource.graphql.operationName** | string | 操作名称                                      |
| **resource.graphql.payload**       | string | 操作内容                                      |
| **resource.graphql.variables**     | string | 操作变量                                      |

#### Datadog 内部字段

| 字段                    | 类型    | 说明                                     |
| ----------------------- | ------- | ---------------------------------------- |
| **\_dd.span_id**        | string  | Span ID（十进制）                        |
| **\_dd.parent_span_id** | string  | 父 Span ID（十进制）                     |
| **\_dd.trace_id**       | string  | Trace ID（64 位十进制或 128 位十六进制） |
| **\_dd.rule_psr**       | number  | Trace 采样率（0-1）                      |
| **\_dd.discarded**      | boolean | 是否应丢弃该资源                         |

---

### 3.6 Long Task Event（长任务事件）

#### Schema 文件

`dd-sdk-android-rum/src/main/json/rum/long_task-schema.json`

#### 数据结构

```json
{
  "type": "long_task",
  "allOf": [
    "_common-schema.json",
    "_action-child-schema.json",
    "_view-container-schema.json"
  ],
  "required": ["type", "view", "long_task"]
}
```

#### 核心字段说明

| 字段                          | 类型    | 必填 | 说明                                          |
| ----------------------------- | ------- | ---- | --------------------------------------------- |
| **type**                      | string  | ✅   | 固定为 `long_task`                            |
| **long_task.id**              | string  | ❌   | Long Task UUID                                |
| **long_task.duration**        | integer | ✅   | 持续时间（纳秒）                              |
| **long_task.start_time**      | number  | ❌   | 开始时间                                      |
| **long_task.entry_type**      | string  | ❌   | 事件类型：`long-task`, `long-animation-frame` |
| **long_task.is_frozen_frame** | boolean | ❌   | 是否为冻结帧                                  |

#### 长动画帧特有字段

| 字段                                   | 类型    | 说明                       |
| -------------------------------------- | ------- | -------------------------- |
| **long_task.blocking_duration**        | integer | 阻塞时间（纳秒）           |
| **long_task.render_start**             | number  | 渲染开始时间（纳秒）       |
| **long_task.style_and_layout_start**   | number  | 样式和布局开始时间（纳秒） |
| **long_task.first_ui_event_timestamp** | number  | 首次 UI 事件时间（纳秒）   |

#### 脚本信息

| 字段                                                     | 类型    | 说明                       |
| -------------------------------------------------------- | ------- | -------------------------- |
| **long_task.scripts**                                    | array   | 长脚本列表                 |
| **long_task.scripts[].duration**                         | integer | 脚本持续时间（纳秒）       |
| **long_task.scripts[].pause_duration**                   | integer | 暂停时间（纳秒）           |
| **long_task.scripts[].forced_style_and_layout_duration** | integer | 强制样式和布局时间（纳秒） |
| **long_task.scripts[].start_time**                       | number  | 开始时间                   |
| **long_task.scripts[].execution_start**                  | number  | 执行开始时间               |
| **long_task.scripts[].source_url**                       | string  | 脚本 URL                   |
| **long_task.scripts[].source_function_name**             | string  | 函数名                     |
| **long_task.scripts[].source_char_position**             | integer | 字符位置                   |
| **long_task.scripts[].invoker**                          | string  | 调用者信息                 |
| **long_task.scripts[].invoker_type**                     | string  | 调用者类型                 |
| **long_task.scripts[].window_attribution**               | string  | 窗口归属                   |

#### Datadog 内部字段

| 字段               | 类型    | 说明               |
| ------------------ | ------- | ------------------ |
| **\_dd.discarded** | boolean | 是否应丢弃该长任务 |
| **\_dd.profiling** | object  | Profiling 上下文   |

---

### 3.7 Vital Event（性能指标事件）

#### Schema 文件

`dd-sdk-android-rum/src/main/json/rum/vital-schema.json`

#### 数据结构

```json
{
  "type": "vital",
  "allOf": ["_common-schema.json", "_view-container-schema.json"],
  "required": ["type", "vital"]
}
```

#### 核心字段说明

| 字段      | 类型   | 必填 | 说明                             |
| --------- | ------ | ---- | -------------------------------- |
| **type**  | string | ✅   | 固定为 `vital`                   |
| **vital** | object | ✅   | Vital 数据（可以是以下三种之一） |

#### Vital 通用字段

所有 Vital 都包含的通用字段：

| 字段                  | 类型   | 必填 | 说明                                                              |
| --------------------- | ------ | ---- | ----------------------------------------------------------------- |
| **vital.id**          | string | ✅   | Vital 的 UUID                                                     |
| **vital.name**        | string | ❌   | Vital 名称，用作 facet 路径，只能包含字母、数字和字符 `- _ . @ $` |
| **vital.description** | string | ❌   | Vital 描述，可用作次要标识符（URL、React 组件名等）               |

#### Vital 类型详解

##### 1. Duration Vital（持续时间指标）

**Schema**: `vital-duration-schema.json`

| 字段               | 类型   | 必填 | 说明              |
| ------------------ | ------ | ---- | ----------------- |
| **vital.type**     | string | ✅   | 固定为 `duration` |
| **vital.duration** | number | ✅   | 持续时间（纳秒）  |

**使用场景**：

- 自定义页面加载时间
- API 响应时间
- 数据库查询时间
- 文件处理时间

**示例**：

```json
{
  "type": "vital",
  "vital": {
    "type": "duration",
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "api_response_time",
    "description": "User profile API response time",
    "duration": 1500000000
  }
}
```

##### 2. App Launch Vital（应用启动指标）

**Schema**: `vital-app-launch-schema.json`

| 字段                                      | 类型    | 必填 | 说明                                 |
| ----------------------------------------- | ------- | ---- | ------------------------------------ |
| **vital.type**                            | string  | ✅   | 固定为 `app_launch`                  |
| **vital.app_launch_metric**               | string  | ✅   | 启动指标类型：`ttid`, `ttfd`         |
| **vital.duration**                        | number  | ✅   | 启动时间（纳秒）                     |
| **vital.startup_type**                    | string  | ❌   | 启动类型：`cold_start`, `warm_start` |
| **vital.is_prewarmed**                    | boolean | ❌   | 是否为预热启动                       |
| **vital.has_saved_instance_state_bundle** | boolean | ❌   | 是否有已保存的实例状态包             |

**关键指标说明**：

- **TTID (Time To Initial Display)**：从应用启动到首个像素显示的时间
- **TTFD (Time To Full Display)**：从应用启动到界面完全可用的时间
- **Cold Start**：应用进程不存在，需要完全启动
- **Warm Start**：应用进程存在但 Activity 不在内存中

**示例**：

```json
{
  "type": "vital",
  "vital": {
    "type": "app_launch",
    "id": "550e8400-e29b-41d4-a716-446655440001",
    "app_launch_metric": "ttid",
    "duration": 2000000000,
    "startup_type": "cold_start",
    "is_prewarmed": false,
    "has_saved_instance_state_bundle": false
  }
}
```

##### 3. Feature Operation Vital（功能操作指标）

**Schema**: `vital-feature-operation-schema.json`

| 字段                     | 类型   | 必填 | 说明                                        |
| ------------------------ | ------ | ---- | ------------------------------------------- |
| **vital.type**           | string | ✅   | 固定为 `operation_step`                     |
| **vital.operation_key**  | string | ❌   | 操作标识符，用于区分同名并行操作            |
| **vital.step_type**      | string | ❌   | 步骤类型：`start`, `update`, `retry`, `end` |
| **vital.failure_reason** | string | ❌   | 失败原因：`error`, `abandoned`, `other`     |

**使用场景**：

- 文件上传流程（开始 → 进度更新 → 重试 → 完成）
- 支付流程（开始 → 验证 → 支付 → 完成）
- 登录流程（开始 → 验证 → 授权 → 完成）

**示例**：

```json
{
  "type": "vital",
  "vital": {
    "type": "operation_step",
    "id": "550e8400-e29b-41d4-a716-446655440002",
    "name": "photo_upload",
    "operation_key": "profile_pic",
    "step_type": "end",
    "failure_reason": null
  }
}
```

#### Datadog 内部字段

| 字段                          | 类型    | 说明                                  |
| ----------------------------- | ------- | ------------------------------------- |
| **\_dd.vital.computed_value** | boolean | 值是否由 SDK 计算（而非用户直接提供） |
| **\_dd.profiling**            | object  | Profiling 上下文                      |

---

## 4. Session Replay（会话回放）

### Schema 文件

- `dd-sdk-android-session-replay/src/main/json/schemas/session-replay/mobile/segment-schema.json`
- `dd-sdk-android-session-replay/src/main/json/schemas/session-replay-mobile-schema.json`

### 数据结构

Session Replay 数据以 **Segment**（段）为单位上报，每个 Segment 包含：

#### Segment 结构

```json
{
  "type": "object",
  "title": "MobileSegment",
  "allOf": ["segment-metadata-schema.json"],
  "required": ["records"]
}
```

### Segment Metadata（段元数据）

继承自 `segment-context-schema.json` 和 `_common-segment-metadata-schema.json`：

| 字段                  | 类型    | 必填 | 说明                                                                        |
| --------------------- | ------- | ---- | --------------------------------------------------------------------------- |
| **source**            | string  | ✅   | 数据源：`android`, `ios`, `flutter`, `react-native`, `kotlin-multiplatform` |
| **application.id**    | string  | ✅   | 应用 ID                                                                     |
| **session.id**        | string  | ✅   | Session ID                                                                  |
| **view.id**           | string  | ✅   | View ID                                                                     |
| **start**             | integer | ✅   | 段开始时间（毫秒）                                                          |
| **end**               | integer | ✅   | 段结束时间（毫秒）                                                          |
| **records_count**     | integer | ✅   | 记录数量                                                                    |
| **has_full_snapshot** | boolean | ✅   | 是否包含完整快照                                                            |
| **index_in_view**     | integer | ✅   | 在 View 中的索引                                                            |
| **source_type**       | string  | ❌   | 源类型                                                                      |

### Records（记录）

每个 Segment 包含多个 Record，Record 类型包括：

#### 1. Full Snapshot Record（完整快照记录）

```json
{
  "type": 10,
  "timestamp": 1234567890,
  "data": {
    "wireframes": [...]
  }
}
```

| 字段                | 类型    | 说明           |
| ------------------- | ------- | -------------- |
| **type**            | integer | 固定为 `10`    |
| **timestamp**       | integer | 时间戳（毫秒） |
| **data.wireframes** | array   | Wireframe 列表 |

#### 2. Incremental Snapshot Record（增量快照记录）

```json
{
  "type": 11,
  "timestamp": 1234567890,
  "data": {
    "source": 0,
    "mutations": [...]
  }
}
```

| 字段          | 类型    | 说明           |
| ------------- | ------- | -------------- |
| **type**      | integer | 固定为 `11`    |
| **timestamp** | integer | 时间戳（毫秒） |
| **data**      | object  | 增量数据       |

#### 3. Meta Record（元数据记录）

用于记录视口信息等元数据。

#### 4. Focus Record（焦点记录）

用于记录焦点变化。

#### 5. View End Record（视图结束记录）

标记视图结束。

#### 6. Visual Viewport Record（可视视口记录）

记录视口的可视区域变化。

### Wireframe（线框）

Wireframe 是 Session Replay 的核心数据结构，用于描述 UI 元素。有以下五种类型：

#### 1. Shape Wireframe（形状线框）

用于描述矩形、圆形等基本形状：

```json
{
  "type": "shape",
  "id": 123,
  "x": 0,
  "y": 0,
  "width": 100,
  "height": 100,
  "clip": {...},
  "shapeStyle": {
    "backgroundColor": "#FFFFFF",
    "opacity": 1.0,
    "cornerRadius": 0
  },
  "border": {
    "color": "#000000",
    "width": 1
  }
}
```

#### 2. Text Wireframe（文本线框）

用于描述文本元素：

```json
{
  "type": "text",
  "id": 124,
  "x": 10,
  "y": 10,
  "width": 80,
  "height": 20,
  "text": "Hello World",
  "textStyle": {
    "family": "sans-serif",
    "size": 14,
    "color": "#000000"
  },
  "textPosition": {
    "alignment": {
      "horizontal": "left",
      "vertical": "top"
    }
  }
}
```

#### 3. Image Wireframe（图片线框）

用于描述图片元素：

```json
{
  "type": "image",
  "id": 125,
  "x": 0,
  "y": 0,
  "width": 100,
  "height": 100,
  "resourceId": "image-resource-id",
  "mimeType": "image/png"
}
```

#### 4. Placeholder Wireframe（占位符线框）

用于隐私遮蔽场景：

```json
{
  "type": "placeholder",
  "id": 126,
  "x": 0,
  "y": 0,
  "width": 100,
  "height": 100,
  "label": "Sensitive Content"
}
```

#### 5. WebView Wireframe（WebView 线框）

用于描述嵌入的 WebView：

```json
{
  "type": "webview",
  "id": 127,
  "x": 0,
  "y": 0,
  "width": 320,
  "height": 480,
  "slotId": "webview-slot-id"
}
```

### Wireframe Update（线框更新）

增量快照中的 mutations 包含 Wireframe 的更新操作：

| 操作类型   | 说明                |
| ---------- | ------------------- |
| **add**    | 添加新的 Wireframe  |
| **remove** | 移除 Wireframe      |
| **update** | 更新 Wireframe 属性 |

### 通用 Wireframe 属性

| 字段            | 类型    | 必填 | 说明                   |
| --------------- | ------- | ---- | ---------------------- |
| **id**          | integer | ✅   | Wireframe 唯一 ID      |
| **x**           | integer | ✅   | X 坐标（密度无关像素） |
| **y**           | integer | ✅   | Y 坐标（密度无关像素） |
| **width**       | integer | ✅   | 宽度（密度无关像素）   |
| **height**      | integer | ✅   | 高度（密度无关像素）   |
| **clip**        | object  | ❌   | 裁剪区域               |
| **clip.top**    | integer | ❌   | 顶部裁剪               |
| **clip.bottom** | integer | ❌   | 底部裁剪               |
| **clip.left**   | integer | ❌   | 左侧裁剪               |
| **clip.right**  | integer | ❌   | 右侧裁剪               |

### 资源管理

Session Replay 使用单独的资源管理系统：

#### Resource Metadata Schema

```json
{
  "identifier": "resource-id",
  "filename": "image.png",
  "resource_hash": "sha256-hash",
  "size": 12345,
  "mime_type": "image/png"
}
```

#### Resource Hashes Entry Schema

用于批量上报资源哈希，避免重复上传。

---

## 5. Telemetry（遥测数据）

### Schema 文件位置

`dd-sdk-android-rum/src/main/json/telemetry/`

### Telemetry 事件类型

Telemetry 用于 SDK 自身的监控和诊断，包含以下事件类型：

#### 1. Configuration Telemetry（配置遥测）

- Schema: `configuration-schema.json`
- 用于上报 SDK 配置信息

#### 2. Debug Telemetry（调试遥测）

- Schema: `debug-schema.json`
- 用于上报调试信息

#### 3. Error Telemetry（错误遥测）

- Schema: `error-schema.json`
- 用于上报 SDK 内部错误

#### 4. Usage Telemetry（使用遥测）

- Schema: `usage-schema.json`
- 用于上报 SDK 功能使用情况

### Usage Telemetry 特性

Usage Telemetry 细分为不同平台的特性：

#### Common Features（通用特性）

- Schema: `usage/common-features-schema.json`
- 所有平台共享的特性

#### Mobile Features（移动特性）

- Schema: `usage/mobile-features-schema.json`
- Android 和 iOS 特有的特性

#### Browser Features（浏览器特性）

- Schema: `usage/browser-features-schema.json`
- 浏览器特有的特性（用于 WebView）

### Telemetry Common Schema

所有 Telemetry 事件继承自 `telemetry/_common-schema.json`：

| 字段                    | 类型    | 说明           |
| ----------------------- | ------- | -------------- |
| **type**                | string  | Telemetry 类型 |
| **date**                | integer | 时间戳（毫秒） |
| **application.id**      | string  | 应用 ID        |
| **session.id**          | string  | Session ID     |
| **view.id**             | string  | View ID        |
| **service**             | string  | 服务名称       |
| **version**             | string  | SDK 版本       |
| **source**              | string  | 数据源         |
| **\_dd.format_version** | integer | 格式版本       |

---

## 附录：数据上报端点

### 默认上报端点

| 数据类型           | 端点                                                           |
| ------------------ | -------------------------------------------------------------- |
| **Logs**           | `https://browser-http-intake.logs.datadoghq.com/api/v2/logs`   |
| **Trace**          | `https://browser-http-intake.logs.datadoghq.com/api/v2/spans`  |
| **RUM**            | `https://browser-http-intake.logs.datadoghq.com/api/v2/rum`    |
| **Session Replay** | `https://browser-http-intake.logs.datadoghq.com/api/v2/replay` |

### 批量上报

所有数据类型都支持批量上报（batch）：

- 多个事件打包成一个数组
- 使用 JSON Lines 格式（每行一个 JSON 对象）
- 支持 gzip 压缩

### 请求头

```
Content-Type: application/json
DD-API-KEY: <api-key>
DD-EVP-ORIGIN: android
DD-EVP-ORIGIN-VERSION: <sdk-version>
DD-REQUEST-ID: <request-uuid>
```

---

## 总结

### 数据类型对比

| 特性            | Log      | Trace      | RUM          | Session Replay    |
| --------------- | -------- | ---------- | ------------ | ----------------- |
| **主要用途**    | 日志记录 | 分布式追踪 | 用户行为监控 | 用户会话回放      |
| **数据量**      | 中       | 中         | 高           | 非常高            |
| **实时性**      | 高       | 高         | 高           | 中                |
| **采样率**      | 100%     | 可配置     | 可配置       | 可配置            |
| **存储格式**    | JSON     | JSON       | JSON         | JSON + 二进制资源 |
| **与 RUM 关联** | 可选     | 可选       | -            | 必需              |

### 共同特性

1. **统一的用户标识**：所有数据类型都支持 `usr` 对象
2. **设备信息**：统一使用 `device` 和 `os` 对象
3. **网络信息**：统一的网络连接状态描述
4. **自定义属性**：都支持添加自定义属性
5. **标签系统**：使用 `ddtags` 进行数据分类
6. **批量上报**：支持批量减少网络请求

### Android SDK 特色

1. **移动端性能指标**：CPU、内存、电池、刷新率等
2. **ANR 监控**：应用无响应检测
3. **Crash 报告**：原生崩溃和线程信息
4. **慢帧检测**：UI 渲染性能监控
5. **跨平台支持**：支持 Flutter、React Native 等混合应用
6. **隐私保护**：Session Replay 支持多级隐私遮蔽

---

## 6. 与 Web SDK 对比

### 相同的核心概念

| 概念         | Android SDK | Web SDK | 说明                                |
| ------------ | ----------- | ------- | ----------------------------------- |
| **Session**  | ✅          | ✅      | 用户会话概念完全一致                |
| **View**     | ✅          | ✅      | 页面/屏幕概念一致，URL 字段略有不同 |
| **Action**   | ✅          | ✅      | 用户交互事件，但交互类型有差异      |
| **Error**    | ✅          | ✅      | 错误监控概念一致                    |
| **Resource** | ✅          | ✅      | 网络请求监控概念一致                |

### 字段差异对比

#### 1. View Event 差异

| 字段类别     | Android SDK 特有                                                           | Web SDK 特有                                           | 共同字段       |
| ------------ | -------------------------------------------------------------------------- | ------------------------------------------------------ | -------------- |
| **加载类型** | `activity_display`, `fragment_display`                                     | `initial_load`, `route_change`                         | -              |
| **移动性能** | `memory_average`, `cpu_ticks_count`, `refresh_rate_average`, `slow_frames` | -                                                      | `loading_time` |
| **Web 性能** | -                                                                          | `dom_complete`, `load_event`, `first_contentful_paint` | -              |
| **用户行为** | `in_foreground_periods`                                                    | 页面可见性 API                                         | `time_spent`   |

#### 2. Action Event 差异

| 交互类型     | Android SDK                                 | Web SDK                     |
| ------------ | ------------------------------------------- | --------------------------- |
| **移动特有** | `tap`, `swipe`, `back`, `application_start` | -                           |
| **Web 特有** | -                                           | `click`, `input`, `keydown` |
| **通用**     | `custom`, `scroll`                          | `custom`, `scroll`          |

#### 3. Error Event 差异

| 错误类型     | Android SDK                                                 | Web SDK               |
| ------------ | ----------------------------------------------------------- | --------------------- |
| **移动特有** | `ANR`, `App Hang`, `Watchdog Termination`, `Memory Warning` | -                     |
| **平台特有** | `ndk`, `ndk+il2cpp` 源类型                                  | JavaScript 运行时错误 |
| **崩溃信息** | `binary_images`, `threads` 详细信息                         | 简化的堆栈信息        |

#### 4. 设备信息差异

| 信息类别     | Android SDK                                                                   | Web SDK                       |
| ------------ | ----------------------------------------------------------------------------- | ----------------------------- |
| **硬件信息** | `device.battery_level`, `device.power_saving_mode`, `device.brightness_level` | -                             |
| **网络信息** | `connectivity.cellular`, `network.client.sim_carrier`                         | `connectivity.effective_type` |
| **系统信息** | Android 版本详细信息                                                          | 浏览器和操作系统信息          |

### 数据上报差异

#### 1. 上报端点

| SDK 类型        | 端点格式                                                |
| --------------- | ------------------------------------------------------- |
| **Android SDK** | `https://<site>.logs.datadoghq.com/api/v2/<type>`       |
| **Web SDK**     | `https://browser-http-intake.logs.<site>/api/v2/<type>` |

#### 2. 请求头差异

| 请求头          | Android SDK                                    | Web SDK                                    |
| --------------- | ---------------------------------------------- | ------------------------------------------ |
| **Origin 标识** | `DD-EVP-ORIGIN: android`                       | `DD-EVP-ORIGIN: browser`                   |
| **版本信息**    | `DD-EVP-ORIGIN-VERSION: <android-sdk-version>` | `DD-EVP-ORIGIN-VERSION: <web-sdk-version>` |
| **内容类型**    | 支持 `application/json` 和 `text/plain`        | 主要使用 `application/json`                |

### Session Replay 差异

#### Android SDK Session Replay

| 特性         | 说明                                                            |
| ------------ | --------------------------------------------------------------- |
| **数据格式** | Wireframe + 资源文件                                            |
| **UI 表示**  | 5 种 Wireframe 类型（Shape、Text、Image、Placeholder、WebView） |
| **隐私控制** | 细粒度控制（Image、Text、Touch 分别配置）                       |
| **性能影响** | 相对较低，基于视图树遍历                                        |

#### Web SDK Session Replay

| 特性         | 说明                |
| ------------ | ------------------- |
| **数据格式** | DOM 快照 + 增量变更 |
| **UI 表示**  | HTML/CSS 结构保持   |
| **隐私控制** | CSS 选择器规则      |
| **性能影响** | 依赖于 DOM 复杂度   |

### 移动端独有特性

#### 1. 移动性能监控

- **ANR 检测**：应用无响应监控
- **慢帧监控**：UI 渲染性能
- **内存/CPU 监控**：资源使用情况
- **电池状态**：电量和省电模式

#### 2. 移动网络特性

- **运营商信息**：SIM 卡运营商和信号强度
- **网络类型切换**：WiFi、蜂窝网络切换监控
- **网络质量评估**：基于连接质量的性能分析

#### 3. 移动应用生命周期

- **应用启动监控**：冷启动、温启动性能
- **前后台切换**：应用状态变化跟踪
- **Activity/Fragment 跟踪**：Android 特有的界面层级

### 跨平台一致性

#### 统一的概念

✅ **Session ID**：跨平台唯一会话标识  
✅ **User ID**：统一的用户标识  
✅ **Trace ID**：分布式追踪标识  
✅ **Error Fingerprint**：错误分组机制  
✅ **Custom Attributes**：自定义属性支持

#### 差异化的实现

⚠️ **View URL**：Android 使用 Activity/Fragment 名称，Web 使用实际 URL  
⚠️ **Action 名称**：Android 基于 View 标识，Web 基于 DOM 选择器  
⚠️ **Resource URL**：Android 为 API 端点，Web 为完整资源 URL

---

## 附录：SDK 版本兼容性

### Android SDK 版本要求

- **最低 Android 版本**：API 21 (Android 5.0)
- **推荐 Android 版本**：API 26+ (Android 8.0+)
- **Kotlin 版本要求**：1.7+
- **编译 SDK 版本**：API 34+

### 功能支持矩阵

| 功能               | Android 5.0-6.0 | Android 7.0-8.1 | Android 9.0+ |
| ------------------ | --------------- | --------------- | ------------ |
| **基础 RUM**       | ✅              | ✅              | ✅           |
| **Session Replay** | ⚠️ 部分支持     | ✅              | ✅           |
| **ANR 检测**       | ❌              | ✅              | ✅           |
| **网络监控**       | ✅              | ✅              | ✅           |
| **崩溃报告**       | ✅              | ✅              | ✅           |

---

**文档版本**: 2.0  
**生成时间**: 2025-10-29  
**基于**: Datadog Android SDK (develop branch)  
**对比版本**: Datadog Browser SDK v4.x
