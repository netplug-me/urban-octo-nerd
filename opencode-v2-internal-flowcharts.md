# OpenCode V2 — Internal Flowcharts

Generated from docs + actual source code analysis (`@opencode-ai/plugin`, `opencode-mem` plugin, `session-rotator`, `opencode.jsonc`).

---

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
        TUI["fa:fa-terminal TUI"]:::client
        CLI["fa:fa-square-terminal CLI"]:::client
        Web["fa:fa-globe Web App"]:::client
        SDK["fa:fa-code SDK / External App"]:::client
    end

    subgraph CORE[" ⚡ Background Service (owns everything) "]
        API["fa:fa-network-wired <b>HTTP API Gateway</b><br/><i>[ 98 operations ]</i>"]:::gateway

        subgraph ENGINES[" 🧩 Core Engines "]
            direction TB
            Loc["fa:fa-folder-tree Location Service Graph"]:::service
            Ses["fa:fa-play Session Runner"]:::service
            Proj["fa:fa-database SessionProjector"]:::service
            Comp["fa:fa-compress SessionCompaction"]:::service
            Perm["fa:fa-shield-halved Permissions Engine"]:::service
            Plug["fa:fa-puzzle-piece Plugin System"]:::service
            MCP["fa:fa-server MCP Servers"]:::service
            Tools["fa:fa-toolbox Tool Registry"]:::service
            Agents["fa:fa-robot Agent Registry"]:::service
            Models["fa:fa-brain Model Catalog"]:::service
        end
    end

    TUI & CLI & Web & SDK ==>|JSON-RPC / HTTP| API

    API --> Loc & Ses & Perm & Plug & MCP & Tools & Agents & Models
    Ses --> Proj & Tools & Comp
    Loc --> Plug & Tools

    class CLIENTS,CORE,ENGINES subStyle;
```

---

## 2. Session Execution Flow (the main loop)

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '12px', 'darkMode': true, 'actorBkg': '#1e1b4b', 'actorBorder': '#818cf8', 'actorTextColor': '#e0e7ff', 'signalColor': '#94a3b8', 'signalTextColor': '#cbd5e1', 'noteBkgColor': '#1e293b', 'noteTextColor': '#38bdf8', 'noteBorderColor': '#0284c7'}}}%%
sequenceDiagram
    autonumber
    actor U as 👤 User
    participant S as ⚡ SessionRunner
    participant H as 🪝 Hooks Pipeline
    participant M as 🧠 Model (LLM)
    participant T as 🛠️ Tool
    participant P as 💾 SessionProjector
    participant DB as 🗄️ SQLite

    U->>S: prompt(text)
    S->>H: hook('prompt', event)
    H-->>S: event enriched with context
    S->>M: LLMClient.stream(request)

    alt Tool Call Invocations
        M-->>S: tool_use delta
        S->>T: execute(tool, args)
        T-->>S: tool result
        S->>H: hook('generate', event)
        S->>M: continue streaming with tool result
    else Final Text Completion
        M-->>S: text delta complete
        S->>P: project message state
        P->>DB: atomic SQLite commit
        S-->>U: streamed response text
    end
```

**Hook order (from docs + `opencode-mem` adapter):**

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '12px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart LR
    classDef hook fill:#1e1b4b,stroke:#818cf8,stroke-width:2px,color:#e0e7ff,rx:6px,ry:6px;
    classDef branch fill:#064e3b,stroke:#34d399,stroke-width:1.5px,color:#ecfdf5,rx:4px,ry:4px;

    A["fa:fa-terminal prompt"]:::hook --> B["fa:fa-book-open context"]:::hook
    B --> C["fa:fa-microchip generate"]:::hook
    C --> D["fa:fa-compress compaction"]:::hook

    A -.-> E["fa:fa-tag title"]:::branch
    C -.-> F["fa:fa-rotate-right retry"]:::branch
    D -.-> G["fa:fa-circle-stop session.end"]:::branch
```

---

## 3. Plugin Transform & Hook Pipeline

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '12px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart TB
    classDef root fill:#312e81,stroke:#a78bfa,stroke-width:2px,color:#f5f3ff,rx:8px,ry:8px;
    classDef transform fill:#064e3b,stroke:#34d399,stroke-width:1.5px,color:#ecfdf5,rx:4px,ry:4px;
    classDef hook fill:#701a75,stroke:#f472b6,stroke-width:1.5px,color:#fdf2f8,rx:4px,ry:4px;
    classDef subStyle fill:#090d16,stroke:#334155,stroke-width:1.5px,stroke-dasharray: 4 4,color:#94a3b8;

    Plug["fa:fa-puzzle-piece <b>Plugin Core Container</b>"]:::root

    subgraph TRANSFORMS[" 🔄 Transforms (Synchronous, Ordered at Load Time) "]
        direction TB
        T1["1. provider.transform"]:::transform --> T2["2. model.transform"]:::transform
        T2 --> T3["3. agent.transform"]:::transform --> T4["4. tool.transform"]:::transform
        T4 --> T5["5. command.transform"]:::transform --> T6["6. skill.transform"]:::transform
        T6 --> T7["7. mcp.transform"]:::transform --> T8["8. reference.transform"]:::transform
        T8 --> T9["9. worktree.transform"]:::transform --> T10["10. websearch.transform"]:::transform
        T10 --> T11["11. permission.rules"]:::transform --> T12["12. integration.transform"]:::transform
        T12 --> T13["13. vcs.transform"]:::transform --> T14["14. plugin.list"]:::transform
    end

    subgraph HOOKS[" ⚡ Hooks (Live, Per-Session Runtime) "]
        direction TB
        H1["session.hook('prompt')"]:::hook
        H2["session.hook('context')"]:::hook
        H3["session.hook('compaction')"]:::hook
        H4["session.hook('generate')"]:::hook
        H5["session.hook('title')"]:::hook
        H6["session.hook('model.request')"]:::hook
        H7["session.hook('http.request')"]:::hook
        H8["session.hook('http.response')"]:::hook
        H9["session.hook('experimental.ws.handshake')"]:::hook
        H10["session.hook('retry')"]:::hook
    end

    Plug ==>|Registers On-Load| TRANSFORMS
    Plug ==>|Binds At-Runtime| HOOKS

    class TRANSFORMS,HOOKS subStyle;
```

**Real plugin example (opencode-mem adapter):**

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '12px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart TB
    classDef plugin fill:#1e1b4b,stroke:#818cf8,stroke-width:2px,color:#e0e7ff,rx:6px,ry:6px;
    classDef adapter fill:#064e3b,stroke:#34d399,stroke-width:2px,color:#ecfdf5,rx:6px,ry:6px;
    classDef subStyle fill:#090d16,stroke:#334155,stroke-width:1.5px,stroke-dasharray: 4 4,color:#94a3b8;

    subgraph PLUGIN_ENTRY[" 📦 opencode-mem/src/v2/plugin.ts "]
        direction TB
        P1["loadOpencodeProvider()"]:::plugin
        P2["registerV2Adapter()"]:::plugin
        P3["createLegacyClient()"]:::plugin
    end

    subgraph ADAPTER[" 🔌 adapter.ts (Registers on ctx) "]
        direction TB
        A1["tool.transform → Add 'memory' tool"]:::adapter
        A2["session.hook('prompt') → Forward to legacy"]:::adapter
        A3["session.hook('context') → Inject memory context"]:::adapter
        A4["ctx.event.subscribe → Bridge events to legacy"]:::adapter
    end

    P1 --> A1
    P3 --> A2 & A3
    A4 --> P3

    class PLUGIN_ENTRY,ADAPTER subStyle;
```

---

## 4. Permission Resolution

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '12px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart TB
    classDef cfg fill:#1e1b4b,stroke:#818cf8,stroke-width:1.5px,color:#e0e7ff,rx:4px,ry:4px;
    classDef arbiter fill:#312e81,stroke:#a78bfa,stroke-width:2px,color:#f5f3ff,rx:6px,ry:6px;
    classDef action fill:#1e293b,stroke:#38bdf8,stroke-width:1.5px,color:#f8fafc,rx:4px,ry:4px;
    classDef allow fill:#064e3b,stroke:#34d399,stroke-width:2px,color:#ecfdf5,rx:4px,ry:4px;
    classDef ask fill:#78350f,stroke:#f59e0b,stroke-width:2px,color:#fffbeb,rx:4px,ry:4px;
    classDef deny fill:#450a0a,stroke:#f87171,stroke-width:2px,color:#fef2f2,rx:4px,ry:4px;
    classDef userCfg fill:#701a75,stroke:#f472b6,stroke-width:2px,color:#fdf2f8,rx:6px,ry:6px;
    classDef subStyle fill:#090d16,stroke:#334155,stroke-width:1.5px,stroke-dasharray: 4 4,color:#94a3b8;

    subgraph CONFIG_LAYERS[" ⚙️ Config Layers (Merged in Order) "]
        direction TB
        G["1. Global Permissions"]:::cfg
        P["2. Project Permissions"]:::cfg
        A["3. Agent-Specific Permissions"]:::cfg
        SA["4. Saved User Approvals"]:::cfg
    end

    subgraph LOCAL_OVERRIDE[" 📝 Local Project Override "]
        YourPerm["fa:fa-file-code <b>opencode.jsonc Config</b><br/>• build: subagent*, edit*, shell*, read* → allow<br/>• plan: edit → ask, shell/read → allow<br/>• explore: glob/grep/read → allow, edit/shell/exec/subagent → deny"]:::userCfg
    end

    LastMatch{"fa:fa-scale-balanced <b>Resolution Engine</b><br/><i>(Last Matching Rule Wins)</i>"}:::arbiter

    subgraph ACTIONS[" 🎯 Permission Actions "]
        direction TB
        R1["read"]:::action
        R2["edit"]:::action
        R5["shell"]:::action
        R11["external_directory"]:::action
        ROther["glob, grep, subagent, skill, question, webfetch, websearch, execute"]:::action
    end

    subgraph EFFECTS[" 🚦 Enforced Outcome "]
        AL["fa:fa-check <b>allow</b>"]:::allow
        ASK["fa:fa-question <b>ask</b>"]:::ask
        DK["fa:fa-ban <b>deny</b>"]:::deny
    end

    G & P & A & SA --> LastMatch
    YourPerm ==> LastMatch

    LastMatch --> ACTIONS
    R1 & R2 & R5 --> AL
    R11 --> ASK
    ROther -.-> AL & ASK & DK

    class CONFIG_LAYERS,LOCAL_OVERRIDE,ACTIONS,EFFECTS subStyle;
```

---

## 5. Agent Types & Hierarchy

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '12px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart TB
    classDef primary fill:#1e1b4b,stroke:#818cf8,stroke-width:2px,color:#e0e7ff,rx:8px,ry:8px;
    classDef subagent fill:#064e3b,stroke:#34d399,stroke-width:2px,color:#ecfdf5,rx:6px,ry:6px;
    classDef system fill:#334155,stroke:#94a3b8,stroke-width:2px,color:#f8fafc,rx:6px,ry:6px;
    classDef subStyle fill:#090d16,stroke:#334155,stroke-width:1.5px,stroke-dasharray: 4 4,color:#94a3b8;

    subgraph PRIMARY[" 🧑‍💻 Primary Agents (Drive Conversation) "]
        Build["fa:fa-hammer <b>Build Agent</b><br/>Qwen3.6-35B"]:::primary
        Plan["fa:fa-compass <b>Plan Agent</b><br/>Qwen3.8-27B"]:::primary
    end

    subgraph WORKERS[" 🤖 Subagent Workers (Spawned by Primary) "]
        direction TB
        General["fa:fa-briefcase <b>General Subagent</b><br/>ornith-1.5:9b<br/><i>Multi-step research/worker</i>"]:::subagent
        Explore["fa:fa-magnifying-glass <b>Explore Subagent</b><br/>gemma-4-E2B<br/><i>Read-only glob/grep</i>"]:::subagent
        Scout["fa:fa-bolt <b>Scout Subagent</b><br/>qwen3.5-4b<br/><i>Shell-based search</i>"]:::subagent
    end

    subgraph SYSTEM[" ⚙️ Hidden System Agents "]
        direction TB
        Compaction["fa:fa-compress <b>Compaction Agent</b><br/>gemma-4-E2B<br/><i>Context summarization</i>"]:::system
        Title["fa:fa-pen <b>Title Agent</b><br/>qwen3.5-4b<br/><i>Session naming</i>"]:::system
        Summary["fa:fa-book <b>Summary Agent</b><br/><i>Auto-generated timeline</i>"]:::system
    end

    Build ==>|Spawns| General & Explore & Scout
    Plan ==>|Spawns| Explore & Scout

    Compaction -.->|Context Auto-Trigger| Build
    Title -.->|Session End Hook| Build

    class PRIMARY,WORKERS,SYSTEM subStyle;
```

---

## 6. Subagent Execution Model

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '12px', 'darkMode': true, 'actorBkg': '#1e1b4b', 'actorBorder': '#818cf8', 'actorTextColor': '#e0e7ff', 'signalColor': '#94a3b8', 'signalTextColor': '#cbd5e1', 'noteBkgColor': '#1e293b', 'noteTextColor': '#38bdf8', 'noteBorderColor': '#0284c7'}}}%%
sequenceDiagram
    autonumber
    participant P as 🧑‍💻 Parent (Build)
    participant SR as ⚡ SessionRunner
    participant C as 🤖 Child Session
    participant M as 🧠 Child's Model

    P->>SR: subagent(agent="explore")
    activate SR
    SR->>C: create({ parentID: parent.id })
    C-->>SR: childID
    deactivate SR

    alt Foreground Execution (Default)
        SR->>+C: prompt(task)
        C->>M: generate
        M-->>C: result tokens
        C-->>-SR: return result payload
        SR-->>P: subagent aggregate output
    else Background Execution
        SR->>C: prompt(task)
        par Async Execution
            C->>M: generate
            M-->>C: result tokens
        and Parent Flow
            SR-->>P: "started" acknowledgment
        end
        C-->>SR: complete (notify EventBus)
        SR-->>P: "done" notification
    end

    Note over C,M: Child maintains fresh context window, independent permissions, and model overrides
```

---

## 7. Context Compaction

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '12px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart TB
    classDef detect fill:#1e293b,stroke:#38bdf8,stroke-width:2px,color:#f8fafc,rx:6px,ry:6px;
    classDef decision fill:#312e81,stroke:#a78bfa,stroke-width:2px,color:#f5f3ff,rx:6px,ry:6px;
    classDef action fill:#064e3b,stroke:#34d399,stroke-width:2px,color:#ecfdf5,rx:4px,ry:4px;
    classDef strategy fill:#701a75,stroke:#f472b6,stroke-width:2px,color:#fdf2f8,rx:4px,ry:4px;
    classDef subStyle fill:#090d16,stroke:#334155,stroke-width:1.5px,stroke-dasharray: 4 4,color:#94a3b8;

    subgraph DETECTION[" 🔍 Detection Triggers "]
        direction TB
        D1["fa:fa-gauge Token Count vs Model Limit"]:::detect
        D2["fa:fa-scissors Turn Splitting"]:::detect
        D3["fa:fa-filter Output Pruning"]:::detect
    end

    subgraph FLOW[" ⚡ Compaction Lifecycle "]
        Start["Session Memory Grows"]:::detect --> Check{Context Overflow?}:::decision
        Check -->|Yes| Split["Split turn & prune large tool payloads"]:::action
        Split --> Compress["Compress prior message history"]:::action
        Compress --> Marker["Insert @b1@ compaction marker"]:::action
        Marker --> Continue["Resume normal session loop"]:::action
        Check -->|No| Continue
    end

    subgraph STRATEGIES[" 📦 Execution Strategy "]
        direction TB
        M["fa:fa-shield-halved Keep Limit: 15,999 tokens<br/><i>(opencode.jsonc)</i>"]:::strategy
        S["fa:fa-file-lines Local Summary via Compaction Agent"]:::strategy
        N["fa:fa-server Native Provider API Compaction"]:::strategy
    end

    D1 & D2 & D3 ==> Check
    Compress ==> S & N
    M -.->|Threshold Constraint| Compress

    class DETECTION,FLOW,STRATEGIES subStyle;
```

---

## 8. Instruction Resolution Order

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '12px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart TB
    classDef promptLayer fill:#1e1b4b,stroke:#818cf8,stroke-width:2px,color:#e0e7ff,rx:6px,ry:6px;
    classDef noteNode fill:#1e293b,stroke:#38bdf8,stroke-width:1.5px,stroke-dasharray: 4 4,color:#e2e8f0,rx:6px,ry:6px;
    classDef subStyle fill:#090d16,stroke:#334155,stroke-width:1.5px,stroke-dasharray: 4 4,color:#94a3b8;

    subgraph PROMPT_LAYERS[" 📜 System Prompt Layers (Priority Order) "]
        direction TB
        L1["1. Agent / Provider System Prompt"]:::promptLayer
        L2["2. Built-in Context"]:::promptLayer
        L3["3. Code Mode Guidance"]:::promptLayer
        L4["4. AGENTS.md Files (Workspace → Home)"]:::promptLayer
        L5["5. Skill / Reference / MCP Guidance"]:::promptLayer
        L6["6. Session Instructions (Top Precedence)"]:::promptLayer

        L1 --> L2 --> L3 --> L4 --> L5 --> L6
    end

    N1["fa:fa-circle-info <b>Build Agent System Override:</b><br/><i>'Be succinct and efficient. Delegate all exploration...'</i>"]:::noteNode

    L1 -.->|Configured in opencode.jsonc| N1

    class PROMPT_LAYERS subStyle;
```

---

## 9. Location Service Graph

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '12px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart TB
    classDef root fill:#312e81,stroke:#a78bfa,stroke-width:3px,color:#f5f3ff,rx:10px,ry:10px;
    classDef item fill:#1e1b4b,stroke:#818cf8,stroke-width:1.5px,color:#e0e7ff,rx:4px,ry:4px;
    classDef subStyle fill:#090d16,stroke:#334155,stroke-width:1.5px,stroke-dasharray: 4 4,color:#94a3b8;

    subgraph GRAPH[" 🌐 Per-Location Service Graph "]
        direction TB
        FS["Filesystem"]:::item
        VCS["VCS Engine"]:::item
        AG["Agents"]:::item
        MD["Models"]:::item
        PR["Providers"]:::item
        TL["Tools"]:::item
        PM["Permissions"]:::item
        PL["Plugins"]:::item
        MC["MCP"]:::item
        IN["Instructions"]:::item
        SK["Skills"]:::item
        CM["Commands"]:::item
        RE["References"]:::item
        PT["PTY"]:::item
        RU["Runner"]:::item
    end

    Loc["fa:fa-folder-tree <b>Location.Ref</b><br/><i>(Project Root Anchor)</i>"]:::root

    %% Dependency Wiring
    VCS --> FS
    FS --> TL
    PL --> TL & PM & IN
    AG --> MD & PM
    MC --> TL
    PM --> TL

    FS & VCS & AG & MD & PR & TL & PM & PL & MC & IN & SK & CM & RE & PT & RU ==> Loc

    class GRAPH subStyle;
```

---

## 10. HTTP API Surface (98 operations)

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '12px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart TB
    classDef gateway fill:#312e81,stroke:#a78bfa,stroke-width:3px,color:#f5f3ff,rx:10px,ry:10px;
    classDef core fill:#1e1b4b,stroke:#818cf8,stroke-width:2px,color:#e0e7ff,rx:6px,ry:6px;
    classDef ext fill:#064e3b,stroke:#34d399,stroke-width:2px,color:#ecfdf5,rx:6px,ry:6px;
    classDef sys fill:#701a75,stroke:#f472b6,stroke-width:2px,color:#fdf2f8,rx:6px,ry:6px;
    classDef subStyle fill:#090d16,stroke:#334155,stroke-width:1.5px,stroke-dasharray: 4 4,color:#94a3b8;

    API["fa:fa-network-wired <b>OpenCode HTTP API Gateway</b><br/><i>[ 98 Registered Operations ]</i>"]:::gateway

    subgraph SESSIONS[" 💬 Session Lifecycle (15 Operations) "]
        direction TB
        S_Ops["• list, create, get, remove, update<br/>• fork, prompt, interrupt, wait<br/>• switchAgent, switchModel<br/>• compact, context, import, export"]:::core
    end

    subgraph RUNTIME[" ⚙️ System & Workspace (18 Operations) "]
        direction TB
        R_Ops["• Server: info, pair, connect (3)<br/>• Location: get, reload (2)<br/>• Agents & Plugins: list, get, check, update (5)<br/>• Runtime: shell, pty, file, vcs (8)"]:::sys
    end

    subgraph INTEGRATIONS[" 🔌 Extensibility & Control (34 Operations) "]
        direction TB
        I_Ops["• Providers & Models: config, credentials (8)<br/>• Extensions: mcp, plugin, skill (9)<br/>• Knowledge: reference, command (5)<br/>• Interaction: question, form, message (6)<br/>• Pipeline: permission, generation, events (6)"]:::ext
    end

    API ==>|Session Control| SESSIONS
    API ==>|System Control| RUNTIME
    API ==>|Extension Control| INTEGRATIONS

    class SESSIONS,RUNTIME,INTEGRATIONS subStyle;
```

---

## 11. SessionProjector Data Persistence

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '12px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart TB
    classDef mem fill:#1e1b4b,stroke:#818cf8,stroke-width:2px,color:#e0e7ff,rx:6px,ry:6px;
    classDef proj fill:#312e81,stroke:#a78bfa,stroke-width:3px,color:#f5f3ff,rx:8px,ry:8px;
    classDef storage fill:#701a75,stroke:#f472b6,stroke-width:2px,color:#fdf2f8,rx:4px,ry:4px;
    classDef noteNode fill:#1e293b,stroke:#38bdf8,stroke-width:1.5px,stroke-dasharray: 4 4,color:#e2e8f0,rx:6px,ry:6px;
    classDef subStyle fill:#090d16,stroke:#334155,stroke-width:1.5px,stroke-dasharray: 4 4,color:#94a3b8;

    subgraph IN_MEMORY[" ⚡ In-Memory State "]
        direction TB
        Msg["Dynamic Messages"]:::mem
        TC["Tool Calls"]:::mem
        TR["Tool Results"]:::mem
        St["Session State"]:::mem
    end

    Proj["fa:fa-microchip-fork <b>SessionProjector</b>"]:::proj

    subgraph SQLITE[" 💾 SQLite Database "]
        direction TB
        MsgDB[("messages Table")]:::storage
        ToolDB[("tools Table")]:::storage
        SessionDB[("sessions Table")]:::storage
    end

    N1["fa:fa-shield-halved <b>Durable Event Log:</b><br/>All state changes projected and committed atomically"]:::noteNode

    Msg & TC & TR & St ==>|Stream Events| Proj
    Proj ==>|ACID Writes| MsgDB & ToolDB & SessionDB
    Proj -.-> N1

    class IN_MEMORY,SQLITE subStyle;
```

---

## 12. Tool Execution Pipeline

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '12px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart TB
    classDef contract fill:#312e81,stroke:#a78bfa,stroke-width:2px,color:#f5f3ff,rx:8px,ry:8px;
    classDef tool fill:#1e1b4b,stroke:#818cf8,stroke-width:1.5px,color:#e0e7ff,rx:4px,ry:4px;
    classDef ext fill:#064e3b,stroke:#34d399,stroke-width:1.5px,color:#ecfdf5,rx:4px,ry:4px;
    classDef perm fill:#701a75,stroke:#f472b6,stroke-width:1.5px,color:#fdf2f8,rx:4px,ry:4px;
    classDef subStyle fill:#090d16,stroke:#334155,stroke-width:1.5px,stroke-dasharray: 4 4,color:#94a3b8;

    subgraph PIPELINE[" ⚙️ Tool.make Contract "]
        Codec["fa:fa-file-code Codec (Serialize / Deserialize)"]:::contract
        Exec["fa:fa-play Execute (Tool Logic)"]:::contract
        Proj["fa:fa-database Projection (Model State)"]:::contract
        Codec --> Exec --> Proj
    end

    subgraph BUILTINS[" 🛠️ Built-in Tools "]
        direction TB
        T1["read, glob, grep"]:::tool
        T2["edit, write, patch"]:::tool
        T3["shell, execute"]:::tool
        T4["webfetch, websearch"]:::tool
        T5["question, skill, subagent"]:::tool
    end

    subgraph EXTERNAL[" 🌐 External Tools "]
        direction TB
        MC["MCP Server Tools"]:::ext
        PL["Plugin Tools"]:::ext
    end

    subgraph PERMISSIONS[" 🛡️ Permission Gating "]
        direction TB
        P_FS["read / edit / write → paths"]:::perm
        P_Search["glob → pattern, grep → regex"]:::perm
        P_Exec["shell / execute → command string"]:::perm
        P_Net["webfetch → URL, websearch → query"]:::perm
        P_Delegation["subagent → agent ID, skill → skill ID"]:::perm
    end

    BUILTINS & EXTERNAL ==> Codec
    Exec ==> PERMISSIONS

    class PIPELINE,BUILTINS,EXTERNAL,PERMISSIONS subStyle;
```

---

## 13. Worktree Management

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '12px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart TB
    classDef strategy fill:#1e1b4b,stroke:#818cf8,stroke-width:2px,color:#e0e7ff,rx:6px,ry:6px;
    classDef op fill:#064e3b,stroke:#34d399,stroke-width:2px,color:#ecfdf5,rx:6px,ry:6px;
    classDef noteNode fill:#1e293b,stroke:#38bdf8,stroke-width:1.5px,stroke-dasharray: 4 4,color:#e2e8f0,rx:6px,ry:6px;
    classDef subStyle fill:#090d16,stroke:#334155,stroke-width:1.5px,stroke-dasharray: 4 4,color:#94a3b8;

    subgraph STRATEGIES[" 🌿 Worktree Strategies "]
        direction TB
        Git["fa:fa-code-branch Git Native Strategy (Default)"]:::strategy
        P1["fa:fa-puzzle-piece Custom Plugin Strategy A"]:::strategy
        P2["fa:fa-puzzle-piece Custom Plugin Strategy B"]:::strategy
    end

    subgraph OPERATIONS[" ⚙️ Worktree Operations "]
        direction TB
        C["create"]:::op
        R["remove"]:::op
        L["list"]:::op
        F["refresh"]:::op
    end

    N1["fa:fa-circle-info <b>Strategy-Scoped:</b> Created per-repository worktree directory"]:::noteNode

    Git & P1 & P2 ==> C & R & L & F
    OPERATIONS -.-> N1

    class STRATEGIES,OPERATIONS subStyle;
```

---

## 14. Event Bus Architecture

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '12px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart TB
    classDef bus fill:#312e81,stroke:#a78bfa,stroke-width:3px,color:#f5f3ff,rx:10px,ry:10px;
    classDef src fill:#1e1b4b,stroke:#818cf8,stroke-width:1.5px,color:#e0e7ff,rx:4px,ry:4px;
    classDef consumer fill:#064e3b,stroke:#34d399,stroke-width:1.5px,color:#ecfdf5,rx:4px,ry:4px;
    classDef noteNode fill:#1e293b,stroke:#38bdf8,stroke-width:1.5px,stroke-dasharray: 4 4,color:#e2e8f0,rx:6px,ry:6px;
    classDef subStyle fill:#090d16,stroke:#334155,stroke-width:1.5px,stroke-dasharray: 4 4,color:#94a3b8;

    EV["fa:fa-bolt-lightning <b>EventV2 Global Message Bus</b><br/><i>(Publish / Subscribe Core)</i>"]:::bus

    subgraph SOURCES[" 📤 Event Sources "]
        direction TB
        ES1["session.created / updated"]:::src
        ES2["session.execution.succeeded / failed"]:::src
        ES3["session.compaction.ended"]:::src
        ES4["session.prompt.sent"]:::src
        ES5["message.created / updated"]:::src
    end

    subgraph CONSUMERS[" 📥 Event Consumers "]
        direction TB
        EC1["opencode-mem plugin"]:::consumer
        EC2["session-rotator plugin"]:::consumer
        EC3["DCP plugin"]:::consumer
        EC4["External UI clients"]:::consumer
    end

    N1["fa:fa-plug <b>Legacy Bridge Note:</b><br/>opencode-mem uses ctx.event.subscribe() and maps v2 events to v1 via toLegacyEvent()"]:::noteNode

    SOURCES ==>|Publish Events| EV
    EV ==>|Broadcast Stream| CONSUMERS
    EC1 -.-> N1

    class SOURCES,CONSUMERS subStyle;
```

---

## 15. Real Plugin Flow — opencode-mem

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '12px', 'darkMode': true, 'actorBkg': '#1e1b4b', 'actorBorder': '#818cf8', 'actorTextColor': '#e0e7ff', 'signalColor': '#94a3b8', 'signalTextColor': '#cbd5e1', 'noteBkgColor': '#1e293b', 'noteTextColor': '#38bdf8', 'noteBorderColor': '#0284c7'}}}%%
sequenceDiagram
    autonumber
    participant OC as ⚡ OpenCode Service
    participant PL as 🧩 opencode-mem plugin
    participant LM as 💾 Legacy Memory System

    critical Setup & Registration
        OC->>PL: setup(ctx)
        PL->>PL: createLegacyClient(ctx)
        PL->>PL: registerV2Adapter(ctx, legacy)
        PL->>PL: tool.transform → add 'memory' tool
        PL->>PL: session.hook('prompt') → forward to legacy
        PL->>PL: session.hook('context') → inject memory context
        PL->>PL: ctx.event.subscribe → bridge events
    end

    alt Tool Execution Flow
        OC->>PL: user calls memory tool
        PL->>LM: legacy.tool.memory.execute(args)
        LM-->>PL: tool result
        PL-->>OC: { content, metadata }
    else Prompt Context Enrichment
        OC->>PL: session.hook('prompt', event)
        PL->>LM: chat.message({ sessionID, output })
        LM-->>PL: injected parts
        PL->>PL: store injected context
        OC->>PL: session.hook('context', event)
        PL->>PL: push injected context to event.system
    end
```

---

## 16. Real Plugin Flow — session-rotator (disabled)

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '12px', 'darkMode': true, 'actorBkg': '#1e1b4b', 'actorBorder': '#818cf8', 'actorTextColor': '#e0e7ff', 'signalColor': '#94a3b8', 'signalTextColor': '#cbd5e1', 'noteBkgColor': '#1e293b', 'noteTextColor': '#38bdf8', 'noteBorderColor': '#0284c7'}}}%%
sequenceDiagram
    autonumber
    participant OC as ⚡ OpenCode Service
    participant SR as 🔄 session-rotator plugin
    participant N as 🆕 New Session

    OC->>SR: setup(ctx)
    SR->>SR: Initialize config (N=3 turns, SKIP_AGENTS={explore, scout})

    OC->>SR: session.hook('context', event)
    SR->>SR: Increment turn count for session
    alt Turn Threshold Reached (n % N == 0)
        SR->>OC: ctx.session.create({ parentID, title })
        OC-->>SR: newSessionID
        SR->>OC: ctx.session.prompt({ sessionID: newSessionID, text })
        SR->>SR: event.system.push(ROTATED message)
    end

    OC->>SR: session.hook('prompt', event)
    opt Summary Detected
        SR->>SR: If prompt starts with '## Session Summary' → reset count to 0
    end
```

---

## 17. Your Current Agent Permission Matrix

| Agent | Mode | Model | read | edit | shell | glob | grep | subagent | execute |
|---|---|---|---|---|---|---|---|---|---|
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
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '12px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart LR
    classDef base fill:#1e1b4b,stroke:#818cf8,stroke-width:1.5px,color:#e0e7ff,rx:6px,ry:6px;
    classDef priority fill:#701a75,stroke:#f472b6,stroke-width:2px,color:#fdf2f8,rx:6px,ry:6px;
    classDef noteNode fill:#1e293b,stroke:#38bdf8,stroke-width:1.5px,stroke-dasharray: 4 4,color:#e2e8f0,rx:6px,ry:6px;

    A["fa:fa-house Global<br/><i>~/.config/...</i>"]:::base --> B["fa:fa-folder Project<br/><i>project/...</i>"]:::base
    B --> C["fa:fa-folder-gear Local<br/><i>.opencode/...</i>"]:::base
    C --> D["fa:fa-puzzle-piece Plugins<br/><i>Synchronous</i>"]:::base
    D --> E["fa:fa-robot Agent Overrides"]:::base
    E --> F["fa:fa-user-check Saved Approvals<br/><i>Highest Priority</i>"]:::priority

    N1["fa:fa-scale-balanced <b>Resolution Rule:</b> Config merged ancestor → current directory. Last matching permission rule wins."]:::noteNode

    F -.-> N1
```

---

## 19. Prompt → Response Lifecycle

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '12px', 'darkMode': true, 'actorBkg': '#1e1b4b', 'actorBorder': '#818cf8', 'actorTextColor': '#e0e7ff', 'signalColor': '#94a3b8', 'signalTextColor': '#cbd5e1', 'noteBkgColor': '#1e293b', 'noteTextColor': '#38bdf8', 'noteBorderColor': '#0284c7'}}}%%
sequenceDiagram
    autonumber
    actor U as 👤 User
    participant T as 💻 TUI / CLI
    participant S as ⚡ SessionRunner
    participant A as 🤖 Agent (Build)
    participant M as 🧠 Model
    participant TL as 🛠️ Tools

    U->>T: Type & submit prompt
    T->>S: session.prompt(text)
    S->>A: Inject system prompt & assembled context
    A->>+M: Stream request
    alt Standard Text Response
        M-->>A: Text delta chunks
        A-->>T: Stream directly to terminal UI
    else Tool Call Interruption
        M-->>A: tool_call event
        A->>TL: execute(tool, args)
        TL-->>A: tool result
        A->>M: Continue streaming with tool response
    end
    deactivate M
    A->>S: Project state delta to SQLite
    S->>T: Session update complete
```

---

## 20. Model Provider Resolution

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'fontFamily': 'Fira Code, monospace', 'fontSize': '12px', 'darkMode': true, 'background': '#0f172a'}}}%%
flowchart TB
    classDef provider fill:#1e1b4b,stroke:#818cf8,stroke-width:1.5px,color:#e0e7ff,rx:6px,ry:6px;
    classDef catalog fill:#312e81,stroke:#a78bfa,stroke-width:3px,color:#f5f3ff,rx:8px,ry:8px;
    classDef agent fill:#064e3b,stroke:#34d399,stroke-width:1.5px,color:#ecfdf5,rx:4px,ry:4px;
    classDef noteNode fill:#1e293b,stroke:#38bdf8,stroke-width:1.5px,stroke-dasharray: 4 4,color:#e2e8f0,rx:6px,ry:6px;
    classDef subStyle fill:#090d16,stroke:#334155,stroke-width:1.5px,stroke-dasharray: 4 4,color:#94a3b8;

    subgraph PROVIDERS[" ☁️ Defined Providers (opencode.jsonc) "]
        direction TB
        P1["nerdbook-pro<br/><i>Qwen3.6, Qwen3.8, Qwen3.5</i>"]:::provider
        P2["nerdbook-air<br/><i>gemma-4-12B, gemma-4-E2B</i>"]:::provider
        P3["serenity<br/><i>glm-4.6v-flash</i>"]:::provider
        P4["enterprise<br/><i>ornith-1.5, qwen3.5:4b</i>"]:::provider
        P5["nerdland<br/><i>gemma4:e2b, qwen3.5-4b</i>"]:::provider
    end

    Cat["fa:fa-book-atlas <b>Unified Model Catalog</b><br/><i>(OpenAI-Compatible Abstraction)</i>"]:::catalog

    subgraph AGENTS[" 🤖 Agent Model Routing "]
        direction TB
        A1["build → nerdbook-pro / Qwen3.6"]:::agent
        A2["plan → nerdbook-pro / Qwen3.8"]:::agent
        A3["general → enterprise / ornith-1.5"]:::agent
        A4["compaction → nerdbook-air / gemma-4-E2B"]:::agent
        A5["title → nerdland / qwen3.5-4b"]:::agent
        A6["explore → nerdbook-air / gemma-4-E2B"]:::agent
        A7["scout → nerdland / qwen3.5-4b"]:::agent
    end

    N1["fa:fa-sliders <b>Runtime Overrides:</b> Each agent can override the provider, context window, and temperature."]:::noteNode

    P1 & P2 & P3 & P4 & P5 ==> Cat
    Cat ==> A1 & A2 & A3 & A4 & A5 & A6 & A7
    Cat -.-> N1

    class PROVIDERS,AGENTS subStyle;
```
