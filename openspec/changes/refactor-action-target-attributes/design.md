# Design Document: Action Target Attributes Refactoring

## Context

当前 Datadog Android SDK 的 RUM Action 事件序列化存在结构不一致的问题。Target 相关属性（如 `action.target.classname`, `action.target.resource_id`）被序列化到事件的最外层，而不是逻辑上应该所在的 `action.target` 对象内部。

这个问题源于 `RumEventSerializer.extractKnownAttributes()` 方法的设计，该方法将预定义的"已知属性"从 `context` 对象中提取到 JSON 的根级别。

## Goals / Non-Goals

### Goals
- 改善 Action 事件的数据结构逻辑性和一致性
- 将 action.target.* 属性移动到 `action.target` 对象内部
- 将 action.gesture.* 属性移动到 `action.gesture` 对象内部
- 保持 API 的易用性
- 确保序列化性能不受显著影响

### Non-Goals
- 不改变现有的 RumAttributes 常量定义（保持 API 兼容性）
- 不影响其他事件类型的序列化逻辑
- 不改变属性收集的机制，只改变最终的序列化结构

## Decisions

### Decision 1: 嵌套结构设计

**选择的方案**：将属性直接嵌套到对应的对象中

**当前结构**：
```json
{
  "action": {
    "target": {
      "name": "Button"
    }
  },
  "action.target.classname": "androidx.appcompat.widget.AppCompatButton",
  "action.target.resource_id": "navigation_crash"
}
```

**新结构**：
```json
{
  "action": {
    "target": {
      "name": "Button",
      "classname": "androidx.appcompat.widget.AppCompatButton",
      "resource_id": "navigation_crash"
    },
    "gesture": {
      "direction": "up",
      "from_state": "idle",
      "to_state": "scrolling"
    }
  }
}
```

**替代方案考虑**：
1. 保持扁平结构但改变命名 - 被拒绝，因为不解决根本问题
2. 创建单独的 metadata 对象 - 被拒绝，因为增加了复杂性
3. 使用配置选项控制结构 - 被拒绝，因为增加了维护成本

### Decision 2: 序列化策略

**选择的方案**：修改数据模型和序列化逻辑，而不是后处理 JSON

**原因**：
- 更清晰的数据流
- 更好的类型安全
- 更容易测试和维护

**实现方式**：
1. 更新 ActionEvent 数据模型添加新的嵌套字段
2. 修改事件创建逻辑直接设置嵌套属性
3. 更新序列化器移除 extractKnownAttributes 对 action 属性的特殊处理

### Decision 3: 向后兼容性策略

**选择的方案**：Breaking Change with Migration Guide

**原因**：
- 这是一个结构性改进，值得进行 breaking change
- 扁平结构和嵌套结构无法同时存在而不产生冗余
- 提供清晰的迁移指南可以减少升级成本

**迁移支持**：
- 详细的迁移文档
- 在发布说明中突出显示变更
- 提供数据处理脚本示例

## Risks / Trade-offs

### Risk 1: 破坏现有的数据处理管道
- **影响**：使用这些事件的下游系统需要更新
- **缓解措施**：提供详细的迁移指南和示例代码

### Risk 2: 性能影响
- **影响**：序列化逻辑的改变可能影响性能
- **缓解措施**：进行性能测试，确保影响在可接受范围内

### Risk 3: 测试覆盖不足
- **影响**：可能引入未发现的回归
- **缓解措施**：全面更新测试套件，增加边界情况测试

### Trade-off: 一致性 vs 兼容性
- **选择**：优先考虑数据结构的一致性
- **原因**：长期来看，一致的数据结构更有价值

## Migration Plan

### Phase 1: 准备阶段
1. 更新数据模型和 schema
2. 实现新的序列化逻辑
3. 更新所有测试

### Phase 2: 发布阶段
1. 在发布说明中突出显示 breaking change
2. 提供迁移指南和示例
3. 更新文档

### Phase 3: 支持阶段
1. 监控社区反馈
2. 提供技术支持
3. 根据需要发布补丁

### Rollback Plan
如果发现严重问题：
1. 可以快速回滚到旧的序列化逻辑
2. 保留旧的测试作为回滚验证
3. 通过配置开关控制新旧行为（紧急情况）

## Open Questions

1. **性能影响**：新的嵌套结构是否会显著影响序列化性能？
   - 需要进行基准测试

2. **Schema 版本管理**：是否需要引入 schema 版本号？
   - 考虑为未来的结构变更做准备

3. **其他事件类型**：是否需要对 Error 事件的类似属性进行相同处理？
   - 可以在后续版本中考虑
