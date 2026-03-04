# Token 管理：输入控制 + 积累清理

> *Context 是有限资源。聪明的 Agent 知道何时加载、何时遗忘。*

## 问题：Context 膨胀的两个来源

```
Context Window
     │
     ├── 输入阶段：预加载不必要的内容
     │     │
     │     └── 传统做法：把所有技能塞进 system prompt
     │         10 个技能 × 2000 tokens = 20,000 tokens (始终占用)
     │
     └── 积累阶段：历史消息持续增长
           │
           └── 每轮添加 tool_results
               读取 1000 行文件 = ~4000 tokens
               30 个文件 + 20 条命令 = 100,000+ tokens
```

没有管理机制，Agent 会：
- 在不使用的技能上浪费 tokens (输入膨胀)
- 任务中途触及 context 上限 (积累膨胀)
- 无法在大型代码库上工作

---

## 双向解决方案

```
┌─────────────────────────────────────────────────────────────────────┐
│                         TOKEN 管理                                    │
├─────────────────────────────┬───────────────────────────────────────┤
│      输入控制 (s05)          │       积累清理 (s06)                   │
├─────────────────────────────┼───────────────────────────────────────┤
│     技能懒加载               │      三层压缩管道                      │
│                             │                                        │
│  ┌───────────────────────┐  │  ┌─────────────────────────────────┐  │
│  │ System Prompt         │  │  │ Layer 1: micro_compact          │  │
│  │ ├─ skill A: desc      │  │  │   (每轮, 静默)                  │  │
│  │ ├─ skill B: desc      │  │  │   tool_result → 占位符          │  │
│  │ └─ skill C: desc      │  │  └─────────────────────────────────┘  │
│  │     ~100 tokens/skill │  │                    │                  │
│  └───────────┬───────────┘  │                    ▼                  │
│              │              │  ┌─────────────────────────────────┐  │
│              │ 按需加载      │  │ Layer 2: auto_compact           │  │
│              ▼              │  │   (tokens > 阈值)               │  │
│  ┌───────────────────────┐  │  │   全部 → summary                │  │
│  │ load_skill(A)         │  │  └─────────────────────────────────┘  │
│  │ ┌───────────────────┐ │  │                    │                  │
│  │ │ Full skill body   │ │  │                    ▼                  │
│  │ │ Step 1: ...       │ │  │  ┌─────────────────────────────────┐  │
│  │ │ Step 2: ...       │ │  │  │ Layer 3: compact tool           │  │
│  │ │ ~2000 tokens      │ │  │  │   (模型手动触发)                │  │
│  │ └───────────────────┘ │  │  └─────────────────────────────────┘  │
│  └───────────────────────┘  │                                        │
├─────────────────────────────┼───────────────────────────────────────┤
│  Motto:                     │  Motto:                                │
│  "用到什么知识,              │  "上下文总会满,                        │
│   临时加载什么知识"          │   要有办法腾地方"                       │
└─────────────────────────────┴───────────────────────────────────────┘
```

---

## 第一部分：技能懒加载 (s05)

### 问题
传统方法预先将所有技能加载到 system prompt。大多数技能与当前任务无关。

### 架构

```
skills/
  pdf/SKILL.md          # ---\n name: pdf\n description: Process PDFs\n ---\n ...
  code-review/SKILL.md  # ---\n name: code-review\n description: Review code\n ---\n ...
  git/SKILL.md
```

**Layer 1 (低成本)** — 技能元数据注入 system prompt：
```
System Prompt:
+--------------------------------------+
| You are a coding agent.              |
| Skills available:                    |
|   - pdf: Process PDF files           |  ~100 tokens
|   - code-review: Review code         |
|   - git: Git workflow helpers        |
+--------------------------------------+
```

**Layer 2 (按需)** — 通过 tool_result 返回完整技能体：
```
When model calls load_skill("pdf"):
+--------------------------------------+
| tool_result:                         |
| <skill name="pdf">                   |
|   Full PDF processing instructions   |  ~2000 tokens
|   Step 1: Extract text...            |
|   Step 2: Chunk into sections...     |
| </skill>                             |
+--------------------------------------+
```

### 实现

```python
class SkillLoader:
    def __init__(self, skills_dir: Path):
        self.skills = {}
        for f in sorted(skills_dir.rglob("SKILL.md")):
            text = f.read_text()
            meta, body = self._parse_frontmatter(text)
            name = meta.get("name", f.parent.name)
            self.skills[name] = {"meta": meta, "body": body}
    
    def get_descriptions(self) -> str:
        """Layer 1: 为 system prompt 提供简短描述。"""
        lines = []
        for name, skill in self.skills.items():
            desc = skill["meta"].get("description", "No description")
            lines.append(f"  - {name}: {desc}")
        return "\n".join(lines)
    
    def get_content(self, name: str) -> str:
        """Layer 2: 在 tool_result 中返回完整内容。"""
        skill = self.skills.get(name)
        if not skill:
            return f"Error: Unknown skill '{name}'."
        return f"<skill name=\"{name}\">\n{skill['body']}\n</skill>"

# Layer 1 注入
SYSTEM = f"""You are a coding agent.
Skills available:
{SKILL_LOADER.get_descriptions()}"""

# Layer 2 工具
TOOL_HANDLERS = {
    "load_skill": lambda **kw: SKILL_LOADER.get_content(kw["name"]),
}
```

---

## 第二部分：三层压缩 (s06)

### 问题
Context 随每轮对话积累。Tool results 无限增长。

### 压缩管道

```
每轮对话:
    Tool Result
         │
         ▼
┌─────────────────────────────────────────┐
│ Layer 1: micro_compact                  │
│   • 静默执行, 每轮运行                   │
│   • 替换旧的 tool_results (>3 轮)       │
│   • "[Previous: used {tool_name}]"      │
└─────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│ 检查: tokens > THRESHOLD (50k)?         │
└─────────────────────────────────────────┘
         │                    │
        NO                   YES
         │                    │
         ▼                    ▼
      continue    ┌─────────────────────────────────────────┐
                   │ Layer 2: auto_compact                  │
                   │   • 保存完整 transcript 到磁盘         │
                   │   • LLM 生成 summary                   │
                   │   • 用 summary 替换所有消息            │
                   └─────────────────────────────────────────┘
                              │
                              ▼
                   ┌─────────────────────────────────────────┐
                   │ Layer 3: compact tool                   │
                   │   • 模型手动调用 compact()              │
                   │   • 触发同样的摘要机制                  │
                   └─────────────────────────────────────────┘
```

### 实现

**Layer 1: micro_compact** — 静默, 每轮执行：
```python
KEEP_RECENT = 3

def micro_compact(messages: list) -> list:
    """将旧的 tool results 替换为占位符。"""
    tool_result_indices = [
        i for i, msg in enumerate(messages) 
        if msg.get("role") == "tool"
    ]
    
    if len(tool_result_indices) <= KEEP_RECENT:
        return messages
    
    # 从 assistant 消息构建 tool_name 查找表
    tool_name_map = {}
    for msg in messages:
        if msg.get("role") == "assistant" and msg.get("tool_calls"):
            for tc in msg["tool_calls"]:
                tool_name_map[tc["id"]] = tc["function"]["name"]
    
    # 清除旧结果
    for idx in tool_result_indices[:-KEEP_RECENT]:
        msg = messages[idx]
        if len(msg.get("content", "")) > 100:
            tool_name = tool_name_map.get(msg.get("tool_call_id"), "unknown")
            msg["content"] = f"[Previous: used {tool_name}]"
    
    return messages
```

**Layer 2: auto_compact** — 阈值触发：
```python
THRESHOLD = 50000
TRANSCRIPT_DIR = Path(".transcripts")

def auto_compact(messages: list) -> list:
    """保存 transcript, 生成摘要, 替换消息。"""
    # 1. 保存完整历史到磁盘 (可恢复)
    TRANSCRIPT_DIR.mkdir(exist_ok=True)
    transcript_path = TRANSCRIPT_DIR / f"transcript_{int(time.time())}.jsonl"
    with open(transcript_path, "w") as f:
        for msg in messages:
            f.write(json.dumps(msg, default=str) + "\n")
    
    # 2. LLM 生成摘要
    response = client.chat.completions.create(
        model=MODEL,
        messages=[{
            "role": "user", 
            "content": "Summarize this conversation for continuity.\n"
                       "Include: 1) What was accomplished, "
                       "2) Current state, 3) Key decisions.\n\n"
                       + json.dumps(messages, default=str)[:80000]
        }],
        max_tokens=2000,
    )
    summary = response.choices[0].message.content
    
    # 3. 替换为压缩后的消息
    return [
        {"role": "user", "content": f"[Compressed. Transcript: {transcript_path}]\n\n{summary}"},
        {"role": "assistant", "content": "Understood. I have the context. Continuing."},
    ]
```

**Layer 3: compact tool** — 模型触发：
```python
TOOLS = [
    # ... 其他工具 ...
    {
        "type": "function",
        "function": {
            "name": "compact",
            "description": "Trigger manual conversation compression.",
            "parameters": {
                "type": "object",
                "properties": {
                    "focus": {"type": "string", "description": "What to preserve"}
                }
            }
        }
    }
]
```

**整合后的循环：**
```python
def agent_loop(messages: list):
    while True:
        # Layer 1: 静默 micro-compact
        micro_compact(messages)
        
        # Layer 2: 阈值 auto-compact
        if estimate_tokens(messages) > THRESHOLD:
            print("[auto_compact triggered]")
            messages[:] = auto_compact(messages)
        
        response = client.chat.completions.create(
            model=MODEL, 
            messages=[{"role": "system", "content": SYSTEM}] + messages,
            tools=TOOLS,
        )
        
        # ... 执行工具 ...
        
        # Layer 3: 手动 compact
        if any(tc.function.name == "compact" for tc in tool_calls):
            print("[manual compact]")
            messages[:] = auto_compact(messages)
```

---

## 协同时间线

```
时间线 ─────────────────────────────────────────────────────────────►

Turn 1: User: "Process this PDF"
        │
        ├─► s05: System prompt 包含 "pdf: Process PDF files..." (~100 tokens)
        │
        └─► Agent 调用 load_skill("pdf")
                  ↓
            s05 返回完整技能体 (Layer 2, 按需)

Turn 2-10: Agent 执行 PDF 处理
        │
        └─► s06 Layer 1: 每轮, micro_compact 静默运行
                  "read_file result (50000 chars)" → "[Previous: used read_file]"

Turn 11: Context 接近 50k tokens
        │
        └─► s06 Layer 2: auto_compact 触发
                  1. 保存完整历史到 .transcripts/transcript_xxx.jsonl
                  2. LLM 生成 summary
                  3. messages[:] = [summary pair]

Turn N: Agent 决定需要压缩
        │
        └─► s06 Layer 3: 调用 compact("focus on auth logic")
                  → 立即触发 auto_compact
```

---

## Token 经济学

### 场景：10 个技能, 每个 2000 tokens

| 方法 | System Prompt | 按需加载 | 浪费总计 |
|------|---------------|----------|----------|
| 传统做法 | 20,000 tokens | 0 | 18,000 tokens (90% 未使用) |
| s05 + s06 | 1,000 tokens | 2,000 × 实际使用数 | 极小 |

**节省：~95% 的 context 用于实际任务**

### 场景：长会话

| 阶段 | 无 s06 | 有 s06 |
|------|--------|--------|
| Turn 1-10 | 5,000 tokens | 5,000 tokens |
| Turn 11-20 | 25,000 tokens | 8,000 tokens (micro_compact) |
| Turn 21-30 | 60,000 tokens (触及上限) | 15,000 tokens (auto_compact) |
| Turn 31+ | 无法继续 | 无限继续 |

---

## 设计哲学对比

| 维度 | s05 懒加载 | s06 压缩 |
|------|-----------|----------|
| **目标** | 输入大小 | 积累大小 |
| **时机** | 使用前 | 积累后 |
| **粒度** | 技能级别 | 消息级别 |
| **数据损失** | 无 (按需获取完整内容) | 部分 (压缩为 summary) |
| **可逆性** | 完全可逆 | 原始数据存盘, 可追溯 |
| **用户控制** | 隐式 (模型选择) | 显式 (compact 工具) |

---

## 核心洞见

1. **输入 vs 积累**
   - s05 从源头阻止浪费 (不加载不需要的内容)
   - s06 在事后管理增长 (压缩不再需要的内容)

2. **渐进式激进**
   - Layer 1: 静默, 有损但极小 (占位符)
   - Layer 2: 可见, 有损但可恢复 (summary + 磁盘)
   - Layer 3: 用户控制, 同 Layer 2

3. **没有任何内容真正丢失**
   - 技能: 随时可通过 `load_skill()` 获取
   - 历史: 始终保存在 `.transcripts/` 磁盘上

4. **循环保持简单**
   - 两种机制都不改变 agent loop
   - 它们嵌入现有架构：
     - s05: System prompt + 一个工具
     - s06: 预调用 hook + 一个工具

---

## 总结

| 问题 | 解决方案 | Session |
|------|----------|---------|
| 技能预加载浪费 context | 懒加载: 元数据 + 按需获取 | s05 |
| 历史消息无限增长 | 三层压缩: 渐进 + 可恢复 | s06 |

**核心原则：**
> Context 是有限资源。聪明的 Agent 策略性地加载、策略性地遗忘。

---

## 参考资料

- [s05: Skill Loading](../zh/s05-skill-loading.md)
- [s06: Context Compact](../zh/s06-context-compact.md)
- [agents/s05_skill_loading.py](../../agents/s05_skill_loading.py)
- [agents/s06_context_compact.py](../../agents/s06_context_compact.py)