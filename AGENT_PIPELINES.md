# Agent Pipelines Documentation - Codex Project

## Содержание

1. [Введение](#введение)
2. [Main Task Execution Pipeline](#1-main-task-execution-pipeline)
3. [Code Review Pipeline](#2-code-review-pipeline)
4. [Context Compaction Pipeline](#3-context-compaction-pipeline)
5. [Sandbox Assessment Pipeline](#4-sandbox-assessment-pipeline)
6. [Issue Triage Pipeline](#5-issue-triage-pipeline)
7. [Planning Pipeline](#6-planning-pipeline)
8. [Tool Execution Pipeline](#7-tool-execution-pipeline)

---

## Введение

Этот документ описывает все основные пайплайны (pipelines) работы AI агента в проекте Codex. Для каждого пайплайна представлены:
- ASCII диаграммы потока выполнения
- Используемые промпты на каждом этапе
- Структура данных, передаваемых между этапами
- Ссылки на исходный код

**Дата создания**: 2025-11-11
**Версия**: Codex AI Agent

---

## 1. Main Task Execution Pipeline

### Описание

Основной пайплайн выполнения пользовательских задач. Обрабатывает ввод пользователя, выполняет инструменты и генерирует ответы.

### Местоположение в коде

- **Основной файл**: `codex-rs/core/src/codex.rs:1734`
- **Event loop**: `codex-rs/core/src/codex.rs:1820-2100`
- **Промпт**: `codex-rs/core/prompt.md` (BASE_INSTRUCTIONS)

### ASCII Диаграмма

```
┌──────────────────────┐
│  User Input          │
│  "Fix the bug in..." │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  1. Load Context                         │
│  - AGENTS.md files                       │
│  - Working directory                     │
│  - Previous messages                     │
│  File: codex.rs:1820                     │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  2. Prepare System Prompt                │
│  Prompt: BASE_INSTRUCTIONS              │
│  or GPT_5_CODEX_INSTRUCTIONS             │
│  File: model_family.rs:15                │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  3. Send to LLM                          │
│  - Anthropic Claude / OpenAI GPT         │
│  - Stream response                       │
│  File: codex.rs:1900                     │
└──────────┬───────────────────────────────┘
           │
           ▼
      ┌────┴────┐
      │ Response│
      │  Type?  │
      └────┬────┘
           │
    ┌──────┼──────┐
    │      │      │
    ▼      ▼      ▼
┌───────┐ ┌──────┐ ┌─────────┐
│ Text  │ │ Tool │ │Thinking │
│Output │ │Call  │ │ Block   │
└───────┘ └──┬───┘ └─────────┘
             │
             ▼
┌──────────────────────────────────────────┐
│  4. Tool Execution                       │
│  - Shell commands                        │
│  - apply_patch                           │
│  - File operations                       │
│  File: orchestrator.rs:33                │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  5. Approval Check (if needed)           │
│  - Sandbox assessment                    │
│  - User confirmation                     │
│  File: approval.rs:45                    │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  6. Return Tool Results                  │
│  File: codex.rs:1950                     │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  7. Loop: Send Results to LLM            │
│  - Continue until done                   │
│  File: codex.rs:1820 (loop)              │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  8. Final Response                       │
│  - User message                          │
│  - Plan updates                          │
└──────────────────────────────────────────┘
```

### Используемые промпты

1. **BASE_INSTRUCTIONS** (`codex-rs/core/prompt.md`)
   - Основной системный промпт для всех моделей
   - Определяет personality, planning, tool usage
   - 310 строк инструкций

2. **GPT_5_CODEX_INSTRUCTIONS** (`codex-rs/core/gpt_5_codex_prompt.md`)
   - Специальный промпт для GPT-5
   - Дополнительные ограничения и правила
   - 107 строк инструкций

### Структура данных

#### Input (от пользователя)
```rust
UserMessage {
    content: String,          // "Fix the bug in app.rs"
    role: "user",
    timestamp: DateTime,
}
```

#### Context передаваемый в LLM
```rust
ConversationContext {
    system_prompt: String,           // BASE_INSTRUCTIONS
    agents_md: Vec<AgentsMdContent>, // AGENTS.md files
    working_directory: PathBuf,
    messages: Vec<Message>,          // История диалога
    tools: Vec<ToolDefinition>,      // Доступные инструменты
}
```

#### Tool Call (от LLM)
```rust
ToolCall {
    name: String,                 // "shell", "apply_patch"
    arguments: HashMap<String, Value>,
    id: String,
}
```

#### Tool Result (возврат агенту)
```rust
ToolResult {
    tool_call_id: String,
    output: String,              // stdout/stderr
    error: Option<String>,
}
```

### Обработка ошибок

- **Token limit exceeded** → Триггерит Context Compaction Pipeline
- **Tool execution failure** → Возвращает ошибку в LLM для retry
- **Sandbox violation** → Триггерит Sandbox Assessment Pipeline
- **Network errors** → Exponential backoff retry (2s, 4s, 8s, 16s)

### Точки интеграции

- **Planning**: При наличии `update_plan` tool call → Planning Pipeline
- **Review**: При команде `/review` → Code Review Pipeline
- **Compact**: При превышении token limit → Context Compaction Pipeline

---

## 2. Code Review Pipeline

### Описание

Пайплайн для автоматического code review. Анализирует изменения кода и выявляет потенциальные баги и проблемы.

### Местоположение в коде

- **Основной файл**: `codex-rs/core/src/tasks/review.rs:35`
- **Промпт**: `codex-rs/core/review_prompt.md` (REVIEW_PROMPT)
- **Обработка результатов**: `codex-rs/core/src/tasks/review.rs:200`

### ASCII Диаграмма

```
┌──────────────────────┐
│ User Input:          │
│ "/review" or         │
│ "review this PR"     │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  1. Collect Code Changes                 │
│  - git diff                              │
│  - Patch files                           │
│  File: review.rs:50                      │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  2. Launch Sub-Agent                     │
│  - Separate LLM session                  │
│  - Isolated context                      │
│  File: review.rs:85                      │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  3. Send Review Prompt                   │
│  Prompt: REVIEW_PROMPT                  │
│  + Code diff                             │
│  + Repo context                          │
│  File: review_prompt.md                  │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  4. LLM Analyzes Code                    │
│  - Bug detection                         │
│  - Security checks                       │
│  - Performance issues                    │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  5. Generate JSON Findings               │
│  {                                       │
│    "findings": [                         │
│      {                                   │
│        "title": "[P1] Bug...",           │
│        "body": "description",            │
│        "confidence_score": 0.9,          │
│        "priority": 1,                    │
│        "code_location": {...}            │
│      }                                   │
│    ],                                    │
│    "overall_correctness": "..."          │
│  }                                       │
│  File: review.rs:150                     │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  6. Parse JSON Results                   │
│  File: review.rs:200                     │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  7. Display Results to User              │
│  - Grouped by priority (P0-P3)           │
│  - With file locations                   │
│  - Overall verdict                       │
└──────────────────────────────────────────┘
```

### Используемые промпты

**REVIEW_PROMPT** (`codex-rs/core/review_prompt.md`):
- Критерии для определения бага
- Приоритизация (P0-P3)
- Правила для комментариев
- JSON схема вывода

### Структура данных

#### Input
```rust
ReviewRequest {
    diff: String,              // git diff output
    base_branch: String,       // "main"
    head_branch: String,       // "feature-branch"
    repo_context: RepoInfo,
}
```

#### Output (JSON от LLM)
```json
{
  "findings": [
    {
      "title": "[P1] Un-padding slices along wrong tensor dimensions",
      "body": "The unpacking operation...",
      "confidence_score": 0.95,
      "priority": 1,
      "code_location": {
        "absolute_file_path": "/path/to/file.py",
        "line_range": {"start": 42, "end": 45}
      }
    }
  ],
  "overall_correctness": "patch is incorrect",
  "overall_explanation": "Contains a P1 bug...",
  "overall_confidence_score": 0.90
}
```

#### Parsed Result
```rust
ReviewResult {
    findings: Vec<Finding>,
    overall_correctness: CorrectnessVerdict,
    overall_explanation: String,
    confidence: f64,
}
```

### Приоритеты

- **P0**: Критический, блокирует релиз
- **P1**: Срочный, должен быть исправлен в следующем цикле
- **P2**: Нормальный, будет исправлен со временем
- **P3**: Низкий, nice to have

---

## 3. Context Compaction Pipeline

### Описание

Автоматическое сжатие истории диалога при превышении token limit. Создает checkpoint summary для передачи следующей сессии.

### Местоположение в коде

- **Основной файл**: `codex-rs/core/src/compact.rs:53`
- **Промпт**: `codex-rs/core/templates/compact/prompt.md` (SUMMARIZATION_PROMPT)
- **Trigger**: Автоматически при превышении лимита токенов

### ASCII Диаграмма

```
┌──────────────────────────────────────────┐
│  Token Limit Check                       │
│  - Current: 85000 tokens                 │
│  - Limit: 100000 tokens                  │
│  - Threshold exceeded: 85%               │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  1. Trigger Compaction                   │
│  File: codex.rs:1850                     │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  2. Collect Current State                │
│  - Messages history                      │
│  - Current plan                          │
│  - Important decisions                   │
│  File: compact.rs:65                     │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  3. Send Summarization Prompt            │
│  Prompt: SUMMARIZATION_PROMPT           │
│  "Create handoff summary..."             │
│  File: compact.rs:80                     │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  4. LLM Creates Summary                  │
│  - Current progress                      │
│  - Key decisions                         │
│  - Context/constraints                   │
│  - Remaining tasks                       │
│  - Critical data                         │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  5. Replace Old Messages                 │
│  Old: 150 messages (80k tokens)          │
│  New: 1 summary message (5k tokens)      │
│  File: compact.rs:120                    │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  6. Continue Main Pipeline               │
│  With reduced context                    │
└──────────────────────────────────────────┘
```

### Используемые промпты

**SUMMARIZATION_PROMPT** (`codex-rs/core/templates/compact/prompt.md`):
```markdown
You are performing a CONTEXT CHECKPOINT COMPACTION.
Create a handoff summary for another LLM that will resume the task.

Include:
- Current progress and key decisions made
- Important context, constraints, or user preferences
- What remains to be done (clear next steps)
- Any critical data, examples, or references needed to continue
```

### Структура данных

#### Input
```rust
CompactionRequest {
    messages: Vec<Message>,     // История диалога
    current_plan: Option<Plan>,
    token_count: usize,
}
```

#### LLM Summary Output
```text
## Current Progress
- Implemented user authentication with JWT
- Fixed 3 bugs in payment processing
- Added unit tests (85% coverage)

## Key Decisions
- Using PostgreSQL instead of MongoDB
- REST API over GraphQL for simplicity

## Remaining Tasks
- Deploy to staging environment
- Run integration tests
- Update API documentation

## Critical Context
- Database schema: users table has email_verified column
- API endpoint: POST /api/auth/login returns {token, user}
```

#### Result
```rust
CompactedContext {
    summary_message: Message,   // Сжатое резюме
    tokens_saved: usize,        // 75000
    original_count: usize,      // 150 messages
}
```

### Когда срабатывает

- При достижении 85% от token limit модели
- Перед критическими операциями (code review, planning)
- По запросу пользователя (`/compact` команда)

---

## 4. Sandbox Assessment Pipeline

### Описание
Оценка безопасности shell команд, заблокированных sandbox'ом.

### Местоположение
- **Файл**: `codex-rs/core/src/sandboxing/assessment.rs:41`
- **Промпт**: `codex-rs/core/templates/sandboxing/assessment_prompt.md`

### ASCII Диаграмма
```
┌──────────────────────┐
│ Command Blocked by   │
│ Sandbox              │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  1. Collect Command Metadata             │
│  - platform, sandbox_policy              │
│  - command_argv, working_directory       │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  2. Render Askama Template               │
│  Prompt: Sandbox Assessment Template    │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  3. LLM Assesses Risk                    │
│  Returns JSON:                           │
│  {                                       │
│    "description": "Installs...",         │
│    "risk_level": "medium"                │
│  }                                       │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  4. Present to User for Approval         │
│  Description + Risk Level                │
└──────────────────────────────────────────┘
```

### Risk Levels
- **low**: read-only, listing files, configuration
- **medium**: modifying files, installing dependencies
- **high**: deleting data, exfiltration, privilege escalation

---

## 5. Planning Pipeline

### Описание
Создание и обновление планов выполнения задач с помощью `update_plan` tool.

### Местоположение
- **Файл**: `codex-rs/core/src/tools/handlers/plan.rs:100`

### ASCII Диаграмма
```
┌──────────────────────┐
│ LLM calls            │
│ update_plan tool     │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  1. Parse Tool Arguments                 │
│  {                                       │
│    "steps": [                            │
│      {                                   │
│        "content": "Add CLI entry",       │
│        "activeForm": "Adding CLI...",    │
│        "status": "in_progress"           │
│      }                                   │
│    ]                                     │
│  }                                       │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  2. Validate Plan                        │
│  - Exactly one in_progress               │
│  - Valid statuses                        │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  3. Update Internal State                │
│  Store plan in session                   │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  4. Render to UI                         │
│  Display progress                        │
└──────────────────────────────────────────┘
```

### Statuses
- **pending**: Not started
- **in_progress**: Currently working on (exactly one)
- **completed**: Finished

---

## 6. Tool Execution Pipeline

### Описание
Маршрутизация и выполнение различных инструментов агента.

### Местоположение
- **Файл**: `codex-rs/core/src/tools/orchestrator.rs:33`

### ASCII Диаграмма
```
┌──────────────────────┐
│ LLM Tool Call        │
│ {name: "shell"}      │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  1. ToolRouter.route()                   │
│  Match tool name to handler              │
└──────────┬───────────────────────────────┘
           │
        ┌──┴────┬──────┬──────┬──────┐
        │       │      │      │      │
        ▼       ▼      ▼      ▼      ▼
    ┌─────┐ ┌─────┐ ┌────┐ ┌────┐ ┌────┐
    │shell│ │patch│ │plan│ │read│ │...│
    └──┬──┘ └──┬──┘ └──┬─┘ └──┬─┘ └─┬──┘
       │       │       │      │      │
       └───────┴───────┴──────┴──────┘
                      │
                      ▼
┌──────────────────────────────────────────┐
│  2. Check Sandbox/Approval               │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  3. Execute Tool                         │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  4. Return Result to LLM                 │
└──────────────────────────────────────────┘
```

### Supported Tools
- shell, apply_patch, update_plan
- read_file, write_file
- grep, glob
- git operations

---

## 7. Issue Triage Pipeline

### Описание
Автоматическая обработка GitHub issues (labeling и deduplication).

### Местоположение
- **Labeler**: `.github/workflows/issue-labeler.yml`
- **Deduplicator**: `.github/workflows/issue-deduplicator.yml`

### ASCII Диаграмма (Issue Labeler)
```
┌──────────────────────┐
│ New GitHub Issue     │
│ Created              │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  1. Workflow Triggered                   │
│  Export env vars:                        │
│  - ISSUE_NUMBER, ISSUE_TITLE, ISSUE_BODY │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  2. Send to LLM                          │
│  Prompt: Issue Labeler Prompt           │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  3. LLM Classifies Issue                 │
│  Returns JSON array:                     │
│  ["bug", "windows-os"]                   │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  4. Apply Labels via GitHub API          │
└──────────────────────────────────────────┘
```

### ASCII Диаграмма (Issue Deduplicator)
```
┌──────────────────────┐
│ New GitHub Issue     │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  1. Fetch Recent Issues                  │
│  Save to codex-existing-issues.json      │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  2. Save Current Issue                   │
│  codex-current-issue.json                │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  3. Send to LLM                          │
│  Prompt: Issue Deduplicator Prompt      │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  4. LLM Finds Duplicates                 │
│  Returns JSON: [123, 456, 789]           │
│  (up to 5 issue numbers)                 │
└──────────┬───────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│  5. Add Comment with Duplicate Links     │
└──────────────────────────────────────────┘
```

---

## Заключение

### Сводная таблица пайплайнов

| № | Название | Триггер | Промпт | Выход |
|---|----------|---------|--------|-------|
| 1 | Main Task Execution | User input | BASE_INSTRUCTIONS | Text + Tool Calls |
| 2 | Code Review | `/review` command | REVIEW_PROMPT | JSON Findings |
| 3 | Context Compaction | Token limit 85% | SUMMARIZATION_PROMPT | Summary message |
| 4 | Sandbox Assessment | Command blocked | Sandbox Assessment | JSON {description, risk_level} |
| 5 | Planning | `update_plan` tool | (none) | UI rendering |
| 6 | Tool Execution | Tool call | (varies) | Tool result |
| 7 | Issue Triage | New GitHub issue | Issue Labeler/Deduplicator | Labels/Duplicate links |

### Общие принципы

1. **Модульность**: Каждый пайплайн независим и может вызываться отдельно
2. **Композируемость**: Пайплайны могут вызывать друг друга (Main → Review, Main → Compact)
3. **Retry логика**: Network errors обрабатываются с exponential backoff
4. **JSON выход**: Структурированные данные для легкой интеграции
5. **Sub-agents**: Code Review использует отдельную LLM сессию

### Точки расширения

- Новые инструменты добавляются через ToolRouter
- Новые промпты регистрируются в model_family.rs
- Новые пайплайны интегрируются в main event loop

---

**Документация создана**: 2025-11-11
**Автор**: AI Agent (Codex)

