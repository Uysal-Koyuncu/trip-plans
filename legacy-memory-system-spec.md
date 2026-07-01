# Legacy Codebase Memory System — Design Spec

## 1. Problem Statement

在大型 legacy codebase 上工作时，工程师在 debug、code review、读工单的过程中会积累大量"暗知识"——系统间的隐式契约、不能动的历史遗留、跨模块的隐藏依赖。这些知识：

- 只存在于人脑中，代码和文档里看不出来
- 随时间遗忘，下次碰到同样的坑还会再踩
- AI 助手在新 session 中完全不知道这些知识

## 2. Design Decisions

以下设计决策来自对问题空间的系统性分析：

| 决策 | 选择 | 理由 |
|------|------|------|
| **定位** | Facilitation tool，不是安全网 | 允许漏报，降低系统复杂度；安全保障仍靠 code review + test + CI |
| **知识粒度** | 只做 L3（系统级隐式契约） | L1（行级）和 L2（模块级）用代码注释解决；memory system 专注注释解决不了的跨系统知识 |
| **视图** | 一份结构化原始数据，自动生成人读 + AI 消费两种视图 | 避免维护两套内容 |
| **知识类型** | 设计意图（流程图）+ 屎山知识（knowledge graph） | 分别回答"应该怎么工作"和"实际怎么工作 & 为什么不一样" |
| **采集方式** | 被动采集，从已有工作行为中自动提取 | 主动录入的系统会因为摩擦力过高而死掉 |
| **Triage** | Session 结束时 30 秒确认 | 嵌入已有仪式（Claude Code recap），摩擦力极低 |
| **匹配方式** | RAG 语义匹配（PR diff vs 知识库） | 允许漏报的定位下，不需要 zero-miss 的复杂兜底 |
| **MVP 优先级** | 工作流集成 > 检索 > 存储 | 最大风险是 adoption，先证明能无摩擦嵌入日常工作 |

## 3. System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Knowledge Lifecycle                    │
│                                                          │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐           │
│  │  Capture  │───▶│  Triage  │───▶│  Store   │           │
│  │ (passive) │    │(30s ack) │    │(raw data)│           │
│  └──────────┘    └──────────┘    └──────────┘           │
│       │                               │                  │
│       │                               ▼                  │
│  Signal Detection:              ┌──────────┐            │
│  - 冲突信号 (AI说没问题,人纠正)    │  Render  │            │
│  - 修正信号 (跑起来不对,调整方向)   │ (2 views)│            │
│  - Review线索 (PR comment/工单)   └──────────┘            │
│                                   │       │              │
│                              Human View  AI View         │
│                              (HTML/图)  (structured)     │
│                                                          │
│  ┌──────────┐    ┌──────────┐                           │
│  │  Surface  │◀───│  Match   │                           │
│  │ (PR review│    │  (RAG)   │                           │
│  │  CI gate) │    │          │                           │
│  └──────────┘    └──────────┘                           │
│                                                          │
│  ┌──────────┐                                           │
│  │  Maintain │  AI auto-rebind (简单变更)                 │
│  │           │  Human confirm (困难变更, 作为PR review一部分)│
│  └──────────┘                                           │
└─────────────────────────────────────────────────────────┘
```

## 4. Knowledge Schema (Draft)

```json
{
  "id": "uuid",
  "type": "implicit_contract | hidden_dependency | historical_constraint | design_intent",
  "title": "支付系统依赖订单服务的 original_amount 字段",
  "description": "退款流程通过 OrderService.getOrder() 获取 original_amount，用于计算退款金额。该字段不能被重命名或删除。",
  "systems": ["payment-service", "order-service"],
  "anchors": [
    {
      "repo": "payment-service",
      "path": "src/refund/RefundCalculator.java",
      "symbol": "RefundCalculator.calculateRefund",
      "type": "semantic"
    },
    {
      "repo": "order-service",
      "path": "src/api/OrderController.java",
      "symbol": "OrderController.getOrder",
      "type": "semantic"
    }
  ],
  "source": {
    "type": "claude_session | pr_comment | ticket",
    "ref": "session-abc123 / PR#456 / JIRA-789",
    "captured_at": "2026-07-01T10:00:00Z"
  },
  "confidence": "confirmed",
  "last_verified": "2026-07-01T10:00:00Z",
  "tags": ["退款", "支付", "字段依赖"]
}
```

## 5. MVP Implementation — Claude Code Hook

### 5.1 核心思路

利用 Claude Code 的 `SessionEnd` hook，在每次 session 结束时：

1. 读取本次 session 的完整对话记录（`transcript_path`）
2. 用 LLM 分析对话，提取 L3 知识候选
3. 输出候选列表供用户确认（或自动存储）

### 5.2 Hook 机制详解

Claude Code 的 hook 系统支持三种类型：

| 类型 | 说明 | 适用场景 |
|------|------|----------|
| `command` | 执行 shell 命令，10 分钟超时 | 复杂处理流程（调 CLI、写文件、调 API） |
| `prompt` | 单轮 LLM 调用，30 秒超时 | 轻量语义分析 |
| `agent` | 多轮 agent（实验性） | 需要工具调用的复杂分析 |

**`SessionEnd` hook 收到的输入（stdin JSON）：**

```json
{
  "session_id": "abc123",
  "transcript_path": "/Users/.../.claude/projects/.../transcript.jsonl",
  "cwd": "/current/working/directory",
  "hook_event_name": "SessionEnd",
  "reason": "other"
}
```

### 5.3 方案 A：Prompt Hook（轻量，推荐作为起步）

直接在 hook 配置中用 `type: "prompt"` 做单轮 LLM 分析。

**优点：** 零代码，配置即用，轻量
**缺点：** 30 秒超时，无法处理长对话；无法调用工具读取 transcript 文件

```jsonc
// ~/.claude/settings.json
{
  "hooks": {
    "SessionEnd": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Review this session. If any system-level implicit contracts, hidden cross-module dependencies, or historical constraints were discovered, output them as a JSON array. Each item should have: title, description, systems involved, and why this knowledge matters. If nothing worth remembering, output an empty array []."
          }
        ]
      }
    ]
  }
}
```

> **局限性：** `type: "prompt"` hook 目前可能无法自动读取 transcript 文件。
> 需验证 prompt hook 是否能访问 stdin 中的 transcript_path 并读取其内容。

### 5.4 方案 B：Command Hook + `claude -p`（推荐）

用 shell 脚本读取 transcript，然后调用 `claude -p` 做语义分析。

**优点：** 完全可控，可处理长对话，可做复杂后处理（写文件、调 API、生成 HTML）
**缺点：** 需要写脚本

#### 5.4.1 Hook 配置

```jsonc
// ~/.claude/settings.json
{
  "hooks": {
    "SessionEnd": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/extract-knowledge.sh"
          }
        ]
      }
    ]
  }
}
```

#### 5.4.2 提取脚本

```bash
#!/bin/bash
# ~/.claude/hooks/extract-knowledge.sh

# 1. 读取 hook 输入
INPUT=$(cat)
TRANSCRIPT_PATH=$(echo "$INPUT" | jq -r '.transcript_path')
SESSION_ID=$(echo "$INPUT" | jq -r '.session_id')
CWD=$(echo "$INPUT" | jq -r '.cwd')

# 2. 检查 transcript 文件是否存在
if [ ! -f "$TRANSCRIPT_PATH" ]; then
  exit 0
fi

# 3. 提取对话内容（只保留用户消息和助手消息，控制大小）
CONVERSATION=$(jq -c '
  [.[] | select(.type == "human" or .type == "assistant")]
  | .[-50:]
' "$TRANSCRIPT_PATH" 2>/dev/null)

if [ -z "$CONVERSATION" ] || [ "$CONVERSATION" = "[]" ]; then
  exit 0
fi

# 4. 知识库目录
KNOWLEDGE_DIR="$HOME/.claude/knowledge"
mkdir -p "$KNOWLEDGE_DIR"

# 5. 调用 Claude 做语义分析
RESULT=$(echo "$CONVERSATION" | claude --bare -p \
  "Analyze this Claude Code conversation transcript.

Extract ONLY system-level knowledge (L3) that would be valuable to remember
for future work on this codebase. This includes:

- Implicit contracts between systems/services
- Hidden cross-module dependencies
- Historical constraints ('don't touch X because Y')
- Non-obvious architectural decisions and their reasons

Detection signals:
- User corrected AI's understanding (conflict signal)
- Something didn't work as expected and was fixed mid-session (correction signal)
- Workarounds or gotchas discovered during debugging

Do NOT extract:
- Simple code changes or bug fixes (L1/L2)
- Things obvious from reading the code
- Generic programming knowledge

Output JSON:
{
  \"knowledge_items\": [
    {
      \"title\": \"short descriptive title\",
      \"description\": \"what the knowledge is and why it matters\",
      \"type\": \"implicit_contract | hidden_dependency | historical_constraint | design_intent\",
      \"systems\": [\"system-a\", \"system-b\"],
      \"tags\": [\"relevant\", \"tags\"]
    }
  ]
}

If nothing worth remembering, output: {\"knowledge_items\": []}" \
  --output-format json \
  --max-turns 1 2>/dev/null)

# 6. 解析结果
ITEMS=$(echo "$RESULT" | jq -r '.result' 2>/dev/null | jq '.knowledge_items' 2>/dev/null)

if [ -z "$ITEMS" ] || [ "$ITEMS" = "[]" ] || [ "$ITEMS" = "null" ]; then
  exit 0
fi

# 7. 保存候选知识（待 triage）
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
OUTPUT_FILE="$KNOWLEDGE_DIR/pending_${TIMESTAMP}_${SESSION_ID}.json"

jq -n \
  --arg sid "$SESSION_ID" \
  --arg cwd "$CWD" \
  --arg ts "$(date -Iseconds)" \
  --argjson items "$ITEMS" \
  '{
    session_id: $sid,
    working_directory: $cwd,
    captured_at: $ts,
    status: "pending_triage",
    items: $items
  }' > "$OUTPUT_FILE"

# 8. 输出提示（会显示在 stderr 给用户看）
COUNT=$(echo "$ITEMS" | jq 'length')
echo "Memory system: extracted $COUNT knowledge candidate(s). Review: $OUTPUT_FILE" >&2

exit 0
```

#### 5.4.3 CLI 关键参数

| Flag | 用途 |
|------|------|
| `--bare` | 跳过配置加载，避免递归触发 hook |
| `-p` | 非交互式单次执行 |
| `--output-format json` | 结构化输出，方便 `jq` 解析 |
| `--max-turns 1` | 限制为单轮，防止 agent 循环 |

### 5.5 方案 C：Command Hook + 外部 API（适合已有基础设施）

如果你们公司已经有 LLM API 访问（Anthropic API / Azure OpenAI），可以在脚本中直接调 API 而不是调 `claude` CLI。

```bash
# 替换方案 B 的第 5 步
RESULT=$(curl -s https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "content-type: application/json" \
  -d "{
    \"model\": \"claude-sonnet-5\",
    \"max_tokens\": 2048,
    \"messages\": [{
      \"role\": \"user\",
      \"content\": \"$ESCAPED_CONVERSATION\n\nExtract L3 knowledge...\"
    }]
  }")
```

**优点：** 更快（不需要启动 CLI），可控性更强
**缺点：** 需要管理 API key

## 6. Triage 流程

### 6.1 MVP：手动 Triage

Session 结束后，pending 文件存在 `~/.claude/knowledge/pending_*.json`。
下次开始新 session 时，用一个 `SessionStart` hook 提示用户 review：

```jsonc
// SessionStart hook
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/triage-reminder.sh"
          }
        ]
      }
    ]
  }
}
```

```bash
#!/bin/bash
# ~/.claude/hooks/triage-reminder.sh
PENDING=$(ls ~/.claude/knowledge/pending_*.json 2>/dev/null | wc -l)
if [ "$PENDING" -gt 0 ]; then
  echo "Memory system: $PENDING knowledge item(s) pending triage." >&2
  echo "Run: claude -p 'review pending knowledge' to triage." >&2
fi
exit 0
```

### 6.2 进阶：交互式 Triage（Stop Hook）

用 `Stop` hook（每次 Claude 回复结束时触发）代替 `SessionEnd`，
这样可以在 session 内做 triage，利用当前对话上下文：

```jsonc
{
  "hooks": {
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/check-knowledge-signal.sh"
          }
        ]
      }
    ]
  }
}
```

这个方案可以在检测到知识信号时实时提示，而不是等 session 结束。

## 7. Knowledge Surfacing（检索 & 展示）

### 7.1 AI 消费：CLAUDE.md 集成

最简单的方式 — 把确认过的知识写入项目的 `CLAUDE.md`：

```bash
# 将确认的知识追加到 CLAUDE.md
cat ~/.claude/knowledge/confirmed_*.json | jq -r '
  .items[] | "## \(.title)\n\(.description)\nSystems: \(.systems | join(", "))\n"
' >> /path/to/project/CLAUDE.md
```

每次新 session，Claude 自动读取 CLAUDE.md，获得历史知识。

### 7.2 人类消费：HTML 生成

你现在已经有 HTML 发布流程。在确认知识后，自动生成 HTML：

```bash
# 生成包含 call graph 和流程图的 HTML
cat ~/.claude/knowledge/confirmed_*.json | claude --bare -p \
  "Generate an HTML page with Mermaid diagrams visualizing these knowledge items.
   Group by system. Include a knowledge graph showing relationships.
   Use clean, readable styling." \
  --output-format text > ~/knowledge-site/index.html
```

### 7.3 PR Review 集成（RAG）

当知识库积累到一定量后，在 CI 中加入 RAG 匹配步骤：

```yaml
# .github/workflows/knowledge-check.yml
- name: Check knowledge base
  run: |
    # 获取 PR diff
    git diff origin/main...HEAD > /tmp/diff.txt
    # 用 embedding 匹配知识库
    python scripts/match_knowledge.py /tmp/diff.txt ~/.claude/knowledge/confirmed_*.json
```

这一步是后期优化，MVP 阶段不需要。

## 8. Knowledge Maintenance

### 8.1 自动维护（简单变更）

当文件被重构时，AI 自动更新 anchor 信息：

- 函数 rename → 更新 `anchors[].symbol`
- 文件移动 → 更新 `anchors[].path`
- 行号变化 → 不跟踪行号（L3 知识不绑定行号）

因为我们只做 L3，anchor 是语义级别的（系统名 + 符号名），大部分简单重构不会破坏 anchor。

### 8.2 人工维护（困难变更）

作为 PR review 的一部分。当 RAG 匹配命中一条知识，reviewer 需要确认：

- 这条知识还有效吗？
- 需要更新描述吗？
- 是否应该标记为过时？

## 9. Roadmap

| Phase | 内容 | 目标 |
|-------|------|------|
| **P0 — MVP** | SessionEnd hook + `claude -p` 提取 + 存 JSON 文件 + SessionStart 提醒 triage | 证明采集循环能跑起来 |
| **P1 — Triage** | 交互式 triage UI（可以是 CLI 也可以是简单 web page） | 降低 triage 摩擦力 |
| **P2 — CLAUDE.md 集成** | 确认的知识自动写入 CLAUDE.md | AI 在新 session 中自动获得历史知识 |
| **P3 — HTML 可视化** | 自动生成 knowledge graph + 流程图 HTML | 人类可浏览的知识视图 |
| **P4 — PR 集成** | RAG 匹配 + CI gate | 在 PR review 时 surface 相关知识 |
| **P5 — 多源采集** | 从 PR comment、工单自动提取（webhook / API） | 扩大采集覆盖面 |

## 10. Open Questions

1. **Transcript JSONL 格式** — 需要验证实际的 JSONL schema，确认消息类型字段名（`human`/`assistant` vs `user_message`/`assistant_message`）
2. **Prompt hook 能力边界** — `type: "prompt"` hook 是否能访问 transcript 文件？如果能，方案 A 更简单
3. **知识去重** — 多次 session 可能提取出相同的知识，需要去重策略
4. **团队共享** — 知识存在本地 `~/.claude/knowledge/` 只有自己能用，团队共享需要集中存储（git repo? database?）
5. **知识过期** — `last_verified` 超过多久未验证的知识应该标记为 stale？
