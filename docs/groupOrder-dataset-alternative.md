# 柱状图组内排序：dataSet 方案 vs groupOrder 源码方案

## 概述

ECharts 原生机制可以通过前端预处理数据实现类似 groupOrder 的组内排序效果，无需修改源码。本文档客观对比两种方案的实现方式、适用场景和各自的局限性。

---

## dataSet 方案实现

### 核心思路

在前端对每个分类下的各系列值进行排序，然后按排序后的位置重新分配到固定的 series 槽位中。ECharts 按 series 索引顺序渲染（堆叠从下到上，非堆叠从左到右），从而实现组内排序效果。

### 实现代码

```javascript
function datasetSort(categories, rawSeriesMap, order) {
    // rawSeriesMap: { seriesName: { category: value } }
    var seriesNames = Object.keys(rawSeriesMap);
    var n = seriesNames.length;
    var slots = [];
    for (var i = 0; i < n; i++) slots.push([]);

    categories.forEach(function(cat) {
        var items = seriesNames.map(function(name) {
            return { name: name, value: rawSeriesMap[name][cat] || 0 };
        });
        items.sort(function(a, b) {
            return order === 'desc' ? b.value - a.value : a.value - b.value;
        });
        items.forEach(function(item, si) {
            slots[si].push(item.value);
        });
    });
    return slots; // slots[0] = 每个分类下排名第1的值, slots[1] = 排名第2的值...
}

// 使用
var sorted = datasetSort(categories, rawData, 'desc');
var option = {
    series: sorted.map(function(data, i) {
        return {
            name: '排名' + (i + 1),
            type: 'bar',
            stack: 'total',  // 堆叠场景
            data: data
        };
    })
};
```

### 视觉效果

dataSet 方案在纯排序视觉效果上与 groupOrder 方案**没有本质区别**：
- 堆叠场景：值大的在底部（或顶部），值小的在另一端 ✅
- 非堆叠场景：值大的靠左（或靠右） ✅

---

## 两种方案对比

### 排序效果

两种方案的排序视觉效果相同，没有差异。

### 功能差异

| 维度 | dataSet 方案 | groupOrder 源码方案 |
|------|-------------|-------------------|
| 排序视觉效果 | ✅ 相同 | ✅ 相同 |
| 是否需要改源码 | ✅ 不需要 | ❌ 需要 |
| 实现复杂度 | 较高（前端预处理逻辑） | 低（配置一个属性） |
| tooltip 显示原始系列名 | ❌ 只能显示"排名1/2/3" | ✅ 显示真实系列名 |
| 图例显示原始系列名 | ❌ 只能显示"排名1/2/3" | ✅ 显示真实系列名 |
| 图例隐藏后自动重排 | ❌ 需要重新计算并 setOption | ✅ 自动重排 |
| 颜色与系列绑定 | ❌ 同一颜色在不同分类可能代表不同系列 | ✅ 颜色始终对应同一系列 |
| 动态数据更新 | 需要重新计算排序 | 自动排序 |
| 版本升级维护 | ✅ 无成本 | ❌ 需要适配新版本 |

### 关键差异详解

#### 1. tooltip / 图例语义

dataSet 方案的 series name 只能是位置标识（如"排名1"、"排名2"），无法显示原始系列名称。

- **如果业务只关心排名效果**（如"每个月销售额最高的是谁"），这不是问题
- **如果业务需要从 tooltip 中识别具体系列**（如"这个柱子是销售漏斗还是客户承诺"），则 dataSet 方案无法满足

#### 2. 颜色标识

两种方案中，同一颜色在不同分类下都可能代表不同的原始系列——这是"组内排序"的本质。区别在于：

- dataSet 方案：颜色绑定的是"排名位置"（蓝色=排名第1）
- groupOrder 方案：颜色绑定的是"原始系列"（蓝色=产品A）

哪种更合适取决于业务需求。

#### 3. 图例交互

groupOrder 方案在图例隐藏某个系列后，剩余系列会自动重新排序。dataSet 方案隐藏一个"排名位置"后，剩余位置的数据不会重新排序，需要业务代码监听图例事件并重新计算。

---

## 适用场景建议

### 适合用 dataSet 方案的场景

- 不需要从 tooltip/图例中识别具体是哪个原始系列
- 只关心"每个分类下的排名效果"
- 不想承担源码修改的升级维护成本
- 图例交互需求简单（不需要隐藏后重排）

### 适合用 groupOrder 源码方案的场景

- tooltip 需要显示原始系列名称（BI 报表常见需求）
- 图例需要显示原始系列名称并支持交互
- 颜色需要始终对应同一系列（用户通过颜色识别维度）
- 需要图例隐藏后自动重排
- 希望配置简单，一个属性搞定

---

## 结论

dataSet 方案是一个**可行的无侵入替代方案**，在纯排序视觉效果上与 groupOrder 没有差异。

groupOrder 源码方案的核心优势在于**保持了系列的语义完整性**——tooltip、图例、颜色都能正确反映原始系列信息，且支持图例交互后自动重排。这在 BI 产品中通常是必要的，因为用户需要从图表中识别具体的业务维度。

选择哪种方案取决于产品对 tooltip/图例/颜色语义的需求强度，以及对源码维护成本的接受程度。
