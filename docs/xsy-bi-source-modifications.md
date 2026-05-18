# xsy-bi 源码修改技术文档

> 基于 Apache ECharts v5.4.2，BI 团队定制修改版本

## 目录

- [1. 柱状图组内排序（groupOrder）](#1-柱状图组内排序grouporder)
- [2. 漏斗图标签垂直对齐（verticalAlignment）](#2-漏斗图标签垂直对齐verticalalignment)
- [3. 漏斗图最小宽度/高度（minWidth / minHeight）](#3-漏斗图最小宽度高度minwidth--minheight)
- [4. 雷达图 Tooltip 维度索引（__dimIdx）](#4-雷达图-tooltip-维度索引__dimidx)
- [5. Y 轴名称水平间距（nameHorizontalGap）](#5-y-轴名称水平间距namehorizontalgap)
- [6. 柱状图 layout undefined 防护](#6-柱状图-layout-undefined-防护)

---

## 1. 柱状图组内排序（groupOrder）

### 功能说明

原版 ECharts 堆叠柱状图中，各系列的堆叠顺序固定为 series 定义顺序。`groupOrder` 实现了**同一分类下各系列按数值大小动态排序**的能力，使得每个 X 轴刻度下的柱子可以按实际值升序或降序排列。

### 应用场景

BI 报表中，堆叠柱状图展示各维度的数值贡献时，用户希望每个分类下各维度按贡献大小排列，快速识别主要贡献者。

### 涉及文件

| 文件 | 修改内容 |
|------|---------|
| `src/chart/bar/BaseBarSeries.ts` | 添加 `groupOrder` 配置项类型定义 |
| `src/layout/barGrid.ts` | 实现数据收集、排序、渲染查找逻辑 |
| `test/bar-groupOrder.html` | 测试用例 |

### 配置方式

```typescript
series: [{
    type: 'bar',
    stack: 'all',
    groupOrder: 'desc',  // 'asc' | 'desc'
    data: [8203, 3489, 9034, 1470]
}, {
    type: 'bar',
    stack: 'all',
    groupOrder: 'desc',
    data: [5000, 7000, 2000, 4000]
}]
```

### 实现原理

#### 数据结构

```typescript
interface layoutItemInfo {
    dataIndex: number
    value: number        // 实际数值，用于排序
    baseValue: number    // X轴分类值
    stackId: string      // 堆叠组ID
    seriesIndex: number
    isOrderData?: boolean // 标记是否为排序专用数据
}

interface layoutInfo {
    orderLayoutDataList: Array<Array<layoutItemInfo>>   // 按分类值索引的排序数据
    noOrderLayoutDataList: Array<Array<layoutItemInfo>> // 非排序数据（含堆叠计算用）
    columnOffsetList: Array<number>                     // 各 series 的偏移量
    groupOrder?: 'asc' | 'desc'
}
```

#### 核心流程

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. getSeriesGroupOrder()                                        │
│    遍历 seriesModels，取第一个定义了 groupOrder 的值             │
└──────────────────────────────┬──────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. collectingLayoutData()                                       │
│    遍历每个 series 的每个数据点，按 baseValue 分组：              │
│    - 有 groupOrder → 加入 orderLayoutDataList                   │
│    - 同时加入 noOrderLayoutDataList（标记 isOrderData=true）     │
│    - 无 groupOrder → 仅加入 noOrderLayoutDataList               │
└──────────────────────────────┬──────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. orderLayoutData()                                            │
│    对 orderLayoutDataList 中每个分类的数据按 value 排序          │
│    columnOffsetList 同步排序                                     │
└──────────────────────────────┬──────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. getLayoutRenderItemInfo() / doGetLayoutRenderItemInfo()       │
│    渲染时查找当前数据项在排序后列表中的位置，返回：               │
│    - columnOffset: 排序后的水平偏移                              │
│    - startValue: 堆叠起始值（前面同方向柱子的累加）              │
└─────────────────────────────────────────────────────────────────┘
```

#### 关键设计点

1. **混合排序/非排序处理**：当排序和非排序 series 共存时，排序数据也加入 `noOrderLayoutDataList` 并标记 `isOrderData = true`，确保堆叠累加计算正确，但渲染时跳过这些标记项。

2. **堆叠值累加**：`doGetLayoutRenderItemInfo` 中通过判断 `value * dataItem.value >= 0 && dataItem.stackId === stackId` 确保只累加同方向、同堆叠组的值。

3. **偏移量映射**：`columnOffsetList` 记录各 series 的偏移量，排序后偏移量与数据项一一对应，实现柱子位置的动态调整。

---

## 2. 漏斗图标签垂直对齐（verticalAlignment）

### 功能说明

漏斗图各块大小不同时，标签引导线的终点默认跟随各自块的边缘，导致文字参差不齐。开启 `verticalAlignment: true` 后，所有标签的引导线终点对齐到最外侧块的边缘，视觉上更整齐。

### 应用场景

BI 漏斗图中，各阶段转化率差异较大时，标签对齐能提升可读性和美观度。

### 涉及文件

| 文件 | 修改内容 |
|------|---------|
| `src/chart/funnel/FunnelSeries.ts` | 添加 `verticalAlignment` 类型定义 |
| `src/chart/funnel/funnelLayout.ts` | 实现对齐逻辑 |

### 配置方式

```typescript
series: [{
    type: 'funnel',
    label: {
        position: 'left',
        verticalAlignment: true  // 标签垂直对齐
    },
    data: [...]
}]
```

### 实现原理

#### 第一步：计算对齐基准点（points0）

遍历所有数据项，根据标签位置（left/right/top/bottom/leftTop/rightTop/leftBottom/rightBottom），找到各方向上最极端的坐标点，存入 `points0` 数组：

```typescript
let points0 = [[0, 0], [0, 0], [0, 0], [0, 0]];
data.each(function (idx) {
    const layout = data.getItemLayout(idx);
    let points = layout.points;
    const labelPosition = labelModel.get('position');
    if (labelPosition === 'left') {
        // 取所有块中最靠左的点
        if (points0[3][0] === 0 || points[3][0] < points0[3][0]) {
            points0[3][0] = points[3][0];
        }
    }
    // ... 其他方向类似
});
```

#### 第二步：标签定位时使用 points0

当 `verticalAlignment === true` 时，标签引导线终点使用 `points0` 而非当前块的 `points`：

```typescript
if (labelPosition === 'left') {
    x2 = x1 - labelLineLen;
    if (verticalAlignment === true) {
        x2 = (points0[3][0] + points0[0][0]) / 2 - labelLineLen;
    }
}
```

### 支持的标签位置

- `left` / `right` — 水平方向对齐
- `top` / `bottom` — 垂直方向对齐
- `leftTop` / `rightTop` / `leftBottom` / `rightBottom` — 角落位置对齐（区分 horizontal/vertical orient）

---

## 3. 漏斗图最小宽度/高度（minWidth / minHeight）

### 功能说明

当漏斗图某些数据值很小时，对应的块会非常窄/矮，几乎不可见。通过设置 `minHeight` / `minWidth`，保证每个块有最小可见尺寸。

### 应用场景

BI 漏斗图中，末端转化率极低的阶段仍需可见展示，避免信息丢失。

### 涉及文件

| 文件 | 修改内容 |
|------|---------|
| `src/chart/funnel/FunnelSeries.ts` | 添加 `minHeight`、`minWidth` 类型定义 |
| `src/chart/funnel/funnelLayout.ts` | 添加 `getItemMinSizeBySizeKey()` 函数 |

### 配置方式

```typescript
// 系列级别（全局生效）
series: [{
    type: 'funnel',
    minHeight: '20',
    minWidth: '30',
    data: [...]
}]

// 数据项级别（优先级更高）
data: [{
    value: 5,
    itemStyle: {
        minHeight: '15',
        minWidth: '25'
    }
}]
```

### 实现原理

```typescript
function getItemMinSizeBySizeKey(
    sizeKey: 'minHeight' | 'minWidth',
    idx: number,
    baseItemSize: number
) {
    // 系列级别的最小尺寸
    var minSize = seriesModel.get([sizeKey]);
    // 数据项级别的最小尺寸（优先级更高）
    var itemModel = data.getItemModel<FunnelDataItemOption>(idx);
    var itemMinSize = itemModel.get(['itemStyle', sizeKey]);
    // 取优先级更高的值
    var currentMinSize = itemMinSize !== undefined ? itemMinSize : minSize;
    // 如果当前尺寸小于最小值，则使用最小值
    if (currentMinSize !== undefined && baseItemSize < Number(currentMinSize)) {
        return Number(currentMinSize);
    }
    return baseItemSize;
}
```

### 优先级

```
数据项 itemStyle.minHeight/minWidth > 系列级 minHeight/minWidth > 不限制
```

---

## 4. 雷达图 Tooltip 维度索引（__dimIdx）

### 功能说明

原版 ECharts 雷达图的 tooltip 回调中无法获知当前悬浮的是哪个维度（指标）。修改后，tooltip 的 params 中会携带 `dimensionIndex`，方便自定义 tooltip 内容。

### 应用场景

BI 雷达图中，tooltip 需要根据不同维度展示不同的单位、描述或格式化逻辑。

### 涉及文件

| 文件 | 修改内容 |
|------|---------|
| `src/util/types.ts` | ECElement 接口添加 `__dimIdx?: number` |
| `src/component/tooltip/TooltipView.ts` | 提取 `__dimIdx` 传入 tooltip params |

### 实现原理

#### 类型扩展

```typescript
// src/util/types.ts - ECElement 接口
interface ECElement {
    //xsy-bi源码修改点： 添加雷达图tip 数据 index
    __dimIdx?: number;
}
```

#### Tooltip 数据注入

```typescript
// src/component/tooltip/TooltipView.ts
if (!isArray(params) && params.seriesType === 'radar') {
    if (el && el.__dimIdx !== undefined) {
        params.dimensionIndex = el.__dimIdx;
    } else {
        params.dimensionIndex = -1;
    }
}
```

### 使用方式

```typescript
tooltip: {
    formatter: function(params) {
        // params.dimensionIndex 即为当前悬浮的维度索引
        const dimIdx = params.dimensionIndex;
        const indicator = radarIndicators[dimIdx];
        return `${indicator.name}: ${params.value[dimIdx]}`;
    }
}
```

---

## 5. Y 轴名称水平间距（nameHorizontalGap）

### 功能说明

原版 ECharts 的 `nameGap` 仅控制轴名称在轴方向上的间距。新增 `nameHorizontalGap` 允许对 Y 轴名称进行水平方向的偏移控制。

### 应用场景

BI 图表中 Y 轴名称需要精确的水平定位，避免与刻度标签重叠或对齐不美观。

### 涉及文件

| 文件 | 修改内容 |
|------|---------|
| `src/coord/axisCommonTypes.ts` | 添加 `nameHorizontalGap?: number` |

### 配置方式

```typescript
yAxis: {
    name: '销售额（万元）',
    nameGap: 15,
    nameHorizontalGap: 10  // 水平方向额外偏移
}
```

### 类型定义

```typescript
// src/coord/axisCommonTypes.ts
interface AxisBaseOptionCommon {
    //xsy-bi源码修改点： y轴添加水平Gap控制nameHorizontalGap.
    nameHorizontalGap?: number;
}
```

---

## 6. 柱状图 layout undefined 防护

### 功能说明

修复柱状图偶发的 `layout` 为 `undefined` 导致的运行时报错，添加防御性检查。

### 应用场景

BI 系统中数据动态加载、图表频繁更新时，偶现数据竞态导致 layout 未初始化。

### 涉及文件

| 文件 | 修改内容 |
|------|---------|
| `src/chart/bar/BarView.ts` | 添加 undefined 防护 |

### 实现代码

```typescript
// src/chart/bar/BarView.ts
if (layout === undefined) {
    return {
        x: 0,
        y: 0,
        width: 0,
        height: 0
    }
}
```

---

## 修改点速查表

| # | 功能 | 配置项 | 文件 | 标记注释 |
|---|------|--------|------|---------|
| 1 | 柱状图组内排序 | `groupOrder: 'asc'\|'desc'` | BaseBarSeries.ts, barGrid.ts | Intra group sorting |
| 2 | 漏斗图标签对齐 | `label.verticalAlignment: true` | FunnelSeries.ts, funnelLayout.ts | xsy-bi源码修改点 |
| 3 | 漏斗图最小尺寸 | `minHeight`, `minWidth` | FunnelSeries.ts, funnelLayout.ts | xsy-bi源码修改点 |
| 4 | 雷达图维度索引 | `params.dimensionIndex` | types.ts, TooltipView.ts | xsy-bi源码修改点 |
| 5 | Y轴水平间距 | `nameHorizontalGap: number` | axisCommonTypes.ts | xsy-bi源码修改点 |
| 6 | layout 防护 | — | BarView.ts | xsy-bi源码修改点 |

---

## 注意事项

1. **版本升级风险**：所有修改均基于 ECharts v5.4.2，升级时需逐一合并。建议通过 `xsy-bi源码修改点` 注释快速定位。
2. **groupOrder 性能**：排序操作在每次布局时执行，数据量极大时需关注性能。
3. **minHeight/minWidth 与排序的交互**：最小尺寸可能影响漏斗图的整体布局比例，需根据实际数据调整。
4. **测试覆盖**：`test/bar-groupOrder.html` 提供了 groupOrder 的测试用例，其他修改建议补充对应测试。
