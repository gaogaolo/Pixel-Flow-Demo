# Pixel Flow 关卡编辑器 — Memory 文档

> 文件路径：`游戏玩法demo/flow/index.html`
> 单文件 HTML + CSS + JavaScript，无外部依赖。

---

## 一、项目概览

这是一个像素消除游戏（Pixel Flow）的**关卡编辑器 + 游戏预览器**，集成在单个 HTML 文件中。编辑器支持：
- 手绘或导入像素图案
- 自动逆向生成发射器队列配置
- 手动编排发射器顺序、连接关系、冷冻配置
- 游戏逻辑预览（支持点击发射、轨道旋转、消除匹配）
- 导出/导入 JSON 关卡数据

---

## 二、JSON 数据结构（关卡格式）

所有字段名首字母大写（PascalCase）：

```json
{
  "Id": 1,
  "GridSize": 10,
  "LauncherCount": 3,
  "StorageSize": 5,
  "ColorList": { "1": "#feca57", "2": "#ff6b6b" },
  "PixelMap": [0,1,2,1,0,...],
  "Launchers": [
    {
      "Column": 0,
      "Items": [
        { "Color": "#feca57", "Ammo": 20, "Hidden": false, "_id": 1 }
      ]
    }
  ],
  "LauncherLinks": [
    { "IdA": 1, "IdB": 2 }
  ],
  "FrozenLaunchers": [
    { "ItemId": 3, "RequiredCount": 4 }
  ]
}
```

### 字段说明
| 字段 | 类型 | 说明 |
|------|------|------|
| Id | number | 关卡配置优先级用的正整数 ID；编辑器预览不使用该字段 |
| GridSize | number | 像素地图边长（方形），3~40 |
| LauncherCount | number | 发射列数量，3~7 |
| StorageSize | number | 暂存槽数量（同时在轨道的上限），3~8 |
| ColorList | object | 颜色索引映射，key 为数字字符串 |
| PixelMap | number[] | 一维数组，长度=GridSize²，0=空，其他=ColorList 的 key |
| Launchers | array | 每列的发射器队列，`Items[末端]` = 顶部（先发射） |
| Items._id | number | 每个 Item 的全局唯一 ID（编辑器内部使用） |
| Items.Hidden | bool | true=不可见发射器（到第一排才显示颜色/弹药） |
| LauncherLinks | array | 两两连接配置，`{IdA, IdB}` |
| FrozenLaunchers | array | 冷冻发射器配置，`{ItemId, RequiredCount}` |

---

## 三、核心功能模块

### 3.1 像素编辑器（editorTab）
- 网格尺寸：3×3 ~ 40×40，支持实时调整
- 操作：点击/拖拽填色、橡皮擦、整体上下左右位移
- 调色板：从 SVG/PNG 自动提取颜色，支持手动添加自定义色
- 颜色合并：根据"颜色合并阈值"用欧式距离聚类合并相近颜色
- 填充/删除：`fillEmptyCells()`用选中色填满空白，`deleteSelectedColor()`删除指定色

### 3.2 SVG 导入
- 支持粘贴 SVG 代码或上传 `.svg` 文件
- 使用 Canvas 栅格化（cellPx=32，每格 32×32 像素精度）提取像素布局
- 颜色吸附：将 Canvas 采样色吸附到 SVG 原始颜色列表中最近的颜色

### 3.3 PNG 导入
- 上传 PNG/GIF/BMP/WebP，支持精确模式（图片尺寸=网格尺寸）和缩放模式
- 精确模式：1图片像素=1格子，零误差
- 缩放模式：按比例采样，每格 32×32 Canvas 像素投票取多数色

### 3.4 关卡生成（generateLevel）
1. 扫描像素数据，按层级（外层 layer 大，内层 layer 小）分组
2. 逆向累计每种颜色的弹药需求（像素数量 = 需要消除的弹药总数）
3. 根据弹药权重配置（格式：`弹药量:权重`）随机生成发射器（精确余量不多不少）
4. 均衡分配到各列，按难度排序（难度1=外层先发，难度5=内层先发）
5. 根据可见/不可见权重随机标记非顶部发射器的 `Hidden` 状态
6. 生成后调用 `ensureItemIds()` 为所有 Item 分配唯一 `_id`

### 3.5 发射器编排（launchersTab）
- 显示每列发射器的全局发射优先级（全局顺序序号）
- **上移/下移**（`moveLauncherItemByDisplay`）：调整发射顺序
- **跨列转移**（`transferLauncherItemByDisplay`）：将 Item 移到其他列，保持既有连接关系
- **可见/不可见切换**（`toggleLauncherItemHidden`）：顶部 Item 不可设为不可见
- **难度评分**（`calcArrangementDifficulty`）：0~100 分，基于颜色冲突、连续相同色块、暂存槽张力、弹药不均衡
- **🔗 连接模式**：两两连接发射器（详见 3.6）
- **❄️ 冷冻模式**：配置冷冻发射器（详见 3.7）

### 3.6 两两连接（LauncherLinks）
**配置规则：**
- 连接模式与冷冻模式互斥
- 同列连接时只能连接相邻发射器（idxFromTop 差=1）；跨列连接可连接任意两列
- 同一发射器只能参与一组连接（不支持三连或多连）

**游戏逻辑：**
- **跨列连接**：两个都在第一排才可点击，同时进入轨道（占 2 个槽位），角度偏移 0.15
- **同列连接**：只要顶部那个在第一排，且伙伴在第二排（紧邻），即可点击同时发射
- 两个发射器的弹药分别各自归零后才一起消失，若只有一个归零则继续转圈直到另一个也归零
- 弹药都归零 → 直接消除（不进暂存槽）；转圈无目标 → 两个一起进暂存槽
- 连接组通过 `groupId` 字段标识

**显示：**
- 金色边框 + 🔗 徽章
- 跨列：伙伴不在第一排时显示灰色禁用态
- 编排页面：已连接的卡片显示 🔗 图标

### 3.7 冷冻发射器（FrozenLaunchers）
**配置规则：**
- 不可选隐藏发射器（Hidden=true）
- 不可选已两两连接的发射器；已冷冻的也不可建立连接
- 解冻次数上限 = `总发射器数 - 同列后排数 - 1(自身) - 已冷冻发射器数`
- 上限 ≤ 0 时禁止配置冷冻（弹出 alert 提示原因）

**游戏逻辑：**
- 冷冻发射器（不论哪一排）显示为冰蓝渐变 + ❄️ + 右下角剩余解冻次数
- 处于第一排时不可点击（cursor: not-allowed）
- 每次任意发射器点击进入轨道，所有冷冻发射器的 `remaining` 各自 `-count`（普通发射 -1，2连发射 -2）
- remaining 归零 → 立即解冻，显示真实颜色和弹药，变为可点击普通发射器
- 由 `tickFrozenLaunchers(count)` 函数执行，普通发射调用 `tickFrozenLaunchers(1)`，连接发射调用 `tickFrozenLaunchers(groupSize)`

---

## 四、游戏核心逻辑

### 4.1 轨道系统（正方形轨迹）
- 轨道为 280×280px 的正方形，发射器沿轨迹旋转
- 位置由 `squarePathPos(t, size, centerX, centerY)` 计算，t ∈ [0,4) 对应四条边
- 逆时针方向：t=0→下边，t=1→右边，t=2→上边，t=3→左边
- 每帧 angle += 0.020（可调节速度）

### 4.2 发射逻辑（findShootableTarget）
- 发射器对准方向：下边→上射，右边→左射，上边→下射，左边→右射
- 精确对准判断：posRatio 在格子中心 ±0.4/GridSize 范围内才触发发射
- 用 `cellKey`（边+格子索引）防止同一对准窗口重复发射
- 遇到同色像素 → 消除，遇到异色像素 → 阻挡不发射

### 4.3 暂存槽逻辑
- 轨道上转圈无法消除所有目标（满一圈后无剩余同色像素）→ 进入暂存槽
- 暂存槽已满 → 游戏结束（Game Over）
- 点击暂存槽中的发射器 → 重新进入轨道（需有空槽位）
- 连接组的两个发射器同时进入/离开暂存槽（联动）

### 4.4 不可见发射器（Hidden）
- `Hidden=true` 的非顶部发射器在游戏中显示为灰色遮罩 + "?"
- 进入第一排后自动显示真实颜色和弹药数

### 4.5 胜利/失败条件
- **胜利**：地图上所有像素全部消除
- **失败**：暂存槽满且无法释放（`storage.length >= storageSize` 时再次需要存入）

---

## 五、数据流向

```
SVG/PNG 输入
    ↓
像素编辑器（pixelData[][]）
    ↓
generateLevel() → levelData（JSON）
    ↓
发射器编排（LauncherLinks / FrozenLaunchers / Hidden 配置）
    ↓
applyLauncherEdits() → 更新 levelData
    ↓
previewGame() → initGameState() → gameState
    ↓
游戏循环（startMainLoop / requestAnimationFrame）
```

---

## 六、关键函数速查

| 函数 | 作用 |
|------|------|
| `generateLevel()` | 从像素数据生成完整关卡 |
| `initGameState()` | 从 levelData 初始化游戏状态（含 frozenState） |
| `renderLaunchers(container)` | 渲染游戏预览中的发射器区域 |
| `handleLauncherClick(colIdx)` | 处理点击发射，含连接/冷冻逻辑 |
| `tickFrozenLaunchers(count)` | 每次发射后触发冷冻倒计时 |
| `getLinkGroupFromGameState(itemId)` | 获取某发射器的连接组信息 |
| `buildGlobalOrder()` | 计算全局发射优先级顺序 |
| `calcArrangementDifficulty()` | 计算当前排列的难度分 |
| `ensureItemIds()` | 为所有 Item 分配唯一 _id |
| `applyImportedLevel(data)` | 导入 JSON 关卡数据，兼容新旧字段名 |
| `flatPixelMapTo2D(pixelMap,gs,colorList)` | 一维 PixelMap 转二维颜色数组 |
| `squarePathPos(t,size,cx,cy)` | 计算正方形轨迹上的坐标 |
| `findShootableTarget(color,t,...)` | 从轨道位置找可射击同色像素 |

---

## 七、全局状态变量

| 变量 | 说明 |
|------|------|
| `levelData` | 当前关卡配置（JSON 对象） |
| `gameState` | 游戏运行时状态（含 pixelMap、launchers、storage、conveyorItems、frozenState） |
| `pixelData[][]` | 像素编辑器网格数据 |
| `linkModeActive` | 连接模式是否激活 |
| `linkPendingItem` | 待连接的第一个发射器 |
| `freezeModeActive` | 冷冻模式是否激活 |
| `_itemIdCounter` | 全局 Item _id 计数器 |
| `gameAnimationId` | requestAnimationFrame ID |

---

## 八、兼容性说明

- `applyImportedLevel()` 同时兼容大写字段名（`LauncherLinks`）和小写字段名（`launcherLinks`）
- PixelMap 同时兼容新格式（数字索引数组 + ColorList）和旧格式（颜色字符串数组）
- 旧版 JSON 中缺少 `FrozenLaunchers` / `LauncherLinks` 字段时默认为 `[]`
