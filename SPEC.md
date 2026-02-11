# Vibe Coding Olympics

## Overview

Vibe Coding Olympics is a multiplayer party game where players use AI coding agents to write bots that compete in a series of live games. Players sit together in a room with a shared big screen showing the action while each player works on their own machine, prompting their AI agent to write and iterate on game-playing code in real time.

The system hot-loads player code into the running game, creating a tight feedback loop: prompt your agent, watch your bot play on the big screen, iterate.

## Core Experience

A typical session looks like this:

1. The host starts the server and opens the spectator view on the big screen.
2. Players join a lobby by entering a room code. IDE users launch the MCP server CLI; others open the web editor in a browser.
3. The host selects a game (e.g. Poker) and starts it.
4. Each player's AI agent fetches the game rules and bot API docs via MCP tools (or reads them in the web editor sidebar), then writes a `play()` function.
5. Players submit code. It's sandboxed and hot-loaded into the running game within seconds.
6. The big screen shows bots playing in real time. Players watch, then prompt their agents to improve strategy.
7. When the game ends, scores are tallied. The host advances to the next game in the olympics.
8. After all games, an overall champion is crowned.

## Architecture

### Components

**Game Server** — The central process. Manages lobbies, runs game logic, executes player code in sandboxes, exposes an HTTP + WebSocket API, and serves the frontend.

**Frontend** — A single-page React app with multiple views:
- Spectator display (the big screen) — animated game rendering, scores, event feed
- Lobby — players join, host picks games, everyone sees who's connected
- Web editor — Monaco-based code editor with a submit button, game state panel, and API docs reference
- Admin controls — start/stop games, kick players, advance the olympics

**MCP Server** — A small CLI that each IDE-connected player runs locally. Translates MCP tool calls into game server HTTP API calls. Connects to the player's AI agent via stdio transport.

**Game Modules** — Pluggable game implementations. Each module defines its own state, rules, bot API, tick/turn logic, and rendering configuration.

### System Diagram

```
 PLAYER MACHINES                          HOST MACHINE
 ──────────────                          ────────────

 ┌──────────────────┐              ┌──────────────────────────────────┐
 │ IDE + AI Agent   │   stdio      │           Game Server            │
 │ (Claude Code,    │◄────────►MCP │                                  │
 │  Cursor, etc.)   │  Server CLI  │  ┌───────┐ ┌─────────────────┐  │
 └──────────────────┘     │        │  │ Lobby │ │  Game Engine    │  │
                          │ HTTP   │  │Manager│ │                 │  │
                          ├───────►│  └───────┘ │ ┌─Sandboxed──┐ │  │
                          │        │            │ │ Player 1   │ │  │
 ┌──────────────────┐     │        │            │ │ Player 2   │ │  │
 │ Browser          │ WebSocket    │            │ │ Player 3   │ │  │
 │ ┌──────────────┐ │◄────┘   ┌──►│            │ └────────────┘ │  │
 │ │ Web Editor + │ │         │   │  └─────────────────┘  │      │  │
 │ │ Monaco       │ │         │   │                       │state │  │
 │ └──────────────┘ │         │   │                       ▼      │  │
 └──────────────────┘         │   │  ┌─────────────────────────┐ │  │
                              │   │  │ Game Module (plugin)    │ │  │
 ┌──────────────────┐         │   │  │ · state shape           │ │  │
 │ Big Screen       │         │   │  │ · rules & bot API docs  │ │  │
 │ ┌──────────────┐ │         │   │  │ · tick/turn logic       │ │  │
 │ │ Spectator    │◄├─────────┘   │  │ · rendering config      │ │  │
 │ │ Display      │ │  WebSocket  │  └─────────────────────────┘ │  │
 │ └──────────────┘ │             │                               │  │
 └──────────────────┘             └───────────────────────────────┘  │
```

### Key Flows

**Player joins via IDE:**
1. Player runs the MCP server CLI, providing a server URL and room code.
2. MCP server registers with the game server via HTTP, receives a player token.
3. The AI agent can now call MCP tools: `get_rules`, `get_game_state`, `get_api_docs`, `submit_code`.

**Player joins via web:**
1. Player opens the app in a browser and enters a room code.
2. The web editor view loads with Monaco, a live game state panel, and the bot API reference.
3. Player writes code (with or without an AI assistant) and clicks submit.

**Game loop:**
1. Host starts a game. Server initializes the game module and creates initial state.
2. The game loop ticks at a fixed interval (configurable per game, e.g. 500ms for Bomberman, per-turn for Poker).
3. Each tick: call each player's sandboxed `play()` function with their visible state, collect returned actions, advance game state, broadcast new state to all WebSocket clients.
4. Spectator display renders each new frame.
5. When the game module reports game-over, results are displayed and scores are recorded.

**Code hot-load:**
1. Player submits code via MCP tool or web editor.
2. Server compiles it in a new V8 isolate to check for syntax errors.
3. On success, the new code replaces the player's previous code and takes effect on the next tick.
4. On failure, the player receives the error message; their previous code continues running.

## Technology Stack

| Concern | Technology | Rationale |
|---------|-----------|-----------|
| Language | TypeScript (everywhere) | Shared types across server, client, MCP. Player-submitted code is JS — no FFI boundary. |
| Server framework | Fastify | Fast, good WebSocket support via `@fastify/websocket`, built-in schema validation. |
| Real-time | WebSockets (`ws`) | Low-latency state push for spectator view and tick-based games. |
| Sandboxing | `isolated-vm` | V8 isolates in-process. Fast startup, strict memory and time limits, no filesystem or network access. |
| Frontend | React + Vite | Standard tooling, fast dev server, good ecosystem. |
| Game rendering | Pixi.js on HTML5 Canvas | 2D sprite-based rendering covers all planned game types. |
| Web code editor | Monaco Editor | VS Code's editor engine. Syntax highlighting, autocomplete, familiar UX. |
| MCP SDK | `@modelcontextprotocol/sdk` | Official MCP TypeScript SDK, stdio transport for IDE integration. |
| Monorepo | npm workspaces | Simple, no extra tooling needed. |

## Project Structure

```
olympics/
├── packages/
│   ├── server/             # Fastify game server
│   │   ├── src/
│   │   │   ├── lobby/          # Room creation, player registration, auth tokens
│   │   │   ├── engine/         # Game loop, tick scheduling, sandbox orchestration
│   │   │   ├── sandbox/        # isolated-vm wrapper, code compilation, resource limits
│   │   │   └── api/            # HTTP routes + WebSocket handlers
│   │   └── package.json
│   ├── client/             # React frontend (all views)
│   │   ├── src/
│   │   │   ├── spectator/      # Big screen game rendering (Pixi.js)
│   │   │   ├── editor/         # Monaco web editor + game state panel
│   │   │   ├── lobby/          # Join/create rooms, player list
│   │   │   └── admin/          # Host controls (game selection, start/stop)
│   │   └── package.json
│   ├── mcp/                # MCP server CLI
│   │   ├── src/
│   │   │   └── index.ts        # Tool definitions, HTTP client to game server
│   │   └── package.json
│   ├── games/              # Game module plugins
│   │   ├── poker/
│   │   ├── rob-the-nest/
│   │   ├── bomberman/
│   │   └── rts/
│   └── shared/             # Shared types, constants, bot API type definitions
│       └── package.json
├── package.json            # Workspace root
└── tsconfig.base.json
```

## Game Module Interface

Every game implements this interface. The game engine is generic; all game-specific behavior lives in the module.

```typescript
interface GameModule<TState, TAction, TVisibleState, TRenderState> {
  // --- Identity ---
  name: string;
  description: string;
  minPlayers: number;
  maxPlayers: number;

  // --- Execution model ---
  // "turn-based" = one player acts at a time (poker)
  // "tick-based" = all players act simultaneously each tick (bomberman, RTS)
  mode: "turn-based" | "tick-based";

  // For tick-based games: milliseconds between ticks
  tickInterval?: number;

  // --- Lifecycle ---
  createInitialState(players: PlayerInfo[]): TState;
  isGameOver(state: TState): boolean;
  getResults(state: TState): PlayerResult[];

  // --- Execution ---
  // For turn-based games:
  executeTurn?(state: TState, playerId: string, action: TAction): TState;
  getActivePlayer?(state: TState): string;

  // For tick-based games:
  tick?(state: TState, actions: Map<string, TAction>): TState;

  // --- Visibility ---
  // What a specific player's bot sees (hides other players' hands, fog of war, etc.)
  getVisibleState(state: TState, playerId: string): TVisibleState;

  // What the spectator display sees (everything needed to render the full game)
  getSpectatorState(state: TState): TRenderState;

  // --- Documentation ---
  // Human-readable rules for AI agents to understand the game
  getRules(): string;

  // TypeScript type definitions for the bot API (state shape, valid actions)
  getBotAPI(): string;

  // --- Defaults ---
  // Action returned when player code throws, times out, or hasn't been submitted yet
  getDefaultAction(state: TState, playerId: string): TAction;
}
```

## Sandbox Specification

### Player Code Contract

Players submit a single JavaScript function. The function receives the game's visible state and must return an action.

```javascript
// Player code template — same shape for every game
function play(state) {
  // 'state' is the game-specific visible state for this player
  // Return a game-specific action object
  return { action: "fold" };       // poker example
  return { action: "move", direction: "up" };  // bomberman example
}
```

### Sandbox Constraints

| Resource | Limit | Behavior on violation |
|----------|-------|----------------------|
| CPU time per invocation | 50ms | Execution terminated, default action used |
| Memory | 8 MB | Isolate terminated, default action used |
| Network access | None | Not available in isolate |
| Filesystem access | None | Not available in isolate |
| Global state | Preserved between ticks for same player | Allows bots to maintain strategy state |
| Available APIs | `Math`, `JSON`, `Array`, `Object`, `Map`, `Set`, basic JS built-ins | No `fetch`, `require`, `import`, `eval`, `Function`, `setTimeout` |

### Hot-Load Behavior

- When new code is submitted, it replaces the player's previous code starting from the next tick/turn.
- Global state accumulated by the previous code is discarded — the new code starts with a clean isolate.
- If the new code has a syntax error, the submission is rejected and the old code continues running.
- Players can submit code at any time, even mid-game. There is no limit on submission frequency (but the server may rate-limit to prevent abuse).

## MCP Server Tools

The MCP server exposes these tools to AI agents:

### `get_rules`
Returns the human-readable rules for the current game. Includes game mechanics, win conditions, and strategy hints.

**Parameters:** None
**Returns:** `{ rules: string }`

### `get_api_docs`
Returns the TypeScript type definitions for the bot API — the shape of the state object passed to `play()` and the valid action types.

**Parameters:** None
**Returns:** `{ api: string }`

### `get_game_state`
Returns the current visible game state for this player. This is the same state object that will be passed to the `play()` function.

**Parameters:** None
**Returns:** `{ state: object, gameStatus: "waiting" | "running" | "finished" }`

### `submit_code`
Submits JavaScript code to be hot-loaded into the game. The code must define a `play(state)` function.

**Parameters:** `{ code: string }`
**Returns:** `{ success: boolean, error?: string }`

### `get_game_log`
Returns recent game events relevant to this player (e.g. "You were eliminated", "You won the pot", "Your bot timed out on tick 42").

**Parameters:** `{ limit?: number }`
**Returns:** `{ events: GameEvent[] }`

### `get_standings`
Returns current scores and rankings across the olympics.

**Parameters:** None
**Returns:** `{ standings: PlayerStanding[] }`

## Game Server API

The game server exposes these HTTP and WebSocket endpoints. The MCP server, web editor, and spectator display all communicate through this API.

### HTTP Endpoints

#### Lobby
- `POST /api/rooms` — Create a room. Returns `{ roomId, hostToken }`.
- `POST /api/rooms/:roomId/join` — Join a room. Body: `{ playerName }`. Returns `{ playerId, playerToken }`.
- `GET /api/rooms/:roomId` — Get room state (players, current game, status).

#### Game Control (host only)
- `POST /api/rooms/:roomId/games/start` — Start a game. Body: `{ gameType }`.
- `POST /api/rooms/:roomId/games/stop` — Stop the current game.
- `POST /api/rooms/:roomId/advance` — Advance to the next game in the olympics schedule.

#### Player Actions
- `GET /api/rooms/:roomId/game/rules` — Get current game rules.
- `GET /api/rooms/:roomId/game/api-docs` — Get current game bot API docs.
- `GET /api/rooms/:roomId/game/state` — Get current visible game state for the authenticated player.
- `POST /api/rooms/:roomId/game/submit` — Submit player code. Body: `{ code: string }`.
- `GET /api/rooms/:roomId/game/log` — Get recent game events for the authenticated player.
- `GET /api/rooms/:roomId/standings` — Get olympics standings.

All player/host endpoints require a `Authorization: Bearer <token>` header.

### WebSocket

Connect to `ws://<host>/ws?roomId=<roomId>&token=<token>`.

**Server-sent messages:**
- `{ type: "game:state", state: <spectator state> }` — Broadcast every tick/turn. Contains the full spectator-visible state for rendering.
- `{ type: "game:event", event: <game event> }` — Notable events (eliminations, big plays, errors).
- `{ type: "game:started", gameType: string }` — A game has begun.
- `{ type: "game:ended", results: PlayerResult[] }` — A game has finished.
- `{ type: "lobby:updated", room: <room state> }` — Player joined/left, settings changed.
- `{ type: "code:accepted", playerId: string }` — Player's code was hot-loaded successfully.
- `{ type: "code:rejected", playerId: string, error: string }` — Player's code had an error.

## Olympics Scoring

An olympics session consists of multiple games played in sequence. Points are awarded based on placement in each game:

| Placement | Points |
|-----------|--------|
| 1st | 10 |
| 2nd | 7 |
| 3rd | 5 |
| 4th | 3 |
| 5th+ | 1 |

The host configures which games are in the olympics and in what order before starting the session. The overall champion is the player with the most total points after all games are played.

## Game Specifications

### Poker (Texas Hold'em)

**Mode:** Turn-based
**Players:** 2-8

Players' bots play a series of hands of no-limit Texas Hold'em. Each hand follows standard dealing and betting rounds (preflop, flop, turn, river). The game ends when only one player has chips remaining or after a configured number of hands.

**Visible state:**
```typescript
{
  hand: Card[];              // Player's hole cards
  communityCards: Card[];    // Shared cards on the table
  pot: number;
  currentBet: number;
  myChips: number;
  myCurrentBet: number;
  players: {
    id: string;
    name: string;
    chips: number;
    currentBet: number;
    folded: boolean;
    active: boolean;
  }[];
  round: "preflop" | "flop" | "turn" | "river";
  dealerIndex: number;
  isMyTurn: boolean;
}
```

**Actions:**
```typescript
{ action: "fold" }
{ action: "call" }
{ action: "raise", amount: number }
{ action: "check" }
{ action: "all_in" }
```

**Default action:** `{ action: "fold" }`

---

### Rob the Nest

**Mode:** Tick-based (500ms ticks)
**Players:** 2-6

Each player controls a team of mobs on a shared 2D grid. Each player has a "nest" containing a score. Mobs can pick up points from the center of the map or steal from other players' nests and carry them back to their own. Mobs that collide with enemy mobs near their own nest "tag" the enemy, sending them back to their spawn and dropping what they're carrying. The game runs for a fixed number of ticks.

**Visible state:**
```typescript
{
  tick: number;
  maxTicks: number;
  gridSize: { width: number; height: number };
  myNest: { x: number; y: number; score: number };
  myMobs: {
    id: string;
    x: number;
    y: number;
    carrying: number;
  }[];
  enemyNests: {
    playerId: string;
    playerName: string;
    x: number;
    y: number;
    score: number;
  }[];
  enemyMobs: {
    playerId: string;
    id: string;
    x: number;
    y: number;
    carrying: number;
  }[];
  centerPile: { x: number; y: number; remaining: number };
}
```

**Actions:**
```typescript
// Return an array of orders, one per mob
{
  orders: {
    mobId: string;
    action: "move";
    direction: "up" | "down" | "left" | "right" | "stay";
  }[];
}
```

**Default action:** All mobs stay in place.

---

### Super Bomberman

**Mode:** Tick-based (300ms ticks)
**Players:** 2-4

A grid-based arena with destructible walls. Players move and place bombs that explode in cardinal directions after a short fuse. Explosions destroy walls and eliminate players. Power-ups may appear in destroyed walls (extra bombs, longer blast range, speed boost). Last player standing wins.

**Visible state:**
```typescript
{
  tick: number;
  gridSize: { width: number; height: number };
  grid: CellType[][];          // "empty" | "wall" | "destructible" | "powerup_bombs" | "powerup_range" | "powerup_speed"
  myPosition: { x: number; y: number };
  myStats: { maxBombs: number; blastRange: number; speed: number; activeBombs: number };
  players: {
    id: string;
    name: string;
    x: number;
    y: number;
    alive: boolean;
  }[];
  bombs: {
    x: number;
    y: number;
    ownerId: string;
    fuseTicksRemaining: number;
    blastRange: number;
  }[];
  explosions: {
    x: number;
    y: number;
    ticksRemaining: number;
  }[];
}
```

**Actions:**
```typescript
{ action: "move", direction: "up" | "down" | "left" | "right" }
{ action: "bomb" }    // Place bomb at current position and stay
{ action: "stay" }    // Do nothing
```

**Default action:** `{ action: "stay" }`

---

### Simplified RTS

**Mode:** Tick-based (1000ms ticks)
**Players:** 2-4

A minimal RTS on a small grid. Each player starts with a base and a few worker units. Workers gather resources from resource nodes on the map. Resources can be spent to build more workers or fighter units. Fighters can attack enemy units and buildings. A player is eliminated when their base is destroyed. Last base standing wins.

**Unit types:**
- **Worker** — Gathers resources. Can build structures. Weak in combat.
- **Fighter** — Attacks enemy units and buildings. Cannot gather.
- **Base** — Produces units. Each player has one (non-replaceable). 100 HP.

**Visible state:**
```typescript
{
  tick: number;
  gridSize: { width: number; height: number };
  myResources: number;
  myBase: { x: number; y: number; hp: number };
  myUnits: {
    id: string;
    type: "worker" | "fighter";
    x: number;
    y: number;
    hp: number;
    carrying: number;       // resources being carried (workers only)
  }[];
  enemyBases: {
    playerId: string;
    playerName: string;
    x: number;
    y: number;
    hp: number;
  }[];
  enemyUnits: {
    playerId: string;
    id: string;
    type: "worker" | "fighter";
    x: number;
    y: number;
    hp: number;
  }[];
  resourceNodes: {
    x: number;
    y: number;
    remaining: number;
  }[];
  // Fog of war: only enemies within vision range of your units are visible
}
```

**Actions:**
```typescript
{
  unitOrders: {
    unitId: string;
    action: "move" | "attack" | "gather" | "return" | "stay";
    target?: { x: number; y: number };  // for move, attack, gather
  }[];
  baseOrder?: {
    produce: "worker" | "fighter";   // if you have enough resources
  };
}
```

**Default action:** All units stay, no production.

## Open Questions

- **Spectator rendering:** Should each game module provide its own Pixi.js rendering code, or should it return a declarative scene description that a generic renderer interprets? The former is more flexible; the latter keeps game modules simpler.
- **Player identity:** Should players create accounts, or is an ephemeral name + room code sufficient? For a party game context, ephemeral is probably fine.
- **AI agent in web editor:** Should the web editor integrate an AI chat panel (calling an LLM API client-side), or is it strictly a manual code editor? Keeping it manual avoids API key management complexity.
- **Replay system:** Should games be recorded for replay? Would add complexity but could be a great post-session feature.
- **Sound:** Should the spectator display have sound effects and music? Adds atmosphere but adds asset and implementation scope.
