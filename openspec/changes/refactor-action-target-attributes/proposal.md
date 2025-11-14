# Refactor Action Target Attributes Structure

## Why

当前 RUM Action 事件的 target 相关属性（如 `action.target.classname`, `action.target.resource_id` 等）被序列化到事件的最外层，而不是在 `action.target` 对象内部。这种结构不够直观和一致，影响了数据的逻辑组织和可读性。

从用户提供的事件样例可以看出，`action.target.classname` 和 `action.target.resource_id` 出现在 JSON 的根级别，而不是在 `action.target` 对象中。这是由于 `RumEventSerializer.extractKnownAttributes()` 方法将这些属性从 `context` 中提取到最外层造成的。

## What Changes

- **重构 Action 事件结构**：将所有 action.target 相关属性移动到 `action.target` 对象内部
- **重构 Action 事件结构**：将所有 action.gesture 相关属性移动到 `action.gesture` 对象内部
- **更新序列化逻辑**：修改 `RumEventSerializer` 以支持新的嵌套结构
- **更新 JSON Schema**：修改 action-schema.json 以反映新的数据结构
- **保持向后兼容性**：考虑渐进式迁移策略

受影响的属性包括：
- `action.target.classname` → `action.target.classname`
- `action.target.resource_id` → `action.target.resource_id`
- `action.target.title` → `action.target.title`
- `action.target.parent.*` → `action.target.parent.*`
- `action.target.selected` → `action.target.selected`
- `action.target.role` → `action.target.role`
- `action.gesture.*` → `action.gesture.*`

## Impact

- **Affected specs**: rum-event-structure
- **Affected code**: 
  - `features/dd-sdk-android-rum/src/main/kotlin/com/datadog/android/rum/internal/domain/event/RumEventSerializer.kt`
  - `features/dd-sdk-android-rum/src/main/json/rum/action-schema.json`
  - Action 事件的数据模型类
  - 相关测试文件
- **Breaking change**: **BREAKING** - 改变了输出的 JSON 结构
- **Migration required**: 需要更新数据处理管道以适应新的嵌套结构
