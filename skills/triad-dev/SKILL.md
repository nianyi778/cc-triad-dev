---
name: triad-dev
description: Use PROACTIVELY for any non-trivial feature implementation, bug fix, or code change. Orchestrates a three-agent pipeline (architect → glm-coder → codex-reviewer, with optional kimi-vision for image inputs) to produce higher-quality code than any single model. Triggers on "实现"/"加功能"/"做一个"/"implement"/"build"/"add feature"/"修 bug"/"fix bug" when the task is larger than a trivial one-liner.
---

# 三叉戟开发流水线

Conductor = 主会话 CC。你不亲自写代码，你只负责**编排 + 裁决**。

## 判断是否启用

启用条件（任一满足）：
- 用户请求实现新功能、新组件、新接口
- 修改涉及 2+ 文件 或 50+ 行
- 用户提供了设计稿/截图/UI mockup
- 用户明确说"用三叉戟"/"用流水线"

**跳过条件**（降级为直接写）：
- 纯粹的 typo / rename / 单行改动
- 用户明确说"你自己直接改"
- 紧急 hotfix（明确时间压力）

## 流程图

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
    architect (你在 CC 内部担当)
    → plan-<feature>.md
          │
          ▼
    glm-coder (spawn sub-agent)
    → 落盘代码
          │
          ▼
    codex-reviewer (spawn sub-agent)
    → punch list
          │
          ▼
    [有 🔴 must-fix?]
       /      \
      yes      no
      │        │
      ▼        ▼
   glm-coder  完成，汇报
   (带 review)
   最多重试 2 轮
```

## 每步的抓手

### Step 1: 视觉解读（仅当有图）
```
spawn kimi-vision with task: "解读 <图片路径> 产出 spec-<feature>.md"
```

### Step 2: 架构设计（你自己做，走 architect agent 的规范）
- 读相关代码
- 产出 `plan-<feature>.md`（包含目标、接口契约、文件清单、数据流、边界条件、验收标准）

### Step 3: 代码实施
```
spawn glm-coder with task: "读 plan-<feature>.md 和 spec-<feature>.md (如有)，实施"
```

### Step 4: 独立审查
```
spawn codex-reviewer with task: "对当前 diff 做独立 review，基准是 plan-<feature>.md"
```

### Step 5: 裁决
- 看 codex 的 punch list
- 🔴 must-fix 存在 → 带着 review 再跑 glm-coder
- 最多 2 轮 retry，超出则回报用户决定

## 汇报格式

跑完后向用户报告：

```
## 三叉戟执行结果
- 📐 Plan: ./plan-<feature>.md
- 👁️  Spec: ./spec-<feature>.md (如有)
- ✏️  改动文件: [列表]
- 🔍 Review: [全绿 / N 个 must-fix / N 个 nice-to-have]
- 🔄 迭代次数: N
```

## 边界
- 不跳步：即使觉得 plan 很简单也要写 plan.md，下游 sub-agent 靠它工作
- 不代劳：不要自己写代码来"救场"，要么让 glm-coder 重试，要么回报用户
- 不粉饰 review：codex 说有问题就是有问题，原汁原味传递
