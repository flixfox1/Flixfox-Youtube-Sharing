# Gemini 分析报告：坐标系统统一方案

> 来源：Gemini 对 Matelink 无限画布坐标系问题的分析
> 用途：供审查 Agent 参考，与 Codex 分析报告交叉比对

---

## 核心诊断

Gemini 指出了该项目坐标系统架构腐化的核心痛点：正确的基础设施如果不好用，业务开发就会自己造轮子——而且往往是错的轮子。项目中存在 6 处手工 `while` 循环遍历 parent chain 计算坐标的重复实现，正是这一问题的直接体现。

Gemini 同样将解决方案划分为三个层次，但在具体建议上与 Codex 存在关键差异。

---

## 一、层次 1：API 缺口与数学陷阱（优先级：最高）

### 1.1 ROI 评估

Gemini 认为这是投入产出比最高的一步。开发者反复编写 `while` 循环，根本原因是解析 `Mat2x3` 对 UI 层来说心智负担过重。

### 1.2 关键警告：`getAncestorOffset` 的数学陷阱

Gemini 对 Codex 提出的 `getAncestorOffset(parentId)` API 提出了明确的安全警告：

> 在支持旋转（Rotation）和缩放（Scale）的系统中，子元素的绝对坐标绝对不是简单的"父坐标 + 子坐标"。

**具体场景**：父 Group 旋转了 90 度，子元素在其内部 `x: 100, y: 0`。在世界坐标系下，该子元素的实际物理偏移应该是向下（Y 轴），而不是向右（X 轴）。简单的 `while` 累加 `x/y` 会导致坐标彻底错乱。

### 1.3 Gemini 的修正建议

Gemini 主张不暴露"偏移量"，而是直接暴露"转换结果"。建议在 `TransformService` 中添加以下便捷 API：

```typescript
// 获取 shape 的世界坐标系包围盒
getWorldBounds(shapeId: MShapeId): Rect;

// 将 shape 局部坐标系下的点转换为世界点
localToWorld(shapeId: MShapeId, localPoint: Point): Point;

// 将世界坐标系下的点转换到 shape 的局部坐标系
worldToLocal(shapeId: MShapeId, worldPoint: Point): Point;
```

Gemini 强调：UI 层开发者只需调用 `localToWorld`，内部依旧调用已有的 `getFullWorldTransform` 矩阵乘法。这在"易用性"和"绝对正确性"之间达到了平衡。

### 1.4 与 Codex 的关键分歧

| 对比项 | Codex | Gemini |
|--------|-------|--------|
| `getAncestorOffset` | 建议提供，作为简化场景的便捷 API | **明确反对**，认为暴露偏移量在旋转/缩放场景下会误导开发者 |
| API 设计哲学 | 提供多层次 API（偏移量 + 矩阵转换） | 只暴露转换结果（`localToWorld` / `worldToLocal`），底层死守矩阵 |

---

## 二、层次 2：架构规则约束（优先级：高，随层次 1 一起落地）

### 2.1 核心观点

Gemini 认为规则写在文档里是没有用的，必须有代码层面的体现。

### 2.2 建议的规则内容

- **规则**：禁止在 View/UI 层手动遍历 `parentId` 链条计算坐标
- **解释**：手动遍历无法正确处理父级的旋转（Rotation）和缩放（Scale），会导致选中框错位、碰撞检测失败。所有跨坐标系的计算必须且只能通过 `TransformService`

### 2.3 落地建议

- 在 PR Review 时全局搜索 `.parent` 或 `while (current.parentId)`
- 一旦发现在 View 层中做 parent chain walk，直接打回要求改用 `TransformService`
- 将规则写入 steering rules 文档，并附带"为什么"的解释

---

## 三、层次 3：Branded Types（优先级：中，渐进式重构）

### 3.1 行业参考

Gemini 指出引入 Branded Types 区分 `LocalPoint` 和 `WorldPoint` 是画布引擎开发的终极最佳实践，Figma、tldraw、Miro 内部均采用此方案。

### 3.2 取舍分析

- **代价**：重构成本高，初期带来开发摩擦。开发者会抱怨"传了 `{x, y}` 为什么 TS 报错"
- **收益**：永远消灭坐标系传错导致的"幽灵 Bug"

### 3.3 渐进式落地策略

**第一步：定义类型和工厂函数**

```typescript
export type WorldPoint = { x: number; y: number } & { readonly __brand: 'world' };
export type LocalPoint = { x: number; y: number } & { readonly __brand: 'local' };

// 显式转换钩子，强制开发者意识到自己在做坐标系转换
export const asWorld = (p: Point): WorldPoint => p as WorldPoint;
export const asLocal = (p: Point): LocalPoint => p as LocalPoint;
```

**第二步：在 TransformService 中强制使用**

```typescript
localToWorld(shapeId: MShapeId, point: LocalPoint): WorldPoint;
worldToLocal(shapeId: MShapeId, point: WorldPoint): LocalPoint;
```

**第三步：老代码豁免，新代码严控**

- 原有 6 处散落代码在替换为 `TransformService` 时，强制套用 `asLocal` 或 `asWorld`
- 未来开发中，鼠标事件 `e.clientX` 直接包装为 `WorldPoint`，从根源掐断混乱

---

## 四、实施建议时间线

| 阶段 | 时间 | 内容 |
|------|------|------|
| 立即 | 今天 | 实现层次 1（补齐 API），重构 6 处手工 `while` 循环。**必须用矩阵乘法，不可用简单数值累加** |
| 短期 | 明天 | 在团队文档/架构原则中写下层次 2 的硬约束 |
| 中期 | 下周起 | 引入层次 3，将 `TransformService` 的入参出参全部替换为 Branded Types |
