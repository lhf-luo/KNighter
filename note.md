# KNighter：LLM 合成静态分析 Checker（全文精读）

> AI 辅助研究草稿。本文按“问题 → 方法 → 实验 → 结论”整理；研究笔记是工作材料，不等同于经过人工复核的 Wiki 知识。

## 论文身份与证据边界

- **研究方式**：全文研究。
- **论文**：Chenyuan Yang、Zijie Zhao、Zichen Xie、Haoyu Li、Lingming Zhang，_KNighter: Transforming Static Analysis with LLM-Synthesized Checkers_，SOSP 2025，DOI `10.1145/3731569.3764827`，arXiv `2503.09002`。[论文，物理 PDF 第 1 页]
- **个人库**：namespace `default`；paper ID `doi-8cced08dc9a7f7ca6fd2`。
- **证据 PDF**：英文正式出版 PDF，`D:\paper-agent\paper-agent\.paper-agent\files\personal\default\doi-8cced08dc9a7f7ca6fd2\KNighter Transforming Static Analysis with LLM-Synthesized Checkers.pdf`；SHA-256 `3d9c46badac33d61be56df4764d198f0869df2e4d7945038210043e4851f8f1b`；共 15 个物理页。
- **附件说明**：当前会话另附 `[zh-CN-dual]` 双语派生 PDF；技术 claim 与页码均以英文正式 PDF 为证据源，不把翻译文本当作独立证据。
- **MinerU 覆盖**：来源 SHA 与上述正式 PDF 一致；已按物理页完整读取第 1–15 页，一次截断调用已沿 cursor 续读完成，无缺页。主要检查了 Figure 1–3、5–8、Figure 9 各统计图、Figure 10、Table 1–3 和相关代码示例。
- **原 PDF 核验**：定点核验第 1–14 页的方法、关键数字、表格、局限与结论；第 15 页为参考文献。未对每张图做原 PDF 像素级裁剪核验。
- **paper_progress**：MinerU 全页覆盖完成；原 PDF 文本核验第 1–14 页；Artifact 已发现、获取并检查。进度工具所称 `full.md traversal: incomplete` 是因为采用全页覆盖而非 Markdown cursor 全遍历，不构成正文缺页。
- **Artifact**：官方仓库 <https://github.com/ise-uiuc/KNighter>，已安全浅克隆并固定到 commit `f4e834b3074104001c31c5c262ab11e28d1a8e38`；工作树 clean。本次未运行代码、未安装依赖、未初始化子模块，也未做 `path:line` 级论文—代码映射，因此正文中的方法与实验结论来自论文，不据仓库推定实现一致性。

## 1. 研究问题、重要性与价值

论文面对静态分析中的双重约束：传统静态分析可扩展到大型代码库，但通常需要专家为每种 bug 手写 checker；LLM 能从代码与补丁中识别多样模式，却不适合逐段扫描 Linux 内核，因为上下文、调用成本与幻觉都会成为瓶颈。[论文，物理 PDF 第 1–2 页，§1、Figure 1]

KNighter 的核心研究问题是：**能否让 LLM 只承担一次性的“理解历史补丁并编写分析器”，再由传统静态分析引擎低成本、可重复地扫描整个代码库？** 系统从 Linux 历史修复补丁中提炼 bug pattern、规划并生成 Clang Static Analyzer（CSA）checker，随后在补丁前后版本上验证，并用全内核报告 triage/refinement 降低误报。[论文，第 2、4 页，§1、§3、Figure 3]

最重要的现实结果是：生成的 checker 共发现 92 个此前未报告的 Linux kernel bug，其中 77 个获开发者确认、57 个已修复、15 个仍 pending、30 个获得 CVE；这些 bug 平均潜伏 4.3 年。[论文，第 9–10 页，§5.2、Table 2、Figure 9]

## 2. 之前如何解决，以及缺口在哪里

1. **手写规则/模型**：CRIX、Goshawk、UBITect、CRed、DCUAF、SUTURE 等针对特定 bug 类型建立规则、状态机或污点模型，运行可靠但开发成本高，新 pattern 需要专家继续编码。[论文，第 13 页，§7.1]
2. **偏差式规格推断**：将多数用法视为正确规范并标记偏离；可降低部分人工成本，但受固定模板、聚类假设和误报影响。[论文，第 13 页，§7.1]
3. **从补丁推断规格**：APHP、Seal 等从历史补丁推断接口规格，但通常仍依赖既有静态分析基础设施执行这些规格。[论文，第 13 页，§7.1]
4. **LLM 辅助静态分析**：IRIS、Artemis、LLift 等让 LLM 补充污点规格、路径约束或资源处理意图，但 analyzer 核心仍主要由人开发。[论文，第 13–14 页，§7.2]
5. **LLM 直接分析代码**：灵活，但面对整个 Linux 内核会受到上下文与推理成本限制。[论文，第 14 页，§7.2]

KNighter 的差异是让 LLM 同时生成 bug pattern、检测计划和完整 checker，并用真实补丁及全库报告形成验证—修订闭环。[论文，第 4–7 页，§3]

边界：论文承认同期 MoCQ 也研究 checker/query synthesis，因此新颖性主要落在“从具体历史补丁自动归纳细粒度 pattern，并生成 CSA checker、执行补丁差分验证和误报 refinement”这一组合上。[论文，第 14 页，§7.2]

## 3. [推断] 重建作者可能的思考路径

以下均为依据论文设计反推，不代表作者自述：

1. 若分析单位是“每段待查代码”，LLM 成本随代码库规模增长；若改成“每种历史 bug pattern”，昂贵推理只发生在 checker 合成阶段。[依赖：论文第 2、4 页]
2. Bug-fix commit 同时提供 buggy 版本、patched 版本、diff 和开发者说明，可充当弱监督输入与验证 oracle。[依赖：论文第 5–6 页]
3. 直接端到端生成复杂 CSA C++ 太难，因此先提取 pattern，再合成 plan，最后实现 checker。[依赖：论文第 5–6、12 页]
4. 可编译不等于语义正确；编译错误可驱动 syntax repair，而语义错误要靠 pre-patch/post-patch 差分过滤。[依赖：论文第 5–7、9 页]
5. 通过原补丁仍可能只是记住局部现象或过度泛化，因此还要全内核扫描、报告 triage 与 checker refinement。[依赖：论文第 6–8 页]
6. 最终产品应是可持续复用的程序，而非一次 LLM 判断；checker 生成后可反复扫描新版本。[依赖：论文第 4 页]

## 4. 核心 intuition

**让历史补丁给 LLM 提供领域知识，再把这种知识“编译”为传统静态分析器能够高效执行的 checker。** LLM 负责灵活归纳与程序生成，CSA 负责路径敏感符号执行和规模化扫描。[论文，第 2–6 页，Figure 2–6]

以 `devm_kzalloc` 为例：补丁显示其返回值在解引用前需要判空；pattern agent 将其概括为“可能返回 NULL，未检查即解引用”；plan agent 决定用 CSA `ProgramState` 记录内存区域；生成 checker 在调用后标记区域、识别判空分支、在解引用处告警并追踪别名；它在 pre-patch 代码上告警、在 patched 代码上减少告警，并进一步发现 CVE-2024-50103。[论文，第 3–6 页，§2.2–§3.1、Figure 2–6]

该设计的上限由 LLM 与 CSA 共同决定：LLM 可能归纳或实现错误；CSA 也可能因跨过程、别名、内联、并发和复杂时序而看不到关键状态。

## 5. 具体方法与实现

### 5.1 输入与 pattern 提取

每个任务输入一个 Linux bug-fix commit，包括 commit message、diff，以及被修改函数的完整代码，以补足 diff 缺失的上下文。[论文，第 5 页，§3.1.1、Figure 4]

作者倾向于提取较窄、补丁特定的模式，而不是“检查所有可能返回 NULL 的函数”一类宽泛规则，因为宽规则更难准确实现并容易产生误报。[论文，第 5 页，§3.1.1]

### 5.2 Checker synthesis

流水线为：

1. **Bug pattern analysis**：从补丁与完整函数提取根因；
2. **Plan synthesis**：规划 `ProgramState`、CSA callbacks 与辅助函数；
3. **Checker implementation**：基于预定义 CSA 模板、few-shot 示例和 utility signatures 生成 C++ checker；
4. **Syntax repair**：把编译错误反馈给调试 agent，默认最多修复 5 次；
5. **Patch validation**：比较 checker 在 pre-patch 与 post-patch 对象文件上的报告数；
6. 若失败则重新 synthesis；实验中每个 commit 最多 10 个 synthesis attempts。[论文，第 5–7、9 页，Algorithm 1、§3.1、§5.1.1]

### 5.3 Checker refinement

通过 patch validation 的 checker 会扫描完整内核。系统提取 warning 的相关代码行和 trace path，由 triage agent 依据原补丁和目标 pattern 判断 bug/not-a-bug；若出现误报，refinement agent 修改 checker。新版本必须不再报告已识别误报，同时继续通过原补丁验证。[论文，第 6–8 页，§3.2、Figure 7]

refinement 阶段每个 checker 最多运行 1 小时或生成 100 个 warnings；固定随机抽取 5 个 warnings 进行 triage；最多 refinement 3 轮。[论文，第 8 页，§4]

### 5.4 Valid 与 plausible 的区别

- **Valid checker**：在原补丁局部能区分 buggy 与 patched 版本。
- **Plausible checker**：valid，且全内核报告量较小，或抽样报告中的误报看起来可接受。[论文，第 4、7–8 页]

[推断] `plausible` 是部署候选而非正确性证明，与自动程序修复中“通过测试但可能仍不正确”的情形类似。

### 5.5 实现脚手架

系统建立在 CSA 之上。作者人工制作 3 个端到端 few-shot 示例，约耗费 40 人时，并实现 9 个 CSA utility functions。[论文，第 7 页，§4]

[推断] 因此“自动化”是指脚手架完成后的逐补丁流水线，不是零人工启动。

## 6. 核心数学与理论背景

论文没有给出 soundness/completeness 定理；其形式化核心是工程接受条件。

### 6.1 Validity 条件

对 checker `c` 与补丁 `p`，令 `N_buggy` 为补丁前相关对象上的报告数，`N_patched` 为补丁后报告数，`T_valid = 50`。接受条件为：

```text
N_buggy > N_patched  且  N_patched < T_valid
```

[论文，第 7 页，§4]

这只证明补丁后报告变少且未超过阈值，不证明报告必然对应目标 bug，也不证明 checker sound 或 complete。

### 6.2 Plausibility 条件

令全内核报告数为 `R(c)`，默认 `T_plausible = 20`。若 `R(c) < 20`，或固定随机抽取的 5 个报告中 triage agent 判定的误报不超过 1 个，则 checker 成为 plausible。[论文，第 7–8 页，§4]

[推断] 这是工程启发式而非统计置信保证：5 个样本很少，而且标签来自也可能出错的 LLM triage agent。

### 6.3 CSA 理论基础

CSA 使用 path-sensitive symbolic execution；每个 `ExplodedNode` 由 `ProgramPoint` 和抽象 `ProgramState` 构成。checker 通过 callbacks 响应调用、分支、内存访问和赋值事件，并扩展 ProgramState 保存自定义状态。[论文，第 3 页，§2.1]

KNighter 并未提出新的符号执行算法；其主要创新在于从补丁生成上述状态与 callback 逻辑。

## 7. 实验如何验证 claim

| Challenge / claim              | 实验问题与对照                                          | 数据、预算、指标                                                         | 结果与证据定位                                                                                                                    | 未排除的解释                                                         |
| ------------------------------ | ------------------------------------------------------- | ------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| 能从多类补丁生成 checker       | 对 10 类 Linux bug-fix commits 运行合成与 refinement    | 61 个 commits；每 commit 最多 10 attempts                                | 39 个产生 valid checker；26 个直接 plausible，11 个 refinement 后 plausible，即 37/61，约 60.7%。[论文，第 7–9 页，Table 1、§5.1] | 22/61 无 valid checker；集合代表性与跨运行方差未报告。               |
| checker 不只是简单 AST pattern | 分析 valid checker 的实现特征                           | 39 个 valid checkers                                                     | 37 个 path-sensitive、13 个 region-sensitive、16 个使用高级状态追踪；平均 125.7 LOC。[论文，第 8 页，§5.1.1]                      | 复杂度不等于语义正确；分类由作者完成。                               |
| refinement 能降低误报          | 对初始不 plausible 的 valid checker 迭代修订            | 13 个 checker；最多 3 轮                                                 | 11 个成功成为 plausible，成功率 84.6%。[论文，第 9 页，§5.1.2]                                                                    | 未系统验证 refinement 是否引入未见 false negatives。                 |
| 能发现现实漏洞                 | 在 Linux v6.9–v6.15 上运行最终 checkers，并与开发者交互 | `allyesconfig`；正式 bug scan 不受 refinement 的 1 小时/100 warning 限制 | 92 个新 bug；77 confirmed、57 fixed、15 pending、30 CVE；平均潜伏 4.3 年。[论文，第 9–10 页，Table 2、Figure 9]                   | confirmed 的统一判定标准及全部逐项证据未在正文完整展开。             |
| 与 Smatch 互补                 | 对同一内核运行 Smatch                                   | Smatch 产生 1970 errors、2870 warnings                                   | 没有命中 KNighter 的 true-positive bugs。[论文，第 11 页，§5.3]                                                                   | Smatch 精确版本、规则和命令行未报告；无重叠不能证明整体优于 Smatch。 |
| 多阶段合成优于单阶段           | 在 20 个 commits 的消融集上移除 multi-stage             | 每类随机 2 个，seed 0                                                    | 默认 12 个 valid；去掉 multi-stage 仅 8 个；syntax errors 从 28 增至 52。[论文，第 12 页，Table 3]                                | 仅一次随机样本，未报告跨 seed 方差。                                 |
| CSA 知识影响模型表现           | 比较四个模型                                            | 同一 20-commit 消融集                                                    | O3-mini 12 valid，GPT-4o 11，DeepSeek-R1 11，Gemini-2-flash 4；Gemini 产生 130 次 syntax errors。[论文，第 12 页，Table 3]        | 模型 snapshot、采样参数和预算公平性未充分报告。                      |

### 7.1 最终报告精度

37 个 plausible checkers 中有 16 个没有发现 bug。其余 checker 经 triage 后共有 90 个报告被标为 bug，人工确认 61 个，因此 precision 为 `61/90 ≈ 67.8%`，false-positive rate 为 `29/90 = 32.2%`。[论文，第 9 页，§5.1.2]

[推断] 摘要所称“high-precision”应谨慎理解：约三分之一候选仍为误报，更稳妥的说法是达到可供人工审查的候选质量，而非低误报、无人值守告警。

### 7.2 Triage agent

在抽样的 79 个报告上，triage agent 得到 7 TP、22 FP、50 TN、0 FN。[论文，第 11 页，§5.4.1]

[推断] 0 FN 对安全筛选有利，但真实正例仅 7 个，不足以稳健证明在新 checker 或其他 bug 类别上仍近乎零漏报。

## 8. Takeaways

1. **较可靠结论**：把 LLM 放在 checker synthesis 阶段，而非直接扫描每段代码，是一个可扩展且有现实产出的架构选择。[论文，第 2、9–10、14 页]
2. **最强证据**：77 个开发者确认、57 个修复和 30 个 CVE，比内部 valid/plausible 指标更能说明现实价值。[论文，第 9–10 页]
3. **条件性结论**：patch-grounded validation 能过滤大量失败生成，但只是弱语义 oracle，不等于 checker 正确性证明。[论文，第 6–7 页；推断]
4. **重要工程事实**：系统仍依赖 CSA、可构建内核环境、3 个手写 few-shot checkers、9 个 utility functions 和人工 bug 分类。[论文，第 7 页]
5. **主要弱项**：复杂状态机、UAF、并发、跨过程语义和内联行为仍会造成实现或检测失败。[论文，第 9、12 页]
6. **尚未验证的外推**：跨 Linux 项目与跨 CSA 后端的泛化是未来设想，不是本文实验结论。[论文，第 12 页，§6]

**暂定判断**：值得保留和继续研究。该工作比“让 LLM 直接判断漏洞”更具工程可扩展性，并有较强真实漏洞证据；但“可靠、高精度、可泛化”只能作为有条件结论。

## 9. 最脆弱的假设

最关键的假设是：**原补丁上的报告数量差分足以近似 checker 的目标语义正确性。** 若补丁恰好重构了控制流、删除代码、改变内联或影响编译产物，即使 checker 捕获的是无关现象，也可能满足 `N_buggy > N_patched`。[论文，第 6–7 页的 validity 条件；推断]

其他脆弱假设：

- 5 个 warnings 可代表总体误报分布；样本过小，难覆盖长尾或子系统差异。[论文，第 8 页；推断]
- triage agent 不会系统性漏掉某类真 bug；当前 0 FN 仅建立在 7 个真实正例上。[论文，第 11 页]
- 历史 patch 中的 pattern 可跨上下文复用；过窄会漏报，过宽会误报。[论文，第 9 页]
- LLM 对 CSA API 与语义足够稳定；22 个无效 commits 中，13 个失败源于 implementation，273 个失败 attempts 中有 207 个语义失败。[论文，第 9 页]
- 单次生成可代表模型能力；模型采样参数和跨运行方差未报告。[未知]
- 无 Smatch 重叠可解释为正交能力；版本、规则和构建配置差异也可能造成无重叠。[推断]
- 模型未通过预训练记忆见过输入 patch 或相关 checker；论文没有 contamination 分析。[未知]

## 10. 一周最小复现实验

> 以下是建议的人工执行计划，尚未运行。

### 目标

验证三个最小 claim：多阶段流水线能生成可编译 checker；checker 能通过 pre/post patch 差分；多阶段方案比直接端到端生成更稳定。

### 计划

1. **第 1 天**：固定官方仓库 commit、Linux/LLVM 环境和模型 snapshot；选择 4 个未被用作 three-shot 示例的 held-out commits。
2. **第 2–3 天**：分别运行默认多阶段与无 multi-stage 版本；每个 commit 重复 3 次，保存 pattern、plan、checker、编译错误、token 与耗时。
3. **第 4 天**：对全部 checker 执行 pre/post patch 验证，保存 `N_buggy`、`N_patched`，不只保存最终 pass/fail。
4. **第 5 天**：对 valid checker 执行最多 1 小时/100-warning 的 refinement scan。
5. **第 6 天**：两名人工独立盲标抽样报告；计算一致性，同时检查 triage agent 是否漏掉正例。
6. **第 7 天**：比较 valid rate、syntax/semantic failures、跨重复稳定性、token、时间与内存。

### 成功与停止条件

- 成功：多个 held-out commits 能重现“可编译 → pre/post 区分 → 全局扫描”的完整链，而不只是脚本成功退出。
- 削弱 claim：同一 commit 的 valid/plausible 结果在重复运行间频繁翻转，或 validity 通过但人工检查显示 checker 捕获的是无关变化。
- 若模型 snapshot、LLVM 或 Linux 配置与论文不同，应报告版本偏差，不能直接判定论文复现失败。

## 11. 反例设计

1. **误导性 commit message**：message 描述一种 bug，diff 实际修复另一原因，测试 pattern agent 是否过度依赖自然语言。
2. **偶然报告下降**：补丁只重构控制流或删除代码，测试 validity 公式是否伪通过。
3. **等价判空写法**：加入 `unlikely(!p)`、`IS_ERR_OR_NULL` 或包装函数，测试状态追踪是否误报。论文 Figure 7 已展示 `unlikely` 的真实失败案例。[论文，第 7 页，Figure 7]
4. **路径上必赋值的 cleanup pointer**：变量表面未初始化，但每条可达清理路径前都赋值，测试 UBI checker 与 triage。Figure 8b 展示类似误报。[论文，第 8–9 页，Figure 8]
5. **跨过程/异步 UAF**：释放与使用位于不同函数、回调或线程，测试 state-machine 与 interprocedural 边界。
6. **调用被编译器内联**：让目标 API 在分析阶段消失，复现论文提到的 `strcpy`/`memset` interception 失败。[论文，第 9 页]
7. **同名 API、不同语义**：在不同子系统使用外观相似但保证条件不同的接口，测试过度泛化。
8. **修复不完整的历史补丁**：输入后来再次修订的 patch，测试“patched version 即真值”的假设。

## 12. 非增量 follow-up idea

### 从“生成 CSA C++”转向“生成可验证的中间分析规格”

让 LLM 生成受限 DSL，而不是直接生成开放式 C++。DSL 描述：

- 事件：call、return、branch、bind、load/store、free；
- 状态变量和转移规则；
- alias/region 条件；
- 报告触发条件；
- 必须满足的正例、反例与不变量。

再把 DSL 编译到 CSA、CodeQL 或 Semgrep 后端，并采用 counterexample-guided synthesis：从补丁生成最小正例和语义保持反例，每次 refinement 检查“不消除历史真例”和“不重新引入旧误报”。

[推断] 这不是简单换模型或增加 prompt，而是把开放式程序生成变成受约束、可验证、可跨后端的规格合成，可能同时缓解虚构 API、refinement 破坏旧行为、后端强绑定和 pre/post 数量 oracle 过弱等问题。

最小判伪实验：在同一 patch 集、同一 LLM 预算下，比较直接 C++ 与 DSL→backend 在 valid rate、跨 seed 稳定性、未见反例 FPR 和跨后端一致性上的差异；若 DSL 在这些指标上无改善且显著限制可表达性，则该方向不成立。

## 复现参数表

| 参数                     | 最终值或状态                                                                              | 类型       | 证据位置               | 置信度与缺口               |
| ------------------------ | ----------------------------------------------------------------------------------------- | ---------- | ---------------------- | -------------------------- |
| 目标分析框架             | Clang Static Analyzer                                                                     | 论文报告   | 第 2–3 页              | 高                         |
| 默认 LLM                 | O3-mini                                                                                   | 论文报告   | 第 8 页                | 高；精确 snapshot 未报告   |
| Linux 合成评估版本       | v6.13                                                                                     | 论文报告   | 第 8 页                | 高                         |
| Bug-finding 版本范围     | v6.9–v6.15                                                                                | 论文报告   | 第 8 页                | 高                         |
| Kernel 配置              | `allyesconfig`                                                                            | 论文报告   | 第 8 页                | 高                         |
| 并行度                   | `-j32`                                                                                    | 论文报告   | 第 8 页                | 高                         |
| 硬件                     | 64 CPU cores、256 GB RAM、4×Nvidia A6000、Ubuntu 20.04.5                                  | 论文报告   | 第 8 页                | 高                         |
| Synthesis 数据           | 61 commits、10 类 bug                                                                     | 论文报告   | 第 7–8 页，Table 1     | 高                         |
| Few-shot 示例            | `3027e7b15b02`、`3948abaa4e2b`、`4575962aeed6`                                            | 论文报告   | 第 7 页                | 高；与评估集是否重叠未明确 |
| Few-shot 人工成本        | 约 40 person-hours                                                                        | 论文报告   | 第 7 页                | 高                         |
| Utility functions        | 9 个                                                                                      | 论文报告   | 第 7 页                | 高；本笔记未做代码定位     |
| 每轮 syntax repair 上限  | 5 次                                                                                      | 论文报告   | 第 5 页，Algorithm 1   | 高                         |
| 每 commit synthesis 上限 | 实验最多 10 attempts                                                                      | 论文报告   | 第 9 页                | 高                         |
| `T_valid`                | 50                                                                                        | 论文报告   | 第 7 页                | 高                         |
| `T_plausible`            | 20                                                                                        | 论文报告   | 第 7–8 页              | 高                         |
| Refinement scan          | 1 小时或 100 warnings                                                                     | 论文报告   | 第 8 页                | 高                         |
| Refinement 抽样          | 5 个 warnings，固定随机 seed                                                              | 论文报告   | 第 8 页                | 高；seed 数值未报告        |
| 抽样可接受误报           | 至多 1/5                                                                                  | 论文报告   | 第 8 页                | 高                         |
| Refinement 轮数          | 最多 3 轮                                                                                 | 论文报告   | 第 8 页                | 高                         |
| Ablation 抽样            | 每类 2 个，共 20 个，seed 0                                                               | 论文报告   | 第 12 页               | 高                         |
| RAG 知识库               | 118 个官方 CSA checkers；`text-embedding-ada-002`；检索 3 个示例                          | 论文报告   | 第 12 页               | 高                         |
| 正式 bug scan 限制       | 不采用 refinement 的 1 小时/100-warning 限制                                              | 论文报告   | 第 8 页                | 高；总资源未报告           |
| 官方 Artifact            | <https://github.com/ise-uiuc/KNighter>，commit `f4e834b3074104001c31c5c262ab11e28d1a8e38` | 已获取材料 | 本地 artifact manifest | 高；未运行、未做代码映射   |

## 仍然未知的问题

- [未知] 各模型的精确 snapshot、API 日期、temperature、top-p 和随机 seed。
- [未知] LLVM/Clang、编译器及 Smatch 的精确 commit/version；Smatch 启用规则与命令行。
- [未知] “developer confirmed”的统一判定规则，以及 92 个 bug 的完整逐项证据清单是否全部包含在 Artifact 中。
- [未知] 合成结果的跨运行方差；论文主要报告单次结果。
- [未知] 无约束正式 bug scan 的总 CPU 时间、峰值内存、每 checker 成本及 LLM token/费用。
- [未知] 3 个 few-shot 示例是否从 61 个评估 commits 中排除；正文没有明确集合关系。
- [未知] LLM 预训练是否见过输入 patch、CSA checker 或相关修复。
- [未知] 跨 Linux 之外项目和跨 CSA 后端的实际成功率。
- [未知] refinement 消除已知误报后，是否在其他路径引入新的 false negatives。
- [未知] 官方仓库 commit 与论文最终实验环境、模型服务及全部配置是否完全一致；本次未做 `path:line` 级复现审计。

## 人工备注

<!-- 留给用户；AI 不代填人工意见，也不覆盖已有人工内容。 -->
