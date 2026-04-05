# OpenCode Android — Implementation Plan

> Flutter Android client for OpenCode servers.
> **External server by default. Local server is opt-in, started manually.**
> Designed for subagent-based execution with review gates to stay within 150k context.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Verified API Surface](#2-verified-api-surface)
3. [Technology Stack](#3-technology-stack)
4. [Project Structure](#4-project-structure)
5. [State Architecture](#5-state-architecture)
6. [Screen Designs](#6-screen-designs)
7. [Local Server — Companion APK (Sideload)](#7-local-server--companion-apk-sideload)
8. [mDNS Discovery](#8-mdns-discovery)
9. [Terminal Panel](#9-terminal-panel)
10. [Android Permissions](#10-android-permissions)
11. [Subagent Execution Strategy](#11-subagent-execution-strategy)
12. [What is NOT in This Plan](#12-what-is-not-in-this-plan)

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                     Flutter App                          │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐ │
│  │ Servers  │  │  Chat    │  │ Sessions │  │Settings │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬────┘ │
│       │              │             │               │      │
│  ┌────▼──────────────▼─────────────▼───────────────▼────┐│
│  │                 Riverpod State Layer                  ││
│  │   activeServer → dioClient → SSE → feature providers ││
│  └───────────────────────┬───────────────────────────────┘│
│                          │                                │
│  ┌───────────────────────▼───────────────────────────────┐│
│  │           OpenCode API Client (Dio)                   ││
│  │  HTTP Basic Auth · SSE streaming · REST calls         ││
│  └───────────────────────┬───────────────────────────────┘│
└──────────────────────────┼──────────────────────────────-─┘
                           │
          ┌────────────────┼──────────────────┐
          │                │                  │
     ┌────▼────┐    ┌──────▼──────┐    ┌──────▼──────┐
     │ Remote  │    │ LAN Server  │    │ Local On-   │
     │ Server  │    │ (mDNS disc.)│    │ Device Srvr │
     │ (VPS)   │    │              │    │ (opt-in,    │
     └─────────┘    └─────────────┘    │  manual)    │
                                       └─────────────┘
```

**Core principle:** The app is a *client* to an OpenCode server. It renders the AI chat, manages sessions, responds to permission requests, and displays tool output. It does **not** require a local server to function. External/remote servers are the default and primary use case. The local on-device server is a power-user feature that must be manually started.

---

## 2. Verified API Surface

> All endpoints verified against OpenCode docs and source. The `/doc` endpoint on a running server provides the authoritative OpenAPI 3.1 spec.

### 2.1 REST Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/global/health` | Server health check |
| `GET` | `/global/event` | SSE event stream (persistent) |
| `GET` | `/session` | List all sessions |
| `POST` | `/session` | Create new session |
| `GET` | `/session/:id` | Get session details |
| `DELETE` | `/session/:id` | Delete session |
| `POST` | `/session/:id/message` | Send a prompt |
| `GET` | `/session/:id/message` | Get message history |
| `POST` | `/question/:requestID/reply` | Reply to permission request |
| `GET` | `/agent` | List available agents |
| `GET` | `/config/providers` | List providers + models |
| `PUT` | `/auth/:providerID` | Configure provider auth |
| `GET` | `/mcp` | MCP server status |

### 2.2 SSE Event Types (via `/global/event`)

All events are JSON: `{ "type": "...", "properties": { ... } }`

**Session:**

| Event | When |
|-------|------|
| `session.created` | New session initialized |
| `session.updated` | Session state changed |
| `session.idle` | Session finished processing |
| `session.deleted` | Session removed |
| `session.error` | Error in session |
| `session.compacted` | Context pruned |
| `session.diff` | Diff info available |

**Messages:**

| Event | When |
|-------|------|
| `message.updated` | Message changed |
| `message.removed` | Message deleted |
| `message.part.updated` | Streaming text chunk |
| `message.part.removed` | Part removed |

**Permissions:**

| Event | When |
|-------|------|
| `permission.asked` | Agent needs user approval |
| `permission.replied` | Answer processed, agent resumes |

**Tools:**

| Event | When |
|-------|------|
| `tool.execute.before` | Tool invocation starting |
| `tool.execute.after` | Tool completed |

**Other:**

| Event | When |
|-------|------|
| `file.edited` | Agent modified a file |
| `todo.updated` | Agent todo list changed |
| `installation.updated` | Server config changed |
| `lsp.client.diagnostics` | LSP diagnostic info |
| `server.connected` | Server established connection |

### 2.3 Auth Model

- HTTP Basic Auth on every request
- Header: `Authorization: Basic base64(opencode:<OPENCODE_SERVER_PASSWORD>)`
- Username is always `opencode`; password from `OPENCODE_SERVER_PASSWORD` env var
- Stored in `flutter_secure_storage` (never in Hive)

---

## 3. Technology Stack

| Layer | Package | Version | Notes |
|-------|---------|---------|-------|
| **State** | `riverpod` + `flutter_riverpod` | ^2.5 | `StreamProvider` for SSE, `AsyncNotifier` for REST |
| **HTTP** | `dio` | ^5.4 | Auth interceptor, `ResponseType.stream` for SSE |
| **Routing** | `go_router` | ^14 | Shell route for bottom nav tabs |
| **Markdown** | `markdown_widget` | ^2.3.2 | Custom code block builder; replaces deprecated `flutter_markdown` |
| **Syntax highlight** | `flutter_highlight` | ^0.7.0 | Code fence rendering inside markdown |
| **mDNS** | `multicast_dns` | ^0.3 | `_opencode._tcp` service discovery |
| **Local storage** | `hive_flutter` | ^1.1 | Server profiles, agent drafts, prefs |
| **Secure storage** | `flutter_secure_storage` | ^9.0 | Passwords + API keys only |
| **Icons** | `lucide_icons` | latest | Matches OpenCode desktop aesthetic |
| **Animations** | `flutter_animate` | ^4.5 | Streaming indicators, token counter |
| **Terminal** | `xterm` | ^3.4 | Full VT100 terminal for local shell + server logs |
| **Fonts** | Google Fonts `Inter` / `JetBrains Mono` | — | Body / code |
| **JSON** | `json_serializable` + `build_runner` | latest | Hand-written models with code-gen serialization |

> **No `freezed` in v1.** The OpenCode API contract is still evolving. `freezed` adds build_runner complexity that isn't worth it until the API stabilizes. Can add in v2.

---

## 4. Project Structure

```
opencode_android/
├── PLAN.md                               ← This file
│
├── lib/
│   ├── main.dart
│   ├── app.dart                          # MaterialApp + GoRouter + ProviderScope
│   │
│   ├── core/
│   │   ├── api/
│   │   │   ├── opencode_client.dart      # Dio factory + auth interceptor
│   │   │   ├── sse_manager.dart          # SSE lifecycle + exponential backoff reconnect
│   │   │   ├── sse_parser.dart           # Raw SSE lines → typed SseEvent objects
│   │   │   └── endpoints.dart            # All typed REST call methods
│   │   │
│   │   ├── models/
│   │   │   ├── server_profile.dart       # name, host, port, tls, isLocal
│   │   │   ├── session.dart
│   │   │   ├── message.dart              # Message + Part (text, tool, error, image)
│   │   │   ├── agent.dart
│   │   │   ├── sse_event.dart            # Union-style for all SSE event types
│   │   │   ├── mcp_status.dart
│   │   │   ├── token_usage.dart          # in/out/total/contextPercent
│   │   │   └── question.dart             # Permission request (permission.asked)
│   │   │
│   │   ├── providers/
│   │   │   ├── server_providers.dart     # serverProfilesProvider, activeServerProvider, dioClientProvider
│   │   │   ├── session_providers.dart    # sessionsProvider, activeSessionProvider
│   │   │   ├── message_providers.dart    # messages + streaming via SSE
│   │   │   ├── agent_providers.dart      # agentsProvider, activeAgentProvider
│   │   │   ├── sse_providers.dart        # sseManagerProvider, sseEventStreamProvider + derived streams
│   │   │   ├── token_providers.dart      # tokenUsageProvider
│   │   │   └── mcp_providers.dart        # mcpStatusProvider
│   │   │
│   │   ├── discovery/
│   │   │   └── mdns_scanner.dart         # Scans _opencode._tcp with multicast lock
│   │   │
│   │   └── local_server/
│   │       └── server_manager.dart       # Process lifecycle for on-device binary
│   │
│   ├── features/
│   │   ├── servers/
│   │   │   ├── server_list_screen.dart   # Server list + mDNS scan + health dots
│   │   │   ├── add_server_sheet.dart     # Manual entry bottom sheet
│   │   │   └── server_tile.dart
│   │   │
│   │   ├── chat/
│   │   │   ├── chat_screen.dart          # Main chat view
│   │   │   ├── message_list.dart         # Scrollable message list
│   │   │   ├── message_bubble.dart       # User / assistant bubbles
│   │   │   ├── chat_input_bar.dart       # Text input + action buttons
│   │   │   ├── permission_card.dart      # Inline permission request (NOT a modal)
│   │   │   ├── tool_pill.dart            # Collapsible tool invocation badge
│   │   │   └── streaming_cursor.dart     # Animated typing indicator
│   │   │
│   │   ├── sessions/
│   │   │   ├── session_list_screen.dart
│   │   │   └── session_tile.dart
│   │   │
│   │   ├── agents/
│   │   │   ├── agent_list_screen.dart    # Browse server agents + local drafts
│   │   │   ├── agent_editor_screen.dart  # YAML form + markdown body editor
│   │   │   └── agent_tile.dart
│   │   │
│   │   ├── terminal/
│   │   │   └── terminal_screen.dart      # xterm shell + local server log mode
│   │   │
│   │   ├── local_server/
│   │   │   ├── local_server_screen.dart  # Start/stop + download status
│   │   │   └── api_key_setup_sheet.dart  # First-run API key entry
│   │   │
│   │   ├── mcp/
│   │   │   └── mcp_status_screen.dart
│   │   │
│   │   └── settings/
│   │       ├── settings_screen.dart
│   │       └── provider_auth_screen.dart # PUT /auth/:id on remote server
│   │
│   └── shared/
│       ├── widgets/
│       │   ├── code_block.dart           # flutter_highlight wrapper with copy button
│       │   ├── token_badge.dart          # ↑N ↓N XX% pill — color-coded by fill %
│       │   ├── connection_banner.dart    # Offline/reconnecting persistent banner
│       │   └── animated_shimmer.dart     # Loading placeholder
│       │
│       └── theme/
│           ├── app_theme.dart            # Material 3 dark + light ThemeData
│           ├── colors.dart               # Full palette (no hardcoded colors elsewhere)
│           └── typography.dart           # Text styles (Inter body, JetBrains Mono code)
│
├── android/
│   └── app/src/main/
│       ├── kotlin/.../MainActivity.kt    # Platform channels for nativeLibraryDir + multicast lock
│       └── AndroidManifest.xml           # REQUEST_INSTALL_PACKAGES + FileProvider
│
├── companion/                            # Separate project — opencode-server.apk
│   └── android/app/src/main/
│       ├── jniLibs/arm64-v8a/
│       │   └── libopencode.so            # The actual binary (GOOS=android GOARCH=arm64)
│       ├── kotlin/.../ServerService.kt   # Bound AIDL service exposing start/stop/status
│       └── IOpenCodeServer.aidl          # AIDL interface definition
│
└── pubspec.yaml
```

---

## 5. State Architecture

### 5.1 Provider Dependency Graph

```
serverProfilesProvider (NotifierProvider)
  └── activeServerProvider (NotifierProvider)
        └── dioClientProvider (Provider<OpenCodeClient>)
              ├── sseManagerProvider (NotifierProvider)
              │     └── sseEventStreamProvider (StreamProvider<SseEvent>)
              │           ├── messageStreamProvider.family (filtered by sessionId)
              │           ├── permissionStreamProvider (permission.asked events)
              │           ├── toolEventProvider (tool.execute.before/after)
              │           └── sessionUpdateProvider (session.* events)
              ├── sessionsProvider (AsyncNotifierProvider)
              │     └── activeSessionProvider (NotifierProvider)
              │           └── tokenUsageProvider (NotifierProvider)
              ├── agentsProvider (FutureProvider)
              │     └── activeAgentProvider (NotifierProvider)
              └── mcpStatusProvider (FutureProvider)
```

**Key rules:**
- When `activeServerProvider` changes → `dioClientProvider` rebuilds → SSE reconnects → all providers refresh. This is intentional; no cross-server state leakage.
- SSE is **one stream per active server**, demuxed by event type and session ID.
- Passwords are never stored in Riverpod state — always fetched from `flutter_secure_storage` at connection time.

### 5.2 SSE Demuxing Pattern

```dart
// sse_providers.dart
final sseEventStreamProvider = StreamProvider<SseEvent>((ref) {
  final manager = ref.watch(sseManagerProvider.notifier);
  return manager.eventStream; // reconnect-aware stream
});

// Derived — only message events for one session
final messageStreamProvider = StreamProvider.family<SseEvent, String>((ref, sessionId) {
  return ref.watch(sseEventStreamProvider.stream)
      .where((e) => e.isMessageEvent && e.sessionId == sessionId);
});

// Derived — permission requests (any session)
final permissionStreamProvider = StreamProvider<QuestionEvent>((ref) {
  return ref.watch(sseEventStreamProvider.stream)
      .where((e) => e.type == 'permission.asked')
      .map((e) => QuestionEvent.fromJson(e.properties));
});
```

### 5.3 SSE Reconnection

```dart
class SseManager extends Notifier<SseConnectionState> {
  // Backoff: 1s → 2s → 4s → 8s → 16s → max 30s
  // Resume: store last received event ID; send as query param on reconnect
  // Pre-health-check: verify /global/health before attempting SSE reconnect
  // On server switch: cancel current stream, reset backoff, reconnect
}
```

---

## 6. Screen Designs

### 6.1 Chat Screen (Primary)

```
┌──────────────────────────────────────────┐
│  [≡]  myserver.local     [claude-4 ▾]    │  AppBar: server + agent picker
│                     [↑2.1k ↓4.8k  12%]  │  Token badge (tap → detail sheet)
├──────────────────────────────────────────┤
│                                          │
│  ┌─ You ─────────────────────────────┐   │
│  │ Fix the JWT refresh bug           │   │
│  └───────────────────────────────────┘   │
│                                          │
│  ┌─ Assistant ───────────────────────┐   │
│  │ I'll look at the refresh logic.   │   │
│  │                                    │   │
│  │  ┌─ 🔧 bash ────────────────┐    │   │  ← Tool pill (tap to expand)
│  │  │ cat src/auth/refresh.ts   │    │   │
│  │  └───────────────────────────┘    │   │
│  │                                    │   │
│  │  ```typescript                     │   │  ← Syntax highlighted code block
│  │  const refresh = async () => {     │   │
│  │    const token = await getToken(); │   │
│  │  }                                 │   │
│  │  ```                               │   │
│  │                                    │   │
│  │  ┌─ ⚠ Permission Required ────┐   │   │  ← Inline card, NOT a modal
│  │  │ Run: npm test               │   │   │
│  │  │ [Allow]  [Deny]  [Always]   │   │   │
│  │  └─────────────────────────────┘   │   │
│  │                                    │   │
│  │ █                                  │   │  ← Streaming cursor
│  └───────────────────────────────────┘   │
│                                          │
├──────────────────────────────────────────┤
│  [/cmd]  [@agent]  [⚑ MCP]  [>_ Term]   │  Action bar
│  ┌──────────────────────────────┐  [⏎]  │
│  │ Ask OpenCode anything...     │        │
│  └──────────────────────────────┘        │
└──────────────────────────────────────────┘
```

**Token badge behavior:**
- `↑N ↓N XX%` — input tokens, output tokens, context fill %
- Green `<50%`, Amber `50-80%`, Red `>80%` (pulse animation at red)
- Tap → opens detail sheet with full breakdown + model context window size

**Permission card behavior:**
- Inserted inline in message flow when `permission.asked` SSE event arrives
- [Allow] → `POST /question/:requestID/reply` `{ allow: true }`
- [Deny] → same with `{ allow: false }`
- [Always] → allow + sets global permission rule in settings

**Input bar:**
- Disabled + shows spinner while `session.idle` not yet received
- Enter to send, Shift+Enter for newline
- `/` triggers slash command autocomplete
- `@` triggers agent mention picker

### 6.2 Server List Screen

```
┌──────────────────────────────────────────┐
│  Servers                        [Scan 📡]│
├──────────────────────────────────────────┤
│  ● Home Lab              [Connected] ◉   │  ← Active server, green dot
│    opencode.local:4096          [mDNS]   │
│                                          │
│  ○ VPS                  [Unreachable] ✕  │  ← Health check failed
│    myserver.com:4096          [Manual]   │
│                                          │
│  ○ Local Device              [Stopped]   │  ← Special card
│    127.0.0.1:4096                        │
│    [Download Binary]  or  [Start ▶]      │  ← Download first, then start
│                                          │
│  ┌──────────────────────────────────────┐│
│  │ [+ Add Server Manually]              ││
│  └──────────────────────────────────────┘│
└──────────────────────────────────────────┘
```

### 6.3 Navigation Structure

Bottom navigation — 4 tabs:

| Tab | Route | Icon |
|-----|-------|------|
| Chat | `/chat` | `MessageSquare` |
| Sessions | `/sessions` | `History` |
| Agents | `/agents` | `Bot` |
| Settings | `/settings` | `Settings` |

Settings tab includes: Servers, MCP Status, Terminal, Provider Auth, App Preferences.

---

## 7. Local Server — Companion APK (Sideload)

### 7.1 Why Not Bundle or Use Play Store

| Option | Problem |
|--------|--------|
| Bundle in main APK | +60-80MB for all users who may never use it |
| Play Feature Delivery (DFM) | **Requires Play Store** — sideloaded APKs cannot trigger DFM downloads |
| Download binary to `filesDir` | **Blocked by SELinux W^X** on Android 10+ — `execve()` is denied |

**Solution: Companion APK** — a separate, small (~70MB) APK (`opencode-server.apk`) that:
- Bundles `libopencode.so` in its own `jniLibs/arm64-v8a/` (installed into its `nativeLibraryDir` by Android)
- Exposes a **Bound AIDL Service** so the main app can tell it to start/stop the server and stream logs
- Is **downloaded on demand** from a URL you host (GitHub Releases, your server, etc.)
- Is **installed via the system package installer** (1-tap user approval prompt)

This is fully sideload-compatible, Play-Store-compatible, and SELinux-safe.

### 7.2 Installation Flow

```
User taps [Install Server Companion]
    → Main app checks: is com.opencode.server installed?
        YES → show [Start ▶]
        NO  →
            1. Download opencode-server.apk from configured URL (~70MB)
               (shown as progress bar in the local server card)
            2. Use FileProvider + ACTION_VIEW intent → system installer dialog
            3. User taps [Install] once in system dialog
            4. Android installs APK, extracts libopencode.so to companion's nativeLibraryDir
            5. Main app detects companion installed → shows [Start ▶]

User taps [Start ▶]
    → Main app binds to ServerService in companion APK via AIDL
    → Calls service.start(port, workDir, apiKeys)
    → Companion process spawns opencode serve inside its own sandbox
    → Logs stream back to main app via AIDL callback
    → Main app connects to http://127.0.0.1:4096 as a normal server profile
```

### 7.3 Companion APK — Separate Project

The companion is a minimal Android app (Kotlin-only, no Flutter) that acts as a process host:

```kotlin
// IOpenCodeServer.aidl  (shared interface)
interface IOpenCodeServer {
    void start(int port, String workDir);   // starts opencode serve
    void stop();                            // kills process
    int getStatus();                        // 0=stopped, 1=starting, 2=running, 3=error
    int getPid();
    void registerCallback(IServerCallback cb);
}

interface IServerCallback {
    void onLog(String line);               // stdout/stderr streaming
    void onStatusChanged(int status);
}
```

```kotlin
// ServerService.kt in companion APK
class ServerService : Service() {
    private var process: Process? = null

    private val binder = object : IOpenCodeServer.Stub() {
        override fun start(port: Int, workDir: String) {
            val binary = "${applicationInfo.nativeLibraryDir}/libopencode.so"
            // API keys passed via IPC from main app's secure storage
            process = ProcessBuilder(binary, "serve", "--port", "$port",
                                     "--hostname", "127.0.0.1")
                .directory(File(workDir))
                .start()
            // Stream stdout/stderr back via registered callbacks
        }
        override fun stop() { process?.destroy() }
    }

    override fun onBind(intent: Intent) = binder
}
```

```xml
<!-- companion AndroidManifest.xml -->
<manifest package="com.opencode.server">
  <application android:extractNativeLibs="true">
    <service android:name=".ServerService"
             android:exported="true"
             android:permission="com.opencode.permission.BIND_SERVER" />
  </application>
</manifest>
```

### 7.4 Main App — Dart Side

```dart
class LocalServerManager extends Notifier<LocalServerState> {
  static const _channel = MethodChannel('com.opencode/native');

  // Check if companion APK is installed
  Future<bool> get isCompanionInstalled async {
    return await _channel.invokeMethod<bool>('isCompanionInstalled') ?? false;
  }

  // Download companion APK from URL, then trigger system installer
  Future<void> downloadAndInstall(String apkUrl) async {
    state = LocalServerState.downloading(progress: 0);
    // Stream download progress via method channel
    await _channel.invokeMethod('downloadAndInstallCompanion', {'url': apkUrl});
    // After install, state updates to 'installed' via event channel
  }

  // Bind AIDL service and start the server
  Future<void> start({int port = 4096}) async {
    final workDir = await _getWorkspaceDir();
    final keys = await _loadApiKeys(); // from flutter_secure_storage only
    await _channel.invokeMethod('startServer', {
      'port': port,
      'workDir': workDir,
      'apiKeys': keys,  // passed over IPC, never logged
    });
    state = LocalServerState.running(port: port);
  }

  Future<void> stop() async {
    await _channel.invokeMethod('stopServer');
    state = const LocalServerState.stopped();
  }

  // Logs stream from companion via EventChannel
  Stream<String> get logStream =>
    const EventChannel('com.opencode/server_logs').receiveBroadcastStream()
        .map((e) => e as String);

  Future<String> _getWorkspaceDir() async {
    final docs = await getApplicationDocumentsDirectory();
    final ws = Directory('${docs.path}/opencode-workspace');
    if (!await ws.exists()) await ws.create(recursive: true);
    return ws.path;
  }
}
```

### 7.5 Main App — Kotlin Platform Channel

```kotlin
// MainActivity.kt — local server platform channel
"isCompanionInstalled" -> {
    val installed = try {
        packageManager.getPackageInfo("com.opencode.server", 0)
        true
    } catch (e: PackageManager.NameNotFoundException) { false }
    result.success(installed)
}

"downloadAndInstallCompanion" -> {
    val url = call.argument<String>("url")!!
    // Download to getExternalFilesDir() then:
    val apkUri = FileProvider.getUriForFile(this, "$packageName.provider", apkFile)
    val intent = Intent(Intent.ACTION_VIEW).apply {
        setDataAndType(apkUri, "application/vnd.android.package-archive")
        flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_GRANT_READ_URI_PERMISSION
    }
    startActivity(intent)
    result.success(null)
}

"startServer" -> {
    val port = call.argument<Int>("port")!!
    val workDir = call.argument<String>("workDir")!!
    val apiKeys = call.argument<Map<String, String>>("apiKeys")!!
    // Bind to companion service via AIDL
    val intent = Intent().apply {
        component = ComponentName("com.opencode.server", "com.opencode.server.ServerService")
    }
    bindService(intent, serverConnection, Context.BIND_AUTO_CREATE)
    // serverConnection.onServiceConnected → call service.start(port, workDir)
    result.success(null)
}
```

### 7.6 Android Permissions (Main App)

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.REQUEST_INSTALL_PACKAGES" />
<uses-permission android:name="android.permission.CHANGE_WIFI_MULTICAST_STATE" />

<!-- FileProvider for sharing downloaded APK with system installer -->
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.provider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/provider_paths" />
</provider>
```

### 7.7 Compile Companion Binary

```bash
# Target Android ABI (not linux)
GOOS=android GOARCH=arm64 \
  go build -ldflags="-s -w" \
  -o companion/android/app/src/main/jniLibs/arm64-v8a/libopencode.so \
  ./cmd/opencode

# Build companion APK
cd companion && ./gradlew assembleRelease
# Output: companion/app/build/outputs/apk/release/opencode-server.apk
# Host this file at a stable URL (e.g. GitHub Releases)
```

### 7.8 Local Server UX Rules

- Default state: **Stopped**.
- Card shows:
  - `[Install Server Companion]` → if companion APK not installed
  - `[Start ▶]` → if companion installed but server stopped
  - Green pulsing dot + port + `[Stop ■]` → if running
- **Never auto-starts.** User must tap [Start ▶] explicitly.
- Download shows a progress bar (bytes downloaded / total) within the server card.
- First start: opens `ApiKeySetupSheet` if no keys in `flutter_secure_storage`.
- Logs from companion stream to the terminal screen's xterm via `EventChannel`.
- API keys are passed over IPC — never logged, never stored in Hive.

---

## 8. mDNS Discovery

Placed in **Phase 4** (alongside server UI) since it needs the same platform channel infrastructure.

### 8.1 Service Type

OpenCode broadcasts `_opencode._tcp` (note: no `.local` in the query string — `.local` is the DNS-SD domain, not part of the service type).

### 8.2 Dart Scanner

```dart
Stream<ServerProfile> scanForServers() async* {
  await _channel.invokeMethod('acquireMulticastLock'); // REQUIRED on Android
  final client = MDnsClient();
  await client.start();

  try {
    await for (final ptr in client.lookup<PtrResourceRecord>(
      ResourceRecordQuery.serverPointer('_opencode._tcp'),
    )) {
      await for (final srv in client.lookup<SrvResourceRecord>(
        ResourceRecordQuery.service(ptr.domainName),
      )) {
        yield ServerProfile(
          name: ptr.domainName.split('.').first,
          host: srv.target,
          port: srv.port,
          discoveryMethod: DiscoveryMethod.mdns,
        );
      }
    }
  } finally {
    client.stop();
    await _channel.invokeMethod('releaseMulticastLock');
  }
}
```

### 8.3 Platform Channel (Kotlin)

```kotlin
// MainActivity.kt
private var multicastLock: WifiManager.MulticastLock? = null

MethodChannel(flutterEngine.dartExecutor.binaryMessenger, "com.opencode/native")
    .setMethodCallHandler { call, result ->
        when (call.method) {
            "getNativeLibDir" -> result.success(applicationInfo.nativeLibraryDir)
            "acquireMulticastLock" -> {
                val wifi = getSystemService(WIFI_SERVICE) as WifiManager
                multicastLock = wifi.createMulticastLock("opencode_mdns").also { it.acquire() }
                result.success(null)
            }
            "releaseMulticastLock" -> {
                multicastLock?.release()
                multicastLock = null
                result.success(null)
            }
            "installLocalServerFeature" -> {
                // SplitInstallManager.startInstall(...)
                result.success(null)
            }
            else -> result.notImplemented()
        }
    }
```

---

## 9. Terminal Panel

The terminal panel has **two modes**:

### Mode 1 — Local Server Logs

When a local on-device server is running, the terminal shows its stdout/stderr output piped to an `xterm` widget. This gives proper ANSI color support for Go's log output.

```dart
// server_manager.dart
_process!.stdout.transform(utf8.decoder).listen((data) {
  terminalController.write(data); // feed xterm
});
```

### Mode 2 — Local Device Shell

A basic interactive shell on the Android device using the system shell:

```dart
Future<void> startShell() async {
  final shell = await Process.start(
    '/system/bin/sh',
    [],
    environment: Platform.environment,
  );
  // Bidirectional: xterm input → shell stdin, shell stdout → xterm
}
```

> This works on Android (system binaries don't have execution restrictions). It gives the user a real shell on their device — useful for managing the OpenCode workspace directory, checking files, etc.

### Mode 3 — Tool Output Replay

When no local server is running, the terminal can show a replay of tool invocation outputs from the current session (bash commands the agent ran, their output). This is read from message history, not a live connection.

### Navigation

Terminal is accessible from:
1. Chat screen action bar `[>_ Term]` button
2. Settings → Terminal

---

## 10. Android Permissions

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.CHANGE_WIFI_MULTICAST_STATE" />
<!-- No WRITE_EXTERNAL_STORAGE needed — workspace is in app documents dir -->
```

The `CHANGE_WIFI_MULTICAST_STATE` permission enables mDNS multicast scanning.

---

## 11. Subagent Execution Strategy

> This project is built using **multiple subagents** with a 150k context limit per conversation. Each subagent gets a focused task. Review agents validate work after each phase.

### 11.1 Phase Map

```
Phase 1  →  [Scaffold Agent]        → project setup, theme, routing skeleton
Phase 2  →  [API Client Agent]      → Dio, SSE parser, all REST methods, models
Phase 3  →  [State Agent]           → all Riverpod providers, dependency graph
              ↓
         [Review Gate 1]            → models, endpoints, provider graph, auth
              ↓
Phase 4  →  [Server UI Agent]       → server list, add server, mDNS, health dots
Phase 5  →  [Chat UI Agent]         → chat screen, bubbles, code blocks, permission cards, token badge
              ↓
         [Review Gate 2]            → server + chat UI correctness, navigation
              ↓
Phase 6  →  [Sessions + Agents Agent] → session browser, agent editor (local Hive store)
Phase 7  →  [SSE Integration Agent]   → wire streaming to live UI, demuxing, auto-scroll
              ↓
         [Review Gate 3]            → live SSE correctness, reconnect, permission flow
              ↓
Phase 8  →  [Platform Agent]        → mDNS multicast lock, companion APK download/install flow, AIDL service binding
Phase 9  →  [Terminal Agent]        → xterm: local shell + log mode + tool replay mode
              ↓
         [Review Gate 4]            → platform features, terminal modes, build passes
              ↓
Phase 10 →  [Theme + Polish Agent]  → dark mode, M3 colors, animations, haptics, empty states
Phase 11 →  [Testing Agent]         → unit tests (parser, client), widget tests, integration test
              ↓
         [Final Review]             → full audit checklist below
```

### 11.2 Subagent Task Template

```markdown
## Task: [Phase Name]

### Context
Building the [component] of the OpenCode Android Flutter client.
Project root: c:\Users\thies\Antigravity\opencode-android
Read PLAN.md for full context.

### Read First
[List exact files to read for interface context]

### Create / Modify
[Exact file paths with one-line description]

### Interface Contract
[Paste exact function signatures / provider names the subagent must expose]

### Rules
[Relevant UX/architecture rules from this plan]

### Verification
1. Run `flutter analyze` — must show 0 errors
2. [Phase-specific checks]

### Return
Files created/modified, issues encountered, deviations from plan.
```

---

### 11.3 Phase-by-Phase Contracts

---

#### Phase 1 — Scaffold `[subagent: scaffold]`

**Creates:**
- `pubspec.yaml` — all dependencies from §3
- `lib/main.dart` — ProviderScope + runApp
- `lib/app.dart` — MaterialApp with GoRouter (4 shell routes: `/chat`, `/sessions`, `/agents`, `/settings`)
- `lib/shared/theme/app_theme.dart` — Material 3, dark + light
- `lib/shared/theme/colors.dart` — full palette (OpenCode-inspired dark/teal)
- `lib/shared/theme/typography.dart` — Inter + JetBrains Mono
- `android/app/src/main/AndroidManifest.xml` — permissions, FileProvider config
- `android/app/src/main/kotlin/.../MainActivity.kt` — platform channel stubs (returns notImplemented for now)
- `android/app/src/main/res/xml/provider_paths.xml` — FileProvider paths config

**Verification:** `flutter pub get` passes. `flutter analyze` 0 errors. App shows bottom nav.

---

#### Phase 2 — API Client `[subagent: api_client]`

**Creates:**
- `lib/core/api/opencode_client.dart`
- `lib/core/api/sse_parser.dart`
- `lib/core/api/sse_manager.dart`
- `lib/core/api/endpoints.dart`
- All model files under `lib/core/models/`

**Must expose:**
```dart
class OpenCodeClient {
  OpenCodeClient({required String baseUrl, String? username, String? password});
  Dio get dio;
  Future<List<Session>> listSessions();
  Future<Session> createSession();
  Future<Session> getSession(String id);
  Future<void> deleteSession(String id);
  Future<void> sendMessage(String sessionId, String text, {String? agentId});
  Future<List<Message>> getMessages(String sessionId);
  Future<List<Agent>> listAgents();
  Future<Map<String, dynamic>> listProviders();
  Future<void> replyToQuestion(String requestId, {required bool allow});
  Future<Map<String, McpStatus>> getMcpStatus();
  Future<bool> checkHealth();
}

class SseManager {
  Stream<SseEvent> get eventStream; // reconnects automatically
  void connect(Dio dio);
  void disconnect();
}
```

**SSE event endpoint:** `GET /global/event`
**Permission reply endpoint:** `POST /question/:requestID/reply`

---

#### Phase 3 — State Layer `[subagent: state]`

**Creates:** All files under `lib/core/providers/`

**Must expose (exact names — other subagents will watch these):**
```dart
final serverProfilesProvider = NotifierProvider<ServerProfileNotifier, List<ServerProfile>>();
final activeServerProvider = NotifierProvider<ActiveServerNotifier, ServerProfile?>();
final dioClientProvider = Provider<OpenCodeClient>();                // rebuilds on activeServer change
final sseManagerProvider = NotifierProvider<SseManagerNotifier, SseConnectionState>();
final sseEventStreamProvider = StreamProvider<SseEvent>();
final sessionsProvider = AsyncNotifierProvider<SessionsNotifier, List<Session>>();
final activeSessionProvider = NotifierProvider<ActiveSessionNotifier, Session?>();
final messageStreamProvider = StreamProvider.family<SseEvent, String>(); // family = sessionId
final permissionStreamProvider = StreamProvider<QuestionEvent>();
final agentsProvider = FutureProvider<List<Agent>>();
final activeAgentProvider = NotifierProvider<ActiveAgentNotifier, Agent?>();
final tokenUsageProvider = NotifierProvider<TokenUsageNotifier, TokenUsage>();
final mcpStatusProvider = FutureProvider<Map<String, McpStatus>>();
```

---

#### 🔍 Review Gate 1 `[subagent: review_1]`

```
Read: all files from Phase 1, 2, 3.
Check:
[ ] All model classes have fromJson / toJson
[ ] SSE events use strings from §2.2 exactly
[ ] Auth header: Authorization: Basic base64(opencode:<password>)
[ ] SSE endpoint: /global/event (not /event)
[ ] Permission reply: POST /question/:id/reply (not /session/.../permissions/...)
[ ] SSE backoff: 1s→2s→4s→8s→16s→30s max
[ ] dioClientProvider invalidates on activeServerProvider change
[ ] No circular provider dependencies
[ ] flutter analyze clean
Output: numbered list of issues.
```

---

#### Phase 4 — Server UI + mDNS `[subagent: server_ui]`

**Creates:**
- `lib/features/servers/server_list_screen.dart`
- `lib/features/servers/add_server_sheet.dart`
- `lib/features/servers/server_tile.dart`
- `lib/core/discovery/mdns_scanner.dart`
- `android/.../MainActivity.kt` — implement `acquireMulticastLock`, `releaseMulticastLock`, `getNativeLibDir`

**Rules:**
- Health status dots: green (healthy), amber (slow response >2s), red (unreachable)
- Health polling: every 30s via `GET /global/health`
- Local server card: always at bottom, shows [Download Binary] or [Start ▶] but NEVER auto-starts
- mDNS: animated scanning icon, results appear in real-time, auto-deduplication

---

#### Phase 5 — Chat UI `[subagent: chat_ui]`

**Creates:**
- `lib/features/chat/chat_screen.dart`
- `lib/features/chat/message_list.dart`
- `lib/features/chat/message_bubble.dart`
- `lib/features/chat/chat_input_bar.dart`
- `lib/features/chat/permission_card.dart`
- `lib/features/chat/tool_pill.dart`
- `lib/features/chat/streaming_cursor.dart`
- `lib/shared/widgets/code_block.dart`
- `lib/shared/widgets/token_badge.dart`
- `lib/shared/widgets/connection_banner.dart`

**Rules:**
- User bubbles: right-aligned, theme primary container color
- Assistant bubbles: left-aligned, theme surface variant color
- Code blocks: `flutter_highlight` dark theme, copy button top-right, language label top-left
- Markdown: `markdown_widget` with custom code block element builder
- Permission card: inline in message list (not a modal dialog)
- Tool pills: collapsed by default, show name + "completed"/"running", tap to expand output
- Token badge: in AppBar, color threshold green/amber/red, tap for detail
- Input: disabled with CircularProgressIndicator while streaming, re-enabled on `session.idle`

---

#### 🔍 Review Gate 2 `[subagent: review_2]`

```
Read: Phase 4 + 5 files.
Check:
[ ] Server health status updates on each poll
[ ] mDNS results append without full list rebuild
[ ] Permission card has Allow/Deny buttons calling correct endpoint
[ ] Token badge color reflects correct thresholds
[ ] Chat input disabled during streaming
[ ] Code blocks copy correctly
[ ] GoRouter bottom nav tabs work
[ ] No hardcoded colors — all from theme
[ ] Responsive at 360px and 414px widths
```

---

#### Phase 6 — Sessions + Agents UI `[subagent: sessions_agents]`

**Creates:**
- `lib/features/sessions/session_list_screen.dart`
- `lib/features/sessions/session_tile.dart`
- `lib/features/agents/agent_list_screen.dart`
- `lib/features/agents/agent_editor_screen.dart`
- `lib/features/agents/agent_tile.dart`

**Agent editor design:**
- Top: form fields for `description`, `model` (DropdownButton from `/config/providers`), `temperature` (Slider 0.0–1.0), `mode` (SegmentedButton: agent/subagent)
- Permissions toggles: list of `edit`, `bash`, `webfetch` with allow/ask/deny selection
- Bottom: full-height multiline `TextField` for system prompt body
- Save: stores to Hive locally. Shows `"ℹ Agent drafts are stored on this device."` info chip.
- Load: merges local drafts with server agents from `GET /agent`

---

#### Phase 7 — SSE Integration `[subagent: sse_integration]`

**Modifies:**
- `chat_screen.dart` — watch `messageStreamProvider(activeSession.id)`
- `message_bubble.dart` — handle `message.part.updated` for live text appending
- `permission_card.dart` — inserted into message list on `permission.asked`
- `tool_pill.dart` — update state on `tool.execute.before/after`
- `session_list_screen.dart` — live updates on `session.created/deleted/updated`

**Key behaviors:**
- `message.part.updated` → append text to current assistant bubble
- `session.idle` → finalize message, stop cursor, enable input
- `permission.asked` → insert PermissionCard at end of message list
- `tool.execute.before` → show tool pill with spinner
- `tool.execute.after` → update pill with completed state + duration
- Auto-scroll: always scroll to bottom on new content unless user has scrolled up >100px

---

#### 🔍 Review Gate 3 `[subagent: review_3]`

```
Read: Phase 6 + 7 files.
Check:
[ ] SSE reconnects after connection drop
[ ] Streaming text appears incrementally (not in one block)
[ ] permission.asked → PermissionCard → Allow → POST correct endpoint
[ ] session.idle re-enables chat input
[ ] Session list updates live without full rebuild
[ ] Agent dropdown populates from /agent endpoint
[ ] Token badge updates on session.idle
[ ] No memory leaks from unclosed streams
[ ] Tool pills show correct before/after states
```

---

#### Phase 8 — Local Server Platform `[subagent: platform]`

**Creates/modifies:**
- `lib/core/local_server/server_manager.dart` — full implementation
- `lib/features/local_server/local_server_screen.dart` — download + start/stop UI
- `lib/features/local_server/api_key_setup_sheet.dart` — first-run API key setup
- `android/localserver/` — DFM module structure with build.gradle + AndroidManifest
- `android/.../MainActivity.kt` — implement `installLocalServerFeature` via SplitInstallManager

**DFM flow:**
1. Check if feature installed → if not, show [Download Binary] button (~60MB)
2. Show download progress (SplitInstallSessionState listener)
3. On success, show [Start ▶] button
4. On tap, check API keys → if missing, open `ApiKeySetupSheet`
5. Start process, pipe output to xterm in terminal screen

---

#### Phase 9 — Terminal `[subagent: terminal]`

**Creates:**
- `lib/features/terminal/terminal_screen.dart`

**Three modes (toggle in screen header):**
- **Server Logs**: receive output from `LocalServerManager.logStream` and feed to `xterm.Terminal`
- **Local Shell**: `Process.start('/system/bin/sh', [])` with bidirectional xterm I/O
- **Tool History**: read-only replay of tool outputs from current session's message history

**xterm integration pattern:**
```dart
final terminal = Terminal();
final terminalController = TerminalController();

// Feed input to process
terminal.onOutput = (data) => shellProcess?.stdin.add(utf8.encode(data));

// Feed process output to terminal
shellProcess?.stdout.transform(utf8.decoder).listen(terminal.write);
shellProcess?.stderr.transform(utf8.decoder).listen(terminal.write);

// Widget
TerminalView(terminal, controller: terminalController)
```

---

#### 🔍 Review Gate 4 `[subagent: review_4]`

```
Read: Phase 8 + 9 files.
Check:
[ ] DFM download flow shows progress
[ ] Binary executes from nativeLibraryDir (not filesDir)
[ ] API keys are never logged to console
[ ] Local server card never auto-starts
[ ] xterm connects to local shell correctly
[ ] xterm receives local server stdout/stderr
[ ] Tool history mode shows correct data from message list
[ ] flutter build apk --debug succeeds
```

---

#### Phase 10 — Theme + Polish `[subagent: theme_polish]`

**Scope:**
- Full Material 3 dark/light theme with proper color scheme (teal-on-dark, inspired by OpenCode's TUI palette)
- Glassmorphism effect on server cards (frosted background, subtle border)
- `flutter_animate`:
  - Message bubbles: fade-in + slide-up on first render
  - Token counter: number animation when value changes
  - Streaming cursor: pulse animation
  - Permission card: shake animation on deny
- Haptic feedback: `HapticFeedback.mediumImpact()` on Allow/Deny taps
- Empty states: no servers (onboarding card), no sessions (create session CTA), server unreachable
- Offline banner: persistent top banner when SSE drops, with reconnect countdown
- App icon + splash screen

---

#### Phase 11 — Testing `[subagent: testing]`

**Tests to write:**
1. **Unit — SseParser:** parse real SSE data samples covering all event types in §2.2
2. **Unit — OpenCodeClient:** mock Dio responses for each REST method
3. **Widget — MessageBubble:** renders text, code, streaming cursor
4. **Widget — PermissionCard:** renders, buttons call correct callback
5. **Widget — TokenBadge:** correct color at <50%, 50-80%, >80%
6. **Widget — CodeBlock:** renders code, copy button works
7. **Integration:** connect to real server → create session → send prompt → verify streaming ends with `session.idle` → token badge updates

---

#### 🔍 Final Review `[subagent: final_review]`

```
Full audit — all files:

API correctness:
[ ] Every REST call uses paths from §2.1 exactly
[ ] SSE event type strings match §2.2 exactly
[ ] Auth: Authorization: Basic base64(opencode:<password>)
[ ] SSE stream: /global/event
[ ] Health check: /global/health
[ ] Permission reply: POST /question/:requestID/reply

Architecture:
[ ] dioClientProvider rebuilds on server switch
[ ] No API keys in logs or Hive
[ ] No hardcoded IP addresses or ports
[ ] SSE reconnect has backoff (max 30s)
[ ] Binary executes from nativeLibraryDir only
[ ] Local server never auto-starts

mDNS:
[ ] Service type: _opencode._tcp (no .local in query)
[ ] Multicast lock acquired before scan, released after

UI:
[ ] flutter analyze — 0 errors, 0 warnings
[ ] flutter build apk --release succeeds
[ ] All navigation routes work
[ ] Token badge thresholds: green/amber/red
[ ] Dark mode — no white flashes
[ ] Responsive at 360px and 414px

Terminal:
[ ] Three modes switchable
[ ] Local shell uses /system/bin/sh
[ ] Server logs feed xterm correctly
```

---

## 12. What is NOT in This Plan

| Feature | Status | Why / When |
|---------|--------|-----------|
| File browser + diff viewer | ❌ v2 | No read-only file browse API in v1 |
| `freezed` models | ❌ v2 | Build complexity not worth it until API stabilizes |
| Session sharing / export | ❌ v2 | No API for it |
| Push notifications for permissions | ❌ v2 | Requires Firebase + server-side push |
| Multi-server simultaneous connections | ❌ v2 | Single active server is sufficient |
| Remote shell (SSH) | ❌ out of scope | Use Termux / a dedicated SSH client |
| Web UI wrapper | ❌ v2 | Different deployment target |
| iOS build | ❌ v2 | Local server binary is Android-only; rest could be ported |

---

*Document version: 2.0 — April 2026*
*Decisions incorporated: local-only agent storage (v1), mDNS in Phase 4, full xterm terminal, on-demand binary via DFM*
