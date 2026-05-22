# xsy-bi 源码修改测试用例

> 覆盖修改点 1-4，用于验证 BI 定制功能在版本升级后是否正常工作

---

## 1. 柱状图组内排序（groupOrder）

### 测试文件
`test/xsy-groupOrder.html`

### 用例 1.1：堆叠 + groupOrder desc
**配置：**
```javascript
series: [
    { name: 'Finance', type: 'bar', stack: 'total', groupOrder: 'desc', data: [190, 30, 80, 20, 50] },
    { name: 'Education', type: 'bar', stack: 'total', groupOrder: 'desc', data: [25, 90, 40, 60, 10] },
    { name: 'Retail', type: 'bar', stack: 'total', groupOrder: 'desc', data: [84, 50, 120, 30, 70] },
    { name: 'Energy', type: 'bar', stack: 'total', groupOrder: 'desc', data: [12, 70, 25, 100, 40] }
]
```
**预期结果：**
- 每个分类下，柱子从底部到顶部按值从大到小排列
- 例如第一个分类：Finance(190) 在底部，Retail(84) 在上面，Education(25) 再上面，Energy(12) 在顶部
- 柱子之间不重叠，正确堆叠

### 用例 1.2：堆叠 + groupOrder asc
**配置：**
```javascript
series: [
    { name: 'A', type: 'bar', stack: 'all', groupOrder: 'asc', data: [100, 30, 80, 50] },
    { name: 'B', type: 'bar', stack: 'all', groupOrder: 'asc', data: [50, 90, 20, 70] },
    { name: 'C', type: 'bar', stack: 'all', groupOrder: 'asc', data: [30, 60, 100, 40] }
]
```
**预期结果：**
- 每个分类下，柱子从底部到顶部按值从小到大排列
- 例如第一个分类：C(30) 在底部，B(50) 在中间，A(100) 在顶部

### 用例 1.3：水平堆叠 + groupOrder desc
**配置：**
```javascript
xAxis: { type: 'value' },
yAxis: { type: 'category', data: ['D1', 'D2', 'D3', 'D4'] },
series: [
    { name: 'A', type: 'bar', stack: 'all', groupOrder: 'desc', data: [120, 40, 80, 60] },
    { name: 'B', type: 'bar', stack: 'all', groupOrder: 'desc', data: [50, 100, 30, 90] },
    { name: 'C', type: 'bar', stack: 'all', groupOrder: 'desc', data: [80, 60, 110, 40] }
]
```
**预期结果：**
- 水平方向堆叠，每个分类下从左到右按值从大到小排列
- 不重叠，正确堆叠

### 用例 1.4：非堆叠 + groupOrder desc
**配置：**
```javascript
series: [
    { name: 'Sales', type: 'bar', groupOrder: 'desc', data: [200, 60, 100, 80] },
    { name: 'Profit', type: 'bar', groupOrder: 'desc', data: [80, 150, 50, 120] },
    { name: 'Cost', type: 'bar', groupOrder: 'desc', data: [120, 90, 180, 60] }
]
```
**预期结果：**
- 每个分类下，并排的柱子按值从大到小排列（左到右）
- 例如第一个分类：Sales(200) 在左，Cost(120) 在中，Profit(80) 在右

### 用例 1.5：稀疏数据 + 堆叠 + groupOrder
**配置：**
```javascript
series: [
    { name: 'A', type: 'bar', stack: 'all', groupOrder: 'desc', data: [100, null, 80, 60, null, 120] },
    { name: 'B', type: 'bar', stack: 'all', groupOrder: 'desc', data: [50, 90, null, 70, 40, 30] },
    { name: 'C', type: 'bar', stack: 'all', groupOrder: 'desc', data: [null, 60, 110, null, 80, 50] },
    { name: 'D', type: 'bar', stack: 'all', groupOrder: 'desc', data: [30, 40, 50, 90, 60, null] }
]
```
**预期结果：**
- null 值的系列在对应分类下不显示
- 有数据的系列正确排序堆叠，不报错
- 例如第一个分类只有 A(100)、B(50)、D(30)，按 desc 排列

### 用例 1.6：堆叠 + 无 groupOrder（基线对照）
**配置：**
```javascript
series: [
    { name: 'ProdA', type: 'bar', stack: 'total', data: [120, 80, 150, 60] },
    { name: 'ProdB', type: 'bar', stack: 'total', data: [90, 130, 70, 110] },
    { name: 'ProdC', type: 'bar', stack: 'total', data: [60, 100, 120, 80] }
]
```
**预期结果：**
- 按 series 定义顺序堆叠（ProdA 在底部，ProdC 在顶部）
- 所有分类下顺序一致，不受数据值影响

### 用例 1.7：图例联动
**操作：** 点击图例隐藏某个系列
**预期结果：**
- 隐藏后剩余系列重新排序堆叠
- 不报错，不出现空白间隙

---

## 2. 漏斗图标签垂直对齐（verticalAlignment）

### 测试文件
`test/xsy-funnel-verticalAlignment.html`

### 用例 2.1：verticalAlignment=false（默认）
**配置：**
```javascript
series: [{
    type: 'funnel',
    label: { position: 'right' },
    data: [
        { value: 100, name: 'Visit' },
        { value: 80, name: 'Inquiry' },
        { value: 60, name: 'Order' },
        { value: 30, name: 'Click' },
        { value: 10, name: 'Show' }
    ]
}]
```
**预期结果：**
- 各标签引导线终点跟随各自块的边缘
- 标签文字水平位置参差不齐

### 用例 2.2：verticalAlignment=true，position=right
**配置：**
```javascript
series: [{
    type: 'funnel',
    label: { position: 'right', verticalAlignment: true },
    data: [...]
}]
```
**预期结果：**
- 所有标签引导线终点对齐到最宽块（Visit）的右边缘
- 标签文字水平位置整齐对齐

### 用例 2.3：verticalAlignment=true，position=left
**配置：**
```javascript
series: [{
    type: 'funnel',
    label: { position: 'left', verticalAlignment: true },
    data: [...]
}]
```
**预期结果：**
- 所有标签引导线终点对齐到最宽块的左边缘
- 标签文字右对齐，整齐排列

### 用例 2.4：verticalAlignment=true，水平方向漏斗
**配置：**
```javascript
series: [{
    type: 'funnel',
    orient: 'horizontal',
    label: { position: 'bottom', verticalAlignment: true },
    data: [...]
}]
```
**预期结果：**
- 水平方向漏斗图，标签在底部
- 所有标签引导线终点对齐到最高块的底边缘

---

## 3. 漏斗图最小宽度/高度（minWidth / minHeight）

### 测试文件
`test/xsy-funnel-minSize.html`

### 用例 3.1：无 minHeight/minWidth（默认）
**配置：**
```javascript
series: [{
    type: 'funnel',
    data: [
        { value: 1000, name: 'Visit' },
        { value: 500, name: 'Inquiry' },
        { value: 100, name: 'Order' },
        { value: 10, name: 'Click' },
        { value: 1, name: 'Show' }
    ]
}]
```
**预期结果：**
- Click 和 Show 的块非常窄，几乎不可见

### 用例 3.2：系列级 minWidth
**配置：**
```javascript
series: [{
    type: 'funnel',
    minWidth: '80',
    data: [...]
}]
```
**预期结果：**
- 所有块的宽度至少为 80px
- Click 和 Show 不再是极窄的尖角，有明显可见宽度

### 用例 3.3：数据项级 minWidth 覆盖系列级
**配置：**
```javascript
series: [{
    type: 'funnel',
    minWidth: '80',
    data: [
        { value: 1000, name: 'Visit' },
        { value: 10, name: 'Click', itemStyle: { minWidth: '150' } },
        { value: 1, name: 'Show', itemStyle: { minWidth: '150' } }
    ]
}]
```
**预期结果：**
- Click 和 Show 的宽度为 150px（数据项级覆盖系列级的 80px）
- 其他块的最小宽度仍为 80px

### 用例 3.4：系列级 minHeight（垂直漏斗，多数据项压缩场景）
**配置：** 容器高度 300px，13 个数据项
```javascript
series: [{
    type: 'funnel',
    minHeight: '25',
    data: [
        { value: 100, name: 'A' }, { value: 90, name: 'B' },
        { value: 80, name: 'C' }, ..., { value: 1, name: 'M' }
    ]
}]
```
**预期结果：**
- 每个块的高度至少为 25px
- 不会出现块被压缩到看不见的情况

### 用例 3.5：水平漏斗 + minHeight
**配置：**
```javascript
series: [{
    type: 'funnel',
    orient: 'horizontal',
    minHeight: '60',
    data: [...]
}]
```
**预期结果：**
- 水平方向漏斗图中，minHeight 控制每个块的高度（窄维度）
- 小值的块不会变成极细的线条

---

## 4. 雷达图 Tooltip 维度索引（__dimIdx）

### 测试文件
`test/xsy-radar-dimIdx.html`

### 用例 4.1：单系列，hover 各轴点
**配置：**
```javascript
tooltip: {
    trigger: 'item',
    formatter: function (params) {
        var dimIdx = params.dimensionIndex;
        if (dimIdx === undefined || dimIdx === -1) {
            return params.name + '<br/>' + params.value.join(', ');
        }
        var indicator = indicators[dimIdx];
        return '<b>' + indicator.name + '</b><br/>'
            + 'dimensionIndex: ' + dimIdx + '<br/>'
            + 'value: ' + params.value[dimIdx];
    }
},
radar: {
    indicator: [
        { name: 'Sales', max: 100 },
        { name: 'Admin', max: 100 },
        { name: 'IT', max: 100 },
        { name: 'Support', max: 100 },
        { name: 'Dev', max: 100 },
        { name: 'Marketing', max: 100 }
    ]
},
series: [{
    type: 'radar',
    data: [{ value: [80, 60, 90, 70, 85, 55], name: 'Team A' }]
}]
```
**操作：** 依次 hover 雷达图各轴上的数据点
**预期结果：**
- hover Sales 轴点时，tooltip 显示 `dimensionIndex: 0`，value 为 80
- hover Admin 轴点时，tooltip 显示 `dimensionIndex: 1`，value 为 60
- hover IT 轴点时，tooltip 显示 `dimensionIndex: 2`，value 为 90
- 以此类推，每个轴点对应正确的 dimensionIndex

### 用例 4.2：hover 连线区域（非轴点）
**操作：** hover 雷达图的连线区域（非数据点位置）
**预期结果：**
- tooltip 中 `dimensionIndex` 为 -1 或 undefined
- 不报错，显示完整数据

### 用例 4.3：多系列雷达图
**配置：**
```javascript
series: [{
    type: 'radar',
    data: [
        { value: [80, 60, 90, 70, 85, 55], name: 'Team A' },
        { value: [50, 80, 70, 90, 60, 75], name: 'Team B' }
    ]
}]
```
**操作：** hover 不同系列的轴点
**预期结果：**
- 正确区分系列和维度
- Team A 的 Sales 点：dimensionIndex=0，value=80
- Team B 的 Sales 点：dimensionIndex=0，value=50

---

## 测试检查清单

| # | 用例 | 通过条件 | 状态 |
|---|------|---------|------|
| 1.1 | 堆叠 desc | 每个分类按值从大到小堆叠，不重叠 | ☐ |
| 1.2 | 堆叠 asc | 每个分类按值从小到大堆叠 | ☐ |
| 1.3 | 水平堆叠 | 水平方向正确排序堆叠 | ☐ |
| 1.4 | 非堆叠排序 | 并排柱子按值排序 | ☐ |
| 1.5 | 稀疏数据 | null 值不显示，有数据的正确排序，不报错 | ☐ |
| 1.6 | 无排序基线 | 按 series 定义顺序堆叠 | ☐ |
| 1.7 | 图例联动 | 隐藏系列后重新排序，不报错 | ☐ |
| 2.1 | 默认不对齐 | 标签位置参差不齐 | ☐ |
| 2.2 | right 对齐 | 标签右侧对齐到最宽块边缘 | ☐ |
| 2.3 | left 对齐 | 标签左侧对齐到最宽块边缘 | ☐ |
| 2.4 | 水平方向对齐 | 水平漏斗标签对齐 | ☐ |
| 3.1 | 无最小尺寸 | 小值块极窄/极薄 | ☐ |
| 3.2 | 系列级 minWidth | 所有块至少指定宽度 | ☐ |
| 3.3 | 数据项级覆盖 | 指定项宽度大于系列级 | ☐ |
| 3.4 | minHeight 压缩场景 | 多数据项时块不被压没 | ☐ |
| 3.5 | 水平漏斗 minHeight | 水平方向窄维度有最小值 | ☐ |
| 4.1 | 单系列 dimIdx | 各轴点返回正确 dimensionIndex | ☐ |
| 4.2 | 非轴点 hover | dimensionIndex 为 -1，不报错 | ☐ |
| 4.3 | 多系列 dimIdx | 区分系列和维度 | ☐ |
