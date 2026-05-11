# oh-my-openagent Agent 模型选型设计

> 状态：设计阶段 | 目标：全 LiteLLM 国产模型 + Zen 免费兜底

---

## 一、可用模型

### 1.1 LiteLLM 国产模型分层

| 层级 | 模型 | 视觉 | 推理 | 编码 | 速度 | 适合角色 |
|---|---|---|---|---|---|---|
| 顶级推理 | `deepseek-v4-pro` | N | 5/5 | 4/5 | 中 | 架构/分析/排错/编排 |
| 强推理 | `deepseek-v3.2` | N | 4/5 | 5/5 | 中 | 代码生成/实现 |
| 全能视觉 | `glm-5.1` | Y | 4/5 | 4/5 | 中 | 通用/视觉/创意 |
| 全能视觉 | `glm-5` | Y | 3/5 | 3/5 | 中 | 视觉备选 |
| 强编码 | `MiniMax-M2.7` | N | 4/5 | 4/5 | 中 | 编码实现备选 |
| 轻量快速 | `qwen3.6-plus` | Y | 3/5 | 4/5 | 快 | 搜索/轻量任务 |
| 轻量快速 | `deepseek-v4-flash` | N | 3/5 | 3/5 | 快 | 批量简单任务 |
| 轻量快速 | `MiniMax-M2.7-highspeed` | N | 3/5 | 3/5 | 快 | 高速备选 |
| 上代 | `MiniMax-M2.5` | N | 2/5 | 3/5 | 中 | 低端备选 |

### 1.2 OpenCode Zen 免费模型（末尾兜底）

| 模型 ID | 说明 |
|---|---|
| `opencode/big-pickle` | Zen 隐身模型 |
| `opencode/minimax-m2.5-free` | MiniMax M2.5 免费版 |
| `opencode/nemotron-3-super-free` | Nemotron 3 Super 免费版 |

### 1.3 约束条件

1. **主模型全部使用 LiteLLM 国产模型**，不依赖 Claude/GPT/Gemini
2. **回退链末尾必须包含 Zen 免费模型** 作为最终保底
3. **能力匹配职责**：不浪费贵模型，不让弱模型做重活

---

## 二、Agents 配置

### 2.1 sisyphus -- 主编排器

| 维度 | 内容 |
|---|---|
| **国产选型** | `LiteLLM/deepseek-v4-pro` |
| **回退链** | `LiteLLM/deepseek-v3.2` -> `LiteLLM/MiniMax-M2.7` -> `LiteLLM/glm-5.1` -> `opencode/big-pickle` |
| **国外参照** | Claude Opus 4.7 / GPT-5.5 |

**适用场景**：
1. 涉及多模块的需求 -- 拆成前端/后端/数据库/测试，并行委派子 agent
2. 模糊请求 -- 先派 explore/librarian 探明代码库再决定
3. 多个子 agent 返回结果后的汇总和矛盾消解
4. 用户中途变更需求时的策略调整

**任务特点**：多轮对话型。不写代码，只做决策。每回合可能派发 3-5 个并行子任务，同时保持全链路上下文。常见动作：task()、todowrite()、question()。核心瓶颈是指令精准度和上下文管理。

### 2.2 oracle -- 高 IQ 只读顾问

| 维度 | 内容 |
|---|---|
| **国产选型** | `LiteLLM/deepseek-v4-pro` |
| **回退链** | `LiteLLM/deepseek-v3.2` -> `LiteLLM/MiniMax-M2.7` -> `LiteLLM/glm-5.1` -> `opencode/big-pickle` |
| **国外参照** | Claude Opus 4.7 / GPT-5.5 / Grok 4 |

**适用场景**：
1. 跨模块架构选型：微服务 vs 单体、REST vs GraphQL、SQL vs NoSQL
2. 疑难 bug 根因分析：同一 crash 修了 3 次无效 -> 从调用栈倒推真正的根因
3. 安全审查：OAuth token 泄露风险、SQL 注入点
4. 性能瓶颈诊断：渲染慢/查询慢 -> 定位算法层/数据库层/网络层
5. 代码异味诊断：复杂度超标模块 -> 判断重构还是重写

**任务特点**：只读分析型。不碰代码。输入："问题描述 + 代码上下文 + 已尝试的方法"；输出：结构化分析结论（根因、证据链、建议方案、风险点）。单次耗时较长但触发频率低。成败取决于在有限信息下做出正确判断。

### 2.3 prometheus -- 计划制定者

| 维度 | 内容 |
|---|---|
| **国产选型** | `LiteLLM/deepseek-v4-pro` |
| **回退链** | `LiteLLM/deepseek-v3.2` -> `LiteLLM/MiniMax-M2.7` -> `LiteLLM/glm-5.1` -> `opencode/minimax-m2.5-free` |
| **国外参照** | Claude Opus 4.7 / GPT-5.5 |

**适用场景**：
1. 新功能开发：收到需求 -> 扫描相关代码模块 -> 输出分步执行计划
2. 大型重构：识别影响范围（哪些文件要改、哪些调用者受影响）
3. 迁移项目：Express -> Fastify -> 识别中间件映射、路由重写、测试迁移顺序
4. 依赖分析：新加 npm 包 -> 判断冲突/替代方案

**任务特点**：规划输出型。输入：需求描述 + 代码库概览；输出：.sisyphus/plans/*.md 文件。要求并行机会识别准确、步骤粒度适中。事后 momus 会评审。

### 2.4 metis -- 预规划顾问

| 维度 | 内容 |
|---|---|
| **国产选型** | `LiteLLM/deepseek-v4-pro` |
| **回退链** | `LiteLLM/deepseek-v3.2` -> `LiteLLM/glm-5.1` -> `LiteLLM/MiniMax-M2.7` -> `opencode/big-pickle` |
| **国外参照** | Claude Opus 4.7 / Gemini 3.1 Pro |

**适用场景**：
1. "优化性能" -- 识别：没说哪个页面、没说指标、没说当前瓶颈
2. "改一下登录" -- 识别：改 UI？改逻辑？改第三方登录？改密码策略？
3. 技术上矛盾的需求（如"快且不缓存"）-- 标记需澄清
4. 看似简单但隐含副作用的需求（如"删除旧接口" -- 警告需先确认调用方）

**任务特点**：前置拦截型。在 prometheus 规划之前运行，只输出"需要澄清的问题清单"和"可能的误解方向"。目标是防止整个执行链跑偏。

### 2.5 momus -- 计划/质量评审

| 维度 | 内容 |
|---|---|
| **国产选型** | `LiteLLM/deepseek-v4-pro` |
| **回退链** | `LiteLLM/deepseek-v3.2` -> `LiteLLM/glm-5.1` -> `LiteLLM/MiniMax-M2.7` -> `opencode/nemotron-3-super-free` |
| **国外参照** | Claude Opus 4.7 / GPT-5.5 |

**适用场景**：
1. 评审 prometheus 输出的计划：检查步骤缺失、依赖标注、成功标准可验证性
2. 评审实现成果：todo 是否完成、lint 是否通过、是否引入新错误
3. QA gate：代码合并前检查是否符合 spec、是否覆盖边界条件

**任务特点**：批判性审查型。输入：计划文件/实现结果；输出：缺陷清单 + 改进建议。需要不放过任何细节的严谨性。

### 2.6 hephaestus -- 默认构建 agent

| 维度 | 内容 |
|---|---|
| **国产选型** | `LiteLLM/deepseek-v3.2` |
| **回退链** | `LiteLLM/MiniMax-M2.7` -> `LiteLLM/deepseek-v4-pro` -> `LiteLLM/glm-5.1` -> `opencode/big-pickle` |
| **国外参照** | Claude Sonnet 4.6 / GPT-5.4 / Grok 4 |

**适用场景**：
1. 执行不指定 category 的 task() 调用 -- 写代码、读文件、跑命令
2. 理解现有代码模式后生成风格一致的代码
3. 工具链操作：npm install、git 命令、LSP 检查
4. 单一文件修改 + 验证

**任务特点**：代码产出型。实际写代码最频繁的 agent。要求模式识别准确、工具链熟练。触发频率最高。

### 2.7 atlas -- 通用执行 agent

| 维度 | 内容 |
|---|---|
| **国产选型** | `LiteLLM/glm-5.1` |
| **回退链** | `LiteLLM/qwen3.6-plus` -> `LiteLLM/MiniMax-M2.7` -> `LiteLLM/deepseek-v3.2` -> `opencode/minimax-m2.5-free` |
| **国外参照** | Gemini 3.1 Pro / Claude Sonnet 4.6 |

**适用场景**：
1. 跨领域任务：代码修改 + 文档整理 + 图片处理
2. 代码库全面分析：梳理项目结构、依赖关系、数据流向
3. 环境配置：检查 .env、tsconfig.json、package.json 一致性
4. 不确定 category 的泛用任务

**任务特点**：泛化执行型。处理代码+图片+文档+配置的混合任务。视觉能力是关键差异化 -- 可能读到架构图、UI 截图等。

### 2.8 explore -- 代码库上下文 grep

| 维度 | 内容 |
|---|---|
| **国产选型** | `LiteLLM/qwen3.6-plus` |
| **回退链** | `LiteLLM/deepseek-v4-flash` -> `LiteLLM/MiniMax-M2.7-highspeed` -> `LiteLLM/MiniMax-M2.5` -> `opencode/big-pickle` |
| **国外参照** | Gemini 3 Flash / GPT-5.4-nano |

**适用场景**：
1. "X 在哪里"：找到所有 auth middleware、handleLogin 函数定义
2. "谁调用了 Y"：搜索所有文件中 getUserToken 的调用
3. 寻找模式：所有 try/catch 块、所有 useEffect 调用
4. 寻找配置：baseURL 在所有配置文件中的设置

**任务特点**：高频并行型。通常一次派发 3-5 个并行实例。要求速度快 + 返回精确。不需要深度推理，只需准确匹配。

### 2.9 librarian -- 外部参考 grep

| 维度 | 内容 |
|---|---|
| **国产选型** | `LiteLLM/qwen3.6-plus` |
| **回退链** | `LiteLLM/deepseek-v4-flash` -> `LiteLLM/MiniMax-M2.7-highspeed` -> `LiteLLM/MiniMax-M2.5` -> `opencode/big-pickle` |
| **国外参照** | Gemini 3 Flash / GPT-5.4-mini |

**适用场景**：
1. 查找不熟悉库的用法：next-intl v4、express-jwt 等
2. 搜索开源实现参考：GitHub code search
3. 查阅官方文档：Context7 搜索 API 签名和参数
4. 查找最佳实践：React Query optimistic update 等
5. 排查依赖行为异常：搜索 issue 和 workaround

**任务特点**：外搜检索型。GitHub code search + Context7 + Web Search 三渠道。要求多源整合、精确提取。和 explore 一样是高频并行任务。

### 2.10 multimodal-looker -- 多模态分析

| 维度 | 内容 |
|---|---|
| **国产选型** | `LiteLLM/glm-5.1` |
| **回退链** | `LiteLLM/qwen3.6-plus` -> `LiteLLM/glm-5` -> `opencode/big-pickle` |
| **国外参照** | Gemini 3.1 Pro / GPT-5.5 |

**适用场景**：
1. 分析架构图：从 .png/.pdf 中提取组件关系、数据流向
2. 解析 UI 截图：提取布局结构、组件层级、颜色方案
3. OCR 提取：从扫描件/截图中提取文字
4. 理解图表：从折线图/柱状图中读取数据和趋势

**任务特点**：视觉密集型。必须有视觉能力，这是硬约束。输入：图像/PDF；输出：结构化文字描述。低频但一旦调用就需可靠的多模态理解。
> ⚠️ 回退链全程保持视觉能力（`glm-5.1` → `qwen3.6-plus` → `glm-5`），无视觉模型（`deepseek-v3.2`）已剔除。

### 2.11 sisyphus-junior -- 任务执行器

| 维度 | 内容 |
|---|---|
| **国产选型** | `LiteLLM/deepseek-v3.2` |
| **回退链** | `LiteLLM/MiniMax-M2.7` -> `LiteLLM/deepseek-v4-pro` -> `LiteLLM/glm-5.1` -> `opencode/minimax-m2.5-free` |
| **国外参照** | Claude Sonnet 4.6 / GPT-5.4 |

**适用场景**：
1. 接收 task(category="xxx") 调用的具体执行
2. 根据 category 加载不同模型
3. 收到明确的文件路径 + 约束条件 + 成功标准后执行实现
4. 执行后自我验证

**任务特点**：指令执行型。不自主决策，不自行委派。最大的特点是被 category 覆盖模型 -- 这是整个编排体系中最底层的执行单元。

## 三、Categories 配置

### 3.1 visual-engineering -- UI/设计/动画

| 维度 | 内容 |
|---|---|
| **国产选型** | `LiteLLM/glm-5.1` |
| **回退链** | `LiteLLM/qwen3.6-plus` -> `LiteLLM/glm-5` -> `opencode/nemotron-3-super-free` |
| **国外参照** | Gemini 3.1 Pro / Claude Opus 4.7 |

**适用场景**：前端页面开发、UI 动画、样式重构、设计稿还原、无障碍优化

**任务特点**：视觉是硬需求。必须理解图像才能做对 UI。同时需要审美判断力 -- 不能只是功能能用，还要看起来像专业工程师写的。
> ⚠️ 回退链全程保持视觉能力（`glm-5.1` → `qwen3.6-plus` → `glm-5`），无视觉模型已剔除。

### 3.2 ultrabrain -- 高难度逻辑

| 维度 | 内容 |
|---|---|
| **国产选型** | `LiteLLM/deepseek-v4-pro` |
| **回退链** | `LiteLLM/deepseek-v3.2` -> `LiteLLM/MiniMax-M2.7` -> `LiteLLM/glm-5.1` -> `opencode/big-pickle` |
| **国外参照** | Claude Opus 4.7 / GPT-5.5 / Grok 4 |

**适用场景**：算法设计、复杂状态机、数学密集型任务、编译器/AST 操作

**任务特点**：纯脑力劳动。代码量不一定大但每步都有逻辑陷阱。在所有 category 中推理深度要求最高。

### 3.3 deep -- 自主端到端

| 维度 | 内容 |
|---|---|
| **国产选型** | `LiteLLM/deepseek-v3.2` |
| **回退链** | `LiteLLM/MiniMax-M2.7` -> `LiteLLM/deepseek-v4-pro` -> `LiteLLM/glm-5.1` -> `opencode/minimax-m2.5-free` |
| **国外参照** | Claude Opus 4.7 / GPT-5.5 |

**适用场景**：端到端功能实现、自主探索型任务、跨 10+ 文件的复杂重构、新模块搭建

**任务特点**：自主探索+实现一体化。需要自己搞清楚改哪些文件、参考哪些模式。长链推理+编码能力并重。

### 3.4 artistry -- 创意解题

| 维度 | 内容 |
|---|---|
| **国产选型** | `LiteLLM/deepseek-v4-pro` |
| **回退链** | `LiteLLM/deepseek-v3.2` -> `LiteLLM/MiniMax-M2.7` -> `LiteLLM/glm-5.1` -> `opencode/big-pickle` |
| **国外参照** | Claude Opus 4.7 / GPT-5.5 |

**适用场景**：非常规解题（黑盒系统集成、无文档遗留系统）、创意生成（文案/命名/可视化创意）、反直觉优化

**任务特点**：发散思维+打破常规。常规问题用常规方法，artistry 处理的是"正常解法走不通"的场景。核心依赖是深度推理 + 代码理解（而非视觉），需要在常见方案之外找出反直觉的路径。选 `deepseek-v4-pro` 而非 `glm-5.1` 是因为：artistry 的"黑魔法"更多是代码层面突破，而非多模态理解。
> 🔄 v1 版本选了 `glm-5.1`，审查后改为 `deepseek-v4-pro`：artistry 任务以代码推理为主，非视觉驱动。

### 3.5 quick -- 单文件快改

| 维度 | 内容 |
|---|---|
| **国产选型** | `LiteLLM/qwen3.6-plus` |
| **回退链** | `LiteLLM/deepseek-v4-flash` -> `LiteLLM/MiniMax-M2.7-highspeed` -> `LiteLLM/MiniMax-M2.5` -> `opencode/big-pickle` |
| **国外参照** | Gemini 3 Flash / GPT-5.4-nano |

**适用场景**：单文件小改（typo/变量名/日志）、配置更新（版本号/env URL）、简单重构（提取常量/拆分函数/添加 early return）、注释补充

**任务特点**：极快极简。单文件、单个修改点、不需要理解全局上下文。触发频率很高但每次只消耗极少推理资源。速度 > 深度。

### 3.6 unspecified-low -- 低投入杂务

| 维度 | 内容 |
|---|---|
| **国产选型** | `LiteLLM/qwen3.6-plus` |
| **回退链** | `LiteLLM/deepseek-v4-flash` -> `LiteLLM/MiniMax-M2.7-highspeed` -> `LiteLLM/MiniMax-M2.5` -> `opencode/big-pickle` |
| **国外参照** | Gemini 3 Flash / GPT-5.4-mini |

**适用场景**：纯文本处理（格式化 JSON、转换 Markdown）、简单脚本（curl/sed/bash）、元数据操作（读取依赖列表、行数统计）、模板填充

**任务特点**：低认知负载。任务本身不复杂，不需要推理，只需要执行。

### 3.7 unspecified-high -- 高投入杂务

| 维度 | 内容 |
|---|---|
| **国产选型** | `LiteLLM/deepseek-v3.2` |
| **回退链** | `LiteLLM/MiniMax-M2.7` -> `LiteLLM/deepseek-v4-pro` -> `LiteLLM/glm-5.1` -> `opencode/nemotron-3-super-free` |
| **国外参照** | Claude Sonnet 4.6 / GPT-5.4 |

**适用场景**：不属于其他 category 但需要较高质量的任务（如内部 CLI 工具）、跨领域混合任务、需要可靠性但不一定需要深度推理的工作

**任务特点**：高阶兜底。当一个任务没有匹配的专用 category 但又需要比 quick 更高的质量保证时使用。

### 3.8 writing -- 文档写作

| 维度 | 内容 |
|---|---|
| **国产选型** | `LiteLLM/glm-5.1` |
| **回退链** | `LiteLLM/qwen3.6-plus` -> `LiteLLM/deepseek-v4-pro` -> `LiteLLM/deepseek-v3.2` -> `opencode/big-pickle` |
| **国外参照** | Claude Opus 4.7 / GPT-5.5 |

**适用场景**：技术文档（README/API文档/架构说明）、changelog、ADR 决策记录、注释撰写

**任务特点**：文字质量优先。需要结构感、准确性和可读性。glm-5.1 在中文文档场景表现出色。

---

## 四、模型分配全景图

```
                    deepseek-v4-pro (推理深度优先)
                   /         |           \            \
            sisyphus    oracle    prometheus    metis    momus    ultrabrain    artistry
           (编排+决策)  (架构诊断)  (计划制定)  (歧义检测) (评审)  (高难逻辑)  (创意解题)

                    deepseek-v3.2 (编码产出优先)
                   /        |          \
            hephaestus  sisyphus-junior  deep    unspecified-high
           (代码生成)  (任务执行)     (端到端)   (高阶兜底)

                    glm-5.1 (视觉+全能优先)
                   /     |          \            \
            atlas   multimodal-looker  visual-engineering  writing
           (通用)    (多模态分析)       (UI/设计)          (文档)

                    qwen3.6-plus (速度+成本优先)
                   /          \            \
            explore        librarian    quick    unspecified-low
           (代码搜索)     (外部搜索)   (快改)    (低投入杂务)

           每个回退链末尾 -> opencode/big-pickle 或 minimax-m2.5-free 或 nemotron-3-super-free
```

---

## 五、回退链设计逻辑

每条链遵循"同能力降级"原则：

```
主模型 -> 同能力备选 -> 增强保底 -> 全能模型兜底 -> Zen 免费
```

**示例 -- oracle 的回退链**：

| 序号 | 模型 | 角色 |
|---|---|---|
| 1 | `deepseek-v4-pro` | 主：顶级推理 |
| 2 | `deepseek-v3.2` | 同族降级：仍为强推理 |
| 3 | `MiniMax-M2.7` | 异构备选：强编码模型 |
| 4 | `glm-5.1` | 全能兜底：什么都能做 |
| 5 | `opencode/big-pickle` | Zen 免费：最终保底 |

**特殊回退链说明**：

**① 编码产出型回退链的"升级"节点**（`hephaestus` / `sisyphus-junior` / `deep`）：

```
deepseek-v3.2 -> MiniMax-M2.7 -> deepseek-v4-pro -> glm-5.1 -> Zen 免费
    主            同能力备选          ⬆ 升级节点      全能兜底      最终保底
```

第 3 位 `deepseek-v4-pro` 比主模型 `deepseek-v3.2` 更强/更贵。这不是标准降级，而是**成本-可靠性权衡**：

| 策略 | 逻辑 |
|---|---|
| 第 2 位 `MiniMax-M2.7` | 异构备选，不同 provider 提供独立可用性 |
| 第 3 位 `deepseek-v4-pro` | 前两位都不可用 → 宁可贵也要保质量，用顶级模型确保代码不崩 |
| 第 4 位 `glm-5.1` | 回到标准降级：全能模型兜底 |

> 💡 设计假设：代码生成任务的容错成本高（错误代码会导致编译/运行失败、连锁 bug），因此在第 3 位投入更多推理资源是值得的。

**② 视觉链的硬约束保护**（`multimodal-looker` / `visual-engineering`）：

```
glm-5.1 -> qwen3.6-plus -> glm-5 -> Zen 免费
  主         视觉备选       视觉备选   最终保底
```

全程保持视觉能力。无视觉模型已从链中剔除，防止视觉任务静默失败。

---

## 六、国产 vs 国外对照总结

| 角色类型 | 国产最优 | 国外最优 | 差距判断 |
|---|---|---|---|
| 深度推理型 (编排/顾问/评审) | deepseek-v4-pro | Claude Opus 4.7 | 接近，Opus 在结构化输出上微优 |
| 编码产出型 (构建/执行) | deepseek-v3.2 | Claude Sonnet 4.6 | 接近，Sonnet 速度更快 |
| 视觉全能型 (通用/多模态/UI) | glm-5.1 | Gemini 3.1 Pro | 有一定差距，Gemini 视觉能力显著更强 |
| 轻量快速型 (搜索/快改) | qwen3.6-plus | Gemini 3 Flash | 接近，Flash 速度略快 |
