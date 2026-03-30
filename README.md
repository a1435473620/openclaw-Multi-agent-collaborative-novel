# 🤖 OpenClaw 多Agent协作小说生成系统

> 基于 [OpenClaw](https://github.com/openclaw/openclaw) 的多Agent协作架构，实现**全自动小说生成**。

## 🎯 项目愿景

构建一个**全自动小说生成系统**——只需提供世界观设定和总体大纲，AI Agent 团队就能自动完成从大纲拆分、章节创作、润色打磨到质量审核的全流程，持续产出高质量长篇小说。

> **核心问题**：在实际运行中，outline-writer 生成的细纲存在**系统性错误**（时间线倒退、人物年龄错乱、历史人物存殁矛盾、与总大纲定位不符）。单一 reviewer 难以保证审核质量，需要**多人/多轮审核 + 修改循环**。当前方案是 outline-writer 写完5章 → reviewer 审核 → 不通过则修改 → 重新审核，直到通过才进入下一步。未来考虑引入**多个 reviewer 并行审核**，或用不同模型交叉检查，降低单点失察风险。

## ✨ 核心特性

### 🏭 全自动生产流水线

```
总大纲 → outline-writer → reviewer审核 → editor创作 → polisher润色 → humanizer去AI → reviewer终审 → uploader发布
```

一套完整的9步生产流程，人工只需设定世界观和总体方向。

### 👥 多Agent协作架构

| Agent | 角色 | Emoji | 职责 |
|-------|------|-------|------|
| **Main Agent** | 调度者（Orchestrator） | 🎯 | 任务分配、流程控制、异常处理 |
| **outline-writer** | 大纲撰写者 | 📝 | 将总大纲拆分为5章一批的细分大纲 |
| **editor** | 小说编辑 | ✏️ | 基于大纲创作章节正文（≥3000字） |
| **polisher** | 润笔 | 🖋️ | 润色文稿，提升文学性 |
| **reviewer** | 审核员 | 🔍 | 质检文稿，检查逻辑、历史、人物一致性 |
| **uploader** | 发布者 | 📤 | 格式化并发布终稿 |

> **会话管理问题**：在长时间运行中，已完成/超时的子Agent会话会堆积，占用 Gateway 内存并拖慢响应速度。必须在每次收到子Agent完成事件后**立即 kill 清理**，并在每次调度新任务前先检查 `subagents list` 清理 done 状态的会话。详见 [已知问题](#已知问题与经验教训) 章节。

### ⚡ 并行调度引擎

- **3-4个并发任务**同时运行，最大化产出效率
- **10分钟超时机制**——超时自动杀掉重来，不死锁
- **会话自动清理**——完成即销毁，不堆积僵尸进程
- **大纲-正文双线并行**——审核和创作同时进行，不干等

### 📊 质量保障体系

- **大纲审核**：节奏合理性 + 与总大纲一致性 + 历史准确性 + 人物正确性
- **文稿审核**：90分以上 + 无逻辑硬伤（硬伤必须修，哪怕96分）
- **AI痕迹检测**：通过 humanizer-zh 检测24种AI写作特征，评分≥45/50
- **伏笔追踪系统**：前面埋的坑，就算在第一千章也要填

### 🔧 灵活的Agent配置

每个Agent都有独立的：
- `IDENTITY.md` — 身份定义（代号、角色、模型、职责）
- `SOUL.md` — 工作指南（审核要点、输出格式、专业领域）

通过修改配置文件即可调整Agent的行为，无需改代码。

## 🏗️ 架构概览

```
┌─────────────────────────────────────────────────┐
│                  Main Agent                      │
│              (🎯 调度者 / Orchestrator)           │
│                                                  │
│  • 读取 SOUL.md 确认身份                          │
│  • 读取 MEMORY.md 获取长期记忆                     │
│  • 调度子Agent，监控进度                           │
│  • 管理会话生命周期                                │
└─────────┬───────────────────────────┬────────────┘
          │                           │
    ┌─────▼─────┐             ┌───────▼───────┐
    │ 大纲流水线  │             │  正文流水线     │
    │            │             │               │
    │ outline-   │             │   editor      │
    │   writer   │────────────►│    (创作)      │
    │   (📝)     │  细分大纲    │     ✏️        │
    └─────┬─────┘             └───────┬───────┘
          │                           │
    ┌─────▼─────┐             ┌───────▼───────┐
    │  reviewer  │             │  polisher     │
    │   (审核)   │             │   (润色)      │
    │    🔍      │             │    🖋️         │
    └───────────┘             └───────┬───────┘
                                      │
                              ┌───────▼───────┐
                              │  reviewer     │
                              │  (终审)       │
                              │   🔍          │
                              └───────┬───────┘
                                      │
                              ┌───────▼───────┐
                              │  uploader     │
                              │  (发布)       │
                              │   📤          │
                              └───────────────┘
```

## 📁 项目结构

```
.
├── README.md                    # 本文件
├── AGENTS.md                    # 主Agent行为规范（调度规则、记忆管理、心跳）
├── SOUL.md                      # 主Agent人格定义
├── IDENTITY.md                  # 主Agent身份
├── USER.md                      # 用户信息
├── TOOLS.md                     # 工具本地笔记
├── HEARTBEAT.md                 # 心跳检查配置
├── BOOTSTRAP.md                 # 初始化脚本
└── agents/
    ├── editor/
    │   ├── IDENTITY.md          # 编辑身份（代号、职责、模型）
    │   └── SOUL.md              # 编辑工作指南（写作要点、输出格式）
    ├── polisher/
    │   ├── IDENTITY.md          # 润笔身份
    │   └── SOUL.md              # 润笔工作指南（含humanizer-zh）
    ├── reviewer/
    │   ├── IDENTITY.md          # 审核员身份
    │   └── SOUL.md              # 审核员工作指南（含大纲审核规则）
    ├── uploader/
    │   ├── IDENTITY.md          # 发布者身份
    │   └── SOUL.md              # 发布者工作指南
    └── outline-writer/
        ├── IDENTITY.md          # 大纲撰写者身份
        └── SOUL.md              # 大纲撰写者工作指南
```

## 🚀 快速开始

### 前置条件

1. 安装 [OpenClaw](https://github.com/openclaw/openclaw)
2. 配置好至少一个 LLM API（OpenRouter / OpenAI / 其他）
3. 创建对应的 Agent 实例

### 安装步骤

```bash
# 1. 克隆本仓库
git clone https://github.com/a1435473620/openclaw-Multi-agent-collaborative-novel.git

# 2. 复制到 OpenClaw workspace
cp -r agents/* ~/.openclaw/agents/
cp AGENTS.md SOUL.md IDENTITY.md USER.md TOOLS.md HEARTBEAT.md BOOTSTRAP.md ~/.openclaw/workspace/

# 3. 根据你的需求修改配置
# - SOUL.md: 修改小说类型、世界观、创作方向
# - agents/*/SOUL.md: 调整各Agent的审核/创作标准
# - USER.md: 填写你的信息

# 4. 启动 OpenClaw Gateway
openclaw gateway start

# 5. 开始对话，告诉Agent你的小说设定
```

### 自定义你的小说

1. 修改 `SOUL.md` 中的小说设定（类型、时代、主角）
2. 在 `agents/` 下调整各Agent的专业领域
3. 准备好总大纲（可以自己写，也可以让AI协助）
4. 告诉 Main Agent 开始工作

## 📋 完整规则体系

### 子Agent调度规则
- 每个子Agent任务的第一句话必须让它读取身份文件
- 产出必须保存到固定目录，严禁乱放
- 完成后立即清理会话（`subagents kill`），不允许堆积
- 统一模型配置，可按需修改

### 大纲-修改循环
```
outline-writer 写5章大纲 → reviewer审核 → 通过→继续 → 不通过→修改→重新审核
```

### 文稿创作流程
```
editor创作(≥3000字) → polisher润色 → humanizer去AI(≥45分) → reviewer终审(≥90分) → 不通过→回润色
```

### 并行调度原则
- 大纲审核和小说正文创作必须并行
- 10分钟超时规则 — 超时就杀掉重来
- 保持3-4个并发任务最优

## 🔮 未来路线

- [ ] 支持更多小说类型（科幻、言情、悬疑等模板）
- [ ] 添加角色数据库Agent（自动维护人物关系图）
- [ ] Web UI 控制面板（可视化流水线状态）
- [ ] 插件系统（自定义Agent和处理节点）
- [ ] 多语言支持
- [ ] 自动出版/分发到各平台

## 🤝 贡献

欢迎提 Issue 和 PR！如果你用这套架构写出了好小说，也欢迎分享。

## 📄 许可证

MIT License

## 🙏 致谢

- [OpenClaw](https://github.com/openclaw/openclaw) — 提供多Agent运行时框架
- 所有贡献者和使用者

---

> **"一群AI Agent，写出了一本小说。"** 📚🤖

---

## ⚠️ 已知问题与经验教训

### 1. 细纲生成质量问题
outline-writer 生成的细纲存在**系统性错误**，在实际项目中几乎每批都有问题：

| 问题类型 | 严重程度 | 说明 |
|---------|---------|------|
| **时间线倒退/断裂** | 🔴 高 | 最常见，章节之间时间跳跃不合理 |
| **人物年龄错误** | 🔴 高 | 主角年龄应随年份推算，不能固定 |
| **历史人物存殁** | 🔴 高 | 需查证每个历史人物的生卒年 |
| **与总大纲定位不符** | 🔴 高 | 不能提前执行后续卷的内容 |
| **节奏不合理** | ⚠️ 中 | 日常章和高潮章比例失调 |

**当前解决方案**：outline-writer 写5章 → reviewer 审核 → 不通过→修改→重新审核 → 通过才继续

**未来优化方向**：
- 多个 reviewer 并行审核（不同模型交叉检查）
- 建立自动化历史人物时间校验工具
- 总大纲时间线约束硬编码到 outline-writer 的 SOUL.md 中

### 2. 会话堆积问题
子Agent完成后，会话不会自动释放，会堆积在 Gateway 内存中，导致：
- Gateway 响应变慢
- 内存占用持续增长
- `subagents list` 返回大量历史记录

**解决方案**：
```python
# 每次收到子Agent完成事件后，立即执行
subagents(action="kill", target="completed-agent-label")

# 每次调度新任务前，先检查并清理
subagents(action="list")  # 检查所有 done/timeout 状态
subagents(action="kill", target="xxx")  # 逐一清理
```

**规则写入 AGENTS.md**：
> ⚠️ 会话清理铁律 — 每次收到子Agent完成事件后，必须立即用 `subagents kill` 清理该会话，然后立即补上新任务。绝不允许已完成的会话堆积。
