# Claude Code 有趣设计合集

> 来源：`github.com/yasasbanukaofficial/claude-code` 源码阅读
> 日期：2026-04-08

---

## 1. 🧰 ToolSearchTool — 工具搜索工具

**问题**：几十个工具全塞进 prompt，token 成本爆炸。

**解法**：工具分"已加载"和"延迟加载"（deferred），用一个搜索工具去发现其他工具。本质是客户端侧 RAG——检索的不是文档，而是工具本身。

**亮点**：
- 关键词评分：精确匹配 10 分、searchHint 4 分、描述 2 分
- MCP 工具前缀匹配（`mcp__server__action` 拆词搜索）
- CamelCase 拆分（`FileWriteTool` → `file write tool`）
- `+` 前缀表示必选项
- `select:ToolA,ToolB` 跳过搜索直接加载
- 结果缓存（deferred 工具集合变化时才失效）

**文件**：`src/tools/ToolSearchTool/ToolSearchTool.ts`

---

## 2. 🌙 autoDream — 自动做梦 / 记忆巩固

**问题**：AI 需要持续整理记忆，不能"写下来就完"。

**解法**：后台子 agent 定期回顾会话 transcript，合并、去重、剪枝记忆文件。命名来自人类睡眠中的记忆巩固（memory consolidation）。

**触发机制（三道门）**：
1. **时间门**：距上次整理 ≥ 24 小时
2. **会话门**：积累了 ≥ 5 个新会话
3. **锁门**：防并发（tryAcquireConsolidationLock）

**执行四阶段**：
1. Orient — 扫描现有记忆目录
2. Gather — 从近期 transcript 搜索值得保留的信号
3. Consolidate — 合并、更新、去重（相对日期 → 绝对日期，删除被推翻的旧事实）
4. Prune — 更新索引文件，控制在 25KB 以内

**TaskType 枚举中唯一以心理学概念命名的类型**：`'dream'`

**文件**：`src/services/autoDream/autoDream.ts`、`consolidationPrompt.ts`、`consolidationLock.ts`

---

## 3. 🐾 Companion Buddy — 虚拟宠物系统

**问题**：……没有问题，纯乐趣。

**解法**：每个用户根据 `hash(userId)` 通过 Mulberry32 种子 PRNG 确定性地抽到一只 ASCII 宠物，渲染在 CLI 输入框旁边。

**物种**：18 种（duck、goose、blob、cat、dragon、octopus、owl、penguin、turtle、snail、ghost、axolotl、capybara、cactus、robot、rabbit、mushroom、chonk）

**稀有度**：common 60%、uncommon 25%、rare 10%、epic 4%、legendary 1%

**属性**：DEBUGGING、PATIENCE、CHAOS、WISDOM、SNARK（稀有度越高下限越高，一个巅峰属性 + 一个拉胯属性）

**帽子**：crown、tophat、propeller、halo、wizard、beanie、tinyduck

**闪亮版**：1% 概率

**精妙架构**：
- 骨骼（bones）从 userId 哈希推导，不存储 → 防刷稀有度
- 灵魂（soul）由 AI 生成名字和个性，存入 config
- 编辑配置改不了稀有度，改不了物种名
- 物种名用 `String.fromCharCode` hex 编码，绕过构建系统的敏感词扫描（一个物种名和模型代号撞车了）

**动画**：每种 3 帧闲置动画（idle sequence 混合休息帧 + 偶尔 fidget + 罕见眨眼），500ms tick，有对话气泡（10 秒渐隐）和抚摸爱心特效。

**文件**：`src/buddy/companion.ts`、`types.ts`、`sprites.ts`、`CompanionSprite.tsx`、`prompt.ts`

---

## 4. 📼 VCR — 录像机测试模式

**问题**：测试依赖真实 API、花钱、不稳定、CI 不可控。

**解法**：录制 API 请求/响应为 fixture 文件（JSON），输入哈希匹配后直接回放，不调用真实 API。

**核心流程**：
- **录制**：`VCR_RECORD=1` 跑测试 → 真实调用 API → 脱水（dehydrate）→ 写入 `fixtures/<hash>.json`
- **回放**：哈希输入 → 找到 fixture → 加水（hydrate）→ 返回缓存结果
- **CI 严格模式**：fixture 缺失直接报错，不允许实时调用

**脱水（Dehydrate）**—— 消除不稳定因素：
```
/Users/alice/project  →  [CWD]
~/.config/claude      →  [CONFIG_HOME]
num_files="42"        →  num_files="[NUM]"
duration_ms="1234"    →  duration_ms="[DURATION]"
cost_usd="0.05"       →  cost_usd="[COST]"
Windows 路径           →  统一正斜杠
```

**加水（Hydrate）**—— 回放时注入当前环境真实值。

**覆盖范围**：
- 普通 API 调用（`withVCR`）
- 流式响应（`withStreamingVCR`）—— 先缓存完整流，再 yield
- Token 计数（`withTokenCountVCR`）—— 额外清理 UUID 和时间戳

**文件**：`src/services/vcr.ts`

---

## 5. ⏱️ slowLogging — 零开销慢操作自动检测

**问题**：需要监控 JSON.stringify、structuredClone 等操作的耗时，但不能给外部用户增加任何开销。

**解法**：用 TC39 `using` 声明 + `[Symbol.dispose]` 做声明式性能监控。一行代码给任意代码块加上自动计时。

**核心用法**：
```typescript
using _ = slowLogging`structuredClone(${value})`
const result = structuredClone(value)
// ← 离开作用域时自动检查耗时，超阈值才报警
```

**零开销技巧**：
- 外部构建：`slowLogging` 返回单例 no-op disposable，零分配零计时
- 内部构建：`AntSlowLogger` 在 `[Symbol.dispose]` 中检查耗时
- `new Error()` 只捕获堆栈引用，V8 惰性求值——只有真慢了才花格式化开销
- 超阈值后 `buildDescription` 才执行昂贵的字符串拼接

**阈值策略**：开发环境 20ms、内部用户 300ms、外部用户 Infinity（不检测）、环境变量可覆盖

**防递归**：模块级 `isLogging` 重入锁，防止 `logForDebugging` → `appendFileSync` → `jsonStringify` → `slowLogging` 死循环

**文件**：`src/utils/slowOperations.ts`

---

## 附：本次未深入但值得关注的其他设计

- **`src/query/tokenBudget.ts`** — Token 预算管理，90% 阈值自动续写 + 收益递减检测
- **`src/services/compact/`** — 上下文压缩（macro compact + micro compact + API 级 microcompact）
- **`src/tools/SyntheticOutputTool/`** — 非交互模式下的结构化输出工具，动态 JSON Schema + AJV 校验 + WeakMap 缓存
- **`src/vim/`** — 完整的 vim 状态机（INSERT/NORMAL 模式、operator-pending、dot-repeat、text objects）
- **`src/services/preventSleep.ts`** — 防 Mac 睡眠（caffeinate 引用计数 + 自愈超时 + SIGKILL 安全退出）
- **`src/utils/forkedAgent.ts`** — 子 agent 派生（共享 prompt cache 的 fork 模式）
- **`src/tools/AgentTool/forkSubagent.ts`** — 隐式 fork（省略 subagent_type 时继承父级完整上下文）
