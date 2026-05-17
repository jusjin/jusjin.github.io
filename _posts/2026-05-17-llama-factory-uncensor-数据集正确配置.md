---
layout: post
title: LLaMA-Factory uncensor 数据集正确配置
date: 2026-05-17 10:50 +0800
category: [AI, datasets]
tags: [LLaMA Factory, Uncensor, datasets]
---

`uncensor` 数据集的实际数据文件是 `harmful.jsonl`，因此需要修改 dataset_info.json 配置，适配该文件格式，才能正常被 LLaMA-Factory 识别，避免报错。

## 一、正确的 dataset_info.json 配置（直接复制）

打开 `~/LLaMA-Factory/data/dataset_info.json`，在文件内部的 JSON 数组中，添加以下配置（放在最前面或任意空白位置均可，确保逗号格式正确）：

```json
  "uncensor": {
    "file_name": "uncensor/harmful.jsonl",
    "columns": {
      "prompt": "question",
      "query": "",
      "response": "answer",
      "history": ""
    },
    "format": "jsonl"
  }
```

`harmful.jsonl` 是对话形式的数据，可以用于 Finetuning 的训练模式.

否则，必须使用Pre-training的模式进行训练。

## 二、如果是 parquet 文件（用于Pre-training的格式）

```json
  "chinese-noval": {
    "file_name": "chinese_novel/all_novel_combined.jsonl",
    "columns": {
        "prompt": "content"
      },
      "format": "json"
  }
```
