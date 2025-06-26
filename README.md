# 🚀 MoE-4x72B-MergeKit: 面向领域专家融合的模块化 MoE 方法

## 📘 项目简介

本项目探索了使用 [MergeKit](https://github.com/arcee-ai/mergekit) 构建 **混合专家模型（Mixture-of-Experts, MoE）** 的高效范式。我们将 4 个基于 Qwen2.5-72B-Instruct 微调的领域专精模型合并为一个总参数量约 **247B** 的 [MoE-4x72B-MergeKit](https://huggingface.co/wenge-research/MoE-4x72B-mergekit) 模型，合并后的模型在基础模型对应的领域任务中均展现出良好性能，验证了基于提示引导的 MoE 融合方案在**快速、可插拔式**领域能力扩展场景中的实用价值。

## 🧠 背景与目标

随着大模型能力的发展，实际应用中对**领域专长能力的可控组合**提出了更高要求。本项目旨在验证以下方向的可行性：

- 无需重新训练，是否能融合多个同源的领域专家模型？
- 基于 Prompt 引导，是否能构建领域明确的专家路由？
- MoE 架构能否实现按需扩展/替换领域能力？
- 合并后的模型是否能在各自领域保持甚至超越原始模型性能？

## 🧩 领域专家模型

本项目中涉及的 4 个领域专家模型属于 Qwen2.5-72B-Instruct 的不同微调版本，详情如下：

| 专家名称 | 能力方向 | 模型路径 |
|----------|----------|------------------|
| 📐 数学专家 | 解题、推理 | [Qwen2.5-Math-72B-Instruct](https://huggingface.co/Qwen/Qwen2.5-Math-72B-Instruct) |
| 🧬 医学专家 | 问诊、医学知识 | [Qwen2.5-Aloe-Beta-72B](https://huggingface.co/HPAI-BSC/Qwen2.5-Aloe-Beta-72B) |
| 🧾 业务专家 | 自我认知、内部业务场景 | - |
| 🔧 共享专家 | 通用能力 | [Qwen2.5-72B-Instruct](https://huggingface.co/Qwen/Qwen2.5-72B-Instruct) |


## ⚙️ MoE 融合原理

本项目使用 MergeKit 的 `mergekit-moe` 工具构建 MoE 模型，所有专家均通过 `positive_prompts` / `negative_prompts` 进行能力域控制，确保领域间的分工明确，避免路由混淆。方案具备如下特性：

- 🔀 **Token 级专家路由**：模型根据每个输入 token 的内容，动态选择最合适的专家模型。
- 🧠 **Prompt 引导领域分工**：通过设置正/负样例提示（prompts）进行领域专家能力划分。
- 🧩 **共享专家机制**：引入 `residual_scale` 控制的通用专家，提高跨任务处理能力。
- ⚡ **轻量高效推理**：设置 `experts_per_token=1`，每个 token 仅激活一个专家，显著降低计算资源需求。

> MergeKit 的 MoE 合并无需对专家模型进行再训练，适合在多模型体系中构建模块化组合。详见 [官方文档](https://github.com/arcee-ai/mergekit/blob/main/docs/moe.md)。

## 📁 模型结构配置示例

以下为用于模型合并的配置文件简要示意：

```yaml
base_model: Qwen2.5-72B-Instruct
architecture: qwen
gate_mode: hidden
dtype: bfloat16
experts_per_token: 1

experts:
  - source_model: Qwen2.5-Math-72B-Instruct
    positive_prompts:
      - "你是一个高中数学老师"
  - source_model: Qwen2.5-Aloe-Beta-72B
    positive_prompts:
      - "你是心脑血管专家"
  - source_model: <Expert for self-identity and internal task handling>
    positive_prompts:
      - "你是谁"
      - <internal task>

shared_experts:
  - source_model: Qwen2.5-72B-Instruct
    positive_prompts:
      - "你是一个AI代码助手"
    residual_scale: 0.1
```

完整配置请见 [`merge_moe.yaml`](./configs/merge_moe.yaml)。

## 📊 模型性能评测

我们使用内部评测脚本，将 MoE-4x72B-MergeKit 模型与各个基础专家模型在下列任务中进行了对比：学科知识（MMLU）、语言理解（CLUEWSC）、阅读理解（DROP）、知识问答（OpenBookQA）、代码生成（HumanEval/MBPP）、逻辑推理（BBH）、数学（CMATH/APE210K）。


| 模型 | Qwen2.5-72B-Instruct | Qwen2.5-Math-72B-Instruct | Qwen2.5-Aloe-Beta-72B | MoE-4x72B-MergeKit |
|------|----------|----------|----------|-----------|
| MMLU | 83.47 | - | - | 81.01 |
| CLUEWSC | 85.59 | - | - | 87.39 |
| DROP | 66.80 | - | - | 67.06 |
| CLUE_C3 | 97.41 | - | - | 96.88 |
| OpenBookQA | 92.40 | - | -| 94.20 |
| HumanEval | 87.20 | - | - | 85.15 |
| MBPP | 79.00 | - | - | 78.40 |
| BBH | 80.00 | - | 45.25 | 87.40 |
| CMATH | 81.17 | 94.30 | - | 88.17 |
| APE210K | 77.30 | - | - | 77.80 |
| MedQA | 77.93 | - | 85.94 | 82.78 |



从上表中的评测结果可以看出，MoE-4x72B-MergeKit 模型成功融合了 Qwen2.5-Math-72B-Instruct 数学专家模型的数学能力与 Qwen2.5-Aloe-Beta-72B 医学专家模型的医学能力，同时在通用任务上保持了与共享专家 Qwen2.5-72B-Instruct 同等的性能。

## 📖 引用

如果您在研究中使用了本项目，请参考以下方式引用：

```bibtex
@misc{MoE-4x72B-mergekit,
  title={MoE-4x72B-MergeKit: A Modular MoE Approach for Domain Expert Fusion},
  author={wenge-research},
  year={2025},
  url={https://github.com/wenge-research/MoE-4x72B-mergekit}
}
```