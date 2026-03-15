# 坐标系统统一方案（融合设计）

> 基于 Codex 与 Gemini 两份独立分析的融合，结合项目实际代码库审查
> 经重构架构师（Refactoring Architect）审查修订
> 用途：作为实施的唯一参考文档

---

## 背景

项目存在三层坐标空间：

```
Screen (e.clientX/Y) ──→ Canvas/World (无限画布) ──→ Local (相对于父 Group)
       CameraController          TransformService
```

两层转换各自存在散弹式重复实现的问题：

**Local ↔ World（TransformService 层）**：转换逻辑被 6 处独立实现。`TransformService` 已提供完整的矩阵变换 API（`getFullWorldTransform`、`localToWorld`、`worldToLocal`），但 View 层的 4 个消费方无一使用，全部手写简化版。核心原因是 `TransformService` 返回 `Mat2x3` 矩阵，对只需 x/y 偏移的场景显得过重，且缺少轻量便捷 API。当前 `CanvasEditor.getShapeWorldPosition()` 采用简单的 `bounds.x/y` 累加，在无旋转场景下正确，但一旦启用 Group 旋转将彻底失效。

**Canvas ↔ Screen（CameraController 层）**：`CameraController` 已提供 `screenToCanvas()` 和 `canvasToScreen()` 方法。Screen→Canvas 方向（输入）合规——所有指针事件入口均正确调用 `camera.screenToCanvas()`。但 Canvas→Screen 方向（输出）存在 5 处内联手写 `x * zoom + panX` 的重复实现，无一调用 `camera.canvasToScreen()`。此外 `CameraController` 缺少 `getVisibleWorldBounds()` 公共方法，导致 `Minimap.tsx` 内联手写可视区域计算。

---

## 设计原则

1. 底层死守矩阵乘法，便捷 API 只是矩阵结果的投影，不引入独立计算路径
2. 不暴露中间量（偏移量），只暴露转换结果
3. 非破坏性新增，不改现有 `getFullWorldTransform()` 签名
4. 所有 Local ↔ World 计算必须且只能通过 `TransformService`
5. 所有 Canvas ↔ Screen 计算必须且只能通过 `CameraController`
6. `TransformService` 不依赖 `CameraController`，跨层组合只在 View 层的 `coordHelpers.ts` 中完成

---

## 层次 1：补齐便捷 API（立即执行）

### 采纳的 API 设计

在 `TransformService` 中新增以下方法：

```typescript
// 返回 shape 在世界坐标系下的 AABB 包围盒
getWorldBounds(shapeId: MShapeId): Rect;

// 将 shape 局部坐标系下的点转换为世界点
localToWorld(point: Vec2, shapeId: MShapeId): Vec2;  // 已存在，无需新增

// 将世界坐标系下的点转换到 shape 的局部坐标系
worldToLocal(point: Vec2, shapeId: MShapeId): Vec2;  // 已存在，无需新增
```

实际上 `TransformService` 已经拥有 `localToWorld` 和 `worldToLocal`。真正缺失的只有 `getWorldBounds`。

> ⚠️ **审查修订：`localToWorld` / `worldToLocal` 存在实现 bug**
>
> 现有 `localToWorld` 和 `worldToLocal` 的签名正确，但内部实现使用的是 `getWorldTransform()`（仅旋转，不含 position offset），而非 `getFullWorldTransform()`（position + rotation）。对于 Group 子元素，`getWorldTransform` 会丢失所有祖先的 position offset，导致结果完全错误。
>
> 必须在本轮工作中修正：将 `localToWorld` 和 `worldToLocal` 的实现改为使用 `getFullWorldTransform()`。这是所有后续迁移的前提——否则消费方迁移到 `localToWorld()` 后在 Group 子元素场景下仍然会得到错误结果。

### `getWorldBounds` 实现

> ⚠️ **审查修订：原实现存在坐标双重计算 bug，已修正**
>
> `getFullWorldTransform(shapeId)` 的矩阵链中，`getFullLocalTransform` 已包含 `matTranslate(bounds.x, bounds.y)`——即该矩阵已将 shape 的局部原点 `(0,0)` 映射到世界空间的正确位置。如果再把 `(bounds.x, bounds.y)` 作为输入点传入，position offset 会被计算两次。shape 的局部空间四角应为 `(0,0)` → `(width, height)`，而非 `(bounds.x, bounds.y)` → `(bounds.x+width, bounds.y+height)`。

```typescript
getWorldBounds(shapeId: MShapeId): Rect {
  const shape = this.store.getShape(shapeId);
  if (!shape) return { x: 0, y: 0, width: 0, height: 0 };

  const { width, height } = shape.bounds;
  const mat = this.getFullWorldTransform(shapeId);

  // 局部空间四角：(0,0) → (w,h)，不是 (bounds.x, bounds.y) → (bounds.x+w, bounds.y+h)
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

### 关于 `getAncestorOffset` 的决定：不采纳

Codex 建议提供 `getAncestorOffset(parentId): Point` 作为"只需 translation"场景的便捷 API。该方案不被采纳，理由如下：

1. 项目中 `ShapeRecord` 已有 `rotation` 字段，Group 旋转是明确的未来方向。暴露一个"偏移量"API 会在 Group 旋转启用后立即产生误用——这正是当前 `getShapeWorldPosition()` 简单累加 `bounds.x/y` 的同类错误
2. 当前 `CanvasEditor.getShapeWorldPosition()` 就是这个思路的产物：它在无旋转时正确，但本质上是一个定时炸弹。新增一个同类 API 等于制造第二个定时炸弹
3. Gemini 的论证是正确的：父 Group 旋转 90° 时，子元素 `(100, 0)` 的世界偏移是 `(0, 100)` 而非 `(100, 0)`，简单累加无法处理
4. 性能方面，Codex 提到"跳过完整矩阵乘法直接取 translation 分量"的优化思路有价值，但应作为 `getFullWorldTransform` 内部的缓存优化实现，而非暴露为独立 API

### 关于 `getShapeWorldPosition` 的处置

`CanvasEditor.getShapeWorldPosition()` 是当前项目中最危险的方法——它用简单累加模拟矩阵变换，在无旋转时碰巧正确，但会在 Group 旋转启用后全面崩溃。处置方案：

> ⚠️ **审查修订：不标 `@deprecated`，直接替换 + 删除**
>
> 原方案的"委托 + 标记 deprecated + 渐进迁移"策略与项目的 Big Bang 重构原则冲突——steering rules 明确规定"No adapters, no gradual migration, no `@deprecated` marking"。这是零用户项目，没有外部消费方需要过渡期。

1. 在同一批次中，将所有调用 `getShapeWorldPosition()` 的消费方直接迁移到 `TransformService.getWorldBounds()` 或 `localToWorld()`
2. 迁移完成后立即删除 `getShapeWorldPosition()`
3. `GroupCommands.ts` 中的调用点优先替换，因为 Group 操作是旋转敏感的

### 关于 `coordinateConvert.ts` 的处置

`src/foundation/geometry/coordinateConvert.ts` 中的 `worldToGroupLocal` / `groupLocalToWorld` 是独立于 `TransformService` 的另一套坐标转换实现。它接受裸参数（`worldX, worldY, worldRotation, groupX, groupY, groupRotation`）而非 `shapeId`，且只处理单层 parent 关系。

该模块目前仅被 `GroupCommands.ts` 使用。处置方案：

1. `GroupCommands` 迁移到使用 `TransformService` 的矩阵方法，因为嵌套 Group 场景下单层转换不够（`coordinateConvert.ts` 只处理单层 parent 关系）
2. 迁移完成后直接删除 `coordinateConvert.ts`——不保留为"降级内部工具"，避免留下诱导绕过 `TransformService` 的入口

### 性能与缓存

采纳 Codex 的缓存建议，但实现方式调整为：

- 在 `TransformService` 中增加 per-frame 缓存，key 为 `shapeId`，在每帧开始时或 Store 变更时 invalidate
- 不在便捷 API 层做缓存，缓存统一在 `getFullWorldTransform` 内部
- 开发模式下记录缓存命中率和热点调用，辅助性能调优
- v1 采用 per-frame 全清策略（简单可靠），后续按需优化为 per-shape invalidation（shape signal 变更时 invalidate 对应 shape 及其所有子孙）

---

## 层次 1.5：Screen ↔ Canvas 统一（随层次 1 同步执行）

> 本节为代码库审查后新增，原 Codex/Gemini 两份报告均未涉及此层。

### 现状审查

`CameraController`（`src/view/camera/CameraController.ts`）已提供标准的坐标转换方法：

```typescript
screenToCanvas(sx: number, sy: number): { x: number; y: number }  // (sx - panX) / zoom
canvasToScreen(cx: number, cy: number): { x: number; y: number }  // cx * zoom + panX
```

**Screen → Canvas（输入方向）：合规。** 所有指针事件入口均正确调用 `camera.screenToCanvas()`：
- `CanvasContent.clientToCanvas()` — 先减去容器偏移，再委托 `camera.screenToCanvas()`
- `TransformHandles.tsx` — 直接调用 `camera.screenToCanvas(e.clientX, e.clientY)`
- `ImageShapeView.tsx` — 通过 `clientToCanvas()` 辅助函数调用

**Canvas → Screen（输出方向）：存在散弹式重复。** `camera.canvasToScreen()` 存在且正确，但 View 层 5 个组件全部内联手写 `x * zoom + panX`，无一调用它：

| # | 文件 | 内联公式 | 应调用 |
|---|------|---------|--------|
| 1 | `SelectionBox.tsx` | `bounds.x * viewport.zoom + viewport.panX` | `camera.canvasToScreen()` |
| 2 | `AlignmentBar.tsx` | `topLeft.x * viewport.zoom + viewport.panX` | `camera.canvasToScreen()` |
| 3 | `FreehandTrail.tsx` | `p.x * viewport.zoom + viewport.panX` | `camera.canvasToScreen()` |
| 4 | `TextEditorOverlay.tsx` | `s.position.x * zoom + panX` | `camera.canvasToScreen()` |
| 5 | `DOMNodeLayer.tsx` | `x * viewport.zoom + viewport.panX` | `camera.canvasToScreen()` |

数学上全部正确，但如果 `canvasToScreen` 的公式将来变化（如加入 canvas rotation 或 DPR 校正），这 5 处都需要手动修改。

**特别注意：`DOMNodeLayer.tsx` 是两个层次问题的交叉点** — 它同时内联了 parent-chain-walk（Local→World）和 canvas→screen 转换（World→Screen），是整个坐标管线中重复最严重的单个文件。

### `CameraController` 缺失的 API

**`getVisibleWorldBounds()`**：`CanvasRenderer.getViewportBounds()` 是私有方法，通过调用两次 `screenToCanvas` 计算可视区域。`Minimap.tsx` 则内联手写 `-vp.panX / vp.zoom` 来计算同样的东西。该方法应提升为 `CameraController` 上的公共方法：

> ⚠️ **审查修订：不使用 `window.innerWidth/Height`**
>
> 画布不占满整个窗口——有 Toolbar、TopBar、RightPanel 等 UI 元素。现有 `CanvasContent.clientToCanvas()` 会先减去容器偏移再调用 `screenToCanvas`，证实 canvas 容器有独立 bounds。应让 `CameraController` 持有容器尺寸 signal（在 `CanvasShell` mount 时设置），或接受容器尺寸作为参数。

```typescript
// 方案 A：接受参数（无状态，更简单）
getVisibleWorldBounds(containerWidth: number, containerHeight: number): Rect {
  const topLeft = this.screenToCanvas(0, 0);
  const bottomRight = this.screenToCanvas(containerWidth, containerHeight);
  return {
    x: topLeft.x,
    y: topLeft.y,
    width: bottomRight.x - topLeft.x,
    height: bottomRight.y - topLeft.y,
  };
}

// 方案 B：CameraController 持有 containerSize signal（更方便，消费方无需传参）
// 在 CanvasShell mount 时调用 camera.setContainerSize(width, height)
// getVisibleWorldBounds() 内部读取 containerSize signal
```

推荐方案 B——`CameraController` 已经是 View 层，持有容器尺寸 signal 不违反分层规则，且 `Minimap` 等消费方调用更简洁。

**`getViewportMatrix(): Mat2x3`**：当前 `CameraController` 只暴露标量方法（`screenToCanvas` / `canvasToScreen`），不暴露矩阵。但 View 层的组合矩阵优化（见下文"层次 1.75"）需要一个 viewport 矩阵才能与 `TransformService.getFullWorldTransform()` 预乘。新增：

```typescript
getViewportMatrix(): Mat2x3 {
  const z = this.zoom.value;
  const px = this.panX.value;
  const py = this.panY.value;
  // Screen = World × zoom + pan → Mat2x3 = [zoom, 0, 0, zoom, panX, panY]
  return [z, 0, 0, z, px, py];
}
```

### 为什么内联 `canvasToScreen` 比内联 parent-chain-walk 风险低

两者是同一模式（正确的基础设施存在但消费方不用），但风险等级不同：

- parent-chain-walk 的内联版本在 Group 旋转启用后会产生错误结果（数学上不等价）
- `canvasToScreen` 的内联版本目前数学上与 `camera.canvasToScreen()` 完全等价，不会产生错误结果

因此 Screen↔Canvas 层的统一优先级低于 Local↔World 层，但仍应作为同批次工作执行，原因是：
1. 防止未来公式变更时的散弹式修改
2. `DOMNodeLayer.tsx` 的迁移会同时触及两个层次，不宜分开做
3. 统一后代码意图更清晰——`camera.canvasToScreen(x, y)` 比 `x * viewport.zoom + viewport.panX` 更具语义

### 需要迁移的 Canvas→Screen 消费方

| # | 文件 | 迁移方式 |
|---|------|---------|
| 1 | `SelectionBox.tsx` | 替换为 `camera.canvasToScreen(bounds.x, bounds.y)` |
| 2 | `AlignmentBar.tsx` | 替换为 `camera.canvasToScreen(topLeft.x, topLeft.y)` |
| 3 | `FreehandTrail.tsx` | 在 map 中调用 `camera.canvasToScreen(p.x, p.y)` |
| 4 | `TextEditorOverlay.tsx` | 替换为 `camera.canvasToScreen(s.position.x, s.position.y)` |
| 5 | `DOMNodeLayer.tsx` | 先通过 `TransformService` 得到 world 坐标，再调用 `camera.canvasToScreen()` |
| 6 | `Minimap.tsx` 可视区域计算 | 替换为 `camera.getVisibleWorldBounds()` |

---

## 层次 1.75：View 层坐标 Helper 与组合矩阵（随层次 1/1.5 完成后执行）

### 架构决策：不在 TransformService 中加入 Screen 相关方法

`DOMNodeLayer.tsx` 的 Local→World→Screen 串联管线引出了一个设计问题：是否提供 `localToScreen` 快捷 API？

**决定：不把 `localToScreen` 放进 `TransformService`。**

理由（参考 `是否暴露中间坐标空间.md` 的分析）：

1. **职责污染**：`TransformService` 是几何空间系统（scene graph transform），`CameraController` 是视图系统（viewport projection）。如果 `TransformService.localToScreen()` 存在，意味着 `TransformService` 依赖 `CameraController`，这是反向依赖。成熟画布系统（Figma、tldraw、Konva）都保持这条边界
2. **World 空间是核心空间，不应被隐藏**：碰撞检测、snapping、alignment、selection box、spatial index、hit testing 全部在 World 空间工作。如果 View 层习惯直接用 `localToScreen`，会导致开发者忘记 World 的存在，长期产生坐标混乱
3. **CanvasRenderer 的矩阵栈已经证明了分层的正确性**：Canvas 2D 的 `ctx.translate` + `ctx.scale` 本质上就是 `Local → World → Screen` 两步由矩阵栈自动完成。DOM 没有矩阵栈，所以需要手动做两步，但管线结构是一样的

### 正确的做法：View 层 `coordHelpers.ts`

在 `src/view/` 下新增一个组合 helper 模块，专门给 View 层使用：

```typescript
// src/view/coordHelpers.ts

import type { MShapeId } from '@/src/foundation/id';
import type { Vec2 } from '@/src/foundation/geometry/Vec';
import type { Mat2x3 } from '@/src/foundation/geometry/Mat';
import { matMultiply, matApply } from '@/src/foundation/geometry/Mat';
import type { TransformService } from '@/src/core/editor/TransformService';
import type { CameraController } from './camera/CameraController';

/**
 * Local → Screen 组合转换。
 * 内部走 TransformService.localToWorld → CameraController.canvasToScreen。
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
 * 获取 Local → Screen 的组合矩阵。
 * screenFromLocal = viewportMatrix × worldFromLocalMatrix
 *
 * 用于批量点转换场景（如 DOMNodeLayer 渲染多个子元素），
 * 避免对每个点做两次函数调用。
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

### 职责分层总结

| 层级 | 模块 | 职责 |
|------|------|------|
| Foundation | `Mat.ts` | 纯矩阵数学 |
| Editor | `TransformService` | Local ↔ World（scene graph transform） |
| View | `CameraController` | World ↔ Screen（viewport projection） |
| View | `coordHelpers.ts` | Local ↔ Screen（组合，不引入新逻辑） |

### `getScreenTransform` 的性能价值

当前 `DOMNodeLayer` 对每个 Group 子元素做两步：遍历 parent chain 得到 world 坐标，再乘以 zoom/pan 得到 screen 坐标。`getScreenTransform` 将这两步预乘为一个矩阵，对同一 Group 下的多个子元素只需算一次矩阵乘法，后续每个子元素只需一次 `matApply`。在大量嵌套节点场景下性能提升明显。

### 关于 Document Space 的备注

`是否暴露中间坐标空间.md` 提到了 Document Space vs World Space 的选择。当前项目是单画布、无 artboard/page 概念，World Space 即 Document Space，无需额外分层。如果未来引入多 artboard 或导出特定区域，需要在 World 和 Document 之间再加一层。此决策点已记录，暂不行动。

---

## 层次 2：架构规则约束（随层次 1 同步落地）

### 硬规则

> **规则 A（Local ↔ World）**：禁止在 View/UI 层手动遍历 `parentId` 链条计算坐标。手动遍历无法正确处理父级的旋转和缩放，会导致选中框错位、碰撞检测失败。所有 Local ↔ World 计算必须且只能通过 `TransformService`。
>
> **规则 B（Canvas ↔ Screen）**：禁止在 View/UI 层内联 `x * zoom + panX` 或 `(x - panX) / zoom` 公式。所有 Canvas ↔ Screen 计算必须且只能通过 `CameraController.canvasToScreen()` / `screenToCanvas()`。

### 落地手段

融合两份报告的建议，按实际可行性排序：

1. **Steering Rule**：写入 `.kiro/steering/` 规则文件，Agent 和人类开发者在编写/审查代码时自动遵循（同时覆盖规则 A 和规则 B）
2. **PR Review checklist**：全局搜索 `view/` 目录下的 `.parentId` 遍历和内联 `* viewport.zoom + viewport.panX` 公式，发现即打回
3. **ESLint/AST 检查**（采纳 Codex 建议，但降低优先级）：检测 `view/` 目录下 `while`/`for` 中含 `parentId` 的模式。该项有价值但不阻塞，因为项目当前没有自定义 ESLint 规则基础设施

> ⚠️ **审查修订：移除 dev-mode runtime guard**
>
> 原方案建议在 `getShapeWorldPosition()` 内添加 `console.warn`。但按 Big Bang 原则，该方法将在迁移完成后直接删除，不存在需要 runtime guard 的过渡期。

### 关于 Codex "培训分享"建议的评论

Codex 建议做 30 分钟内部分享。该建议不适用于当前项目——这是一个单人/小团队项目，Steering Rule + runtime guard 已足够覆盖。

### 需要迁移的 6 个消费方

| # | 文件 | 当前实现 | 迁移目标 |
|---|------|---------|---------|
| 1 | `CanvasEditor.getShapeWorldPosition()` | 累加 bounds.x/y | 迁移所有调用方到 `TransformService`，然后直接删除 |
| 2 | `CanvasRenderer.ts` 内联 ancestor 数组 | ctx.translate + ctx.rotate | 使用 `TransformService.getFullWorldTransform()` 设置 canvas transform |
| 3 | `DOMNodeLayer.tsx` 内联累加 offset | 累加 bounds.x/y | 使用 `TransformService.localToWorld()` 或 `getWorldBounds()` |
| 4 | `SelectionBox.tsx` → `unionBounds()` | 内联累加 offset | 使用 `TransformService.getWorldBounds()` |
| 5 | `TransformHandles.tsx` → `getParentWorldOffset()` | 已修复但仍是内联实现 | 使用 `TransformService.getWorldBounds()` |
| 6 | `GroupCommands.ts` | 调用 `getShapeWorldPosition()` + `coordinateConvert.ts` | 使用 `TransformService` 矩阵方法 |

---

## 层次 3：Branded Types（渐进式引入）

### 采纳的方案

融合两份报告，采用 Gemini 的 `readonly __brand` 方案（比 Codex 的 `unique symbol` 更简洁）：

```typescript
// foundation/geometry/types.ts
export type WorldPoint = { x: number; y: number } & { readonly __brand: 'world' };
export type LocalPoint = { x: number; y: number } & { readonly __brand: 'local' };

export const asWorld = (p: { x: number; y: number }): WorldPoint => p as WorldPoint;
export const asLocal = (p: { x: number; y: number }): LocalPoint => p as LocalPoint;
```

### 关于 Codex 泛型 Point 方案的评论

Codex 提出的 `Point<Space extends 'local' | 'world'>` 泛型方案在理论上更优雅，但在本项目中不采纳：

1. 项目已有 `Vec2`（`{ x: number; y: number }`）和 `Point`（同结构）两种点类型定义，引入泛型 Point 会增加第三种，加剧类型碎片化
2. `readonly __brand` 方案与现有 `Vec2` 兼容性更好——只需在边界处 `as` 转换，不需要改变整个类型体系
3. 泛型方案在 IDE 提示中显示为 `Point<'local'>` 而非 `LocalPoint`，可读性略差

### 落地策略

1. 在 `TransformService` 的入参出参上首先引入 branded types
2. 鼠标事件入口处（`CanvasContent.clientToCanvas()`）将 `e.clientX/Y` 经 camera 转换后直接包装为 `WorldPoint`
3. 老代码通过 `asWorld` / `asLocal` 显式转换，不做大规模重写
4. 新代码强制使用 branded types
5. 长期可选：引入 `ScreenPoint` branded type 覆盖 Screen 空间，使三层坐标管线全部类型安全

### 关于 Codex "不在库边界过早强制品牌化"的评论

该建议合理但不适用——本项目不是对外发布的库，没有外部集成方。`TransformService` 是内部 API，可以直接强制。

---

## 实施计划

不设时间线。按依赖顺序执行，每步完成即推进下一步：

0. 修正 `TransformService.localToWorld()` 和 `worldToLocal()` 使用 `getFullWorldTransform` 而非 `getWorldTransform`（所有后续迁移的前提）
1. 在 `TransformService` 中新增 `getWorldBounds()`
2. 在 `CameraController` 中新增 `getVisibleWorldBounds()` 和 `getViewportMatrix()`；如采用方案 B，同步新增 `containerSize` signal 和 `setContainerSize()`
3. 逐个迁移 Local→World 的 6 个消费方到 `TransformService`，包括 `getShapeWorldPosition()` 的所有调用方
4. 删除 `CanvasEditor.getShapeWorldPosition()`
5. 逐个迁移 Canvas→Screen 的 5 个内联消费方到 `camera.canvasToScreen()`
6. 迁移 `Minimap.tsx` 可视区域计算到 `camera.getVisibleWorldBounds()`
7. 新建 `src/view/coordHelpers.ts`，实现 `localToScreen()` 和 `getScreenTransform()`
8. 将 `DOMNodeLayer.tsx` 迁移到使用 `coordHelpers.getScreenTransform()`（一次性解决双重散弹）
9. 写入 Steering Rule（同时覆盖规则 A 和规则 B）
10. 引入 Branded Types 到 `TransformService` 和 `CameraController` 边界
11. 删除 `coordinateConvert.ts`

> 注：步骤 3 和 5 可并行执行。步骤 7 依赖步骤 2（需要 `getViewportMatrix()`）。步骤 8 依赖步骤 3 + 5 + 7（三个前置条件）。步骤 0 是一切的前提，必须最先完成。

---

## 附录：完整取舍对照

### A. Codex vs Gemini 报告取舍

| 议题 | Codex 建议 | Gemini 建议 | 本方案决定 | 理由 |
|------|-----------|-------------|-----------|------|
| `getAncestorOffset` API | 提供 | 反对 | **不提供** | Group 旋转是明确的未来方向，偏移量 API 会误导 |
| 便捷 API 范围 | 4 个方法 | 3 个方法 | **1 个新增**（`getWorldBounds`），其余已存在 | 实际代码库已有 `localToWorld`/`worldToLocal` |
| Branded Types 语法 | `unique symbol` 或泛型 | `readonly __brand` | **`readonly __brand`** | 与现有 `Vec2` 兼容性更好 |
| ESLint 规则 | 高优先级 | 未提及 | **低优先级** | 项目无自定义 ESLint 基础设施 |
| Runtime guard | 强烈推荐 | 未提及 | **不适用** | Big Bang 原则下 `getShapeWorldPosition` 直接删除，无过渡期 |
| 培训分享 | 建议 30 分钟 | 未提及 | **不适用** | 单人/小团队项目 |
| 时间线 | 1-2 周 / 2-6 周 / 数月 | 今天 / 明天 / 下周 | **不设时间线** | 按依赖顺序推进，完成即下一步 |
| 缓存策略 | per-frame 或 per-invalidation | 未提及 | **per-frame，统一在 `getFullWorldTransform` 内部** | 避免多层缓存的一致性问题 |

### B. Screen ↔ Canvas 层审查发现（两份报告均未覆盖）

| 发现 | 现状 | 风险等级 | 处置 |
|------|------|---------|------|
| Canvas→Screen 5 处内联 | 数学正确，但公式散落 | 中（散弹式修改风险） | 迁移到 `camera.canvasToScreen()` |
| Screen→Canvas 输入方向 | 全部正确调用 `camera.screenToCanvas()` | 无 | 无需处置 |
| `getVisibleWorldBounds()` 缺失 | `CanvasRenderer` 私有实现 + `Minimap` 内联实现 | 低 | 提升为 `CameraController` 公共方法 |
| `getViewportMatrix()` 缺失 | `CameraController` 不暴露矩阵 | 中（阻塞组合矩阵优化） | 新增为 `CameraController` 公共方法 |
| `DOMNodeLayer.tsx` 双重散弹 | 同时内联 parent-chain-walk + canvasToScreen | 高（两个层次的问题叠加） | 迁移到 `coordHelpers.getScreenTransform()` |
| Branded Types 扩展到 Screen 空间 | 未规划 | — | 长期可选：引入 `ScreenPoint` branded type |

### C. 跨层架构决策

| 决策 | 结论 | 理由 |
|------|------|------|
| `localToScreen` 放在哪里？ | View 层 `coordHelpers.ts`，不放 `TransformService` | 避免 TransformService 反向依赖 CameraController；保持 scene graph / viewport 职责分离 |
| 是否隐藏 World 空间？ | 不隐藏 | World 是碰撞检测、snapping、spatial index 等核心逻辑的工作空间，隐藏会导致坐标混乱 |
| 组合矩阵 `getScreenTransform` | 提供，放在 `coordHelpers.ts` | 对 DOMNodeLayer 等批量渲染场景有明确性能价值 |
| Document Space vs World Space | 当前不分离 | 单画布无 artboard，World = Document。未来引入多 artboard 时再评估 |

### D. 重构架构师审查修订记录

| # | 严重度 | 问题 | 修订内容 |
|---|--------|------|---------|
| 1 | 🔴 CRITICAL | `getWorldBounds` 实现对 `(bounds.x, bounds.y)` 做 `matApply`，与 `getFullLocalTransform` 内含的 `matTranslate(bounds.x, bounds.y)` 双重计算 | 改为对局部空间四角 `(0,0)→(w,h)` 做 `matApply` |
| 2 | 🔴 MODERATE | `@deprecated` 渐进迁移策略与 Big Bang steering rule 冲突 | 改为同批次迁移所有调用方 + 直接删除，不标 deprecated |
| 3 | 🟡 MODERATE | `getVisibleWorldBounds` 使用 `window.innerWidth/Height`，但画布不占满窗口 | 改为接受容器尺寸参数或持有 containerSize signal |
| 4 | 🟡 MODERATE | 现有 `localToWorld`/`worldToLocal` 使用 `getWorldTransform`（仅旋转）而非 `getFullWorldTransform`（position+rotation），Group 子元素结果错误 | 新增步骤 0：修正实现为使用 `getFullWorldTransform` |
| 5 | 🟢 MINOR | `coordinateConvert.ts` 处置为"可降级为内部工具或删除" | 改为迁移完成后直接删除，不保留 |
| 6 | 🟢 MINOR | dev-mode runtime guard 在直接删除策略下无意义 | 移除该落地手段 |
