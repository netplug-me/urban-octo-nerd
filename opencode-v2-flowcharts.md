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
    classDef subStyle fill:#090d16,stroke:#334155,stroke-width:1
