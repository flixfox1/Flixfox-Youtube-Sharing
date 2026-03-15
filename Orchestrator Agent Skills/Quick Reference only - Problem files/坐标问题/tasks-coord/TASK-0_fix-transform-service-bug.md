# TASK-0: 修正 TransformService.localToWorld / worldToLocal 实现 bug

> 严重度：🔴 CRITICAL — 所有后续迁移的前提

## 前置依赖

无

## 问题描述

`TransformService.localToWorld()` 和 `worldToLocal()` 内部调用的是 `getWorldTransform()`，该方法只组合旋转矩阵（`getLocalTransform` 不含 position offset），不含 `bounds.x/y` 的平移。对于 Group 子元素，`getWorldTransform` 会丢失所有祖先的 position offset，导致结果完全错误。

必须改为使用 `getFullWorldTransform()`（组合 `getFullLocalTransform`，包含 position + rotation）。

## 涉及文件

| 文件 | 操作 |
|------|------|
| `src/core/editor/TransformService.ts` | 修改 `localToWorld` 和 `worldToLocal` 方法 |

## 实施步骤

### 步骤 1：修改 `localToWorld`

将：
```typescript
localToWorld(point: Vec2, shapeId: MShapeId): Vec2 {
  return matApply(this.getWorldTransform(shapeId), point);
}
```

改为：
```typescript
localToWorld(point: Vec2, shapeId: MShapeId): Vec2 {
  return matApply(this.getFullWorldTransform(shapeId), point);
}
```

### 步骤 2：修改 `worldToLocal`

将：
```typescript
worldToLocal(point: Vec2, shapeId: MShapeId): Vec2 {
  const inv = matInvert(this.getWorldTransform(shapeId));
  return inv ? matApply(inv, point) : point;
}
```

改为：
```typescript
worldToLocal(point: Vec2, shapeId: MShapeId): Vec2 {
  const inv = matInvert(this.getFullWorldTransform(shapeId));
  return inv ? matApply(inv, point) : point;
}
```

## 验收标准

1. `localToWorld` 内部调用 `getFullWorldTransform` 而非 `getWorldTransform`
2. `worldToLocal` 内部调用 `getFullWorldTransform` 而非 `getWorldTransform`
3. `getDiagnostics` 零错误
4. 现有 `getWorldTransform` 和 `getLocalTransform` 方法保持不变（它们仍有独立用途，如 hit-testing 中的纯旋转变换）
