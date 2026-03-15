# TASK-3: 迁移 Local→World 消费方到 TransformService

## 前置依赖

- TASK-0（localToWorld/worldToLocal 修正）
- TASK-1（getWorldBounds 可用）

## 问题描述

5 个文件内联手写 parent-chain-walk 来计算 Local→World 坐标转换，绕过了 `TransformService`。需要逐个迁移到使用 `TransformService` 的矩阵方法。

**注意：DOMNodeLayer.tsx 不在本任务范围内**，它同时涉及 Local→World 和 Canvas→Screen 两层问题，将在 TASK-7 中统一处理。

## 涉及文件

| 文件 | 操作 |
|------|------|
| `src/core/editor/TransformService.ts` | 只读参考 |
| `src/core/editor/CanvasEditor.ts` | 修改 `getWorldSpatialEntry` 使用 TransformService |
| `src/view/components/canvas/CanvasRenderer.ts` | 修改 ancestor chain walk 使用 TransformService |
| `src/view/components/canvas/SelectionBox.tsx` | 修改 `unionBounds` 使用 TransformService |
| `src/view/components/overlays/TransformHandles.tsx` | 修改 `getParentWorldOffset` 使用 TransformService |
| `src/core/editor/commands/GroupCommands.ts` | 迁移到 TransformService 矩阵方法，移除 coordinateConvert 依赖 |

## 实施步骤

### 步骤 0：全局搜索 getShapeWorldPosition 的所有调用方

在开始迁移前，先 `grepSearch` 搜索 `getShapeWorldPosition`（排除 .md 文件），确认完整的调用方列表。已知调用方：
- `CanvasEditor.getWorldSpatialEntry`（内部调用）
- `GroupCommands.ts`（外部调用）
- `GroupCommands.test.ts`（测试中的断言，在 TASK-4 处理）

如果发现其他调用方，一并在本任务中迁移。

### 步骤 1：CanvasEditor.getWorldSpatialEntry

当前 `getWorldSpatialEntry` 调用 `this.getShapeWorldPosition(shape)` 获取世界坐标。

修改为使用 `this.transformService.getWorldBounds(shape.id)`：

```typescript
private getWorldSpatialEntry(shape: ShapeRecord) {
  if (!shape.parentId) return shapeToSpatialEntry(shape);
  const worldBounds = this.transformService.getWorldBounds(shape.id);
  return shapeToSpatialEntry({
    ...shape,
    bounds: worldBounds,
  });
}
```

**前提**：确认 `CanvasEditor` 已持有 `TransformService` 实例。如果没有，需要在构造函数中注入或创建。搜索 `CanvasEditor` 的构造函数确认。

### 步骤 2：CanvasRenderer.ts — ancestor chain walk

当前代码（约 L274-301）在渲染 Group 子元素时手动遍历 ancestor chain 并逐个 `ctx.translate` + `ctx.rotate`。

修改为使用 `TransformService.getFullWorldTransform(shapeId)` 获取组合矩阵，然后用 `ctx.setTransform` 或 `ctx.transform` 一次性应用：

```typescript
if (isGroupChild) {
  ctx.save();
  // 获取从 shape 的 parent 到 root 的组合变换矩阵
  // 注意：这里需要的是 parent 的世界变换，不是 shape 自身的
  const parentId = shape.parentId!;
  const parentMat = this.transformService.getFullWorldTransform(parentId);
  // Mat2x3 = [a, b, c, d, tx, ty]
  ctx.transform(parentMat[0], parentMat[1], parentMat[2], parentMat[3], parentMat[4], parentMat[5]);
}
```

**前提**：确认 `CanvasRenderer` 已持有 `TransformService` 实例。如果没有，需要在构造函数中注入。

### 步骤 3：SelectionBox.tsx — unionBounds

当前 `unionBounds` 函数内联遍历 parent chain 累加 offset。

修改为接受 `TransformService` 参数，使用 `getWorldBounds`：

```typescript
export function unionBounds(
  shapes: ShapeRecord[],
  overrides?: Map<MShapeId, Point> | null,
  store?: CanvasStore,
  transformService?: TransformService,
): Rect | null {
  // ...
  for (const s of shapes) {
    // Group 子元素：使用 TransformService 获取世界坐标
    if (s.parentId && transformService) {
      const wb = transformService.getWorldBounds(s.id);
      x = wb.x; y = wb.y; w = wb.width; h = wb.height;
    } else {
      // 顶层元素直接用 bounds
      // ...existing logic...
    }
  }
}
```

同步更新所有调用 `unionBounds` 的地方，传入 `transformService`。

### 步骤 4：TransformHandles.tsx — getParentWorldOffset

当前 `getParentWorldOffset` 函数遍历 parent chain 累加 `bounds.x/y`。

修改为使用 `TransformService`。由于该函数返回的是 parent 的世界偏移（不是 shape 自身的），可以改为：

```typescript
function getParentWorldOffset(
  shapes: ShapeRecord[],
  transformService: TransformService,
): { x: number; y: number } {
  const first = shapes[0];
  if (!first?.parentId) return { x: 0, y: 0 };
  const parentBounds = transformService.getWorldBounds(first.parentId);
  return { x: parentBounds.x, y: parentBounds.y };
}
```

同步更新调用方传入 `transformService`。

### 步骤 5：GroupCommands.ts — 迁移到 TransformService

当前 `GroupCommand.execute()` 中：
1. 调用 `this.editor.getShapeWorldPosition(s)` 获取世界位置
2. 使用 `worldToGroupLocal` / `groupLocalToWorld`（来自 `coordinateConvert.ts`）做坐标转换

修改为：
1. 使用 `TransformService.getWorldBounds(s.id)` 获取世界位置
2. 使用 `TransformService.worldToLocal()` / `localToWorld()` 替代 `coordinateConvert` 的函数

**关键**：`GroupCommands` 中的 `worldToGroupLocal` 接受裸参数（worldX, worldY, groupX, groupY, groupRotation），需要改为使用 `TransformService.worldToLocal(point, groupId)` 的 shapeId 方式。这要求 Group shape 已经创建并写入 Store 后才能调用 `worldToLocal`。检查执行顺序确保这一点。

同步移除 `import { worldToGroupLocal, groupLocalToWorld } from '@/src/foundation/geometry/coordinateConvert';`

## ⚠️ 关键注意事项

1. 每个步骤修改后立即 `getDiagnostics` 检查
2. `CanvasRenderer` 和 `SelectionBox` 可能需要注入 `TransformService` 实例——检查它们当前如何获取依赖
3. `GroupCommands` 的迁移最复杂，因为涉及 Group 创建时序问题——`worldToLocal(point, groupId)` 要求 Group 已在 Store 中
4. 不要修改 `DOMNodeLayer.tsx`，它在 TASK-7 中处理

## 验收标准

1. `CanvasEditor.getWorldSpatialEntry` 使用 `TransformService.getWorldBounds` 而非 `getShapeWorldPosition`
2. `CanvasRenderer.ts` 不再有 ancestor chain walk 代码，改用 `TransformService.getFullWorldTransform`
3. `SelectionBox.tsx` 的 `unionBounds` 使用 `TransformService.getWorldBounds` 而非内联 parent walk
4. `TransformHandles.tsx` 的 `getParentWorldOffset` 使用 `TransformService` 而非内联 parent walk
5. `GroupCommands.ts` 不再 import `coordinateConvert`，改用 `TransformService` 矩阵方法
6. `getDiagnostics` 零错误（所有修改的文件）
7. `view/` 目录下不再有手动遍历 `parentId` 链条的代码（DOMNodeLayer 除外）
