---
layout: post
title: RTX 5070 Ti 12G + WSL2 Ubuntu 24.04 搭建 LLaMA-Factory 训练小说AI完整教程（避坑版）
date: 2026-05-17 09:22 +0800
category: [AI, Finetuning]
tags: [LLaMA Factory, WSL2]
---

前言：本文记录了从零搭建 LLaMA-Factory 环境，最终成功训练 Qwen2-7B-Instruct 小说模型的全过程，包含完整可重用步骤、环境细节、所有易错点及报错解决方案，适配 RTX 5070 Ti 12G 显存、WSL2 Ubuntu 24.04 系统，新手可直接照搬，避免踩坑。

## 一、环境准备（必看！可重用核心细节）

环境是成功的基础，所有细节必须严格匹配，否则会出现依赖冲突、显存不足、启动失败等问题，以下是经过实测的可用环境配置：

### 1. 硬件配置

- 显卡：RTX 5070 Ti 12G（核心，显存必须≥12G，否则无法加载 4-bit 量化的 7B 模型）

- CPU：无特殊要求（建议≥4核，加速模型下载和训练）

- 内存：≥16G（避免训练时内存溢出）

- 存储：≥50G 空闲空间（用于存放模型、数据集和虚拟环境，Qwen2-7B-Instruct 模型约10G）

### 2. 系统配置

- 宿主系统：Windows 10/11（需开启 WSL2）

- WSL2 系统：Ubuntu 24.04（实测兼容，其他版本可能出现依赖版本不匹配问题）

- NVIDIA 驱动：Windows 端安装 572.90 版本（对应 CUDA 12.8，无需在 Ubuntu 内安装 CUDA Toolkit 和 nvcc）

- WSL2 内核版本：≥5.15（确保 GPU 虚拟化正常，可通过 `uname -r` 查看，低于此版本需更新 WSL2）

### 3. 核心依赖版本（关键！避免冲突）

所有依赖版本经过严格匹配，禁止随意升级/降级，否则会出现兼容性问题：

- Python：3.12（WSL2 Ubuntu 24.04 自带，无需额外安装）

- PyTorch：2.10.0（适配 CUDA 12.8，自带 CUDA 运行库，无需系统安装 CUDA）

- LLaMA-Factory：0.9.5.dev0（实测可用版本，新版启动方式不同，旧版依赖冲突多）

- bitsandbytes：≥0.39.0（4-bit 量化必需，否则无法加载量化模型）

- 其他依赖：由 LLaMA-Factory 官方命令自动安装，无需手动干预

## 二、完整可行步骤（从零到训练完成，可直接复制运行）

所有命令均在 WSL2 Ubuntu 24.04 终端执行，全程以普通用户权限操作（无需 sudo，避免权限问题），每一步顺序不可乱。

### 步骤1：检查 WSL2 GPU 兼容性（必做）

确保 WSL2 能正常识别 NVIDIA 显卡，执行以下命令：

```bash
nvidia-smi
```

正常输出应包含：`NVIDIA-SMI 570.137 Driver Version: 572.90 CUDA Version: 12.8`，说明 GPU 虚拟化成功，无需额外配置。

### 步骤2：删除旧环境（若有，避免冲突）

如果之前尝试过搭建环境，先删除旧虚拟环境，确保环境干净：

```bash
# 安装依赖
sudo apt update && sudo apt install -y python3-pip git python3.12-venv

# 若当前在虚拟环境中，先退出
deactivate
# 删除旧的虚拟环境（假设旧环境名为 llm，路径 ~/llm）
rm -rf ~/llm
```

### 步骤3：创建并激活新的虚拟环境

```bash
# 创建虚拟环境（路径 ~/llm，名称 llm）
python3 -m venv ~/llm
# 激活虚拟环境（每次操作前都需执行）
source ~/llm/bin/activate
```

激活成功后，终端前缀会显示 `(llm)`，表示当前在虚拟环境中。

### 步骤4：安装正确版本的 PyTorch（适配 CUDA 12.8）

直接安装带 CUDA 12.8 的 PyTorch 2.10.0，无需手动安装 CUDA：

```bash
pip install torch==2.10.0 torchvision==0.25.0 torchaudio==2.10.0 --index-url https://download.pytorch.org/whl/cu128
```

安装完成后，可执行 `python -c "import torch; print(torch.cuda.is_available())"`，输出 `True` 表示 PyTorch 能正常调用 GPU。

### 步骤5：克隆 LLaMA-Factory 项目

```bash
# 进入用户主目录
cd ~
# 克隆项目（确保网络通畅，若克隆失败可更换国内源）
git clone https://github.com/hiyouga/LLaMA-Factory.git
```

### 步骤6：安装 LLaMA-Factory 官方依赖（避免版本冲突）

进入项目目录，使用官方命令安装依赖，自动匹配所有正确版本：

```bash
cd ~/LLaMA-Factory
# 安装核心依赖 + webui + metrics + modelscope（缺一不可）
pip install -e ".[webui,metrics,modelscope]"
```

安装过程中若出现警告，可忽略（只要不报错即可）。

### 步骤7：安装 4-bit 量化必需依赖（关键易错点）

LLaMA-Factory 4-bit 量化依赖 bitsandbytes，必须手动安装指定版本：

```bash
pip install bitsandbytes>=0.39.0
```

### 步骤8：启动 LLaMA-Factory WebUI（正确启动方式）

重点：新版 LLaMA-Factory 启动方式已变更，旧命令（如 `python src/train.py webui`）会报错，需使用以下正确命令：

```bash
cd ~/LLaMA-Factory
# 唯一正确的启动命令（适配当前项目结构）
python -m llamafactory.cli webui
```

启动成功后，终端会显示：`* Running on local URL:  http://0.0.0.0:7860`，此时忽略 `gio: Operation not supported`（仅为自动弹浏览器失败，不影响使用）。

### 步骤9：WebUI 配置（适配 12G 显存，小说训练专用）

打开 Windows 浏览器（Chrome/Edge），输入 `http://localhost:7860`，进入 WebUI 后，按以下配置（无需修改其他参数，避免报错）：

1. Model（模型配置）：
        

    - Model name or path：手动输入 `Qwen/Qwen2-7B-Instruct`（下拉无此选项，手动输入即可识别）

    - Template：手动选择 `qwen`

    - Quantization：选择 `4-bit`（必选，12G 显存唯一能稳定运行的量化方式）

2. Finetuning（微调配置）：
       

    - Finetuning type：选择 `LoRA`（仅训练部分参数，省显存、速度快）

    - LoRA rank：保持默认 `8`，无需修改

3. Data（数据集配置）：
        

    - Dataset：先选择内置的 `identity`（用于测试环境，确保能正常训练，避免自定义数据集报错）

4. Training（训练参数）：
        

    - Num Epochs：设置为 `1~2`（足够测试，过多会导致训练时间过长）

    - 其他参数保持默认（无需修改 batch size、learning rate 等，避免显存溢出）

### 步骤10：开始训练并验证结果

1. 点击 WebUI 底部的 `Start Training`，开始训练；

2. 训练过程中，终端会显示 loss 变化（loss 逐步下降即为正常）；

3. 训练完成后，终端会显示 `Training completed`，并将模型保存至 `saves/Qwen2-7B-Instruct/lora/...` 目录；

4. 验证：修改 WebUI 的Checkpoint path指向上面保存的模型，然后进入 `Chat` 页面，点击按钮 `Load module`。模型加载成功后，输入小说相关指令（如“写一段玄幻小说开头”），模型能正常生成内容，即为训练成功。

### 步骤11：导入自定义小说数据集（可选，正式训练用）

若需用自己的小说内容训练，按以下步骤操作（避免报错）：

1. 新建数据集文件：在 `~/LLaMA-Factory/data/` 目录下，新建 `novel_data.json`，内容如下（可修改为自己的小说内容）：
        `[
    {
        "instruction": "你是一位玄幻小说作者，请根据设定续写剧情",
        "input": "主角林峰意外获得上古玉佩，觉醒先天灵体",
        "output": "林峰掌心温热流转，那枚古朴玉佩融入经脉之中，刹那间周身灵气如潮水般涌入体内。沉寂多年的先天灵体骤然解封，经脉拓宽，丹田生云，周遭山林草木皆为之俯首，天地灵气疯狂朝他汇聚而来。"
    },
    {
        "instruction": "请以古风文笔描写宗门大殿场景",
        "input": "青云宗宗主大殿，长老齐聚",
        "output": "青云主峰之巅，琼楼玉宇凌空而立，大殿琉璃覆顶，仙鹤盘旋于檐角。殿内诸位列坐两侧，白袍长老神色肃穆，仙雾缭绕其间，钟磬余音袅袅，一派大宗气象。"
    },
    {
        "instruction": "帮我构思小说人物性格与背景",
        "input": "女配角，清冷剑修",
        "output": "自幼孤苦无依，被宗门长老收养，一心向剑，性情清冷寡言，不喜与人交好。剑法凌厉决绝，外冷内热，看似疏离冷漠，实则重情重义，背负师门旧怨，独行于江湖修行。"
    }
]`

2. 配置数据集：编辑 `~/LLaMA-Factory/data/dataset_info.json`，在文件中添加一行（确保格式正确，逗号分隔）：
        `"novel_data": {
  "file_name": "novel_data.json"
}`

3. 重启 WebUI，在 Dataset 下拉列表中选择 `novel_data`，点击 `Start Training`，即可用自己的小说内容训练模型。

### 步骤12：模型训练后推理（核心步骤，实现小说生成）

训练完成后，无需额外配置，直接在 WebUI 中即可实现小说生成、续写、人设构思等功能，步骤如下（可直接复用）：

1. 加载训练好的模型：


    - 保持 WebUI 正常运行，若已关闭，重新执行启动命令 `cd ~/LLaMA-Factory ; python -m llamafactory.cli webui`；

    - 进入 WebUI 顶部的 `Chat` 页面；

    - 页面顶部的 `Checkpoint path` 指向训练好的模型（路径为 `saves/Qwen2-7B-Instruct/lora/...`），然后点击 `Load module` 加载模型；

    - 确认模型加载成功。

2. 推理参数配置（适配小说生成，无需修改核心参数）：
        

    - Temperature：保持默认 `0.7`（控制生成内容的随机性，0.7 兼顾流畅度和多样性，适合小说创作）；

    - Top P：保持默认 `0.8`（控制生成内容的相关性，避免偏离主题）；

    - Max New Tokens：设置为 `512`（单次生成的最大字符数，足够生成一段小说片段或完整开头）；

    - 其他参数保持默认，无需修改（避免生成内容错乱）。

3. 小说生成实操（3种常用场景，可直接复制指令）：
        

    - 场景1：小说开头生成，输入指令：`帮我写一段玄幻小说开头，主角是林峰，设定为废柴逆袭，意外获得上古传承`，点击 `Generate`，模型会生成符合设定的开头片段；

    - 场景2：剧情续写，输入指令：`续写剧情：林峰获得上古传承后，回到宗门，发现宗门被仇家偷袭，长老们重伤倒地`，模型会延续前文逻辑，续写合理剧情；

    - 场景3：人设+文风定制，输入指令：`以古风仙侠文风，描写主角苏清寒，清冷孤傲，是青玄宗最年轻的剑尊，擅长御剑飞行，背景是宗门恩怨`，模型会生成符合人设和文风的人物描写及相关剧情片段。

4. 推理结果优化（可选）：
        

    - 若生成内容偏离预期，可调整 Temperature（调低至 0.5 更贴合指令，调高至 0.8 更具创造性）；

    - 若生成内容过短，可增大 Max New Tokens（如调整为 1024）；

    - 若想让文风更贴合自己的小说，可增加自定义数据集的训练轮数（调整为 2~3 轮），重新训练后再推理。

5. 模型保存与复用：
        

    - 训练后的模型会自动保存至 `saves/Qwen2-7B-Instruct/lora/` 目录，无需手动备份；

## 三、易错点总结（高频踩坑，必看！）

以下是搭建和训练过程中最容易出错的步骤，以及对应的解决方案，避免重复踩坑：

### 易错点1：启动 WebUI 报错“ValueError: Please provide `model_name_or_path`”

- 原因：使用了旧版启动命令（如 `python src/train.py webui`），新版 LLaMA-Factory 已变更启动方式；

- 解决方案：必须使用命令 `python -m llamafactory.cli webui`，且确保在 LLaMA-Factory 项目目录下执行。

### 易错点2：启动 WebUI 报错“command not found”或“No such file or directory”

- 原因：项目结构不匹配，误判启动文件路径（如寻找 webui.py 但实际启动入口为 cli.py）；

- 解决方案：无需寻找单独的 webui.py，直接使用 `python -m llamafactory.cli webui`，适配所有新版 LLaMA-Factory 项目结构。

### 易错点3：训练时报错“PackageNotFoundError: No package metadata was found for bitsandbytes”

- 原因：忘记安装 bitsandbytes，或安装版本低于 0.39.0，4-bit 量化无法生效；

- 解决方案：执行 `pip install bitsandbytes>=0.39.0`，确保版本达标。

### 易错点4：自定义数据集报错“ValueError: Undefined dataset xxx in dataset_info.json”

- 原因：只添加了数据集 JSON 文件，未在 dataset_info.json 中配置，系统无法识别；

- 解决方案：严格按照步骤11，在 dataset_info.json 中添加数据集配置，确保 file_name 与 JSON 文件名一致。

### 易错点5：训练时显存溢出（OOM error）

- 原因：未开启 4-bit 量化，或选择了全量微调（Full Finetune），12G 显存无法承载；

- 解决方案：
        

    - Quantization 必须选择 4-bit；

    - Finetuning type 必须选择 LoRA，不可选择 Full Finetune；

    - Num Epochs 设为 1~2，避免训练数据过多导致显存占用过高。

### 易错点6：依赖冲突（如 accelerate、datasets、peft 版本不兼容）

- 原因：手动安装了其他依赖（如 unsloth），或随意升级了依赖版本，与 LLaMA-Factory 要求的版本冲突；

- 解决方案：
        

    - 删除旧环境，重建干净的虚拟环境；

    - 不额外手动安装其他库；

    - 禁止执行 `pip install -U pip` 或升级任何已安装的依赖。

### 易错点7：浏览器无法访问 http://localhost:7860

- 原因：WSL2 端口未映射，或 WebUI 未正常启动；

- 解决方案：
        

    - 确认终端显示 `Running on local URL:  http://0.0.0.0:7860`，说明 WebUI 已启动；

    - 浏览器必须在 Windows 端打开，输入 `http://localhost:7860`（不可在 WSL2 内打开浏览器）；

    - 若仍无法访问，重启 WSL2（Windows 终端执行 `wsl --shutdown`，再重新打开 WSL2）。

### 易错点8：模型下载缓慢或失败

- 原因：网络问题，或未配置 Hugging Face 镜像；

- 解决方案：
        

    - 配置 Hugging Face 国内镜像，执行命令：
                `export HF_ENDPOINT=https://hf-mirror.com`

    - 若仍下载失败，手动下载模型文件，解压至 `~/.cache/huggingface/hub/models--Qwen--Qwen2-7B-Instruct/` 目录。

### 易错点9：推理时模型无法加载或生成内容错乱

- 原因：训练未正常完成、模型保存路径错误，或推理参数设置不合理；

- 解决方案：


    - 确认终端显示 `Training completed`，且 `saves` 目录下有完整的模型文件；

    - 恢复推理参数默认值，避免 Temperature 过高（超过 0.9）或 Max New Tokens 过小（低于 256）。

## 四、附录

### 附录1：WSL2 满血性能配置（32G内存+i9专用）

- 编辑WSL配置文件

```bash
notepad ~/.wslconfig
```

- 粘贴下方配置内容

```ini
[wsl2]
memory=28GB
processors=24
swap=32GB
localhostForwarding=true
guiApplications=true
```

- 保存并关闭文件。

- 返回Windows终端执行重启命令生效

```powershell
wsl --shutdown
```

- 重新打开WSL即可启用满配性能。

- 验证是否生效，进入 WSL2 终端输入：

```bash
free -h
nproc
```

## 五、总结

本文基于 RTX 5070 Ti 12G + WSL2 Ubuntu 24.04 环境，完整梳理了 LLaMA-Factory 从环境搭建、模型训练、自定义数据集到推理生成的全流程，所有步骤均经过实测可重用，重点规避了依赖冲突、启动失败、显存溢出、推理异常等高频问题。

核心要点：环境细节必须严格匹配、启动命令必须正确、4-bit 量化和 LoRA 微调是 12G 显存的关键、自定义数据集需配置 dataset_info.json、推理参数无需过度调整即可实现流畅的小说生成。按照本文步骤操作，新手也能顺利搭建环境，训练出属于自己的小说 AI 模型，实现小说开头、剧情续写、人设构思等功能。
