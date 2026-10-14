# Qwen 多专家训练与模型合并

状态：方案与提交目录初始化。尚未训练、合并或评测；没有模型权重、实测分数或最终报告。本仓库骨架不满足完整实验验收。

公开仓库：https://github.com/SpoonShawn/qwen36-expert-merging

## 任务依据与版本冲突

依据用户提供的七张题目截图，执行多专家训练、模型合并和四项评测的路线。
模板：https://github.com/apexin-ai/qwen36-35b-takehome
参考快照：994e2c329c5ce3f6beaff788f5df98d686442147（核对日期 2026-10-13）。原始 README 保存在 docs/upstream-template-README.md。

当前上游 README 与截图存在实质差异：

| 项目 | 用户截图 | 当前模板 README |
|---|---|---|
| 训练路线 | 多专家训练并合并成单模型 | SFT 后继续第二阶段优化 |
| 模型大小 | 算力不足允许同系列约 7B/8B 并披露 | 不接受缩小模型替代 |
| 代码仓库 | 自建独立仓库 | 面试方分配的私有仓库 |
| 权重 | 各专家及最终合并模型公开 | SFT 和第二阶段模型公开 |

用户明确要求创建公开 GitHub 仓库。本初始化按截图组织；正式实验前需向面试方确认实际适用版本，不能宣称同时满足两个版本。尚未联系面试方。

## 研究问题与建议路线

研究问题：相同基座和训练 token 总预算下，领域专家合并是否比统一混合数据微调获得更好的跨任务表现？

建议先考虑三个外部微调专家（与基座内部 MoE 专家不同）：

| 专家 | 数据方向 | 目标评测 |
|---|---|---|
| code | 仓库级定位、代码修复、单元测试 | SWE-bench Pro v2-hard |
| tools | Shell、环境配置、工具执行轨迹 | Terminal-Bench 2.0 |
| reasoning | 学科推理及经确认的 ASI 任务类型 | HLE、ASI-Bench |

ASI 的任务类型、协议与第三个专家划分需先核对指定版本，再冻结；不能仅凭名称假定内容。
基座首选 Qwen/Qwen3.6-35B-A3B。是否采用截图允许的 Qwen3-8B 备选，取决于硬件、预算和题目版本确认，尚未决定。
采用同一基座 revision、tokenizer、chat template 和一致 adapter 结构；优先评估 LoRA/QLoRA 的可行性。目标模块必须检查实际模型结构后填写，不能直接套用其他 Qwen 模型配置。

## 实验矩阵

| 对象 | 训练 | 正式评测范围 |
|---|---|---|
| Base | 无 | 四项全量 |
| code 专家 | 代码数据 | 至少 SWE 全量 |
| tools 专家 | 工具轨迹 | 至少 Terminal 全量、五次 |
| reasoning 专家 | 推理数据 | 至少 HLE 和 ASI 全量 |
| pooled 对照 | 混合全部专家数据；总 token 预算匹配 | 建议四项全量 |
| 合并候选 A/B | delta 加权平均 / TIES | 独立开发集比较 |
| 冻结最终合并模型 | 不再依据测试反馈选择 | 四项全量 |

合并方案比较在独立开发集完成，冻结后再正式评测。不得使用测试结果挑选合并系数、checkpoint 或专家数据。
建议以完整权重增量 ΔW=sBA 的加权合并为基线，再比较 TIES 的裁剪与符号冲突处理。分别平均 LoRA 的 A/B 矩阵通常不等于平均 ΔW；必须写明实际实现与缩放。合并 QLoRA 时核对反量化、导出精度和重新量化路径。PEFT 官方方法说明：https://huggingface.co/docs/peft/developer_guides/model_merging 。具体架构兼容性尚未实测。

## 数据与污染控制

- 每条数据保留来源 URL、仓库与 commit、许可证、生成方法、哈希及所属 split。
- 按仓库/问题族分组划分训练与独立开发集；按文本、代码及任务来源进行精确去重与近似去重。
- 排除评测答案、参考补丁、隐藏测试及测试反馈；评测资产与训练流水线分离。
- 报告各专家样本数、token 数、难度层次、比例和清洗前后规模。
- 公开 issue/commit 不等于自动允许再分发；逐来源记录许可。

## 评测协议

四项固定为 SWE-bench Pro v2-hard、HLE、ASI-Bench、Terminal-Bench 2.0；参考资料中的 LiveCodeBench 不替代这四项。
训练前锁定数据 URL、revision/commit、split、完整任务 ID 与校验值、评测器及容器版本、主指标、提示、工具权限、上下文、预算、thinking、preserve_thinking、量化和种子。同一 benchmark 所有模型完全一致。

- SWE：需取得指定 v2-hard 完整任务集与 hard 划分依据，不能用 SWE-bench Verified 或其他 Pro 子集替代。
- Terminal：按截图指定 Harbor/Terminus-2，五组固定种子，报告平均成功率和标准差。题面参数为 3 小时超时、32 CPU、48 GB RAM、temperature=1、top_p=0.95、top_k=20、max_tokens=80K、256K context；这些是待核验的题面配置，不代表已实测或官方完整披露。
- HLE：明确是否包括图像题、工具、答案抽取规则和评分器；text-only 与多模态分开报告。
- ASI：锁定指定官方版本、任务清单、运行模式、主指标及聚合方式。
- 保留环境失败、超时和错误，固定分母和处理规则，不能剔除失败后提高成绩。

宏平均提升 = 四项百分制主指标之差的等权平均；目标 +3 个百分点，+5 个百分点为优秀参考，不是保证或官方门槛。使用相同任务配对 bootstrap，并保留五次运行的任务分组关系与重复运行波动。

## 资源评估与阻塞项

35B 是总参数规模，A3B 是每次激活量，不能按 3B 估计权重显存。仅按题面约 35B 参数估算，BF16 权重约 70 GB、4-bit 原始载荷约 17.5 GB；均为十进制理论值，不含量化元数据、激活、KV cache、梯度、优化器或框架开销，不能据此承诺可运行。
正式预算需先测短程训练与推理吞吐，再按训练 token 和评测任务数估算。长上下文与 Agent 工具评测可能成为主要成本。
若 Terminal 有 N 个任务，每个模型需 5N 次任务运行；按题面上限是 15N 个任务小时，串行三对象（Base、目标专家、最终合并）为 45N 个任务小时的上限，不等于实际 GPU 小时或预计耗时。并发会增加 CPU、RAM 和推理服务需求。

当前待确定：GPU 型号/数量/显存、可用时长与费用上限、截止日期、HF 账号、实际题目版本、SWE v2-hard 获取方式、HLE 模态协议、ASI 版本与主指标。训练软件版本尚未选择，不生成虚假的依赖锁定文件。

## 提交目录

configs/ 保存待冻结配置；data/ 保存数据说明与清单；src/ 与 scripts/ 后续实现流水线；logs/ 保存完整运行日志；curves/source/ 保存曲线源数据；results/{base,experts,merged}/ 保存逐任务结果；analysis/ 保存失败分析与置信区间；models/ 保存公开模型链接与 revision。

详见 [验收清单](docs/acceptance-checklist.md)。真实运行后还需 requirements.lock、可执行命令与不超过一页的 report.pdf。目前没有可运行训练脚本，不将空目录标为实现完成。
