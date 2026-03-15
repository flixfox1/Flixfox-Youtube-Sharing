# TASK-5: 迁移 Canvas→Screen 内联消费方到 camera.canvasToScreen()

## 前置依赖

- TASK-2（CameraController API 补齐）

> 注：本任务与 TASK-3 无依赖关系，可并行执行。

## 问题描述

4 个 View 层组件内联手写 `x * viewport.zoom + viewport.panX` 公式进行 Canvas→Screen 转换，而非调用 `camera.canvasToScreen()`。数学上等价但违反单一来源原则。

**注意：DOMNodeLayer.tsx 不在本任务范围内**，它在 TASK-7 中统一处理。

## 涉及文件

| 文件 | 当前内联公式 | 迁移目标 |
|------|------------|---------|
| `src/view/components/canvas/SelectionBox.tsx` | `bounds.x * viewport.zoom + viewport.panX` | `camera.canvasToScreen(bounds.x, bounds.y)` |
| `src/view/components/ui/AlignmentBar.tsx` | `topLeft.x * viewport.zoom + viewport.panX` | `camera.canvasToScreen(topLeft.x, topLeft.y)` |
| `src/view/components/canvas/FreehandTrail.tsx` | `p.x * viewport.zoom + viewport.panX` | `camera.canvasToScreen(p.x, p.y)` |
| `src/view/shapes/dom/TextEditorOverlay.tsx` | `s.position.x * zoom + panX` | `camera.canvasToScreen(s.position.x, s.position.y)` |

## 实施步骤

### 步骤 1：SelectionBox.tsx

找到渲染部分中使用 `bounds.x * viewport.zoom + viewport.panX` 的代码。替换为调用 `camera.canvasToScreen()`。

需要确认组件是否已有 `camera` 引用（通过 `useBootstrap()` 或 props）。如果没有，通过 `useBootstrap()` 获取。

```typescript
// 旧：
const left = bounds.x * viewport.zoom + viewport.panX;
const top = bounds.y * viewport.zoom + viewport.panY;

// 新：
const screenPos = camera.canvasToScreen(bounds.x, bounds.y);
const left = screenPos.x;
const top = screenPos.y;
```

注意：`width` 和 `height` 的缩放（`bounds.w * viewport.zoom`）不是坐标转换，是尺寸缩放，保持 `* viewport.zoom` 即可。

### 步骤 2：AlignmentBar.tsx

找到 `topLeft.x * viewport.zoom + viewport.panX` 的代码（约 L184-186）。

```typescript
// 旧：
const screenX = topLeft.x * viewport.zoom + viewport.panX;
const screenY = topLeft.y * viewport.zoom + viewport.panY - 48;

// 新：
const screenPos = camera.canvasToScreen(topLeft.x, topLeft.y);
const screenX = screenPos.x;
const screenY = screenPos.y - 48;
```

### 步骤 3：FreehandTrail.tsx

找到 map 中的内联转换（约 L105-108）：

```typescript
// 旧：
const screenPts = fresh.map(p => ({
  x: p.x * viewport.zoom + viewport.panX,
  y: p.y * viewport.zoom + viewport.panY,
  time: p.time,
}));

// 新：
const screenPts = fresh.map(p => {
  const sp = camera.canvasToScreen(p.x, p.y);
  return { x: sp.x, y: sp.y, time: p.time };
});
```

### 步骤 4：TextEditorOverlay.tsx

找到 `useOverlayPosition` hook 中的内联转换：

```typescript
// 旧：
left: s.position.x * zoom + panX,
top: s.position.y * zoom + panY,
width: s.size.width * zoom,
height: s.size.height * zoom,

// 新：
const screenPos = camera.canvasToScreen(s.position.x, s.position.y);
// ...
left: screenPos.x,
top: screenPos.y,
width: s.size.width * zoom,  // 尺寸缩放保持不变
height: s.size.height * zoom,
```

需要确认 `useOverlayPosition` hook 是否已有 `camera` 引用。

## ⚠️ 关键注意事项

1. `width * zoom` 和 `height * zoom` 是尺寸缩放，不是坐标转换，不需要改
2. 每个文件修改后立即 `getDiagnostics` 检查
3. 确认每个组件获取 `camera` 实例的方式（`useBootstrap()` hook 或 props）
4. `canvasToScreen` 返回 `{ x, y }` 对象，注意解构

## 验收标准

1. `SelectionBox.tsx` 中不再有 `* viewport.zoom + viewport.panX` 的坐标转换公式
2. `AlignmentBar.tsx` 中不再有 `* viewport.zoom + viewport.panX` 的坐标转换公式
3. `FreehandTrail.tsx` 中不再有 `* viewport.zoom + viewport.panX` 的坐标转换公式
4. `TextEditorOverlay.tsx` 中不再有 `* zoom + panX` 的坐标转换公式
5. 所有替换均调用 `camera.canvasToScreen()`
6. 尺寸缩放（`width * zoom`）保持不变
7. `getDiagnostics` 零错误
