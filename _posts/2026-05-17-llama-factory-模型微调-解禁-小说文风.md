---
layout: post
title: LLaMA Factory 模型微调（解禁+小说文风）
date: 2026-05-17 11:06 +0800
category: [AI, Finetuning]
tags: [LLaMA Factory, Uncensor, Novel]
---

# 一、概述

核心目标：基于 Qwen2-7B-Instruct 模型，通过两轮 LoRA 微调，实现「模型解禁（无限制输出）+ 中文小说文风迁移」，最终获得可自由生成小说、无审查限制的轻量 LoRA 模型。全程解决多轮格式报错、样本丢失、训练卡顿等问题，最终顺利完成训练，效果达标。

关键环境与工具：

- 基础模型：Qwen2-7B-Instruct

- 微调工具：LLaMA Factory（WebUI 版本）

- 量化配置：4bit 量化（bitsandbytes）

- 数据集格式：JSONL、Parquet（多字段）

# 二、全流程步骤（含细节+规范）

## 阶段1：前期准备（核心铺垫，避免后续踩坑）

### 1.1 数据集准备（两类核心数据集）

本次训练需两类数据集，核心区别的是「是否含对话结构」，直接决定后续训练模式，这是关键前提：

- 数据集1：解禁数据集（含 user/assistant 对话结构）
        

    - 格式：JSON/JSONL，字段需包含「user」（提问/指令）和「assistant」（无限制回复）

    - 用途：第一轮 SFT 微调，让模型突破审查限制，实现无拒绝输出

    - 存放路径：`{数据集根目录}/unlock_data/unlock.jsonl`

- 数据集2：中文小说数据集（纯文本，无对话结构）
        

    - 原始格式：Parquet（含 category、title、content 等多字段），核心有效字段为「content」（小说正文）

    - 转换后格式：JSONL（因 LLaMA Factory 读取多子文件夹 JSONL 需合并，最终统一为单文件）

    - 用途：第二轮 Pre-training 微调，让模型学习小说叙事、描写、文风

    - 原始存放路径：`{数据集根目录}/chinese_novel/（含多级子文件夹，每个子文件夹下有若干.jsonl）`

    - 最终存放路径：`{数据集根目录}/chinese_novel/all_novel_combined.jsonl`（合并后单文件）

### 1.2 数据集合并（解决「多子文件夹 JSONL 无法读取」问题）

小说数据集原始为「主文件夹+多级子文件夹+多个JSONL文件」，LLaMA Factory 不支持直接读取文件夹，需通过命令行合并为单文件，步骤如下：

1. 进入小说主目录（终端命令）：
        `cd {数据集根目录}/chinese_novel`

2. 递归合并所有子文件夹下的 JSONL 文件（生成合并文件 all_novel_combined.jsonl）：
        `find . -name "*.jsonl" -exec cat {} \; > all_novel_combined.jsonl`

### 1.3 dataset_info.json 配置（核心，格式错误直接导致训练失败）

该文件用于告诉 LLaMA Factory 数据集的路径、格式、有效字段，需分两类数据集配置，最终版本如下（直接复制可用）：

```json
[
  "uncensor": {
    "file_name": "uncensor/harmful.jsonl",
    "columns": {
      "prompt": "question",
      "query": "",
      "response": "answer",
      "history": ""
    },
    "format": "jsonl"
  },
  "chinese-noval": {
    "file_name": "chinese_novel/all_novel_combined.jsonl",
    "columns": {
        "prompt": "content"
      },
      "format": "json"
  }
]
```

## 阶段2：第一轮微调（解禁 SFT 微调）

### 2.1 核心配置（必须严格匹配，否则无法解禁或报错）

| 配置项                | 配置值                             | 说明（关键细节）                                                                    |
| --------------------- | ---------------------------------- | ----------------------------------------------------------------------------------- |
| 训练任务              | SFT（有监督微调）                  | 因解禁数据集含 user/assistant 对话，必须用 SFT 模式，用 Pre-training 会导致样本全丢 |
| Chat Template         | qwen                               | 匹配 Qwen2 模型，无需修改为空（WebUI 无 empty 选项时，SFT 模式用 qwen 模板正常）    |
| 数据集                | unlock_data（对应配置文件中的key） | 确保 file_name 路径正确，字段匹配 user/assistant                                    |
| 量化配置              | 4bit（bitsandbytes）               | 节省显存，不影响解禁效果，避免爆显存                                                |
| 精度                  | bf16                               | 匹配 Qwen2 模型，提速且保证效果                                                     |
| Cutoff length         | 2048                               | 无需 4096，节省显存、提升速度，对话类微调足够                                       |
| Batch size            | 1                                  | 避免显存不足，配合梯度累积使用                                                      |
| Gradient Accumulation | 8                                  | 弥补 Batch size 过小的问题，保证训练效果                                            |
| Epoch                 | 1                                  | 避免过拟合，解禁微调1轮足够                                                         |
| 学习率                | 2e-5                               | LoRA 微调最佳学习率，过高易过拟合，过低学不到效果                                   |
| LoRA 参数             | Rank=32，Alpha=64，Target=all      | 轻量 LoRA，不占空间，且能保证解禁效果                                               |
| 额外参数              | upcast_layernorm=True              | 避免训练中数值溢出，防止报错                                                        |
| LoRA 保存路径         | {LoRA保存目录}/train_unlock        | 后续第二轮需加载该 LoRA，需记住路径                                                 |

### 2.2 训练结果验证

训练结束后，日志会显示「Training completed」，且无「Dropped invalid example」提示，说明样本无丢失，解禁微调成功。可初步测试：输入无限制指令，模型无拒绝、无审查，即为达标。

## 阶段3：第二轮微调（小说文风 Pre-training 微调）

核心：小说数据集为纯文本（无对话），必须切换为 Pre-training 模式，否则会导致样本全丢，这是本次训练最容易踩坑的点。

### 3.1 核心配置（继承第一轮 LoRA，保证解禁效果不丢失）

| 配置项        | 配置值                                                          | 说明（关键细节）                                                                            |
| ------------- | --------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| 训练任务      | Pre-training（预训练）                                          | 重中之重！小说为纯文本，无 user/assistant，用 SFT 会批量丢弃样本（Dropped invalid example） |
| Chat Template | qwen（保持不变）                                                | WebUI 无 empty 选项，Pre-training 模式下用 qwen 模板不影响，无需修改                        |
| 数据集        | chinese_novel（对应配置文件中的key）                            | file_name 指向合并后的 all_novel_combined.jsonl，columns 为 prompt:content                  |
| 预加载 LoRA   | {LoRA保存目录}/train_unlock                                     | 继承第一轮解禁效果，避免第二轮微调覆盖解禁能力                                              |
| Max samples   | 3000                                                            | 控制样本量，避免训练过久，3000条足够学习小说文风，且5-7分钟可跑完                           |
| 其他参数      | 与第一轮一致（量化、精度、Batch、梯度累积、学习率、LoRA参数等） | 保证训练稳定性，避免参数混乱导致报错                                                        |

### 3.2 训练结果验证

训练结束后，日志显示「Training completed」，无样本丢失提示，最终 train_loss 约 2.8 左右（Pre-training 模式下，小说微调 loss 2.5-3.0 均为正常，无需追求过低）。

## 阶段4：最终测试（验证双目标是否达标）

### 4.1 测试准备

在 LLaMA Factory WebUI 的 Chat 界面，加载第二轮训练好的 LoRA（{LoRA保存目录}/train_novel），设置：温度 0.7-0.8，最大生成长度拉满。

### 4.2 三类测试用例（覆盖核心目标）

1. 解禁测试：自由创作无限制内容，模型无拒绝、无审查，输出流畅。

2. 文风测试：写一段小说场景描写（如古风、都市），验证文笔细腻度、叙事节奏是否贴合小说风格。

3. 长文本测试：续写一段小说开头，验证长文本生成能力，无逻辑混乱、无重复。

# 三、核心出错点汇总（含原因+解决方案，重点！）

本次训练共遇到4个关键错误，均导致训练失败或样本丢失，整理如下，避免后续复现：

## 错误1：小说数据集用 SFT 模式训练，导致「Dropped invalid example」批量丢样本

- 错误原因：SFT 模式强制要求数据有 user/assistant 对话结构，小说为纯文本（只有 content 字段），模型无法识别「回答」部分，直接判定样本无效丢弃。

- 解决方案：将训练任务切换为 Pre-training（预训练）模式，该模式只读取 prompt 字段（content），不校验对话结构，样本100%保留。

## 错误2：dataset_info.json 中 file_name 填写文件夹路径，导致数据集无法读取

- 错误原因：LLaMA Factory 不支持自动读取文件夹下的所有文件，file_name 必须填写具体的单个文件（.jsonl/.parquet/.json）。

- 解决方案：将多子文件夹下的 JSONL 文件合并为单文件（用 find 命令递归合并），file_name 指向合并后的单文件路径。

## 错误3：Chat Template 找不到 empty 选项，无法设置无模板

- 错误原因：WebUI 下拉菜单中无 empty 选项，最初计划设置 empty 模板适配纯文本训练，无法实现。

- 解决方案：无需修改模板，保持 qwen 模板不变，只要切换为 Pre-training 模式，模板不会影响纯文本训练（Pre-training 不套对话模板）。

## 错误4：训练日志停在「trainable params: XXX」，误以为卡死

- 错误原因：Pre-training 模式下，模型加载完 LoRA 后，会对100MB 左右的 JSONL 文件进行分词、样本构建，需要1-3分钟，日志无输出，易误判为卡死。

- 解决方案：耐心等待，无需重启，1-3分钟后会自动输出训练日志（loss、epoch 等），属于正常加载过程。

## 错误5：忽略 Preprocessing num workers 参数，导致训练速度偏慢

- 错误原因：未设置 Preprocessing num workers，默认值较低，数据预处理速度慢，影响整体训练效率。

- 解决方案：设置 Preprocessing num workers 为4或8，提升数据预处理速度，最终训练速度稳定在598 token/s 左右。

## 错误6：日志末尾出现「No metric eval_loss to plot」，误以为训练失败

- 错误原因：训练时未开启验证集（eval set），无法生成 eval_loss 相关图表，属于 WARNING（警告），不是错误。

- 解决方案：无需处理，不影响训练效果和模型权重，属于正常日志提示。

# 四、关键经验总结（可复用）

1. 数据集格式决定训练模式：**含对话（user/assistant）→ SFT 模式；纯文本 → Pre-training 模式**，这是核心原则，错则必丢样本。

2. dataset_info.json 配置是基础：file_name 必须是单个文件路径，columns 必须匹配数据集有效字段（对话用 user/assistant，纯文本用 prompt:content）。

3. 多子文件夹 JSONL 处理：用 find 命令递归合并，避免手动复制粘贴导致格式混乱。

4. LoRA 继承：多轮微调时，后续轮次需预加载前一轮的 LoRA，避免覆盖之前的效果（如第二轮预加载第一轮解禁 LoRA，保证解禁能力不丢失）。

5. 参数设置平衡：Cutoff length 设为2048（兼顾速度和效果），Batch size=1+梯度累积=8（避免爆显存），学习率2e-5（LoRA 微调最佳）。

6. 日志解读：「Training completed」即为训练结束，WARNING 无需关注，只有 ERROR 才需要排查；Pre-training 模式下，loss 2.5-3.0 为正常，无需追求过低。

# 五、最终成果

1. 模型效果：获得可自由生成中文小说、无审查限制的 LoRA 模型，兼具解禁能力和小说文风，长文本生成流畅、文笔细腻。

2. 训练效率：第二轮3000条样本，速度稳定在598 token/s，全程19分52秒完成，无报错、无样本丢失。

3. 可复用性：本次总结的步骤、配置、错误解决方案，可直接复用至「Qwen2 系列模型 + 纯文本/对话类数据集」的 LoRA 微调任务。

