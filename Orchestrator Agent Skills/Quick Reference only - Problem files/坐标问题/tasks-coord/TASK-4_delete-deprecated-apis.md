# TASK-4: 删除 getShapeWorldPosition + coordinateConvert.ts + 更新测试

## 前置依赖

- TASK-3（所有消费方已迁移完毕）

## 问题描述

TASK-3 完成后，`CanvasEditor.getShapeWorldPosition()` 和 `coordinateConvert.ts` 不再有运行时消费方。但 `GroupCommands.test.ts` 中大量测试断言仍在调用 `getShapeWorldPosition`。需要：

1. 更新测试文件，将 `getShapeWorldPosition` 调用替换为 `TransformService` 方法
2. 删除 `CanvasEditor.getShapeWorldPosition()` 方法
3. 删除 `src/foundation/geometry/coordinateConvert.ts`

## 涉及文件

| 文件 | 操作 |
|------|------|
| `src/core/editor/CanvasEditor.ts` | 删除 `getShapeWorldPosition` 方法 |
| `src/foundation/geometry/coordinateConvert.ts` | 整个文件删除 |
| `src/core/editor/commands/GroupCommands.test.ts` | 更新测试断言 |

## 实施步骤

### 步骤 1：更新 GroupCommands.test.ts

测试文件中大量使用 `editor.getShapeWorldPosition(shape)` 来验证世界坐标。替换为使用 `TransformService`：

```typescript
// 旧：
const worldA = editor.getShapeWorldPosition(editor.store.getShape(a)!);

// 新：
const worldBounds = transformService.getWorldBounds(a);
// worldBounds.x 和 worldBounds.y 即为世界坐标
```

**注意**：需要在测试 setup 中创建 `TransformService` 实例：

```typescript
const transformService = new TransformService(editor.store);
```

逐个替换所有 `getShapeWorldPosition` 调用。测试的断言逻辑不变，只是获取世界坐标的方式从 `getShapeWorldPosition` 改为 `TransformService.getWorldBounds`。

### 步骤 2：删除 CanvasEditor.getShapeWorldPosition

从 `src/core/editor/CanvasEditor.ts` 中删除 `getShapeWorldPosition` 方法及其 JSDoc 注释。

### 步骤 3：删除 coordinateConvert.ts

删除整个文件 `src/foundation/geometry/coordinateConvert.ts`。

用 grepSearch 确认没有其他文件 import 它。如果有遗漏的 import，一并清理。

### 步骤 4：全局搜索确认

搜索以下关键词确认无残留引用：
- `getShapeWorldPosition`
- `coordinateConvert`
- `worldToGroupLocal`
- `groupLocalToWorld`

排除 `.md` 文件和 `.Felix workflow` 目录中的引用（文档引用不需要清理）。

## 验收标准

1. `CanvasEditor` 上不再有 `getShapeWorldPosition` 方法
2. `src/foundation/geometry/coordinateConvert.ts` 文件不存在
3. 全局搜索 `getShapeWorldPosition`（排除 .md 和文档目录）无结果
4. 全局搜索 `coordinateConvert`（排除 .md 和文档目录）无结果
5. `GroupCommands.test.ts` 使用 `TransformService.getWorldBounds` 替代
6. `getDiagnostics` 零错误
7. 测试通过：`npx vitest run src/core/editor/commands/GroupCommands.test.ts`
