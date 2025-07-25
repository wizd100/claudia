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

## 핵심 기능 영역 요약

### 1. 프로젝트 및 세션 관리
- **파일**: `claude.rs`, `ProjectList.tsx`, `SessionList.tsx`
- **기능**: Claude Code 프로젝트 탐색, 세션 관리, 히스토리 추적

### 2. 커스텀 AI 에이전트
- **파일**: `agents.rs`, `CCAgents.tsx`, `CreateAgent.tsx`
- **기능**: 특수 목적 AI 에이전트 생성, 실행, 관리

### 3. 사용량 분석
- **파일**: `usage.rs`, `UsageDashboard.tsx`
- **기능**: Claude API 비용 추적, 토큰 분석, 사용량 리포트

### 4. MCP 서버 관리
- **파일**: `mcp.rs`, `MCPManager.tsx`
- **기능**: Model Context Protocol 서버 통합 및 관리

### 5. 타임라인 및 체크포인트
- **파일**: `checkpoint/` 디렉토리, `TimelineNavigator.tsx`
- **기능**: 세션 버전 관리, 체크포인트 시스템

### 6. CLAUDE.md 관리
- **파일**: `ClaudeFileEditor.tsx`, `MarkdownEditor.tsx`
- **기능**: 프로젝트 문서 편집 및 미리보기

이 분석을 통해 클라우디아 프로젝트는 Claude Code CLI의 모든 기능을 시각적으로 접근 가능하게 만들고, 추가적인 생산성 도구들(에이전트, 사용량 추적, 체크포인트)을 제공하는 종합적인 개발 도구임을 알 수 있습니다.