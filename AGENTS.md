# AGENTS.md

## 项目概览

本仓库是 ACL 2025 论文 **A Joint Optimization Framework for Enhancing Efficiency of Tool Utilization in LLM Agents** 的代码实现。项目目标是优化 LLM Agent 的工具使用上下文，包括 Agent 指令和工具描述，从而在尽量保持任务有效性的同时减少不必要的工具调用，提高工具使用效率。

整体实验基于 StableToolBench/ToolBench 工作流，主要包含训练集推理、上下文联合优化、测试集推理、答案格式转换，以及 Pass Rate 和 Cost-Aware Pass Rate（CAPR）评估。

## 核心目录

- `optimization/`：项目的联合优化框架。`joint_optimization.py` 是优化入口，`optimizer.py` 负责反馈生成、改进建议生成和工具描述优化，`LLM.py` 封装 OpenAI 函数调用式优化引擎，`optimization_prompt.*` 是优化提示词和函数配置。
- `new_metrics/`：新增评估指标目录。`CAPR.py` 实现 Cost-Aware Pass Rate，用工具调用成本约束来衡量方法的效率和成功率。
- `solvable_queries/`：实验查询数据。包含训练集、Tool Test Set、Agent Test Set，以及对应的 query id 文件。
- `toolbench/`：ToolBench/StableToolBench 相关代码。包括推理管线、CoT/DFS 搜索算法、工具环境封装、模型适配和 ToolEval 自动评估代码。
- `asset/`：README 中使用的论文动机图和优化框架图。

## 主要运行流程

1. 安装依赖并准备环境。
2. 按 StableToolBench 要求下载数据、部署工具服务，并配置 OpenAI/ToolBench key。
3. 运行训练集推理，生成优化所需 trial：
   ```sh
   bash inference_chatgpt_training.sh
   ```
4. 运行联合优化框架，生成优化后的工具描述和任务描述：
   ```sh
   bash run_optimization.sh
   ```
5. 在 Tool Test Set 和 Agent Test Set 上运行测试推理：
   ```sh
   bash inference_chatgpt_testing.sh
   ```
6. 将原始答案转换成 ToolEval 需要的格式：
   ```sh
   bash run_convert_answer.sh
   ```
7. 运行 Pass Rate 评估：
   ```sh
   bash run_pass_rate.sh
   ```
8. 运行 CAPR 评估：
   ```sh
   bash run_capr.sh
   ```

## 关键脚本

- `inference_chatgpt_training.sh`：在训练集 `solvable_queries/training_instruction/trainset.json` 上运行 `toolbench/inference/qa_pipeline.py`，并将结果转换为优化输入格式。
- `run_optimization.sh`：调用 `python -m optimization.joint_optimization`，默认使用 `gpt-4o-mini` 对训练集结果做上下文优化。
- `inference_chatgpt_testing.sh`：在 Tool Test Set 和 Agent Test Set 上分别运行 `CoT@5` 与 `DFS_woFilter_w2`，并使用优化后的文本。
- `run_convert_answer.sh`：把 `data/answer/...` 下的原始推理结果转换到 `data/model_predictions_converted/...`。
- `run_pass_rate.sh`：进入 `toolbench/tooleval`，调用 `eval_pass_rate.py`，用 OpenAI evaluator 评估答案是否解决问题。
- `run_capr.sh`：调用 `new_metrics.CAPR`，基于 Pass Rate 结果和工具调用成本计算 CAPR。

## 依赖与环境

- Python 依赖集中在 `requirements.txt`，包含 PyTorch、Transformers、OpenAI、FastAPI、LangChain、ToolEval 相关库等。
- 需要准备 OpenAI key、ToolBench key，以及 StableToolBench 数据和工具服务。
- 根目录的 `openai_key.json` 用于 OpenAI API 池配置，但属于敏感文件，不应提交或复制到文档中。
- 推理脚本默认使用虚拟工具服务：
  ```sh
  SERVICE_URL="http://localhost:8080/virtual"
  ```
- 多个脚本是 Bash 脚本；在 Windows 环境下通常需要 Git Bash、WSL 或等价的 Bash 运行环境。

## 开发注意事项

- 不要把真实 API key、组织 ID、账号密码或服务凭据写入代码、README、AGENTS.md 或提交记录。
- `.gitignore` 已忽略 `openai_key.json`、虚拟环境、缓存和常见构建产物；新增敏感配置时也应加入忽略列表。
- 运行实验会生成 `data/`、`chatGPT35_optimization/`、`pass_rate_results/` 等输出目录；提交前确认这些是否是需要版本管理的源码或结果。
- `optimization/optimizer.py` 会读写优化结果文件，默认路径与 `run_optimization.sh` 中的 `--save_path`、`--training_data_path` 强相关，改路径前要同步检查脚本。
- `toolbench/inference/Downstream_tasks/rapidapi.py` 中的 `using_optimized_text` 逻辑会按 `optimization_method` 和 `optimization_iteration` 查找优化后的文本文件，测试前应确认文件名匹配。
- `toolbench/tooleval/README_ZH.md` 当前存在编码显示异常；需要中文说明时优先参考根目录 README 和实际脚本。

## 给后续 Agent 的建议

- 开始修改前先读 `README.md`、相关运行脚本和目标模块，不要只依赖论文描述推断代码行为。
- 涉及实验复现时，先确认 `data/`、`server/tools`、`chatGPT35_optimization/` 是否已存在且路径与脚本一致。
- 涉及模型或 API 调用时，先检查环境变量和脚本中的 `OPENAI_KEY`、`OPENAI_API_BASE`、`TOOLBENCH_KEY`、`SERVICE_URL` 配置，不要硬编码密钥。
- 修改推理方法时重点看 `toolbench/inference/qa_pipeline.py`、`toolbench/inference/Downstream_tasks/rapidapi.py`、`toolbench/inference/Algorithms/`。
- 修改优化流程时重点看 `optimization/joint_optimization.py`、`optimization/optimizer.py`、`optimization/LLM.py` 和 `optimization/optimization_prompt.*`。
- 修改评估逻辑时重点看 `toolbench/tooleval/` 和 `new_metrics/CAPR.py`。
- 提交前用 `git diff` 检查是否误加入了生成结果、缓存文件或敏感配置。
