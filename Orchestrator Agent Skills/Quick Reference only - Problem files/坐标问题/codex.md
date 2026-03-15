# Codex 分析报告：坐标系统统一方案

> 来源：Codex (OpenAI) 对 Matelink 无限画布坐标系问题的分析
> 用途：供审查 Agent 参考，与 Gemini 分析报告交叉比对

---

## 总体结论

Codex 建议按优先级依次推进三个层次：层次 1（API 缺口）→ 层次 2（架构规则）→ 层次 3（类型安全）。该策略以最小成本快速消除重复实现，同时为更严格的类型安全做铺垫。

---

## 一、优先级与取舍总览

| 层次 | 优先级 | 评估 |
|------|--------|------|
| 层次 1：API 缺口 | 最高 | 低成本、高收益。增加便捷 API 可立即消除大部分重复代码，减少 bug 面，且不破坏现有代码 |
| 层次 2：架构规则 | 中等 | 在 API 就绪后强制执行（lint/CI/runtime guard/代码审查），防止旧习惯回流 |
| 层次 3：Branded Types | 可选/低（长期） | 最"干净"的方案，但改动范围大，会触及大量调用方类型签名，适合作为后续强化项 |

---

## 二、层次 1：API 设计与实现（立即执行）

### 2.1 目标

将常见坐标转换需求包装为语义清晰、轻量的函数，避免调用者接触矩阵细节。

### 2.2 建议 API

```typescript
// 返回 ancestor 的累计偏移（不包括自身），用于 local → world 的简化场景
getAncestorOffset(parentId: MShapeId | null): { x: number; y: number };

// 返回 shape 在世界坐标系下的 bounding rect（包含 ancestor 偏移 & 自身 transform）
getWorldBounds(shapeId: MShapeId): { x: number; y: number; width: number; height: number };

// 将 localPoint 转为 worldPoint
localToWorld(shapeId: MShapeId, local: Point): Point;

// 将 worldPoint 转为 localPoint
worldToLocal(shapeId: MShapeId, world: Point): Point;
```

### 2.3 实现要点

Codex 建议利用已存在的 `getFullWorldTransform()` 作为"真相来源"：

```typescript
function getAncestorOffset(parentId: MShapeId | null): Point {
  if (!parentId) return { x: 0, y: 0 };
  const mat = transformService.getFullWorldTransform(parentId);
  return mat.applyToPoint({ x: 0, y: 0 });
}

function getWorldBounds(shapeId: MShapeId): Rect {
  const shape = store.getShape(shapeId);
  const localBounds = shape.localBounds;
  const mat = transformService.getFullWorldTransform(shapeId);
  const pts = [
    mat.applyToPoint({ x: localBounds.x, y: localBounds.y }),
    mat.applyToPoint({ x: localBounds.x + localBounds.width, y: localBounds.y }),
    mat.applyToPoint({ x: localBounds.x, y: localBounds.y + localBounds.height }),
    mat.applyToPoint({ x: localBounds.x + localBounds.width, y: localBounds.y + localBounds.height }),
  ];
  return boundingRectFromPoints(pts);
}
```

### 2.4 性能与缓存注意事项

- `getFullWorldTransform()` 可能较重，需加缓存（per-frame 或 per-invalidation），缓存 key 可用 `shape.version` 或 `transformStamp`
- 提供 dev/profiler hook，在 dev 模式记录耗时热点
- 对于只需 translation 的场景，`getAncestorOffset` 应尽量避免完整矩阵乘法，可直接取 translation 分量（需保证一致性）

### 2.5 回退与兼容策略

- 新 API 设计为非破坏性：先新增方法，不改现有 `getFullWorldTransform()`
- 在文档中提供"推荐使用场景"表格，方便 View 开发者选择合适的 API

---

## 三、层次 2：禁止内联 parent chain walk

### 3.1 目标

阻止未来再次直接遍历 parent chain 进行坐标计算。

### 3.2 组合策略（按优先级）

1. **Code-style / Steering Rule 文档**：明确写入"所有 local → world 坐标转换必须走 TransformService"，纳入 contributor handbook、PR 模板、代码评审 checklist
2. **静态检测（推荐）**：编写 ESLint 规则或 AST 检查脚本，检测 `view/` 目录下出现 `while (node.parent)` / `for(parent=...)` 等模式，标记为错误或警告
3. **Dev-mode runtime guard（强烈推荐）**：在 debug 构建中将 `parentChainWalk()` 标记为 deprecated，首次调用时抛出或 `console.error` 并打印栈
4. **CI/PR Checks**：CI 运行 AST scan，违规则 fail
5. **代码审查 + 培训**：30 分钟内部分享，讲解统一原因、新 API 用法、典型反模式对比

### 3.3 ESLint 思路（伪代码）

检测 `while` / `for` 中含 `getParent` 或 `parentId = parent.parentId` 且出现在 `view/` 路径下 → 报警并建议替换为 `transformService.getAncestorOffset(parentId)`。

---

## 四、层次 3：Branded Types（类型安全）

### 4.1 优缺点

- **优点**：从编译阶段捕获坐标系混用错误，对大型代码库有长期收益
- **缺点**：需修改大量签名，可能触发连锁改动

### 4.2 实现方式

```typescript
// 方式一：unique symbol brand
type LocalPoint = { x: number; y: number; __brand_local?: unique symbol };
type WorldPoint = { x: number; y: number; __brand_world?: unique symbol };

// 方式二：泛型 Point（更干净）
type Point<Space extends 'local' | 'world'> = { x: number; y: number } & { __space?: Space };
type LocalPoint = Point<'local'>;
type WorldPoint = Point<'world'>;
```

### 4.3 迁移策略（渐进）

1. 在新代码、新模块或边界处先引入 branded types，不改老 API
2. 增量重写渲染路径中的关键函数，从 `getAncestorOffset` 开始标记返回类型为 `WorldPoint`
3. 使用 `// TODO` 注释与 codemods 自动替换调用 site
4. 在大版本发布或边界清理时将类型改为强制（breaking change）
5. 不在库边界（外部 API）过早强制品牌化，避免对集成方造成兼容负担

---

## 五、落地计划

| 阶段 | 时间 | 内容 |
|------|------|------|
| 短期 | 1-2 周 | 添加 `getAncestorOffset` 和 `getWorldBounds`；在 4 个 View 消费方中手动替换；dev 模式记录直接 parent-walk |
| 中期 | 2-6 周 | 实施 ESLint/AST 检查与 CI gate；在更多模块中统一替换并验证性能 |
| 长期 | 数月 | 引入 branded types 到新模块；定期回顾删除重复 util、合并冗余代码 |

---

## 六、边界条件与验证要点

- **旋转和 scale**：任意对角缩放或 shear 会导致世界 bounds 不是简单平移/宽高加法，必须用矩阵变换四角点再包围盒化
- **锚点 / pivot**：确保 localBounds 定义和变换顺序（translate → rotate → scale）一致
- **Scrollable parent / viewport offset / CSS transforms**：区分 DOM 层面的 visual offset 和模型层的 transform，TransformService 应只处理模型坐标
- **性能监控**：添加单元测试（数学正确性）+ 基准测试（大量节点下的延迟），确保缓存/invalidations 正确
