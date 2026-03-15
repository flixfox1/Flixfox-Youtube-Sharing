# TASK-8: 写入坐标系统 Steering Rule

## 前置依赖

- TASK-3（Local→World 迁移完成）
- TASK-5（Canvas→Screen 迁移完成）

## 问题描述

迁移完成后，需要通过 Steering Rule 防止未来代码再次引入内联坐标转换。覆盖两条硬规则：

- **规则 A（Local ↔ World）**：禁止在 View/UI 层手动遍历 `parentId` 链条计算坐标
- **规则 B（Canvas ↔ Screen）**：禁止在 View/UI 层内联 `x * zoom + panX` 或 `(x - panX) / zoom` 公式

## 涉及文件

| 文件 | 操作 |
|------|------|
| `.kiro/steering/coordinate-system.md` | 新建（如果已有同名文件则更新） |

## 实施步骤

### 步骤 1：检查现有 steering 文件

检查 `.kiro/steering/` 目录下是否已有坐标相关的 steering 文件。如果有，在其基础上更新；如果没有，新建。

### 步骤 2：创建 steering 文件

```markdown
---
inclusion: always
---

# 坐标系统规则

## 三层坐标管线

```
Local (shape 局部) ──→ World (无限画布) ──→ Screen (屏幕像素)
    TransformService       CameraController
```

组合转换（Local→Screen）只在 View 层的 `src/view/coordHelpers.ts` 中完成。

## 规则 A：Local ↔ World

所有 Local ↔ World 坐标计算必须且只能通过 `TransformService`。

禁止：
- 在 View/UI 层手动遍历 `parentId` 链条累加 `bounds.x/y`
- 在 View/UI 层手写 parent-chain-walk 计算世界坐标
- 绕过 `TransformService` 直接构造坐标转换矩阵

正确做法：
- `TransformService.getWorldBounds(shapeId)` — 获取世界 AABB
- `TransformService.localToWorld(point, shapeId)` — 点转换
- `TransformService.worldToLocal(point, shapeId)` — 反向点转换
- `TransformService.getFullWorldTransform(shapeId)` — 获取完整矩阵

## 规则 B：Canvas ↔ Screen

所有 Canvas(World) ↔ Screen 坐标计算必须且只能通过 `CameraController`。

禁止：
- 内联 `x * zoom + panX`（Canvas→Screen）
- 内联 `(x - panX) / zoom`（Screen→Canvas）
- 内联 `-panX / zoom` 计算可视区域

正确做法：
- `camera.canvasToScreen(cx, cy)` — Canvas→Screen
- `camera.screenToCanvas(sx, sy)` — Screen→Canvas
- `camera.getVisibleWorldBounds()` — 可视区域
- `camera.getViewportMatrix()` — 获取 viewport 矩阵

## 跨层组合

需要 Local→Screen 转换时，使用 `src/view/coordHelpers.ts`：
- `localToScreen(transform, camera, shapeId, point)` — 单点转换
- `getScreenTransform(transform, camera, shapeId)` — 获取组合矩阵（批量场景）

不得在 `TransformService` 中引入 Screen 相关方法（避免反向依赖 CameraController）。
```

## 验收标准

1. `.kiro/steering/` 目录下存在坐标系统相关的 steering 文件
2. 文件包含规则 A（Local↔World 通过 TransformService）
3. 文件包含规则 B（Canvas↔Screen 通过 CameraController）
4. 文件包含跨层组合规则（coordHelpers.ts）
5. front-matter 设置为 `inclusion: always`
