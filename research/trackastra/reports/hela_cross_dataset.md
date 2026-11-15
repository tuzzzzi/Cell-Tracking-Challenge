# Fluo-N2DL-HeLa 分割与 Trackastra 跨数据集评估

**Multi-scale + 原始 `general_2d` + ILP 在两条 HeLa 序列上均取得最高 TRA 和 OP_CTB。** `01` 的 TRA 为 **0.93070**，`02` 为 **0.95500**。固定 Mask 与权重后，Greedy 在两条序列上都取得更高的 CT、TF 和严格分裂分数 BC(0)。[完整评分表](../results/hela_cross_dataset/comparison.csv)

## 实验设计

本实验检验在 Fluo-N2DH-GOWT1 上选定的 **Multi-scale + 原始 `general_2d` + ILP**，直接应用到荧光核标记的 Fluo-N2DL-HeLa 后表现如何。两条 HeLa 序列都来自 CTC 公布的有标注训练包，本项目将它们作为跨数据集评估。分割和追踪使用已有权重及固定参数，GT/TRA 和 GT/SEG 用于评分。[CTC 数据集说明](https://celltrackingchallenge.net/2d-datasets/) · [此前的 GOWT1 实验](integrated_conclusion.md)

| HeLa 序列 | 图像帧 | 每帧尺寸（高 × 宽） | GT/TRA 帧 | GT/SEG 标注帧 |
|---|---:|---:|---:|---:|
| 01 | 92 | 700 × 1100 | 92 | 28 |
| 02 | 92 | 700 × 1100 | 92 | 8 |

- **分割输入。** 用 Cellpose-SAM v2（`cpsam_v2`）生成 `diameter=48` 的单尺度 Mask。Multi-scale 在保留原实例的基础上，从 `diameter=32` 的预测中补入满足重叠阈值 0.10、最小面积 200 像素的候选。沿用组员合并代码的提交 `72ed2c1`。[分割合并实现](https://github.com/wyl2003/Cell-Tracking-Challenge/blob/72ed2c19d55ac606cf28f3d928e97e8362301248/src/segmentation_postprocess.py)
- **追踪权重与解码器。** 两套 Mask 分别给原始 `general_2d` 和此前选定的 GOWT1 微调 checkpoint（seed 42，最佳 epoch 5），各用 Greedy 和 ILP 运行一次。每条序列 8 组，对照时固定其余因素。[运行脚本](../scripts/run_multiscale_tracking.py)
- **评分口径。** 本地使用 py-ctcmetrics 1.3.3。端到端 DET、SEG、TRA 和 CT 基于导出的 CTC 结果，OP_CTB 为 SEG 和 TRA 的平均值。另对追踪前的 Mask 计算 DET 和 SEG，用于确认变化来自输入分割还是追踪后的输出。两条序列分别报告，SEG 仅基于各自有 GT/SEG 的帧。[CTC 指标定义](https://celltrackingchallenge.net/evaluation-methodology/)

## 分割结果 · 追踪前的 Mask

| 序列 | 输入 Mask | 实例总数 | DET ↑ | SEG ↑ | 漏检 FN ↓ | 误检 FP ↓ | 需拆分 NS ↓ |
|---|---|---:|---:|---:|---:|---:|---:|
| 01 | 单尺度 | 8533 | 0.92535 | 0.71934 | 591 | 494 | 9 |
| 01 | Multi-scale | 8629 | **0.93379** | **0.72419** | **516** | 515 | 9 |
| 02 | 单尺度 | 26827 | 0.95059 | 0.82933 | 954 | 2471 | 110 |
| 02 | Multi-scale | 27123 | **0.95804** | **0.83591** | **752** | 2570 | 115 |

- **01** 的 52 帧增加了 96 个候选。漏检减少 75，误检增加 21，SEG 提高约 0.00484。
- **02** 的 89 帧增加了 296 个候选。漏检减少 202，误检增加 99，另增加 5 次节点拆分操作，SEG 提高约 0.00658。

两条序列都以部分新增误检换取了更多检测匹配。候选数、漏检减少量和误检增加量分别按各阶段定义统计。GT/SEG 的评测样本分别有 491 和 895 个标注实例。

## HeLa 01 · 端到端对照

| 输入 Mask | Trackastra | 解码器 | DET ↑ | SEG ↑ | TRA ↑ | CT ↑ | OP_CTB ↑ | AOGM ↓ |
|---|---|---|---:|---:|---:|---:|---:|---:|
| 单尺度 | 原始 | Greedy | 0.92160 | 0.71606 | 0.91904 | 0.24433 | 0.81755 | 8034.0 |
| 单尺度 | 原始 | ILP | 0.92535 | 0.71934 | 0.92216 | 0.22830 | 0.82075 | 7724.0 |
| 单尺度 | GOWT1 微调 | Greedy | 0.92183 | 0.71606 | 0.91911 | 0.23448 | 0.81759 | 8026.5 |
| 单尺度 | GOWT1 微调 | ILP | 0.92535 | 0.71934 | 0.92156 | 0.21362 | 0.82045 | 7784.0 |
| Multi-scale | 原始 | Greedy | 0.93051 | 0.72090 | 0.92806 | **0.25862** | 0.82448 | 7138.5 |
| Multi-scale | 原始 | ILP | **0.93379** | **0.72419** | **0.93070** | 0.23567 | **0.82745** | **6876.5** |
| Multi-scale | GOWT1 微调 | Greedy | 0.93086 | 0.72090 | 0.92816 | 0.24662 | 0.82453 | 7129.0 |
| Multi-scale | GOWT1 微调 | ILP | **0.93379** | **0.72419** | 0.93012 | 0.21746 | 0.82715 | 6934.5 |

在 `01` 固定原始权重和 ILP，换成 Multi-scale 使 TRA 增加 **0.00854**，AOGM 下降 **847.5**。固定 Multi-scale 和原始权重，Greedy 给出更高的 CT（0.25862 对 0.23567），ILP 的 TRA 更高（0.93070 对 0.92806）。

## HeLa 02 · 端到端对照

| 输入 Mask | Trackastra | 解码器 | DET ↑ | SEG ↑ | TRA ↑ | CT ↑ | OP_CTB ↑ | AOGM ↓ |
|---|---|---|---:|---:|---:|---:|---:|---:|
| 单尺度 | 原始 | Greedy | 0.94937 | 0.82845 | 0.94662 | 0.14568 | 0.88753 | 15585.0 |
| 单尺度 | 原始 | ILP | 0.95059 | 0.82933 | 0.94734 | 0.13165 | 0.88834 | 15373.0 |
| 单尺度 | GOWT1 微调 | Greedy | 0.94957 | 0.82845 | 0.94654 | 0.13723 | 0.88750 | 15606.5 |
| 单尺度 | GOWT1 微调 | ILP | 0.95059 | 0.82933 | 0.94677 | 0.11681 | 0.88805 | 15540.0 |
| Multi-scale | 原始 | Greedy | 0.95691 | **0.83591** | 0.95430 | **0.19503** | 0.89510 | 13343.0 |
| Multi-scale | 原始 | ILP | **0.95804** | **0.83591** | **0.95500** | 0.17822 | **0.89546** | **13138.0** |
| Multi-scale | GOWT1 微调 | Greedy | 0.95719 | **0.83591** | 0.95435 | 0.18863 | 0.89513 | 13327.5 |
| Multi-scale | GOWT1 微调 | ILP | **0.95804** | **0.83591** | 0.95457 | 0.16349 | 0.89524 | 13262.5 |

在 `02` 固定原始权重和 ILP，换成 Multi-scale 使 TRA 增加 **0.00766**，CT 增加 **0.04656**。两条 HeLa 序列的 GT/TRA 标注实例分别为 8639 和 25420。密度差异和 GT/SEG 覆盖差异都会影响分数的解释，本文的主要结论来自每条序列内的配对比较。

## 分割和追踪各自带来了什么变化

### Multi-scale 的收益延续到了 HeLa

固定序列、权重与解码器后的八个配对中，Multi-scale 的 **TRA、CT 和 OP_CTB 都提高**。TRA 的增量为 **0.00766–0.00904**。此前 GOWT1 的四个配对中，TRA 也全部上升，增量为 0.00422–0.00454。GOWT1 `02` 的 SEG 持平，HeLa 两条序列的追踪前 SEG 均提高。

固定原始权重与 ILP 时，AOGM 的分项变化如下。FN、FP 和待补连接列中的数字是操作次数，AOGM 是所有修正操作的加权总代价。

| 序列 | 漏检 FN | 误检 FP | 待补连接 EA | AOGM |
|---|---:|---:|---:|---:|
| 01 | 591 → 516 | 494 → 515 | 812 → 729 | 7724.0 → 6876.5 |
| 02 | 954 → 752 | 2471 → 2570 | 1698 → 1476 | 15373.0 → 13138.0 |

两条序列都减少了漏检和待补连接，新增误检的代价被这些收益抵消。以 `02` 为例，202 个减少的漏检和 222 条减少的待补连接是 AOGM 下降的主要来源。[CTC 原始评分和图修正明细](../results/hela_cross_dataset/ctc_metrics.csv)

### ILP 提高 TRA，Greedy 在完整轨迹和分裂上占优

固定 Multi-scale 和原始权重，切换到 ILP 后保留了更多检测节点，TRA 在 `01` 增加 **0.00264**，在 `02` 增加 **0.00070**。同期 CT 下降 **0.02295** 和 **0.01681**，TF 也下降。ILP 保留的节点能改善总体图评分，但两条序列的完整轨迹匹配和严格分裂评分均以 Greedy 更高。

| 序列 | Multi-scale + 原始权重 | 正确分裂 / GT | 误报分裂 | BC(0) ↑ | TF ↑ |
|---|---|---:|---:|---:|---:|
| 01 | Greedy | 21 / 94 | 21 | **0.30882** | **0.91763** |
| 01 | ILP | 21 / 94 | 36 | 0.27815 | 0.91105 |
| 02 | Greedy | 53 / 209 | 129 | **0.27110** | **0.90582** |
| 02 | ILP | 53 / 209 | 189 | 0.23503 | 0.89827 |

这里的分裂计数使用 CTC 的严格时间匹配。ILP 与 Greedy 找到同样多的真实分裂，但 ILP 误报更多，因此 BC(0) 更低。

### GOWT1 微调权重在 HeLa 上没有稳定优势

同用 Multi-scale Mask 和 ILP 时，原始权重在 `01`、`02` 的 TRA 分别比微调权重高 **0.00058、0.00043**，CT 和 BC(0) 也更高。改用 Greedy 后，微调仅带来很小的 TRA 增益，且两条序列上的 CT 都降低。课程报告可把微调作为对照，把 **Multi-scale + 原始权重 + ILP** 作为总体指标的主配置，另列同样 Mask 与权重下的 **Greedy**，展示轨迹完整性和分裂表现。

## 结论范围与结果文件

这是从 GOWT1 固定配置到 HeLa 两条有标注序列的迁移评估。当前 `general_2d` 随附的训练配置列有 `DIC-C2DH-HeLa`，未列出这里的 `Fluo-N2DL-HeLa`。本实验未在 HeLa 上重新训练或调参，Cellpose 预训练语料的逐序列重叠情况未作独立核验。结果支持在这两条序列上观察到的改进，跨数据集结论仍限于本次的样本和配置。[Trackastra 预训练模型说明](https://github.com/weigertlab/trackastra/blob/aa57a95160002e0fc70b915ab74178b39c99fd6a/trackastra/model/pretrained.json)

本地导出了 16 组 CTC 结果，每组含 92 张追踪 Mask 和一份 `res_track.txt`。已逐帧核对尺寸、标签与轨迹表，并检查父子关系的时间顺序。原始 CTC 输出可在本地 `runs/trackastra/hela_cross_dataset/seq01` 和 `seq02` 中查看，仓库报告所需的精简记录列在下方。

- [16 组端到端对照](../results/hela_cross_dataset/comparison.csv) · [CTC 原始评分](../results/hela_cross_dataset/ctc_metrics.csv)
- [追踪前的 DET 和 SEG](../results/hela_cross_dataset/segmentation_only.csv) · [逐帧新增候选统计](../results/hela_cross_dataset/segmentation_counts.csv)
- [数据、权重指纹和运行参数](../results/hela_cross_dataset/provenance.json) · [分割和追踪脚本](../scripts/run_multiscale_tracking.py)
