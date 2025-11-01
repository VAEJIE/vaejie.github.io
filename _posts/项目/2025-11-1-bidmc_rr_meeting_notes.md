---
layout: post
title: "cmssl复现每日记录"
description: ""
categories: "项目"
---
# BIDMC_RR 微调实验总结

## 1. 训练模式差异与损失曲线解读

| 编号 | 训练配置 | 关键超参 / 损失函数 | Loss 走势（前几轮 → 最佳 → 早停时） | 验证集表现小结 |
| ---- | -------- | ------------------- | ----------------------------------- | --------------- |
| A | `full_finetuning` + `MSELoss`（`lr=1e-4`, `backbone_lr=1e-5`, `weight_decay=1e-4`） | 损失函数保持为原始 `MSELoss` | Epoch1: Train 1.078, Val 0.724 → Epoch2: Train 0.995, Val 0.747（最佳 Val MAE≈2.425）→ 之后 Val Loss 快速上涨，Epoch7 提前停止 | 快速过拟合；Val/Test MAE ≈ 2.42 / 2.62 |
| B | `linear_probing` + `MSELoss`（冻结 backbone，`lr=1e-4`） | 仅训练 head；backbone 冻结 | Epoch1: Train 1.025, Val 0.775 → Epoch8: Train 0.628, Val 0.506（最佳 Val MAE≈2.243）→ Loss 稳定在 ~0.62 / 0.51，Epoch15 早停 | 最稳定；Val/Test MAE ≈ 2.24 / 2.37 |
| C | `full_finetuning` + `MSELoss`（`lr=5e-5`, `backbone_lr=5e-6`, `weight_decay=1e-3`） | 更强正则尝试 | Epoch1: Train 1.026, Val 0.757 → Epoch5: Train 0.932, Val 0.676（最佳 Val MAE≈2.466）→ Epoch8 开始 Val Loss 激增至 >1.3，Epoch10 强制停止 | 仍然过拟合，甚至早停后 Test MAE ≈ 3.82 |

### 1.1 Loss 数值为何看似“偏大”

1. **标签已做 z-score 标准化**：脚本先对呼吸率标签做 `mean/std` 归一化，Loss 的 0.6 / 0.5 其实是标准化空间里的均方误差。
2. **换算回原始单位**：
   - `bidmc_rr` 训练集标签标准差为 **4.43**。
   - 若标准化 Loss = 0.52，则原始 RMSE ≈ `sqrt(0.52) × 4.43 ≈ 3.2`，MAE ≈ `0.52 × 4.43 ≈ 2.3`。
   - 日志同步打印的 `Val MAE / Test MAE`（原空间）分别约 2.2~2.6，与换算完全一致。
3. **Val Loss 早于 Train Loss 下降并更低**：验证集标签标准差更小（3.86），且验证阶段关闭 Dropout/BN 的噪声，导致 Val Loss 常常比训练 Loss 略低，这在正常范围内。

## 2. 标签统计（用于解释 HR 与 RR Loss 曲线差异）

| 数据集 | Split | 均值 | 标准差 | 极值 |
| ------ | ----- | ---- | ------ | ---- |
| `bidmc_rr` | train | 17.22 | **4.43** | 0.0 ~ 36.0 |
| `bidmc_hr` | train | 88.12 | **11.76** | 52.2 ~ 131.2 |

> **解释**：HR 标签方差远大于 RR。标准化后 HR 任务的误差被“压缩”得更厉害，loss 很容易降到 <0.1；RR 标签方差小，同样的 2~3 单位误差在标准化空间里要表现为 0.5~0.6 的损失值，这属于数据尺度差异，而非训练失败。

## 3. 会议汇报要点

1. **预训练背景**：直接使用师兄提供的 `jtwbio_encoder_full_weight_ppg_large` 权重，没有重新训练 encoder / tokenizer。
2. **两条主要路线**：
   - `linear_probing + MSELoss`（仅训练 head） → 最稳定，Val/Test MAE ≈ 2.24 / 2.37。
   - `full_finetuning + MSELoss`（解冻全部参数） → 最佳 Val MAE ≈ 2.42，但很快过拟合；调低学习率和加 weight decay 后仍无改善。
3. **Loss 数值解读**：日志里的 0.6 / 0.5 来自标准化空间，应配合原空间 `Val/Test MAE` 解读；验证 loss 稍低属于正常；与 HR 任务不同是因为标签尺度差。
4. **后续计划建议**：
   - 对 RR 任务优先采用 `linear_probing` 作为 baseline；
   - 若需探索 full finetuning，考虑只解冻后几层或改用更鲁棒的损失（如 Smooth L1），并配合更严格的正则；
   - 在组会中强调“预训练任务与 RR 特征差异导致线性 probing 更稳定、full finetuning 易扭曲表示”。


