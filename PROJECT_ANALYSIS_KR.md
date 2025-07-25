# 클라우디아(Claudia) 프로젝트 핵심 파일 및 기능 분석

## 프로젝트 개요

**클라우디아(Claudia)**는 Claude Code CLI 도구를 위한 강력한 데스크톱 GUI 애플리케이션입니다. 이 프로젝트는 Claude Code의 명령줄 인터페이스와 시각적 경험 사이의 격차를 메워주어 AI 지원 개발을 보다 직관적이고 생산적으로 만듭니다.

### 기술 스택
- **프론트엔드**: React 18 + TypeScript + Vite 6
- **백엔드**: Rust + Tauri 2 프레임워크
- **UI 프레임워크**: Tailwind CSS v4 + shadcn/ui
- **데이터베이스**: SQLite (rusqlite 사용)
- **패키지 매니저**: Bun (npm 호환)

## 핵심 파일 구조 및 역할

### 1. 설정 및 구성 파일

#### `package.json`
**역할**: 프론트엔드 의존성 및 빌드 스크립트 관리
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
- React, TypeScript, Vite 및 UI 컴포넌트 의존성 정의
- 개발 및 프로덕션 빌드 스크립트 포함
- Tauri 플러그인 및 API 의존성 관리

#### `src-tauri/Cargo.toml`
**역할**: Rust 백엔드 의존성 및 빌드 설정
```toml
[package]
name = "claudia"
description = "GUI app and Toolkit for Claude Code"

[dependencies]
tauri = { version = "2", features = ["protocol-asset", "tray-icon", "image-png"] }
rusqlite = { version = "0.32", features = ["bundled"] }
tokio = { version = "1", features = ["full"] }
```
- Tauri 프레임워크 및 플러그인 정의
- 데이터베이스(SQLite), 비동기 처리(Tokio) 의존성
- 시스템 통합을 위한 추가 라이브러리들

#### `src-tauri/tauri.conf.json`
**역할**: Tauri 애플리케이션 전체 설정
- 애플리케이션 메타데이터 (이름, 버전, 식별자)
- 윈도우 설정 (크기, 제목)
- 보안 정책 (CSP, 파일 시스템 접근 권한)
- 플러그인 권한 설정

### 2. 백엔드 핵심 파일 (Rust/Tauri)

#### `src-tauri/src/main.rs`
**역할**: 애플리케이션 메인 엔트리 포인트
```rust
fn main() {
    tauri::Builder::default()
        .plugin(tauri_plugin_dialog::init())
        .plugin(tauri_plugin_shell::init())
        .setup(|app| {
            // 데이터베이스 초기화
            let conn = init_database(&app.handle())?;
            app.manage(AgentDb(Mutex::new(conn)));
            
            // 체크포인트 상태 초기화
            app.manage(CheckpointState::new());
            
            // 프로세스 레지스트리 초기화
            app.manage(ProcessRegistryState::default());
            
            Ok(())
        })
        .invoke_handler(tauri::generate_handler![
            // 모든 명령 핸들러 등록
        ])
        .run(tauri::generate_context!())
}
```
**핵심 기능**:
- 애플리케이션 상태 관리 초기화
- 데이터베이스 연결 설정
- 모든 Tauri 명령 핸들러 등록
- 플러그인 초기화

#### `src-tauri/src/commands/claude.rs`
**역할**: Claude Code 프로젝트 및 세션 관리
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
    // ... 기타 필드
}
```
**핵심 기능**:
- `~/.claude/projects/` 디렉토리의 프로젝트 관리
- Claude Code CLI 명령 실행
- 세션 히스토리 및 메타데이터 추적
- 실시간 명령 출력 스트리밍
- 체크포인트 생성 및 복원

**주요 함수들**:
- `list_projects()`: 모든 Claude 프로젝트 나열
- `execute_claude_code()`: Claude Code 명령 실행
- `get_project_sessions()`: 프로젝트의 세션 목록 조회
- `create_checkpoint()`: 세션 체크포인트 생성

#### `src-tauri/src/commands/agents.rs`
**역할**: 커스텀 AI 에이전트 관리 및 실행
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
    // ... 기타 설정
}

pub struct AgentRun {
    pub id: Option<i64>,
    pub agent_id: i64,
    pub task: String,
    pub project_path: String,
    pub status: String,
    // ... 실행 정보
}
```
**핵심 기능**:
- 특수 목적 AI 에이전트 생성 및 관리
- 에이전트 백그라운드 실행
- 실행 히스토리 및 성능 메트릭 추적
- SQLite 데이터베이스를 통한 영구 저장
- GitHub에서 에이전트 가져오기/내보내기

**주요 함수들**:
- `create_agent()`: 새 에이전트 생성
- `execute_agent()`: 에이전트 실행
- `list_agents()`: 모든 에이전트 나열
- `get_agent_run()`: 에이전트 실행 결과 조회

#### `src-tauri/src/commands/mcp.rs`
**역할**: Model Context Protocol 서버 관리
```rust
pub struct MCPServer {
    pub name: String,
    pub transport: String,  // "stdio" 또는 "sse"
    pub command: Option<String>,
    pub args: Option<Vec<String>>,
    pub env: Option<HashMap<String, String>>,
}
```
**핵심 기능**:
- MCP 서버 추가, 제거, 설정
- Claude Desktop 설정에서 서버 가져오기
- 서버 연결 테스트
- 프로젝트별 MCP 서버 구성

**주요 함수들**:
- `mcp_add()`: 새 MCP 서버 추가
- `mcp_test_connection()`: 서버 연결 테스트
- `mcp_add_from_claude_desktop()`: Claude Desktop에서 설정 가져오기

#### `src-tauri/src/commands/usage.rs`
**역할**: 사용량 분석 및 비용 추적
**핵심 기능**:
- Claude API 사용량 모니터링
- 토큰 및 비용 분석
- 날짜별, 프로젝트별, 모델별 통계
- 사용량 데이터 내보내기

**주요 함수들**:
- `get_usage_stats()`: 전체 사용량 통계
- `get_usage_by_date_range()`: 기간별 사용량
- `get_session_stats()`: 세션별 통계

#### `src-tauri/src/checkpoint/`
**역할**: 타임라인 및 체크포인트 관리
**핵심 기능**:
- 세션 버전 관리
- 체크포인트 생성 및 복원
- 시각적 타임라인 네비게이션
- 세션 분기(fork) 기능
- 차이점(diff) 뷰어

### 3. 프론트엔드 핵심 파일 (React/TypeScript)

#### `src/App.tsx`
**역할**: 메인 React 애플리케이션 컴포넌트
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
  // ... 상태 관리
}
```
**핵심 기능**:
- 전체 애플리케이션 라우팅 및 뷰 관리
- 컨텍스트 프로바이더 설정
- 전역 상태 초기화

#### `src/lib/api.ts`
**역할**: Rust 백엔드와의 API 통신 클라이언트
```typescript
export const api = {
  listProjects: () => invoke<Project[]>('list_projects'),
  executeClaudeCode: (params) => invoke('execute_claude_code', params),
  createAgent: (agent) => invoke('create_agent', { agent }),
  // ... 기타 API 호출들
};
```
**핵심 기능**:
- Tauri의 `invoke` 함수를 통한 백엔드 호출
- TypeScript 타입 안정성 제공
- API 응답 데이터 구조 정의

#### `src/components/` 디렉토리
**주요 컴포넌트들**:

**`ProjectList.tsx`**: 프로젝트 목록 및 탐색
**`SessionList.tsx`**: 세션 히스토리 관리
**`CCAgents.tsx`**: 에이전트 관리 인터페이스
**`MCPManager.tsx`**: MCP 서버 관리 UI
**`UsageDashboard.tsx`**: 사용량 분석 대시보드
**`TimelineNavigator.tsx`**: 체크포인트 타임라인 UI
**`ClaudeCodeSession.tsx`**: Claude Code 세션 실행 인터페이스

### 4. 특수 설정 파일들

#### `cc_agents/` 디렉토리
**역할**: 사전 정의된 에이전트 템플릿
- `unit-tests-bot.claudia.json`: 단위 테스트 봇
- `git-commit-bot.claudia.json`: Git 커밋 메시지 생성 봇
- `security-scanner.claudia.json`: 보안 스캔 봇

#### `src-tauri/tests/`
**역할**: Rust 백엔드 테스트 스위트

## 핵심 기능들의 상세 동작 원리

### 1. 프로젝트 및 세션 관리 시스템

#### 📁 프로젝트 디스커버리 메커니즘
```rust
// claude.rs - list_projects() 함수의 동작 원리
pub async fn list_projects() -> Result<Vec<Project>, String> {
    // 1. ~/.claude/projects 디렉토리 스캔
    let claude_dir = dirs::home_dir()
        .unwrap()
        .join(".claude")
        .join("projects");
    
    // 2. 각 하위 디렉토리를 프로젝트로 인식
    for entry in fs::read_dir(claude_dir) {
        let project_id = entry.file_name().to_string_lossy().to_string();
        
        // 3. URL 디코딩으로 원본 경로 복원
        let decoded_path = urlencoding::decode(&project_id)
            .map_err(|e| format!("Failed to decode project path: {}", e))?
            .to_string();
            
        // 4. .jsonl 파일들을 세션으로 수집
        let sessions = collect_sessions(&project_path)?;
        
        projects.push(Project {
            id: project_id,
            path: decoded_path,
            sessions,
            created_at: get_directory_creation_time(&project_path),
        });
    }
}
```

**동작 과정**:
1. **디렉토리 스캔**: `~/.claude/projects/` 디렉토리를 순회
2. **경로 디코딩**: URL 인코딩된 디렉토리명을 원본 프로젝트 경로로 복원
3. **세션 수집**: 각 프로젝트 디렉토리 내의 `.jsonl` 파일들을 세션으로 인식
4. **메타데이터 추출**: 파일 시스템 정보에서 생성 시간 등을 추출

#### 🔄 Claude Code 실행 시스템
```rust
// execute_claude_code() 함수의 내부 동작
pub async fn execute_claude_code(
    params: ExecuteClaudeCodeParams,
    process_state: State<'_, ClaudeProcessState>,
    app_handle: AppHandle,
) -> Result<String, String> {
    // 1. 기존 프로세스 종료
    let mut current_process = process_state.current_process.lock().await;
    if let Some(mut process) = current_process.take() {
        let _ = process.kill().await;
    }
    
    // 2. Claude Code 바이너리 찾기
    let claude_binary = find_claude_binary(&app_handle)?;
    
    // 3. 명령어 구성
    let mut command = Command::new(claude_binary);
    command
        .args(&params.args)
        .current_dir(&params.project_path)
        .stdout(Stdio::piped())
        .stderr(Stdio::piped());
    
    // 4. 프로세스 시작
    let mut child = command.spawn()
        .map_err(|e| format!("Failed to start claude: {}", e))?;
    
    // 5. 출력 스트리밍 시작
    if let Some(stdout) = child.stdout.take() {
        let app_handle_clone = app_handle.clone();
        tokio::spawn(async move {
            let reader = BufReader::new(stdout);
            let mut lines = reader.lines();
            
            while let Some(line) = lines.next_line().await? {
                // 실시간으로 프론트엔드에 출력 전송
                app_handle_clone.emit("claude-output", line)?;
            }
        });
    }
    
    // 6. 프로세스 저장
    *current_process = Some(child);
}
```

**실행 흐름**:
1. **프로세스 정리**: 기존 실행 중인 Claude 프로세스 종료
2. **바이너리 탐지**: 시스템에서 Claude Code CLI 바이너리 위치 찾기
3. **명령 구성**: 사용자 인수와 프로젝트 경로로 명령어 빌드
4. **비동기 실행**: Tokio를 사용하여 비동기 프로세스 시작
5. **실시간 스트리밍**: stdout/stderr을 실시간으로 프론트엔드에 전송
6. **프로세스 추적**: 전역 상태에서 현재 프로세스 관리

### 2. 커스텀 AI 에이전트 실행 엔진

#### 🤖 에이전트 생성 및 저장 메커니즘
```rust
// agents.rs - create_agent() 함수 상세 분석
pub async fn create_agent(
    db: State<'_, AgentDb>,
    agent: Agent,
) -> Result<i64, String> {
    let conn = db.0.lock().unwrap();
    
    // 1. SQLite 데이터베이스에 에이전트 정보 저장
    let result = conn.execute(
        r#"
        INSERT INTO agents (
            name, icon, system_prompt, default_task, model,
            enable_file_read, enable_file_write, enable_network,
            hooks, created_at, updated_at
        ) VALUES (?1, ?2, ?3, ?4, ?5, ?6, ?7, ?8, ?9, ?10, ?11)
        "#,
        params![
            agent.name,
            agent.icon,
            agent.system_prompt,
            agent.default_task,
            agent.model,
            agent.enable_file_read,
            agent.enable_file_write,
            agent.enable_network,
            agent.hooks,
            now,
            now,
        ],
    );
    
    // 2. 생성된 에이전트 ID 반환
    Ok(conn.last_insert_rowid())
}
```

#### ⚡ 에이전트 실행 시스템
```rust
// execute_agent() 함수의 복잡한 실행 로직
pub async fn execute_agent(
    db: State<'_, AgentDb>,
    process_registry: State<'_, ProcessRegistryState>,
    params: ExecuteAgentParams,
    app_handle: AppHandle,
) -> Result<i64, String> {
    // 1. 에이전트 정보 조회
    let agent = get_agent(db.clone(), params.agent_id).await?;
    
    // 2. 실행 레코드 생성
    let run_id = {
        let conn = db.0.lock().unwrap();
        conn.execute(
            r#"
            INSERT INTO agent_runs (
                agent_id, agent_name, agent_icon, task, model,
                project_path, status, started_at
            ) VALUES (?1, ?2, ?3, ?4, ?5, ?6, 'running', ?7)
            "#,
            params![
                agent.id.unwrap(),
                agent.name,
                agent.icon,
                params.task,
                agent.model,
                params.project_path,
                now
            ],
        ).map_err(|e| format!("Database error: {}", e))?;
        
        conn.last_insert_rowid()
    };
    
    // 3. Claude Code 명령어 구성
    let mut claude_args = vec![
        "code".to_string(),
        "--model".to_string(),
        agent.model.clone(),
        "--prompt".to_string(),
        format!("{}\n\nTask: {}", agent.system_prompt, params.task),
    ];
    
    // 4. 권한별 플래그 추가
    if agent.enable_file_read {
        claude_args.push("--read".to_string());
    }
    if agent.enable_file_write {
        claude_args.push("--write".to_string());
    }
    if agent.enable_network {
        claude_args.push("--network".to_string());
    }
    
    // 5. 비동기 프로세스 실행
    let claude_binary = find_claude_binary(&app_handle)?;
    let mut command = Command::new(claude_binary);
    command
        .args(&claude_args)
        .current_dir(&params.project_path)
        .stdout(Stdio::piped())
        .stderr(Stdio::piped());
    
    let mut child = command.spawn()
        .map_err(|e| format!("Failed to start claude: {}", e))?;
    
    // 6. 프로세스 레지스트리에 등록
    let child_id = child.id().unwrap_or(0);
    process_registry.register_process(run_id, child_id);
    
    // 7. 출력 모니터링 태스크 시작
    let app_handle_clone = app_handle.clone();
    let db_clone = db.clone();
    tokio::spawn(async move {
        monitor_agent_execution(run_id, child, app_handle_clone, db_clone).await;
    });
    
    Ok(run_id)
}
```

**에이전트 실행 흐름**:
1. **에이전트 조회**: 데이터베이스에서 에이전트 설정 로드
2. **실행 레코드 생성**: 새로운 실행 세션을 데이터베이스에 기록
3. **명령어 빌드**: 에이전트 설정에 따라 Claude Code 명령어 구성
4. **권한 적용**: 파일 읽기/쓰기, 네트워크 접근 권한 설정
5. **프로세스 시작**: 비동기로 Claude Code 프로세스 실행
6. **프로세스 추적**: 전역 레지스트리에서 실행 중인 프로세스 관리
7. **모니터링 시작**: 별도 태스크에서 실행 상태 및 출력 추적

#### 📊 실시간 메트릭 수집
```rust
// 에이전트 실행 중 실시간 메트릭 추출
async fn monitor_agent_execution(
    run_id: i64,
    mut child: Child,
    app_handle: AppHandle,
    db: State<'_, AgentDb>,
) {
    let mut output_buffer = String::new();
    
    if let Some(stdout) = child.stdout.take() {
        let reader = TokioBufReader::new(stdout);
        let mut lines = reader.lines();
        
        while let Some(line) = lines.next_line().await.unwrap_or(None) {
            output_buffer.push_str(&line);
            output_buffer.push('\n');
            
            // Claude API 호출 패턴 감지
            if line.contains("API call:") {
                extract_api_metrics(&line, run_id, &db).await;
            }
            
            // 토큰 사용량 추출
            if line.contains("tokens:") {
                extract_token_usage(&line, run_id, &db).await;
            }
            
            // 실시간으로 프론트엔드에 전송
            app_handle.emit("agent-output", AgentOutput {
                run_id,
                content: line,
                timestamp: chrono::Utc::now().to_rfc3339(),
            }).unwrap();
        }
    }
    
    // 프로세스 완료 처리
    let exit_status = child.wait().await.unwrap();
    update_agent_run_completion(run_id, exit_status, output_buffer, &db).await;
}
```

### 3. MCP (Model Context Protocol) 서버 통합 시스템

#### 🔌 MCP 서버 연결 메커니즘
```rust
// mcp.rs - MCP 서버 추가 및 테스트
pub async fn mcp_add(
    config: MCPServerConfig,
    app_handle: AppHandle,
) -> Result<(), String> {
    // 1. Claude Desktop 설정 파일 읽기
    let claude_config_path = dirs::home_dir()
        .unwrap()
        .join("Library/Application Support/Claude/claude_desktop_config.json");
    
    let mut claude_config: Value = if claude_config_path.exists() {
        let content = fs::read_to_string(&claude_config_path)
            .map_err(|e| format!("Failed to read Claude config: {}", e))?;
        serde_json::from_str(&content)
            .map_err(|e| format!("Failed to parse Claude config: {}", e))?
    } else {
        json!({
            "mcpServers": {}
        })
    };
    
    // 2. MCP 서버 설정 추가
    claude_config["mcpServers"][&config.name] = json!({
        "command": config.command,
        "args": config.args,
        "env": config.env
    });
    
    // 3. 설정 파일 업데이트
    let updated_content = serde_json::to_string_pretty(&claude_config)
        .map_err(|e| format!("Failed to serialize config: {}", e))?;
    
    fs::write(&claude_config_path, updated_content)
        .map_err(|e| format!("Failed to write config: {}", e))?;
    
    Ok(())
}
```

#### 🧪 MCP 서버 연결 테스트
```rust
// MCP 서버 연결 상태 확인
pub async fn mcp_test_connection(
    server_name: String,
) -> Result<MCPConnectionStatus, String> {
    // 1. 서버 프로세스 시작
    let mut command = Command::new(&server.command);
    if let Some(args) = &server.args {
        command.args(args);
    }
    
    command
        .stdin(Stdio::piped())
        .stdout(Stdio::piped())
        .stderr(Stdio::piped());
    
    let mut child = command.spawn()
        .map_err(|e| format!("Failed to start MCP server: {}", e))?;
    
    // 2. JSON-RPC 초기화 메시지 전송
    let init_message = json!({
        "jsonrpc": "2.0",
        "id": 1,
        "method": "initialize",
        "params": {
            "protocolVersion": "2024-11-05",
            "capabilities": {},
            "clientInfo": {
                "name": "claudia",
                "version": "1.0.0"
            }
        }
    });
    
    // 3. 응답 대기 및 파싱
    if let Some(mut stdin) = child.stdin.take() {
        let message_str = serde_json::to_string(&init_message).unwrap();
        stdin.write_all(message_str.as_bytes()).await
            .map_err(|e| format!("Failed to send init message: {}", e))?;
    }
    
    // 4. 응답 처리
    let timeout = tokio::time::sleep(Duration::from_secs(5));
    tokio::select! {
        result = read_mcp_response(&mut child) => {
            match result {
                Ok(response) => Ok(MCPConnectionStatus::Connected { response }),
                Err(e) => Ok(MCPConnectionStatus::Error { error: e }),
            }
        }
        _ = timeout => {
            let _ = child.kill().await;
            Ok(MCPConnectionStatus::Timeout)
        }
    }
}
```

### 4. 체크포인트 및 타임라인 시스템

#### 📸 체크포인트 생성 메커니즘
```rust
// checkpoint/ - 체크포인트 생성 및 관리
pub async fn create_checkpoint(
    session_id: String,
    project_path: String,
    description: Option<String>,
    checkpoint_state: State<'_, CheckpointState>,
) -> Result<CheckpointInfo, String> {
    // 1. 현재 세션 상태 스냅샷
    let session_data = read_session_jsonl(&session_id, &project_path).await?;
    let current_state = SessionState::from_jsonl(&session_data);
    
    // 2. 체크포인트 ID 생성
    let checkpoint_id = Uuid::new_v4().to_string();
    let timestamp = chrono::Utc::now();
    
    // 3. 체크포인트 데이터 구조 생성
    let checkpoint = CheckpointInfo {
        id: checkpoint_id.clone(),
        session_id: session_id.clone(),
        description: description.unwrap_or_else(|| {
            format!("Checkpoint at {}", timestamp.format("%Y-%m-%d %H:%M:%S"))
        }),
        created_at: timestamp.to_rfc3339(),
        message_count: current_state.messages.len(),
        token_usage: extract_token_usage_from_state(&current_state),
        file_changes: detect_file_changes(&current_state),
    };
    
    // 4. 체크포인트 파일 저장
    let checkpoint_dir = Path::new(&project_path)
        .join(".claude")
        .join("checkpoints")
        .join(&session_id);
    
    fs::create_dir_all(&checkpoint_dir)
        .map_err(|e| format!("Failed to create checkpoint directory: {}", e))?;
    
    let checkpoint_file = checkpoint_dir.join(format!("{}.json", checkpoint_id));
    let checkpoint_content = serde_json::to_string_pretty(&checkpoint)
        .map_err(|e| format!("Failed to serialize checkpoint: {}", e))?;
    
    fs::write(&checkpoint_file, checkpoint_content)
        .map_err(|e| format!("Failed to write checkpoint: {}", e))?;
    
    // 5. 세션 데이터 백업
    let session_backup = checkpoint_dir.join(format!("{}_session.jsonl", checkpoint_id));
    fs::copy(
        Path::new(&project_path).join(".claude").join(format!("{}.jsonl", session_id)),
        session_backup
    ).map_err(|e| format!("Failed to backup session data: {}", e))?;
    
    // 6. 타임라인 상태 업데이트
    let mut state = checkpoint_state.0.lock().await;
    state.add_checkpoint(checkpoint.clone());
    
    Ok(checkpoint)
}
```

#### 🔄 체크포인트 복원 시스템
```rust
// 체크포인트로부터 세션 복원
pub async fn restore_checkpoint(
    checkpoint_id: String,
    session_id: String,
    project_path: String,
    create_branch: Option<String>,
) -> Result<RestoreResult, String> {
    // 1. 체크포인트 정보 로드
    let checkpoint_file = Path::new(&project_path)
        .join(".claude")
        .join("checkpoints")
        .join(&session_id)
        .join(format!("{}.json", checkpoint_id));
    
    let checkpoint_content = fs::read_to_string(&checkpoint_file)
        .map_err(|e| format!("Failed to read checkpoint: {}", e))?;
    
    let checkpoint: CheckpointInfo = serde_json::from_str(&checkpoint_content)
        .map_err(|e| format!("Failed to parse checkpoint: {}", e))?;
    
    // 2. 분기 생성 또는 기존 세션 복원
    let target_session_id = if let Some(branch_name) = create_branch {
        let new_session_id = format!("{}_{}", session_id, branch_name);
        
        // 새 분기 세션 생성
        let session_backup = Path::new(&project_path)
            .join(".claude")
            .join("checkpoints")
            .join(&session_id)
            .join(format!("{}_session.jsonl", checkpoint_id));
        
        let new_session_file = Path::new(&project_path)
            .join(".claude")
            .join(format!("{}.jsonl", new_session_id));
        
        fs::copy(session_backup, new_session_file)
            .map_err(|e| format!("Failed to create branch: {}", e))?;
        
        new_session_id
    } else {
        // 기존 세션 백업 후 복원
        let current_session = Path::new(&project_path)
            .join(".claude")
            .join(format!("{}.jsonl", session_id));
        
        let backup_name = format!("{}_backup_{}.jsonl", 
            session_id, 
            chrono::Utc::now().timestamp()
        );
        let backup_path = Path::new(&project_path)
            .join(".claude")
            .join("backups")
            .join(backup_name);
        
        fs::create_dir_all(backup_path.parent().unwrap())
            .map_err(|e| format!("Failed to create backup directory: {}", e))?;
        
        fs::copy(&current_session, backup_path)
            .map_err(|e| format!("Failed to backup current session: {}", e))?;
        
        // 체크포인트 데이터로 복원
        let session_backup = Path::new(&project_path)
            .join(".claude")
            .join("checkpoints")
            .join(&session_id)
            .join(format!("{}_session.jsonl", checkpoint_id));
        
        fs::copy(session_backup, current_session)
            .map_err(|e| format!("Failed to restore session: {}", e))?;
        
        session_id
    };
    
    Ok(RestoreResult {
        session_id: target_session_id,
        checkpoint_info: checkpoint,
        restored_message_count: checkpoint.message_count,
    })
}
```

### 5. 사용량 추적 및 분석 시스템

#### 📈 실시간 사용량 모니터링
```rust
// usage.rs - 사용량 데이터 수집 및 분석
pub async fn track_api_usage(
    session_output: &str,
    session_id: &str,
    project_path: &str,
) -> Result<UsageMetrics, String> {
    // 1. Claude API 응답에서 메트릭 추출
    let api_pattern = regex::Regex::new(
        r"API call: (\w+) model, (\d+) input tokens, (\d+) output tokens, \$([0-9.]+)"
    ).unwrap();
    
    let mut total_input_tokens = 0;
    let mut total_output_tokens = 0;
    let mut total_cost = 0.0;
    let mut model_usage = HashMap::new();
    
    for line in session_output.lines() {
        if let Some(captures) = api_pattern.captures(line) {
            let model = captures.get(1).unwrap().as_str();
            let input_tokens: u32 = captures.get(2).unwrap().as_str().parse().unwrap();
            let output_tokens: u32 = captures.get(3).unwrap().as_str().parse().unwrap();
            let cost: f64 = captures.get(4).unwrap().as_str().parse().unwrap();
            
            total_input_tokens += input_tokens;
            total_output_tokens += output_tokens;
            total_cost += cost;
            
            // 모델별 사용량 집계
            let model_stats = model_usage.entry(model.to_string()).or_insert(ModelUsage {
                input_tokens: 0,
                output_tokens: 0,
                cost: 0.0,
                call_count: 0,
            });
            
            model_stats.input_tokens += input_tokens;
            model_stats.output_tokens += output_tokens;
            model_stats.cost += cost;
            model_stats.call_count += 1;
        }
    }
    
    // 2. 사용량 데이터 저장
    let usage_metrics = UsageMetrics {
        session_id: session_id.to_string(),
        project_path: project_path.to_string(),
        timestamp: chrono::Utc::now().to_rfc3339(),
        total_input_tokens,
        total_output_tokens,
        total_cost,
        model_usage,
        duration_seconds: calculate_session_duration(session_id, project_path).await?,
    };
    
    save_usage_metrics(&usage_metrics).await?;
    
    Ok(usage_metrics)
}
```

#### 📊 사용량 대시보드 데이터 생성
```rust
// 종합 사용량 통계 생성
pub async fn generate_usage_dashboard(
    date_range: Option<DateRange>,
    project_filter: Option<String>,
) -> Result<UsageDashboard, String> {
    // 1. 모든 사용량 데이터 로드
    let usage_data = load_usage_data_in_range(date_range).await?;
    
    // 2. 프로젝트별 필터링
    let filtered_data = if let Some(project) = project_filter {
        usage_data.into_iter()
            .filter(|usage| usage.project_path.contains(&project))
            .collect()
    } else {
        usage_data
    };
    
    // 3. 집계 계산
    let mut daily_usage: BTreeMap<String, DailyUsage> = BTreeMap::new();
    let mut model_breakdown: HashMap<String, ModelStats> = HashMap::new();
    let mut project_breakdown: HashMap<String, ProjectStats> = HashMap::new();
    
    for usage in &filtered_data {
        let date = usage.timestamp[..10].to_string(); // YYYY-MM-DD
        
        // 일별 사용량 집계
        let daily = daily_usage.entry(date).or_insert(DailyUsage::default());
        daily.total_tokens += usage.total_input_tokens + usage.total_output_tokens;
        daily.total_cost += usage.total_cost;
        daily.session_count += 1;
        
        // 모델별 집계
        for (model, model_usage) in &usage.model_usage {
            let stats = model_breakdown.entry(model.clone()).or_insert(ModelStats::default());
            stats.total_input_tokens += model_usage.input_tokens;
            stats.total_output_tokens += model_usage.output_tokens;
            stats.total_cost += model_usage.cost;
            stats.call_count += model_usage.call_count;
        }
        
        // 프로젝트별 집계
        let project_name = extract_project_name(&usage.project_path);
        let stats = project_breakdown.entry(project_name).or_insert(ProjectStats::default());
        stats.total_tokens += usage.total_input_tokens + usage.total_output_tokens;
        stats.total_cost += usage.total_cost;
        stats.session_count += 1;
    }
    
    // 4. 비용 예측 계산
    let cost_projection = calculate_monthly_projection(&daily_usage);
    
    Ok(UsageDashboard {
        summary: UsageSummary {
            total_cost: filtered_data.iter().map(|u| u.total_cost).sum(),
            total_tokens: filtered_data.iter().map(|u| u.total_input_tokens + u.total_output_tokens).sum(),
            total_sessions: filtered_data.len(),
            date_range: calculate_actual_date_range(&filtered_data),
        },
        daily_usage,
        model_breakdown,
        project_breakdown,
        cost_projection,
        efficiency_metrics: calculate_efficiency_metrics(&filtered_data),
    })
}
```

### 6. 데이터베이스 및 저장소 관리

#### 🗄️ SQLite 스키마 및 마이그레이션
```rust
// storage.rs - 데이터베이스 초기화 및 마이그레이션
pub fn init_database(app: &AppHandle) -> SqliteResult<Connection> {
    let app_data_dir = app.path().app_data_dir()
        .map_err(|e| rusqlite::Error::SqliteFailure(
            rusqlite::ffi::Error::new(rusqlite::ffi::SQLITE_CANTOPEN),
            Some(format!("Failed to get app data dir: {}", e))
        ))?;
    
    std::fs::create_dir_all(&app_data_dir)
        .map_err(|e| rusqlite::Error::SqliteFailure(
            rusqlite::ffi::Error::new(rusqlite::ffi::SQLITE_CANTOPEN),
            Some(format!("Failed to create app data dir: {}", e))
        ))?;
    
    let db_path = app_data_dir.join("claudia.db");
    let conn = Connection::open(&db_path)?;
    
    // 1. 에이전트 테이블 생성
    conn.execute(
        r#"
        CREATE TABLE IF NOT EXISTS agents (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL UNIQUE,
            icon TEXT NOT NULL,
            system_prompt TEXT NOT NULL,
            default_task TEXT,
            model TEXT NOT NULL,
            enable_file_read BOOLEAN NOT NULL DEFAULT 0,
            enable_file_write BOOLEAN NOT NULL DEFAULT 0,
            enable_network BOOLEAN NOT NULL DEFAULT 0,
            hooks TEXT, -- JSON string
            created_at TEXT NOT NULL,
            updated_at TEXT NOT NULL
        )
        "#,
        [],
    )?;
    
    // 2. 에이전트 실행 테이블 생성
    conn.execute(
        r#"
        CREATE TABLE IF NOT EXISTS agent_runs (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            agent_id INTEGER NOT NULL,
            agent_name TEXT NOT NULL,
            agent_icon TEXT NOT NULL,
            task TEXT NOT NULL,
            model TEXT NOT NULL,
            project_path TEXT NOT NULL,
            session_id TEXT,
            status TEXT NOT NULL, -- 'running', 'completed', 'failed', 'cancelled'
            started_at TEXT NOT NULL,
            completed_at TEXT,
            duration_seconds INTEGER,
            output TEXT,
            error_message TEXT,
            input_tokens INTEGER DEFAULT 0,
            output_tokens INTEGER DEFAULT 0,
            total_cost REAL DEFAULT 0.0,
            FOREIGN KEY (agent_id) REFERENCES agents (id) ON DELETE CASCADE
        )
        "#,
        [],
    )?;
    
    // 3. 사용량 추적 테이블
    conn.execute(
        r#"
        CREATE TABLE IF NOT EXISTS usage_metrics (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            session_id TEXT NOT NULL,
            project_path TEXT NOT NULL,
            timestamp TEXT NOT NULL,
            model TEXT NOT NULL,
            input_tokens INTEGER NOT NULL,
            output_tokens INTEGER NOT NULL,
            cost REAL NOT NULL,
            duration_seconds INTEGER,
            created_at TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
        )
        "#,
        [],
    )?;
    
    // 4. 인덱스 생성
    conn.execute(
        "CREATE INDEX IF NOT EXISTS idx_agent_runs_agent_id ON agent_runs(agent_id)",
        [],
    )?;
    
    conn.execute(
        "CREATE INDEX IF NOT EXISTS idx_agent_runs_status ON agent_runs(status)",
        [],
    )?;
    
    conn.execute(
        "CREATE INDEX IF NOT EXISTS idx_usage_metrics_timestamp ON usage_metrics(timestamp)",
        [],
    )?;
    
    // 5. 스키마 버전 관리
    create_migration_table(&conn)?;
    run_pending_migrations(&conn)?;
    
    Ok(conn)
}
```

### 7. 프론트엔드-백엔드 통신 아키텍처

#### 🌉 Tauri IPC (Inter-Process Communication)
```typescript
// src/lib/api.ts - 타입 안전한 API 클라이언트
export const api = {
    // 프로젝트 관리
    listProjects: (): Promise<Project[]> => 
        invoke('list_projects'),
    
    executeClaudeCode: (params: ExecuteClaudeCodeParams): Promise<string> => 
        invoke('execute_claude_code', { params }),
    
    // 에이전트 관리
    createAgent: (agent: Agent): Promise<number> => 
        invoke('create_agent', { agent }),
    
    executeAgent: (params: ExecuteAgentParams): Promise<number> => 
        invoke('execute_agent', { params }),
    
    listAgentRuns: (filters?: AgentRunFilters): Promise<AgentRun[]> => 
        invoke('list_agent_runs', { filters }),
    
    // 실시간 이벤트 구독
    subscribeToOutput: (callback: (output: string) => void) => {
        return listen<string>('claude-output', (event) => {
            callback(event.payload);
        });
    },
    
    subscribeToAgentOutput: (callback: (output: AgentOutput) => void) => {
        return listen<AgentOutput>('agent-output', (event) => {
            callback(event.payload);
        });
    },
};
```

#### ⚡ 실시간 상태 동기화
```typescript
// React 컴포넌트에서의 실시간 업데이트 처리
export function ClaudeCodeSession({ sessionId, projectPath }: Props) {
    const [output, setOutput] = useState<string[]>([]);
    const [isRunning, setIsRunning] = useState(false);
    
    useEffect(() => {
        // 1. Claude Code 출력 구독
        const outputUnlisten = api.subscribeToOutput((newOutput) => {
            setOutput(prev => [...prev, newOutput]);
        });
        
        // 2. 프로세스 상태 구독
        const statusUnlisten = listen<ProcessStatus>('process-status', (event) => {
            if (event.payload.sessionId === sessionId) {
                setIsRunning(event.payload.status === 'running');
            }
        });
        
        // 3. 컴포넌트 언마운트 시 구독 해제
        return () => {
            outputUnlisten.then(fn => fn());
            statusUnlisten.then(fn => fn());
        };
    }, [sessionId]);
    
    const executeCommand = async (command: string) => {
        try {
            setIsRunning(true);
            setOutput([]);
            
            await api.executeClaudeCode({
                sessionId,
                projectPath,
                args: command.split(' '),
            });
        } catch (error) {
            console.error('Failed to execute command:', error);
        }
    };
    
    return (
        <div className="claude-session">
            <CommandInput onExecute={executeCommand} disabled={isRunning} />
            <OutputDisplay output={output} isRunning={isRunning} />
        </div>
    );
}
```

## 핵심 기능 영역 요약

### 1. 프로젝트 및 세션 관리
- **파일**: `claude.rs`, `ProjectList.tsx`, `SessionList.tsx`
- **기능**: Claude Code 프로젝트 탐색, 세션 관리, 히스토리 추적
- **동작**: 파일 시스템 스캔 → 경로 디코딩 → 세션 수집 → 실시간 실행

### 2. 커스텀 AI 에이전트
- **파일**: `agents.rs`, `CCAgents.tsx`, `CreateAgent.tsx`
- **기능**: 특수 목적 AI 에이전트 생성, 실행, 관리
- **동작**: SQLite 저장 → 권한 설정 → 비동기 실행 → 실시간 모니터링

### 3. 사용량 분석
- **파일**: `usage.rs`, `UsageDashboard.tsx`
- **기능**: Claude API 비용 추적, 토큰 분석, 사용량 리포트
- **동작**: 실시간 메트릭 추출 → 데이터 집계 → 대시보드 생성

### 4. MCP 서버 관리
- **파일**: `mcp.rs`, `MCPManager.tsx`
- **기능**: Model Context Protocol 서버 통합 및 관리
- **동작**: 설정 파일 수정 → 연결 테스트 → JSON-RPC 통신

### 5. 타임라인 및 체크포인트
- **파일**: `checkpoint/` 디렉토리, `TimelineNavigator.tsx`
- **기능**: 세션 버전 관리, 체크포인트 시스템
- **동작**: 상태 스냅샷 → 파일 백업 → 분기 생성 → 복원

### 6. CLAUDE.md 관리
- **파일**: `ClaudeFileEditor.tsx`, `MarkdownEditor.tsx`
- **기능**: 프로젝트 문서 편집 및 미리보기

이 상세한 분석을 통해 클라우디아 프로젝트는 Claude Code CLI의 모든 기능을 시각적으로 접근 가능하게 만들고, 추가적인 생산성 도구들(에이전트, 사용량 추적, 체크포인트)을 제공하는 종합적인 개발 도구임을 알 수 있습니다. 각 시스템은 비동기 처리, 실시간 통신, 타입 안전성을 기반으로 하여 안정적이고 효율적인 사용자 경험을 제공합니다.