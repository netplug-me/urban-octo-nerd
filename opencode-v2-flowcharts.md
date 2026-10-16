# OpenCode V2 Architecture Flowcharts

## 1. System Architecture Overview

```mermaid 
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '13px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart TB
    %% Styling Definitions
    classDef client fill:#1e1b4b,stroke:#818cf8,stroke-width:2px,color:#e0e7ff,rx:8px,ry:8px;
    classDef gateway fill:#312e81,stroke:#a78bfa,stroke-width:3px,color:#f5f3ff,rx:10px,ry:10px;
    classDef service fill:#064e3b,stroke:#34d399,stroke-width:2px,color:#ecfdf5,rx:6px,ry:6px;
    classDef storage fill:#701a75,stroke:#f472b6,stroke-width:2px,color:#fdf2f8,rx:4px,ry:4px;
    classDef subStyle fill:#090d16,stroke:#334155,stroke-width:1.5px,stroke-dasharray: 4 4,color:#94a3b8;

    subgraph CLIENTS[" 💻 Client Layer "]
        direction LR
        TUI["fa:fa-terminal Terminal UI"]:::client
        CLI["fa:fa-square-terminal Command Line"]:::client
        Web["fa:fa-globe Web App"]:::client
        SDK["fa:fa-code TypeScript SDK"]:::client
        ClientLib["fa:fa-box-open @opencode/client"]:::client
    end

    subgraph CORE[" ⚡ Background Service Core "]
        HttpApi["fa:fa-network-wired <b>HTTP API Gateway</b><br/><i>[ 98 endpoints ]</i>"]:::gateway

        subgraph MANAGERS[" 🧩 Orchestration Engines "]
            direction TB
            SessionMgr["fa:fa-user-clock Session Manager"]:::service
            LocationMgr["fa:fa-compass Location Manager"]:::service
            PluginMgr["fa:fa-puzzle-piece Plugin Manager"]:::service
            PermMgr["fa:fa-shield-halved Permissions Manager"]:::service
            ModelMgr["fa:fa-brain Model Manager"]:::service
            MCPManager["fa:fa-server MCP Server Manager"]:::service
        end
    end

    subgraph PERSISTENCE[" 💾 Persistence & State "]
        direction TB
        SQLite[("fa:fa-database SQLite DB<br/><i>(Active Sessions)</i>")]:::storage
        Config["fa:fa-file-code Config Storage<br/><i>(opencode.jsonc)</i>"]:::storage
        Logs["fa:fa-file-lines Audit & Logs"]:::storage
    end

    %% Client -> API Gateway Ingress
    TUI & CLI & Web & SDK & ClientLib ==>|JSON-RPC / HTTP| HttpApi

    %% Gateway -> Internal Dispatch
    HttpApi --> SessionMgr & LocationMgr & PluginMgr & PermMgr & ModelMgr & MCPManager

    %% Service -> Storage Egress
    SessionMgr -->|RW State| SQLite
    PluginMgr & LocationMgr -->|Sync Config| Config
    HttpApi -.->|Telemetry| Logs

    %% Subgraph Classes
    class CLIENTS,CORE,MANAGERS,PERSISTENCE subStyle;
```

```mermaid
graph TB
    subgraph Clients
        TUI[Terminal UI]
        CLI[Command Line]
        Web[Web App]
        SDK[TypeScript SDK]
        ClientLib[@opencode/client]
    end

    subgraph "Background Service"
        HttpApi[HTTP API<br/>98 operations]
        SessionMgr[Session Manager]
        LocationMgr[Location Manager]
        PluginMgr[Plugin Manager]
        PermMgr[Permissions Manager]
        ModelMgr[Model Manager]
        MCPManager[MCP Server Manager]
    end

    subgraph Storage
        SQLite[(SQLite<br/>Session DB)]
        Config[Config Files<br/>opencode.jsonc]
        Logs[Log Files]
    end

    TUI --> HttpApi
    CLI --> HttpApi
    Web --> HttpApi
    SDK --> HttpApi
    ClientLib --> HttpApi

    HttpApi --> SessionMgr
    HttpApi --> LocationMgr
    HttpApi --> PluginMgr
    HttpApi --> PermMgr
    HttpApi --> ModelMgr
    HttpApi --> MCPManager

    SessionMgr --> SQLite
    PluginMgr --> Config
    LocationMgr --> Config
```

## 2. Session Execution Flow

```mermaid
sequenceDiagram
    participant U as User
    participant C as Client (TUI/CLI)
    participant S as Session Runner
    participant P as Plugin Manager
    participant M as Model/Provider
    participant L as LLM Client
    participant T as Tools
    participant DB as SQLite

    U->>C: Send prompt
    C->>S: prompt(sessionID, message)
    S->>S: session.hook('prompt')
    S->>P: Apply transforms
    S->>M: Resolve model/catalog
    S->>S: Build context<br/>(system + AGENTS.md + skills + MCP)
    S->>L: stream(request)
    L-->>S: Token stream (tool calls + text)
    S->>S: Detect tool call
    S->>T: Execute tool
    T-->>S: Tool result
    S->>S: session.hook('generate')
    S->>DB: Project to SQLite
    S->>L: Continue stream<br/>(next turn)
    L-->>S: Next tokens
    Note over S,L: Loop until model stops
    S->>DB: Final projection + compaction
    S-->>C: Final response
    C-->>U: Display output
```

## 3. Plugin Transform & Hook Pipeline

```mermaid
graph TB
    subgraph "Transforms (Ordered, Synchronous)"
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

    subgraph "Hooks (Live Operations)"
        H1[prompt]
        H2[context]
        H3[compaction]
        H4[generate]
        H5[title]
        H6[model.request]
        H7[http.request]
        H8[http.response]
        H9[ws.handshake]
        H10[retry]
    end

    PluginDef --> T1 --> T2 --> T3 --> T4 --> T5 --> T6 --> T7 --> T8 --> T9 --> T10 --> T11 --> T12 --> T13 --> T14

    PluginDef --> Hooks
    Hooks --> H1 & H2 & H3 & H4 & H5 & H6 & H7 & H8 & H9 & H10

    style PluginDef fill:#f9f,stroke:#333
    style Hooks fill:#ccf,stroke:#333
```

## 4. Permission Resolution Flow

```mermaid
graph TB
    subgraph "Permission Rules (Last Match Wins)"
        R1[Global Rules]
        R2[Project Rules]
        R3[Agent-Specific Rules]
        R4[Saved Approvals]
    end

    subgraph "Policy Hard-Denies (After Rules)"
        P1[Policy: deny]
    end

    subgraph "Default Base Policy"
        D1[* → allow]
        D2[external_directory → ask]
        D3[*.env → ask]
        D4[*.env.example → allow]
    end

    ToolCall --> R1 --> R2 --> R3 --> R4 --> P1

    R1 -.-> D1 & D2 & D3 & D4
    R2 -.-> D1 & D2 & D3 & D4
    R3 -.-> D1 & D2 & D3 & D4

    style ToolCall fill:#ff9,stroke:#333
    style P1 fill:#f99,stroke:#333
```

## 5. Agent Types & Hierarchy

```mermaid
graph TB
    subgraph "Primary Agents (user-facing)"
        Build[Build Agent]
        Plan[Plan Agent]
    end

    subgraph "Subagent Agents (spawned by build/plan)"
        General[General<br/>Multi-step research/worker]
        Explore[Explore<br/>Read-only file search]
        Scout[Scout<br/>Quick file finder]
    end

    subgraph "Hidden Agents (system)"
        Compaction[Compaction<br/>Context management]
        Title[Title<br/>Session naming]
        Summary[Summary<br/>Conversation summary]
    end

    Build --> General & Explore & Scout
    Plan --> General & Explore & Scout

    style Build fill:#9f9,stroke:#333
    style Plan fill:#9f9,stroke:#333
    style General fill:#9cf,stroke:#333
    style Explore fill:#9cf,stroke:#333
    style Scout fill:#9cf,stroke:#333
    style Compaction fill:#ccc,stroke:#333
    style Title fill:#ccc,stroke:#333
    style Summary fill:#ccc,stroke:#333
```

## 6. Subagent Execution Model

```mermaid
sequenceDiagram
    participant Parent as Parent Agent<br/>(Build/Plan)
    participant SM as Session Manager
    participant Child as Child Session
    participant DB as SQLite

    Parent->>SM: Launch subagent<br/>(agent, model, context)
    SM->>SM: Create child session<br/>(parentID, agent, model)
    SM->>Child: Initialize fresh context
    Child->>DB: Record messages
    Child->>Child: Execute tools independently
    Child-->>Parent: Result (foreground) / notify (background)
    Parent->>Parent: Continue main flow
```

## 7. Context Compaction Flow

```mermaid
graph TB
    subgraph "Detection"
        D1[Token count vs model limit]
        D2[Turn splitting check]
    end

    subgraph "Compaction Process"
        P1[Output pruning]
        P2[Compaction markers]
        P3[Durable barrier]
    end

    subgraph "Compaction Methods"
        M1[Local summary<br/>(SessionCompaction)]
        M2[Native provider compaction]
    end

    D1 --> P1 --> P3 --> M1
    D2 --> P2 --> P3 --> M1
    P3 --> M2

    M1 --> DB[(SQLite)]
    M2 --> DB

    style DB fill:#ccf,stroke:#333
```

## 8. Instruction Resolution Order

```mermaid
graph LR
    I1[Agent/Provider<br/>System Prompt] --> I2[Built-in<br/>Context]
    I2 --> I3[Code Mode<br/>Guidance]
    I3 --> I4[AGENTS.md<br/>Discovery<br/>workspace→home]
    I4 --> I5[Skill/Reference<br/>MCP Guidance]
    I5 --> I6[Session<br/>Instructions]

    style I1 fill:#f9f,stroke:#333
    style I6 fill:#9cf,stroke:#333
```

## 9. Location & Service Graph

```mermaid
graph TB
    subgraph "Location (Project Root)"
        Loc[Location.Ref]
    end

    Loc --> FS[Filesystem]
    Loc --> VCS[VCS]
    Loc --> Ag[Agents]
    Loc --> Md[Models]
    Loc --> Pr[Providers]
    Loc --> To[Tools]
    Loc --> Pe[Permissions]
    Loc --> Pl[Plugins]
    Loc --> Mc[MCP]
    Loc --> In[Instructions]
    Loc --> Sk[Skills]
    Loc --> Cm[Commands]
    Loc --> Re[References]
    Loc --> PT[PTY]
    Loc --> Ru[Runner]

    style Loc fill:#f9f,stroke:#333
```

## 10. HTTP API Surface (98 operations)

```mermaid
graph TB
    subgraph "Server"
        S1[info]
        S2[pair]
        S3[connect]
    end

    subgraph "Location"
        L1[get]
        L2[reload]
    end

    subgraph "Agent"
        A1[list]
        A2[get]
    end

    subgraph "Plugin"
        P1[list]
        P2[check]
        P3[update]
    end

    subgraph "Session"
        SE1[list]
        SE2[create]
        SE3[get]
        SE4[remove]
        SE5[update]
        SE6[fork]
        SE7[prompt]
        SE8[interrupt]
        SE9[wait]
        SE10[switchAgent]
        SE11[switchModel]
        SE12[compact]
        SE13[context]
        SE14[import]
        SE15[export]
    end

    subgraph "Other"
        O1[message]
        O2[form]
        O3[question]
        O4[permission]
        O5[file]
        O6[vcs]
        O7[shell]
        O8[pty]
        O9[model]
        O10[provider]
        O11[integration]
        O12[credential]
        O13[mcp]
        O14[skill]
        O15[reference]
        O16[command]
        O17[generation]
        O18[event]
    end

    API[HTTP API] --> S1 & S2 & S3
    API --> L1 & L2
    API --> A1 & A2
    API --> P1 & P2 & P3
    API --> SE1 & SE2 & SE3 & SE4 & SE5 & SE6 & SE7 & SE8 & SE9 & SE10 & SE11 & SE12 & SE13 & SE14 & SE15
    API --> O1 & O2 & O3 & O4 & O5 & O6 & O7 & O8 & O9 & O10 & O11 & O12 & O13 & O14 & O15 & O16 & O17 & O18
```

## 11. Data Persistence: SessionProjector

```mermaid
graph TB
    subgraph "In-Memory"
        Msg[Messages]
        ToolCalls[Tool Calls]
        State[Session State]
    end

    subgraph "SessionProjector"
        Proj[Projector]
    end

    subgraph "SQLite Database"
        DB[(Sessions Table)]
        DB2[(Messages Table)]
        DB3[(ToolResults Table)]
        DB4[(Compactions Table)]
    end

    Msg --> Proj
    ToolCalls --> Proj
    State --> Proj
    Proj --> DB
    Proj --> DB2
    Proj --> DB3
    Proj --> DB4

    style Proj fill:#f9f,stroke:#333
```

## 12. Tool Execution Pipeline

```mermaid
graph TB
    subgraph "Tool Contract"
        Make[Tool.make<br/>Codecs + Execution + Projection]
    end

    Make --> Builtins
    Make --> MCP_Tools
    Make --> Plugin_Tools

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

    subgraph "External Tools"
        M1[MCP Tools]
        P1[Plugin Tools]
    end

    Builtins --> T1 & T2 & T3 & T4 & T5 & T6 & T7 & T8 & T9 & T10 & T11 & T12 & T13
    MCP_Tools --> M1
    Plugin_Tools --> P1

    subgraph "Permission Actions"
        PA1[read]
        PA2[edit]
        PA3[glob]
        PA4[grep]
        PA5[shell]
        PA6[subagent]
        PA7[skill]
        PA8[question]
        PA9[webfetch]
        PA10[websearch]
        PA11[external_directory]
        PA12[execute]
    end

    T1 -.-> PA1
    T4 -.-> PA2
    T2 -.-> PA3
    T3 -.-> PA4
    T7 -.-> PA5
    T12 -.-> PA6
    T11 -.-> PA7
    T10 -.-> PA8
    T8 -.-> PA9
    T9 -.-> PA10
    T7 -.-> PA11
    T13 -.-> PA12
```

## 13. Worktree Management

```mermaid
graph TB
    subgraph "Git Strategy (Default)"
        WT1[worktree create]
        WT2[worktree list]
        WT3[worktree remove]
    end

    subgraph "Plugin Strategies"
        PS1[Custom strategy]
    end

    Location --> WT1 & WT2 & WT3
    Location --> PS1

    WT1 --> Dir[Worktree Directory]
    Dir --> Session[Session Working Dir]
    Session --> Runner[SessionRunner]
```

## 14. Event Bus Architecture

```mermaid
graph TB
    subgraph "EventV2"
        Bus[Global Event Bus]
    end

    subgraph "Event Sources"
        S1[SessionProjector]
        S2[SessionRunner]
        S3[PluginManager]
        S4[ToolExecution]
        S5[ModelProvider]
    end

    subgraph "Event Consumers"
        C1[SQLite Storage]
        C2[UI Clients]
        C3[Plugin Hooks]
        C4[Logging]
    end

    S1 --> Bus
    S2 --> Bus
    S3 --> Bus
    S4 --> Bus
    S5 --> Bus

    Bus --> C1
    Bus --> C2
    Bus --> C3
    Bus --> C4
```

---

*Generated from OpenCode V2 source code and documentation analysis.*
