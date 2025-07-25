# Claudia Project: Core Files and Functionality Analysis

## Project Overview

**Claudia** is a powerful desktop GUI application and toolkit for Claude Code CLI. It serves as a bridge between Claude Code's command-line interface and a visual experience, making AI-assisted development more intuitive and productive.

### Technology Stack
- **Frontend**: React 18 + TypeScript + Vite 6
- **Backend**: Rust + Tauri 2 framework
- **UI Framework**: Tailwind CSS v4 + shadcn/ui
- **Database**: SQLite (via rusqlite)
- **Package Manager**: Bun (npm compatible)

## Core File Structure and Roles

### 1. Configuration and Setup Files

#### `package.json`
**Role**: Frontend dependencies and build script management
```json
{
  "name": "claudia",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "tauri": "tauri"
  }
}
```
- Defines React, TypeScript, Vite, and UI component dependencies
- Contains development and production build scripts
- Manages Tauri plugins and API dependencies

#### `src-tauri/Cargo.toml`
**Role**: Rust backend dependencies and build configuration
```toml
[package]
name = "claudia"
description = "GUI app and Toolkit for Claude Code"

[dependencies]
tauri = { version = "2", features = ["protocol-asset", "tray-icon", "image-png"] }
rusqlite = { version = "0.32", features = ["bundled"] }
tokio = { version = "1", features = ["full"] }
```
- Defines Tauri framework and plugins
- Database (SQLite) and async processing (Tokio) dependencies
- Additional libraries for system integration

#### `src-tauri/tauri.conf.json`
**Role**: Overall Tauri application configuration
- Application metadata (name, version, identifier)
- Window settings (size, title)
- Security policies (CSP, filesystem access permissions)
- Plugin permission settings

### 2. Backend Core Files (Rust/Tauri)

#### `src-tauri/src/main.rs`
**Role**: Application main entry point
```rust
fn main() {
    tauri::Builder::default()
        .plugin(tauri_plugin_dialog::init())
        .plugin(tauri_plugin_shell::init())
        .setup(|app| {
            // Initialize database
            let conn = init_database(&app.handle())?;
            app.manage(AgentDb(Mutex::new(conn)));
            
            // Initialize checkpoint state
            app.manage(CheckpointState::new());
            
            // Initialize process registry
            app.manage(ProcessRegistryState::default());
            
            Ok(())
        })
        .invoke_handler(tauri::generate_handler![
            // All command handlers registration
        ])
        .run(tauri::generate_context!())
}
```
**Core Functions**:
- Application state management initialization
- Database connection setup
- Registration of all Tauri command handlers
- Plugin initialization

#### `src-tauri/src/commands/claude.rs`
**Role**: Claude Code project and session management
```rust
pub struct Project {
    pub id: String,
    pub path: String,
    pub sessions: Vec<String>,
    pub created_at: u64,
}

pub struct Session {
    pub id: String,
    pub project_id: String,
    pub project_path: String,
    // ... other fields
}
```
**Core Functions**:
- Manages projects in `~/.claude/projects/` directory
- Executes Claude Code CLI commands
- Tracks session history and metadata
- Real-time command output streaming
- Checkpoint creation and restoration

**Key Functions**:
- `list_projects()`: Lists all Claude projects
- `execute_claude_code()`: Executes Claude Code commands
- `get_project_sessions()`: Retrieves project session lists
- `create_checkpoint()`: Creates session checkpoints

#### `src-tauri/src/commands/agents.rs`
**Role**: Custom AI agent management and execution
```rust
pub struct Agent {
    pub id: Option<i64>,
    pub name: String,
    pub icon: String,
    pub system_prompt: String,
    pub model: String,
    pub enable_file_read: bool,
    pub enable_file_write: bool,
    pub enable_network: bool,
    // ... other settings
}

pub struct AgentRun {
    pub id: Option<i64>,
    pub agent_id: i64,
    pub task: String,
    pub project_path: String,
    pub status: String,
    // ... execution info
}
```
**Core Functions**:
- Create and manage specialized AI agents
- Background agent execution
- Execution history and performance metrics tracking
- Persistent storage via SQLite database
- Import/export agents from/to GitHub

**Key Functions**:
- `create_agent()`: Creates new agents
- `execute_agent()`: Executes agents
- `list_agents()`: Lists all agents
- `get_agent_run()`: Retrieves agent execution results

#### `src-tauri/src/commands/mcp.rs`
**Role**: Model Context Protocol server management
```rust
pub struct MCPServer {
    pub name: String,
    pub transport: String,  // "stdio" or "sse"
    pub command: Option<String>,
    pub args: Option<Vec<String>>,
    pub env: Option<HashMap<String, String>>,
}
```
**Core Functions**:
- Add, remove, and configure MCP servers
- Import servers from Claude Desktop configuration
- Test server connections
- Project-specific MCP server configuration

**Key Functions**:
- `mcp_add()`: Adds new MCP servers
- `mcp_test_connection()`: Tests server connections
- `mcp_add_from_claude_desktop()`: Imports settings from Claude Desktop

#### `src-tauri/src/commands/usage.rs`
**Role**: Usage analytics and cost tracking
**Core Functions**:
- Monitor Claude API usage
- Token and cost analysis
- Statistics by date, project, and model
- Usage data export

**Key Functions**:
- `get_usage_stats()`: Overall usage statistics
- `get_usage_by_date_range()`: Usage by date range
- `get_session_stats()`: Session-specific statistics

#### `src-tauri/src/checkpoint/`
**Role**: Timeline and checkpoint management
**Core Functions**:
- Session version management
- Checkpoint creation and restoration
- Visual timeline navigation
- Session forking capabilities
- Diff viewer

### 3. Frontend Core Files (React/TypeScript)

#### `src/App.tsx`
**Role**: Main React application component
```typescript
type View = 
  | "welcome" 
  | "projects" 
  | "editor" 
  | "claude-file-editor" 
  | "settings"
  | "cc-agents"
  | "mcp"
  | "usage-dashboard"
  | "tabs";

function AppContent() {
  const [view, setView] = useState<View>("tabs");
  // ... state management
}
```
**Core Functions**:
- Overall application routing and view management
- Context provider setup
- Global state initialization

#### `src/lib/api.ts`
**Role**: API communication client with Rust backend
```typescript
export const api = {
  listProjects: () => invoke<Project[]>('list_projects'),
  executeClaudeCode: (params) => invoke('execute_claude_code', params),
  createAgent: (agent) => invoke('create_agent', { agent }),
  // ... other API calls
};
```
**Core Functions**:
- Backend calls via Tauri's `invoke` function
- TypeScript type safety provision
- API response data structure definitions

#### `src/components/` Directory
**Key Components**:

**`ProjectList.tsx`**: Project listing and navigation
**`SessionList.tsx`**: Session history management
**`CCAgents.tsx`**: Agent management interface
**`MCPManager.tsx`**: MCP server management UI
**`UsageDashboard.tsx`**: Usage analytics dashboard
**`TimelineNavigator.tsx`**: Checkpoint timeline UI
**`ClaudeCodeSession.tsx`**: Claude Code session execution interface

### 4. Special Configuration Files

#### `cc_agents/` Directory
**Role**: Predefined agent templates
- `unit-tests-bot.claudia.json`: Unit testing bot
- `git-commit-bot.claudia.json`: Git commit message generation bot
- `security-scanner.claudia.json`: Security scanning bot

#### `src-tauri/tests/`
**Role**: Rust backend test suite

## Core Functionality Areas Summary

### 1. Project and Session Management
- **Files**: `claude.rs`, `ProjectList.tsx`, `SessionList.tsx`
- **Features**: Claude Code project exploration, session management, history tracking

### 2. Custom AI Agents
- **Files**: `agents.rs`, `CCAgents.tsx`, `CreateAgent.tsx`
- **Features**: Specialized AI agent creation, execution, and management

### 3. Usage Analytics
- **Files**: `usage.rs`, `UsageDashboard.tsx`
- **Features**: Claude API cost tracking, token analysis, usage reporting

### 4. MCP Server Management
- **Files**: `mcp.rs`, `MCPManager.tsx`
- **Features**: Model Context Protocol server integration and management

### 5. Timeline and Checkpoints
- **Files**: `checkpoint/` directory, `TimelineNavigator.tsx`
- **Features**: Session version management, checkpoint system

### 6. CLAUDE.md Management
- **Files**: `ClaudeFileEditor.tsx`, `MarkdownEditor.tsx`
- **Features**: Project documentation editing and preview

## Technical Architecture

### Communication Flow
```
React Frontend (TypeScript) 
    ↕ (Tauri IPC)
Rust Backend (Tauri Commands)
    ↕ (SQLite/File System)
Local Data Storage
```

### Key Design Patterns
1. **Command Pattern**: Tauri commands handle specific operations
2. **State Management**: Shared state via Tauri's `manage()` system
3. **Process Isolation**: Agents run in separate processes for security
4. **Database Integration**: SQLite for persistent local storage
5. **Plugin Architecture**: Modular functionality via Tauri plugins

## Security Features
1. **Process Isolation**: Agents run in separate processes
2. **Permission Control**: Configurable file and network access per agent
3. **Local Storage**: All data stays on the user's machine
4. **No Telemetry**: No data collection or tracking
5. **Open Source**: Full transparency through open source code

This analysis reveals that Claudia is a comprehensive development tool that makes all Claude Code CLI functionality visually accessible while providing additional productivity tools (agents, usage tracking, checkpoints) for enhanced AI-assisted development workflows.