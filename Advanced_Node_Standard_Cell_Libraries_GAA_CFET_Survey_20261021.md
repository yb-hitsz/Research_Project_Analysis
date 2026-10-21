# 先进工艺标准单元研究：小于 7 nm、GAA、CFET 的库开发、表征与优化

调研日期：2026-10-21。

同日补充：后续调研新增核查了 GT2N 公开 2 nm GAA 平台，并进一步检查 PKP 手册与正式申请渠道。相关资源已补入本报告；资料获取、规则缺口和验证方法详见[没有晶圆厂 PDK 时的研究策略报告](Advanced_Process_Research_Without_Foundry_PDK_Sources_Validation_and_Strategy_20261021.md)。

## 1. 先给结论

**这个方向已有相当系统的研究，而且研究对象远不止新晶体管本身。** 与标准单元直接相关的工作主要集中在以下方面：

1. **新器件对应的完整标准单元库。** 包括晶体管尺寸选择、单元架构、版图、寄生提取、时序功耗表征，以及综合和布局布线验证。NS3K、GT3、PROBE3.0 是不同类型的代表。
2. **低轨高单元的布局布线。** 当单元高度降到少数金属轨道，问题往往不是还能否放下晶体管，而是是否能连接、是否有可访问的引脚、放进实际设计后是否仍能布通。
3. **CFET 上下层器件、折叠和连接的联合优化。** 把 NMOS、PMOS 叠起来并不自动得到优秀的单元，需要重新处理配对、栅连接、跨层连接和不同尺寸器件的排列。
4. **背面供电之后的背面信号布线。** 研究已从把电源移到背面，推进到把信号、时钟和部分单元内部连接移到背面，并生成不同引脚位置的单元变体。
5. **库和设计共同优化。** 单个 cell 面积更小，不保证模块面积更小；不同 nanosheet 宽度、不同高度、不同引脚变体，需要结合实际综合和布线需求选择。
6. **考虑新寄生和工作条件的表征与优化。** 包括布局相关效应、背面接触、局部互连、低温条件，以及更完整的 timing arc、负载、输入转换时间和 PVT 条件。

**对你最直接的建议：先从公开 2/3 nm GAA 研究平台上的单元版图、寄生与性能优化入手，再考虑 CFET。** GAA 路线已有较完整的公开库和模型，更容易把仿真、表征和模块级验证连起来；CFET 的研究空间很大，但工艺规则、寄生模型和完整签核条件更容易成为障碍。这个优先顺序是结合可获得资源作出的研究建议，不是器件优劣排名；具体选择仍取决于工具、许可和参考流程的可运行性。

下文将“已有论文做了什么”“资源实际公开了什么”“还值得研究什么”分开，避免把路线图、预测性实验和真实工业工艺混为一谈。

## 2. 本报告的筛选口径

### 2.1 两条准入条件，满足其一即可

| 条件 | 纳入 | 不纳入或单独说明 |
| --- | --- | --- |
| 工艺节点 | 明确采用标称小于 7 nm 的工艺或预测模型，例如 5 nm、3 nm、2 nm | 普通 7 nm FinFET、ASAP7 本身不满足这一条 |
| 器件结构 | GAA nanosheet、forksheet、CFET、相关双面器件、垂直 GAA 等新结构 | 仅在标题中说“面向先进节点”，但实际仍是普通 7 nm FinFET，不能自动算作符合范围 |
| 研究内容 | 标准单元架构、库开发、自动生成、尺寸与布局优化、表征、库选型、库与模块协同 | 仅开发晶体管、仅展示器件 I-V 曲线、仅 SRAM bitcell，不作为主体 |

特别注意：**某篇 CFET 论文使用 ASAP7 的逻辑网表作为输入，仍可因 CFET 架构符合第二条；但不能因此宣称它验证了真实的 3 nm 或 2 nm 工艺。** 逻辑网表来源、几何规则来源、器件模型来源，是三个不同问题。

### 2.2 怎样理解论文质量和证据强度

优先查阅 DAC、ICCAD、DATE、ASPDAC、ISPD、TCAD、TVLSI，以及与工艺和电路交界紧密的 IEDM、VLSI Symposium、TED。JXCDC 等专业期刊中的直接相关论文也纳入；SOCC、JSTS 的相关工作明确注明 venue，不一概称作“顶会顶刊”。

证据分成以下几类：

| 类型 | 可以支持什么 | 不能直接支持什么 |
| --- | --- | --- |
| 工业工艺设计研究 | 对具体工业规则和模型的适用性 | 未说明时不能推断所有结果来自流片测量 |
| 预测 PDK 下的完整设计流程 | 同一假设下的算法比较、设计探索、库到模块的影响 | 不能保证在商业工艺投片可制造 |
| 架构或算法研究 | 新布局约束、求解方法、引脚和布线收益 | 不能替代真实工艺的全套 DRC/LVS/PEX |
| 官方摘要或报告 | 已公开的问题、方法和摘要级结果 | 本次未获取全文的细节不补猜 |
| 预印本 | 新进展和潜在竞争工作 | 不当作已经通过顶会顶刊评审的成果 |

本次以论文、作者机构、作者公开稿、官方会议材料和代码库为主要来源。不是覆盖所有数据库的系统综述，也没有在本次调研中重新运行这些工具。文中百分比均为**对应作者在其设定和基线上报告的结果**，不能直接横向排列成算法排行榜。

## 3. 先弄清楚：先进工艺具体改变了标准单元的什么

| 变化 | 对标准单元设计的直接影响 | 对应的算法问题 |
| --- | --- | --- |
| FinFET 转为 nanosheet GAA | 尺寸不再只是 fin 数，sheet 宽度、层数等会影响驱动和电容 | 尺寸选择、驱动档位设计、不同宽度的混合库 |
| Forksheet | 允许更紧凑的 n/p 排列和不同宽度配置，具体受工艺限制 | 库内宽度配置、面积与驱动折中 |
| CFET 的 n/p 垂直堆叠 | 上下层器件共享投影空间，连接未必同样简单 | 折叠、配对、上下层对齐、共同栅或独立栅、跨层连线 |
| 单元轨高降低 | 可用局部布线和引脚位置减少 | 引脚可访问性、布局与布线联合求解、混合高度 |
| 埋入式电源轨 BPR | 电源不再占用相同的正面空间 | 单元高度、供电、寄生和局部布线重新权衡 |
| 背面供电 BSPDN | 正面部分布线资源释放，同时引入背面接触与供电约束 | 单元架构、供电网络与时序共同设计 |
| 背面信号布线 | 同一逻辑 cell 可以有不同前后表面引脚配置 | 单元变体生成、库压缩、变体选择与模块布线 |
| 局部互连和布局相关效应变强 | 相同逻辑和近似晶体管尺寸不保证相同延时 | 寄生提取、布局排序、局部 RC 建模与表征 |

**5T 不等于 5 nm。** T 通常描述单元高度相对于某层布线节距的关系，但不同论文可能计入不同电源轨或采用不同参考层。比较 4T、5T、7.5T 时，必须一起检查实际高度、金属节距、电源位置和允许的布线层。

**CFET 与 GAA 也不是完全并列、互斥的分类。** GAA 描述栅对沟道的包围结构；CFET 强调互补器件的垂直堆叠，堆叠的器件本身可以是 nanosheet GAA。垂直沟道 GAA 与“把 n/p 器件上下叠放”也不是同一件事。

## 4. 优先阅读的十条线索

以下是阅读顺序，不是质量排名。完整标题和来源见后续对应小节。

| 优先级 | 论文或系列 | 最值得看什么 |
| --- | --- | --- |
| 1 | NS3K，TVLSI 2023 | 一套 3 nm nanosheet 库怎样从单元落实到模块 |
| 2 | Nanosheet/Forksheet Width Modulation，TVLSI 2022 | 器件宽度变化怎样转化成“混合库选择”问题 |
| 3 | GT3，TED 2025；PKP，2026 | 现在能获取哪些先进工艺研究资源 |
| 4 | CFET Design Enablement，DATE 2022 | CFET 单元的实际设计约束和流程 |
| 5 | CFET SMT，TVLSI/JXCDC 2021 | 为什么 CFET 的布局和布线要共同处理 |
| 6 | CFET-FP，DAC 2024 / TCAD 2026 | 折叠、放置与可布线性之间的关系 |
| 7 | Half-Height Double-Row CFET，ICCAD 2025 | 单元架构如何与模块布局结合 |
| 8 | CFET Backside Library，DAC 2025；ASPDAC 2026 | 背面互连如何影响库和引脚访问 |
| 9 | Dual-Sided 3D-Stacked Cell Synthesis，ASPDAC 2026 | 双面引脚、布局和布线如何自动生成 |
| 10 | PROBE3.0，TCAD 2024 | 怎样避免仅用 cell 面积判断技术是否有价值 |

## 5. GAA、nanosheet 和小于 7 nm 的标准单元库

### 5.1 NS3K：完整 3 nm nanosheet 标准单元库

**T. Kim et al., “NS3K: A 3-nm Nanosheet FET Standard Cell Library Development and its Impact,” TVLSI, 2023.** DOI: 10.1109/TVLSI.2022.3229442。

论文把 3 nm nanosheet 器件、单元架构、BPR 和标准单元库连接起来，并在模块级设计中评估效果。它不是只展示一个反相器，而是研究新器件怎样变成可供综合和物理实现使用的库。4T 单元和局部互连变化是重要内容。

作者相对于其 5 nm FinFET 基线报告了较大的面积和功耗改善，其中面积下降 47.6%、功耗下降 27.4%。**这些是器件、单元与互连共同变化的流程结果，不能解释为某一个布局算法独立带来的收益。** 这是预测性研究库，本次没有核实到与论文完全一致、可直接下载运行的完整公开签核包。ISCAS 2021 的同名早期工作应视作同一研究线索，而非另一份独立验证。[机构论文记录](https://research.knu.ac.kr/en/publications/ns3k-a-3-nm-nanosheet-fet-standard-cell-library-development-and-i/)

### 5.2 不同 nanosheet/forksheet 宽度组成混合库

**G. Sisto et al., “Evaluation of Nanosheet and Forksheet Width Modulation for Digital IC Design in the Sub-3-nm Era,” TVLSI, 2022.** DOI: 10.1109/TVLSI.2022.3190080。

这篇与标准单元库选型非常相关。研究为不同器件宽度建立库，让综合和实现流程同时使用低电容单元与高驱动单元，而不是整个设计只使用一种宽度。其研究采用 5T 单元，库覆盖 80 个单元；论文中的 imec iN3 对应其 2 nm 技术设定，不能只按名称中的数字判断节点。

在其比较条件下，混合宽度方案相对于单一高驱动宽度库，在相同频率下分别降低约 13% 和 16% 功耗，对应 nanosheet 和 forksheet。关键不是“宽一点一定更快”，而是**不关键的位置用较小输入电容，关键位置保留较强驱动**。因此，“给 GAA 单元提供多种尺寸，再做库混选”已有明确先例。[论文](https://ieeexplore.ieee.org/document/9834262/)；[作者公开稿](https://imec-publications.be/bitstreams/45c3c2ee-f0d2-48a8-a191-bc17f415dd97/download)

### 5.3 三星 3 nm GAA 的局部互连优化

**G. Yang et al., “Standard Cell Design Optimization with Advanced MOL Technology in 3nm GAA Process,” VLSI Symposium, 2022.** DOI: 10.1109/VLSITechnologyandCir46769.2022.9830450。

研究对象是标准单元中的连接方式，而非重新发明晶体管。通过先进 MOL 层连接 p/n 器件的漏端，减少连接电阻并优化单元结构。作者报告连线电阻下降超过 20%，以及组合单元和新触发器的速度收益。

它说明先进节点中，**器件参数相同，改变单元内的接触和局部连接仍可能明显改变时序**。这也是只看网表和器件尺寸的 PPA 模型容易遗漏的信息。该工作来自工业工艺环境，不意味着对应规则和库可以公开取得；本报告不将未核实的电路结果称为流片测量。[机构论文记录](https://pure.uos.ac.kr/en/publications/standard-cell-design-optimization-with-advanced-mol-technology-in/)；[官方会议日程](https://www.vlsisymposium.org/wp-content/uploads/2022/04/22-program-rev6.pdf)

### 5.4 自动版图生成器迁移到三星 SF2

**I. Hwang et al., “2nm GAA Cell Architecture Exploration Driven by Automated Standard Cell Layout Generator for Design-Technology Co-Optimization,” SOCC, 2025.** DOI: 10.1109/SOCC66126.2025.11235359。

这是值得读的工业适配案例，但其 venue 是 SOCC，不能写成 DAC 或 ASPDAC。作者把原来针对 7 nm 规则的单元生成器扩展到三星 SF2，需要处理 nanosheet 架构、SF2 设计规则和单元内部布线约束。

626 个 gate-type 网表中生成了 494 个满足设计规则的单元；与 30 个手工单元比较，自动结果平均延时长 1.5%、面积大 0.59%，漏电略低。**这更接近“自动化做到接近人工，同时便于探索规则”，而不是所有 PPA 都超过人工。** 对你而言，它证明已有生成器可以迁移，但迁移不是把 PDK 路径换一下。[作者机构记录与摘要](https://snu.elsevierpure.com/en/publications/2nm-gaa-cell-architecture-exploration-driven-by-automated-standar/)

### 5.5 2026 年 2 nm GAA：跨行借用空间，而非继续硬压高度

**“High-Performance 2nm GAA Standard Cell Architecture Exploiting Backside Power Delivery via Dynamic Area Borrowing,” VLSI Symposium, 2026, T2.2；三星团队。**

官方摘要指出，5T 单元限制了可用 active width。借助背面供电和直接背面接触，研究允许通过行边界附近的空间配置容纳更大的有效器件宽度，在保持统一单元占地的条件下改善驱动。摘要报告单元延时降低 5%–9%，以及代表性 ARM 模块最高频率提高 9.8%。

这是**标准单元架构与合法拼接关系一起设计**的例子，不应理解成所有单元都可以自由越过边界。本次核实到官方会议摘要，未获取全文和完整验证设置；会议信息页可能随年度更新重定向，引用具体实现前应再获取论文。[官方报告页](https://vlsi26.mapyourshow.com/8_0/sessions/session-details.cfm?scheduleid=293)

### 5.6 BPR 下的 sub-3 nm 单元 PPA 分析

**J.-S. Yoon et al., “Performance, Power, and Area of Standard Cells in Sub 3 nm Node Using Buried Power Rail,” TED, 2022.**

该工作结合器件模型、寄生提取和库表征，比较 FinFET 与 nanosheet 单元。价值在于把电流优势与接触、电阻、电容一起计算，而不是从孤立器件电流直接推断 cell 延时。

在特定尺寸与负载条件下，新器件的局部优势不必然转化成整体 RC 延时优势。适合用来建立一个正确的研究习惯：**比较器件或单元架构，至少要保持面积、驱动或性能约束中的某一项一致，并把寄生纳入。** 这是仿真和模型驱动的比较，不是两个商业库的直接实测排名。[作者公开全文](https://www.gtcad.gatech.edu/www/papers/ted22-a.pdf)

### 5.7 历史参照：5 nm 库的尺寸与电压优化

**Q. Xie et al., “5nm FinFET Standard Cell Library Optimization and Circuit Synthesis in Near- and Super-Threshold Voltage Regimes,” ISVLSI, 2014.** DOI: 10.1109/ISVLSI.2014.101。

这篇不是最新成果，但说明“先进节点的库尺寸优化、低压表征和电路综合”已研究多年。其 5 nm 是当时的预测模型，不能写成 2014 年已经有量产 5 nm 标准单元库。建议只作为历史背景，不用它的跨代大幅度收益为今天的方法背书。[作者公开全文](https://www.mjdousti.com/papers/finfet/finfet-opt-isvlsi14.pdf)

## 6. CFET：从单元架构到自动布局布线

### 6.1 低寄生 CFET 标准单元库

**E. Park and T. Song, “Complementary FET (CFET) Standard Cell Design for Low Parasitics and Its Impact on VLSI Prediction at 3-nm Process,” TVLSI, 2023.** DOI: 10.1109/TVLSI.2022.3220339。

该工作把 CFET 单元结构、寄生与模块级 PPA 联系起来，而不是仅证明 n/p 器件能叠放。与其 nanosheet 基线相比，作者报告模块面积降低 48.2%、功耗降低 29.4%。

读这篇主要看：哪些连接因上下堆叠而缩短，哪些连接反而增加，以及这些变化怎样进入库表征。不同器件架构和单元结构一起变化，不能把上述收益都归因于布局算法。本次未核实到完整公开 PDK 与生成器。[论文](https://ieeexplore.ieee.org/abstract/document/9956796)

### 6.2 CFET 的设计使能

**O. Zografos et al., “Design enablement of CFET devices for sub-2nm CMOS nodes,” DATE, 2022.** DOI: 10.23919/DATE54114.2022.9774720。

这是从器件走到可设计标准单元的重要桥梁：不仅讨论堆叠结构，还讨论低轨高单元及其物理实现条件。对初次进入 CFET 的研究者，它比只读工艺制造论文更容易定位 EDA 问题。

建议重点核对单元可用连接层、器件控制方式、引脚和设计规则。所谓“支持 CFET”，如果不说明这些设定，无法判断两个算法在解决同一问题。[官方会议全文](https://past.date-conference.com/proceedings-archive/2022/pdf/0965.pdf)

### 6.3 用 SMT 联合处理 CFET 放置和布线

**C.-K. Cheng et al., “Complementary-FET (CFET) Standard Cell Synthesis Framework for Design and System Technology Co-Optimization Using SMT,” TVLSI, 2021.** DOI: 10.1109/TVLSI.2021.3065639。

该系列把器件放置、连接可行性和引脚相关约束放入求解框架。它的重要性在于：在低轨高 CFET 中，先把面积压到最小再交给路由器，可能已经把合法连通的机会破坏掉。因此需要尽早把布线条件放入放置决策。

这不是训练神经网络的方法。研究涉及 3.5T CFET 与模块级验证，但应按其预测规则解释；报告的布线违规减少也不等同于全套商业 DRC 清零。ICCAD 2020 的前期论文是同一研究线索。[TVLSI 论文](https://ieeexplore.ieee.org/document/9390403)；[ICCAD 2020 论文](https://ieeexplore.ieee.org/document/9256570)

### 6.4 多行 CFET 单元

**C.-K. Cheng et al., “Multirow Complementary-FET (CFET) Standard Cell Synthesis Framework Using Satisfiability Modulo Theories (SMTs),” JXCDC, 2021.** DOI: 10.1109/JXCDC.2021.3092769。

研究允许更复杂的单元跨多行实现，并分析极低轨高下的连接选项。它表明“固定单行、固定高度”不是唯一标准单元组织方式，多行可以换取更好的器件排列和连接条件。

实验使用 ASAP7 代表性网表作为逻辑输入，但实现对象是 CFET；本报告按器件结构纳入，而不把它写成真实先进节点制造验证。评估多行方案时，除了单元面积，还需要看放置器能否利用这些形状、是否引入空洞。[论文 DOI](https://doi.org/10.1109/JXCDC.2021.3092769)

### 6.5 CFET-FP：折叠和放置的最优化搜索

**S. Kim and T. Kim, “CFET-FP: Complementary FET Standard Cell Synthesis with Optimal Transistor Folding and Placement for Design and Technology Co-optimization,” TCAD, 2026.** DOI: 10.1109/TCAD.2026.3654920。前期版本见 DAC 2024。

该工作把折叠、放置和可布线性放在同一搜索过程中，通过代价计算和剪枝避免穷举所有排列。其思想不是只优化“图上的逻辑连接”，而是把器件折叠产生的几何变化纳入搜索。

公开仓库展示多个轨高、单行和多行 CFET 库，但 **README 说明的是生成的单元成果，不能直接当作完整生成器源码已公开**。另外，其 3T/4T 多行配置允许双向 M1；如果你的实验要求严格的逐层横/竖/横，就不能直接拿这类结果作同约束比较。论文含 7 nm 参考环境，纳入依据是 CFET，而不是工艺小于 7 nm。[作者机构记录](https://khu.elsevierpure.com/en/publications/cfet-fp-complementary-fet-standard-cell-synthesis-with-optimal-tr/)；[公开单元库](https://github.com/SNU-suwan/CFET_fp)

### 6.6 同时保证单元内部可布线性的综合流程

**Z. Zheng et al., “Comprehensive Placement and Routing Framework with Guaranteed In-Cell Routability for Synthesizing Complementary-FET Cells,” DAC, 2025.** DOI: 10.1109/DAC63849.2025.11132738。

研究把复杂单元分解、放置阶段的部分布线检查、引脚确定和后续分层布线组合起来，使用 SMT、启发式和整数规划等方法处理不同子问题。其目标不仅是压低宽度，还要避免放置完成后才发现无法连接。

这里的 guaranteed 是相对于论文所建模的布线和设计约束而言，不能外推为任意工艺、任意单元、全套制造规则都保证通过。它是后续研究布局失败检测、增量求解和复杂 cell 扩展时必须比较的工作。[论文](https://ieeexplore.ieee.org/abstract/document/11132738)

### 6.7 利用背面金属合成 CFET 库

**T.-X. Lin and Y.-L. Li, “Synthesis of CFET Cell Library Leveraging Backside Metal Routing,” DAC, 2025.** DOI: 10.1109/DAC63849.2025.11133087。

研究结合 Euler 路径相关宽度界、动态规划和 SMT，将部分布线需求移到背面。它不仅增加一层金属，还要协调折叠、轨道分配以及正反面连接。

作者报告平均 CPP 数下降约 1%、M2 轨道需求下降约 45%，运行时间也有所降低。一个有用的观察是：**新互连资源的重要收益可能是减少上层路由占用，而非让 cell 宽度大幅缩小。** 研究若只报告面积，会漏掉最重要的价值。[作者机构记录](https://scholar.nycu.edu.tw/en/publications/synthesis-of-cfet-cell-library-leveraging-backside-metal-routing/)

### 6.8 把合法引脚访问作为库综合目标

**T.-W. Lee et al., “Scalable CFET Cell Library Synthesis with A DRC-Aware Lookup Table to Optimize Valid Pin Access,” ISPD, 2025.** DOI: 10.1145/3698364.3705348。

这篇关注合法 pin access，而不只是引脚在图上存在。通过规则相关的查询机制降低反复检查的成本，让单元生成阶段就考虑外部路由能否接入。

对标准单元研究而言，“单元内所有 net 都连上”与“单元放入拥挤模块后仍容易连接”是两层问题。后续如果做可布线性预测或变体筛选，应以它为相关工作，而不是只与面积最小化工具比较。[官方报告材料](https://ispd.cc/ispd2026/slides/2025/protected/11_2_slides.pdf)

### 6.9 2026 年 CFET 背面互连与引脚访问

**H. Park and T. Kim, “Synthesis of CFET Standard Cells Utilizing Backside Interconnects Towards Improving Pin Accessibility,” ASPDAC, 2026.** DOI: 10.1109/ASP-DAC66049.2026.11420461。

该工作继续沿着自动 CFET 单元生成与背面连接结合的路线，目标明确包含引脚可访问性。它表明到 2026 年，“背面互连帮助 CFET 引脚访问”已是直接研究主题，而不是空白方向。

本次核实到官方日程及报告材料，不在缺少完整实验复核时抄录细碎收益百分比。适合与前述 DAC 2025、ISPD 2025 两篇放在一起比较约束、访问点定义和模块级指标。[官方报告](https://www.aspdac.com/aspdac2026/archive/pdf/6E-4.pdf)；[会议日程](https://www.aspdac.com/aspdac2026/pdf/ASP-DAC_2026_Full_Program.pdf)

### 6.10 双行 CFET 与半高单元：解决整行组织造成的浪费

**H. Kükner et al., “Double-Row CFET: Design Technology Co-Optimization for Area Efficient A7 Technology Node,” IEDM, 2024.** DOI: 10.1109/IEDM50854.2024.10873524。

双行 CFET 通过跨行组织器件与共享连接探索更紧凑的单元架构。这里的 A7 是先进技术路线图命名，不能理解为已量产的几何栅长。[论文](https://ieeexplore.ieee.org/document/10873524/)

后续 **“Half-Height Double-Row CFET Standard Cells for Area Optimized Placement in A7 CMOS Node,” ICCAD, 2025**，DOI: 10.1109/ICCAD66269.2025.11240689，进一步针对较小逻辑单元提出半高结构，减少整行或双行模板中的浪费，并把这些单元加入库后做模块级放置评估。

论文在其 ARM Cortex-M0、90% 利用率条件下，加入 6 类半高单元后报告约 10% 模块面积下降；更丰富库的约 14% 是进一步预测，不能当作同一实验已实现的结果。**它的重要性在于库的形状集合本身就是优化变量。** [ICCAD 论文](https://ieeexplore.ieee.org/abstract/document/11240689)

### 6.11 2026 年的共同栅与独立栅兼容问题

imec 的 2026 年官方研究更新介绍了 double-flip sequential CFET：同一架构需要兼容共同栅和独立栅连接，尤其不能只服务于简单逻辑而让多路选择器、触发器承担大量无效空间。其库级分析还涉及更密的背面 M0。

这里最值得注意的是 **复杂 cell 与 INV/NAND 的最佳堆叠方式可能不同**。官方材料给出的 98% 是其布局效率指标，不是良率、布通成功率或速度改善。本次以研究机构对 VLSI 2026 工作的说明作为最新补充，不把它当作已复现的完整公开平台，也不展开其中 SRAM 内容。[imec 官方研究说明](https://www.imec-int.com/en/articles/advancing-cfet-based-device-roadmap-novel-integration-modules-and-standard-cell-2)

### 6.12 另一个器件方向：多层垂直 GAA 单元综合

**D. Lee et al., “Many-Tier Vertical Gate-All-Around Nanowire FET Standard Cell Synthesis for Advanced Technology Nodes,” JXCDC, 2021.** DOI: 10.1109/JXCDC.2021.3089095。

研究多层垂直 GAA 器件对应的单元综合，涉及层数、引脚和垂直连接。它值得作为“新器件造成新的几何和互连优化问题”的例子。

不要把所有 vertical FET 都统称 CFET：沟道方向、器件层数、互补器件是否堆叠，是不同结构维度。复用 CFET 算法前要先核对这一层定义。[论文](https://ieeexplore.ieee.org/document/9454552)

## 7. 双面器件和背面信号：从 cell 到 block

### 7.1 Flip-FET 的双面物理实现

**H. Lu et al., “A Tale of Two Sides of Wafer: Physical Implementation and Block-Level PPA on Flip FET with Dual-sided Signals,” DATE, 2025.**

研究双面信号资源如何进入物理实现流程，包括引脚配置和正反面布线分配。其价值不是单独画出更小的 cell，而是将新 cell 组织接到模块实现，检查布线和 PPA 后果。

Flip-FET、CFET 背面信号、double-flip sequential CFET 的具体工艺结构不应混称；比较论文时需要逐一核对图示和互连假设。[作者公开稿](https://arxiv.org/abs/2501.15275)

### 7.2 FFET 单元变体与模块实现共同优化

**J. Ahn and T. Kim, “Design and Technology Co-optimization Utilizing Flip-FET (FFET) Standard Cells,” DAC, 2025.** DOI: 10.1109/DAC63849.2025.11132817。

该工作使用不同引脚配置的单元变体，结合实际模块需求进行替换和布线资源调整。它直接说明“生成多个等功能 cell，再根据模块布线选择”不是新概念。

如果做后续研究，需要回答更具体的问题：变体多到无法全部表征时怎么办？如何保留少量仍覆盖主要设计需求的变体？怎样避免局部引脚优化导致时序或功耗退化？[作者机构记录](https://snu.elsevierpure.com/en/publications/design-and-technology-co-optimization-utilizing-flip-fet-ffet-sta/)

### 7.3 双面引脚的 FFET 单元结构优化

**R. Guo et al., “Design Optimization of Flip FET Standard Cells with Dual-sided Pins for Ultimate Scaling,” TED, 2025.**

研究扩散合并、栅相关合并、双面连接及多行等结构选择，比较它们对面积和寄生的影响。复杂逻辑的最佳结构未必适用于简单逻辑。

这类研究适合帮助定义一个合理的候选空间，但不能仅凭某个 AOI22 的改善就推断整个库同幅度受益。公开稿注明 TED 接收；完整库及生成工具的可用性仍需单独核实。[作者公开稿](https://arxiv.org/abs/2504.10122)

### 7.4 2026 年双面堆叠器件的自动单元综合

**K. Guo et al., “Standard Cell Layout Synthesis for Dual-Sided 3D-Stacked Transistors,” ASPDAC, 2026.**

该工作将考虑合并的放置、双面布线和引脚分配结合起来，采用 SMT/SAT 等求解技术。它考虑的不只是电气连通，还包括几何合并后出现的合法连接方式。

双面引脚带来变体数量快速增长：多个输入各自选择前侧或后侧，会产生组合空间。因此真正的问题逐渐从“能否生成一个 cell”变成“怎样以合理成本生成并保留有用变体”。作者还报告了库用于模块实现的效果；本次未核实到完整公开生成器。[作者公开全文](https://kairongguo.com/files/ASPDAC26_FFET.pdf)

### 7.5 背面不仅供电，也承载信号和时钟

**A. A. Kedilaya et al., “Beyond Backside Power: Backside Signal Routing as Technology Booster for Standard-Cell Scaling,” JXCDC, 2025.** DOI: 10.1109/JXCDC.2025.3617784。

这项工作将背面接触用于标准单元的信号和时钟路由，并探索更低单元高度及不同正反面引脚变体。它连接了单元架构、模块布线和供电完整性，适合与纯 cell 生成论文对读。

本报告引用 IEEE 学会和论文入口确认研究主题，不用二手摘要中的最大百分比替代完整实验条件。尤其不能把减少某层信号轨道数直接解释成同等幅度的全部单元面积缩小。[IEEE 官方介绍](https://sscs.ieee.org/tag/high-performance-compute/)；[论文 DOI](https://doi.org/10.1109/JXCDC.2025.3617784)

### 7.6 单元内部与单元之间的背面信号布线

**J. Lee et al., “A Novel Backside Signal Inter/Intra-Cell Routing Method Beyond Backside Power for Angstrom Nodes,” VLSI Symposium, 2025.**

该工作分别研究有背面引脚时的单元间连接，以及保持对外接口时的单元内部背面连接。区别很重要：前者要求后端工具支持新的引脚和路由资源，后者可以先改善单元内部，但也会改变寄生和供电资源分配。

全文采用预测性 forksheet 器件、80 nm 高度的 4T 单元，设计了 42 个标准单元，因此明确符合新器件这一筛选条件。它同时讨论性能收益和 IR drop 影响，说明将信号移到背面不是没有代价。**论文允许双向金属布线**，不能不加说明地与严格单向层结果比较。这里仍是预测性埃级技术研究，不是已验证的商业节点。[官方会议摘要，T7-2](https://www.vlsisymposium.org/wp-content/uploads/VLSI2025_Advanceprogram0611.pdf)；[作者全文](https://gtcad.gatech.edu/www/papers/Lee-VLSI25.pdf)

## 8. 表征、优化和测试：不只是版图生成

### 8.1 sub-5 nm 布局相关效应下的延时排序

**Y.-R. Hsu et al., “Machine-Learning-Based Ranking of Cell Layout Delay Considering Layout-Dependent Effects,” TVLSI, 2025.** DOI: 10.1109/TVLSI.2025.3543851。

研究工业 sub-5 nm FinFET 环境中，同一个 cell 的不同布局怎样排序。目标是减少昂贵评估次数，尽快找到优秀布局，而不是只优化一个总体回归误差。

这与你的性能预测研究很接近，但边界不同：它主要是候选布局选择，不能自动推成未见功能、全套 PPA 或完整 Liberty 表征。相应地，你若做新工作，除了 MAPE，应报告选出的真实最优性差距和省下的仿真成本。[作者机构记录](https://scholar.nycu.edu.tw/en/publications/machine-learning-based-ranking-of-cell-layout-delay-considering-l/)

### 8.2 5 nm 低温 CMOS 库开发

**V. M. van Santen et al., “Design Automation for Cryogenic CMOS Circuits,” DAC, 2023.** DOI: 10.1109/DAC56929.2023.10247824。

该工作基于商业 5 nm 工艺器件的低温测量和模型校准，建立适用于低温条件的设计与库流程，再进入电路综合和功耗优化。

它说明“先进节点库表征”可以针对新的应用条件展开，而不一定依赖新的布局生成算法。需要区分：器件参数有测量依据，不等于文中全部数字电路 PPA 都来自流片。低温只是明确案例，类似思想也可延伸到电压、温度和可靠性条件，但后者属于研究建议。[作者公开全文](https://www.cda.cit.tum.de/files/eda/2023_dac_design_automation_for_cryogenic_cmos.pdf)

### 8.3 3 nm 库的 cell-aware test

**Z. Gao et al., “Application of Cell-Aware Test on an Advanced 3nm CMOS Technology Library,” ITC, 2019.** DOI: 10.1109/ITC44170.2019.9000164。

研究将单元内部的开路、短路等缺陷模型用于测试生成和覆盖分析，采用实验性先进工艺库。论文中的 imec 节点名称不能直接按字面数字解释。

它属于库的测试与缺陷表征，而不是常规时序功耗表征。若希望把研究范围拓宽到标准单元可靠性、可测性和测试成本，这是一条独立方向；本报告不把它混入 delay prediction 的同类基线。[论文](https://ieeexplore.ieee.org/abstract/document/9000164)

### 8.4 3 nm GAA 的代理模型与贝叶斯优化

**H. Jeong et al., “ML-Driven Optimization of Standard Cell Performance and Timing in Advanced Nodes,” JSTS, 2026.** DOI: 10.5573/JSTS.2026.26.2.130。

采用 3 nm nanosheet、5T/BPR 设定，通过采样、寄生提取、代理模型和多目标优化探索单元性能，同时约束 rise/fall 平衡。实验集中在少数基本逻辑单元，并有小型电路验证。

它不是本报告中最高优先级的 venue，但与选题非常直接：**“新工艺 + 一个 MLP 预测器 + BO 优化尺寸”已经有人做。** 要形成更强工作，需要进一步解决跨功能泛化、合法布局、表征成本或模块级收益，而不能只换器件名称。[期刊全文](https://www.jsts.org/jsts/XmlViewer/f451686)

### 8.5 2026 年新线索，单独标为预印本

**TOPCELL，2026 年预印本**，研究用 LLM 与强化训练进行标准单元拓扑优化，摘要包含工业 2 nm 环境和向 7 nm 的迁移。应关注其功能验证、候选合法性和优化成本，但本次未确认正式会议或期刊版本；其中针对 7 nm 的加速数字不能写成 2 nm 的结果。[作者预印本](https://arxiv.org/abs/2604.14237)

**CFET 系统与工艺联合评估，2026 年预印本**，比较预测性 CFET 与 nanosheet 方案，将寄生、库、系统实现以及热和老化因素联系起来。它提示库评估正在向更多物理条件扩展，但不能据此宣布某个未来节点已具备量产能力。[作者预印本](https://arxiv.org/abs/2609.15326)

## 9. 现在有哪些真正可用的研究平台

### 9.1 GT3：目前值得优先检查的公开 3 nm GAA 平台

**D. E. Shim et al., “GT3: An Open-Source 3-nm GAAFET PDK and Platform for End-to-End Evaluation of Emerging Technologies,” TED, 2025.** DOI: 10.1109/TED.2025.3540760。

本次核对了公开仓库和 README：提供 65 个标准单元，目录包含 GDS、库视图、LEF、Liberty、HSPICE 模型、提取相关资料和 IC Validator runset。README 给出了 TT、0.7 V、25°C 的 Liberty 文件入口。

**适合用途：** 先进 GAA 单元的版图和寄生研究、库到模块的基线搭建。**限制：** 预测 PDK，不是商业可投片规则；提供 ICV 规则不代表服务器上的 Calibre 可以直接使用；也不能从一个列出的 `.lib` 推断完整多 PVT 表征齐全。本次未安装或运行该 PDK。[论文](https://ieeexplore.ieee.org/document/10906664)；[公开仓库](https://github.com/azadnaeemi/GT3)

### 9.1.1 补充：GT2N，公开 2 nm nanosheet 平台

**D. Jang et al., “GT2N: An Open-Source 2nm Nanosheet PDK Enabling Multi-Width/VT Benchmarking,” ISCAS, 2026.** 它提供带背面供电的 GAA 研究平台，可纳入公开资源候选；部分工艺角仍在开发。本次仅核查公开材料，未安装验证，不能将其等同于某家晶圆厂的量产工艺。[官方仓库](https://github.com/azadnaeemi/GT2N)；[论文](https://ieeexplore.ieee.org/document/11562058)

### 9.2 PKP3 与 PKP2：GAA、背面互连的另一个入口

**B. Peng et al., “PKP: Open-source Compact Models and Predictive PDK for GAA Technology,” Science China Information Sciences, 2026.** DOI: 10.1007/s11432-025-4805-5。出版方标为 **News & Views**，不应当作长篇完整研究论文评价。

PKP3 是预测性 3 nm nanosheet GAA 平台，公开库中有模型、标准单元、技术配置和提取相关文件。**Calibre 的 DRC/LVS/PEX runset 不直接在 GitHub，需要按官网要求申请。** 因此“仓库能下载”不等于“本机即可完成所有物理验证”。[出版方页面](https://link.springer.com/article/10.1007/s11432-025-4805-5)；[PKP3 仓库](https://github.com/PHIMO-Group/PKP3)

官网另宣布了 2026 年的 PKP3E 更新，以及 **2026 年 8 月 PKP2**：后者等效缩放到 2 nm GAA，支持直接背面接触和背面供电，并列出 4.5T、32 类单元、56 个驱动变体。**本次仅核实官网发布信息，未下载核验 PKP2 全包，也未确认其与公开 PKP3 仓库完全同步。** 它不是任何晶圆厂可制造工艺。[官方版本与申请说明](https://pkp.pkueda.org.cn/)

同日补查：官方文件中心已列出 2026-10-16 的 PKP2 V1.0 及对应规则包申请条目；发布条目不等于已取得并运行该包。[公开发布页](https://pkp.pkueda.org.cn/collection/PKP)；[规则包申请页](https://pkp.pkueda.org.cn/collection/PKP_runsets)

### 9.3 PROBE3.0：预测 3 nm FinFET 的完整设计探索

**S. Choi et al., “PROBE3.0: A Systematic Framework for Design-Technology Pathfinding With Improved Design Enablement,” TCAD, 2024.** DOI: 10.1109/TCAD.2023.3334591。

它不是简单把 ASAP7 标签改成 3 nm：研究使用预测性 3 nm FinFET 设定，构建相应的单元和互连条件。流程覆盖库生成、版图视图、寄生与表征、模块实现和供电分析，论文中每套库含 41 个单元。

相比只做 cell 面积，PROBE3.0 更值得学习的是**可配置工艺和单元参数如何经过完整实现流程形成可比较结论**。软件和资料开放，并不表示所有依赖的签核、表征和后端工具免费；具体可运行程度必须检查安装环境。[作者全文](https://vlsicad.ucsd.edu/Publications/Journals/j143.pdf)；[仓库](https://github.com/ABKGroup/PROBE3.0)

### 9.4 一个很有启发性的前期结果：小 cell 不一定组成小 block

**C. Chidambaram et al., “A Novel Framework for DTCO: Fast and Automatic Routability Assessment with Machine Learning for Sub-3nm Technology Options,” VLSI Technology, 2021.**

该工作跨多组 cell 高度、金属节距、CPP 和供电选项评估可布线性。其中一组比较中，**cell 面积更大的库，反而获得更小的 block 面积**，原因是拥塞和布线可行性改变了可实现的利用率。

虽然其快速评估器用了 ML，但这里对库研究最重要的启示与 ML 无关：**局部目标和最终设计目标可能方向相反。** 这也是新库工作为什么需要至少一个可信模块级闭环。[作者全文](https://vlsicad.ucsd.edu/Publications/Conferences/384/c384.pdf)

### 9.5 公开程度对照

| 平台 | 节点/结构 | 本次确认公开或声明提供的内容 | 主要限制 |
| --- | --- | --- | --- |
| GT3 | 预测 3 nm GAA | 65-cell 库、模型、GDS/LEF/Liberty、ICV 规则、提取资料 | 商业工具依赖；不是 Calibre deck；未本地验证完整流程 |
| GT2N | 2 nm GAA 研究平台、背面供电 | 标准单元、模型、库视图、ICV 规则、提取资料 | 工艺角仍有开发中部分；未本地验证完整流程 |
| PKP3 | 预测 3 nm GAA | 模型、标准单元、技术文件、提取资料 | Calibre runset 申请制；模型和仿真器兼容性需测试 |
| PKP2 | 预测 2 nm GAA、背面接触 | 官网宣布的 4.5T 库与背面互连配套 | 仅核实发布信息，尚未下载核验完整资源 |
| PROBE3.0 | 预测 3 nm FinFET | 设计探索框架及生成资源 | 不是 foundry 3 nm；完整流程有商业工具条件 |
| CFET-FP 公开仓库 | CFET，多个轨高 | 生成的单元库和技术相关文件 | 不等于生成器源码全公开；某些配置允许双向 M1 |
| SMTCell 系列 | 不同版本支持不同结构与轨高 | 公开研究代码 | 不同论文版本不能互相替代；必须固定版本核对支持范围 |
| USC-3N-2D | 预测 3 nm GAA | 面向研究的库与实现资源 | 作者声明校准及签核简化；部分资产衍生自已有公开平台 |

USC-3N-2D 的资源和限制可见[作者仓库](https://github.com/SCCAD-LAB/USC-3N-2D)。它不应被当作与 GT3 完全独立的工业工艺证据。

SMTCellUCSD-2.0 当前 README 中的实现路线和支持矩阵，与早期论文或历史版本可能不同。不能仅凭仓库名认定它同时支持所有 CFET、背面互连和任意轨高。[当前仓库](https://github.com/ckchengucsd/SMTCellUCSD-2.0)

**最实际的顺序：先核对模型能否仿真，再核对 DRC/LVS/PEX，再确认一个 INV 和一个复杂组合单元，最后扩成库。** 不要先生成几万份版图，之后才发现寄生模型、规则 deck 或器件参数不兼容。

## 10. 哪些题目已经做过，不能只换一个名字

| 想法 | 相关先例 | 新工作至少还需回答什么 |
| --- | --- | --- |
| 用 SMT 自动生成 CFET cell | TVLSI/JXCDC 2021 等 | 更强约束、更大单元、更好可扩展性或更完整后端收益 |
| 同时优化折叠和放置 | CFET-FP | 为什么新的搜索或约束建模优于已有精确/剪枝方案 |
| 背面布线减少正面拥塞 | DAC 2025、ASPDAC 2026 | 引脚/供电/时序冲突怎样联合处理 |
| 不同 nanosheet 宽度组成库 | TVLSI 2022 | 不同设计需求下怎样控制变体数量与表征成本 |
| 生成多种前后面引脚变体再选择 | FFET DAC 2025、ASPDAC 2026 | 更低成本的变体选择、全局反馈、可靠性能约束 |
| 用 ML 排序 cell 布局 | TVLSI 2025 LDE 排序 | 更广单元、跨布局/功能泛化、带约束的优化效果 |
| 用代理模型和 BO 调尺寸 | JSTS 2026 等 | 合法几何、复杂逻辑、全 arcs/PVT 或完整库收益 |
| 用低轨高减少面积 | 多个 CFET/GAA 工作 | 是否仍有足够 pin access，最终是否减少 block 面积 |

这些先例并不意味着问题做完了，而是意味着**创新需要落在新的约束、更准确的目标、更低的实际成本或更强的验证上**。

## 11. 对研究机会的判断

以下是基于上述文献提出的研究建议，不是已经证明有效的方法。

### 11.1 方向一：同一网表、多种合法布局的寄生与性能优化

**研究问题：** 对相同逻辑和器件尺寸，先进 GAA/CFET 的连接选择如何影响各个 timing arc、功耗和输入电容？能否用较少昂贵仿真，选出真正有用的布局？

这个问题比“换成 3 nm 再做一次 PPA 回归”更强。它明确隔离了逻辑、尺寸和几何三类变化，能够检验模型是否学习到局部互连和寄生，而不是记住功能类别。

**最低可行实验：** 从 GT3、GT2N 或获得规则后的 PKP3 中选一个通过环境核对的平台，包含 INV、NAND/NOR、AOI/OAI、多路选择结构及部分时序单元；为同一网表生成多种合法布局。保留各候选的 DRC/LVS、提取 RC、多个负载和输入转换条件，而不是仅留下五个压缩指标。

**基线与指标：** 几何/RC 启发式、传统回归或排序、现有图模型、随机筛选，以及相同仿真预算下的搜索。报告选中候选距真实最优的差距、进入前若干名的比例、真实仿真调用量、失败率和总耗时。

**风险：** 新布局实际很少、只有简单门，或者最优布局始终由面积决定，就难以证明复杂模型的价值。若布局生成器不可用，先做少量可解释变体，不要伪造“自动完整库”。

### 11.2 方向二：限制库规模和表征成本的单元变体选择

**研究问题：** 宽度、驱动、Vt、轨高、前后表面引脚、单行多行会形成大量组合。能否保留一小部分单元，使多个不同电路仍获得好的时序、功耗和布线结果？

现有混合宽度和 FFET 变体选择论文已经提供先例。更值得做的是**在明确的库规模、表征预算和时序覆盖约束下，选择有用集合**，而非只对一个设计无限扩库。

**非 AI 方案也完全成立：** 基于候选支配关系的剪枝、整数规划、子集选择、列生成、结合真实后端反馈的迭代。学习器可以辅助估计代价，但不是必要条件。

**必须验证：** 未参与选库的电路、最终 block PPA、布线违规、库文件规模、表征总成本。还要对比“完整候选库”和“相同数量的简单筛选库”，否则无法知道收益来自选择算法还是多提供了单元。

### 11.3 方向三：CFET 复杂单元的失败诊断与增量求解

**研究问题：** 复杂 MUX/DFF 或低轨高单元失败时，究竟是结构不可行、轨道不足、引脚限制，还是求解器没有及时找到解？能否针对失败原因只修改相关局部，而不是每次从头搜索？

这个方向不要求先获得大量 PPA 标签，适合已有自动版图经验的团队。可用约束冲突、不可满足核心、分解和增量 SAT/SMT/CP-SAT 等方法。

**关键对照：** 不同求解预算下的成功率、首次可行解时间、最优性界、验证通过率、内存，以及长尾复杂单元。不能把 timeout 计作“已证明不可布通”。

**主要障碍：** 要有可信规则和验证器；如果只在大幅简化的规则下解决了不真实问题，求解成功率再高也说服力不足。

### 11.4 方向四：跨器件/跨工艺的少量表征适配

**研究问题：** 从 FinFET 学到的哪些信息可以迁移到 GAA？从平面排列到 CFET，哪些几何和寄生信息必须重新获取？

对你已有工作而言，不能直接把 `n_fin` 改名为 `n_sheet`。可能需要显式保留 sheet 宽度、层数、堆叠位置、共同栅/独立栅、接触方式、层间 via 和正反面路由等信息。具体保留哪些应通过实验判断。

**合理实验：** 先保持相同逻辑集合，分别控制尺寸变化、布局变化与器件变化，比较从头训练、少样本适配和物理/几何基线。划分应避免同一 cell 的近似复制版本跨训练测试泄漏。

**边界：** 如果目标只是现有五指标在一个预测 PDK 上精度改善，研究贡献可能有限；若能说明哪些结构因素决定迁移失败，并减少真实表征成本，价值更清楚。

### 11.5 方向五：表征条件下的精度分配，而非只预测一个 worst delay

完整单元库要覆盖输入输出 arc、上升下降、输入转换、负载、PVT，以及时序单元的额外约束。cell-level 五指标可以用于早期筛选，但不是完整库的替代品。

可以研究：哪些条件必须精确仿真，哪些可插值或近似；怎样在固定预算下保证最关键 arc 不被漏掉；新背面接触或堆叠结构是否带来新的误差集中区域。

应与已有自适应采样、表征插值和代理模型比较，并在需要最终精确值时回到可信仿真。误差小不能以掩盖未测条件或未通过验证的样本为代价。

## 12. 对你现有研究和工具的具体影响

### 12.1 AutoCellGen 不能直接视作任意先进工艺生成器

既有 ASAP7 流程的网表处理、任务调度、失败记录和部分搜索结构可以复用，但 GAA/CFET 还涉及器件几何、层定义、接触、折叠和不同布线规则。CFET 尤其不能只修改 cell height 和 layermap。

先做一个小型适配验证，比直接宣布“支持 2 nm”更可靠：网表解析正确、器件参数与模型一致、版图验证通过、寄生提取可解释、表征结果正常，五项都完成后再扩展。

### 12.2 你要求的横/竖/横布线需要在论文中成为明确约束

不能把允许双向 M1、额外背面金属或更多局部连接的工具，与严格单向层工具直接比较成功率。应固定每层方向、可用轨道、局部互连、pin access 和电源方式，再比较算法。

不同 track 的困难也不是简单“数字越小越难”：如果同时增加背面信号、允许跨行或改变引脚层，实际可用资源可能增加。应比较完整架构，而非只比较 T 数。

### 12.3 保留验证信息，而不是只保留成功的图片和 PPA

新库研究应记录 DRC、LVS、PEX、仿真完成情况和具体规则版本。DRC 违规样本可以用于方法开发或失败分析，但应与合法库分开；不能在最终库结论中默默视作可制造结果。

如果要发布数据，至少区分未验证、已检查但有违规、验证通过、提取失败、仿真失败。它们分别回答不同问题，不能合并为一个“success”。

## 13. 推荐的实施顺序与停止条件

| 阶段 | 要完成什么 | 暂停或换路线的条件 |
| --- | --- | --- |
| 资源核对 | 确认模型、规则 deck、提取工具、许可和库视图 | 只有图片或 LEF，没有可验证器件与寄生模型 |
| 小规模闭环 | INV、一个复杂组合门、一个时序单元的验证与表征 | 基本单元的电流、逻辑、RC 或单位不一致 |
| 候选生成 | 为相同网表形成多个合法且有意义的候选 | 候选差异只来自无效几何或不满足同一规则 |
| 方法评估 | 固定预算比较搜索、筛选或预测方案 | 优势仅来自多跑仿真、额外布线层或放松规则 |
| 库级闭环 | 生成必要视图，放入综合与后端流程 | cell-level 改善在模块端普遍消失且原因不清楚 |
| 发布 | 公开可分发代码、设置、划分和验证状态 | 依赖不能分发的资产，却声称完全可复现 |

**如果只选一条近期路线：优先做公开 GAA 平台上的“合法布局候选 + 寄生/时序评估 + 候选或库变体选择”。** 它既能衔接你已有的版图与 PPA 工作，也能避免从工艺器件开发重新开始。

## 14. 本报告没有声称的事情

- 没有声称所有列出的工作都使用真实可投片的商业先进节点。
- 没有把 7 nm ASAP7 FinFET 实验算作小于 7 nm 的实验。
- 没有把 CFET 的理论面积优势等同于同等比例的模块收益。
- 没有把所有论文都称为顶会顶刊，也没有把预印本混作已录用论文。
- 没有声称公开库、公开生成器、公开 PDK 和开源端到端工具链是同一件事。
- 没有在本次调研中安装、运行或验证 GT3、GT2N、PKP、PROBE 或新的 CFET 工具。
- 没有把拟议研究方向写成“首次提出”或已完成的实验证据。

总体判断：**先进标准单元研究的中心，正在从“把逻辑装进更小的盒子”，转向“在新的器件、连接和供电约束下，设计真正能被后端有效使用的库”。** 这为布局算法、库表征、变体选择和性能预测都留下了空间；最有说服力的工作往往能解释并验证从局部单元到最终设计的收益是如何传递的。
