# collab-dev — 多 Agent 协作开发 Skill

> Multi-agent collaborative development skill for [OpenClaw](https://github.com/openclaw/openclaw).  
> 七轮 A/B 双盲测试验证，从真实 bug 中迭代 9 个版本。

## 🎯 这是什么

一个让 4 个 AI Agent 分工协作开发项目的 OpenClaw Skill：

```
Gemini (设计规范 + UI代码)
    ↓
OpenClaw (架构 + 基础设施 + 逻辑骨架)
    ↓
Claude Code (填充业务逻辑)
    ↓
Codex / Claude Code (代码审查)
```

每个 Agent 只做自己最擅长的事：

| Agent | 职责 | 擅长 | 不做 |
|-------|------|------|------|
| **Gemini** | 设计规范 + 表现层代码 | UI/UX、配色、动效、响应式 | 不写业务逻辑 |
| **OpenClaw** | 架构、安全模板、集成 | 数据库、认证、SSE、API 骨架 | 不写 UI 样式 |
| **Claude Code** | 填充 TODO 函数体 | 算法、数据处理、状态管理 | 不改 CSS/HTML 结构 |
| **Codex** | 代码审查 | 找 bug、安全漏洞、性能问题 | 不改代码 |

## 📊 七轮双盲测试数据

每轮都生成「协作版」和「Solo 版」（单 Agent 独立完成），随机分配端口，用户不知道哪个是哪个。

| 轮次 | 项目 | 项目类型 | 用户选择 | Codex 评分 | Claude Code 评分 | 设计一致性 |
|:----:|------|---------|---------|-----------|----------------|:---------:|
| 1 | TaskFlow（任务看板）| 标准工具 | 未盲测 | 39 vs 40 | — | — |
| 2 | PollBox（在线投票）| **视觉型** | **协作** ✅ | 6.0 vs 7.3 | — | 4/10 |
| 3 | ChatRoom（实时聊天）| 标准工具 | Solo | 36 vs 44 | — | 4/10 |
| 4 | ChatRoom v2 | 标准工具 | **协作** ✅ | **42 vs 38** | — | 5/10 |
| 5 | NoteBoard（便签板）| 标准工具 | **协作** ✅ | **36 vs 33** | — | **9/10** |
| 6 | LinkBox（书签收藏）| 标准工具 | **协作** ✅ | 37 vs 39 | 26 vs 26 | 7/10 |
| 7 | Pomodoro（番茄钟）| **视觉型** | **协作** ✅ | **42 vs 31** | **49 vs 35** | 8-9/10 |

### 关键结论

- **用户选协作版 5/6 次**（排除未盲测的第 1 轮）
- **视觉密集型项目**：协作版完胜（Pomodoro 轮三方一致选协作版）
- **标准工具型项目**：协作版小幅领先，但差距不大
- **设计一致性**：从 v1 的 4 分提升到 v9 的 9 分（Phase 1 拆两步是关键突破）
- **Codex Review 是最稳定的价值**：每轮都能发现真实 bug

## 🔄 工作流概览

```
Phase 0: OpenClaw 写 spec.md（需求规格 + 验收标准）
    ↓
Phase 1a: Gemini 写 design.md（设计规范 + CSS 变量 + 命名体系）
Phase 1b: Gemini 写 index.html（纯表现层代码，不含 JS 逻辑）
    ↓
Phase 2: OpenClaw 写 server.js 骨架 + 注入 JS 骨架
         （安全模板 + SSE + 认证 + 逻辑标记 // TODO）
    ↓
Phase 3: Claude Code 填 TODO ← 同时 → Codex 预审骨架
    ↓
Phase 4: OpenClaw 自动化验证（语法 + 设计一致性 + CSS/JS集成 + 安全 + DOM结构 + Spec覆盖率）
    ↓
Phase 5: OpenClaw 自审 + 修复
    ↓
Phase 6: Codex 终审（写 review.md）
    ↓
Phase 7: OpenClaw 根据 review 修复 Critical/High
    ↓
Phase 8: 验收交付
```

**预计耗时**：10-17 分钟

## 🏗️ 架构决策（从踩坑中学到的）

### Phase 0: Spec 写法

| 规则 | 原因 | 来源 |
|------|------|------|
| 不写 UI 风格提示 | 留给 Gemini 差异化，否则两版 UI 趋同 | 第 6 轮 LinkBox |
| 明确数据模型（共享/私有）| 避免"协作语义不一致" | 第 5 轮 NoteBoard |
| 功能标 P0/P1/P2 | 避免"spec 写了但没人实现" | 第 5 轮 NoteBoard |
| 字段命名统一 | 避免 `desc` vs `description` 混用 | 第 6 轮 Claude Code Review |
| 搜索策略二选一 | 避免 server+client 双重搜索 | 第 6 轮 双 Review 共识 |

### Phase 1: Gemini 设计

| 规则 | 原因 | 来源 |
|------|------|------|
| 拆两步（先 design.md 再 HTML）| 设计一致性从 4 分提升到 9 分 | 第 5 轮 NoteBoard |
| 统一 `.hidden` class 显隐 | Gemini CSS 用 `display:none`，JS 用 `.hidden` 对不上 | 第 5 轮 弹窗打不开 bug |
| 移动端不能隐藏 P0 功能区 | 侧边栏被 `display:none` 直接隐藏 | 第 6 轮 Codex Review |
| 清理 markdown 包裹 | Gemini 经常输出 ````html ... ```` | 多轮经验 |

### Phase 2: OpenClaw 骨架

| 规则 | 原因 | 来源 |
|------|------|------|
| bcrypt 必须异步 | `hashSync` 阻塞事件循环 | 第 6 轮 Claude Code Review |
| JWT 密钥支持 env var | 硬编码密钥是致命漏洞 | 第 5、6 轮 Solo 版 |
| SSE 排除当前连接 | 否则当前页双重刷新 | 第 6 轮 双 Review 共识 |
| 空状态放 grid 外 | `innerHTML` 会覆盖 grid 内元素 | 第 6 轮 Codex Review |
| 事件委托替代 onclick | 内联 onclick 拼接不安全 | 第 6 轮 双 Review 共识 |
| DB 路径 `path.join(__dirname, ...)` | 不依赖启动目录 | 第 6 轮 Codex Review |
| 枚举字段白名单 | Solo 版 color 无校验导致 XSS | 第 5 轮 NoteBoard |
| 保存 Gemini 快照 | Phase 5 diff 检查 Claude 有没有偷改表现层 | 第 4 轮 ChatRoom |

### Phase 4: 自动化验证

| 检查项 | 内容 |
|--------|------|
| 4a 代码完整性 | 语法检查 + TODO 残留 = 0 |
| 4b 设计一致性 | design.md 命名 vs 代码命名 diff |
| 4c CSS/JS 集成 | JS 用的 class 在 CSS 有定义 + 无效 CSS 属性 |
| 4d 安全检查 | 枚举白名单 + JWT env var + bcrypt 异步 |
| 4e DOM 结构 | 空状态位置 + 无内联 onclick |
| 4f Spec 覆盖率 | 逐条对照验收标准，P0 全绿 |

## 📈 版本演进

```
v1  Gemini ACP 设计 → Claude 写 → Codex review
    ❌ Gemini ACP 从未成功

v2  Gemini CLI 可执行规范 → 参数内联 Claude prompt
    ⚠️ 设计参数在传递中丢失

v3  OpenClaw 写基础设施 → Claude 填充 → 双重 review
    ⚠️ Claude 不遵守设计规范

v4  🚀 Gemini 直接写 UI 代码 → 突破！
    ✅ 用户首次选协作版

v7  Phase 1 拆两步（先 design.md 再 HTML）
    ✅ 设计一致性 4→9 分

v8  安全模板 + CSS/JS 集成检查
    ✅ 第五轮 Codex Review 驱动

v9  双 Review（Codex + Claude Code）驱动
    ✅ 异步安全 + 移动端铁律 + 搜索策略
```

## ⚠️ Known Issues

| 问题 | 应对 |
|------|------|
| Gemini ACP 不可用 | 一律用 `gemini -p` CLI 模式 |
| Gemini 输出含 markdown 包裹 | Phase 1b 标准步骤用 `sed` 清理 |
| Gemini 输出可能截断 | 分模块多次调用 |
| Claude ACP 可能卡住 | 5min 超时 + 3min 文件未改检测 → OpenClaw 回退自己填 |
| Claude 偷改表现层 | Phase 5 diff 验证 `.gemini-original` 快照 |
| ACP session 计数器不释放 | `maxConcurrentSessions` 建议设为 10 |
| 标准工具型项目 Gemini 价值有限 | 最适合视觉密集型项目 |

## 🎯 适用场景

| 场景 | 推荐度 | 说明 |
|------|:------:|------|
| Dashboard / 数据可视化 | ⭐⭐⭐ | Gemini 设计系统价值最大 |
| 动画交互型应用 | ⭐⭐⭐ | 渐变、SVG 动画、主题切换 |
| 需要设计系统的中大型项目 | ⭐⭐⭐ | design.md 确保全局一致性 |
| 标准 CRUD 工具 | ⭐⭐ | 可用但 Solo 差距不大 |
| 简单单页应用 | ⭐ | 协作开销不值得 |

## 📦 安装

### 方式一：OpenClaw CLI
```bash
openclaw skills install laolin5564/collab-dev
```

### 方式二：手动安装
```bash
mkdir -p ~/.openclaw/skills/collab-dev
curl -o ~/.openclaw/skills/collab-dev/SKILL.md \
  https://raw.githubusercontent.com/laolin5564/collab-dev/master/SKILL.md
```

## 🚀 使用

安装后对 OpenClaw 说以下任意一句即可触发：

- `协作开发一个番茄钟应用`
- `collab dev a dashboard`
- `multi-agent build 一个投票系统`

OpenClaw 会自动按 Phase 0→8 执行完整流程。

## 🔗 相关项目

- [OpenClaw](https://github.com/openclaw/openclaw) — AI Agent 运行时
- [ClawHub](https://clawhub.com) — OpenClaw Skill 市场

## 📄 License

MIT
