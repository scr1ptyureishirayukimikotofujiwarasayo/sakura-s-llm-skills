# Universal Video Game Cheat Development Skill

A specialized skill for AI agents to analyze **any videogame** (online or offline) and create high-quality cheat scripts across all platforms, engines, and game genres.

---

**Supported Platforms:** Unity, Unreal Engine, Roblox, Godot, Custom Engines, Native Games

**Supported Game Types:** FPS, RPG, MMO, MOBA, Battle Royale, Simulation, Strategy, Racing, Sports, Indie, AAA

**Online Games:** Client-side cheats, prediction, memory manipulation, packet inspection

**Offline Games:** Full memory access, save editing, trainer-style modifications, speedruns

---

## Table of Contents

1. [Overview](#overview)
2. [Supported Platforms](#supported-platforms)
3. [MCP Integration](#mcp-integration)
4. [Game Type Detection](#game-type-detection)
5. [Game Analysis Workflow](#game-analysis-workflow)
6. [World-to-Screen Math (ESP Foundation)](#world-to-screen-math-esp-foundation)
7. [Game Update Resilience](#game-update-resilience)
8. [Online vs Offline Cheat Strategies](#online-vs-offline-cheat-strategies)
9. [Network / Packet Manipulation](#network-packet-manipulation-online-games)
10. [Cheat Features](#cheat-features)
11. [Humanization (Behavioral Anti-Detection)](#humanization-behavioral-anti-detection)
12. [Undetection Techniques](#undetection-techniques)
13. [Configuration System](#configuration-system)
14. [UI System](#ui-system-universal)
15. [Quality Checklist](#quality-checklist)
16. [Testing Workflow](#testing-workflow)
17. [Workflow: Creating a Cheat for Any Game](#workflow-creating-a-cheat-for-any-game)
18. [Quick Start Template](#quick-start-template-universal)
19. [Platform-Specific Adapters](#platform-specific-adapters)
20. [MCP Commands](#mcp-commands)
21. [Best Practices](#best-practices)
22. [Skill Usage](#skill-usage)
23. [Quick Reference](#quick-reference)

## Overview

This skill provides workflows for:
- **Game Detection & Analysis** - Auto-detect game type, engine, and anti-cheat
- **Platform-Agnostic Development** - Write cheats that work across any game
- **Online vs Offline Optimization** - Different strategies for each game type
- **Memory & Pattern Scanning** - Find variables, functions, and game data
- **Feature Implementation** - ESP, aimbot, auto-farm, speed hacks, and more
- **Undetection Techniques** - Bypass anti-cheat and hide from detection
- **UI & Configuration** - Build menus, keybinds, and save settings

## Supported Platforms

### Game Engines
- **Unity** - C# IL2CPP, Mono, AOT
- **Unreal Engine** - C++ with Lua/Python scripting (UE4/UE5)
- **Roblox** - Lua bytecode (Luau)
- **Godot** - GDScript, C#
- **Custom Engines** - Native DLL injection, memory hooks

### Game Distribution
- **Steam** - Offline/Online games with Steamworks
- **Epic Games** - Unreal-based titles
- **GOG** - DRM-free games
- **Console Ports** - PC versions of console games
- **Indie Platforms** - itch.io, GameJolt, etc.

## MCP Integration

### Connecting to Game Instance
```lua
-- MCP connection for game analysis (universal)
local mcp = require("mcp-client")
local connection = mcp.connect({
    endpoint = "ws://localhost:8080",
    auth_token = get_auth_token(),
    platform = "auto-detect"  -- unity, unreal, roblox, custom
})

-- Query game state
local gameState = connection.query({
    type = "game_state",
    include = {"players", "objects", "functions", "memory"}
})
```

### Analyzing Game Data
```lua
-- Fetch game configuration
local config = mcp.analyze({
    target = "game",
    depth = 3,
    include_modules = true
})

-- Extract function signatures
local functions = mcp.discover({
    category = "functions",
    filter = "all",
    include_source = true
})
```

## Game Type Detection

### Auto-Detect Game Category
```lua
local function detect_game_type()
    -- Check for online multiplayer indicators
    local has_network = mcp.network.is_active()
    local player_count = mcp.player.get_count()
    
    -- Detect game genre
    local genre = mcp.game.detect_genre()
    
    return {
        is_online = has_network,
        player_count = player_count,
        genre = genre,  -- fps, rpg, mmo, moba, battle_royale, etc.
        anti_cheat = mcp.anticheat.detect(),  -- battlEye, EAC, VAC, etc.
        engine = mcp.platform.detect()
    }
end

-- Configure cheat based on game type
local game_info = detect_game_type()
if game_info.is_online then
    -- Use stealthy client-side cheats
    configure_stealth_mode()
else
    -- Use full-featured cheats
    configure_full_mode()
end
```

## Game Analysis Workflow

### 1. Object/Remote Discovery
```lua
-- Find all interactable objects/remotes (universal)
local function find_interactables()
    local interactables = {}
    
    -- Platform-specific discovery
    local discovery = mcp.discover({
        type = "interactables",
        include = {"remotes", "sockets", "events", "functions"}
    })
    
    for _, obj in pairs(discovery) do
        table.insert(interactables, {
            name = obj.name,
            type = obj.type,
            path = obj.path,
            signature = obj.signature,
            parameters = obj.parameters
        })
    end
    
    return interactables
end
```

### 2. Function Analysis
```lua
-- Analyze function behavior (universal)
local function analyze_function(func, test_args)
    local results = {}
    
    -- Fire with test arguments
    local success, result = pcall(function()
        return func:call(unpack(test_args))
    end)
    
    table.insert(results, {success = success, result = result})
    return results
end
```

### 3. Variable Discovery
```lua
-- Find important game variables (universal)
local function find_game_variables()
    local variables = {}
    
    -- MCP-based discovery
    local discovered = mcp.scan({
        type = "variables",
        patterns = {"Config", "Settings", "Data", "Player"},
        depth = 5
    })
    
    for _, var in pairs(discovered) do
        table.insert(variables, {
            name = var.name,
            type = var.type,
            address = var.address,
            value = var.value
        })
    end
    
    return variables
end
```

### 4. Memory Pattern Scanning
```lua
-- Scan memory for patterns
local function scan_memory(pattern, module)
    local results = mcp.memory.scan({
        pattern = pattern,
        module = module or "game",
        max_results = 100
    })
    
    return results
end

-- Example: Find health variable
local health_addr = scan_memory("?? ?? 38 00 00 00", "game.dll")
```

### 5. Online vs Offline Game Detection
```lua
-- Determine if game is online or offline
local function get_game_mode()
    local mode = {
        is_online = false,
        has_anti_cheat = false,
        server_authority = false,
        p2p = false
    }
    
    -- Check for network activity
    local sockets = mcp.network.list_sockets()
    mode.is_online = #sockets > 0
    
    -- Detect anti-cheat
    mode.has_anti_cheat = mcp.anticheat.is_present()
    
    -- Determine server authority (for online games)
    if mode.is_online then
        mode.server_authority = mcp.network.is_server_auth()
        mode.p2p = mcp.network.is_p2p()
    end
    
    return mode
end

-- Adjust cheat features based on game mode
local mode = get_game_mode()
if mode.is_online and mode.server_authority then
    -- Server-authoritative: focus on visual cheats (ESP, radar)
    enable_visual_only()
elseif mode.is_online and mode.p2p then
    -- P2P: can manipulate more, but still need stealth
    enable_stealth_mode()
else
    -- Offline: full features available
    enable_full_features()
end
```

### 6. Pointer Chains & Multi-Level Offsets
```lua
-- Most games store player data via pointer chains:
-- [base.exe + 0x12345] -> +0x10 -> +0x24 -> health
local PointerChain = {
    -- Resolve a multi-level pointer
    resolve = function(self, base_module, offsets)
        local addr = mcp.memory.get_module_base(base_module)
        
        for i, offset in ipairs(offsets) do
            if i == #offsets then
                -- Last offset: this is the target address
                return addr + offset
            end
            -- Dereference pointer
            addr = mcp.memory.read_uint64(addr + offset)
            if addr == 0 or addr == nil then
                return nil  -- Invalid chain, game not loaded yet
            end
        end
    end,
    
    -- Example: health = game.exe + 0x12345 -> +0x10 -> +0x24
    get_health = function(self)
        return self:resolve("game.exe", {0x12345, 0x10, 0x24})
    end
}

-- Offset database for multiple game versions
local OffsetDatabase = {
    ["game v1.0.0"] = {
        health = {base = "game.exe", offsets = {0x12345, 0x10, 0x24}},
        ammo   = {base = "game.exe", offsets = {0x12345, 0x10, 0x30}},
        money  = {base = "game.exe", offsets = {0x54321, 0x8, 0x14}},
    },
    ["game v1.1.0"] = {
        health = {base = "game.exe", offsets = {0x12399, 0x10, 0x24}},  -- Patch moved it
        ammo   = {base = "game.exe", offsets = {0x12399, 0x10, 0x30}},
        money  = {base = "game.exe", offsets = {0x54388, 0x8, 0x14}},
    },
    
    -- Auto-detect game version and select offsets
    get_offsets = function(self)
        local version = mcp.game.get_version()
        return self[version] or self["game v1.0.0"]
    end
}
```

### 7. Anti-Cheat Detection
```lua
local function detect_anticheat()
    local anti_cheats = {
        "Easy Anti-Cheat",
        "BattlEye",
        "VAC / VAC2",
        "NProtect",
        "Tencent ACE",
        "Ricochet",
        " Vanguard",
        "Custom"
    }
    
    local detected = {}
    for _, ac in pairs(anti_cheats) do
        if mcp.anticheat.detect(ac) then
            table.insert(detected, ac)
        end
    end
    
    return detected
end

-- Configure based on anti-cheat
local ac_list = detect_anticheat()
if #ac_list > 0 then
    print("Detected anti-cheat:", table.concat(ac_list, ", "))
    apply_anticheat_bypass(ac_list)
end
```

### 8. Value Scanning (Find Unknown Addresses)
```lua
-- Classic Cheat Engine style scanning: find address by value changes
local ValueScanner = {
    scan_id = nil,
    
    -- Step 1: First scan for known value
    first_scan = function(self, value, type)
        self.scan_id = mcp.memory.value_scan_start(value, type or "float")
        return mcp.memory.value_scan_count(self.scan_id)
    end,
    
    -- Step 2: Narrow down after value changes in-game
    next_scan = function(self, new_value)
        mcp.memory.value_scan_filter(self.scan_id, new_value)
        return mcp.memory.value_scan_count(self.scan_id)
    end,
    
    -- Step 3: Repeat until a few addresses remain, then test
    find_health = function(self)
        -- Player at full health: 100
        self:first_scan(100)
        -- Player takes damage: 73
        self:next_scan(73)
        -- Heals: 100
        self:next_scan(100)
        -- Get remaining candidates
        return mcp.memory.value_scan_results(self.scan_id)
    end
}
```

### 9. Entity List Traversal
```lua
-- Games store entities in arrays/lists - walk them for ESP/aimbot
local EntityManager = {
    cache = {},
    cache_time = 0,
    cache_duration = 0.1,  -- Re-scan every 100ms
    
    get_entities = function(self)
        -- Refresh cache periodically (massive performance gain)
        if tick() - self.cache_time < self.cache_duration then
            return self.cache
        end
        
        local entities = {}
        local list_head = mcp.memory.read_uint64(entity_list_ptr)
        local current = list_head
        
        local i = 0
        while current ~= 0 and current ~= list_head and i < 100 do
            local entity = mcp.memory.parse_entity(current)
            if entity and entity.alive then
                table.insert(entities, entity)
            end
            current = entity.next_ptr
            i = i + 1
        end
        
        self.cache = entities
        self.cache_time = tick()
        return entities
    end
}
```

```lua
local function detect_anticheat()
    local anti_cheats = {
        "Easy Anti-Cheat",
        "BattlEye",
        "VAC / VAC2",
        "EAC",
        "NProtect",
        "Tencent ACE",
        "Kolibri",
        "Rich Presence",
        "Custom"
    }
    
    local detected = {}
    for _, ac in pairs(anti_cheats) do
        if mcp.anticheat.detect(ac) then
            table.insert(detected, ac)
        end
    end
    
    return detected
end

-- Configure based on anti-cheat
local ac_list = detect_anticheat()
if #ac_list > 0 then
    print("Detected anti-cheat:", table.concat(ac_list, ", "))
    apply_anticheat_bypass(ac_list)
end
```

## World-to-Screen Math (ESP Foundation)

The core of every visual cheat — projecting 3D world positions to 2D screen coordinates:

### Manual W2S Calculation
```lua
local W2S = {
    -- Compute screen position from world position using view matrix
    project = function(self, world_pos, view_matrix, screen_size)
        -- Transform world position by view-projection matrix
        local x = view_matrix[1][1] * world_pos.x + view_matrix[1][2] * world_pos.y
                 + view_matrix[1][3] * world_pos.z + view_matrix[1][4]
        local y = view_matrix[2][1] * world_pos.x + view_matrix[2][2] * world_pos.y
                 + view_matrix[2][3] * world_pos.z + view_matrix[2][4]
        local w = view_matrix[3][1] * world_pos.x + view_matrix[3][2] * world_pos.y
                 + view_matrix[3][3] * world_pos.z + view_matrix[3][4]
        
        -- Behind camera check
        if w < 0.01 then
            return nil, false
        end
        
        -- NDC (Normalized Device Coordinates) -> screen pixels
        local inv_w = 1 / w
        local screen_x = (x * inv_w * 0.5 + 0.5) * screen_size.width
        local screen_y = (1 - (y * inv_w * 0.5 + 0.5)) * screen_size.height
        
        return {x = screen_x, y = screen_y}, true
    end,
    
    -- Get the view matrix from the camera (engine-specific)
    get_view_matrix = function(self)
        -- Unity: Camera.current.worldToCameraMatrix
        -- Unreal: PlayerController.PlayerCameraManager.GetCameraCachePOV
        -- Source: IEngineClient.WorldToScreenInfo
        return mcp.camera.get_view_matrix()
    end
}
```

### Bounding Box Calculation (Box ESP)
```lua
local BoxESP = {
    -- Draw a box around an entity using head + feet positions
    get_box = function(self, entity)
        local head_pos = entity:get_bone("head"):get_position()
        local feet_pos = entity:get_bone("feet"):get_position()
        
        local head_screen = mcp.camera.world_to_screen(head_pos)
        local feet_screen = mcp.camera.world_to_screen(feet_pos)
        if not head_screen or not feet_screen then return nil end
        
        -- Height in pixels, width proportional to body
        local height = math.abs(feet_screen.y - head_screen.y)
        local width = height * 0.45  -- typical humanoid ratio
        
        return {
            x = head_screen.x - width / 2,
            y = head_screen.y,
            w = width,
            h = height
        }
    end
}
```

### Bone/Skeleton System
```lua
local SkeletonSystem = {
    -- Common humanoid bone map (Biped-style, works for most games)
    bones = {
        "head", "neck",
        "spine_1", "spine_2", "spine_3", "pelvis",
        "l_shoulder", "l_elbow", "l_hand",
        "r_shoulder", "r_elbow", "r_hand",
        "l_hip", "l_knee", "l_foot",
        "r_hip", "r_knee", "r_foot"
    },
    
    -- Skeleton connections for skeleton ESP
    connections = {
        {"head", "neck"}, {"neck", "spine_1"},
        {"spine_1", "spine_2"}, {"spine_2", "spine_3"}, {"spine_3", "pelvis"},
        {"neck", "l_shoulder"}, {"l_shoulder", "l_elbow"}, {"l_elbow", "l_hand"},
        {"neck", "r_shoulder"}, {"r_shoulder", "r_elbow"}, {"r_elbow", "r_hand"},
        {"pelvis", "l_hip"}, {"l_hip", "l_knee"}, {"l_knee", "l_foot"},
        {"pelvis", "r_hip"}, {"r_hip", "r_knee"}, {"r_knee", "r_foot"}
    },
    
    draw = function(self, entity, color)
        -- Resolve bone world positions
        local positions = {}
        for _, bone in pairs(self.bones) do
            local b = entity:get_bone(bone)
            if b then
                positions[bone] = mcp.camera.world_to_screen(b:get_position())
            end
        end
        
        -- Draw skeleton lines
        for _, conn in pairs(self.connections) do
            local from, to = positions[conn[1]], positions[conn[2]]
            if from and to then
                mcp.graphics.draw_line(from, to, color)
            end
        end
    end
}
```

## Game Update Resilience

Games patch frequently and hardcoded addresses break. Pattern-based offsets survive updates:

### Signature-Based Offsets
```lua
local ResilientOffsets = {
    -- Instead of: health_addr = 0x12345678 (breaks every patch)
    -- Use: find the code that ACCESS health, then extract the offset
    find_health_offset = function(self)
        -- Find the instruction that reads health, e.g.:
        -- movss xmm0, [rcx + 0x24]   <- "0x24" is the health offset
        local result = mcp.memory.scan({
            pattern = "F3 0F 10 81 24 00 00 00",  -- movss xmm0,[rcx+24]
            module = "game.exe"
        })
        
        if result then
            -- Extract the offset bytes from the instruction itself
            local offset = mcp.memory.read_ordinal_bytes(result.address + 4, 4)
            return offset  -- Automatically correct after patches!
        end
        return nil  -- Pattern changed, needs manual re-analysis
    end,
    
    -- Verify offsets are still valid (sanity checks)
    validate = function(self, addr, expected_range)
        local value = mcp.memory.read_float(addr)
        -- Health should be 0-1000, if we read 4.2e8 the offset is wrong
        return value >= 0 and value <= expected_range
    end
}
```

### Auto-Healing Cheat
```lua
local AutoHeal = {
    -- Self-correct when offsets break after a game update
    check_interval = 60,  -- Re-validate every 60 seconds
    
    validate_all = function(self, offsets)
        local broken = {}
        for name, addr in pairs(offsets) do
            if not ResilientOffsets:validate(addr, 10000) then
                table.insert(broken, name)
            end
        end
        
        if #broken > 0 then
            Notifications:warning("Offsets broken after game update: " .. table.concat(broken, ", "))
            -- Attempt auto-recovery via patterns
            for _, name in pairs(broken) do
                self:recover_offset(name)
            end
        end
    end,
    
    recover_offset = function(self, name)
        -- Re-scan patterns to find the new offset
        local new_offset = ResilientOffsets["find_" .. name](ResilientOffsets)
        if new_offset then
            offset_database[name] = new_offset
            Notifications:success("Auto-recovered offset: " .. name)
        else
            Notifications:error("Failed to recover: " .. name .. " (manual analysis needed)")
        end
    end
}
```

### Version Tracking
```lua
-- Log the game version with each offset set for debugging
local OffsetLog = {
    record = function(self)
        local entry = {
            game_version = mcp.game.get_version(),
            timestamp = os.time(),
            offsets = offset_database
        }
        mcp.file.append("offset_log.json", mcp.json.stringify(entry))
    end
}
```

## Online vs Offline Cheat Strategies

### Online Games (Server-Authoritative)
```lua
-- Focus on client-side visual enhancements
local OnlineCheat = {
    -- Safe features for server-authoritative games
    visual_features = {
        ESP = true,          -- Extra Sensory Perception
        Radar = true,        -- Mini radar
        Skeleton = true,     -- Skeleton ESP
        Triggerbot = true,   -- Auto-fire on crosshair
        SkinChanger = true   -- Visual only
    },
    
    -- Risky features (may get detected)
    risky_features = {
        Aimbot = "low_smooth",   -- Use high smoothing
        SpeedHack = "prediction", -- Predict server reconciliation
        NoRecoil = "visual"       -- Visual only
    },
    
    configure = function(self)
        -- Apply stealth settings
        self.aimbot.smooth = 15  -- Higher = smoother, less detectable
        self.triggerbot.reaction = 0.15  -- Human-like delay
        self.esp.draw_distance = 500  -- Don't show too far
    end
}
```

### Online Games (P2P / Client-Authoritative)
```lua
-- More features available but still need stealth
local P2PCheat = {
    features = {
        ESP = true,
        Aimbot = true,
        SpeedHack = true,
        FlyHack = true,
        NoClip = true,
        DamageBoost = true
    },
    
    configure = function(self)
        -- Moderate stealth
        self.aimbot.smooth = 8
        self.speedhack.multiplier = 2.5  -- Keep reasonable
    end
}
```

### Offline Games (Full Features)
```lua
-- All features available, no detection concerns
local OfflineCheat = {
    features = {
        -- Movement
        FlyHack = true,
        SpeedHack = true,
        NoClip = true,
        BunnyHop = true,
        JumpBoost = true,
        
        -- Combat
        Aimbot = true,
        Triggerbot = true,
        OneHitKill = true,
        InfiniteAmmo = true,
        NoRecoil = true,
        RapidFire = true,
        
        -- Visual
        ESP = true,
        Radar = true,
        Wireframe = true,
        NightVision = true,
        FOVChanger = true,
        
        -- Game Mechanics
        InfiniteHealth = true,
        InfiniteStamina = true,
        GodMode = true,
        UnlimitedMoney = true,
        UnlockAll = true,
        FastTravel = true,
        AutoLevel = true
    },
    
    configure = function(self)
        -- No stealth needed
        self.aimbot.smooth = 1  -- Instant aim
        self.speedhack.multiplier = 10  -- Any value
    end
}
```

### Save File Editing (Offline Games)
```lua
local SaveEditor = {
    save_path = nil,
    encrypted = false,
    
    -- Find save file location
    find_save = function(self)
        -- Common save locations
        local paths = {
            "~\\Documents\\My Games\\",
            "~\\AppData\\Local\\",
            "~\\AppData\\Roaming\\",
            "~\\Saved Games\\",
            "~\\Documents\\"
        }
        
        for _, path in pairs(paths) do
            local full_path = path .. mcp.game.get_title()
            if mcp.file.exists(full_path) then
                self.save_path = full_path
                return true
            end
        end
        
        return false
    end,
    
    -- Load and parse save file
    load = function(self)
        local data = mcp.file.read(self.save_path)
        
        if self.encrypted then
            data = self.decrypt(data)
        end
        
        return mcp.json.parse(data)
    end,
    
    -- Modify save data
    modify = function(self, save_data, modifications)
        for key, value in pairs(modifications) do
            save_data[key] = value
        end
        return save_data
    end,
    
    -- Common modifications
    cheats = {
        unlimited_money = function(save)
            save.money = 999999999
            save.credits = 999999999
        end,
        max_level = function(save)
            save.level = 999
            save.xp = math.huge
        end,
        unlock_all = function(save)
            save.unlocked_items = {}
            save.unlocked_levels = {}
            save.unlocked_characters = {}
            -- Mark everything as unlocked
            for key in pairs(save) do
                if string.find(key, "unlock") then
                    save[key] = true
                end
            end
        end,
        god_mode = function(save)
            save.invincible = true
            save.max_health = 9999
            save.health = 9999
        end
    },
    
    -- Apply cheat and save
    apply = function(self, cheat_name)
        local save_data = self:load()
        local cheat = self.cheats[cheat_name]
        
        if cheat then
            cheat(save_data)
        end
        
        -- Save back
        local data = mcp.json.stringify(save_data)
        if self.encrypted then
            data = self.encrypt(data)
        end
        mcp.file.write(self.save_path, data)
    end
}
```

### Trainer-Style Features (Offline Games)
```lua
local Trainer = {
    features = {},
    hotkeys = {},
    
    -- Add trainer feature
    add_feature = function(self, name, toggle_func, hotkey)
        self.features[name] = {
            enabled = false,
            toggle = toggle_func
        }
        self.hotkeys[hotkey] = name
        
        -- Bind hotkey
        mcp.input.bind(hotkey, function()
            self:toggle_feature(name)
            Notifications:info(name .. " toggled: " .. tostring(not self.features[name].enabled))
        end)
    end,
    
    -- Toggle feature
    toggle_feature = function(self, name)
        local feature = self.features[name]
        feature.enabled = not feature.enabled
        feature.toggle(feature.enabled)
    end,
    
    -- Common trainer features
    create_common = function(self)
        -- F1: God Mode
        self:add_feature("God Mode", function(enabled)
            mcp.memory.write_float(god_mode_addr, enabled and 1 or 0)
        end, "F1")
        
        -- F2: Unlimited Ammo
        self:add_feature("Unlimited Ammo", function(enabled)
            if enabled then
                mcp.memory.write_int(ammo_addr, 9999)
                coroutine.spawn(function()
                    while self.features["Unlimited Ammo"].enabled do
                        mcp.memory.write_int(ammo_addr, 9999)
                        coroutine.wait(0.5)
                    end
                end)
            end
        end, "F2")
        
        -- F3: Speed Up
        self:add_feature("Speed Up", function(enabled)
            mcp.game.set_time_scale(enabled and 2.0 or 1.0)
        end, "F3")
        
        -- F4: Infinite Health
        self:add_feature("Infinite Health", function(enabled)
            if enabled then
                coroutine.spawn(function()
                    while self.features["Infinite Health"].enabled do
                        local player = mcp.player.get_local()
                        if player then
                            player:set_health(player:max_health())
                        end
                        coroutine.wait(0.1)
                    end
                end)
            end
        end, "F4")
        
        -- F5: Unlimited Money
        self:add_feature("Unlimited Money", function(enabled)
            if enabled then
                mcp.memory.write_int(money_addr, 999999999)
            end
        end, "F5")
    end
}
```

## Network / Packet Manipulation (Online Games)

### Packet Sniffing
```lua
-- Inspect network traffic between client and server
local PacketSniffer = {
    hooks = {},
    
    start = function(self)
        -- Hook the game's send/recv functions
        mcp.network.hook_send(function(packet)
            self:on_send(packet)
            return packet  -- Return modified or original
        end)
        
        mcp.network.hook_recv(function(packet)
            self:on_recv(packet)
            return packet
        end)
    end,
    
    on_send = function(self, packet)
        -- Log all outgoing packets
        print(string.format("[SEND] opcode=%d size=%d", packet.opcode, packet.size))
        
        -- Example: detect movement packets
        if packet.opcode == OPCODES.MOVE then
            local move = parse_move_packet(packet)
            table.insert(self.hooks, {type = "move", data = move})
        end
    end,
    
    on_recv = function(self, packet)
        -- Log incoming packets (server -> client)
        print(string.format("[RECV] opcode=%d size=%d", packet.opcode, packet.size))
    end
}
```

### Packet Modification
```lua
-- Modify packets before they reach the server
local PacketMod = {
    -- Example: remove recoil info from shooting packets
    modify_shoot = function(self, packet)
        if packet.opcode == OPCODES.SHOOT then
            -- Zero out recoil/spread values
            packet.data.spread = 0.0
            packet.data.recoil = 0.0
        end
        return packet
    end,
    
    -- Example: teleport via malformed movement packet
    teleport = function(self, new_pos)
        local packet = create_move_packet()
        packet.position = new_pos
        packet.velocity = {x = 0, y = 0, z = 0}  -- Suspicious: instant teleport
        mcp.network.send(packet)
    end,
    
    -- Safer: move in small steps to avoid server-side speed checks
    safe_teleport = function(self, new_pos, step_distance)
        coroutine.spawn(function()
            local current = mcp.player.get_local():get_position()
            while (mcp.math.distance(current, new_pos) > step_distance) do
                local direction = mcp.math.normalize(new_pos - current)
                current = current + direction * step_distance
                mcp.network.send(create_move_packet(current))
                coroutine.wait(0.05)  -- ~20 moves/sec looks like lag, not cheat
            end
            mcp.network.send(create_move_packet(new_pos))
        end)
    end
}
```

### Common Protocol Structures
```lua
-- Typical packet layout (varies by game)
local PacketStructure = {
    header = {
        size = 2,      -- uint16 packet length
        opcode = 2,    -- uint16 operation code
        checksum = 4,  -- uint32 CRC/validation
    },
    payload = "variable"
}

-- Rebuild the checksum after modifying packets
local function rebuild_checksum(packet)
    packet.checksum = mcp.network.crc32(packet.payload)
    return packet
end
```

## Cheat Features

### Game Genre Detection & Feature Selection
```lua
-- Auto-select features based on game genre
local function configure_for_genre(genre)
    local genre_configs = {
        fps = {
            essential = {"Aimbot", "ESP", "Triggerbot", "Radar"},
            optional = {"NoRecoil", "SkinChanger", "ScopeESP"}
        },
        rpg = {
            essential = {"AutoQuest", "AutoLoot", "ESP"},
            optional = {"SpeedHack", "AutoLevel", "UnlockAll"}
        },
        mmo = {
            essential = {"AutoFarm", "AutoQuest", "ESP"},
            optional = {"SpeedHack", "AutoLoot", "Radar"}
        },
        moba = {
            essential = {"ESP", "Radar", "AutoLevel"},
            optional = {"AutoFarm", "Triggerbot", "Prediction"}
        },
        battle_royale = {
            essential = {"ESP", "Radar", "Aimbot"},
            optional = {"AutoLoot", "Triggerbot", "BoxESP"}
        },
        racing = {
            essential = {"SpeedHack", "UnlockAll"},
            optional = {"NitroBoost", "NoCrash", "TimeSlow"}
        },
        strategy = {
            essential = {"AutoBuild", "UnlimitedResources"},
            optional = {"FogOfWar", "SpeedHack", "AutoAttack"}
        }
    }
    
    local config = genre_configs[genre] or genre_configs.fps
    load_features(config.essential, config.optional)
end
```

### Auto Farm (Universal)
```lua
local AutoFarm = {
    enabled = false,
    target = nil,
    delay = 0.5,
    auto_collect = true,
    
    function = function(self)
        while self.enabled do
            local target = self.find_target()
            if target then
                self.interact(target)
            end
            coroutine.wait(self.delay)
        end
    end,
    
    find_target = function(self)
        -- Platform-agnostic target finding
        local targets = mcp.scan({
            type = "farmable",
            filters = self.filters
        })
        
        if #targets > 0 then
            return targets[1]
        end
        return nil
    end,
    
    interact = function(self, target)
        -- Trigger interaction (platform-specific)
        local interaction = mcp.interact(target)
        if interaction then
            interaction:call()
        end
    end
}
```

### Auto Click (Universal)
```lua
local AutoClick = {
    enabled = false,
    click_delay = 0.1,
    target_filter = function(obj) return true end,
    
    function = function(self)
        while self.enabled do
            local mouse = self.get_mouse_position()
            local target = self.find_click_target(mouse)
            
            if target and self.target_filter(target) then
                self.click(target)
            end
            
            coroutine.wait(self.click_delay)
        end
    end,
    
    get_mouse_position = function(self)
        -- Platform-agnostic mouse position
        return mcp.input.get_mouse()
    end,
    
    click = function(self, target)
        -- Send click event
        mcp.input.click(target)
    end
}
```

### Speed Hack (Universal)
```lua
local SpeedHack = {
    enabled = false,
    multiplier = 2,
    original_value = nil,
    
    function = function(self)
        if self.enabled then
            local player = mcp.player.get_local()
            if player then
                local speed_addr = player:get_field("speed")
                if speed_addr then
                    speed_addr:set(speed_addr:get() * self.multiplier)
                end
            end
        else
            local player = mcp.player.get_local()
            if player then
                local speed_addr = player:get_field("speed")
                if speed_addr and self.original_value then
                    speed_addr:set(self.original_value)
                end
            end
        end
    end,
    
    init = function(self)
        local player = mcp.player.get_local()
        if player then
            local speed_addr = player:get_field("speed")
            if speed_addr then
                self.original_value = speed_addr:get()
            end
        end
    end
}
```

### Auto Farm (Advanced - Universal)
```lua
local AdvancedAutoFarm = {
    enabled = false,
    farm_items = {},
    priority = "value",
    auto_sell = false,
    sell_threshold = 100,
    
    function = function(self)
        while self.enabled do
            -- Find best farm target
            local target = self.find_best_target()
            if target then
                self.farm(target)
            end
            
            -- Auto sell if enabled
            if self.auto_sell then
                self.check_and_sell()
            end
            
            coroutine.wait(0.1)
        end
    end,
    
    find_best_target = function(self)
        local best = nil
        local best_value = 0
        
        for _, item in pairs(self.farm_items) do
            local found = mcp.scan({type = "item", name = item})
            if found and #found > 0 then
                local value = self.calculate_value(found[1])
                if value > best_value then
                    best = found[1]
                    best_value = value
                end
            end
        end
        
        return best
    end,
    
    calculate_value = function(self, item)
        if self.priority == "value" then
            return item:get_field("value") or 0
        elseif self.priority == "rarity" then
            return self.get_rarity_score(item)
        end
        return 0
    end
}
```

### Auto Loot (Universal)
```lua
local AutoLoot = {
    enabled = false,
    filter = "all",  -- all, valuable, rare, custom
    auto_equip = false,
    
    function = function(self)
        while self.enabled do
            local lootables = mcp.scan({type = "lootable"})
            
            for _, loot in pairs(lootables) do
                if self.should_loot(loot) then
                    self.loot(loot)
                end
            end
            
            coroutine.wait(0.05)
        end
    end,
    
    should_loot = function(self, item)
        if self.filter == "all" then return true end
        if self.filter == "valuable" then
            return item:get_field("value") > 100
        end
        return true
    end
}
```

### Auto Quest (Universal)
```lua
local AutoQuest = {
    enabled = false,
    auto_accept = true,
    priority = "reward",
    
    function = function(self)
        while self.enabled do
            local quests = mcp.query({type = "active_quests"})
            
            for _, quest in pairs(quests) do
                if quest.accepted and not quest.completed then
                    self.complete_quest(quest)
                elseif self.auto_accept and not quest.accepted then
                    self.accept_quest(quest)
                end
            end
            
            coroutine.wait(0.1)
        end
    end
}
```

### ESP (Extra Sensory Perception - Universal)
```lua
local ESP = {
    enabled = false,
    settings = {
        draw_boxes = true,
        draw_names = true,
        draw_health = true,
        draw_distance = true,
        draw_skeleton = false,
        colors = {
            friendly = {r=0, g=255, b=0},
            enemy = {r=255, g=0, b=0},
            neutral = {r=255, g=255, b=0}
        }
    },
    
    function = function(self)
        while self.enabled do
            self.draw_esp()
            coroutine.wait()
        end
    end,
    
    draw_esp = function(self)
        local entities = mcp.scan({type = "entities", visible = true})
        local camera = mcp.camera.get()
        
        for _, entity in pairs(entities) do
            if entity == mcp.player.get_local() then continue end
            
            local position = entity:get_position()
            local screen_pos, on_screen = camera:world_to_screen(position)
            
            if on_screen then
                self.draw_entity_info(entity, screen_pos, position)
            end
        end
    end,
    
    draw_entity_info = function(self, entity, screen_pos, world_pos)
        local camera_pos = mcp.camera.get():get_position()
        
        -- Draw box
        if self.settings.draw_boxes then
            mcp.graphics.draw_box(screen_pos, entity.bounding_box, self.get_color(entity))
        end
        
        -- Draw name
        if self.settings.draw_names then
            mcp.graphics.draw_text(screen_pos.x, screen_pos.y - 15, entity.name, self.get_color(entity))
        end
        
        -- Draw health bar
        if self.settings.draw_health then
            local health = entity:get_field("health") or 100
            local max_health = entity:get_field("max_health") or 100
            mcp.graphics.draw_health_bar(screen_pos, health, max_health)
        end
        
        -- Draw distance
        if self.settings.draw_distance then
            local distance = mcp.math.distance(world_pos, camera_pos)
            mcp.graphics.draw_text(screen_pos.x, screen_pos.y + 15, string.format("%.1fm", distance))
        end
    end,
    
    get_color = function(self, entity)
        if entity.team == "friendly" then
            return self.settings.colors.friendly
        elseif entity.team == "enemy" then
            return self.settings.colors.enemy
        end
        return self.settings.colors.neutral
    end
}
```

### Aimbot (Universal)
```lua
local Aimbot = {
    enabled = false,
    fov = 180,
    target_priority = "distance",
    smooth = 5,
    auto_fire = true,
    fire_delay = 0.1,
    bones = "head",  -- head, spine, auto
    
    function = function(self)
        while self.enabled do
            local target = self.find_target()
            if target then
                self.aim_at(target)
                
                if self.auto_fire then
                    self.fire_weapon()
                    coroutine.wait(self.fire_delay)
                end
            end
            coroutine.wait()
        end
    end,
    
    find_target = function(self)
        local best_target = nil
        local best_score = 0
        
        local entities = mcp.scan({type = "enemies", visible = true})
        
        for _, entity in pairs(entities) do
            local target_bone = self.get_target_bone(entity)
            if not target_bone then continue end
            
            local angle = self.calculate_angle(target_bone)
            if angle <= self.fov then
                local score = self.calculate_score(entity, target_bone, angle)
                if score > best_score then
                    best_score = score
                    best_target = target_bone
                end
            end
        end
        
        return best_target
    end,
    
    get_target_bone = function(self, entity)
        if self.bones == "head" then
            return entity:get_bone("head")
        elseif self.bones == "spine" then
            return entity:get_bone("spine")
        end
        return entity:get_center()
    end,
    
    calculate_score = function(self, entity, target, angle)
        if self.target_priority == "distance" then
            local camera_pos = mcp.camera.get():get_position()
            return mcp.math.distance(target.position, camera_pos)
        elseif self.target_priority == "health" then
            return entity:get_field("health") or 0
        elseif self.target_priority == "angle" then
            return angle
        end
        return 0
    end,
    
    aim_at = function(self, target)
        local camera = mcp.camera.get()
        local screen_pos = camera:world_to_screen(target.position)
        
        -- Smooth aim
        local current = mcp.input.get_mouse()
        local target_pos = {
            x = current.x + (screen_pos.x - current.x) / self.smooth,
            y = current.y + (screen_pos.y - current.y) / self.smooth
        }
        
        mcp.input.move_mouse(target_pos)
    end,
    
    fire_weapon = function(self)
        mcp.input.click("left")
    end
}
```

### Triggerbot (Universal)
```lua
local Triggerbot = {
    enabled = false,
    fov = 50,
    auto_reload = true,
    reaction_time = 0.05,
    
    function = function(self)
        while self.enabled do
            local target = self.in_crosshair()
            if target then
                coroutine.wait(self.reaction_time)
                self.fire()
            end
            coroutine.wait()
        end
    end,
    
    in_crosshair = function(self)
        local entities = mcp.scan({type = "enemies"})
        local mouse_pos = mcp.input.get_mouse()
        
        for _, entity in pairs(entities) do
            local bone = self.get_target_bone(entity)
            if not bone then continue end
            
            local screen_pos = mcp.camera.get():world_to_screen(bone.position)
            local angle = self.calculate_angle(screen_pos, mouse_pos)
            
            if angle <= self.fov then
                return entity
            end
        end
        return nil
    end
}
```

### Radar (Universal)
```lua
local Radar = {
    enabled = false,
    size = 150,
    position = {x = 0, y = 0},
    show_enemies = true,
    show_teammates = true,
    
    function = function(self)
        while self.enabled do
            self.draw_radar()
            coroutine.wait()
        end
    end,
    
    draw_radar = function(self)
        local player_pos = mcp.player.get_local():get_position()
        local entities = mcp.scan({type = "all_players"})
        
        mcp.graphics.draw_circle(self.position.x, self.position.y, self.size, {r=50, g=50, b=50})
        
        for _, entity in pairs(entities) do
            local pos = entity:get_position()
            local angle = mcp.math.angle(player_pos, pos)
            local distance = mcp.math.distance(player_pos, pos)
            
            if distance <= self.size * 2 then
                local radar_x = self.position.x + math.cos(angle) * distance / 2
                local radar_y = self.position.y + math.sin(angle) * distance / 2
                
                mcp.graphics.draw_point(radar_x, radar_y, self.get_color(entity))
            end
        end
    end
}
```

### Genre-Specific Features

#### FPS Games (Call of Duty, CS:GO, Apex, Valorant)
```lua
local FPSCheats = {
    -- Combat
    Aimbot = Aimbot,
    Triggerbot = Triggerbot,
    NoRecoil = {
        enabled = false,
        strength = 1.0,
        function = function(self)
            while self.enabled do
                mcp.player.remove_recoil(self.strength)
                coroutine.wait()
            end
        end
    },
    
    -- Movement
    BunnyHop = {
        enabled = false,
        auto_crouch = true,
        function = function(self)
            while self.enabled do
                local player = mcp.player.get_local()
                if player:is_airborne() then
                    player:jump()
                elseif player:is_grounded() then
                    if self.auto_crouch then player:crouch() end
                    player:jump()
                    coroutine.wait(0.1)
                    player:uncrouch()
                end
                coroutine.wait()
            end
        end
    },
    
    -- Visual
    ScopeESP = {
        enabled = false,
        draw_through_walls = true,
        function = function(self)
            while self.enabled do
                local enemies = mcp.scan({type = "enemies"})
                for _, enemy in pairs(enemies) do
                    if self.draw_through_walls or mcp.math.has_line_of_sight(enemy) then
                        mcp.graphics.draw_scope_highlight(enemy)
                    end
                end
                coroutine.wait()
            end
        end
    }
}
```

#### RPG/MMO Games (WoW, FFXIV, Elden Ring, Skyrim)
```lua
local RPGCheats = {
    -- Progression
    AutoQuest = AutoQuest,
    AutoLevel = {
        enabled = false,
        target_level = 999,
        function = function(self)
            while self.enabled do
                local player = mcp.player.get_local()
                local current_level = player:get_level()
                if current_level < self.target_level then
                    player:add_xp(math.huge)
                end
                coroutine.wait(1)
            end
        end
    },
    
    -- Combat
    AutoFarm = AutoFarm,
    OneHitKill = {
        enabled = false,
        function = function(self)
            while self.enabled do
                local player = mcp.player.get_local()
                player:set_damage_multiplier(9999)
                coroutine.wait()
            end
        end
    },
    
    -- Survival
    GodMode = {
        enabled = false,
        infinite_health = true,
        infinite_stamina = true,
        infinite_mana = true,
        function = function(self)
            while self.enabled do
                local player = mcp.player.get_local()
                if self.infinite_health then player:set_health(player:max_health()) end
                if self.infinite_stamina then player:set_stamina(player:max_stamina()) end
                if self.infinite_mana then player:set_mana(player:max_mana()) end
                coroutine.wait()
            end
        end
    },
    
    -- Economy
    UnlimitedMoney = {
        enabled = false,
        amount = 999999999,
        function = function(self)
            while self.enabled do
                local player = mcp.player.get_local()
                player:set_money(self.amount)
                coroutine.wait(1)
            end
        end
    }
}
```

#### Battle Royale Games (Fortnite, PUBG, Warzone)
```lua
local BattleRoyaleCheats = {
    -- Essential
    ESP = ESP,
    Radar = Radar,
    AutoLoot = AutoLoot,
    
    -- Building (Fortnite-style)
    AutoBuild = {
        enabled = false,
        build_type = "wall",  -- wall, ramp, floor, roof
        rapid_build = true,
        function = function(self)
            while self.enabled do
                local direction = mcp.player.get_look_direction()
                mcp.building.place(self.build_type, direction)
                if self.rapid_build then
                    coroutine.wait(0.05)
                else
                    coroutine.wait()
                end
            end
        end
    },
    
    -- Zone
    ZoneESP = {
        enabled = false,
        show_timer = true,
        function = function(self)
            while self.enabled do
                local zone = mcp.game.get_zone()
                mcp.graphics.draw_zone(zone)
                if self.show_timer then
                    mcp.graphics.draw_text(zone.timer)
                end
                coroutine.wait()
            end
        end
    }
}
```

#### Racing Games (Forza, Need for Speed, Mario Kart)
```lua
local RacingCheats = {
    -- Speed
    SpeedHack = {
        enabled = false,
        max_multiplier = 1.8,  -- Keep under physics-break threshold
        function = function(self)
            while self.enabled do
                local vehicle = mcp.player.get_vehicle()
                if vehicle then
                    vehicle:set_speed(vehicle:get_speed() * self.max_multiplier)
                end
                coroutine.wait(0.05)
            end
        end
    },
    
    NitroBoost = {
        enabled = false,
        infinite = true,
        function = function(self)
            while self.enabled do
                local vehicle = mcp.player.get_vehicle()
                if vehicle and self.infinite then
                    vehicle:set_nitro(100)
                end
                coroutine.wait(0.1)
            end
        end
    },
    
    -- Handling
    NoCrash = {
        enabled = false,
        function = function(self)
            while self.enabled do
                mcp.player.set_invulnerable(true)
                mcp.player.set_collision(false)  -- Ghost through cars/walls
                coroutine.wait()
            end
        end
    },
    
    PerfectHandling = {
        enabled = false,
        grip_multiplier = 2.0,
        function = function(self)
            while self.enabled do
                local vehicle = mcp.player.get_vehicle()
                if vehicle then
                    vehicle:set_grip(vehicle:get_grip() * self.grip_multiplier)
                end
                coroutine.wait()
            end
        end
    },
    
    -- Time trials
    FreezeTimer = {
        enabled = false,
        function = function(self)
            while self.enabled do
                mcp.game.set_race_time(0)
                coroutine.wait(0.5)
            end
        end
    }
}
```

#### Sports Games (FIFA, NBA 2K, Madden)
```lua
local SportsCheats = {
    -- Stamina/performance
    InfiniteStamina = {
        enabled = false,
        function = function(self)
            while self.enabled do
                local players = mcp.team.get_own_players()
                for _, p in pairs(players) do
                    p:set_stamina(100)
                end
                coroutine.wait(0.5)
            end
        end
    },
    
    PerfectShot = {
        enabled = false,
        function = function(self)
            while self.enabled do
                mcp.game.set_shot_accuracy(100)
                mcp.game.set_shot_power(100)
                coroutine.wait()
            end
        end
    },
    
    -- Economy (career modes)
    UnlimitedBudget = {
        enabled = false,
        function = function(self)
            while self.enabled do
                mcp.team.set_budget(999999999)
                coroutine.wait(5)
            end
        end
    },
    
    MaxStats = {
        enabled = false,
        function = function(self)
            local players = mcp.team.get_own_players()
            for _, p in pairs(players) do
                for stat, _ in pairs(p:get_all_stats()) do
                    p:set_stat(stat, 99)
                end
            end
        end
    }
}
```

#### Survival Horror (Resident Evil, Dead Space, Lethal Company)
```lua
local HorrorCheats = {
    -- Survival
    InfiniteHealth = {
        enabled = false,
        function = function(self)
            while self.enabled do
                mcp.player.get_local():set_health(mcp.player.get_local():max_health())
                coroutine.wait(0.1)
            end
        end
    },
    
    InfiniteAmmo = {
        enabled = false,
        no_reload = true,
        function = function(self)
            while self.enabled do
                local weapon = mcp.player.get_local():get_weapon()
                if weapon then
                    weapon:set_ammo(weapon:max_ammo())
                    if self.no_reload then weapon:set_needs_reload(false) end
                end
                coroutine.wait(0.1)
            end
        end
    },
    
    -- Atmosphere breakers
    FullBright = {
        enabled = false,
        function = function(self)
            while self.enabled do
                mcp.graphics.set_ambient_light(1.0)  -- Remove darkness
                mcp.graphics.disable_fog(true)
                coroutine.wait()
            end
        end
    },
    
    MonsterESP = {
        enabled = false,
        show_through_walls = true,
        function = function(self)
            while self.enabled do
                local monsters = mcp.scan({type = "hostile_npcs"})
                for _, m in pairs(monsters) do
                    local pos = mcp.camera.get():world_to_screen(m:get_position())
                    if pos then
                        mcp.graphics.draw_text(pos.x, pos.y, "[THREAT]", {r=255, g=0, b=0})
                        -- Distance warning
                        local dist = mcp.math.distance(m:get_position(), mcp.player.get_local():get_position())
                        mcp.graphics.draw_text(pos.x, pos.y + 12, string.format("%dm", dist), {r=255, g=255, b=0})
                    end
                end
                coroutine.wait()
            end
        end
    },
    
    InfiniteInventory = {
        enabled = false,
        function = function(self)
            mcp.player.get_local():set_weight_limit(math.huge)  -- Carry everything
        end
    }
}
```

#### Sandbox/Survival Games (Minecraft, Rust, Terraria, ARK)
```lua
local SandboxCheats = {
    -- Movement
    FlyMode = {
        enabled = false,
        speed = 2.0,
        function = function(self)
            while self.enabled do
                local player = mcp.player.get_local()
                player:set_gravity(0)
                player:set_no_clip(true)
                
                -- Fly with directional input
                local move = mcp.input.get_move_vector()
                local look = mcp.camera.get():get_forward_vector()
                player:set_velocity(look * self.speed * move.forward)
                coroutine.wait()
            end
        end
    },
    
    -- Resources
    InfiniteResources = {
        enabled = false,
        function = function(self)
            while self.enabled do
                local inv = mcp.player.get_local():get_inventory()
                for slot, item in pairs(inv) do
                    item:set_count(999)
                end
                coroutine.wait(1)
            end
        end
    },
    
    InstantBreak = {
        enabled = false,
        function = function(self)
            mcp.game.set_break_speed(math.huge)  -- 1-tap all blocks/resources
        end
    },
    
    XRay = {
        enabled = false,
        function = function(self)
            while self.enabled do
                -- Make terrain semi-transparent, highlight resources
                mcp.graphics.set_terrain_opacity(0.3)
                local resources = mcp.scan({type = "resource_nodes"})
                for _, node in pairs(resources) do
                    local pos = mcp.camera.get():world_to_screen(node:get_position())
                    if pos then
                        mcp.graphics.draw_text(pos.x, pos.y, node.name, {r=0, g=255, b=255})
                    end
                end
                coroutine.wait()
            end
        end
    },
    
    GodMode = {
        enabled = false,
        function = function(self)
            while self.enabled do
                local p = mcp.player.get_local()
                p:set_health(p:max_health())
                p:set_hunger(100)
                p:set_thirst(100)
                p:set_temperature(100)
                coroutine.wait(0.5)
            end
        end
    }
}
```

#### MOBA Games (League of Legends, Dota 2)
```lua
local MOBACheats = {
    -- Vision
    FogOfWar = {
        enabled = false,
        function = function(self)
            while self.enabled do
                mcp.game.reveal_map()
                coroutine.wait()
            end
        end
    },
    
    -- Farming
    AutoLastHit = {
        enabled = false,
        minions_only = true,
        function = function(self)
            while self.enabled do
                local minions = mcp.scan({type = "minions", low_health = true})
                for _, minion in pairs(minions) do
                    if minion:health_percent() < 20 then
                        mcp.player.attack(minion)
                    end
                end
                coroutine.wait(0.1)
            end
        end
    },
    
    -- Prediction
    SkillShotPrediction = {
        enabled = false,
        draw_trajectories = true,
        function = function(self)
            while self.enabled do
                local enemies = mcp.scan({type = "enemies"})
                for _, enemy in pairs(enemies) do
                    local prediction = mcp.game.predict_movement(enemy)
                    if self.draw_trajectories then
                        mcp.graphics.draw_line(enemy.position, prediction)
                    end
                end
                coroutine.wait()
            end
        end
    }
}
```

## Humanization (Behavioral Anti-Detection)

Modern anti-cheats (VACnet, Vanguard, EAC behavior detection) analyze *how* you play, not just *what's in memory*. These patterns make cheats statistically look human:

### Human-like Aim
```lua
local HumanAim = {
    -- Never aim in a perfectly straight line
    bezier_aim = function(self, start_pos, end_pos, duration)
        -- Use a curved path instead of linear interpolation
        local control_point = {
            x = (start_pos.x + end_pos.x) / 2 + math.random(-80, 80),
            y = (start_pos.y + end_pos.y) / 2 + math.random(-60, 60)
        }
        
        local steps = math.random(8, 15)  -- Vary number of movements
        for i = 1, steps do
            local t = i / steps
            -- Quadratic bezier interpolation
            local pos = {
                x = (1-t)^2 * start_pos.x + 2*(1-t)*t * control_point.x + t^2 * end_pos.x,
                y = (1-t)^2 * start_pos.y + 2*(1-t)*t * control_point.y + t^2 * end_pos.y
            }
            mcp.input.move_mouse(pos)
            coroutine.wait(duration / steps + math.random() * 0.004)
        end
    end,
    
    -- Miss occasionally (never aim at the exact same pixel)
    apply_error = function(self, target_pos)
        -- Gaussian-ish error: mostly small, occasionally larger
        local error_dist = math.abs(math.random() - math.random()) * 12  -- avg ~4px
        local error_angle = math.random() * math.pi * 2
        return {
            x = target_pos.x + math.cos(error_angle) * error_dist,
            y = target_pos.y + math.sin(error_angle) * error_dist
        }
    end,
    
    -- Human reaction: never react in under ~120ms
    reaction_delay = function(self)
        return 0.12 + math.random() * 0.08  -- 120-200ms
    end
}
```

### Human-like Clicks & Timing
```lua
local HumanTiming = {
    -- Random click delay instead of fixed intervals
    click = function(self)
        coroutine.wait(0.08 + math.random() * 0.12)  -- 80-200ms
        mcp.input.click("left")
    end,
    
    -- Occasionally pause (humans get distracted)
    micro_pause = function(self)
        if math.random() < 0.03 then  -- 3% chance per iteration
            coroutine.wait(0.5 + math.random() * 2.0)
        end
    end,
    
    -- Variable loop timing (never perfectly periodic)
    loop_wait = function(self, base)
        coroutine.wait(base + math.random() * base * 0.5)
    end,
    
    -- Session pacing: don't farm 8 hours straight at perfect efficiency
    fatigue = function(self)
        -- Reduce performance slightly over long sessions
        local uptime = mcp.game.get_uptime()
        if uptime > 3600 then  -- After 1 hour
            return 0.85 + math.random() * 0.1  -- 85-95% effectiveness
        end
        return 1.0
    end
}
```

### Detection Risk Levels
```
RISK LEVELS (online games):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🟢 LOW    - ESP/radar (read-only, no input)
🟢 LOW    - Triggerbot with 150ms+ reaction
🟡 MEDIUM - Aimbot with bezier curves + error
🟡 MEDIUM - Speed hack under 1.3x
🟠 HIGH   - Fixed-interval auto-click
🟠 HIGH   - Linear/perfect aimbot
🔴 CRITICAL - Instant teleport
🔴 CRITICAL - Packet flooding
🔴 CRITICAL - Stat values beyond server limits
```

## Undetection Techniques

### Anti-Cheat Bypass Guide

#### Easy Anti-Cheat (EAC)
```lua
local EACBypass = {
    -- Common techniques for EAC
    techniques = {
        "delayed_injection",
        "driver_bypass",
        "hook_hiding",
        "memory_encryption"
    },
    
    apply = function(self)
        -- Delay injection until EAC initializes
        coroutine.wait(5)
        
        -- Hide from EAC scan
        mcp.memory.hide_regions(cheat_regions)
        
        -- Use signed driver if available
        if mcp.driver.has_signed() then
            mcp.driver.load("eac_bypass.sys")
        end
    end
}
```

#### BattlEye
```lua
local BattEyeBypass = {
    apply = function(self)
        -- BattlEye runs in kernel mode
        -- Need kernel driver for full bypass
        
        -- 1. Wait for BE initialization
        coroutine.wait(10)
        
        -- 2. Load kernel driver
        mcp.driver.load("beye_bypass.sys")
        
        -- 3. Hide DLL injection
        mcp.process.hide_injection()
        
        -- 4. Use SSDT hooks instead of IAT
        mcp.hooks.use_ssdt()
    end
}
```

#### VAC / VAC2 (Valve)
```lua
local VACBypass = {
    apply = function(self)
        -- VAC is signature-based with delayed bans
        
        -- 1. Obfuscate cheat signatures
        StringEncrypt:key = math.random(0xFFFFFFFF)
        
        -- 2. Delay execution until after VAC scan
        coroutine.wait(30)
        
        -- 3. Use external cheat if possible
        if mcp.process.can_external() then
            mcp.cheat.run_external()
        end
    end
}
```

#### Custom Anti-Cheat
```lua
local function bypass_custom_anticheat(name)
    -- Detect custom anti-cheat type
    local ac_info = mcp.anticheat.get_info(name)
    
    if ac_info.type == "memory_scan" then
        -- Encrypt cheat memory
        mcp.memory.encrypt_regions()
    elseif ac_info.type == "behavior" then
        -- Add human-like randomness
        apply_human_patterns()
    elseif ac_info.type == "signature" then
        -- Randomize code patterns
        mcp.memory.randomize_patterns()
    end
end
```

### Hook Obfuscation (Universal)
```lua
local function obfuscate_hooks()
    -- Platform-agnostic hook obfuscation
    local original_funcs = mcp.hooks.get_originals()
    
    -- Randomize hook names
    local hook_names = {
        "_L", "_X", "_E", "_F", "_G",
        string.char(math.random(65, 90)),
        string.char(math.random(65, 90))
    }
    
    local random_hook = hook_names[math.random(1, #hook_names)]
    
    mcp.hooks.register(random_hook, function(...)
        -- Hook logic here
        return original_funcs.loadstring(...)
    end)
end
```

### Delayed Execution (Universal)
```lua
local function delayed_exec(script, delay)
    coroutine.spawn(function()
        coroutine.wait(delay)
        pcall(function()
            script()
        end)
    end)
end

-- Use varying delays
for i = 1, 10 do
    delayed_exec(function()
        -- Part of script
    end, math.random(1, 5))
end
```

### String Encryption (Universal)
```lua
local StringEncrypt = {
    key = 0x12345678,
    
    encrypt = function(self, str)
        local result = {}
        for i = 1, #str do
            table.insert(result, string.char(string.byte(str, i) ~ ((self.key * i) % 256)))
        end
        return table.concat(result)
    end,
    
    decrypt = function(self, str)
        local result = {}
        for i = 1, #str do
            table.insert(result, string.char(string.byte(str, i) ~ ((self.key * i) % 256)))
        end
        return table.concat(result)
    end
}

-- Usage
local encrypted = StringEncrypt.encrypt("target_function")
local decrypted = StringEncrypt.decrypt(encrypted)
```

### Flow Obfuscation (Universal)
```lua
local function obfuscate_flow()
    local function_a = function() end
    local function_b = function() end
    local function_c = function() end
    
    local functions = {function_a, function_b, function_c}
    local order = {3, 1, 2} -- Randomized order
    
    for i, idx in pairs(order) do
        pcall(functions[idx])
        coroutine.wait(math.random() * 0.1)
    end
end
```

### Memory Obfuscation (Universal)
```lua
local function obfuscate_memory()
    -- Hide cheat in memory
    mcp.memory.hide({
        regions = cheat_memory_regions,
        method = "nop"  -- nop, encrypt, compress
    })
    
    -- Randomize memory patterns
    mcp.memory.randomize_patterns()
end
```

### Anti-Tamper (Universal)
```lua
local AntiTamper = {
    enabled = true,
    
    check = function(self)
        -- Check for common anti-cheat signatures
        local signatures = mcp.memory.scan_anticheat()
        
        for _, sig in pairs(signatures) do
            if sig.detected then
                self.handle_detection(sig)
            end
        end
    end,
    
    handle_detection = function(self, signature)
        -- Bypass or hide
        mcp.memory.patch(signature.address, signature.bypass)
    end
}
```

## Configuration System

### Save/Load User Settings
```lua
local ConfigSystem = {
    path = "cheat_config.json",
    defaults = {
        menu_key = "F11",
        esp = {enabled = true, boxes = true, names = true, health = true, distance = false},
        aimbot = {enabled = false, fov = 90, smooth = 12, bone = "head", auto_fire = false},
        triggerbot = {enabled = false, reaction = 0.18},
        speed = {enabled = false, multiplier = 1.5},
        theme = "dark"
    },
    
    config = nil,
    
    load = function(self)
        -- Load user config or fall back to defaults
        if mcp.file.exists(self.path) then
            local ok, data = pcall(function()
                return mcp.json.parse(mcp.file.read(self.path))
            end)
            if ok then
                -- Merge with defaults (deep) so missing keys are filled
                self.config = self:deep_merge(self.defaults, data)
            end
        else
            self.config = self.defaults
        end
        return self.config
    end,
    
    save = function(self)
        mcp.file.write(self.path, mcp.json.stringify(self.config, true))
    end,
    
    deep_merge = function(self, base, override)
        local result = {}
        for k, v in pairs(base) do
            if type(v) == "table" and type(override[k]) == "table" then
                result[k] = self:deep_merge(v, override[k])
            elseif override[k] ~= nil then
                result[k] = override[k]
            else
                result[k] = v
            end
        end
        -- Include keys only in override
        for k, v in pairs(override) do
            if result[k] == nil then result[k] = v end
        end
        return result
    end
}

-- Auto-save on menu close or feature change
local function on_config_changed()
    ConfigSystem:save()
end
```

### Per-Game Config Profiles
```lua
-- Store separate configs for each game
local Profiles = {
    active_game = nil,
    
    get_path = function(self, game_name)
        return string.format("configs/%s.json", game_name)
    end,
    
    load_for_game = function(self, game_name)
        self.active_game = game_name
        ConfigSystem.path = self:get_path(game_name)
        return ConfigSystem:load()
    end,
    
    -- Export/import for sharing configs
    export = function(self)
        return mcp.base64.encode(mcp.json.stringify(ConfigSystem.config))
    end,
    
    import = function(self, encoded)
        ConfigSystem.config = mcp.json.parse(mcp.base64.decode(encoded))
        ConfigSystem:save()
    end
}
```

## UI System (Universal)

### Key System
```lua
local KeySystem = {
    key = "YOUR-KEY-HERE",
    encrypted_key = nil,
    online_validation = true,
    
    validate = function(self)
        if self.online_validation then
            return mcp.network.validate_key(self.encrypted_key)
        end
        
        local decrypted = self.decrypt_key(self.encrypted_key)
        return decrypted == self.key
    end,
    
    encrypt_key = function(self, key)
        -- Encrypt key for storage
        return StringEncrypt.encrypt(key)
    end,
    
    decrypt_key = function(self, encrypted)
        -- Decrypt key for validation
        return StringEncrypt.decrypt(encrypted)
    end
}
```

### Settings Menu (Universal)
```lua
local SettingsMenu = {
    open = false,
    features = {},
    bindings = {},
    
    create = function(self)
        -- Platform-agnostic UI creation
        local ui = mcp.ui.create_window({
            title = "Cheat Menu",
            size = {x = 400, y = 500},
            position = {x = 100, y = 100},
            resizable = true,
            draggable = true
        })
        
        -- Add feature toggles
        for name, feature in pairs(self.features) do
            self.create_feature_toggle(ui, name, feature)
        end
        
        -- Add keybinding section
        self.create_bindings_panel(ui)
    end,
    
    create_feature_toggle = function(self, ui, name, feature)
        local toggle = mcp.ui.create_toggle({
            label = name,
            callback = function(state)
                feature.enabled = state
                if state then
                    coroutine.spawn(feature.function)
                end
            end
        })
        
        ui:add_widget(toggle)
    end,
    
    create_bindings_panel = function(self, ui)
        local panel = mcp.ui.create_panel("Keybindings")
        
        for key, action in pairs(self.bindings) do
            local binding = mcp.ui.create_binding({
                label = action,
                key = key,
                callback = function(new_key)
                    self.bindings[new_key] = action
                end
            })
            
            panel:add_widget(binding)
        end
        
        ui:add_panel(panel)
    end,
    
    toggle = function(self)
        self.open = not self.open
        mcp.ui.set_visible(self.window, self.open)
    end
}
```

### Notification System (Universal)
```lua
local Notifications = {
    position = "top-right",
    duration = 3,
    
    show = function(self, message, type)
        type = type or "info"  -- info, success, warning, error
        
        mcp.ui.notify({
            message = message,
            type = type,
            duration = self.duration
        })
    end,
    
    info = function(self, message)
        self.show(message, "info")
    end,
    
    success = function(self, message)
        self.show(message, "success")
    end,
    
    warning = function(self, message)
        self.show(message, "warning")
    end,
    
    error = function(self, message)
        self.show(message, "error")
    end
}
```

## Quality Checklist

### Code Quality
- [ ] Proper error handling with pcall
- [ ] Clean variable naming
- [ ] Commented sections
- [ ] Modular structure
- [ ] No hardcoded paths

### Feature Quality
- [ ] Configurable options
- [ ] Performance optimized
- [ ] Memory efficient
- [ ] No lag spikes
- [ ] Smooth operation

### Undetection Quality
- [ ] Obfuscated strings
- [ ] Delayed execution
- [ ] Hook randomization
- [ ] No obvious patterns
- [ ] Clean memory footprint

## Testing Workflow

### 1. Unit Testing
```lua
local function test_feature(feature)
    local success, result = pcall(function()
        feature.enabled = true
        task.wait(1)
        feature.enabled = false
    end)
    
    return success, result
end
```

### 2. Integration Testing
```lua
local function test_all_features()
    local results = {}
    
    for name, feature in pairs(features) do
        local success, result = test_feature(feature)
        results[name] = {success = success, result = result}
    end
    
    return results
end
```

### 3. Performance Testing
```lua
local function benchmark(feature)
    local start = tick()
    feature.enabled = true
    task.wait(10)
    feature.enabled = false
    local end_time = tick()
    
    return end_time - start
end
```

## Workflow: Creating a Cheat for Any Game

### Step 1: Game Analysis
```lua
-- 1. Launch and connect to the game
mcp.game.launch("game.exe")
coroutine.wait(3)

-- 2. Detect game properties
local game_info = {
    title = mcp.game.get_title(),
    engine = mcp.platform.detect(),
    is_online = mcp.network.is_active(),
    anti_cheat = detect_anticheat(),
    genre = mcp.game.detect_genre()
}

print("Game:", game_info.title)
print("Engine:", game_info.engine)
print("Online:", game_info.is_online)
print("Anti-Cheat:", table.concat(game_info.anti_cheat, ", "))
print("Genre:", game_info.genre)
```

### Step 2: Memory Exploration
```lua
-- 1. Find player structure
local player_addr = mcp.memory.find_player()
print("Player address:", player_addr)

-- 2. Scan for common offsets
local offsets = {
    health = mcp.memory.scan_from(player_addr, "health", "float"),
    ammo = mcp.memory.scan_from(player_addr, "ammo", "int"),
    money = mcp.memory.scan_from(player_addr, "money", "int"),
    position = mcp.memory.scan_from(player_addr, "position", "vector3")
}

-- 3. Verify offsets work
local player = mcp.player.get_local()
print("Health:", player:get_health())
print("Position:", player:get_position())
```

### Step 3: Feature Selection
```lua
-- Choose features based on game type
local features_to_enable = {}

if game_info.genre == "fps" then
    features_to_enable = {"Aimbot", "ESP", "Triggerbot"}
elseif game_info.genre == "rpg" then
    features_to_enable = {"AutoQuest", "GodMode", "UnlimitedMoney"}
elseif game_info.genre == "mmo" then
    features_to_enable = {"AutoFarm", "ESP", "SpeedHack"}
end

-- Adjust for online games
if game_info.is_online then
    -- Remove risky features for server-authoritative games
    if game_info.server_authority then
        features_to_enable = {"ESP", "Radar", "Triggerbot"}
    end
end
```

### Step 4: Implementation
```lua
-- Load selected features
for _, feature_name in pairs(features_to_enable) do
    local feature = FEATURES[feature_name]
    if feature then
        feature.enabled = false  -- Start disabled
        features[feature_name] = feature
    end
end

-- Create UI
create_menu(features)

-- Apply anti-cheat bypass
if #game_info.anti_cheat > 0 then
    apply_anticheat_bypass(game_info.anti_cheat)
end
```

### Step 5: Testing
```lua
-- Test each feature
for name, feature in pairs(features) do
    print("Testing:", name)
    feature.enabled = true
    coroutine.wait(2)
    feature.enabled = false
    print(name, "tested successfully")
end
```

## Quick Start Template (Universal)

```lua
-- Universal Cheat Script Template
local CheatScript = {
    name = "MyGame Cheat",
    version = "1.0.0",
    author = "YourName",
    platform = "auto",  -- unity, unreal, roblox, custom
    
    features = {
        AutoFarm = AutoFarm,
        AutoClick = AutoClick,
        SpeedHack = SpeedHack,
        ESP = ESP,
        Aimbot = Aimbot,
        Triggerbot = Triggerbot,
        Radar = Radar
    },
    
    config = {
        toggle_key = "F9",
        esp_enabled = true,
        aimbot_enabled = false,
        auto_update = true
    },
    
    init = function(self)
        -- Detect platform
        self.platform = mcp.platform.detect()
        
        -- Initialize all features
        for name, feature in pairs(self.features) do
            feature.enabled = false
            if feature.init then
                feature:init()
            end
        end
        
        -- Setup UI
        SettingsMenu.features = self.features
        SettingsMenu:create()
        
        -- Setup keybindings
        mcp.input.bind(self.config.toggle_key, function()
            SettingsMenu:toggle()
        end)
        
        -- Setup notifications
        Notifications:info(string.format("%s v%s loaded!", self.name, self.version))
    end,
    
    run = function(self)
        self:init()
        
        -- Main loop
        coroutine.spawn(function()
            while true do
                -- Run enabled features
                for name, feature in pairs(self.features) do
                    if feature.enabled and feature.function then
                        coroutine.spawn(feature.function)
                    end
                end
                
                -- Anti-tamper check
                AntiTamper:check()
                
                coroutine.wait(0.1)
            end
        end)
    end
}

-- Run the cheat
CheatScript:run()
```

## Platform-Specific Adapters

### Roblox Adapter
```lua
local RobloxAdapter = {
    init = function(self)
        mcp.platform.set("roblox")
        -- Roblox: no memory scanning needed - full API access via executor
    end,
    
    get_player = function(self)
        return game:GetService("Players").LocalPlayer
    end,
    
    -- Roblox-specific: hook RemoteEvents (network manipulation)
    hook_remotes = function(self)
        for _, obj in pairs(game:GetDescendants()) do
            if obj:IsA("RemoteEvent") or obj:IsA("RemoteFunction") then
                print("Remote found: " .. obj:GetFullName())
            end
        end
    end,
    
    -- Roblox-specific: safe teleport (avoid anti-cheat checks)
    safe_teleport = function(self, cframe, steps)
        steps = steps or 10
        local start = self:get_player().Character.HumanoidRootPart.CFrame
        for i = 1, steps do
            self:get_player().Character.HumanoidRootPart.CFrame = start:Lerp(cframe, i / steps)
            coroutine.wait(0.03)
        end
    end
}
```

### Unity Adapter (IL2CPP / Mono)
```lua
local UnityAdapter = {
    il2cpp = true,  -- true: IL2CPP (newer games), false: Mono (older)
    
    init = function(self)
        mcp.platform.set("unity")
        
        if self.il2cpp then
            -- IL2CPP: C# compiled to native, needs il2cpp dumps
            mcp.memory.load_il2cpp()
            mcp.memory.load_dump("il2cpp_dump.json")  -- From Il2CppDumper
        else
            -- Mono: full .NET runtime, can use reflection
            mcp.memory.load_mono()
            mcp.mono.attach("Assembly-CSharp.dll")
        end
    end,
    
    get_player = function(self)
        if self.il2cpp then
            return mcp.memory.find("PlayerController")
        end
        return mcp.mono.find_class("Assembly-CSharp", "PlayerController")
    end,
    
    -- Unity-specific: find all objects of a class
    find_all = function(self, class_name)
        if self.il2cpp then
            return mcp.memory.il2cpp_find_objects(class_name)
        end
        return mcp.mono.find_objects("UnityEngine.GameObject", class_name)
    end
}
```

### Unreal Engine Adapter (UE4/UE5)
```lua
local UnrealAdapter = {
    init = function(self)
        mcp.platform.set("unreal")
        mcp.memory.load_ue4()
        mcp.memory.load_sdk("ue4_sdk.json")  -- From Dumper-7 / UE4SS
        
        -- Unreal uses GNames/GObjects for reflection
        mcp.unreal.init_gnames()
        mcp.unreal.init_gobjects()
    end,
    
    get_player = function(self)
        return mcp.memory.find("APlayerController")
    end,
    
    -- Unreal-specific: UWorld traversal
    find_actors = function(self, class_name)
        local world = mcp.unreal.get_uworld()
        return mcp.unreal.get_all_actors_of_class(world, class_name)
    end,
    
    -- Unreal-specific: world-to-screen via player controller
    world_to_screen = function(self, world_pos)
        local player = self:get_player()
        return mcp.unreal.project_world_to_screen(player, world_pos)
    end
}
```

### Godot Adapter
```lua
local GodotAdapter = {
    init = function(self)
        mcp.platform.set("godot")
        mcp.memory.load_godot()
    end,
    
    get_player = function(self)
        return mcp.memory.find("Player")  -- GDScript class
    end,
    
    -- Godot-specific: SceneTree traversal
    find_nodes = function(self, group_name)
        return mcp.godot.get_tree():get_nodes_in_group(group_name)
    end
}
```

### Native / Custom Engine Adapter
```lua
local NativeAdapter = {
    -- For Source Engine, id Tech, Snowdrop, RE Engine, etc.
    init = function(self)
        mcp.platform.set("native")
    end,
    
    -- Everything is manual: pattern scan for functions and structures
    find_functions = function(self, patterns)
        local found = {}
        for name, pattern in pairs(patterns) do
            found[name] = mcp.memory.scan({pattern = pattern, module = "game.exe"})
        end
        return found
    end,
    
    -- Example: Source Engine patterns (CreateMove is the golden hook target)
    source_engine = {
        create_move = "55 8B EC 83 EC F4 80 B9 ?? ?? ?? ?? 00",
        paint_traverse = "55 8B EC 56 8B 75",
    },
    
    hook_create_move = function(self, callback)
        local addr = mcp.memory.scan({pattern = self.source_engine.create_move, module = "client.dll"})
        mcp.hooks.install(addr, callback)
    end
}
```

## MCP Commands

```bash
# Analyze game structure
mcp analyze game --depth 5

# Find all interactables/remotes
mcp find interactables --include-internal

# Extract function signatures
mcp extract functions --output json

# Monitor game state
mcp monitor state --interval 100ms

# Scan memory for patterns
mcp memory scan --pattern "?? ?? 38 00 00 00"

# List all entities
mcp scan entities --type all

# Get player info
mcp player get --local

# Camera operations
mcp camera get --position --rotation

# Input simulation
mcp input click --button left --position 500 300
```

## Best Practices

### General Development
1. **Modular Design** - Keep features separate and reusable
2. **Error Handling** - Use pcall for all operations
3. **Performance** - Optimize loops and minimize memory usage
4. **Undetection** - Implement multiple obfuscation layers
5. **Testing** - Test each feature independently
6. **Documentation** - Comment all major sections
7. **Version Control** - Track changes with git

### Online Game Specific
8. **Human-like Behavior** - Add randomness to aim, reaction times
9. **Configurable Limits** - Don't show ESP beyond reasonable distances
10. **Delayed Features** - Start cheats after game load completes
11. **Memory Cleanup** - Remove hooks when features disable

### Offline Game Specific
12. **Save Backups** - Always backup save files before editing
13. **Config Persistence** - Save user preferences between sessions
14. **Hotkey System** - Make features easily toggleable in-game
15. **Trainer Mode** - Support classic trainer-style hotkeys (F1-F12)

### Memory Manipulation
16. **Pattern Scanning** - Use patterns instead of hardcoded addresses
17. **Multi-Game Support** - Store patterns for multiple game versions
18. **Offset Validation** - Verify offsets before writing memory
19. **Thread Safety** - Lock memory regions during reads/writes

### Anti-Cheat Evasion
20. **Driver Signing** - Use signed drivers for kernel-level cheats
21. **External Process** - Run from external process when possible
22. **Code Obfuscation** - Encrypt and obfuscate cheat code
23. **Signature Randomization** - Randomize code patterns each run

## Skill Usage

Load this skill when:
- Starting work on a new game cheat
- Analyzing game mechanics via MCP
- Creating cheats for online or offline games
- Implementing specific cheat features (ESP, aimbot, auto-farm, etc.)
- Building cheat UI systems and menus
- Optimizing for undetection against anti-cheat
- Reverse engineering game mechanics
- Creating trainer-style modifications
- Editing game save files
- Developing genre-specific cheat features

## Quick Reference

### Feature Matrix (All Games)

| Feature | FPS | RPG/MMO | Battle Royale | MOBA | Racing | Sports | Horror | Sandbox | Offline Risk | Online Risk |
|---------|-----|---------|---------------|------|--------|--------|--------|---------|--------------|-------------|
| ESP/Box | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ | None | Low |
| Skeleton | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ | None | Low |
| Radar | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ | None | Low |
| Aimbot | ✅ | ⚠️ | ✅ | ⚠️ | ❌ | ❌ | ✅ | ❌ | None | Medium |
| Triggerbot | ✅ | ⚠️ | ✅ | ⚠️ | ❌ | ❌ | ✅ | ❌ | None | Medium |
| NoRecoil | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ | None | Medium |
| Speed Hack | ⚠️ | ✅ | ⚠️ | ❌ | ✅ | ❌ | ✅ | ✅ | None | High |
| Fly/NoClip | ⚠️ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | None | High |
| God Mode | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | None | High* |
| Infinite Ammo | ✅ | ✅ | ⚠️ | ❌ | ✅ | ❌ | ✅ | ✅ | None | High |
| Auto Farm | ❌ | ✅ | ❌ | ⚠️ | ❌ | ❌ | ❌ | ✅ | None | Medium |
| Auto Loot | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ | None | Medium |
| Auto Quest | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | None | Medium |
| Unlimited Money | ❌ | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | None | High* |
| XRay | ⚠️ | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ | None | Low |
| Packet Mods | ⚠️ | ✅ | ⚠️ | ⚠️ | ❌ | ❌ | ❌ | ❌ | None | Critical |

*God Mode / Unlimited Money in online games: usually server-checked and rejected, may trigger instant flag.

### Memory Manipulation Quick Reference
```
Float (health):     100.0f = 00 00 C8 42
Integer (ammo):     3000 = B8 0B 00 00
Double (money):     999999.0 as double = 00 00 00 00 40 3F 42 00 (verify via scanner)
Vector3 (position): 3 consecutive floats (x, y, z)
String (name):      length-prefixed or null-terminated
Pointer (chains):   8-byte aligned on x64, 4-byte on x32
```

### Common Value Types When Scanning
```
Health:    float, 0.0-1000.0 typical, changes on damage
Ammo:      int or byte, decreases on fire, resets on reload
Money:     int/double, large values, changes on purchase
Stamina:   float, drains on sprint, regens when idle
Position:  3x float, changes constantly when moving
Speed:     float, ~100-600 typical, base movement speed
```

### World-to-Screen Checklist (Per Engine)
```
Unity:   Camera.main.worldToCameraMatrix + projectionMatrix
Unreal:  PlayerCameraManager.GetActorMatrices / project_world_to_screen
Source:  IEngineViewRender + IClientEntityList
Roblox:  Camera:WorldToViewportPoint (built-in!)
Godot:   Camera3D.unproject_position
Custom:  Find view matrix via pattern scan (usually near FOV/aspect)
```

### Common Hotkey Assignments
```
F1 - God Mode
F2 - Unlimited Ammo
F3 - Speed Hack
F4 - Infinite Health
F5 - Unlimited Money
F6 - Fly Mode
F7 - No Clip
F8 - Aimbot
F9 - ESP
F10 - Triggerbot
F11 - Menu Toggle
F12 - Unload (safely remove all hooks)
```

### Anti-Cheat Detection Order (Stealthiest First)
```
1. Visual ESP        (read-only memory, safest)
2. Radar             (read-only, safe)
3. Triggerbot        (input, moderate)
4. Aimbot smoothed    (input, moderate)
5. Speed Hack        (write, risky)
6. Memory mods       (write, varies)
7. DLL injection     (presence, risky)
8. Packet editing    (network, critical)
9. Kernel hooks      (deep, depends on driver)
```

### Safety Checklist Before Release
- [ ] All memory writes validated before writing (crash prevention)
- [ ] Feature toggles properly restore original values on disable
- [ ] Unload function cleanly removes all hooks and drawings
- [ ] pcall wraps every risky operation
- [ ] Config saved/loaded without corrupting on missing keys
- [ ] Works after game restart (no stale addresses)
- [ ] Graceful failure when offsets invalid (not instant crash)
- [ ] No performance degradation (frame drops = detectable)
