# TASK-7: 创建 coordHelpers.ts + 迁移 DOMNodeLayer.tsx

## 前置依赖

- TASK-2（CameraController.getViewportMatrix 可用）
- TASK-3（TransformService 消费方迁移完成，模式已验证）
- TASK-5（Canvas→Screen 迁移完成，模式已验证）

## 问题描述

`DOMNodeLayer.tsx` 是整个坐标管线中重复最严重的单个文件——它同时内联了：
1. parent-chain-walk（Local→World）：遍历 `parentId` 累加 `bounds.x/y`
2. canvas→screen 转换（World→Screen）：`x * viewport.zoom + viewport.panX`

需要先创建 View 层的 `coordHelpers.ts` 组合模块，然后将 DOMNodeLayer 迁移到使用 `getScreenTransform()`。

## 涉及文件

| 文件 | 操作 |
|------|------|
| `src/view/coordHelpers.ts` | 新建 |
| `src/view/components/canvas/DOMNodeLayer.tsx` | 迁移到使用 coordHelpers |

## 实施步骤

### 步骤 1：创建 `src/view/coordHelpers.ts`

```typescript
/**
 * View-layer coordinate helpers — combines TransformService (Local↔World)
 * with CameraController (World↔Screen) for convenience.
 *
 * Architecture rule: TransformService does NOT depend on CameraController.
 * Cross-layer composition happens only here, in the View layer.
 *
 * @module view/coordHelpers
 */

import type { MShapeId } from '@/src/foundation/id';
import type { Vec2 } from '@/src/foundation/geometry/Vec';
import type { Mat2x3 } from '@/src/foundation/geometry/Mat';
import { matMultiply, matApply } from '@/src/foundation/geometry/Mat';
import type { TransformService } from '@/src/core/editor/TransformService';
import type { CameraController } from './camera/CameraController';

/**
 * Local → Screen point conversion.
 * Internally: TransformService.localToWorld → CameraController.canvasToScreen.
 */
export function localToScreen(
  transform: TransformService,
  camera: CameraController,
  shapeId: MShapeId,
  local: Vec2,
): Vec2 {
  const world = transform.localToWorld(local, shapeId);
  return camera.canvasToScreen(world.x, world.y);
}

/**
 * Get the combined Local → Screen transform matrix.
 * screenFromLocal = viewportMatrix × worldFromLocalMatrix
 *
 * Use for batch point transforms (e.g., DOMNodeLayer rendering multiple children)
 * to avoid two function calls per point.
 */
export function getScreenTransform(
  transform: TransformService,
  camera: CameraController,
  shapeId: MShapeId,
): Mat2x3 {
  const worldFromLocal = transform.getFullWorldTransform(shapeId);
  const viewportMat = camera.getViewportMatrix();
  return matMultiply(viewportMat, worldFromLocal);
}
```

### 步骤 2：迁移 DOMNodeLayer.tsx

当前 DOMNodeLayer 中每个 shape 的渲染逻辑：

```typescript
// 1. Local→World: 内联 parent chain walk
let x = shape.position.x;
let y = shape.position.y;
if (shape.parentId) {
  let current = editor.store.getShape(shape.parentId);
  while (current) {
    x += current.bounds.x;
    y += current.bounds.y;
    current = current.parentId ? editor.store.getShape(current.parentId) : undefined;
  }
}
// 2. World→Screen: 内联公式
const tx = x * viewport.zoom + viewport.panX;
const ty = y * viewport.zoom + viewport.panY;
```

替换为使用 `getScreenTransform`：

```typescript
import { getScreenTransform } from '@/src/view/coordHelpers';
import { matApply } from '@/src/foundation/geometry/Mat';

// 对于每个 shape：
const screenMat = getScreenTransform(transformService, camera, shape.id);
// matApply 将 shape 的局部原点 (0,0) 映射到 screen 坐标
const screenPos = matApply(screenMat, { x: 0, y: 0 });
const tx = screenPos.x;
const ty = screenPos.y;
```

**注意**：`getScreenTransform` 返回的矩阵已包含 shape 自身的 `bounds.x/y` 平移（因为 `getFullWorldTransform` 包含 `getFullLocalTransform`）。所以输入点是 `(0, 0)` 而非 `(shape.position.x, shape.position.y)`。

经代码库验证，`position.x === bounds.x` 且 `position.y === bounds.y` 始终成立（`CanvasEditor` 在所有更新路径中同步维护两者），因此直接用 `(0, 0)` 即可。

### ⚠️ CSS Transform 与矩阵的交互

当前 DOMNodeLayer 的 CSS transform 结构是：

```typescript
const outerTransform = `translate(${tx}px, ${ty}px) scale(${viewport.zoom}) rotate(${shapeRotation}rad)`;
```

其中 `tx/ty` 已经是 screen 坐标（乘过 zoom + panX）。迁移到 `getScreenTransform` 后：

1. `matApply(screenMat, {x:0, y:0})` 给出的 screen position 等价于当前的 `tx/ty`，可以直接用于 `translate()`
2. `scale(viewport.zoom)` 是内容尺寸缩放，不是坐标转换，保持不变
3. **rotation 需要特别处理**：`getFullWorldTransform` 已包含 shape 的旋转，但 `getScreenTransform` 的矩阵用于计算位置时，旋转只影响位置偏移（对于 `(0,0)` 输入点，旋转不影响结果）。CSS 的 `rotate()` 仍然需要单独从 `shape.rotation` 读取，因为它控制的是 DOM 元素的视觉旋转，不是位置计算。

结论：CSS transform 结构保持 `translate + scale + rotate` 不变，只是 `translate` 的值从内联计算改为 `matApply(screenMat, {x:0, y:0})`。不要试图用 CSS `matrix()` 替代整个 transform——那会把 zoom 和 rotation 混在一起，破坏当前的 depth-focus 动画分层。

### 步骤 3：处理 override 场景

DOMNodeLayer 中可能有 position override（拖拽预览等）。如果有 override，需要：

```typescript
if (override) {
  // override 提供的是世界坐标，直接用 canvasToScreen
  const screenPos = camera.canvasToScreen(override.x, override.y);
  tx = screenPos.x;
  ty = screenPos.y;
} else {
  const screenMat = getScreenTransform(transformService, camera, shape.id);
  const screenPos = matApply(screenMat, { x: 0, y: 0 });
  tx = screenPos.x;
  ty = screenPos.y;
}
```

### 步骤 4：确保 DOMNodeLayer 能获取 TransformService

检查 DOMNodeLayer 的 props 或 context，确认它能获取 `TransformService` 实例。如果不能，通过 `useBootstrap()` 或 `useEditor()` 获取。

## ⚠️ 关键注意事项

1. `getScreenTransform` 的矩阵已包含 shape 的 position 平移，输入点应为 `(0, 0)`
2. `viewport.zoom` 的缩放（用于 CSS `scale()`）仍然需要单独获取，`getScreenTransform` 只用于计算 translate 位置
3. shape 的 rotation 也已包含在矩阵中，但 CSS `rotate()` 仍需单独从 `shape.rotation` 读取——矩阵用于位置计算，CSS rotate 用于视觉旋转（见上文"CSS Transform 与矩阵的交互"）
4. 性能：对同一 parent 下的多个子元素，可以缓存 parent 的 `getFullWorldTransform`，但当前实现中每个 shape 独立调用 `getScreenTransform` 也是可接受的（TransformService 内部有 per-frame 缓存）
5. 不要用 CSS `matrix()` 替代 `translate + scale + rotate`——会破坏 depth-focus 动画的分层结构

## 验收标准

1. `src/view/coordHelpers.ts` 文件存在，包含 `localToScreen` 和 `getScreenTransform` 两个函数
2. `coordHelpers.ts` 只 import `TransformService`（type-only）和 `CameraController`（type-only），不引入新的计算逻辑
3. `DOMNodeLayer.tsx` 不再有 parent-chain-walk 代码
4. `DOMNodeLayer.tsx` 不再有 `* viewport.zoom + viewport.panX` 内联公式
5. `DOMNodeLayer.tsx` 使用 `getScreenTransform` 或 `localToScreen` 进行坐标转换
6. `getDiagnostics` 零错误
