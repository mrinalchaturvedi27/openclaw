# OpenClaw Architecture Diagrams

This document contains detailed architectural diagrams for the OpenClaw system.

## System Architecture Overview

```mermaid
graph TB
    subgraph "External Channels"
        TG[Telegram]
        DC[Discord]
        SL[Slack]
        WA[WhatsApp]
        SG[Signal]
        IM[iMessage]
    end
    
    subgraph "Channel Layer"
        CNorm[Channel Normalization]
        TG --> CNorm
        DC --> CNorm
        SL --> CNorm
        WA --> CNorm
        SG --> CNorm
        IM --> CNorm
    end
    
    subgraph "Gateway Core"
        Route[Routing & Dispatch]
        Auth[Authorization]
        Session[Session Manager]
        CNorm --> Route
        Route --> Auth
        Auth --> Session
    end
    
    subgraph "Gateway Server"
        WS[WebSocket Server<br/>:18789]
        HTTP[HTTP Server<br/>Control UI]
        Cron[Cron Scheduler]
        Session --> WS
        Session --> HTTP
        Session --> Cron
    end
    
    subgraph "AI Layer"
        Agent[Pi Agent Runtime]
        ModelSel[Model Selection]
        LLM[LLM APIs<br/>Anthropic/OpenAI/Google]
        WS --> Agent
        Agent --> ModelSel
        ModelSel --> LLM
    end
    
    subgraph "Tool Layer"
        Tools[Tool Executor]
        Browser[Browser Tool<br/>Playwright]
        Memory[Memory Tool<br/>Vector DB]
        Bash[Bash Tool<br/>Sandboxed]
        Msg[Message Tool]
        LLM --> Tools
        Tools --> Browser
        Tools --> Memory
        Tools --> Bash
        Tools --> Msg
    end
    
    subgraph "Storage"
        Config[Config Files<br/>JSON5]
        Sessions[Session Store<br/>JSONL]
        Creds[Credentials<br/>Keychain/Env]
        Tools --> Sessions
        Agent --> Config
        Agent --> Creds
    end
    
    Msg --> CNorm
```

## Message Flow Sequence

```mermaid
sequenceDiagram
    participant U as User
    participant C as Channel<br/>(Telegram)
    participant G as Gateway
    participant A as Agent Runtime
    participant L as LLM API
    participant T as Tools
    
    U->>C: Send message: "What's the weather?"
    C->>G: Raw message event
    G->>G: Normalize message
    G->>G: Check authorization (DM pairing)
    G->>G: Load session from disk
    G->>A: Dispatch to agent
    A->>L: Call LLM with history + tools
    L->>A: Response with tool call: get_weather
    A->>T: Execute: browser_navigate(weather.com)
    T->>A: Tool result: "45°F, Partly Cloudy"
    A->>L: Send tool result back to LLM
    L->>A: Final response: "It's 45°F and partly cloudy"
    A->>G: Return response
    G->>G: Save to session
    G->>C: Send via channel API
    C->>U: Display response
```

## Gateway Architecture Detail

```mermaid
graph LR
    subgraph "Gateway Process"
        subgraph "WebSocket Layer"
            WSServer[WS Server<br/>ws://0.0.0.0:18789]
            WSHandlers[RPC Handlers]
            WSAuth[Token Auth]
            WSServer --> WSAuth
            WSAuth --> WSHandlers
        end
        
        subgraph "HTTP Layer"
            HTTPServer[HTTP Server]
            ControlUI[Control UI<br/>React SPA]
            API[REST API<br/>/api/*]
            Webhooks[Webhooks<br/>/webhooks/*]
            HTTPServer --> ControlUI
            HTTPServer --> API
            HTTPServer --> Webhooks
        end
        
        subgraph "Core Services"
            ConfigMgr[Config Manager<br/>Hot Reload]
            SessionMgr[Session Manager<br/>JSONL I/O]
            CronMgr[Cron Manager<br/>Scheduled Tasks]
            NodeReg[Node Registry<br/>iOS/Android]
            WSHandlers --> ConfigMgr
            WSHandlers --> SessionMgr
            WSHandlers --> CronMgr
            WSHandlers --> NodeReg
        end
        
        subgraph "Channel Connectors"
            TGBot[Telegram Bot<br/>Grammy]
            DCBot[Discord Bot<br/>Carbon]
            SLBot[Slack App<br/>Bolt]
            WAClient[WhatsApp Client<br/>Baileys]
            SessionMgr --> TGBot
            SessionMgr --> DCBot
            SessionMgr --> SLBot
            SessionMgr --> WAClient
        end
    end
```

## Agent Runtime Components

```mermaid
graph TB
    subgraph "Pi Agent Runtime"
        subgraph "Input Processing"
            Input[Message Input]
            History[Load History]
            Context[Build Context]
            Input --> History
            History --> Context
        end
        
        subgraph "Model Layer"
            Select[Model Selection]
            Auth[Auth Resolver]
            Failover[Failover Logic]
            Context --> Select
            Select --> Auth
            Select --> Failover
        end
        
        subgraph "LLM Integration"
            API[LLM API Client]
            Stream[Stream Handler]
            Parse[Response Parser]
            Auth --> API
            API --> Stream
            Stream --> Parse
        end
        
        subgraph "Tool Execution"
            ToolDec[Tool Decision]
            ToolExec[Tool Executor]
            ToolRes[Result Handler]
            Parse --> ToolDec
            ToolDec --> ToolExec
            ToolExec --> ToolRes
        end
        
        subgraph "Output"
            Format[Format Response]
            Save[Save to Session]
            Deliver[Deliver to Channel]
            ToolRes --> Format
            Format --> Save
            Format --> Deliver
        end
    end
```

## Security Layers

```mermaid
graph TD
    subgraph "Defense in Depth"
        L1[Layer 1: Network<br/>Firewall, TLS]
        L2[Layer 2: Authentication<br/>DM Pairing, Tokens]
        L3[Layer 3: Authorization<br/>Allowlists, RBAC]
        L4[Layer 4: Input Validation<br/>Sanitization, Size Limits]
        L5[Layer 5: Execution<br/>Sandboxing, Approval]
        L6[Layer 6: Data Protection<br/>Encryption, Redaction]
        L7[Layer 7: Monitoring<br/>Logging, Alerting]
        
        L1 --> L2
        L2 --> L3
        L3 --> L4
        L4 --> L5
        L5 --> L6
        L6 --> L7
    end
```

## Data Flow: Session Persistence

```mermaid
graph LR
    subgraph "Session Lifecycle"
        Create[Create Session]
        Load[Load from Disk]
        Update[Append Message]
        Save[Save to Disk]
        
        Create --> Save
        Load --> Update
        Update --> Save
    end
    
    subgraph "Storage Format"
        JSONL[JSONL File<br/>~/.openclaw/sessions/]
        Entry1["{ role: 'user', content: '...', timestamp: ... }"]
        Entry2["{ role: 'assistant', content: '...', timestamp: ... }"]
        Entry3["{ role: 'tool', content: '...', tool_call_id: '...' }"]
        
        JSONL --> Entry1
        JSONL --> Entry2
        JSONL --> Entry3
    end
    
    Save --> JSONL
    Load --> JSONL
```

## Skill/Plugin Architecture

```mermaid
graph TB
    subgraph "Skill Discovery"
        Bundled[Bundled Skills<br/>skills/*]
        NodeModules[npm Packages<br/>node_modules/@openclaw-plugins/*]
        Workspace[Workspace Skills<br/>./openclaw-skills/*]
        
        Bundled --> Loader
        NodeModules --> Loader
        Workspace --> Loader
    end
    
    subgraph "Plugin Loader"
        Loader[Plugin Loader<br/>jiti dynamic import]
        Manifest[Read Manifests<br/>package.json]
        Validate[Validate Schema]
        
        Loader --> Manifest
        Manifest --> Validate
    end
    
    subgraph "Plugin Registry"
        Register[Plugin Registry]
        Tools[Register Tools]
        Hooks[Register Hooks]
        Commands[Register Commands]
        
        Validate --> Register
        Register --> Tools
        Register --> Hooks
        Register --> Commands
    end
    
    subgraph "Runtime"
        Agent[Agent Runtime]
        ToolExec[Tool Executor]
        HookExec[Hook Executor]
        
        Tools --> ToolExec
        Hooks --> HookExec
        ToolExec --> Agent
    end
```

## Multi-Channel Routing

```mermaid
graph TB
    subgraph "Inbound Message"
        Msg[Message from Channel]
        Channel[Channel ID]
        Sender[Sender ID]
        Chat[Chat ID/Type]
        
        Msg --> Channel
        Msg --> Sender
        Msg --> Chat
    end
    
    subgraph "Routing Logic"
        Rules[Routing Rules<br/>openclaw.json]
        Match1{Channel Match?}
        Match2{Sender Match?}
        Match3{Chat Type Match?}
        Default[Default Agent]
        
        Channel --> Match1
        Match1 -->|Yes| Match2
        Match1 -->|No| Default
        
        Sender --> Match2
        Match2 -->|Yes| Match3
        Match2 -->|No| Default
        
        Chat --> Match3
        Match3 -->|Yes| Agent1
        Match3 -->|No| Default
    end
    
    subgraph "Target Agents"
        Agent1[Agent: trading-bot]
        Agent2[Agent: support-bot]
        AgentDef[Agent: default]
        
        Default --> AgentDef
    end
    
    subgraph "Session Resolution"
        SessionKey[Build Session Key<br/>channel:sender:chat]
        SessionStore[Load/Create Session]
        
        Agent1 --> SessionKey
        Agent2 --> SessionKey
        AgentDef --> SessionKey
        SessionKey --> SessionStore
    end
```

## Deployment Topologies

```mermaid
graph TB
    subgraph "Topology 1: Single Machine"
        Local[Local Machine<br/>Mac/Linux]
        Gateway1[Gateway Process]
        Channels1[Channel Connections]
        LLM1[LLM APIs<br/>Cloud]
        
        Local --> Gateway1
        Gateway1 --> Channels1
        Gateway1 --> LLM1
    end
    
    subgraph "Topology 2: Distributed"
        Pi[Raspberry Pi<br/>Gateway Host]
        Mac[macOS<br/>Control UI]
        iOS[iOS Device<br/>Node + Canvas]
        Android[Android Device<br/>Node + Camera]
        
        Pi --> Gateway2[Gateway Process]
        Mac -.WebSocket.-> Gateway2
        iOS -.WebSocket.-> Gateway2
        Android -.WebSocket.-> Gateway2
        Gateway2 --> Channels2[Channels]
        Gateway2 --> LLM2[LLM APIs]
    end
    
    subgraph "Topology 3: Cloud VPS"
        VPS[VPS/Cloud Server]
        Gateway3[Gateway Process<br/>Docker]
        Tailscale[Tailscale Tunnel]
        Firewall[Firewall<br/>SSH Only]
        
        VPS --> Gateway3
        VPS --> Tailscale
        VPS --> Firewall
        Gateway3 --> Channels3[Channels]
        Gateway3 --> LLM3[LLM APIs]
    end
```

---

## ASCII Art Diagrams (for terminals)

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     MESSAGING CHANNELS                          │
│  WhatsApp │ Telegram │ Discord │ Slack │ Signal │ iMessage │... │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                   CHANNEL NORMALIZATION                         │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                ROUTING & AUTHORIZATION                          │
│   • DM Pairing Check                                            │
│   • Allowlist Verification                                      │
│   • Agent Assignment                                            │
│   • Session Key Building                                        │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                   GATEWAY (Control Plane)                       │
│                ws://127.0.0.1:18789                             │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────┐       │
│  │ WS Server    │  │ HTTP Server  │  │ Cron Scheduler │       │
│  │ (RPC)        │  │ (Control UI) │  │ (Tasks)        │       │
│  └──────────────┘  └──────────────┘  └────────────────┘       │
│                                                                  │
│  ┌────────────────────────────────────────────────────┐        │
│  │           Session Manager (JSONL Store)             │        │
│  └────────────────────────────────────────────────────┘        │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                  PI AGENT RUNTIME (RPC Mode)                    │
│                                                                  │
│  ┌────────────────┐  ┌────────────────┐  ┌─────────────────┐  │
│  │ Model Selection│  │ LLM API Client │  │ Tool Executor   │  │
│  │ & Failover     │  │ (Streaming)    │  │ (20+ tools)     │  │
│  └────────────────┘  └────────────────┘  └─────────────────┘  │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                   EXTERNAL SERVICES                             │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │ Anthropic    │  │ OpenAI       │  │ Google       │         │
│  │ Claude API   │  │ GPT-4 API    │  │ Gemini API   │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │ Ollama       │  │ Browser      │  │ Vector DB    │         │
│  │ (Local LLM)  │  │ (Playwright) │  │ (sqlite-vec) │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
```

### Security Layers

```
┌──────────────────────────── SECURITY LAYERS ────────────────────────────┐
│                                                                            │
│  Layer 7: Monitoring                                                      │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ • Structured Logging (tslog)                                      │   │
│  │ • Security Event Audit Trail                                      │   │
│  │ • Anomaly Detection (future)                                      │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  Layer 6: Data Protection                                                 │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ • PII Redaction in Logs                                           │   │
│  │ • Session Encryption (recommended)                                │   │
│  │ • Credential Keychain Storage                                     │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  Layer 5: Execution Control                                               │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ • Exec Approval for Bash Commands                                 │   │
│  │ • Sandboxed Browser (Playwright)                                  │   │
│  │ • Tool Result Size Limits                                         │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  Layer 4: Input Validation                                                │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ • Message Sanitization (null bytes, control chars)                │   │
│  │ • Length Limits (100KB max)                                       │   │
│  │ • Zod Schema Validation                                           │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  Layer 3: Authorization                                                   │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ • Allowlists (channels.{channel}.allowFrom)                       │   │
│  │ • Command Authorization (commands.allowFrom)                      │   │
│  │ • Session Isolation (per-agent, per-sender)                       │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  Layer 2: Authentication                                                  │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ • DM Pairing (4-digit codes, 10min expiry)                        │   │
│  │ • Gateway Token (OPENCLAW_GATEWAY_TOKEN)                          │   │
│  │ • Channel API Keys (bot tokens, OAuth)                            │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  Layer 1: Network                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ • Bind to Loopback (default: 127.0.0.1)                           │   │
│  │ • TLS for Remote Access (Tailscale recommended)                   │   │
│  │ • Firewall Rules (OS-level)                                       │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Technology Stack Visual

```
┌───────────────────────────────────────────────────────────────┐
│                        TECHNOLOGY STACK                        │
├───────────────────────────────────────────────────────────────┤
│                                                                │
│  Runtime Layer:                                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ Node.js 22+ (ESM) │ Bun (dev optional)               │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Language:                                                     │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ TypeScript 5.9+ (strict mode)                        │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Build Tools:                                                  │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ pnpm 10.23 │ tsdown │ Rolldown │ tsx                 │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Testing:                                                      │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ Vitest 4.0+ │ V8 Coverage │ 1,094 test files         │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Web Layer:                                                    │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ Express 5.2+ │ ws 8.19+ │ React (Control UI)         │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  AI/LLM:                                                       │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ @mariozechner/pi-* │ Anthropic │ OpenAI │ Gemini    │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Messaging:                                                    │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ Grammy (TG) │ Carbon (Discord) │ Bolt (Slack)        │    │
│  │ Baileys (WA) │ signal-cli │ LINE SDK                │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Storage:                                                      │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ File-based (JSONL) │ sqlite-vec (vectors)            │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Security:                                                     │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ detect-secrets │ Zod │ ajv │ CodeQL                  │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Browser:                                                      │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ Playwright 1.58 (Chromium, headless)                 │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                                │
│  Native Apps:                                                  │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ Swift/SwiftUI (macOS/iOS) │ Kotlin (Android)         │    │
│  └──────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────────┘
```

---

**Note**: Mermaid diagrams require rendering in a Markdown viewer that supports Mermaid (GitHub, VS Code with extension, Mintlify, etc.). ASCII diagrams work in any terminal or text editor.
