# Claude Code Chat - Architecture & Design

## Overview

**Claude Code Chat** is a VS Code extension that provides a beautiful chat interface for interacting with Claude Code (Anthropic's AI CLI tool). This document explains the implementation, architecture, and key APIs.

## Core Files & Their Roles

### **src/extension.ts** (Main Extension Logic - ~2400 lines)
- **Entry point**: `activate()` function registers commands and initializes providers
- **ClaudeChatProvider**: Core class managing:
  - Webview panels (main chat window)
  - Claude CLI process spawning and communication
  - Session/conversation management
  - Backup/restore system using git
  - Permissions and MCP server configuration
  - File operations and state management
- **ClaudeChatWebviewProvider**: Handles sidebar webview integration

### **src/ui.ts** (HTML Template - ~750 lines)
- Returns complete HTML structure for the webview
- Includes chat interface, modals (settings, file picker, slash commands)
- Defines all UI components and their structure

### **src/ui-styles.ts** (CSS Styling)
- Complete styling for the chat interface
- VS Code theme integration

### **src/script.ts** (Client-side JavaScript - ~31k lines)
- All webview client-side logic
- Message rendering and UI interactions
- Communication with extension backend via `vscode` API
- Handles: file picker, image attachments, diff rendering, permissions UI, etc.

## Architecture Components

### 1. Communication Flow

```
User Input → Webview (script.ts)
          → postMessage to extension
          → ClaudeChatProvider processes
          → Spawns Claude CLI process
          → Streams JSON responses back to webview
          → Webview renders messages
```

### 2. Claude CLI Integration

**Location**: `extension.ts:408-655`

- Spawns `claude` CLI with args: `--output-format stream-json --verbose`
- Supports WSL mode for Windows users
- Session management with `--resume` flag
- Processes streaming JSON responses:
  - `type: 'system'` - initialization/session info
  - `type: 'assistant'` - Claude's responses (text, thinking, tool_use)
  - `type: 'user'` - tool results
  - `type: 'result'` - final statistics

**Key Code**:
```typescript
// extension.ts:535-544
claudeProcess = cp.spawn('claude', args, {
  shell: process.platform === 'win32',
  cwd: cwd,
  stdio: ['pipe', 'pipe', 'pipe'],
  env: {
    ...process.env,
    FORCE_COLOR: '0',
    NO_COLOR: '1'
  }
});
```

### 3. Backup/Restore System

**Location**: `extension.ts:970-1110`

- Git-based checkpoints stored in workspace storage
- Creates commit before each Claude interaction
- Allows instant restoration to any previous state
- Uses `--git-dir` and `--work-tree` for separate git repo

**Key Features**:
- Automatic backup before every request
- Visual restore points in chat UI
- One-click restoration to any checkpoint
- Empty commits for no-change checkpoints

### 4. Permissions System

**Location**: `extension.ts:1188-1602`

- MCP-based permission requests via file watcher
- Watches `*.request` files in storage directory
- Shows permission dialogs in UI
- Supports "always allow" patterns for Bash commands
- YOLO mode to skip all permissions

**Permission Flow**:
1. MCP server creates `*.request` file
2. File watcher triggers `_handlePermissionRequest()`
3. UI shows permission dialog
4. User response written to `*.response` file
5. MCP server reads response and proceeds

### 5. MCP Server Management

**Location**: `extension.ts:1134-1719`

- Manages `mcp-servers.json` configuration
- Includes built-in permissions MCP server
- UI for adding/removing MCP servers
- Supports stdio, HTTP, and SSE server types

**Configuration Structure**:
```json
{
  "mcpServers": {
    "claude-code-chat-permissions": {
      "command": "node",
      "args": ["/path/to/mcp-permissions.js"],
      "env": {
        "CLAUDE_PERMISSIONS_PATH": "/path/to/permission-requests"
      }
    }
  }
}
```

### 6. Conversation Management

**Location**: `extension.ts:1112-2136`

- Auto-saves conversations to JSON files
- Maintains conversation index in workspace state
- Session resume from latest conversation
- Message history with timestamps, costs, tokens

**Conversation Structure**:
```typescript
interface ConversationData {
  sessionId: string;
  startTime: string | undefined;
  endTime: string;
  messageCount: number;
  totalCost: number;
  totalTokens: {
    input: number;
    output: number;
  };
  messages: Array<{ timestamp: string, messageType: string, data: any }>;
  filename: string;
}
```

## Important APIs

### VS Code Extension API

- `vscode.window.createWebviewPanel()` - Creates chat window
- `vscode.window.registerWebviewViewProvider()` - Sidebar integration
- `vscode.workspace.fs.*` - File system operations
- `webview.postMessage()` / `onDidReceiveMessage()` - Webview communication
- `vscode.workspace.getConfiguration()` - Settings access
- `vscode.workspace.createFileSystemWatcher()` - Permission file monitoring

### Child Process Management

- `cp.spawn('claude', args)` - Spawns Claude CLI
- Streams stdout/stderr for real-time responses
- Process lifecycle management (SIGTERM/SIGKILL)
- WSL support via `wsl -d <distro> bash -ic <command>`

### Message Types (Webview ↔ Extension)

#### To Extension:
- `sendMessage` - User input with optional plan/thinking modes
- `newSession` - Clear session and start fresh
- `restoreCommit` - Restore to checkpoint by SHA
- `loadConversation` - Load conversation from history
- `stopRequest` - Stop Claude process
- `permissionResponse` - Permission approval/denial
- `executeSlashCommand` - Run slash command in terminal
- `getSettings` / `updateSettings` - Settings management
- `selectModel` - Change AI model
- `saveMCPServer` / `deleteMCPServer` - MCP server management
- `saveCustomSnippet` / `deleteCustomSnippet` - Custom commands
- `enableYoloMode` - Enable auto-approve permissions

#### From Extension:
- `userInput` - Display user message
- `output` - Claude text response
- `thinking` - Claude's thinking blocks
- `toolUse` - Tool execution notification
- `toolResult` - Tool execution results
- `error` - Error messages
- `sessionCleared` - Reset UI
- `permissionRequest` - Show permission dialog
- `sessionInfo` - Session ID, tools, MCP servers
- `updateTokens` - Real-time token usage
- `updateTotals` - Cost and token totals
- `ready` - Extension ready state
- `modelSelected` - Current model info

## Key Features Implementation

### File References (@)
**Location**: `extension.ts:1890-1938`

Searches workspace files and returns filtered results:
```typescript
const files = await vscode.workspace.findFiles(
  '**/*',
  '{**/node_modules/**,**/.git/**,...}',
  500
);
```

### Image Support
**Location**: `extension.ts:1940-1966` (file picker), `2340-2386` (paste/drop)

- Drag & drop support in webview
- Clipboard paste (Ctrl+V)
- Multiple image selection via VS Code picker
- Auto-saves to `.claude/claude-code-chat-images/`
- Supported formats: PNG, JPG, JPEG, GIF, SVG, WebP, BMP

### Model Selection
**Location**: `extension.ts:2214-2267`

- Supports: `opus`, `sonnet`, `default`
- Persisted in workspace state
- Passed to CLI via `--model` flag
- Visual confirmation in UI

### Slash Commands
**Location**: `extension.ts:2269-2304`

Executes Claude CLI commands in VS Code terminal:
```typescript
const terminal = vscode.window.createTerminal(`Claude /${command}`);
terminal.sendText(`claude /${command} --resume ${sessionId}`);
terminal.show();
```

### WSL Support
**Location**: `extension.ts:1788-1798` (path conversion), `518-531` (spawn)

- Path conversion: `C:\Users\...` → `/mnt/c/Users/...`
- Command wrapper: `wsl -d Ubuntu bash -ic "node ... claude ..."`
- Settings for distro, node path, claude path

### Thinking Modes
**Location**: `extension.ts:413-441`

Prepends mode instructions to user message:
- **Plan Mode**: "PLAN FIRST FOR THIS MESSAGE ONLY: ..."
- **Thinking Mode**: "THINK/THINK HARD/THINK HARDER/ULTRATHINK THROUGH THIS STEP BY STEP: ..."

Intensity levels controlled by `thinking.intensity` setting.

### Session Management
**Location**: `extension.ts:132-151`, `869-924`

- Sessions persist across extension restarts
- Resume via `--resume <session_id>` flag
- Session ID captured from CLI JSON stream
- Stored in conversation metadata

### Webview State Management
**Location**: `extension.ts:98-406`

- Manages both panel and sidebar webviews
- Auto-closes conflicting views (panel vs sidebar)
- Restores conversation on visibility change
- Message handler lifecycle management

## Extension Lifecycle

### Activation (`extension.ts:9-43`)
1. Create `ClaudeChatProvider` instance
2. Register `claude-code-chat.openChat` command
3. Register `claude-code-chat.loadConversation` command
4. Register webview view provider for sidebar
5. Listen for configuration changes (WSL settings)
6. Create status bar item
7. Initialize backup repo, conversations, MCP config

### Initialization Flow
1. **Backup Repository**: Git repo created in workspace storage
2. **Conversations Directory**: JSON storage for chat history
3. **MCP Config**: `mcp-servers.json` with permissions server
4. **Permission Watcher**: File system watcher for `*.request` files
5. **Load Latest Conversation**: Resume from last session

### Message Processing (`extension.ts:250-340`)
Routes webview messages to appropriate handlers:
- User messages → `_sendMessageToClaude()`
- Session control → `_newSession()`
- Restore → `_restoreToCommit()`
- Permissions → `_handlePermissionResponse()`
- Settings → `_updateSettings()`
- And 20+ other message types

## Storage & Persistence

### Workspace Storage
- **Backups**: `.git/` repository for checkpoints
- **Conversations**: JSON files with full history
- **MCP Config**: `mcp/mcp-servers.json`
- **Permissions**: `permission-requests/permissions.json`

### Workspace State
- `claude.conversationIndex` - Conversation metadata
- `claude.selectedModel` - Model preference

### Global State
- `customPromptSnippets` - Custom slash commands
- `wslAlertDismissed` - WSL alert preference

## Security Features

### Permission System
- All tool executions require approval (unless YOLO mode)
- Command pattern matching for Bash tools
- Workspace-specific permission storage
- Always-allow functionality with smart defaults

### Safe Experimentation
- Automatic checkpoints before changes
- Git-based restoration system
- No permanent data loss risk

## Performance Optimizations

### Streaming JSON Processing
Processes Claude CLI output line-by-line as it arrives:
```typescript
claudeProcess.stdout.on('data', (data) => {
  rawOutput += data.toString();
  const lines = rawOutput.split('\n');
  rawOutput = lines.pop() || '';

  for (const line of lines) {
    const jsonData = JSON.parse(line.trim());
    this._processJsonStreamData(jsonData);
  }
});
```

### Smart Auto-Scroll
Only auto-scrolls if user is near bottom of chat:
```javascript
function shouldAutoScroll(messagesDiv) {
  const threshold = 100; // pixels from bottom
  return (scrollTop + clientHeight >= scrollHeight - threshold);
}
```

### Conversation Truncation
- Limits conversation index to 50 entries
- First/last user messages stored for preview
- Full conversations stored as separate JSON files

## Error Handling

### Claude CLI Errors
- Login required detection → Opens terminal for auth
- Process spawn errors → Installation instructions
- Command not found → WSL configuration hints

### Permission Errors
- Detects permission-related errors
- Shows YOLO mode suggestion
- Graceful degradation

### Process Lifecycle
- Graceful termination (SIGTERM)
- Force kill after timeout (SIGKILL)
- Cleanup on process exit/error

## Extension Points

### Custom MCP Servers
Users can add any MCP server through UI:
- stdio (command + args)
- HTTP (URL + headers)
- SSE (URL + headers)

### Custom Slash Commands
Users can create custom prompt snippets:
- Stored in global state
- Accessible via `/` modal
- Reusable across workspaces

### Platform Integration
- Windows native support
- WSL integration with path conversion
- MacOS compatibility
- Linux support

## Future Enhancement Opportunities

1. **Multiple Model Support**: Beyond opus/sonnet/default
2. **Export Conversations**: Share or backup externally
3. **Custom Themes**: Beyond VS Code theme matching
4. **Voice Input**: Speech-to-text integration
5. **Collaborative Sessions**: Multi-user conversations
6. **Plugin System**: Community extensions
7. **Advanced Diff Views**: Side-by-side comparisons
8. **Search in Conversations**: Full-text search across history
