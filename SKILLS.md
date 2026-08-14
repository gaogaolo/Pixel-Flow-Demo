# Pixel Flow 关卡编辑器 — Skills 操作指南

> 适用于：向 AI 模型描述如何使用此编辑器新增/修改功能时的参考手册。

---

## Skill 1：新增关卡配置字段

**场景：** 需要在 JSON 关卡数据中添加一个新的配置项（如 `SpeedMultiplier`）。

**步骤：**
1. 在 `generateLevel()` 函数中的 `levelData = { ... }` 对象里添加新字段（首字母大写）
2. 在 `applyImportedLevel(data)` 函数中同步读取新字段，兼容大小写：
   ```js
   NewField: data.NewField || data.newField || defaultValue
   ```
3. 在 `initGameState()` 中若游戏状态需要用到此字段，将其映射到 `gameState`
4. 在 `renderLauncherEditor()` 或相关 UI 中添加配置界面（若需要）

**注意：** 字段名统一使用 PascalCase（首字母大写），`_id` 是唯一例外。

---

## Skill 2：修改发射器队列行为

**场景：** 修改发射器点击后的入轨逻辑。

**关键函数：** `handleLauncherClick(columnIndex)`

**逻辑流程：**
```
1. 获取顶部 item（column.items[column.items.length - 1]）
2. 查询连接组（getLinkGroupFromGameState(topItem._id)）
3. if (有连接组) → 走联动发射分支
   - 找到所有伙伴的位置（同列/跨列判断不同规则）
   - 同时 pop 所有连接的 item，push 到 conveyorItems
   - 调用 tickFrozenLaunchers(groupSize)
4. else → 普通发射
   - pop 顶部 item，push 到 conveyorItems
   - 调用 tickFrozenLaunchers(1)
5. 调用 startMainLoop()、renderLaunchers()、updateGameUILight()
```

**关键数据结构（conveyorItem）：**
```js
{
  id: number,          // 唯一 ID
  color: string,       // 颜色
  ammo: number,        // 剩余弹药
  angle: number,       // 当前轨道角度 t ∈ [0, +∞)
  groupId: string,     // 连接组 ID（无连接时不存在）
  _id: number,         // 对应 Item._id
  hasMatched: bool,    // 是否消除过像素
  totalRotations: number, // 已完成的圈数
  lockedPixels: Set,   // 已锁定（发射中）的像素坐标
  _ammoZero: bool,     // 弹药归零标记（连接组使用）
  _partnerAmmoZero: bool // 伙伴弹药归零标记
}
```

---

## Skill 3：新增发射器特殊类型

**参考案例：** 冷冻发射器（`FrozenLaunchers`）

**实现模式：**

1. **数据结构**：在 `levelData` 中添加数组字段，每个元素包含 `ItemId`（对应 `_id`）和配置参数
2. **配置端**（`renderLauncherEditor`）：
   - 添加模式切换按钮（与其他模式互斥，通过全局 bool 变量控制）
   - 在发射器卡片渲染中加入特殊状态显示（颜色/图标）
   - 添加 `handleXxxClick(colIdx, idxFromTop, itemId)` 处理点击
   - 添加配置弹窗（modal）和 `confirmXxxItem()` 确认函数
3. **游戏状态初始化**（`initGameState`）：
   - 将配置转换为运行时状态对象（如 `frozenState: { [itemId]: remaining }`）
4. **游戏渲染**（`renderLaunchers`）：
   - 在顶部 item 渲染分支中添加特殊类型判断（先验证 `levelData` 中有此配置，再查 `gameState`）
   - 在非顶部 item 渲染分支中也添加特殊显示（避免只有顶部才显示的 bug）
5. **游戏逻辑触发**：在 `handleLauncherClick` 的普通发射和连接发射两个分支末尾都调用触发函数

**互斥规则模板：**
```js
function toggleXxxMode() {
    xxxModeActive = !xxxModeActive;
    if (xxxModeActive) {
        linkModeActive = false;  // 关闭其他互斥模式
        freezeModeActive = false;
        linkPendingItem = null;
    }
    renderLauncherEditor();
}
```

---

## Skill 4：修改连接规则

**关键函数：** `handleLinkClick(colIdx, idxFromTop, itemId)`

**校验点（按顺序）：**
1. 已连接 → 解除连接（找到 link，过滤掉后重渲）
2. 第一次点击 → 记录 `linkPendingItem`
3. 第二次点击：
   - 同一个 → 取消选中
   - **同列时**：`Math.abs(idxFromTop 差) !== 1` → 拒绝（必须相邻）
   - 任一方已连接 → 拒绝
   - 通过 → `levelData.LauncherLinks.push({ IdA, IdB })`

**游戏端发射规则修改（`handleLauncherClick` 中的同列判断）：**
```js
if (isSameCol) {
    // 伙伴必须在 items.length - 2（紧邻顶部的第二排）
    if (pidxInCol === col2.items.length - 2) {
        // 可以联动
    } else {
        partnerCols.length = -1; // 连接失效，走普通发射
    }
} else {
    // 跨列：伙伴必须在顶部（items.length - 1）
    if (!partnerIsTop) return; // 不可发射
}
```

---

## Skill 5：修改难度评分算法

**函数：** `calcArrangementDifficulty()`

**当前评分维度：**
- 颜色冲突（同列相邻不同颜色）：每个冲突 +8 分
- 连续相同颜色：每个连续 -3 分
- 暂存槽张力（发射器总数 - 暂存槽容量×2）：每个超出 +5 分
- 弹药标准差：每点 +2 分

**新增评分维度示例：**
```js
// 在 calcArrangementDifficulty() 末尾 diffScore = Math.max(0, Math.min(100, ...)) 之前添加：
const frozenCount = (levelData.FrozenLaunchers || []).length;
diffScore += frozenCount * 10; // 每个冷冻发射器 +10 难度
```

---

## Skill 6：导入 JSON 关卡

**函数：** `applyImportedLevel(data)`

**兼容性处理模式：**
```js
// 字段读取：同时兼容大写和小写
const gs = data.GridSize || data.gridSize;

// 数组字段：缺失时默认空数组
LauncherLinks: data.LauncherLinks || data.launcherLinks || []

// 初始化后必须调用：
ensureItemIds();          // 为缺少 _id 的 Item 补充 ID
cleanupInvalidLinks();    // 清理无效连接
cleanupInvalidFrozenLaunchers(); // 清理无效冷冻配置
```

**导入后必须执行的 UI 更新：**
```js
refreshGridDisplay();   // 刷新像素网格
displayLevelData();     // 更新 JSON 数据面板
renderLauncherEditor(); // 刷新发射器编排
switchTab('game');      // 切换到游戏预览
previewGame();          // 启动预览
```

---

## Skill 7：添加新的游戏事件触发

**场景：** 需要在某个游戏事件发生时触发额外逻辑（如每次像素消除时触发）。

**关键位置：** `startMainLoop()` 内的 `gameState.conveyorItems.forEach(ci => { ... })` 循环

**像素消除触发点：**
```js
// 在 ci.ammo-- 之后添加自定义逻辑：
pixel.active = false;
ci.ammo--;
// ↓ 在此添加消除触发逻辑
onPixelEliminated(ci, target.x, target.y);
```

**弹药归零触发点：**
```js
// 在 toRemoveSet.add(ci.id) 之前
if (ci.ammo <= 0) {
    // ↓ 在此添加弹药归零触发逻辑
    onAmmoEmpty(ci);
    toRemoveSet.add(ci.id);
}
```

---

## Skill 8：修改轨道形状

**当前：** 正方形轨迹，由 `squarePathPos(t, size, centerX, centerY)` 实现。

**若改为圆形轨迹：**
```js
function circlePathPos(t, radius, centerX, centerY) {
    const angle = (t / 4) * Math.PI * 2; // t ∈ [0,4) → 角度 [0, 2π)
    return {
        x: centerX + radius * Math.cos(angle),
        y: centerY + radius * Math.sin(angle)
    };
}
```

**注意：** 修改轨迹形状后需同步修改 `findShootableTarget()` 中的射击方向判断逻辑（该函数依赖正方形四边的方向映射）。

---

## Skill 9：PixelMap 格式转换

**一维数组（新格式）→ 二维颜色数组：**
```js
// 使用内置函数：
const pixelMap2D = flatPixelMapTo2D(levelData.PixelMap, levelData.GridSize, levelData.ColorList);
// 结果：pixelMap2D[y][x] = 颜色字符串（如 "#feca57"）或 null（空格）
```

**生成 PixelMap（编辑器内部）：**
```js
// 构建 ColorList 和 PixelMap
const colorToIndex = {};
const colorListObj = {};
let colorIdx = 1;
// 遍历 pixelData[y][x]，为每种颜色分配序号
// PixelMap[y*GridSize + x] = colorToIndex[color] || 0
```

---

## Skill 10：向发射器编排页添加新控件

**步骤：**
1. 在 `renderLauncherEditor()` 的 HTML 模板字符串中添加按钮/选择器
2. 使用 `onclick="yourFunction(${c}, ${idxFromTop}, ${itemId})"` 传递参数（注意：此处使用模板字符串拼接，参数直接嵌入）
3. 在全局作用域定义 `yourFunction(colIdx, idxFromTop, itemId)` 函数
4. 函数内操作 `levelData.Launchers` 数据后调用 `renderLauncherEditor()` 刷新

**获取 col.Items 中的原始索引：**
```js
// idxFromTop: 0=顶部（先发射），colLen-1=底部（最后发射）
// originalIdx: col.Items 中的实际下标（末端=顶部）
const originalIdx = col.Items.length - 1 - idxFromTop;
const item = col.Items[originalIdx];
```

---

## 常见陷阱

| 陷阱 | 说明 | 解决方案 |
|------|------|----------|
| 渲染发射器时误触冷冻逻辑 | `frozenState[id]` 用数字 key 存，Object.keys() 返回字符串 | 先查 `levelData.FrozenLaunchers` 确认是否真的是冷冻发射器 |
| 连接发射漏调 tickFrozenLaunchers | 普通/连接两个分支都需要调用 | 两个分支末尾各自调用，传不同 count |
| 同列连接判断 | 同列伙伴不是必须在顶部，而是在 `items.length - 2`（紧邻） | 区分 `isSameCol` 和跨列两种规则 |
| items 末端 vs 顶部 | `col.Items[末端]` = 顶部（先发射），`col.Items[0]` = 底部（最后发射） | idxFromTop=0 对应 originalIdx=length-1 |
| 游戏预览每帧重建按钮 | 在 `updateGameUILight` 中调用 `renderLaunchers` 会导致点击事件丢失 | 只在发射/解冻等关键事件后才调用 `renderLaunchers`，动画每帧只调用 `updateAllConveyorItems` |
| 字段名大小写 | 所有 JSON 字段首字母大写，`_id` 例外 | 新增字段统一用 PascalCase，导入时做双向兼容 |
