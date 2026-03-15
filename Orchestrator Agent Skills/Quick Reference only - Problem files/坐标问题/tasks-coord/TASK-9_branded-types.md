# TASK-9: 引入 Branded Types 到坐标系统边界

## 前置依赖

- TASK-0（TransformService 修正）
- TASK-1（getWorldBounds 可用）
- TASK-2（CameraController API 补齐）
- TASK-7（coordHelpers.ts 已创建，本任务需修改其类型签名）

> 注：建议在 TASK-3~8 全部完成后执行，避免迁移过程中频繁处理类型转换。

## 问题描述

当前坐标系统中所有点都是 `{ x: number; y: number }`（`Vec2` 或 `Point`），编译器无法区分 Local、World、Screen 空间的点。引入 Branded Types 使类型系统能在编译期捕获坐标空间混用错误。

## 涉及文件

| 文件 | 操作 |
|------|------|
| `src/foundation/geometry/coordTypes.ts` | 新建：Branded Types 定义 |
| `src/core/editor/TransformService.ts` | 修改入参出参类型 |
| `src/view/camera/CameraController.ts` | 修改入参出参类型 |
| `src/view/coordHelpers.ts` | 修改入参出参类型 |

## 实施步骤

### 步骤 1：创建 Branded Types 定义文件

新建 `src/foundation/geometry/coordTypes.ts`：

```typescript
/**
 * Branded coordinate types — compile-time safety for coordinate spaces.
 *
 * @module foundation/geometry/coordTypes
 */

/** A point in shape-local coordinate space */
export type LocalPoint = { x: number; y: number } & { readonly __brand: 'local' };

/** A point in world (canvas) coordinate space */
export type WorldPoint = { x: number; y: number } & { readonly __brand: 'world' };

/** A point in screen (pixel) coordinate space */
export type ScreenPoint = { x: number; y: number } & { readonly __brand: 'screen' };

/** Cast a plain point to LocalPoint */
export const asLocal = (p: { x: number; y: number }): LocalPoint => p as LocalPoint;

/** Cast a plain point to WorldPoint */
export const asWorld = (p: { x: number; y: number }): WorldPoint => p as WorldPoint;

/** Cast a plain point to ScreenPoint */
export const asScreen = (p: { x: number; y: number }): ScreenPoint => p as ScreenPoint;
```

### 步骤 2：在 TransformService 边界引入

修改 `TransformService` 的方法签名：

```typescript
localToWorld(point: LocalPoint, shapeId: MShapeId): WorldPoint;
worldToLocal(point: WorldPoint, shapeId: MShapeId): LocalPoint;
```

内部实现中，`matApply` 返回的 `Vec2` 需要用 `as WorldPoint` / `as LocalPoint` 转换。

`getWorldBounds` 的返回类型保持 `Rect`（Rect 不是点，不需要 brand）。

### 步骤 3：在 CameraController 边界引入

修改 `CameraController` 的方法签名：

```typescript
screenToCanvas(sx: number, sy: number): WorldPoint;
canvasToScreen(cx: number, cy: number): ScreenPoint;
```

### 步骤 4：在 coordHelpers 边界引入

```typescript
export function localToScreen(
  transform: TransformService,
  camera: CameraController,
  shapeId: MShapeId,
  local: LocalPoint,
): ScreenPoint;
```

### 步骤 5：修复编译错误

引入 branded types 后，现有调用方传入的 `{ x, y }` 普通对象会产生类型错误。在每个调用点使用 `asLocal()` / `asWorld()` 显式转换：

```typescript
// 旧：
const world = transformService.localToWorld({ x: 0, y: 0 }, shapeId);

// 新：
const world = transformService.localToWorld(asLocal({ x: 0, y: 0 }), shapeId);
```

**策略**：只在 TransformService / CameraController / coordHelpers 的边界强制 branded types。内部计算和 Foundation 层的纯数学函数（`matApply` 等）保持 `Vec2`，不强制 brand。

## ⚠️ 关键注意事项

1. 不要在 `Mat.ts` 的 `matApply` 等纯数学函数上引入 branded types — 它们是空间无关的
2. `Rect` 类型不需要 brand — 它表示区域而非点
3. 老代码通过 `asWorld` / `asLocal` 显式转换，不做大规模重写
4. 如果编译错误过多，可以分批处理：先只改 TransformService，验证通过后再改 CameraController

## 验收标准

1. `src/foundation/geometry/coordTypes.ts` 文件存在，包含 `LocalPoint`、`WorldPoint`、`ScreenPoint` 类型和 `asLocal`、`asWorld`、`asScreen` 转换函数
2. `TransformService.localToWorld` 入参为 `LocalPoint`，出参为 `WorldPoint`
3. `TransformService.worldToLocal` 入参为 `WorldPoint`，出参为 `LocalPoint`
4. `CameraController.canvasToScreen` 出参为 `ScreenPoint`
5. `CameraController.screenToCanvas` 出参为 `WorldPoint`
6. `getDiagnostics` 零错误
7. `Mat.ts` 的函数签名不变（保持 `Vec2`）
