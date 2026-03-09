# IPC Architecture

T3 Code uses a multi-layered inter-process communication architecture to connect the React UI with backend services while maintaining security through process isolation.

## Problem

Desktop applications need secure communication between:
1. The browser-based UI (renderer process)
2. Native OS capabilities (Electron main process)
3. Backend services (Node.js server)
4. External AI providers (codex app-server)

Each boundary has different security and type-safety requirements.

## Architecture Layers

### Layer 1: Electron IPC (UI to Desktop)

Handles native desktop operations that require OS-level access:
- Folder picker dialogs
- Confirmation dialogs
- Native context menus
- Opening external URLs
- Auto-update management

**Type Safety**: One-directional. The preload script uses `satisfies DesktopBridge` to ensure the exposed API matches the interface, but the main process handlers accept `unknown` parameters and perform manual runtime validation.

### Layer 2: WebSocket RPC (UI to Server)

Handles all application logic through a WebSocket connection:
- Terminal session management
- Project file operations
- Git operations
- Orchestration commands
- Provider session control

**Type Safety**: Full bidirectional. Uses Effect/Schema for runtime validation on both client and server. Request/response types are defined in `packages/contracts/src/ws.ts`.

### Layer 3: Process Management (Electron to Server)

The Electron main process spawns and manages the Node.js server:
- Random port allocation on loopback interface
- 24-byte auth token generation
- Auto-restart with exponential backoff on crashes
- Log capture to rotating files

### Layer 4: External Provider (Server to Codex)

The server communicates with `codex app-server` via JSON-RPC over stdio:
- One process per provider session
- Structured event streaming
- Process lifecycle management

## Design Choices

### Why Two IPC Mechanisms?

1. **Electron IPC** is required for native OS operations that can't run in a browser context
2. **WebSocket** enables the same UI to work in both desktop and web modes with identical backend communication

### Why Minimal Type Safety on Electron IPC?

The Electron IPC surface is small (8 methods) and stable. The team chose to invest type-safety effort in the WebSocket layer which has 25+ RPC methods and evolves more frequently.

### Security Model

- Context isolation enabled (renderer can't access Node.js APIs)
- Token-based authentication on WebSocket connections
- URL validation before opening external links
- Sandbox mode for preload scripts

## Common Workflows

### Desktop Startup
1. Electron main generates auth token and allocates port
2. Server spawned as child process with token in environment
3. Main creates browser window with WebSocket URL in preload environment
4. Renderer connects to server via WebSocket with token auth

### Native Dialog
1. Renderer calls `window.desktopBridge.pickFolder()`
2. Preload forwards via `ipcRenderer.invoke()`
3. Main handles with `ipcMain.handle()`, shows native dialog
4. Result flows back through same path

### Server RPC
1. Renderer calls `nativeApi.git.status()`
2. WsTransport sends tagged JSON message
3. Server validates with Effect/Schema, executes handler
4. Response sent back, client resolves promise

## Error Handling

- **WebSocket disconnect**: Auto-reconnect with exponential backoff (500ms to 8s)
- **Server crash**: Electron restarts child process with backoff (500ms to 10s)
- **Invalid IPC message**: Logged and ignored (doesn't crash handler)
- **Schema validation failure**: Error response sent to client
