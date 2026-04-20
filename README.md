# cc-triad-dev

> Claude Code 三叉戟开发流水线——把"一个模型干所有事"换成"每个阶段用最强的模型"。

## 底层逻辑

单一模型在所有阶段都表现不顶尖。这个 plugin 把开发流程拆成四个角色，每个角色交给它最擅长的模型：

| 阶段 | 角色 | 后端模型 | 为什么是它 |
|---|---|---|---|
| 看图 | `kimi-vision` | `kimi-k2.5:cloud` | 原生多模态 |
| 设计 | `architect` | Claude Opus 4.7 | 最强推理做架构 |
| 写码 | `glm-coder` | `glm-5.1:cloud` | SoTA on SWE-Bench Pro |
| 审查 | `codex-reviewer` | `gpt-5.3-codex` | 独立训练分布，第二意见 |

CC 的 sub-agent 壳本身用 haiku/sonnet 做工具人调用，把 opus 省给 architect。

## 架构图

```
          用户需求
             │
             ▼
      [有图像输入?]
         /      \
        yes      no
        │        │
        ▼        │
   kimi-vision   │
   → spec.md     │
        │        │
        └───┬────┘
            ▼
      architect (Opus)
      → plan.md
            │
            ▼
      glm-coder (Sonnet 壳 → GLM-5.1 cloud)
      → 代码落盘
            │
            ▼
      codex-reviewer (Haiku 壳 → gpt-5.3-codex)
      → punch list
            │
            ▼
     [🔴 must-fix?]
        /      \
       yes      no
       │        │
       ▼        ▼
    重跑      完成汇报
    (最多 2 轮)
```

## 前置依赖

- Claude Code CLI
- Codex CLI: `brew install codex` → 登录 + `~/.codex/config.toml` 配置默认模型
- Ollama: `brew install ollama` + `ollama signin`（Cloud 订阅）
- Ollama Cloud 至少 pull 一次 `glm-5.1:cloud` 和 `kimi-k2.5:cloud`

## 安装

### 方式 A：作为 Claude Code plugin 安装（推荐）

```bash
# 在 Claude Code 会话里
/plugin marketplace add nianyi778/cc-triad-dev
/plugin install cc-triad-dev@cc-triad-dev
```

### 方式 B：手动 clone + 软链

```bash
git clone https://github.com/nianyi778/cc-triad-dev ~/personage/cc-triad-dev
ln -sf ~/personage/cc-triad-dev/agents/*.md ~/.claude/agents/
mkdir -p ~/.claude/skills/triad-dev
ln -sf ~/personage/cc-triad-dev/skills/triad-dev/SKILL.md ~/.claude/skills/triad-dev/SKILL.md
```

## 使用

装完新开 Claude Code 会话，直接说人话：

```
# 纯代码功能
帮 <项目> 加个 XX 功能

# 有设计稿
[贴图] 按这个设计稿实现 XX 组件

# 修 bug
这个 bug 帮我修了：<描述>
```

CC 会自动识别任务类型，走三叉戟流程：先让 architect 出 plan，再 glm-coder 实施，最后 codex-reviewer 审查。你只看最终结果。

## 跳过流水线

轻量任务（typo、rename、单行改动）不会触发 skill，CC 直接处理。你也可以显式：

```
直接改：<描述>
```

## 自定义

- 换模型：编辑 `agents/*.md` 的 frontmatter `model:` 字段
- 换 Codex 后端：编辑 `~/.codex/config.toml` 或调用时 `codex exec -m <model>`
- 换 Ollama 模型：改 `agents/glm-coder.md` 和 `agents/kimi-vision.md` 里的模型名

## License

MIT
