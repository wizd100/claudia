# Claudia Technical Architecture Documentation

## System Architecture Overview

Claudia follows a modern desktop application architecture using Tauri framework, which combines a Rust backend with a web-based frontend. This allows for native performance while maintaining web development flexibility.

```
┌─────────────────────────────────────────────────────────────┐
│                    Claudia Desktop App                      │
├─────────────────────────────────────────────────────────────┤
│  Frontend (React + TypeScript)                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐│
│  │   Components    │  │     Stores      │  │    Contexts     ││
│  │   - UI/UX       │  │  - Zustand      │  │  - Tab Mgmt     ││
│  │   - Views       │  │  - State Mgmt   │  │  - Output Cache ││
│  └─────────────────┘  └─────────────────┘  └─────────────────┘│
├─────────────────────────────────────────────────────────────┤
│                    Tauri IPC Layer                         │
├─────────────────────────────────────────────────────────────┤
│  Backend (Rust + Tauri)                                    │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐│
│  │    Commands     │  │     State       │  │    Processes    ││
│  │  - Claude API   │  │  - Checkpoint   │  │  - Agents       ││
│  │  - Agents       │  │  - Registries   │  │  - Claude CLI   ││
│  │  - MCP          │  │  - Sessions     │  │  - Background   ││
│  └─────────────────┘  └─────────────────┘  └─────────────────┘│
├─────────────────────────────────────────────────────────────┤
│                    Data Layer                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐│
│  │    SQLite DB    │  │   File System   │  │   External      ││
│  │  - Agents       │  │  - Projects     │  │  - Claude CLI   ││
│  │  - Usage Stats  │  │  - Sessions     │  │  - GitHub API   ││
│  │  - Agent Runs   │  │  - Checkpoints  │  │  - MCP Servers  ││
│  └─────────────────┘  └─────────────────┘  └─────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

## Core Modules and Responsibilities

### 1. Frontend Layer (React/TypeScript)

#### State Management
- **Zustand Stores**: Centralized state management for complex data
  - `sessionStore.ts`: Manages session data and real-time updates
  - `agentStore.ts`: Handles agent state and execution status
- **Context Providers**: Local state for specific features
  - `TabContext`: Tab management and navigation
  - `OutputCacheProvider`: Caches command outputs for performance

#### Component Architecture
```typescript
// Main component hierarchy
App.tsx
├── Topbar.tsx
├── TabManager.tsx
│   └── TabContent.tsx
│       ├── ProjectList.tsx
│       ├── SessionList.tsx
│       ├── CCAgents.tsx
│       ├── MCPManager.tsx
│       └── UsageDashboard.tsx
└── Various Modal/Dialog components
```

#### Key Frontend Patterns
- **Command Pattern**: API calls abstracted through `api.ts`
- **Observer Pattern**: Real-time updates via WebSocket-like mechanisms
- **Provider Pattern**: Context for cross-cutting concerns
- **Composition Pattern**: Reusable UI components

### 2. Backend Layer (Rust/Tauri)

#### Command Structure
The backend is organized into command modules, each handling specific functionality:

```rust
// Command organization in src-tauri/src/commands/
commands/
├── claude.rs       // Claude Code project management
├── agents.rs       // AI agent creation and execution
├── mcp.rs         // Model Context Protocol integration
├── usage.rs       // Usage analytics and billing
├── storage.rs     // Database operations
└── slash_commands.rs // Custom slash commands
```

#### State Management
```rust
// Global application state
pub struct AppState {
    agent_db: AgentDb,              // SQLite connection pool
    checkpoint_state: CheckpointState, // Session checkpoints
    process_registry: ProcessRegistryState, // Running processes
    claude_process: ClaudeProcessState,     // Claude CLI instances
}
```

#### Process Management
The system manages multiple types of processes:

1. **Agent Processes**: Background AI agents
2. **Claude Sessions**: Interactive Claude Code sessions
3. **MCP Servers**: Model Context Protocol servers

```rust
// Process registry structure
pub struct ProcessRegistry {
    processes: HashMap<String, ProcessHandle>,
    next_id: AtomicU64,
}

pub struct ProcessHandle {
    info: ProcessInfo,
    child: Arc<Mutex<Option<Child>>>,
    live_output: Arc<Mutex<String>>,
}
```

### 3. Data Layer

#### Database Schema (SQLite)
```sql
-- Agents table
CREATE TABLE agents (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    icon TEXT NOT NULL,
    system_prompt TEXT NOT NULL,
    model TEXT NOT NULL,
    enable_file_read BOOLEAN DEFAULT 0,
    enable_file_write BOOLEAN DEFAULT 0,
    enable_network BOOLEAN DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Agent execution runs
CREATE TABLE agent_runs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    agent_id INTEGER NOT NULL,
    task TEXT NOT NULL,
    project_path TEXT NOT NULL,
    status TEXT NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    started_at DATETIME,
    finished_at DATETIME,
    FOREIGN KEY(agent_id) REFERENCES agents(id)
);

-- Usage tracking
CREATE TABLE usage_stats (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT,
    model TEXT NOT NULL,
    input_tokens INTEGER,
    output_tokens INTEGER,
    cost_usd REAL,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

#### File System Structure
```
~/.claude/
├── projects/                    # Claude Code projects
│   ├── {encoded_path_1}/
│   │   ├── {session_id_1}.jsonl
│   │   └── {session_id_2}.jsonl
│   └── {encoded_path_2}/
└── claudia/                     # Claudia-specific data
    ├── agents.db               # SQLite database
    ├── checkpoints/            # Session checkpoints
    │   └── {session_id}/
    │       ├── {checkpoint_id}/
    │       └── metadata.json
    └── config/
        ├── mcp_servers.json    # MCP server configuration
        └── settings.json       # Application settings
```

## Key Technical Patterns

### 1. Command-Query Responsibility Segregation (CQRS)
- **Commands**: Modify state (create_agent, execute_claude_code)
- **Queries**: Read state (list_projects, get_usage_stats)
- Clear separation between read and write operations

### 2. Event-Driven Architecture
```rust
// Events emitted to frontend
app.emit("agent_output", output_data)?;
app.emit("session_status_changed", status)?;
app.emit("checkpoint_created", checkpoint_info)?;
```

### 3. Process Isolation
- Each agent runs in a separate process for security
- Configurable permissions (file read/write, network access)
- Process lifecycle management with cleanup

### 4. Asynchronous Processing
```rust
// Non-blocking command execution
#[tauri::command]
pub async fn execute_agent(
    app_handle: AppHandle,
    agent: Agent,
    task: String,
    project_path: String,
) -> Result<i64, String> {
    // Spawn background process
    tokio::spawn(async move {
        // Agent execution logic
    });
    Ok(run_id)
}
```

## Security Architecture

### 1. Sandboxing
- **Process Isolation**: Agents run in separate processes
- **File System Access**: Configurable per agent
- **Network Access**: Optional and controllable
- **Command Validation**: Input sanitization and validation

### 2. Permission Model
```rust
pub struct AgentPermissions {
    pub enable_file_read: bool,
    pub enable_file_write: bool, 
    pub enable_network: bool,
    pub allowed_commands: Vec<String>,
}
```

### 3. Data Privacy
- **Local Storage**: All data remains on user's machine
- **No Telemetry**: No data collection or external tracking
- **Encrypted Preferences**: Sensitive settings encrypted at rest

## Performance Optimizations

### 1. Frontend Optimizations
- **Virtual Scrolling**: For large session lists
- **Output Caching**: Prevents redundant API calls
- **Debounced Search**: Reduces API load
- **Lazy Loading**: Components load on demand

### 2. Backend Optimizations
- **Connection Pooling**: Reuse database connections
- **Process Reuse**: Keep processes alive for multiple commands
- **Streaming Output**: Real-time command output without buffering
- **Background Processing**: Non-blocking agent execution

### 3. Database Optimizations
```sql
-- Indexes for common queries
CREATE INDEX idx_agent_runs_agent_id ON agent_runs(agent_id);
CREATE INDEX idx_usage_stats_timestamp ON usage_stats(timestamp);
CREATE INDEX idx_agent_runs_status ON agent_runs(status);
```

## Extensibility Points

### 1. Plugin Architecture
- **Tauri Plugins**: Add system-level capabilities
- **Command Extensions**: New backend commands
- **UI Components**: Reusable frontend components

### 2. Agent System
- **Custom Agents**: User-defined AI agents
- **Agent Templates**: Shareable agent configurations
- **GitHub Integration**: Import/export agents

### 3. MCP Integration
- **Server Registry**: Dynamic MCP server management
- **Protocol Extensions**: Support for new MCP features
- **Configuration Import**: From Claude Desktop and other sources

## Development Workflow

### 1. Build Process
```bash
# Frontend development
npm run dev

# Full application development
npm run tauri dev

# Production build
npm run tauri build
```

### 2. Testing Strategy
- **Unit Tests**: Rust backend functions
- **Integration Tests**: Command handlers
- **E2E Tests**: Full application workflows
- **Manual Testing**: UI and UX validation

### 3. Code Organization
```
src/                 # Frontend code
├── components/      # React components
├── lib/            # Utilities and API client
├── stores/         # State management
├── types/          # TypeScript definitions
└── styles/         # CSS and styling

src-tauri/          # Backend code
├── src/
│   ├── commands/   # Tauri command handlers
│   ├── checkpoint/ # Checkpoint management
│   ├── process/    # Process management
│   └── lib.rs      # Main library
└── tests/          # Rust tests
```

This architecture provides a robust, secure, and extensible foundation for the Claudia application, supporting both current features and future enhancements.