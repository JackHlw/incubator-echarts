---
name: xsy-bi ECharts 源码升级
description: 将 xsy-bi 团队的定制源码修改同步到新版本 ECharts 的标准操作流程
inclusion: manual
---

# xsy-bi ECharts 源码升级 Skill

## 概述

本 skill 描述了将 xsy-bi 团队在 ECharts 上的定制修改，从当前版本迁移到新版本 ECharts 的标准操作流程。

## 前置条件

- 已配置好 Git SSH key，能正常访问 origin 和 apache 远端
- 了解当前 xsy-bi 修改点（参考 `docs/xsy-bi-source-modifications.md`）

## 操作流程

### 第一步：准备远端和分支

1. 确认 apache remote 指向正确的上游仓库：
   ```bash
   git remote set-url apache https://github.com/apache/echarts.git
   ```

2. 拉取目标版本 tag：
   ```bash
   git fetch apache tag <目标版本号>
   ```

3. Stash 当前未提交的修改（如有）：
   ```bash
   git stash push -m "保存当前修改"
   ```

4. 基于目标版本 tag 创建新分支：
   ```bash
   git checkout -b xsy-echarts<版本号> <目标版本tag>
   ```

### 第二步：逐个适配源码修改

按以下顺序逐个适配修改点，每个修改点需要：
1. 在新版本中找到对应文件和位置
2. 对比新版本代码结构，确认接口/函数签名是否变化
3. 适配修改代码到新结构
4. 保留 `//xsy-bi源码修改点` 注释标记

#### 修改点清单

| # | 功能 | 涉及文件 | 复杂度 |
|---|------|---------|--------|
| 1 | Y轴水平间距 nameHorizontalGap | `src/coord/axisCommonTypes.ts` | 低 |
| 2 | 雷达图维度索引 __dimIdx | `src/util/types.ts`, `src/component/tooltip/TooltipView.ts` | 低 |
| 3 | 漏斗图标签对齐 verticalAlignment | `src/chart/funnel/FunnelSeries.ts`, `src/chart/funnel/funnelLayout.ts` | 中 |
| 4 | 漏斗图最小尺寸 minHeight/minWidth | `src/chart/funnel/FunnelSeries.ts`, `src/chart/funnel/funnelLayout.ts` | 中 |
| 5 | 柱状图 layout undefined 防护 | `src/chart/bar/BarView.ts` | 低 |
| 6 | 柱状图组内排序 groupOrder | `src/chart/bar/BaseBarSeries.ts`, `src/layout/barGrid.ts` | 高 |

#### 适配要点

**简单类型定义修改（复杂度低）：**
- 在对应接口中找到相邻属性，在其后添加新属性
- 确认属性类型在新版本中仍然兼容

**漏斗图修改（复杂度中）：**
- `labelLayout` 函数结构可能变化，需确认 `data.each` 回调内的变量名
- `funnelLayout` 函数中 `getLinePoints` 和 for 循环结构需对照
- `getItemMinSizeBySizeKey` 辅助函数放在 for 循环之后、`labelLayout(data)` 之前

**groupOrder 修改（复杂度高）：**
- 需要在 `layout` 函数前添加接口定义和模块级 `layoutInfo` 变量
- 修改 `layout` 函数：添加 `collectingLayoutData` 调用和 `orderLayoutData` 调用
- 修改 `createProgressiveLayout`：用 `getLayoutRenderItemInfo` 替代固定 `columnOffset`
- 需要额外 import：`clone`, `isNumber` from `zrender/src/core/util`
- 堆叠计算逻辑：`startValue` 替代原有的 `stackStartValue`

### 第三步：验证

1. 运行 TypeScript 诊断检查（IDE 或 `npx tsc --noEmit`）
2. 确认修改文件无新增编译错误（项目本身的依赖错误可忽略）
3. 如有测试用例（如 `test/bar-groupOrder.html`），在浏览器中验证功能

### 第四步：提交

验证通过后提交：
```bash
git add <修改的文件列表>
git commit -m "feat: sync xsy-bi source modifications to ECharts <版本号>"
```

## 常见问题

### Q: 新版本中文件路径变了怎么办？
A: 使用 `fileSearch` 工具搜索文件名，ECharts 的模块命名通常保持一致。

### Q: 接口名或函数签名变了怎么办？
A: 用 `grepSearch` 搜索关键字（如 `FunnelLabelOption`、`BaseBarSeriesOption`），找到新位置后对照修改。

### Q: TypeScript 报类型错误怎么办？
A: 检查新版本中相关类型是否有变化（如属性名、泛型参数），按新版本的类型系统调整。

### Q: groupOrder 的 `isNumber` 在新版本中不存在？
A: 检查 zrender 的 util 导出，可能改名为 `isNumeric` 或需要从其他路径导入。

## 参考文档

- 完整修改说明：`docs/xsy-bi-source-modifications.md`
- groupOrder 测试用例：`test/bar-groupOrder.html`
