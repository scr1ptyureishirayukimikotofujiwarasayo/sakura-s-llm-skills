---
name: roblox-luau
description: Write, review, debug, and refactor Lua and Roblox Luau code correctly. Use this skill whenever the user mentions Roblox, Roblox Studio, Luau, .luau files, Rojo, Wally, Roblox services (Players, ReplicatedStorage, DataStoreService, RunService, etc.), RemoteEvents, ModuleScripts, game scripts, or any Lua code (.lua files, Lua 5.1/LuaJIT, Neovim/Love2D/game-engine scripting), even if they don't explicitly say "Luau". Covers Luau type annotations, the task library, client/server security, DataStore safety, memory leak prevention, project layout, and linting/type-check tooling.
license: MIT
compatibility: opencode
metadata:
  domain: lua-roblox
  audience: coding-agents
---

# Roblox Luau and Lua Coding

Guidance for LLM coding agents working on Lua and Roblox Luau codebases. The goal is code that runs on the first try, is secure by default on the Roblox client/server boundary, and passes `luau-lsp`, `selene`, and `stylua`.

## Step 0: Identify the target runtime

Before writing code, work out which Lua you are targeting. Check the repo and ask only if it's truly ambiguous.

| Signal | Runtime |
|---|---|
| `default.project.json`, `*.project.json`, `wally.toml`, `.server.luau`, `.client.luau` | Roblox via Rojo (Luau) |
| `*.rbxl`, `*.rbxm`, user says "Studio", "Roblox", "Script", "LocalScript" | Roblox (Luau) |
| `.luau` files, `.luaurc`, `--!strict` | Luau (Roblox or standalone) |
| `main.lua` + `conf.lua` | Love2D (LuaJIT) |
| `init.lua`, `lua/` directory, `vim.*` APIs | Neovim (LuaJIT) |
| `.rockspec`, plain `.lua` with no engine | Vanilla Lua 5.1 to 5.4, check version |

**Never mix dialects.** Luau code (type annotations, `continue`, `+=`, `if-then-else` expressions, `task.*`) is a syntax error in vanilla Lua. Vanilla Lua features (`goto`, `io.*`, `os.execute`, `require("path.to.mod")`) either don't exist or behave differently in Roblox.

## Step 1: Inspect before writing

1. Read existing code in the same area and match its conventions (module style, naming, tabs vs spaces, type strictness).
2. Look for tool configs: `.luaurc`, `selene.toml`, `stylua.toml`, `aftman.toml`/`rokit.toml`, `wally.toml`, `default.project.json`.
3. Find where shared code lives (`ReplicatedStorage`, `Packages`, `Shared`) before adding new modules.
4. Never invent module paths, Remote names, or attribute names. Search for them (`grep`) and reuse what exists.

## Roblox project layout (Rojo)

Common layout, adapt to what the repo already uses:

```
default.project.json
wally.toml
src/
  client/    -> StarterPlayer.StarterPlayerScripts   (*.client.luau, modules)
  server/    -> ServerScriptService                  (*.server.luau, modules)
  shared/    -> ReplicatedStorage.Shared             (modules both sides can require)
Packages/    -> ReplicatedStorage.Packages           (Wally, generated, don't edit)
ServerPackages/ -> ServerScriptService.ServerPackages
```

File naming conventions in Rojo:
- `Name.server.luau` creates a `Script` (RunContext Server)
- `Name.client.luau` creates a `LocalScript`
- `Name.luau` creates a `ModuleScript`
- `init.luau` / `init.server.luau` makes the folder itself the script, with children inside it

Never edit `Packages/` or `sourcemap.json` by hand. They are generated.

## Luau language rules

### Always use modern Luau, not legacy Lua 5.1 style

```lua
--!strict

-- Type annotations
local function damage(target: Humanoid, amount: number): boolean
	if target.Health <= 0 then
		return false
	end
	target.Health -= amount -- compound assignment: += -= *= /= //= %= ^= ..=
	return true
end

-- Generalized iteration: no pairs()/ipairs() needed
for index, value in someArray do end
for key, value in someDict do end

-- continue exists
for _, player in Players:GetPlayers() do
	if not player.Character then
		continue
	end
end

-- if-then-else expression
local label = if health > 50 then "Healthy" else "Hurt"

-- String interpolation with backticks
print(`{player.Name} has {coins} coins`)

-- Integer division
local half = total // 2
```

### Type system essentials

```lua
--!strict

export type Weapon = {
	Name: string,
	Damage: number,
	Cooldown: number,
	Tags: { string },
	Owner: Player?, -- optional
}

type Callback<T> = (value: T) -> ()

-- Union and literal types
type State = "Idle" | "Attacking" | "Dead"

-- Typed dictionaries and arrays
local cache: { [Player]: Weapon } = {}
local names: { string } = {}

-- Cast when the type checker can't infer (use sparingly)
local part = workspace:WaitForChild("Door") :: BasePart

-- Narrowing
local character = player.Character
if character then
	local humanoid = character:FindFirstChildOfClass("Humanoid")
	if humanoid then
		humanoid.Health = 100
	end
end
```

Rules:
- Put `--!strict` at the top of every new file unless the repo uses `--!nonstrict`. Don't downgrade existing files.
- Annotate all function parameters and return types on public/module functions.
- Prefer `export type` for types other modules need.
- Don't use `any` to silence errors. Fix the type or use a narrow `::` cast with a comment.
- `FindFirstChildOfClass("Humanoid")` returns `Humanoid?`. Handle nil.
- `WaitForChild` can hang forever. Give a timeout when the child is optional: `WaitForChild("X", 5)`.

### Class pattern (OOP)

```lua
--!strict

local Weapon = {}
Weapon.__index = Weapon

export type Weapon = typeof(setmetatable(
	{} :: {
		Name: string,
		Damage: number,
		_connections: { RBXScriptConnection },
	},
	Weapon
))

function Weapon.new(name: string, damage: number): Weapon
	local self = setmetatable({
		Name = name,
		Damage = damage,
		_connections = {},
	}, Weapon)
	return self
end

function Weapon.Destroy(self: Weapon)
	for _, connection in self._connections do
		connection:Disconnect()
	end
	table.clear(self._connections)
end

return Weapon
```

Always provide a `Destroy` method for any class that holds connections, instances, or tasks.

### Deprecated APIs: don't generate these

| Don't use | Use instead |
|---|---|
| `wait(n)` | `task.wait(n)` |
| `spawn(fn)` | `task.spawn(fn)` |
| `delay(n, fn)` | `task.delay(n, fn)` |
| `game.Workspace` | `workspace` or `game:GetService("Workspace")` |
| `game.Players`, `game.ReplicatedStorage` | `game:GetService("Players")` |
| `Instance.new("Part", parent)` | create, configure, then set `.Parent` last |
| `BodyVelocity`, `BodyGyro`, `BodyPosition` | `LinearVelocity`, `AlignOrientation`, `AlignPosition` |
| `Humanoid:LoadAnimation` | `Animator:LoadAnimation` |
| `table.getn`, `unpack` | `#t`, `table.unpack` |
| `os.time()` for game timing | `workspace:GetServerTimeNow()` or `os.clock()` for durations |
| `Debris:AddItem` for long-lived stuff | explicit `:Destroy()` or `task.delay` |
| `StarterGui:SetCore` assumptions | wrap in `pcall`, it can fail on load |

### Performance and style rules

- Cache services once at the top: `local Players = game:GetService("Players")`.
- Cache `Instance` references outside hot loops (`RenderStepped`, `Heartbeat`).
- Use `table.create(n)`, `table.insert`, `table.clear`, `table.freeze` for constants.
- Use `Heartbeat` for game logic, `RenderStepped` only on the client for camera and visual updates, and `Stepped` for pre-physics.
- Don't poll in `while true do task.wait() end` when an event exists (`.Changed`, `GetPropertyChangedSignal`, `AttributeChanged`, `ChildAdded`).
- Use `CollectionService` tags (and `GetInstanceAddedSignal`) to attach behavior to many objects instead of one script per object.
- Parent last: build an instance fully, then set `.Parent`.

## Client/server model and security

This is the most common source of serious bugs. **The client is untrusted.**

### Where code runs

| Container | Runs on | Visible to |
|---|---|---|
| `ServerScriptService`, `ServerStorage` | Server | Server only |
| `ReplicatedStorage` | Both | Server and clients (exploiters can read it) |
| `StarterPlayerScripts`, `StarterGui`, `StarterCharacterScripts` | Client | That client |
| `ReplicatedFirst` | Client at load | Client |

- Secrets, balancing tables, and admin lists go in `ServerStorage`/`ServerScriptService`, never `ReplicatedStorage`.
- `Script` with `RunContext = Client` and `LocalScript` run only in client containers.

### Remotes: validate everything on the server

```lua
--!strict
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")

local BuyItem = ReplicatedStorage.Remotes.BuyItem :: RemoteFunction
local lastRequest: { [Player]: number } = {}

BuyItem.OnServerInvoke = function(player: Player, itemId: unknown)
	-- 1. Type-check every argument (clients can send anything)
	if typeof(itemId) ~= "string" then
		return false, "Invalid item"
	end

	-- 2. Rate limit
	local now = os.clock()
	if now - (lastRequest[player] or 0) < 0.25 then
		return false, "Too fast"
	end
	lastRequest[player] = now

	-- 3. Check against server-side data, never trust client-sent prices/IDs/amounts
	local item = ItemCatalog[itemId]
	if not item then
		return false, "Unknown item"
	end

	-- 4. Authorize and apply on the server
	return PurchaseService.tryBuy(player, item)
end

Players.PlayerRemoving:Connect(function(player)
	lastRequest[player] = nil -- clean up per-player state
end)
```

Rules:
- The first argument of every `OnServerEvent`/`OnServerInvoke` is the player. Never accept a `player` argument from the client.
- Validate types with `typeof`, validate ranges (`math.clamp`, `NaN` check: `x ~= x`), validate that referenced Instances exist and are allowed.
- Never let the client decide damage, currency, inventory contents, cooldowns, or positions of record.
- Prefer `RemoteEvent` for fire-and-forget. Avoid `RemoteFunction` from server to client (`InvokeClient`), because a client can hang or error the server thread.
- Use `UnreliableRemoteEvent` for high-frequency cosmetic data.
- Don't use `FilteringEnabled = false` ideas, they no longer exist. Replication is always filtered.
- Client-side checks (UI gating, cooldown bars) are UX only, and the server must repeat them.

### Player lifecycle: handle players already in the game

```lua
local function onPlayerAdded(player: Player)
	-- setup
end

Players.PlayerAdded:Connect(onPlayerAdded)
for _, player in Players:GetPlayers() do
	task.spawn(onPlayerAdded, player) -- players who joined before this script ran
end

Players.PlayerRemoving:Connect(function(player)
	-- save data, clear caches
end)
```

For characters: connect `player.CharacterAdded` and also handle `player.Character` if it already exists. Set `ResetOnSpawn = false` on ScreenGuis you don't want recreated.

## DataStores: save data safely

```lua
--!strict
local DataStoreService = game:GetService("DataStoreService")
local store = DataStoreService:GetDataStore("PlayerData_v1")

local function loadData(userId: number): ({ [string]: any }?, string?)
	local ok, result = pcall(function()
		return store:GetAsync(`Player_{userId}`)
	end)
	if not ok then
		return nil, tostring(result) -- do NOT overwrite saved data after a failed load
	end
	return result or { Coins = 0 }, nil
end

local function saveData(userId: number, data: { [string]: any }): boolean
	for attempt = 1, 3 do
		local ok, err = pcall(function()
			store:UpdateAsync(`Player_{userId}`, function(old)
				return data -- merge/compare with `old` here if needed
			end)
		end)
		if ok then
			return true
		end
		warn(`Save attempt {attempt} failed: {err}`)
		task.wait(2 ^ attempt)
	end
	return false
end

game:BindToClose(function()
	-- save all players here, then yield until saves finish (30s limit)
end)
```

Rules:
- Wrap every DataStore, MarketplaceService, HttpService, and `SetCore` call in `pcall`.
- Key by `UserId`, never `Name`.
- If a load fails, kick the player or run in "no-save" mode. Never write default data over a record you failed to read.
- Use `UpdateAsync` (not `SetAsync`) for anything that can be modified from two servers.
- Save on `PlayerRemoving` **and** `BindToClose`. Autosave periodically.
- Mind rate limits: batch changes and don't save on every stat change.
- Data must be serializable: no Instances, functions, mixed array/dict tables, or `Vector3`/`CFrame` (convert to plain numbers).
- Use a version suffix in the store name so schema migrations are possible.
- Prefer a proven library (ProfileService/ProfileStore) if the project already depends on one.

## Memory leaks and cleanup

Every connection and instance you create needs an owner responsible for cleaning it up.

```lua
-- Bad: connection outlives the object it serves
part.Touched:Connect(function() end)

-- Good: track and disconnect
local connections: { RBXScriptConnection } = {}
table.insert(connections, part.Touched:Connect(onTouched))
-- later:
for _, c in connections do c:Disconnect() end
table.clear(connections)
```

- Use `Connection:Once` / `Signal:Once` for one-shot handlers.
- Clear per-player tables on `PlayerRemoving`.
- Call `:Destroy()` on instances you're done with. It also disconnects their connections.
- Use `Maid`, `Janitor`, or `Trove` if the project already includes one. Don't introduce a new library without asking.
- Avoid closures that capture large tables in long-lived connections.

## Async, yielding, and errors

```lua
-- Run something without blocking the current thread
task.spawn(function() ... end)

-- Run at end of current resumption cycle (safer ordering)
task.defer(function() ... end)

-- Delay
task.delay(5, function() ... end)

-- Protected call with traceback
local ok, err = xpcall(function()
	riskyOperation()
end, debug.traceback)
if not ok then
	warn(err)
end
```

- `error(msg, 2)` blames the caller. Use `assert(cond, msg)` for invariants.
- Throw errors with strings or tables, not bare nil.
- Don't yield in places that can't yield (metamethods, `table.sort` comparators, some callbacks).
- Use `task.cancel(thread)` for cancellable timers.
- Avoid `coroutine.wrap` for new code unless you need coroutine semantics. `task.spawn` has better error reporting in Studio.
- Promise libraries (evaera Promise, roblox-lua-promise) are fine if already used in the repo.

## Modules

```lua
--!strict
local module = {}

local privateState = 0 -- module-level state is per-environment (server vs client each get one copy)

function module.doThing(x: number): number
	privateState += x
	return privateState
end

return module -- ModuleScripts MUST return exactly one value
```

- Requires happen once per environment and results are cached.
- Avoid **circular requires**. They deadlock/error. Break cycles with dependency injection or by moving shared types to a separate `Types` module.
- Use `require(script.Parent.Foo)` or `require(ReplicatedStorage.Shared.Foo)`. Never string-path requires like `require("src/foo")` unless the repo uses a Luau-compatible require-by-string setup (check `.luaurc` aliases).
- Export types with `export type` and import with `local Types = require(...)` then `Types.Weapon`.

## Common Roblox gotchas

- `Part.Touched` fires many times; debounce per object.
- `Humanoid.Died` fires once; `Health` changes fire repeatedly.
- `workspace.CurrentCamera` can be nil early on the client. Wait for it.
- `Character` descendants load asynchronously. Use `WaitForChild` or `CharacterAppearanceLoaded`.
- `Player.Character` is `nil` between death and respawn.
- Physics ownership: set `part:SetNetworkOwner(nil)` for server-authoritative projectiles.
- `Anchored` parts don't move from physics and welded unanchored assemblies move together.
- `task.wait()` with no argument waits one frame. Use it for polling only when necessary.
- Strings: `string.split` returns a table. `utf8` library for non-ASCII. `#` counts bytes.
- `table.sort` with an inconsistent comparator throws "invalid order function".
- Arrays with `nil` holes break `#`. Use `table.remove` or explicit counters.
- Attributes only support a limited set of types (string, number, bool, Vector/CFrame/Color3/UDim/etc.).
- Studio-only checks: `RunService:IsStudio()`. Context checks: `RunService:IsServer()`, `IsClient()`.

## Vanilla Lua differences (when NOT on Roblox)

- No type annotations, no `continue`, no `+=`, no backtick strings, no `task` library.
- Lua 5.1/LuaJIT: no integer type, no `//`, no bitwise operators (use `bit` in LuaJIT).
- Lua 5.3+: integer/float split, `//`, bitwise ops, `utf8` library, `table.move`.
- `goto` exists in 5.2+ and LuaJIT; not in Luau.
- Prefer `local` everywhere; global leaks are the #1 Lua bug. Check with `luacheck` or `selene`.
- Use `ipairs` for arrays, `pairs` for dictionaries. Never rely on `pairs` order.
- For Love2D/Neovim, use the engine's own APIs and follow its module conventions.

## Tooling and verification

After editing, run whichever of these the project has configured. Don't guess flags. Check `package.json`, `Makefile`, CI config, or `aftman.toml`/`rokit.toml`.

```bash
# Format
stylua src/

# Lint
selene src/

# Generate sourcemap for the type checker (Rojo projects)
rojo sourcemap default.project.json --output sourcemap.json

# Type check with luau-lsp
luau-lsp analyze --sourcemap=sourcemap.json --definitions=globalTypes.d.luau src/

# Build place file to verify the project structure
rojo build default.project.json -o build.rbxl

# Install packages
wally install

# Run tests, if the project uses TestEZ/Jest-Lua/Lune
lune run tests   # example: check what the repo actually uses
```

Notes:
- Roblox API definitions for `luau-lsp` come from `globalTypes.d.luau`. Download it with `luau-lsp` tooling if missing. Don't fabricate one.
- If a tool isn't installed, say so and continue with careful manual review. Don't claim code was verified when it wasn't.
- You can't run Roblox game code outside Studio. State clearly which behavior was verified statically and which needs in-Studio testing.

## Output checklist

Before finishing, confirm:

- [ ] Correct dialect for the target runtime, with no Luau-only syntax in vanilla Lua files
- [ ] `--!strict` header and type annotations on new Luau modules
- [ ] No deprecated APIs (`wait`, `spawn`, `delay`, `Body*` movers)
- [ ] Services fetched with `GetService` and cached at the top
- [ ] Server validates all remote arguments (type, range, permission, rate limit)
- [ ] No secrets or authoritative data in `ReplicatedStorage`
- [ ] All DataStore/HTTP/Marketplace calls wrapped in `pcall`, with retries where appropriate
- [ ] Connections, tasks, and per-player state cleaned up
- [ ] Existing players handled alongside `PlayerAdded`
- [ ] No circular requires; module returns a single value
- [ ] Matches existing project style, file naming, and folder layout
- [ ] Linters/formatters run, or the user is told they weren't available

## Working style

- Make the smallest change that solves the problem. Don't rewrite unrelated scripts.
- When a request would be insecure (e.g. "let the client set its own coins"), implement the secure server-authoritative version and explain briefly why.
- When a Roblox API is uncertain (new or recently changed), say so instead of inventing method names. Point to the Creator Docs (create.roblox.com/docs) for confirmation.
- Explain Studio-side steps (inserting a RemoteEvent, setting RunContext) in numbered steps when the repo isn't managed by Rojo.
