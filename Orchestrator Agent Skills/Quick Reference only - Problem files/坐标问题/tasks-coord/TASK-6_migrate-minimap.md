# TASK-6: 迁移 Minimap 可视区域计算到 camera.getVisibleWorldBounds()

## 前置依赖

- TASK-2（CameraController.getVisibleWorldBounds 可用）

> 注：与 TASK-3、TASK-5 无依赖关系，可并行执行。

## 问题描述

`Minimap.tsx` 中的 `getViewbox` 函数内联手写 `-vp.panX / vp.zoom` 来计算可视区域的世界坐标，且使用 `window.innerWidth/Height` 作为容器尺寸（不准确，画布不占满窗口）。应迁移到使用 `camera.getVisibleWorldBounds()`。

## 涉及文件

| 文件 | 操作 |
|------|------|
| `src/view/components/ui/Minimap.tsx` | 修改 `getViewbox` 函数使用 `camera.getVisibleWorldBounds()` |

## 实施步骤

### 步骤 1：分析当前 getViewbox 实现

当前代码：
```typescript
export function getViewbox(
  vp: Viewport,
  bounds: CanvasBounds,
  scale: number, offX: number, offY: number,
): ViewboxState {
  const vx = -vp.panX / vp.zoom;
  const vy = -vp.panY / vp.zoom;
  const vw = (typeof window !== 'undefined' ? window.innerWidth : 1920) / vp.zoom;
  const vh = (typeof window !== 'undefined' ? window.innerHeight : 1080) / vp.zoom;
  return {
    x: (vx - bounds.minX) * scale + offX,
    y: (vy - bounds.minY) * scale + offY,
    width: vw * scale,
    height: vh * scale,
  };
}
```

`vx, vy, vw, vh` 就是可视区域在世界坐标系下的 rect。这正是 `camera.getVisibleWorldBounds()` 返回的内容。

### 步骤 2：修改 getViewbox 接受 visibleBounds 参数

```typescript
export function getViewbox(
  visibleBounds: Rect,
  canvasBounds: CanvasBounds,
  scale: number, offX: number, offY: number,
): ViewboxState {
  return {
    x: (visibleBounds.x - canvasBounds.minX) * scale + offX,
    y: (visibleBounds.y - canvasBounds.minY) * scale + offY,
    width: visibleBounds.width * scale,
    height: visibleBounds.height * scale,
  };
}
```

### 步骤 3：更新调用方

在 Minimap 组件中，将 `getViewbox(vp, ...)` 调用改为：

```typescript
const visibleBounds = camera.getVisibleWorldBounds();
const viewbox = getViewbox(visibleBounds, canvasBounds, scale, offX, offY);
```

### 步骤 4：清理

移除 `getViewbox` 对 `Viewport` 类型的依赖（如果不再需要）。

## 验收标准

1. `Minimap.tsx` 中不再有 `-vp.panX / vp.zoom` 内联计算
2. `Minimap.tsx` 中不再有 `window.innerWidth / window.innerHeight` 引用
3. 可视区域通过 `camera.getVisibleWorldBounds()` 获取
4. `getDiagnostics` 零错误
