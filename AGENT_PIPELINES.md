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

