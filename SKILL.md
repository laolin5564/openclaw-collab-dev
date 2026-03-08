---
name: collab-dev
description: Multi-agent collaborative development. Gemini handles presentation layer (UI/UX/design), Claude handles logic layer (business code), Codex handles quality review. Use when user says "协作开发", "collab dev", "multi-agent build".
---

# Collaborative Development v9

六轮 A/B 双盲测试验证的通用多 Agent 协作开发流程。

## 核心原则

| Agent | 擅长 | 职责边界 | 调用方式 |
|-------|------|---------|---------|
| **Gemini** | UI / 表现层 / 用户体验 | 设计规范 + 所有用户可见的代码 | `gemini -p` |
| **Claude** | 编码 / 逻辑 / 算法 | 只填充 TODO 标记的函数体 | ACP `sessions_spawn` |
| **Codex** | Review / 质量审查 | 写审查报告，不写代码 | ACP `sessions_spawn` |
| **OpenClaw** | 架构 / 集成 / 验证 | spec + 骨架 + TODO + 验证 + 修复 | 直接执行 |

## 工作流

### Phase 0: 需求规格（OpenClaw）

写 `spec.md`，必须明确：

1. **数据模型**：共享还是私有？
2. **功能优先级**：P0/P1/P2，避免写了但不实现
3. **交互定义**：弹窗用 `.hidden` class toggle
4. **实时通信语义**：SSE 广播范围 + **排除当前连接**
5. **安全要求**：枚举字段白名单
6. **搜索策略**：明确用服务端 OR 客户端过滤（不能两个都写）
7. **字段命名统一**：API 字段名 = DB 列名（禁止 `desc` vs `description` 混用）
8. 技术约束 + 文件结构 + 验收标准

**禁止写 UI 风格提示**（如"类似 Raindrop/Pocket"），留给 Gemini 发挥设计差异化。

### Phase 1a: 设计规范（Gemini）

Gemini 输出 `design.md`。Prompt 必须包含：
```
铁律：
1. .hidden class 控制所有 JS 显隐（CSS: .hidden { display: none !important }）
2. 移动端必须保留所有 P0 功能的入口，不允许 display:none 隐藏整个功能区
3. 移动端用 off-canvas/drawer 替代侧边栏直接隐藏
```

### Phase 1b: 表现层代码（Gemini）

Prompt 铁律：
```
1. 命名必须和 design.md 完全一致
2. 不写业务逻辑
3. 所有 JS 切换显隐用 .hidden class，不在元素样式里写 display: none
4. 移动端功能区用 drawer/off-canvas，不能直接隐藏
```

**输出清洗** → **质量门**（检查 design.md 命名覆盖率）

### Phase 2: 基础设施 + 逻辑骨架（OpenClaw）

#### 安全模板
- 所有枚举字段白名单校验
- 所有用户输入 trim + 长度限制
- JWT：`process.env.JWT_SECRET || crypto.randomBytes(32).toString('hex')`
- **bcrypt 必须异步**：`await bcrypt.hash()` / `await bcrypt.compare()`
- Prepared statements / 参数化查询

#### 数据库模板
- **路径**：`path.join(__dirname, 'xxx.db')`（不依赖启动目录）
- 关联表用复合主键 + `INSERT OR IGNORE`
- 标签查询用批量聚合，避免 N+1

#### SSE/实时通信模板
- 连接用 uuid 作 map key
- 在线人数 = 去重 userId
- **broadcastToUser 必须传 excludeConnId**（排除当前连接，避免双重刷新）
- 心跳 30s + 连接关闭清理

#### DOM 结构规则
- **空状态元素放在 grid 容器外面**（grid.innerHTML 不会覆盖它）
- **禁止内联 onclick 拼接**：用事件委托 `container.addEventListener('click', e => { const btn = e.target.closest('[data-action]'); ... })`

#### 集成 Gemini 代码
- 补 id（不改样式和结构）
- 清理示例数据
- 检查 Gemini CSS 显隐机制，统一为 `.hidden` class
- 注入 JS 骨架

**保存快照**：`cp public/index.html public/index.html.gemini-original`

### Phase 3: Claude 填充 + Codex 预审（并行）

同 v8（含 5min 超时 + 回退）

### Phase 4: 自动化验证（OpenClaw）

#### 4a. 代码完整性
- 语法检查 / TODO 残留 = 0 / 文件完整性

#### 4b. 设计一致性
- design.md 命名 vs 代码命名 diff

#### 4c. CSS/JS 集成检查
```bash
# JS 用的 class 在 CSS 有定义
for cls in hidden loading active; do
  JS=$(grep -c "classList.*'$cls'" public/index.html)
  CSS=$(grep -c "\.$cls" public/index.html)
  [ "$JS" -gt 0 ] && [ "$CSS" -eq 0 ] && echo "⚠️ .$cls 无 CSS 定义"
done
# 无效 CSS 属性（shadow 应为 box-shadow 等）
grep -n "^\s*shadow:" public/index.html
# display:none 在 .hidden 外（媒体查询除外）
grep -n "display: none\|display:none" public/index.html | grep -v "\.hidden" | grep -v "@media"
```

#### 4d. 安全检查
- 枚举字段白名单 / JWT env var / bcrypt 异步（无 Sync）

#### 4e. DOM 结构检查
```bash
# 空状态不在 grid 内（会被 innerHTML 覆盖）
grep -A2 "empty-state" public/index.html  # 应在 grid 外
# 无内联 onclick 拼接（应用事件委托）
grep -c "onclick=" public/index.html  # 目标 = 0（事件绑定区除外）
```

#### 4f. Spec 覆盖率
- 逐条对照验收标准，P0 全绿

### Phase 5-8: 自审 → 修复 → Codex 终审 → 验收（同 v8）

## 六轮验证数据

| 轮 | 项目 | 老林选 | Codex 分 | 设计一致性 | 类型 |
|----|------|--------|---------|-----------|------|
| 1 | TaskFlow | 未盲测 | 39v40 | — | 标准 |
| 2 | PollBox | **协作** ✅ | 6v7.3 | 4 | 视觉型 |
| 3 | ChatRoom | Solo | 36v44 | 4 | 标准 |
| 4 | ChatRoom v2 | **协作** ✅ | 42v38 | 5 | 标准 |
| 5 | NoteBoard | **协作** ✅ | 36v33 | **9** | 标准 |
| 6 | LinkBox | **协作** ✅ | 37v39 | 7 | 标准 |

## v8→v9 改进（来自第六轮双 Review）

| 发现来源 | 问题 | v9 改进 |
|---------|------|---------|
| 两个共识 | SSE 回推当前页导致双重刷新 | Phase 2：broadcastToUser 必须传 excludeConnId |
| Codex | 移动端侧边栏被 display:none 隐藏 | Phase 1a/1b：移动端铁律 + off-canvas |
| Codex | 空状态 DOM 在 grid 内被 innerHTML 覆盖 | Phase 2：空状态放 grid 外 + Phase 4e 检查 |
| Codex | DB 路径依赖启动目录 | Phase 2：`path.join(__dirname, ...)` |
| Claude Code | bcrypt 同步阻塞事件循环 | Phase 2：必须异步 |
| Claude Code | 搜索双重实现（server+client） | Phase 0：明确搜索策略 |
| Claude Code | API 字段名 vs DB 列名不一致 | Phase 0：字段命名统一 |
| 老林 | UI 趋同，Gemini 设计价值没体现 | Phase 0：禁止写 UI 风格提示 |
| 两个共识 | renderTags 内联 onclick 拼接 | Phase 2：事件委托替代 |

## Known Issues

| 问题 | 应对 |
|------|------|
| Gemini ACP 不可用 | 一律 `gemini -p` |
| Gemini 输出含 markdown 包裹 | Phase 1b 标准步骤清理 |
| Claude ACP 卡住 | 5min 超时 + 回退 |
| Claude 偷改表现层 | Phase 5 diff 验证 `.gemini-original` |
| ACP session 计数器不释放 | maxConcurrentSessions 改大（10） |
| 标准工具型项目 Gemini 价值有限 | 适合视觉密集型项目 |
