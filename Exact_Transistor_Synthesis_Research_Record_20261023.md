# 自主研究记录：面向顶会论文的 EDA 与算法研究

> GitHub归档说明：本文记录本条研究线的已有工作、暂停归档及恢复后的探索；工件相对路径用于定位实验归档，原始数据、第三方源码、求解器和商业资料不随本文上传。`<EXPERIMENT_ROOT>` 与 `<LOCAL_ARCHIVE_ROOT>` 是路径占位符。本记录独立于仓库其他选题报告，不改写其他研究线的判断。

开始时间：2026-10-22；2026-10-23按用户要求恢复探索。本文是唯一完整研究记录：同时记录主线的价值判断、相关工作、问题与挑战、方法假设、实施细节、原始实验位置、结果与失败，以及并行调查的备选方向。第12节保留暂停归档时的历史状态，第13节起记录恢复后的研究。

## 0. 当前状态与执行约束

- **2026-10-23目标升级：形成一篇面向顶会投稿、至少包含三个实质创新点的完整研究成果。用户已恢复探索，目标尚未完成。** 下文第1–10节是筛选阶段，第11节是后续深入研究，第12节是暂停时的判断，第13节是恢复后的实验。优先候选为晶体管网络综合中的共享cofactor资源兼容性；已做小规模结构验证、通用原型及三输入完整NP类回归，仍缺较大实例的强对照、新颖性证据及三个成立的创新点。
- 初始主线是有限总计算预算下的布尔电路局部精确优化（T01）。实测和查重后，普通调度/缓存切口已降级；多约束超图的重顶点精确修复（T04）也完成了小规模反证实验。CPU 树集成推理（T05）的单变量候选未超过强对照，二维网格候选的初步收益也被更简单的阈值等价查询合并超过。已经完成本轮强控制与结果核验，尚未确认原创方法。本文保留每次转向的依据，不能将公开基线的进步写成我们的贡献。
- **全部安装、编译、测试和实验在 .44 执行**：`实验服务器（本文简称 .44）`，工作目录 `<EXPERIMENT_ROOT>`。当前机器只查资料、编辑文档/脚本、下载中转公开资源与连接远程。
- .44 探测：64 个逻辑 CPU，约 125 GiB 内存，探测时可用约 85.7 GiB；`/data` 剩余约 1.5 TiB。系统 Python 3.8.10，另有 `<EXPERIMENT_HOME>/anaconda3/bin/python` 3.12.2，后续建项目独立环境。
- 资源策略：初始最多四个并行 CPU 实验作业，每个 solver 单线程，编译最多四线程；连同子进程总数不得超过用户此前规定的 16。初始实验总内存预算不超过约 32 GiB，按实测调整；不占用 GPU。若以后用 GPU，只用物理 CUDA0、一个实验进程。
- .44 无法直接访问 GitHub/PyPI，采用当前服务器下载公开源码/二进制 wheel，再 SCP 传输；这不涉及在当前服务器运行实验。
- 用户追加授权：需要仿真软件、尤其商业软件时，可在 .24 执行相应仿真/验证；常规优化与算法实验仍在 .44。当前尚未调用商业工具。
- 多代理分工：主代理负责研究判断、实验协调与本文；三个子代理分别承担算法/工件审查、文献查重、独立结果核验。阶段性更换分工，实验并发统一管理。已实际使用多代理并行工作。

当前可读结论：已建立MiniTNtk及L2L的同域对照、完整小函数库和边界回归，并完成冻结的38个NSP代表＋12个EPFL真实电路局部函数评估，共150项性能运行。相同求解器/控制/预算下，两种partition方法均完成50/50，Mini配置完成44/50；但单partition与cofactor消元整体接近，NSP上前者反而更快，因此主要收益应归于表示方式。开发集仅ex275/R11展示明显额外消元收益；新增net-ID强控制又将直接编码从约11.76秒降至6.99秒，cofactor约2.17秒仍只是单例，不能外推。当前模型下ex271_o5、ex275_o1最优均12，ex271_o6已认证下界14、14边上界仍未找到；EPFL新增8边最优例已获独立证书，6个Mini未完成例的已知解均可嵌入其编码。完整及partial背景oracle都无真实端到端收益，已停止。正在审查更强串联长度限制下的接口细分与路径解释，先做正确性及极小验证。**目前有可复核的算法效率改进，尚未确认足以形成顶会论文的原创机制，更未形成三个成立创新点。** 最新细节见第13节。

## 1. 为什么先选这个题目

目标是对一个已经正确的布尔电路做等价替换，减少二输入 AND 节点数或降低逻辑深度。小子电路可以用 SAT 精确综合，大电路靠窗口选择和多轮替换。它接近 MiniTNtk 的研究风格：输入、约束、目标与结果正确性都能精确定义，主要背景是布尔逻辑、DAG、SAT，不需要先掌握器件物理。

应用价值来自现有综合流程仍需在运行时间与实现质量之间取舍。**这个大问题有价值，不等于任意缓存/调参方案都有论文价值。** 首轮需要回答：公开强基线在哪里花时间？它们在相同预算下还留多少可重复的改进空间？这个空间来自算法机制、工程实现，还是仅仅一个错误/默认参数？

采用 [Cirbo](https://github.com/SPbSAT/cirbo)、[ABC](https://github.com/berkeley-abc/abc) 和 [IWLS 2026](https://github.com/alanminko/iwls2026-ls-contest/) 起步，并审查 [eSLIM](https://github.com/fxreichl/eSLIM) 的强基线机制。所有“可能有空间”均暂定为待证伪假设。

## 2. 深入查重后已排除的宽泛想法

| 直觉想法 | 已有直接相关工作 | 对本研究的影响 |
|---|---|---|
| 智能选择 SAT 初始规模、编码和提前退出 | [BESWAC，DATE 2024](https://doi.org/10.23919/DATE58400.2024.10546591) | 不能把调用顺序/动态预算本身说成空白 |
| 在相邻精确综合任务之间增量求解 | IncSyn，GLSVLSI 2024（完整接口和公开工件仍在核查） | 必须比较具体可复用状态与条件，不能只换增量 solver API |
| 缓存已求出的局部最优网络 | DATE 2020 exact DAG-aware rewriting、DAC 2023 MFFW；[mockturtle exact.hpp](https://github.com/lsils/mockturtle/blob/master/include/mockturtle/algorithms/node_resynthesis/exact.hpp) | 成功缓存、规范化及不同窗口缓存已有先例 |
| 失败/超时预算缓存 | 同上 `exact.hpp` 有 blacklist cache，并区分预算与非超时失败 | 不能把 timeout 和 UNSAT 分开存就算新算法 |
| 根据历史 SAT 时间动态调 timeout | [eSLIM subcircuitSynthesiser.py](https://github.com/fxreichl/eSLIM/blob/master/src/subcircuitSynthesiser.py) 的 TimeManager | 必须有动态预算强对照，不能只赢 Cirbo 默认 15 秒 |
| 缩小上下文来计算 don't-care | [eSLIM 原论文](https://d-nb.info/1367524075/34) 已有 enclosing-window 分区与 Boolean relation | 局部上下文近似与关系生成已有体系，需更具体的问题证据 |

这些发现使最初“重复子问题＋缓存＋预算分配”的宽泛提议不能直接立题。接下来先做瓶颈和机会测量；若没有新的机制证据，应记录否定结果，转向备选，而非继续堆搜索次数。

## 3. 问题定义、任务边界与评价

先限定组合电路，输出功能必须完全保持。主成本是 AIG 的二输入 AND 数；NOT 由补边表达，不计为 AND。XAIG 中一个 XOR 不能当成一个 AIG 节点。深度与面积分别记录；以不超过输入电路深度的面积改进为一个固定任务，面积—深度 Pareto 为另一个任务，不事后混用。

区分两条协议：

1. **从真值表综合**：输入公开 truth table，所有方法从独立构建的共同初始电路出发，不能拿测试集的参赛优胜电路初始化。
2. **已有电路后优化**：给定公开 AIG 后再优化，这可以研究，但必须明确初始电路来源及已有强优化程度，不能声称是从真值表得到的成绩。

IWLS 2026 提供 100 个完整多输出真值表及各队结果。VBS 是结果的 Pareto 并集，**不是可重跑的算法**。固定历史 VBS 作质量参照，原始比值允许超过历史最优，但不称为官方原版竞赛分数；最终加入新结果后的 contest-style 评价另列。

数据划分在看到策略成绩前冻结；以整张多输出函数为单位，精确重复/已知等价变体放同组。官方隐藏高层功能描述，不能虚构可靠 family 标签。候选按输入/输出数分层，不能按 VBS 成绩挑简单/难例。测试 truth table 是任务允许的输入，禁止的是利用测试成绩调策略或把已有测试结果作为初始化。开发集、验证集、最终测试集的结果分开报告。

官方 `score_iwls2026.py` 会移动失败/超时文件、重建 VBS，默认并发可能用全部核心，缺例时仍除以 100。因此在副本上受限运行，开发子集另做正确分母的统计；timeout/unknown 不能直接当成行为错误。

## 4. 现有实现的具体问题与挑战

以下为源码静态审查，尚未全部实测。Cirbo 固定参考提交 `ca19a9f29691459563a8a0a90d8f5b6cdc4e5006`。

- `minimize_subcircuits` 默认不做末端全局 validation；本研究必须开启，并另存原始电路用于独立验证。
- `_eval_dont_cares` 枚举全电路的 `2^PI` 个主输入组合并处理全门真值表；cut 很小不代表前处理便宜。首轮限制小输入，分别测 cut、don't-care、CNF、SAT 和替换成本。
- `CircuitFinderSat` 实际默认 XAIG；AIG 主实验显式指定 AIG。`gates_number()` 与 `.size` 含义不同，最终序列化 AIG 后再用 ABC 复读计数。
- `find_circuit(time_limit)` 只限制 SAT 子进程，CNF 构造和前处理不包含在内；0 会成为无限时。必须另设整任务硬超时，记录全流程墙钟。
- 原局部优化路径一次 sweep、每个窗口只试 `size-1`，没有全局深度约束。各策略使用相同的接受条件，不能通过牺牲深度取得表面优势。
- Cirbo 已支持电路数据库，但该 minimizer 默认未传数据库；“默认这条路径未用”不等于“Cirbo 没有缓存”。含 don't-care 的补全查库还可能指数爆炸，非最优库结果不能冒充 UNSAT 证明。
- 全输出为 cut 输入或其反相时存在丢失 NOT 的快路径，已在 .44 用 `NAND(OR(A,B),A)=NOT(A)` 复现。修复统一应用所有方法，修 bug 不计新算法收益。
- 时间与质量都有强次序效应：一次替换会改变后续窗口。不能只离线重排日志就声称等价于真实重新运行。

科学挑战是：正确利用上下文但不让关系生成成本失控；把预算用在可产生全局收益的窗口上；保持精确功能和明确的面积/深度约束；在未见函数上保持收益；解释相对 eSLIM/BESWAC/传统 ABC 的真正差异。

## 5. 分阶段实验方案

本节保留T01最初的设计，用来说明原定验收条件；目前只完成开发集机会测量，切口降级后未启动最终留出集外推。下文T04文件名中的E04属于后续探索编号，不表示T01的最终测试已经完成。

### E00：环境、语义和 checker

锁定源码与依赖、记录二进制哈希。检查 truth table 位序与 AIG 转换；对已知等价与不等价小例子验证 checker；复核上述反相快路径。只把 checker 明确返回等价的正常退出视为通过，超时为 unknown。

### E01：强基线与机会摸底

先用开发集的小实例跑 ABC 固定 portfolio：`strash`、`resyn2`、`dc2; resyn2; dc2`、显式 `resub` 序列、LUT 4/6 映射与 `mfs2` 后重新转 AIG，精确命令以 `scripts/abc_baselines.py` 的 `RECIPES` 和每次 result.json 为准。不同 recipe 的全部运行时间都计费，保留非支配结果。再对同一初始电路跑 Cirbo，并记录各阶段耗时、SAT/UNSAT/timeout、窗口规模、重复率和全局改进。

### E02：简单但必要的对照

同窗口生成、门库、solver、checker、初始化和总预算下，比较原次序、固定随机次序、廉价结构排序以及若干合理固定 timeout 档位。只有超越这些对照，才值得做更复杂的方法。动态预算必须审查 eSLIM 的对应机制，不能拿默认参数作唯一 baseline。

### E03：由测量结果决定的候选方法

暂不预定最终方法。若前处理支配，先诊断上下文关系计算与代价；若 SAT 硬实例支配，诊断编码与无收益证明；若全局接受率低，诊断窗口收益与深度约束；若只有少量重复，放弃缓存主张。可先写一个保守、可独立验证的候选，再做消融；工程加速与算法贡献分别报告。

### E04：稳定性和外推

固定策略后增加时限、种子、输入/输出规模与不同初始电路；最终留出集只用于结论。检查收益是否来自更多资源、弱 baseline、未处理超时、遗漏错误、改变成本定义或测试反馈。没有这些证据前不报告“取得突破”。

## 6. 实验记录与结果

### 2026-10-22：环境与 E00

- 通过 `rrliu` 和端口 2223 连接成功。此前 `bhli` 登录失败是用户名不匹配，不再把它当实验阻塞。
- 直接网络安装失败：.44 对 PyPI/GitHub 连接拒绝/网络不可达。改为离线包和源码中转，保留日志。
- 已建立独立 `.venv`，Python 3.12.2；离线安装 `cirbo==1.1.0`、`python-sat==1.9.dev17` 及依赖。首批实验使用wheel内ABC wrapper，之后另行编译固定提交的独立ABC CLI；两者分别记录，不能把wheel内ABC冒充最新主线提交。
- IWLS 真值表包锁定仓库提交 `91ad493f4b7732fcbf5384c24c18baef3e07a0c2`，压缩包 SHA-256 `b795e284b5911baf84814e4f85ce8e10c3c8a277365967dcb83a5487a244cf72`。100 个实例；按完整函数哈希去重并按输入/输出规模分层、seed 20261022，冻结为 dev 60、validation 20、test 20。官方隐藏来源族，当前分组只能防精确重复，不能保证所有未知函数变体都隔离。
- `results/e00_complement_probe.json`：`NAND(OR(a,b),a)` 原 truth 为 `[1,1,0,0]`。原 Cirbo 开启 validation 时抛 `FailedValidationError`；关闭后返回 `[0,0,1,1]`、输出接到 `a`，独立 SAT 等价检查为 false。因此疑似反相错误已实证；所有后续策略统一使用保护路径并保留原版诊断结果。
- ABC 基线的验证采用另写的 Python 大整数并行 truth evaluator，穷举所有输入组合，但不逐行调用原综合器；检查实际输出 BENCH，不仅看进程退出码。当前两个 8-input 真实实例、六条流程均通过，12/12 等价。`strash` 后输出为 AIG 门库，XOR 若仍以单节点出现会被 checker 拒绝。

初始正确性摸底（开发实例，尚不是新方法结果）：

| 实例 | strash AND/深度 | resyn2 | dc2 组合 | resub 组合 | LUT4 组合 | LUT6 组合 |
|---|---:|---:|---:|---:|---:|---:|
| ex255 | 456 / 16 | 337 / 12 | 323 / 12 | 324 / 12 | 330 / 12 | 355 / 13 |
| ex260 | 364 / 16 | 273 / 11 | 248 / 13 | 266 / 13 | 241 / 11 | 251 / 11 |

这说明 ABC recipe 选择本身会产生大幅差异。后续不能用 `strash` 或单一较弱 recipe 当唯一基线。这里尚未给出同总预算的 portfolio 与候选方法比较，不能宣称新方法改善。

| 实验 | 当前状态 | 原始记录 | 可下结论 |
|---|---|---|---|
| E00 | checker通过；Cirbo反相错误复现并修复，另补常量/IFF兼容 | .44 `results/e00_abc_protocol/`、`results/e00_complement_probe.json` | 原默认路径的错误有实证 |
| E01 | 72个ABC输出等价；Cirbo统一修复后的36次诊断已完成并审计 | `results/e01_abc_dev_small/`、`results/cirbo_repaired_merged_audit.json` | 简单排序/下界没有足够研究收益 |
| E02–E03 | 新版ABC基线24次与边界关系消融24次完成 | `results/e02_abc_eslim_d1_t5_dev_s01/`、`results/e03_abc_eslim_boundary_d1_t5_dev_s01/` | 上下文机制有作用，但已被现有工作覆盖 |
| 原计划E04外推 | 未启动；没有通过开发集筛选的新策略，保留validation/test | 无最终测试成绩 | 不把探索性负结果推广为领域结论 |

### E01 更新：72 个基线输出与一次无效的搜索空间保护

固定小规模开发集的 72/72 个 ABC 输出通过独立完整真值表检查。以下是每个实例六条流程中最小 AND 数的起点，面积并列时取更小深度；这是后优化协议的共同输入，不是新的算法成绩。完整各流程结果和耗时在 `results/e01_abc_dev_small/`。

| 实例 | 输入/输出 | 最佳流程 | AND / 深度 |
|---|---:|---|---:|
| ex255 | 8/8 | dc2 | 323 / 12 |
| ex260 | 8/8 | lut4 | 241 / 11 |
| ex265 | 8/4 | lut4 | 91 / 11 |
| ex270 | 8/16 | resub | 459 / 12 |
| ex275 | 8/4 | lut4 | 59 / 8 |
| ex256 | 10/10 | resub | 1327 / 18 |
| ex266 | 10/5 | dc2 | 280 / 15 |
| ex271 | 10/20 | resub | 1948 / 17 |
| ex276 | 10/5 | dc2 | 224 / 13 |
| ex262 | 12/12 | dc2 | 3961 / 21 |
| ex267 | 12/6 | dc2 | 746 / 20 |
| ex272 | 12/24 | dc2 | 8405 / 20 |

首轮 Cirbo 诊断使用 60 秒总预算、单查询 1 秒、cut 输入上限 5、cut limit 15、窗口最多 9 个门，4 作业并行；原次序、窗口大小排序、输出不相容筛选各一组，共 36 次。结果为 21 次正常结束、6 次墙钟超时保留原始有效电路、9 次入口报错；完成的运行未改进。**这不是有效的负面研究结论**，有两个明确实施问题：

1. 为避开已复现的极性 bug，临时 `alias-policy=skip` 过滤了边界输出与 cut 输入/其他输出相同或反相的窗口。在 ex255 的小诊断中，482 个大小合法的窗口有 374 个被过滤，所有多输出窗口消失；剩下的树形窗口已达到基本支持集门数下界。无改进是保护策略造成的明显偏差。应修复输出映射并恢复窗口，而非继续加大该保护策略的实验规模。
2. 9 次入口报错来自 ex270、ex271、ex272 的合法 AIG 常量 `ALWAYS_TRUE` 被脚本拒绝，不是实例不可综合或算法超时。需要统一支持常量真值、零面积计数及窗口生成。

小诊断 `results/e00_profile_smoke/` 的 108 条查询也不能写成 108 次 SAT：92 条在 CNF 中已有空子句，16 条才实际调用 SAT。该轮总耗时 3.626 秒，窗口生成约 1.112 秒、DC 约 0.251 秒、CNF 约 0.053 秒、真正 SAT solve 约 0.00385 秒。阶段耗时必须区分，不应把 Python 前处理归因于 SAT 难度。

修复方案是所有策略共用正确的极性与输出端口映射、保留逐次全局等价验证，再比较相同窗口空间；原版、保护版和修复版分别留档。另加入经典支持集下界筛选作为普通对照，不当作新贡献。这一阶段开始准备旧eSLIM离线编译，现已完成，下节给出运行结果；其默认CLI是QBF，SAT比较显式指定，且默认不限制全局深度、迭代时限不是硬总超时，协议单独处理。

修复回归已完成：`results/e00_repair_checks_v3/` 的5个极性/多输出/共享常量例子全部通过独立真值表验证；反相与同相trivial例均降为0门，保持输出功能。前两次失败记录保留：v1是补丁变量名笔误，v2发现 `into_bench()` 会把常量展开为1个AND并改变深度；现保留常量原语并重新解析校验。经典支持集下界另外通过6339个部分真值表的独立枚举检查，非新方法。真实ex255修复版30秒smoke恢复988个窗口（未按alias过滤），进入473个查询、实际调用215次solver、无已接受改进。其中988窗口有823个多输出，合法规模860个中695个多输出；搜索空间确已恢复。473个查询只有21个不被两种经典下界解释，真实SAT时间约0.938秒，而find_circuit调用管线累计23.309秒，说明此实例存在明显调用开销；但30秒仍只走到2/3输入窗口，不能称cut5空间耗尽。

12实例×3策略的修复版开发诊断，`results/e01_profiles_repaired_k5_s0/`，2作业并行，各60秒。第一版14次正常结束、13次墙钟超时、9次入口错误；9次来自ex270/271/272仍含0面积的直通门IFF，不能算正常无收益。在其余共同9个实例上，原次序/大小排序/经典支持下界筛选分别合计减3/3/4个AND，累计wall约421.15/420.66/321.03秒，全部保留输出通过独立truth检查。旧harness快照固定为 `artifacts/profile_cirbo_repaired_v3_before_iff.py`，SHA256 `85921ec3f0d4a1cd57c499c4425ee448d130d08fdea115420dbc762c7671f787`。

IFF补齐后，6个针对性回归例全部通过；只重试上述3个实例的9次入口失败，记录在 `results/e01_profiles_repaired_v4_identity_retry/`。合并审计 `results/cirbo_repaired_merged_audit.json` 明确标记使用v3与v4两版harness，v4只修正直通门兼容性。36个保留输出全部通过独立truth检查，无残留入口错误。原次序/大小排序/支持下界筛选在12例上各减3/3/4个AND，累计完整wall为491.074/490.331/390.351秒；正常结束为6/6/8次，其余为6/6/4次硬超时。经典筛选节省工程开销，但收益非常有限，不能据此立项。v4脚本SHA256 `0ad70b301cb8722d600ccb79759a0efb12739e0666e99c3f9225c7226646bc6c`。

### eSLIM 机会测量与更强的 2026 基线

旧版 eSLIM 已在 .44 编译通过，固定提交 `5a230e339cff9e6872fa26bcb3a6804f452a551e`；AIGER 子模块 `9b243f2d93ed9d797b8064d54c863863980c1b2b`，CaDiCaL 子模块 `2df7b7fed0f9c522fd4cdf6e88cecad4cac8a2df`。离线添加 pybind11 2.13.6、bitarray 3.7.2、CMake 3.27.9。日志 `logs/eslim_build.log`。

新增独立 `scripts/aig_check.py`，按 AIGER FORMAT 直接解析 binary/ASCII AIG，以大整数穷举完整输入空间；不使用 ABC/Cirbo/eSLIM 的求值器。72 个 ABC 的实际 `.aig` 全部功能等价，AND 数和深度也全部与 BENCH 记录一致，审计见 `results/e01_abc_dev_small/aig_audit.json`。

两实例 eSLIM 试跑配置：seed=0、固定局部门数6、SAT初始时限1秒、关系生成时限2秒、保留原动态timeout；不开并行windows、不用ABC后处理，外层严格30秒墙钟并保留原子checkpoint。按原规则允许等面积替换，每次成功替换后保存；完整truth验证在优化后执行，耗时单列。它是**纯面积机会测量**，不能与逐次全局验证、拒绝深度增长的 Cirbo 直接算时间优势。

| 实例 | 共同初始 AND/深度 | 30秒内最佳面积点 | 已保存且深度不超初始的最佳点 |
|---|---:|---:|---:|
| ex255 | 323 / 12 | 307 / 16 | 320 / 12 |
| ex260 | 241 / 11 | 224 / 14 | 241 / 11 |

两次均为预期的外层超时，使用截止前完整checkpoint；全部保存候选另经独立完整truth验证。数据在 `results/e00_eslim_smoke/`。这些数值来自已有eSLIM，证明至少存在面积机会，同时显示深度代价不可忽略；不是新方法收益，也不是全局最优结论。保存的depth可行点不等于运行过depth约束搜索。

查重又确认 **DAC 2026《Delay-Constrained Area Optimisation Using SAT-Based Local Improvement》已经直接解决了在 eSLIM 中约束深度的问题**：[官方摘要](https://63dac.conference-program.com/presentation/?id=RESEARCH754&sess=sess172)。固定的 ABC `fba0adf4ccca0be59ddc6e77a85eb0cc64e04a8b` 包含 `&eslim -D 1`，给全局电路深度上限做面积优化，已作为强基线。`-D 0/2/3` 分别为面积、先面积后延迟、延迟目标，不能混比。`-T` 使用软CPU时间限制，单查询可超时；另加严格墙钟与checkpoint。[固定源码](https://github.com/berkeley-abc/abc/tree/fba0adf4ccca0be59ddc6e77a85eb0cc64e04a8b/src/opt/eslim)。ABC首轮编译缺可选readline头文件，按官方支持的 `ABC_USE_NO_READLINE=1` clean重建后通过，失败日志保留。

### 新版 ABC 深度约束基线：24 次实测已完成

ABC clean 重建成功（372秒，首次失败166秒），二进制 SHA256 `1e5530c7486d4760b5a804f2b86f08c41df64549992a967c03c1634e5d6b6708`，仅关闭可选readline，不改算法源码。使用同一12个开发实例的最佳ABC起点，固定 `&eslim -D 1 -T 5 -R 1 -S 6 -Z {0,1} -i`，`-i`关闭额外inprocessing。每次单线程、4GiB虚拟内存上限、外层完整worker硬上限60秒；实际耗时均远小于硬上限。脚本 `scripts/run_abc_eslim.py`，记录 `results/e02_abc_eslim_d1_t5_dev_s01/`。

| 实例 | 初始AND | seed0 | seed1 |
|---|---:|---:|---:|
| ex255 | 323 | 317 | 317 |
| ex260 | 241 | 233 | 233 |
| ex265 | 91 | 75 | 82 |
| ex270 | 459 | 452 | 450 |
| ex275 | 59 | 47 | 54 |
| ex256 | 1327 | 1324 | 1315 |
| ex266 | 280 | 259 | 253 |
| ex271 | 1948 | 1942 | 1944 |
| ex276 | 224 | 200 | 197 |
| ex262 | 3961 | 3956 | 3957 |
| ex267 | 746 | 715 | 723 |
| ex272 | 8405 | 8403 | 8405 |

24/24实际输出AIG通过独立全truth检查与初始depth上限检查，23次AND下降、1次持平。两seed各减少141/134个AND；每实例平均降5.640%/4.291%，但按总门数加权仅降0.781%/0.742%，必须同时报告，不能用小电路百分比掩盖大电路改善很少。每任务完整wall约5.15–6.31秒，ABC累计CPU约124.86秒，最大RSS约29.7MiB。采样核实ABC各只有1线程，未用GPU。

**对选题的影响：** 这是公开新基线的成绩，不能包装成我们的算法。Cirbo慢还可能来自不同问题定义：当前cut最多5输入，而新版eSLIM的6门窗口最多可有10输入；Cirbo在可达cut输入赋值上保持边界输出truth，不可达赋值记为don't-care；eSLIM直接对受影响PO构造Boolean relation，还可利用下游可观测don't-care及输出间关系。搜索空间不同，不能只看秒数就归因于调度。后续仅保留少量机制消融；单纯减少Cirbo进程启动开销或加经典下界，很可能不足以构成新研究方向。

已完成一个现成机制消融：同12个开发输入、两个seed、5秒软CPU时限、原版ABC `-D1 -i`，仅增加 `-C 0`。该选项将比较点收回subcircuit的边界输出，仍保留完整reference TFI的可达性约束，因此不是“关掉所有DC”。它收掉下游ODC和联合输出补偿，但窗口选择、增量SAT、等面积重构等仍与Cirbo不同。同seed也不保证访问同一窗口序列，因为替换和耗时会改变后续轨迹。记录在 `results/e03_abc_eslim_boundary_d1_t5_dev_s01/`，只用来解释机制，不作为新算法。

24/24消融输出同样通过全truth和depth检查。`-C0`在两个seed分别只减少12/12个AND（分别7/12、6/12实例改善），完整关系为141/134。差异支持“上下文关系的优化自由度很重要”的端到端解释；因为窗口序列与每轮耗时会一起变化，不能将差额逐个归因为ODC。现成eSLIM已覆盖该机制，不能把重新实现它作为新贡献。

另有2026学位论文 *A Tool for Iterative Boolean Circuit Optimization Using SAT-Based Local Improvements* 已描述按边界输入/输出数与目标门数分组，共享结构CNF并用selector激活任务；代码未核实，因此不当实测baseline，但也不能把“同形窗口摊销SAT”直接称为空白。[机构记录](https://hephaestus.nup.ac.cy/browse/dateissued?startsWith=2026-03)。当前仍处于机会与强基线测量，没有确认原创方向。

## 7. 并行探索其他方向

### 7.1 e-graph 提取

已发现 [ASPEN，MLCAD 2025](https://www.csl.cornell.edu/~zhiruz/pdfs/aspen-mlcad2025.pdf) 有 PPA 反馈后改 e-node cost 并重复提取的真实流程，但使用快速 greedy/bottom-up，外层综合可能支配时间；未找到可直接回放的公开查询轨迹。Foresight 已有增量树提取，E-Boost 已有剪枝与 warm start，[SCIP 原生 reoptimization](https://www.scipopt.org/doc/html/REOPT.php) 还能复用树与解。因此泛泛“多查询缓存/热启动”降级。仅保留一个待测问题：真实 cost 更新导致的反复剪枝/重建是否显著，并能否超越 solver 原生 reopt；没有轨迹和耗时证据前不开发完整新算法。

### 7.2 张量重计算调度

查重后确认：HiRemate 本身已有子图多预算策略集、representative cluster 复用；固定 peak budget 时 HILP 会复用已 build 的模型扫描 save budget；保存/读取策略也已有接口。Bonsai 涉及分段粒度，BladeDISC++ 涉及动态 shape。原“跨预算复用”宽泛提议不成立，降为备选。

源码补充纠正前次调研：HILP 默认可用免费 `PULP_CBC_CMD`，Gurobi 不是该路径硬依赖；但完整 Rockmate 构造会访问 CUDA，CPU 离线实验必须有导出图，不能声称完整框架无 GPU。

另发现有限求解时间下的具体状态问题：HILP 用 PuLP 的 `LpStatus==1` 标记 Optimal，而 PuLP 2.7 的 CBC 适配器可能将有 incumbent 的 timeout 映射为该状态、另用 `sol_status` 区分；无 incumbent 的 unknown 又可能让预算循环提前返回。**只读发现，尚未复现，不可先称漏洞或性能问题。** 可比官方顺序、仅修状态分类、再改预算顺序三组，记录真实 gap 和有效策略前沿；若收益只来自修状态，应归为工程修正。[HILP 模型](https://github.com/topal-team/rockmate/blob/main/rockmate/src/rockmate/solvers/ilp/ilp_model.py)、[预算循环](https://github.com/topal-team/rockmate/blob/main/rockmate/src/rockmate/solvers/ilp/ilp_solver.py)、[PuLP CBC 状态映射](https://github.com/coin-or/pulp/blob/2.7.0/pulp/apis/coin_api.py)。

### 7.3 多约束超图划分

[ESA 2026 的 Mt-KaHyPar-MC](https://drops.dagstuhl.de/entities/document/10.4230/LIPIcs.ESA.2026.12) 明确承认多维权重和重顶点可能导致单点移动无效、再平衡失败；但公开集可行率已超过 99%。源码还有默认关闭的 bin-packing fallback，TritonPart 也已有群组移动/小规模 ILP。因此“加联合交换/精确修复”本身亦不新。

最初提出的待测问题是（下文已完成有限机会测量）：局部搜索的 cut 收益有多少被昂贵再平衡抵消，有限联合修复能否在同预算下保留更多收益？必须对照默认 fallback、内置 packing、增加 fallback 轮次；先读公开小型结果和 Set V，不下载全 25.9 GB。若只修少数人工难例或仅靠额外时间，则淘汰。该方向现有问题证据比泛化 e-graph 缓存更直接，但仍未验证新意/收益。

#### 公开原始结果与输入审计更新

已取得 [官方 Zenodo](https://zenodo.org/records/19630135) 的 `results_comparison.zip`、`results_fallback.zip`、Set V（80,413,004 B，27个输入，官方MD5匹配）与源码固定包，存于 .44 `mtkahypar_esa2026_static_audit/`。以下是**作者公开机器上的结果，不是 .44 新实验**，ε=.03：

| 输入 | k | 作者单线程5个seed的平衡结果 | 作者中位时间 |
|---|---:|---|---:|
| ISPD98_ibm10，约6.9万点 | 11 | seed0、4失败；其余3次可行 | 2.97秒 |
| ISPD98_ibm12，约7.1万点 | 8 | seed0、2、4失败；其余2次可行 | 2.21秒 |
| superblue16 | 32 | 5次全部失败 | 23.19秒 |

需要纠正“可行率超过99%”的直觉：论文按一个graph,k的5个seed全失败才判该实例失败；例如Set V、ε=.03、默认96线程是18/540次运行不平衡，而五次全失败是2/108个实例。后续同时报告单次成功率和五次至少成功一次，不能混用。[评测定义](https://arxiv.org/html/2605.28333v1#S6.SS1)。CSV `failed=no` 也不能替代实际权重平衡检查，重平衡timer可能嵌套，不能直接求和。

隐藏强基线 `--r-rebalancing-use-binpacking-fallback=true`（初始划分还有 `--i-r-rebalancing-use-binpacking-fallback=true`）已经包含7类装箱策略、每类默认10种顺序、最多8步容量二分并再次重平衡；公开fallback归档只有12种评分变体，没有该packing对照。两种评分变体已让superblue16各有1/5次成功，所以简单换评分也不新。[源码](https://github.com/kahypar/mt-kahypar/blob/135b23da324e6c35b0cde5a9a7db03200272f3b7/mt-kahypar/partition/refinement/rebalancing/binpacking_fallback.cpp)。按默认2%容量阈值，ibm10/ibm12仅41/15个重顶点，先验证现成开关成本很低。

还发现附带统计表与实际输入不同，例如ibm10实际area总和47,534,336，统计表33,366,704；原因未确认。后续须从输入重新算容量并核对默认复现，不能依赖统计表推不可行。两个输入SHA256分别为 `e969acb416646892264306ade2cec296573c64ece9b2ea501829e6e635c4fe5b`、`a1ace23d528264a69710bb8193716ad8c02e5c2d70f05a6b9d45124b7b860645`。

当时保留更窄的假设：重顶点packing忽略连线代价和轻顶点剩余容量，是否造成后续恢复困难或cut恶化？源码把 `other_loss` 存到 `estimated_quality_loss`，但该连线损失未参与 `applyBetterResult` 的候选选择条件；这只是线索，不是收益证据。计划先测默认与原生packing，若仍有失败再加评分变体；实际B全部可行，因此未追加该变体。源码tag esa2026固定为 `135b23da324e6c35b0cde5a9a7db03200272f3b7`；需要CMake≥3.26、C++17、Boost≥1.69、oneTBB≥2021.5，现已在.44构建并完成下述实验。

#### .44 实测：现成packing已经解决这两个小实例的失败

工具已经构建并运行。使用既有Anaconda Boost1.82、oneTBB2022.3和兼容C++运行库，构建仅修改离线依赖定位与 `Threads::Threads` 链接，算法源码未改；可选hwloc关闭。二进制SHA256 `60f498467133eea4e54cfeefa1cf20e5ae4faa8f75715ce73f7ab7068ff7a206`，构建目录 `mtkahypar_esa2026_build/`。一次审计发现项目进程含闲置CMake包装合计18个，超过16进程约束，当时实际忙碌约3个。已改为直接 `exec make -j1`，把构建树降为6进程；后续将包装进程也纳入预算，实验runner直接启动算法并wait4，不叠加time/timeout包装。

在两个实际输入、ε=.03、各seed0–4、单线程下比较A默认与B原生packing，共20次。没有启用初始阶段advanced rebalancer；因此初始packing布尔开关即使打开也不生效，B实际测试的是主阶段packing，不能称为“两阶段均已启用”。每次4GiB地址空间上限、60秒硬超时。输出标签长度/范围、每块三维整数负载、km1均重新从hgr和partition独立计算，与工具自报一致。

| 实例 | A默认合法次数 | B原生packing合法次数 |
|---|---:|---:|
| ibm10 / k11 | 2/5 | 5/5 |
| ibm12 / k8 | 5/5 | 5/5 |
| 总计 | 7/10 | 10/10 |

20次算法累计wall71.48秒，峰值RSS85.82MiB，无超时；实际算法线程数1。当前锁定输入/二进制的结果与作者同seed原值不逐项一致，因此这里只报告自洽的配对比较，不称为精确复现作者表格。两种常见平衡口径（ceil平均与严格平均）在这20个结果上的合法性结论一致。原始记录、完整命令、分区及独立checker在 `mtkahypar_esa2026_baseline/`。

**可行性修复切口降级。** B没有失败，所以没有立即启动更多初始阶段开关组合。质量方面，ibm10仅两组双方合法的同seed对照，km1为8144→8475（+4.06%）、8925→10785（+20.84%）；ibm12五组km1比值平均1.00253。样本很少，且触发修复会改变后续轨迹，不能据此声称原生packing普遍损害质量，更不能把A的不合法解当更优质量基线。

随后执行了以下限定的小机会试验：冻结其余顶点，给原生阈值选出的重顶点做容量受限、考虑incident hyperedge的km1目标的联合整数规划；先在现有合法分区上测试是否能降低连接代价。最多64个活动顶点、单线程、solver10秒、显式模型规模和内存限制。若固定轻顶点已超出某块某维容量，即剩余容量为负，可直接报告该邻域不可行，不等于全图无解；剩余容量恰好为零本身不构成不可行证明。其他可行性由模型判断，timeout无解是unknown。最终用整数负载和独立km1复核。该MILP是普通基线式原型，已有局部ILP/组移动先例，不预设原创性；还须比较相同初始状态的原生再优化和同额外时间预算，才能谈优劣。

#### 重顶点 MILP：实现与正确性

原型为 `scripts/heavy_vertex_repair.py`，SHA256 `1b815219743928cf88ebe8ed5d5c2af96012b3607622fbe274bc28958dabf462`。严格按原生2%容量阈值选活动顶点，不事后取top-k。每个活动顶点分配到一个块，三维剩余容量扣除固定顶点负载；用二值变量精确表示每条相关超边是否占用某块，完整km1保留固定超边和固定端点的常数项。使用既有SciPy1.16.3/HiGHS，1线程，presolve开，目标相对gap为0，solver10秒、进程硬上限60秒、地址空间4GiB。还限制12,000条相关超边、200,000个pin、100,000变量、500,000约束、2,000,000非零系数，超限报告跳过。

初始合法时保留其解作为外部incumbent，只接受独立复核合法且km1严格更小的结果；未向SciPy传warm start，solver的限时解可能比已有解差，此时拒绝。初始不合法时，恢复合法性单独评价，不能用不合法起点的低cut虚构质量优势。HiGHS status2仅表示固定轻顶点的这个邻域不可行；超时无incumbent表示unknown。若本来合法的起点被同一模型判不可行，则作为实现错误中止。

另外做了非镜像式模型检查：3个104顶点、2块、4活动顶点的特造例子，由独立枚举16种活动分配得到真值，覆盖只有活动端点、混合端点、多个固定占用块、固定超边与singleton等情况。合法起点与可修复不合法起点各有6个可行分配，最优km1均为4；第三例剩余容量逐项非负、单个活动点也放得下，但联合分配无解。MILP状态、整数负载、完整目标、常数项和最优值均匹配。3/3通过，记录 `results/heavy_model_regression_v1/regression_report.json`。这验证有限例子的建模语义，不是任意输入正确性的形式证明。

#### 同起点原生 V-cycle 对照

原生CLI没有加载已有分区后继续改进的入口，因此增加薄包装，直接调用原版 `PartitionerFacade::improve`，复用已有算法，不重新随机初始化。每次先保存实际载入标签并比较B起点哈希与独立负载/目标；然后一轮V-cycle、同seed、packing开、单线程、4GiB与60秒进程上限。包装二进制SHA256 `8b620f1ed8e89cc2313aba000a093c2f47533a6fdeecafadbeb65f40f21d2a66`。原生方法允许移动所有顶点，MILP只动重顶点；这是两个方法的能力差异，正是小邻域是否值得继续的比较，不能称为相同搜索空间。

两个seed0先导例中，MILP为ibm10减少2个km1、ibm12不变；原生V-cycle分别减少802、142，且更快。为避免仅凭两个例子判断，已扩展到B的剩余8个起点，并试修A的三个非法分区。另一代理重新读取50份分区文件核验，结果见 `results/e04_paired_audit.json` / `.csv`；10个B配对中两种方法最终保留的输出、以及成功修复的A/seed2输出均满足三维容量，较严格的平均容量口径也通过；另外两个A修复结果分别为该邻域不可行和unknown，不能称为已恢复合法。

| 输入 / seed | B共同起点km1 | MILP最终保留km1 | 原生一轮V-cycle km1 |
|---|---:|---:|---:|
| ibm10 / 0 | 9776 | 9774 | 8974 |
| ibm10 / 1 | 10461 | 10461 | 10458 |
| ibm10 / 2 | 8731 | 8731 | 8695 |
| ibm10 / 3 | 8475 | 8462 | 8467 |
| ibm10 / 4 | 10785 | 10436 | 9562 |
| ibm12 / 0 | 8874 | 8874 | 8732 |
| ibm12 / 1 | 8807 | 8807 | 8766 |
| ibm12 / 2 | 8969 | 8969 | 8941 |
| ibm12 / 3 | 9106 | 9026 | 8842 |
| ibm12 / 4 | 8674 | 8674 | 8630 |
| 合计 | 92658 | 92214（下降0.479%） | 90067（下降2.796%） |

MILP在4/10起点改善，原生在10/10改善；两者直接比较，原生9次更好，MILP只有ibm10/seed3少5个km1。MILP有7次邻域最优、3次限时有解；其中ibm10/seed1给出11194的较差候选，被拒绝后保留10461，不能把该候选写成方法最终退化。

计时口径分别保留：10次MILP建模合计6.927秒、milp调用38.030秒、脚本总56.243秒；10次原生V-cycle优化段9.511秒、进程总11.973秒。MILP两个先导例缺外层进程wall，不补造；其余8对MILP进程总47.538秒、原生9.150秒。MILP计时含独立checker，原生checker在进程外，不能把商值直接称为严格加速比；但仅MILP建模＋求解已明显贵于原生整进程。它也不是相同预算的最终竞赛：MILP最多10秒solver，原生仅一轮。当前证据已足以降低小邻域的优先级，不需先给更强原生方法追加预算才承认负结果。

修复A的三个非法起点：seed0在固定轻顶点邻域内不可行；seed1超时无incumbent，可行性未知；seed2恢复为合法km1=11951，最优性未证。最后一个起点原km1=11566但不合法，不能将+385当合法质量恶化。现成packing已使B全部可行，因此这些修复结果不构成优于现有算法的证据。

**决策与原因：T04重顶点独立精确修复降级。** 邻域太小会把真正需要协同移动的轻顶点锁住；扩大邻域则增加模型代价，并重新进入已有局部ILP、group move与多层refinement覆盖的领域。这是机制上的合理解释，当前未做移动轨迹因果验证。样本只有两个刻意挑选的压力输入，不能声称整个多约束划分方向无空间，但已没有依据继续堆这个原型的种子和时限。

原始运行见 `results/e04_heavy_milp_smoke/`、`results/e04_heavy_milp_development_expansion/`、`mtkahypar_esa2026_vcycle/`、`results/e04_native_vcycle_development_expansion/`。

### 7.4 T05：分组输入的 CPU 树集成推理

**为什么值得初探。** 许多what-if请求在一组样本中只改变少数输入，重复执行整棵树浪费共享计算；目标是保持同一个已训练模型的输出，降低CPU延迟和内存。问题可抽象成树、谓词、位集合与执行计划，无需大GPU或电路物理背景。但适用输入必须由任务语义提供分组，不能把普通独立样本强行拼成有利分组。[TreeWalker论文](https://arxiv.org/html/2610.03939v1)

**基线与公开工件。** [TreeWalker官方仓库](https://github.com/Shopify/treewalker)为MIT许可，有Rust核心、Python实验、数据校验和模型转换；固定main为 `e7fdd9c6c422b91c80c64fdb4d02f1ed04abdb83`，源包SHA256 `7fa05b584d192477c6392ca25a11458ec8a23cbac6fa9762152b2363af12531b`。初步候选是官方Credit what-if任务：每个测试申请人生成G种情景；实际官方pilot有4个可变特征，不是只有额度变化。它是受控情景生成，不是自然独立batch；单变量原型另标为自定义子场景。强对照包括原生LightGBM/XGBoost、TreeWalker以及适用的QuickScorer；完整评估还需tl2cgen/lleaves等编译方案。补充公开跨实现基准可参考 [DFInferBench](https://github.com/asu-cactus/DFInferBench)。

**查重明显收窄了空间。** RapidScorer已有跨树相同predicate合并；TreeWalker也已做全森林predicate去重及共享掩码。因此“发现共享阈值”不是新方法。[RapidScorer原文](https://ai.stanford.edu/~wzou/kdd_rapidscorer.pdf) TreeWalker原论文明确提出可与QuickScorer组合，直接实现作者future work不足以证明新颖性。更关键的是作者2026-10-16公开的v3栈：每树constant-heavy-path trie前缀用动态规划选择，另有机器缓存探测、chunk/lane选择与constant/varying执行阶段分离。不能只和2.x比较。[PR22及算法](https://github.com/Shopify/treewalker/pull/22)、[固定prefix源码](https://github.com/Shopify/treewalker/blob/d8fe928503b5208219745ace06d725b2f576fb67/src/rules/prefix.rs)、[PR21](https://github.com/Shopify/treewalker/pull/21)、[PR24](https://github.com/Shopify/treewalker/pull/24)

v3还未合并；作者报告其候选版在两种架构比2.x更快，但最终x86实现与其计时版本有差异。本文不把作者报值当.44实测；下文已固定实际源码并在兼容工具链重跑。[PR30说明](https://github.com/Shopify/treewalker/pull/30) 更宽泛的“按层/节点混合执行”已有Register Your Forests，自动推理策略选择与布局搜索也有先例。[Register Your Forests](https://arxiv.org/html/2404.06846v1)、[PACT 2023自适应并行](https://research.ibm.com/publications/accelerating-decision-tree-based-inference-through-adaptive-parallelization)、[SilvanForge](https://www.microsoft.com/en-us/research/wp-content/uploads/2024/09/sosp24-final168.pdf)

还需纠正文献/工件前提：除了quickscorer-rs，Google Yggdrasil也公开了QuickScorer实现；其 `quick_scorer_extended` 的实际 `LeafMask=uint64_t` 支持64叶，不能把注释的128节点当128叶，更不能当整个算法家族的极限。[Google源码](https://github.com/google/yggdrasil-decision-forests/blob/main/yggdrasil_decision_forests/serving/decision_forest/quick_scorer_extended.h)

**保留备选、未执行的窄问题。** TreeWalker的bit表示行，QuickScorer的bit表示叶，不能将两者加速比相乘。如果残余子树共享predicate，预计算成本与选中predicate的集合并集有关，逐树独立代价可能误导选择。可研究受内存预算约束的联合执行选择，但新颖性与收益都尚未证实。先做固定规则、按组宽/叶数切换和纯方法best-of；oracle只用来估计潜在空间，绝不能作为可部署方法成绩。

**最初预定的最小实验。** 少量浅/深模型与G=1/16/64等配置，训练用固定seed、1线程，同模型跨执行器；比较逐树叶输出和最终累加语义。计时包含分组/掩码准备、内核切换与输出累加，另列模型加载/索引构建与内存，交替重复计时避免顺序偏差。初始只测Credit；要形成研究证据必须再扩到真实分组任务。若oracle对最快强基线也缺乏约15%的稳定空间，或收益只出现在人为有利分组/漏算转换成本，则停止这个切口。该阈值是立项筛选标准，不是统计显著性的替代。

**实现进度。** 原版2.x核心已经在.44通过全部62项官方检查（25 unit、14 import、9 predictor、8 research、6 doc），编译＋检查wall102.13秒、CPU101.23秒、峰值RSS约380MiB，限定一个CPU核，无GPU。仓库标注Rust1.97.1/Python≥3.14，实际使用现有Rust1.93/Python3.12；保留原Cargo.lock，以 `--ignore-rust-version` 通过核心构建，Python仅复用兼容的官方数据/训练函数和薄驱动。因此属于兼容环境复现，不冒充作者原工具链。

#### 第一轮原版运行结果

已训练一次50树、max_depth4、num_leaves15的Credit LightGBM，learning_rate=.05、seed42、1线程；官方split为24000训练/6000测试，从测试固定取前128个申请人，每人G=1或16个情景。4个可变特征为0-based `[0,11,18,19]`，按训练集模型gain从官方金额特征候选池选择，不使用测试标签；variant0保留原值，其余乘 `2**U[-1,1]`。G之间各自seed42生成，非嵌套切片；28.125%的组至少一个被扰动值原为0，因此保持不变。该比例是输入特性，不删掉这些组来提高收益。

训练0.133秒、Treelite转换0.00889秒，模型加载/索引约0.25–0.33毫秒。计时用Rust直接调用TreeWalker及LightGBM C API，避免用Python调用开销充当算法收益；固定CPU核、单线程、每方法约10毫秒一轮、7轮正反交替顺序。TreeWalker继承作者 `target-cpu=native`，LightGBM用公开4.7.0 CPU wheel，尚非作者重新编译的native二进制；这也是pilot与论文完整复现的差异。下面是**每组ns的中位数**，warm cache、同一进程测量；不是跨进程置信区间，也不是新方法成绩。

| G | TreeWalker 2.x | 同库逐行完整遍历 | LightGBM C FastRowLoop | LightGBM C PredictForMat |
|---|---:|---:|---:|---:|
| 1 | 1046 | 603 | 1143 | 6316 |
| 16 | 3264 | 7554 | 14364 | 17307 |

此处2.x显式 `exact_sums=false` 对齐作者camera-ready设置，不能冒充默认配置；下文已补齐 `exact_sums=true` 及v3强制精确累加对照。全2176行、5种方法（另含作者no-precompute消融）对Treelite GTIL的最大绝对概率误差≤2.23e-16，grouped与同库fullwalk逐bit一致。driver全流程wall0.61秒、RSS约25MiB。结果说明真实分组任务有收益，单行时额外准备反而有成本；它不证明我们的新切口有收益。原始结果 `treewalker_20261022/results/credit_probe.json`，数据/训练参数/文件哈希 `results/credit_nt50_md4/preparation.json`（此路径相对treewalker子目录）。LightGBM原生模型SHA256 `4d4ca52ee8e02c24485a6a78943374ab2db3e5f374220753da69420561f5ff89`，同一模型供后续所有执行器使用。

#### 另一个待验证机制：单变化特征的区间累加

这个候选针对一个更明确的成本：一组情景只改变一个数值特征时，每棵残余树在该变量上是分段常数。可以对查询值排序，把叶子作用表示为查询索引区间，跨树对区间做两端差分，最后仅做一次前缀求和。它有可能减少每棵树给每条查询逐次写输出的成本。TreeWalker分析中的T×G叶累加是该算法的工作量，不能当成问题的不可突破下界。

这是普通区间算法的待测应用，**不是已确认的新算法**。解释性领域已有[FastPD](https://openreview.net/forum?id=JTOOU5SsXT)、[Woodelf++及Full PDP](https://arxiv.org/html/2605.14578v1)、[实现](https://github.com/ron-wettenstein/woodelf)，它们可用单行background表达ICE，不能用“PDP不同任务”简单排除。需要核查是否已有相同的跨树事件累加，并区分最终逐申请人预测与平均PDP输出。

最小原型固定同一Credit模型和申请人，预定G=1/16/64/256：同C++逐行treewalk、区间遍历但逐点累加、跨树差分累加三组。每组计入排序、阈值定位和输出还原，模型转换另列。先验证每个叶的选择和所有输出，不接受未知类别/缺失语义。差分改变浮点累加顺序，必须报告对原生预测的数值误差并用强抵消例子检查风险；不能以容差正确冒充bit-identical。只有同样输入上超过TreeWalker v3等强对照后，才考虑扩大或发展方法。

**最新查重已否定这个核心的新颖性。** v3 PR24的实际代码已把叶row mask分解成连续区间，向差分数组的两个端点加/减，再prefix sum；只有一个连续run还有专门快路径，并用统一定点/宽累加器处理重排求和。单变量排序后每叶正好一个区间，因此“跨树区间差分减少叶写”已基本被覆盖。[PR24](https://github.com/Shopify/treewalker/pull/24)、[固定mask源码](https://github.com/Shopify/treewalker/blob/a99e2a2282976aa2744dc5acc3486c9195e4d40e/src/predict/mask/rows.rs#L141)、[差分实现](https://github.com/Shopify/treewalker/blob/a99e2a2282976aa2744dc5acc3486c9195e4d40e/src/exact/runs.rs)。保留已开始的极小原型作为机会测量；仅剩“直接传区间端点能否省掉bitmask处理”的较窄差异，需要v3在相同排序/还原输入上作对照。原型若改用普通浮点而原生保持精确累加，更不能据速度表宣布优势。

v3完整PR30 head固定为 `c42e942ec976022b5178ba797b43209e89ec39bb`，复用原锁定依赖；Rust1.93的CPUID函数签名与作者1.97不同，原版首次编译在3处报unsafe调用错误。仅对3个相同参数的调用加unsafe块，补丁单存 `treewalker_20261022/scripts/v3_rust193_cpuid_compat.patch`、哈希记录 `results/v3_compat_provenance.json`，标为compat-v3。没有更改算法/缓存选择逻辑。该版本已通过官方默认release检查：128通过、0失败、1个重新记录golden vectors的测试忽略；包含默认1742个property cases。编译＋测试205.78秒，峰值RSS约1.16GiB，单核8、4GiB限制；未运行额外full-run5000case、cross-target或Miri，不能声称作者全部验证流程已复现。同模型性能配对已完成，结果见下文。

源码审查又确认：差分累加并非始于v3，当前2.x的 `exact_sums=true` 路径已有 `for_each_run`→diff；v3增加了后续执行组织优化。因此最初与camera-ready false路径比较，只能作为历史配置对照，不能把已有exact路径的收益算成新机制。补测同官方4变量输入，2.x exact=true在G1/G16约1216/3733ns每组，compat-v3 exact=true约756/3141ns；全2176行对GTIL最大误差≤2.23e-16，v3与自己的exact fullwalk逐bit一致。v3索引约0.88–0.90ms，首次predictor含机器probe约0.181ms，均在内核计时外单列。不同进程、warm cache pilot只作机会判断，不给精确跨版本加速置信区间。

#### 自有单变量原型的实际控制实验

同一Credit模型的前8名申请人×4个分别变化的特征，构造32个条件组；各组只动一个数值变量，G=1/16/64/256为同一256变体的嵌套前缀。**只有8名独立申请人**，尤其G1的四个特征条件是同一个原始输入的重复，不能算32个独立测试样本。编译 `g++ -O3 -std=c++17 -march=native -fno-fast-math -ffp-contract=off`，Intel Xeon Platinum8375C核8，1线程、4GiB限制，实际峰值RSS约7.1MiB。

每个特征条件各9轮、每轮20次×8组，方法顺序轮换，全部原始sample保存。以下为每个特征的中位数再等权平均，单位μs/组：

| G | 自写逐行treewalk | 区间遍历＋逐点累加 | 区间遍历＋普通double差分 |
|---|---:|---:|---:|
| 1 | 0.411 | 0.967 | 0.956 |
| 16 | 6.238 | 1.912 | 1.318 |
| 64 | 24.661 | 4.837 | 2.383 |
| 256 | 99.247 | 23.952 | 14.908 |

计时包含每组内存分配、排序/恢复、阈值upper_bound、固定特征裁剪和sigmoid；模型flatten约0.00538秒另列，共同dense输入materialization和reference校验不在内核时间中。TreeWalker可复用scratch，原型每组新建vector，这会影响小G。G1首个条件的前三轮明显慢于后几轮，可能有热身/频率或调度影响，不能宣称小幅差异有统计意义。**这些倍率只相对自写标量控制；尚不能当成对强基线的提升。** 数据与完整samples在 `results/e05_interval_control/`，脚本 `scripts/export_interval_control.py`、`scripts/interval_control.cpp`。

所有真实输入的逐树leaf ID与LightGBM相同，逐点累加与原生raw分数逐bit一致；普通差分最大raw误差8.89e-16、概率误差1.12e-16。但压力例明确推翻了通用bit-exact保证：包含±1e16抵消、NaN/default、相等阈值及相邻double，区间划分和leaf完全正确，普通差分仍可与逐树累加差2。它不是对任意模型数值安全的直接替换，若继续发展必须实现合适累加语义，不能仅靠当前Credit的小误差宣称正确。

同query的v3强对照已完成16个cell，另加每次调用内排序索引、gather完整23列row、预测、scatter恢复的版本，scratch容量预分配。恢复后与v3原始结果逐bit相同。按4特征等权平均的中位时延（μs/组）为：

| G | v3原始顺序 | v3排序＋恢复（均计费） | 普通double区间原型 |
|---|---:|---:|---:|
| 1 | 0.636 | 0.653 | 0.956 |
| 16 | 1.304 | 1.518 | 1.318 |
| 64 | 2.679 | 3.854 | 2.383 |
| 256 | 8.266 | 12.996 | 14.908 |

显式排序控制在这些cell都更慢，强对照必须取原始v3，不能专挑更慢的排序版。原型在G64约有一成时间差，但测量在不同短进程中、热身状态不同、原型求和语义还弱于v3，所以不构成可靠优势；其余宽度没有收益，G256明显落后。**单变量区间原型不继续扩大。** 原始v3记录在 `treewalker_20261022/results/singlevar_probe_v3_sorted_control_f*.json`；跨方法配对审计另存结果文件。单纯赢过自写逐行baseline的4–10倍没有足够研究意义。

#### T05b：二维 Cartesian 情景网格的限定验证（预先登记）

单变量失败后，只保留一个在实现机制上确有差别的假设：真正二维网格中，固定其他特征后，每个叶对应一个查询矩形。v3把矩形展平为多条一维run；四角二维差分能否减少更新次数，并抵消双前缀和、边界定位的额外成本？二维差分本身是基础算法，不能以应用它就宣称创新。需要先证明明确场景价值和相对强基线的收益，再谈研究贡献。

查重边界：v3官方 `whatif-grid-v1` 确实生成Cartesian网格，但注释明确是 **validation only**，不是作者已发布的性能基准；核心只接收flat row masks、没有二维shape或四角更新。Woodelf++ Joint-PDP、FastPD和TreeHFD也涉及邻近任务，本次检索未找到相同四角推理内核，这不是穷尽性新颖证明。[v3网格生成器](https://github.com/Shopify/treewalker/blob/c42e942ec976022b5178ba797b43209e89ec39bb/experiments/treewalker_exp/workloads.py#L368)、[FastPD实现](https://github.com/jyliuu/glex-rust/blob/main/core/src/fastpd/evaluate.rs)、[TreeHFD](https://treehfd.readthedocs.io/en/latest/)

预定仅复用原50树模型及前8名申请人，G=256/1024/4096，官方网格形状分别64×4、64×16、64×64；量化实际不同levels、叶矩形数、v3 run更新数，保留重复quantile。每个矩形最多64条run，不能说随G无限增长。v3同时测两种轴顺序，取更强对照并披露选择/转置成本，不扩新模型或筛选有利样本。

数值方案先限定可验证域：从float64叶值精确提取共同二进制定点单位，用signed128位整数做四角更新和两遍prefix；保守检查全部叶绝对整数值和乘4小于2^126、缩放不出现下溢/溢出，否则明确不支持。最后正确舍入到double，独立核对 `math.fsum`、v3 exact与全部leaf ID；逐树原生浮点值可作为另一参考，不能混淆它与精确求和。该边界设计还会由另一代理独立检查。

计时区分两个接口：已经有dense矩阵时的内核，以及调用者原本只有base申请人＋两条轴的compact请求。每组axes提取/排序/恢复计费，dense展开、转置、模型加载/机器probe单列；若收益主要来自免去展开，不应归因于四角更新。仍为核8、单线程、4GiB上限、无GPU的小规模机会实验；实测与消融见后文。

实际输入已按固定PR30官方函数生成，仍复用原50树模型；官方grid按模型split-count选择两轴 `[11,0]`（x12账单、x1额度），与前面4变量场景按gain选特征的规则不同。三个宽度实际distinct levels为60×4、60×15、60×34；4096点只有2040种不同特征组合。这些重复保留并报告，若后续主要收益集中于这个宽度，还要排除普通去重＋恢复就能解释收益的可能。数据/模型/逐树oracle在 `treewalker_20261022/results/credit_grid_nt50_md4/`。

实现的核心关系为：叶权重w落在半开索引矩形 `[lx,rx)×[ly,ry)` 时，在差分表四角分别写 `+w,-w,-w,+w`，再沿两轴前缀还原整个raw分数表。固定特征只走一条分支，变化特征用upper_bound切轴区间，空矩形不访问。预期代价是裁剪后的节点访问、轴边界查找和O(G)前缀/输出；排序及dense结构验证另计。若v3实际run很少，双前缀额外工作可能抵消全部收益，不能预设必然更快。

独立设计审查确认整数界与舍入：对每叶每组至多一矩形的有限数值域，所有四角事件绝对值和≤4A，中间prefix也是事件子集的和。手写53位guard/sticky、ties-to-even后进行二进制缩放；s=0避免负移位，检查移位/绝对值前的溢出，零规范+0。varying轴的NaN/inf明确拒绝，固定特征NaN按模型语义处理，不扩成多矩形后继续错误沿用单矩形的界。必须先通过正负halfway、强抵消、共边/单格/重复轴等小例；`math.fsum`只作交叉参考，并非不依平台的唯一数学oracle。[Python说明](https://docs.python.org/3/library/math.html#math.fsum)

#### T05b 第一轮实测、成本计数与必要消融

原型已完成，不再是仅有设计。此模型共同整数单位为 `2^-67`，四倍绝对叶值和只需75位，满足预设的128位安全域。导出器用每个float64叶值的精确有理数表示验证整数编码，独立 `Fraction` 求和后正确舍入形成oracle；三种G、全部叶ID及exact raw输出检查通过。手写舍入的正负halfway、强抵消、指数进位、最小正常数和几何共边/重复轴/固定NaN小例也通过。原生逐树浮点和与exact和的raw最大差约1.34e-15，不能把两种语义写成逐bit相同。模型转换0.0210秒、oracle与导出1.176秒，均不是推理耗时。

每方法先热身至少50毫秒，再校准每轮约10毫秒、轮换7轮；同核8串行进程、4GiB上限。以下单位μs/组，仍只有一个模型、8名独立申请人，warm cache；没有独立进程重复的置信区间：

| G | 二维原型：compact请求 | 二维原型：dense可信结构提取 | v3：canonical A dense强对照 | v3：compact强对照 |
|---|---:|---:|---:|---:|
| 256 | 9.214 | 9.376 | 12.737 | 13.450 |
| 1024 | 27.917 | 28.048 | 31.733 | 38.182 |
| 4096 | 88.232 | 88.372 | 122.779 | 141.797 |

canonical A是官方给定的展开顺序。dense强对照取直接A内核与“计入A→B转置＋B预测＋scatter恢复”的较快者；compact强对照允许两种轴展开并计入恢复。若调用者本就提供B顺序，G256的v3纯内核只有10.680μs，必须披露，不能说所有输入布局的v3都需12.737μs。原型dense提取信任已验证的Cartesian结构，类似v3信任分组常量契约；每调用重验全部G×F条目的防御版本另存，明显更慢，不能漏掉验证成本后宣称任意dense输入都适用。

相对表中canonical dense，观察到的时间比例约0.736/0.884/0.720，即时延下降约26.4%/11.6%/28.0%；不是“下降36%/13%/39%”。compact优势还包含免于展开23列矩阵的收益。原始数据为 `results/e06_rectangle_control/summary.json`，独立配对审计为同目录 `v3_paired_audit.json`，v3原始记录为 `treewalker_20261022/results/credit_grid_probe_v3.json`。

**计数削弱了最初的因果解释。** 8组总计，v3在A顺序的run写入从15763到17284、17923，树遍历步数1901到1916、1925，增长很小；但逐行预计算从4096到16384、65536，排序比较从11144到55096、245232。因此不能把延迟增长简单归于“每矩形很多run”，省掉dense逐行掩码准备更可能是重要来源。这是机制推断，不是已有因果证明。

进一步在**同一C++矩形生成、排序、阈值查找、int128精确语义、compact输入与输出还原**下，只替换累加方法：一种用二维四角＋双prefix；另一种将矩形展开成一维run、两个端点更新＋单prefix，整内轴宽矩形合并成一个run。新进程仍50毫秒热身、7轮交替；结果如下：

| G | 二维四角 μs/组 | 一维run μs/组 | 一维run数（8组） | 二维角写入数（8组） |
|---|---:|---:|---:|---:|
| 256 | 9.003 | 11.596 | 15763 | 2556 |
| 1024 | 29.574 | 30.664 | 17284 | 2604 |
| 4096 | 105.116 | 98.582 | 17923 | 2640 |

全部叶ID与Fraction exact检查再次通过；run数恰与v3的A计数相同。这是共享矩形生成器后的累加消融，不是完整TreeWalker替代实现。大G下四角更新反而较慢，故目前收益不能主要归因于四角算法。不同进程中G4096绝对时间从约88到105μs也显示频率/系统状态漂移，不能拿跨进程小差异当稳定收益。消融记录为 `results/e06_rectangle_ablation/summary.json`，源代码快照保存在同目录。

#### T05b 更强控制：模型阈值等价查询合并

对于当前有限数值轴、所有分裂均为 `x <= threshold` 的模型，令轴编码为 `lower_bound(sorted_unique_thresholds, x)`。编码相同意味着该特征所有分裂谓词相同；两轴编码都相同且其他特征固定，则整模型的叶选择完全相同。可以每个轴只保留一个等价代表，在压缩后的Cartesian网格上调用原v3，再将结果复制到全部原始点。它既消除完全重复值，也消除不同取值但同阈值区间的重复预测。当前驱动依赖官方网格轴本来非降序，已在输入验证中检查；乱序轴须另加排序或全局bin映射，当前计时没有包含这种未声明接口的成本。缺失值/类别/Zero-missing尚不在此证明范围内，不作泛化声称。

该控制已在同一3G/8人上实现：每次调用计入轴提取、分箱、代表行构造、v3调用与scatter，不缓存请求预测；模型阈值表预处理、压缩组宽的索引加载和机器probe单列。静态阈值与实际v3二进制模型阈值逐bit一致，18个方法×cell的输出全部逐bit等于原v3 exact reference。独立代理核查了边界、代表点选择、计时循环及无预测缓存。此模型688个内节点全是数值`<=`、missing_type None；驱动拒绝NaN、类别和其他比较类型。

| 原始G | 阈值bin网格 | 实际预测点G′ | 合并＋v3＋还原：compact μs/组 | 同控制：dense可信提取 μs/组 |
|---|---|---:|---:|---:|
| 256 | 23×4 | 92 | 7.256 | 7.359 |
| 1024 | 23×12 | 276 | 16.892 | 17.007 |
| 4096 | 23×16 | 368 | 26.419 | 26.623 |

表中为两种轴顺序较快的B，恢复成canonical A输出的成本已计入。模型阈值JSON读取、解析、去重构建合计3.886ms，一次成本；每G的压缩森林load约0.25–0.30ms及probe另列，预分配scratch不计入warm请求。全部运行wall2.30秒、RSS约16MiB。G4096同进程plain v3出现约181.5μs的中位数、末轮又降至约129μs，明显有系统/频率漂移，因此不拿跨进程倍率当严谨加速结论；较简单控制相对矩形原型约88–105μs的大幅差距仍足以否定继续优化四角实现的优先级。原始记录 `treewalker_20261022/results/credit_grid_bin_probe_v3.json`，驱动为同子目录 `scripts/credit_grid_bin_probe_v3.rs`。

首版控制按该请求实际G′加载压缩森林，不能直接称为任意新请求的端到端服务成本。源码确认 `Predictor` 支持每次长度不超过 `max_group_width`，默认dispatch按实际输出长度选择mask；两轴全模型阈值各33/15个，因此可用**只依赖模型的固定上界34×16=544**一次加载，避免请求依赖的重新准备。[v3运行时dispatch源码](https://github.com/Shopify/treewalker/blob/c42e942ec976022b5178ba797b43209e89ec39bb/src/predict/kernel.rs#L115)

最终补测已用同一个固定544 Forest和同一个Predictor跨G256→1024→4096复用，18项输出全部逐bit一致，无每请求load/probe。每次重新映射、构造代表网格、预测、恢复输出仍全部计费。得到以下稳态结果，μs/组：

| G | 固定544合并控制 compact | 固定544合并控制 dense | 同进程未合并v3 dense A | 同进程未合并v3 compact A |
|---|---:|---:|---:|---:|
| 256 | 7.165 | 7.255 | 12.666 | 15.152 |
| 1024 | 16.991 | 17.075 | 40.980 | 44.561 |
| 4096 | 22.503 | 22.616 | 121.463 | 141.888 |

合并列仍为B顺序预测＋scatter；右两列是同进程A控制，不能替代前述允许两轴的完整强基线。静态阈值索引5.191ms、Forest加载0.831ms、Predictor及probe0.384ms，每项只发生一次，结果cell重复记载时不能累计三遍。缓冲区预分配，未包含冷启动、文件IO、任意输入结构验证；这属于已配置服务的warm请求性能。wall2.31秒、RSS约16MiB、核8单线程，未扩新数据。原始样本和精确检查在 `treewalker_20261022/results/credit_grid_bin_fixed_probe_v3.json`，汇总为同目录 `credit_grid_bin_fixed_summary.json`，最终驱动 `treewalker_20261022/scripts/credit_grid_bin_fixed_probe_v3.rs`。

**决策：二维四角切口停止扩展。** 初始赢过未合并v3的现象确实存在，但更简单的强控制更快，且大G消融也不支持四角是主要原因。阈值等价合并本身有实测价值，却不因这张表就成为新算法；仍缺真实分组场景、跨模型外推与相对解释性近邻工具的完整比较。

查重已确认阈值等价性的观察本身已有大量先例。[Treelite阈值量化](https://treelite.readthedocs.io/en/0.90/tutorials/optimize.html)早已用阈值整数编码减少比较成本；[Woodelf++ Full-PDP](https://arxiv.org/html/2605.14578v1#S4)利用阈值间预测恒定生成代表网格，[固定实现](https://github.com/ron-wettenstein/woodelf/blob/e3a528e0309882042e885078913a3d7852203ddd/woodelf/pdp.py#L41)接近本控制的应用。所审TreeWalker 2/3未见整查询去重再scatter：v3的 `identical_rows_go_together` 测试仍为8个相同行保留8个mask位，不能误读成压为1行。[v3源码](https://github.com/Shopify/treewalker/blob/c42e942ec976022b5178ba797b43209e89ec39bb/src/predict/precompute.rs#L871) [InferDB](https://www.vldb.org/pvldb/vol17/p1830-salazar-diaz.pdf)的索引替代则是近似方法，不能用来证明当前exact控制已被完整实现，也不能忽略其近邻意义。

### 7.5 T06：张量收缩顺序与内存布局联合优化（文献候选）

输入是einsum表达式、维度和内存上限，输出计算树与中间张量布局。简单的“减少FLOPs”可能与实际CPU延迟不一致，但这不是新发现：[EinsumTrees，ASPLOS 2025](https://scalable.uni-jena.de/research/2025/03/02/einsum-trees.html)已有固定收缩树的布局优化，[OMECO](https://github.com/GiggleLiu/omeco)已考虑读写成本，[cotengra](https://github.com/jcmgray/cotengra)有树搜索/重写，[Tensor Seeks Layout](https://arxiv.org/abs/2608.21555)进一步研究布局。宽泛的“考虑transpose与数据搬运”应排除。

可能的窄切口是4–8个张量局部计算树与布局的联合选择，并明确峰值内存、规划时间和重复执行的回本次数；已进一步查重，结论见本节后段；未启动此方向实验。必要对照是**所有path都送入同一个现有布局优化后端**，不能拿优化布局的候选与未经布局优化的基线比。先看 FLOP较优path 与实际较快path 的差距，再决定是否做联合求解；单纯更换BLAS后端的速度差不算算法收益。

公开资源：[EinsumBenchmark](https://benchmark.einsum.org/)有168个实例，[Zenodo数据](https://zenodo.org/records/11477304)约551.8MB、CC BY 4.0；[实验仓库](https://github.com/ti2-group/einsum_benchmark_experiments)与 [einsum_ir](https://github.com/scalable-analyses/einsum_ir)提供开源实现。CPU后端仍有libxsmm/OpenBLAS/LibTorch构建成本，当前不为凑题目先下载/运行全套。若后续启动，按规模与来源预选12–20个可承受实例，完整输出做容差验证，分开报告规划、执行、峰值内存和重复次数；对强path＋现有layout后端不足约10%则降级。

进一步源码审查使它降为低优先级机会验证：cotengra不仅有FLOPS＋读写组合目标与局部重写，还能用预计执行速率/重复次数控制搜索时间，并有 `sort_contraction_indices()` 全树安排中间index次序。维护者讨论中未观察到后者的普遍显著加速，但这只是已有经验，不能推出任意后端无收益。[搜索预算文档](https://cotengra.readthedocs.io/en/latest/advanced.html)、[源码](https://github.com/jcmgray/cotengra/blob/main/cotengra/core.py)、[维护者讨论](https://github.com/jcmgray/cotengra/discussions/56)

也不能过度判重：[CoNST 2024](https://arxiv.org/html/2401.04836v1)正文给定binary contraction tree，其order不直接等于完整重关联搜索；[Euro-Par2026全局布局工作](https://research.birmingham.ac.uk/en/publications/constraint-driven-global-data-layout-optimization-for-tensor-expr/)已涉及经验代价与BLAS/融合后端，当前未核到完整可复现实物，不能忽略也不能说已跑通。若后续做实验，强基线应包含cotengra combo＋subtree_reconfigure、四种index排序priority，以及相同预算K个候选path分别经ET布局后取优。候选若不能超过best-of-K，不足以证明联合搜索贡献。ET的[ASPLOS25工件](https://github.com/scalable-analyses/et_asplos25)与[2411版本](https://github.com/scalable-analyses/einsum_ir/releases/tag/2411)可用于固定后端；完整benchmark构建依赖不只是NumPy。

### 7.6 其他已快速筛掉的宽泛提议

这些只是查重，不是实验否定：流式BPE/增量tokenization已有[ICML2026](https://proceedings.mlr.press/v306/jiang26l.html)、[TokTier](https://arxiv.org/abs/2607.29678)、[DeltaTokenizer](https://arxiv.org/abs/2610.19801)；2:4稀疏通道排列已有[多种数学规划与列生成](https://doi.org/10.1007/s11590-023-02084-8)及[PermLLM](https://github.com/lanchengzou/PermLLM)；无损权重压缩已有[ZipNN](https://github.com/zipnn/zipnn)、[DFloat11](https://arxiv.org/abs/2504.11651)、[ECF8](https://github.com/zeyuyang8/ecf8)。它们资源上可能可做，但当前没有比已有方法更清楚的窄问题，暂不安排实验。

## 8. 文件与复现信息

- 本文是单一可读研究记录；脚本、原始 JSON/日志另存，避免把每次输出全文塞入 Markdown。
- 远程实验目录：`<EXPERIMENT_ROOT>`。
- 当前服务器镜像：`<LOCAL_ARCHIVE_ROOT>`，仅保存文档、脚本与中转资源。
- 每个最终实验应记录：实例哈希、split、source commit、环境/二进制哈希、方法参数、seed、超时/内存上限、完整命令、耗时、AIG 节点与深度、equivalence 状态、结果文件路径。

| 内容 | 入口或证据位置（相对远程根目录） |
|---|---|
| ABC六流程与truth检查 | `scripts/abc_baselines.py`、`scripts/aig_check.py`、`results/e01_abc_dev_small/` |
| Cirbo修复、原次序和两种简单策略 | `scripts/profile_cirbo.py`、`scripts/run_profiles.py`、`results/cirbo_repaired_merged_audit.json` |
| 新版ABC eSLIM及C0消融 | `scripts/run_abc_eslim.py`、`results/e02_abc_eslim_d1_t5_dev_s01/`、`results/e03_abc_eslim_boundary_d1_t5_dev_s01/` |
| 超图原始A/B、实际输入与checker | `mtkahypar_esa2026_baseline/run_baseline.py`、`mtkahypar_esa2026_static_audit/inputs/` |
| 重顶点MILP与独立模型枚举 | `scripts/heavy_vertex_repair.py`、`scripts/check_heavy_vertex_repair.py` |
| 超图最终配对审计 | `scripts/audit_e04_pairs.py`、`results/e04_paired_audit.json` / `.csv` |
| TreeWalker离线工件与官方检查 | `treewalker_20261022/sources/`、`treewalker_20261022/logs/core_checks*` |
| Credit唯一训练和执行器pilot | `treewalker_20261022/scripts/prepare_credit.py`、`scripts/credit_probe.rs`（后者也在treewalker子目录） |
| 单变量自有原型及v3配对 | `scripts/export_interval_control.py`、`scripts/interval_control.cpp`、`results/e05_interval_control/` |
| 二维精确原型与强对照审计 | `scripts/export_rectangle_control.py`、`scripts/rectangle_control.cpp`、`results/e06_rectangle_control/` |
| 共享矩形生成器的一维/二维消融 | `scripts/run_rectangle_ablation.py`、`results/e06_rectangle_ablation/` |
| 阈值等价合并强控制 | `treewalker_20261022/scripts/credit_grid_bin_probe_v3.rs`、同子目录 `results/credit_grid_bin_probe_v3.json` |
| 固定模型容量、跨请求复用的最终控制 | `treewalker_20261022/scripts/credit_grid_bin_fixed_probe_v3.rs`、同子目录 `results/credit_grid_bin_fixed_summary.json` / `credit_grid_bin_fixed_probe_v3.json` |

脚本中的主机检查禁止在当前服务器误跑实验。不同方向使用隔离环境，不能把TreeWalker的NumPy2.x环境混用于既有Cirbo/超图脚本。完整命令保存在对应结果和日志，保留了失败与修正记录；开发集机会测量没有消耗IWLS最终留出集。

## 9. 何时继续、何时换题

继续条件：强基线有真实且可重复的瓶颈，候选能在同预算下改善质量/时间，独立正确性检查完全通过，且有区别于近邻工作的机制证据。

换题条件：宽泛想法已被覆盖且没有更窄的机会；原任务趋于饱和；收益只来自实现 bug/弱基线/额外预算；或前处理/数据/依赖成本使研究偏离用户资源条件。保留负结果与具体原因，不把负结果删去后重新叙述成成功。

## 10. 本轮研究判断

| 方向 | 已做程度 | 当前判断与下一项必要证据 |
|---|---|---|
| T01 布尔局部精确综合 | 源码、正确性修复、12开发输入、强ABC基线与上下文消融 | 普通窗口排序/缓存切口降级。只有发现强eSLIM中新的具体代价瓶颈，才值得恢复；不继续刷Cirbo的弱基线成绩。 |
| T02 重复e-graph提取 | 文献与公开实现静态审查 | 缺真实重复查询轨迹及其成本证据；原生reoptimization已是必要对照，暂不开发新缓存器。 |
| T03 多预算rematerialization | 查重、实现及依赖审计 | 多预算复用已有覆盖，完整框架仍涉及GPU profiling。状态分类疑点未复现，不当作论文贡献。 |
| T04 多约束超图修复 | 2个压力输入、20次原生起点、10组后优化配对、3个失败起点修复 | 重顶点独立MILP代价更高、通常不如原生V-cycle。这个小邻域停止扩样，不能据此否定整个超图方向。 |
| T05 CPU树集成分组查询 | 两版官方检查、同模型执行器、单变量、二维与两项强控制 | 最有明确实测机会的是避免重复评价模型等价查询；但核心观察已有近邻工作。二维四角不能作为当前主贡献，不据一个模型立题。 |
| T06 收缩顺序与布局 | 近期论文、工件、代码和必要强对照审计 | 背景可控、CPU可做，但宽泛联合优化已拥挤；尚无真实机会测量，不宣称有收益。 |

若从这轮结果继续投入，优先级最高的**验证问题**是T05的等价合并是否在自然产生的分组任务、不同树规模和少重复输入中仍有净收益，以及如何以低于预测节省的成本判定何时不合并。需要先找到真实任务与直接近邻的完整对照，固定跨数据集协议，再做有限外推；仅在当前重复网格上扩G或调实现没有足够意义。这是待验证的研究方向，不是已经找到可发表的新题目。

本轮未使用GPU，也未调用.24商业工具。商业工具的使用条件是题目确有PPA/形式验证/仿真需求，并且这项验证能决定研究判断；当前三个实测方向已有公开工具可完成必要验证。

## 11. 2026-10-23：转为完整顶会论文目标

用户明确指出：目标是形成顶会论文，至少三个创新点，而不是随便探索。此前筛选记录不足以完成该目标；新的完成条件是形成统一问题、至少三个经查重与实验支撑的实质贡献、可复现的方法实现、完整强基线与消融、外推证据，以及可审阅论文草稿和工件。当前持续目标已建立，不能以单轮负结果或完成微基准为终点。

### 11.1 论文主线的选择与贡献验收

至少三个创新点必须共同解决一个重要问题，不能拼接三个不相干的小优化。修复bug、调参数、更换求解器、接一个LLM、增加普通缓存、仅优化已有实现均不直接算创新。新问题定义或新评价基准也必须有实际价值和现有评价缺口的证据，不能只换任务名称。每项候选贡献都记录以下内容：

| 必答项 | 需要的证据 |
|---|---|
| 具体解决什么困难 | 可复现的现有方法失效例、瓶颈分解，或严格定义的表达能力/复杂度限制 |
| 最近强工作为何没有解决 | 原论文的适用域、算法细节和固定版本代码；不能仅凭关键词不同判断新颖 |
| 新的技术机制是什么 | 可实现的算法/表示/求解或推断过程，以及关键不变量、复杂度或理论条件 |
| 能被什么结果推翻 | 预先明确的负对照、同预算简单方法、失效域；不能只报告有利例子 |
| 如何单独验证贡献 | 去掉该机制或替换为最接近已有机制的消融；整体收益与单项收益分开 |
| 如何支撑论文结论 | 多数据/设计家族、不同规模、真实总成本、正确性、统计稳定性、可复现实物 |

先建立贡献间的依赖关系，再扩实验。只有主线同时具备重要性、可做性和明确的未解决挑战，才投入较长实现；但一旦选定，不因一两个失败配置立即换题，应诊断机制、修改方法并继续验证。目标是形成经得住强对照的论文成果，不能倒推编造三条“创新”。

### 11.2 当前并行推进

三个代理重新分工：一条深入晶体管网络/SAT及相邻纯EDA问题；一条判断T05是否存在超出普通分箱、TreeWalker v3、Woodelf++等的论文级问题；一条调查低算力AI/算法系统的可成篇主线。主代理并行检查几何规则代码的语义验证、符号区分测试等方向，并统一审查证据。所有候选仍须满足CPU≤16进程、内存有余量、GPU仅CUDA0一个进程；常规实验在.44，商业验证在.24。

当前阶段只把技术设想写作假设，等待原文/源码查重和可证伪机会测量。旧T05分箱结果可作为已具备的强控制与基础设施，但不预设必须围绕它凑出三点。

### 11.3 首选主线：有认证依据的晶体管网络精确综合

当前优先验证的问题是：**能否在保持完整搜索空间和正确性的前提下，减少“证明更小网络不存在”的代价，让精确综合覆盖更多函数？** 初试采用单个导通网络、输入及其反相均可用、无串联高度限制的清晰模型。输入为完整真值表；输出为无向两端开关网络，连通当且仅当函数为1。这个模型是实际CMOS单元的一项松弛；开关数不能直接等同完整单元晶体管数、延迟或版图面积。若该主线成立，还需分别处理PUN/PDN、反相器成本和物理约束，必要时用.24做商业仿真。

**为何不直接声称已有三个创新点：** MiniTNtk已有多输出和代表pattern加速；MuSTNet/TransSyn已有多级、布局感知或cut映射相关机制。进一步读到 [Free-Topology TNS，ICCAD2026作者工件](https://github.com/b8kang/Free-Topology-TNS-ICCAD2026)，其正文已有直接terminal-to-net编码、方向及编号对称打破等。因此“去掉MiniTNtk冗余wire图”“改成直接net编号”“加多输出”均不能作为我们的新贡献。该仓库提供论文、网络和日志，但明确不提供完整综合源码。另发现可作为较新开源对照的 [L2L，DAC2026](https://github.com/b8kang/L2L-Logic_to_Layout_Exploration)：有C++/Z3综合源码和单元工件，需核实模式一致后再比较。固定版本分别为 `441d7cc5c1e588cb0d2ef599da65793fa4db7d86`、`fe8ab8d31307257a4cfcfc9c71c437a2f4917c23`。

第一项候选机制是**由小函数的认证资源要求产生可用于较大函数的约束**。设变量i在原网络出现cᵢ次（合并正负文字）；固定变量集合U以外的输入后，关闭的边被删除、开启的边被收缩，得到cofactor g，其U内开关数量逐分量不超过原c。因此 `Σ(i∈U)cᵢ ≥ LB(g)` 是必要条件。不同U可以重叠，但必须保留共同的c变量求整数覆盖，不能直接把各块下界相加。

这个限制/计数思想本身是经典方法，见 [Jukna教材，§15.1 Nečiporuk switching-network下界](https://web.vu.lt/mif/s.jukna/boolean/bool-V7.pdf)。普通PB聚合也不是新贡献。待考察的更窄机制是：机器认证的分变量/分极性资源要求、共享cofactor之间的兼容性，以及这些信息如何以可审计的方式参与搜索。它们目前尚未形成三个成立的创新点，不能把“建表—加约束—调用求解器”直接拆成三点。

**正确性边界已提前锁定：**

- 三输入多数函数Maj3不仅有计数(2,2,1)的5开关实现；允许反相输入时还有(1,1,3)。后一构造为 `a:s–u, b:v–t, ¬z:u–v, z:s–v, z:u–t`，z=0为a和b串联，z=1为并联。已独立检查8个输入。仅保留单调网络会漏解。
- 计数profile必须覆盖当前预算下所有可能实现；仅枚举最小总数层不能保证完整Pareto frontier。较大总数可能减少某个变量或某种极性的次数。有限枚举只提供有界事实，不能冒充无界认证表。
- 免费输入反相允许按输入置换和输入取反归约（NP）；不能未经证明把输出取反也合并为NPN。同样不能默认PUN/PDN成本相同。
- 无stack限制的下界可安全用作受限问题的松弛；受限模型的profile不能反过来剪枝无限制问题。上述(1,1,3)实现含长度3的导通路径，而常见(2,2,1)实现最长有效路径为2。
- 对加权计数，不可用只枚举有限边数的表约束含零权重且边数无界的问题；初试限定明确的总预算，或使用严格正权重并给出完整有限覆盖范围。

### 11.4 已完成的原编码检查与下一项机会测量

MiniTNtk采用固定提交 `933255d573f2d92a0745cb47202bfe49db80195b`。新增薄入口只导出原CNF及变量到网络的映射，没有改上游编码，也不调用SIS或Graphviz。输入采用完整on/off minterms，关闭两项加速，并在parser之后显式允许全部2n种文字；仅关闭加速不足以解除parser从SOP推断的极性限制。程序检查确认每个开关的全部selector及one-hot子句存在。常量函数由零开关断路/短路单独处理，不交给原parser。

| 函数、开关预算 | PySAT Glucose3结果 | 求解秒数 | 独立验证 |
|---|---|---:|---|
| Maj3，3 | UNSAT | 0.0759 | DRUP由drat-trim验证 |
| Maj3，4 | UNSAT | 4.4597 | DRUP由drat-trim验证 |
| Maj3，5 | SAT | 0.00861 | 全CNF子句及8输入BFS通过 |
| XOR2，3 | UNSAT | 0.01597 | DRUP由drat-trim验证 |
| XOR2，4 | SAT | 0.00329 | 全CNF子句及4输入BFS通过 |

验证器使用官方 [drat-trim](https://github.com/marijnheule/drat-trim) 提交 `2e3b2dc0ecf938addbd779d42877b6ed69d9a985`，三份证明均返回 `s VERIFIED`。这验证了导出公式的UNSAT证明；公式与网络问题的对应仍需模型完备性论证及独立小规模枚举检查。fixed-r模板允许不接入有效网络的开关，因此可用无用开关padding，存在性对r单调。当前得到的最小值只限上述单网络模型。

五例均在.44核9串行执行，进程峰值RSS约94.5MiB。工件为 `scripts/minitntk_fixed_r.cpp`、`scripts/audit_minitntk_fixed_r.py`，结果镜像 `results/e07_minitntk_fixed_r/pilot/`；原始 `audit.json` 保留生成时证明尚未验证的状态，后续证明与selector审计见 `independent_checks.json`，不可只读取原始状态误报。

后续实验分三项并行，但不同时放大规模：核9检查parity3及其求解难度；核10从无向多重图直接枚举≤5边全部拓扑与文字标号，建立独立小函数对照；核8对全部65,536个四输入真值表测量下界覆盖面。最后一项按真值表统计而非等价类均匀统计，不作为最终benchmark。

四输入的初试先用变量极性必要次数（无关0、单调1、双向依赖2）和已证Maj3下界。令bᵢ为上述单变量下界，tᵢ为除i以外三变量cofactor的最大下界，则重叠覆盖的最优总数可直接计算为 `max(Σbᵢ, maxᵢ(bᵢ+tᵢ), ceil(Σtᵢ/3))`。推导是总数R下每个cᵢ须落在整数区间 `[bᵢ,R−tᵢ]`，这些区间能凑出和R的条件恰为该式；脚本另用小规模穷举核对。作为例子，四输入“至少两个为1”每个三变量cofactor包含Maj3，得到下界7，而基本下界为4。**这是待测覆盖面的普通约束控制，不称作创新，也不说明已有7开关实现。**

### 11.5 小函数枚举、覆盖面与强控制的实际结果

独立枚举器不调用MiniTNtk或SAT。先删除不在任何简单s–t路径上的边和无用内点（不会改变任何输入下的连通性）；r边有效网络至多r+1个顶点。枚举所有无自环的无向**多重图**边集合，端点固定，仅按内部顶点重命名及两端交换归约，再遍历每条边的6种文字标签。多重边不可省略，因为并联不同文字本身就是有效网络。此过程涵盖所有≤r边网络经删冗余后的实现；profile是逐分量预算意义，不是包含任意死边padding的全部exact-count列表。

| 最大枚举边数 | 新增该层拓扑数 | 该层非串并联拓扑数 | 本层新增达到最小值的三输入函数数 |
|---:|---:|---:|---:|
| 0 | 常量另行处理 | — | 2 |
| 1 | 1 | 0 | 6 |
| 2 | 2 | 0 | 24 |
| 3 | 4 | 0 | 64 |
| 4 | 11 | 0 | 30 |
| 5 | 31 | 1 | 80 |
| 6 | 104 | 6 | 32 |
| 7 | 369 | 41 | 16 |

≤7枚举约34.85秒，得到254个三输入函数的最小值，两个parity函数尚只有下界8；结合已找到并BFS验证的8边实现，可确定parity3的最小值也是8。这里下界依据是完整枚举及覆盖论证，不能混称所有256项都有DRUP证明。所有6,014个三维profile网络代表另经Python BFS检查。单独重放拓扑记录正负分开的六维profile约38.85秒，输出326个有界Pareto项（含两个常量），全部代表再次BFS通过；峰值内存约12MiB。原始脚本和输出保留，使枚举范围、位顺序和规范化可复查。

四输入覆盖面测量统一使用 `scripts/audit_cofactor_bounds.py`，全部65,536真值表，包括64,594个四输入都有效的函数，其中34,226个每个输入均双向依赖。初始库与完整≤5库给出相同结果；更大的库才明显改善：

| 三输入下界库 | 比基本极性下界更强的四输入函数数 | 全双向依赖组中获益数 | 重叠聚合比单独最佳cofactor更强的函数数 |
|---|---:|---:|---:|
| 基本极性＋Maj3下界 | 3,312 | 0 | 32 |
| 全部≤5边枚举 | 3,312 | 0 | 32 |
| 全部≤7边枚举 | 26,010 | 17,666 | 818 |

≤7库中，相对基本下界提高1、2、3的数量分别为20,848、5,032、130。每轮全表测量约3.6–3.9秒，RSS约20MiB。下界库构建成本必须在后续性能评价中报告，并区分首次构建和跨函数复用；这些数字**不是完整综合加速、更优晶体管数或新颖性证据**。

进一步强化MiniTNtk控制：若存在某输入从0变1使函数从0变1，则正文字必需；若能使函数从1变0，则负文字必需。根据这一语义条件把必要文字固定在前若干个可交换开关槽，是已有的文字/槽对称打破思想，不限制剩余槽使用反相输入。与关闭全部加速的原始检查相比，Maj3四边UNSAT从4.46秒降至0.242秒；parity3六边从57秒仍UNKNOWN变为3.40秒UNSAT，且增广CNF的DRUP证明通过独立检查；parity3七边20秒仍UNKNOWN。此结果说明不能把完全关闭加速的编码当作最终强基线。

原始parity试验的七边硬超时、六边部分证明均明确保留为UNKNOWN；六边约306MB部分证明留在.44，没有传回当前机器，也没有拿它认证无解。强控制结果见 `paper_phase/results/mandatory_control/`；独立枚举为 `paper_phase/profiles_r7/`、`paper_phase/literal_profiles_r7/`；四输入覆盖面在 `paper_phase/results/cofactor_{initial,r5,r7}/`。

**profile方向的负证据也保留：** 在三维计数、预算≤5域，全部不可行预算均被单变量最小次数或总数最小值排除，profile没有额外信息。六维正负计数用更大库检查预算≤5时，有120个预算需额外凸约束排除，但未发现整数非凸空洞。不能因为抽象示例能构造整数hole，就声称真实网络中已找到；目前“非凸profile cuts”不列为成立贡献。

### 11.6 从计数转向共同连接结构：一个已验证的小例子

新的候选机制研究**两个cofactor如何共享同一批开关**。把不受控制变量x驱动的m个开关各保留两个独立端点，再加s/t，共2m+2个边界点。固定x=0或1后，其余无条件导通连接在这些边界点上诱导partition π₀、π₁；这里不能预先并入保留开关在某个输入下的导通连接。

令B(π)为partition的块数，π₀∧π₁为共同细分。对固定的保留开关身份，两个状态之间最少需要的x/¬x开关数为：

`d(π₀,π₁)=2B(π₀∧π₁)−B(π₀)−B(π₁)`。

下界理由：共同固定连接产生的partition β细于两个状态；每添加一条x或¬x边，边界partition最多合并两个块，即使存在仅供外部连接使用的隐藏节点也如此。上界构造：取meet作共同免费连线，对π₀每一块中的meet块用¬x边组成树，对π₁对应地用x边组成树，两个状态互斥开启，恰好用d条边。**这个距离表达式已明确见于 [Rossi 2011《Partition distances》§3式(6)](https://arxiv.org/html/1106.4579)，不能当新定理。** 待查明的差别是固定开关身份后，在满足两个cofactor的全部边界partition上联合最小化该经典代价，并据此设计综合/认证机制；目前未以搜索暂未命中作为新颖性证明。

已执行一个全量小实验：保留边为a、¬a、b、¬b各一条，枚举10个边界点全部115,975个partition；其中96个实现XOR2、96个实现XNOR2。检查全部配对，最小d为4；自动取meet并增加4条控制边所得8边网络，对parity3的全部8输入BFS通过。总耗时0.595秒，输出 `paper_phase/results/partition_gluing_xor/`，脚本 `scripts/audit_cofactor_partition_gluing.py`。

这解释了一个计数下界不能捕捉的冲突：XOR3的每个两变量cofactor都只需4边，但三组变量都出现2次的6边方案无法把它们连成同一个网络。甚至总数≤7时，每个变量都至少出现正负各一次，故最多为(3,2,2)；选出现最多的变量作为x，其余四条边恰为上述保留集合，而它仍需要至少4条控制边。因此该小例可用明确结构论证排除≤7，而不仅是报告求解器无解。

下一步原型把这个固定边界分析转成通用SMT：对两个partition分别编码canonical代表、共同细分块数、同一组保留文字，并用m步导通可达性约束全部cofactor输入；要求 `m+d≤R`。SAT后按meet显式生成网络并独立BFS。不同m必须完整覆盖，timeout保持UNKNOWN。只有证明它在更大函数及强基线上有实际优势，且相对已有分解/图合并方法确有差别，才能升级为论文贡献；目前第三个实质贡献仍未确立，不用普通工程步骤填补。

多控制变量扩展有具体反例，不能照搬：仅有可见s/t，两个控制变量y、z通过隐藏点u串联，能实现只在y=z=1时连接。若只保留可见partition并把每个literal活跃状态的partition取meet，这些meet全是离散的，错误地判定无法连接。单control的隐藏节点消除证明不直接延伸到多control；逐对距离仍是必要条件，逐对可实现却不保证全局兼容。若研究递归消元，必须保留足够的隐藏接口结构。

额外查重已核到 [Mokhov 2015《Algebra of switching networks》](https://ietresearch.onlinelibrary.wiley.com/doi/10.1049/iet-cdt.2014.0135)：其§3–5已有以Boolean connectivity描述规范网络、闭包后消去辅助节点、条件网络合成和公共结构提取。我们不能把这些表示或语义消元称新。尚待充分查明的差别是**恢复物理literal开关资源的最小代价**及不可实现证书，而不是新连通代数；复合条件开关在代数中可以直接表示，但实现为单输入控制开关有另外成本。[Possani等图方法](https://cseweb.ucsd.edu/classes/fa23/cse248-a/papers/logic/Graph-Based_Transistor_Network.pdf)的部分网络发现与开关共享也必须正面对比。

首版通用SMT已实际执行，使用整数canonical partition代表和全部m步可达性；各m子问题最长15秒。结果如下，求解时间之外另记录了每项约0.05–0.36秒编码成本：

| 函数、预算 | 子问题求解结果 | 整体状态 |
|---|---|---|
| Maj3，4边 | m2：0.0195秒UNSAT；m3：0.337秒UNSAT | UNSAT，全部m已覆盖 |
| Maj3，5边 | m2：0.0123秒SAT，构造5边网络 | SAT，8输入BFS通过 |
| Parity3，6边 | 唯一m4：0.449秒UNSAT | UNSAT，全部m已覆盖 |
| Parity3，7边 | m4：0.504秒UNSAT；m5：15秒UNKNOWN | UNKNOWN |
| Parity3，8边 | m4：0.0778秒SAT，构造8边网络 | SAT，8输入BFS通过 |

全过程17.76秒、峰值RSS约71MiB，输出 `paper_phase/partition_gluing_smt_v1/`。这些UNSAT尚未独立proof-check，也未将枚举库中已知的parity下界倒灌到SMT中。当前瓶颈是m5的额外保留边和partition搜索；下一版保持同样语义，改为Boolean等价关系、Tseitin可达性与计数CNF，观察瓶颈是否来自整数表达。不先加parity专属变量对称性来掩盖通用表示问题；更换表达本身也不自动算第三创新。

### 11.6.1 较新L2L编码的同域对照已准备

在.44用GCC9.4、官方Z3 4.15.3 wheel解包的头文件和动态库编译了L2L源码，无系统安装。原资源限制版本只运行了help；实际小试采用单独fixed-r入口，保留原core编码，关闭compound gate catalog，仅允许全部2n种PI文字并免费提供反相输入，传播步数覆盖全部简单路径，并严格区分SAT、UNSAT、UNKNOWN。PMOS开关极性及输入LSB顺序经独立BFS核对。原默认输入池和目标不同，不能把适配成绩当作论文默认配置的复现结果。

| 同域L2L小例 | 状态 | 总墙钟秒数 |
|---|---|---:|
| Maj3，4边 | UNSAT | 0.183 |
| Maj3，5边 | SAT，8输入BFS通过 | 0.285 |
| XOR2，3边 | UNSAT | 0.042 |
| XOR2，4边 | SAT，4输入BFS通过 | 0.041 |
| Parity3，7边 | UNKNOWN，超时 | 18.032 |

峰值RSS约50.4MiB，一次只运行一个求解进程。Z3额外的timeout watchdog不进行并行求解，所有线程固定核9。UNSAT尚无独立证明检查；保留的anchors/net-usage/strict-lex约束也需要继续审查其完整性，且可能不允许无用开关padding，因此固定r无解不能自动推为≤r无解。结果 `results/e08_l2l_static_build/`，API、源码哈希和适配patch均在该目录manifest。真实NSP/标准单元函数正在收集，后续至少包括5输入以上外推，避免只在少数toy函数上论证方法价值。

### 11.7 同期备选方向的进一步查重

以下内容作为主线失败时的可追溯后备；没有借这些方向凑三条互不相干的贡献。

**树集成分组推理：** 更强近邻包括 [InferF，SIGMOD2026](https://github.com/asu-cactus/InferF-ML-Inference-Factorization)、[CactusDB](https://arxiv.org/html/2602.23469v1)、[Counterfactual Maps](https://arxiv.org/html/2602.09128v1)。InferF已有按特征生成并合并树叶掩码，不能把该分解算新；Woodelf++/FastPD也不能因接口名是PDP就排除。剩余假设是请求的固定上下文和变化轴经常改变时，非恒定残余子树能否被低成本共享；尚无真实请求轨迹证据。若恢复，应先量化实际非恒定共享率，并与全局阈值合并、上下文阈值合并、InferF式分解、全变量TreeWalker及有限配置池公平比较，完整计入每请求预处理和一次构建成本。

**ANN的低成本召回审计：** [ConANN，PVLDB19](https://www.vldb.org/pvldb/vol19/p29-horchidan.pdf)在分布变化下有召回风险失配，重新校准需要exact top-k标签。候选统一问题是减少审计标签代价，并对已发生查询窗口给出可核验风险界；候选机制为逐步收紧的部分精确检索证书、对自适应区间标签有效的统计证书、共享状态的检索/审计预算分配。三个目前均未成立。[Quake，OSDI2025](https://www.usenix.org/system/files/osdi25-mohoney.pdf)、[DARTH](https://github.com/MChatzakis/DARTH)、[Active anytime-valid RCPS](https://arxiv.org/abs/2406.10490)、[SOLO](https://arxiv.org/html/2610.02387v1)均为必要近邻；不能把主动标注或漂移后再校准当新点。主要风险是高维安全界太松，partial audit可能几乎等价完整扫描。若后续做，先用公开SIFT1M/GIST1M原向量与GT做CPU机会测量，计入全部GT/校准成本；前25%审计预算若基本仍只给[0,1]，就不继续开发控制器。尚未安装或跑ANN实验。

**DRC规则程序的语义验证：** [Rule2DRC](https://github.com/snu-mllab/Rule2DRC)已有候选规则间的区分测试；[DRVerify白皮书](https://semiwiki.com/wp-content/uploads/2018/06/sage_DRVerify_WP_v2-3-1.pdf)已有形式化规则意图与系统化测试；[DRDebug作者论文](https://nanocad.ee.ucla.edu/wp-content/papercite-data/pdf/j69.pdf)已有规则匹配相关工作。因而泛泛的“LLM写规则＋SMT生成边界测试”空间很窄。已在.44对Rule2DRC公开1,000任务parquet做schema审查，未运行DRC，也未用测试标签优化方法；脚本 `paper_phase/audit_drc_schema.py`，结果 `paper_phase/results/drc_schema/`。正则粗数会混入字符串与注释，不能把其方法计数当正式DSL覆盖率。更窄的实际几何DSL语义等价可能有价值，但完整操作集/非矩形几何代价高，当前不优先。

### 11.8 新的实际约束：最少边数拼接不保留stack预算

为了使主线最终能用于真实单元，而不是只优化抽象开关数，进一步分析了stack约束。这里stack定义为所有需要导通的输入中，最短s–t路径长度的最大值；它是结构约束，不能等同实测延迟。

已找到并逐输入验证固定接口的反例。保留6条正文字边为：`(s,A,a), (B,t,a), (s,A,b), (C,t,b), (s,B,c), (C,t,c)`。x=0不连接A/B/C，x=1要求三者连通，因此rank最少只需2条x边。三种生成树却各有一个输入导致4级路径：仅a为1要求A–B、仅b要求A–C、仅c要求B–C，若要求stack≤3，这三对都必须有直接x边。增加为三边clique后，全部16输入满足stack≤3。函数为 `ac ∨ x(a∨b∨c)`，真值为 `0xfea0`（a为LSB）。

这个例子有明确限制：同一个函数**全局上**可改写成6开关、stack2的串并联网络，脚本也构建并验证了该反向控制。故它不证明整个函数需要9开关；它只证明，在给定保留网络和两个partition时，边数最小的拼接未必满足路径预算。不能拿局部反例冒充完整综合优势。

脚本 `scripts/audit_stack_gluing_example.py` 在.44核8检查所有三种tree、clique及全局反向控制，输出 `paper_phase/results/stack_gluing_example.json`。下一项候选难题是：如何以比重新求整个网络更低的成本，为给定接口实现满足全部truth路径需求的控制边，并把失败原因反馈给接口搜索。普通spanner求解、增加已有stack开关或泛用Benders调用本身不算新贡献，需继续查重并寻找专门机制。此候选与主线有直接联系，但尚未实现为可比较的方法。

六维profile的整数空洞进一步扩查到预算≤7，仍未发现：≤6域236,544点、344次精确有理LP；≤7域439,296点、752次LP。两轮共44.914秒且全部完成。这使当前“非凸资源空洞”动机正式降级，保留原结果而不继续为凑创新点扩同样小表。

### 11.9 后续验证协议与当前未完成项

原始53_NSP目录已从UFRGS取回并完成表达式双实现检查：53个真值不同，但输入置换/取反归约后为43个NP类；输入数为4输入1个、5输入8个、6输入44个。全部有效变量均为positive-unate，这是基准的重要偏向。不能只在它上面论证双向依赖函数的扩展性，正在另收真实4–6输入双向依赖控制。L2L两份Liberty中的128条输出函数几乎均≤3输入，不能把它们称作5输入以上外推。

下一阶段先进行编码正确性和同域检查，再测完整综合流程：

1. 三输入按NP类挑代表，用已独立枚举的最小值对照“最小值−1”和“最小值”，同时检查新表示、解码、极性和常量边界。不能只验证SAT上界而不检查模型完整性。
2. 在NSP的43个NP代表及独立双向依赖函数上固定开发/评估协议；重复等价实例不当独立样本。先用少量开发函数查瓶颈，再冻结机制后扩大留出评估。
3. 同时保留两种清楚区分的输入池：原论文常用的受限极性池与免费互补输入池。受限池中MiniTNtk代表pattern加速可能很强，必须纳入；免费互补域不能错误套用单调性剪枝。两个域的最小值和耗时不得混报。
4. 对比MiniTNtk必要文字/槽对称控制、L2L同域适配、新表示，以及资源库和结构机制的逐项消融；报告构造、求解、证明、检查、库冷启动与复用成本，并统一总预算和UNKNOWN处理。
5. 如边数机制成立，再加入真实stack约束和相应强基线；商业仿真仅在它能检验实际PPA结论时使用.24。当前小例的stack改善不自动等于延迟改善。

当前尚缺：较大函数端到端收益、充分的新颖性边界、至少三个成立的实质创新、完整消融与外推、以及可投稿论文草稿。用户随后因额度限制暂停探索；后续任务均为恢复后的候选步骤，本次归档不重启研究。

## 12. 暂停总结：目前值得保留什么，哪些尚未成立

### 12.1 核心判断及直观解释

目前最值得保留的是**面向精确晶体管网络综合的共享子函数结构与资源约束**。输入仍是完整真值表，输出是在全部输入下正确、开关数尽可能少的网络。它适合CPU实验，主要背景是布尔逻辑、图和SAT；若最终要讨论物理效果，还需额外验证。

这里“子函数不一定能低成本共用网络”的含义是：把z固定后，两个子函数可以各自很简单，但需要不同接线。以三输入多数函数 `xy+xz+yz` 为例，z=0时是xy，对应x/y串联；z=1时是x+y，对应并联。同一对开关的物理接线固定，不能把两种情况下各自的“两开关实现”直接视为共同的两开关实现，再只增加一个z开关。当前单网络、免费互补输入的模型下，其最小值是5，已用独立枚举和证明检查确认。

**这不是现有研究普遍忽略的错误。** MiniTNtk、L2L等直接搜索整个网络，已经考虑连接与功能约束。共享、cofactor分解、连通代数也有长期研究。我们待验证的机会是：能否把“共同结构在这个预算下无法实现”的理由转成更强、计算代价更低的约束，减少整体求解器为证明无解所做的搜索。不能把这个基本现象本身当首次发现，更不能据小例子判断已有工作错误。

### 12.2 暂停前纯SAT v2的最终结果

第二版保持第一版单控制变量消元的完整语义：两组Boolean等价关系表示cofactor的边界连线，传递性子句保证partition，共同细分的块数用于控制开关预算，全部truth由m轮可达性约束。与第一版的区别是将整数SMT表达改为Tseitin与顺序计数器CNF；未增加parity专用变量对称性，也未输入已知最优答案作为约束。求解采用PySAT 1.9.dev17/Glucose3，无解证明由drat-trim独立检查。

| 函数及总预算R | 结果 | 主要证据与内部全流程秒数 |
|---|---|---|
| Maj3，R=4 | UNSAT | m=2、3的证明均VERIFIED；0.443秒 |
| Maj3，R=5 | SAT，5边 | 全8输入BFS通过；0.014秒 |
| XOR3，R=6 | UNSAT | m=4证明VERIFIED；0.262秒 |
| XOR3，R=7 | UNSAT | m=4、5证明均VERIFIED；8.705秒 |
| XOR3，R=8 | SAT，8边 | 全8输入BFS通过；0.034秒 |

关键的XOR3/R7/m5含6,277个变量、27,413条子句：纯求解4.214秒，证明检查4.019秒。五例内部全流程9.472秒，外部测得墙钟约9.57秒，峰值RSS86,068KiB（约84MiB）；均在实验服务器核10串行执行。第一版同一m5在15秒后仍UNKNOWN，因而该次迭代确实解决了一个已观察到的瓶颈。

L2L同域适配在parity3固定七边约18秒后UNKNOWN，MiniTNtk必要文字对称控制在七边20秒后UNKNOWN。这些只能作为**同一小例上的机会证据**：不同编码的子问题划分和完整性前提有差别，不能将某个子问题的4.214秒与对手的整体超时直接解释为普遍加速比。后续公平评价必须统一输入池、总成本、问题范围和预算。

原始报告为 `paper_phase/partition_gluing_sat_v2/summary.json`；CNF、DRUP及验证日志在实验机同目录各case子目录。实际执行的SAT脚本SHA-256为 `7aff9599b144d2c7330e794b389e2e3c3ef86202c51290854c8b978c28b92515`。随后加入计时线程cancel后join及通用case入口，不能把后续源码哈希冒充首次执行版本。

### 12.3 三输入NP类正确性回归已经完成

暂停整理时核对已有checkpoint：三输入共有22个NP类，覆盖256个真值表，归约只允许输入置换和逐输入取反，不合并输出取反。20个非恒定类各测试最小预算与其减1，共40项；两个常量以零开关断路/短路处理，再加2项，总计42项。为节约资源，其中4项引用已完成且已验证的Maj3/parity记录，新运行38项。

新38项的结果为20个SAT、18个UNSAT，全部与独立枚举给出的期望一致，无UNKNOWN。SAT项检查完整truth；常量明确为解析处理。UNSAT区分必要文字次数的解析下界与真正的求解器证明；后者均经过DRUP验证。结合复用的4项，共22个SAT边界与20个UNSAT边界结果。新批内部墙钟1.250秒、峰值RSS34,968KiB。

这支持编码及解码在三输入范围的正确性，但不等价于对任意输入规模的形式化完备性证明，也不代表已取得大规模性能优势。回归固定control=2，没有逐类选择最有利控制变量。复现入口是 `paper_phase/partition_gluing_np_regression_v1/manifest.json`，结果为其 `run/summary.json`；对应SAT脚本SHA-256为 `05ec181577a1fbb30c858db607eea9706c00988409873f4f08b95d42078d00d8`。

### 12.4 真实基准已就绪，仍有明显局限

原始53个NSP函数已按输入置换/取反归约为43个代表：4输入1个、5输入7个、6输入35个。全部positive-unate。43个代表中，5个目录图标签数已等于必要文字下界，38个仅高1；目录图标签数只作为已给实现的规模参照，不是新认证最优值。它们适合检验正确性和单步最优性证明，不能单独支撑广泛双向依赖或大规模扩展的结论。

从冻结IWLS开发集提取了5个NP独立的双向依赖控制，均为原输出去除unused输入后的完整函数，并逐原输入域验证；没有读取留出集或优胜电路：

| 来源输出 | 有效输入数 | 真值表hex（首变量为LSB） | 双向依赖变量数 |
|---|---:|---|---:|
| ex256_o1 | 4 | `6ac0` | 4 |
| ex270_o4 | 4 | `1ab0` | 3 |
| ex271_o5 | 5 | `66aacc0` | 5 |
| ex271_o6 | 6 | `1e665aaab4ccf00` | 6 |
| ex275_o1 | 6 | `fffe000ffe00fe0e` | 4 |

来源、表达式、哈希、NP证据和对照接口整理在 `paper_phase/benchmark_manifest/`。`nsp_np_representatives.json`、`iwls_dev_controls.json` 可作为后续输入；`runner_contract.json` 是统一计时和状态语义的接口约定，尚未据此完成新的大批量综合实验。基准本身不能算新增论文贡献。

### 12.5 论文价值与创新点的当前状态

| 内容 | 暂停时的判断 |
|---|---|
| 共享边身份下的cofactor资源兼容性 | 有明确小例、成本推导和可运行原型，当前最值得继续验证；在综合领域的新颖性尚未充分确立 |
| 小函数下界及重叠资源约束 | 能改善部分四输入下界；restriction、普通计数/PB聚合已有，不能据覆盖面直接算新创新 |
| stack约束下的接口实现 | 已有固定接口的反例；没有通用方法及整体收益证据；spanner/情景路径设计近邻很强 |
| 六维profile非凸整数空洞 | 完整检查到预算7仍未找到，相关动机降级 |
| 纯SAT替代SMT | 解决了本轮实际超时，是有效实现迭代；更换编码不自动成为独立创新 |
| 至少三个实质创新点、顶会完整证据 | **尚未形成，也未完成论文** |

总体判断：已经找到值得保留并继续验证的研究切口，当前证据比单纯的选题猜想更具体；仍没有足够依据承诺可发表论文或顶会结果。若恢复，优先测真实4–6输入函数的端到端性能、严格对照已有最强加速并分析失败原因，再决定是否值得围绕同一主线形成更多实质机制。不能先凑三条名称再倒推实验。

### 12.6 暂停与归档状态

用户因Codex额度限制明确暂停探索。三个代理均已确认停止，实验服务器无本任务遗留计算进程；既有源码、CNF/SMT、证明、JSON及checkpoint保留。本次只读取和总结已有结果、同步文档并上传GitHub，未重启实验或持续研究目标。

## 13. 2026-10-23恢复：四至六输入的可扩展性与强控制

### 13.1 本轮问题与预先固定的开发查询

用户明确要求继续探索。恢复时沿用第12节的源码和结论，不重做已完成的三输入枚举。第一优先级是检验单控制变量消元能否在较大函数上带来可解释的收益，避免围绕一个parity小例形成过强判断。仍以至少三个实质且相互支撑的创新点为最终目标；目前没有提前认定三点成立。

开发集合固定为10个函数：NSP按4/5/6输入各取ID最前两个NP代表（4输入只有一个），加上此前已冻结的全部5个IWLS开发函数。每个函数查询基本必要文字下界R₀和R₀+1，共20个查询。NSP为01、02、03、10、11；IWLS为ex256_o1、ex270_o4、ex271_o5、ex271_o6、ex275_o1。未用余下NSP代表或IWLS留出集调方法。

所有比较使用免费互补输入、单输出、无stack限制的同一网络域；MiniTNtk保留必要文字槽对称控制。性能轮关闭proof，编码/导出、求解器初始化及全部子问题均计入18秒软预算，外部20秒硬限；Python检查与最后一次编码可能使软预算略超，但没有增加下一轮完整求解预算。SAT检查CNF及全部truth的独立BFS；UNSAT在性能表中明确为solver-only，需要独立证书轮才能写成已检查证明。L2L适配仍为固定r语义，不能将其无解自动转为≤r；新方法是≤R语义。初步时间为单次开发测量，不作统计显著性或普遍倍数结论。

本轮本任务分配核0/2给强基线、核4给结构分支、核6给编码核验、核8/9给主方法，每solver单线程、每进程地址空间≤4GiB；没有使用GPU。运行中发现服务器有其他研究目录的ABC任务，未修改或终止他人进程；正式性能评价仍需记录机器负载并复测关键差异。

### 13.2 原型外推及已知下界控制

纯SAT v2原型在20查询中得到12 SAT、3 UNSAT、5 UNKNOWN。5个NSP函数全部达到已给目录规模；两个四输入IWLS函数直接达到必要文字下界（ex256_o1最小8、ex270_o4最小7），因此它们不能单独证明较难问题的突破。较困难部分如下：

| 查询 | 原v2状态/端到端秒数 | 加三输入restriction count cuts | OFF-cut＋同样count cuts |
|---|---|---|---|
| ex271_o5 R10 | UNSAT / 2.840 | UNSAT / 0.648 | UNSAT / 0.244 |
| ex271_o5 R11 | UNKNOWN / 18.042 | UNKNOWN / 18.569 | UNKNOWN / 18.431 |
| ex271_o6 R12 | UNKNOWN / 18.127 | UNSAT / 2.532 | UNSAT / 1.181 |
| ex271_o6 R13 | UNKNOWN / 18.133 | UNKNOWN / 18.384 | UNKNOWN / 18.046 |
| ex275_o1 R10 | UNKNOWN / 18.088 | UNKNOWN / 18.117 | UNKNOWN / 18.108 |
| ex275_o1 R11 | UNKNOWN / 18.103 | UNKNOWN / 18.050 | UNKNOWN / 18.032 |

这里新增决定的R12不能算新结构机制独有的成果。为每个三变量集合U枚举外部输入并查三输入完整库，得到 `Σ(i∈U)cᵢ≥max_a LB(f|a)`，再枚举必要文字次数之上的整数资源分配。ex271_o5的共同下界从10升到11，ex271_o6从12升到13，其他开发函数不提高；整批预处理仅约毫秒量级，库构建成本另计。于是R10/R12本可在进入任何拓扑求解器前排除。这些约束也已提供给MiniTNtk，不把经典restriction/count-cover当新贡献。

约束的内容有助于定位下一项瓶颈：ex271_o5的三变量集合{0,2,3}需要≥7条边，较各变量必要正负文字合计多1；当R=11时，唯一多出的边必属于其中之一。ex271_o6的集合{0,1,3}、{0,3,4}均需要≥7，当R=13时额外边只能属于交集{0,3}。ex275_o1所有三变量约束均被已有文字必要次数推出，本库对它没有新的逻辑信息。

代码为 `scripts/run_gluing_resume.py`、`audit_resume_resource_bounds.py`、`partition_gluing_resource.py`、`partition_gluing_cut_resource.py`。输出分别为 `paper_phase/resume_gluing_v2_20261023/`、`resume_resource_bounds_20261023/`、`resume_gluing_resource_20261023/`、`resume_gluing_cut_resource_20261023/`。保留每查询、每m的耗时、子句规模、模型、源码哈希及终止原因。

### 13.3 编码瓶颈对照：断开cut及后续flow

原型对每个truth行展开m轮可达性，生成量约O(2ⁿm³)，而非此前审查消息粗估的O(2ⁿm⁴)。对于OFF输入，可以为每个pin设置分离cut标志，s/t异侧，固定连线和开启保留边的两端必须同侧；存在这种cut等价不连通。OFF行因此降为O(m²)，ON行暂保留原reach。MiniTNtk等已有类似连通/断开编码，故这是必要的工程强控制，不是新的论文创新。

`partition_gluing_cut.py`的42项三输入NP边界回归得到22 SAT/20 UNSAT，零UNKNOWN，与独立枚举全部一致。较难6查询单独测量时，R10/o5为0.524秒、R12/o6为3.634秒UNSAT，其余四项仍UNKNOWN。后续Maj3 R4、XOR3 R7、o5 R10、o6 R12共6个m子问题的DRUP均由drat-trim验证；o6 R12证书轮求解6.147秒、证明检查3.569秒、完整11.080秒，证书约43MB留在.44，不与无proof性能轮混算。记录为 `paper_phase/resume_cut_{regression,development,proof}_20261023/`。

随后继续把ON输入改为单位路径flow作对照：源、汇及内部点守恒，选择的弧只能走当前partition连线或开启保留边；每点入/出至多一条，允许无关环但源路径必须到汇。它可能减少编码规模，也可能增加求解对称性，必须实测。此项仍是工程控制，不能因加速就自动成为独立创新。

### 13.4 当前正在检验的机制与新增查重

新的完整性保持分支按可行开关次数选择control：先枚举满足共同资源约束的次数向量，选择次数最大的变量（平手按索引），保留对应 `(control,m=total−c_control)` 的分支并集。这样不必固定一个出现较少的control而保留更大的接口。每个可行网络的次数向量必落在该覆盖中，故遍历并集完整；若只选择最快的一支则不完整。所有分支UNSAT才可报告整体UNSAT，timeout仍UNKNOWN。

这在o5/R11理论上把大接口m9改为三个候选control上的m8；o6/R13则是两个候选control上的m10。分支本身是已有资源推理与消元顺序思想，是否能形成独特机制、是否对真实难例有效仍须验证。首轮诊断每case总预算60秒，不能直接和18秒基线声称加速，后续有效结果需匹配预算及相同资源约束。

查重补充两点。其一，[Jukna书§15.2](https://web.vu.lt/mif/s.jukna/boolean/bool-V7.pdf)记载Cardot(1952)已有parityₙ一般开关网络最优4n−4；因此parity3=8是正确性和算法压力测试，不能当新下界发现。其二，[DAC2026 TransSyn官方摘要](https://63dac.conference-program.com/presentation/?id=RESEARCH2007&sess=sess172)已有cut-based全局映射及跨cut子电路的SAT再综合；泛泛“分块后局部SAT优化”不能作为空白。此处只有官方摘要证据，不能据此猜测其未公开实现细节。

stack扩展得到一个待完整写证的固定接口结论：单control、仅要求各输入最短路径不超过H时，同色control专属隐藏节点可以用邻接星重接消除，不增加开关数或任何边界点对最短距离；正反control可先拆开，因为不会同时导通。因此该局部问题仍可只在可见meet块上求解。4,096项穷举和2,000随机图审查通过，但测试不是一般证明的替代。若某控制边在理想clique中删除就使某ON输入超出H，该边对该固定接口必需，必需边集的cycle rank可补足生成树代价。这与经典spanner/网络设计问题紧邻，目前只作为候选，不列为已成立创新。

### 13.5 强基线完成及进一步资源推理

两个基线的20查询均完成。MiniTNtk＋必要文字对称控制得到12 SAT、2 UNSAT、6 UNKNOWN；L2L同域适配得到5 SAT、2 UNSAT、13 UNKNOWN。所有SAT独立BFS通过，Mini两项UNSAT另行DRUP VERIFIED。当前Mini明显强于L2L适配，不能只选择较弱的L2L作论文对手。L2L的原模型选择、固定r和完整性前提依旧保留，不用适配试验否定其论文。

进一步将完整三输入表中的正负文字分开，形成6维预算。对全局必要文字预算及额外一只开关的所有可能位置，检查每个三输入restriction的投影是否支配至少一个有界Pareto profile；只有投影总数≤7才可使用完整表拒绝，超过7一律不拒绝。结果：

| 函数/预算 | 六维profile后结论 |
|---|---|
| ex271_o5 R11 | 所有10个额外文字位置均排除；下界提高到12 |
| ex271_o6 R13 | 只剩额外¬x0或额外¬x3两个全局预算 |
| ex275_o1 R10 | 唯一必要文字预算被排除；下界提高到11 |
| ex275_o1 R11 | 只剩额外+x5一个全局预算 |

实际每函数约0.4–3.8毫秒，表的离线构建成本另计。o5的主要冲突来自不同cofactor对额外正/负文字的要求；这是更细的线性/凸资源约束，不是此前未找到的非凸整数空洞。检查脚本 `audit_dev_literal_profiles_20261023.py` 保存每个失败预算的具体restriction、投影预算与完整可行profile列表，未把幸存预算当物理网络。

将完全相同的两个/一个幸存预算施加到Mini，o6/R13、ex275/R11仍分别18.134/18.097秒UNKNOWN；用同预算的gluing固定保留文字并消去最多次变量，结果也均UNKNOWN，内部约18.07/18.04秒。**现阶段这两项公平对照尚未给出新表示优于强基线的证据。** 只给对手三维计数而给自己六维profile会造成不公平，已避免。

另外对造成profile拒绝的8个去重三输入literal预算，使用原Mini核心固定可交换槽的文字重放独立证明。得到3项DRUP VERIFIED、5项UNKNOWN。已认证one-hot3各literal一次的6边预算、Maj3各literal一次的6边预算，以及Maj3的`[1,0,1,1,1,1]`五边预算。最后一条直接支持ex275的10边拒绝；o5的11边排除仍有五个七边小预算未获DRUP，当前下界依据是完整枚举表及restriction覆盖，而不是全部SAT证书。大证明保留在.44。

为排除“只需多看一个变量就能轻松拒绝”的解释，抽取剩余3个全局预算的四输入cofactor，按NP同时变换truth和预算后得到80个唯一查询，选择8个低slack且覆盖各全局预算的查询做3秒诊断。结果3 SAT、5 UNKNOWN、0 UNSAT；没有低成本排除剩余预算。全部SAT逐16输入BFS通过。一个NP等价于已解四输入开发函数的查询也超时，说明变量/槽顺序会显著影响短时SAT结果，不能将UNKNOWN解释成结构阻碍。

工件为 `resume_mini_literalprofile_20261023/`、`resume_literal_profile_audit_20261023/`、`resume_profile_rejection_certificates_20261023/`、`resume_fourinput_obstructions_20261023/`，均在`paper_phase/`下。

### 13.6 诊断轮、可行上界及编码负结果

只用三维count向量选pivot的60秒诊断，o5/R11三个分支在19.906秒全部solver-UNSAT；o6/R13两个分支54.880秒全部solver-UNSAT；ex275/R10一支0.902秒solver-UNSAT，R11仍UNKNOWN。其中o5/R11和ex275/R10随后被六维表毫秒排除，不能继续算结构独有收益。54.880秒是诊断结果，不是18秒公平比较结果，且尚无完整独立证书。

施加精确幸存literal预算后的独立认证轮中，o6/R13的额外¬x0分支获得DRUP VERIFIED（全程20.413秒，其中检查6.824秒），额外¬x3分支超时；ex275/R11也超时。所以两个整case的已认证状态仍是UNKNOWN。

随后枚举必要文字预算+2的全部弱组合，三输入投影>7时不拒绝，按较小保留接口优先寻找网络；每case60秒，仅作为上界诊断：

| 函数、预算 | 候选预算/幸存预算 | 结果与证据 |
|---|---:|---|
| ex271_o5 R12 | 55 / 14 | 前2分支中找到12边网络；全32输入BFS；总4.987秒 |
| ex271_o6 R14 | 78 / 27 | 总60.087秒未找到网络；UNKNOWN，不能宣称14边无解 |
| ex275_o1 R12 | 78 / 13 | 首分支找到12边网络；全64输入BFS；总1.082秒 |

因此当前模型下，**ex271_o5最优值为12**（完整有界枚举表支持下界，BFS网络支持上界）；**ex275_o1已认证范围11≤opt≤12**；ex271_o6至少13，13边有solver-only排除但认证尚不完整，14边仍未找到上界。离完整论文仍有明显距离，但已从纯三输入toy验证推进到真实开发函数的可复核界定。

ON-flow工程对照形成明确负结果：42项NP3回归全通过，但六项困难查询全部18秒UNKNOWN；加同样三输入资源约束后的四个困难查询也全UNKNOWN。Parity3 R7从cut+reach的2.198秒变为15.188秒。普通flow峰值125.8MiB、加资源181.6MiB，规模更小却增加搜索困难；没有继续无限叠加flow变体。模块与数据归档在 `partition_gluing_flow.py`、`partition_gluing_flow_resource.py`及`paper_phase/resume_flow_resource_20261023/audit_summary.json`。

本轮得到的结论不是已完成三个创新点，而是进一步缩小了有效机会：表驱动的正负资源推理对真实函数确实有用，但属于必须与基线共享的强控制；接下来的连接结构机制需要在它之后仍有额外收益。当前正在检查保留边端点朝向的安全对称消除，并研究控制变量改变所涉及的更小接口，避免将普通求解器配置改进包装为新贡献。

### 13.7 端点朝向控制后的结构证明与公平补强

对每条保留无向边，比较其两端到所有更早pins的双状态连接signature，按字典序选择朝向。只使用更早pins很关键：按edge顺序贪心翻转两端时，不会改变已经确定的前缀约束；每次翻转同时作用于两cofactor partition，文字身份不变。没有使用未来pins的随意lex，也没有按函数答案指定朝向。这是标准对称消除，作用在新的表示上，不单列创新。

`partition_gluing_prefix_sym.py`与独立driver完成全部42项NP3边界回归：22 SAT、20 UNSAT、零UNKNOWN，与枚举一致。两项精确literal-profile查询的18秒无proof比较中，ex275/R11从UNKNOWN变为5.024秒solver-UNSAT；o6/R13仍UNKNOWN。

独立证书轮得到全部三个分支的DRUP VERIFIED：

| 查询/分支 | 求解秒数 | 检查秒数 | 分支完整秒数 |
|---|---:|---:|---:|
| ex271_o6 R13，额外¬x0 | 10.894 | 6.824 | 19.275 |
| ex271_o6 R13，额外¬x3 | 24.984 | 23.043 | 49.940 |
| ex275_o1 R11，额外+x5 | 6.871 | 5.922 | 13.700 |

o6整个查询69.219秒，两个证书约73MB与149MB；ex275证书约15.6MB。控制profile覆盖依赖已完整审查的三输入枚举库，分支CNF的无解由独立检查器验证；不能将组合证据误写成一份从裸truth开始的单一DRUP。结合此前SAT上界，**ex275_o1在当前模型下最优值为12；ex271_o6已认证下界为14，14边可行性仍未知。** 这更新第13.6节较早的证据状态。

Mini也补上可安全施加的同类控制：用原wire-selector表示每条开关两端到source/sink及更早slot端点的prefix lex。12个已有SAT网络均能按slot顺序重命名并保持全truth BFS；固定重命名结构后，增强CNF的辅助流/割变量也全部SAT。Mini wire表示直接连线，不是等价关系，不能未经证明把它直接强制成partition clique；原约束还要求每根wire被某个ON-flow使用。

Mini＋必要文字＋全部三变量count cuts＋相同6维profile＋安全endpoint lex，在ex275/R11仍18.017秒UNKNOWN，增强CNF含78,738变量、578,847子句。这是一项超过已补同组对称控制的Mini配置的开发证据，仍只有一个实例；另做有限的literal-slot排序对照，防止把易受顺序影响的短时结果误当普遍优势。数据为 `resume_prefix_sym_{regression,perf,proof}_20261023/`、`resume_mini_profile_endpointlex_20261023/`与`resume_mini_endpointlex_correctness_20261023/`。

需要保留另一限制：排除完整六输入函数不等于证明一个不可约六变量冲突。后续检查它的五输入cofactor是否已有更小的结构阻碍；若存在，应把全函数下界解释成较小证据的提升，而非夸大需要联合分析的变量数。

### 13.8 排除表示、排序及输入行选择带来的混淆

对Mini再做两个预先固定的slot排序控制：按变量交错排列正负literal，以及反向交错排列。保留全部共同资源约束和安全prefix，不继续搜索最有利的随机种子。ex275/R11分别18.479和18.019秒UNKNOWN。短预算仍可能受顺序影响，这只排除了这两个具体配置；不能声称排除所有可能的Mini优化。记录为 `resume_mini_slotorder_controls_20261023/`。

随后实现**不消去control的单partition编码**：全部R条开关保留，只用一个共同连线partition，ON用reach、OFF用cut，加入相同精确literal预算与端点prefix。它与gluing的差别是是否将控制开关消去，而不是有没有等价类连线表示。12个已知真实SAT profile均在自由搜索下重新找到模型，耗时0.055–0.659秒，全truth BFS与CNF检查通过；两个小UNSAT profile另行DRUP VERIFIED。ex275/R11在全64行下仍18.273秒UNKNOWN，o6/R13为18.161秒UNKNOWN。前者约220,102变量、1,074,915子句，峰值327MiB。工件为 `direct_partition_sat.py`、`run_direct_partition_diagnostic_20261023.py`及`resume_direct_partition_20261023/`。

还补上与已知单极性一致的代表性truth行。对固定literal profile，仅有正literal的变量单调递增，仅有负literal的变量单调递减，双极性变量保持不动。目标truth先核对该偏序的单调性；只保留极小ON行和极大OFF行即可覆盖其他行。未使用变量先核对无关性，但当前实现保守地保持该位不动。所有模型最终仍按全部truth行检查。这是经典单调性化简，必须三个方法共用，不能列为新创新。针对Mini原wire必须被ON-flow使用的要求，另证明保留每个极小ON行的一条路径并移除其他wire即可保持全部truth，因此缩减行不会错误排除网络。

审查覆盖三输入的64种文字极性/缺失配置、256种truth，兼容目标对应125,932次候选truth比较；另有2,000个随机网络和12个真实SAT网络的路径保留、全truth BFS及固定结构CNF检查。实现为 `profile_representative_rows.py`，审查记录 `resume_profile_selected_rows_20261023/`。

ex275/R11因此由64行缩为同样24行，得到以下匹配18秒预算的开发对照（证明输出关闭，包含构造及求解开销）。三种方法均将各自CNF交给PySAT Glucose3，Mini使用原核心导出的CNF；此处没有将MiniSAT与Glucose的差别混作编码收益：

| 方法，均含同资源、精确profile、prefix与24行 | 结果 | 完整秒数 | 变量/子句数 |
|---|---|---:|---:|
| Mini原核心＋共同控制 | UNKNOWN | 18.110 | 33,258 / 231,149 |
| 单partition，保留全部开关 | solver-UNSAT | 11.760 | 86,130 / 423,049 |
| 双cofactor partition，消去control | solver-UNSAT | 2.168 | 详见逐查询JSON |

这些performance CNF未全部另行生成新证明；该函数11边无解已由前一节全truth证明独立确立。o6/R13所有输入均双极性，没有这种行缩减，gluing仍18.039秒UNKNOWN。记录为 `resume_mini_profile_representative_rows_20261023/`、`resume_direct_partition_rows_20261023/`、`resume_gluing_representative_rows_20261023/`。

这项消融修正了归因：partition表示加经典行缩减本身就能解掉这个实例，control消元在此基础上继续加速；不能把从Mini超时到2.168秒的全部差距都归给新的共享机制。至今仍只有开发样本证据，没有足够独立样本支持普遍优势。

### 13.9 五输入解释与仍未解决的上界

进一步核查发现ex275的六输入truth精确满足 `f=g(x0 OR x1,x2,x3,x4,x5)`，其中五输入g的truth为`0xfe03e0e2`，全部64行一致。R11唯一幸存profile投影为g的R10固定profile：正文字次数`[1,1,1,1,2]`，负文字次数`[0,1,1,1,1]`。该固定profile无解，性能轮1.921秒；另行证明全程4.817秒、DRUP VERIFIED。因此这里的六输入排除可解释为较小固定资源冲突的提升，不能称作不可约六输入发现，也不能据此断言g在所有其他profile下的最优值。

对o6的12个五输入cofactor穷举输入置换与输入取反，没有任何一个与已知最优12边的o5 NP等价。另一方面，加入prefix后继续尝试o6的14边上界，在27个幸存预算之间预设均分90秒，各分支仍UNKNOWN，总90.116秒。不能把“没有找到14边网络”升级成14边无解，也不反复追加同类短重启。记录为 `resume_five_input_lift_20261023/`与`resume_o6_prefix_upper_20261023/`。

### 13.10 控制接口oracle：可验证机制与端到端负结果

一个可明确写出的接口界是：设两cofactor partition为P0、P1，meet为M，最少控制开关数为 `k=2B(M)−B(P0)−B(P1)`。将每个meet块视为P0块与P1块之间的一条二分关联边；非平凡连通分量含e条边、v个顶点，其贡献为2e−v，且至少e/2。因此发生改变的meet块至多2k个。它只约束控制接口，**不能说明整个综合问题关于k固定参数可解**：即使只需一条控制边，`f=x AND g`仍包含任意g的综合困难。partition距离公式、普通控制消元与图连通学习均有先例，不把这些重新命名为贡献。

基于这个观察实现另一条精确路线：SAT master只生成固定保留文字的共同背景partition；对每个背景的全部truth行，oracle穷举给定极性最多0、1、2条控制边的所有端点选择。两条边时精确考虑串联组合，而非只对各单边结果做OR。若正反状态均可实现，则还原完整网络并逐truth BFS；否则给出覆盖**全部允许控制集合**的失败行集合。

从失败行进一步构造区域排除子句：ON行背景只变得更分离时，原本无法接通的控制集合仍失败；OFF行背景只变得更连通时，原本错误接通的控制集合仍失败。将ON旧连通块之间的P变量加入正literal，将OFF旧块内由当前true P构成的生成森林加入负literal，即得到可供SAT学习的解释。它比仅排除一个完整背景assignment更广；但“更短解释”不保证搜索更快。普通路径/cut学习已有[MonoSAT](https://github.com/sambayless/monosat)等工作，此候选若要有贡献，需要证明跨多行、量化所有控制集合的解释带来额外效果。

完整实现为 `interface_oracle_synthesis.py`，独立审查为 `audit_interface_oracle.py`。对候选失败mask去重、按连通块构造连通矩阵、优先检查控制预算较小状态，并在区域解释和完整assignment no-good对照中同时加入相同prefix。8个小固定profile全部得到正确决定；三个UNSAT例的全部新增oracle子句均经独立控制集合遍历检查，再对增广master生成DRUP并通过检查。parity7的固定profile例共7,606条lemma、414,226次独立控制集合检查。SAT全truth BFS。这里审查的是固定profile及其lemma，不自动覆盖所有文字预算。

但两个真实难例o6/R13与ex275/R11中，区域解释与完整背景no-good对照均30秒UNKNOWN。优化后的区域版本已排除约19,379和22,633个完整背景，平均失败核仅约2行，仍没有端到端优势。此前开销分析显示，oracle枚举/失败核及背景矩阵构建占主要时间，master SAT本身并非唯一瓶颈。**完整背景枚举版本暂时停止；不能把小例证书通过当作真实问题已取得突破。** 全部记录及证明链在 `resume_interface_oracle_20261023/`，技术附件为该目录`机制审查与证书定义.md`。

仅保留一个有具体因果假设的后续变体：在P尚未完全赋值时，ON行用“已true＋未赋值”构造乐观上界图，OFF行只用已true连线构造下界图；若即使如此仍无共同控制集合满足所有行，则可提前排除整个分支。使用CaDiCaL外部propagator时必须同时补同求解器的完整模型oracle对照，正确处理回溯、超时和子句检查。若仍不能减少真实难例的端到端时间，就停止此变体，而不是靠不断调参延续希望。

查重补充：[AAAI2015 SAT Modulo Monotonic Theories](https://ojs.aaai.org/index.php/AAAI/article/view/9755)已给出基于单调性的传播与学习；[TACAS2024 DRAT Proofs of Unsatisfiability for SAT Modulo Monotonic Theories](https://link.springer.com/chapter/10.1007/978-3-031-57246-3_1)进一步给出theory lemma的DRAT认证、单侧定义及动态Horn上界。因此“partial图上下界传播”和“给理论子句生成证书”本身均不能独立算作本工作的创新。可能保留的差异必须具体到跨truth行共享有限控制边集合的联合必要条件，并以超过通用方法的效果验证。

### 13.11 当前冻结评估计划

目前还不能声称已经形成三个创新点。下一步首先检验泛化：43个NSP NP代表中，开发已用01、02、03、10、11，剩余38个作为未参与调试的评估集；在运行前固定manifest、源代码哈希、分支顺序、共同预处理及每函数每方法18秒总预算（20秒外部硬限）。比较Mini、单partition和cofactor消元三者，全部共用适用的有界六维profile过滤、代表性truth行与prefix。从共同必要下界向目录给出的n/n+1上界搜索，目录上界只是终止边界，不替代算法实际生成的SAT网络。某分支UNKNOWN不能升高已认证下界，只有全部分支UNSAT才能报告整预算无解。

另外独立准备EPFL原始电路AIG的5/6叶cut函数层级，固定算术和控制电路各两个、每node保留cut数、按来源哈希选择及NP去重规则，不根据求解器输赢挑选。它衡量局部映射函数，不能与真实整电路输出混为一谈，也不代表已改善全芯片面积、时序或功耗。两项新评估的代码与选择规则在看到结果后不再修改；出现负结果照实保留。

冻结NSP manifest实际执行版为 `paper_phase/nsp_heldout_frozen_20261023_v2/manifest.json`，SHA-256 `d75d4b7fd572561d9798c1274bc06993a33c4bd85613f5c2b1c9dac23e79624c`。初版`nsp_heldout_frozen_20261023/manifest.json`（hash `72883d3ce0815f49ffa95065f24418efc9b4db0ddd983ef4d5ad5d80f75adbe5`）未启动任何solver即被审查修订版替代；v2仅澄清计时并增加记录/截止guard：**18秒为含预处理和构造的求解软截止，SAT验证允许使用20秒进程总硬限内的余量**，统一记录实际耗时、完成时刻及是否在18秒内，不声称所有验证严格在18秒内结束。

38个固定ID为04、05、06、08、09、12–35、37、38、43–48、50（原始目录编号非连续）。37个源码/库/原Mini源码及driver二进制文件复制入`frozen_bundle`逐文件hash，外部PySAT扩展亦锁定hash；统一Glucose3。主程序 `nsp_frozen_evaluation.py`、共同预处理 `fixed_profile_evaluation_protocol.py`。已有开发NSP01/02的6项集成smoke全部找到最优5并通过CNF与全truth BFS，单项0.062–0.119秒，随后启动114项评估。NSP目录多为单调函数，不能凭此一组结果代表双极性真实函数。

EPFL新tier提取完成：从[官方benchmark仓库](https://github.com/lsils/benchmarks)固定commit `82d8cc6910419298e713a46644ed59fd3df53038`，取算术目录文件名排序前两个adder/bar，以及控制目录前两个arbiter/cavlc。四个已有原始AIG均与官方固定版本SHA一致，原目录始终只读。每node至多16个cut、K=6，按固定规则截断；候选要求全部叶子essential、至少一个双极性变量；每电路每5/6叶分层最多2项，按固定来源哈希排序并全局NP去重，排除已有whole-output类。

实际得到12个函数：adder/bar/cavlc各4，5/6输入各6；arbiter无合格候选，保留空缺，没有事后换电路。所有32/64行均通过独立标量AIG仿真，保存cone重建再次通过，所有root可达原主输出。多数只有1–2个双极性变量，仅一个五输入函数全双极性，运行前不称它们为难例。cut的叶信号在局部truth中视为独立变量，不利用原电路可能存在的相关性或don't-care。

工件 `paper_phase/epfl_cut_tier_20261023/`，manifest SHA-256 `a9fa0175281a391b9b02455a2c9bdbb62144854d4cd9fb4ce8e9d3444ef068cf`，functions SHA-256 `3eb0388ca0f5e4f40c1a9fe0cccbced24c2c9d666d8ea18c949a81553835cfb4`。提交文档不附带原AIG；本地归档保留MIT许可证与来源。尚未运行该tier综合时预定：沿用冻结三个编码及控制，从必要literal下界搜索至下界+2，每函数每方法18秒；找不到只报告该区间UNKNOWN，不能推断整体不可实现。

### 13.12 Partial背景传播完成：停止该候选的依据

安装环境PySAT 1.9.dev17的CaDiCaL195外部Propagator接口实际可用。实现固定频率 `max(4,ceil(P变量数/8))` 个新赋值后检查partial，同时始终检查完整模型；保持相同solver、prefix、固定profile及控制oracle，以full-model模式作对照。CaDiCaL195不支持interrupt，因此每256个conflict分段solve_limited，callback记录截止状态，外部32秒硬限；任何截止后未经验证的返回均覆盖为UNKNOWN。代码 `partial_interface_oracle.py`、独立检查 `audit_partial_interface.py`。

两模式小回归各8/8符合预期；parity7固定profile的UNSAT由full的2.030秒降至partial的0.792秒，parity8 SAT却从1.642秒变为3.792秒。两模式各3个UNSAT的全部lemma都逐控制集合直接图遍历检查，增广master DRUP也全部VERIFIED；10个SAT模型全部truth BFS。完整证明链仅覆盖这些小例；真实UNKNOWN只做包含partial/full两类lemma的固定抽查，没有冒充完整证书。

| 真实固定profile查询 | 同CaDiCaL full-model | 同CaDiCaL partial |
|---|---|---|
| ex271_o6 R13，首幸存profile | UNKNOWN / 30.177秒 | UNKNOWN / 30.138秒 |
| ex275_o1 R11，唯一幸存profile | UNKNOWN / 30.179秒 | UNKNOWN / 30.124秒 |

o6的partial冲突共6,148次，发生时平均92.66%的P变量已赋值，中位93.94%；ex275有5,291次，平均93.59%、中位95.42%。两例各仅2次冲突发生在≤50%赋值时。说明乐观图松弛尚不能足够早地识别不可能背景；并非没有调用传播。背景连通矩阵构建约占22/30秒。即使以后用更快语言降低该开销，也不能据此预言更强剪枝或新算法贡献。**当前partial版本没有真实端到端收益，按预定标准停止，不继续调触发频率、随机种子或无限延长时限。** 工件 `paper_phase/resume_partial_interface_20261023/`保存源码hash、逐例结果、lemma和检查报告。

### 13.13 多control扩展为何不能直接套用现有公式

纯理论审查明确了一个边界。若所有节点都可见、不允许隐藏节点，对两个control的四个目标partition P_ab，定义 `Qx_a=P_a0∧P_a1`、`Qy_b=P_0b∧P_1b`，则存在相应开关网络，当且仅当每个状态满足 `P_ab=Qx_a∨Qy_b`。必要性来自每条x_a边的两端在两个y状态中均须相连，y同理；充分性是在各Q块中加入相应literal的生成树。全状态meet M已包含在各Q中，可免费保留而不改变判据。这是存在性判据，并不是最低成本公式。

允许隐藏节点后，最小反例只需两个可见端点、一个隐藏点、两条串联边x和y。只有11状态接通，四个Q却都离散，因此上述可见partition判据会错误拒绝一个两边网络。代数原因是“限制到可见节点”不与partition join交换。两可见端点的16种两输入状态函数中，零隐藏点只能实现10种OR-of-literals/常量；其余4种minterm及XOR/XNOR均需要隐藏点，一个隐藏点加允许的平行边已足够。单control下的隐藏点消除证明不能直接推广。

即使没有隐藏点，也不能将各Q的树rank简单相加计算成本。三可见点s,u,t，取三条边 `{x:s–u, y:u–t, ¬y:s–t}`，已有3边实现；Qx_1全连通rank为2，Qy_0和Qy_1各rank为1，简单相加却为4，因为一些连接由不同状态下的替代路径共同产生。完整短证明见 `paper_phase/resume_mechanism_20261023/multi_control_boundary_note.md`。这些反例用于约束下一步推导，不声称是原创定理，也未据此启动新的大规模搜索。

### 13.14 冻结NSP留出评估完成：收益主要来自partition表示

38函数×3方法共114项完成，零worker错误。函数包含5个五输入、33个六输入；未根据这批结果修改策略、分支顺序或超时。113个SAT结果均检查每个CNF子句及全部truth行BFS，唯一UNKNOWN来自Mini的NSP18。`OPTIMUM_FOUND`表示完整覆盖下界与新构造上界相等；本轮求解关闭proof，因此涉及solver-UNSAT的下界仍为solver-only，不冒充所有最优性均已DRUP认证。

| 方法 | 找到最优网络 / 38 | UNKNOWN | worker中位秒数 | 全部进程累计秒数 | 峰值MiB |
|---|---:|---:|---:|---:|---:|
| Mini原核心＋共同控制 | 37 | 1 | 4.523 | 197.007 | 62.8 |
| 单partition＋共同控制 | 38 | 0 | 0.146 | 9.200 | 47.2 |
| cofactor消元＋共同控制 | 38 | 0 | 0.166 | 10.444 | 44.8 |

共同完成的37例，以实际全进程时间计算，单partition相对Mini的几何平均加速15.09倍，cofactor为13.55倍。不能仅报这个大比值：在二者直接比较的38例中，单partition快24例，cofactor快14例，且本批绝对时间已经很短，解释器启动、库加载等占相当比例。本结果支持**共同控制下partition类表示比当前Mini编码更高效**，但不支持**cofactor消元普遍优于不消元**。直接net/partition的表示思想已有近年工作，也不能因为快15倍就把它单独包装成全新论文机制。两批表中RSS为Python worker记录值，不是含Mini导出子进程的进程树峰值；没有据此提出内存优势结论。

NSP18中Mini已完成6边无解、下界为7，但18秒内未找到7边网络；另外两种方法约0.145/0.159秒找到7边模型。其余已完成函数均达到目录标注计数，但目录数字本身没有替代模型构造和下界证明。小目录、单调性和已知低开关数限制了外推，不能用于宣称已覆盖一般双极性网络，更不能用于芯片PPA结论。

完整逐例表、实际进程时间、全部SAT模型及统计在 `paper_phase/nsp_heldout_frozen_20261023_v2/{REPORT.md,aggregate.json,results/}`。据此保持原定EPFL评估协议继续，而不是在NSP上调pivot/种子获得更好数字。

### 13.15 新公开先例：TransiLog不是纯愿景，也不是同成本精确基线

读完LATTE2026三页原文 [A Position on Language Abstraction for Custom Circuit Design](https://capra.cs.cornell.edu/latte26/paper/latte26-final8.pdf)，并由[作者主页](https://zsisco.net/)定位到[公开TransLog-Demo](https://github.com/Boucii/TransLog-Demo/tree/demo-public)。它已有结构DSL、晶体管级e-graph rewrite和prototype实验，不应误写成只有设想。原文较早版本报告NSP53合计395只、约三分钟；当前代码pin `0d126d82d882ee035f3d5812d54ca9f9559728c7`附带逐例报告为367只、16.379秒。这是作者存档结果，本轮没有独立运行，也不能跨机器与我们的时间比。来源：[固定版NSP report](https://github.com/Boucii/TransLog-Demo/blob/0d126d82d882ee035f3d5812d54ca9f9559728c7/NetlistOpt/rpt/nsp53_orig_dnf_best_report.txt)。

公开demo含Rust源码、Cargo.lock、NSP53和3984个四输入P-class输入及结果，但README明确没有公开delay/simulation/SPICE流程，当前文件树未见LICENSE/package license，故这里只读审查，不称已核验宽松许可。[固定版README](https://github.com/Boucii/TransLog-Demo/blob/0d126d82d882ee035f3d5812d54ca9f9559728c7/README.md)。它可作为后续启发式上界候选，不能直接当同域最优值基线：`join`用于组装PUN/PDN，`!`计2只反相器并考虑cell共享；我们的join是边界pin等价关系并集闭包，当前输入反相免费、仅单网络。NSP流any-root和四输入流cell-root也不同。[作者语义定义](https://github.com/Boucii/TransLog-Demo/blob/0d126d82d882ee035f3d5812d54ca9f9559728c7/NetlistOpt/doc/semantics.md)。

代码的`shannon-reduction`只是`xy+x¬y→x`，另有bridge改写等，没有在核查版本中看到本文的cofactor-pair边界partition、单control隐藏点消除及rank成本覆盖；这限定了直接重合范围，不能据“未见”就认定我们新颖。[固定版规则](https://github.com/Boucii/TransLog-Demo/blob/0d126d82d882ee035f3d5812d54ca9f9559728c7/NetlistOpt/src/rules.rs)。其context-DP/fixed-point提取文档也区分局部去重、未精确收敛fallback；e-graph内的minimum提取不等于所有网络拓扑的最优证明。[提取器说明](https://github.com/Boucii/TransLog-Demo/blob/0d126d82d882ee035f3d5812d54ca9f9559728c7/NetlistOpt/doc/min_transistor_fixed_point_extractor.md)。因此“加入网络rewrite/equality saturation/cell共享”不能作为新的第三创新点。完整只读记录在 `paper_phase/transilog_review_20261023/技术附件.md`。

### 13.16 冻结EPFL局部函数评估完成

新的12函数×3方法共36项完成，零worker错误。wrapper从NSP v2冻结bundle按hash复制三个编码、共同控制、库和Mini driver，只有来源元数据与搜索区间变为预先指定的LB至LB+2，余量0/1/2完整枚举；没有根据NSP结果调编码。44个文件锁定hash，manifest为 `paper_phase/epfl_interval_frozen_20261023/manifest.json`，SHA-256 `c6d049a8b0189a38399cd26e47dba1b310e7285cd9fe35292b86144582898a5b`。全部三层profile/rows准备成本即使在首层SAT也照计入，三法相同。

| 方法 | 找到最优网络 / 12 | UNKNOWN | worker中位秒数 | 全进程累计秒数 | 峰值MiB |
|---|---:|---:|---:|---:|---:|
| Mini原核心＋共同控制 | 7 | 5 | 3.559 | 100.013 | 143.8 |
| 单partition＋共同控制 | 12 | 0 | 0.125 | 3.916 | 133.1 |
| cofactor消元＋共同控制 | 12 | 0 | 0.113 | 3.485 | 98.8 |

Mini未完成的五项为adder_n5_2、adder_n6_1、adder_n6_2、bar_n6_2、cavlc_n6_2。其中adder_n5_2是本tier全双极性的五输入函数，两种partition法分别1.415/1.052秒构造10边网络，达到必要literal下界；Mini18秒UNKNOWN。cavlc_n6_2的必要下界为7，两种partition法先排除7边、再构造8边模型，分别0.201/0.215秒；这条非平凡下界在性能轮仍solver-only，正另行独立审查。其他11个函数均在必要下界构造网络。所有SAT检查全部CNF与全truth BFS。

cofactor法在12例中快8例，但全进程几何平均仅约1.056倍；单例时间很短，不能由此推断稳定优势。NSP和EPFL共同支持表示方式的搜索效率收益，尚未证明消元机制广泛带来额外提升。两批合计150项性能运行，不包含单独的smoke、proof及模型可表示性检查；它们是小规模精确综合任务，不是面积/时序改善的完整电路实验。逐例报告与模型在 `paper_phase/epfl_interval_frozen_20261023/{REPORT.md,aggregate.json,results/}`。

为进一步排除基线适配问题，正在把两批中Mini未完成的6个函数的direct已知SAT网络固定进相同Mini CNF，检查slot对应、路径裁剪、prefix后辅助变量是否仍可满足。这只检查配置能否表示已知解，不把带答案的求解计入算法性能。此外，另建明确同域的net-ID控制，与近年terminal-to-net表示接近，检验两个较难开发查询中的cofactor收益是否还能由更直接的等价关系编码解释；它不是Free-Topology论文的复现，也不计作新创新。

### 13.17 新结果的独立证书与Mini可表示性审查

EPFLcut_cavlc_n6_2的7边下界已完成独立认证。两种partition编码分别在独立proof轮生成DRUP，drat-trim均VERIFIED，完整进程0.681/0.632秒；变量/子句数与对应性能轮一致。R7只有一个必要literal profile、没有被库过滤的其他预算，使用8个经单调性证明充分的代表行。因此证据链为：必要literal覆盖唯一7边预算、代表行等价性、两个编码各自的完整性、DRUP无解检查；8边上界来自全部64行BFS。**该局部函数在当前模型下最优值为8**。证书检查不混入§13.16的性能数据，目录 `paper_phase/epfl_cavlc_lower_bound_certificate_20261023/`。

另外，两批中Mini未完成的6个函数，其direct见证全部成功嵌入同配置Mini CNF。具体操作为：按固定文字槽对应端点；保留各代表ON行一条路径的wire并集；按原prefix安全定向；裁剪前后均全truth BFS；固定全部wire与label，只让辅助变量自由，并验证返回model满足每个CNF子句。6/6均REPRESENTABLE_SAT，完整检查每例0.271–0.982秒，辅助SAT约1–5毫秒。

其中5例对应Mini超时的相同R与profile；cavlc_n6_2检查的是8边见证，而Mini性能轮尚停在7边未知，明确不混作相同固定预算。该审查排除了“这些新见证被基线适配意外排掉”的解释；它不是完整的编码等价证明，也不是让Mini从零找网络的速度结果。全部映射、wire裁剪、CNF及BFS证据保存在 `paper_phase/frozen_mini_witness_embedding_20261023/`。

### 13.18 更强的net-ID控制及先例边界

[Free-Topology作者工件](https://github.com/b8kang/Free-Topology-TNS-ICCAD2026/tree/441d7cc5c1e588cb0d2ef599da65793fa4db7d86)没有完整综合源码，但正文已有terminal-to-net、编号及朝向约束；L2L源码也已有端点到net的整数表示。两者包含完整CMOS或节点使用/anchor等条件，不能直接照搬成当前单网络、固定profile问题。尤其声明较多net并假定可闲置，与原来的“所有internal net均使用”条件不同；某些先按gate排序的slot，也不能再无证明地强制首尾slot锚定在电源/输出。完整审查在 `resume_partial_interface_20261023/FreeTopology与terminal_net编码审查.md`，只审查作者公开内容，不称已检查不存在的源码。

据此独立实现**同域net-ID控制**，不是复现上述论文：为每个pin选择一个net ID，将E定义为ID相等，替代显式三元组传递性。最多2R+2个标签足以容纳全部pin partition，内部ID按首次出现顺序编号；先满足E上的端点prefix，再按最终pin顺序重命名net，不改变E，因此两种对称控制相容。其余固定profile、代表行、ON reach/OFF cut、onehot文字变量、g3全部保持。常一允许源汇同net，单独处理。编码为 `direct_partition_netid.py`，SHA-256 `5f1d12f8a6943601489cf2af3725b0de3650c1ba96a68d8d0b957485ca75f4d8`。

12个已知开发SAT结构固定E后的可扩展性检查全部通过；另对相同12个profile自由搜索，12/12重新找到模型，0.049–0.362秒，CNF、文字计数及全truth BFS全部通过。两个小UNSAT固定profile另获DRUP VERIFIED，零边常零/常一也通过。这些是正确性回归，不能混入真实难例的性能统计。

两个既有开发查询的18秒求解软限/20秒硬限结果：ex275/R11为6.992秒solver-UNSAT，峰值157.7MiB；o6/R13为18.091秒UNKNOWN，首profile耗尽总预算，不能声称整个R13已排除，峰值408.4MiB。前者比原direct约11.760秒更快，但仍慢于cofactor约2.168秒。它说明已知net表示解释了部分收益，当前仍只有一个开发实例的额外消元差距，不能按单次测量宣称稳定倍数。没有用此编码重新调NSP/EPFL，也没有事后向留出表追加有利个例；本轮不继续调参。完整工件 `paper_phase/resume_direct_netid_20261023/`。

### 13.19 串联长度方向的新检查：必须区分两种问题

继续寻找与实际网络约束有关的切口时，核对[MiniTNtk原文§IV-B](https://fchxwh.github.io/preprint/ICCAD_2023_MiniTNtk__An_Exact_Synthesis_based_Method_for_Minimal_Transistor_Network_Generation.pdf)及源码发现：Eq.(15)只要求每个ON输入存在一条长度≤H的选中flow；随后程序另枚举全部文字一致、可能同时导通的simple源汇路径，检查其中最长者≤H。后者严格更强。上游最多检查50轮、每次阻断整个SAT assignment，超过后可能增加R，所以这段有限流程不能自动作为带stack的最优性证明。该限制不影响本文此前明确关闭stack的实验。

源码的CLI第5参数是depth开关，H实际由onset SOP的最长项决定，并非直接读用户给定数值；当前thin exporter没有H。给新exporter调用公开的 `CreateKLimitedPathConstraints(H)`只能重用前置短flow条件，不能声称复现了最终的完整stack语义。精确行号与手工例子见 `paper_phase/stack_semantics_static_review_20261023/README.md`。本文后续分别称“最短导通见证上限”和“所有可导通simple path上限”，不混用。

隐藏的同色control星形节点可通过收缩一条开启边消除；新simple path可以提升回旧simple path，长度不会变短，因此这一步对强上限仍安全。但此前把共同免费连线直接取最大meet M的做法**不保持强上限**。一个固定接口反例只需五点a,b,c,d,u，源c、汇b：控制+x边a–c、c–b；控制¬x边a–d、d–b；保留三条+y边c–d、d–u、u–a。原truth为x OR y，所有可导通simple源汇路径最长2；两个control状态的meet却把a、b放入同块。免费识别a=b后，纯y路径c–d–u–a(=b)长3。新增控制边无法删除这条路径，所以这不是补边就能修复的问题。

这只否定固定保留结构下的安全变换，不是证明函数本身需要7边；x OR y全局仅需两条并联边且H=1即可。为独立核实，在.44穷举该固定接口的全部10种森林组合：B=M时唯一组合控制成本2，最短见证H2通过、强H2失败；B将a/b分开时9组合均控制成本4，最短H2有6个通过、强H2仅1个通过。所有组合×四输入的truth均为0xe，DFS与独立有序点子集逐path检查完全一致。该验证约0.0016秒、RSS约13MiB，未调用solver。记录为 `paper_phase/stack_interface_review_20261023/fixed_counterexample_full_enumeration.json`。

由此得到的精确局部形式是：保留共同partition B为M的细分，不能强制B=M。固定B、P0、P1，**若存在强stack可行控制图**，则可逐状态删除控制cycle而保持其closure，全部输入的功能不变、simple paths只会减少。因此存在每色都是B商图森林的可行代表，最少控制边恰为 `2|B|−|P0|−|P1|`；仍须寻找符合强约束的具体森林，不能任取生成树。相对最大meet的rank下界，细分额外代价为 `2(|B|−|M|)`，两极性各增同数。该等式仅适用于固定保留图和固定两状态partition，不能推广成任意函数最优网络之间的偶数差。

这里的forest结论针对成本最小化或≤预算问题，允许删去冗余控制边；不表示每个强制精确literal multiplicity的诊断模型自身都已是森林，它可能含冗余边或padding。Balanced增量比较的也是同一固定接口的最小可行控制成本。

这与前面最短见证约束下“可能需要cycle缩短多条路径”的结论并不矛盾：两个问题的约束方向不同。当前只是澄清模型、证明安全性边界和一个极小固定接口验证，还不是第三项已成立创新或真实基准加速。下一项仅验证一个具体候选：对违反强上限的simple path学习结构子句；除保留路径相接的正E前提，还必须保留路径顶点两两不等的前提，否则未来合并非相邻点可能令该路径不再simple。强对照将是整个已解码拓扑no-good，而非故意只与上游含辅助变量的完整SAT assignment阻断比较。先做小图子句正确性审查，再决定是否值得性能实验。
