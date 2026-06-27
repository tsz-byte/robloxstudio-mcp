# ROGUE MCP — Extended Roblox Studio MCP for Autonomous Game Development

Fork of `@chrrxs/robloxstudio-mcp` with gameplay-aware tools for autonomous AI game studios.

## Architecture

```
┌──────────────────────┐     HTTP      ┌───────────────────────────┐
│   MCP Server         │ ←──────────→  │   Roblox Studio Plugin    │
│   (Node.js/TS)       │   /poll       │   (roblox-ts → Luau)      │
│                      │               │                           │
│  packages/core/      │               │  studio-plugin/src/       │
│    tools/            │               │    handlers/              │
│      definitions.ts  │               │    Communication.ts       │
│      index.ts        │               │    EvalBridges.ts         │
│    http-server.ts    │               │    LuauExec.ts            │
│    server.ts         │               │    modules/               │
│                      │               │                           │
│  packages/           │               │                           │
│    robloxstudio-mcp/ │               │                           │
│      src/index.ts    │               │                           │
└──────────────────────┘               └───────────────────────────┘
         ↑                                        ↑
    AI Agent                             Roblox Studio DataModel
```

### Tool Registration Flow
1. `definitions.ts` — tool schemas (name, description, inputSchema)
2. `http-server.ts` — `TOOL_HANDLERS` mapping (name → function)
3. `tools/index.ts` — `RobloxStudioTools` class methods
4. `server.ts` — MCP server wiring

### Adding a New Tool
1. Add `ToolDefinition` to `definitions.ts`
2. Add method to `RobloxStudioTools` in `tools/index.ts`
3. Add entry to `TOOL_HANDLERS` in `http-server.ts`
4. If plugin-side handler needed: add handler in `studio-plugin/src/modules/handlers/`

## Tool Tiers

### Phase 0: MCP Server Wrappers (No Plugin Changes)
Tools that generate Luau code and call `execute_luau`/`eval_server_runtime`/`eval_client_runtime`.

| Tool | Purpose | Luau Target |
|------|---------|-------------|
| `teleport_player` | Move player to position | eval_server_runtime |
| `get_player_state` | Read player position, health, camera | eval_server_runtime |
| `get_nearby_parts` | Spatial query around a point | eval_server_runtime |
| `activate_prompt` | Fire nearest ProximityPrompt | eval_server_runtime |
| `gui_snapshot` | Read full PlayerGui tree | eval_client_runtime |
| `compound_eval` | Run N Luau steps sequentially | eval_*_runtime |
| `raycast_from_camera` | What is the player looking at | eval_client_runtime |
| `read_npc_state` | Read enemy/NPC state | eval_server_runtime |

### Phase 1: Plugin-Side Handlers (Fork Required)
Dedicated handlers for stateful/complex operations.

| Handler File | Tools |
|-------------|-------|
| `GameplayHandlers.ts` | teleport_player (fast path), activate_prompt, collect_nearby |
| `SpatialHandlers.ts` | spatial_query, raycast, pathfind_to |
| `GUIHandlers.ts` | gui_snapshot (native, faster than eval) |
| `TelemetryHandlers.ts` | track_events, get_tracked_events |
| `TestSequenceHandlers.ts` | run_dungeon_test, run_combat_test, run_navigation_test |

### Phase 2: Game-Side Persistent Module
ModuleScript injected into game VM via `eval_server_runtime`.

Provides: `GameplayHelper.getPlayerSnapshot()`, `.getNearbyParts()`, `.getGuiTree()`, `.autoAttack()`, `.trackEvent()`, `.getEvents()`

### Phase 3: Compound Test Sequences
MCP-server-side orchestration that chains Phase 0-2 tools.

| Sequence | What It Does |
|----------|-------------|
| `run_dungeon_test` | Full dungeon playthrough: spawn → all rooms → boss → extract |
| `run_combat_test` | Spawn enemies, auto-attack, measure DPS/balance |
| `run_navigation_test` | Walk waypoints, verify reachability, capture evidence |

## Build & Test

```bash
npm install
npm run build          # Build MCP server + plugin
npm run build:plugin   # Build plugin only (.rbxmx)
npm test               # Run tests
```

## Usage

```bash
# Use this fork instead of the upstream
npx tsz-byte/robloxstudio-mcp@latest --auto-install-plugin
```
