# TASK-2: 补齐 CameraController API（getViewportMatrix, getVisibleWorldBounds, containerSize）

## 前置依赖

无（与 TASK-0、TASK-1 无依赖关系，可并行）

## 问题描述

`CameraController` 缺少以下 API：
1. `getViewportMatrix(): Mat2x3` — View 层组合矩阵优化需要
2. `getVisibleWorldBounds(): Rect` — Minimap 等消费方需要，当前内联手写
3. `containerSize` signal + `setContainerSize()` — 画布不占满窗口，需要容器尺寸才能正确计算可视区域

## 涉及文件

| 文件 | 操作 |
|------|------|
| `src/view/camera/CameraController.ts` | 新增 3 个 API |

## 实施步骤

### 步骤 1：添加 import

在文件顶部添加：

```typescript
import type { Mat2x3 } from '@/src/foundation/geometry/Mat';
import type { Rect } from '@/src/core/store/records/types';
```

### 步骤 2：添加 containerSize signal

在 constructor 中的 `this.isMoving = signal(false);` 之后添加：

```typescript
readonly containerWidth: Signal<number>;
readonly containerHeight: Signal<number>;
```

在 constructor 体内初始化：

```typescript
this.containerWidth = signal(window.innerWidth);
this.containerHeight = signal(window.innerHeight);
```

### 步骤 3：添加 `setContainerSize` 方法

```typescript
/** Update container dimensions (call from CanvasShell on mount/resize) */
setContainerSize(width: number, height: number): void {
  this.containerWidth.value = width;
  this.containerHeight.value = height;
}
```

### 步骤 4：添加 `getViewportMatrix` 方法

```typescript
/**
 * Returns the viewport transform as a Mat2x3 matrix.
 * Screen = World × zoom + pan → [zoom, 0, 0, zoom, panX, panY]
 */
getViewportMatrix(): Mat2x3 {
  const z = this.zoom.value;
  const px = this.panX.value;
  const py = this.panY.value;
  return [z, 0, 0, z, px, py];
}
```

### 步骤 5：添加 `getVisibleWorldBounds` 方法

```typescript
/**
 * Returns the visible world-space rectangle based on current viewport and container size.
 */
getVisibleWorldBounds(): Rect {
  const w = this.containerWidth.value;
  const h = this.containerHeight.value;
  const topLeft = this.screenToCanvas(0, 0);
  const bottomRight = this.screenToCanvas(w, h);
  return {
    x: topLeft.x,
    y: topLeft.y,
    width: bottomRight.x - topLeft.x,
    height: bottomRight.y - topLeft.y,
  };
}
```

## 验收标准

1. `CameraController` 上存在 `getViewportMatrix(): Mat2x3` 方法，返回 `[zoom, 0, 0, zoom, panX, panY]`
2. `CameraController` 上存在 `getVisibleWorldBounds(): Rect` 方法
3. `CameraController` 上存在 `containerWidth` / `containerHeight` signal 和 `setContainerSize()` 方法
4. `getVisibleWorldBounds` 内部使用 `containerWidth/Height` signal 而非 `window.innerWidth/Height`
5. `getDiagnostics` 零错误
6. 现有方法（`screenToCanvas`, `canvasToScreen`, `fitToViewport` 等）不受影响

## 后续集成（不在本任务范围，但需记录）

`CanvasShell.tsx` 需要在 mount 时和 resize 时调用 `camera.setContainerSize(width, height)`。可通过 `ResizeObserver` 监听容器尺寸变化。此集成在 TASK-5/6/7 的消费方迁移过程中自然触发——当 `getVisibleWorldBounds()` 被实际调用时，需要确保 containerSize 已被正确设置。
