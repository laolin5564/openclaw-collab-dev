# collab-dev — 多 Agent 协作开发 Skill

OpenClaw Agent Skill，用于多 Agent 协作开发。Gemini 负责 UI/设计，Claude 负责逻辑编码，Codex 负责代码审查。

## 七轮 A/B 双盲测试验证

| 轮次 | 项目 | 用户体感 | Review 评分 | 类型 |
|------|------|---------|-----------|------|
| 2 | PollBox | 协作 ✅ | Solo 略赢 | 视觉型 |
| 3 | ChatRoom | Solo | Solo 赢 | 标准工具 |
| 4 | ChatRoom v2 | 协作 ✅ | 协作赢 | 标准工具 |
| 5 | NoteBoard | 协作 ✅ | 协作赢 | 标准工具 |
| 6 | LinkBox | 协作 ✅ | Solo 略赢 | 标准工具 |
| 7 | Pomodoro | 协作 ✅ | 协作完胜 | 视觉型 |

**用户选协作版 5/6 次。视觉密集型项目协作版完胜。**

## 核心架构

```
Gemini (设计) → OpenClaw (架构) → Claude (逻辑) → Codex (审查)
```

- **Gemini**: UI/前端/表现层 — 设计规范 + 可见代码
- **Claude**: 编码/逻辑 — 只填 TODO 函数体
- **Codex**: Review/质量审查 — 写报告不改代码
- **OpenClaw**: 架构/集成/验证 — spec + 骨架 + 修复

## 安装

```bash
openclaw skills install laolin5564/collab-dev
```

或手动复制 `SKILL.md` 到 `~/.openclaw/skills/collab-dev/`。

## 使用

对 OpenClaw 说「协作开发」「collab dev」或「multi-agent build」即可触发。

## 最适合的场景

- ✅ 视觉密集型项目（Dashboard、数据可视化、动画交互）
- ✅ 需要设计系统的中大型项目
- ⚠️ 标准工具型项目（效果和 Solo 差异不大）
- ❌ 简单 CRUD（协作开销不值得）

## License

MIT
