<p align="center">
  <h1 align="center">GEE Tile Temporal Mosaic Skill</h1>
</p>

<p align="center">
  <a href="README.md">English</a>
</p>

这是一个面向 Google Earth Engine 的研究型 skill：在尽可能少的 Sentinel-2
或 Landsat tile 中，为研究区构建空间完整、低云量、时间尽量协调的影像合成。

它不是普通的 `qualityMosaic` 教程，而是一个 tile 级的影像选择工作流：每个
tile 选择一景完整影像，先评价多个基础 tile 日期候选，再协调其他 tile 的日期，
最后按固定优先级处理 tile 重叠区。

## 适用问题

例如：

- 一个城市跨越三个 Sentinel-2 MGRS tile，需要找到覆盖研究区所需的最少 tile。
- 在 2019-2023 年每年 5-9 月中搜索一幅全局最佳季节合成。
- 基础 tile 的局部云量尽量低于 3%，其他 tile 低于 20%，并且与基础日期相差不超过 5 天。
- 云量必须计算在 ROI 与 tile 的相交区域内，而不是直接使用整景元数据。
- tile 重叠时，让覆盖贡献最大的 tile 优先显示，低优先级 tile 只填补上层的 masked 区域。

默认结果是 `priority_mosaic`：每个必要 tile 一景影像，覆盖贡献最大的 tile
位于最上层。它不会默认使用跨日期 `qualityMosaic`、median 或逐像元自由挑选日期。

## 兼容性

- OpenAI Codex：安装到 `~/.codex/skills`。
- Claude Code：安装到 `~/.claude/skills`。
- 支持 GEE Code Editor JavaScript 和 Earth Engine Python API/geemap，调用时由用户选择。

该仓库只包含 skill 指令和参考模板，不包含 Earth Engine 凭证、Cloud Project ID、
ROI asset 或其他私有数据。

## 安装

### OpenAI Codex

```bash
git clone https://github.com/NightSensingLab/gee-tile-temporal-mosaic.git \
  ~/.codex/skills/gee-tile-temporal-mosaic
```

显式调用：

```text
$gee-tile-temporal-mosaic
```

### Claude Code

```bash
git clone https://github.com/NightSensingLab/gee-tile-temporal-mosaic.git \
  ~/.claude/skills/gee-tile-temporal-mosaic
```

请完整克隆仓库，不要只复制 `SKILL.md`；`agents/openai.yaml` 和 `references/`
中的参考文件共同组成这个 skill。

## 核心流程

1. 根据 tile footprint 和 ROI 的增量覆盖面积确定最少 tile 集合，避免把重叠面积重复相加。
2. 为每景候选影像计算 ROI 局部云量、清晰比例、云影比例和 footprint 覆盖率。
3. 保留多个基础 tile 候选，不把“基础 tile 最低云量”直接当作最终答案。
4. 对每个基础候选寻找其他 tile 的时间匹配影像，评估完整的 tile/date 组合。
5. 使用阈值和最大日期差作为硬约束，再按最终清晰覆盖率、masked 缺口、日期跨度和目标日期距离排序。
6. 按固定 tile 优先级进行拼接，并输出每个 tile 的 scene ID、日期、云量、覆盖和回退状态。

如果没有合格组合，返回 `incomplete_masked` 或 `no_solution`，不会静默扩大时间范围。

## 仓库结构

```text
SKILL.md                         核心指令和 guardrails
agents/openai.yaml               Codex UI 元数据
references/selection-design.md   set cover、候选组合、评分和重叠语义
references/sentinel2-javascript.md
                                 Sentinel-2 Code Editor 模板
references/python-geemap.md      Python/geemap 模板
references/landsat.md            Landsat Collection 2 掩膜说明
examples/                        真实请求和预期诊断示例
```

## 调用示例

### 三 tile 季节性 Sentinel-2 合成

```text
使用 $gee-tile-temporal-mosaic。我的 ROI 跨越三个 Sentinel-2 MGRS tile。
搜索 2019-2023 年 5-9 月影像，基础 tile 局部云量 <= 3%，其他 tile <= 20%，
其他 tile 与基础日期最大相差 5 天。每个 tile 只选一景，评估多个基础日期候选，
使用 priority_mosaic，输出 GEE Code Editor JavaScript。打印 tile 日期、局部云量、
最终清晰覆盖率、masked 缺口率和 fallback 状态。不要使用 qualityMosaic 或跨日期 median。
```

### Python/geemap 模式

```text
使用 $gee-tile-temporal-mosaic 的 Python/geemap 模式。保留相同的 tile 选择、ROI 局部云量、
日期差和重叠区语义。使用 PROJECT_ID 初始化 Earth Engine，保持 reducer 在服务端运行，
Drive 导出必须显式开启。
```

### 严格 tile 归属

```text
使用 $gee-tile-temporal-mosaic，并设置 overlapMode=exclusive_tile。
低优先级 tile 不得填补其他 tile 的 masked 区域；如果选中 tile 有云，返回不完整的 masked 结果。
```

## 重要限制

- 五年 5-9 月搜索后只输出一幅，是整个窗口的全局最佳季节合成，不是逐年代表性结果。
- `priority_mosaic` 允许低优先级 tile 填补上层 masked 区域，因此必须报告各 tile 日期和日期跨度。
- tile 数量和候选数量增加后，精确组合搜索会迅速变大；大范围任务应限制 `topN` 或使用有界 beam search，并说明近似策略。
- 局部云量依赖云概率和场景分类阈值；积雪、亮屋顶、雾霾和云影需要结合研究目标复核。

## 许可证

本仓库原创 skill 文件采用 MIT License。数据集和 Earth Engine 数据集合仍遵循各自提供方的许可，
详见 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)。

