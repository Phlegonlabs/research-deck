# 宇树科技：$16K 人形机器人出货量超过特斯拉、Figure、Agility 总和

## 摘要

宇树科技（Unitree Robotics），2016年由前大疆工程师王兴兴带着4个人和27.5万美元创立，现在是全球人形机器人出货量第一。2025年出货5,500+台人形机器人——超过特斯拉、Figure AI和Agility Robotics的总和。2026年目标20,000台。计划以70亿美元估值在上海科创板IPO。他们的秘诀：执行器垂直整合、激进定价（G1售价$16K vs 竞争对手$100K+）、开源强化学习工具、以及中国制造生态系统。

---

## 1. 公司背景

### 创立与历史

| 信息 | 详情 |
|------|------|
| **成立时间** | 2016年8月，中国杭州 |
| **创始人** | 王兴兴，1990年生，浙江宁波人 |
| **初始团队** | 4人 |
| **种子资金** | 200万元人民币（约$27.5万）天使轮 |
| **总部** | 杭州市滨江区东流路88号丰达创意园1幢3楼 |
| **员工数** | 约40人（公开数据），考虑到生产规模实际可能更多 |
| **公司全称** | 杭州宇树科技有限公司 |

### 王兴兴的成长之路

王兴兴小时候做航模，中学就组装微型涡喷发动机。大学一年级，他用200元（约30美元）从废旧零件中组装了一个双足行走机器人。硕士期间，独立开发了 **XDog**——一款使用低成本外转子无刷电机和全自由度驱动系统的四足机器人。

2016年硕士毕业后，王兴兴短暂加入了**大疆**（全球领先的无人机制造商）。但几个月内，他的XDog项目在科技媒体上火了，于是他离职，用天使融资创立了宇树科技。他的偶像是Marc Raibert（波士顿动力创始人），但他的使命恰恰相反：让足式机器人**平价且易得**。

王兴兴曾受邀参加习近平主持的座谈会，标志着宇树对中国科技议程的战略重要性。

### 融资历史

| 轮次 | 时间 | 金额 | 主要投资者 | 投后估值 |
|------|------|------|-----------|---------|
| 天使轮 | 2016 | 200万元（约$27.5万） | — | — |
| A轮 | — | — | — | — |
| B轮 | — | — | — | — |
| C轮 | 2025年6月 | 约7亿元（约$9900万） | 腾讯、阿里巴巴、蚂蚁集团、中国移动、吉利、金秋资本、红杉中国 | 12亿美元（独角兽） |
| **总融资** | — | **约2.636亿美元** | 6轮融资，31个投资者 | — |

### 营收与盈利

- **年营收**（截至2025年中）：约10亿元人民币（约1.4亿美元）
- **盈利状况**：已实现盈利（IPO申报确认）
- **海外销售**：四足机器人收入的50%来自海外

### IPO 计划

- **目标**：上海科创板（STAR）——中国的纳斯达克
- **时间线**：2026年Q2提交IPO申请，中信证券为保荐机构
- **报道估值目标**：**70亿美元**
- 2025年11月完成上市辅导
- 辅导期：2025年7月至12月

---

## 2. 完整产品线

### 人形机器人

| 型号 | 身高 | 重量 | 自由度 | 价格范围 | 状态 | 目标用途 |
|------|------|------|--------|---------|------|---------|
| **R1 AIR** | 123cm | 25kg | 20 | $4,900 | 预售，2026年Q2交付 | 消费、教育 |
| **R1** | 123cm | 29kg | 26 | $5,900 | 预售，2026年Q2交付 | 消费、教育、开发 |
| **G1 基础版** | 132cm | 35kg | 23 | $13,500–$21,600 | **在售** | 研究、教育 |
| **G1 EDU** | 132cm | 47kg | 23–43 | 最高$73,900 | **在售** | 高级研究、开发 |
| **H1** | 180cm | 47kg | — | $90,000–$150,000 | **在售** | 研究、工业 |
| **H1-2** | 180cm | — | — | $99,900–$128,900 | **在售** | 研究、工业 |
| **H2** | 180cm | 70kg | 31 | $29,900–$68,900（EDU） | 预售，2026年Q2交付 | 工业、商业 |

### 四足机器人

| 型号 | 价格 | 目标用途 |
|------|------|---------|
| **Go2** | 起价$1,600 | 消费、教育、爱好 |
| **B2** | $100,000 | 工业巡检、物流 |
| **B2-W** | — | 重载（轮式变体） |

### 组件与传感器

| 产品 | 描述 |
|------|------|
| **4D激光雷达 L1** | 21,600点/秒，IMU，开源SLAM |
| **4D激光雷达 L2** | 64,000点/秒，30米距离，360°×96° FOV |
| **M107关节电机** | 189 N·m/kg扭矩密度，107×74mm |
| **灵巧手** | 为G1/H2新开发，可操纵道具 |

### 核心产品详情

#### G1——走量之王

G1是宇树的突破性产品。基础价$13,500，仅为同类人形机器人的1/6。2025年上半年出货5,000+台。

- **传感器**：Intel RealSense D435深度相机、Livox MID360 3D激光雷达、共5个深度相机
- **计算**：EDU版最高NVIDIA Jetson Orin（100 TOPS）
- **12种配置**：从基础研究平台到完整自主开发套件
- **软件**：ROS2兼容、开源RL训练框架、UnifoLM集成

#### H2——新一代全尺寸

H2代表宇树向类人交互和工业部署的推进。

- **仿生面部**：类人外观设计，提升社会接受度
- **视觉**：仿人双目宽视场相机系统
- **音频**：麦克风阵列 + 大功率扬声器，自然语音交互
- **执行器**：腿部360 N·m，臂部120 N·m，手臂额定负载约15kg
- **计算**：Intel Core i5基础版，EDU版可选NVIDIA Jetson AGX Thor（2,070 TOPS）
- **材料**：航空级铝合金、钛合金、高强度工程塑料
- **电池**：快拆式15Ah智能电池，约3小时续航
- **连接**：Wi-Fi 6、蓝牙5.2

#### R1——消费者之选

获评 **TIME 2025年最佳发明**。$4,900的价格，是第一款真正面向消费者的人形机器人。

- **AI**：集成大型多模态模型（LMM），支持语音和图像理解
- **电池**：约1小时续航，快换系统
- **计算**：8核高性能CPU，EDU版可选NVIDIA Jetson Orin（40–100 TOPS）
- **相机**：单目（AIR）/ 双目带深度（标准版）

---

## 3. 技术架构

### 3.1 UnifoLM——统一大模型框架

UnifoLM（Unitree Robot Unified Large Model，宇树机器人统一大模型）是宇树的AI框架，为跨实体机器人学习而设计。

#### 架构变体

| 模型 | 类型 | 用途 |
|------|------|------|
| **UnifoLM-WMA-0** | 世界模型-动作 | 通过物理仿真进行通用机器人学习 |
| **UnifoLM-VLA-0** | 视觉-语言-动作 | 基于视觉+语言输入的人形操作 |
| **UnifoLM-X1-0** | 工业具身AI | 工厂车间任务执行 |

#### WMA-0 工作原理

两个核心组件：

1. **世界模型**——理解机器人与环境之间的物理交互。充当**仿真引擎**，为机器人学习生成合成数据。相当于"机器人的水晶球"，预测未来状态。

2. **动作头**——与世界模型配对，通过预测未来交互和选择最优动作来优化决策。

该架构是**跨实体**的——同一模型可用于四足、人形和机械臂。这就是"统一"的含义。

**开源**：UnifoLM-WMA-0已在Hugging Face发布。这是策略性的——构建生态、吸引研究者、收集社区改进。

### 3.2 数据引擎——机器人造机器人

这是宇树最被低估的战略资产。概念如下：

1. **在宇树自己的产线上部署G1人形机器人**
2. **机器人装配电机组件和机器人零件**（工厂视频已展示）
3. **每次装配操作都生成运动数据**——真实世界的，不是仿真的
4. **数据反馈到UnifoLM-X1-0**以改善操作技能
5. **更好的AI → 更好的装配 → 更多数据 → 更好的AI**（飞轮效应）

该方法使用 **Embodied Avatar 遥操作平台**进行"人在回路"初始训练——从人类操作者那里获取运动学准确的动作，然后AI学习。

这与主要依赖仿真（sim-to-real迁移）的竞争对手策略根本不同。宇树将**真实世界数据作为生产的副产品**获取。

### 3.3 执行器设计——M107

M107关节电机是宇树的核心硬件差异化因素。王兴兴硕士论文中使用低成本外转子无刷电机的原始洞察，发展成了这个：

| 规格 | 数值 |
|------|------|
| **扭矩密度** | 189 N·m/kg（号称世界最高） |
| **最大扭矩** | 360 N·m（膝关节）、220 N·m（髋关节）、45 N·m（踝关节） |
| **尺寸** | 107 × 74 mm |
| **电机类型** | 低惯量、高速内转子PMSM |
| **特性** | 中空轴（走线）、双编码器、工业级交叉滚柱轴承 |

集成关节模组将：驱动器 + 电机 + 减速器 + 编码器 + 传感器——全部集成在一个模组中。这减少了装配复杂度，节省空间，关键是通过消除组件间布线和外壳来**降低成本**。

### 3.4 传感器栈

| 传感器 | 型号 | 用途 |
|--------|------|------|
| **3D激光雷达** | Livox MID360 | 空间建图、定位 |
| **深度相机** | Intel RealSense D435（×5） | 物体检测、操作 |
| **4D激光雷达** | 宇树 L1/L2（自研） | 高密度点云 |
| **IMU** | 内置6轴 | 平衡、姿态 |
| **超声波** | 3组，150°×170° | 避障 |

### 3.5 控制与软件栈

- **ROS2兼容**，覆盖所有平台
- **开源RL训练**：`unitree_rl_gym`（基于Isaac Gym）、`unitree_rl_lab`（Isaac Lab）、`unitree_rl_mjlab`（MuJoCo）
- **仿真到真实管线**：训练 → 运行 → Sim2Sim → Sim2Real
- **支持平台**：Go2、H1、H1-2、G1
- **OTA更新**：售后持续软件改进

### 3.6 如何做到$16K的价格

G1售价$16K。特斯拉Optimus目标$20–30K（尚未上市）。波士顿动力Atlas：$140–150K（不出售）。怎么做到的？

| 成本因素 | 宇树的做法 |
|---------|-----------|
| **执行器** | 自研M107。垂直整合消除供应商利润 |
| **尺寸** | G1只有132cm（4尺4）——比6英尺人形机器人用更少材料和更小电机 |
| **制造地** | 中国杭州——劳动力成本比美国低50–70% |
| **供应链** | 深度嵌入中国电子/电机/传感器生态系统 |
| **规模** | 已出货5,500+台。规模经济开始发力 |
| **计算** | 分层——基础版使用标准CPU，昂贵的Jetson只在EDU版 |
| **无奢华装饰** | 功能性设计，G1没有仿生面部（那是H2的） |
| **组件共享** | 执行器技术在Go2四足 → G1 → H1 → H2全产品线共享 |

核心洞察：宇树不试图造最强的机器人。他们造的是**每个价格段上最强的机器人**。R1 $4,900、G1 $13,500、H2 $29,900——每个都是其价位的最佳选择。

---

## 4. 2026春节联欢晚会——"赛博真功夫"

### 事件

**2026年2月17日**，宇树的机器人登上央视春晚——中国的超级碗，7亿+观众观看。节目名为"赛博真功夫"。

### 发生了什么

数十台 **G1** 和 **H2** 人形机器人与人类武术家一起表演了**全球首次全自主人形机器人集群武术表演**。B2-W机器狗也登场。在义乌分会场，一台H2穿着**齐天大圣铠甲骑在四足机器狗上**。

### 世界首创技术成就

| 成就 | 细节 |
|------|------|
| **连续自由式跳台跑酷** | 世界首创 |
| **抛射空翻** | 最大翻转高度超过**3米** |
| **连续单腿翻转** | 世界首创 |
| **两步蹬壁后空翻** | 世界首创 |
| **大风车回旋** | **7.5圈**——世界首创 |
| **集群奔跑速度** | 最高**4 m/s**（14.4 km/h） |
| **灵巧手道具操控** | 双截棍、长棍——新开发的手部 |

### 表演的技术架构

1. **高并发集群控制系统**——低延迟实时同步数十台机器人
2. **自研AI融合定位算法**——整合本体感受数据 + 3D激光雷达实现动态运动中的定位精度（无外部动捕）
3. **预训练运动控制模型**——每个动作精调到十分之一秒
4. **编队变换算法**——机器人在高速奔跑中完成编队变换和运动转换

### 为什么重要

这不是带外部位置跟踪的预编程舞蹈。机器人是**全自主**的——使用机载激光雷达和本体感受传感器进行定位。在4 m/s速度下进行多机器人协调同时执行杂技，是一个真正困难的技术问题，展示了：

- 大规模实时多智能体协调
- 极端动作中的动态平衡控制
- 运动拥挤环境中的机载SLAM
- 足够鲁棒以应对直播（没有第二次机会）

### 商业影响

春晚之后，宇树和其他机器人公司报告**订单积压**。这场表演本质上是一个零成本营销活动，触达7亿+观众，展示了真正的自主能力。

---

## 5. 生产与规模

### 出货数据

| 年份 | 人形机器人出货 | 生产量 | 全球市场份额 |
|------|--------------|--------|------------|
| **2025** | **5,500+** 出货 | 6,500+生产 | 约全球市场30% |
| **2026目标** | **10,000–20,000** | — | — |

### 全球背景

- **IDC**：2025年全球人形机器人市场同比增长**508%**
- **2025年全球人形机器人总出货**：约13,000–18,000台
- 宇树一家约占**全球市场的1/3**
- 宇树出货量超过**特斯拉、Figure AI和Agility Robotics的总和**（Omdia数据）

### 制造方式

宇树杭州工厂采用混合方式：

1. **传统流水线**进行批量生产
2. **G1人形机器人在产线上**装配电机组件（数据引擎策略）
3. **UnifoLM-X1-0**驱动机器人装配任务
4. **Embodied Avatar遥操作**训练新任务

"机器人造机器人"策略有双重目的：实际产能AND训练数据生成。

---

## 6. 商业模式与客户

### 收入来源

| 来源 | 描述 |
|------|------|
| **硬件销售** | 主要收入——直接机器人销售 |
| **EDU配置** | 更高利润率的研究/开发版本 |
| **组件** | 激光雷达传感器、执行器单独销售 |
| **消费品** | Go系列四足机器人在亚马逊销售 |

### 客户细分

| 细分 | 产品 | 示例 |
|------|------|------|
| **研究与教育** | G1 EDU、H1、R1 EDU | 大学、机器人实验室、AI研究中心 |
| **工业** | H2、B2、G1 | 巡检、物流、工厂部署 |
| **商业** | H2、R1 | 导引、接待、娱乐 |
| **消费** | Go2、R1 | 爱好者、发烧友、早期采用者 |

### 关键财务数据

- **年营收**：约10亿元人民币（约1.4亿美元），截至2025年中
- **50%海外**四足机器人销售
- **已盈利**——对于IPO前的机器人公司来说非常罕见
- 通过亚马逊和直销面向消费者——不像大多数竞争对手只做企业级

### 2026年商业化重点

王兴兴表示商业化将聚焦：
- **商业导引**（接待、零售）
- **相对固定的工业场景**（装配、巡检）
- **教育和研究**（R1以消费级价格切入）

---

## 7. 竞争对比

### 正面对决：七大玩家

| 公司 | 机器人 | 身高 | 自由度 | 价格 | 状态（2026年2月） | 2025年出货 |
|------|--------|------|--------|------|-------------------|-----------|
| **宇树** | G1 | 132cm | 23–43 | $13,500+ | **在售** | **5,500+** |
| **宇树** | H2 | 180cm | 31 | $29,900+ | 预售 | 含上述 |
| **智元机器人** | — | — | — | — | 在售 | **5,100+** |
| **特斯拉** | Optimus Gen 3 | 173cm | 40+ | $20–30K（目标） | **未上市** | <5,000（目标未达成） |
| **Figure AI** | Figure 03 | 168cm | 16/手 | 未公开 | **未上市** | 极少 |
| **波士顿动力** | Atlas（电动） | 150cm | 50 | $140–150K（估计） | **未上市** | 仅研究用 |
| **Agility Robotics** | Digit | 175cm | 20 | 仅企业级 | 企业试点 | 极少 |
| **1X Technologies** | NEO | 165cm | 75 | $20,000 | 预售/早期用户 | 极少 |
| **傅利叶智能** | GR-2 | 175cm | 53 | — | 在售 | — |

### 宇树的差异化

1. **真正在规模化出货**。大多数竞争对手是"未上市"或"仅企业试点"。宇树有5,500+台在客户手中。

2. **价格颠覆**。G1 $16K、R1 $5K让竞争对手显得荒谬。特斯拉Optimus的$20–30K目标还只是愿望；宇树已经做到了。

3. **完整产品线**。从$1,600（Go2）到$150K（H1）——覆盖每个市场细分。竞争对手通常只有一款产品。

4. **开放生态**。开源RL训练、ROS2支持、SDK文档——吸引研究社区并通过生态构建护城河。

5. **执行器垂直整合**。自研M107电机，号称世界最高扭矩密度。大多数竞争对手从供应商购买执行器。

6. **中国制造优势**。生产成本低50–70%，深度供应链整合，政府支持。

### 竞品深度分析

#### vs 特斯拉Optimus

特斯拉有巨大的资本和制造专长，但Optimus截至2026年2月**尚未上市**。特斯拉目标2027年底消费者销售。宇树在实际出货上领先2年。特斯拉承认来自中国公司的"激烈竞争"。

#### vs Figure AI

Figure 03在某些指标上被评为"最佳"人形机器人，但**不出售**。Figure专注于宝马工厂部署和商业试点。策略不同：Figure走企业优先，宇树走量优先。

#### vs 波士顿动力

Atlas是动态运动的标杆，但仍是**研究平台**，不商业化。波士顿动力2024年转向电动Atlas以便最终商用。现代汽车母公司提供制造能力但无上市时间表。

#### vs Agility Robotics（Digit）

Digit专注物流（运输、分拣、搬运）。仅企业级定价。有限生产。亚马逊是关键合作伙伴。比宇树的通用方向更窄。

#### vs 1X Technologies（NEO）

NEO在消费策略上与宇树最接近，售价$20,000。由OpenAI投资。专为家庭使用设计（每只手22自由度）。刚开始早期用户交付。不同的市场定位：家庭助手 vs 通用平台。

#### vs 傅利叶智能（GR-2）

中国同行竞争对手。GR-2有53自由度、380 N·m扭矩，为医疗保健设计（搬运病人）。傅利叶来自康复机器人背景。能力更强但聚焦更窄。

#### vs 智元机器人（AgiBot）

最接近的国内对手。智元2025年出货5,100+台人形机器人——几乎与宇树持平。两家公司在中国市场领导权上竞争激烈。智元、Figure AI、特斯拉、优必选和宇树被视为全球**第一梯队**。

---

## 8. 中国机器人生态系统

### 政府政策

中国政府已将人形机器人作为**国家战略优先**：

| 政策 | 详情 |
|------|------|
| **2025年3月政府工作报告** | 具身AI与量子、6G、生物制造并列为核心技术 |
| **十五五规划** | 中共建议将具身AI纳入经济增长驱动力 |
| **"机器人+"行动计划**（2023） | 推动制造、医疗、物流、教育领域的机器人应用 |
| **国家AI产业投资基金** | **82亿美元**用于前沿AI包括机器人 |
| **北京人形机器人政策**（2025） | 覆盖人形机器人全价值链的补贴 |
| **首批国家标准**（2025） | 批准制定人形机器人国家标准 |

### 财政支持

- **82亿美元国家AI产业投资基金**（2026年1月）
- 地方政府补贴和税收优惠
- **上海和深圳**产业园聚集机器人制造商、研究机构、供应商
- 北京人形机器人产业收入2025年上半年增长**约40%**，占全国总量的1/3

### 中国人形机器人公司

| 公司 | 特色 |
|------|------|
| **宇树** | 出货量领先，平价人形机器人 |
| **智元机器人** | 出货量第二（2025年5,100+） |
| **优必选** | 全球专利数量领先，深圳上市 |
| **傅利叶智能** | 医疗保健方向GR-2 |
| **Galbot** | 人形机器人便利店 |
| **擎朗智能** | 模块化设计，服务机器人 |

### 生态规模

- 中国有**150+家人形机器人公司**
- 行业以**年50%+**的速度扩张
- 预计2030年市场规模达**1000亿元（约142亿美元）**
- 按出货量计，中国控制**约90%的人形机器人市场**（宇树+智元大规模出货）
- 全球首个**7S人形机器人体验店**于2025年11月在武汉开业——政府支持

---

## 9. 关键技术决策与权衡

### 为什么选双足？

王兴兴从四足起步（公司前7年）。转向双足是战略性的：

- **人类环境为人类形态设计**——楼梯、门、工作站
- **客户需求**——工厂和办公室想要能融入现有人类设计空间的机器人
- **市场天花板**——四足市场较小（巡检、爱好） vs 人形（工业、服务、消费）
- **AI进步**——强化学习使大规模双足控制可行

### 为什么选平价？

王兴兴从第一天起的设计哲学：让足式机器人民主化。

- **量产创造数据**——更多机器人在现场 = 更多真实世界数据用于AI改进
- **市场扩张**——$100K+只有研究实验室买得起。$16K，大学、小公司和发烧友都买得起
- **制造飞轮**——量产证明工厂投资合理性，推动成本下降
- **生态建设**——平价平台吸引开发者，开发者构建应用，应用吸引更多客户

### 平价路线的权衡

| 牺牲了什么 | 为什么重要 |
|-----------|-----------|
| **峰值能力** | G1无法匹配Atlas的杂技或GR-2的380 N·m扭矩 |
| **负载能力** | G1：3kg/臂 vs Digit的16kg或GR-2的50kg |
| **电池续航** | R1：1小时。H2：3小时。工业部署需要8+小时 |
| **灵巧度** | 新开发的手部，仍落后于Figure 03的16自由度/手 |
| **外观** | G1是功能性的，不像人类（H2解决了这个） |
| **软件成熟度** | UnifoLM处于早期阶段，与特斯拉端到端神经网络相比 |

### 竞争对手高端路线的权衡

| 牺牲了什么 | 为什么重要 |
|-----------|-----------|
| **上市时间** | Atlas、Optimus、Figure 03仍"未上市" |
| **真实世界数据** | 没有部署单元就无法收集部署数据 |
| **成本降低学习** | 没有制造规模 = 没有成本曲线 |
| **生态系统** | 没有可及的硬件就没有开发者社区 |

### 核心赌注

宇树赌的是**出货20,000台不完美的机器人胜过开发1台完美的机器人**。已部署单元的数据飞轮最终会弥补能力差距，而竞争对手还在艰难地从实验室走向生产。

---

## 10. 近期新闻（2026年2月）

| 日期 | 事件 |
|------|------|
| **2月17日** | 春晚"赛博真功夫"表演——7亿+观众 |
| **2月17日** | 王兴兴宣布2026年10,000–20,000台人形机器人出货目标 |
| **2026年2月** | 春晚后订单积压 |
| **2026年2月** | 工厂视频展示G1使用UnifoLM-X1-0装配电机组件 |
| **2026年1月** | 宇树确认2025年人形机器人出货5,500+台，生产6,500+台 |
| **2026年1月** | 获得两项人形机器人外观设计专利 |
| **2026年1月** | 据报上海IPO计划2026年Q2，估值70亿美元 |

### 2026年战略重点

1. **产能扩展**至20,000台
2. **上海科创板IPO**，估值$7B+
3. **工厂部署**——用"机器人造机器人"证明工业ROI
4. **H2和R1交付**从Q2 2026开始
5. **商业化**聚焦导引和固定工业场景

---

## 11. 可借鉴的模式

### 对机器人公司

1. **分层产品线**：覆盖$5K、$15K、$30K、$100K价位。不同形态对应不同市场，共享执行器技术。

2. **通过自我部署构建数据引擎**：在自己的工厂使用自己的机器人。生产变成数据生成管线。一次投资两种产出。

3. **开源你的训练栈**：`unitree_rl_gym`吸引研究者发表使用你机器人的论文，同时创造营销和生态。

4. **执行器垂直整合**：自研M107电机是最大的成本和能力差异化因素。如果你造机器人，就要造自己的执行器。

### 对所有硬件公司

5. **出货不完美，快速迭代**：5,500台在现场产生真实数据，胜过实验室里5台完美原型。特斯拉和波士顿动力纸面上有更好的机器人，但零消费者部署。

6. **惊艳演示是免费营销**：春晚触达7亿观众，零广告费。造一些视觉震撼的东西来展示真实能力。

7. **定价以扩大市场，而非占领市场**：$100K，市场是1,000个研究实验室。$16K，是100,000个实验室+大学+公司+发烧友。更大的市场为高端竞争提供研发资金。

### 对AI公司

8. **跨实体模型**：UnifoLM适用于四足、人形和机械臂。不要构建特定任务模型——构建利用所有数据源的实体无关架构。

9. **世界模型+动作头**：WMA-0的双重架构（预测物理→决定动作）比纯强化学习更具样本效率。行动前先仿真。

---

## 最新動態 (2026)

### CES 2026 展示（2026年1月）

宇树在拉斯维加斯CES 2026上大放异彩，中国人形机器人企业约占所有人形机器人参展商的一半。宇树展示了完整产品线——H2、R1、G1，以及四足机器人A2和Go2。亮点是现场**G1拳击演示**：两台G1机器人戴着护头和拳套，以类似综合格斗选手的节奏互相出拳踢腿。公司代表确认**H2客户交付将于2026年4月开始**。宇树还展示了Go2-W，一款轮腿混合四足变体，可根据地形在行走和滚动之间切换。

### 世界人形机器人运动会——11枚奖牌，4枚金牌（2025年8月）

在北京举办的首届世界人形机器人运动会上——来自16个国家的280支队伍参加26个项目——宇树的H1机器人统治了田径赛场。他们赢得**四枚金牌**：400米短跑、1500米赛跑、100米跨栏和4×100米接力。总计宇树获得**11枚奖牌**。比赛中H1跑出**4.78 m/s**的速度，公司透露内部测试已突破**5 m/s**——为人形机器人技术树立了新标杆。独立团队使用G1平台也获得了一金一银一铜，展示了宇树硬件在外部开发者手中的多功能性。

### UnifoLM-VLA-0 开源（2025年）

继在Hugging Face上发布UnifoLM-WMA-0（世界模型-动作）之后，宇树开源了**UnifoLM-VLA-0**——一个为通用人形机器人操作设计的视觉-语言-动作模型。通过在机器人操作数据上持续预训练，该模型从纯"视觉-语言理解"进化为具备物理常识的"具身大脑"。在真实机器人验证中，仅用单一策略即可完成**12类复杂操作任务**。WMA-0和VLA-0均在GitHub和Hugging Face上提供预训练权重和训练/推理代码，延续宇树的生态建设策略。

### IPO 快速通道（2025年11月 – 2026年Q2）

宇树仅用**四个月**（2025年7-11月）就完成了与中信证券的IPO辅导——远快于通常的6-12个月流程，显示出强大的政府支持信号。据财新报道，上海科创板国内IPO预计在**2026年Q2末**完成，目标估值**70亿美元**。公司已连续五年盈利（2020-2024），毛利率超过**50%**——对于IPO前的硬件机器人公司来说极为罕见。

### 摩根士丹利翻倍中国人形机器人预测

摩根士丹利将2026年中国人形机器人销售预测翻倍至**28,000台**，高于此前约14,000台的估计。这标志着行业从研究原型向工业规模部署的转变。2025年宇树和智元合计占全球人形机器人出货量的近**80%**（总计约13,000台），中国在量产方面的主导地位正在加速。

### 现场娱乐与文化部署

除春晚之外，宇树扩展到现场娱乐领域：在CES 2026举办了**全球首场人形机器人拳击赛**，2025年全年将机器人融入戏剧演出和现场音乐会，并在商业导引场景中部署人形机器人。这些部署有双重目的——营销奇观和为操作及导航技能收集真实世界数据。

---

## 12. 风险因素与诚实评估

### 看多理由
- 具有实际出货量的主导市场地位
- 在烧钱行业中IPO前已盈利
- 中国生态：政府支持、供应链、人才
- 来自5,500+已部署单元的数据飞轮
- 产品线覆盖每个价格段

### 看空理由
- **70亿美元估值对应1.4亿美元营收** = 50倍收入倍数。对硬件公司来说很激进
- **40名员工**（如果准确）要支撑20,000台生产，似乎不可持续地精简
- **电池续航**限制工业部署可行性
- **UnifoLM处于早期阶段**——开源但尚未大规模验证
- **地缘政治风险**——美国/欧盟可能限制中国人形机器人进口
- **竞争加剧**——智元5,100台，特斯拉/Figure资本化$100B+
- **"出货" vs "部署"**——5,500台中有多少在积极使用 vs 在实验室吃灰？
- **尚无杀手级应用**——大多数部署是研究/演示，不是生产ROI

### 最大的问题

宇树的走量策略能否产生足够的部署数据来弥补与资金更充裕竞争对手的能力差距？还是特斯拉/Figure最终在制造上追赶上来的同时拥有更优的AI？

答案可能取决于**真实世界部署数据**（宇树的优势）还是**大规模算力+仿真**（特斯拉/OpenAI的优势）对具身AI更有价值。历史表明真实世界数据获胜，但投入仿真的算力规模是前所未有的。

---

## References

### Company Background & Funding

- [Unitree Robotics Plans Shanghai IPO in 2026 Q2 — $7B Valuation](https://www.caproasia.com/2026/01/06/china-unitree-robotics-plans-shanghai-ipo-in-2026-q2-with-previous-report-of-7-billion-valuation-raised-series-c-funding-at-1-7-billion-valuation-in-2025-june-founded-in-2016-by-wang-xingxing-key/) — Caproasia
- [Unitree Robotics Plans $7 Billion IPO in Shanghai](https://mlq.ai/news/unitree-robotics-plans-7-billion-ipo-in-shanghai/) — MLQ.ai
- [Unitree Completes Nearly 700M Yuan Series C Financing](https://eu.36kr.com/en/p/3344368397190018) — 36kr
- [Unitree Completes Pre-IPO Tutoring for Onshore Listing](https://www.scmp.com/business/banking-finance/article/3333047/chinas-unitree-robotics-completes-pre-ipo-tutoring-onshore-listing) — SCMP
- [Unitree Robotics IPO: A Strategic Play in China's Robotics Boom](https://www.ainvest.com/news/unitree-robotics-ipo-strategic-play-china-robotics-boom-2507/) — Ainvest
- [Unitree Robotics — Crunchbase](https://www.crunchbase.com/organization/unitree-robotics/company_financials) — Crunchbase
- [Unitree — CBInsights](https://www.cbinsights.com/company/unitree/financials) — CBInsights
- [Unitree — Tracxn Company Profile](https://tracxn.com/d/companies/unitree/__o1e8b3ZlyUCcjIECfbM9csfhnJyv1_fOku8o_K8gCYg) — Tracxn
- [Unitree Robotics is China's No.1 AI Firm](https://theprint.in/feature/unitree-robotics-robodog-ai-summit-galgotias/2857728/) — ThePrint
- [Unitree IPO Time Is Officially Set](https://eu.36kr.com/en/p/3450191506904711) — 36kr
- [Robot-Maker Unitree's IPO Expected by Mid-2026](https://www.caixinglobal.com/2026-01-05/robot-maker-unitrees-ipo-expected-by-mid-2026-source-says-102400282.html) — Caixin Global

### Founder — Wang Xingxing

- [The Rise of Wang XingXing's Unitree Robotics](https://mikekalil.com/blog/rise-of-unitree/) — Mike Kalil
- [Wang Xingxing — Wikipedia](https://en.wikipedia.org/wiki/Wang_Xingxing)
- [Unitree Founder Wang Xingxing: A Post-90s "Robotics Genius"](https://www.ourchinastory.com/en/14416/Unitree-founder-Wang-Xingxing:-A-post-90s-) — Our China Story
- [Meet Wang Xingxing at Xi Jinping's Symposium](https://www.scmp.com/tech/big-tech/article/3299435/meet-wang-xingxing-young-chinese-robotics-star-unitree-xi-jinpings-symposium) — SCMP
- [Inside Unitree: Robotics with Purpose](https://www.ceotodaymagazine.com/2025/06/inside-the-lab-how-unitree-is-engineering-the-future-of-accessible-robotic-mobility/) — CEO Today
- [About Us — Unitree Robotics](https://shop.unitree.com/pages/about-us) — Unitree Official

### Product Line

- [Unitree G1 Official Page](https://www.unitree.com/g1/) — Unitree
- [Unitree G1 Review 2026: Full Specs, Pricing](https://blog.robozaps.com/b/unitree-g1-review) — Robozaps
- [Unitree G1 — Pricing, Specs & Consultation](https://botinfo.ai/articles/unitree-g1) — BotInfo
- [Unitree G1 — UnitreeRobotics Shop](https://shop.unitree.com/products/unitree-g1) — Unitree Shop
- [Unitree H1 Robot: Complete Specs & Buying Guide](https://botinfo.ai/articles/unitree-h1-humanoid) — BotInfo
- [Unitree H2 Overview: New Features & Key Differences](https://community.robotshop.com/blog/show/unitree-h2-overview-new-features-key-differences-from-the-h1-h1-2) — RobotShop
- [Unitree H2 Official Page](https://www.unitree.com/mobile/H2/) — Unitree
- [Unitree H2 — Humanoid.guide](https://humanoid.guide/product/h2/) — Humanoid Guide
- [Unitree H2: Specifications, Bionics, and Next-Gen Humanoid](https://humanoid.press/database/humanoid-press-database-unitree-h2-bionic-humanoid-robot-specs/) — Humanoid Press
- [Unitree H2 EDU — RobotShop](https://www.robotshop.com/products/unitree-h2-edu-humanoid-robot) — RobotShop
- [Unitree R1 Official Page](https://www.unitree.com/R1/) — Unitree
- [Unitree R1 Review 2026: $4,900 Humanoid](https://blog.robozaps.com/b/unitree-r1-review) — Robozaps
- [Unitree R1: The Best Inventions of 2025](https://time.com/collections/best-inventions-2025/7318495/unitree-r1/) — TIME
- [Unitree R1 — $5,900 Humanoid That Flips and Kicks](https://www.techeblog.com/unitree-r1-humanoid-robot-price-video/) — TechEBlog
- [Unitree R1 — Affordable Humanoid Robot Launched](https://newatlas.com/ai-humanoids/unitree-r1-humanoid-robot/) — New Atlas
- [Unitree Robotics — Wikipedia](https://en.wikipedia.org/wiki/Unitree_Robotics) — Wikipedia

### Technical Architecture — UnifoLM

- [Unitree Unveils Open-Source World-Model for Robots](https://robohorizon.com/en-us/news/2025/09/unitree-unveils-open-source-world-model-for-robots/) — RoboHorizon
- [China's Unitree Open-Sources World Model](https://www.yicaiglobal.com/news/chinas-unitree-open-sources-world-model-to-advance-robotics-ecosystem) — Yicai Global
- [Unitree G1-D End-to-End Platform](https://www.unitree.com/mobile/G1-D/) — Unitree
- [Unitree G1 Hypermobility with UnifoLM](https://xpert.digital/en/unitree-g1/) — Xpert Digital
- [Unitree Robotics — GitHub](https://github.com/unitreerobotics) — GitHub
- [unitree_rl_gym — Reinforcement Learning Gym](https://github.com/unitreerobotics/unitree_rl_gym) — GitHub
- [unitree_rl_lab — IsaacLab Based RL](https://github.com/unitreerobotics/unitree_rl_lab) — GitHub
- [unitree_rl_mjlab — MuJoCo Based RL](https://github.com/unitreerobotics/unitree_rl_mjlab) — GitHub
- [UnifoLM-VLA-0 — Unitree Open-Sources Multimodal VLA Model](https://pandaily.com/unitree-robotics-open-sources-multimodal-vision-language-action-model-unifo-lm-vla-0-1) — Pandaily
- [UnifoLM-WMA-0 on Hugging Face](https://huggingface.co/unitreerobotics/UnifoLM-WMA-0-Base) — Hugging Face
- [UnifoLM-VLA-0 Collection on Hugging Face](https://huggingface.co/collections/unitreerobotics/unifolm-vla-0) — Hugging Face

### Data Engine & Factory Automation

- [Robots Making Robots: Unitree Factory](https://interestingengineering.com/videos/robots-making-robots-unitree-factory) — Interesting Engineering
- [Unitree Deploys G1 Humanoids to Manufacture Robot Parts](https://www.humanoidsdaily.com/news/unitree-deploys-g1-humanoids-to-manufacture-robot-parts) — Humanoids Daily

### Actuator & Sensor Technology

- [H1 Overview — M107 Joint Motor](https://www.docs.quadruped.de/projects/h1/html/h1_overview.html) — Quadruped Docs
- [QDD Actuator — Unitree Robotics](https://unitree.arcsecondrobo.net/technology) — Unitree (Arcsecond)
- [Unitree Go2 Motor Teardown](https://www.simplexitypd.com/blog/unitree-go2-motor-teardown/) — Simplexity PD
- [Unitree 4D LiDAR L2](https://www.unitree.com/L2/) — Unitree
- [Unitree 4D LiDAR L1](https://www.unitree.com/LiDAR/) — Unitree
- [G1 Overview — Sensors & Specs](https://docs.quadruped.de/projects/g1/html/g1_overview.html) — Quadruped Docs

### Spring Festival Gala 2026

- [Kung Fu Meets Spring — Unitree Gala Robots](https://theaiinsider.tech/2026/02/17/unitree-spring-festival-gala-robots-present-cyber-real-kung-fu-in-the-year-of-the-horse/) — The AI Insider
- [Parkour, Drunken Fist and Nunchaku — Unitree Robots at 2026 Gala](https://www.globaltimes.cn/page/202602/1355439.shtml) — Global Times
- [Kung Fu Meets Spring — PRNewswire](http://www.prnewswire.com/news-releases/kung-fu-meets-spring--unitree-spring-festival-gala-robots-present-cyber-real-kung-fu-in-the-year-of-the-horse-302689281.html) — PRNewswire
- [Humanoid Robots Perform Drunken Kung Fu with Nunchucks](https://interestingengineering.com/ai-robotics/humanoid-robots-wield-nunchucks-china) — Interesting Engineering
- [Humanoid Robots Take Center Stage at 2026 Gala](https://technode.com/2026/02/17/humanoid-robots-take-center-stage-at-2026-spring-festival-gala-revealing-chinas-latest-robotics-advances/) — TechNode
- [How Unitree Robots Prepare for Spring Festival Gala](https://english.news.cn/20260218/5cf1ec9585f644bd84c60d6c976c483f/c.html) — Xinhua
- [Robot-Heavy Gala Meets Sceptical Youth](https://www.digitimes.com/news/a20250217VL212/2026-industrial-robot-robotics-unitree.html) — Digitimes
- [China's Unitree Showcases Eerily Lifelike Robot Kung-Fu](https://www.livescience.com/technology/robotics/humanoid-robots-show-off-creepily-impressive-kung-fu-moves-during-lunar-new-year-festival-in-china) — Live Science
- [Kung Fu Robot at the 2026 Spring Festival Gala](https://www.geopolitechs.org/p/kung-fu-robot-at-the-2026-spring) — Geopolitechs
- [Search for AI Behind the Horse-Year Spring Festival Gala](https://eu.36kr.com/en/p/3686529041690246) — 36kr

### Production & Scale

- [Unitree Targets 20,000 Humanoid Robots with Fourfold Capacity Increase](https://interestingengineering.com/ai-robotics/unitree-targets-20000-humanoid-robots) — Interesting Engineering
- [Unitree Eyes 20,000-Robot Output in 2026](https://www.scmp.com/tech/big-tech/article/3343825/kung-fu-somersaults-and-scale-unitree-eyes-20000-robot-output-2026-after-gala) — SCMP
- [Unitree Claims 5,500 Humanoid Robots Shipped](https://briefglance.com/articles/unitree-claims-5500-humanoid-robots-shipped-igniting-market-race) — BriefGlance
- [China's Unitree Ships More Than 5,500 Humanoids in 2025](https://www.scmp.com/tech/tech-trends/article/3340446/chinas-unitree-ships-more-5500-humanoid-robots-2025-surpassing-us-peers) — SCMP
- [Unitree Granted Two Humanoid Robot Design Patents](https://technode.com/2026/01/20/unitree-granted-two-humanoid-robot-design-patents-ships-over-5500-units-in-2025/) — TechNode
- [Unitree Founder Expects Up to 20,000 Shipments](https://cntechpost.com/2026/02/17/unitree-founder-expects-up-to-20000-humanoid-robot-shipments-2026/) — CnTechPost
- [Robotics Firms See Backlog in Orders After Gala](https://www.scmp.com/tech/tech-trends/article/3343908/robotics-firms-see-backlog-orders-after-humanoids-steal-show-spring-festival-gala) — SCMP
- [Unitree Targets 20,000 Humanoid Robots in 2026](https://startupnews.fyi/2026/02/18/unitree-20000-humanoid-robots-2026/) — StartupNews

### Business Model & Customers

- [How Does Unitree Robotics Company Work?](https://canvasbusinessmodel.com/blogs/how-it-works/unitree-robotics-how-it-works) — Canvas Business Model
- [Unitree: The Cost-Effective Challenger](https://site.financialmodelingprep.com/market-news/unitree-the-costeffective-challenger-in-aipowered-robotics) — FMP
- [Unitree Customer Demographics & Target Market](https://canvasbusinessmodel.com/blogs/target-market/unitree-robotics-target-market) — Canvas Business Model
- [Unitree Becomes a Legged Robot Unicorn](https://www.therobotreport.com/unitree-becomes-a-legged-robot-unicorn-with-series-c-funding/) — The Robot Report
- [A Deep-Dive into Unitree](https://www.investing.com/news/stock-market-news/a-deepdive-into-unitree-a-chinabased-developer-of-costeffective-ai-robots-3841614) — Investing.com
- [Introducing Unitree, China's Leading AI-Embodied Robotics Company](https://aiproem.substack.com/p/introducing-unitree-chinas-leading) — AI Proem (Substack)

### Competition Comparison

- [Humanoid Robot Comparison 2026: Complete Breakdown of Top 12 Models](https://botinfo.ai/articles/humanoid-robot-comparison) — BotInfo
- [Humanoid Robots in Comparison: Tesla, Boston Dynamics, Agility, Unitree](https://xpert.digital/en/robot-comparison/) — Xpert Digital
- [Top 12 Humanoid Robotics Companies to Watch in 2026](https://standardbots.com/blog/humanoid-robotics-companies) — Standard Bots
- [28 Best Humanoid Robots 2026 Ranked](https://blog.robozaps.com/b/best-humanoid-robots) — Robozaps
- [Humanoid Robot Builders Cheatsheet (Nov 2025)](https://cheatsheets.davidveksler.com/humanoid-robots.html) — David Veksler
- [1X NEO Humanoid Robot: Complete 2026 Buyer's Guide](https://botinfo.ai/articles/1x-neo-home-robot) — BotInfo
- [Tesla Optimus: Complete Analysis](https://botinfo.ai/articles/tesla-optimus) — BotInfo
- [Fourier GR-2 Official Page](https://www.fftai.com/products-gr2) — Fourier Intelligence
- [Fourier GR-2 — Human-Like Motion and Flexibility](https://interestingengineering.com/innovation/fouriers-gr-2-upgraded-humanoid-robot) — Interesting Engineering
- [Fourier Trains Humanoid Robots Using NVIDIA Isaac Gym](https://developer.nvidia.com/blog/spotlight-fourier-trains-humanoid-robots-for-real-world-roles-using-nvidia-isaac-gym) — NVIDIA

### China vs US & AgiBot

- [Chinese Firms Outpace US Rivals in 2025 Humanoid Shipments](https://www.scmp.com/tech/tech-trends/article/3339346/chinese-firms-outpace-us-rivals-2025-humanoid-robot-shipments-agibot-takes-lead) — SCMP
- [Chinese Firms Lead Global Humanoid Robot Production in 2025](https://english.news.cn/20260109/bab6612656664145bb5becc3781edd59/c.html) — Xinhua
- [China Is Winning the Humanoid Robot Race](https://restofworld.org/2026/china-humanoid-robots-unitree-agibot-tesla-optimus/) — Rest of World
- [China Dominates Humanoid Robot Shipments](https://www.silicon.co.uk/e-innovation/artificial-intelligence/robot-china-628278) — Silicon UK
- [Unitree Heats Up Humanoid Robot Race — $7B IPO](https://www.cnbc.com/2025/09/09/chinas-unitree-plans-7-billion-ipo-valuation-as-humanoid-robot-race-heats-up.html) — CNBC

### China Robotics Ecosystem & Policy

- [Embodied Intelligence: The PRC's Whole-of-Nation Push into Robotics](https://jamestown.org/embodied-intelligence-the-prcs-whole-of-nation-push-into-robotics/) — Jamestown Foundation
- [Embodied AI: China's Big Bet on Smart Robots](https://carnegieendowment.org/research/2025/11/embodied-ai-china-smart-robots?lang=en) — Carnegie Endowment
- [China Plans to Mass Produce Humanoids by 2025](https://www.therobotreport.com/china-plans-to-mass-produce-humanoids-by-2025/) — The Robot Report
- [Beijing Unveils Comprehensive Humanoid Robot Policies](https://www.investing.com/news/economy-news/beijing-unveils-comprehensive-humanoid-robot-policies-at-wrc-2025-93CH-4182024) — Investing.com
- [China's First National Standards for Humanoid Robots](https://english.beijing.gov.cn/beijinginfo/sci/event/202504/t20250424_4073087.html) — Beijing Gov
- [China's First 7S Humanoid Robot Store](http://www.china.org.cn/2026-02/17/content_118337117.shtml) — China.org
- [China's Humanoid Robots Step Toward Scalable Industrial Reality](http://english.scio.gov.cn/in-depth/2026-01/04/content_118259919.html) — SCIO
- [Humanoid Robots — USCC Report](https://www.uscc.gov/sites/default/files/2024-10/Humanoid_Robots.pdf) — US-China Economic and Security Review Commission
- [China's Service Robots Rollout Propelled by Government Policy](https://www.globaltimes.cn/page/202509/1344626.shtml) — Global Times

### CES 2026 & World Humanoid Robot Games

- [Unitree Robotics at CES 2026: A Clear Signal of What's Coming Next](https://community.robotshop.com/blog/show/unitree-robotics-at-ces-2026-a-clear-signal-of-whats-coming-next) — RobotShop
- [9 Humanoid Robots at CES 2026 That Showed the Future Is Already Here](https://interestingengineering.com/ai-robotics/9-humanoid-robots-at-ces-2026) — Interesting Engineering
- [China's Humanoid Robot Firms Make Up Half of Exhibitors at CES 2026](https://interestingengineering.com/ces-2026/china-leads-humanoid-robotics-at-ces-2026) — Interesting Engineering
- [Real Steel Fantasy Turns Real as Humanoid Robots Fight at CES 2026](https://interestingengineering.com/ai-robotics/humanoid-robots-fight-ces-2026) — Interesting Engineering
- [Unitree Dominates Inaugural World Humanoid Robot Games with Four Gold Medals](https://roboticsandautomationnews.com/2025/08/26/unitree-dominates-inaugural-world-humanoid-robot-games-with-four-gold-medals/93926/) — Robotics and Automation News
- [Unitree, X-Humanoid Top Medal Total in World's First Humanoid Robot Games](https://www.scmp.com/tech/tech-trends/article/3322251/chinas-unitree-x-humanoid-top-medal-total-worlds-first-humanoid-robot-games) — SCMP
- [Unitree Dominates Inaugural Humanoid Robot Games with 11 Medals](https://interestingengineering.com/innovation/humanoid-robot-games-unitree-dominates-with-11-medals) — Interesting Engineering
