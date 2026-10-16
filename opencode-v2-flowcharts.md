# OpenCode V2 Architecture Flowcharts

## 1. System Architecture Overview

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '13px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart TB
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

    TUI & CLI & Web & SDK & ClientLib ==>|JSON-RPC / HTTP| HttpApi
    HttpApi --> SessionMgr & LocationMgr & PluginMgr & PermMgr & ModelMgr & MCPManager

    SessionMgr -->|RW State| SQLite
    PluginMgr & LocationMgr -->|Sync Config| Config
    HttpApi -.->|Telemetry| Logs

    class CLIENTS,CORE,MANAGERS,PERSISTENCE subStyle;
```

## 2. Session Execution Flow

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '12px', 'darkMode': true, 'actorBkg': '#1e1b4b', 'actorBorder': '#818cf8', 'actorTextColor': '#e0e7ff', 'signalColor': '#94a3b8', 'signalTextColor': '#cbd5e1', 'noteBkgColor': '#1e293b', 'noteTextColor': '#38bdf8', 'noteBorderColor': '#0284c7'}}}%%
sequenceDiagram
    autonumber
    actor U as 👤 User
    participant C as 💻 Client (TUI/CLI)
    participant S as ⚡ Session Runner
    participant P as 🔌 Plugin Manager
    participant M as 🧠 Model Resolver
    participant L as 🌐 LLM Client
    participant T as 🛠️ Tools
    participant DB as 💾 SQLite DB

    U->>C: Submit prompt
    C->>S: prompt(sessionID, message)
    critical Context Preparation
        S->>S: session.hook('prompt')
        S->>P: Apply transform pipeline
        S->>M: Resolve model catalog & config
        S->>S: Assemble context (System + AGENTS.md + Skills + MCP)
    end

    S->>+L: stream(request)
    loop Generation & Tool Invocation
        L-->>S: Token stream (tool calls + text)
        alt Tool Call Triggered
            S->>T: Execute tool call
            T-->>S: Return tool execution result
            S->>S: session.hook('generate')
            S->>DB: Project partial state to SQLite
            S->>L: Continue stream with tool context
        end
    end
    L-->>-S: Final completion signal

    S->>DB: Final projection + compaction
    S-->>C: Complete response payload
    C-->>U: Render formatted output
```

## 3. Plugin Transform & Hook Pipeline

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '12px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart TB
    classDef source fill:#1e1b4b,stroke:#818cf8,stroke-width:2px,color:#e0e7ff,rx:6px,ry:6px;
    classDef transform fill:#064e3b,stroke:#34d399,stroke-width:1.5px,color:#ecfdf5,rx:4px,ry:4px;
    classDef hook fill:#701a75,stroke:#f472b6,stroke-width:1.5px,color:#fdf2f8,rx:4px,ry:4px;
    classDef subStyle fill:#090d16,stroke:#334155,stroke-width:1.5px,stroke-dasharray: 4 4,color:#94a3b8;

    PluginDef["fa:fa-puzzle-piece <b>Plugin Definition</b><br/>(Core Specification)"]:::source

    subgraph TRANSFORMS[" 🔄 Ordered Synchronous Transforms "]
        direction TB
        T1["1. provider.transform"]:::transform --> T2["2. model.transform"]:::transform
        T2 --> T3["3. agent.transform"]:::transform --> T4["4. tool.transform"]:::transform
        T4 --> T5["5. command.transform"]:::transform --> T6["6. skill.transform"]:::transform
        T6 --> T7["7. mcp.transform"]:::transform --> T8["8. reference.transform"]:::transform
        T8 --> T9["9. worktree.transform"]:::transform --> T10["10. websearch.transform"]:::transform
        T10 --> T11["11. permission.rules"]:::transform --> T12["12. integration.transform"]:::transform
        T12 --> T13["13. vcs.transform"]:::transform --> T14["14. plugin.list"]:::transform
    end

    subgraph HOOKS[" ⚡ Live Operational Hooks "]
        direction TB
        H1["fa:fa-terminal prompt"]:::hook
        H2["fa:fa-book-open context"]:::hook
        H3["fa:fa-compress compaction"]:::hook
        H4["fa:fa-microchip generate"]:::hook
        H5["fa:fa-tag title"]:::hook
        H6["fa:fa-cloud model.request"]:::hook
        H7["fa:fa-arrow-up-right-from-square http.request"]:::hook
        H8["fa:fa-arrow-down-left-and-up-right-to-spread http.response"]:::hook
        H9["fa:fa-handshake ws.handshake"]:::hook
        H10["fa:fa-rotate-right retry"]:::hook
    end

    PluginDef ==>|Pipeline Initialization| T1
    PluginDef ==>|Runtime Event Binding| HOOKS

    class TRANSFORMS,HOOKS subStyle;
```

## 4. Permission Resolution Flow

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '13px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart TB
    classDef start fill:#1e1b4b,stroke:#818cf8,stroke-width:2px,color:#e0e7ff,rx:6px,ry:6px;
    classDef rule fill:#064e3b,stroke:#34d399,stroke-width:2px,color:#ecfdf5,rx:4px,ry:4px;
    classDef deny fill:#450a0a,stroke:#f87171,stroke-width:2px,color:#fef2f2,rx:4px,ry:4px;
    classDef default fill:#1e293b,stroke:#94a3b8,stroke-width:1.5px,color:#f8fafc,rx:4px,ry:4px;
    classDef subStyle fill:#090d16,stroke:#334155,stroke-width:1.5px,stroke-dasharray: 4 4,color:#94a3b8;

    ToolCall["fa:fa-terminal <b>Tool Call Request</b>"]:::start

    subgraph RULES[" 🛡️ Resolution Priority (Last Match Wins) "]
        direction TB
        R1["1. Global Rules"]:::rule
        R2["2. Project Rules"]:::rule
        R3["3. Agent-Specific Rules"]:::rule
        R4["4. Saved User Approvals"]:::rule
        R1 --> R2 --> R3 --> R4
    end

    subgraph POLICY[" 🚫 Enforcement Stage "]
        P1["fa:fa-ban Hard Policy Denials"]:::deny
    end

    subgraph DEFAULTS[" ⚖️ Default Base Policy (Fallback) "]
        direction TB
        D1["* → allow"]:::default
        D2["external_directory → ask"]:::default
        D3["*.env → ask"]:::default
        D4["*.env.example → allow"]:::default
    end

    ToolCall ==> RULES
    R4 ==> POLICY
    RULES -.->|Fallback if Unmatched| DEFAULTS

    class RULES,POLICY,DEFAULTS subStyle;
```

## 5. Agent Types & Hierarchy

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '13px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart TB
    classDef primary fill:#1e1b4b,stroke:#818cf8,stroke-width:2px,color:#e0e7ff,rx:8px,ry:8px;
    classDef subagent fill:#064e3b,stroke:#34d399,stroke-width:2px,color:#ecfdf5,rx:6px,ry:6px;
    classDef system fill:#334155,stroke:#94a3b8,stroke-width:2px,color:#f8fafc,rx:6px,ry:6px;
    classDef subStyle fill:#090d16,stroke:#334155,stroke-width:1.5px,stroke-dasharray: 4 4,color:#94a3b8;

    subgraph PRIMARY[" 🧑‍💻 User-Facing Primary Agents "]
        Build["fa:fa-hammer <b>Build Agent</b><br/>Full generation & execution"]:::primary
        Plan["fa:fa-compass <b>Plan Agent</b><br/>Architectural planning & specs"]:::primary
    end

    subgraph WORKERS[" 🤖 Specialized Subagents "]
        direction TB
        General["fa:fa-briefcase <b>General Subagent</b><br/>Multi-step research and workflows"]:::subagent
        Explore["fa:fa-magnifying-glass <b>Explore Subagent</b><br/>Read-only codebase search"]:::subagent
        Scout["fa:fa-bolt <b>Scout Subagent</b><br/>High-speed targeted file lookup"]:::subagent
    end

    subgraph SYSTEM[" ⚙️ Hidden System Agents "]
        direction TB
        Compaction["fa:fa-compress <b>Compaction Agent</b><br/>Token trimming & summarization"]:::system
        Title["fa:fa-pen <b>Title Agent</b><br/>Contextual session labelling"]:::system
        Summary["fa:fa-book <b>Summary Agent</b><br/>Conversation timeline caching"]:::system
    end

    Build & Plan ==>|Spawn Delegate| General & Explore & Scout

    class PRIMARY,WORKERS,SYSTEM subStyle;
```

## 6. Subagent Execution Model

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '12px', 'darkMode': true, 'actorBkg': '#1e1b4b', 'actorBorder': '#818cf8', 'actorTextColor': '#e0e7ff', 'signalColor': '#94a3b8', 'signalTextColor': '#cbd5e1', 'noteBkgColor': '#1e293b', 'noteTextColor': '#38bdf8', 'noteBorderColor': '#0284c7'}}}%%
sequenceDiagram
    autonumber
    participant Parent as 🧑‍💻 Parent Agent (Build/Plan)
    participant SM as ⚡ Session Manager
    participant Child as 🤖 Child Session
    participant DB as 💾 SQLite DB

    Parent->>SM: launchSubagent(agentType, model, initialContext)
    activate SM
    SM->>SM: createSession(parentID, agent, model)
    SM->>+Child: Initialize isolated environment
    deactivate SM

    Child->>DB: Register initial message stream
    loop Independent Processing
        Child->>Child: Execute assigned tools & compute steps
    end

    alt Foreground Invocation
        Child-->>Parent: Synchronous aggregated result payload
    else Background Invocation
        Child-->>Parent: Async notification (EventBus completion)
    end
    deactivate Child

    Parent->>Parent: Incorporate result into primary context
```

## 7. Context Compaction Flow

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '13px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart TB
    classDef detection fill:#1e293b,stroke:#38bdf8,stroke-width:2px,color:#f8fafc,rx:6px,ry:6px;
    classDef process fill:#312e81,stroke:#818cf8,stroke-width:2px,color:#f5f3ff,rx:6px,ry:6px;
    classDef method fill:#064e3b,stroke:#34d399,stroke-width:2px,color:#ecfdf5,rx:6px,ry:6px;
    classDef storage fill:#701a75,stroke:#f472b6,stroke-width:2px,color:#fdf2f8,rx:4px,ry:4px;
    classDef subStyle fill:#090d16,stroke:#334155,stroke-width:1.5px,stroke-dasharray: 4 4,color:#94a3b8;

    subgraph DETECTION[" 🔍 Detection Layer "]
        direction TB
        D1["fa:fa-gauge Token Count vs Limit"]:::detection
        D2["fa:fa-scissors Turn Splitting Check"]:::detection
    end

    subgraph PROCESS[" ⚙️ Compaction Pipeline "]
        direction TB
        P1["fa:fa-filter Output Pruning"]:::process
        P2["fa:fa-bookmark Compaction Markers"]:::process
        P3["fa:fa-shield-halved Durable Barrier"]:::process
    end

    subgraph METHODS[" 📦 Compaction Strategies "]
        direction TB
        M1["fa:fa-file-lines Local Summary<br/><i>(SessionCompaction)</i>"]:::method
        M2["fa:fa-server Native Provider Compaction"]:::method
    end

    subgraph PERSISTENCE[" 💾 State & Storage "]
        DB[("fa:fa-database SQLite DB")]:::storage
    end

    D1 --> P1 --> P3
    D2 --> P2 --> P3

    P3 ==>|Local Fallback| M1
    P3 ==>|Provider API| M2

    M1 & M2 -->|Persist Compressed Context| DB

    class DETECTION,PROCESS,METHODS,PERSISTENCE subStyle;
```

## 8. Instruction Resolution Order

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '12px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart LR
    classDef root fill:#1e1b4b,stroke:#818cf8,stroke-width:2px,color:#e0e7ff,rx:6px,ry:6px;
    classDef step fill:#064e3b,stroke:#34d399,stroke-width:2px,color:#ecfdf5,rx:6px,ry:6px;
    classDef priority fill:#701a75,stroke:#f472b6,stroke-width:2px,color:#fdf2f8,rx:6px,ry:6px;

    I1["fa:fa-gear <b>System Prompt</b><br/>Provider & Agent Base"]:::root
    I2["fa:fa-box <b>Built-in Context</b><br/>Runtime Metadata"]:::step
    I3["fa:fa-code <b>Code Mode</b><br/>Execution Guidance"]:::step
    I4["fa:fa-file-lines <b>AGENTS.md</b><br/>Workspace → Home"]:::step
    I5["fa:fa-puzzle-piece <b>Skills & MCP</b><br/>Dynamic Tools"]:::step
    I6["fa:fa-user-check <b>Session Instructions</b><br/><i>Highest Precedence</i>"]:::priority

    I1 ==>|Pave Base| I2
    I2 ==>|Inject| I3
    I3 ==>|Augment| I4
    I4 ==>|Extend| I5
    I5 ==>|Override| I6
```

## 9. Location & Service Graph

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '13px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart TB
    classDef hub fill:#312e81,stroke:#a78bfa,stroke-width:3px,color:#f5f3ff,rx:10px,ry:10px;
    classDef infra fill:#1e293b,stroke:#38bdf8,stroke-width:1.5px,color:#f8fafc,rx:6px,ry:6px;
    classDef agent fill:#064e3b,stroke:#34d399,stroke-width:1.5px,color:#ecfdf5,rx:6px,ry:6px;
    classDef ext fill:#701a75,stroke:#f472b6,stroke-width:1.5px,color:#fdf2f8,rx:6px,ry:6px;
    classDef subStyle fill:#090d16,stroke:#334155,stroke-width:1.5px,stroke-dasharray: 4 4,color:#94a3b8;

    Loc["fa:fa-folder-tree <b>Location.Ref</b><br/><i>(Project Root Workspace)</i>"]:::hub

    subgraph SYSTEM_SERVICES[" ⚙️ System & Execution "]
        FS["fa:fa-hard-drive Filesystem"]:::infra
        VCS["fa:fa-code-branch Version Control"]:::infra
        PT["fa:fa-terminal PTY Service"]:::infra
        Ru["fa:fa-play Session Runner"]:::infra
    end

    subgraph AGENT_CONFIG[" 🧠 Agent & Context Resources "]
        Ag["fa:fa-robot Agents"]:::agent
        Md["fa:fa-brain Models"]:::agent
        Pr["fa:fa-cloud Providers"]:::agent
        In["fa:fa-book Instructions"]:::agent
        Pe["fa:fa-shield-halved Permissions"]:::agent
    end

    subgraph EXTENSIONS[" 🔌 Capabilities & Tooling "]
        To["fa:fa-toolbox Tools"]:::ext
        Pl["fa:fa-puzzle-piece Plugins"]:::ext
        Mc["fa:fa-server MCP Servers"]:::ext
        Sk["fa:fa-bolt Skills"]:::ext
        Cm["fa:fa-terminal Commands"]:::ext
        Re["fa:fa-bookmark References"]:::ext
    end

    Loc ==> SYSTEM_SERVICES
    Loc ==> AGENT_CONFIG
    Loc ==> EXTENSIONS

    class SYSTEM_SERVICES,AGENT_CONFIG,EXTENSIONS subStyle;
```

## 10. HTTP API Surface (98 operations)

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '12px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart TB
    classDef gateway fill:#312e81,stroke:#a78bfa,stroke-width:3px,color:#f5f3ff,rx:10px,ry:10px;
    classDef core fill:#1e1b4b,stroke:#818cf8,stroke-width:2px,color:#e0e7ff,rx:6px,ry:6px;
    classDef ext fill:#064e3b,stroke:#34d399,stroke-width:2px,color:#ecfdf5,rx:6px,ry:6px;
    classDef sys fill:#701a75,stroke:#f472b6,stroke-width:2px,color:#fdf2f8,rx:6px,ry:6px;
    classDef subStyle fill:#090d16,stroke:#334155,stroke-width:1.5px,stroke-dasharray: 4 4,color:#94a3b8;

    API["fa:fa-network-wired <b>OpenCode HTTP API</b><br/><i>[ 98 Registered Operations ]</i>"]:::gateway

    subgraph SESSIONS[" 💬 Session Lifecycle (15 ops) "]
        direction TB
        S_Ops["• list, create, get, remove, update<br/>• fork, prompt, interrupt, wait<br/>• switchAgent, switchModel<br/>• compact, context, import, export"]:::core
    end

    subgraph RUNTIME[" ⚙️ System & Workspace (18 ops) "]
        direction TB
        R_Ops["• Server: info, pair, connect (3)<br/>• Location: get, reload (2)<br/>• Agents & Plugins: list, get, check, update (5)<br/>• Runtime: shell, pty, file, vcs (8)"]:::sys
    end

    subgraph INTEGRATIONS[" 🔌 Extensibility & Control (34 ops) "]
        direction TB
        I_Ops["• Providers & Models: config, credentials (8)<br/>• Extensions: mcp, plugin, skill (9)<br/>• Knowledge: reference, command (5)<br/>• Interaction: question, form, message (6)<br/>• Pipeline: permission, generation, events (6)"]:::ext
    end

    API ==>|RPC / HTTP REST| SESSIONS
    API ==>|System Control| RUNTIME
    API ==>|Protocol Dispatch| INTEGRATIONS

    class SESSIONS,RUNTIME,INTEGRATIONS subStyle;
```

## 11. Data Persistence: SessionProjector

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '13px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart TB
    classDef memory fill:#1e1b4b,stroke:#818cf8,stroke-width:2px,color:#e0e7ff,rx:6px,ry:6px;
    classDef projector fill:#312e81,stroke:#a78bfa,stroke-width:3px,color:#f5f3ff,rx:8px,ry:8px;
    classDef storage fill:#701a75,stroke:#f472b6,stroke-width:2px,color:#fdf2f8,rx:4px,ry:4px;
    classDef subStyle fill:#090d16,stroke:#334155,stroke-width:1.5px,stroke-dasharray: 4 4,color:#94a3b8;

    subgraph IN_MEMORY[" ⚡ Active In-Memory State "]
        direction TB
        Msg["fa:fa-comment-dots Dynamic Messages"]:::memory
        ToolCalls["fa:fa-terminal Tool Invocations"]:::memory
        State["fa:fa-sliders Session Context & Flags"]:::memory
    end

    Proj["fa:fa-microchip-fork <b>SessionProjector</b><br/><i>(Atomic Commit Engine)</i>"]:::projector

    subgraph PERSISTENCE[" 💾 SQLite Relational Database "]
        direction TB
        DB[("fa:fa-table Sessions Table")]:::storage
        DB2[("fa:fa-table Messages Table")]:::storage
        DB3[("fa:fa-table ToolResults Table")]:::storage
        DB4[("fa:fa-table Compactions Table")]:::storage
    end

    Msg & ToolCalls & State ==>|Batch Events| Proj
    Proj ==>|Synchronous Write Transaction| DB & DB2 & DB3 & DB4

    class IN_MEMORY,PERSISTENCE subStyle;
```

## 12. Tool Execution Pipeline

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '12px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart TB
    classDef contract fill:#312e81,stroke:#a78bfa,stroke-width:2px,color:#f5f3ff,rx:8px,ry:8px;
    classDef tool fill:#1e1b4b,stroke:#818cf8,stroke-width:1.5px,color:#e0e7ff,rx:4px,ry:4px;
    classDef ext fill:#064e3b,stroke:#34d399,stroke-width:1.5px,color:#ecfdf5,rx:4px,ry:4px;
    classDef perm fill:#701a75,stroke:#f472b6,stroke-width:1.5px,color:#fdf2f8,rx:4px,ry:4px;
    classDef subStyle fill:#090d16,stroke:#334155,stroke-width:1.5px,stroke-dasharray: 4 4,color:#94a3b8;

    Make["fa:fa-file-signature <b>Tool.make Contract</b><br/>Codecs • Execution • Projection"]:::contract

    subgraph BUILTINS[" 🛠️ Built-in Capabilities "]
        direction TB
        T1["read"]:::tool
        T2["glob"]:::tool
        T3["grep"]:::tool
        T4["edit"]:::tool
        T5["write"]:::tool
        T6["patch"]:::tool
        T7["shell"]:::tool
        T8["webfetch"]:::tool
        T9["websearch"]:::tool
        T10["question"]:::tool
        T11["skill"]:::tool
        T12["subagent"]:::tool
        T13["execute"]:::tool
    end

    subgraph EXTERNAL[" 🌐 External Tools "]
        direction TB
        M1["fa:fa-server MCP Server Tools"]:::ext
        P1["fa:fa-puzzle-piece Plugin Tools"]:::ext
    end

    subgraph PERMISSIONS[" 🛡️ Enforced Permission Actions "]
        direction TB
        PA1["action: read"]:::perm
        PA2["action: edit / write"]:::perm
        PA3["action: glob / grep"]:::perm
        PA4["action: shell / pty"]:::perm
        PA5["action: subagent"]:::perm
        PA6["action: websearch / fetch"]:::perm
        PA7["action: external_directory"]:::perm
    end

    Make ==> BUILTINS
    Make ==> EXTERNAL

    T1 & T2 & T3 -.-> PA1
    T4 & T5 & T6 -.-> PA2
    T2 & T3 -.-> PA3
    T7 & T13 -.-> PA4
    T12 -.-> PA5
    T8 & T9 -.-> PA6
    T7 -.-> PA7

    class BUILTINS,EXTERNAL,PERMISSIONS subStyle;
```

## 13. Worktree Management

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '13px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart TB
    classDef loc fill:#1e1b4b,stroke:#818cf8,stroke-width:2px,color:#e0e7ff,rx:8px,ry:8px;
    classDef git fill:#064e3b,stroke:#34d399,stroke-width:1.5px,color:#ecfdf5,rx:4px,ry:4px;
    classDef custom fill:#701a75,stroke:#f472b6,stroke-width:1.5px,color:#fdf2f8,rx:4px,ry:4px;
    classDef runtime fill:#1e293b,stroke:#38bdf8,stroke-width:2px,color:#f8fafc,rx:6px,ry:6px;
    classDef subStyle fill:#090d16,stroke:#334155,stroke-width:1.5px,stroke-dasharray: 4 4,color:#94a3b8;

    Location["fa:fa-folder-tree <b>Location Core</b>"]:::loc

    subgraph STRATEGIES[" 🌿 Worktree Strategies "]
        subgraph GIT[" Git Native Strategy (Default) "]
            direction TB
            WT1["fa:fa-plus worktree create"]:::git
            WT2["fa:fa-list worktree list"]:::git
            WT3["fa:fa-trash worktree remove"]:::git
        end
        subgraph PLUGINS[" Custom Extensions "]
            PS1["fa:fa-puzzle-piece Plugin Sandbox Strategy"]:::custom
        end
    end

    subgraph WORKSPACE[" 📂 Execution Workspace "]
        direction TB
        Dir["fa:fa-folder-open Isolated Worktree Dir"]:::runtime
        Session["fa:fa-terminal Session Working Dir"]:::runtime
        Runner["fa:fa-play Session Runner Context"]:::runtime
    end

    Location ==> GIT & PLUGINS
    WT1 & PS1 ==> Dir --> Session --> Runner

    class STRATEGIES,GIT,PLUGINS,WORKSPACE subStyle;
```

## 14. Event Bus Architecture

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '13px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart TB
    classDef source fill:#1e1b4b,stroke:#818cf8,stroke-width:2px,color:#e0e7ff,rx:6px,ry:6px;
    classDef bus fill:#312e81,stroke:#a78bfa,stroke-width:3px,color:#f5f3ff,rx:10px,ry:10px;
    classDef consumer fill:#064e3b,stroke:#34d399,stroke-width:2px,color:#ecfdf5,rx:6px,ry:6px;
    classDef subStyle fill:#090d16,stroke:#334155,stroke-width:1.5px,stroke-dasharray: 4 4,color:#94a3b8;

    subgraph SOURCES[" 📤 Event Producers "]
        direction TB
        S1["fa:fa-table SessionProjector"]:::source
        S2["fa:fa-play SessionRunner"]:::source
        S3["fa:fa-puzzle-piece PluginManager"]:::source
        S4["fa:fa-terminal ToolExecution"]:::source
        S5["fa:fa-brain ModelProvider"]:::source
    end

    Bus["fa:fa-bolt-lightning <b>EventV2 Global Message Bus</b><br/><i>(Publish / Subscribe Core)</i>"]:::bus

    subgraph CONSUMERS[" 📥 Event Consumers "]
        direction TB
        C1[("fa:fa-database SQLite Storage")]:::consumer
        C2["fa:fa-desktop UI Clients"]:::consumer
        C3["fa:fa-plug Plugin Hooks"]:::consumer
        C4["fa:fa-file-lines Audit Log Output"]:::consumer
    end

    S1 & S2 & S3 & S4 & S5 ==>|Dispatch Events| Bus
    Bus ==>|Broadcast Streams| C1 & C2 & C3 & C4

    class SOURCES,CONSUMERS subStyle;
```

---

*Generated from OpenCode V2 source code and documentation analysis.*
