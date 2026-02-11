# Game Server Architecture

This document provides a comprehensive technical specification for the Game Server component of the Vibe Coding Olympics platform. It details the server architecture, component breakdown, technology choices, design decisions, and development roadmap.

> **Reference:** See [SPEC.md](../../SPEC.md) for authoritative requirements and game specifications.

## Overview

The Game Server is the central process that orchestrates the entire Vibe Coding Olympics experience. It manages player lobbies, executes game logic, sandboxes player code, and exposes APIs for all client types (MCP servers, web editors, and spectator displays).

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Game Server                                    │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                        API Layer                                   │  │
│  │   HTTP Routes (Fastify)  │  WebSocket Handlers (@fastify/websocket)│  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                              │                                           │
│         ┌────────────────────┼────────────────────┐                     │
│         ▼                    ▼                    ▼                     │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐               │
│  │   Lobby     │     │    Game     │     │   Sandbox   │               │
│  │  Manager    │◄────│   Engine    │────►│   System    │               │
│  │             │     │             │     │             │               │
│  │ • Rooms     │     │ • Game Loop │     │ • isolated-vm│              │
│  │ • Players   │     │ • Tick/Turn │     │ • Compilation│              │
│  │ • Auth      │     │ • Modules   │     │ • Limits    │               │
│  └─────────────┘     └─────────────┘     └─────────────┘               │
│                              │                                           │
│                              ▼                                           │
│                    ┌─────────────────┐                                   │
│                    │  Game Modules   │                                   │
│                    │   (plugins)     │                                   │
│                    └─────────────────┘                                   │
└─────────────────────────────────────────────────────────────────────────┘
```

## Server Components

### 1. Lobby Manager (`src/lobby/`)

The Lobby Manager handles room lifecycle and player sessions.

#### Responsibilities

- **Room Creation & Management**
  - Create rooms with unique room codes
  - Track room state (waiting, playing, finished)
  - Clean up inactive rooms after timeout
  - Support Olympics session configuration (game sequence)

- **Player Registration**
  - Accept player join requests with display names
  - Generate ephemeral session tokens (JWT or similar)
  - Track connected players per room
  - Handle player disconnect/reconnect

- **Host Management**
  - Issue special host tokens with elevated permissions
  - Authorize game start/stop operations
  - Support player kick functionality

#### Key Interfaces

```typescript
interface Room {
  id: string;
  code: string;           // 4-6 character join code
  host: HostInfo;
  players: Map<string, PlayerInfo>;
  status: 'waiting' | 'playing' | 'finished';
  olympicsConfig?: OlympicsConfig;
  currentGameId?: string;
  createdAt: Date;
}

interface PlayerInfo {
  id: string;
  name: string;
  token: string;          // Ephemeral session token
  connectedAt: Date;
  isConnected: boolean;
  submittedCode?: string;
}

interface LobbyManager {
  createRoom(hostCredentials: HostCredentials): Promise<Room>;
  joinRoom(roomCode: string, playerName: string): Promise<PlayerInfo>;
  leaveRoom(roomId: string, playerId: string): Promise<void>;
  getRoom(roomId: string): Room | undefined;
  kickPlayer(roomId: string, hostToken: string, playerId: string): Promise<void>;
}
```

#### Security Considerations

- Room codes should be short but hard to guess (e.g., alphanumeric, excluding ambiguous characters like 0/O, 1/l)
- Tokens should expire after session ends
- Host authentication required for web-based room creation
- CLI mode can bypass host auth for local development

---

### 2. Game Engine (`src/engine/`)

The Game Engine manages the game loop and orchestrates game module execution.

#### Responsibilities

- **Game Lifecycle**
  - Initialize games with loaded game modules
  - Create initial game state from player roster
  - Detect game-over conditions
  - Calculate and report results

- **Tick/Turn Scheduling**
  - Fixed-interval timer for tick-based games (with drift compensation)
  - Turn sequencing for turn-based games
  - Player action collection within time windows

- **Module Orchestration**
  - Load and validate game modules
  - Invoke module methods at appropriate lifecycle points
  - Pass player actions through visibility filters

- **State Broadcast**
  - Generate spectator state for display clients
  - Generate per-player visible state for bot APIs
  - Emit game events for UI feeds

#### Key Interfaces

```typescript
interface GameEngine {
  startGame(room: Room, gameModule: GameModule): Promise<GameSession>;
  stopGame(sessionId: string): Promise<GameResults>;
  submitAction(sessionId: string, playerId: string, action: unknown): void;
  getVisibleState(sessionId: string, playerId: string): unknown;
  getSpectatorState(sessionId: string): unknown;

  // Event emitters for replay-friendly architecture
  on(event: 'tick', handler: (state: GameState) => void): void;
  on(event: 'action', handler: (playerId: string, action: unknown) => void): void;
  on(event: 'gameOver', handler: (results: GameResults) => void): void;
}

interface GameSession {
  id: string;
  roomId: string;
  module: GameModule;
  state: unknown;
  tickNumber: number;
  startedAt: Date;
  players: Map<string, PlayerSandbox>;
}

interface TickScheduler {
  start(intervalMs: number, callback: () => Promise<void>): void;
  stop(): void;
  // Drift compensation: if tick 5 ran 10ms late, tick 6 runs 10ms early
  compensateDrift: boolean;
}
```

#### Tick Loop Implementation

```typescript
// Pseudocode for the main game loop (tick-based games)
async function tickLoop(session: GameSession) {
  const actions = new Map<string, unknown>();

  for (const [playerId, sandbox] of session.players) {
    const visibleState = session.module.getVisibleState(session.state, playerId);
    try {
      const action = await sandbox.execute('play', [visibleState], {
        timeout: 50,  // 50ms CPU limit
      });
      actions.set(playerId, action);
    } catch (error) {
      // Timeout or error: use default action
      actions.set(playerId, session.module.getDefaultAction(session.state, playerId));
      emitPlayerError(playerId, error);
    }
  }

  // Advance game state atomically
  session.state = session.module.tick(session.state, actions);
  session.tickNumber++;

  // Broadcast to all connected clients
  broadcastSpectatorState(session.module.getSpectatorState(session.state));

  if (session.module.isGameOver(session.state)) {
    endGame(session);
  }
}
```

---

### 3. Sandbox System (`src/sandbox/`)

The Sandbox System provides secure execution of untrusted player code.

#### Responsibilities

- **Code Compilation**
  - Parse and validate JavaScript syntax
  - Wrap player code in sandbox-safe execution context
  - Detect prohibited constructs (eval, Function, etc.)

- **Isolated Execution**
  - Create V8 isolates per player via `isolated-vm`
  - Enforce CPU time limits (50ms per invocation)
  - Enforce memory limits (8MB per isolate)
  - Expose only safe built-ins (Math, JSON, Array, etc.)

- **Hot-Loading**
  - Replace player code between ticks
  - Preserve global state option (or reset on new code)
  - Report compilation errors to players

#### Key Interfaces

```typescript
interface SandboxManager {
  createSandbox(playerId: string): Promise<PlayerSandbox>;
  loadCode(playerId: string, code: string): Promise<CompileResult>;
  destroySandbox(playerId: string): void;
}

interface PlayerSandbox {
  playerId: string;
  execute(fnName: string, args: unknown[], options: ExecuteOptions): Promise<unknown>;
  reset(): void;  // Clear global state
  dispose(): void;
}

interface CompileResult {
  success: boolean;
  error?: {
    message: string;
    line?: number;
    column?: number;
  };
}

interface ExecuteOptions {
  timeout: number;       // CPU time in milliseconds
  memoryLimit?: number;  // Memory in bytes (default: 8MB)
}
```

#### Sandbox Configuration

```typescript
const SANDBOX_CONFIG = {
  // CPU limits
  cpuTimePerInvocation: 50,       // 50ms max per play() call

  // Memory limits
  memoryLimit: 8 * 1024 * 1024,   // 8MB per isolate

  // Allowed globals (whitelist approach)
  allowedGlobals: [
    'Math', 'JSON', 'Array', 'Object', 'Map', 'Set',
    'String', 'Number', 'Boolean', 'Date', 'RegExp',
    'parseInt', 'parseFloat', 'isNaN', 'isFinite',
    'encodeURIComponent', 'decodeURIComponent',
    'console',  // For debugging (logs captured, not output)
  ],

  // Explicitly blocked (defense in depth)
  blockedPatterns: [
    /\beval\b/,
    /\bFunction\b/,
    /\bimport\b/,
    /\brequire\b/,
    /\bprocess\b/,
    /\bglobalThis\b/,
  ],
};
```

#### Hot-Load Behavior

1. Player submits new code via API
2. Sandbox Manager attempts to compile in a temporary isolate
3. If compilation fails: return error, keep old code running
4. If compilation succeeds: dispose old isolate, install new code
5. New code takes effect on next tick (no mid-tick swaps)
6. Global state from old code is discarded

---

### 4. API Layer (`src/api/`)

The API Layer handles all client communication via HTTP and WebSocket.

#### Responsibilities

- **HTTP Routes**
  - RESTful endpoints for lobby operations
  - Game control endpoints (host-only)
  - Player action endpoints (get state, submit code)
  - Authentication via Bearer tokens

- **WebSocket Handlers**
  - Real-time state broadcasting
  - Game event push notifications
  - Lobby update notifications
  - Code acceptance/rejection notifications

- **Request Validation**
  - Schema validation via Fastify + Zod
  - Token verification
  - Rate limiting (configurable)

#### HTTP Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/rooms` | Host | Create a room |
| POST | `/api/rooms/:roomId/join` | None | Join a room |
| GET | `/api/rooms/:roomId` | Token | Get room state |
| POST | `/api/rooms/:roomId/games/start` | Host | Start a game |
| POST | `/api/rooms/:roomId/games/stop` | Host | Stop current game |
| POST | `/api/rooms/:roomId/advance` | Host | Advance to next game |
| GET | `/api/rooms/:roomId/game/rules` | Token | Get game rules |
| GET | `/api/rooms/:roomId/game/api-docs` | Token | Get bot API docs |
| GET | `/api/rooms/:roomId/game/state` | Token | Get visible state |
| POST | `/api/rooms/:roomId/game/submit` | Token | Submit player code |
| GET | `/api/rooms/:roomId/game/log` | Token | Get player's event log |
| GET | `/api/rooms/:roomId/standings` | Token | Get Olympics standings |

#### WebSocket Protocol

```typescript
// Connection: ws://<host>/ws?roomId=<roomId>&token=<token>

// Server -> Client messages
type ServerMessage =
  | { type: 'game:state'; state: SpectatorState }
  | { type: 'game:event'; event: GameEvent }
  | { type: 'game:started'; gameType: string }
  | { type: 'game:ended'; results: PlayerResult[] }
  | { type: 'lobby:updated'; room: RoomState }
  | { type: 'code:accepted'; playerId: string }
  | { type: 'code:rejected'; playerId: string; error: string };

// Client -> Server messages (minimal, most actions via HTTP)
type ClientMessage =
  | { type: 'ping' }
  | { type: 'subscribe'; channels: string[] };
```

#### Request Flow Example

```
Player submits code:
1. POST /api/rooms/:roomId/game/submit { code: "..." }
2. API validates token, extracts playerId
3. SandboxManager.loadCode(playerId, code)
4. If success:
   - HTTP response: { success: true }
   - WebSocket broadcast: { type: 'code:accepted', playerId }
5. If failure:
   - HTTP response: { success: false, error: "..." }
   - WebSocket broadcast: { type: 'code:rejected', playerId, error }
```

---

## Technology Stack

### Library Selection

| Concern | Library | Version | Rationale |
|---------|---------|---------|-----------|
| HTTP Framework | Fastify | ^5.x | High performance, native schema validation, excellent plugin ecosystem |
| WebSocket | @fastify/websocket | ^11.x | First-party Fastify integration, uses `ws` under the hood |
| Sandboxing | isolated-vm | ^5.x | In-process V8 isolates, strict resource limits, fast startup |
| Validation | Zod | ^3.x | Type-safe runtime validation, excellent TypeScript integration |
| JWT Tokens | @fastify/jwt | ^9.x | Fastify-integrated JWT signing/verification |
| Rate Limiting | @fastify/rate-limit | ^10.x | Per-route rate limiting, configurable backends |
| Logging | pino | ^9.x | Fast JSON logging, Fastify default |
| Testing | Vitest | ^3.x | Fast, TypeScript-native, compatible with Jest API |

### Alternatives Considered

#### Sandboxing: isolated-vm vs Alternatives

| Option | Pros | Cons | Decision |
|--------|------|------|----------|
| **isolated-vm** | Fast startup, in-process, strict limits, mature | Requires native compilation | **Selected** |
| Deno isolates | Good security, modern API | Requires Deno runtime, slower startup | Rejected |
| WebWorkers | Browser-compatible | Less strict isolation, harder to limit resources | Rejected |
| vm2 | Pure JS, easy setup | Security vulnerabilities, deprecated | Rejected |
| QuickJS | Lightweight, portable | Less mature ecosystem, slower execution | Rejected |

**Decision rationale:** `isolated-vm` provides the best balance of security, performance, and resource control for our use case. The ability to set precise CPU and memory limits per isolate is critical for fair gameplay.

#### HTTP Framework: Fastify vs Alternatives

| Option | Pros | Cons | Decision |
|--------|------|------|----------|
| **Fastify** | Fast, schema validation, WebSocket plugin | Slightly more complex than Express | **Selected** |
| Express | Ubiquitous, simple | Slower, requires separate WebSocket library | Rejected |
| Koa | Modern, lightweight | Smaller ecosystem, no built-in validation | Rejected |
| Hono | Very fast, edge-compatible | Newer, smaller community | Rejected |

**Decision rationale:** Fastify's performance benchmarks, native schema validation, and first-party WebSocket support make it ideal for a real-time game server.

#### Real-time: WebSockets vs Alternatives

| Option | Pros | Cons | Decision |
|--------|------|------|----------|
| **WebSockets (ws)** | Low latency, simple, wide support | Requires reconnection handling | **Selected** |
| Socket.io | Auto-reconnection, rooms built-in | More overhead, larger bundle | Rejected |
| Server-Sent Events | Simple, HTTP-based | Unidirectional only | Rejected |
| WebRTC | Lowest latency | Complex, overkill for our use case | Rejected |

**Decision rationale:** Raw WebSockets provide the lowest latency for tick-based game state broadcast. The slight additional complexity of handling reconnection is worth the performance benefit.

---

## Design Decisions

### 1. Stateless Game Engine with Observable Events

**Decision:** The game engine emits events for all state transitions rather than persisting state internally.

**Rationale:**
- Enables future replay system without refactoring
- Simplifies testing (can verify state transitions)
- Allows multiple consumers (WebSocket broadcast, logging, analytics)

**Trade-off:** Slightly more complex implementation than direct state mutation.

### 2. Per-Player Isolates with Clean Reset

**Decision:** Each player gets their own V8 isolate. When new code is submitted, the isolate is destroyed and recreated.

**Alternatives considered:**
- Shared isolate with code namespacing (rejected: harder to enforce limits per player)
- Preserve global state across code updates (rejected: causes confusing behavior)

**Rationale:** Clean isolation prevents one player's runaway code from affecting others. Clean resets on code update eliminate confusion about stale state.

### 3. Fixed-Interval Ticks with Drift Compensation

**Decision:** Use a fixed-interval timer for tick-based games, with compensation for accumulated drift.

**Implementation:**
```typescript
class DriftCompensatedTimer {
  private expectedTime: number;

  tick() {
    const now = Date.now();
    const drift = now - this.expectedTime;

    // Execute tick logic...

    // Schedule next tick, compensating for drift
    const nextDelay = Math.max(0, this.interval - drift);
    this.expectedTime += this.interval;
    setTimeout(() => this.tick(), nextDelay);
  }
}
```

**Rationale:** Prevents tick timing from drifting over long games. A 300ms tick game running for 10 minutes could drift by several seconds without compensation.

### 4. Ephemeral Sessions, No Persistent State

**Decision:** The server stores nothing after a session ends. No user accounts (except for web room creation), no leaderboards, no replay storage.

**Rationale:** Keeps the MVP simple. All persistence features can be added later without changing core architecture.

**Trade-off:** Users cannot review past games or maintain rankings across sessions.

### 5. Game Modules as Self-Contained Packages

**Decision:** Each game owns its state, logic, visibility rules, rendering, and sound. The engine is generic.

**Rationale:**
- Games can be developed independently
- No engine changes needed for new games
- Each game can optimize its own rendering

**Trade-off:** More code per game, potential for inconsistent patterns across games.

### 6. Single-Threaded with Async I/O

**Decision:** The server runs on a single Node.js thread, using async I/O for HTTP/WebSocket and isolated-vm for sandboxing.

**Rationale:**
- Simpler mental model
- `isolated-vm` handles CPU isolation within the process
- Node.js event loop handles I/O concurrency naturally

**Scaling path:** For high player counts, run multiple server instances behind a load balancer, with sticky sessions or shared state via Redis.

---

## Development Roadmap

### Phase 1: Core Infrastructure

**Duration:** 1-2 weeks
**Goal:** Basic Fastify server with WebSocket support and project scaffolding

**Deliverables:**
- [ ] Fastify server with health check endpoint
- [ ] WebSocket connection handling with ping/pong
- [ ] Request validation schemas with Zod
- [ ] Pino logging configuration
- [ ] TypeScript project structure (`packages/server/`)
- [ ] Basic test infrastructure with Vitest

**Acceptance Criteria:**
- Server starts and accepts HTTP/WebSocket connections
- Logs all requests in structured JSON format
- Tests pass with >80% coverage on new code

---

### Phase 2: Lobby System

**Duration:** 1-2 weeks
**Goal:** Room creation, player registration, token-based auth

**Deliverables:**
- [ ] Room creation endpoint with unique room codes
- [ ] Player join endpoint with ephemeral tokens
- [ ] Host authentication (JWT signing/verification)
- [ ] Room state management (in-memory)
- [ ] Lobby WebSocket broadcasts
- [ ] Basic rate limiting

**Acceptance Criteria:**
- Host can create room and receive code
- Players can join with room code
- Host can see all connected players
- WebSocket broadcasts player join/leave

---

### Phase 3: Sandbox System

**Duration:** 2-3 weeks
**Goal:** Secure player code execution with isolated-vm

**Deliverables:**
- [ ] `isolated-vm` wrapper with resource limits
- [ ] Code compilation with syntax validation
- [ ] Safe global environment (whitelist approach)
- [ ] Execution timeout handling
- [ ] Memory limit enforcement
- [ ] Console capture for debugging
- [ ] Hot-load support (replace code between ticks)

**Acceptance Criteria:**
- Player code runs in isolated environment
- Exceeding CPU limit returns default action
- Exceeding memory limit terminates isolate gracefully
- Code with syntax errors is rejected with clear message
- New code replaces old code on next tick

---

### Phase 4: Game Engine

**Duration:** 2-3 weeks
**Goal:** Game loop with tick scheduling and module support

**Deliverables:**
- [ ] Game module loading interface
- [ ] Tick scheduler with drift compensation
- [ ] Turn-based game support
- [ ] Player action collection and validation
- [ ] State transition with module.tick()/executeTurn()
- [ ] Spectator state generation
- [ ] Per-player visible state generation
- [ ] Game-over detection and results

**Acceptance Criteria:**
- Can load and initialize a game module
- Tick-based games run at configured interval
- Turn-based games sequence player turns correctly
- Spectator state broadcasts every tick
- Game ends when module reports game-over

---

### Phase 5: Integration

**Duration:** 1-2 weeks
**Goal:** Full integration with hot-loading and state broadcast

**Deliverables:**
- [ ] Code submission API integrated with sandbox
- [ ] Hot-loading: new code takes effect next tick
- [ ] WebSocket broadcast for game state
- [ ] WebSocket broadcast for code accept/reject
- [ ] Event logging for player actions
- [ ] Olympics scoring system
- [ ] End-to-end integration tests

**Acceptance Criteria:**
- Player can submit code via API
- Code is validated and hot-loaded
- Spectator display receives state updates
- Player can iterate on code while game runs
- Olympics standings track scores across games

---

## Appendix: File Structure

```
packages/server/
├── src/
│   ├── index.ts              # Entry point, Fastify app setup
│   ├── config.ts             # Environment configuration
│   │
│   ├── lobby/
│   │   ├── index.ts          # Lobby module exports
│   │   ├── manager.ts        # LobbyManager implementation
│   │   ├── room.ts           # Room class
│   │   ├── auth.ts           # Token generation/verification
│   │   └── types.ts          # Lobby-related types
│   │
│   ├── engine/
│   │   ├── index.ts          # Engine module exports
│   │   ├── engine.ts         # GameEngine implementation
│   │   ├── scheduler.ts      # TickScheduler with drift compensation
│   │   ├── session.ts        # GameSession management
│   │   └── types.ts          # Engine-related types
│   │
│   ├── sandbox/
│   │   ├── index.ts          # Sandbox module exports
│   │   ├── manager.ts        # SandboxManager implementation
│   │   ├── isolate.ts        # PlayerSandbox wrapper
│   │   ├── compiler.ts       # Code compilation and validation
│   │   └── types.ts          # Sandbox-related types
│   │
│   └── api/
│       ├── index.ts          # API module exports
│       ├── routes/
│       │   ├── lobby.ts      # Lobby HTTP routes
│       │   ├── game.ts       # Game HTTP routes
│       │   └── player.ts     # Player action routes
│       ├── websocket/
│       │   ├── handler.ts    # WebSocket connection handler
│       │   └── broadcaster.ts # State broadcast utilities
│       └── schemas/          # Zod validation schemas
│
├── test/
│   ├── lobby/
│   ├── engine/
│   ├── sandbox/
│   └── api/
│
├── package.json
└── tsconfig.json
```

---

## Appendix: Error Handling Strategy

### Error Categories

| Category | Example | Response |
|----------|---------|----------|
| Validation | Invalid room code format | 400 Bad Request |
| Authentication | Invalid/expired token | 401 Unauthorized |
| Authorization | Non-host tries to start game | 403 Forbidden |
| Not Found | Room doesn't exist | 404 Not Found |
| Player Code | Syntax error in submitted code | 200 OK with `{ success: false, error }` |
| Sandbox Timeout | play() exceeded 50ms | Default action used, player notified |
| Internal | Unexpected server error | 500 Internal Server Error, logged |

### Graceful Degradation

- **Player code throws:** Use default action, log error, notify player
- **Player code times out:** Use default action, log warning, notify player
- **Sandbox crashes:** Recreate isolate, use default action for current tick
- **WebSocket disconnect:** Game continues, player reconnects when possible
- **Game module error:** Stop game, report error to host

---

## References

- [SPEC.md](../../SPEC.md) - Authoritative project specification
- [isolated-vm Documentation](https://github.com/laverdet/isolated-vm)
- [Fastify Documentation](https://www.fastify.io/)
- [@fastify/websocket](https://github.com/fastify/fastify-websocket)
