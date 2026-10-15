# OpenCode V2 — Internal Flowcharts

Generated from docs + actual source code analysis (`@opencode-ai/plugin`, `opencode-mem` plugin, `session-rotator`, `opencode.jsonc`).

---

## 1. System Architecture Overview

```mermaid
graph TB
    subgraph Clients
        TUI[TUI]
        CLI[CLI]
        Web[Web App]
        SDK[SDK / External App]
    end

    subgraph "Background Service (owns everything)"
        API[HTTP API<br/>98 operations]
        Loc[Location<br/>Service Graph]
        Ses[Session Runner]
        Proj[SessionProjector<br/>→ SQLite]
        Comp[SessionCompaction]
        Perm[Permissions Engine]
        Plug[Plugin System]
        MCP[MCP Servers]
        Tools[Tool Registry<br/>built-in + MCP + plugin]
        Agents[Agent Registry]
        Models[Model Catalog<br/>providers]
    end

    TUI --> API
    CLI --> API
    Web --> API
    SDK --> API

    API --> Loc
    API --> Ses
    API --> Perm
    API --> Plug
    API --> MCP
    API --> Tools
    API --> Agents
    API --> Models

    Ses --> Proj
    Ses --> Tools
    Ses --> Comp
    Loc --> Plug
    Loc --> Tools
```

---

## 2. Session Execution Flow (the main loop)

```mermaid
sequenceDiagram
    participant U as User
    participant S as SessionRunner
    participant H as Hooks<br/>(prompt→context→generate)
    participant M as Model (LLM)
    participant T as Tool
    participant P as SessionProjector
    participant DB as SQLite

    U->>S: prompt(text)
    S->>H: hook('prompt', event)
    H-->>S: event enriched
    S->>M: LLMClient.stream(request)
    M-->>S: tool_use / text delta
    alt tool_use
        S->>T: execute(tool, args)
        T-->>S: result
        S->>H: hook('generate', event)
        S->>M: continue with tool result
    else text delta (final)
        S->>P: project message
        P->>DB: write to SQLite
        S-->>U: response text
    end
```

**Hook order (from docs + `opencode-mem` adapter):**

```mermaid
graph LR
    A[prompt] --> B[context] --> C[generate] --> D[compaction]
    A --> E[title]
    C --> F[retry]
    D --> G[session.end]
```

---

## 3. Plugin Transform & Hook Pipeline

```mermaid
graph TB
    subgraph Transforms<br/>Synchronous, ordered at load time
        T1[provider.transform]
        T2[model.transform]
        T3[agent.transform]
        T4[tool.transform]
        T5[command.transform]
        T6[skill.transform]
        T7[mcp.transform]
        T8[reference.transform]
        T9[worktree.transform]
        T10[websearch.transform]
        T11[permission.rules]
        T12[integration.transform]
        T13[vcs.transform]
        T14[plugin.list]
    end

    subgraph Hooks<br/>Live, per-session
        H1[session.hook('prompt')]
        H2[session.hook('context')]
        H3[session.hook('compaction')]
        H4[session.hook('generate')]
        H5[session.hook('title')]
        H6[session.hook('model.request')]
        H7[session.hook('http.request')]
        H8[session.hook('http.response')]
        H9[session.hook('experimental.ws.handshake')]
        H10[session.hook('retry')]
    end

    Plug --> T1 & T2 & T3 & T4 & T5 & T6 & T7 & T8 & T9 & T10 & T11 & T12 & T13 & T14
    Plug --> H1 & H2 & H3 & H4 & H5 & H6 & H7 & H8 & H9 & H10
```

**Real plugin example (opencode-mem adapter):**

```mermaid
graph TB
    subgraph "opencode-mem/src/v2/plugin.ts"
        P1[loadOpencodeProvider]
        P2[registerV2Adapter]
        P3[createLegacyClient]
    end

    subgraph "adapter.ts — registers on ctx"
        A1[tool.transform → add 'memory' tool]
        A2[session.hook('prompt') → forward to legacy]
        A3[session.hook('context') → inject memory context]
        A4[ctx.event.subscribe → bridge events to legacy]
    end

    P3 --> A2 & A3
    P1 --> A1
    A4 --> P3
```

---

## 4. Permission Resolution

```mermaid
graph TB
    subgraph "Config layers (merged in order)"
        G[global permissions]
        P[project permissions]
        A[agent-specific permissions]
        SA[saved approvals]
    end

    subgraph "Permission rules: {action, resource, effect}"
        R1[read]
        R2[edit]
        R3[glob]
        R4[grep]
        R5[shell]
        R6[subagent]
        R7[skill]
        R8[question]
        R9[webfetch]
        R10[websearch]
        R11[external_directory]
        R12[execute]
    end

    subgraph "Effects"
        AL[allow]
        DK[deny]
        ASK[ask]
    end

    G --> LastMatch["Last matching rule wins"]
    P --> LastMatch
    A --> LastMatch
    SA --> LastMatch

    LastMatch --> R1 & R2 & R3 & R4 & R5 & R6 & R7 & R8 & R9 & R10 & R11 & R12
    R1 --> AL
    R2 --> AL
    R5 --> AL
    R11 --> ASK

    subgraph "Your config (opencode.jsonc)"
        YourPerm["build: subagent*, edit*, shell*, read* allow<br/>plan: edit ask, shell/read allow<br/>explore: glob/grep/read allow, edit/shell/execute/subagent deny"]
    end

    YourPerm --> LastMatch
```

---

## 5. Agent Types & Hierarchy

```mermaid
graph TB
    subgraph "Primary Agents (drive conversation)"
        Build[build<br/>Qwen3.6-35B<br/>color: #5f87ff]
        Plan[plan<br/>Qwen3.8-27B<br/>color: #ff8c00]
    end

    subgraph "Subagent Agents (spawned workers)"
        General[general<br/>ornith-1.5:9b<br/>multi-step research/worker]
        Explore[explore<br/>gemma-4-E2B<br/>read-only glob/grep]
        Scout[scout<br/>qwen3.5-4b<br/>shell-based search]
    end

    subgraph "Hidden Agents (system)"
        Compaction[compaction<br/>gemma-4-E2B<br/>context summarization]
        Title[title<br/>qwen3.5-4b<br/>session naming]
        Summary[summary<br/>auto-generated]
    end

    Build -->|spawns| General
    Build -->|spawns| Explore
    Build -->|spawns| Scout
    Comp -->|auto| Compaction
    SessionEnd -->|auto| Title
```

---

## 6. Subagent Execution Model

```mermaid
sequenceDiagram
    participant P as Parent (build)
    participant SR as SessionRunner
    participant C as Child Session
    participant M as Child's Model

    P->>SR: subagent(agent="explore")
    SR->>C: create({parentID: parent.id})
    C-->>SR: childID

    alt foreground (default)
        SR->>C: prompt(task)
        C->>M: generate
        M-->>C: result
        C-->>SR: return result
        SR-->>P: subagent output
    else background
        SR->>C: prompt(task)
        C->>M: generate
        M-->>C: result
        C-->>SR: complete (notify)
        SR-->>P: "done" notification
    end

    Note over C: Child has fresh context,<br/>own permissions,<br/>inherits/overrides model
```

---

## 7. Context Compaction

```mermaid
graph TB
    subgraph "Detection"
        D1[Token count vs model limit]
        D2[Turn splitting]
        D3[Output pruning]
    end

    subgraph "Action"
        M["Keep: 15999 tokens (your config)"]
        S[Local summary via compaction agent]
        N[Native provider compaction]
    end

    subgraph "Flow"
        Start[Session grows] --> Detect{Overflow?}
        Detect -->|yes| Split[Split turn / prune output]
        Split --> Compress[Compress old content]
        Compress --> Marker[Insert @b1@ placeholder]
        Marker --> Continue[Continue session]
        Detect -->|no| Continue
    end

    D1 & D2 & D3 --> Detect
    Compress --> S & N
```

---

## 8. Instruction Resolution Order

```mermaid
graph TB
    subgraph "System prompt layers (priority order)"
        L1[1. Agent/provider system prompt]
        L2[2. Built-in context]
        L3[3. Code Mode guidance]
        L4[4. AGENTS.md files<br/>(workspace → home)]
        L5[5. Skill/reference/MCP guidance]
        L6[6. Session instructions]
    end

    L1 --> L2 --> L3 --> L4 --> L5 --> L6

    Note: Your build agent has custom system prompt:<br/>
    "Be succinct and efficient. Delegate all exploration..."
```

---

## 9. Location Service Graph

```mermaid
graph TB
    subgraph "Per-Location Service Graph"
        FS[filesystem]
        VCS[VCS]
        AG[agents]
        MD[models]
        PR[providers]
        TL[tools]
        PM[permissions]
        PL[plugins]
        MC[MCP]
        IN[instructions]
        SK[skills]
        CM[commands]
        RE[references]
        PT[PTY]
        RU[runner]
    end

    subgraph "Key relationships"
        FS --> TL
        VCS --> FS
        PL --> TL & PM & IN
        AG --> MD & PM
        MC --> TL
        PM --> TL
    end

    FS & VCS & AG & MD & PR & TL & PM & PL & MC & IN & SK & CM & RE & PT & RU --> Loc[Location.Ref]
```

---

## 10. HTTP API Surface (98 operations)

```mermaid
graph TB
    subgraph "Resource Groups"
        SV[server<br/>info, pair, connect]
        LC[location<br/>get, reload]
        AG[agent<br/>list, get]
        PL[plugin<br/>list, check, update]
        SE[session<br/>list, create, get, remove,<br/>update, fork, prompt,<br/>interrupt, wait,<br/>switchAgent, switchModel,<br/>compact, context,<br/>import/export]
        MS[message]
        FM[form]
        QS[question]
        PM[permission<br/>list, approve, deny]
        FI[file]
        VC[vcs]
        SH[shell]
        PT[pty]
        MO[model<br/>list, info]
        PV[provider]
        IG[integration]
        CR[credential]
        MP[mcp<br/>list, add, remove]
        SK[skill]
        RF[reference]
        CM[command]
        GE[generation]
        EV[event]
    end

    SV & LC & AG & PL & SE & MS & FM & QS & PM & FI & VC & SH & PT & MO & PV & IG & CR & MP & SK & RF & CM & GE & EV --> API[HTTP API]
```

---

## 11. SessionProjector Data Persistence

```mermaid
graph TB
    subgraph "In-memory"
        Msg[Messages]
        TC[Tool Calls]
        TR[Tool Results]
        St[State]
    end

    subgraph "Projection"
        Proj[SessionProjector]
        Sql[SQLite]
    end

    Msg --> Proj
    TC --> Proj
    TR --> Proj
    St --> Proj

    Proj --> MsgDB[(messages table)]
    Proj --> ToolDB[(tools table)]
    Proj --> SessionDB[(sessions table)]

    Note: Durable event log. All changes projected atomically.
```

---

## 12. Tool Execution Pipeline

```mermaid
graph TB
    subgraph "Tool.make contract"
        Codec[Codec<br/>serialize/deserialize]
        Exec[Execute<br/>run the tool]
        Proj[Optional model projection]
    end

    subgraph "Built-in Tools"
        T1[read]
        T2[glob]
        T3[grep]
        T4[edit]
        T5[write]
        T6[patch]
        T7[shell]
        T8[webfetch]
        T9[websearch]
        T10[question]
        T11[skill]
        T12[subagent]
        T13[execute]
    end

    subgraph "MCP Tools"
        MC1[MCP server 1 tools]
        MC2[MCP server 2 tools]
    end

    subgraph "Plugin Tools"
        PL1[plugin tool 1]
        PL2[plugin tool 2]
    end

    Codec --> Exec --> Proj
    T1 & T2 & T3 & T4 & T5 & T6 & T7 & T8 & T9 & T10 & T11 & T12 & T13 --> Codec
    MC1 & MC2 --> Codec
    PL1 & PL2 --> Codec

    subgraph "Permission gating"
        PermRead[read → path]
        PermEdit[edit → edit/write/patch paths]
        PermGlob[glob → pattern]
        PermGrep[grep → regex]
        PermShell[shell → command string]
        PermSub[subagent → agent ID]
        PermSkill[skill → skill ID]
        PermQ[question → *]
        PermFetch[webfetch → URL]
        PermSearch[websearch → query]
        PermExt[external_directory → dir boundary]
        PermExec[execute → *]
    end

    Exec --> PermRead & PermEdit & PermGlob & PermGrep & PermShell & PermSub & PermSkill & PermQ & PermFetch & PermSearch & PermExt & PermExec
```

---

## 13. Worktree Management

```mermaid
graph TB
    subgraph "Worktree Strategies"
        Git[Git (default)]
        P1[Plugin strategy 1]
        P2[Plugin strategy 2]
    end

    subgraph "Operations"
        C[create]
        R[remove]
        L[list]
        F[refresh]
    end

    Git & P1 & P2 --> Ops[C & R & L & F]

    Note: Strategy-scoped. Created per-repository.
```

---

## 14. Event Bus Architecture

```mermaid
graph TB
    subgraph "EventV2 — Global Event Bus"
        EV[EventV2]
    end

    subgraph "Event Sources"
        ES1[session.created]
        ES2[session.updated]
        ES3[session.execution.succeeded]
        ES4[session.execution.failed]
        ES5[session.compaction.ended]
        ES6[session.prompt.sent]
        ES7[message.created]
        ES8[message.updated]
    end

    subgraph "Event Consumers"
        EC1[opencode-mem plugin]
        EC2[session-rotator plugin]
        EC3[DCP plugin]
        EC4[external clients]
    end

    ES1 & ES2 & ES3 & ES4 & ES5 & ES6 & ES7 & ES8 --> EV

    EV --> EC1
    EV --> EC2
    EV --> EC3
    EV --> EC4

    Note: opencode-mem uses ctx.event.subscribe() to listen<br/>and bridges v2 events to v1-compatible format via toLegacyEvent()
```

---

## 15. Real Plugin Flow — opencode-mem

```mermaid
sequenceDiagram
    participant OC as OpenCode Service
    participant PL as opencode-mem plugin
    participant LM as Legacy Memory System

    OC->>PL: setup(ctx)
    PL->>PL: createLegacyClient(ctx)
    PL->>PL: registerV2Adapter(ctx, legacy)

    PL->>PL: tool.transform → add 'memory' tool
    PL->>PL: session.hook('prompt') → forward to legacy
    PL->>PL: session.hook('context') → inject memory context
    PL->>PL: ctx.event.subscribe → bridge events

    OC->>PL: user calls memory tool
    PL->>LM: legacy.tool.memory.execute(args)
    LM-->>PL: tool result
    PL-->>OC: {content, metadata}

    OC->>PL: session.hook('prompt', event)
    PL->>LM: chat.message({sessionID, output})
    LM-->>PL: injected parts
    PL->>PL: store injected context

    OC->>PL: session.hook('context', event)
    PL->>PL: push injected context to event.system
```

---

## 16. Real Plugin Flow — session-rotator (disabled)

```mermaid
sequenceDiagram
    participant OC as OpenCode Service
    participant SR as session-rotator plugin
    participant N as New Session

    OC->>SR: setup(ctx)
    SR->>SR: N=3, SKIP_AGENTS={explore, scout}

    OC->>SR: session.hook('context', event)
    SR->>SR: count turns for session
    SR->>SR: if n%N==0 → hardFork

    SR->>OC: ctx.session.create({parentID, title})
    OC-->>SR: newSessionID

    SR->>OC: ctx.session.prompt({sessionID: newSessionID, text})
    SR->>SR: event.system.push(ROTATED message)

    OC->>SR: session.hook('prompt', event)
    SR->>SR: if prompt starts with "## Session Summary" → reset count to 0
```

---

## 17. Your Current Agent Permission Matrix

| Agent | Mode | Model | read | edit | shell | glob | grep | subagent | execute |
|-------|------|-------|------|------|-------|------|------|----------|---------|
| **build** | primary | Qwen3.6-35B | allow | allow | allow | — | — | allow | — |
| **plan** | primary | Qwen3.8-27B | allow | ask | allow | — | — | — | — |
| **general** | subagent | ornith-1.5:9b | — | allow | allow | — | — | — | — |
| **compaction** | subagent | gemma-4-E2B | — | allow | allow | — | — | — | — |
| **title** | subagent | qwen3.5-4b | — | allow | allow | — | — | — | — |
| **explore** | subagent | gemma-4-E2B | allow | deny | deny | allow | allow | deny | deny |
| **scout** | subagent | qwen3.5-4b | allow | deny | allow | allow | allow | deny | deny |

---

## 18. Configuration Merge Order

```mermaid
graph LR
    A[~/.config/opencode/opencode.jsonc<br/>global] --> B[project/opencode.jsonc<br/>project-local]
    B --> C[.opencode/opencode.jsonc<br/>project/.opencode]
    C --> D[plugin transforms<br/>synchronous]
    D --> E[agent-specific rules<br/>override]
    E --> F[saved approvals<br/>runtime]

    Note: Last matching rule wins for permissions.
    Note: Config merged farthest ancestor → current directory.
```

---

## 19. Prompt → Response Lifecycle

```mermaid
sequenceDiagram
    participant U as User
    participant T as TUI/CLI
    participant A as Agent (build)
    participant S as SessionRunner
    participant M as Model
    participant TL as Tools

    U->>T: type message
    T->>S: session.prompt(text)
    S->>A: inject system prompt + context
    A->>M: stream request
    alt text response
        M-->>A: text delta
        A-->>T: stream to user
    else tool_use
        M-->>A: tool_call
        A->>TL: execute(tool, args)
        TL-->>A: result
        A->>M: continue with result
    end
    A->>S: project to SQLite
    S->>T: update session
```

---

## 20. Model Provider Resolution

```mermaid
graph TB
    subgraph "Providers (opencode.jsonc)"
        P1[nerdbook-pro<br/>Qwen3.6, Qwen3.8, Qwen3.5]
        P2[nerdbook-air<br/>gemma-4-12B, gemma-4-E2B]
        P3[serenity<br/>glm-4.6v-flash]
        P4[enterprise<br/>ornith-1.5, qwen3.5:4b]
        P5[nerdland<br/>gemma4:e2b, qwen3.5-4b]
    end

    subgraph "Agent model assignment"
        A1[build → nerdbook-pro/Qwen3.6]
        A2[plan → nerdbook-pro/Qwen3.8]
        A3[general → enterprise/ornith-1.5]
        A4[compaction → nerdbook-air/gemma-4-E2B]
        A5[title → nerdland/qwen3.5-4b]
        A6[explore → nerdbook-air/gemma-4-E2B]
        A7[scout → nerdland/qwen3.5-4b]
    end

    P1 & P2 & P3 & P4 & P5 --> Cat[Model Catalog]
    Cat --> A1 & A2 & A3 & A4 & A5 & A6 & A7

    Note: Each agent can override the default model.
    Note: Providers use OpenAI-compatible format.
```
