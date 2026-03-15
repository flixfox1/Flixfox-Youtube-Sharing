# TASK-1: 在 TransformService 中新增 getWorldBounds()

## 前置依赖

- TASK-0（localToWorld/worldToLocal 修正后，getFullWorldTransform 才可信赖）

## 问题描述

View 层多个消费方需要获取 shape 在世界坐标系下的 AABB 包围盒，但 `TransformService` 只提供矩阵级 API，缺少便捷的 `getWorldBounds` 方法。导致消费方各自手写 parent-chain-walk 累加 `bounds.x/y`。

## 涉及文件

| 文件 | 操作 |
|------|------|
| `src/core/editor/TransformService.ts` | 新增 `getWorldBounds` 方法 |
| `src/core/store/records/types.ts` | 读取 `Rect` 类型定义（只读，不修改） |

## 实施步骤

### 步骤 1：在 TransformService 中新增 `getWorldBounds` 方法

在 `getFullWorldTransform` 方法之后新增：

```typescript
/**
 * Returns the axis-aligned bounding box (AABB) of a shape in world coordinates.
 * Handles rotation correctly by transforming all four corners.
 */
getWorldBounds(shapeId: MShapeId): Rect {
  const shape = this.store.getShape(shapeId);
  if (!shape) return { x: 0, y: 0, width: 0, height: 0 };

  const { width, height } = shape.bounds;
  const mat = this.getFullWorldTransform(shapeId);

  // 局部空间四角：(0,0) → (w,h)
  // 注意：不是 (bounds.x, bounds.y) → (bounds.x+w, bounds.y+h)
  // 因为 getFullLocalTransform 已包含 matTranslate(bounds.x, bounds.y)
  const pts = [
    matApply(mat, { x: 0, y: 0 }),
    matApply(mat, { x: width, y: 0 }),
    matApply(mat, { x: 0, y: height }),
    matApply(mat, { x: width, y: height }),
  ];
  const xs = pts.map(p => p.x);
  const ys = pts.map(p => p.y);
  const minX = Math.min(...xs), minY = Math.min(...ys);
  const maxX = Math.max(...xs), maxY = Math.max(...ys);
  return { x: minX, y: minY, width: maxX - minX, height: maxY - minY };
}
```

### 步骤 2：确保 `Rect` 类型已导入

在文件顶部添加 `Rect` 的 import（如果尚未存在）：

```typescript
import type { Rect } from '../store/records/types';
```

## ⚠️ 关键注意事项

`getFullWorldTransform(shapeId)` 的矩阵链中，`getFullLocalTransform` 已包含 `matTranslate(bounds.x, bounds.y)`。因此 shape 的局部空间四角是 `(0,0)` → `(width, height)`，**不是** `(bounds.x, bounds.y)` → `(bounds.x+width, bounds.y+height)`。如果用后者，position offset 会被计算两次。

## 验收标准

1. `TransformService` 上存在 `getWorldBounds(shapeId: MShapeId): Rect` 方法
2. 方法内部使用 `getFullWorldTransform` 获取矩阵
3. 对局部空间四角 `(0,0)→(w,h)` 做 `matApply`，而非 `(bounds.x, bounds.y)→(bounds.x+w, bounds.y+h)`
4. `getDiagnostics` 零错误
