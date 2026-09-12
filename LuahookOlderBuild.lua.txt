--[[
═══════════════════════════════════════════════════════════════════════════════
  LuaHook v1 beta
═══════════════════════════════════════════════════════════════════════════════
]]

if _G["\76\72"] then
    pcall(function() _G["\76\72"]:Unload() end)
    _G["\76\72"] = nil
    task.wait(0.15)
end

-- ── AC BYPASS  Layer 1 (RESTORED — the load-bearing bypass) — INSTALLED FIRST ───────────────
-- Installed at the very TOP, BEFORE waitForGameReady()/loadGameModule() (each up to ~15s). The
-- MiscellaneousController weak-GC probe is a background loop; if it completes one cycle under join-time
-- GC pressure before our hook is live, that cycle uses the REAL setmetatable and can report. This block
-- depends ONLY on executor globals (getgenv/hookfunction/newcclosure/getrenv) + game/debug/string — all
-- available instantly — so it goes first to win the race on the single most important bypass.
--
-- Decompiled-client RE (workflow wf_9d8face0, 2026-07-01) + operator ground truth: the ORIGINAL LuaHook
-- (which ran this exact setmetatable hook) was UNDETECTED for a long time; REMOVING it → next-day RedFlag
-- bans. Loading it first also makes the paid cheat abort with "failed to run bypasses" → independent proof
-- it contends for the exact same surface (so this IS the right one).
--
-- Target: the MiscellaneousController weak-table GC integrity probe (READABLE code, ~L2520-2532 — NOT
-- inside the encrypted VM, so debug.traceback DOES surface the "MiscellaneousController" name):
--     local v553 = setmetatable({ {}, 1, "String", mouse }, { __mode = "kv" })
--     repeat task.wait() until not v553[1]          -- waits for the weak {} to be GC-collected
--     task.wait(0.2)
--     return #v553 ~= 3 or rawlen(v553) ~= 3        -- flags executors with abnormal weak-GC → shared.r report
-- When the caller is MiscellaneousController and the mt is weak, we hand back a NON-weak {1,2,3}:
-- `until not v553[1]` becomes `until not 1` → never satisfied → the probe HANGS harmlessly forever in its
-- background task.spawn and never reaches the report line. newcclosure keeps setmetatable a genuine
-- C-closure (iscclosure stays true) and EVERY other call passes straight through unchanged. `Config` isn't
-- in scope this early — gate ONLY on executor capability (unconditional by design; the proven baseline).
do
    if getgenv and getgenv().__LH_SetmtBP ~= game and hookfunction and newcclosure and getrenv then
        getgenv().__LH_SetmtBP = game
        local ok = pcall(function()
            local oldsetmt
            oldsetmt = hookfunction(getrenv().setmetatable, newcclosure(function(t, mt)
                if mt and typeof(mt) == "table" and rawget(mt, "__mode") == "kv" then
                    local okt, trace = pcall(debug.traceback)
                    if okt and trace and string.find(trace, "MiscellaneousController", 1, true) then
                        return oldsetmt({ 1, 2, 3 }, {})   -- [1]=1 (never nil) ⇒ probe hangs, never reports
                    end
                end
                return oldsetmt(t, mt)
            end))
        end)
        if not ok then getgenv().__LH_SetmtBP = nil end   -- crash-safe: leave the surface clean on failure
    end
end

-- FIX #4: cloneref identity hygiene. Wraps SERVICE handles ONLY (never lp, never game,
-- never Workspace — those are used in == identity comparisons / sentinels and MUST stay raw).
-- OFF by default; opt-in via getgenv().__LH_CloneRef = true (mirrors the 00b __LH_FireServerBackstop
-- pattern). cloneref proxies the SAME instance with a different Lua identity, so property/child/method
-- access is unchanged; we only clone handles NEVER compared with ==/~= or used as a table key. Defined
-- AFTER the setmetatable bypass (that block must stay the first-executed code) and guarded so a missing-
-- cloneref executor falls back to identity (no hard-error on load — see RISK 4.2). EXCLUSION SET (must
-- stay raw): game, workspace/Workspace, Camera, lp/LocalPlayer, shared, getrenv() tables, and anything
-- the 00b scrub / Layer-1 setmetatable hook touches.
local cloneref = clonereference or cloneref or function(x) return x end
local _CR = (getgenv and getgenv().__LH_CloneRef == true)
local function cr(x) if _CR and x then return cloneref(x) else return x end end

local Players           = cr(game:GetService("Players"))
local RunService        = cr(game:GetService("RunService"))
local UserInputService  = cr(game:GetService("UserInputService"))
local VirtualInputMgr   = cr(game:GetService("VirtualInputManager"))
local ReplicatedStorage  = cr(game:GetService("ReplicatedStorage"))
local CollectionService  = cr(game:GetService("CollectionService"))
local Lighting           = cr(game:GetService("Lighting"))
local Debris             = cr(game:GetService("Debris"))
local Workspace          = workspace                -- RAW: compared at 10b (a==/~=Workspace ancestry walk)
local Camera             = workspace.CurrentCamera   -- RAW
workspace:GetPropertyChangedSignal("CurrentCamera"):Connect(function()
    Camera = workspace.CurrentCamera
end)
local lp                 = Players.LocalPlayer

-- ── AC BYPASS · Layers 2+3 — TELEMETRY SUPPRESSION ─────────────────────────
-- Restored verbatim from the Jun-10 OldLuahook baseline. That build ran these two layers with NO
-- shared.r scrub and is live-confirmed UNDETECTED (2026-07-26) while running far more egregious rage
-- than we ship; the builds that dropped them and swapped in the scrub-at-emitter-input ban within hours.
-- Same lesson as Layer 1 above: what the original ran is load-bearing. Do NOT "improve" either layer.
-- L2 silences the AnalyticsPipeline reporters (GC-scanned controller closures + the OnClientEvent
-- handlers). L3 blocks the two telemetry remotes outright; every other namecall passes straight through.
-- Both are self-guarded per game and fully pcall'd — a missing executor global degrades to a no-op.
do
    task.spawn(function()
        if shared._LH_GC == game then return end
        shared._LH_GC = game
        if not (getgc and hookfunction and newcclosure) then return end
        task.wait(0.1)
        local AC_ID = "AnalyticsPipelineController"
        local noop  = newcclosure(function() return nil end)
        local processed = 0
        local gcList = getgc(true)   -- true = objects only; a full memory walk freezes the thread
        for i = 1, #gcList do
            local v = gcList[i]
            if typeof(v) == "function" then
                local ok, src = pcall(debug.info, v, "s")
                if ok and src and string.find(src, AC_ID, 1, true) then
                    pcall(hookfunction, v, noop)
                end
            end
            processed = processed + 1
            if processed % 150 == 0 then task.wait() end
        end
        pcall(function()
            local remote = ReplicatedStorage.Remotes.AnalyticsPipeline.RemoteEvent
            for _, c in ipairs(getconnections(remote.OnClientEvent) or {}) do
                local f = c.Function
                if f then pcall(hookfunction, f, noop) end
            end
        end)
    end)

    pcall(function()
        if shared._LH_NC == game then return end
        if not (hookmetamethod and getnamecallmethod) then return end
        shared._LH_NC = game
        local blockedRemotes = {
            VerifyClientEntity = true,
            AnalyticsPipeline = true,
        }
        local oldNamecall
        oldNamecall = hookmetamethod(game, "__namecall", function(self, ...)
            local method = getnamecallmethod()
            if typeof(self) == "Instance" and (method == "FireServer" or method == "InvokeServer") then
                local ok, name = pcall(function() return self.Name end)
                if ok and blockedRemotes[name] then
                    return
                end
            end
            return oldNamecall(self, ...)
        end)
    end)
end

local _safePlayersCache = nil
local function getSafePlayers()
    if _safePlayersCache then return _safePlayersCache end
    local list = {}
    for _, p in ipairs(Players:GetChildren()) do
        if p:IsA("Player") then list[#list+1] = p end
    end
    _safePlayersCache = list   -- read-only; rebuilt only when the player list changes
    return list
end
Players.PlayerAdded:Connect(function() _safePlayersCache = nil end)
Players.PlayerRemoving:Connect(function() _safePlayersCache = nil end)

-- ═══════════════════════════════════════════════════════════════════════
-- LOADING GUARD — prevents crashes when joining a new game
-- ═══════════════════════════════════════════════════════════════════════
local function waitForGameReady()
    local Players = game:GetService("Players")
    local lp = Players.LocalPlayer
    repeat task.wait() until game:IsLoaded()
    repeat task.wait() until lp and lp.Character
    local ps = lp:FindFirstChild("PlayerScripts")
    if not ps then
        repeat task.wait() until lp:FindFirstChild("PlayerScripts")
    end
    return true
end
waitForGameReady()

-- ═══════════════════════════════════════════════════════════════════════
-- SAFE REQUIRE — timeout‑based, no more nil crashes
-- ═══════════════════════════════════════════════════════════════════════
local function safeRequire(path, timeout)
    if not path then return nil end
    timeout = timeout or 5
    local start = tick()
    while tick() - start < timeout do
        local ok, result = pcall(require, path)
        if ok then return result end
        task.wait(0.2)
    end
    return nil
end

-- Resolve a module path WITHOUT throwing if a segment hasn't replicated yet (rare slow joins): WaitForChild
-- per segment with a timeout, so we never error at the top level and never index a nil child.
local function waitModule(root, names, timeout)
    timeout = timeout or 10
    local obj = root
    for _, n in ipairs(names) do
        if not obj then return nil end
        local ok, child = pcall(function() return obj:WaitForChild(n, timeout) end)
        obj = ok and child or nil
    end
    return obj
end

-- ═══════════════════════════════════════════════════════════════════════
-- MOBILE SUPPORT DETECTION & INPUT ABSTRACTION
-- ═══════════════════════════════════════════════════════════════════════
local isMobile = UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled

local function isMouseButtonDown()
    if isMobile then
        return true   -- on mobile, always allow firing (can be toggled via button)
    else
        return UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton1)
    end
end

local function isInputActive(key)
    if key == "MB1" then return isMouseButtonDown() end
    if key == "MB2" then
        if isMobile then return true end
        return UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton2)
    end
    if key == "Always" then return true end
    local kc = Enum.KeyCode[key]
    if kc and not isMobile then
        return UserInputService:IsKeyDown(kc)
    end
    -- fallback for mobile – assume active if not a keyboard key
    return false
end


-- ═══════════════════════════════════════════════════════════════════════
-- RIVALS MODULES — loaded once, no poller needed
-- ═══════════════════════════════════════════════════════════════════════
-- THE RARE "whole game breaks / no guns" crash: if WE require a game ModuleScript before the game does,
-- it can run before its deps are ready, ERROR, and Roblox CACHES that error — so the game's own require
-- re-throws forever and the weapon system never initializes. Fix: if the executor exposes getloadedmodules,
-- WAIT until the GAME has already loaded the module, so our require only ever returns the game's cached
-- (good) result and can never be the one that runs (and bricks) it. No-op if the API is unavailable.
local function awaitGameLoaded(moduleInst, timeout)
    if not moduleInst or type(getloadedmodules) ~= "function" then return end
    local deadline = tick() + (timeout or 15)
    repeat
        local ok, loaded = pcall(getloadedmodules)
        if ok and type(loaded) == "table" then
            for _, m in ipairs(loaded) do
                if m == moduleInst then return end
            end
        end
        task.wait(0.1)
    until tick() >= deadline
end
local function loadGameModule(root, names)
    local inst = waitModule(root, names)
    awaitGameLoaded(inst, 15)
    return safeRequire(inst)
end

local Rivals = { Ready = false }
Rivals.Util    = loadGameModule(ReplicatedStorage, {"Modules","Utility"})
Rivals.Fighter = loadGameModule(lp.PlayerScripts, {"Controllers","FighterController"})
Rivals.Gun     = loadGameModule(lp.PlayerScripts, {"Modules","ItemTypes","Gun"})
Rivals.Enums   = loadGameModule(ReplicatedStorage, {"Modules","EnumLibrary"})
Rivals.Ready   = (Rivals.Util ~= nil and Rivals.Gun ~= nil)

-- Forward-declared here so the CharacterAdded closure below captures it as an upvalue. The body is
-- assigned later in 08_gunhook.lua (bundled after this file); without this hoist that reference would
-- bind to a nil global and the respawn re-hook would silently never run.
local hookGunModule

-- When the local character respawns (new round), ensure hooks are re‑applied
lp.CharacterAdded:Connect(function()
    task.wait(0.5)
    if Rivals.Ready and hookGunModule then
        hookGunModule()
    end
end)

local function getEquippedItem()
    if not Rivals.Ready then return nil end
    local f = Rivals.Fighter and Rivals.Fighter.LocalFighter
    return f and f.EquippedItem or nil
end

-- ── AC BYPASS  Layer 1 — installed at the TOP of this file (right after the unload guard, before the
-- game-load waits) to win the race against the MiscellaneousController weak-GC probe. See that block. ──

-- ── CONFIG ─────────────────────────────────────────────────────────────────
local Config = {
    -- App settings
    GUIToggleKey = "RightShift",	
    -- Silent Aim
    SilentAim = false, SilentAimVisCheck = false, SilentAimJitter = true,
    SilentAimTargetPart = "Head", SilentAimFOV = 250, AvoidDeflect = true,
    SilentAimStickiness = 0.05,      -- target-switch bias (0..0.5); was hardcoded 0.05 in the gunhook, now tunable
    SilentAimMultipoint = false,     -- probe a ring of candidate points around the part for clear LOS
    SilentAimMultipointCount = 5,    -- ring sample points probed per part when multipoint is on
    SilentAimTorsoFallback = false,  -- if head has no clear LOS, fall back to torso hitbox parts

    -- Aimbot
    Aimbot = false, AimbotVisCheck = true, AimbotKey = "MB2",  -- bug#8: hold-to-aim default (MB2/ADS) so hardlock engages only while aiming, not glued-always. 'Always' still selectable.
    AimbotSmooth = 0, AimbotFOV = 120, AimbotTargetPart = "Head",
    AimbotStickiness = 0.15, AimbotShotOverride = false,
    AimbotShowFOV = false, AimbotShowLock = false,
    AimbotDeadzone = 0, AimbotSpeedCap = 0, AimbotEasing = "Linear",
    AimbotHardLock = true, AimbotSensMultiplier = 1.0,
    AimbotPrediction = false,   -- OFF = lock directly on target (hitscan). ON = lead the target (projectiles).
    AimbotFOVMode = "Pixels",   -- "Pixels" (screen radius) | "Degrees" (angular cone via camera FieldOfView)
    AimbotFOVDegrees = 8,       -- FOV cone in degrees, used when AimbotFOVMode == "Degrees"
    AimbotAccelTime = 0.12,     -- EaseInOut acceleration ramp-up time (s) after acquiring a target
    AimbotReactionMs = 0,       -- humanization: delay (ms) before tracking a freshly acquired target (0 = instant)
    AimbotNoise = 0,            -- humanization: sub-pixel per-frame aim noise amplitude (px, dt-scaled; 0 = rigid)
    AimbotOvershoot = 0,        -- humanization: overshoot-and-settle amount (0..1) during approach (0 = rigid)
    AimbotSwitchThreshold = 0,  -- hysteresis: a competitor must be this many px closer before the lock switches (0 = any closer)
    AimbotAutoFire = false,     -- auto-fire/trigger: click when locked within AimbotAutoFireFOV
    AimbotAutoFireFOV = 8,      -- pixel threshold within which auto-fire triggers
    AimbotAutoFireDelay = 0,    -- minimum ms between auto-fire clicks

    -- Shared targeting
    MaxDistance = 1200, TeamCheck = true,
    PredictiveLead = true, LeadCap = 15, ServerProcessingMs = 30,
    ProjectileLead = false,     -- add projectile travel-time lead (distance/speed) for non-hitscan weapons (off = hitscan)
    ProjectileSpeed = 300,      -- projectile speed (studs/s) used for travel-time lead when ProjectileLead is on

    -- ── RAGE (POLAR — the live-proven Kicia-beater, 7-0 / 5-0) ──────────────────
    -- ONE point-blank behavior: every Heartbeat, teleport the replicated body onto the enemy's live
    -- HitboxHead (LOS + <=400 range + eye-body consistency FOR FREE) and FIRE a burst of taps; when the
    -- enemy voids their head, surface at real + fire a proactive first-strike burst. Deep-void hide ONLY
    -- when there is nobody to shoot. No modes, no warmup, no fire-rate handcuff — SIMPLER beat complex.
    Rage = false,
    -- true (DEFAULT — the live-proven Polar delivery) = FORGED multi-tap: we FireServer the UseItem remote the
    -- tuned tap-count ourselves (the 6/4-tap burst that beat Kicia). false = NATURAL fallback: drive the game's
    -- own Gun.StartShooting via lf:Input (re-enters 08 -> encodeRageShot), one rate-gated shot per weapon
    -- cadence — it CANNOT deliver the burst and is kept only as a non-default fallback.
    RageDirectFire       = true,
    -- RATE DIAL (natural path only; the forged Polar path fires every frame, no rate gate). 0 = LEGAL: the
    -- real Gun.StartShooting self-throttles lf:Input to the weapon's own Info.ShootCooldown (self-calibrating
    -- per weapon; the per-instance cooldown is left intact). >0 = force interval (s): zeroes the per-instance
    -- cooldown so the gun stops throttling and this interval binds via the natural-path pre-gate = the >legal
    -- increment dial. Info.* stays READ-ONLY (never mutate the shared Info table).
    RageFireRateOverride = 0,
    -- VOID HIDE. true (DEFAULT — standalone parity) = deep-void teleport whenever there is nobody to shoot
    -- (no target / protected / reloading), so the lulls are spent un-hittable. false = stay at the real
    -- position between engagements (the restore keeps us home) — hittable through every reload.
    RageVoidHide         = true,
    RageEyeMuzzleSep     = 0.07,   -- vertical eye->muzzle separation so char0 ~= char1 (07a legacy encoder)

    -- Targeting
    RageSelfProtectHP    = 0,      -- deep-void hide below this fraction of max HP (0 = never self-protect)
    RageSkipImmune       = true,   -- never engage a spawn-protected / immune target
    RageHPPriority       = true,   -- prefer the lowest-HP target (else nearest)
    RageFastTargetSwitch = true,   -- drop a target the instant it becomes invalid
    RageVisCheck         = false,  -- require LOS to acquire (findTarget); off = point-blank guarantees LOS anyway

    -- Weapon economy (empty mag)
    RageWeaponPick      = "Primary", -- "Primary" | "Secondary" | "Melee" (main weapon)
    RageOutOfAmmo       = "Reload",  -- "Reload" (standalone parity: 0.5s-gated StartReloading, hide through it) | "Switch" (equip the next loaded weapon)
    RageSwitchMelee     = false,     -- include Melee in the switch rotation
    RageSwitchRateLimit = 0.06,      -- min seconds between weapon-equip attempts

    -- Backstab geometry (the only remaining rage raycasts — flank a deflect/shield front arc). ORBIT/APEX
    -- only: Polar always parks point-blank on the head, the geometry the standalone proved.
    AvoidDeflect       = true,   -- get behind a katana user (its deflect only guards the front)
    RageShieldBackstab = true,   -- get behind a riot-shield user (its block only guards the front)
    RageKnifeBackstab  = true,   -- when WE hold a knife, prefer the backstab spot

    RageKillPlaneBuffer = 200,   -- studs above FallenPartsDestroyHeight we clamp the point-blank park to

    RageLab = false,             -- in-game telemetry overlay (11_lab)

    RageVoidPhase = true,  -- LIVE: 11_lab reads this as the hide-detection gate (keep true; NOT dead)

    -- ── RAGE MODE (4-mode system) ───────────────────────────────────────────
    RageMode = "Polar",              -- "Polar" | "Apex" | "Orbit" | "Phantom" (dropdown; default = proven Polar)

    -- ── APEX (strict-superset Polar: same-frame acquire + head-sane swing + fire-through-economy) ──
    RageInEngineAcquire   = true,   -- APEX reads target in-engine same-frame (Polar keeps its own path)
    RageHeadSaneFirst     = true,   -- swing to a currently head-surfaced enemy (lobby; inert 1v1); dwell-guarded
    RageTargetDwell       = 8,      -- frames a voided lock is held before a head-sane swing (anti-UE-abandon)
    RageSwapAmmoWatermark = 2,      -- pre-emptive fire-then-swap at this ammo (0 = pure-Polar economy)
    RageReloadOnVoidOnly  = true,   -- defer the physical reload to a target-void lull (dry+sane still eats it)
    RageHeadMissGrace     = 2,      -- frames firing at the cached head instance through a FindFirstChild miss
    RageTaps              = 6,      -- taps/sane frame (was POLAR_TAPS_SANE; sweep-set K, ship 6)
    RageTapsVoid          = 6,      -- taps/void first-strike (was 4; VOID→K rate fix — inert if sweep gives K=4)

    -- ORBIT tuning (anti-normal-player stepped side-vantage rager)
    RageCombatOrbitRadius = 60,      -- base vantage distance
    RageOrbitDwell        = 0.09,    -- seconds each vantage is held (replication settle window)
    RageCombatOrbitHeight = 8,       -- vantage height above the head
    RageCombatOrbitJitter = true,    -- +/- vertical spread per candidate
    RagePBEyeUp           = 3,       -- eye height above the vantage (Orbit reads this; Polar hardcodes +2.5)

    -- ── PHANTOM (Polar fire + detached-hitbox defense; knobs opt-in, MODE default stays Polar) ──
    -- Keep HRP + visible rig point-blank (Polar fire, eye-body gate passes) while our OWN EntityHitbox parts
    -- are weld-severed and CFrame-flung past MAX_RAYCAST=400 -> enemy damage rays miss us while we keep killing.
    RageDetachDist     = 500,    -- fling distance (studs); MAX_RAYCAST=400, the margin is float-slop headroom
    RageDetachJitter   = true,   -- re-roll a few studs/frame (defeats a last-frame lucky ray)
    RageDetachKillFast = true,   -- fling ONLY during the lethal burst (RageFiring); re-attach between bursts
    RageDetachBodyOnly = false,  -- degraded fallback: fling only the Body hitboxes (stay head-shottable)

    -- ── Stale-config stubs (autoload safety — a saved config from the old modes must not nil-index) ──
    RageMultiTap  = 1,             -- dead stub (fire rate is self-paced at the weapon interval)
    RageCombatMode = "Nullpoint",  -- dead stub (legacy mode name)

    -- ESP — global
    ESP = false, ESPTeamCheck = true,
    ESPColorMode = "Static",          -- Static | Team | Visibility | Distance | Rainbow (drives element colors)
    ESPOutline = true, ESPOutlineTransparency = 0.35,  -- uniform 1px black outline/shadow behind every primitive
    ESPMaxDistance = 1200, ESPMaxPlayers = 0,          -- ESPMaxPlayers 0 = render all; >0 = nearest-N only
    ESPFont = "Plex", ESPTextScaling = false,
    ESPTextSize = 13, ESPInfoTextSize = 11, ESPHealthTextSize = 11,

    -- ESP — elements (per-element toggles)
    ESPBox = true, ESPBoxMode = "Static", ESPBoxBrackets = true, ESPCornerLength = 0.25,
    ESPBoxFill = false, ESPBoxThickness = 1,
    ESPName = true, ESPDistance = true, ESPWeapon = false,
    ESPHealth = true, ESPHealthOrientation = "Vertical", ESPHealthNumberMode = "OnDamage", ESPHealthGradient = true,
    ESPSkeleton = false, ESPSkeletonThickness = 1,
    ESPChams = false,
    ESPTracers = false, ESPTracerThickness = 1, ESPTracerOrigin = "Bottom",
    ESPArrows = true,
    ESPHeadDot = false, ESPHeadDotSize = 4,   -- small filled circle at head position (aim reference dot)

    -- ESP — per-element colors (base color in Static mode / accent otherwise)
    ESPBoxColor          = Color3.fromRGB(235, 235, 245),
    ESPBoxFillColor      = Color3.fromRGB(255, 59, 78),
    ESPNameColor         = Color3.fromRGB(243, 246, 250),   -- SIGNAL TEXT-1 (#F3F6FA): names stay neutral in every color mode
    ESPInfoColor         = Color3.fromRGB(174, 185, 197),   -- SIGNAL TEXT-2 (#AEB9C5): dist/weapon footer sits a tier below the TEXT-1 name
    ESPHealthColor       = Color3.fromRGB(61, 224, 122),    -- SIGNAL hp4 green (#3DE07A); used when ESPHealthGradient=false
    ESPSkeletonColor     = Color3.fromRGB(215, 220, 235),
    ESPTracerColor       = Color3.fromRGB(255, 59, 78),
    ESPHeadDotColor      = Color3.fromRGB(255, 255, 255),
    ESPChamsFillColor    = Color3.fromRGB(255, 59, 78),
    ESPChamsOutlineColor = Color3.fromRGB(255, 255, 255),

    -- ESP — Team / Visibility / Distance color-mode palette
    ColorEnemy       = Color3.fromRGB(255, 59, 78),    -- SIGNAL ENEMY (#FF3B4E)
    ColorTeam        = Color3.fromRGB(53, 215, 199),   -- SIGNAL ALLY (#35D7C7)
    ColorEnemyOcc    = Color3.fromRGB(168, 85, 96),    -- SIGNAL enemy-occ (#A85560): same hue pulled toward grey
    ColorTeamOcc     = Color3.fromRGB(92, 153, 147),   -- SIGNAL ally-occ  (#5C9993): same hue pulled toward grey
    ColorVisible     = Color3.fromRGB(41, 224, 255),   -- SIGNAL cyan visible (#29E0FF)
    ESPDistNearColor = Color3.fromRGB(80, 255, 140),
    ESPDistFarColor  = Color3.fromRGB(255, 70, 90),

    -- ESP — per-element transparency (0 = opaque, 1 = invisible)
    ESPBoxTransparency          = 0,
    ESPBoxFillTransparency      = 0.75,
    ESPNameTransparency         = 0,
    ESPHealthTransparency       = 0.1,
    ESPSkeletonTransparency     = 0.2,
    ESPTracerTransparency       = 0.35,
    ESPHeadDotTransparency      = 0,
    ESPChamsFillTransparency    = 0.6,
    ESPChamsOutlineTransparency = 0,

    -- Visuals
    Visuals = false, VisualsPreset = "Neutral", VisualsPerformanceMode = false,
    VisualsFullbright = false,  -- master fullbright override (Ambient/OutdoorAmbient white, no global shadows); layered over preset, gated off in Performance mode
    VisualsNoFog = false,       -- remove distance fog (FogEnd/FogStart -> 1e6, preset Atmosphere Density -> 0); gated off in Performance mode
    VisualsHolograms = false, VisualsRainbowMap = false,
    VisualsRainbowMapSpeed = 0.15, VisualsStretch = 1.0,
    VisualsStretchMin = 0.5, VisualsStretchMax = 1.2,
    VisualsCameraSway = false,         -- purely-local handheld "breathing" camera drift (shares the stretch RenderStep + Rage stand-down)
    VisualsCameraSwayAmount = 0.5,     -- 0–1: sub-degree at 0.5, ~1.6° roll at 1.0
    VisualsHologramDuration = 3.5, VisualsHologramRange = 300,
    VisualsHologramVisibility = 1.4,   -- bug#6: 0.2–2 master dial (opacity + bone thickness + orb scale). 1.0 = old look; 1.4 default = pops on fresh load; 2.0 = near-solid.
    VisualsHologramColor  = Color3.fromRGB(0, 220, 255),
    VisualsHologramAccent = Color3.fromRGB(255, 60, 200),

    -- ── COSMETICS / HUD / RADAR (added) ──────────────────────────────────────
    -- Visuals: grade + afterimage (06_visuals.lua already READS these via cfg())
    VisualsGrade = "Crisp",                 -- None | Crisp | Cold | Warm | Comp
    VisualsGradeStrength = 0.6,
    VisualsBloom = false,                   -- master BloomEffect layered over the active preset
    VisualsBloomIntensity = 1.0,            -- 0–3 extra-glow dial
    -- Camera frame FX (06_visuals camera-frame subsystem; all purely local, no camera-fn hooks)
    VisualsVignette = false,                -- cinematic edge darkening (ScreenGui overlay)
    VisualsVignetteStrength = 0.6,          -- 0–1 edge opacity
    VisualsLetterbox = false,               -- cinema bars top+bottom
    VisualsLetterboxSize = 0.10,            -- 0.04–0.18 fraction of screen height per bar
    VisualsDOF = false,                     -- lens depth-of-field (Lighting DepthOfFieldEffect)
    VisualsDOFDistance = 28,                -- focus distance (studs)
    VisualsDOFBlur = 0.5,                   -- 0–1 → Far/Near blur intensity
    VisualsHologramStyle = "Orb",           -- Orb | Skeleton | Wraith
    VisualsHologramLethal = true,
    VisualsHologramLethalColor = Color3.fromRGB(255, 200, 60),
    -- ── HUD master (STRUCTURAL: decouples FX from the Visuals master) ──────────
    HUD = false,                   -- bug#2: default OFF — clean fresh load; user opts in (then #3 makes the hitmarker work). independent hit/threat-feedback master; 12_init starts it,
                                   -- autoload toggle reconciles saved state (matches FX sub-toggle defaults)
    -- Feedback FX (06_visuals.lua Visuals.FX already READS these)
    FXHitMarker = true,
    FXHitMarkerColor = Color3.fromRGB(255, 255, 255),
    FXHitMarkerCritColor = Color3.fromRGB(255, 194, 75),   -- SIGNAL GOLD (#FFC24B)
    FXHitMarkerLethalColor = Color3.fromRGB(255, 64, 78),
    FXHitMarkerGap = 5,
    FXHitMarkerLen = 8,
    FXHitMarkerThickness = 2,
    FXHitSound = true,
    FXHitSoundId = "",
    FXKillSoundId = "",
    FXHitSoundVolume = 0.5,
    FXDamageNumbers = true,
    FXDamageAccumWindow = 0.9,
    FXKillBanner = true,
    FXKillBannerColor = Color3.fromRGB(255, 194, 75),   -- SIGNAL GOLD (#FFC24B)
    FXKillFeed = true,
    FXHeadshotSpark = true,
    FXHitFlash = true,
    FXDamageDirection = true,
    FXLowHPVignette = true,
    FXLowHPThreshold = 0.35,
    FXCritDamage = 30,
    -- ── §07 Beam bullet tracer (06_visuals FX, default OFF) ────────────────────
    FXBeamTracer     = false,
    FXBeamStyle      = "Glow",                          -- "Line" | "Glow"
    FXBeamHitColor   = Color3.fromRGB(255, 194, 75),    -- #FFC24B
    FXBeamMissColor  = Color3.fromRGB(143, 160, 176),   -- #8FA0B0
    -- ── §07 Gradient FOV ring (06_visuals FX, default OFF) ─────────────────────
    FXFovRing        = false,
    FXFovColorA      = Color3.fromRGB(53, 215, 199),    -- #35D7C7
    FXFovColorB      = Color3.fromRGB(255, 194, 75),    -- #FFC24B
    FXFovThickness   = 1.5,                             -- 0.5–4
    FXFovDriftSpeed  = 0.15,                            -- rev/s
    FXFovFill        = false,                           -- faint #FFC24B @0.05 disc
    FXFovRotate      = true,                            -- drift-linked rotation
    -- ── §07 World hit/kill sparks (06_visuals FX, default OFF) ─────────────────
    FXWorldSpark     = false,                           -- hit spark + kill ring (colors code-const #FFE9B8/#FFC24B)
    -- ── §v3 Beam tracer polish (06_visuals spawnBeamTracer) ──
    FXBeamWidth0    = 0.18,   -- muzzle width (studs)
    FXBeamWidth1    = 0.04,   -- impact width (studs) — taper
    FXBeamDur       = 0.55,   -- lifetime (s)
    FXBeamGlowLight = true,   -- brief muzzle PointLight when FXBeamStyle == "Glow"
    FXBeamTravel      = true, -- comet window races muzzle->hit (the whip; off = instant full beam)
    FXBeamTravelSpeed = 1400, -- studs/s (travel time capped at 0.25s)
    FXBeamImpact      = true, -- pooled micro-flash (neon pop + light spike) at the tracer endpoint
    -- FXBeamMissColor retained above but now UNUSED (onHit always lands; stale-save stub only)
    -- ── §v3 World spark (06_visuals spawnWorldSpark) ──
    FXWorldSparkBloom = true, -- PointLight bloom with the hit/kill spark
    -- ── FABLE v6 · Kill flourish (06_visuals kill FX, default OFF) ──
    FXKillPillar      = false,                          -- light column + ground ring + rising motes
    FXKillPillarColor = Color3.fromRGB(255, 194, 75),   -- SIGNAL GOLD (#FFC24B)
    FXKillShards      = false,                          -- neon shatter burst on ballistic arcs
    FXKillShardsColor = Color3.fromRGB(155, 232, 255),  -- SIGNAL EDGE (#9BE8FF)
    FXKillPulse       = false,                          -- 0.35s impact-frame lighting pulse
    FXKillPulseAmount = 0.6,                            -- pulse strength (0.2-1)
    -- ── §v3 HUD casing (06_visuals FX.update) ──
    FXFovCasing     = true,   -- dark under-ring beneath the FOV ring
    -- ── §v3 ESP health seg ticks (10_esp, OPTIONAL) ──
    ESPHealthSegTicks = false, -- 25/50/75% dark tick marks on the HP bar
    -- ── SIGNAL v4 · Custom crosshair (06_visuals FX) ──
    FXCrosshair          = false,
    FXCrosshairStyle     = "Cross",  -- Cross | X | T | Dot
    FXCrosshairColor     = Color3.fromRGB(243, 246, 250),   -- SIGNAL TEXT-1
    FXCrosshairDot       = true,
    FXCrosshairGap       = 4,
    FXCrosshairLen       = 7,
    FXCrosshairThickness = 2,
    FXCrosshairOutline   = true,
    FXCrosshairHitPop    = true,     -- +2px tick length for 60ms after a hit (rides _hm.t0)
    -- ── SIGNAL v4 · Watermark + fps/ping/session widget (06_visuals FX) ──
    HUDWatermark      = true,
    HUDWatermarkStats = true,        -- off => brand-only pill
    -- ── SIGNAL v4 · Target info panel (06_visuals FX; reads State.PrimaryTarget from 10_esp) ──
    FXTargetInfo       = false,
    FXTargetInfoOffset = 110,        -- px below screen center
    -- ── SIGNAL v4 · Active-features list (06_visuals FX) ──
    HUDBindList     = false,
    HUDBindListSide = "Left",        -- Left | Right (corner preset; no dragging by design)
    -- ── SIGNAL v4 · Hit marker style (06_visuals FX) ──
    FXHitMarkerStyle = "X",          -- X | Plus | Ring | Dot
    -- ── crosshair v2 (06_visuals FX) ──
    FXCrosshairBloom = false,        -- gap eases +3px per shot, decays over 120ms (reads State.Shots — no hooks)
    -- ── Instrument cluster (06c_hudplus.lua, default OFF, rides the HUD master) ──
    HUDCompass       = false,        -- top-center bearing rail: 15° ticks, cardinal labels, gold N, enemy pips
    HUDCompassWidth  = 380,          -- rail width (px); the ±60° window maps onto it
    HUDCompassPips   = true,         -- project alive enemies onto the rail by relative bearing (+N overflow pill)
    HUDThreatArc     = false,        -- 30° arc at 42px around the crosshair pointing at the nearest enemy bearing
    HUDRangeReadout  = false,        -- one mono "34m" line under the crosshair (nearest enemy)
    -- ── SIGNAL v4 · Radar sonar sweep + grid (10_esp renderRadar) ──
    ESPRadarGrid  = true,
    ESPRadarSweep = false,
    -- ── SIGNAL v4 · ESP spawn fade-in (10_esp) ──
    ESPFadeIn = true,                -- 180ms alpha fade when a target enters the render set
    -- ── SIGNAL v4 · Off-screen arrow polish (10_esp) ──
    ESPArrowDistFade  = true,        -- arrow alpha 1 -> 0.35 across ESPMaxDistance
    ESPArrowDistLabel = false,       -- small "212m" label just inside the arrow
    -- ── SIGNAL v4 · Look-direction line (10_esp) ──
    ESPLookLine       = false,
    ESPLookLineLength = 8,           -- studs of aim direction projected from the head
    -- ESP refinements (10_esp.lua — being implemented this pass)
    ESPSmoothing = true,
    ESPHealthSmooth = true,
    ESPHealthGhost = true,
    ESPDeclutter = true,
    ESPTextOutline = true,
    ESPChamsVisSplit = true,
    -- Radar (10_esp.lua)
    ESPRadar = false,   -- bug#2: default OFF — enabling ESP gives box/name/health, not an auto-radar
    ESPRadarSize = 200,
    ESPRadarRange = 150,
    ESPRadarRotate = true,
    ESPRadarVisSplit = true,
    ESPRadarInset = 24,
    -- Net-new ESP overlays (10_esp.lua — default OFF)
    ESPPeekAlert = false,
    ESPFacingIndicator = false,
    ESPThreatCount = false,
    ESPPrimaryEmphasis = false,
    ESPHealTick = false,
    -- ── §08 ESP refinements (10_esp.lua, default OFF except ESPChamsStyle) ──────
    ESPBoxGradient         = false,   -- rank 1: animated gradient box outline
    ESPBoxGradientA        = Color3.fromRGB(255, 59, 78),    -- #FF3B4E
    ESPBoxGradientB        = Color3.fromRGB(255, 194, 75),   -- #FFC24B
    ESPBoxGradientSpeed    = 0.12,    -- rev/s
    ESPTracerGradient      = false,   -- rank 2: alpha-ramp origin→target
    ESPNameHealthUnderline = false,   -- rank 3: health-tinted underline under name
    ESPLockChevron         = false,   -- rank 4: gold chevron over the primary/locked target
    ESPChamsStyle          = "Shade", -- rank 5: "Shade" | "Neon" | "Ghost"
    ESPNameMode            = "Display", -- §02/§09 name source: "Display" | "Username"
    -- ── Far-target legibility (10_esp.lua): floors, not restyling ────────────
    ESPMinBoxHeight = 24,          -- min on-screen box height px (width keeps aspect); 0 = off
    ESPTextMinSize  = 12,          -- min ESP text px at range (near targets keep their slider size)

    -- ── UTILITY ESP (10b_utility_esp.lua — thrown-gadget world ESP, default OFF) ──
    UtilityESP            = false,   -- grenades / molotovs / flashbangs / tripmines / satchels / warpstones
    UtilityESPMaxDistance = 250,     -- studs
    UtilityESPRing        = true,    -- breathing danger ring on explosives/traps
    UtilityESPLabels      = true,    -- "GRENADE · 23m" tag above the marker

    -- ── WEATHER / ENVIRONMENT (06b_weather.lua — independent world-FX masters, all default OFF) ──
    Weather          = false,          -- particle-dome weather master
    WeatherType      = "Rain",         -- Rain | Snow | Mist | Embers | Fireflies | Petals | Autumn | Ash | Sandstorm | BloodMoon
    WeatherIntensity = 1.0,            -- 0.15–2 AMBIENT density scale (rain/snow/mist/... + puddles/mood/rays).
                                       -- Sky events have no density, only a rate — they use their own dials below.
    WeatherMeteors   = false,          -- meteor shower: arcing comets (radiant flare head + long luminous tail +
                                       -- shed ember wake) on two tiers, far + near; ~35% burn out mid-sky, rest
                                       -- impact. Cadence AND live caps both ride WeatherMeteorRate below.
    WeatherMeteorRate = 1.0,           -- 0.25–3 meteor frequency ×; 1 = stock (far ~8.5s, near ~19s, caps 2 far/1 near)
    WeatherStarRate  = 1.0,            -- 0.25–3 shooting-star frequency ×; 1 = stock (event ~8s, cap 6 live)
    WeatherClockDial = false,          -- drift-guarded day/night cycle (only while the Visuals master is OFF)
    WeatherClockCycleMin = 8,          -- minutes per full 24h ClockTime loop
    WeatherStorm     = false,          -- lightning bolts + sky flash (+ thunder if id set)
    WeatherStormFlash= true,           -- global brightness pop with each bolt
    WeatherStormMin  = 4,              -- min seconds between bolts (the storm's frequency control)
    WeatherStormVar  = 8,              -- added random seconds
    WeatherThunderId = "rbxassetid://9113169432",   -- Artillery Distant 2 (PSE, 6.8s — a real thunder roll; old 138186576 was "Explosion", a 4.8s blast)
    WeatherSoundIds  = {                            -- ambient loops per mood — every id load-verified in Studio 2026-07-13
        rain  = "rbxassetid://9112858162",         --   Rainstorm Heavy Summer Downpour 2 (PSE, 61s). Alts: 9112857744 lighter / 9112854469 rain-on-water.
                                                   --   (old 142376088 was literally "Raining Tacos" — the reported rain-sound bug)
        wind  = "rbxassetid://9112854440",         --   wind ambience (56s) — Snow/Mist/Ash
        fire  = "rbxassetid://2787093357",         --   crackling fire (43.6s) — Embers
        night = "rbxassetid://9112764573",         --   night crickets (PSE, 64s) — Fireflies
        birds = "rbxassetid://9112749254",         --   songbirds (PSE, 36s) — Petals
    },
    WeatherSoundVolume = 0.35,         -- ambient-loop volume; applies LIVE to the playing loop (no retoggle).
                                       -- One-shots scale off it: thunder ×8, meteor impact ×6 (impacts must punch).
    WeatherMood      = true,           -- subtle per-type ambient colour grade (cool rain, warm embers, ...)
    -- Skybox preset swap (06b_weather.lua)
    SkyboxPreset        = "Off",       -- Off | Space | Sunset | Clouds | Storm | Winter | Vaporwave
    SkyboxHideCelestial = false,       -- zero the sun/moon/stars on the active preset
    WeatherGodRays      = false,       -- volumetric sun shafts (SunRaysEffect); independent of the weather master
    WeatherRainbow      = false,       -- ROYGBIV arch across the sky (independent; pairs with rain)
    WeatherShootingStars = false,      -- sky-event scheduler: brief white-blue streaks high overhead (pairs with Nebula/Space)
    WeatherPuddles   = false,          -- interactive rain puddles: wet Glass patches + rain ripples + footfall
                                       -- splashes for every nearby character (active while type=Rain or Storm is on)
}

-- Mobile default: no physical aim key, so always on
if isMobile then
    Config.AimbotKey = "Always"
end

-- ── CAMERA-ROTATION ANTI-AIM (REMOVED, STAGE A) ─────────────────────────────
-- This module used to hook Util:EncodeCameraRotation to forge our replicated look-pitch. It was
-- structurally weak (it moved only our VISIBLE head ~1.5 studs, versus a rigid 2-stud damage hitbox,
-- so it could not reliably make an aimed shot miss) and it hooked a game function for zero defensive
-- value = pure detection surface. The clean 3-mode rage engine (07b_rage_engine.lua) owns position +
-- fire directly and never touched this path. Removed entirely: no hook, no config, no GUI, no sentinel.

-- ── STATE ──────────────────────────────────────────────────────────────────
local State = {
    Target = nil, CamPos = Vector3.zero,

    -- Aimbot
    AimbotTarget = nil, AimbotPart = nil,
    AimbotLastTarget = nil, AimbotLastTargetTime = 0,
    AimbotKeyHeld = false, SilentLastTarget = nil,

    -- ── RAGE (clean engine — the single-loop rewrite) ──────────────────────────
    -- Desync/park state (one writer = Heartbeat; two position-only restores):
    RageRealCF   = nil,   -- our real CFrame (restored at Stepped + render while desynced; CFrame/position ONLY)
    RageRealChar = nil,   -- Character RageRealCF belongs to (per-life restore TOKEN — blocks the respawn yank)
    RageTarget   = nil,   -- sticky target (survives the enemy's void-flickers)
    RageVoidCF   = nil,   -- the held far void spot (re-rolled on expiry, cleared on respawn)
    RageVoidNext = 0,     -- tick() when the current void spot expires and re-rolls
    -- Fire discipline:
    RageLastFireTime = 0, -- tick() of the last emitted packet (fire-rate gate: one shot per weapon interval)
    RageReloadLast   = 0, -- tick() of the last StartReloading Input (empty->reload rate-limit)
    RageSwitchLast   = 0, -- tick() of the last weapon-equip attempt (rate limit)
    -- ── Mode: Orbit (stepped side-vantage) ──
    OrbitAngle        = 0,     -- golden-angle vantage accumulator (advances per repick)
    OrbitVantage      = nil,   -- currently held side-vantage (Vector3) for the dwell
    OrbitVantageUntil = 0,     -- tick() the current vantage dwell expires
    -- ── Mode: Apex (strict-superset Polar: same-frame acquire + head-sane swing + fire-through-economy) ──
    RageLastSaneHeadPos  = nil,-- last SANE HitboxHead pos of the current target (hittability anchor; reset on target change/death)
    RageLastSaneHeadTgt  = nil,-- player the cached head pos belongs to (invalidates the cache on target switch)
    RagePendingSwap      = nil, -- slot queued by apexEconomy; equipped AFTER this frame's taps (fire-then-swap)
    ParkGen              = 0,   -- park generation token (tap-spread guard scaffold; ++ each fire frame)
    RageGhostCanary      = 0,   -- Iron-Law machine-check: ++ if emitTaps sees a non-sane hrp; MUST stay 0
    -- ── Mode: Phantom (detached-hitbox defense; per-life weld cache + Iron-Law canary) ──
    RageDetachWelds      = nil,  -- cached { {joint=<cloneref weld>, wasEnabled=<bool>, part=<raw ref>}, ... }; nil = not severed
    RageDetachChar       = nil,  -- Character the weld cache belongs to (per-life token, like RageRealChar)
    RageDetachCanary     = 0,    -- Iron-Law machine-check: ++ if we ever fling while not isSanePos(hrp); MUST stay 0
    -- Frame flags / status:
    RageKnifeHintLast = 0,      -- tick() of the last "switch to Orbit" knife-death hint (rate-limit gate)
    RageInMatch  = false, -- cached inMatch() result (the restores read it)
    RageStatus   = "Idle",-- human-readable engine state (Lab/GUI status bar)
    -- Natural-delivery handoff (08 hook reads these in encodeRageShot):
    RageFireFromPos = nil, -- eye world pos our shot fires from this frame
    RageFireAimPos  = nil, -- aim world pos (the live head)
    RageFireHitPart = nil, -- the live HitboxHead instance (char2) this shot is locked onto
    -- Lab telemetry:
    RageDealtTotal  = 0,   -- cumulative damage we've dealt to targets (Lab; Lab-gated poll)
    -- External-reader stubs (indexed by 05_utils / 11_lab; must stay non-nil tables):
    RageFiring          = false, -- "engaged this frame" (09_aimbot yield gate + 11_lab ATTACK readout)
    RageVoidActive      = false, -- true while the replicated body is in the void (11_lab HIDE readout)
    RageTrueVelocityMap = {},    -- per-player finite-diff velocity fallback (05_utils calculateLead; filled by target loop)
    RageSuspectedProtection = {},-- per-player protection TTL (05_utils isProtected read)
    RageBacktrackBuf    = {},    -- per-target head history (11_lab enemyDesync read; left empty by the clean engine)
    RageCharTokens      = {},    -- per-player character-add token (respawn-yank guard)

    Shots = 0, Hits = 0,
    ESPObjects = {}, RainbowHue = 0,
    VisualsCurrentPreset = nil,
}

-- ── SCREENDRAW SHIM ─────────────────────────────────────────────────────────
-- Drawing-API-compatible overlay backed by pooled ScreenGui Instances so the overlay
-- renders INSIDE the game viewport (immune to 2nd-monitor / windowed desktop offset).
-- WorldToViewportPoint is inset-INCLUSIVE (absolute) pixels; a ScreenGui with
-- IgnoreGuiInset=true shares that origin, so UDim2.fromOffset(wvp.X, wvp.Y) lands on the
-- projected point with NO GetGuiInset math. Drawing.Transparency is OPACITY -> instance
-- transparency = 1 - it. Layering = creation order via an incrementing ZIndex.
local screenDraw
do
    local CoreGui = game:GetService("CoreGui")
    -- Per-layer bundles so the FX HUD can parent to a higher-DisplayOrder ScreenGui (sits ABOVE ESP).
    local _layers = {}   -- [layer] = { gui = Instance|nil, z = number, pools = { [kind] = {…} } }
    local LAYER_ORDER = { base = 100000, fx = 100100 }
    local LAYER_NAME  = { base = "LH_Overlay", fx = "LH_Overlay_FX" }

    local FONT_MAP = {
        [0] = Enum.Font.Gotham, [1] = Enum.Font.SourceSans,
        [2] = Enum.Font.GothamMedium, [3] = Enum.Font.Code,   -- 3 = Monospace
    }
    local BLACK = Color3.new(0, 0, 0)
    local function op(t) return 1 - (t or 1) end               -- opacity -> transparency

    local function gui(layer)
        local Lr = _layers[layer]
        if not Lr then Lr = { gui = nil, z = 0, pools = {} }; _layers[layer] = Lr end
        if Lr.gui and Lr.gui.Parent then return Lr.gui end
        local g = Instance.new("ScreenGui")
        g.Name = LAYER_NAME[layer] or "LH_Overlay"
        g.IgnoreGuiInset = true          -- ★ match WorldToViewportPoint absolute origin
        g.ResetOnSpawn  = false
        g.DisplayOrder  = LAYER_ORDER[layer] or 100000
        g.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
        local ok = pcall(function() g.Parent = (gethui and gethui()) or CoreGui end)
        if not ok or not g.Parent then
            pcall(function() g.Parent = lp:FindFirstChildOfClass("PlayerGui") end)
        end
        Lr.gui = g
        return g
    end

    -- build(kind) -> inst, apply(state, key)   (nil on unknown kind)
    local function build(kind)
        if kind == "Square" then
            local f = Instance.new("Frame")
            f.BorderSizePixel = 0; f.BackgroundTransparency = 1
            f.AnchorPoint = Vector2.new(0, 0); f.Visible = false
            local st = Instance.new("UIStroke"); st.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
            st.Enabled = false; st.Parent = f
            -- lazy UIGradient on the UIStroke: the shim-native "animated gradient box outline" (esp-v3 §02).
            -- Created on first Gradient write (never per frame), spun via GradientRotation. Stays nil/off for
            -- every non-gradient Square so the base render path is byte-identical.
            local ug = nil
            local function apply(s, k)
                if k == "Position" then if s.Position then f.Position = UDim2.fromOffset(s.Position.X, s.Position.Y) end
                elseif k == "Size" then if s.Size then f.Size = UDim2.fromOffset(s.Size.X, s.Size.Y) end
                elseif k == "Color" then if s.Color then f.BackgroundColor3 = s.Color; st.Color = s.Color end
                elseif k == "Thickness" then st.Thickness = math.max(s.Thickness or 1, 0.1)
                elseif k == "Transparency" or k == "Filled" then
                    local o = op(s.Transparency)
                    if s.Filled == false then f.BackgroundTransparency = 1; st.Enabled = true; st.Transparency = o
                    else f.BackgroundTransparency = o; st.Enabled = false end
                elseif k == "Gradient" then
                    if s.Gradient then
                        if not ug then ug = Instance.new("UIGradient"); ug.Parent = st end
                        ug.Color = s.Gradient; ug.Enabled = true
                    elseif ug then ug.Enabled = false end
                elseif k == "GradientRotation" then if ug then ug.Rotation = s.GradientRotation or 0 end
                elseif k == "Visible" then f.Visible = s.Visible and true or false end
            end
            return f, apply

        elseif kind == "Line" then
            local f = Instance.new("Frame")
            f.BorderSizePixel = 0; f.AnchorPoint = Vector2.new(0.5, 0.5); f.Visible = false
            local _len = nil   -- cached endpoint distance; lets Thickness update Size.Y with no trig
            local function geom(s)
                if not (s.From and s.To) then return end
                local dx, dy = s.To.X - s.From.X, s.To.Y - s.From.Y
                local len = math.sqrt(dx * dx + dy * dy)
                _len = len
                f.Position = UDim2.fromOffset((s.From.X + s.To.X) * 0.5, (s.From.Y + s.To.Y) * 0.5)
                f.Size     = UDim2.fromOffset(len, math.max(s.Thickness or 1, 0.1))
                f.Rotation = math.deg(math.atan2(dy, dx))
            end
            local function apply(s, k)
                -- geom recomputes ONCE per frame. Every caller co-writes the pair in order From THEN To
                -- (and no caller moves From without also moving To), so trigger on the "To" write only —
                -- From is already in state — mirroring the Triangle PointC pattern. From is a no-op. This
                -- halves the sqrt/atan2/UDim2 churn for moving lines vs. firing on both endpoints.
                if k == "To" then geom(s)
                elseif k == "From" then -- no-op: geom fires on the paired To write
                elseif k == "Thickness" then if _len then f.Size = UDim2.fromOffset(_len, math.max(s.Thickness or 1, 0.1)) end
                elseif k == "Color" then if s.Color then f.BackgroundColor3 = s.Color end
                elseif k == "Transparency" then f.BackgroundTransparency = op(s.Transparency)
                elseif k == "Visible" then f.Visible = s.Visible and true or false end
            end
            return f, apply

        elseif kind == "Text" then
            local t = Instance.new("TextLabel")
            t.BackgroundTransparency = 1; t.BorderSizePixel = 0; t.Visible = false
            t.AutomaticSize = Enum.AutomaticSize.XY; t.RichText = false
            t.TextYAlignment = Enum.TextYAlignment.Top
            t.AnchorPoint = Vector2.new(0, 0); t.TextXAlignment = Enum.TextXAlignment.Left
            local st = Instance.new("UIStroke"); st.Thickness = 1; st.Color = BLACK
            st.Enabled = false; st.Parent = t
            local function apply(s, k)
                if k == "Text" then t.Text = tostring(s.Text or "")
                elseif k == "Size" then t.TextSize = math.max(s.Size or 12, 1)
                elseif k == "Font" then t.Font = FONT_MAP[s.Font or 2] or Enum.Font.GothamMedium
                elseif k == "Color" then if s.Color then t.TextColor3 = s.Color end
                elseif k == "Center" or k == "RightAlign" then
                    -- RightAlign (shim extension): Center=false + RightAlign=true anchors the
                    -- text's RIGHT edge on Position.X (title-block corner docking) — no TextBounds math,
                    -- no frame lag. Both keys resolve through one 3-way anchor pick so write order between
                    -- them never matters (Center wins when set).
                    if s.Center then t.AnchorPoint = Vector2.new(0.5, 0); t.TextXAlignment = Enum.TextXAlignment.Center
                    elseif s.RightAlign then t.AnchorPoint = Vector2.new(1, 0); t.TextXAlignment = Enum.TextXAlignment.Right
                    else t.AnchorPoint = Vector2.new(0, 0); t.TextXAlignment = Enum.TextXAlignment.Left end
                    if s.Position then t.Position = UDim2.fromOffset(s.Position.X, s.Position.Y) end
                elseif k == "Position" then if s.Position then t.Position = UDim2.fromOffset(s.Position.X, s.Position.Y) end
                elseif k == "Outline" then st.Enabled = s.Outline and true or false
                elseif k == "OutlineColor" then if s.OutlineColor then st.Color = s.OutlineColor end
                elseif k == "Transparency" then local o = op(s.Transparency); t.TextTransparency = o; st.Transparency = o
                elseif k == "Visible" then t.Visible = s.Visible and true or false end
            end
            return t, apply

        elseif kind == "Circle" then
            local f = Instance.new("Frame")
            f.BorderSizePixel = 0; f.AnchorPoint = Vector2.new(0.5, 0.5)
            f.BackgroundTransparency = 1; f.Visible = false
            local uc = Instance.new("UICorner"); uc.CornerRadius = UDim.new(1, 0); uc.Parent = f
            local st = Instance.new("UIStroke"); st.Enabled = false; st.Parent = f
            local function apply(s, k)
                if k == "Radius" then local d = 2 * (s.Radius or 0); f.Size = UDim2.fromOffset(d, d)
                elseif k == "Position" then if s.Position then f.Position = UDim2.fromOffset(s.Position.X, s.Position.Y) end
                elseif k == "Color" then if s.Color then f.BackgroundColor3 = s.Color; st.Color = s.Color end
                elseif k == "Thickness" then st.Thickness = math.max(s.Thickness or 1, 0.1)
                elseif k == "Transparency" or k == "Filled" then
                    local o = op(s.Transparency)
                    if s.Filled == false then f.BackgroundTransparency = 1; st.Enabled = true; st.Transparency = o
                    else f.BackgroundTransparency = o; st.Enabled = false end
                elseif k == "Visible" then f.Visible = s.Visible and true or false end
                -- NumSides ignored (true circle)
            end
            return f, apply

        elseif kind == "Triangle" then
            -- Direction marker with TWO render paths in one asset-free primitive (rotated Frame "lines"):
            --  • Filled == false → bold CASED caret: dark casing legs UNDER thick color legs + a short apex
            --    stem. Self-cased here so callers (off-screen arrows / radar chevrons) need no companion
            --    outline. Reads as a solid arrowhead pointing at the apex; legible on bright scenes.
            --  • Filled == true  → genuine FAN-FILL: 5 rotated strips base→apex (overlap kills seams). Used
            --    by self-pip / lock chevron so they read as solid triangles, not hollow carets.
            -- A transparent container Frame is the pooled `inst` (ZIndex/Parent/Visible act on a real
            -- GuiObject); all legs are its children under Sibling ZIndex.
            local box = Instance.new("Frame")
            box.BackgroundTransparency = 1; box.BorderSizePixel = 0
            box.AnchorPoint = Vector2.new(0, 0); box.Position = UDim2.fromOffset(0, 0)
            box.Size = UDim2.fromScale(1, 1); box.Visible = false   -- legs positioned in viewport px
            local BLK = Color3.new(0, 0, 0)
            local function mkleg(col)
                local l = Instance.new("Frame")
                l.BorderSizePixel = 0; l.AnchorPoint = Vector2.new(0.5, 0.5)
                l.BackgroundColor3 = col or Color3.new(1, 1, 1); l.Visible = false; l.Parent = box
                return l
            end
            -- casing legs created FIRST → lower sibling ZIndex → render UNDER the color legs
            local cas1, cas2 = mkleg(BLK), mkleg(BLK)
            local leg1, leg2, leg3, leg4, leg5 = mkleg(), mkleg(), mkleg(), mkleg(), mkleg()
            local legs = { leg1, leg2, leg3, leg4, leg5 }
            local stem = mkleg()   -- caret apex stem
            local function mid(p, q) return Vector2.new((p.X + q.X) * 0.5, (p.Y + q.Y) * 0.5) end
            local function leg(l, p, q, thick)
                local dx, dy = q.X - p.X, q.Y - p.Y
                local len = math.sqrt(dx * dx + dy * dy)
                l.Position = UDim2.fromOffset((p.X + q.X) * 0.5, (p.Y + q.Y) * 0.5)
                l.Size     = UDim2.fromOffset(math.max(len, 1), thick)
                l.Rotation = math.deg(math.atan2(dy, dx))
                l.Visible  = true
            end
            local function geom(s)
                local A, B, C = s.PointA, s.PointB, s.PointC
                if not (A and B and C) then return end
                local dA = (A - mid(B, C)).Magnitude
                local dB = (B - mid(A, C)).Magnitude
                local dC = (C - mid(A, B)).Magnitude
                local apex, b1, b2
                if dA >= dB and dA >= dC then apex, b1, b2 = A, B, C
                elseif dB >= dC then apex, b1, b2 = B, A, C
                else apex, b1, b2 = C, A, B end
                local base = mid(b1, b2)
                local h    = math.max((apex - base).Magnitude, 1)
                if s.Filled == true then
                    -- FAN-FILL: hide caret extras; stripe base→apex with 5 overlapping strips
                    cas1.Visible = false; cas2.Visible = false; stem.Visible = false
                    local ft = h / 5 + 1
                    for k = 1, 5 do
                        local t = (k - 0.5) / 5
                        leg(legs[k], b1:Lerp(apex, t), b2:Lerp(apex, t), ft)
                    end
                else
                    -- CASED CARET: dark casing legs (thick+2) under color legs (thick) + apex stem
                    local thick = math.clamp(h * 0.34, 3, 6)
                    leg(cas1, apex, b1, thick + 2)
                    leg(cas2, apex, b2, thick + 2)
                    leg(leg1, apex, b1, thick)
                    leg(leg2, apex, b2, thick)
                    local dir = apex - base
                    if dir.Magnitude > 0 then dir = dir.Unit else dir = Vector2.new(0, -1) end
                    leg(stem, apex, apex + dir * (h * 0.5), thick)
                    leg3.Visible = false; leg4.Visible = false; leg5.Visible = false
                end
            end
            local function apply(s, k)
                -- All three points are co-written each frame in order PointA,PointB,PointC → recompute on
                -- PointC (A/B no-op). Filled recomputes as a cheap safety when the mode flips.
                if k == "PointC" or k == "Filled" then geom(s)
                elseif k == "PointA" or k == "PointB" then -- no-op
                elseif k == "Color" then
                    if s.Color then
                        leg1.BackgroundColor3 = s.Color; leg2.BackgroundColor3 = s.Color
                        leg3.BackgroundColor3 = s.Color; leg4.BackgroundColor3 = s.Color
                        leg5.BackgroundColor3 = s.Color; stem.BackgroundColor3 = s.Color
                    end
                elseif k == "Transparency" then
                    local o = op(s.Transparency)
                    leg1.BackgroundTransparency = o; leg2.BackgroundTransparency = o
                    leg3.BackgroundTransparency = o; leg4.BackgroundTransparency = o
                    leg5.BackgroundTransparency = o; stem.BackgroundTransparency = o
                    -- casing keeps its own opaque-black alpha so the caret stays cased on bright scenes
                elseif k == "Visible" then box.Visible = s.Visible and true or false end
            end
            return box, apply
        end
        return nil
    end

    screenDraw = function(kind, layer)
        layer = layer or "base"
        local Lr = _layers[layer]
        if not Lr then Lr = { gui = nil, z = 0, pools = {} }; _layers[layer] = Lr end
        local pool = Lr.pools[kind]; if not pool then pool = {}; Lr.pools[kind] = pool end
        local inst, applyFn
        local reused = table.remove(pool)
        if reused then
            inst, applyFn = reused.inst, reused.apply
        else
            inst, applyFn = build(kind)
            if not inst then return nil end
            inst.Parent = gui(layer)
        end
        -- LAYERING: re-stamp an incrementing ZIndex on EVERY serve (fresh AND pooled reuse), so a
        -- draw object's ZIndex always reflects its creation-request order for THIS player — exactly
        -- reproducing first-build stacking. Stamping only at build time let a pooled instance keep a
        -- stale ZIndex from a prior role/player (LIFO churn), inverting intra-kind layers after
        -- players leave/rejoin (e.g. boxFill over box outline, hpFill under hpBg, skeleton shadow
        -- over its line). Newest-served always on top; per-player call order is preserved verbatim.
        Lr.z = Lr.z + 1; inst.ZIndex = Lr.z
        inst.Visible = false
        local state = {}
        return setmetatable({}, {
            __index = function(_, k)
                if k == "Remove" then
                    return function()
                        inst.Visible = false
                        pool[#pool + 1] = { inst = inst, apply = applyFn }
                    end
                elseif k == "TextBounds" then
                    return inst.TextBounds
                end
                return state[k]
            end,
            __newindex = function(_, k, v)
                if state[k] == v then return end   -- skip-write: unchanged Vector2/UDim2/Color3 compare by value
                state[k] = v
                applyFn(state, k)
            end,
        })
    end
end

-- ── UTILS ──────────────────────────────────────────────────────────────────
local HEAD_PARTS    = { "HitboxHead", "HitboxHeadSmall", "Head" }
local TORSO_PARTS   = { "HitboxBody", "UpperTorso", "HumanoidRootPart", "LowerTorso" }
local CLOSEST_PARTS = {
    "HitboxHead","Head","UpperTorso","LowerTorso","HumanoidRootPart",
    "LeftHand","RightHand","LeftFoot","RightFoot",
    "LeftUpperArm","RightUpperArm","LeftUpperLeg","RightUpperLeg",
}

local function isTeammate(player)
    if player == lp then return true end
    local a, b = lp:GetAttribute("TeamID"), player:GetAttribute("TeamID")
    if a and b then return a == b end
    if lp.Team and player.Team and lp.Team == player.Team then return true end
    return false
end

local function isAlive(player)
    local c = player.Character
    local h = c and c:FindFirstChildOfClass("Humanoid")
    return h ~= nil and h.Health > 0
end

-- Reject NaN / absurd coordinates. Dying or ragdolling characters can fling
-- their parts to the float limit; if we resolve/teleport onto those we get
-- launched to oblivion and die "while hiding". Everything that aims at or
-- teleports to a world position must pass this first.
local SANE_POS_LIMIT = 100000
local function isSanePos(p)
    return p == p   -- NaN ~= NaN
        and math.abs(p.X) < SANE_POS_LIMIT
        and math.abs(p.Y) < SANE_POS_LIMIT
        and math.abs(p.Z) < SANE_POS_LIMIT
end

-- VOID SHELL (faithful port of Instance's snapVoid coords): a random integer in
-- ±2.1e9 but NEVER inside the ±1.147e9 deadzone, so every axis lands deep in the void
-- shell — far beyond any OOB volume and far from the map. Used for the velocity-spoofed
-- hide teleport (the huge matching velocity is what makes the server ACCEPT the jump).
local VFR_LIM, VFR_DEAD = 2147483646, 1147483646
local function rndSkip()
    local v
    repeat v = math.random(-VFR_LIM, VFR_LIM) until v < -VFR_DEAD or v > VFR_DEAD
    return v
end
local function voidShellVec()
    return Vector3.new(rndSkip(), rndSkip(), rndSkip())
end

-- Is a world point inside a part's (possibly rotated) box?
local function posInPart(pos, part)
    if not part or not part.Parent then return false end
    local lpv = part.CFrame:PointToObjectSpace(pos)
    local s = part.Size * 0.5
    return math.abs(lpv.X) <= s.X and math.abs(lpv.Y) <= s.Y and math.abs(lpv.Z) <= s.Z
end

-- True if a world position sits inside one of the map's OUT-OF-BOUNDS kill volumes
-- (replicates OutOfBoundsMachine/GameplayUtility:IsWithinOOBPart via CollectionService
-- WITHOUT requiring the heavy game module — requiring it at load broke the game). Safe
-- parts override. We must never PARK our replicated body in an OOB volume. The deep void
-- (Y=-1e11) is beyond all finite volumes so this is false — that's why the void survives.
local function posIsOOB(pos)
    local ok, result = pcall(function()
        for _, p in ipairs(CollectionService:GetTagged("OutOfBoundsSafePart")) do
            if posInPart(pos, p) then return false end
        end
        for _, p in ipairs(CollectionService:GetTagged("OutOfBoundsPart")) do
            if posInPart(pos, p) then return true end
        end
        return false
    end)
    return ok and result == true
end

-- Clamp a teleport target so the per-frame jump stays under the server's
-- anti-teleport threshold. Huge jumps (e.g. the old 2500-radius hide-orbit that
-- moved ~5000 studs/frame) get REJECTED by the server and never replicate, so
-- the "hide" was cosmetic. Small legal hops actually desync our server position.
local function clampHop(targetPos, fromPos, maxHop)
    local d = targetPos - fromPos
    local m = d.Magnitude
    if m <= maxHop or m == 0 then return targetPos end
    return fromPos + d * (maxHop / m)
end

-- IsProtected: raw check + 200ms TTL cache
local PROTECT_NAMES = {
    "shield","aura","forcefield","protect","immune","spawn","guard",
    "invuln","barrier","spawnprotection","protection",
}
local PROTECT_ATTRS = {
    "Immune","SpawnProtected","Invulnerable","Protected","Ghost",
    "SpawnProtect","Invincible","NoDamage","Untouchable",
    "InvulnerableUntil","ProtectionActive",
}
local _protCache = {}
local PROT_TTL   = 0.2

local function _isProtectedRaw(player)
    if not player or not player.Character then return false end
    local c = player.Character
    if c:FindFirstChildOfClass("ForceField") then return true end
    for _, d in ipairs(c:GetDescendants()) do
        if d:IsA("ForceField") then return true end
    end
    for _, attr in ipairs(PROTECT_ATTRS) do
        local v = c:GetAttribute(attr) or player:GetAttribute(attr)
        if v == true or (type(v) == "number" and v > tick()) then return true end
    end
    for _, child in ipairs(c:GetDescendants()) do
        if child:IsA("BasePart") or child:IsA("ParticleEmitter") then
            local n = child.Name:lower()
            for _, pat in ipairs(PROTECT_NAMES) do
                if n:find(pat, 1, true) then return true end
            end
        end
    end
    local sus = State.RageSuspectedProtection[player]
    if sus and tick() < sus then return true end
    return false
end

local function isProtected(player)
    local now    = tick()
    local cached = _protCache[player]
    if cached and (now - cached.t) < PROT_TTL then return cached.v end
    local result = _isProtectedRaw(player)
    _protCache[player] = { t = now, v = result }
    return result
end

local function getHealth(player)
    if not player.Character then return 0, 100 end
    local h = player.Character:FindFirstChildOfClass("Humanoid")
    if not h then return 0, 100 end
    return h.Health, h.MaxHealth
end

-- SIGNAL · shared 5-stop health ramp (red->orange->amber->lime->green), lerped in RGB.
-- HOISTED here from 10_esp (SIGNAL v4) so both the ESP HP bar/name underline AND the
-- 06_visuals FX target-info panel call the exact same ramp (no duplicated stops).
local HP_RAMP_STOPS = {
    { 0.00, Color3.fromRGB(255,  68,  54) },   -- #FF4436
    { 0.20, Color3.fromRGB(255, 122,  61) },   -- #FF7A3D
    { 0.40, Color3.fromRGB(255, 194,  75) },   -- #FFC24B
    { 0.60, Color3.fromRGB(196, 226,  78) },   -- #C4E24E
    { 1.00, Color3.fromRGB( 61, 224, 122) },   -- #3DE07A
}
local function hpRamp(frac)
    frac = math.clamp(frac or 0, 0, 1)
    for i = 1, #HP_RAMP_STOPS - 1 do
        local a, b = HP_RAMP_STOPS[i], HP_RAMP_STOPS[i + 1]
        if frac <= b[1] then
            local span = b[1] - a[1]
            local t = span > 0 and (frac - a[1]) / span or 0
            return a[2]:Lerp(b[2], math.clamp(t, 0, 1))
        end
    end
    return HP_RAMP_STOPS[#HP_RAMP_STOPS][2]
end

local function getWeaponName(player)
    if not player or not player.Character then return "?" end
    local ok, res = pcall(function()
        if Rivals.Ready and Rivals.Fighter and Rivals.Fighter._player_to_fighter then
            local f = Rivals.Fighter._player_to_fighter[player]
            if f and f.EquippedItem and f.EquippedItem.Info then
                return f.EquippedItem.Info.Name
            end
        end
        return nil
    end)
    if ok and type(res) == "string" and res ~= "" then return res end

    local ok2, fallback = pcall(function()
        for _, c in ipairs(player.Character:GetChildren()) do
            if c:IsA("Model") and not c:FindFirstChildOfClass("Humanoid") then
                if c.PrimaryPart or c:FindFirstChildWhichIsA("BasePart") then
                    return c.Name
                end
            end
            if c:IsA("Tool") then return c.Name end
        end
        return "?"
    end)
    return (ok2 and type(fallback) == "string") and fallback or "?"
end

local function pickPart(char, mode)
    if not char then return nil end
    if mode == "Closest" then
        local best, bestDist = nil, math.huge
        local vp     = Camera.ViewportSize
        local center = Vector2.new(vp.X * 0.5, vp.Y * 0.5)
        for _, name in ipairs(CLOSEST_PARTS) do
            local p = char:FindFirstChild(name)
            if p and p:IsA("BasePart") then
                local sp, on = Camera:WorldToViewportPoint(p.Position)
                if on and sp.Z > 0 then
                    local d = (Vector2.new(sp.X, sp.Y) - center).Magnitude
                    if d < bestDist then best, bestDist = p, d end
                end
            end
        end
        if best then return best end
    end
    local list = mode == "Torso" and TORSO_PARTS or HEAD_PARTS
    for _, name in ipairs(list) do
        local p = char:FindFirstChild(name)
        if p and p:IsA("BasePart") then return p end
    end
    return char:FindFirstChild("HumanoidRootPart")
end

local visParams = RaycastParams.new()
visParams.FilterType = Enum.RaycastFilterType.Exclude

local _visFilterChar = nil
local function isVisible(worldPos)
    local origin = Camera.CFrame.Position
    local _vc = lp.Character
    if _vc ~= _visFilterChar then
        visParams.FilterDescendantsInstances = { _vc }
        _visFilterChar = _vc
    end
    local result = Workspace:Raycast(origin, worldPos - origin, visParams)
    if not result then return true end
    local hitModel = result.Instance and result.Instance:FindFirstAncestorOfClass("Model")
    if hitModel and Players:GetPlayerFromCharacter(hitModel) then return true end
    return (result.Position - worldPos).Magnitude < 3
end

-- REAL Rivals deflect ANIMATION asset ids (from AnimationLibrary, the katana_*_deflect* set).
-- The old list here was wrong: it held item-MODEL ids (e.g. 118899310989170 = Keytana's model),
-- which never appear as a playing AnimationTrack, so the id branch was dead.
local DEFLECT_ANIM_IDS = {
    -- shared deflect anims: base Katana + Saber/Lightning Bolt/Pixel/Evil Trident/Stellar/Glorious/New Year/Linked Sword all map Deflect1-5 -> these
    ["14761240825"] = true, ["14761220206"] = true,                                                    -- deflect_loop, deflect_idle
    ["14761234917"] = true, ["14761221711"] = true, ["14761223422"] = true, ["14761225204"] = true, ["14761232380"] = true, -- deflect1-5
    -- Keytana
    ["90436105114997"] = true, ["90797895557136"] = true, ["77995180947430"] = true, ["111943779640553"] = true,
    -- Arch Katana
    ["131072510521727"] = true, ["132022220827223"] = true, ["116315405171252"] = true, ["110358509711635"] = true, ["98242486936084"] = true, ["81132288854196"] = true,
    -- Crystal Katana
    ["123293403148826"] = true, ["136354716301184"] = true, ["120567011479119"] = true, ["92502373956550"] = true, ["83541611040586"] = true, ["92773106977434"] = true,
    -- Cutlass, Riptide (deflect_idle ids differ from the shared base 14761220206)
    ["75844592081515"] = true, ["75381142568185"] = true,
}

-- FALLBACK deflect signal: animation-track scan. OR'd (union) with the module hook below —
-- it covers the windup (tracks play before _StartDeflecting fires) and any build where the
-- Katana module path moves. Never suppressed by the hook, so detection only ever gets better.
local function isDeflectingAnim(player)
    if not player or not player.Character then return false end
    local hum = player.Character:FindFirstChildOfClass("Humanoid")
    if not hum then return false end
    local animator = hum:FindFirstChildOfClass("Animator")
    if not animator then return false end
    for _, track in ipairs(animator:GetPlayingAnimationTracks()) do
        if track.Name:lower():find("deflect") then return true end
        local anim = track.Animation
        local id = anim and anim.AnimationId
        if id then
            local num = id:match("(%d+)")
            if num and DEFLECT_ANIM_IDS[num] then return true end
        end
    end
    return false
end

-- PRIMARY deflect signal (Elysium's, elisium20src :13522): hook the game's own
-- Katana._StartDeflecting — fires for EVERY fighter routed through the shared client
-- module, all skins, exact Info.DeflectDuration active window. Keyed by UserId (number)
-- so the key never depends on Instance-ref identity and no Instance is held. O(1) read.
local _deflecting = {}
local _deflGen    = {}
local DEFLECT_CLEAR_PAD = 0.05  -- err past the nominal window: one held shot beats one eaten shot

local function isDeflecting(player)
    if not player then return false end
    if _deflecting[player.UserId] then return true end
    return isDeflectingAnim(player)
end

;(function()
    local function recordDeflect(self)
        local fighter = self and self.ClientFighter
        local plr = fighter and fighter.Player
        if not plr then return end
        local uid = plr.UserId
        local dur = self.Info and self.Info.DeflectDuration
        if type(dur) ~= "number" then dur = 0.1 end
        _deflecting[uid] = true
        -- generation token: a re-deflect refreshes the window; the OLD timer can never
        -- stale-clear the NEW deflect (the overlapping-timer bug Elysium ships with).
        local gen = (_deflGen[uid] or 0) + 1
        _deflGen[uid] = gen
        task.delay(dur + DEFLECT_CLEAR_PAD, function()
            if _deflGen[uid] == gen then
                _deflecting[uid] = nil
            end
        end)
    end
    task.spawn(function()
        -- Route through the safe-load guard (bootstrap:190) — never a RAW require. Requiring a game
        -- ModuleScript before the game has can run+error it and get that error CACHED, bricking the
        -- weapon system forever ("no guns" crash). Matches how Rivals.Gun loads at bootstrap:199.
        local ok, katana = pcall(loadGameModule, lp.PlayerScripts, {"Modules", "Items", "Katana"})
        if not ok or type(katana) ~= "table" or type(katana._StartDeflecting) ~= "function" then
            return  -- path moved/absent: the anim fallback carries the feature, zero regression
        end
        -- re-execute safe: reuse a prior instance's saved TRUE original, never clone a stale wrapper.
        local orig = shared._LH_KatanaDeflOrig
        if not orig then orig = clonefunction(katana._StartDeflecting) end
        shared._LH_KatanaMod     = katana
        shared._LH_KatanaDeflOrig = orig
        if setreadonly then pcall(setreadonly, katana, false) end
        -- plain lclosure table-field reassign (same hook class as Gun.StartShooting): NO newcclosure
        -- (would flip iscclosure = readable anomaly), no setfenv needed. Recorder fully pcall-guarded
        -- so a field-shape change can never break the game's real deflect.
        katana._StartDeflecting = function(self, ...)
            pcall(recordDeflect, self)
            return orig(self, ...)
        end
    end)
end)()

local function isRiotShield(player)
    local w = getWeaponName(player):lower()
    return w:find("riot shield") or w:find("energy shield") or w:find("tombstone shield")
        or w:find("broken surfboard", 1, true) or w == "door" or w == "sled" or w == "masterpiece"
end

-- Katana CLASS = every deflect-capable melee. Many skins carry no "katana" in the
-- name (Saber, Lightning Bolt, Evil Trident, Linked Sword, Keytana), so a bare
-- substring check missed half of them and the anti-deflect flank never triggered.
-- Only the NON-"katana"-named skins need listing; every "* Katana" skin is caught by "katana".
-- Collision-free phrases only ("tridant" = Elysium's literal spelling; bare "trident"/"bolt" omitted).
local KATANA_NAMES = { "katana", "saber", "lightning bolt", "evil trident", "tridant", "devil's trident", "linked sword", "keytana", "cutlass", "swordfish", "riptide" }
local function isKatana(player)
    local w = getWeaponName(player):lower()
    for _, n in ipairs(KATANA_NAMES) do
        if w:find(n, 1, true) then return true end
    end
    -- Fallback: only katana-class weapons play deflect anims, so an active deflect
    -- proves the class even for a skin we haven't named yet.
    return isDeflecting(player)
end

local function isLocalKnife()
    local lf = Rivals.Fighter and Rivals.Fighter.LocalFighter
    if lf and lf.EquippedItem then
        local name = lf.EquippedItem.Name:lower()
        if name:find("knife") or name:find("karambit") or name:find("balisong") or name:find("chancla") or name:find("machete") or name:find("candy cane") or name:find("armature") or name:find("daggers") or name:find("axe") then
            return true
        end
    end
    return false
end

-- Enemy melee-knife detection (for Orbit distancing): a knifer can walk back during our fire and
-- backstab us if we sit close, so vs a knifer we keep to the larger orbit radius / void.
local KNIFE_NAMES = { "knife", "karambit", "balisong", "chancla", "machete", "candy cane", "armature", "daggers", "axe" }
local function isEnemyKnife(player)
    if not player then return false end
    local w = getWeaponName(player):lower()
    for _, n in ipairs(KNIFE_NAMES) do
        if w:find(n, 1, true) then return true end
    end
    return false
end

-- IN-MATCH detection for the rage lobby-gate. Evidence: the "rage teleports onto people in the lobby"
-- bug exists *because* the hub assigns no team (so isTeammate is false for everyone there). A match,
-- by contrast, puts us on a team. So an assigned team = we're in a match. A live combat Fighter is a
-- second positive signal. Default-safe: if neither holds we treat it as lobby and idle. The gate is
-- always-on (no player toggle); if a mode ever fails to set a team this stays lobby-safe. (Confirm live.)
local function inMatch()
    -- Lobby-disable is HARDCODED ON (no player toggle) — the in-match check ALWAYS runs so
    -- rage can never fire in the hub, even under a stale saved config. Returns true only in a match.
    if lp:GetAttribute("TeamID") ~= nil then return true end   -- match assigns a TeamID; hub does not
    if lp.Team ~= nil then return true end
    -- secondary: we hold a real combat Fighter with an equipped item (only inside a round)
    local lf = Rivals.Ready and Rivals.Fighter and Rivals.Fighter.LocalFighter
    if lf then
        local ok, objId = pcall(function() return lf.EquippedItem and lf.EquippedItem:Get("ObjectID") end)
        if ok and objId then return true end
    end
    return false
end

local function isValidTarget(player, checkVis, keepDeflect)
    if not player or player == lp then return false end
    if Config.TeamCheck and isTeammate(player) then return false end
    if not isAlive(player) then return false end
    -- keepDeflect: RAGE passes true (it FLANKS a deflecting katana via 07b flankPoint instead of
    -- dropping it); aim callers pass NOTHING -> drop while the deflect is active. Caller-scoped —
    -- never re-key this on Config.Rage (that global coupling was the bug: aim lost avoidance under rage).
    if Config.AvoidDeflect and not keepDeflect and isDeflecting(player) then return false end
    if Config.RageSkipImmune and isProtected(player) then return false end
    local char = player.Character
    local hrp  = char and char:FindFirstChild("HumanoidRootPart")
    if not hrp then return false end
    if not isSanePos(hrp.Position) then return false end  -- ragdoll flung to the void
    local myChar = lp.Character
    local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
    if myRoot and (hrp.Position - myRoot.Position).Magnitude > Config.MaxDistance then return false end
    if checkVis and not isVisible(hrp.Position) then return false end
    return true
end

-- ── SHARED VELOCITY TRACKER ────────────────────────────────────────────────
-- Finite-difference HRP velocity for EVERY player, ~30Hz, running whenever AIMBOT,
-- SILENT-AIM, or RAGE is enabled. This DECOUPLES calculateLead's resolver from
-- Config.Rage: the RageTrueVelocityMap only populates while the rage target loop
-- runs (07a startTargetLoop), so previously plain aim/silent users got NO real
-- finite-difference lead. This shared map fills for all three feature paths.
-- Re-execute safe: we disconnect any prior instance's tracker and rebind a fresh
-- one, so connections never stack (and the newest instance always owns it).
local _sharedVelMap = {}
do
    local _svPos, _svTime = {}, {}
    local _svAccum = 0
    local SV_INTERVAL = 1 / 30
    local function velTrackerStep(dt)
        if not (Config.Aimbot or Config.SilentAim or Config.Rage) then return end
        _svAccum = _svAccum + (dt or 0)
        if _svAccum < SV_INTERVAL then return end
        _svAccum = 0
        local now = tick()
        for _, p in ipairs(getSafePlayers() or {}) do
            if p ~= lp and p.Character then
                local hrp = p.Character:FindFirstChild("HumanoidRootPart")
                if hrp then
                    local pos = hrp.Position
                    local lt  = _svTime[p]
                    if _svPos[p] and lt then
                        local d = now - lt
                        if d > 0 then
                            local v = (pos - _svPos[p]) / d
                            -- Reject void-flicker garbage (astronomical displacement).
                            if v.Magnitude < 500 then _sharedVelMap[p] = v end
                        end
                    end
                    _svPos[p]  = pos
                    _svTime[p] = now
                end
            end
        end
    end
    if shared._LH_velConn then pcall(function() shared._LH_velConn:Disconnect() end) end
    shared._LH_velConn = RunService.Heartbeat:Connect(velTrackerStep)
end

-- calculateLead(targetChar, fromPos [, wantLead])
--   wantLead nil  => defaults to Config.PredictiveLead (silent-aim / rage encoder path)
--   wantLead true => force velocity lead (aimbot passes Config.AimbotPrediction here,
--                    fixing the old double-gate where AimbotPrediction was silently
--                    zeroed unless the shared PredictiveLead toggle was ALSO on).
local function calculateLead(targetChar, fromPos, wantLead)
    if wantLead == nil then wantLead = Config.PredictiveLead end
    if not targetChar then return Vector3.new() end
    local hum = targetChar:FindFirstChildOfClass("Humanoid")
    local hrp = targetChar:FindFirstChild("HumanoidRootPart")
    if not hum or not hrp then return Vector3.new() end
    -- Base latency = ping + server processing. Optional projectile TRAVEL-TIME lead
    -- adds distance/speed (for non-hitscan weapons); off by default (hitscan).
    local lat = (lp:GetNetworkPing() or 0.05) + ((Config.ServerProcessingMs or 30) / 1000)
    if Config.ProjectileLead and (Config.ProjectileSpeed or 0) > 0 then
        lat = lat + (hrp.Position - fromPos).Magnitude / Config.ProjectileSpeed
    end
    local lead = Vector3.new()
    if wantLead then
        -- True Velocity Resolver: prefer the SHARED finite-difference velocity (always
        -- fresh for aim/silent/rage), then the rage map, then the reported physics vel.
        local ply     = Players:GetPlayerFromCharacter(targetChar)
        local trueVel = hrp.AssemblyLinearVelocity
        local calcVel = ply and (_sharedVelMap[ply] or State.RageTrueVelocityMap[ply])
        if calcVel then
            -- Reported vs physical mismatch => velocity desync (spoof): trust the finite diff.
            if (trueVel - calcVel).Magnitude > 25 then trueVel = calcVel end
        else
            -- First-frame protection: an impossible frame-1 velocity defaults to 0.
            if trueVel.Magnitude > 100 then trueVel = Vector3.new() end
        end
        -- DESYNC GUARD: a void-flickering enemy pollutes the estimate with an astronomical
        -- speed. No real player exceeds ~120 studs/s => implausible speed = artifact => lead 0.
        if trueVel.Magnitude > 120 then trueVel = Vector3.new() end
        local md = hum.MoveDirection
        if md.Magnitude > 0.1 then
            lead = lead + Vector3.new(trueVel.X, 0, trueVel.Z) * lat
        end
        lead = lead + Vector3.new(0, trueVel.Y * lat, 0)
    end
    local cap = Config.LeadCap or 15
    if lead.Magnitude > cap then lead = lead.Unit * cap end
    return lead
end

local function selectTarget(opts)
    opts = opts or {}
    local fov         = opts.fov or 90
    local checkVis    = opts.checkVis or false
    local mode        = opts.partMode or "Head"
    local sticky      = opts.stickyTarget
    local stickyBonus = opts.stickyBonus or 0
    local vp     = Camera.ViewportSize
    local center = Vector2.new(vp.X * 0.5, vp.Y * 0.5)
    local best, bestPart, bestScore = nil, nil, math.huge
    for _, player in ipairs(getSafePlayers()) do
        -- Visibility is checked against the ACTUAL aim PART below (not the HRP), so pass
        -- false here to skip isValidTarget's HRP-based ray (it aimed at the wrong point).
        if player ~= lp and isValidTarget(player, false) then
            local char = player.Character
            local part = pickPart(char, mode)
            if part and (not checkVis or isVisible(part.Position)) then
                local sp, on = Camera:WorldToViewportPoint(part.Position)
                if on and sp.Z > 0 then
                    local d = (Vector2.new(sp.X, sp.Y) - center).Magnitude
                    if d <= fov then
                        local score = d
                        if player == sticky then score = score * (1 - stickyBonus) end
                        if score < bestScore then bestScore, best, bestPart = score, player, part end
                    end
                end
            end
        end
    end
    return best, bestPart
end

local function aimbotKeyDown()
    return isInputActive(Config.AimbotKey)
end

-- ── VISUALS ────────────────────────────────────────────────────────────────
local Visuals = {}
-- Visuals internals run in their own function proto (same pattern as 07a rage): keeps ~40 named
-- locals off the shared main-chunk register budget. `Visuals` (the export table) stays outer.
;(function()
    local _origLighting, _origClones = nil, {}
    local _hologramFolder, _hologramCooldowns = nil, {}
    local _stretchBound, _rainbowConn = false, nil
    local _rainbowParts, _rainbowHue, _rainbowBatchIdx = {}, 0, 1
    local _perfBackup, _origParticleRates = nil, {}
    local _reassertConn, _reassertLastT = nil, 0

    local LIGHTING_PROPS = {
        "Brightness","ExposureCompensation","GlobalShadows","ShadowSoftness",
        "EnvironmentDiffuseScale","EnvironmentSpecularScale","ClockTime",
        "OutdoorAmbient","Ambient","FogEnd","FogStart","FogColor",
        "ColorShift_Top","ColorShift_Bottom",
    }

    local function snapshotLighting()
        if _origLighting then return end
        _origLighting = {}
        for _, p in ipairs(LIGHTING_PROPS) do
            local ok, v = pcall(function() return Lighting[p] end)
            if ok then _origLighting[p] = v end
        end
        for _, c in ipairs(Lighting:GetChildren()) do
            if not c:GetAttribute("VS_Custom") then
                local ok, clone = pcall(function() return c:Clone() end)
                if ok and clone then table.insert(_origClones, clone) end
            end
        end
    end

    local function clearTagged()
        for _, c in ipairs(Lighting:GetChildren()) do
            if c:GetAttribute("VS_Custom") then c:Destroy() end
        end
    end

    local function restore()
        if not _origLighting then return end
        clearTagged()
        for k, v in pairs(_origLighting) do pcall(function() Lighting[k] = v end) end
        local exist = {}
        for _, c in ipairs(Lighting:GetChildren()) do exist[c.Name] = true end
        for _, clone in ipairs(_origClones) do
            if not exist[clone.Name] then clone:Clone().Parent = Lighting end
        end
    end

    local function fx(cls, props)
        local f = Instance.new(cls)
        f:SetAttribute("VS_Custom", true)
        for k, v in pairs(props) do f[k] = v end
        f.Parent = Lighting
        return f
    end

    local Presets = {}
    -- Presets Studio-verified 2026-07-14 on the Rivals lobby geometry. Each explicitly sets
    -- ExposureCompensation (0 when unused) so switching OFF an exposure-boosted preset can't leave a
    -- stale over-exposure. The night looks (Cyberpunk/Vaporwave/Toxic/Void) bathe surfaces in coloured
    -- ambient + bloom so they read as "neon-lit", not "dark" — the old versions were near-unplayable.
    Presets.Neutral = function()
        clearTagged()
        Lighting.Brightness = 2; Lighting.ExposureCompensation = 0
        Lighting.GlobalShadows = false; Lighting.ShadowSoftness = 0.2
        Lighting.EnvironmentDiffuseScale = 0.5; Lighting.EnvironmentSpecularScale = 0.5
        Lighting.ClockTime = 14; Lighting.OutdoorAmbient = Color3.fromRGB(70,70,70)
        Lighting.Ambient = Color3.fromRGB(0,0,0); Lighting.FogEnd = 100000
        fx("Atmosphere", { Density=0.3, Offset=0.25, Color=Color3.fromRGB(199,199,199),
            Decay=Color3.fromRGB(106,112,125), Glare=0, Haze=0 })
    end
    Presets.Cyberpunk = function()   -- neon-lit midnight: magenta bath + cyan atmosphere decay + strong bloom
        clearTagged()
        Lighting.Brightness = 2.6; Lighting.ExposureCompensation = 0.5
        Lighting.GlobalShadows = false; Lighting.ShadowSoftness = 0.7
        Lighting.EnvironmentDiffuseScale = 0.7; Lighting.EnvironmentSpecularScale = 1
        Lighting.ClockTime = 0; Lighting.OutdoorAmbient = Color3.fromRGB(120,80,165)
        Lighting.Ambient = Color3.fromRGB(80,55,120)
        fx("Atmosphere", { Density=0.3, Offset=0.3, Color=Color3.fromRGB(160,70,215),
            Decay=Color3.fromRGB(75,200,240), Glare=2.2, Haze=1 })
        fx("BloomEffect", { Intensity=1.15, Size=24, Threshold=0.72 })
        fx("ColorCorrectionEffect", { Brightness=0.04, Contrast=0.2, Saturation=0.45,
            TintColor=Color3.fromRGB(220,195,255) })
    end
    Presets.Anime = function()   -- vivid dreamy pastel day: broad soft bloom + gentle pink cast
        clearTagged()
        Lighting.Brightness = 2.3; Lighting.ExposureCompensation = 0.2
        Lighting.GlobalShadows = false; Lighting.ShadowSoftness = 0.7
        Lighting.EnvironmentDiffuseScale = 0.7; Lighting.EnvironmentSpecularScale = 0.7
        Lighting.ClockTime = 15; Lighting.OutdoorAmbient = Color3.fromRGB(150,140,170)
        Lighting.Ambient = Color3.fromRGB(95,85,120)
        fx("Atmosphere", { Density=0.28, Offset=0.35, Color=Color3.fromRGB(255,200,230),
            Decay=Color3.fromRGB(150,195,255), Glare=1, Haze=0.8 })
        fx("BloomEffect", { Intensity=1.0, Size=26, Threshold=0.8 })
        fx("ColorCorrectionEffect", { Brightness=0.03, Contrast=0.14, Saturation=0.32,
            TintColor=Color3.fromRGB(255,228,242) })
    end
    Presets.Sunset = function()   -- golden hour: sun near the horizon (17.75) reddens the sky, high exposure keeps it luminous
        clearTagged()
        Lighting.Brightness = 2.3; Lighting.ExposureCompensation = 0.4
        Lighting.GlobalShadows = false; Lighting.ShadowSoftness = 0.5
        Lighting.EnvironmentDiffuseScale = 0.75; Lighting.EnvironmentSpecularScale = 0.9
        Lighting.ClockTime = 17.75; Lighting.OutdoorAmbient = Color3.fromRGB(185,115,80)
        Lighting.Ambient = Color3.fromRGB(105,60,45)
        fx("Atmosphere", { Density=0.38, Offset=0.55, Color=Color3.fromRGB(255,150,80),
            Decay=Color3.fromRGB(255,105,60), Glare=1.8, Haze=1.6 })
        fx("BloomEffect", { Intensity=0.9, Size=24, Threshold=0.78 })
        fx("ColorCorrectionEffect", { Brightness=0.03, Contrast=0.16, Saturation=0.32,
            TintColor=Color3.fromRGB(255,195,150) })
    end
    Presets.Vaporwave = function()   -- dreamy pastel-neon twilight: pink + cyan gradient sky, luminous
        clearTagged()
        Lighting.Brightness = 2.3; Lighting.ExposureCompensation = 0.4
        Lighting.GlobalShadows = false; Lighting.ShadowSoftness = 0.7
        Lighting.EnvironmentDiffuseScale = 0.6; Lighting.EnvironmentSpecularScale = 0.9
        Lighting.ClockTime = 18.4; Lighting.OutdoorAmbient = Color3.fromRGB(150,90,165)
        Lighting.Ambient = Color3.fromRGB(95,60,120)
        fx("Atmosphere", { Density=0.34, Offset=0.4, Color=Color3.fromRGB(255,130,205),
            Decay=Color3.fromRGB(110,200,255), Glare=1.8, Haze=1.3 })
        fx("BloomEffect", { Intensity=1.05, Size=26, Threshold=0.74 })
        fx("ColorCorrectionEffect", { Brightness=0.04, Contrast=0.18, Saturation=0.38,
            TintColor=Color3.fromRGB(255,205,240) })
    end
    Presets.Void = function()   -- eerie cold-blue: readable but moody, dark starfield sky
        clearTagged()
        Lighting.Brightness = 2.0; Lighting.ExposureCompensation = 0.25
        Lighting.GlobalShadows = false; Lighting.ShadowSoftness = 0.9
        Lighting.EnvironmentDiffuseScale = 0.5; Lighting.EnvironmentSpecularScale = 0.7
        Lighting.ClockTime = 0; Lighting.OutdoorAmbient = Color3.fromRGB(85,95,135)
        Lighting.Ambient = Color3.fromRGB(55,62,95)
        fx("Atmosphere", { Density=0.35, Offset=0.2, Color=Color3.fromRGB(55,65,110),
            Decay=Color3.fromRGB(95,110,180), Glare=0.3, Haze=0.8 })
        fx("BloomEffect", { Intensity=0.8, Size=22, Threshold=0.76 })
        fx("ColorCorrectionEffect", { Brightness=0.03, Contrast=0.16, Saturation=-0.2,
            TintColor=Color3.fromRGB(190,200,255) })
    end

    Presets.Clarity = function()   -- flat max-visibility competitive look (intentionally anti-cinematic)
        clearTagged()
        Lighting.Brightness = 2.6; Lighting.ExposureCompensation = 0
        Lighting.GlobalShadows = false; Lighting.ShadowSoftness = 1
        Lighting.EnvironmentDiffuseScale = 0.2; Lighting.EnvironmentSpecularScale = 0.1
        Lighting.ClockTime = 14; Lighting.OutdoorAmbient = Color3.fromRGB(150,150,155)
        Lighting.Ambient = Color3.fromRGB(120,120,125); Lighting.FogEnd = 1000000
        fx("ColorCorrectionEffect", { Brightness=0.05, Contrast=0.25, Saturation=-0.2,
            TintColor=Color3.fromRGB(255,255,255) })
    end
    Presets.Toxic = function()   -- glowing radioactive-green haze: sickly, luminous, readable
        clearTagged()
        Lighting.Brightness = 2.2; Lighting.ExposureCompensation = 0.35
        Lighting.GlobalShadows = false; Lighting.ShadowSoftness = 0.7
        Lighting.EnvironmentDiffuseScale = 0.6; Lighting.EnvironmentSpecularScale = 0.8
        Lighting.ClockTime = 1; Lighting.OutdoorAmbient = Color3.fromRGB(80,135,70)
        Lighting.Ambient = Color3.fromRGB(45,85,50)
        fx("Atmosphere", { Density=0.34, Offset=0.35, Color=Color3.fromRGB(95,220,110),
            Decay=Color3.fromRGB(55,180,80), Glare=1.8, Haze=1.4 })
        fx("BloomEffect", { Intensity=1.1, Size=24, Threshold=0.74 })
        fx("ColorCorrectionEffect", { Brightness=0.04, Contrast=0.2, Saturation=0.45,
            TintColor=Color3.fromRGB(210,255,205) })
    end
    Presets.Sakura = function()   -- dreamy pink day: soft rose bath + broad bloom (pairs with Petals). Studio-verified 2026-07-14
        clearTagged()
        Lighting.Brightness = 2.2; Lighting.ExposureCompensation = 0.35
        Lighting.GlobalShadows = false; Lighting.ShadowSoftness = 0.8
        Lighting.EnvironmentDiffuseScale = 0.7; Lighting.EnvironmentSpecularScale = 0.7
        Lighting.ClockTime = 15.5; Lighting.OutdoorAmbient = Color3.fromRGB(200,155,180)
        Lighting.Ambient = Color3.fromRGB(120,85,110)
        fx("Atmosphere", { Density=0.3, Offset=0.4, Color=Color3.fromRGB(255,205,225),
            Decay=Color3.fromRGB(255,175,215), Glare=1.2, Haze=1 })
        fx("BloomEffect", { Intensity=1.1, Size=26, Threshold=0.78 })
        fx("ColorCorrectionEffect", { Brightness=0.04, Contrast=0.15, Saturation=0.28,
            TintColor=Color3.fromRGB(255,225,240) })
    end
    Presets.Nebula = function()   -- starlit deep-space night: rich violet bath over a starfield (pairs with Shooting Stars). Studio-verified 2026-07-14
        clearTagged()
        Lighting.Brightness = 2.2; Lighting.ExposureCompensation = 0.4
        Lighting.GlobalShadows = false; Lighting.ShadowSoftness = 0.9
        Lighting.EnvironmentDiffuseScale = 0.55; Lighting.EnvironmentSpecularScale = 0.85
        Lighting.ClockTime = 0; Lighting.OutdoorAmbient = Color3.fromRGB(128,94,168)   -- nudged up from preview for enemy readability
        Lighting.Ambient = Color3.fromRGB(82,60,120)
        fx("Atmosphere", { Density=0.34, Offset=0.25, Color=Color3.fromRGB(120,70,180),
            Decay=Color3.fromRGB(220,90,190), Glare=1.4, Haze=1.1 })
        fx("BloomEffect", { Intensity=1.15, Size=24, Threshold=0.72 })
        fx("ColorCorrectionEffect", { Brightness=0.03, Contrast=0.2, Saturation=0.4,
            TintColor=Color3.fromRGB(235,205,255) })
    end

    -- Scalar-only snapshot for each preset — re-asserted by the throttled loop without
    -- recreating fx instances (no flicker, safe to run every second).
    -- Mirrors the scalar half of each Presets.* fn (values match the Studio-verified 2026-07-14 looks).
    -- The 1Hz reassert re-writes only-when-drifted, so ExposureCompensation is carried here too — else a
    -- game lighting recompute could silently drop a preset's exposure and mute the look.
    local PresetScalars = {
        Neutral   = { Brightness=2,   ExposureCompensation=0,    ClockTime=14,   OutdoorAmbient=Color3.fromRGB(70,70,70),
                      Ambient=Color3.fromRGB(0,0,0),      FogEnd=100000,
                      EnvironmentDiffuseScale=0.5,  EnvironmentSpecularScale=0.5 },
        Clarity   = { Brightness=2.6, ExposureCompensation=0,    ClockTime=14,   OutdoorAmbient=Color3.fromRGB(150,150,155),
                      Ambient=Color3.fromRGB(120,120,125), FogEnd=1000000,
                      EnvironmentDiffuseScale=0.2,  EnvironmentSpecularScale=0.1 },
        Cyberpunk = { Brightness=2.6, ExposureCompensation=0.5,  ClockTime=0,    OutdoorAmbient=Color3.fromRGB(120,80,165),
                      Ambient=Color3.fromRGB(80,55,120),
                      EnvironmentDiffuseScale=0.7,  EnvironmentSpecularScale=1 },
        Anime     = { Brightness=2.3, ExposureCompensation=0.2,  ClockTime=15,   OutdoorAmbient=Color3.fromRGB(150,140,170),
                      Ambient=Color3.fromRGB(95,85,120),
                      EnvironmentDiffuseScale=0.7,  EnvironmentSpecularScale=0.7 },
        Sunset    = { Brightness=2.3, ExposureCompensation=0.4,  ClockTime=17.75, OutdoorAmbient=Color3.fromRGB(185,115,80),
                      Ambient=Color3.fromRGB(105,60,45),
                      EnvironmentDiffuseScale=0.75, EnvironmentSpecularScale=0.9 },
        Vaporwave = { Brightness=2.3, ExposureCompensation=0.4,  ClockTime=18.4, OutdoorAmbient=Color3.fromRGB(150,90,165),
                      Ambient=Color3.fromRGB(95,60,120),
                      EnvironmentDiffuseScale=0.6,  EnvironmentSpecularScale=0.9 },
        Toxic     = { Brightness=2.2, ExposureCompensation=0.35, ClockTime=1,    OutdoorAmbient=Color3.fromRGB(80,135,70),
                      Ambient=Color3.fromRGB(45,85,50),
                      EnvironmentDiffuseScale=0.6,  EnvironmentSpecularScale=0.8 },
        Void      = { Brightness=2.0, ExposureCompensation=0.25, ClockTime=0,    OutdoorAmbient=Color3.fromRGB(85,95,135),
                      Ambient=Color3.fromRGB(55,62,95),
                      EnvironmentDiffuseScale=0.5,  EnvironmentSpecularScale=0.7 },
        Sakura    = { Brightness=2.2, ExposureCompensation=0.35, ClockTime=15.5, OutdoorAmbient=Color3.fromRGB(200,155,180),
                      Ambient=Color3.fromRGB(120,85,110),
                      EnvironmentDiffuseScale=0.7,  EnvironmentSpecularScale=0.7 },
        Nebula    = { Brightness=2.2, ExposureCompensation=0.4,  ClockTime=0,    OutdoorAmbient=Color3.fromRGB(128,94,168),
                      Ambient=Color3.fromRGB(82,60,120),
                      EnvironmentDiffuseScale=0.55, EnvironmentSpecularScale=0.85 },
    }

    -- Fullbright / No-fog are drift-guarded overrides layered ON TOP of the active preset.
    -- They write scalar Lighting props only (same cheap pattern as reassertScalars) so the
    -- 1Hz loop keeps them alive even when a preset or the game engine tries to fight them.
    local _WHITE = Color3.new(1, 1, 1)
    local function applyFullbrightOverride()
        if not Config.VisualsFullbright then return end
        pcall(function() if Lighting.Ambient ~= _WHITE then Lighting.Ambient = _WHITE end end)
        pcall(function() if Lighting.OutdoorAmbient ~= _WHITE then Lighting.OutdoorAmbient = _WHITE end end)
        pcall(function() if Lighting.GlobalShadows ~= false then Lighting.GlobalShadows = false end end)
        pcall(function() if Lighting.Brightness < 2 then Lighting.Brightness = 2 end end)
    end
    local function applyFogOverride()
        if not Config.VisualsNoFog then return end
        pcall(function() if Lighting.FogEnd ~= 1e6 then Lighting.FogEnd = 1e6 end end)
        pcall(function() if Lighting.FogStart ~= 1e6 then Lighting.FogStart = 1e6 end end)
        -- neutralize any preset's custom Atmosphere haze (it reads as fog)
        for _, c in ipairs(Lighting:GetChildren()) do
            if c:IsA("Atmosphere") then
                pcall(function() if c.Density ~= 0 then c.Density = 0 end end)
            end
        end
    end

    -- Config read with a default (new keys may not exist in 02_config yet — nil ⇒ default).
    local function cfg(key, default)
        local v = Config[key]
        if v == nil then return default end
        return v
    end

    -- ── COLOUR GRADE — ONE ColorCorrection composed OVER the active preset, kept alive by the
    -- same drift-guard pattern as reassertScalars (write only-when-drifted, ~1Hz). Tagged with its
    -- OWN attribute (not VS_Custom) so clearTagged() on a preset switch can't destroy it.
    -- Studio-verified 2026-07-14 (composited over Neutral, checked to strength 1.0). Each grade has a
    -- distinct filmic identity (the old set was too timid — Crisp C=0.08 read as no-op): Crisp = clean
    -- punch, Cold = icy desaturated blue, Warm = golden, Comp = high-contrast cinematic.
    local _GRADES = {
        Crisp = { B = 0.03,  C = 0.20, S = 0.18,  tint = _WHITE },
        Cold  = { B = -0.03, C = 0.28, S = -0.22, tint = Color3.fromRGB(196, 220, 255) },
        Warm  = { B = 0.04,  C = 0.22, S = 0.15,  tint = Color3.fromRGB(255, 222, 180) },
        Comp  = { B = -0.01, C = 0.40, S = 0.28,  tint = Color3.fromRGB(255, 248, 236) },
    }
    local _gradeFx = nil

    local function getGradeFx()
        if _gradeFx and _gradeFx.Parent then return _gradeFx end
        local cc = Instance.new("ColorCorrectionEffect")
        cc.Name = "_vs_grade"
        cc:SetAttribute("VS_Grade", true)
        cc.Parent = Lighting
        _gradeFx = cc
        return cc
    end

    local function reassertGrade()
        local g = _GRADES[cfg("VisualsGrade", "None")]
        if not g then   -- "None" (or unknown) => no grade; just make sure the handle is inert
            if _gradeFx and _gradeFx.Parent then
                pcall(function() if _gradeFx.Enabled then _gradeFx.Enabled = false end end)
            end
            return
        end
        -- one strength dial: lerp B/C/S toward 0 and Tint toward white by (1 - strength)
        local s  = math.clamp(cfg("VisualsGradeStrength", 1), 0, 1)
        local tB, tC, tS = g.B * s, g.C * s, g.S * s
        local tT = g.tint:Lerp(_WHITE, 1 - s)
        local cc = getGradeFx()
        pcall(function()
            if not cc.Enabled then cc.Enabled = true end
            if math.abs(cc.Brightness - tB) > 0.001 then cc.Brightness = tB end
            if math.abs(cc.Contrast   - tC) > 0.001 then cc.Contrast   = tC end
            if math.abs(cc.Saturation - tS) > 0.001 then cc.Saturation = tS end
            if cc.TintColor ~= tT then cc.TintColor = tT end
        end)
    end

    local function clearGrade()
        if _gradeFx then pcall(function() _gradeFx:Destroy() end); _gradeFx = nil end
    end

    -- ── BLOOM MASTER — one user-driven BloomEffect layered OVER the active preset, kept alive by the
    -- same drift-guard as the grade (write only-when-drifted, ~1Hz). Own attribute (VS_Bloom) so a
    -- preset switch's clearTagged() can't destroy it. Default OFF; the intensity dial (0–3) is the one
    -- exposed knob (Size/Threshold fixed at a broad, filmic sweet spot). Stacks with a preset's own
    -- bloom by design — that's the "extra glow" master. Verified 2026-07-14 (Sakura/Nebula previews).
    local _bloomFx = nil
    local function getBloomFx()
        if _bloomFx and _bloomFx.Parent then return _bloomFx end
        local b = Instance.new("BloomEffect")
        b.Name = "_vs_bloom"; b:SetAttribute("VS_Bloom", true)
        b.Size = 24; b.Threshold = 0.8; b.Intensity = 0
        b.Parent = Lighting
        _bloomFx = b
        return b
    end
    local function reassertBloom()
        if not cfg("VisualsBloom", false) then
            if _bloomFx and _bloomFx.Parent then
                pcall(function() if _bloomFx.Enabled then _bloomFx.Enabled = false end end)
            end
            return
        end
        local tI = math.clamp(cfg("VisualsBloomIntensity", 1), 0, 3)
        local b = getBloomFx()
        pcall(function()
            if not b.Enabled then b.Enabled = true end
            if math.abs(b.Intensity - tI) > 0.01 then b.Intensity = tI end
        end)
    end
    local function clearBloom()
        if _bloomFx then pcall(function() _bloomFx:Destroy() end); _bloomFx = nil end
    end

    local function reassertScalars()
        local sc = PresetScalars[State.VisualsCurrentPreset or Config.VisualsPreset]
        if sc then
            -- only write when the game actually drifted a value — redundant writes
            -- (esp. ClockTime/Ambient) can force a lighting recompute = a periodic hitch.
            for k, v in pairs(sc) do pcall(function() if Lighting[k] ~= v then Lighting[k] = v end end) end
            pcall(function() if Lighting.GlobalShadows ~= false then Lighting.GlobalShadows = false end end)
        end
        applyFullbrightOverride()
        applyFogOverride()
        reassertGrade()
        reassertBloom()
    end

    local function startReassert()
        if _reassertConn then return end
        _reassertConn = RunService.Heartbeat:Connect(function()
            if not Config.Visuals or Config.VisualsPerformanceMode then return end
            local now = tick()
            if (now - _reassertLastT) < 1.0 then return end
            _reassertLastT = now
            reassertScalars()
        end)
    end

    local function stopReassert()
        if _reassertConn then _reassertConn:Disconnect(); _reassertConn = nil end
    end

    Visuals.PresetOrder = { "Neutral", "Clarity", "Cyberpunk", "Anime", "Sunset", "Vaporwave", "Toxic", "Void", "Sakura", "Nebula" }

    local function applyPreset(name)
        if not Config.Visuals or Config.VisualsPerformanceMode then return end
        local fn = Presets[name]; if not fn then return end
        pcall(fn); State.VisualsCurrentPreset = name; Config.VisualsPreset = name
        reassertGrade()   -- a preset switch (clearTagged) must not drop the grade
        reassertBloom()   -- ...or the master bloom
    end

    local function getHoloFolder()
        if _hologramFolder and _hologramFolder.Parent then return _hologramFolder end
        local f = Instance.new("Folder"); f.Name = "_vs_holos"; f.Parent = Workspace
        _hologramFolder = f; return f
    end

    -- ── AFTERIMAGE STYLES (Config.VisualsHologramStyle: Orb | Skeleton | Wraith) ──────────────
    local _GOLD = Color3.fromRGB(255, 200, 60)
    -- transient-motion / secondary-FX tokens (design §1.1): EDGE = "something just
    -- happened" accents (Prism beam bow, TriDot spin), VISIBLE = LOS-true cyan (Prism beam bow).
    local _EDGE    = Color3.fromRGB(155, 232, 255)   -- #9BE8FF
    local _VISIBLE = Color3.fromRGB(41, 224, 255)    -- #29E0FF
    local function easeInOut(a) return a * a * (3 - 2 * a) end

    -- Mirrors the ESP skeleton's rig maps (SKEL_R15/SKEL_R6 in the ESP module are do-block locals,
    -- so we keep our own copies here — same name pairs).
    local HOLO_SKEL_R15 = {
        {"Head","UpperTorso"},{"UpperTorso","LowerTorso"},
        {"UpperTorso","LeftUpperArm"},{"LeftUpperArm","LeftLowerArm"},{"LeftLowerArm","LeftHand"},
        {"UpperTorso","RightUpperArm"},{"RightUpperArm","RightLowerArm"},{"RightLowerArm","RightHand"},
        {"LowerTorso","LeftUpperLeg"},{"LeftUpperLeg","LeftLowerLeg"},{"LeftLowerLeg","LeftFoot"},
        {"LowerTorso","RightUpperLeg"},{"RightUpperLeg","RightLowerLeg"},{"RightLowerLeg","RightFoot"},
    }
    local HOLO_SKEL_R6 = {
        {"Head","Torso"},
        {"Torso","Left Arm"},{"Torso","Right Arm"},
        {"Torso","Left Leg"},{"Torso","Right Leg"},
    }

    -- One bounded Heartbeat fading a list of parts tr0 -> 1 ease-in-out (shared by Skeleton/Wraith).
    -- tr0 = start transparency (default 0.55 = old look; the visibility dial lowers it so holos pop).
    local function fadePop(container, parts, hl, dur, tr0)
        tr0 = tr0 or 0.55
        local startT = tick()
        local conn
        conn = RunService.Heartbeat:Connect(function()
            if not container.Parent then if conn then conn:Disconnect() end return end
            local a  = math.clamp((tick() - startT) / dur, 0, 1)
            local e  = easeInOut(a)
            local tr = tr0 + (1 - tr0) * e
            for _, b in ipairs(parts) do b.Transparency = tr end
            if hl then hl.OutlineTransparency = e end
            if a >= 1 and conn then conn:Disconnect() end
        end)
        Debris:AddItem(container, dur + 0.2)
    end

    -- Skeleton: thin neon cylinders along the bone pairs at hit-pose (no Highlight). All bones live
    -- in ONE Model so the folder's 16-cap counts afterimages, not individual parts.
    local function popSkeleton(char, dur, color)
        local vis = math.clamp(cfg("VisualsHologramVisibility", 1.4), 0.2, 2)
        local tr0 = math.clamp(1 - 0.45 * vis, 0, 0.91)   -- vis 1.0->0.55 (old), 1.4->0.37, 2.0->0.10
        local hum  = char:FindFirstChildOfClass("Humanoid")
        local isR6 = (hum and hum.RigType == Enum.HumanoidRigType.R6) or (char:FindFirstChild("Torso") ~= nil)
        local rig  = isR6 and HOLO_SKEL_R6 or HOLO_SKEL_R15
        local model = Instance.new("Model")
        model.Name = "_hs" .. math.random(10000, 99999)
        local parts, n = {}, 0
        for _, pair in ipairs(rig) do
            local a, b = char:FindFirstChild(pair[1]), char:FindFirstChild(pair[2])
            if a and b then
                local ap, bp = a.Position, b.Position
                local len = (bp - ap).Magnitude
                if len > 0.05 and len < 20 then
                    local bone = Instance.new("Part")
                    bone.Shape = Enum.PartType.Cylinder
                    bone.Size = Vector3.new(len, 0.1 + 0.06 * vis, 0.1 + 0.06 * vis)   -- vis 1.4 -> 0.18-stud bones
                    bone.Material = Enum.Material.Neon
                    bone.Color = color
                    bone.Transparency = tr0
                    bone.Anchored = true; bone.CanCollide = false; bone.CanQuery = false
                    bone.CanTouch = false; bone.CastShadow = false; bone.Massless = true
                    bone:SetAttribute("VS_Holo", true)
                    -- cylinder long axis is X: aim X down the bone
                    bone.CFrame = CFrame.lookAt((ap + bp) * 0.5, bp) * CFrame.Angles(0, math.rad(90), 0)
                    bone.Parent = model
                    n = n + 1; parts[n] = bone
                end
            end
        end
        if n == 0 then model:Destroy(); return end
        model.Parent = getHoloFolder()
        fadePop(model, parts, nil, dur, tr0)
    end

    -- Wraith: full rig clone frozen at hit-pose, ForceField material @0.55 + ONE white-outline
    -- Highlight. Hard-cap <=4 concurrent (ForceField-only fallback if the Highlight fails) so we
    -- can never contribute past Roblox's 31-Highlight budget.
    local _wraiths = {}
    local function wraithCount()
        local n = 0
        for i = #_wraiths, 1, -1 do
            local m = _wraiths[i]
            if m and m.Parent then n = n + 1 else table.remove(_wraiths, i) end
        end
        return n
    end

    local STRIP_CLASSES = {
        "Humanoid","Sound","ParticleEmitter","Trail","Beam","Fire","Smoke","Sparkles",
        "ForceField","Highlight","BillboardGui","SurfaceGui","BaseScript",
    }
    local function popWraith(char, dur, color)
        if wraithCount() >= 4 then return end   -- hard concurrency cap
        local vis = math.clamp(cfg("VisualsHologramVisibility", 1.4), 0.2, 2)
        local tr0 = math.clamp(1 - 0.45 * vis, 0, 0.91)   -- vis 1.0->0.55 (old), 1.4->0.37, 2.0->0.10
        local clone
        pcall(function()
            local was = char.Archivable
            char.Archivable = true
            clone = char:Clone()
            char.Archivable = was
        end)
        if not clone then return end
        clone.Name = "_hw" .. math.random(10000, 99999)
        local parts, n = {}, 0
        for _, d in ipairs(clone:GetDescendants()) do
            local strip = false
            for _, cls in ipairs(STRIP_CLASSES) do
                if d:IsA(cls) then strip = true break end
            end
            if strip then
                pcall(function() d:Destroy() end)
            elseif d:IsA("BasePart") then
                d.Anchored = true; d.CanCollide = false; d.CanQuery = false
                d.CanTouch = false; d.CastShadow = false; d.Massless = true
                d:SetAttribute("VS_Holo", true)
                if d.Transparency < 0.98 then
                    d.Material = Enum.Material.ForceField
                    d.Color = color
                    d.Transparency = tr0
                    n = n + 1; parts[n] = d
                else
                    d.Transparency = 1   -- keep invisible rig parts (HRP) invisible
                end
            end
        end
        if n == 0 then clone:Destroy(); return end
        clone:SetAttribute("VS_Holo", true)
        clone.Parent = getHoloFolder()
        local hl
        pcall(function()   -- ForceField-only fallback: a failed Highlight never kills the wraith
            local h = Instance.new("Highlight")
            h.FillTransparency = 1
            h.OutlineColor = _WHITE
            h.OutlineTransparency = 0
            h.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
            h.Adornee = clone
            h.Parent = clone
            hl = h
        end)
        table.insert(_wraiths, clone)
        fadePop(clone, parts, hl, dur, tr0)
    end

    -- LIGHT glowing-blue dot at the hit point (replaces the old full-character clone, which was the
    -- "Silent Aim drops FPS when I fire" culprit). Two anchored neon Balls (a bright core + a soft
    -- halo) that rise a little and fade out, then self-destruct. One bounded Heartbeat per pop;
    -- ~free vs cloning ~15 parts + a Highlight every shot.
    local function createHologram(character, lethal)
        if not character or not character.Parent then return end
        local rp = character:FindFirstChild("HitboxHead")
            or character:FindFirstChild("Head")
            or character:FindFirstChild("HumanoidRootPart")
            or character:FindFirstChild("UpperTorso")
        if not rp then return end
        if (rp.Position - Camera.CFrame.Position).Magnitude > Config.VisualsHologramRange then return end
        -- cap concurrent afterimages so a long fight can't accumulate parts (still trivially cheap)
        if #getHoloFolder():GetChildren() >= 16 then return end
        local dur = math.clamp(Config.VisualsHologramDuration or 3.5, 0.25, 10)  -- honors the 0.5–10 slider
        -- gold core on a killing blow (opt-out via VisualsHologramLethal)
        local lethalOn  = lethal and cfg("VisualsHologramLethal", true)
        local mainColor = lethalOn and cfg("VisualsHologramLethalColor", _GOLD) or Config.VisualsHologramColor
        local style = cfg("VisualsHologramStyle", "Orb")
        -- Wraith is the expensive style (rig clone + Highlight) — fall to Orb under Performance mode
        if Config.VisualsPerformanceMode and style == "Wraith" then style = "Orb" end
        if style == "Skeleton" then popSkeleton(character, dur, mainColor); return end
        if style == "Wraith"   then popWraith(character, dur, mainColor);   return end
        -- Orb (default): the original 2-ball pop
        local folder = getHoloFolder()
        local function mkBall(size, transp, color)
            local b = Instance.new("Part")
            b.Shape = Enum.PartType.Ball
            b.Size = Vector3.new(size, size, size)
            b.Material = Enum.Material.Neon
            b.Color = color
            b.Transparency = transp
            b.Anchored = true; b.CanCollide = false; b.CanQuery = false
            b.CanTouch = false; b.CastShadow = false; b.Massless = true
            b:SetAttribute("VS_Holo", true)
            return b
        end
        -- visuals-v3 §02: cyan core + CYAN halo (both = mainColor; gold on a lethal blow). The accent
        -- (#FF3CC8 pink) lives in the one-shot spawn shockwave below — NOT painted onto the persistent halo.
        local vis = math.clamp(cfg("VisualsHologramVisibility", 1.4), 0.2, 2)
        local sc  = 0.75 + 0.25 * vis           -- vis 1.4 -> x1.1, vis 2 -> x1.25
        local h0  = math.clamp(0.65 / vis, 0.1, 0.9)   -- halo start transparency (brighter as vis rises)
        local core = mkBall(0.7 * sc, 0.05, mainColor)
        local halo = mkBall(1.7 * sc, h0, mainColor)
        local startCF = CFrame.new(rp.Position)
        core.CFrame = startCF; halo.CFrame = startCF
        core.Name = "_h" .. math.random(10000, 99999); halo.Name = core.Name .. "_g"
        core.Parent = folder; halo.Parent = folder
        -- one-shot ACCENT shockwave ring on spawn — the "pop" flourish that sells the marker (§02). A flat
        -- Neon Cylinder disc FACING THE CAMERA (same primitive as the §07 spark rings; that helper is out of
        -- lexical scope up here, so it's inlined) that grows ~0.5->4 studs quad-out over 0.3s then fades.
        local shockColor = Config.VisualsHologramAccent or Config.VisualsHologramColor
        local shock = mkBall(0.5, 0.15, shockColor)
        shock.Shape = Enum.PartType.Cylinder
        shock.Size  = Vector3.new(0.1, 0.5, 0.5)
        do
            local cam = workspace.CurrentCamera
            if cam then shock.CFrame = CFrame.lookAt(rp.Position, cam.CFrame.Position) * CFrame.Angles(0, math.rad(90), 0)
            else shock.CFrame = startCF * CFrame.Angles(0, 0, math.rad(90)) end
        end
        shock.Name = core.Name .. "_s"
        shock.Parent = folder
        local startT = tick()
        local conn
        conn = RunService.Heartbeat:Connect(function()
            if not core.Parent then if conn then conn:Disconnect() end return end
            local alpha = math.clamp((tick() - startT) / dur, 0, 1)
            local rise  = 2.5 * (1 - (1 - alpha) * (1 - alpha))     -- eased rise
            local cf    = startCF + Vector3.new(0, rise, 0)
            core.CFrame = cf; halo.CFrame = cf
            core.Transparency = math.clamp(0.05 + 0.95 * alpha, 0, 1)
            halo.Transparency = math.clamp(h0 + (1 - h0) * alpha, 0, 1)
            -- shockwave: 0.5 -> 4 stud over 0.3s quad-out, then fully faded (Debris reclaims it at 0.5s)
            if shock.Parent then
                local sa = math.clamp((tick() - startT) / 0.3, 0, 1)
                local se = 1 - (1 - sa) * (1 - sa)
                local sd = 0.5 + 3.5 * se
                shock.Size = Vector3.new(0.1, sd, sd)
                shock.Transparency = math.clamp(0.15 + 0.85 * sa, 0, 1)
            end
            if alpha >= 1 and conn then conn:Disconnect() end
        end)
        Debris:AddItem(core, dur + 0.2)
        Debris:AddItem(halo, dur + 0.2)
        Debris:AddItem(shock, 0.5)
    end

    -- ── §07 BEAM BULLET TRACER v2 — comet-style bolt: a white-hot tapered CORE + coloured inner
    -- GLOW + wide soft HALO fly muzzle→impact as a short bright window (whip) with a PointLight
    -- riding the head; on arrival a pooled micro-flash pops at the endpoint and the full-length
    -- trail ghosts out ease-out. POOL of 8 reusable rigs built ONCE and parked in getHoloFolder()
    -- (NO per-shot Instance.new — the impact flash is part of the rig); a fired rig repositions its
    -- two anchor Parts and restarts one bounded Heartbeat. World-space; READS the camera only.
    -- Gated by cfg("FXBeamTracer") at the callsite (onHit always lands, so there is no miss branch;
    -- FXBeamMissColor stays in config purely as a stale-save stub).
    local _beamPool, _beamInit = {}, false
    local function buildBeamRig()
        local model = Instance.new("Model")
        model.Name = "_bt" .. math.random(10000, 99999)
        local function anchor()
            local pt = Instance.new("Part")
            pt.Size = Vector3.new(0.05, 0.05, 0.05)
            pt.Anchored = true; pt.CanCollide = false; pt.CanQuery = false
            pt.CanTouch = false; pt.CastShadow = false; pt.Massless = true
            pt.Transparency = 1
            pt:SetAttribute("VS_Holo", true)
            pt.Parent = model
            local at = Instance.new("Attachment"); at.Parent = pt
            return pt, at
        end
        local part0, a0 = anchor()   -- tail
        local part1, a1 = anchor()   -- head (the light + impact flash live here)
        local function mkBeam()
            local b = Instance.new("Beam")
            b.Attachment0 = a0; b.Attachment1 = a1
            b.FaceCamera = true; b.Segments = 1; b.Enabled = false
            b.Parent = part0
            return b
        end
        -- three coaxial layers: core (white-hot) / glow (colour) / halo (wide + faint = bloom).
        -- Prism repurposes glow+halo as its two chromatic satellites (opposite CurveSize bows).
        local core, glow, halo = mkBeam(), mkBeam(), mkBeam()
        local light = Instance.new("PointLight")   -- rides the comet head; spikes at impact
        light.Range = 12; light.Brightness = 0; light.Enabled = false
        light.Parent = part1
        local impact = Instance.new("Part")        -- pooled endpoint micro-flash (parked invisible)
        impact.Shape = Enum.PartType.Ball
        impact.Size = Vector3.new(0.2, 0.2, 0.2)
        impact.Material = Enum.Material.Neon
        impact.Transparency = 1
        impact.Anchored = true; impact.CanCollide = false; impact.CanQuery = false
        impact.CanTouch = false; impact.CastShadow = false; impact.Massless = true
        impact:SetAttribute("VS_Holo", true)
        impact.Parent = model
        model.Parent = getHoloFolder()
        return { model = model, part0 = part0, part1 = part1, core = core, glow = glow, halo = halo,
                 light = light, impact = impact, on = false, conn = nil, startT = 0 }
    end
    local function ensureBeamPool()
        if _beamInit then return end
        _beamInit = true
        for i = 1, 8 do _beamPool[i] = buildBeamRig() end
    end
    -- beam-length transparency ramp: `head` alpha at the impact end (a1), `tail` at the muzzle end
    -- (a0), biased so the bright zone hugs the head — the comet/ghost profile in one helper.
    local function beamRamp(head, tail)
        return NumberSequence.new({
            NumberSequenceKeypoint.new(0, tail),
            NumberSequenceKeypoint.new(0.55, tail + (head - tail) * 0.65),
            NumberSequenceKeypoint.new(1, head),
        })
    end
    local function spawnBeamTracer(muzzlePos, hitPos)
        if not (muzzlePos and hitPos) then return end
        local dist = (hitPos - muzzlePos).Magnitude
        if dist < 0.5 then return end   -- degenerate shot: bail BEFORE stealing a live rig
        ensureBeamPool()
        -- pick a free rig, else steal the oldest; re-park any rig whose model was reclaimed
        local rig, oldest, oldT = nil, nil, math.huge
        for _, r in ipairs(_beamPool) do
            if not (r.model and r.model.Parent) then
                pcall(function() if r.model then r.model.Parent = getHoloFolder() end end)
            end
            if not r.on then rig = r break end
            if r.startT < oldT then oldest, oldT = r, r.startT end
        end
        rig = rig or oldest
        if not rig then return end
        if rig.conn then rig.conn:Disconnect(); rig.conn = nil end
        local dirU  = (hitPos - muzzlePos).Unit
        local color = cfg("FXBeamHitColor", _GOLD)
        local style = cfg("FXBeamStyle", "Glow")
        local lit   = style == "Line"                -- crisp/lit single layer; every other style is emissive
        local prism = style == "Prism"               -- chromatic split: white core + EDGE/VISIBLE bows
        local useLight  = (not lit) and cfg("FXBeamGlowLight", true)
        local useImpact = cfg("FXBeamImpact", true)
        local core, glow, halo = rig.core, rig.glow, rig.halo
        -- layer palette: near-white core, colour glow with a white-hot tip, soft colour halo
        core.Color = ColorSequence.new(lit and color or (prism and _WHITE or color:Lerp(_WHITE, 0.78)))
        local gCol = prism and _EDGE or color
        glow.Color = ColorSequence.new({
            ColorSequenceKeypoint.new(0, gCol),
            ColorSequenceKeypoint.new(0.8, gCol),
            ColorSequenceKeypoint.new(1, prism and _EDGE or color:Lerp(_WHITE, 0.5)),
        })
        halo.Color = ColorSequence.new(prism and _VISIBLE or color)
        core.LightEmission = lit and 0 or 1
        core.LightInfluence = lit and 1 or 0
        glow.LightEmission = 1; glow.LightInfluence = 0
        halo.LightEmission = 1; halo.LightInfluence = 0
        -- widths: tapered trio (glow ~3x core, halo ~6x soft bloom; prism satellites slim)
        local w0, w1 = cfg("FXBeamWidth0", 0.18), cfg("FXBeamWidth1", 0.04)
        core.Width0 = w0 * 0.55; core.Width1 = w1 * 0.55
        local gw = prism and 0.45 or 1.7
        local hw = prism and 0.45 or 3.4
        glow.Width0 = w0 * gw; glow.Width1 = w1 * gw
        halo.Width0 = w0 * hw; halo.Width1 = w1 * hw
        -- curvature: "Arc" bows the trio together; Prism splits glow/halo into ± chromatic bows
        local curve = (style == "Arc") and math.clamp(dist * 0.06, 0.5, 9) or 0
        local split = prism and math.clamp(dist * 0.02, 0.3, 1.6) or 0
        core.CurveSize0 = curve;         core.CurveSize1 = curve
        glow.CurveSize0 = curve + split; glow.CurveSize1 = curve + split
        halo.CurveSize0 = curve - split; halo.CurveSize1 = curve - split
        local seg = (curve ~= 0 or split ~= 0) and 10 or 1
        core.Segments = seg; glow.Segments = seg; halo.Segments = seg
        -- comet ramps (bright head, fading tail) — static during flight, so the per-frame flight
        -- cost is exactly two CFrame writes
        core.Transparency = beamRamp(0, 0.85)
        glow.Transparency = beamRamp(0.3, 0.92)
        halo.Transparency = beamRamp(0.72, 0.985)
        -- travel: a bright window (`trail` studs) races muzzle→hit at FXBeamTravelSpeed
        local travDur = 0
        if cfg("FXBeamTravel", true) then
            travDur = math.min(dist / math.max(cfg("FXBeamTravelSpeed", 1400), 100), 0.25)
            if travDur < 0.02 then travDur = 0 end
        end
        local trail = math.clamp(dist * 0.35, 4, 30)
        rig.part0.CFrame = CFrame.new(muzzlePos)
        rig.part1.CFrame = CFrame.new(travDur > 0 and (muzzlePos + dirU * 0.5) or hitPos)
        core.Enabled = true
        glow.Enabled = not lit
        halo.Enabled = not lit
        rig.light.Color = prism and _EDGE or color
        rig.light.Brightness = 2.5
        rig.light.Enabled = useLight
        rig.impact.Color = color:Lerp(_WHITE, 0.55)
        rig.impact.Transparency = 1
        rig.on = true; rig.startT = tick()
        local dur = cfg("FXBeamDur", 0.55)
        local landed = travDur <= 0
        local conn
        conn = RunService.Heartbeat:Connect(function()
            if not (rig.model and rig.model.Parent) then
                rig.on = false; if conn then conn:Disconnect() end; rig.conn = nil; return
            end
            local t = tick() - rig.startT
            if t < travDur then
                -- flight: constant-speed head + trailing window (the whip)
                local headD = (t / travDur) * dist
                pcall(function()
                    rig.part1.CFrame = CFrame.new(muzzlePos + dirU * headD)
                    rig.part0.CFrame = CFrame.new(muzzlePos + dirU * math.max(headD - trail, 0))
                end)
                return
            end
            if not landed then
                landed = true
                pcall(function()
                    rig.part1.CFrame = CFrame.new(hitPos)
                    rig.part0.CFrame = CFrame.new(muzzlePos)   -- ghost = full-length trail
                    if useImpact then rig.impact.CFrame = CFrame.new(hitPos) end
                end)
            end
            local a = math.clamp((t - travDur) / dur, 0, 1)
            local e = 1 - (1 - a) * (1 - a)   -- ease-out fade
            pcall(function()
                -- ghost trail lifts to clear (stays brightest at the impact end)
                core.Transparency = beamRamp(0.25 + 0.75 * e, 0.9 + 0.1 * e)
                if not lit then
                    glow.Transparency = beamRamp(0.5 + 0.5 * e, 0.95 + 0.05 * e)
                    halo.Transparency = beamRamp(0.85 + 0.15 * e, 1)
                end
                if useImpact then
                    -- endpoint micro-flash: pop 0.25→1.15 studs quad-out (~120ms), gone by 300ms
                    local ie = 1 - (1 - math.clamp((t - travDur) / 0.12, 0, 1)) ^ 2
                    local d = 0.25 + 0.9 * ie
                    rig.impact.Size = Vector3.new(d, d, d)
                    rig.impact.Transparency = 0.05 + 0.95 * math.clamp((t - travDur) / 0.3, 0, 1)
                end
                if useLight then rig.light.Brightness = 4 * (1 - e) end   -- impact spike -> decay
            end)
            if a >= 1 then
                rig.on = false
                pcall(function()
                    core.Enabled = false; glow.Enabled = false; halo.Enabled = false
                    rig.light.Enabled = false; rig.light.Brightness = 0
                    rig.impact.Transparency = 1
                end)
                if conn then conn:Disconnect() end; rig.conn = nil
            end
        end)
        rig.conn = conn
    end

    -- ── §07 WORLD HIT/KILL SPARKS — neon dot burst (hit) or an expanding shockwave disc (kill) at the
    -- world hit point. Pooled <=12 (reuse oldest), same Debris/bounded-Heartbeat pattern. World-space;
    -- no camera writes. Colors are code-const per the spec. Gated by cfg("FXWorldSpark") at the callsite.
    local _SPARK_HIT  = Color3.fromRGB(255, 233, 184)   -- #FFE9B8
    local _SPARK_KILL = Color3.fromRGB(255, 194, 75)    -- #FFC24B
    local _sparks = {}
    local function sparkPrune()
        for i = #_sparks, 1, -1 do
            local m = _sparks[i]
            if not (m and m.Parent) then table.remove(_sparks, i) end
        end
    end
    -- Flat neon disc (Cylinder — circular faces along local X) oriented to FACE THE CAMERA (visuals-v3
    -- §03: "FaceCamera") so the expanding ring reads as a clean circle around the impact on any surface
    -- / mid-air, not a floor decal. Grown from d0->d1 by the caller.
    local function mkSparkRing(parent, pos, dia, color)
        local r = Instance.new("Part")
        r.Shape = Enum.PartType.Cylinder
        r.Size = Vector3.new(0.1, dia, dia)
        r.Material = Enum.Material.Neon
        r.Color = color
        r.Transparency = 0.1
        r.Anchored = true; r.CanCollide = false; r.CanQuery = false
        r.CanTouch = false; r.CastShadow = false; r.Massless = true
        r:SetAttribute("VS_Holo", true)
        -- point the disc's flat normal (local X) at the camera: lookAt gives -Z toward the camera, then
        -- yaw +90° maps local X onto that -Z direction. Falls back to a flat/up disc if no camera.
        local cam = workspace.CurrentCamera
        if cam then
            r.CFrame = CFrame.lookAt(pos, cam.CFrame.Position) * CFrame.Angles(0, math.rad(90), 0)
        else
            r.CFrame = CFrame.new(pos) * CFrame.Angles(0, 0, math.rad(90))
        end
        r.Parent = parent
        return r
    end
    local function spawnWorldSpark(pos, kill)
        if not pos then return end
        sparkPrune()
        while #_sparks >= 12 do
            local old = table.remove(_sparks, 1)
            if old then pcall(function() old:Destroy() end) end
        end
        local folder = getHoloFolder()
        local color  = kill and _SPARK_KILL or _SPARK_HIT
        local model  = Instance.new("Model")
        model.Name = (kill and "_ks" or "_hs") .. math.random(10000, 99999)

        -- PointLight bloom (gated) — a brief, bright flash at the world hit point
        local bloom, bloomB0 = nil, kill and 6 or 4
        if cfg("FXWorldSparkBloom", true) then
            local lpart = Instance.new("Part")
            lpart.Size = Vector3.new(0.05, 0.05, 0.05)
            lpart.Transparency = 1
            lpart.Anchored = true; lpart.CanCollide = false; lpart.CanQuery = false
            lpart.CanTouch = false; lpart.CastShadow = false; lpart.Massless = true
            lpart:SetAttribute("VS_Holo", true)
            lpart.CFrame = CFrame.new(pos)
            lpart.Parent = model
            local light = Instance.new("PointLight")
            light.Color = color; light.Range = kill and 14 or 9; light.Brightness = bloomB0
            light.Parent = lpart
            bloom = light
        end

        -- rings + sparks: {part, d0, d1} discs; kill = 2 staggered, hit = 1 + 3 short sparks
        local rings, sparks, dirs = {}, {}, {}
        local dur
        if kill then
            dur = 0.32
            rings[1] = { p = mkSparkRing(model, pos, 0.6, color), d0 = 0.6, d1 = 6.0 }  -- r 0.3 -> 3.0
            rings[2] = { p = mkSparkRing(model, pos, 1.0, color), d0 = 1.0, d1 = 4.4 }  -- r 0.5 -> 2.2
        else
            dur = 0.22
            rings[1] = { p = mkSparkRing(model, pos, 0.6, color), d0 = 0.6, d1 = 4.0 }  -- r 0.3 -> 2.0
            for i = 1, 3 do
                local b = Instance.new("Part")
                b.Shape = Enum.PartType.Ball
                b.Size = Vector3.new(0.12, 0.12, 0.12)
                b.Material = Enum.Material.Neon
                b.Color = color
                b.Transparency = 0
                b.Anchored = true; b.CanCollide = false; b.CanQuery = false
                b.CanTouch = false; b.CastShadow = false; b.Massless = true
                b:SetAttribute("VS_Holo", true)
                b.CFrame = CFrame.new(pos)
                b.Parent = model
                sparks[i] = b
                local ang  = math.random() * math.pi * 2
                local elev = (math.random() - 0.5) * 1.2
                dirs[i] = Vector3.new(math.cos(ang), elev, math.sin(ang)).Unit
            end
        end

        model.Parent = folder
        _sparks[#_sparks + 1] = model
        local startT = tick()
        local conn
        conn = RunService.Heartbeat:Connect(function()
            if not model.Parent then if conn then conn:Disconnect() end return end
            local a = math.clamp((tick() - startT) / dur, 0, 1)
            local e = 1 - (1 - a) * (1 - a)                 -- quad-out grow
            local tr = math.clamp(0.1 + 0.9 * a, 0, 1)      -- 0.1 -> 1 fade
            pcall(function()
                for _, rg in ipairs(rings) do
                    local d = rg.d0 + (rg.d1 - rg.d0) * e
                    rg.p.Size = Vector3.new(0.1, d, d)
                    rg.p.Transparency = tr
                end
                for i, b in ipairs(sparks) do
                    if b and b.Parent then
                        b.CFrame = CFrame.new(pos + dirs[i] * (0.6 * e) + Vector3.new(0, 0.4 * e, 0))
                        b.Transparency = math.clamp(a, 0, 1)
                    end
                end
                if bloom then bloom.Brightness = bloomB0 * (1 - e) end
            end)
            if a >= 1 and conn then conn:Disconnect() end
        end)
        Debris:AddItem(model, dur + 0.2)
    end
    -- exported for the 06b meteor impact (same pooled spark, no duplicate FX path)
    Visuals.worldSpark = spawnWorldSpark

    -- ── FABLE v6 · KILL FLOURISH — world-anchored lethal FX, fired from FX.onHit on a killing blow.
    -- Three independent, default-OFF features (FXKillPillar / FXKillShards / FXKillPulse), each a
    -- one-shot burst: concurrency-capped, one bounded Heartbeat, Debris-reclaimed — the same budget
    -- rules as the sparks above. IIFE keeps the subsystem's ~30 locals off this proto's registers.
    local spawnKillPillar, spawnKillShards, triggerKillPulse, clearKillPulse
    ;(function()
        local _pillars, _bursts = {}, {}
        local function alive(list)   -- prune dead models, return the live count (concurrency gate)
            for i = #list, 1, -1 do
                local m = list[i]
                if not (m and m.Parent) then table.remove(list, i) end
            end
            return #list
        end
        local function mkNeon(model, shape, color, tr)
            local p = Instance.new("Part")
            p.Shape = shape
            p.Material = Enum.Material.Neon
            p.Color = color
            p.Transparency = tr
            p.Anchored = true; p.CanCollide = false; p.CanQuery = false
            p.CanTouch = false; p.CastShadow = false; p.Massless = true
            p:SetAttribute("VS_Holo", true)
            p.Parent = model
            return p
        end
        local VERT = CFrame.Angles(0, 0, math.rad(90))   -- cylinder long axis (X) -> world Y

        -- LIGHT PILLAR: a column of light claims the kill spot — the beam stretches skyward and
        -- thins while a ground ring rolls out beneath it and ember motes float up through it,
        -- all under one decaying PointLight bloom. ~1s, <=3 concurrent.
        spawnKillPillar = function(pos)
            if alive(_pillars) >= 3 then return end
            local color = Config.FXKillPillarColor or _SPARK_KILL
            local model = Instance.new("Model")
            model.Name = "_kf" .. math.random(10000, 99999)
            local dur = 1.0
            local col = mkNeon(model, Enum.PartType.Cylinder, color, 0.3)
            col.Size = Vector3.new(3, 1.1, 1.1)
            -- base pinned at pos-1: center = pos + h/2 - 1 (matches the growth formula below)
            col.CFrame = CFrame.new(pos + Vector3.new(0, 0.5, 0)) * VERT
            local ring = mkNeon(model, Enum.PartType.Cylinder, color, 0.15)
            ring.Size = Vector3.new(0.12, 1.4, 1.4)
            ring.CFrame = CFrame.new(pos - Vector3.new(0, 2.2, 0)) * VERT   -- flat disc at the feet
            local motes, mBase, mVel = {}, {}, {}
            for i = 1, 6 do
                local m = mkNeon(model, Enum.PartType.Ball, color:Lerp(_WHITE, 0.35), 0.1)
                local s = 0.14 + math.random() * 0.12
                m.Size = Vector3.new(s, s, s)
                mBase[i] = pos + Vector3.new((math.random() - 0.5) * 2.2,
                    math.random() * 1.5 - 1.5, (math.random() - 0.5) * 2.2)
                m.CFrame = CFrame.new(mBase[i])
                local a = math.random() * math.pi * 2
                mVel[i] = Vector3.new(math.cos(a) * (0.6 + math.random()), 6 + math.random() * 5,
                    math.sin(a) * (0.6 + math.random()))
                motes[i] = m
            end
            local anchorP = mkNeon(model, Enum.PartType.Ball, color, 1)   -- invisible bloom carrier
            anchorP.Size = Vector3.new(0.05, 0.05, 0.05)
            anchorP.CFrame = CFrame.new(pos)
            local bloom = Instance.new("PointLight")
            bloom.Color = color; bloom.Range = 16; bloom.Brightness = 7
            bloom.Parent = anchorP
            model.Parent = getHoloFolder()
            _pillars[#_pillars + 1] = model
            local t0 = tick()
            local conn
            conn = RunService.Heartbeat:Connect(function()
                if not model.Parent then if conn then conn:Disconnect() end return end
                local a = math.clamp((tick() - t0) / dur, 0, 1)
                local e = 1 - (1 - a) * (1 - a)
                pcall(function()
                    local h = 3 + 21 * e                       -- column reaches ~24 studs
                    local w = 1.1 * (1 - 0.55 * e)             -- and thins as it climbs
                    col.Size = Vector3.new(h, w, w)
                    col.CFrame = CFrame.new(pos + Vector3.new(0, h * 0.5 - 1, 0)) * VERT
                    -- brighten for the first ~30% (0.3 -> 0.15), then lift to clear
                    col.Transparency = a < 0.3 and (0.3 - 0.5 * a) or (0.15 + 0.85 * (a - 0.3) / 0.7)
                    local d = 1.4 + 8.6 * e
                    ring.Size = Vector3.new(0.12, d, d)
                    ring.Transparency = 0.15 + 0.85 * e
                    for i = 1, 6 do
                        motes[i].CFrame = CFrame.new(mBase[i] + mVel[i] * (dur * e))   -- eased rise
                        motes[i].Transparency = 0.1 + 0.9 * a
                    end
                    bloom.Brightness = 7 * (1 - e)
                end)
                if a >= 1 and conn then conn:Disconnect() end
            end)
            Debris:AddItem(model, dur + 0.2)
        end

        -- SHATTER BURST: the victim "breaks" — 10 neon glass shards explode outward on faux-gravity
        -- arcs, tumbling, shrinking and burning out. 0.75s, <=3 concurrent (30 parts worst case).
        spawnKillShards = function(pos)
            if alive(_bursts) >= 3 then return end
            local color = Config.FXKillShardsColor or _EDGE
            local model = Instance.new("Model")
            model.Name = "_kb" .. math.random(10000, 99999)
            local dur = 0.75
            local shards, sBase, sVel, sRot, sSpin, sSize = {}, {}, {}, {}, {}, {}
            for i = 1, 10 do
                local s = mkNeon(model, Enum.PartType.Block,
                    i % 3 == 0 and color:Lerp(_WHITE, 0.5) or color, 0.05)
                local k = 0.7 + math.random() * 0.8
                sSize[i] = Vector3.new(0.42 * k, 0.26 * k, 0.09 * k)
                s.Size = sSize[i]
                sBase[i] = pos + Vector3.new((math.random() - 0.5) * 1.4,
                    (math.random() - 0.5) * 1.8, (math.random() - 0.5) * 1.4)
                sRot[i] = CFrame.Angles(math.random() * 6.283, math.random() * 6.283, math.random() * 6.283)
                s.CFrame = CFrame.new(sBase[i]) * sRot[i]
                local a = (i / 10) * math.pi * 2 + math.random()
                sVel[i] = Vector3.new(math.cos(a) * (6 + math.random() * 7), 5 + math.random() * 9,
                    math.sin(a) * (6 + math.random() * 7))
                sSpin[i] = Vector3.new(math.random() * 10 - 5, math.random() * 10 - 5, math.random() * 10 - 5)
                shards[i] = s
            end
            model.Parent = getHoloFolder()
            _bursts[#_bursts + 1] = model
            local t0 = tick()
            local conn
            conn = RunService.Heartbeat:Connect(function()
                if not model.Parent then if conn then conn:Disconnect() end return end
                local a = math.clamp((tick() - t0) / dur, 0, 1)
                local t = a * dur
                local tr = a < 0.4 and 0.05 or (0.05 + 0.95 * (a - 0.4) / 0.6)   -- hold, then burn out
                local sc = 1 - 0.45 * a                                          -- shrink to ~55%
                pcall(function()
                    for i = 1, 10 do
                        local p = sBase[i] + sVel[i] * t - Vector3.new(0, 26 * t * t, 0)   -- ballistic arc
                        local sp = sSpin[i]
                        shards[i].CFrame = CFrame.new(p) * sRot[i] * CFrame.Angles(sp.X * t, sp.Y * t, sp.Z * t)
                        shards[i].Size = sSize[i] * sc
                        shards[i].Transparency = tr
                    end
                end)
                if a >= 1 and conn then conn:Disconnect() end
            end)
            Debris:AddItem(model, dur + 0.2)
        end

        -- IMPACT-FRAME PULSE: one transient ColorCorrection pops a desaturated/brighter "impact
        -- frame" for 0.35s on each kill (a retrigger just restarts the decay — one loop, no stack).
        -- Own instance + own attribute so preset clearTagged() / the grade CC never fight it.
        local _pulseCC, _pulseConn, _pulseT0 = nil, nil, 0
        triggerKillPulse = function()
            if not (_pulseCC and _pulseCC.Parent) then
                local cc = Instance.new("ColorCorrectionEffect")
                cc.Name = "_vs_pulse"
                cc:SetAttribute("VS_Pulse", true)
                cc.Parent = Lighting
                _pulseCC = cc
            end
            _pulseT0 = tick()
            if _pulseConn then return end
            _pulseConn = RunService.Heartbeat:Connect(function()
                local cc = _pulseCC
                if not (cc and cc.Parent) then
                    if _pulseConn then _pulseConn:Disconnect(); _pulseConn = nil end
                    return
                end
                local a = (tick() - _pulseT0) / 0.35
                if a >= 1 then
                    _pulseConn:Disconnect(); _pulseConn = nil
                    pcall(function() cc.Saturation = 0; cc.Brightness = 0; cc.Contrast = 0 end)
                    return
                end
                local k = (1 - a) * (1 - a) * math.clamp(Config.FXKillPulseAmount or 0.6, 0, 1)
                pcall(function()
                    cc.Saturation = -0.5 * k     -- sharp attack, quad decay back to neutral
                    cc.Brightness = 0.07 * k
                    cc.Contrast   = 0.12 * k
                end)
            end)
        end
        clearKillPulse = function()
            if _pulseConn then _pulseConn:Disconnect(); _pulseConn = nil end
            if _pulseCC then pcall(function() _pulseCC:Destroy() end); _pulseCC = nil end
        end
    end)()

    -- ── Visuals.FX — screen feedback subsystem ─────────────────────────────────────────────────
    -- One persistent ScreenGui (full-screen tints), FIXED Drawing/Sound pools (.Visible-toggled,
    -- never allocated per frame) and ONE RenderStepped updater advancing every active timer.
    -- Fired from the shared hit context in notifyTarget (outgoing) and the read-only Humanoid
    -- watcher (incoming). Reads the camera; NEVER writes CFrame/FieldOfView.
    local FX = {}
    Visuals.FX = FX
    -- IIFE (not a bare `do`): moves the FX subsystem's ~100 block-locals off the main-chunk
    -- register file. The bundle is one flat chunk (=one function); a bare `do` keeps these
    -- locals live in the chunk and pushes it past Luau's ~200-active-locals-per-function limit
    -- (loadstring-time "Out of local registers"). FX is an upvalue (line 740); every outer
    -- symbol used inside is read-only via upvalue, so wrapping in a function is behavior-neutral.
    ;(function()
        local SoundService = game:GetService("SoundService")
        local StatsService = game:GetService("Stats")
        local hasDrawing = screenDraw ~= nil

        local C_GREY   = Color3.fromRGB(200, 200, 200)
        local C_AMBER  = Color3.fromRGB(255, 170, 60)
        local C_ORANGE = Color3.fromRGB(255, 120, 30)
        local C_RED    = Color3.fromRGB(255, 59, 78)   -- SIGNAL ENEMY #FF3B4E (marker lethal fallback; NOT incoming)
        local C_SCREENRED = Color3.fromRGB(194, 30, 47) -- SIGNAL screen-red #C21E2F (incoming cues only)
        local C_BLACK  = Color3.new(0, 0, 0)
        -- SIGNAL GOLD #FFC24B — the reserved HUD gold gate (hud-v3 §02). Distinct from the hologram
        -- lethal-core gold _GOLD #FFC83C (255,200,60); the HUD must not show two different golds side by
        -- side, so crit/lethal damage numbers and the headshot spark use THIS token, not _GOLD.
        local C_GOLD   = Color3.fromRGB(255, 194, 75)
        -- SIGNAL v4 · HUD-widget fill/track/secondary-text tokens (match the radar disc / ESP HP track)
        local C_FILL   = Color3.fromRGB(13, 18, 25)    -- SIGNAL FILL   #0D1219 (@0.62 opacity)
        local C_HPBG   = Color3.fromRGB(11, 15, 22)    -- SIGNAL HP track #0B0F16 (@0.86 opacity)
        local C_TEXT2  = Color3.fromRGB(174, 185, 197) -- SIGNAL TEXT-2 #AEB9C5

        -- continuous body-damage colour ramp: grey(<=0) -> amber(~25) -> orange(>=50).
        -- Gold stays gated to crit/kill at the callsite; this is the non-lethal body path only.
        local function dnRamp(total)
            if total <= 25 then
                return C_GREY:Lerp(C_AMBER, math.clamp(total / 25, 0, 1))
            end
            return C_AMBER:Lerp(C_ORANGE, math.clamp((total - 25) / 25, 0, 1))
        end

        local _started, _alloc = false, false
        local _updConn, _gui = nil, nil

        -- pools + per-feature state
        local _hmLines, _hmLinesBlk = nil, nil                   -- hit marker: 4 Lines (+ 4 dark casing)
        local _hm = { on = false, t0 = 0, pop = 0, color = _WHITE }
        local _snds, _sndIdx, _sndLastT = nil, 1, 0              -- hit sound: 4 Sounds
        local _dnPool, _dnActive, _dnByPlr = nil, {}, {}         -- damage numbers: 24 Texts
        local _kbL1, _kbL2, _kbRing = nil, nil, nil              -- kill banner: 2 Texts + 1 Circle
        local _kb = { on = false, t0 = 0, streak = 0, lastKillT = 0, line1 = "", line2 = "" }
        local _ddArcs = nil                                      -- damage direction: 6 arcs x 9 segs x2
        local _kfPool, _kfItems = nil, {}                        -- kill feed: 5 Texts
        local _hsLines, _hsLinesBlk = nil, nil                   -- headshot spark: 6 Lines (+ 6 dark casing)
        local _hs = { on = false, t0 = 0, pos = nil }
        local _flashFrame = nil                                  -- incoming flash: 1 Frame
        local _hf = { on = false, t0 = 0 }
        local _vgFrames = nil                                    -- low-HP vignette: 4 edge Frames
        local _vgHP, _vgLastPoll = 1, 0
        local _hcConn, _charConn, _lastHP = nil, nil, nil        -- incoming watcher
        local _fovA, _fovB, _fovFill, _fovCas = nil, nil, nil, nil -- §07 gradient FOV ring: 2 stacked rings + fill disc (+ dark casing)
        -- SIGNAL v4 · new fixed pools/state (all allocated once in allocate(), idle = shim skip-write free)
        local _fxLastT = nil                                     -- FX-updater frame dt (fps EMA + panel smoothing)
        local _fxThrT  = 0                                        -- bug#1 rank2: throttle FX.update to ~120Hz (was every RenderStepped, uncapped — 144/240Hz baseline cost)
        local _hmRing, _hmRingBlk = nil, nil                     -- hit marker Ring/Dot styles: 1 Circle (+ casing)
        local _chLines, _chLinesBlk, _chDot, _chDotBlk = nil, nil, nil, nil   -- crosshair: 4 ticks (+casing) + dot (+casing)
        local _chTri, _chTriBlk = nil, nil                       -- TriDot style: 3 orbit dots (+ casings)
        local _ch2 = { shots = 0, shotT = -10, rot = 0, rotTgt = 0 }   -- crosshair v2: bloom decay + TriDot spin (reads State.Shots)
        local _wmBg, _wmAccent, _wmText = nil, nil, nil          -- watermark plate: bg + gold LEFT accent + brand ("LuaHook"); stats Text lives in _wm.stats
        local _wm = { fps = 60, ping = 0, pingT = 0, str = "", strT = 0, bw = nil, pw = 60 }
        local _sessKills, _sessT0 = 0, tick()                    -- session kill counter (fed by FX.onHit)
        local _tiBg, _tiAccent, _tiName, _tiHpBg, _tiHpFill, _tiInfo = nil, nil, nil, nil, nil, nil  -- target info panel
        local _ti = { tgt = nil, a = 0, hp = nil, frac = 1, pollT = 0, name = nil, info = "" }
        local _blTexts, _blHead, _blBar = nil, nil, nil          -- active-features list: 10 rows + header + spine
        local _bl = { scanT = 0, rows = {}, on = {}, t0 = {} }
        local _kfTicks = nil                                     -- kill feed: 5 gold left-edge accent ticks

        local function mkDraw(t, props)
            if not hasDrawing then return nil end
            local ok, d = pcall(screenDraw, t, "fx")   -- FX HUD markers parent to the higher-DisplayOrder FX layer (above ESP)
            if not ok then return nil end
            d.Visible = false
            if props then for k, v in pairs(props) do pcall(function() d[k] = v end) end end
            return d
        end

        local function ensureGui()
            if _gui and _gui.Parent then return _gui end
            local g = Instance.new("ScreenGui")
            g.Name = "_vs_fx"
            g.IgnoreGuiInset = true
            g.ResetOnSpawn = false
            g.DisplayOrder = 999
            local ok = pcall(function() g.Parent = (gethui and gethui()) or game:GetService("CoreGui") end)
            if not ok or not g.Parent then
                pcall(function() g.Parent = lp:FindFirstChildOfClass("PlayerGui") end)
            end
            _gui = g
            return g
        end

        local function allocate()
            if _alloc then return end
            _alloc = true
            local g = ensureGui()
            -- incoming-hit flash: ONE full-screen frame, coalesced per burst
            local fr = Instance.new("Frame")
            fr.Name = "_fl"; fr.BackgroundColor3 = C_SCREENRED; fr.BackgroundTransparency = 1
            fr.BorderSizePixel = 0; fr.Size = UDim2.new(1, 0, 1, 0); fr.Visible = false
            fr.ZIndex = 1; fr.Parent = g
            _flashFrame = fr
            -- low-HP vignette: four thin edge frames, each a UIGradient fading INWARD (no assets)
            _vgFrames = {}
            local SIDES = {
                { size = UDim2.new(1, 0, 0.16, 0),  pos = UDim2.new(0, 0, 0, 0),     rot = 90  },  -- top
                { size = UDim2.new(1, 0, 0.16, 0),  pos = UDim2.new(0, 0, 0.84, 0),  rot = 270 },  -- bottom
                { size = UDim2.new(0.12, 0, 1, 0),  pos = UDim2.new(0, 0, 0, 0),     rot = 0   },  -- left
                { size = UDim2.new(0.12, 0, 1, 0),  pos = UDim2.new(0.88, 0, 0, 0),  rot = 180 },  -- right
            }
            for i, s in ipairs(SIDES) do
                local f = Instance.new("Frame")
                f.Name = "_vg" .. i; f.BackgroundColor3 = C_SCREENRED; f.BackgroundTransparency = 1
                f.BorderSizePixel = 0; f.Size = s.size; f.Position = s.pos; f.Visible = false
                f.ZIndex = 2
                local grad = Instance.new("UIGradient")
                grad.Rotation = s.rot
                grad.Transparency = NumberSequence.new(0, 1)   -- opaque at the edge, clear inward
                grad.Parent = f
                f.Parent = g
                _vgFrames[i] = f
            end
            -- Drawing pools (fixed caps; black underlays created BEFORE their red lines = under)
            _hmLinesBlk = {}
            for i = 1, 4 do _hmLinesBlk[i] = mkDraw("Line", { Color = C_BLACK }) end   -- casing (under)
            _hmLines = {}
            for i = 1, 4 do _hmLines[i] = mkDraw("Line") end
            _dnPool = {}
            for i = 1, 24 do _dnPool[i] = mkDraw("Text", { Center = true, Outline = true, Font = 0 }) end
            _kbL1   = mkDraw("Text", { Center = true, Outline = true, Font = 3, Size = 12 })
            _kbL2   = mkDraw("Text", { Center = true, Outline = true, Font = 0, Size = 22 })
            _kbRing = mkDraw("Circle", { Filled = false, NumSides = 48 })
            _ddArcs = {}
            for i = 1, 6 do
                local arc = { on = false, t0 = 0, ang = 0, dscale = 0, red = {}, blk = {} }
                for j = 1, 9 do
                    arc.blk[j] = mkDraw("Line", { Color = C_BLACK, Thickness = 8 })   -- 2px underlay each side of the 4px line
                    arc.red[j] = mkDraw("Line", { Color = C_SCREENRED, Thickness = 4 })
                end
                _ddArcs[i] = arc
            end
            _kfPool = {}
            for i = 1, 5 do _kfPool[i] = mkDraw("Text", { Center = false, Outline = true, Font = 3, Size = 13 }) end
            _hsLinesBlk = {}
            for i = 1, 6 do _hsLinesBlk[i] = mkDraw("Line", { Color = C_BLACK, Thickness = 4 }) end   -- casing (under)
            _hsLines = {}
            for i = 1, 6 do _hsLines[i] = mkDraw("Line", { Thickness = 2 }) end
            -- §07 gradient FOV ring: a faint fill disc + dark casing + 2 stacked drift-shimmering outline circles
            _fovFill = mkDraw("Circle", { Filled = true,  NumSides = 96 })
            _fovCas  = mkDraw("Circle", { Filled = false, NumSides = 96 })   -- casing (under the outline rings, above the fill)
            _fovA    = mkDraw("Circle", { Filled = false, NumSides = 96 })
            _fovB    = mkDraw("Circle", { Filled = false, NumSides = 96 })
            -- SIGNAL v4 · hit-marker Ring/Dot style geometry (casing created FIRST = renders under)
            _hmRingBlk = mkDraw("Circle", { Filled = false, Color = C_BLACK })
            _hmRing    = mkDraw("Circle", { Filled = false })
            -- SIGNAL v4 · custom crosshair: dark casing ticks under 4 color ticks + center dot (+ casing)
            _chLinesBlk = {}
            for i = 1, 4 do _chLinesBlk[i] = mkDraw("Line", { Color = C_BLACK }) end
            _chLines = {}
            for i = 1, 4 do _chLines[i] = mkDraw("Line") end
            _chDotBlk = mkDraw("Circle", { Filled = true, Color = C_BLACK })
            _chDot    = mkDraw("Circle", { Filled = true })
            -- TriDot crosshair: 3 orbit dots (casings first = under)
            _chTriBlk = {}
            for i = 1, 3 do _chTriBlk[i] = mkDraw("Circle", { Filled = true, Color = C_BLACK }) end
            _chTri = {}
            for i = 1, 3 do _chTri[i] = mkDraw("Circle", { Filled = true }) end
            -- LuaHook watermark plate: FILL bg + 2px gold LEFT accent + GothamMedium brand + muted mono stats
            _wmBg     = mkDraw("Square", { Filled = true, Color = C_FILL })
            _wmAccent = mkDraw("Line",   { Color = C_GOLD, Thickness = 2 })
            _wmText   = mkDraw("Text",   { Center = false, Outline = true, Font = 2, Size = 13, Text = "LuaHook", Color = _WHITE })
            _wm.stats = mkDraw("Text",   { Center = false, Outline = true, Font = 3, Size = 12, Color = C_TEXT2 })
            -- SIGNAL v4 · target info panel: bg + gold left accent + name / HP track+fill / info footer
            _tiBg     = mkDraw("Square", { Filled = true, Color = C_FILL })
            _tiAccent = mkDraw("Line",   { Color = C_GOLD, Thickness = 2 })
            _tiHpBg   = mkDraw("Square", { Filled = true, Color = C_HPBG })
            _tiHpFill = mkDraw("Square", { Filled = true })
            _tiName   = mkDraw("Text",   { Center = true, Outline = true, Font = 0, Size = 13 })
            _tiInfo   = mkDraw("Text",   { Center = true, Outline = true, Font = 0, Size = 11, Color = C_TEXT2 })
            -- SIGNAL v4 · active-features list: gold spine + gold "SIGNAL" header + 10 pooled rows
            _blBar  = mkDraw("Line", { Color = C_GOLD, Thickness = 2 })
            _blHead = mkDraw("Text", { Center = false, Outline = true, Font = 3, Size = 12, Color = C_GOLD })
            _blTexts = {}
            for i = 1, 10 do _blTexts[i] = mkDraw("Text", { Center = false, Outline = true, Font = 3, Size = 12 }) end
            -- SIGNAL v4 · kill feed accent: 2px gold left tick per visible row
            _kfTicks = {}
            for i = 1, 5 do _kfTicks[i] = mkDraw("Line", { Color = C_GOLD, Thickness = 2 }) end
            -- Sound pool x4 under SoundService
            _snds = {}
            for i = 1, 4 do
                local s = Instance.new("Sound")
                s.Name = "_fxs" .. i
                s.Volume = 0.5
                s.Parent = SoundService
                _snds[i] = s
            end
        end

        -- ── triggers ───────────────────────────────────────────────────────────
        local function triggerHitMarker(crit, lethal)
            if not (_hmLines and _hmLines[1]) then return end
            local now = tick()
            -- sustained fire RE-TRIGGERS the one marker: reset life + small pop (<= +3px), never restack
            if _hm.on and (now - _hm.t0) < 0.18 then
                _hm.pop = math.min(_hm.pop + 1, 3)
            else
                _hm.pop = 0
            end
            _hm.on = true; _hm.t0 = now
            _hm.color = (lethal and cfg("FXHitMarkerLethalColor", C_RED))
                or (crit and cfg("FXHitMarkerCritColor", _GOLD))
                or cfg("FXHitMarkerColor", _WHITE)
        end

        local function playHitSound(dmg, lethal)
            if not _snds then return end
            local id = lethal and cfg("FXKillSoundId", "") or cfg("FXHitSoundId", "")
            if not id or id == "" then return end       -- no-op until an id is configured
            local now = tick()
            if (now - _sndLastT) < 0.035 then return end -- anti-spam
            _sndLastT = now
            local s = _snds[_sndIdx]
            _sndIdx = (_sndIdx % #_snds) + 1
            if not (s and s.Parent) then return end
            pcall(function()
                if s.SoundId ~= id then s.SoundId = id end
                if lethal then
                    s.PlaybackSpeed = 1.0; s.Volume = 0.7
                else
                    s.PlaybackSpeed = 0.9 + math.clamp(dmg / 50, 0, 1) * 0.45
                    s.Volume = cfg("FXHitSoundVolume", 0.5)
                end
                s:Play()
            end)
        end

        local function dnFree(d)
            for _, e in ipairs(_dnActive) do if e.d == d then return false end end
            return true
        end
        local function pushDamageNumber(p, dmg, crit, lethal, hitPos)
            if not (_dnPool and hitPos) then return end
            local now = tick()
            local e = _dnByPlr[p]
            -- a hit inside the accumulation window ADDS to the number + resets life + re-pops
            if e and e.alive and (now - e.lastT) <= cfg("FXDamageAccumWindow", 0.9) then
                e.total = e.total + dmg
                e.t0 = now; e.lastT = now; e.popT = now; e.pos = hitPos
                e.crit = e.crit or crit; e.lethal = e.lethal or lethal
                return
            end
            local d
            for _, cand in ipairs(_dnPool) do
                if cand and dnFree(cand) then d = cand break end
            end
            if not d then return end   -- pool cap 24: drop overflow
            e = { d = d, p = p, total = dmg, pos = hitPos, t0 = now, lastT = now, popT = now,
                  drift = math.random(-8, 8), crit = crit, lethal = lethal, alive = true }
            _dnByPlr[p] = e
            table.insert(_dnActive, e)
        end

        local function trackText(s)   -- +0.3-tracking approximation for the 12px mono line
            return (s:gsub("(.)", "%1 ")):sub(1, -2)
        end
        local KB_STREAK = { [2] = "DOUBLE", [3] = "TRIPLE", [4] = "QUAD" }
        local function triggerKillBanner(p)
            if not (_kbL1 and _kbL2) then return end
            local now = tick()
            -- multi-kill within 4s escalates; burst opacity climbs +0.1/step (applied in update)
            if (now - _kb.lastKillT) <= 4 then _kb.streak = _kb.streak + 1 else _kb.streak = 1 end
            _kb.lastKillT = now
            _kb.on = true; _kb.t0 = now
            local label = "ELIMINATED"
            if _kb.streak >= 5 then label = _kb.streak .. "x"
            elseif _kb.streak >= 2 then label = KB_STREAK[_kb.streak] end
            _kb.line1 = trackText(label)
            _kb.line2 = tostring(p.DisplayName or p.Name)
        end

        local function pushKillFeed(p, crit)
            if not _kfPool then return end
            -- SIGNAL v4 · crit kills carry a gold row (crit flag rides the item)
            table.insert(_kfItems, 1, { text = "You  ·  " .. tostring(p.DisplayName or p.Name), t0 = tick(), crit = crit and true or false })
            while #_kfItems > 5 do table.remove(_kfItems) end
        end

        local function triggerSpark(hitPos)
            if not (_hsLines and hitPos) then return end
            _hs.on = true; _hs.t0 = tick(); _hs.pos = hitPos
        end

        local function triggerFlash()
            if not _flashFrame then return end
            _hf.on = true; _hf.t0 = tick()   -- coalesced: a burst just restarts the one flash
        end

        -- nearest VISIBLE enemy heuristic, reusing the ESP's cached o._vis + root positions
        local function nearestEnemyPos()
            local myC = lp.Character
            local myR = myC and myC:FindFirstChild("HumanoidRootPart")
            if not myR then return nil end
            local best, bestD, bestVis, bestVisD = nil, math.huge, nil, math.huge
            for plr, o in pairs(State.ESPObjects) do
                local r = o.root
                if plr ~= lp and r and r.Parent then
                    local d = (r.Position - myR.Position).Magnitude
                    if o._vis and d < bestVisD then bestVis, bestVisD = r.Position, d end
                    if d < bestD then best, bestD = r.Position, d end
                end
            end
            return bestVis or best
        end

        local function triggerDirection(drop)
            if not _ddArcs then return end
            local src = nearestEnemyPos(); if not src then return end
            -- camera-SPACE bearing (READS the camera only — never writes it)
            local rel = Camera.CFrame:PointToObjectSpace(src)
            local ang = math.atan2(rel.X, -rel.Z)   -- Luau keeps atan2 (math.atan is 1-arg there)
            local arc, oldest, oldT = nil, nil, math.huge
            for _, a in ipairs(_ddArcs) do
                if not a.on then arc = a break end
                if a.t0 < oldT then oldest, oldT = a, a.t0 end
            end
            arc = arc or oldest                          -- <= 6 concurrent: steal the oldest
            if not arc then return end
            arc.on = true; arc.t0 = tick(); arc.ang = ang
            arc.dscale = math.clamp(drop / 50, 0, 1)
            -- build the 9-segment geometry ONCE here (static for the arc's <=1.12s life); update() only fades.
            local vp = Camera.ViewportSize
            local cx, cy = vp.X * 0.5, vp.Y * 0.5
            local r = 0.22 * math.min(vp.X, vp.Y)
            local width = math.rad(24 + 24 * arc.dscale)             -- 24 -> 48 deg by damage
            local step = width / 9
            local th0 = ang - width * 0.5
            for j = 1, 9 do
                local t1 = th0 + step * (j - 1)
                local t2 = t1 + step * 0.8                           -- short segments, small gaps
                local p1 = Vector2.new(cx + math.sin(t1) * r, cy - math.cos(t1) * r)
                local p2 = Vector2.new(cx + math.sin(t2) * r, cy - math.cos(t2) * r)
                local lb, lr = arc.blk[j], arc.red[j]
                if lb then lb.From = p1; lb.To = p2 end
                if lr then lr.From = p1; lr.To = p2 end
            end
            arc.built = true
        end

        -- ── public fan-in ──────────────────────────────────────────────────────
        function FX.onHit(p, dmg, crit, lethal, hitPos)
            if not _started then return end
            if cfg("FXHitMarker", true) then triggerHitMarker(crit, lethal) end
            if dmg > 0 or lethal then
                if cfg("FXHitSound", true) then playHitSound(dmg, lethal) end
                if dmg > 0 and cfg("FXDamageNumbers", true) then pushDamageNumber(p, dmg, crit, lethal, hitPos) end
            end
            if crit and cfg("FXHeadshotSpark", true) then triggerSpark(hitPos) end
            if lethal then
                _sessKills = _sessKills + 1   -- SIGNAL v4 · watermark session-kill counter (zero new hooks)
                if cfg("FXKillBanner", true) then triggerKillBanner(p) end
                if cfg("FXKillFeed", true) then pushKillFeed(p, crit) end
            end
            -- §07/§v6 world FX (default OFF): tracer + spark + kill flourish at the world hit point
            if hitPos then
                if cfg("FXBeamTracer", false) then
                    -- barrel offset: spawn the bolt low-right of the camera so it reads as fired
                    -- from the gun, not the player's face (READS the camera only)
                    local cc = Camera.CFrame
                    pcall(spawnBeamTracer,
                        cc.Position + cc.RightVector * 1.4 - cc.UpVector * 1.05 + cc.LookVector * 1.5,
                        hitPos)
                end
                if cfg("FXWorldSpark", false) then pcall(spawnWorldSpark, hitPos, lethal) end
                if lethal then
                    if cfg("FXKillPillar", false) then pcall(spawnKillPillar, hitPos) end
                    if cfg("FXKillShards", false) then pcall(spawnKillShards, hitPos) end
                    if cfg("FXKillPulse", false)  then pcall(triggerKillPulse) end
                end
            end
        end

        function FX.onIncoming(drop)
            if not _started then return end
            if cfg("FXHitFlash", true) then triggerFlash() end
            if cfg("FXDamageDirection", true) then triggerDirection(drop) end
        end

        -- ── the ONE updater ────────────────────────────────────────────────────
        local DIAG = { Vector2.new(1, 1), Vector2.new(-1, 1), Vector2.new(1, -1), Vector2.new(-1, -1) }
        local INV_SQ2 = 0.70710678
        -- SIGNAL v4 · axis-aligned unit vectors (N/E/S/W): crosshair Cross/T ticks + hit-marker "Plus".
        -- Index 1 is NORTH so the "T" style can skip the top tick by index.
        local PLUS = { Vector2.new(0, -1), Vector2.new(1, 0), Vector2.new(0, 1), Vector2.new(-1, 0) }
        -- SIGNAL v4 · active-features list: label -> Config master key (exact 02_config masters)
        local BL_FEATURES = {
            { "RAGE",    "Rage" },
            { "SILENT",  "SilentAim" },
            { "AIMBOT",  "Aimbot" },
            { "ESP",     "ESP" },
            { "VISUALS", "Visuals" },
        }

        local function hideMarker()
            _hm.on = false
            for _, l in ipairs(_hmLines) do if l then l.Visible = false end end
            if _hmLinesBlk then for _, l in ipairs(_hmLinesBlk) do if l then l.Visible = false end end end
            if _hmRing then _hmRing.Visible = false end
            if _hmRingBlk then _hmRingBlk.Visible = false end
        end

        local function update()
            local now = tick()
            -- bug#1 rank2: ~120Hz cap (matches ESP). Skipped frames leave _fxLastT untouched so the next
            -- processed frame's dt correctly reflects real elapsed time (fps EMA / panel smoothing stay accurate).
            if now - _fxThrT < 0.0083 then return end
            _fxThrT = now
            -- SIGNAL v4 · frame dt for the fps EMA + target-panel smoothing (clamped: menu hitches don't spike)
            local dt = now - (_fxLastT or now)
            _fxLastT = now
            if dt > 0.1 then dt = 0.1 end
            local vp = Camera.ViewportSize
            local cx, cy = vp.X * 0.5, vp.Y * 0.5

            -- hit marker: 70ms snap-out, 180ms life. SIGNAL v4 · 4 styles (X | Plus | Ring | Dot):
            -- _hm state/timing verbatim; only the geometry differs per style.
            if _hm.on and _hmLines then
                local a = now - _hm.t0
                if a >= 0.18 then
                    hideMarker()
                else
                    local style = cfg("FXHitMarkerStyle", "X")
                    local snap = math.clamp(a / 0.07, 0, 1)
                    snap = 1 - (1 - snap) * (1 - snap)
                    local gap = cfg("FXHitMarkerGap", 5)
                    local len = cfg("FXHitMarkerLen", 8) * snap + _hm.pop
                    local th  = cfg("FXHitMarkerThickness", 2)
                    local tr  = 1 - math.clamp((a - 0.09) / 0.09, 0, 1)
                    if style == "Ring" or style == "Dot" then
                        for i, l in ipairs(_hmLines) do
                            if l then l.Visible = false end
                            local lb = _hmLinesBlk and _hmLinesBlk[i]
                            if lb then lb.Visible = false end
                        end
                        if _hmRing then
                            local ctr = Vector2.new(cx, cy)
                            if style == "Ring" then
                                -- ring expands gap -> gap+len (quad-out via snap), fading with tr
                                local r = math.max(gap + len, 1)
                                if _hmRingBlk then
                                    _hmRingBlk.Filled = false; _hmRingBlk.Position = ctr; _hmRingBlk.Radius = r
                                    _hmRingBlk.Thickness = th + 2; _hmRingBlk.Color = C_BLACK
                                    _hmRingBlk.Transparency = tr; _hmRingBlk.Visible = true
                                end
                                _hmRing.Filled = false; _hmRing.Position = ctr; _hmRing.Radius = r
                                _hmRing.Thickness = th; _hmRing.Color = _hm.color
                                _hmRing.Transparency = tr; _hmRing.Visible = true
                            else
                                -- dot pops 1.4x -> 1 over the snap
                                local r = (th + 1) * (1.4 - 0.4 * snap)
                                if _hmRingBlk then
                                    _hmRingBlk.Filled = true; _hmRingBlk.Position = ctr; _hmRingBlk.Radius = r + 1
                                    _hmRingBlk.Color = C_BLACK; _hmRingBlk.Transparency = tr; _hmRingBlk.Visible = true
                                end
                                _hmRing.Filled = true; _hmRing.Position = ctr; _hmRing.Radius = r
                                _hmRing.Color = _hm.color; _hmRing.Transparency = tr; _hmRing.Visible = true
                            end
                        end
                    else
                        if _hmRing then _hmRing.Visible = false end
                        if _hmRingBlk then _hmRingBlk.Visible = false end
                        local plus = style == "Plus"
                        for i, l in ipairs(_hmLines) do
                            if l then
                                local nx, ny
                                if plus then nx, ny = PLUS[i].X, PLUS[i].Y
                                else nx, ny = DIAG[i].X * INV_SQ2, DIAG[i].Y * INV_SQ2 end
                                local from = Vector2.new(cx + nx * gap, cy + ny * gap)
                                local to   = Vector2.new(cx + nx * (gap + len), cy + ny * (gap + len))
                                local lb = _hmLinesBlk and _hmLinesBlk[i]
                                if lb then   -- dark casing under each tick (legibility on bright scenes)
                                    lb.From = from; lb.To = to
                                    lb.Color = C_BLACK; lb.Thickness = th + 2
                                    lb.Transparency = tr; lb.Visible = true
                                end
                                l.From = from
                                l.To   = to
                                l.Color = _hm.color
                                l.Thickness = th
                                l.Transparency = tr
                                l.Visible = true
                            end
                        end
                    end
                end
            end

            -- SIGNAL v4 · custom crosshair: 4 ticks (Cross/X/T) + optional center dot, dark casing,
            -- 60ms hit pop (+2px) riding the existing _hm.t0 hit timestamp. Static frames are pure
            -- value-equal writes => the shim skip-write stage makes an idle crosshair ~free.
            if _chLines then
                if cfg("FXCrosshair", false) then
                    local style = cfg("FXCrosshairStyle", "Cross")
                    local gap  = cfg("FXCrosshairGap", 4)
                    local len  = cfg("FXCrosshairLen", 7)
                    local th   = cfg("FXCrosshairThickness", 2)
                    local col  = cfg("FXCrosshairColor", _WHITE)
                    local outl = cfg("FXCrosshairOutline", true)
                    if cfg("FXCrosshairHitPop", true) and _hm.t0 > 0 then
                        len = len + 2 * (1 - math.clamp((now - _hm.t0) / 0.06, 0, 1))
                    end
                    -- crosshair v2: State.Shots delta (gunhook-maintained — zero new hooks)
                    -- feeds the bloom decay + the TriDot spin target.
                    local sh = State.Shots or 0
                    if sh ~= _ch2.shots then
                        _ch2.rotTgt = _ch2.rotTgt + 15 * math.min(math.abs(sh - _ch2.shots), 3)
                        _ch2.shots = sh; _ch2.shotT = now
                    end
                    if cfg("FXCrosshairBloom", false) then
                        gap = gap + 3 * (1 - math.clamp((now - _ch2.shotT) / 0.12, 0, 1))
                    end
                    local tri  = style == "TriDot"
                    local chev = style == "Chevron"
                    local diag = style == "X"
                    for i = 1, 4 do
                        local l  = _chLines[i]
                        local lb = _chLinesBlk and _chLinesBlk[i]
                        -- T skips the top tick; Chevron uses only legs 1-2; Dot/TriDot use no ticks
                        local hide = tri or style == "Dot" or (style == "T" and i == 1) or (chev and i > 2)
                        if l then
                            if hide then
                                l.Visible = false; if lb then lb.Visible = false end
                            else
                                local from, to
                                if chev then
                                    -- two 45° legs under the point — an upward caret cradling the aim
                                    local sx = (i == 1) and -1 or 1
                                    from = Vector2.new(cx + sx, cy + gap)
                                    to   = Vector2.new(cx + sx * (1 + len * 0.85), cy + gap + len * 0.85)
                                else
                                    local nx, ny
                                    if diag then nx, ny = DIAG[i].X * INV_SQ2, DIAG[i].Y * INV_SQ2
                                    else nx, ny = PLUS[i].X, PLUS[i].Y end
                                    from = Vector2.new(cx + nx * gap, cy + ny * gap)
                                    to   = Vector2.new(cx + nx * (gap + len), cy + ny * (gap + len))
                                end
                                if lb then
                                    if outl then
                                        lb.From = from; lb.To = to; lb.Thickness = th + 2
                                        lb.Color = C_BLACK; lb.Transparency = 1; lb.Visible = true
                                    else lb.Visible = false end
                                end
                                l.From = from; l.To = to; l.Thickness = th
                                l.Color = col; l.Transparency = 1; l.Visible = true
                            end
                        end
                    end
                    -- TriDot: 3 dots at 120° on a gap-radius orbit; each shot nudges the
                    -- orientation +15°, eased outQuad-ish (~120ms) — the reticle visibly "cycles" fire.
                    if _chTri then
                        if tri then
                            _ch2.rot = _ch2.rot + (_ch2.rotTgt - _ch2.rot) * math.clamp(dt * 18, 0, 1)
                            local r    = gap + len * 0.6
                            local dr   = th + 1
                            local base = math.rad(_ch2.rot) - 1.5707963
                            for i = 1, 3 do
                                local ang = base + (i - 1) * 2.0943951   -- 120° apart
                                local p = Vector2.new(cx + math.cos(ang) * r, cy + math.sin(ang) * r)
                                local db = _chTriBlk and _chTriBlk[i]
                                if db then
                                    if outl then db.Position = p; db.Radius = dr + 1; db.Transparency = 1; db.Visible = true
                                    else db.Visible = false end
                                end
                                local d2 = _chTri[i]
                                if d2 then d2.Position = p; d2.Radius = dr; d2.Color = col; d2.Transparency = 1; d2.Visible = true end
                            end
                        else
                            for i = 1, 3 do
                                local d2 = _chTri[i]; if d2 and d2.Visible then d2.Visible = false end
                                local db = _chTriBlk and _chTriBlk[i]; if db and db.Visible then db.Visible = false end
                            end
                        end
                    end
                    if _chDot then
                        if style == "Dot" or cfg("FXCrosshairDot", true) then
                            local r = math.max(th * 0.5 + 0.5, 1)
                            if style == "Dot" then r = th + 1 end
                            if _chDotBlk then
                                if outl then
                                    _chDotBlk.Filled = true; _chDotBlk.Position = Vector2.new(cx, cy)
                                    _chDotBlk.Radius = r + 1; _chDotBlk.Color = C_BLACK
                                    _chDotBlk.Transparency = 1; _chDotBlk.Visible = true
                                else _chDotBlk.Visible = false end
                            end
                            _chDot.Filled = true; _chDot.Position = Vector2.new(cx, cy)
                            _chDot.Radius = r; _chDot.Color = col
                            _chDot.Transparency = 1; _chDot.Visible = true
                        else
                            _chDot.Visible = false
                            if _chDotBlk then _chDotBlk.Visible = false end
                        end
                    end
                else
                    for i = 1, 4 do
                        local l = _chLines[i]; if l and l.Visible then l.Visible = false end
                        local lb = _chLinesBlk and _chLinesBlk[i]; if lb and lb.Visible then lb.Visible = false end
                    end
                    if _chDot and _chDot.Visible then _chDot.Visible = false end
                    if _chDotBlk and _chDotBlk.Visible then _chDotBlk.Visible = false end
                    if _chTri then
                        for i = 1, 3 do
                            local d2 = _chTri[i]; if d2 and d2.Visible then d2.Visible = false end
                            local db = _chTriBlk and _chTriBlk[i]; if db and db.Visible then db.Visible = false end
                        end
                    end
                end
            end

            -- damage numbers: rise 42px / 700ms ease-out, x-drift, fade last 40%
            if _dnActive[1] then
                for i = #_dnActive, 1, -1 do
                    local e = _dnActive[i]
                    local a = (now - e.t0) / 0.7
                    if a >= 1 then
                        if e.d then e.d.Visible = false end
                        e.alive = false
                        if _dnByPlr[e.p] == e then _dnByPlr[e.p] = nil end
                        table.remove(_dnActive, i)
                    elseif e.d then
                        local sp = Camera:WorldToViewportPoint(e.pos)
                        if sp.Z <= 0 then
                            e.d.Visible = false
                        else
                            local d = e.d
                            local ease = 1 - (1 - a) * (1 - a)
                            local pop  = 1 + 0.25 * (1 - math.clamp((now - e.popT) / 0.12, 0, 1))  -- 1.25 -> 1
                            d.Size = math.floor((14 + math.clamp(e.total / 50, 0, 1) * 8) * pop + 0.5)
                            d.Text = tostring(math.floor(e.total + 0.5))
                            if e.crit or e.lethal then d.Color = C_GOLD   -- SIGNAL GOLD (not the #FFC83C hologram gold)
                            else d.Color = dnRamp(e.total) end   -- continuous grey->amber->orange by total
                            d.Position = Vector2.new(sp.X + e.drift * a, sp.Y - 42 * ease)
                            d.Transparency = a < 0.6 and 1 or 1 - (a - 0.6) / 0.4
                            d.Visible = true
                        end
                    end
                end
            end

            -- kill banner: scale-in 90ms -> hold 700ms -> fade+rise 8px 380ms (+ ring burst 320ms)
            if _kb.on then
                local a = now - _kb.t0
                if a >= 1.17 then
                    _kb.on = false
                    if _kbL1 then _kbL1.Visible = false end
                    if _kbL2 then _kbL2.Visible = false end
                    if _kbRing then _kbRing.Visible = false end
                else
                    local y = cy - 140
                    local scale, tr, rise = 1, 1, 0
                    if a < 0.09 then
                        scale = 0.6 + 0.4 * (a / 0.09)
                    elseif a > 0.79 then
                        local f = (a - 0.79) / 0.38
                        tr = 1 - f
                        rise = 8 * f
                    end
                    if _kbL1 then
                        -- banner v2: the tracked streak line reveals over 240ms outCubic
                        -- (alpha + 3px settle) instead of popping with the name — a typographic ease.
                        local lt = math.clamp(a / 0.24, 0, 1)
                        local le = 1 - (1 - lt) * (1 - lt) * (1 - lt)
                        _kbL1.Text = _kb.line1
                        _kbL1.Size = math.floor(12 * scale + 0.5)
                        _kbL1.Color = _WHITE
                        _kbL1.Position = Vector2.new(cx, y - rise - 3 * (1 - le))
                        _kbL1.Transparency = tr * le
                        _kbL1.Visible = true
                    end
                    if _kbL2 then
                        _kbL2.Text = _kb.line2
                        _kbL2.Size = math.floor(22 * scale + 0.5)
                        _kbL2.Color = cfg("FXKillBannerColor", _GOLD)
                        _kbL2.Position = Vector2.new(cx, y - rise + 16)
                        _kbL2.Transparency = tr
                        _kbL2.Visible = true
                    end
                    if _kbRing then
                        if a < 0.32 then
                            local f = a / 0.32
                            local fe = 1 - (1 - f) * (1 - f)
                            _kbRing.Position = Vector2.new(cx, cy)
                            _kbRing.Radius = 6 + 40 * fe                       -- r 6 -> 46
                            _kbRing.Thickness = 2 - 1.5 * f                    -- thick 2 -> 0.5
                            _kbRing.Color = cfg("FXKillBannerColor", _GOLD)
                            _kbRing.Transparency = math.min(1, 0.6 + 0.1 * (_kb.streak - 1)) * (1 - f)
                            _kbRing.Visible = true
                        else
                            _kbRing.Visible = false
                        end
                    end
                end
            end

            -- damage direction: geometry built once at trigger; per-frame = fade only (no sin/cos/alloc)
            if _ddArcs then
                for _, arc in ipairs(_ddArcs) do
                    if arc.on then
                        local a = now - arc.t0
                        if a >= 1.12 then
                            arc.on = false
                            for j = 1, 9 do
                                if arc.red[j] then arc.red[j].Visible = false end
                                if arc.blk[j] then arc.blk[j].Visible = false end
                            end
                        else
                            local op = 0.6 + 0.35 * arc.dscale                 -- 0.6 -> 0.95 by damage
                            if a > 0.22 then op = op * (1 - (a - 0.22) / 0.9) end
                            for j = 1, 9 do
                                local lb, lr = arc.blk[j], arc.red[j]
                                if lb then lb.Transparency = op * 0.8; lb.Visible = true end
                                if lr then lr.Transparency = op; lr.Visible = true end
                            end
                        end
                    end
                end
            end

            -- kill feed: top-right stack, slide-in 120ms, 5s life
            if _kfPool then
                -- anchor below the radar disc when it's shown (else the legacy 110px top-right origin)
                local y0 = Config.ESPRadar and (Config.ESPRadarInset + Config.ESPRadarSize + 16) or 110
                for i, d in ipairs(_kfPool) do
                    local it = _kfItems[i]
                    local tk = _kfTicks and _kfTicks[i]
                    if d then
                        if not it or (now - it.t0) >= 5 then
                            d.Visible = false
                            if tk then tk.Visible = false end
                        else
                            local a = now - it.t0
                            local slide = math.clamp(a / 0.12, 0, 1)
                            slide = 1 - (1 - slide) * (1 - slide)
                            d.Text = it.text
                            d.Color = it.crit and C_GOLD or _WHITE   -- SIGNAL v4 · crit kill = gold row
                            d.Size = 13
                            local rx = vp.X - 16 - d.TextBounds.X + (1 - slide) * 30
                            local ry = y0 + (i - 1) * 18
                            local tr = a < 4 and 1 or 1 - (a - 4)
                            d.Position = Vector2.new(rx, ry)
                            d.Transparency = tr
                            d.Visible = true
                            if tk then   -- SIGNAL v4 · 2px gold left accent tick per visible row
                                tk.From = Vector2.new(rx - 8, ry + 2)
                                tk.To   = Vector2.new(rx - 8, ry + 13)
                                tk.Thickness = 2; tk.Color = C_GOLD
                                tk.Transparency = tr; tk.Visible = true
                            end
                        end
                    end
                end
                for i = #_kfItems, 1, -1 do
                    if (now - _kfItems[i].t0) >= 5 then table.remove(_kfItems, i) end
                end
            end

            -- headshot spark: 6 gold ticks expanding 4 -> 12px over 180ms
            if _hs.on and _hsLines then
                local a = (now - _hs.t0) / 0.18
                if a >= 1 then
                    _hs.on = false
                    for _, l in ipairs(_hsLines) do if l then l.Visible = false end end
                    if _hsLinesBlk then for _, l in ipairs(_hsLinesBlk) do if l then l.Visible = false end end end
                else
                    local sp = Camera:WorldToViewportPoint(_hs.pos)
                    if sp.Z <= 0 then
                        for _, l in ipairs(_hsLines) do if l then l.Visible = false end end
                        if _hsLinesBlk then for _, l in ipairs(_hsLinesBlk) do if l then l.Visible = false end end end
                    else
                        local rad = 4 + 8 * a
                        local tr = 1 - a * a
                        for i, l in ipairs(_hsLines) do
                            if l then
                                local th = (i - 1) * (math.pi / 3)
                                local dx, dy = math.cos(th), math.sin(th)
                                local from = Vector2.new(sp.X + dx * rad, sp.Y + dy * rad)
                                local to   = Vector2.new(sp.X + dx * (rad + 5), sp.Y + dy * (rad + 5))
                                local lb = _hsLinesBlk and _hsLinesBlk[i]
                                if lb then   -- dark casing under each tick
                                    lb.From = from; lb.To = to
                                    lb.Transparency = tr; lb.Visible = true
                                end
                                l.From = from
                                l.To   = to
                                l.Color = C_GOLD   -- SIGNAL GOLD (hud-v3 §04: "6 gold Lines")
                                l.Transparency = tr
                                l.Visible = true
                            end
                        end
                    end
                end
            end

            -- incoming flash: 0.78 -> 1 over 160ms
            if _hf.on and _flashFrame then
                local a = (now - _hf.t0) / 0.16
                if a >= 1 then
                    _hf.on = false
                    _flashFrame.Visible = false
                else
                    _flashFrame.BackgroundTransparency = 0.78 + 0.22 * a
                    _flashFrame.Visible = true
                end
            end

            -- low-HP vignette: severity + breathe (health polled at ~10Hz, animated every frame)
            if _vgFrames then
                local show = false
                if cfg("FXLowHPVignette", true) then
                    if (now - _vgLastPoll) > 0.1 then
                        _vgLastPoll = now
                        local hp, mh = getHealth(lp)
                        _vgHP = (mh and mh > 0) and hp / mh or 1
                    end
                    local thr = cfg("FXLowHPThreshold", 0.35)
                    if _vgHP > 0 and _vgHP < thr then
                        local sev = math.clamp((thr - _vgHP) / math.max(thr - 0.10, 0.01), 0, 1)
                        local freq = 0.8 + 0.6 * sev                           -- breathe 0.8 -> 1.4Hz
                        local tr = (0.85 - 0.35 * sev)                         -- opacity 0.85 -> 0.5
                            + 0.06 * (0.5 + 0.5 * math.sin(now * freq * 6.283185))
                        tr = math.clamp(tr, 0, 1)
                        for _, f in ipairs(_vgFrames) do
                            f.BackgroundTransparency = tr
                            if not f.Visible then f.Visible = true end
                        end
                        show = true
                    end
                end
                if not show then
                    for _, f in ipairs(_vgFrames) do if f.Visible then f.Visible = false end end
                end
            end

            -- §07 gradient FOV ring: two stacked screen circles at radius = AimbotFOV, shimmering
            -- between colour A/B via a drift phase (rotation is a no-op on a circle, so FXFovRotate
            -- simply gates whether the shimmer advances). READS Config only; no camera write.
            if _fovA then
                if cfg("FXFovRing", false) then
                    local R  = math.max(4, cfg("AimbotFOV", 120))
                    local th = math.clamp(cfg("FXFovThickness", 1.5), 0.5, 4)
                    local cA = cfg("FXFovColorA", _WHITE)
                    local cB = cfg("FXFovColorB", _GOLD)
                    local phase = cfg("FXFovRotate", true) and (now * cfg("FXFovDriftSpeed", 0.15)) or 0
                    local t = 0.5 + 0.5 * math.sin(phase * 6.283185)
                    local center = Vector2.new(cx, cy)
                    if _fovFill then
                        if cfg("FXFovFill", false) then
                            _fovFill.Position = center; _fovFill.Radius = R
                            _fovFill.Color = _SPARK_KILL     -- #FFC24B
                            _fovFill.Transparency = 0.05
                            _fovFill.Visible = true
                        elseif _fovFill.Visible then
                            _fovFill.Visible = false
                        end
                    end
                    if _fovCas then   -- dark under-ring (legibility on bright scenes)
                        if cfg("FXFovCasing", true) then
                            _fovCas.Position = center; _fovCas.Radius = R
                            _fovCas.Thickness = th + 2; _fovCas.Color = C_BLACK
                            _fovCas.Transparency = 0.5; _fovCas.Visible = true
                        elseif _fovCas.Visible then
                            _fovCas.Visible = false
                        end
                    end
                    _fovA.Position = center; _fovA.Radius = R
                    _fovA.Thickness = th; _fovA.Color = cA:Lerp(cB, t)
                    _fovA.Transparency = 0.5; _fovA.Visible = true
                    if _fovB then
                        _fovB.Position = center; _fovB.Radius = math.max(1, R - th)
                        _fovB.Thickness = th; _fovB.Color = cB:Lerp(cA, t)
                        _fovB.Transparency = 0.5; _fovB.Visible = true
                    end
                else
                    if _fovA.Visible then _fovA.Visible = false end
                    if _fovB and _fovB.Visible then _fovB.Visible = false end
                    if _fovFill and _fovFill.Visible then _fovFill.Visible = false end
                    if _fovCas and _fovCas.Visible then _fovCas.Visible = false end
                end
            end

            -- LuaHook watermark: "▌ LuaHook  142 fps · 38 ms · 12:04 · 7 kills". fps = EMA of 1/dt;
            -- ping 1Hz; stats string + plate width rebuilt only every 0.25s; brand width measured ONCE.
            if _wmText then
                if cfg("HUDWatermark", true) then
                    if dt > 0 then _wm.fps = _wm.fps + (1 / dt - _wm.fps) * 0.1 end
                    local stats = cfg("HUDWatermarkStats", true)
                    if stats and (now - _wm.pingT) > 1 then
                        _wm.pingT = now
                        pcall(function()
                            _wm.ping = math.floor(StatsService.Network.ServerStatsItem["Data Ping"]:GetValue() + 0.5)
                        end)
                    end
                    _wmText.Position = Vector2.new(26, 21)
                    _wmText.Transparency = 1
                    _wmText.Visible = true
                    if not _wm.bw then   -- one-time brand measure (brand text never changes)
                        pcall(function() local tb = _wmText.TextBounds; if tb and tb.X > 0 then _wm.bw = tb.X end end)
                    end
                    local bw = _wm.bw or 52
                    if (now - _wm.strT) > 0.25 then
                        _wm.strT = now
                        if stats then
                            local sess = now - _sessT0
                            _wm.str = string.format("%d fps · %d ms · %02d:%02d · %d kills",
                                math.floor(_wm.fps + 0.5), _wm.ping,
                                math.floor(sess / 60), math.floor(sess % 60), _sessKills)
                        else
                            _wm.str = ""
                        end
                        local sw = 0
                        if _wm.stats and _wm.str ~= "" then
                            _wm.stats.Text = _wm.str
                            pcall(function() local tb = _wm.stats.TextBounds; if tb then sw = tb.X end end)
                        end
                        _wm.pw = (sw > 0) and (10 + bw + 12 + sw + 10) or (10 + bw + 10)
                    end
                    if _wm.stats then
                        if _wm.str ~= "" then
                            _wm.stats.Position = Vector2.new(26 + bw + 12, 22)
                            _wm.stats.Transparency = 1
                            _wm.stats.Visible = true
                        elseif _wm.stats.Visible then _wm.stats.Visible = false end
                    end
                    if _wmBg then
                        _wmBg.Filled = true
                        _wmBg.Position = Vector2.new(16, 16); _wmBg.Size = Vector2.new(_wm.pw, 24)
                        _wmBg.Color = C_FILL; _wmBg.Transparency = 0.72; _wmBg.Visible = true
                    end
                    if _wmAccent then
                        _wmAccent.From = Vector2.new(16, 16); _wmAccent.To = Vector2.new(16, 40)
                        _wmAccent.Thickness = 2; _wmAccent.Color = C_GOLD
                        _wmAccent.Transparency = 1; _wmAccent.Visible = true
                    end
                else
                    if _wmText.Visible then _wmText.Visible = false end
                    if _wm.stats and _wm.stats.Visible then _wm.stats.Visible = false end
                    if _wmBg and _wmBg.Visible then _wmBg.Visible = false end
                    if _wmAccent and _wmAccent.Visible then _wmAccent.Visible = false end
                end
            end

            -- SIGNAL v4 · target info panel: compact card under the crosshair for the ESP's locked
            -- primary target (State.PrimaryTarget, stamped in 10_esp render()). Name / EMA-smoothed
            -- hpRamp bar / dist+weapon footer. Health+info polled at 10Hz; 120ms fade on lock change.
            if _tiBg and _tiAccent and _tiName and _tiHpBg and _tiHpFill and _tiInfo then
                local tgt = nil
                if cfg("FXTargetInfo", false) then
                    tgt = State.PrimaryTarget
                    if tgt and not (tgt.Parent and tgt.Character and isAlive(tgt)) then tgt = nil end
                end
                if tgt ~= _ti.tgt then
                    _ti.tgt = tgt
                    if tgt then
                        _ti.name = tostring(tgt.DisplayName or tgt.Name)
                        _ti.hp = nil; _ti.pollT = 0   -- re-poll + re-seed the smooth fill immediately
                    end
                end
                if tgt and (now - (_ti.pollT or 0)) > 0.1 then
                    _ti.pollT = now
                    pcall(function()
                        local hp, mh = getHealth(tgt)
                        _ti.frac = math.clamp((mh or 0) > 0 and hp / mh or 0, 0, 1)
                        local d = 0
                        local myR = lp.Character and lp.Character:FindFirstChild("HumanoidRootPart")
                        local tR  = tgt.Character and tgt.Character:FindFirstChild("HumanoidRootPart")
                        if myR and tR then d = (myR.Position - tR.Position).Magnitude end
                        _ti.info = string.format("%dm  ·  %s", math.floor(d + 0.5), getWeaponName(tgt))
                    end)
                end
                -- 120ms fade in/out via an alpha EMA (fade-out keeps drawing the cached card)
                _ti.a = (_ti.a or 0) + ((tgt and 1 or 0) - (_ti.a or 0)) * math.clamp(dt * 16, 0, 1)
                if _ti.a > 0.02 and _ti.name then
                    local frac = _ti.frac or 1
                    if _ti.hp == nil then _ti.hp = frac
                    else _ti.hp = _ti.hp + (frac - _ti.hp) * (1 - math.exp(-18 * dt)) end
                    local aa = math.clamp(_ti.a, 0, 1)
                    local px = cx - 90
                    local py = cy + cfg("FXTargetInfoOffset", 110)
                    _tiBg.Filled = true; _tiBg.Position = Vector2.new(px, py); _tiBg.Size = Vector2.new(180, 46)
                    _tiBg.Color = C_FILL; _tiBg.Transparency = 0.62 * aa; _tiBg.Visible = true
                    _tiAccent.From = Vector2.new(px + 1, py); _tiAccent.To = Vector2.new(px + 1, py + 46)
                    _tiAccent.Thickness = 2; _tiAccent.Color = C_GOLD
                    _tiAccent.Transparency = aa; _tiAccent.Visible = true
                    _tiName.Text = _ti.name; _tiName.Position = Vector2.new(px + 90, py + 3)
                    _tiName.Color = _WHITE; _tiName.Transparency = aa; _tiName.Visible = true
                    local fillW = math.max(164 * math.clamp(_ti.hp, 0, 1), 1)
                    _tiHpBg.Filled = true; _tiHpBg.Position = Vector2.new(px + 8, py + 23)
                    _tiHpBg.Size = Vector2.new(164, 5); _tiHpBg.Color = C_HPBG
                    _tiHpBg.Transparency = 0.86 * aa; _tiHpBg.Visible = true
                    _tiHpFill.Filled = true; _tiHpFill.Position = Vector2.new(px + 8, py + 23)
                    _tiHpFill.Size = Vector2.new(fillW, 5); _tiHpFill.Color = hpRamp(_ti.hp)
                    _tiHpFill.Transparency = aa; _tiHpFill.Visible = true
                    _tiInfo.Text = _ti.info or ""; _tiInfo.Position = Vector2.new(px + 90, py + 31)
                    _tiInfo.Color = C_TEXT2; _tiInfo.Transparency = aa; _tiInfo.Visible = true
                else
                    if _tiBg.Visible then
                        _tiBg.Visible = false; _tiAccent.Visible = false; _tiName.Visible = false
                        _tiHpBg.Visible = false; _tiHpFill.Visible = false; _tiInfo.Visible = false
                    end
                end
            end

            -- LuaHook active-features list: gold "LuaHook" header + white rows for each enabled
            -- master, 120ms slide-in per row. The Config walk is throttled to 4Hz; per-frame writes
            -- settle to value-equal (shim skip-write) once the slide finishes. Corner preset only
            -- (no dragging by design — the Linoria 2nd-monitor cursor offset makes drags untrustworthy).
            if _blTexts then
                if cfg("HUDBindList", false) then
                    if (now - (_bl.scanT or 0)) >= 0.25 then
                        _bl.scanT = now
                        local n = 0
                        for _, f in ipairs(BL_FEATURES) do
                            if Config[f[2]] then
                                n = n + 1
                                _bl.rows[n] = f[1]
                                if not _bl.on[f[2]] then _bl.on[f[2]] = true; _bl.t0[f[1]] = now end
                            else
                                _bl.on[f[2]] = false
                            end
                        end
                        for i = #_bl.rows, n + 1, -1 do _bl.rows[i] = nil end
                    end
                    local right = cfg("HUDBindListSide", "Left") == "Right"
                    local bx = right and (vp.X - 110) or 16
                    local by = vp.Y * 0.35
                    if _blHead then
                        _blHead.Text = "LuaHook"; _blHead.Color = C_GOLD
                        _blHead.Position = Vector2.new(bx, by)
                        _blHead.Transparency = 1; _blHead.Visible = true
                    end
                    local nRows = #_bl.rows
                    for i, d in ipairs(_blTexts) do
                        local label = _bl.rows[i]
                        if label then
                            local a = math.clamp((now - (_bl.t0[label] or 0)) / 0.12, 0, 1)
                            local e = 1 - (1 - a) * (1 - a)   -- same easing as the kill feed
                            d.Text = label; d.Color = _WHITE
                            d.Position = Vector2.new(bx + (1 - e) * 14 * (right and 1 or -1), by + 17 + (i - 1) * 15)
                            d.Transparency = e
                            d.Visible = true
                        elseif d.Visible then
                            d.Visible = false
                        end
                    end
                    if _blBar then
                        _blBar.From = Vector2.new(bx - 6, by)
                        _blBar.To   = Vector2.new(bx - 6, by + 17 + nRows * 15)
                        _blBar.Thickness = 2; _blBar.Color = C_GOLD
                        _blBar.Transparency = 0.9; _blBar.Visible = true
                    end
                else
                    if _blHead and _blHead.Visible then _blHead.Visible = false end
                    if _blBar and _blBar.Visible then _blBar.Visible = false end
                    for _, d in ipairs(_blTexts) do if d.Visible then d.Visible = false end end
                end
            end
        end

        local function hideAllFX()
            if _hmLines then hideMarker() end
            for i = #_dnActive, 1, -1 do
                local e = _dnActive[i]
                if e.d then e.d.Visible = false end
                _dnActive[i] = nil
            end
            table.clear(_dnByPlr)
            _kb.on = false
            if _kbL1 then _kbL1.Visible = false end
            if _kbL2 then _kbL2.Visible = false end
            if _kbRing then _kbRing.Visible = false end
            if _ddArcs then
                for _, arc in ipairs(_ddArcs) do
                    arc.on = false
                    for j = 1, 9 do
                        if arc.red[j] then arc.red[j].Visible = false end
                        if arc.blk[j] then arc.blk[j].Visible = false end
                    end
                end
            end
            table.clear(_kfItems)
            if _kfPool then for _, d in ipairs(_kfPool) do if d then d.Visible = false end end end
            _hs.on = false
            if _hsLines then for _, l in ipairs(_hsLines) do if l then l.Visible = false end end end
            if _hsLinesBlk then for _, l in ipairs(_hsLinesBlk) do if l then l.Visible = false end end end
            _hf.on = false
            if _flashFrame then _flashFrame.Visible = false end
            if _vgFrames then for _, f in ipairs(_vgFrames) do f.Visible = false end end
            if _fovA then _fovA.Visible = false end
            if _fovB then _fovB.Visible = false end
            if _fovFill then _fovFill.Visible = false end
            if _fovCas then _fovCas.Visible = false end
            -- SIGNAL v4 · widgets
            if _hmRing then _hmRing.Visible = false end
            if _hmRingBlk then _hmRingBlk.Visible = false end
            if _chLines then for _, l in ipairs(_chLines) do if l then l.Visible = false end end end
            if _chLinesBlk then for _, l in ipairs(_chLinesBlk) do if l then l.Visible = false end end end
            if _chDot then _chDot.Visible = false end
            if _chDotBlk then _chDotBlk.Visible = false end
            if _chTri then for i = 1, 3 do if _chTri[i] then _chTri[i].Visible = false end end end
            if _chTriBlk then for i = 1, 3 do if _chTriBlk[i] then _chTriBlk[i].Visible = false end end end
            if _wmBg then _wmBg.Visible = false end
            if _wmAccent then _wmAccent.Visible = false end
            if _wmText then _wmText.Visible = false end
            if _wm.stats then _wm.stats.Visible = false end
            if _tiBg then _tiBg.Visible = false end
            if _tiAccent then _tiAccent.Visible = false end
            if _tiName then _tiName.Visible = false end
            if _tiHpBg then _tiHpBg.Visible = false end
            if _tiHpFill then _tiHpFill.Visible = false end
            if _tiInfo then _tiInfo.Visible = false end
            _ti.tgt = nil; _ti.a = 0
            if _blHead then _blHead.Visible = false end
            if _blBar then _blBar.Visible = false end
            if _blTexts then for _, d in ipairs(_blTexts) do if d then d.Visible = false end end end
            if _kfTicks then for _, l in ipairs(_kfTicks) do if l then l.Visible = false end end end
        end

        -- incoming-damage watcher: READ-ONLY Humanoid.HealthChanged, reconnected on respawn
        local function hookHumanoid(char)
            if _hcConn then _hcConn:Disconnect(); _hcConn = nil end
            if not char then return end
            task.spawn(function()
                local hum = char:FindFirstChildOfClass("Humanoid")
                if not hum then
                    pcall(function() hum = char:WaitForChild("Humanoid", 5) end)
                end
                if not hum or not _started or char ~= lp.Character then return end
                _lastHP = hum.Health
                _hcConn = hum.HealthChanged:Connect(function(h)
                    local prev = _lastHP or h
                    _lastHP = h
                    local drop = prev - h
                    if drop > 0.5 then FX.onIncoming(drop) end
                end)
            end)
        end

        function FX.start()
            if _started then return end
            _started = true
            allocate()
            ensureGui()
            hookHumanoid(lp.Character)
            _charConn = lp.CharacterAdded:Connect(function(c) if _started then hookHumanoid(c) end end)
            if not _updConn then
                _updConn = RunService.RenderStepped:Connect(function()
                    if _started then pcall(update) end
                end)
            end
        end

        function FX.stop()
            if not _started then return end
            _started = false
            if _updConn then _updConn:Disconnect(); _updConn = nil end
            if _hcConn then _hcConn:Disconnect(); _hcConn = nil end
            if _charConn then _charConn:Disconnect(); _charConn = nil end
            hideAllFX()   -- pools persist (fixed alloc); only hidden while disabled
        end

        function FX.destroy()
            FX.stop()
            local function rm(d) if d then pcall(function() d:Remove() end) end end
            if _hmLines then for _, d in ipairs(_hmLines) do rm(d) end _hmLines = nil end
            if _hmLinesBlk then for _, d in ipairs(_hmLinesBlk) do rm(d) end _hmLinesBlk = nil end
            if _dnPool then for _, d in ipairs(_dnPool) do rm(d) end _dnPool = nil end
            rm(_kbL1); rm(_kbL2); rm(_kbRing); _kbL1, _kbL2, _kbRing = nil, nil, nil
            if _ddArcs then
                for _, arc in ipairs(_ddArcs) do
                    for j = 1, 9 do rm(arc.red[j]); rm(arc.blk[j]) end
                end
                _ddArcs = nil
            end
            if _kfPool then for _, d in ipairs(_kfPool) do rm(d) end _kfPool = nil end
            if _hsLines then for _, d in ipairs(_hsLines) do rm(d) end _hsLines = nil end
            if _hsLinesBlk then for _, d in ipairs(_hsLinesBlk) do rm(d) end _hsLinesBlk = nil end
            rm(_fovA); rm(_fovB); rm(_fovFill); rm(_fovCas); _fovA, _fovB, _fovFill, _fovCas = nil, nil, nil, nil
            -- SIGNAL v4 · widget pools
            rm(_hmRing); rm(_hmRingBlk); _hmRing, _hmRingBlk = nil, nil
            if _chLines then for _, d in ipairs(_chLines) do rm(d) end _chLines = nil end
            if _chLinesBlk then for _, d in ipairs(_chLinesBlk) do rm(d) end _chLinesBlk = nil end
            rm(_chDot); rm(_chDotBlk); _chDot, _chDotBlk = nil, nil
            if _chTri then for _, d in ipairs(_chTri) do rm(d) end _chTri = nil end
            if _chTriBlk then for _, d in ipairs(_chTriBlk) do rm(d) end _chTriBlk = nil end
            rm(_wmBg); rm(_wmAccent); rm(_wmText); _wmBg, _wmAccent, _wmText = nil, nil, nil
            rm(_wm.stats); _wm.stats = nil
            rm(_tiBg); rm(_tiAccent); rm(_tiName); rm(_tiHpBg); rm(_tiHpFill); rm(_tiInfo)
            _tiBg, _tiAccent, _tiName, _tiHpBg, _tiHpFill, _tiInfo = nil, nil, nil, nil, nil, nil
            rm(_blHead); rm(_blBar); _blHead, _blBar = nil, nil
            if _blTexts then for _, d in ipairs(_blTexts) do rm(d) end _blTexts = nil end
            if _kfTicks then for _, d in ipairs(_kfTicks) do rm(d) end _kfTicks = nil end
            if _snds then for _, s in ipairs(_snds) do pcall(function() s:Destroy() end) end _snds = nil end
            if _gui then pcall(function() _gui:Destroy() end); _gui = nil end
            _flashFrame = nil; _vgFrames = nil
            _alloc = false
        end
    end)()

    -- ── shared HIT CONTEXT: built ONCE per registered hit, fanned out to every feedback ────────
    local _hpPrev = {}
    function Visuals.notifyTarget(p, info)
        if not p or p == lp or not p.Character then return end
        local cur, maxHP = getHealth(p)
        local dmg = math.max(0, (_hpPrev[p] or maxHP) - cur)
        _hpPrev[p] = cur
        local lethal = (not isAlive(p)) or cur <= 0
        local char = p.Character
        local rp = char:FindFirstChild("HitboxHead")          -- SAME resolve chain as createHologram
            or char:FindFirstChild("Head")
            or char:FindFirstChild("HumanoidRootPart")
            or char:FindFirstChild("UpperTorso")
        local hitPos = rp and rp.Position or nil
        local crit = (info and info.crit) or (dmg >= cfg("FXCritDamage", 30))
        -- afterimage (independent of the lighting master toggle, same 0.35s per-target cooldown)
        if Config.VisualsHolograms then
            local now = tick()
            if (now - (_hologramCooldowns[p] or 0)) >= 0.35 then
                _hologramCooldowns[p] = now
                createHologram(char, lethal)
            end
        end
        FX.onHit(p, dmg, crit, lethal, hitPos)
    end

    -- ── CAMERA FRAME FX — cinematic vignette / letterbox / depth-of-field. Purely local, hook-free:
    -- vignette + letterbox are ONE ScreenGui of static frames (zero per-frame cost — writes happen only
    -- on toggle/slider), DoF is a Lighting DepthOfFieldEffect with its own attribute (VS_DoF) so a
    -- preset switch's clearTagged can't reap it. Follows the Visuals master; DoF also stands down in
    -- Performance mode. Nested IIFE keeps ~15 locals off this proto's registers (kill-flourish pattern).
    local applyCamFrame, clearCamFrame
    ;(function()
        local TweenService = game:GetService("TweenService")
        local _camGui, _vgFrames, _lbTop, _lbBot, _dof = nil, nil, nil, nil, nil
        local C_BLACK_FRAME = Color3.new(0, 0, 0)

        local function ensureCamGui()
            if _camGui and _camGui.Parent then return _camGui end
            local g = Instance.new("ScreenGui")
            g.Name = "_vs_cam"
            g.IgnoreGuiInset = true; g.ResetOnSpawn = false
            g.DisplayOrder = 990   -- under the FX HUD layer (999): combat feedback renders over the frame
            local ok = pcall(function() g.Parent = (gethui and gethui()) or game:GetService("CoreGui") end)
            if not ok or not g.Parent then
                pcall(function() g.Parent = lp:FindFirstChildOfClass("PlayerGui") end)
            end
            _camGui = g
            return g
        end

        -- vignette: 4 edge frames, each a UIGradient fading inward (asset-free; the strength dial
        -- drives edge opacity only, so a slider drag is 4 property writes)
        local VG_SIDES = {
            { size = UDim2.new(1, 0, 0.24, 0),  pos = UDim2.new(0, 0, 0, 0),     rot = 90  },
            { size = UDim2.new(1, 0, 0.24, 0),  pos = UDim2.new(0, 0, 0.76, 0),  rot = 270 },
            { size = UDim2.new(0.17, 0, 1, 0),  pos = UDim2.new(0, 0, 0, 0),     rot = 0   },
            { size = UDim2.new(0.17, 0, 1, 0),  pos = UDim2.new(0.83, 0, 0, 0),  rot = 180 },
        }
        local function vgApply()
            local on = Config.Visuals and cfg("VisualsVignette", false)
            if not on then
                if _vgFrames then
                    for i = 1, #_vgFrames do _vgFrames[i].Visible = false end
                end
                return
            end
            if not _vgFrames then
                local g = ensureCamGui()
                _vgFrames = {}
                for i = 1, #VG_SIDES do
                    local s = VG_SIDES[i]
                    local f = Instance.new("Frame")
                    f.Name = "_cv" .. i
                    f.BackgroundColor3 = C_BLACK_FRAME; f.BorderSizePixel = 0
                    f.Size = s.size; f.Position = s.pos; f.Visible = false; f.ZIndex = 1
                    local grad = Instance.new("UIGradient")
                    grad.Rotation = s.rot
                    grad.Transparency = NumberSequence.new(0, 1)   -- opaque at the edge, clear inward
                    grad.Parent = f
                    f.Parent = g
                    _vgFrames[i] = f
                end
            end
            local tr = 1 - 0.85 * math.clamp(cfg("VisualsVignetteStrength", 0.6), 0, 1)
            for i = 1, #_vgFrames do
                local f = _vgFrames[i]
                f.BackgroundTransparency = tr; f.Visible = true
            end
        end

        -- letterbox: 2 solid bars that slide in/out (tween on toggle only; static while shown)
        local LB_TI = TweenInfo.new(0.3, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
        local function lbApply(animate)
            local on = Config.Visuals and cfg("VisualsLetterbox", false)
            if not on and not _lbTop then return end   -- never shown yet: nothing to hide
            if not _lbTop then
                local g = ensureCamGui()
                for i = 1, 2 do
                    local f = Instance.new("Frame")
                    f.Name = "_clb" .. i
                    f.BackgroundColor3 = C_BLACK_FRAME; f.BackgroundTransparency = 0
                    f.BorderSizePixel = 0; f.ZIndex = 2
                    f.Parent = g
                    if i == 1 then _lbTop = f else _lbBot = f end
                end
                _lbTop.Position = UDim2.new(0, 0, -0.2, 0)
                _lbBot.Position = UDim2.new(0, 0, 1, 0)
            end
            local sz = math.clamp(cfg("VisualsLetterboxSize", 0.10), 0.04, 0.18)
            local topP = on and UDim2.new(0, 0, 0, 0)      or UDim2.new(0, 0, -sz - 0.02, 0)
            local botP = on and UDim2.new(0, 0, 1 - sz, 0) or UDim2.new(0, 0, 1.02, 0)
            _lbTop.Size = UDim2.new(1, 0, sz, 0); _lbBot.Size = UDim2.new(1, 0, sz, 0)
            if animate then
                pcall(function()
                    TweenService:Create(_lbTop, LB_TI, { Position = topP }):Play()
                    TweenService:Create(_lbBot, LB_TI, { Position = botP }):Play()
                end)
            else
                _lbTop.Position = topP; _lbBot.Position = botP
            end
        end

        -- depth of field: focus plane at FocusDistance, blur dial maps to Far/Near intensity.
        -- Engine post effect — code-tuned defaults (28-stud focus, 22-stud sharp band); needs a live glance.
        local function dofApply()
            local on = Config.Visuals and not Config.VisualsPerformanceMode and cfg("VisualsDOF", false)
            if not on then
                if _dof then pcall(function() _dof:Destroy() end); _dof = nil end
                return
            end
            if not (_dof and _dof.Parent) then
                local d = Instance.new("DepthOfFieldEffect")
                d.Name = "_vs_dof"; d:SetAttribute("VS_DoF", true)
                d.InFocusRadius = 22
                d.Parent = Lighting
                _dof = d
            end
            local blur = math.clamp(cfg("VisualsDOFBlur", 0.5), 0, 1)
            pcall(function()
                _dof.FocusDistance = math.clamp(cfg("VisualsDOFDistance", 28), 5, 100)
                _dof.FarIntensity  = 0.75 * blur
                _dof.NearIntensity = 0.5  * blur
            end)
        end

        applyCamFrame = function(animate) vgApply(); lbApply(animate); dofApply() end
        clearCamFrame = function()
            if _camGui then pcall(function() _camGui:Destroy() end) end
            _camGui, _vgFrames, _lbTop, _lbBot = nil, nil, nil, nil
            if _dof then pcall(function() _dof:Destroy() end); _dof = nil end
        end

        function Visuals.setVignette(on) Config.VisualsVignette = on; vgApply() end
        function Visuals.setVignetteStrength(v)
            Config.VisualsVignetteStrength = math.clamp(v, 0, 1); vgApply()
        end
        function Visuals.setLetterbox(on) Config.VisualsLetterbox = on; lbApply(true) end
        function Visuals.setLetterboxSize(v)
            Config.VisualsLetterboxSize = math.clamp(v, 0.04, 0.18); lbApply(false)
        end
        function Visuals.setDOF(on) Config.VisualsDOF = on; dofApply() end
        function Visuals.setDOFDistance(v)
            Config.VisualsDOFDistance = math.clamp(v, 5, 100); dofApply()
        end
        function Visuals.setDOFBlur(v)
            Config.VisualsDOFBlur = math.clamp(v, 0, 1); dofApply()
        end
    end)()

    -- Both Visuals camera writes ride ONE RenderStep (priority Last) so they share the single Rage
    -- stand-down guard and can never contend for Camera.CFrame. Both are PURELY LOCAL cosmetic writes
    -- (no camera-fn hooks): stretch = aspect skew, sway = sub-degree handheld "breathing" rotation.
    local function bindStretch()
        if _stretchBound then return end
        _stretchBound = true
        RunService:BindToRenderStep("VS_Stretch", Enum.RenderPriority.Last.Value, function()
            if not Config.Visuals then return end
            -- never fight Rage's camera lock (RageCameraLockOnHide) — stand down entirely while Rage owns it.
            if Config.Rage then return end
            local s = Config.VisualsStretch
            local doStretch = math.abs(s - 1.0) >= 0.001
            local doSway = Config.VisualsCameraSway
            if not (doStretch or doSway) then return end   -- idle: no camera write this frame
            local c = Camera.CFrame
            if doSway then
                -- layered sines about the camera's own axes = organic handheld drift (never a loop-visible
                -- period). Amplitudes are sub-degree at the 0.5 default; the dial reaches ~1.6° roll at 1.0.
                local amt = math.clamp(Config.VisualsCameraSwayAmount or 0.5, 0, 1)
                local t = tick()
                local roll  = (math.sin(t * 0.9) + math.sin(t * 0.37) * 0.6) * amt
                local pitch =  math.sin(t * 1.3) * 0.7 * amt
                local yaw   =  math.sin(t * 0.7) * 0.8 * amt
                c = c * CFrame.Angles(math.rad(pitch), math.rad(yaw), math.rad(roll))
            end
            if doStretch then
                c = CFrame.fromMatrix(c.Position, c.RightVector * s, c.UpVector)
            end
            Camera.CFrame = c
        end)
    end

    function Visuals.setStretch(v)
        Config.VisualsStretch = math.clamp(v, Config.VisualsStretchMin, Config.VisualsStretchMax)
    end

    local function startRainbow()
        if _rainbowConn then return end
        _rainbowBatchIdx = 1; table.clear(_rainbowParts)
        -- bug#1 rank4: was O(descendants × players) — GetPlayerFromCharacter() per BasePart on a real map
        -- (tens of thousands of parts) synchronously stalled the client. Snapshot player character models
        -- into a set ONCE, then do an O(1) parent-membership test per part → O(descendants + players).
        local charSet = {}
        for _, pl in ipairs(Players:GetPlayers()) do
            if pl.Character then charSet[pl.Character] = true end
        end
        for _, d in ipairs(Workspace:GetDescendants()) do
            if d:IsA("BasePart") and not d:GetAttribute("VS_Holo")
                and not charSet[d.Parent]
                and d.Name ~= "Terrain" then
                table.insert(_rainbowParts, { part = d, originalColor = d.Color })
            end
        end
        _rainbowConn = RunService.Heartbeat:Connect(function(dt)
            if not Config.Visuals or not Config.VisualsRainbowMap then return end
            _rainbowHue = (_rainbowHue + dt * Config.VisualsRainbowMapSpeed) % 1
            local total = #_rainbowParts; if total == 0 then return end
            local batch = math.min(250, total)
            for i = 1, batch do
                local idx = ((_rainbowBatchIdx - 1 + i - 1) % total) + 1
                local e   = _rainbowParts[idx]
                if e and e.part and e.part.Parent then
                    e.part.Color = Color3.fromHSV((_rainbowHue + (idx / total) * 0.3) % 1, 0.85, 1)
                end
            end
            _rainbowBatchIdx = ((_rainbowBatchIdx + batch - 1) % total) + 1
        end)
    end

    local function stopRainbow()
        if _rainbowConn then _rainbowConn:Disconnect(); _rainbowConn = nil end
        for _, e in ipairs(_rainbowParts) do
            if e.part and e.part.Parent then pcall(function() e.part.Color = e.originalColor end) end
        end
        table.clear(_rainbowParts)
    end

    function Visuals.toggleRainbowMap(on)
        Config.VisualsRainbowMap = on
        if on and Config.Visuals then startRainbow() else stopRainbow() end
    end

    local function applyPerf()
        if not _perfBackup then
            _perfBackup = {
                GlobalShadows = Lighting.GlobalShadows,
                EnvironmentDiffuseScale = Lighting.EnvironmentDiffuseScale,
                EnvironmentSpecularScale = Lighting.EnvironmentSpecularScale,
                Brightness = Lighting.Brightness,
                ShadowSoftness = Lighting.ShadowSoftness,
            }
        end
        clearTagged(); clearGrade(); clearBloom()   -- perf mode strips ALL post-FX (own-attribute FX survive clearTagged)
        Lighting.GlobalShadows = false; Lighting.EnvironmentDiffuseScale = 0
        Lighting.EnvironmentSpecularScale = 0; Lighting.Brightness = 2
        Lighting.ShadowSoftness = 0
        for _, d in ipairs(Workspace:GetDescendants()) do
            if d:IsA("ParticleEmitter") and not d:GetAttribute("VS_Holo") then
                if not _origParticleRates[d] then _origParticleRates[d] = d.Rate end
                d.Rate = 0
            end
        end
    end

    local function disablePerf()
        if _perfBackup then
            for k, v in pairs(_perfBackup) do pcall(function() Lighting[k] = v end) end
            _perfBackup = nil
        end
        for em, rate in pairs(_origParticleRates) do
            if em and em.Parent then pcall(function() em.Rate = rate end) end
        end
        table.clear(_origParticleRates)
        if State.VisualsCurrentPreset then applyPreset(State.VisualsCurrentPreset) end
    end

    function Visuals.togglePerf(on)
        Config.VisualsPerformanceMode = on
        if on then applyPerf() else disablePerf() end
        applyCamFrame(false)   -- DoF stands down under perf mode (vignette/letterbox are free, stay)
    end

    function Visuals.setPreset(name)
        if Presets[name] then
            Config.VisualsPreset = name   -- always persist the choice (even if Visuals is off)
            applyPreset(name)             -- applies now if Visuals is enabled
        end
    end
    function Visuals.toggleHolograms(on) Config.VisualsHolograms = on end
    function Visuals.setHologramStyle(name) Config.VisualsHologramStyle = name end
    Visuals.HologramStyleOrder = { "Orb", "Skeleton", "Wraith" }

    Visuals.GradeOrder = { "None", "Crisp", "Cold", "Warm", "Comp" }
    function Visuals.setGrade(name)
        Config.VisualsGrade = name
        if not Config.Visuals or Config.VisualsPerformanceMode then return end
        reassertGrade()
    end
    function Visuals.setGradeStrength(v)
        Config.VisualsGradeStrength = math.clamp(v, 0, 1)
        if not Config.Visuals or Config.VisualsPerformanceMode then return end
        reassertGrade()
    end

    function Visuals.setBloom(on)
        Config.VisualsBloom = on
        if not Config.Visuals or Config.VisualsPerformanceMode then return end
        reassertBloom()
    end
    function Visuals.setBloomIntensity(v)
        Config.VisualsBloomIntensity = math.clamp(v, 0, 3)
        if not Config.Visuals or Config.VisualsPerformanceMode then return end
        reassertBloom()
    end

    function Visuals.toggleFullbright(on)
        Config.VisualsFullbright = on
        if not Config.Visuals or Config.VisualsPerformanceMode then return end
        if on then applyFullbrightOverride()
        else applyPreset(State.VisualsCurrentPreset or Config.VisualsPreset or "Neutral") end
    end
    function Visuals.toggleNoFog(on)
        Config.VisualsNoFog = on
        if not Config.Visuals or Config.VisualsPerformanceMode then return end
        if on then applyFogOverride()
        else applyPreset(State.VisualsCurrentPreset or Config.VisualsPreset or "Neutral") end
    end

    -- ── HUD MASTER (STRUCTURAL): the screen-feedback subsystem (Visuals.FX) is an INDEPENDENT master,
    -- no longer gated behind the World/Lighting master. 12_init starts it when Config.HUD; the HUD tab's
    -- master toggle drives enableHUD/disableHUD; FX.onHit/onIncoming self-gate on the internal _started flag.
    function Visuals.enableHUD()  Config.HUD = true;  FX.start() end
    function Visuals.disableHUD() Config.HUD = false; FX.stop()  end

    function Visuals.init()   snapshotLighting(); if Config.Visuals then bindStretch() end end
    function Visuals.enable()
        Config.Visuals = true; applyPreset(Config.VisualsPreset or "Neutral")
        if not Config.VisualsPerformanceMode then
            applyFullbrightOverride(); applyFogOverride()  -- immediate; loop keeps them alive
        end
        bindStretch()                 -- FPS: only bind the camera-stretch RenderStep while Visuals is on
        if Config.VisualsRainbowMap then startRainbow() end
        if Config.VisualsPerformanceMode then applyPerf() end
        startReassert()
        applyCamFrame(false)          -- vignette/letterbox/DoF follow the master
        -- FX (HUD hit-feedback) is DECOUPLED — its own master (Config.HUD / enableHUD) drives it now.
    end
    function Visuals.disable()
        Config.Visuals = false; stopRainbow(); stopReassert()
        -- FX left running: the HUD master owns it independently of the World/Lighting master.
        clearGrade(); clearBloom(); clearCamFrame()
        if Config.VisualsPerformanceMode then disablePerf() end
        if _stretchBound then
            pcall(function() RunService:UnbindFromRenderStep("VS_Stretch") end)
            _stretchBound = false
        end
        restore()
    end
    function Visuals.unload()
        Visuals.disable(); stopReassert()
        FX.destroy()
        clearKillPulse()
        -- reclaim the world-FX folder (parked tracer rigs + any in-flight bursts); active conns
        -- self-disconnect on the model.Parent guard next frame
        if _hologramFolder then pcall(function() _hologramFolder:Destroy() end); _hologramFolder = nil end
        if _stretchBound then
            pcall(function() RunService:UnbindFromRenderStep("VS_Stretch") end)
            _stretchBound = false
        end
    end
end)()

-- End of Part 1 -- Part 2 follows in the next message.
-- ── WEATHER / ENVIRONMENT ───────────────────────────────────────────────────
-- World-FX layer alongside the Visuals lighting presets (06_visuals). Independent, save/restore-
-- clean subsystems, each self-gating on its own config master:
--   • Weather — camera-following precipitation. RAIN is a real geometry sim: a POOL of thin neon
--     Beam "streaks" that fall through a volume around the player and recycle — a Beam is a line, so
--     it renders as a crisp streak with NO texture dependency (the old sprite approach read as blobs).
--     SNOW / PETALS / ASH / EMBERS / FIREFLIES / MIST use volume-emitted ParticleEmitters (soft round
--     sprites are correct for those). Optional per-type mood colour grade + storm layer (procedural
--     lightning + sky flash + delayed thunder).
--   • Skybox — swap the game Sky for a preset (or restore), with sun/moon/star disabling.
-- NO camera CFrame is ever written (the follow loop only READS Camera.CFrame.Position), so this never
-- contends with Rage's camera lock or the Visuals stretch RenderStep. Every Instance is tagged
-- (WX_Custom) and reclaimed on disable.
local Weather = {}
-- Weather internals run in their own function proto (same pattern as 06_visuals/07a): keeps its ~40
-- named locals off the shared main-chunk register budget (a bare `do` would leave them live in the
-- main chunk and — with the pass-2 additions — risk the regcheck 150-main-chunk-local cap). Weather
-- (the export table) stays the outer local; every symbol used inside is an earlier-defined upvalue.
;(function()
    local SoundService = game:GetService("SoundService")
    local TweenService = game:GetService("TweenService")

    local function cfg(key, default)
        local v = Config[key]
        if v == nil then return default end
        return v
    end

    -- ── INTENSITY RESPONSE ──────────────────────────────────────────────────
    -- A LINEAR slider driving a LINEAR rate (`rate * I`) is why the top of the bar read
    -- flat: perceived density is logarithmic, so `rate ∝ I` spent ln(1/0.15)=1.90 of
    -- perceptual range over the bottom 46% of the bar and only ln(2)=0.69 over the top
    -- 54%. An EXPONENTIAL response makes equal slider travel = equal perceived change.
    -- kLo (below 1) is steeper than kHi (above 1) on purpose: the bottom has to reach
    -- "almost off" while the top stays inside the particle budget — the top's drama is
    -- bought from fall speed, size, opacity and wind coupling instead of from raw count.
    -- Every response is anchored at exactly 1.0 for I = 1, so the approved default look
    -- of every type is byte-identical to before. AMBIENT ONLY — see evRate.
    local function iAmt()
        return math.clamp(cfg("WeatherIntensity", 1), 0.15, 2)
    end
    local function iRate(I, kLo, kHi)
        if I < 1 then
            return math.exp(kLo * (I - 1))
        end
        return math.exp(kHi * (I - 1))
    end
    -- SKY EVENTS are a rate, not a density — intensity does not reach them. Frequency and
    -- density are separate axes (light drizzle + frequent meteors has to be reachable), so
    -- each sky event owns a multiplier that DIVIDES its interval: higher = more often,
    -- 1 = the stock cadence. Storm uses its own WeatherStormMin/Var seconds instead.
    local function evRate(key)
        return math.clamp(cfg(key, 1), 0.25, 3)
    end
    -- Per-type response coefficients. lo/hi = rate exponents; acc/sz/alp/gst are linear
    -- coefficients b in the multiplier (1 - b) + b*I, so every one is 1.0 at I = 1.
    --   acc = fall/drive (Acceleration + Speed)   sz = sprite size
    --   alp = OPACITY multiplier (transparency is recomputed as 1 - (1-t)*m)
    --   gst = wind coupling (per-layer gust ax/az)
    -- sz is deliberately tiny on the leaf/petal/snow types: their hero layers carry a
    -- hard-won size cap, and ±8% is already inside the per-particle size envelope those
    -- keypoints ship with. The fog types have no such cap and scale size properly.
    local IR = {
        Snow      = { lo = 1.75, hi = 0.85, acc = 0.45, sz = 0.08, alp = 0.35, gst = 0.30 },
        Petals    = { lo = 1.75, hi = 0.85, acc = 0.40, sz = 0.08, alp = 0.30, gst = 0.30 },
        Autumn    = { lo = 1.75, hi = 0.85, acc = 0.40, sz = 0.08, alp = 0.30, gst = 0.30 },
        Mist      = { lo = 1.35, hi = 1.10, acc = 0.30, sz = 0.25, alp = 0.75, gst = 0.20 },
        Ash       = { lo = 1.45, hi = 1.05, acc = 0.35, sz = 0.15, alp = 0.55, gst = 0.20 },
        Sandstorm = { lo = 1.35, hi = 1.60, acc = 0.45, sz = 0.30, alp = 0.75, gst = 0.20 },
        Embers    = { lo = 1.45, hi = 1.05, acc = 0.30, sz = 0.10, alp = 0.25, gst = 0.20 },
        Fireflies = { lo = 1.45, hi = 1.00, acc = 0.10, sz = 0.08, alp = 0.20, gst = 0.15 },
    }
    local IR_DEF = { lo = 1.6, hi = 0.9, acc = 0.3, sz = 0.10, alp = 0.25, gst = 0.25 }
    -- ground carpets ride a much gentler rate curve: a settled layer is accumulation, not
    -- fall rate, so it must thin without ever leaving the floor bare-grey
    local SET_LO, SET_HI = 0.75, 0.45

    local Vec   = Vector3.new
    local WHITE = Color3.new(1, 1, 1)
    local TX_SOFT  = "rbxasset://textures/particles/smoke_main.dds"
    local TX_SPARK = "rbxasset://textures/particles/sparkles_main.dds"
    -- TX kit ids are RESOLVED image ids (Decal ids set from code render invisible).
    local TX_FLAKE    = "rbxassetid://101749163393113"   -- ornate 6-arm lace flake (hero/body scales verified)
    local TX_SNOWSOFT = "rbxassetid://78582616787441"    -- clean soft radial disc (bokeh/far haze/firefly motes)
    local TX_PUFF     = "rbxassetid://77935321198144"    -- billowy smoke puff (firefly ground mist)
    local TX_PETAL    = "rbxassetid://243344623"         -- single soft-pink sakura petal
    local TX_LEAFA    = "rbxassetid://8590047664"        -- bright orange maple leaf (Autumn species A)
    local TX_LEAFB    = "rbxassetid://5970677338"        -- fiery red maple leaf (Autumn species B)
    local TX_LEAFC    = "rbxassetid://9239040931"        -- dark red-brown maple (Autumn species C: the value anchor)
    local TX_DUST     = "rbxassetid://341828512"         -- warm tan dust billow (leaf-devil ground kick)

    -- active weather type — declared UP HERE so the rain sim + follow + apply closures below all
    -- capture the SAME upvalue (not a stray global).
    local _curType = nil

    -- ════════════════════════════════════════════════════════════════════════
    -- WIND CONTROLLER — ONE shared wind field for every weather effect.
    -- Direction slowly veers (summed slow sines around a random base heading);
    -- strength breathes (base + three sines, ~0.3..1.7) and spikes ×2-3 on
    -- scheduled gust events (every 8-25s, 2-4s smooth sine envelope).
    -- Wind.update mutates x/z IN PLACE — zero per-frame allocation; consumers
    -- compose their own Vector3 (they write Acceleration anyway). Driven from
    -- the follow loop only, so it runs ONLY while weather is enabled and dies
    -- with stopFollow; Wind.reset re-seeds the gust schedule on every enable.
    local Wind = { x = 0, z = 0 }
    ;(function()
        local baseAng = math.random() * 6.283
        local gustT0, gustDur, gustAmp = 0, 1, 0
        local nextGustT = math.huge
        local cbs = {}

        -- cb fired at gust START with (duration, peakMultiplier) — leaf bursts /
        -- wind streaks hook here. Registry is load-time + permanent (no unhook).
        function Wind.onGust(fn)
            cbs[#cbs + 1] = fn
        end
        function Wind.reset(now)
            nextGustT = now + 8 + math.random() * 17
            gustT0, gustAmp = 0, 0
            Wind.x, Wind.z = 0, 0
        end
        function Wind.update(now)
            if now >= nextGustT then
                gustT0 = now
                gustDur = 2 + math.random() * 2
                gustAmp = 1 + math.random()               -- envelope peak = ×2..×3
                nextGustT = now + 8 + math.random() * 17
                for i = 1, #cbs do
                    pcall(cbs[i], gustDur, 1 + gustAmp)
                end
            end
            local ang = baseAng + 0.6 * math.sin(now * 0.013) + 0.9 * math.sin(now * 0.031 + 2.6)
            local str = 1 + 0.35 * math.sin(now * 0.17) + 0.22 * math.sin(now * 0.41 + 1.3)
                          + 0.15 * math.sin(now * 0.07 + 4.1)
            local ga = (now - gustT0) / gustDur
            if gustAmp > 0 and ga < 1 then
                str = str * (1 + gustAmp * math.sin(ga * math.pi))   -- smooth ramp up + down
            end
            Wind.x = math.cos(ang) * str
            Wind.z = math.sin(ang) * str
        end
        -- kept: documented infra API (INFRA_API.md), currently unused (allocates: call ≤ once/frame)
        function Wind.vec()
            return Vec(Wind.x, 0, Wind.z)
        end
    end)()

    -- ── host folder ──
    local _folder = nil
    local function getFolder()
        if _folder and _folder.Parent then return _folder end
        local f = Instance.new("Folder"); f.Name = "_wx"; f:SetAttribute("WX_Custom", true)
        f.Parent = Workspace
        _folder = f; return f
    end

    -- ════════════════════════════════════════════════════════════════════════
    -- RAIN — geometry streak sim (pool of Beams). Cannot look like blobs.
    -- ════════════════════════════════════════════════════════════════════════
    local _rain = { drops = {}, conn = nil, folder = nil, dir = nil, applied = nil }
    local RAIN_DIR      = Vec(-0.16, -1, 0.05).Unit    -- seed slant only; live slant follows Wind
    local RAIN_LEAN_K   = 0.12                         -- wind strength → horizontal lean gain
    local RAIN_LEAN_MAX = 0.2126                       -- tan(12°): slant cap vs vertical
    local RAIN_RADIUS = 70                             -- XZ spawn disc around camera
    local RAIN_TOP    = 70                             -- studs above camera to spawn
    local RAIN_BOT    = -28                            -- recycle when this far below camera
    -- pool ceiling. The old 240 was reached at I = 1.60, so the top fifth of the slider
    -- was a literal no-op. The exponential response peaks at 350 drops (I = 2.0), so this
    -- is headroom, not a working limit — it only exists so a bad Config value can never
    -- ask for an unbounded pool.
    local RAIN_MAX    = 420

    local function rainFolder()
        if _rain.folder and _rain.folder.Parent then return _rain.folder end
        local f = Instance.new("Folder"); f.Name = "_wxRain"; f:SetAttribute("WX_Custom", true)
        f.Parent = getFolder()
        _rain.folder = f; return f
    end

    local function makeDrop(parent, streakLen, width, color, glow, transp)
        local part = Instance.new("Part")
        part.Anchored = true; part.CanCollide = false; part.CanQuery = false; part.CanTouch = false
        part.CastShadow = false; part.Massless = true; part.Transparency = 1; part.Size = Vec(0.05,0.05,0.05)
        part:SetAttribute("WX_Custom", true)
        local a0 = Instance.new("Attachment"); a0.Parent = part
        local a1 = Instance.new("Attachment"); a1.Position = _rain.dir * streakLen; a1.Parent = part
        local beam = Instance.new("Beam")
        beam.Attachment0 = a0; beam.Attachment1 = a1
        beam.Segments = 1; beam.FaceCamera = true
        beam.Width0 = width; beam.Width1 = width * 0.55   -- slight taper = "falling" feel
        local emission = 0.35
        if glow then
            emission = 1
        end
        beam.LightEmission = emission
        beam.LightInfluence = 0
        beam.Color = ColorSequence.new(color)
        beam.Transparency = NumberSequence.new(transp)
        beam.Parent = part
        part.Parent = parent
        return part, a1, beam
    end

    local function seedDrop(d, camPos)
        local ang = math.random() * math.pi * 2
        local rad = math.sqrt(math.random()) * RAIN_RADIUS   -- uniform disc
        local y   = camPos.Y + RAIN_TOP - math.random() * (RAIN_TOP - RAIN_BOT)  -- stagger down the column
        d.pos = Vec(camPos.X + math.cos(ang) * rad, y, camPos.Z + math.sin(ang) * rad)
    end

    -- one drop, at BASE (I = 1) geometry — tuneRain applies the intensity response.
    -- Two visual tiers: heavy near streaks + finer far streaks (parallax depth).
    local function newDrop(folder, i, camPos)
        local base, width, color, transp, spd0
        if (i % 3) ~= 0 then
            base, width, transp = 5 + math.random() * 3, 0.10, 0.22
            spd0 = 150 + math.random() * 30
            color = Color3.fromRGB(180, 202, 232)
        else
            base, width, transp = 3 + math.random() * 2, 0.06, 0.55
            spd0 = 120 + math.random() * 30
            color = Color3.fromRGB(158, 182, 214)
        end
        local part, a1, beam = makeDrop(folder, base, width, color, false, transp)
        local d = { part = part, a1 = a1, beam = beam, base = base, len = base,
                    w0 = width, t0 = transp, spd0 = spd0, spd = spd0 }
        seedDrop(d, camPos)
        part.CFrame = CFrame.new(d.pos)
        return d
    end

    -- INTENSITY, applied IN PLACE. Count grows/trims and every surviving drop is retuned,
    -- so a slider drag thickens the rain instead of teleporting a freshly re-seeded field
    -- (the old path destroyed the whole folder on every step). Speed and streak length
    -- move TOGETHER — a streak length IS speed × exposure, so scaling one alone reads wrong.
    local function tuneRain()
        local folder = rainFolder()
        local I = iAmt()
        local n = math.clamp(math.floor(150 * iRate(I, 1.75, 0.85)), 16, RAIN_MAX)
        local drops = _rain.drops
        local camPos = Vec(0, 0, 0)
        if Camera then
            camPos = Camera.CFrame.Position
        end
        for i = #drops, n + 1, -1 do
            drops[i].part:Destroy()
            drops[i] = nil
        end
        for i = #drops + 1, n do
            drops[i] = newDrop(folder, i, camPos)
        end
        local sm = 0.55 + 0.45 * I                     -- fall speed + streak length
        local wm = 0.85 + 0.15 * I                     -- streak width
        local am = 0.78 + 0.22 * I                     -- opacity
        local dir = _rain.applied or _rain.dir or RAIN_DIR
        for i = 1, n do
            local d = drops[i]
            d.spd = d.spd0 * sm
            d.len = d.base * sm
            d.a1.Position = dir * d.len
            local w = d.w0 * wm
            d.beam.Width0 = w
            d.beam.Width1 = w * 0.55
            d.beam.Transparency = NumberSequence.new(math.clamp(1 - (1 - d.t0) * am, 0.02, 1))
        end
    end

    local function buildRain()
        -- clear old
        if _rain.folder then pcall(function() _rain.folder:Destroy() end); _rain.folder = nil end
        table.clear(_rain.drops)
        if not _rain.dir then _rain.dir = RAIN_DIR end   -- keep a live slant across intensity rebuilds
        _rain.applied = _rain.dir
        tuneRain()
    end

    local function startRain()
        if _rain.conn then return end
        _rain.conn = RunService.Heartbeat:Connect(function(dt)
            if not Config.Weather or _curType ~= "Rain" then return end
            local cam = Camera; if not cam then return end
            local camPos = cam.CFrame.Position
            -- wind slant: damped follow of the shared wind lean (no gust jitter), capped ~12°
            local lx, lz = Wind.x * RAIN_LEAN_K, Wind.z * RAIN_LEAN_K
            local lm = math.sqrt(lx * lx + lz * lz)
            if lm > RAIN_LEAN_MAX then
                local s = RAIN_LEAN_MAX / lm
                lx, lz = lx * s, lz * s
            end
            local step = _rain.dir:Lerp(Vec(lx, -1, lz).Unit, math.min(dt * 2, 1)).Unit
            _rain.dir = step
            if (step - _rain.applied).Magnitude > 0.015 then   -- retilt streaks only on real change
                _rain.applied = step
                for i = 1, #_rain.drops do
                    local d = _rain.drops[i]
                    d.a1.Position = step * d.len
                end
            end
            local r2 = RAIN_RADIUS * RAIN_RADIUS
            for i = 1, #_rain.drops do
                local d = _rain.drops[i]
                local p = d.pos + step * (d.spd * dt)
                local relX, relY, relZ = p.X - camPos.X, p.Y - camPos.Y, p.Z - camPos.Z
                if relY < RAIN_BOT or (relX * relX + relZ * relZ) > r2 then
                    seedDrop(d, camPos)   -- recycle to the top at a fresh spot
                    p = d.pos
                end
                d.pos = p
                if d.part then d.part.CFrame = CFrame.new(p) end
            end
        end)
    end
    local function stopRain()
        if _rain.conn then _rain.conn:Disconnect(); _rain.conn = nil end
        if _rain.folder then pcall(function() _rain.folder:Destroy() end); _rain.folder = nil end
        table.clear(_rain.drops)
        _rain.dir = nil; _rain.applied = nil   -- next build re-seeds from RAIN_DIR
    end

    -- ════════════════════════════════════════════════════════════════════════
    -- PARTICLE TYPES — volume-emitted soft sprites (correct for snow/petals/etc.)
    -- VISIBILITY MODEL (the old presets' shared bug): a falling particle only reads on screen if
    -- it CROSSES EYE LEVEL within its lifetime. Terminal fall speed ≈ |accel.Y| / (drag·ln2), so
    -- the spawn height must satisfy oy/terminal < ~0.6·life — the old specs spawned 42-80 studs
    -- up with ~3-6 studs/s terminals and 4-10s lives, so almost everything faded out ABOVE THE
    -- PLAYER'S HEAD (hence "barely visible"). Falling presets now spawn 16-42 studs up with
    -- matched speed/life, and sizes are tuned to subtend real pixels at each layer's distance.
    -- spec extras: color accepts a ColorSequence; tkeys = explicit transparency keypoints
    -- ({t,v} pairs, overrides transp0/transp1) — powers the firefly double-blink; skeys = size
    -- keypoints with a per-particle random envelope ({t,v,env}, overrides size0/size1) — powers
    -- the snow flake-size variety.
    -- ════════════════════════════════════════════════════════════════════════
    local ND = Enum.NormalId
    -- Petal flutter: Squash alternating edge-on ↔ flat across the particle's life, so a petal
    -- reads as tipping over and over instead of spinning as a rigid card. Built ONCE at load
    -- (NumberSequence is immutable — the preset table just holds the finished object).
    local function flutterSeq(lo, hi, n, env, flip)
        local kps = table.create(n + 1)
        for i = 0, n do
            local v = lo
            if (i + flip) % 2 == 1 then
                v = hi
            end
            kps[i + 1] = NumberSequenceKeypoint.new(i / n, v, env)
        end
        return NumberSequence.new(kps)
    end
    local PRESETS = {
        Snow = {
            -- Alpine hush, 5-layer parallax. HERO = very sparse LARGE in-focus lace flakes
            -- drifting close to camera (the lens-focus layer); BOKEH = few SMALL dim out-of-focus
            -- discs — a soft disc near the lens must stay small AND low-emission or it reads as
            -- floating cotton (same blow-out the Petals bokeh hit at glow 0.22), and only the
            -- lace-flake layers may carry size this close; BODY carries the readable shaped
            -- snowfall; FAR is soft atmospheric depth (small + dense, so it stays a veil);
            -- GLINTS = rare crystalline sparkle. Cool split near→far: white → ice
            -- blue. Base accel is PURE fall — all lateral drift comes from the shared Wind
            -- via each layer's gust field, so gusts sweep every layer as one coherent wave.
            -- Studio-verified 2026-07-24 (TX-kit lace flake replaces the old soft-disc-only
            -- look; slower fall + higher drag + tumble/squash flutter).
            { size=Vec(22,4,22), oy=5, tex=TX_FLAKE, color=WHITE,                            -- HERO flakes
              skeys={{0,1.7,0.35},{1,1.35,0.3}}, squash=-0.12, transp0=0.18,transp1=0.55,
              rate=0.7, speed={0.8,1.6}, life={5,7}, spread=28, rot={-60,60}, rotSpd=26,
              accel=Vec(0,-2.4,0), drag=2.2, glow=0.2, dir=ND.Bottom,
              gust={ax=1.8,az=1.2,wx=0.4,wz=0.31,ph=0.5} },
            { size=Vec(46,4,46), oy=10, tex=TX_SNOWSOFT, color=WHITE,                        -- NEAR bokeh
              skeys={{0,0.48,0.15},{1,0.38,0.10}}, squash=-0.05, transp0=0.7,transp1=0.92,
              rate=12, speed={1.5,3}, life={5,8}, spread=45, rot={-180,180}, rotSpd=10,
              accel=Vec(0,-3.5,0), drag=1.8, glow=0.12, dir=ND.Bottom,
              gust={ax=2.5,az=1.6,wx=0.37,wz=0.29,ph=0} },
            { size=Vec(140,4,140), oy=20, tex=TX_FLAKE, color=Color3.fromRGB(242,248,255),   -- BODY
              skeys={{0,0.5,0.2},{1,0.38,0.14}}, squash=-0.08, transp0=0.08,transp1=0.5,
              rate=190, speed={2,4}, life={7,10}, spread=50, rot={-180,180}, rotSpd=42,
              accel=Vec(0,-5,0), drag=1.7, glow=0.35, dir=ND.Bottom,
              gust={ax=2.2,az=1.4,wx=0.33,wz=0.26,ph=1.9} },
            { size=Vec(220,4,220), oy=32, tex=TX_SNOWSOFT, color=Color3.fromRGB(214,231,255), -- FAR haze
              skeys={{0,0.3,0.08},{1,0.22,0.06}}, transp0=0.5,transp1=0.85,
              rate=265, speed={2,4}, life={8,11}, spread=55, rot={-30,30}, rotSpd=12,
              accel=Vec(0,-5.5,0), drag=1.5, glow=0.3, dir=ND.Bottom,
              gust={ax=1.5,az=1,wx=0.29,wz=0.25,ph=3.8} },
            { size=Vec(100,4,100), oy=12, tex=TX_SPARK, color=WHITE,                         -- GLINTS (subtle)
              skeys={{0,0.4,0.12},{1,0.32,0.1}},
              tkeys={{0,1},{0.2,0.5},{0.45,0.78},{0.62,0.42},{0.85,0.78},{1,1}},             -- soft double-pulse
              rate=10, speed={2,4}, life={5,8}, spread=45, rot={-180,180}, rotSpd=60,
              accel=Vec(0,-4.5,0), drag=1.6, glow=0.7, dir=ND.Bottom,
              gust={ax=2.2,az=1.4,wx=0.35,wz=0.27,ph=1} },
        },
        Mist = {
            -- rolling ground fog. The old single layer peaked at 10% opacity (transp 0.9-0.99) =
            -- effectively invisible; these two layers carry the look on opacity, not count.
            { size=Vec(150,8,150), oy=0, tex=TX_SOFT, color=Color3.fromRGB(210,216,226),
              size0=26,size1=44, transp0=0.68,transp1=0.94,
              rate=16, speed={0.8,2.2}, life={9,13}, spread=20, rot={-5,5}, rotSpd=3,
              accel=Vec(2.2,0.25,1.2), drag=0.8, dir=ND.Top },
            { size=Vec(110,6,110), oy=1, tex=TX_SOFT, color=Color3.fromRGB(228,232,240),    -- near wisps
              size0=12,size1=22, transp0=0.75,transp1=0.95,
              rate=10, speed={1.5,3}, life={6,9}, spread=25, rot={-8,8}, rotSpd=5,
              accel=Vec(3,0.4,1.6), drag=0.8, dir=ND.Top },
        },
        Embers = {
            -- drifting fire motes. NEAR = round soft sprites with LOW emission so the saturated
            -- orange survives a bright sky (full additive glow washed them to white — Studio 07-14);
            -- FAR = a fine additive spark layer for the hot twinkle.
            { size=Vec(90,28,90), oy=2, tex=TX_SOFT, color=ColorSequence.new({
                  ColorSequenceKeypoint.new(0,   Color3.fromRGB(255,170,60)),
                  ColorSequenceKeypoint.new(0.5, Color3.fromRGB(255,95,25)),
                  ColorSequenceKeypoint.new(1,   Color3.fromRGB(165,35,12)) }),
              size0=0.85,size1=0.3, transp0=0.05,transp1=0.85,
              rate=80, speed={3,7}, life={3,5}, spread=40, rot={-60,60}, rotSpd=90,
              accel=Vec(1.5,9,1), drag=1.6, glow=0.35, dir=ND.Top },
            { size=Vec(120,30,120), oy=6, tex=TX_SPARK, color=ColorSequence.new({
                  ColorSequenceKeypoint.new(0, Color3.fromRGB(255,220,140)),
                  ColorSequenceKeypoint.new(1, Color3.fromRGB(255,110,40)) }),
              size0=0.3,size1=0.08, transp0=0.05,transp1=0.9,
              rate=55, speed={2,5}, life={2.5,4}, spread=55, rot={-40,40}, rotSpd=60,
              accel=Vec(1,7,0.8), drag=1.4, glow=0.9, dir=ND.Top },
        },
        Fireflies = {
            -- enchanted night ambience (Studio-redesigned 2026-07-24): SOFT-GLOW motes (radial
            -- disc, not spark) with a SHARP-attack/slow-decay blink — tkeys snap on at 8% life
            -- then fade out long — and a size pulse in sync (skeys peak with the blink).
            -- spread 180 + high drag = drift-stop-drift wander; layers sit BELOW eye level
            -- (ground-hugging). Motes RESIST wind (no gust field); the thin humid-night mist
            -- underlayer drifts with it. Hero fireflies + swarm spectacle = subsystem IIFE below.
            { size=Vec(70,8,70), oy=-2, tex=TX_SNOWSOFT, color=ColorSequence.new({           -- NEAR motes
                  ColorSequenceKeypoint.new(0,   Color3.fromRGB(220,255,130)),
                  ColorSequenceKeypoint.new(0.5, Color3.fromRGB(205,245,100)),
                  ColorSequenceKeypoint.new(1,   Color3.fromRGB(255,200,80)) }),
              skeys={{0,0.1},{0.08,0.62,0.15},{0.45,0.5,0.1},{1,0.08}},
              tkeys={{0,1},{0.08,0.08},{0.4,0.45},{0.75,0.85},{1,1}},
              rate=18, speed={0.5,1.6}, life={2.5,4}, spread=180, rot={0,0}, rotSpd=0,
              accel=Vec(0,0.3,0), drag=2.5, glow=0.95, dir=ND.Top },
            { size=Vec(140,14,140), oy=-1, tex=TX_SNOWSOFT, color=ColorSequence.new({        -- FAR dim motes
                  ColorSequenceKeypoint.new(0, Color3.fromRGB(175,225,95)),
                  ColorSequenceKeypoint.new(1, Color3.fromRGB(200,190,70)) }),
              skeys={{0,0.06},{0.1,0.32,0.08},{0.5,0.26},{1,0.05}},
              tkeys={{0,1},{0.1,0.3},{0.5,0.6},{1,1}},
              rate=18, speed={0.4,1.2}, life={3,5}, spread=180, rot={0,0}, rotSpd=0,
              accel=Vec(0,0.3,0), drag=2.5, glow=0.95, dir=ND.Top },
            { size=Vec(120,4,120), oy=-4, tex=TX_PUFF, color=Color3.fromRGB(148,158,178),    -- ground mist
              size0=13,size1=22, tkeys={{0,1},{0.2,0.8},{0.7,0.89},{1,1}},
              rate=8, speed={0.5,1.5}, life={8,12}, spread=14, rot={-180,180}, rotSpd=4,
              accel=Vec(1.2,0.12,0.7), drag=0.7, glow=0, dir=ND.Top,
              gust={ax=1.2,az=0.8,wx=0.23,wz=0.19,ph=2.2} },
        },
        Petals = {
            -- SAKURA FALL, 6-layer hanami (Studio-verified 2026-07-25). Real petal texture, and
            -- the motion carries it: every layer's Squash alternates edge-on ↔ flat over its life
            -- (flutterSeq) while RotSpeed spans ±full so petals tumble at wildly different rates —
            -- that pair IS the flutter. Base accel is PURE fall; ALL lateral drift comes from the
            -- shared Wind via each layer's gust field, so a gust event sweeps the whole field
            -- sideways as ONE visible wave (SakuraGale makes that a set piece). Depth ladder:
            -- defocus bokeh at the lens → two big-petal hero layers, one blush and one apricot-cream
            -- (two tints read far richer than one) → the readable body flurry → a fine far dust
            -- → a sparse carpet skittering on the ground. COLOUR: pale blush, NOT candy pink —
            -- the texture is already pink, so a saturated tint double-dips and reads as plastic
            -- confetti; LightEmission stays ≤0.2 for the same reason (the bokeh discs blew out to
            -- white hot-spots against the sky at 0.22).
            -- LENS bokeh. At world depth (zoff=0) these discs painted saturated patches ON walls;
            -- the four fixes are alpha floored at 0.78 (stacking reads as haze, never a hot-spot),
            -- a small desaturated disc, a blush tint matched to the other layers (a more saturated
            -- pink here is what made the artifact read magenta), and zoff.
            -- ZOFFSET IS DEPTH-TOWARD-CAMERA, and this host is centred ON the camera: measured in
            -- Studio, zoff >= 6 pushes every disc past the near plane and the layer renders NOTHING
            -- (invisible against open sky, not just occluded). zoff=3 is the working value — it
            -- composites the discs 3 studs in front of world geometry AND culls exactly the
            -- sub-3-stud ones that would otherwise blow up frame-filling. No squash: a defocus
            -- bokeh is a circle, and the -0.25 ellipse is what read as a vertical plume.
            { size=Vec(26,6,26), oy=3, tex=TX_SNOWSOFT, zoff=3, color=ColorSequence.new({
                  ColorSequenceKeypoint.new(0, Color3.fromRGB(255,222,232)),
                  ColorSequenceKeypoint.new(1, Color3.fromRGB(250,205,220)) }),
              skeys={{0,1.75,0.5},{1,1.4,0.3}}, transp0=0.78,transp1=0.92,
              rate=7, speed={0.4,1.1}, life={5,7}, spread=30, rot={-180,180}, rotSpd=22,
              accel=Vec(0,-3,0), drag=1.7, glow=0.04, dir=ND.Bottom,
              gust={ax=2.2,az=1.5,wx=0.43,wz=0.33,ph=1.4} },
            { size=Vec(64,12,64), oy=5, tex=TX_PETAL, color=ColorSequence.new({              -- HERO blush
                  ColorSequenceKeypoint.new(0,    Color3.fromRGB(255,222,234)),
                  ColorSequenceKeypoint.new(0.55, Color3.fromRGB(248,178,206)),
                  ColorSequenceKeypoint.new(1,    Color3.fromRGB(226,132,176)) }),
              skeys={{0,2.05,0.5},{1,1.7,0.35}}, qseq=flutterSeq(-0.88, 0.08, 14, 0.12, 0),
              transp0=0.1,transp1=0.38,
              rate=13, speed={0.6,1.4}, life={6,8}, spread=35, rot={-180,180}, rotSpd=95,
              accel=Vec(0,-3.5,0), drag=1.6, glow=0.08, dir=ND.Bottom,
              gust={ax=2.6,az=1.8,wx=0.4,wz=0.31,ph=0.5} },
            -- HERO apricot-cream. The second hero tint has to carry real chroma: at (255,246,248)
            -- it was functionally white, so the two hero layers read as "pink and white" and the
            -- pale petals vanished into the pale ground. Ivory alone still washed to white against
            -- a bright sky — apricot is the tint that separates from the blush AT A GLANCE. Low
            -- glow keeps it shaded; emissive apricot just blows back out to white under the key.
            { size=Vec(76,12,76), oy=7, tex=TX_PETAL, color=ColorSequence.new({
                  ColorSequenceKeypoint.new(0,    Color3.fromRGB(250,224,200)),
                  ColorSequenceKeypoint.new(0.55, Color3.fromRGB(244,203,178)),
                  ColorSequenceKeypoint.new(1,    Color3.fromRGB(230,176,150)) }),
              skeys={{0,1.55,0.4},{1,1.28,0.28}}, qseq=flutterSeq(-0.8, 0.15, 12, 0.15, 1),
              transp0=0.14,transp1=0.44,
              rate=12, speed={0.8,1.8}, life={6,8.5}, spread=42, rot={-180,180}, rotSpd=140,
              accel=Vec(0,-3.2,0), drag=1.55, glow=0.05, dir=ND.Bottom,
              gust={ax=3,az=2,wx=0.37,wz=0.29,ph=2.1} },
            { size=Vec(130,6,130), oy=15, tex=TX_PETAL, color=ColorSequence.new({            -- BODY flurry
                  ColorSequenceKeypoint.new(0, Color3.fromRGB(255,222,234)),
                  ColorSequenceKeypoint.new(1, Color3.fromRGB(246,178,204)) }),
              skeys={{0,0.85,0.26},{1,0.68,0.18}}, qseq=flutterSeq(-0.7, 0.1, 8, 0.2, 0),
              transp0=0.06,transp1=0.44,
              rate=120, speed={1.4,3}, life={7,9}, spread=48, rot={-180,180}, rotSpd=170,
              accel=Vec(0,-5.2,0), drag=1.5, glow=0.1, dir=ND.Bottom,
              gust={ax=3.2,az=2.2,wx=0.33,wz=0.26,ph=1.9} },
            { size=Vec(240,6,240), oy=26, tex=TX_PETAL, color=ColorSequence.new({            -- FAR dust
                  ColorSequenceKeypoint.new(0, Color3.fromRGB(250,214,228)),
                  ColorSequenceKeypoint.new(1, Color3.fromRGB(238,190,210)) }),
              skeys={{0,0.3,0.09},{1,0.24,0.06}}, squash=-0.5, transp0=0.42,transp1=0.8,
              rate=170, speed={1.5,3.2}, life={8,11}, spread=55, rot={-180,180}, rotSpd=90,
              accel=Vec(0,-6,0), drag=1.35, glow=0.08, dir=ND.Bottom,
              gust={ax=2.4,az=1.6,wx=0.29,wz=0.25,ph=3.8} },
            -- GROUND carpet: oy puts the thin slab a hair above the feet (camera sits ~4.5
            -- studs over the floor), so settled petals skitter ON the ground instead of
            -- half-clipping through it. tag="settle" keeps it out of the SakuraGale Rate swell.
            { size=Vec(56,1.2,56), oy=-3.7, tex=TX_PETAL, tag="settle",
              color=ColorSequence.new({
                  ColorSequenceKeypoint.new(0, Color3.fromRGB(255,214,228)),
                  ColorSequenceKeypoint.new(1, Color3.fromRGB(240,178,204)) }),
              skeys={{0,1.1,0.3},{1,0.95,0.2}}, qseq=flutterSeq(-0.92, -0.2, 10, 0.1, 0),
              transp0=0.05,transp1=0.42,
              rate=34, speed={0.05,0.5}, life={14,20}, spread=90, rot={-180,180}, rotSpd=25,
              accel=Vec(0,-0.15,0), drag=2.6, glow=0.08, dir=ND.Bottom,
              gust={ax=3.6,az=3,wx=0.5,wz=0.44,ph=0.9} },
        },
        Autumn = {
            -- LEAF FALL, 16-layer (Studio-verified 2026-07-25, golden-hour stage). SIX species on
            -- separate emitters — gold 25%, russet 22%, straw 18%, wine 13%, brown 12%, olive 10%
            -- by particle count. Autumn reads from HUE SPREAD: a value ramp on one orange hue is
            -- still one colour at gameplay distance, which is what made the 4-species pass read as
            -- confetti. ParticleEmitter.Color MULTIPLIES the sprite, so olive and wine are only
            -- reachable because the maple textures are not fully saturated (olive = a green-biased
            -- tint on the orange leaf; wine = a blue-lifted tint on the red one) — do not "fix"
            -- those tints toward the colour they look like in the table.
            -- LIGHT IS THE EFFECT: gold and straw carry LightEmission ~0.6 AND a tint that
            -- BRIGHTENS toward mid-life, so they read as translucent leaves lit through from
            -- behind; russet/wine/brown hold 0.07-0.16 and stay silhouettes. That glow-vs-
            -- silhouette contrast is the whole golden-hour look — raising LightEmission globally
            -- destroys it (every species clips to the same flame vermilion). Nothing goes near
            -- black either: the old near-black brown read as dirt specks against sky, so brown's
            -- tint floor is lifted and brown is kept out of the FAR layers entirely.
            -- MOTION: Squash flips edge-on <-> flat (flutterSeq) while RotSpeed spans +-full, so
            -- leaves tumble and glide at wildly different rates instead of floating like dots.
            -- Squash past ~-0.7 renders as a vertical smear, not a leaf — capped at -0.66.
            -- The gust field's per-layer phase breath uses wx ~= wz, which rotates each layer's
            -- accel vector elliptically — that IS the spiral tendency in the descent; do NOT add
            -- private sine wind on top. Autumn couples to the shared Wind HARDER than any other
            -- ambient type (ax up to 9 on the carpet).
            -- WIND AS AN EVENT: body rate sits at ~55% of the old always-on density so the lull
            -- between gusts is real negative space, `uw` biases each volume UPWIND of the camera
            -- (windward/leeward density gradient), and the two rate-0 STREAK layers are Emit()'d
            -- by the gust hook — host aimed downwind, 5-8x body speed, high Drag so they
            -- decelerate back into the ambient field. Leaf streams cross frame, then quiet.
            -- HERO SIZE IS CLAMPED to ~0.62 and the host sits 13-15 studs ABOVE the camera in a
            -- shallow 6-stud slab: the host box follows the camera, so a bigger sprite passing
            -- within a stud subtends a third of the frame and reads as a decal stuck on the lens.
            { size=Vec(20,6,20), oy=13, uw=8, tex=TX_LEAFA, color=ColorSequence.new({   -- HERO gold
                  ColorSequenceKeypoint.new(0,    Color3.fromRGB(226,190,128)),
                  ColorSequenceKeypoint.new(0.45, Color3.fromRGB(255,246,200)),
                  ColorSequenceKeypoint.new(1,    Color3.fromRGB(190,132,62)) }),
              skeys={{0,0.62,0.14},{1,0.54,0.10}}, qseq=flutterSeq(-0.60, 0.14, 13, 0.10, 0),
              transp0=0.08,transp1=0.42,
              rate=14, speed={0.6,1.6}, life={7,9}, spread=34, rot={-180,180}, rotSpd=130,
              accel=Vec(0,-5,0), drag=1.5, glow=0.62, dir=ND.Bottom,
              gust={ax=3.4,az=2.2,wx=0.41,wz=0.3,ph=0.5} },
            { size=Vec(22,6,22), oy=15, uw=8, tex=TX_LEAFB, color=ColorSequence.new({   -- HERO wine
                  ColorSequenceKeypoint.new(0, Color3.fromRGB(156,74,255)),
                  ColorSequenceKeypoint.new(1, Color3.fromRGB(106,46,200)) }),
              skeys={{0,0.60,0.14},{1,0.52,0.10}}, qseq=flutterSeq(-0.56, 0.20, 11, 0.14, 1),
              transp0=0.12,transp1=0.46,
              rate=9, speed={0.8,1.9}, life={7,9}, spread=40, rot={-180,180}, rotSpd=185,
              accel=Vec(0,-5.6,0), drag=1.5, glow=0.07, dir=ND.Bottom,
              gust={ax=3.9,az=2.6,wx=0.36,wz=0.27,ph=2.1} },
            { size=Vec(150,8,150), oy=16, uw=26, tex=TX_LEAFA, color=ColorSequence.new({ -- BODY gold
                  ColorSequenceKeypoint.new(0,    Color3.fromRGB(226,190,128)),
                  ColorSequenceKeypoint.new(0.45, Color3.fromRGB(255,246,200)),
                  ColorSequenceKeypoint.new(1,    Color3.fromRGB(190,132,62)) }),
              skeys={{0,0.56,0.16},{1,0.44,0.11}}, qseq=flutterSeq(-0.50, 0.10, 8, 0.16, 0),
              transp0=0.08,transp1=0.50,
              rate=24, speed={1.4,3}, life={7,9}, spread=54, rot={-180,180}, rotSpd=210,
              accel=Vec(0,-7,0), drag=1.45, glow=0.62, dir=ND.Bottom,
              gust={ax=4.2,az=2.8,wx=0.33,wz=0.24,ph=1.9} },
            { size=Vec(150,8,150), oy=16, uw=26, tex=TX_LEAFB, color=ColorSequence.new({ -- BODY russet
                  ColorSequenceKeypoint.new(0, Color3.fromRGB(160,80,52)),
                  ColorSequenceKeypoint.new(1, Color3.fromRGB(104,44,30)) }),
              skeys={{0,0.54,0.16},{1,0.42,0.11}}, qseq=flutterSeq(-0.54, 0.14, 9, 0.16, 1),
              transp0=0.10,transp1=0.50,
              rate=21, speed={1.4,3.1}, life={7,9}, spread=54, rot={-180,180}, rotSpd=200,
              accel=Vec(0,-7.2,0), drag=1.45, glow=0.07, dir=ND.Bottom,
              gust={ax=4,az=2.7,wx=0.31,wz=0.26,ph=3.3} },
            { size=Vec(150,8,150), oy=16, uw=26, tex=TX_LEAFA, color=ColorSequence.new({ -- BODY straw
                  ColorSequenceKeypoint.new(0,   Color3.fromRGB(230,222,176)),
                  ColorSequenceKeypoint.new(0.5, Color3.fromRGB(255,254,228)),
                  ColorSequenceKeypoint.new(1,   Color3.fromRGB(200,180,122)) }),
              skeys={{0,0.52,0.15},{1,0.41,0.10}}, qseq=flutterSeq(-0.52, 0.12, 10, 0.16, 1),
              transp0=0.10,transp1=0.52,
              rate=17, speed={1.3,2.9}, life={7,9}, spread=52, rot={-180,180}, rotSpd=190,
              accel=Vec(0,-6.8,0), drag=1.45, glow=0.60, dir=ND.Bottom,
              gust={ax=4.1,az=2.7,wx=0.35,wz=0.28,ph=2.7} },
            { size=Vec(150,8,150), oy=16, uw=26, tex=TX_LEAFB, color=ColorSequence.new({ -- BODY wine
                  ColorSequenceKeypoint.new(0, Color3.fromRGB(156,74,255)),
                  ColorSequenceKeypoint.new(1, Color3.fromRGB(106,46,200)) }),
              skeys={{0,0.53,0.16},{1,0.42,0.11}}, qseq=flutterSeq(-0.52, 0.16, 9, 0.16, 0),
              transp0=0.10,transp1=0.50,
              rate=12, speed={1.4,3.1}, life={7,9}, spread=54, rot={-180,180}, rotSpd=220,
              accel=Vec(0,-7.1,0), drag=1.45, glow=0.07, dir=ND.Bottom,
              gust={ax=4,az=2.7,wx=0.3,wz=0.25,ph=5.1} },
            -- brown keeps a NEARER volume (uw 20 / body box only): its silhouette is legible
            -- inside ~25 studs and reads as dirt beyond that, so it never enters the far field.
            { size=Vec(150,8,150), oy=16, uw=20, tex=TX_LEAFC, color=ColorSequence.new({ -- BODY brown
                  ColorSequenceKeypoint.new(0, Color3.fromRGB(196,156,112)),
                  ColorSequenceKeypoint.new(1, Color3.fromRGB(146,110,74)) }),
              skeys={{0,0.50,0.15},{1,0.40,0.10}}, qseq=flutterSeq(-0.52, 0.10, 7, 0.16, 0),
              transp0=0.14,transp1=0.55,
              rate=11, speed={1.5,3.2}, life={7,9}, spread=56, rot={-180,180}, rotSpd=240,
              accel=Vec(0,-7.6,0), drag=1.45, glow=0.08, dir=ND.Bottom,
              gust={ax=3.8,az=2.6,wx=0.29,wz=0.23,ph=0.9} },
            { size=Vec(150,8,150), oy=16, uw=26, tex=TX_LEAFA, color=ColorSequence.new({ -- BODY olive (still turning)
                  ColorSequenceKeypoint.new(0, Color3.fromRGB(152,255,255)),
                  ColorSequenceKeypoint.new(1, Color3.fromRGB(118,214,240)) }),
              skeys={{0,0.52,0.15},{1,0.41,0.10}}, qseq=flutterSeq(-0.50, 0.14, 8, 0.16, 1),
              transp0=0.12,transp1=0.52,
              rate=10, speed={1.4,3}, life={7,9}, spread=54, rot={-180,180}, rotSpd=195,
              accel=Vec(0,-7,0), drag=1.45, glow=0.16, dir=ND.Bottom,
              gust={ax=4.1,az=2.7,wx=0.34,wz=0.27,ph=4.2} },
            -- FAR: aerial perspective, not a confetti belt — desaturated by alpha, and a TALL host
            -- (30) so it never stacks into a bright stripe at the horizon.
            { size=Vec(300,30,300), oy=30, uw=60, tex=TX_LEAFA, color=ColorSequence.new({
                  ColorSequenceKeypoint.new(0,    Color3.fromRGB(226,190,128)),
                  ColorSequenceKeypoint.new(0.45, Color3.fromRGB(255,246,200)),
                  ColorSequenceKeypoint.new(1,    Color3.fromRGB(190,132,62)) }),
              skeys={{0,0.24,0.06},{1,0.18,0.04}}, squash=-0.35, transp0=0.68,transp1=0.92,
              rate=30, speed={1.5,3.2}, life={8,11}, spread=55, rot={-180,180}, rotSpd=110,
              accel=Vec(0,-8,0), drag=1.35, glow=0.55, dir=ND.Bottom,
              gust={ax=3.1,az=2.1,wx=0.27,wz=0.24,ph=1.2} },
            { size=Vec(300,30,300), oy=30, uw=60, tex=TX_LEAFB, color=ColorSequence.new({
                  ColorSequenceKeypoint.new(0, Color3.fromRGB(160,80,52)),
                  ColorSequenceKeypoint.new(1, Color3.fromRGB(104,44,30)) }),
              skeys={{0,0.23,0.06},{1,0.17,0.04}}, squash=-0.35, transp0=0.70,transp1=0.93,
              rate=24, speed={1.5,3.2}, life={8,11}, spread=55, rot={-180,180}, rotSpd=120,
              accel=Vec(0,-8,0), drag=1.35, glow=0.07, dir=ND.Bottom,
              gust={ax=3,az=2,wx=0.29,wz=0.22,ph=3.8} },
            -- GUST STREAKS: idle at Rate 0, Emit()'d in a 1s stagger by the gust hook. Host is
            -- AIMED downwind (aim=true) and emits from its Front face, so Speed is genuinely
            -- along the wind vector; VelocityParallel + Squash -0.55 stretches the sprite along
            -- travel so it still reads as a streak in a frozen frame; Drag 2.6 bleeds the speed
            -- off inside a second and the leaves rejoin the ambient fall.
            { size=Vec(120,16,120), oy=6, tex=TX_LEAFA, tag="streak", aim=true,
              orient=Enum.ParticleOrientation.VelocityParallel,
              color=ColorSequence.new({
                  ColorSequenceKeypoint.new(0,    Color3.fromRGB(226,190,128)),
                  ColorSequenceKeypoint.new(0.45, Color3.fromRGB(255,246,200)),
                  ColorSequenceKeypoint.new(1,    Color3.fromRGB(190,132,62)) }),
              skeys={{0,0.66,0.16},{1,0.46,0.10}}, squash=-0.55,
              tkeys={{0,1},{0.08,0.06},{0.72,0.34},{1,1}},
              rate=0, speed={16,26}, life={1,1.9}, spread=13, rot={-180,180}, rotSpd=300,
              accel=Vec(0,-3.4,0), drag=2.6, glow=0.62, dir=ND.Front,
              gust={ax=5,az=4,wx=0.4,wz=0.35,ph=0} },
            { size=Vec(120,16,120), oy=5, tex=TX_LEAFB, tag="streak", aim=true,
              orient=Enum.ParticleOrientation.VelocityParallel,
              color=ColorSequence.new({
                  ColorSequenceKeypoint.new(0, Color3.fromRGB(160,80,52)),
                  ColorSequenceKeypoint.new(1, Color3.fromRGB(104,44,30)) }),
              skeys={{0,0.62,0.16},{1,0.44,0.10}}, squash=-0.55,
              tkeys={{0,1},{0.08,0.08},{0.72,0.38},{1,1}},
              rate=0, speed={15,24}, life={1,1.8}, spread=14, rot={-180,180}, rotSpd=280,
              accel=Vec(0,-3.6,0), drag=2.6, glow=0.07, dir=ND.Front,
              gust={ax=5,az=4,wx=0.37,wz=0.33,ph=1.7} },
            -- GROUND skitter: near-zero fall, long life, hardest wind coupling of the set (ax 9) —
            -- high Drag makes each gust a scuttle that then stalls, instead of a constant hover.
            -- tag="settle" keeps the carpet out of the gust Rate surge and the devil suppression.
            -- The carpet slabs are THIN and sit ENTIRELY above the floor (camera rides ~4.6 studs
            -- up): a slab straddling the floor plane spawns half its leaves under it, where they
            -- are depth-culled — that read as a bare floor.
            { size=Vec(96,0.9,96), oy=-3.9, tex=TX_LEAFA, tag="settle",
              color=ColorSequence.new({
                  ColorSequenceKeypoint.new(0,   Color3.fromRGB(200,150,86)),
                  ColorSequenceKeypoint.new(0.5, Color3.fromRGB(168,110,58)),
                  ColorSequenceKeypoint.new(1,   Color3.fromRGB(130,80,44)) }),
              skeys={{0,0.50,0.14},{1,0.44,0.10}}, qseq=flutterSeq(-0.62, -0.05, 10, 0.1, 0),
              transp0=0.06,transp1=0.46,
              rate=34, speed={0.8,3.2}, life={10,16}, spread=90, rot={-180,180}, rotSpd=90,
              accel=Vec(0,-0.15,0), drag=3.2, glow=0.32, dir=ND.Bottom,
              gust={ax=9,az=8,wx=0.5,wz=0.44,ph=0.9} },
            -- GROUND tumblers: the few that go end-over-end along the floor (full-flip Squash +
            -- 340 deg/s spin + a short arc) instead of sliding flat.
            { size=Vec(70,0.8,70), oy=-3.85, tex=TX_LEAFB, tag="settle",
              color=ColorSequence.new({
                  ColorSequenceKeypoint.new(0, Color3.fromRGB(160,80,52)),
                  ColorSequenceKeypoint.new(1, Color3.fromRGB(104,44,30)) }),
              skeys={{0,0.54,0.12},{1,0.46,0.08}}, qseq=flutterSeq(-0.66, 0.26, 16, 0.06, 0),
              transp0=0.05,transp1=0.44,
              rate=14, speed={0.6,2.2}, life={3,5}, spread=70, rot={-180,180}, rotSpd=340,
              accel=Vec(0,-1.2,0), drag=1.9, glow=0.10, dir=ND.Top,
              gust={ax=8,az=7,wx=0.62,wz=0.55,ph=2.4} },
            -- LEAF LITTER (the base the skitter skitters across): dense, 20-28s life, effectively
            -- static. VelocityPerpendicular lays each sprite FLAT on the floor — as camera-facing
            -- cards these read as leaves standing on edge and the ground stayed bare-grey no
            -- matter how high the rate went. A tiled litter Decal was tried and rejected: any tile
            -- pitch shows an obvious repeating grid.
            -- ARTIFACT (fixed, do not undo): a VelocityPerpendicular sprite whose velocity drags
            -- to ~0 — or gets swung horizontal by a wind term — has an undefined/edge-on plane and
            -- renders as a hard stretched white polyline. This layer therefore emits DOWNWARD and
            -- carries NO gust field, so its velocity is a small, permanently-downward terminal
            -- (~0.03 st/s) and the plane always resolves horizontal. Wind-driven ground motion is
            -- the skitter/tumbler layers' job.
            { size=Vec(74,0.6,74), oy=-3.95, tex=TX_LEAFC, tag="settle",
              orient=Enum.ParticleOrientation.VelocityPerpendicular,
              color=ColorSequence.new({
                  ColorSequenceKeypoint.new(0, Color3.fromRGB(190,146,96)),
                  ColorSequenceKeypoint.new(1, Color3.fromRGB(150,110,72)) }),
              skeys={{0,0.76,0.16},{1,0.72,0.12}}, squash=0, transp0=0.04,transp1=0.30,
              rate=42, speed={0.04,0.1}, life={20,28}, spread=90, rot={-180,180}, rotSpd=3,
              accel=Vec(0,-0.12,0), drag=4, glow=0.22, dir=ND.Bottom },
            -- GUST DUST: idle at Rate 0 and Emit()'d by the gust hook — a low tan puff skimming
            -- the floor so a gust is felt on the ground, not only in the air. Small + wide spread:
            -- at Size 2.6 this rendered as horizontal grey smears against the wall bases.
            { size=Vec(80,0.8,80), oy=-4, tex=TX_DUST, tag="dust",
              color=ColorSequence.new({
                  ColorSequenceKeypoint.new(0, Color3.fromRGB(206,180,140)),
                  ColorSequenceKeypoint.new(1, Color3.fromRGB(150,124,92)) }),
              skeys={{0,1.1,0.35},{1,3.4,0.5}}, tkeys={{0,1},{0.18,0.84},{0.7,0.93},{1,1}},
              rate=0, speed={2,6}, life={1.1,2}, spread=85, rot={-180,180}, rotSpd=22,
              accel=Vec(0,0.6,0), drag=2.4, glow=0.06, dir=ND.Top,
              gust={ax=8,az=7,wx=0.4,wz=0.35,ph=0} },
        },
        Ash = {
            -- slow drifting DARK grey flakes (darkened from near-white so they contrast a bright
            -- sky — Studio 07-14) + the occasional live spark riding the same air
            { size=Vec(110,4,110), oy=16, tex=TX_SOFT, color=Color3.fromRGB(96,90,86),
              size0=0.4,size1=0.3, transp0=0.05,transp1=0.5,
              rate=140, speed={2,4}, life={6,9}, spread=45, rot={-60,60}, rotSpd=45,
              accel=Vec(-3.5,-5,2), drag=1.6, glow=0, dir=ND.Bottom },
            { size=Vec(170,4,170), oy=26, tex=TX_SPARK, color=Color3.fromRGB(255,140,70),
              size0=0.14,size1=0.05, transp0=0.15,transp1=0.9,
              rate=26, speed={1.5,3.5}, life={4,7}, spread=60, rot={-40,40}, rotSpd=70,
              accel=Vec(-2.5,-2,1.5), drag=1.5, glow=0.9, dir=ND.Bottom },
        },
        -- Sandstorm reuses the Mist emitter machinery: warm tan dust driven hard
        -- laterally (a wall of blowing sand), not a settling fall.
        Sandstorm = {
            { size=Vec(160,20,160), oy=6, tex=TX_SOFT, color=Color3.fromRGB(194,168,120),   -- #C2A878
              size0=24,size1=38, transp0=0.58,transp1=0.93,
              rate=24, speed={8,15}, life={5,8}, spread=28, rot={-8,8}, rotSpd=6,
              accel=Vec(20,0.5,7), drag=0.6, dir=ND.Right },
        },
    }
    -- BloodMoon is a SPECIAL build (not a particle preset) — see buildBloodMoon.
    Weather.TypeOrder = { "Rain", "Snow", "Mist", "Embers", "Fireflies", "Petals", "Autumn",
                          "Ash", "Sandstorm", "BloodMoon" }

    local MOOD = {
        Rain      = { B=-0.03, C=0.08,  S=-0.12, tint=Color3.fromRGB(205,220,255) },
        Snow      = { B= 0.03, C=0.05,  S=-0.08, tint=Color3.fromRGB(226,240,255) },
        Mist      = { B=-0.01, C=-0.05, S=-0.20, tint=Color3.fromRGB(220,224,230) },
        Embers    = { B= 0.02, C=0.08,  S= 0.10, tint=Color3.fromRGB(255,238,220) },
        Fireflies = { B=-0.04, C=0.06,  S= 0.02, tint=Color3.fromRGB(238,228,205) },   -- warm dusk
        Petals    = { B= 0.00, C=0.09,  S= 0.05, tint=Color3.fromRGB(255,234,234) },   -- warm blush hanami
        Autumn    = { B= 0.005, C=0.08, S= 0.04, tint=Color3.fromRGB(253,244,232) },   -- near-neutral: a strong warm tint here salmon-shifts the ground under a cool sky
        Ash       = { B=-0.04, C=0.06,  S=-0.25, tint=Color3.fromRGB(225,220,215) },
        Sandstorm = { B=-0.02, C=0.10,  S= 0.05, tint=Color3.fromRGB(224,196,150) },   -- warm dust haze
        BloodMoon = { B=-0.05, C=0.12,  S=-0.20, tint=Color3.fromRGB(255,180,180) },   -- red pall
    }
    local SOUND = { Rain="rain", Snow="wind", Mist="wind", Embers="fire",
                    Fireflies="night", Petals="birds", Autumn="wind", Ash="wind",
                    Sandstorm="wind", BloodMoon="night" }

    local function ambientId(mood)
        local map = cfg("WeatherSoundIds", nil)
        if type(map) == "table" and map[mood] and map[mood] ~= "" then return map[mood] end
        return nil
    end

    -- ── active state ──
    local _partLayers = {}
    local _ambient  = nil
    local _moodCC   = nil
    local _moodTween = nil
    local FADE_TI = TweenInfo.new(2, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut)   -- layer Rate crossfade
    local MOOD_TI = TweenInfo.new(3, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut)   -- mood CC lerp
    local _followConn = nil
    local _lightFolder = nil
    local _rays = nil   -- SunRaysEffect; declared here so setIntensity can retune it live
    -- special-type + scheduler state
    local _special = { moon = nil, shell = nil, sky = nil, cc = nil, atmo = nil, halo = nil, kind = nil }
    local _specialConn = nil
    local _clockConn = nil

    local function clearPartLayers()
        for _, L in _partLayers do
            if L.rateTween then L.rateTween:Cancel() end
            if L.host then pcall(function() L.host:Destroy() end) end
        end
        table.clear(_partLayers)
    end

    -- ── crossfade plumbing ──────────────────────────────────────────────────
    -- PARTICLE-layer type switches fade instead of hard-cutting: outgoing layers
    -- ramp Rate→0 over FADE_TI, then die after their longest Lifetime has drained
    -- (still camera-followed while fading — no stranded volumes); incoming layers
    -- ramp 0→target (mkPartLayer rampIn). At most ONE outgoing batch lives at a
    -- time: a switch mid-fade fast-forwards the pending batch, and killFade bumps
    -- _fadeToken so the stale task.delay no-ops (no orphan hosts, no stacked
    -- delays). Rain (beam pool) + BloodMoon keep their instant swap — out of scope.
    local _fadeLayers = {}   -- { host, oy } draining out
    local _fadeToken = 0

    local function killFade()
        _fadeToken = _fadeToken + 1
        for _, F in _fadeLayers do
            local host = F.host
            pcall(function() host:Destroy() end)
        end
        table.clear(_fadeLayers)
    end
    local function beginLayerFade()
        killFade()
        local maxLife = 0
        for _, L in _partLayers do
            if L.rateTween then L.rateTween:Cancel() end
            if L.emitter then
                TweenService:Create(L.emitter, FADE_TI, { Rate = 0 }):Play()
            end
            if L.life2 > maxLife then maxLife = L.life2 end
            _fadeLayers[#_fadeLayers + 1] = { host = L.host, oy = L.oy }
        end
        table.clear(_partLayers)
        local tok = _fadeToken
        task.delay(FADE_TI.Time + maxLife, function()
            if tok == _fadeToken then killFade() end
        end)
    end

    -- ── spectacle-event scheduler (mechanism only; events register later) ──
    -- registerSpectacle(name, weight, fn): weighted pick, ~one event / 2-4 min
    -- while weather is on, fn run in pcall from the follow loop. Empty registry
    -- = pure no-op. Registration is load-time + permanent (no unregister).
    local _spectacles, _spectWeight, _spectNextT = {}, 0, 0
    local function registerSpectacle(name, weight, fn)
        _spectacles[#_spectacles + 1] = { name = name, weight = weight, fn = fn }
        _spectWeight = _spectWeight + weight
    end
    -- shared wind-swell envelope (SnowBlizzard / SakuraGale spectacles): while tick() is inside
    -- [_blizzT0, _blizzT0+_blizzDur] the follow loop multiplies every gust layer's wind
    -- term by one shared sine swell — a coherent surge through all layers. Self-expires.
    local _blizzT0, _blizzDur = 0, 1

    -- This layer's emitter Rate at intensity I, before any effect multiplier.
    local function layerRate(L, I)
        if L.spec.tag == "settle" then
            return L.spec.rate * iRate(I, SET_LO, SET_HI)
        end
        return L.spec.rate * iRate(I, L.ir.lo, L.ir.hi)
    end
    -- LIVE intensity re-apply for one layer, everything except Rate. Acceleration, Speed,
    -- Size and Transparency are all writable on a RUNNING ParticleEmitter, so the slider
    -- never rebuilds a layer — which is what makes "more than rate" affordable at all.
    -- `settle` layers keep their fall and drive untouched: a carpet doesn't fall harder,
    -- and the litter layer's VelocityPerpendicular plane depends on its tiny fixed
    -- downward terminal (see the preset comment) — scaling it is not worth the artifact.
    local function tuneLayer(L, I)
        local spec, R, e = L.spec, L.ir, L.emitter
        local am = 1
        if spec.tag ~= "settle" then
            am = 1 - R.acc + R.acc * I
        end
        L.accel = spec.accel * am
        if not spec.gust then
            e.Acceleration = L.accel   -- gust layers get theirs written by the follow loop
        end
        e.Speed = NumberRange.new(spec.speed[1] * am, spec.speed[2] * am)
        if spec.gust then
            local gm = 1 - R.gst + R.gst * I
            L.gax, L.gaz = spec.gust.ax * gm, spec.gust.az * gm
        end
        local sm = 1 - R.sz + R.sz * I
        if spec.skeys then
            local ks = spec.skeys
            local kps = table.create(#ks)
            for i = 1, #ks do
                local k = ks[i]
                kps[i] = NumberSequenceKeypoint.new(k[1], k[2] * sm, (k[3] or 0) * sm)
            end
            e.Size = NumberSequence.new(kps)
        else
            e.Size = NumberSequence.new(spec.size0 * sm, spec.size1 * sm)
        end
        -- tkeys layers encode blink/pulse CHOREOGRAPHY, not density — leave them alone.
        -- Opacity scales, so the same clamp self-limits: an already-opaque layer has
        -- nothing to give and barely moves, while a faint veil layer moves a lot.
        if not spec.tkeys then
            local om = 1 - R.alp + R.alp * I
            e.Transparency = NumberSequence.new({
                NumberSequenceKeypoint.new(0,    1),
                NumberSequenceKeypoint.new(0.12, math.clamp(1 - (1 - spec.transp0) * om, 0, 1)),
                NumberSequenceKeypoint.new(0.8,  math.clamp(1 - (1 - spec.transp1) * om, 0, 1)),
                NumberSequenceKeypoint.new(1,    1),
            })
        end
    end

    local function mkPartLayer(spec, ir, rampIn)
        local host = Instance.new("Part")
        host.Name = "_wxHost"; host.Anchored = true; host.CanCollide = false; host.CanQuery = false
        host.CanTouch = false; host.CastShadow = false; host.Transparency = 1; host.Massless = true
        host.Size = spec.size; host:SetAttribute("WX_Custom", true)
        host.Parent = getFolder()
        local e = Instance.new("ParticleEmitter")
        e:SetAttribute("WX_Custom", true)
        e.Texture = spec.tex
        local colorSeq = spec.color
        if typeof(colorSeq) ~= "ColorSequence" then
            colorSeq = ColorSequence.new(colorSeq)
        end
        e.Color = colorSeq
        local glow = spec.glow or 0
        e.LightEmission  = glow
        e.LightInfluence = 1 - math.min(glow, 1)   -- partial glow keeps daylight shading
        e.Lifetime = NumberRange.new(spec.life[1], spec.life[2])
        e.SpreadAngle = Vector2.new(spec.spread, spec.spread)
        local rot0, rot1 = 0, 360
        if spec.rot then
            rot0, rot1 = spec.rot[1], spec.rot[2]
        end
        e.Rotation = NumberRange.new(rot0, rot1)
        e.RotSpeed = NumberRange.new(-(spec.rotSpd or 45), spec.rotSpd or 45)
        -- qseq = a prebuilt Squash sequence (flutterSeq); plain `squash` is the constant case.
        -- 0 = undistorted (engine default; the old `or 1` stretched every sprite).
        if spec.qseq then
            e.Squash = spec.qseq
        else
            e.Squash = NumberSequence.new(spec.squash or 0)
        end
        -- Size / Transparency / Acceleration / Speed are intensity-driven: tuneLayer below
        -- owns them. Only the choreographed tkeys sequence is static.
        if spec.tkeys then
            local kps = table.create(#spec.tkeys)
            for i = 1, #spec.tkeys do kps[i] = NumberSequenceKeypoint.new(spec.tkeys[i][1], spec.tkeys[i][2]) end
            e.Transparency = NumberSequence.new(kps)
        end
        e.Drag = spec.drag or 0
        -- zoff pulls a layer forward in depth: a defocus/lens layer composites in FRONT of the
        -- midground instead of intersecting walls and blooming on them.
        e.ZOffset = spec.zoff or 0
        e.EmissionDirection = spec.dir
        -- orient: VelocityPerpendicular lays a sprite FLAT on the ground (emit up, speed ~0) —
        -- the only way settled leaves stop reading as cards standing on edge. VelocityParallel
        -- + negative Squash stretches a sprite ALONG travel, which is what makes a fast
        -- gust-driven leaf read as a streak in a still frame.
        if spec.orient then
            e.Orientation = spec.orient
        end
        e.Parent = host
        -- accel/gax/gaz stored so the follow loop can retune live wind (gust layers only);
        -- uw/aim = upwind spawn bias + downwind host aim (both need the unit wind vector);
        -- life2 = longest particle Lifetime (crossfade teardown waits it out);
        -- spec+ir = the intensity source of truth, so a slider move is a pure re-read;
        -- rateMul = the effect multiplier currently in force (swell / gust surge / devil
        -- duck), so an intensity change mid-effect re-targets instead of snapping to base.
        local L = { host = host, oy = spec.oy, emitter = e, spec = spec, ir = ir,
                    accel = spec.accel, gust = spec.gust, tag = spec.tag,
                    gax = 0, gaz = 0, rateMul = 1,
                    uw = spec.uw, aim = spec.aim,
                    life2 = spec.life[2], rateTween = nil }
        local I = iAmt()
        tuneLayer(L, I)
        local targetRate = layerRate(L, I)
        if rampIn then   -- crossfade in: emitter starts silent, ramps to target
            e.Rate = 0
            L.rateTween = TweenService:Create(e, FADE_TI, { Rate = targetRate })
            L.rateTween:Play()
        else
            e.Rate = targetRate
        end
        _partLayers[#_partLayers + 1] = L
    end

    local function clearMood()
        if _moodTween then _moodTween:Cancel(); _moodTween = nil end
        if _moodCC then pcall(function() _moodCC:Destroy() end); _moodCC = nil end
    end
    -- mood rides ONE persistent CC: type switches lerp its grade over MOOD_TI
    -- instead of popping; a fresh apply (nothing on screen yet) sets it instantly
    local function applyMood(name)
        if not cfg("WeatherMood", true) then
            clearMood()
            return
        end
        local m = MOOD[name]
        if not m then
            clearMood()
            return
        end
        if _moodTween then _moodTween:Cancel(); _moodTween = nil end
        -- grade strength rides intensity (0.66 / 1.0 / 1.4): a heavy sky grades harder.
        -- Tint is NOT scaled — the hues were picked per type and a stronger tint just
        -- colour-casts the ground. This is the only "visibility" axis the weather layer
        -- owns outright; a shared Lighting Atmosphere would fight 06_visuals (see notes).
        local g = 0.6 + 0.4 * iAmt()
        local cc = _moodCC
        if cc and cc.Parent then
            _moodTween = TweenService:Create(cc, MOOD_TI,
                { Brightness = m.B * g, Contrast = m.C * g, Saturation = m.S * g, TintColor = m.tint })
            _moodTween:Play()
            return
        end
        cc = Instance.new("ColorCorrectionEffect")
        cc.Name = "_wxMood"; cc:SetAttribute("WX_Custom", true)
        cc.Brightness = m.B * g; cc.Contrast = m.C * g; cc.Saturation = m.S * g; cc.TintColor = m.tint
        cc.Parent = Lighting
        _moodCC = cc
    end

    local function startAmbient(mood)
        if _ambient then pcall(function() _ambient:Destroy() end); _ambient = nil end
        local id = mood and ambientId(mood)
        if not id then return end
        local s = Instance.new("Sound")
        s.Name = "_wxAmb"; s:SetAttribute("WX_Custom", true)
        s.SoundId = id; s.Looped = true; s.Volume = cfg("WeatherSoundVolume", 0.35)
        s.Parent = SoundService
        pcall(function() s:Play() end)
        _ambient = s
    end

    -- one-shot SFX (thunder cracks, meteor impacts). These are IMPACT sounds: they scale the
    -- ambient slider up hard (the old code played them AT loop volume ≈0.35 — inaudible).
    local function oneShot(id, vol, pitch)
        if not id or id == "" then return end
        local s = Instance.new("Sound")
        s.Name = "_wxSfx"; s:SetAttribute("WX_Custom", true)
        s.SoundId = id; s.Volume = math.clamp(vol, 0, 10)
        if pitch then s.PlaybackSpeed = pitch end
        s.Parent = SoundService
        pcall(function() s:Play() end)
        Debris:AddItem(s, 8)   -- covers the 6.8s thunder roll
    end

    -- ── STORM ──
    -- SHARED host for storm bolts + meteors + shooting stars — only stopStorm (disable/unload) destroys it
    local function getLightFolder()
        if _lightFolder and _lightFolder.Parent then return _lightFolder end
        local f = Instance.new("Folder"); f.Name = "_wxBolts"; f:SetAttribute("WX_Custom", true)
        f.Parent = getFolder()
        _lightFolder = f; return f
    end
    local function mkBoltPart(color, transp, size, cf, parent)
        local p = Instance.new("Part")
        p.Anchored = true; p.CanCollide = false; p.CanQuery = false; p.CanTouch = false
        p.CastShadow = false; p.Massless = true; p.Material = Enum.Material.Neon
        p.Color = color; p.Transparency = transp; p:SetAttribute("WX_Custom", true)
        p.Size = size; p.CFrame = cf; p.Parent = parent
        return p
    end
    -- THUNDERSTORM (rework 2026-07-24, Studio-verified): pre-flash telegraph in the cloud →
    -- 18-seg meandering channel + recursive branches (cylinder segs: round cross-section, no
    -- boxy sleeves up close) in THREE layers — white-violet CORE (dies fast), wide soft GLOW,
    -- violet AFTERGLOW that outlives both (retinal persistence) → 30% same-channel re-strike
    -- 80-150ms later → distance-delayed thunder (dist/343 at 1/3 sound speed) + CC pop with
    -- flicker pulses. Nested IIFE keeps ~14 storm locals off the weather proto budget.
    local startStorm, stopStormLoop, stopStorm, rescheduleStorm
    ;(function()
        local TX_BGLOW   = "rbxassetid://78582616787441"     -- soft radial glow (flarecore)
        local TX_BSTREAK = "rbxassetid://102481842205398"    -- thin tapered streak (contact sparks)
        local BOLT_CORE  = Color3.fromRGB(245, 242, 255)
        local BOLT_GLOW  = Color3.fromRGB(162, 145, 255)
        local BOLT_AFTER = Color3.fromRGB(126, 96, 228)
        local ROT90 = CFrame.Angles(0, math.rad(90), 0)      -- cylinder X axis → segment axis
        local FLICK = 0.12                                    -- initial hot-flicker window (s)
        local _stormConn, _stormNextT = nil, 0
        local _boltRp = RaycastParams.new()
        _boltRp.FilterType = Enum.RaycastFilterType.Exclude

        local function mkSeg(color, transp, dia, len, cf, parent)
            local p = Instance.new("Part")
            p.Shape = Enum.PartType.Cylinder
            p.Anchored = true; p.CanCollide = false; p.CanQuery = false; p.CanTouch = false
            p.CastShadow = false; p.Massless = true; p.Material = Enum.Material.Neon
            p.Color = color; p.Transparency = transp; p:SetAttribute("WX_Custom", true)
            p.Size = Vec(len, dia, dia); p.CFrame = cf * ROT90; p.Parent = parent
            return p
        end
        -- one channel: n segments top→bot, lateral offsets follow a random-walking drift angle
        -- (meander, not zigzag). Returns the path points so branches can attach anywhere.
        -- segs records: { p, on = peak transparency, fade = seconds to die after flicker ends }.
        local function boltChannel(model, segs, top, bot, n, thick, jitter, layer3)
            local pts = { top }
            local prev = top
            local driftAng = math.random() * 6.283
            for i = 1, n do
                local t = i / n
                local point = top:Lerp(bot, t)
                if i < n then
                    driftAng = driftAng + (math.random() - 0.5) * 2.2
                    local j = jitter * (0.5 + 0.5 * (1 - t))
                    local off = j * (0.3 + math.random() * 0.7)
                    point = point + Vec(math.cos(driftAng) * off, (math.random() - 0.5) * j * 0.25, math.sin(driftAng) * off)
                end
                local len = math.max((point - prev).Magnitude, 0.1)
                local cf = CFrame.lookAt((point + prev) * 0.5, point)
                segs[#segs + 1] = { p = mkSeg(BOLT_CORE, 0.02, thick, len + thick * 1.6, cf, model), on = 0.02, fade = 0.16 }
                segs[#segs + 1] = { p = mkSeg(BOLT_GLOW, 0.78, thick * 5.5, len + thick * 1.8, cf, model), on = 0.78, fade = 0.38 }
                if layer3 then
                    segs[#segs + 1] = { p = mkSeg(BOLT_AFTER, 0.88, thick * 2.2, len + thick * 1.6, cf, model), on = 0.75, fade = 0.85 }
                end
                pts[#pts + 1] = point
                prev = point
            end
            return pts
        end
        -- recursive sub-branches: 2-4 off the main channel, thinner + shorter, each may fork again
        local function boltBranch(model, segs, fromPts, groundY, thick, depth)
            local count = 1
            if depth == 1 then count = math.random(2, 4) end
            for _ = 1, count do
                local idx = math.random(math.floor(#fromPts * 0.2), math.floor(#fromPts * 0.7))
                local a = fromPts[math.max(idx, 2)]
                local ang = math.random() * 6.283
                local drop = (a.Y - groundY) * (0.3 + math.random() * 0.25)
                local outw = drop * (0.55 + math.random() * 0.4)
                local endp = a + Vec(math.cos(ang) * outw, -drop, math.sin(ang) * outw)
                local n = 5
                if depth == 1 then n = 8 end
                -- jitter scales with branch length (fixed jitter left long branches ruler-straight)
                local blen = (endp - a).Magnitude
                local pts = boltChannel(model, segs, a, endp, n, thick, math.max(5, blen * 0.1) / depth, false)
                if depth < 3 and math.random() < 0.5 then
                    boltBranch(model, segs, pts, groundY, thick * 0.45, depth + 1)
                end
            end
        end
        -- pre-flash telegraph: two soft glow pulses at the cloud base over the strike azimuth.
        -- Also reused at ignite (scale >1) as the channel lighting the deck it exits.
        local function preFlash(pos, scale)
            local host = mkBoltPart(BOLT_CORE, 1, Vec(2, 2, 2), CFrame.new(pos), getLightFolder())
            local e = Instance.new("ParticleEmitter")
            e:SetAttribute("WX_Custom", true)
            e.Texture = TX_BGLOW
            e.Color = ColorSequence.new(Color3.fromRGB(206, 192, 255))
            e.LightEmission = 0.85; e.LightInfluence = 0
            e.Size = NumberSequence.new(60 * scale, 85 * scale)
            e.Transparency = NumberSequence.new({
                NumberSequenceKeypoint.new(0, 1), NumberSequenceKeypoint.new(0.25, 0.55),
                NumberSequenceKeypoint.new(0.7, 0.75), NumberSequenceKeypoint.new(1, 1),
            })
            e.Lifetime = NumberRange.new(0.22, 0.3)
            e.Rate = 0; e.Speed = NumberRange.new(0, 0.5)
            e.Parent = host
            e:Emit(1)
            task.delay(0.09, function() pcall(function() e:Emit(1) end) end)
            Debris:AddItem(host, 1.2)
        end
        -- screen flash: CC pop decaying over ~0.45s with 1-2 rapid flicker pulses riding the decay
        local function screenFlash()
            local cc = Instance.new("ColorCorrectionEffect")
            cc.Name = "_wxFlash"; cc:SetAttribute("WX_Custom", true); cc.Brightness = 0.35; cc.Parent = Lighting
            local pulseA = 0.3 + math.random() * 0.25
            local pulseB = -1
            if math.random() < 0.6 then pulseB = pulseA + 0.18 + math.random() * 0.12 end
            local st = tick(); local c2
            c2 = RunService.Heartbeat:Connect(function()
                if not cc.Parent then if c2 then c2:Disconnect() end return end
                local a = (tick() - st) / 0.45
                if a >= 1 then pcall(function() cc:Destroy() end); if c2 then c2:Disconnect() end return end
                local b = 0.35 * (1 - a)
                if a >= pulseA and a < pulseA + 0.14 then
                    b = b + 0.22 * (1 - (a - pulseA) / 0.14)
                elseif pulseB > 0 and a >= pulseB and a < pulseB + 0.14 then
                    b = b + 0.16 * (1 - (a - pulseB) / 0.14)
                end
                cc.Brightness = b
            end)
            Debris:AddItem(cc, 0.7)
        end
        local function igniteBolt(ground, top)
            local model = Instance.new("Model"); model.Name = "_bolt"; model:SetAttribute("WX_Custom", true)
            local segs = {}
            local mainPts = boltChannel(model, segs, top, ground, 18, 1.0, 14, true)
            boltBranch(model, segs, mainPts, ground.Y, 0.55, 1)
            -- ground hit: flat expanding shock disc (a swelling ball read as a solid pearl in
            -- Studio verify) + strike light + one bloom flash + upward spark streaks
            local dome = mkBoltPart(Color3.fromRGB(228, 240, 255), 0.5, Vec(0.6, 5, 5),
                CFrame.new(ground + Vec(0, 0.3, 0)) * CFrame.Angles(0, 0, math.rad(90)), model)
            dome.Shape = Enum.PartType.Cylinder
            local fl = Instance.new("PointLight"); fl.Color = BOLT_GLOW; fl.Range = 150; fl.Brightness = 8; fl.Parent = dome
            TweenService:Create(dome, TweenInfo.new(0.45, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
                { Size = Vec(0.6, 30, 30), Transparency = 1 }):Play()
            TweenService:Create(fl, TweenInfo.new(0.5), { Brightness = 0 }):Play()
            local cb = mkBoltPart(BOLT_CORE, 1, Vec(2, 2, 2), CFrame.new(ground + Vec(0, 1.5, 0)), model)
            local ce = Instance.new("ParticleEmitter")
            ce:SetAttribute("WX_Custom", true)
            ce.Texture = TX_BGLOW
            ce.Color = ColorSequence.new(Color3.fromRGB(235, 228, 255))
            ce.LightEmission = 1; ce.LightInfluence = 0
            ce.Size = NumberSequence.new(13, 17)
            ce.Transparency = NumberSequence.new({
                NumberSequenceKeypoint.new(0, 0.3), NumberSequenceKeypoint.new(1, 1),
            })
            ce.Lifetime = NumberRange.new(0.3, 0.4)
            ce.Rate = 0; ce.Speed = NumberRange.new(0, 0.1)
            ce.Parent = cb
            local se = Instance.new("ParticleEmitter")
            se:SetAttribute("WX_Custom", true)
            se.Texture = TX_BSTREAK
            se.Color = ColorSequence.new(Color3.fromRGB(220, 205, 255))
            se.LightEmission = 1; se.LightInfluence = 0
            se.Size = NumberSequence.new(2, 3)
            se.Transparency = NumberSequence.new(0.25, 1)
            se.Lifetime = NumberRange.new(0.5, 0.9)
            se.Rate = 0; se.Speed = NumberRange.new(26, 44)
            se.SpreadAngle = Vector2.new(38, 38)
            se.Orientation = Enum.ParticleOrientation.VelocityParallel
            se.EmissionDirection = Enum.NormalId.Top
            se.Acceleration = Vec(0, -40, 0)
            se.Parent = cb
            ce:Emit(1)
            se:Emit(9)
            model.Parent = getLightFolder()
            preFlash(top + Vec(0, 10, 0), 1.5)   -- the channel lights the cloud base it exits
            -- animation: hot flicker → (30%) re-strike of the SAME channel → per-layer fade.
            -- segs prebuilt = zero per-frame allocation.
            local flickEnd = FLICK
            local restrikeAt = -1
            if math.random() < 0.3 then restrikeAt = FLICK + 0.08 + math.random() * 0.07 end
            local startT = tick(); local conn
            conn = RunService.Heartbeat:Connect(function()
                if not model.Parent then if conn then conn:Disconnect() end return end
                local a = tick() - startT
                if restrikeAt > 0 and a >= restrikeAt then
                    restrikeAt = -1
                    flickEnd = a + 0.1
                    fl.Brightness = 6
                    TweenService:Create(fl, TweenInfo.new(0.4), { Brightness = 0 }):Play()
                end
                if a < flickEnd then
                    local dim = 0   -- flicker: mostly ON, brief dips toward off
                    if math.random() < 0.28 then dim = 0.85 end
                    for i = 1, #segs do
                        local s = segs[i]
                        s.p.Transparency = s.on + (1 - s.on) * dim
                    end
                    return
                end
                local f = a - flickEnd
                if f >= 0.85 then
                    if conn then conn:Disconnect() end
                    pcall(function() model:Destroy() end)
                    return
                end
                for i = 1, #segs do
                    local s = segs[i]
                    local k = math.clamp(f / s.fade, 0, 1)
                    s.p.Transparency = s.on + (1 - s.on) * k
                end
            end)
            Debris:AddItem(model, 1.8)   -- covers max flicker+restrike+fade
            if cfg("WeatherStormFlash", true) then screenFlash() end
            local tid = cfg("WeatherThunderId", "")
            if tid ~= "" then
                local cam = Camera
                local dist = 120
                if cam then dist = (ground - cam.CFrame.Position).Magnitude end
                task.delay(dist * (3 / 343), function()   -- dist/343 at 1/3 sound speed: readable delay
                    oneShot(tid, cfg("WeatherSoundVolume", 0.35) * 8 * (0.85 + math.random() * 0.3),
                        0.92 + math.random() * 0.16)
                end)
            end
        end
        local function spawnBolt()
            -- re-gate: barrage task.delay may land after disable/unload (getFolder would resurrect _wx)
            if not Config.Weather or not cfg("WeatherStorm", false) then return end
            local cam = Camera; if not cam then return end
            local base = cam.CFrame.Position
            -- Place the strike IN VIEW: project ahead along the camera's flattened look direction with a
            -- moderate lateral spread, so the bolt lands in FRONT of the player.
            local lv = cam.CFrame.LookVector
            local flat = Vec(lv.X, 0, lv.Z)
            if flat.Magnitude < 0.05 then flat = Vec(0, 0, -1) else flat = flat.Unit end
            local right = Vec(flat.Z, 0, -flat.X)      -- ground-plane perpendicular
            local fwd  = 55 + math.random() * 95        -- studs ahead of the camera
            local side = (math.random() - 0.5) * 85     -- lateral spread (stays within FOV)
            local gx = base.X + flat.X * fwd + right.X * side
            local gz = base.Z + flat.Z * fwd + right.Z * side
            -- plant the strike on real ground when there is any (lit floor + flash dome sell the hit)
            local ground = Vec(gx + (math.random() - 0.5) * 20, base.Y - 45, gz + (math.random() - 0.5) * 20)
            local ch = lp and lp.Character
            if ch then
                _boltRp.FilterDescendantsInstances = { getFolder(), ch }
            else
                _boltRp.FilterDescendantsInstances = { getFolder() }
            end
            local hit = Workspace:Raycast(Vec(gx, base.Y + 140, gz), Vec(0, -400, 0), _boltRp)
            if hit then ground = hit.Position end
            -- channel leans: cloud origin offset from the ground point (real bolts rarely drop plumb)
            local top = Vec(gx + (math.random() - 0.5) * 90, base.Y + 280, gz + (math.random() - 0.5) * 90)
            preFlash(Vec(top.X, top.Y - 20, top.Z), 1)
            task.delay(0.1 + math.random() * 0.2, function()   -- storms telegraph: flicker leads the strike
                if not Config.Weather or not cfg("WeatherStorm", false) then return end
                pcall(function() igniteBolt(ground, top) end)
            end)
        end
        function startStorm()
            if _stormConn then return end
            _stormNextT = tick() + 2 + math.random() * 3
            _stormConn = RunService.Heartbeat:Connect(function()
                if not Config.Weather or not cfg("WeatherStorm", false) then return end
                local now = tick()
                if now >= _stormNextT then
                    -- the Min/Var sliders ARE the storm's frequency control, alone
                    _stormNextT = now + cfg("WeatherStormMin", 4)
                        + math.random() * cfg("WeatherStormVar", 8)
                    pcall(spawnBolt)
                end
            end)
        end
        -- slider drag: pull a stale wait down into the new window. Clamping (never zeroing)
        -- is what keeps a drag from machine-gunning bolts one per tick.
        function rescheduleStorm()
            if not _stormConn then
                return
            end
            local cap = tick() + cfg("WeatherStormMin", 4) + cfg("WeatherStormVar", 8)
            _stormNextT = math.min(_stormNextT, cap)
        end
        function stopStormLoop()   -- toggle-off path: drop the scheduler, keep the shared folder
            if _stormConn then _stormConn:Disconnect(); _stormConn = nil end
        end
        function stopStorm()   -- disable/unload only: also destroys the SHARED light folder
            stopStormLoop()
            if _lightFolder then pcall(function() _lightFolder:Destroy() end); _lightFolder = nil end
        end
        -- spectacle: barrage — three telegraphed strikes in quick succession (self-gated)
        registerSpectacle("StormBarrage", 1, function()
            if not Config.Weather or not cfg("WeatherStorm", false) then return end
            for i = 0, 2 do
                task.delay(i * (0.35 + math.random() * 0.3), function() pcall(spawnBolt) end)
            end
        end)
    end)()

    -- ── camera-follow for PARTICLE layers (rain sim + special types follow in their own loops) ──
    -- Also the weather-gated tick that drives Wind.update + the spectacle scheduler.
    -- Layers with spec.gust ride the SHARED wind: Acceleration = base + Wind × per-layer
    -- scale (g.ax/g.az) × a slow per-layer phase breath (g.wx/g.wz/g.ph keep each layer's
    -- character; the direction + gust surges are one coherent field). Emitter Acceleration
    -- acts on ALREADY-LIVE particles, so the whole sky sways together — one property write
    -- per gust layer per tick. Generic (any preset may set gust); only Snow uses it today.
    local function startFollow()
        if _followConn then return end
        local now0 = tick()
        Wind.reset(now0)
        _spectNextT = now0 + 120 + math.random() * 120
        _followConn = RunService.Heartbeat:Connect(function()
            if not Config.Weather then return end
            local now = tick()
            Wind.update(now)
            local amb = _ambient   -- LIVE volume sync: slider/saved-config applies WITHOUT retoggling weather
            if amb then
                local vol = cfg("WeatherSoundVolume", 0.35)
                if amb.Volume ~= vol then amb.Volume = vol end
            end
            local cam = Camera; if not cam then return end
            local pos = cam.CFrame.Position
            -- one weighted event / 2-4 min, FIXED (no intensity): 3 of the 7 entries are
            -- sky events and the other 4 are type-gated, so at most ONE ambient spectacle
            -- is ever eligible — an intensity-driven period would mostly just re-couple
            -- the meteor/storm/star peaks we are decoupling.
            if now >= _spectNextT then
                _spectNextT = now + 120 + math.random() * 120
                if _spectWeight > 0 then
                    local r = math.random() * _spectWeight
                    for i = 1, #_spectacles do
                        local s = _spectacles[i]
                        r = r - s.weight
                        if r <= 0 then
                            pcall(s.fn)
                            break
                        end
                    end
                end
            end
            local wgain = 1   -- shared blizzard-swell envelope (1 outside a swell)
            local bt = (now - _blizzT0) / _blizzDur
            if bt >= 0 and bt < 1 then
                wgain = 1 + 1.7 * math.sin(bt * math.pi)
            end
            -- unit wind, hoisted: drives the upwind spawn bias (a windward/leeward density
            -- gradient is the strongest "the wind is real" cue in a still frame) and the
            -- downwind aim of the gust-streak hosts.
            local nwx, nwz = 1, 0
            local wl = math.sqrt(Wind.x * Wind.x + Wind.z * Wind.z)
            if wl > 0.05 then
                nwx = Wind.x / wl
                nwz = Wind.z / wl
            end
            for i = 1, #_partLayers do
                local L = _partLayers[i]
                if L.host and L.host.Parent then
                    local hx, hz = pos.X, pos.Z
                    if L.uw then
                        hx = hx - nwx * L.uw
                        hz = hz - nwz * L.uw
                    end
                    local hy = pos.Y + L.oy
                    if L.aim then
                        L.host.CFrame = CFrame.lookAt(Vec(hx, hy, hz), Vec(hx + nwx, hy, hz + nwz))
                    else
                        L.host.CFrame = CFrame.new(hx, hy, hz)
                    end
                    local g = L.gust
                    if g then
                        -- gax/gaz = the layer's coupling AFTER the intensity response
                        L.emitter.Acceleration = L.accel + Vec(
                            Wind.x * wgain * L.gax * (1 + 0.3 * math.sin(now * g.wx + g.ph)), 0,
                            Wind.z * wgain * L.gaz * (1 + 0.3 * math.cos(now * g.wz + g.ph)))
                    end
                end
            end
            for i = 1, #_fadeLayers do   -- draining layers keep following (no stranded volumes)
                local F = _fadeLayers[i]
                if F.host and F.host.Parent then
                    F.host.CFrame = CFrame.new(pos.X, pos.Y + F.oy, pos.Z)
                end
            end
        end)
    end
    local function stopFollow()
        if _followConn then _followConn:Disconnect(); _followConn = nil end
    end

    -- spectacle: WIND SWELL — a stretch where the fall thickens (Rate tween) while the wind
    -- surges (peak ×~2.7 via the _blizz envelope the follow loop reads), then eases back.
    -- SnowBlizzard = alpine whiteout; SakuraGale = the hanami gale that strips the trees and
    -- drives one long petal wave across the frame. Layers tagged "settle" (the sakura ground
    -- carpet) ride the wind but are NOT thickened — a carpet doesn't fall harder. Self-gated on
    -- type; the ease-back only touches layers still live in _partLayers (a mid-swell type switch
    -- hands them to beginLayerFade, which owns their Rate).
    do
        local SWELL_UP = TweenInfo.new(2.5, Enum.EasingStyle.Sine, Enum.EasingDirection.Out)
        local SWELL_DOWN = TweenInfo.new(3, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut)
        local function windSwell(kind, dur, rateMul)
            if not Config.Weather or _curType ~= kind then return end
            _blizzT0, _blizzDur = tick(), dur
            local surged = {}
            local I = iAmt()
            for _, L in _partLayers do
                if L.emitter and L.tag ~= "settle" then
                    if L.rateTween then L.rateTween:Cancel() end
                    L.rateMul = rateMul
                    local tw = TweenService:Create(L.emitter, SWELL_UP, { Rate = layerRate(L, I) * rateMul })
                    L.rateTween = tw
                    tw:Play()
                    surged[#surged + 1] = L
                end
            end
            task.delay(dur - 3, function()
                local I2 = iAmt()
                for _, L in surged do
                    local live = false
                    for _, cur in _partLayers do
                        if cur == L then live = true break end
                    end
                    if live and L.emitter and L.emitter.Parent then
                        if L.rateTween then L.rateTween:Cancel() end
                        L.rateMul = 1
                        local tw = TweenService:Create(L.emitter, SWELL_DOWN, { Rate = layerRate(L, I2) })
                        L.rateTween = tw
                        tw:Play()
                    end
                end
            end)
        end
        registerSpectacle("SnowBlizzard", 1, function()
            windSwell("Snow", 11 + math.random() * 3, 1.9)
        end)
        registerSpectacle("SakuraGale", 1, function()
            windSwell("Petals", 9 + math.random() * 4, 1.7)
        end)
    end

    -- AUTUMN gust surge — a gust is an EVENT, not a rate multiplier: streak layers fire leaf
    -- streams downwind across frame (staggered over ~1s so the wave has a front and a tail), a
    -- dust puff kicks off the floor, and the ambient fall thickens behind them. Ambient body rate
    -- sits low between gusts on purpose, so the lull is real negative space.
    -- Self-gated on type, rides the shared gust scheduler; the ground carpet ("settle") keeps its
    -- density (a carpet doesn't fall harder). Cancels L.rateTween first and re-stores it, so it
    -- obeys the same Rate-tween precedence as the crossfade ramp / intensity slider.
    do
        local GUST_UP   = TweenInfo.new(1.1, Enum.EasingStyle.Sine, Enum.EasingDirection.Out)
        local GUST_DOWN = TweenInfo.new(2.4, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut)
        local STREAK_BURST = { {0.3, 18}, {0.65, 14}, {1.05, 9} }
        Wind.onGust(function(dur)
            if not Config.Weather or _curType ~= "Autumn" then return end
            local I = iAmt()
            -- the burst counts were hard literals — identical at 0.15 and 2.0, which made
            -- the signature moment of the type ignore the slider entirely. They ride the
            -- same rate response as the ambient fall; the surge multiplier opens up too.
            local em = iRate(I, IR.Autumn.lo, IR.Autumn.hi)
            local gm = 1 + 0.4 * I
            local surged = {}
            for _, L in _partLayers do
                if L.emitter and L.tag == "dust" then
                    local e = L.emitter
                    local n2 = math.max(1, math.floor(10 * em + 0.5))
                    e:Emit(math.max(1, math.floor(14 * em + 0.5)))
                    task.delay(0.4, function()
                        if e.Parent then e:Emit(n2) end
                    end)
                elseif L.emitter and L.tag == "streak" then
                    local e = L.emitter
                    e:Emit(math.max(1, math.floor(20 * em + 0.5)))
                    for _, b in STREAK_BURST do
                        local n = math.max(1, math.floor(b[2] * em + 0.5))
                        task.delay(b[1], function()
                            if e.Parent then e:Emit(n) end
                        end)
                    end
                elseif L.emitter and L.tag ~= "settle" then
                    if L.rateTween then L.rateTween:Cancel() end
                    L.rateMul = gm
                    local tw = TweenService:Create(L.emitter, GUST_UP, { Rate = layerRate(L, I) * gm })
                    L.rateTween = tw
                    tw:Play()
                    surged[#surged + 1] = L
                end
            end
            task.delay(dur, function()
                local I2 = iAmt()
                for _, L in surged do
                    local live = false
                    for _, cur in _partLayers do
                        if cur == L then live = true break end
                    end
                    if live and L.emitter and L.emitter.Parent then
                        if L.rateTween then L.rateTween:Cancel() end
                        L.rateMul = 1
                        local tw = TweenService:Create(L.emitter, GUST_DOWN, { Rate = layerRate(L, I2) })
                        L.rateTween = tw
                        tw:Play()
                    end
                end
            end)
        end)
    end

    -- ════════════════════════════════════════════════════════════════════════
    -- SPECIAL WEATHER TYPES (not particle presets): BloodMoon. Its own
    -- ~10Hz anim loop parallax-follows the camera XZ; cleared
    -- like every other layer on type-switch/disable. All Instances tagged WX_Custom.
    -- ════════════════════════════════════════════════════════════════════════
    local function clearSpecial()
        if _specialConn then _specialConn:Disconnect(); _specialConn = nil end
        if _special.moon  then pcall(function() _special.moon:Destroy() end);  _special.moon  = nil end
        if _special.shell then pcall(function() _special.shell:Destroy() end); _special.shell = nil end
        if _special.sky   then pcall(function() _special.sky:Destroy() end);   _special.sky   = nil end
        if _special.cc    then pcall(function() _special.cc:Destroy() end);    _special.cc    = nil end
        if _special.atmo  then pcall(function() _special.atmo:Destroy() end);  _special.atmo  = nil end
        _special.halo = nil   -- parented to the moon; dies with it
        _special.kind = nil
    end

    -- BloodMoon's whole intensity axis. It owns its own Atmosphere already, so the pall
    -- (density / haze / glare) is the one place in the file where reduced visibility can
    -- ride the slider without a second Lighting.Atmosphere fighting the Visuals presets.
    local function tuneBloodMoon(I)
        local ms = 0.72 + 0.28 * I          -- 0.762 / 1.0 / 1.28 on every diameter
        local gr = 0.5 + 0.5 * I            -- grade strength
        if _special.moon then
            _special.moon.Size = Vec(130, 130, 130) * ms
        end
        if _special.shell then
            _special.shell.Size = Vec(160, 160, 160) * ms
        end
        if _special.halo then
            _special.halo.Rate = 25 * iRate(I, 1.45, 1.05)
            _special.halo.Size = NumberSequence.new(330 * ms)
        end
        if _special.cc then
            _special.cc.Saturation = -0.15 * gr
            _special.cc.Brightness = -0.04 * gr
        end
        if _special.atmo then
            _special.atmo.Density = 0.12 + 0.18 * I
            _special.atmo.Haze    = 0.9 + 0.9 * I
            _special.atmo.Glare   = 0.4 + 0.4 * I
        end
        if _special.sky then
            _special.sky.StarCount = math.floor(4000 * (0.55 + 0.45 * I))
        end
    end

    local function buildBloodMoon()
        local folder = getFolder()
        -- night sky (Space set) with celestial zeroed. Ids inlined (the SKY table is declared LATER
        -- in this file, so an upvalue ref here would bind to a nil global — inline keeps it correct).
        local s = Instance.new("Sky"); s.Name = "_wxSkyBM"; s:SetAttribute("WX_Custom", true)
        s.SkyboxBk = "rbxassetid://159454299"; s.SkyboxDn = "rbxassetid://159454296"
        s.SkyboxFt = "rbxassetid://159454293"; s.SkyboxLf = "rbxassetid://159454286"
        s.SkyboxRt = "rbxassetid://159454300"; s.SkyboxUp = "rbxassetid://159454288"
        s.SunAngularSize = 0; s.MoonAngularSize = 0; s.StarCount = 4000
        s.Parent = Lighting; _special.sky = s
        -- the moon: crimson disc + rim shell + soft red halo emitter, parallax-locked ~700 out
        -- (closer than the old 1500 so the red atmosphere doesn't swallow it — Studio-verified).
        local moon = Instance.new("Part")
        moon.Shape = Enum.PartType.Ball; moon.Size = Vec(130, 130, 130)
        moon.Material = Enum.Material.Neon; moon.Color = Color3.fromRGB(215, 45, 35)
        moon.Anchored = true; moon.CanCollide = false; moon.CanQuery = false; moon.CanTouch = false
        moon.CastShadow = false; moon.Massless = true; moon:SetAttribute("WX_Custom", true)
        local att = Instance.new("Attachment"); att.Parent = moon
        local halo = Instance.new("ParticleEmitter")
        halo.Texture = TX_SOFT; halo.Color = ColorSequence.new(Color3.fromRGB(255, 60, 42))
        halo.LightEmission = 1; halo.LightInfluence = 0
        halo.Rate = 25; halo.Lifetime = NumberRange.new(0.22, 0.28)
        halo.Speed = NumberRange.new(0, 0); halo.Size = NumberSequence.new(330)
        halo.Transparency = NumberSequence.new(0.72); halo.RotSpeed = NumberRange.new(0, 0)
        halo.Parent = att; _special.halo = halo
        moon.Parent = folder; _special.moon = moon
        local shell = Instance.new("Part")
        shell.Shape = Enum.PartType.Ball; shell.Size = Vec(160, 160, 160)
        shell.Material = Enum.Material.Neon; shell.Color = Color3.fromRGB(255, 60, 45); shell.Transparency = 0.7
        shell.Anchored = true; shell.CanCollide = false; shell.CanQuery = false; shell.CanTouch = false
        shell.CastShadow = false; shell.Massless = true; shell:SetAttribute("WX_Custom", true)
        shell.Parent = folder; _special.shell = shell
        -- CC tint + red atmosphere pall
        local cc = Instance.new("ColorCorrectionEffect"); cc.Name = "_wxBM"; cc:SetAttribute("WX_Custom", true)
        cc.TintColor = Color3.fromRGB(255, 165, 160); cc.Saturation = -0.15; cc.Brightness = -0.04
        cc.Parent = Lighting; _special.cc = cc
        local at = Instance.new("Atmosphere"); at.Name = "_wxBMAtmo"; at:SetAttribute("WX_Custom", true)
        at.Density = 0.3; at.Color = Color3.fromRGB(125, 35, 35); at.Decay = Color3.fromRGB(190, 55, 50)
        at.Glare = 0.8; at.Haze = 1.8; at.Parent = Lighting; _special.atmo = at
        _special.kind = "BloodMoon"
        tuneBloodMoon(iAmt())   -- the slider was a dead control on this type until now
    end

    local function startSpecialAnim()
        if _specialConn then return end
        local accum = 0
        _specialConn = RunService.Heartbeat:Connect(function(dt)
            if not Config.Weather or not _special.kind then return end
            accum = accum + dt
            if accum < 0.1 then return end   -- ~10Hz is plenty for slow sky motion
            accum = 0
            local cam = Camera; if not cam then return end
            local pos = cam.CFrame.Position
            if _special.kind == "BloodMoon" and _special.moon then
                -- parallax-lock moon + rim shell at a fixed azimuth ~700 studs out (reads celestial)
                local cf = CFrame.new(pos + Vec(0.45, 0.62, -0.64).Unit * 700)
                _special.moon.CFrame = cf
                if _special.shell then _special.shell.CFrame = cf end
            end
        end)
    end

    -- shared sequence constructors + the soft radial glow id (also used by the sky-event
    -- and lantern blocks below) — declared here so they outlive the meteor IIFE.
    local CSK, NSK = ColorSequenceKeypoint.new, NumberSequenceKeypoint.new
    local TX_MGLOW = "rbxassetid://78582616787441"     -- soft radial glow (flarecore)

    -- ════════════════════════════════════════════════════════════════════════
    -- METEORS → CINEMATIC BOLIDE. Ground-up rebuild 2026-07-25 (Studio-graded,
    -- WX_DEV_meteor v3). Own toggle WeatherMeteors, own scheduler.
    --
    -- TWO DISTANCE TIERS run in the same sky — depth is only legible relatively:
    --   FAR  (cap 2) crosses 3000-3900 studs of sky 1400-2600 out and 820-1180 up,
    --        ~12s per crossing (≈7°/s angular). Colours lerp toward MET_HAZE and
    --        base alpha rises with `hz` → smaller, dimmer, lower contrast. Ends in
    --        a terminal ablation FLARE then extinction — no impact.
    --   NEAR (cap 1) lands 90-200 studs ahead of the player after a long shallow
    --        1750-2200 stud entry from 640-800 up (~25°/s). Full contrast, sparks,
    --        shed fragment, impact. That band is the play area underfoot, so a plain
    --        ground raycast is all the landing needs.
    -- The two angular speeds side by side are what sells "huge and far away".
    --
    -- HEAD = a BURNING ROCK, never a ball: 7 dark Slate chunks tumbling as one rigid
    -- irregular body (silhouette changes in flight) + 4 velocity-locked Neon ablation
    -- fissures over the LEADING hemisphere + a nose PointLight that burns the leading
    -- faces and leaves the trailing side dark. Plasma is a TEARDROP, not a halo: four
    -- LockedToPart glow sprites stepped BACK along the velocity axis, each larger and
    -- dimmer (compact white-hot nose ahead of the rock → wide faint deep-orange wake).
    -- Three incommensurate sines (≈6.5/10.7/16.5 Hz) drive an irregular combustion
    -- flicker on every glow, fissure and light.
    --
    -- TAIL carries the visual mass (head glow ≈7% of ribbon length). Continuity is
    -- structural: a Trail cannot bead. Four CONCENTRIC Stretch-mapped soft-glow Trails
    -- with staggered lifetimes (0.26 / 0.72 / 1.85 / 3.2s) and a cross-section
    -- temperature ramp — white → gold → deep orange → red → dark ember. (Wrap-tiling a
    -- radial texture beads at speed; an untextured Trail reads as a hard polyline;
    -- one Stretch layer alone fades at head AND tail. The stagger fixes all three.)
    -- SMOKE is particles only, in two ranks, both LightInfluence-shaded (never
    -- self-lit) and both wind-driven: a dense TRAIN behind the fire and a sparse,
    -- huge, 24-38s SCAR that lingers and shears with the wind after the pass. Both
    -- rates are SOLVED from flight speed so spacing stays a fraction of the puff
    -- diameter at any speed — that is what kills the old string-of-pearls.
    --
    -- IMPACT: soft-sprite flash + brief hard core + light pop → ground shockwave → flat
    -- 360° dust ring → ember fountain → fire column → mushrooming smoke pillar, all
    -- under a lingering crater afterglow light so the dust column reads at night.
    -- The flash is a SPRITE: an expanding Neon ball big enough to read from 100 studs
    -- shows a hard silhouette rim and looks like a glass dome, not an explosion.
    -- SPECTACLE "MeteorBigOne": a big near bolide that calves 2-3 children mid-flight.
    -- Live-particle budget ≈460 at full occupancy (2 far + 1 near).
    -- ════════════════════════════════════════════════════════════════════════
    local startMeteors, stopMeteors, rescheduleMeteors
    ;(function()
        local MET_TXS  = "rbxassetid://77935321198144"   -- billowy smoke puff
        local MET_TXK  = "rbxassetid://102481842205398"  -- thin tapered streak
        local MET_TXD  = "rbxassetid://341828512"        -- tan dust billow
        local MET_HAZE = Color3.fromRGB(128, 146, 176)   -- atmospheric-perspective target
        -- plasma teardrop: z = offset along velocity (−ahead), b0/b1 = flicker base/gain
        local MET_GLOW = {
            { z = -0.80, size = 0.85, tr = 0.03, col = Color3.fromRGB(255, 246, 216), life = 0.09, rate = 34, b0 = 1.35, b1 = 2.1 },
            { z = -0.10, size = 1.75, tr = 0.55, col = Color3.fromRGB(255, 194, 106), life = 0.10, rate = 30, b0 = 0.85, b1 = 1.4 },
            { z =  1.30, size = 3.10, tr = 0.80, col = Color3.fromRGB(255, 124,  46), life = 0.11, rate = 26, b0 = 0.5, b1 = 0.8 },
            { z =  3.20, size = 5.20, tr = 0.92, col = Color3.fromRGB(224,  72,  26), life = 0.12, rate = 22, b0 = 0.4, b1 = 0.6 },
        }
        -- rock: { sx, sy, sz, ox, oy, oz, rx, ry, rz, r, g, b, transparency, tumbles }
        local MET_ROCK = {
            { 1.00, 0.74, 1.28,  0.00,  0.00,  0.00, 0.25, 0.40, 0.18, 58, 51, 47, 0.00, true },
            { 0.70, 0.60, 0.76,  0.40,  0.20,  0.12, 0.80, 0.50, 1.10, 47, 41, 39, 0.00, true },
            { 0.58, 0.48, 0.64, -0.38, -0.26,  0.34, 1.20, 0.90, 0.30, 36, 31, 30, 0.00, true },
            { 0.46, 0.40, 0.52,  0.06, -0.42, -0.24, 0.40, 1.30, 0.70, 52, 45, 42, 0.00, true },
            { 0.36, 0.30, 0.40, -0.32,  0.36, -0.08, 0.90, 0.20, 1.40, 42, 36, 34, 0.00, true },
            { 0.30, 0.26, 0.34,  0.22, -0.10,  0.46, 1.50, 0.70, 0.20, 38, 33, 32, 0.00, true },
            { 0.26, 0.34, 0.22, -0.20, -0.34, -0.02, 0.60, 1.10, 0.90, 33, 29, 28, 0.00, true },
            { 0.52, 0.40, 0.16,  0.10,  0.06, -0.56, 0.00, 0.00, 0.50, 255, 244, 214, 0.10, false },
            { 0.30, 0.46, 0.14, -0.30, -0.14, -0.44, 0.00, 0.00, -0.90, 255, 190, 104, 0.28, false },
            { 0.34, 0.20, 0.12,  0.16, -0.34, -0.40, 0.00, 0.00, 0.20, 255, 158,  66, 0.36, false },
            { 0.22, 0.16, 0.10, -0.10,  0.34, -0.38, 0.00, 0.00, 1.20, 255, 214, 150, 0.30, false },
        }
        local MET_ABL0 = 8            -- MET_ROCK index where the Neon ablation fissures start
        local MET_ABLA = { 0.06, 0.20, 0.26, 0.22 }   -- fissure base alpha (flicker rides on top)
        local MET_ABLK = { 0.20, 0.24, 0.26, 0.26 }
        local MET_UP, MET_DOWN = Vec(0, 300, 0), Vec(0, -800, 0)
        local _metConn = nil
        local _metNextFar, _metNextNear = 0, 0
        local _metFar, _metNear = 0, 0
        local _metLive = {}
        local _metRp = RaycastParams.new()
        _metRp.FilterType = Enum.RaycastFilterType.Exclude

        local function hz(c, k)
            return c:Lerp(MET_HAZE, k)
        end
        local function metPart(sx, sy, sz, cf)
            local p = Instance.new("Part")
            p.Anchored = true; p.CanCollide = false; p.CanQuery = false; p.CanTouch = false
            p.CastShadow = false; p.Massless = true
            p.TopSurface = Enum.SurfaceType.Smooth; p.BottomSurface = Enum.SurfaceType.Smooth
            p.Size = Vec(sx, sy, sz); p.CFrame = cf
            p:SetAttribute("WX_Custom", true)
            p.Parent = getLightFolder()
            return p
        end
        -- our own folder and the player must never be raycast against
        local function metFilter()
            local ch = nil
            if lp then ch = lp.Character end
            if ch then
                _metRp.FilterDescendantsInstances = { getFolder(), ch }
            else
                _metRp.FilterDescendantsInstances = { getFolder() }
            end
        end
        -- Land in the play area ahead of the player: 90-200 studs out with a modest lateral
        -- spread, plain ground raycast. That band is the ground the player is standing on,
        -- so no visibility/occlusion cleverness is needed. One retry at half distance covers
        -- a cramped map; only a player over a true void falls through to an airburst.
        local function metSolveLand(base, out, tang)
            local d = 90 + math.random() * 110
            local lat = (math.random() - 0.5) * 160
            for _ = 1, 2 do
                local c = base + out * d + tang * lat
                local h = Workspace:Raycast(c + MET_UP, MET_DOWN, _metRp)
                if h then
                    return h.Position + Vec(0, 1.5, 0)
                end
                d = d * 0.5
                lat = lat * 0.5
            end
            return nil
        end
        -- LockedToPart glow sprite. Parent MUST be a Part: a "locked" emitter parented
        -- to an Attachment does not lock and smears a dotted line along the arc.
        local function metGlow(host, size, transp, color, life, rate)
            local g = Instance.new("ParticleEmitter")
            g:SetAttribute("WX_Custom", true)
            g.Texture = TX_MGLOW; g.Color = ColorSequence.new(color)
            g.LightEmission = 1; g.LightInfluence = 0
            g.Rate = rate; g.Lifetime = NumberRange.new(life, life)
            g.Size = NumberSequence.new(size); g.Transparency = NumberSequence.new(transp)
            g.Speed = NumberRange.new(0, 0); g.LockedToPart = true
            g.Parent = host
            return g
        end
        -- one concentric ribbon layer; sep is the MAX width (WidthScale only scales down)
        local function metRibbon(part, sep, c0, c1, a0, w1, life, emit, inf)
            local aT = Instance.new("Attachment"); aT.Position = Vec(0, sep * 0.5, 0); aT.Parent = part
            local aB = Instance.new("Attachment"); aB.Position = Vec(0, -sep * 0.5, 0); aB.Parent = part
            local tr = Instance.new("Trail")
            tr:SetAttribute("WX_Custom", true)
            tr.Attachment0 = aT; tr.Attachment1 = aB; tr.FaceCamera = true
            tr.Texture = TX_MGLOW; tr.TextureMode = Enum.TextureMode.Stretch; tr.TextureLength = 1
            tr.Color = ColorSequence.new(c0, c1)
            tr.Transparency = NumberSequence.new({ NSK(0, a0), NSK(0.6, a0 + (1 - a0) * 0.45), NSK(1, 1) })
            tr.WidthScale = NumberSequence.new({ NSK(0, 1), NSK(1, w1) })
            tr.Lifetime = life; tr.LightEmission = emit; tr.LightInfluence = inf
            tr.MinLength = 0.05
            tr.Parent = part
            return { t = tr, a = aT, b = aB, sep = sep }
        end
        -- shaded smoke rank. rate is SOLVED by the caller from flight speed so that
        -- spacing (spd/rate) stays a fraction of the birth diameter → never beads.
        local function metSmoke(host, sb, lo, hi, rate, aBase, drift, spread, k)
            local e = Instance.new("ParticleEmitter")
            e:SetAttribute("WX_Custom", true)
            e.Texture = MET_TXS
            e.Color = ColorSequence.new({
                CSK(0,   hz(Color3.fromRGB(178, 164, 152), k * 0.5)),
                CSK(0.2, hz(Color3.fromRGB(146, 142, 138), k * 0.5)),
                CSK(1,   hz(Color3.fromRGB(84, 82, 80), k * 0.5)) })
            e.LightEmission = 0.02; e.LightInfluence = 0.6   -- shaded, never self-illuminated
            e.Rate = rate; e.Lifetime = NumberRange.new(lo, hi)
            e.Size = NumberSequence.new({ NSK(0, sb), NSK(0.35, sb * 1.9), NSK(1, sb * 3) })
            e.Transparency = NumberSequence.new({
                NSK(0, 0.94), NSK(0.07, aBase + 0.16 * k), NSK(0.62, aBase + 0.16), NSK(1, 1) })
            e.Speed = NumberRange.new(0, spread); e.SpreadAngle = Vector2.new(60, 60)
            e.Drag = 0.5
            e.RotSpeed = NumberRange.new(-8, 8); e.Rotation = NumberRange.new(0, 360)
            -- one-shot wind sample: the scar outlives the bolide by ~40s, so a live
            -- coupling would need its own loop for an imperceptible gain
            e.Acceleration = Vec(Wind.x * drift, 0.35, Wind.z * drift)
            e.Parent = host
            return e
        end
        local function bez(p0, p1, p2, a)
            return p0:Lerp(p1, a):Lerp(p1:Lerp(p2, a), a)
        end

        local function metImpact(pos, sc)
            -- lens safety: pulled-in landings can be close, and a flash that sweeps
            -- THROUGH the camera reads as a frame-filling cutout. Scale down, never up.
            if Camera then
                sc = math.min(sc, math.max(0.85, (pos - Camera.CFrame.Position).Magnitude / 44))
            end
            local host = metPart(4 * sc, 4 * sc, 4 * sc, CFrame.new(pos))
            host.Shape = Enum.PartType.Ball; host.Material = Enum.Material.Neon
            host.Color = Color3.fromRGB(255, 240, 202); host.Transparency = 0.05
            local fl = Instance.new("PointLight")
            fl.Color = Color3.fromRGB(255, 176, 90); fl.Range = 150; fl.Brightness = 9; fl.Parent = host
            -- crater afterglow: without it the shaded dust column reads as a black blob at night
            local glow = metPart(2, 2, 2, CFrame.new(pos + Vec(0, 3, 0)))
            glow.Transparency = 1
            local gl = Instance.new("PointLight")
            gl.Color = Color3.fromRGB(255, 148, 68); gl.Range = 120 * sc; gl.Brightness = 4; gl.Parent = glow
            local ring = metPart(12 * sc, 2 * sc, 12 * sc, CFrame.new(pos))
            ring.Shape = Enum.PartType.Ball; ring.Material = Enum.Material.Neon
            ring.Color = Color3.fromRGB(255, 150, 60); ring.Transparency = 0.35
            local scorch = metPart(15 * sc, 0.6, 15 * sc, CFrame.new(pos - Vec(0, 0.4, 0)))
            scorch.Shape = Enum.PartType.Ball; scorch.Material = Enum.Material.Neon
            scorch.Color = Color3.fromRGB(255, 104, 24); scorch.Transparency = 0.2
            local att = Instance.new("Attachment"); att.Parent = host
            -- sideways attachment + yaw-only spread sweeps the dust as a flat 360° disc
            local datt = Instance.new("Attachment"); datt.Orientation = Vec(0, 0, -90); datt.Parent = host

            -- the flash proper: soft radial sprite, so there is no silhouette rim
            local flash = Instance.new("ParticleEmitter")
            flash:SetAttribute("WX_Custom", true)
            flash.Texture = TX_MGLOW; flash.Rate = 0
            flash.Color = ColorSequence.new({
                CSK(0,    Color3.fromRGB(255, 246, 214)),
                CSK(0.45, Color3.fromRGB(255, 186, 96)),
                CSK(1,    Color3.fromRGB(226, 96, 34)) })
            flash.LightEmission = 1; flash.LightInfluence = 0
            flash.Lifetime = NumberRange.new(0.30, 0.30)
            flash.Size = NumberSequence.new({ NSK(0, 14 * sc), NSK(0.35, 34 * sc), NSK(1, 44 * sc) })
            flash.Transparency = NumberSequence.new({ NSK(0, 0.05), NSK(0.45, 0.42), NSK(1, 1) })
            flash.Speed = NumberRange.new(0, 0); flash.Rotation = NumberRange.new(0, 360)
            flash.Parent = att
            flash:Emit(2)

            local dust = Instance.new("ParticleEmitter")
            dust:SetAttribute("WX_Custom", true)
            dust.Texture = MET_TXD; dust.Rate = 0
            dust.Color = ColorSequence.new(Color3.fromRGB(198, 178, 152), Color3.fromRGB(120, 112, 104))
            dust.LightEmission = 0.05; dust.LightInfluence = 0.5
            dust.Lifetime = NumberRange.new(5.5, 9)
            dust.Size = NumberSequence.new({ NSK(0, 6 * sc), NSK(1, 22 * sc) })
            dust.Transparency = NumberSequence.new({ NSK(0, 0.6), NSK(0.1, 0.42), NSK(0.6, 0.68), NSK(1, 1) })
            dust.Speed = NumberRange.new(22, 38); dust.SpreadAngle = Vector2.new(0, 180)
            dust.Drag = 1.9; dust.EmissionDirection = ND.Top
            dust.RotSpeed = NumberRange.new(-16, 16); dust.Rotation = NumberRange.new(0, 360)
            dust.Acceleration = Vec(Wind.x * 6, 0.4, Wind.z * 6)
            dust.Parent = datt
            dust:Emit(math.floor(24 * sc))

            local fount = Instance.new("ParticleEmitter")
            fount:SetAttribute("WX_Custom", true)
            fount.Texture = MET_TXK; fount.Rate = 0
            fount.Color = ColorSequence.new(Color3.fromRGB(255, 200, 110), Color3.fromRGB(196, 52, 14))
            fount.LightEmission = 1; fount.LightInfluence = 0
            fount.Lifetime = NumberRange.new(1.1, 2.2)
            fount.Size = NumberSequence.new({ NSK(0, 1.7 * sc), NSK(1, 0.08) })
            fount.Transparency = NumberSequence.new({ NSK(0, 0), NSK(0.7, 0.35), NSK(1, 1) })
            fount.Orientation = Enum.ParticleOrientation.VelocityParallel
            fount.Speed = NumberRange.new(48, 96); fount.SpreadAngle = Vector2.new(30, 30)
            fount.Acceleration = Vec(0, -74, 0); fount.Drag = 0.35
            fount.EmissionDirection = ND.Top; fount.Parent = att
            fount:Emit(math.floor(52 * sc))

            local col = Instance.new("ParticleEmitter")
            col:SetAttribute("WX_Custom", true)
            col.Texture = TX_MGLOW; col.Rate = 0
            col.Color = ColorSequence.new({
                CSK(0,    Color3.fromRGB(255, 236, 180)),
                CSK(0.35, Color3.fromRGB(255, 130, 40)),
                CSK(1,    Color3.fromRGB(122, 40, 18)) })
            col.LightEmission = 0.9; col.LightInfluence = 0
            col.Lifetime = NumberRange.new(0.55, 1.05)
            col.Size = NumberSequence.new({ NSK(0, 5 * sc), NSK(1, 14 * sc) })
            col.Transparency = NumberSequence.new({ NSK(0, 0.12), NSK(0.6, 0.55), NSK(1, 1) })
            col.Speed = NumberRange.new(32, 62); col.SpreadAngle = Vector2.new(10, 10)
            col.EmissionDirection = ND.Top; col.Parent = att
            col:Emit(18)

            local pil = Instance.new("ParticleEmitter")
            pil:SetAttribute("WX_Custom", true)
            pil.Texture = MET_TXS; pil.Rate = 0
            pil.Color = ColorSequence.new({
                CSK(0,   Color3.fromRGB(186, 150, 118)),   -- fire-lit base
                CSK(0.3, Color3.fromRGB(146, 140, 132)),
                CSK(1,   Color3.fromRGB(88, 85, 82)) })
            pil.LightEmission = 0.02; pil.LightInfluence = 0.5
            pil.Lifetime = NumberRange.new(4.5, 8)
            pil.Size = NumberSequence.new({ NSK(0, 4.5 * sc), NSK(1, 20 * sc) })
            pil.Transparency = NumberSequence.new({ NSK(0, 0.52), NSK(0.5, 0.68), NSK(1, 1) })
            pil.Speed = NumberRange.new(16, 30); pil.SpreadAngle = Vector2.new(9, 9)
            pil.Acceleration = Vec(Wind.x * 7, 2.6, Wind.z * 7); pil.Drag = 0.6
            pil.EmissionDirection = ND.Top
            pil.RotSpeed = NumberRange.new(-12, 12); pil.Rotation = NumberRange.new(0, 360)
            pil.Parent = att
            -- staggered bursts = a COLUMN; one same-age burst reads as a detached cannonball
            task.delay(0.12, function() if pil.Parent then pil:Emit(7) end end)
            task.delay(0.62, function() if pil.Parent then pil:Emit(5) end end)
            task.delay(1.30, function() if pil.Parent then pil:Emit(4) end end)

            local t0 = tick()
            local conn
            conn = RunService.Heartbeat:Connect(function()
                if not host.Parent then
                    conn:Disconnect()
                    return
                end
                local k = tick() - t0
                local f1 = math.clamp(k / 0.10, 0, 1)   -- core snaps out; the sprite carries the flash
                host.Size = Vec(1, 1, 1) * (4 + 6 * f1) * sc
                host.Transparency = 0.05 + 0.95 * f1
                fl.Brightness = 9 * (1 - math.clamp(k / 0.55, 0, 1))
                gl.Brightness = 4 * (1 - math.clamp(k / 5, 0, 1))
                local f2 = math.clamp(k / 0.55, 0, 1)
                ring.Size = Vec(12 + 52 * f2, 2 - 1.2 * f2, 12 + 52 * f2) * sc
                ring.Transparency = 0.35 + 0.65 * f2
                scorch.Transparency = 0.2 + 0.8 * math.clamp(k / 7, 0, 1)
                if k > 7.2 then
                    conn:Disconnect()
                end
            end)
            Debris:AddItem(ring, 1.1)
            Debris:AddItem(glow, 5.2)
            Debris:AddItem(scorch, 7.5)
            Debris:AddItem(host, 11)   -- dust (9) + pillar (1.3 delay + 8 life) need the host alive
        end

        -- One bolide. opts (all optional): big, ang, land, start, scaleMul, noSplit.
        local function launch(far, opts)
            local ang = math.random() * 6.28318
            local mul = 1
            local big = false
            if opts then
                if opts.ang then ang = opts.ang end
                if opts.scaleMul then mul = opts.scaleMul end
                big = opts.big == true
            end
            local tang = Vec(-math.sin(ang), 0, math.cos(ang))
            local out = Vec(math.cos(ang), 0, math.sin(ang))
            local sgn = 1
            if math.random() < 0.5 then sgn = -1 end
            local base = Camera.CFrame.Position

            local startP, endP, spd, rs, ws, k, doImpact, smk
            if far then
                local ctr = base + out * (1400 + math.random() * 1200)
                          + Vec(0, 820 + math.random() * 360, 0)
                local travel = 3000 + math.random() * 900
                local drop = 320 + math.random() * 200
                startP = ctr - tang * (travel * 0.5 * sgn) + Vec(0, drop * 0.5, 0)
                endP = ctr + tang * (travel * 0.5 * sgn) - Vec(0, drop * 0.5, 0)
                spd = 255 + math.random() * 70
                rs = 14 + math.random() * 5
                ws = 3.9 + math.random()
                k = 0.14 + math.random() * 0.10
                doImpact = false
                smk = 1.6   -- far puffs are 4x wider; the same overlap needs proportionally less rate
            else
                metFilter()
                local land = nil
                if opts and opts.land then
                    local h = Workspace:Raycast(opts.land + MET_UP, MET_DOWN, _metRp)
                    if h then land = h.Position + Vec(0, 1.5, 0) end
                end
                if not land then land = metSolveLand(base, out, tang) end
                doImpact = land ~= nil
                if not land then
                    land = base + out * 150 + Vec(0, 40, 0)   -- player over a void ⇒ airburst
                end
                local travel = 1750 + math.random() * 450
                startP = land - tang * (travel * sgn) + out * (190 + math.random() * 190)
                         + Vec(0, 640 + math.random() * 160, 0)
                if opts and opts.start then startP = opts.start end
                endP = land
                spd = 175 + math.random() * 38
                -- head is deliberately scaled UP without ws: the ratio, not the absolute
                -- size, is what makes the rock read against its own ribbon
                rs = (7.0 + math.random() * 2.6) * mul
                ws = (1.3 + math.random() * 0.28) * mul
                k = 0
                smk = 1
            end
            if big then
                rs = rs * 1.7
                ws = ws * 1.5
                spd = spd * 0.92
            end
            local flare = far or (not doImpact)
            local splitAt = nil
            if big and not (opts and opts.noSplit) then
                splitAt = 0.42 + math.random() * 0.12
            end

            local mid = startP:Lerp(endP, 0.5) + Vec(0, (endP - startP).Magnitude * 0.075, 0)
            local dur = ((startP - mid).Magnitude + (mid - endP).Magnitude) / spd
            local cf0 = CFrame.lookAt(startP, endP)
            local root = metPart(0.2, 0.2, 0.2, cf0)
            root.Transparency = 1
            local a0 = Instance.new("Attachment"); a0.Parent = root

            local glows = table.create(4)
            for i, s in MET_GLOW do
                local hp = metPart(0.2, 0.2, 0.2, cf0)
                hp.Transparency = 1
                glows[i] = { p = hp, z = s.z * rs, b0 = s.b0, b1 = s.b1,
                    e = metGlow(hp, s.size * rs, math.min(0.97, s.tr + 0.18 * k),
                        hz(s.col, k * 0.55), s.life, s.rate) }
            end
            local chunks = table.create(#MET_ROCK)
            for i, r in MET_ROCK do
                local p = metPart(r[1] * rs, r[2] * rs, r[3] * rs, cf0)
                p.Transparency = r[13]
                if r[14] then
                    p.Material = Enum.Material.Slate
                    p.Color = Color3.fromRGB(r[10], r[11], r[12])
                else
                    p.Material = Enum.Material.Neon
                    p.Color = hz(Color3.fromRGB(r[10], r[11], r[12]), k * 0.4)
                end
                chunks[i] = { p = p, tumble = r[14],
                    off = CFrame.new(r[4] * rs, r[5] * rs, r[6] * rs) * CFrame.Angles(r[7], r[8], r[9]) }
            end
            local noseLight, bodyLight = nil, nil
            if not far then
                -- sits AHEAD of the rock: burns the leading faces, trailing side stays dark
                noseLight = Instance.new("PointLight")
                noseLight.Color = Color3.fromRGB(255, 186, 108)
                noseLight.Range = 4.2 * rs; noseLight.Brightness = 11
                noseLight.Parent = glows[1].p
                bodyLight = Instance.new("PointLight")
                bodyLight.Color = Color3.fromRGB(255, 150, 70)
                bodyLight.Range = 16 * rs; bodyLight.Brightness = 3
                bodyLight.Parent = root
            end

            local ha = 0.20 * k
            local ribbons = {
                metRibbon(root, 7.5 * ws, hz(Color3.fromRGB(255, 253, 248), k * 0.3),
                    hz(Color3.fromRGB(255, 226, 156), k * 0.6), ha, 0.45, 0.26, 1, 0),
                metRibbon(root, 14 * ws, hz(Color3.fromRGB(255, 222, 146), k * 0.5),
                    hz(Color3.fromRGB(255, 136, 40), k * 0.9), 0.06 + ha, 0.36, 0.72, 1, 0),
                metRibbon(root, 23 * ws, hz(Color3.fromRGB(255, 138, 42), k * 0.9),
                    hz(Color3.fromRGB(196, 50, 14), k), 0.22 + ha, 0.28, 1.85, 1, 0),
                metRibbon(root, 33 * ws, hz(Color3.fromRGB(196, 62, 18), k),
                    hz(Color3.fromRGB(88, 26, 14), k), 0.52 + ha, 0.30, 3.20, 0.55, 0.12),
            }
            -- rate is solved so consecutive puffs sit ~0.30 of a birth diameter apart.
            -- At 0.55 (train) and 0.85 (scar) the ranks beaded into a string of pearls
            -- the moment the landing came inside ~200 studs; the old clamp CEILINGS were
            -- the binding limit, so both the ceiling and the birth size went up.
            local trainRate = math.clamp(spd / (17 * ws * 0.30 * smk), 6, 30)
            local train = metSmoke(a0, 17 * ws, 5, 7.5, trainRate, 0.56, 5.6, 3, k)
            local scarRate = math.clamp(spd / (50 * ws * 0.28 * smk), 1.6, 9)
            local scar = metSmoke(a0, 50 * ws, 20, 32, scarRate, 0.74, 7.8, 7, k)

            local sparks = nil
            if not far then
                sparks = Instance.new("ParticleEmitter")
                sparks:SetAttribute("WX_Custom", true)
                sparks.Texture = MET_TXK
                sparks.Color = ColorSequence.new(Color3.fromRGB(255, 172, 70), Color3.fromRGB(186, 44, 12))
                sparks.LightEmission = 1; sparks.LightInfluence = 0
                sparks.Rate = 60; sparks.Lifetime = NumberRange.new(0.6, 1.5)
                sparks.Size = NumberSequence.new({ NSK(0, 2.2 * ws), NSK(1, 0.06) })
                sparks.Transparency = NumberSequence.new({ NSK(0, 0.08), NSK(1, 1) })
                sparks.Orientation = Enum.ParticleOrientation.VelocityParallel
                sparks.Speed = NumberRange.new(6, 20); sparks.SpreadAngle = Vector2.new(32, 32)
                sparks.Acceleration = Vec(0, -12, 0); sparks.Drag = 0.6
                sparks.Parent = a0
            end

            if far then _metFar = _metFar + 1 else _metNear = _metNear + 1 end
            local rec = { root = root }
            _metLive[#_metLive + 1] = rec

            local t0 = tick()
            local ph = math.random() * 20
            local spinX, spinY, spinZ = (math.random() - 0.5) * 3, (math.random() - 0.5) * 3.4,
                0.6 + math.random() * 1.6
            local flareAt = 0.76 + math.random() * 0.10
            local shedAt = 0.25 + math.random() * 0.2
            local dead, endT = false, nil
            local conn
            local function retire()
                if dead then
                    return
                end
                dead = true
                if far then
                    _metFar = math.max(0, _metFar - 1)
                else
                    _metNear = math.max(0, _metNear - 1)
                end
                for i = #_metLive, 1, -1 do
                    if _metLive[i] == rec then table.remove(_metLive, i) end
                end
            end
            conn = RunService.Heartbeat:Connect(function()
                if not root.Parent then
                    conn:Disconnect()
                    retire()
                    return
                end
                local now = tick()
                local a = math.clamp((now - t0) / dur, 0, 1)
                local p = bez(startP, mid, endP, a)
                local d = (mid - startP) * (2 * (1 - a)) + (endP - mid) * (2 * a)
                if d.Magnitude < 1e-3 then d = endP - startP end
                local cf = CFrame.lookAt(p, p + d)
                -- irregular combustion: three incommensurate sines ≈ 6.5 / 10.7 / 16.5 Hz
                local n = math.clamp(0.5 + 0.27 * math.sin(now * 41 + ph)
                    + 0.17 * math.sin(now * 67.3 + ph * 2.1)
                    + 0.09 * math.sin(now * 103.7 + ph * 0.7), 0, 1)
                local burn = 1
                if flare and a > flareAt then
                    local u = (a - flareAt) / (1 - flareAt)
                    burn = math.clamp((1 + 1.7 * math.exp(-((u - 0.14) ^ 2) / 0.011))
                        * (1 - u) ^ 1.5, 0, 3)   -- terminal ablation flare, then extinction
                end
                if endT then burn = burn * math.clamp(1 - (now - endT) / 0.4, 0, 1) end
                local bw = math.clamp(burn, 0, 1)

                if not endT then
                    root.CFrame = cf
                    for _, g in glows do
                        g.p.CFrame = cf * CFrame.new(0, 0, g.z)
                    end
                    local tum = CFrame.Angles(now * spinX, now * spinY, now * spinZ)
                    for _, c in chunks do
                        if c.tumble then
                            c.p.CFrame = cf * tum * c.off
                        else
                            c.p.CFrame = cf * c.off
                        end
                    end
                end
                for _, g in glows do
                    g.e.Brightness = (g.b0 + g.b1 * n) * burn
                end
                for i = 1, 4 do
                    local c = chunks[MET_ABL0 + i - 1]
                    c.p.Transparency = math.clamp(1 - (1 - (MET_ABLA[i] + MET_ABLK[i] * (1 - n))) * bw, 0, 1)
                end
                if noseLight then
                    noseLight.Brightness = (7 + 9 * n) * bw
                    bodyLight.Brightness = (2 + 3 * n) * bw
                end
                train.Rate = trainRate * bw
                scar.Rate = scarRate * bw
                if sparks then sparks.Rate = 60 * bw * (0.6 + 0.8 * n) end
                local pinch = (0.5 + 0.5 * bw) * (0.9 + 0.2 * n)   -- flame flutter + burn-out taper
                for _, L in ribbons do
                    local h = L.sep * 0.5 * pinch
                    L.a.Position = Vec(0, h, 0)
                    L.b.Position = Vec(0, -h, 0)
                end

                if splitAt and a >= splitAt then   -- the big one calves child bolides
                    splitAt = nil
                    if sparks then sparks:Emit(30) end
                    for _ = 1, math.random(2, 3) do
                        launch(false, {
                            ang = ang,
                            noSplit = true,
                            scaleMul = 0.40 + math.random() * 0.14,
                            start = p + Vec((math.random() - 0.5) * 30, (math.random() - 0.5) * 20,
                                (math.random() - 0.5) * 30),
                            land = endP + Vec((math.random() - 0.5) * 110, 0, (math.random() - 0.5) * 110),
                        })
                    end
                end
                if (not far) and a > shedAt then   -- ablation: shed one burning fragment
                    shedAt = 2
                    local fp = metPart(0.3 * rs, 0.24 * rs, 0.38 * rs, CFrame.new(p))
                    fp.Material = Enum.Material.Slate
                    fp.Color = Color3.fromRGB(40, 35, 33)
                    metRibbon(fp, 6, Color3.fromRGB(255, 220, 160), Color3.fromRGB(150, 34, 12),
                        0.12, 0.2, 0.8, 1, 0)
                    local fdir = (d.Unit * 0.65
                        + Vec(math.random() - 0.5, -0.5, math.random() - 0.5).Unit * 0.5).Unit
                    local fv, ft0 = fdir * spd * 0.6, tick()
                    local fc
                    fc = RunService.Heartbeat:Connect(function()
                        if not fp.Parent then
                            fc:Disconnect()
                            return
                        end
                        local age = tick() - ft0
                        if age > 2.4 then
                            fc:Disconnect()
                            fp:Destroy()
                            return
                        end
                        fv = fv + Vec(0, -0.88, 0)
                        fp.CFrame = CFrame.new(fp.Position + fv * 0.016)
                            * CFrame.Angles(age * 5, age * 3, age * 4)
                    end)
                end

                if not endT then
                    if a >= 1 then
                        endT = now
                        if doImpact then
                            -- coefficient tracks rs so a bigger head does NOT grow the blast
                            metImpact(endP, rs * 0.215 + 0.6)
                            local tid = cfg("WeatherThunderId", "")
                            if tid ~= "" then
                                task.delay(0.15, function()
                                    oneShot(tid, cfg("WeatherSoundVolume", 0.35) * 6)
                                end)
                            end
                        end
                    elseif flare and burn < 0.02 and a > flareAt then
                        endT = now
                    end
                elseif now - endT >= 0.4 then
                    retire()
                    conn:Disconnect()
                    train.Enabled = false; scar.Enabled = false
                    if sparks then sparks.Enabled = false end
                    for _, g in glows do
                        g.e.Enabled = false
                        g.p:Destroy()
                    end
                    for _, L in ribbons do L.t.Enabled = false end
                    for _, c in chunks do c.p:Destroy() end
                    if noseLight then noseLight:Destroy(); bodyLight:Destroy() end
                    Debris:AddItem(root, 42)   -- the 24-38s scar has to finish drifting first
                end
            end)
            Debris:AddItem(root, dur + 46)   -- safety net if the loop never reaches a terminal state
        end

        local function spawnMeteor(far, opts)
            -- re-gate: a scheduled spawn may land after disable/unload
            if not Config.Weather or not cfg("WeatherMeteors", false) then
                return
            end
            -- the rate slider also widens how many may share the sky (a fast cadence into
            -- a cap of 1 just silently drops spawns), but both tiers stay hard-bounded:
            -- 3 near + 4 far at the top of the bar.
            local r = evRate("WeatherMeteorRate")
            local capped
            if far then
                capped = _metFar >= math.clamp(math.floor(1 + 1.5 * r), 1, 4)
            elseif r >= 2.5 then
                capped = _metNear >= 3
            elseif r >= 1.5 then
                capped = _metNear >= 2
            else
                capped = _metNear >= 1
            end
            if capped then
                return
            end
            launch(far, opts)
        end

        startMeteors = function()
            if _metConn then
                return
            end
            local t = tick()
            _metNextFar = t + 1 + math.random() * 2
            _metNextNear = t + 5 + math.random() * 6
            _metConn = RunService.Heartbeat:Connect(function()
                if not Config.Weather or not cfg("WeatherMeteors", false) then
                    return
                end
                local now = tick()
                local r = evRate("WeatherMeteorRate")
                if now >= _metNextFar then
                    _metNextFar = now + (6 + math.random() * 5) / r
                    pcall(spawnMeteor, true)
                end
                if now >= _metNextNear then
                    _metNextNear = now + (14 + math.random() * 10) / r
                    pcall(spawnMeteor, false)
                end
            end)
        end
        -- slider drag: clamp a stale wait to the new tier maximum (11s / 24s at rate 1) so
        -- the change is felt now. Clamping never lands ON `now`, so a drag can't burst.
        rescheduleMeteors = function()
            if not _metConn then
                return
            end
            local now, r = tick(), evRate("WeatherMeteorRate")
            _metNextFar = math.min(_metNextFar, now + 11 / r)
            _metNextNear = math.min(_metNextNear, now + 24 / r)
        end
        stopMeteors = function()
            if _metConn then _metConn:Disconnect(); _metConn = nil end
            -- the shared light folder survives a toggle-off, so kill live bolides here:
            -- destroying the root drops each flight loop on its next tick
            for _, rec in _metLive do
                pcall(function() rec.root:Destroy() end)
            end
            table.clear(_metLive)
            -- counters are NOT zeroed here: each destroyed root's flight loop still fires
            -- once and retires itself, which is what brings them back to 0
        end

        -- rare spectacle: the big one, which fragments mid-flight. Self-gated (the
        -- scheduler does not filter by type/toggle) and cap-exempt by design.
        registerSpectacle("MeteorBigOne", 1, function()
            if not Config.Weather or not cfg("WeatherMeteors", false) then
                return
            end
            launch(false, { big = true })
        end)
    end)()

    -- ════════════════════════════════════════════════════════════════════════
    -- SKY EVENTS — shooting stars (own toggle). Studio-verified night 2026-07-25.
    -- Each streak is a three-beat event: a star-point TWINKLE at the spawn site
    -- 0.3-0.6s early (the catch-your-eye), then the STREAK — a tapered FaceCamera
    -- Trail led by a tiny locked-sprite head — and, on long crossings only, a faint
    -- ionization TRAIN that outlives the head. Three flight classes (zip / medium /
    -- long graceful) × three compositions (white / ice-blue / rare faint gold);
    -- gold runs slow and long, as real iron-rich meteors do. ~6% of events are a
    -- RADIANT SHOWER: 3-5 streaks over a couple of seconds on great circles leaving
    -- one sky point, sharing one composition.
    -- TRAIL SAFETY (this ribbon moves 100-350 studs/s — the hard-white-polyline
    -- failure mode lives here): textured (soft radial glow) + Stretch + FaceCamera +
    -- width AND transparency tapers + Lifetime derived from the desired tail length
    -- (life = tail / speed, so a fast zip gets a short-lived ribbon, never a smear).
    -- The head-end pinch-ball is killed structurally: over the last 30% the
    -- attachment separation is lerped to zero, so the ribbon tapers to nothing at
    -- the head while still travelling, and Enabled goes false at flight end so the
    -- remainder ages out instead of collapsing onto a stopped part.
    -- Nested IIFE keeps ~23 locals off this proto's registers.
    -- ════════════════════════════════════════════════════════════════════════
    local startStars, stopStars, rescheduleStars
    ;(function()
        local TX_SGLOW = "rbxassetid://78582616787441"     -- soft radial glow (flarecore)
        local TX_STAR4 = "rbxassetid://17726943419"        -- 4-point star (twinkle-in)
        local GLINT_C  = Color3.fromRGB(246, 250, 255)
        local MIN_SIN  = 0.3                               -- ≈17°: never seed a streak into the horizon
        local FLOOR_H  = 60                                -- studs above the eye the head may never sink past
        -- head, mid, tail per composition
        local PAL = {
            { WHITE, Color3.fromRGB(222, 234, 255), Color3.fromRGB(150, 184, 240) },
            { Color3.fromRGB(242, 250, 255), Color3.fromRGB(160, 206, 255), Color3.fromRGB(78, 138, 255) },
            { Color3.fromRGB(255, 246, 220), Color3.fromRGB(255, 205, 122), Color3.fromRGB(222, 140, 44) },
        }
        -- dist0, distSpan, speed0, speedSpan, tail0, tailSpan, ribbon width
        local CLS = {
            { 80, 55, 250, 105, 50, 28, 3.6 },
            { 145, 90, 158, 62, 76, 44, 5.4 },
            { 240, 150, 98, 46, 118, 62, 7.2 },
        }
        local TR_TRANSP  = NumberSequence.new({ NSK(0, 0.04), NSK(0.12, 0.12), NSK(0.45, 0.55), NSK(1, 1) })
        local TR_WIDTH   = NumberSequence.new({ NSK(0, 0.4), NSK(0.08, 1), NSK(1, 0.02) })
        local ION_TRANSP = NumberSequence.new({ NSK(0, 0.86), NSK(0.3, 0.9), NSK(1, 1) })
        local ION_WIDTH  = NumberSequence.new({ NSK(0, 0.5), NSK(0.25, 1), NSK(1, 0.35) })
        local HEAD_TR    = NumberSequence.new({ NSK(0, 1), NSK(0.05, 0), NSK(0.72, 0.06), NSK(1, 1) })
        local HALO_TR    = NumberSequence.new({ NSK(0, 1), NSK(0.07, 0.82), NSK(0.7, 0.88), NSK(1, 1) })
        local GLINT_TR   = NumberSequence.new({ NSK(0, 1), NSK(0.3, 0.12), NSK(0.55, 0.42), NSK(1, 1) })
        local _starConn, _starNextT = nil, 0
        local _starLive = {}

        -- one locked billboard sprite: emitted once, its own sequences do the fade,
        -- so a streak needs zero per-frame work beyond position + ribbon taper
        local function mkSprite(parent, tex, col, sizeSeq, transpSeq, life, emit)
            local e = Instance.new("ParticleEmitter")
            e.Texture = tex
            e.Color = ColorSequence.new(col)
            e.Size = sizeSeq
            e.Transparency = transpSeq
            e.Lifetime = NumberRange.new(life)
            e.Rate = 0
            e.Speed = NumberRange.new(0, 0)
            e.SpreadAngle = Vector2.new(0, 0)
            e.LightEmission = emit
            e.LightInfluence = 0
            e.Drag = 0
            e.Parent = parent
            return e
        end

        -- `floorY` = the eye height + FLOOR_H, resolved by the caller (spawns are
        -- camera-relative, but a streak is world-anchored once it exists)
        local function spawnStreak(start, dir, ci, pi, floorY)
            if not Config.Weather or not cfg("WeatherShootingStars", false) then
                return
            end
            if #_starLive >= math.clamp(math.floor(4 + 2 * evRate("WeatherStarRate")), 4, 8) then
                return
            end
            local cls, pal = CLS[ci], PAL[pi]
            local dist = cls[1] + math.random() * cls[2]
            local spd  = cls[3] + math.random() * cls[4]
            local tail = cls[5] + math.random() * cls[6]
            -- re-pitch so a long flight can never carry the head down through the horizon
            if start.Y + dir.Y * dist < floorY then
                local ny = (floorY - start.Y) / dist
                local hl = math.sqrt(dir.X * dir.X + dir.Z * dir.Z)
                if hl > 1e-4 then
                    local k = math.sqrt(math.max(0, 1 - ny * ny)) / hl
                    dir = Vec(dir.X * k, ny, dir.Z * k)
                end
            end
            local dur  = dist / spd
            local life = math.min(tail / spd, dur * 0.9)   -- tail length in TIME: never a smear
            local sep  = cls[7] * (0.85 + math.random() * 0.3)
            local glintD = 0.3 + math.random() * 0.3
            local cf0 = CFrame.lookAt(start, start + dir)
            local host = mkBoltPart(WHITE, 1, Vec(0.2, 0.2, 0.2), cf0, getLightFolder())
            local aT = Instance.new("Attachment"); aT.Position = Vec(0, sep * 0.5, 0); aT.Parent = host
            local aB = Instance.new("Attachment"); aB.Position = Vec(0, -sep * 0.5, 0); aB.Parent = host
            local tr = Instance.new("Trail")
            tr.Attachment0 = aT; tr.Attachment1 = aB
            tr.FaceCamera = true
            tr.Texture = TX_SGLOW; tr.TextureMode = Enum.TextureMode.Stretch; tr.TextureLength = 1
            tr.Color = ColorSequence.new({ CSK(0, pal[1]), CSK(0.24, pal[2]), CSK(1, pal[3]) })
            tr.Transparency = TR_TRANSP; tr.WidthScale = TR_WIDTH
            tr.Lifetime = life; tr.LightEmission = 1; tr.LightInfluence = 0
            tr.MinLength = 0.08; tr.Enabled = false
            tr.Parent = host
            local ion = nil
            if ci == 3 then                                -- long crossings keep a faint train
                ion = Instance.new("Trail")
                ion.Attachment0 = aT; ion.Attachment1 = aB
                ion.FaceCamera = true
                ion.Texture = TX_SGLOW; ion.TextureMode = Enum.TextureMode.Stretch; ion.TextureLength = 1
                ion.Color = ColorSequence.new({ CSK(0, pal[2]), CSK(1, pal[3]) })
                ion.Transparency = ION_TRANSP; ion.WidthScale = ION_WIDTH
                ion.Lifetime = life * 2.6; ion.LightEmission = 0.85; ion.LightInfluence = 0
                ion.MinLength = 0.1; ion.Enabled = false
                ion.Parent = host
            end
            local hub = Instance.new("Attachment"); hub.Parent = host
            local coreS = sep * 0.62                       -- head stays a POINT, never an orb
            local core = mkSprite(hub, TX_SGLOW, pal[1], NumberSequence.new({
                NSK(0, coreS * 0.55), NSK(0.1, coreS), NSK(1, coreS * 0.3) }), HEAD_TR, dur, 1)
            core.LockedToPart = true
            local haloS = sep * 1.7
            local halo = mkSprite(hub, TX_SGLOW, pal[2], NumberSequence.new({
                NSK(0, haloS * 0.5), NSK(0.12, haloS), NSK(1, haloS * 0.35) }), HALO_TR, dur, 1)
            halo.LockedToPart = true
            -- twinkle-in: world-space (NOT locked) so it stays behind at the seed point;
            -- LightEmission 0.55 keeps the 4-point silhouette from blooming into a blob
            local gs = sep * 1.9
            local gl = mkSprite(hub, TX_STAR4, GLINT_C, NumberSequence.new({
                NSK(0, gs * 0.1), NSK(0.32, gs), NSK(1, gs * 0.14) }), GLINT_TR, glintD * 1.2, 0.55)
            gl.Rotation = NumberRange.new(0, 90)
            gl.RotSpeed = NumberRange.new(-16, 16)
            gl:Emit(1)
            table.insert(_starLive, {
                host = host, tr = tr, ion = ion, core = core, halo = halo, aT = aT, aB = aB,
                cf0 = cf0, dir = dir, dist = dist, dur = dur, sep = sep,
                t0 = tick() + glintD, started = false,
            })
            Debris:AddItem(host, glintD + dur + life * 2.8 + 0.6)
        end

        local function pickPal()
            local r = math.random()
            if r < 0.55 then
                return 1
            end
            if r < 0.92 then
                return 2
            end
            return 3
        end
        local function pickCls(pi)
            if pi == 3 then                                -- gold burns slow: long or medium only
                if math.random() < 0.6 then
                    return 3
                end
                return 2
            end
            local r = math.random()
            if r < 0.25 then
                return 1
            end
            if r < 0.7 then
                return 2
            end
            return 3
        end

        -- soft bias toward the camera's facing so the effect is actually SEEN, without
        -- pinning every streak to screen centre
        local function viewAz(cam)
            local lv = cam.CFrame.LookVector
            local az = math.atan2(lv.Z, lv.X)
            if math.random() < 0.35 then
                return math.random() * 6.283
            end
            return az
        end

        local function spawnSingle()
            local cam = Camera
            if not cam then
                return
            end
            local base = cam.CFrame.Position
            local az = viewAz(cam) + (math.random() - 0.5) * 1.5
            local el = math.rad(24 + math.random() * 30)
            local r  = 190 + math.random() * 130
            local ce = math.cos(el)
            local start = base + Vec(ce * math.cos(az) * r, math.sin(el) * r, ce * math.sin(az) * r)
            local hd = az + 1.5708 + (math.random() - 0.5) * 2.2
            local pi = pickPal()
            spawnStreak(start, Vec(math.cos(hd), -(0.05 + math.random() * 0.34), math.sin(hd)).Unit,
                pickCls(pi), pi, base.Y + FLOOR_H)
        end

        -- radiant shower: every member leaves the SAME sky point along a great circle,
        -- angular offset drives class (near the radiant = short zip, far = long crossing)
        local function spawnShower()
            local cam = Camera
            if not cam then
                return
            end
            local base = cam.CFrame.Position
            local floorY = base.Y + FLOOR_H
            local raz = viewAz(cam) + (math.random() - 0.5) * 0.9
            local rel = math.rad(36 + math.random() * 16)
            local cr = math.cos(rel)
            local rv = Vec(cr * math.cos(raz), math.sin(rel), cr * math.sin(raz))
            local dn = (Vec(0, -1, 0) + rv * rv.Y).Unit    -- tangent at the radiant pointing down
            local sd = rv:Cross(dn).Unit
            local pi = pickPal()
            local t = 0
            for _ = 1, 3 + math.random(0, 2) do
                t = t + 0.16 + math.random() * 0.42
                task.delay(t, function()
                    if not Config.Weather or not cfg("WeatherShootingStars", false) then
                        return
                    end
                    local phi = (math.random() - 0.5) * 2.8
                    local off = dn * math.cos(phi) + sd * math.sin(phi)
                    local th = math.rad(9 + math.random() * 24)
                    local ct, st = math.cos(th), math.sin(th)
                    local u = (rv * ct + off * st).Unit
                    if u.Y < MIN_SIN then                  -- mirror to the far side of the radiant
                        off = -off
                        u = (rv * ct + off * st).Unit
                    end
                    if u.Y < MIN_SIN then
                        return
                    end
                    local ci = 2
                    if th < 0.22 then
                        ci = 1
                    elseif th > 0.44 then
                        ci = 3
                    end
                    pcall(spawnStreak, base + u * (215 + math.random() * 75),
                        (off * ct - rv * st).Unit, ci, pi, floorY)
                end)
            end
        end

        local function stepStars(now)
            for i = #_starLive, 1, -1 do                   -- reverse: entries are removed in place
                local s = _starLive[i]
                if not s.host.Parent then
                    table.remove(_starLive, i)
                else
                    local a = (now - s.t0) / s.dur
                    if a >= 1 then
                        s.tr.Enabled = false               -- ribbon ages out; never collapses on a stopped part
                        if s.ion then
                            s.ion.Enabled = false
                        end
                        table.remove(_starLive, i)
                    elseif a >= 0 then
                        if not s.started then
                            s.started = true
                            s.tr.Enabled = true
                            s.core:Emit(1)
                            s.halo:Emit(1)
                            if s.ion then
                                s.ion.Enabled = true
                            end
                        end
                        s.host.CFrame = s.cf0 + s.dir * (s.dist * a)
                        if a > 0.7 then                    -- taper the ribbon to nothing at the head
                            local h = s.sep * 0.5 * (1 - (a - 0.7) / 0.3)
                            s.aT.Position = Vec(0, h, 0)
                            s.aB.Position = Vec(0, -h, 0)
                        end
                    end
                end
            end
        end

        startStars = function()
            if _starConn then
                return
            end
            _starNextT = tick() + 2 + math.random() * 4
            _starConn = RunService.Heartbeat:Connect(function()
                if not Config.Weather or not cfg("WeatherShootingStars", false) then
                    return
                end
                local now = tick()
                if now >= _starNextT then
                    local r = evRate("WeatherStarRate")
                    _starNextT = now + (5 + math.random() * 6) / r
                    if math.random() < math.min(0.06 * r, 0.35) then
                        pcall(spawnShower)
                    else
                        pcall(spawnSingle)
                        if math.random() < math.min(0.22 * r, 0.6) then   -- occasional companion
                            task.delay(0.3 + math.random() * 0.5, function()
                                pcall(spawnSingle)
                            end)
                        end
                    end
                end
                pcall(stepStars, now)
            end)
        end
        -- slider drag: clamp a stale wait to the new maximum event gap (11s at rate 1)
        rescheduleStars = function()
            if not _starConn then
                return
            end
            _starNextT = math.min(_starNextT, tick() + 11 / evRate("WeatherStarRate"))
        end
        stopStars = function()
            if _starConn then
                _starConn:Disconnect()
                _starConn = nil
            end
            -- the shared light folder survives a toggle-off, so drop our hosts ourselves
            for _, s in _starLive do
                pcall(function() s.host:Destroy() end)
            end
            table.clear(_starLive)
        end

        -- rare peak: a radiant shower on the spectacle scheduler (registry is permanent —
        -- module load only, gated on our own toggle)
        registerSpectacle("StarShower", 1, function()
            if not cfg("WeatherShootingStars", false) then
                return
            end
            spawnShower()
        end)
    end)()

    -- ════════════════════════════════════════════════════════════════════════
    -- HERO FIREFLIES — 4 real neon motes, each with a warm PointLight, gliding
    -- between smoothstep-lerped waypoints near the ground (the light pools grazing
    -- floor/walls sell the night). One ~20Hz loop; waypoint picks ground-raycast.
    -- Runs only while _curType == "Fireflies" (started/stopped by applyType).
    -- SPECTACLE "FireflySwarm": heroes converge on a point, orbit it trailing
    -- soft-glow Trail ribbons, then disperse; an Emit-burst emitter at the point
    -- densifies the swarm. Studio-verified 2026-07-24.
    -- TRAIL SAFETY (the ribbon degenerates to a hard white polyline the moment one
    -- frame's segment stretches over many studs): FaceCamera + textured + tapered +
    -- short Lifetime, movement hard-clamped to FF_STEP studs per update, and the
    -- ribbon only ever runs during the calm phases (wander + orbit) below FF_SEG —
    -- every jump/sprint leg runs Enabled=false + :Clear(). Measured under an
    -- adversarial 44-stud-teleport stress: max segment while enabled = 1.16 studs.
    -- Nested IIFE keeps ~20 locals off the weather proto budget.
    -- ════════════════════════════════════════════════════════════════════════
    local startFireflies, stopFireflies, refreshFireflies
    ;(function()
        local FF_N     = 7                               -- built once; intensity PARKS the spares
        local FF_BODY  = Color3.fromRGB(215, 255, 120)
        local FF_LIGHT = Color3.fromRGB(244, 232, 120)
        local FF_DOWN  = Vec(0, -80, 0)
        local WANDER   = 28                              -- waypoint box (studs) around the camera
        local FF_STEP  = 2                               -- hard movement clamp, studs per update
        local FF_SEG   = 1.2                             -- ribbon allowed only below this segment
        local FF_TRC = ColorSequence.new({ CSK(0, Color3.fromRGB(198, 255, 128)),
            CSK(0.45, Color3.fromRGB(216, 238, 108)), CSK(1, Color3.fromRGB(255, 196, 86)) })
        local FF_TRT = NumberSequence.new({ NSK(0, 0.36), NSK(0.35, 0.66), NSK(1, 1) })
        local FF_TRW = NumberSequence.new({ NSK(0, 0.8), NSK(0.3, 1), NSK(1, 0) })
        local _ff = { list = nil, conn = nil, folder = nil, burst = nil, burstHost = nil, n = 4 }
        local _swarm = { active = false, t0 = 0, cx = 0, cz = 0, y = 0 }
        local _ffRp = RaycastParams.new()
        _ffRp.FilterType = Enum.RaycastFilterType.Exclude

        -- ground height near (x, z); fixed drop below the camera on void maps
        local function groundY(camPos, x, z)
            local ch = lp and lp.Character
            if ch then
                _ffRp.FilterDescendantsInstances = { getFolder(), ch }
            else
                _ffRp.FilterDescendantsInstances = { getFolder() }
            end
            local hit = Workspace:Raycast(Vec(x, camPos.Y + 6, z), FF_DOWN, _ffRp)
            if hit then
                return hit.Position.Y
            end
            return camPos.Y - 7
        end
        local function pickWaypoint(H, camPos)
            local x = camPos.X + (math.random() - 0.5) * WANDER
            local z = camPos.Z + (math.random() - 0.5) * WANDER
            H.a = H.pos
            H.bpt = Vec(x, groundY(camPos, x, z) + 0.5 + math.random() * 3, z)
            H.dur = math.clamp((H.bpt - H.a).Magnitude / 2.2, 1.5, 6)
            H.t = 0
        end
        -- move `cur` toward `target` but never further than `maxStep` in one update:
        -- caps every segment the ribbon can lay down, whatever the choreography asks for
        local function stepToward(cur, target, maxStep)
            local d = target - cur
            local m = d.Magnitude
            if m <= maxStep or m < 1e-4 then
                return target
            end
            return cur + d * (maxStep / m)
        end
        local function trailOff(H)
            H.calm = 0
            if H.trail.Enabled then
                H.trail.Enabled = false
                H.trail:Clear()
            end
        end
        local function buildFlies()
            local folder = Instance.new("Folder")
            folder.Name = "_wxFlies"; folder:SetAttribute("WX_Custom", true)
            folder.Parent = getFolder()
            _ff.folder = folder
            _ff.list = {}
            local cam = Camera
            local camPos
            if cam then camPos = cam.CFrame.Position else camPos = Vec(0, 0, 0) end
            for i = 1, FF_N do
                local b = mkBoltPart(FF_BODY, 0, Vec(0.25, 0.25, 0.25), CFrame.new(camPos), folder)
                b.Shape = Enum.PartType.Ball
                local light = Instance.new("PointLight")
                light.Color = FF_LIGHT; light.Range = 8; light.Brightness = 2.5
                light.Parent = b
                local halo = Instance.new("ParticleEmitter")
                halo:SetAttribute("WX_Custom", true)
                halo.Texture = TX_MGLOW
                halo.Color = ColorSequence.new(FF_BODY)
                halo.LightEmission = 1; halo.LightInfluence = 0
                halo.Size = NumberSequence.new(1.1, 1.5)
                halo.Transparency = NumberSequence.new({ NSK(0, 0.75), NSK(1, 1) })
                halo.Rate = 12; halo.Lifetime = NumberRange.new(0.25, 0.4)
                halo.Speed = NumberRange.new(0, 0)
                halo.LockedToPart = true    -- unlocked, a fast hero smears it into a beaded dot-line
                halo.Parent = b
                local a0 = Instance.new("Attachment")
                a0.Position = Vec(0, 0.6, 0); a0.Parent = b
                local a1 = Instance.new("Attachment")
                a1.Position = Vec(0, -0.6, 0); a1.Parent = b
                local tr = Instance.new("Trail")
                tr:SetAttribute("WX_Custom", true)
                tr.Attachment0 = a0; tr.Attachment1 = a1
                tr.Texture = TX_MGLOW; tr.TextureMode = Enum.TextureMode.Stretch
                tr.Color = FF_TRC; tr.Transparency = FF_TRT; tr.WidthScale = FF_TRW
                tr.Lifetime = 0.65
                tr.LightEmission = 1; tr.LightInfluence = 0
                tr.FaceCamera = true        -- an edge-on untextured ribbon is the white-line failure
                tr.MinLength = 0.25; tr.MaxLength = 14
                tr.Enabled = false          -- armed by the loop once the hero is moving slowly
                tr.Parent = b
                local H = { part = b, light = light, trail = tr, halo = halo, bm = 1,
                            calm = 0, jump = true,
                            pos = camPos, a = camPos, bpt = camPos, t = 0, dur = 1,
                            ph = math.random() * 6.283, spin = 1.7 + i * 0.25,
                            blinkT = math.random() * 3, period = 2.4 + math.random() * 1.4 }
                pickWaypoint(H, camPos)     -- first pick = spawn point
                H.pos = H.bpt
                pickWaypoint(H, camPos)     -- real first leg starts from the spawn point
                b.CFrame = CFrame.new(H.pos)
                _ff.list[i] = H
            end
            -- swarm density burst: tiny invisible host repositioned at the swarm point
            local bh = mkBoltPart(FF_BODY, 1, Vec(1.5, 1.5, 1.5), CFrame.new(camPos), folder)
            local be = Instance.new("ParticleEmitter")
            be:SetAttribute("WX_Custom", true)
            be.Texture = TX_MGLOW
            be.Color = ColorSequence.new(Color3.fromRGB(220, 255, 130), Color3.fromRGB(255, 200, 80))
            be.LightEmission = 0.95; be.LightInfluence = 0.05
            be.Size = NumberSequence.new({ NSK(0, 0.1), NSK(0.08, 0.62, 0.15), NSK(0.45, 0.5, 0.1), NSK(1, 0.08) })
            be.Transparency = NumberSequence.new({ NSK(0, 1), NSK(0.08, 0.08), NSK(0.4, 0.45), NSK(0.75, 0.85), NSK(1, 1) })
            be.Rate = 0; be.Lifetime = NumberRange.new(1.6, 2.8)
            be.Speed = NumberRange.new(0.8, 2.2); be.SpreadAngle = Vector2.new(180, 180)
            be.Drag = 2.5; be.EmissionDirection = Enum.NormalId.Top
            be.Parent = bh
            _ff.burst = be; _ff.burstHost = bh
        end
        -- INTENSITY for the hero motes: how many are awake (1 / 4 / 7) and how far their
        -- light pools reach. All FF_N exist from the start, so the slider parks or wakes
        -- them with property writes only — no rebuild, no reseeded wander.
        refreshFireflies = function()
            local list = _ff.list
            if not list then
                return
            end
            local I = iAmt()
            _ff.n = math.clamp(math.floor(1.5 + 3 * I), 1, FF_N)
            local rm, bm = 0.8 + 0.2 * I, 0.7 + 0.3 * I
            for i = 1, FF_N do
                local H = list[i]
                local on = i <= _ff.n
                H.bm = bm
                H.light.Range = 8 * rm
                H.halo.Enabled = on
                if not on then
                    H.light.Brightness = 0
                    H.part.Transparency = 1
                    H.jump = true   -- so waking it later can't lay a ribbon across the gap
                    trailOff(H)
                end
            end
        end
        startFireflies = function()
            if _ff.conn then return end
            buildFlies()
            refreshFireflies()
            local accum = 0
            _ff.conn = RunService.Heartbeat:Connect(function(dt)
                if not Config.Weather then return end
                accum = accum + dt
                if accum < 0.05 then return end          -- ~20Hz is plenty for a glide
                local step = accum; accum = 0
                local cam = Camera; if not cam then return end
                local camPos = cam.CFrame.Position
                local now = tick()
                local sw = -1
                if _swarm.active then
                    sw = now - _swarm.t0
                    if sw > 9 then _swarm.active = false; sw = -1 end
                end
                local list = _ff.list
                for i = 1, _ff.n do
                    local H = list[i]
                    local prev = H.pos
                    local newPos
                    if sw >= 0 then
                        -- swarm: converge (0-2.5s) → orbit (2.5-6.5s) → disperse (6.5-9s)
                        local ang = H.ph + now * H.spin
                        local r
                        if sw < 2.5 then r = 10 - 7.4 * (sw / 2.5)
                        elseif sw < 6.5 then r = 2.6
                        else r = 2.6 + (sw - 6.5) * 6 end
                        local target = Vec(_swarm.cx + math.cos(ang) * r,
                            _swarm.y + 0.55 * math.sin(now * 1.3 + H.ph * 2),
                            _swarm.cz + math.sin(ang) * r)
                        newPos = stepToward(prev, prev:Lerp(target, math.min(step * 3, 1)), FF_STEP)
                        H.bpt = newPos; H.t = 1; H.dur = 1  -- swarm exit rolls straight into a fresh waypoint
                    else
                        H.t = H.t + step
                        local relX, relZ = prev.X - camPos.X, prev.Z - camPos.Z
                        if H.t >= H.dur or (relX * relX + relZ * relZ) > 3600 then
                            pickWaypoint(H, camPos)      -- leg done, or the player outran the flies
                        end
                        local u = H.t / H.dur
                        u = u * u * (3 - 2 * u)          -- smoothstep glide
                        newPos = stepToward(prev, H.a:Lerp(H.bpt, u), FF_STEP)
                    end
                    H.pos = newPos
                    -- ribbon gate: calm phases only (wander + orbit), and only while this
                    -- update's segment stayed short — anything else clears it before it draws
                    if H.jump or (sw >= 0 and (sw < 2.5 or sw >= 6.4))
                        or (newPos - prev).Magnitude > FF_SEG then
                        H.jump = false
                        trailOff(H)
                    else
                        H.calm = H.calm + step
                        if H.calm >= 0.25 and not H.trail.Enabled then
                            H.trail:Clear()
                            H.trail.Enabled = true
                        end
                    end
                    H.part.CFrame = CFrame.new(H.pos.X, H.pos.Y + 0.3 * math.sin(now * 1.7 + H.ph), H.pos.Z)
                    -- blink: sharp attack, slow exponential decay, never fully dark
                    local pulse = math.exp(-((now + H.blinkT) % H.period) * 1.4)
                    H.light.Brightness = (0.5 + 2.6 * pulse) * H.bm
                    H.part.Transparency = 0.4 * (1 - pulse)
                end
            end)
        end
        stopFireflies = function()
            if _ff.conn then _ff.conn:Disconnect(); _ff.conn = nil end
            if _ff.folder then pcall(function() _ff.folder:Destroy() end); _ff.folder = nil end
            _ff.list = nil; _ff.burst = nil; _ff.burstHost = nil
            _swarm.active = false
        end
        -- spectacle: converging swarm — self-gated (the scheduler doesn't filter by type)
        registerSpectacle("FireflySwarm", 1, function()
            if not Config.Weather or _curType ~= "Fireflies" or not _ff.list then return end
            local cam = Camera; if not cam then return end
            local camPos = cam.CFrame.Position
            local ang = math.random() * 6.283
            local d = 8 + math.random() * 8
            local cx = camPos.X + math.cos(ang) * d
            local cz = camPos.Z + math.sin(ang) * d
            _swarm.cx = cx; _swarm.cz = cz
            _swarm.y = groundY(camPos, cx, cz) + 2.2
            _swarm.t0 = tick(); _swarm.active = true
            _ff.burstHost.CFrame = CFrame.new(cx, _swarm.y, cz)
            for _, H in _ff.list do
                trailOff(H)                               -- kill the ribbon before the convergence sprint
            end
            local em = math.max(1, math.floor(7 * iRate(iAmt(), IR.Fireflies.lo, IR.Fireflies.hi) + 0.5))
            for n = 0, 5 do
                task.delay(1.6 + n * 0.7, function()
                    if _swarm.active and _ff.burst then _ff.burst:Emit(em) end
                end)
            end
        end)
    end)()

    -- ════════════════════════════════════════════════════════════════════════
    -- LEAF DEVIL (Autumn spectacle) — a ground vortex that spins up out of the
    -- floor, climbs, spirals leaves up a flaring column, wanders downwind, then
    -- LETS GO and blows the whole spiral away.
    -- What makes it read as a VORTEX and not a clump:
    --  (1) a STACK of ring emitters, one per height band, each ORBITING its own
    --      radius — every band paints an actual ring of leaves in world space,
    --      and a per-band phase twist up the stack lays those rings into a
    --      helix silhouette. Scattered "arms" produce a cloud instead.
    --  (2) radius profile: narrow foot, u^0.9 flare, extra crown flare — the
    --      silhouette is a funnel, never a bar.
    --  (3) rate proportional to ring radius (wide bands carry more leaves) and
    --      an 8° spread, so SKY STAYS VISIBLE BETWEEN THE ORBITS. Anything that
    --      fills the column interior kills the rotation read.
    --  (4) each ring emits along its orbit TANGENT at a speed below the ring's
    --      own sweep, so leaves stream sideways and get left behind — that lag
    --      is what the eye reads as spin.
    --  (5) a FOOT: leaf skirt + dry-dust skirt + a ground scuff decal, so the
    --      column visibly touches down. A dark tall smoke column was the single
    --      loudest "this is a fire" cue — the base is dry tan dust, low.
    --  (6) it ARRIVES: rings ramp in bottom-up and the column climbs over ~1.5s.
    --  (7) ambient leaf rate is suppressed to 45% for the event — against ~800
    --      ambient leaves already in frame, a local column simply does not
    --      register as a spectacle.
    -- Nested IIFE keeps its locals off the weather proto budget. One folder,
    -- one connection, self-terminating; stopLeafDevil is the disable/unload path.
    -- ════════════════════════════════════════════════════════════════════════
    local stopLeafDevil
    ;(function()
        local RING_N   = 11        -- ring emitters stacked up the column
        local DEV_H    = 25        -- column height (studs)
        local DEV_R    = 9.5       -- crown radius; the foot is 15% of it
        local DEV_LIFE = 14        -- total event: climb + sustain + release + drain
        local DEV_HOLD = 9.6       -- sustain ends / release begins
        local DEV_DOWN = Vec(0, -80, 0)
        local RELEASE  = Vec(-8, -3.5, 3)
        -- species palette, cycled up the stack: gold and straw glow, the rest stay silhouettes
        local RING_TEX  = { TX_LEAFA, TX_LEAFB, TX_LEAFA, TX_LEAFB, TX_LEAFC }
        local RING_GLOW = { 0.62, 0.07, 0.6, 0.08, 0.1 }
        local RING_COL  = {
            ColorSequence.new({ CSK(0, Color3.fromRGB(226,190,128)), CSK(0.45, Color3.fromRGB(255,246,200)),
                                CSK(1, Color3.fromRGB(190,132,62)) }),
            ColorSequence.new({ CSK(0, Color3.fromRGB(160,80,52)), CSK(1, Color3.fromRGB(104,44,30)) }),
            ColorSequence.new({ CSK(0, Color3.fromRGB(230,222,176)), CSK(0.5, Color3.fromRGB(255,254,228)),
                                CSK(1, Color3.fromRGB(200,180,122)) }),
            ColorSequence.new({ CSK(0, Color3.fromRGB(156,74,255)), CSK(1, Color3.fromRGB(106,46,200)) }),
            ColorSequence.new({ CSK(0, Color3.fromRGB(196,156,112)), CSK(1, Color3.fromRGB(146,110,74)) }),
        }
        local RING_SIZE = NumberSequence.new({ NSK(0, 0.56, 0.14), NSK(1, 0.46, 0.1) })
        local RING_TR   = NumberSequence.new({ NSK(0, 1), NSK(0.07, 0.05), NSK(0.78, 0.4), NSK(1, 1) })
        local RING_SQ   = NumberSequence.new({ NSK(0, -0.6), NSK(0.25, 0.16), NSK(0.5, -0.6),
                                               NSK(0.75, 0.16), NSK(1, -0.6) })
        local SKIRT_SIZE = NumberSequence.new({ NSK(0, 1.5), NSK(1, 4.4) })
        local SKIRT_TR   = NumberSequence.new({ NSK(0, 1), NSK(0.22, 0.82), NSK(0.75, 0.92), NSK(1, 1) })
        local LEAF_SIZE  = NumberSequence.new({ NSK(0, 0.52, 0.12), NSK(1, 0.44, 0.1) })
        local LEAF_TR    = NumberSequence.new({ NSK(0, 1), NSK(0.1, 0.08), NSK(0.75, 0.42), NSK(1, 1) })
        local DUCK_TI = TweenInfo.new(1.2, Enum.EasingStyle.Sine, Enum.EasingDirection.Out)
        local BACK_TI = TweenInfo.new(2.6, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut)
        local _dev = { folder = nil, conn = nil }
        local _devRp = RaycastParams.new()
        _devRp.FilterType = Enum.RaycastFilterType.Exclude

        -- ambient duck/restore: untagged layers only (the carpet and the event-driven layers keep
        -- their own rates). Same rateTween precedence as the gust surge.
        local function ambientRate(mul, ti)
            local I = iAmt()
            for _, L in _partLayers do
                if L.emitter and L.tag == nil then
                    if L.rateTween then L.rateTween:Cancel() end
                    L.rateMul = mul
                    local tw = TweenService:Create(L.emitter, ti, { Rate = layerRate(L, I) * mul })
                    L.rateTween = tw
                    tw:Play()
                end
            end
        end

        stopLeafDevil = function()
            if _dev.conn then _dev.conn:Disconnect(); _dev.conn = nil end
            if _dev.folder then pcall(function() _dev.folder:Destroy() end); _dev.folder = nil end
            if _dev.rings then   -- flag: an ambient duck is still in effect
                _dev.rings = nil
                ambientRate(1, BACK_TI)
            end
        end

        -- one orbiting ring emitter. EmissionDirection = Front so leaves inherit the orbit
        -- tangent the update loop aims the part along.
        local function mkRing(folder, cf, i, r)
            local p = mkBoltPart(WHITE, 1, Vec(0.3, 0.3, 0.3), cf, folder)
            local k = ((i - 1) % 5) + 1
            local glow = RING_GLOW[k]
            local e = Instance.new("ParticleEmitter")
            e:SetAttribute("WX_Custom", true)
            e.Texture = RING_TEX[k]
            e.Color = RING_COL[k]
            e.LightEmission = glow; e.LightInfluence = 1 - glow
            e.Size = RING_SIZE; e.Transparency = RING_TR; e.Squash = RING_SQ
            e.Rate = 0
            e.Lifetime = NumberRange.new(2, 3)
            e.Speed = NumberRange.new(5, 9)
            e.SpreadAngle = Vector2.new(8, 8)
            e.Rotation = NumberRange.new(-180, 180)
            e.RotSpeed = NumberRange.new(-300, 300)
            e.Acceleration = Vec(0, 2.6, 0)
            e.Drag = 1.4
            e.EmissionDirection = ND.Front
            e.Parent = p
            return { part = p, e = e, r = r }
        end

        registerSpectacle("LeafDevil", 1, function()
            if not Config.Weather or _curType ~= "Autumn" or _dev.conn then return end
            local cam = Camera; if not cam then return end
            local camPos = cam.CFrame.Position
            local bearing = math.random() * 6.283
            local dist = 18 + math.random() * 12
            local cx = camPos.X + math.cos(bearing) * dist
            local cz = camPos.Z + math.sin(bearing) * dist
            local ch = lp and lp.Character
            if ch then
                _devRp.FilterDescendantsInstances = { getFolder(), ch }
            else
                _devRp.FilterDescendantsInstances = { getFolder() }
            end
            local hit = Workspace:Raycast(Vec(cx, camPos.Y + 6, cz), DEV_DOWN, _devRp)
            local gy = camPos.Y - 4.6
            if hit then gy = hit.Position.Y end

            local folder = Instance.new("Folder")
            folder.Name = "_wxDevil"; folder:SetAttribute("WX_Custom", true)
            folder.Parent = getFolder()
            _dev.folder = folder

            -- the column's own density rides the slider like the ambient fall does
            local em = iRate(iAmt(), IR.Autumn.lo, IR.Autumn.hi)
            local seed = CFrame.new(cx, gy + 1, cz)
            local rings = table.create(RING_N)
            for i = 1, RING_N do
                local h = 0.8 + (i - 1) * (DEV_H - 0.8) / (RING_N - 1)
                local u = h / DEV_H
                local r = DEV_R * (0.15 + 0.85 * u ^ 0.9)
                if u > 0.72 then r = r * (1 + (u - 0.72) * 1.5) end
                local R = mkRing(folder, seed, i, r)
                R.h = h
                R.ph = (i - 1) * 0.62          -- helix twist up the stack
                R.t0 = 0.12 * (i - 1)          -- bottom-up spin-up
                R.rate = (10 + 34 * (r / DEV_R)) * em   -- rate follows ring circumference
                rings[i] = R
            end
            _dev.rings = rings

            local foot = mkBoltPart(WHITE, 1, Vec(6.5, 0.4, 6.5), CFrame.new(cx, gy + 0.35, cz), folder)
            local leafSkirt = Instance.new("ParticleEmitter")
            leafSkirt:SetAttribute("WX_Custom", true)
            leafSkirt.Texture = TX_LEAFA
            leafSkirt.Color = RING_COL[1]
            leafSkirt.LightEmission = 0.3; leafSkirt.LightInfluence = 0.7
            leafSkirt.Size = LEAF_SIZE; leafSkirt.Transparency = LEAF_TR; leafSkirt.Squash = RING_SQ
            leafSkirt.Rate = 0
            leafSkirt.Lifetime = NumberRange.new(1.2, 2.2)
            leafSkirt.Speed = NumberRange.new(2, 5)
            leafSkirt.SpreadAngle = Vector2.new(70, 70)
            leafSkirt.Rotation = NumberRange.new(-180, 180)
            leafSkirt.RotSpeed = NumberRange.new(-260, 260)
            leafSkirt.Acceleration = Vec(0, 1.2, 0)
            leafSkirt.Drag = 2.2
            leafSkirt.EmissionDirection = ND.Top
            leafSkirt.Parent = foot
            local dustSkirt = Instance.new("ParticleEmitter")
            dustSkirt:SetAttribute("WX_Custom", true)
            dustSkirt.Texture = TX_DUST
            dustSkirt.Color = ColorSequence.new(Color3.fromRGB(190,166,130), Color3.fromRGB(140,118,90))
            dustSkirt.LightEmission = 0.04; dustSkirt.LightInfluence = 0.96
            dustSkirt.Size = SKIRT_SIZE; dustSkirt.Transparency = SKIRT_TR
            dustSkirt.Rate = 0
            dustSkirt.Lifetime = NumberRange.new(0.9, 1.4)
            dustSkirt.Speed = NumberRange.new(1, 2.6)
            dustSkirt.SpreadAngle = Vector2.new(85, 85)
            dustSkirt.RotSpeed = NumberRange.new(-25, 25)
            dustSkirt.Acceleration = Vec(0, 1, 0)
            dustSkirt.Drag = 2.4
            dustSkirt.EmissionDirection = ND.Top
            dustSkirt.Parent = foot
            local scuff = mkBoltPart(WHITE, 1, Vec(15, 0.08, 15), CFrame.new(cx, gy + 0.05, cz), folder)
            scuff.Material = Enum.Material.SmoothPlastic   -- Neon would bloom the scuff decal
            local sd = Instance.new("Decal")
            sd:SetAttribute("WX_Custom", true)
            sd.Face = ND.Top; sd.Texture = TX_DUST
            sd.Color3 = Color3.fromRGB(96, 74, 52); sd.Transparency = 0.55
            sd.Parent = scuff

            ambientRate(0.45, DUCK_TI)

            local t0 = tick()
            local ang, px, pz = 0, cx, cz
            local lean = math.random() * 6.283
            _dev.conn = RunService.Heartbeat:Connect(function(dt)
                if not folder.Parent then stopLeafDevil() return end
                local t = tick() - t0
                if t > DEV_LIFE then stopLeafDevil() return end
                ang = ang + dt * 2.7               -- slow enough that the orbit sweep is READ, not blurred
                px = px + Wind.x * dt * 0.75       -- the devil wanders downwind
                pz = pz + Wind.z * dt * 0.75
                -- a slow lean on a wander: a perfectly vertical column reads as fire
                local lx = math.cos(lean + t * 0.21) * 0.13
                local lz = math.sin(lean + t * 0.17) * 0.13
                local climb = math.min(t / 1.5, 1)
                local release = t > DEV_HOLD
                for i = 1, RING_N do
                    local R = rings[i]
                    local h = R.h * climb
                    local th = ang + R.ph
                    local cs, sn = math.cos(th), math.sin(th)
                    local r = R.r * (0.55 + 0.45 * climb)
                    local p = Vec(px + lx * h + cs * r, gy + 0.35 + h, pz + lz * h + sn * r)
                    R.part.CFrame = CFrame.lookAt(p, p + Vec(-sn, 0.3, cs))
                    if release then
                        R.e.Rate = 0
                        R.e.Acceleration = RELEASE
                    else
                        R.e.Rate = R.rate * math.clamp((t - R.t0) / 0.6, 0, 1)
                    end
                end
                foot.CFrame = CFrame.new(px, gy + 0.35, pz)
                scuff.CFrame = CFrame.new(px, gy + 0.05, pz) * CFrame.Angles(0, t * 0.35, 0)
                if release then
                    leafSkirt.Rate = 0
                    dustSkirt.Rate = 0
                    leafSkirt.Acceleration = RELEASE
                    sd.Transparency = math.min(0.55 + (t - DEV_HOLD) * 0.35, 1)
                    if t > DEV_HOLD + 0.1 and _dev.rings then
                        _dev.rings = nil
                        ambientRate(1, BACK_TI)
                    end
                else
                    local k = math.min(t / 0.8, 1)
                    leafSkirt.Rate = 26 * em * k
                    dustSkirt.Rate = 15 * em * k
                    sd.Transparency = 0.95 - 0.4 * math.min(t / 1.2, 1)
                end
            end)
        end)
    end)()

    -- ════════════════════════════════════════════════════════════════════════
    -- INTERACTIVE PUDDLES (WeatherPuddles) — standing water that reacts to the
    -- players walking through it. Dark slate tint at LOW Reflectance reads as wet
    -- ground — Roblox Reflectance mirrors only the skybox, so pushing it any
    -- higher just makes an ice rink.
    -- THE FIELD IS A PARTITION, NOT A SCATTER. Water lives on a fixed world grid of
    -- CELL-sized cells and EVERY water surface is contained inside the one cell (or
    -- sub-cell) that owns it. Cells are disjoint, so two water pieces can never
    -- intersect — that is a geometric guarantee, not a spacing heuristic. It matters
    -- because the slabs are translucent: two that OVERLAP composite to roughly double
    -- opacity and read as a dark blotch, while two that merely ABUT read as one sheet.
    -- A smooth noise field over world XZ decides each cell: above the intensity
    -- threshold = water, and the threshold sweeps from "a few isolated cells" at 0.15
    -- to BELOW the noise minimum at 2.0, where every cell floods and the floor is
    -- solid water. A cell whose four neighbours are all water is INTERIOR: it tiles
    -- its area with a 2×2 of rectangular slabs that abut edge-to-edge at one shared
    -- Y — no stacking, no seam. Any other water cell is the SHORELINE and gets the
    -- organic pool (one elongated base slab plus 4-5 rotated lobes on a random spine,
    -- two sheen patches), so the flood never ends on a straight line. Intensity moves
    -- how MUCH of the floor floods, never how much water stacks. Slider moves
    -- re-decide cells ONCE, debounced (`puddlesIntensity`), never per tick; the grid
    -- itself never moves, so a drag grows and shrinks the flood instead of rebuilding.
    -- OVERHANG RULE (general, no map tuning): no part of a puddle may extend past
    -- the surface it rests on. Every slab is fitted independently against the ground
    -- by its own RIM (fitSlab for lobes, fitRect for tiles) and is shrunk, slid or
    -- dropped on its own, so the outline hugs a ledge instead of the whole pool being
    -- accepted or thrown away. Tiles may only SHRINK about their sub-cell centre —
    -- never slide — which is what keeps containment true after a trim.
    -- Ambient rain rings per puddle: VelocityPerpendicular + EmissionDirection
    -- Top + a hair of Speed makes the ring texture lie FLAT and expand. The emitter
    -- host is clipped to the surviving footprint so rings never spray over the drop.
    -- INTERACTION is triggered VFX, no fluid sim: a 20Hz loop reads nearby
    -- characters through the swappable `actorsProvider` seam (shipped = real
    -- players, a Studio rig swaps in a dummy and exercises the SAME detection
    -- and splash code), detects a grounded footfall inside a puddle's ellipse
    -- union under a per-character stride throttle, and fires a POOLED splash at
    -- the contact point — flat expanding ring + white foam streaks + droplets
    -- arcing up and falling back under gravity, all scaled by move speed; a hard
    -- landing crowns at full scale; leaving the water drops fading wet
    -- footprints. Nothing is created per footstep. Active while type == Rain or
    -- Storm is on; type switch fades out, disable/unload tears down hard.
    -- Nested IIFE keeps ~45 locals off the weather proto budget.
    -- ════════════════════════════════════════════════════════════════════════
    local puddlesRefresh, puddlesIntensity, stopPuddles
    ;(function()
        local TXP_RING = "rbxassetid://17738857765"   -- thin circle outline (TX kit ripple)
        local TXP_DROP = "rbxassetid://14964503448"   -- glassy teardrop (TX kit droplet)
        local TXP_STAR = "rbxassetid://17726943419"   -- 4-point star (rain-hit glints)
        local WATER_C  = Color3.fromRGB(22, 28, 37)
        local SHEEN_C  = Color3.fromRGB(150, 175, 205)
        local FOAM_C   = Color3.fromRGB(236, 246, 255)
        local PUD_DOWN = Vec(0, -90, 0)
        local FIT_DOWN = Vec(0, -4.2, 0)              -- rim probe: just outreaches FIT_TOL
        local HALF_PI  = 1.5707963
        local FIT_TOL  = 1.5                          -- off the water plane by this = a drop
        local FIT_R    = 1.09                         -- rim probes sit 9% PAST the water: the
        -- 8-sample octagon's incircle is then 1.09*cos(22.5°) = 1.007 of the rim, so a
        -- straight ledge that cuts the pool anywhere is always caught between samples
        -- shrink ladder; the last rung is what lets a narrow walkway or a crate top
        -- carry a thin ribbon of water instead of staying bone dry
        local FIT_K    = { 1, 0.66, 0.44, 0.28 }
        local FIT_C = {                               -- 8 rim directions, cos/sin interleaved
            1, 0, 0.70711, 0.70711, 0, 1, -0.70711, 0.70711,
            -1, 0, -0.70711, -0.70711, 0, -1, 0.70711, -0.70711,
        }
        local FIT_Q = {                               -- 12 rect-rim points: 4 corners + 2 per edge
            1, 1, 0.333, 1, -0.333, 1, -1, 1,
            -1, 0.333, -1, -0.333, -1, -1, -0.333, -1,
            0.333, -1, 1, -1, 1, -0.333, 1, 0.333,
        }
        local CELL     = 32                           -- flood-grid pitch; also the pool spacing
        local HALF     = 16                           -- CELL/2: containment limit AND sub-cell side
        local NOISE_F  = 0.018                        -- ~55-stud flood regions (2-3 cells across)
        local POOL_F   = 0.74                         -- shoreline pool span as a fraction of CELL
        local TILE_K   = 1 / 470                      -- tile ring rate per stud², matched to pools
        local K_CAP    = 48                           -- aggregate ring/glint budget over the field
        local MAX_REC  = 96                           -- hard ceiling on live water pieces
        local _pud = { folder = nil, conn = nil }
        local _plist = {}                             -- puddle records (see mkPuddle/mkFloodCell)
        local _pmap = {}                              -- cell key → record, so a cell is placed once
        local _pfail = {}                             -- cell key → true: no ground here, stop retrying
        -- one reused cell descriptor: keeps the constructors at 4 args and allocates nothing
        local _cd = { i = 0, j = 0, key = 0, kind = 0, cx = 0, cz = 0, gy = 0 }
        local _units, _uIdx = {}, 1                   -- splash pool
        local _prints, _pIdx = {}, 1                  -- wet-footprint pool
        local _pstates = {}                           -- per-actor stride/wet/fall state
        local _abuf = {}                              -- pooled actor records, zero alloc per tick
        local _prp = RaycastParams.new()
        _prp.FilterType = Enum.RaycastFilterType.Exclude
        local _pfilter = {}                           -- reused raycast filter list
        local _fieldToken, _rebuild = 0, false
        local _int, _thr, _placeR, _seed = 1, 0, 50, 0   -- field params, frozen between rebuilds
        local _wTr, _wRf = 0.35, 0.16                 -- water look, deepens with intensity
        local _homeI, _homeJ = 1e9, 1e9               -- player's cell, drives the _pfail flush

        local function pudActive()
            if not Config.Weather or not cfg("WeatherPuddles", false) then
                return false
            end
            if _curType == "Rain" then
                return true
            end
            return cfg("WeatherStorm", false)
        end
        local function pudIntensity()
            return iAmt()
        end

        local function mkFlatPart(size, cf, parent)
            local p = Instance.new("Part")
            p.Anchored = true; p.CanCollide = false; p.CanQuery = false; p.CanTouch = false
            p.CastShadow = false; p.Transparency = 1
            p.Size = size; p.CFrame = cf
            p:SetAttribute("WX_Custom", true)
            p.Parent = parent
            return p
        end
        -- one flat elliptical water lobe; sheen slabs are the surface-variation glints
        local function mkSlab(d1, d2, cf, parent, sheen)
            local p = Instance.new("Part")
            p.Shape = Enum.PartType.Cylinder
            p.Anchored = true; p.CanCollide = false; p.CanQuery = false; p.CanTouch = false
            p.CastShadow = false; p.Material = Enum.Material.SmoothPlastic
            if sheen then
                p.Color = SHEEN_C; p.Transparency = 0.74; p.Reflectance = 0.38
            else
                p.Color = WATER_C; p.Transparency = _wTr; p.Reflectance = _wRf
                p:SetAttribute("WX_W", true)   -- water proper: retinted on an intensity change
            end
            p.Size = Vec(0.05, d1, d2)
            p.CFrame = cf
            p:SetAttribute("WX_Custom", true)
            p.Parent = parent
        end
        -- flooded-interior tile: the SAME water look as a lobe on a square footprint,
        -- because only squares tile a plane edge-to-edge without stacking
        local function mkTile(w, l, cf, parent)
            local p = Instance.new("Part")
            p.Anchored = true; p.CanCollide = false; p.CanQuery = false; p.CanTouch = false
            p.CastShadow = false; p.Material = Enum.Material.SmoothPlastic
            p.Color = WATER_C; p.Transparency = _wTr; p.Reflectance = _wRf
            p.Size = Vec(w, 0.05, l)
            p.CFrame = cf
            p:SetAttribute("WX_Custom", true)
            p:SetAttribute("WX_W", true)
            p.Parent = parent
        end

        -- FOOTPRINT FIT — the overhang rule. A slab may only exist where the ground
        -- actually holds it, so sample the slab's RIM: 8 points ON the ellipse
        -- boundary, which is exactly what hangs off a ledge (interior probes can never
        -- see it). A sample is supported only over near-vertical collidable ground
        -- within FIT_TOL of the water plane — a hit far below is a drop-off, no hit at
        -- all is a void or the inside of a prop. If any rim sample fails, shrink and
        -- slide toward the centroid of the supported ones and retest; if the ladder
        -- runs out the caller drops that slab and the pool simply ends at the ledge.
        -- Bounded at 32 short rays per slab, placement-time only.
        local function fitSlab(cx, cz, d1, d2, yaw, gy)
            local cs, sn = math.cos(yaw), math.sin(yaw)
            for k = 1, 4 do
                local f = FIT_K[k] * FIT_R * 0.5
                local a, b = d1 * f, d2 * f
                local ok, gx, gz, n = true, 0, 0, 0
                for i = 1, 8 do
                    local ct, st = FIT_C[i * 2 - 1], FIT_C[i * 2]
                    local ox = sn * b * st - cs * a * ct
                    local oz = sn * a * ct + cs * b * st
                    local h = Workspace:Raycast(Vec(cx + ox, gy + 2, cz + oz), FIT_DOWN, _prp)
                    local sup = false
                    if h and h.Normal.Y >= 0.86 and h.Instance.CanCollide then
                        local dy = h.Position.Y - gy
                        sup = dy < FIT_TOL and dy > -FIT_TOL
                    end
                    if sup then
                        gx, gz, n = gx + ox, gz + oz, n + 1
                    else
                        ok = false
                    end
                end
                if ok then
                    return cx, cz, d1 * FIT_K[k], d2 * FIT_K[k]
                end
                if n == 0 then
                    return nil
                end
                cx, cz = cx + gx / n * 0.5, cz + gz / n * 0.5
            end
            return nil
        end

        -- TILE FIT — the same overhang rule for a square footprint, sampled at 12 rim
        -- points (corners plus thirds, so an 8-stud-wide ledge cannot slip between two
        -- samples of a 16-stud tile). A tile may only SHRINK about its own centre: any
        -- slide would let it cross into the neighbouring sub-cell and stack, and the
        -- no-overlap guarantee is worth more than the last few studs of a trimmed edge.
        local function fitRect(cx, cz, w, l, gy)
            for k = 1, 4 do
                local f = FIT_K[k] * FIT_R * 0.5
                local a, b = w * f, l * f
                local ok = true
                for i = 1, 12 do
                    local h = Workspace:Raycast(
                        Vec(cx + FIT_Q[i * 2 - 1] * a, gy + 2, cz + FIT_Q[i * 2] * b), FIT_DOWN, _prp)
                    local sup = false
                    if h and h.Normal.Y >= 0.86 and h.Instance.CanCollide then
                        local dy = h.Position.Y - gy
                        sup = dy < FIT_TOL and dy > -FIT_TOL
                    end
                    if not sup then
                        ok = false
                        break
                    end
                end
                if ok then
                    return w * FIT_K[k], l * FIT_K[k]
                end
            end
            return nil
        end

        -- ambient rain rings + glints for one water piece. The host is clipped to the
        -- footprint that SURVIVED the fit, so nothing sprays over a trimmed edge, and
        -- the rate rides `k` (area-scaled) so density per stud² is constant everywhere.
        local function mkHost(m, cf, size, k, I)
            local host = mkFlatPart(size, cf, m)
            local e = Instance.new("ParticleEmitter")
            e:SetAttribute("WX_Custom", true)
            e.Texture = TXP_RING
            e.Orientation = Enum.ParticleOrientation.VelocityPerpendicular
            e.EmissionDirection = Enum.NormalId.Top
            e.Speed = NumberRange.new(0.05, 0.05)
            e.Rate = (2.1 + 4.2 * I) * k
            e.Lifetime = NumberRange.new(1.1, 1.6)
            e.Size = NumberSequence.new({ NSK(0, 0.12), NSK(1, 4.4) })
            e.Transparency = NumberSequence.new({ NSK(0, 0.9), NSK(0.14, 0.12), NSK(0.62, 0.5), NSK(1, 1) })
            e.Color = ColorSequence.new(Color3.fromRGB(215, 234, 250))
            e.LightEmission = 0.45
            e.Parent = host
            local g = Instance.new("ParticleEmitter")
            g:SetAttribute("WX_Custom", true)
            g.Texture = TXP_STAR
            g.EmissionDirection = Enum.NormalId.Top
            g.Speed = NumberRange.new(0, 0)
            g.Rate = 0.8 * I * k
            g.Lifetime = NumberRange.new(0.16, 0.28)
            g.Size = NumberSequence.new({ NSK(0, 0.06), NSK(0.4, 0.3), NSK(1, 0.04) })
            g.Transparency = NumberSequence.new({ NSK(0, 0.6), NSK(1, 1) })
            g.Color = ColorSequence.new(Color3.fromRGB(240, 248, 255))
            g.LightEmission = 0.5
            g.Parent = host
            return e, g
        end

        -- SHORELINE POOL — one cell's worth of water where the flood ends: the
        -- elongated base slab plus lobes and sheen patches, i.e. exactly the pool a
        -- lone puddle has always been. The only new rule is CONTAINMENT: the pool is
        -- anchored in its cell and any piece that would reach past the cell edge is
        -- dropped rather than trimmed inwards, so it can never touch a neighbour's water.
        local function mkPuddle(cd, span, nslab, I)
            local x, z, gy = cd.cx, cd.cz, cd.gy
            local m = Instance.new("Model")
            m.Name = "_pud"
            -- anchor jittered off the cell centre so a sparse field never reads as a lattice
            local px = x + (math.random() - 0.5) * CELL * 0.16
            local pz = z + (math.random() - 0.5) * CELL * 0.16
            -- lobes strung along one spine → an elongated organic pool, not a rosette
            local spine = math.random() * 6.28318
            local sx, sz = math.cos(spine), math.sin(spine)
            local lx, lz = -sz, sx
            local yaw0 = math.pi - spine   -- puts the slab's d1 axis ON the spine
            local slabs, rmax = {}, 0
            -- ring-host extent, in the spine frame, of the slabs that SURVIVED the fit
            local s0, s1, l0, l1 = 1e9, -1e9, 1e9, -1e9
            local function place(cx, cz, d1, d2, yaw, yoff, sheen)
                local fx, fz, fd1, fd2 = fitSlab(cx, cz, d1, d2, yaw, gy)
                if not fx then
                    return false
                end
                -- bounding circle vs the cell square: holds for any yaw and after any
                -- fit slide, and it is measured from the CELL centre, not the anchor
                local rad = math.max(fd1, fd2) * 0.5
                local ex, ez = fx - x, fz - z
                if math.abs(ex) + rad > HALF or math.abs(ez) + rad > HALF then
                    return false
                end
                mkSlab(fd1, fd2, CFrame.new(fx, gy + yoff, fz)
                    * CFrame.Angles(0, yaw, 0) * CFrame.Angles(0, 0, HALF_PI), m, sheen)
                if not sheen then   -- sheen glints are decoration, not part of the water union
                    slabs[#slabs + 1] = {
                        cx = fx, cz = fz,
                        cs = math.cos(yaw), sn = math.sin(yaw),
                        ia = 2 / fd1, ib = 2 / fd2, rect = false,
                    }
                    rmax = math.max(rmax, math.sqrt(ex * ex + ez * ez) + rad)
                end
                local dx, dz = fx - px, fz - pz
                local t, lat = dx * sx + dz * sz, dx * lx + dz * lz
                local ins = math.min(fd1, fd2) * 0.375   -- keep rings INSIDE the water
                s0, s1 = math.min(s0, t - ins), math.max(s1, t + ins)
                l0, l1 = math.min(l0, lat - ins), math.max(l1, lat + ins)
                return true
            end
            -- base first: a spot that cannot hold even a shrunk base is a bad spot, and
            -- bailing here keeps a rejected candidate at ≤32 rays instead of ~250
            if not place(px, pz, span, span * 0.62, yaw0, 0.002, false) then
                m:Destroy()
                return false
            end
            for i = 1, nslab do
                local t = ((i - 0.5) / nslab - 0.5) * span * 0.62 + (math.random() - 0.5) * span * 0.14
                local lat = (math.random() - 0.5) * span * 0.24
                local d1 = span * (0.34 + math.random() * 0.3)
                -- stacked micro-offsets: never coplanar with the floor or each other
                place(px + sx * t + lx * lat, pz + sz * t + lz * lat,
                    d1, d1 * (0.6 + math.random() * 0.34),
                    yaw0 + (math.random() - 0.5) * 1.6, 0.008 + i * 0.004, false)
            end
            for _ = 1, 2 do
                local t = (math.random() - 0.5) * span * 0.5
                local lat = (math.random() - 0.5) * span * 0.16
                local d1 = span * (0.14 + math.random() * 0.18)
                place(px + sx * t + lx * lat, pz + sz * t + lz * lat,
                    d1, d1 * (0.4 + math.random() * 0.35), math.random() * math.pi, 0.036, true)
            end
            -- host = the approved rect CLIPPED to what survived, so flat ground is
            -- pixel-identical and a trimmed pool never sprays rings over the drop
            s0, s1 = math.max(s0, span * -0.425), math.min(s1, span * 0.425)
            l0, l1 = math.max(l0, span * -0.225), math.min(l1, span * 0.225)
            local hw, hl = math.max(s1 - s0, 1), math.max(l1 - l0, 1)
            local hm, hn = (s0 + s1) * 0.5, (l0 + l1) * 0.5
            -- ring/glint rate follows the surviving AREA (ratio 1 on open ground)
            local k = (span / 26) ^ 1.5 * hw * hl / (span * span * 0.3825)
            local e, g = mkHost(m, CFrame.new(px + sx * hm + lx * hn, gy + 0.1, pz + sz * hm + lz * hn)
                * CFrame.Angles(0, -spine, 0), Vec(hw, 0.05, hl), k, I)
            m.Parent = _pud.folder
            local P = {
                cx = x, cz = z, top = gy + 0.06, r2 = rmax * rmax, k = k,
                key = cd.key, ci = cd.i, cj = cd.j, kind = cd.kind,
                slabs = slabs, ring = e, glint = g, model = m,
            }
            _plist[#_plist + 1] = P
            _pmap[cd.key] = P
            return true
        end

        -- FLOODED INTERIOR — a cell with water on all four sides, so nothing here needs
        -- an organic outline: it just has to be WATER, edge to edge, with no stacking.
        -- The cell is split 2×2 and each sub-cell gets one tile fitted inside itself.
        -- Every tile shares one Y, one colour and one transparency, so abutting tiles
        -- are literally indistinguishable from a single sheet — and because each is
        -- confined to its own sub-cell they abut and never intersect.
        local function mkFloodCell(cd, I)
            local gy = cd.gy
            local m = Instance.new("Model")
            m.Name = "_pud"
            local slabs = {}
            local area, rmax = 0, 0
            local x0, x1, z0, z1 = 1e9, -1e9, 1e9, -1e9
            for a = 0, 1 do
                for b = 0, 1 do
                    local scx = cd.cx + (a - 0.5) * HALF
                    local scz = cd.cz + (b - 0.5) * HALF
                    local fw, fl = fitRect(scx, scz, HALF, HALF, gy)
                    if fw then
                        mkTile(fw, fl, CFrame.new(scx, gy + 0.002, scz), m)
                        slabs[#slabs + 1] = {
                            cx = scx, cz = scz, cs = 1, sn = 0,
                            ia = 2 / fw, ib = 2 / fl, rect = true,
                        }
                        area = area + fw * fl
                        rmax = math.max(rmax, 0.7072 * (HALF + fw))
                        x0 = math.min(x0, scx - fw * 0.5)
                        x1 = math.max(x1, scx + fw * 0.5)
                        z0 = math.min(z0, scz - fl * 0.5)
                        z1 = math.max(z1, scz + fl * 0.5)
                    end
                end
            end
            if area <= 0 then
                m:Destroy()
                return false
            end
            -- one sheen patch on some UNTRIMMED cells: it is BRIGHT over dark water, so
            -- the pair composites lighter and reads as a glint, never as the dark blotch
            -- two stacked water slabs would make. Skipped on a trimmed cell, where it
            -- could land on the dry part. Fitted and contained like everything else.
            if area > CELL * CELL * 0.98 and math.random() < 0.45 then
                local d1 = CELL * (0.15 + math.random() * 0.1)
                local sx = cd.cx + (math.random() - 0.5) * CELL * 0.4
                local sz = cd.cz + (math.random() - 0.5) * CELL * 0.4
                local yaw = math.random() * math.pi
                local fx, fz, fd1, fd2 = fitSlab(sx, sz, d1, d1 * (0.4 + math.random() * 0.35), yaw, gy)
                if fx and math.abs(fx - cd.cx) + fd1 * 0.5 < HALF
                    and math.abs(fz - cd.cz) + fd1 * 0.5 < HALF then
                    mkSlab(fd1, fd2, CFrame.new(fx, gy + 0.036, fz)
                        * CFrame.Angles(0, yaw, 0) * CFrame.Angles(0, 0, HALF_PI), m, true)
                end
            end
            local hw, hl = math.max(x1 - x0, 1), math.max(z1 - z0, 1)
            local e, g = mkHost(m, CFrame.new((x0 + x1) * 0.5, gy + 0.1, (z0 + z1) * 0.5),
                Vec(hw, 0.05, hl), area * TILE_K, I)
            m.Parent = _pud.folder
            local P = {
                cx = cd.cx, cz = cd.cz, top = gy + 0.06, r2 = rmax * rmax, k = area * TILE_K,
                key = cd.key, ci = cd.i, cj = cd.j, kind = cd.kind,
                slabs = slabs, ring = e, glint = g, model = m,
            }
            _plist[#_plist + 1] = P
            _pmap[cd.key] = P
            return true
        end

        local function mkSplashUnit(parent)
            local p = mkFlatPart(Vec(1.1, 0.1, 1.1), CFrame.new(0, -900, 0), parent)
            local ring = Instance.new("ParticleEmitter")
            ring:SetAttribute("WX_Custom", true)
            ring.Texture = TXP_RING
            ring.Orientation = Enum.ParticleOrientation.VelocityPerpendicular
            ring.EmissionDirection = Enum.NormalId.Top
            ring.Speed = NumberRange.new(0.05, 0.05)
            ring.Rate = 0
            ring.Lifetime = NumberRange.new(0.45, 0.7)
            ring.Size = NumberSequence.new({ NSK(0, 0.1), NSK(1, 4) })
            ring.Transparency = NumberSequence.new({ NSK(0, 0.75), NSK(0.1, 0.05), NSK(0.5, 0.4), NSK(1, 1) })
            ring.Color = ColorSequence.new(Color3.fromRGB(230, 244, 255))
            ring.LightEmission = 0.5
            ring.Parent = p
            -- foam = the white water torn off the surface; the streak read comes from
            -- VelocityParallel + Squash, and the arc from the downward Acceleration
            local foam = Instance.new("ParticleEmitter")
            foam:SetAttribute("WX_Custom", true)
            foam.Texture = TX_SNOWSOFT
            foam.Orientation = Enum.ParticleOrientation.VelocityParallel
            foam.EmissionDirection = Enum.NormalId.Top
            foam.Rate = 0
            foam.SpreadAngle = Vector2.new(56, 56)
            foam.Acceleration = Vec(0, -80, 0)
            foam.Lifetime = NumberRange.new(0.3, 0.6)
            foam.Size = NumberSequence.new({ NSK(0, 0.6, 0.22), NSK(1, 0.2) })
            foam.Squash = NumberSequence.new({ NSK(0, 1.5), NSK(1, 0.4) })
            foam.Speed = NumberRange.new(6, 12)
            foam.Color = ColorSequence.new(FOAM_C)
            foam.Transparency = NumberSequence.new({ NSK(0, 0.18), NSK(0.7, 0.45), NSK(1, 1) })
            foam.LightEmission = 0.18
            foam.Parent = p
            local drops = Instance.new("ParticleEmitter")
            drops:SetAttribute("WX_Custom", true)
            drops.Texture = TXP_DROP
            drops.Orientation = Enum.ParticleOrientation.VelocityParallel
            drops.EmissionDirection = Enum.NormalId.Top
            drops.Rate = 0
            drops.SpreadAngle = Vector2.new(52, 52)
            drops.Acceleration = Vec(0, -80, 0)
            drops.Lifetime = NumberRange.new(0.32, 0.62)
            drops.Size = NumberSequence.new({ NSK(0, 0.55, 0.18), NSK(1, 0.28) })
            drops.Speed = NumberRange.new(6, 12)
            drops.Color = ColorSequence.new(Color3.fromRGB(210, 232, 250))
            drops.Transparency = NumberSequence.new({ NSK(0, 0.1), NSK(0.75, 0.3), NSK(1, 1) })
            drops.LightEmission = 0.25
            drops.Parent = p
            local mist = Instance.new("ParticleEmitter")
            mist:SetAttribute("WX_Custom", true)
            mist.Texture = TX_SNOWSOFT
            mist.EmissionDirection = Enum.NormalId.Top
            mist.Rate = 0
            mist.Speed = NumberRange.new(1.2, 2.8)
            mist.SpreadAngle = Vector2.new(48, 48)
            mist.Lifetime = NumberRange.new(0.3, 0.5)
            mist.Size = NumberSequence.new({ NSK(0, 0.5), NSK(1, 1.6) })
            mist.Transparency = NumberSequence.new({ NSK(0, 0.75), NSK(1, 1) })
            mist.Color = ColorSequence.new(FOAM_C)
            mist.LightEmission = 0.1
            mist.Parent = p
            return { part = p, ring = ring, foam = foam, drops = drops, mist = mist }
        end

        local function fireSplash(x, y, z, speed)
            local u = _units[_uIdx]
            if not u then
                return
            end
            _uIdx = (_uIdx % #_units) + 1
            local s = math.clamp(speed, 4, 36)
            u.part.CFrame = CFrame.new(x, y + 0.12, z)
            u.ring.Size = NumberSequence.new({ NSK(0, 0.1), NSK(1, 3.2 + s * 0.18) })
            local lo, hi = 5 + s * 0.38, 8 + s * 0.6
            u.foam.Speed = NumberRange.new(lo, hi)
            u.drops.Speed = NumberRange.new(lo * 0.8, hi * 0.9)
            u.ring:Emit(2)
            u.foam:Emit(9 + math.floor(s * 0.75))
            u.drops:Emit(4 + math.floor(s * 0.3))
            u.mist:Emit(3)
        end
        local function dropPrint(x, y, z, heading, side)
            local p = _prints[_pIdx]
            if not p then
                return
            end
            _pIdx = (_pIdx % #_prints) + 1
            p.CFrame = CFrame.new(x, y + 0.02, z) * CFrame.Angles(0, heading, 0) * CFrame.new(side * 0.4, 0, 0)
            p.Transparency = 0.42
            TweenService:Create(p, TweenInfo.new(2.2), { Transparency = 1 }):Play()
        end

        -- ACTOR SOURCE SEAM: the shipped path reports every nearby character
        -- (local player AND opponents — seeing someone else kick up water is the
        -- whole point). A Studio harness swaps this local for a dummy provider
        -- and gets the identical detection + splash path. Records are pooled.
        local function defaultActors()
            local n = 0
            local cam = Camera
            if not cam then
                return _abuf, 0
            end
            local camPos = cam.CFrame.Position
            for _, plr in Players:GetPlayers() do
                local ch = plr.Character
                local hrp = ch and ch:FindFirstChild("HumanoidRootPart")
                if hrp then
                    local pos = hrp.Position
                    local dx, dy, dz = pos.X - camPos.X, pos.Y - camPos.Y, pos.Z - camPos.Z
                    if dx * dx + dy * dy + dz * dz < 12100 then   -- hard 110-stud cull
                        local vel = hrp.AssemblyLinearVelocity
                        -- tolerant grounded: small vertical speed, else the humanoid
                        -- floor (remote characters often replicate FloorMaterial Air)
                        local grounded = vel.Y > -6 and vel.Y < 6
                        if not grounded then
                            local hum = ch:FindFirstChildOfClass("Humanoid")
                            grounded = hum ~= nil and hum.FloorMaterial ~= Enum.Material.Air
                        end
                        n = n + 1
                        local rec = _abuf[n]
                        if not rec then
                            rec = {}
                            _abuf[n] = rec
                        end
                        rec.key = plr
                        rec.fx, rec.fy, rec.fz = pos.X, pos.Y - 2.9, pos.Z
                        rec.vx, rec.vy, rec.vz = vel.X, vel.Y, vel.Z
                        rec.speed = math.sqrt(vel.X * vel.X + vel.Z * vel.Z)
                        rec.grounded = grounded
                    end
                end
            end
            return _abuf, n
        end
        local actorsProvider = defaultActors

        -- point vs the piece's footprint union: coarse radius reject, then per slab.
        -- Same normalised frame for both shapes — a lobe is |u,v| inside the unit
        -- circle, a tile inside the unit square — so splashes work on every surface.
        local function inPuddle(P, x, z)
            local dx, dz = x - P.cx, z - P.cz
            if dx * dx + dz * dz > P.r2 then
                return false
            end
            for _, sl in P.slabs do
                local ux, uz = x - sl.cx, z - sl.cz
                local u = (-ux * sl.cs + uz * sl.sn) * sl.ia
                local v = (ux * sl.sn + uz * sl.cs) * sl.ib
                if sl.rect then
                    if u > -1 and u < 1 and v > -1 and v < 1 then
                        return true
                    end
                elseif u * u + v * v <= 1 then
                    return true
                end
            end
            return false
        end

        -- THE FLOOD FIELD. One smooth world-space noise field decides which cells hold
        -- water; the threshold is the only thing intensity moves. It sweeps from well
        -- above the field's typical value (a handful of isolated cells) to below its
        -- theoretical MINIMUM at I = 2 — where the test cannot fail and every cell with
        -- ground under it floods. That is what makes "max = whole floor" a guarantee
        -- rather than a tuning hope.
        local function fieldAt(wx, wz)
            return math.noise(wx * NOISE_F, wz * NOISE_F, _seed)
        end
        -- knots calibrated against the MEASURED spread of math.noise (sd ≈ 0.27, observed
        -- range ±0.93 over 6e5 samples) — an assumed spread puts the whole curve out by
        -- half the field. 0.3 → scattered pools, 1.0 → ~45% of cells flooded interior,
        -- 1.6 → a flooded arena. -1.05 at I = 2 is past improved Perlin's absolute bound,
        -- so the water test cannot fail there and the whole floor floods.
        local function floodThr(I)
            if I <= 1 then
                return -0.4043 * (I - 0.3)
            end
            if I <= 1.6 then
                return -0.283 - 0.3417 * (I - 1)
            end
            -- NOT retuned. The top-end slope was re-solved against measured math.noise
            -- data; two independent Node re-implementations of Luau's noise disagree on
            -- the field's spread (sd 0.271 vs 0.365, min -0.99 vs -1.50), so any reslope
            -- here would be calibrated against a distribution nobody has actually
            -- confirmed. Coverage is already the best-behaved intensity consumer in the
            -- file. Leave it to a Studio session with a real sample.
            return -0.488 - 1.405 * (I - 1.6)
        end
        local function refreshParams()
            local I = pudIntensity()
            _int = I
            _thr = floodThr(I)
            _placeR = 46 + 20 * I
            -- heavier rain = deeper, darker, glossier standing water, not just more film.
            -- Reflectance stays under ~0.22: Roblox mirrors only the skybox, so higher
            -- turns the field into an ice rink.
            _wTr = 0.46 - 0.11 * I
            _wRf = 0.11 + 0.05 * I
        end
        -- 0 = dry, 1 = shoreline (organic pool), 2 = flooded interior (tiled).
        -- Interior means all four neighbours are water too, so a tiled cell is never
        -- the thing the player sees the flood END on — the outline is always pools.
        local function cellKind(i, j)
            local ccx, ccz = (i + 0.5) * CELL, (j + 0.5) * CELL
            if fieldAt(ccx, ccz) <= _thr then
                return 0
            end
            if fieldAt(ccx + CELL, ccz) > _thr and fieldAt(ccx - CELL, ccz) > _thr
                and fieldAt(ccx, ccz + CELL) > _thr and fieldAt(ccx, ccz - CELL) > _thr then
                return 2
            end
            return 1
        end

        local function placeCell(i, j, key, kind, by)
            local folder = _pud.folder
            if not folder then
                return false
            end
            local ccx, ccz = (i + 0.5) * CELL, (j + 0.5) * CELL
            _pfilter[1] = folder
            local ch = lp and lp.Character
            if ch then
                _pfilter[2] = ch
            else
                _pfilter[2] = folder
            end
            _prp.FilterDescendantsInstances = _pfilter
            local hit = Workspace:Raycast(Vec(ccx, by + 6, ccz), PUD_DOWN, _prp)
            if not hit or hit.Normal.Y < 0.94 then   -- flat solid ground only
                return false
            end
            local inst = hit.Instance
            if not inst.CanCollide then
                return false
            end
            if inst:IsA("Terrain") and hit.Material == Enum.Material.Water then
                return false
            end
            local cd = _cd
            cd.i, cd.j, cd.key, cd.kind = i, j, key, kind
            cd.cx, cd.cz, cd.gy = ccx, ccz, hit.Position.Y
            -- no whole-cell fit test: both constructors validate and trim PIECE BY
            -- PIECE, so a cell by a ledge keeps the supported part of its water
            if kind == 2 then
                return mkFloodCell(cd, _int)
            end
            local nslab = 5
            if _int > 1.15 then
                nslab = 4   -- flood: fewer, larger lobes keeps the overdraw sane
            end
            return mkPuddle(cd, CELL * POOL_F * (0.82 + 0.24 * math.random()), nslab, _int)
        end

        -- one grid cell: returns true when a placement was ATTEMPTED (budget spent).
        -- A cell with no ground under it is blacklisted so a void or a rooftop edge
        -- cannot burn the whole placement budget every tick.
        local function tryCell(i, j, bx, bz, by)
            local key = (i + 32768) * 65536 + j + 32768
            if _pmap[key] or _pfail[key] then
                return false
            end
            local dx = (i + 0.5) * CELL - bx
            local dz = (j + 0.5) * CELL - bz
            if dx * dx + dz * dz > _placeR * _placeR then
                return false
            end
            local kind = cellKind(i, j)
            if kind == 0 then
                return false
            end
            if not placeCell(i, j, key, kind, by) then
                _pfail[key] = true
            end
            return true
        end

        local function maintain(bx, by, bz)
            local ci, cj = math.floor(bx / CELL), math.floor(bz / CELL)
            local budget = 8
            if ci ~= _homeI or cj ~= _homeJ then
                _homeI, _homeJ = ci, cj
                table.clear(_pfail)   -- new neighbourhood: re-test what failed before
            end
            if _rebuild then
                _rebuild = false
                refreshParams()
                table.clear(_pfail)
                budget = 18
                -- retint the water that SURVIVES the re-decide, so a slider move deepens
                -- the whole field instead of only the pieces it happens to replace
                for _, P in _plist do
                    for _, d in P.model:GetChildren() do
                        if d:GetAttribute("WX_W") then
                            d.Transparency = _wTr
                            d.Reflectance = _wRf
                        end
                    end
                end
                -- the grid never moves, so a settled slider only RE-DECIDES cells: keep
                -- every piece the new field still wants and grow/shrink around them
                local n = 1
                while n <= #_plist do
                    local P = _plist[n]
                    if P.kind == cellKind(P.ci, P.cj) then
                        n = n + 1
                    else
                        _pmap[P.key] = nil
                        P.model:Destroy()
                        _plist[n] = _plist[#_plist]
                        _plist[#_plist] = nil
                    end
                end
            end
            local cull = _placeR + CELL * 0.75   -- hysteresis: a cell at the rim cannot thrash
            local cull2 = cull * cull
            local n = 1
            while n <= #_plist do
                local P = _plist[n]
                local dx, dz = P.cx - bx, P.cz - bz
                if dx * dx + dz * dz > cull2 then
                    _pmap[P.key] = nil
                    P.model:Destroy()
                    _plist[n] = _plist[#_plist]
                    _plist[#_plist] = nil
                else
                    n = n + 1
                end
            end
            -- fill nearest-first, in square rings out from the player's own cell
            local nr = math.ceil(_placeR / CELL)
            for ring = 0, nr do
                for di = -ring, ring do
                    for dj = -ring, ring do
                        if budget > 0 and #_plist < MAX_REC
                            and math.max(math.abs(di), math.abs(dj)) == ring
                            and tryCell(ci + di, cj + dj, bx, bz, by) then
                            budget = budget - 1
                        end
                    end
                end
            end
            -- a flooded floor is far more water than a scatter of pools, so the rain
            -- ring/glint load is capped in AGGREGATE: density stays even, the particle
            -- count at max intensity stays where the old field's was. Runs last so a
            -- piece placed this tick is already inside the budget.
            local ksum = 0
            for _, P in _plist do
                ksum = ksum + P.k
            end
            local damp = 1
            if ksum > K_CAP then
                damp = K_CAP / ksum
            end
            local rr = (2.1 + 4.2 * _int) * damp
            local gr = 0.8 * _int * damp
            for _, P in _plist do
                -- P.k already carries the trimmed-area scale from placement
                P.ring.Rate = rr * P.k
                P.glint.Rate = gr * P.k
            end
        end

        local function tick20(now, doMaintain)
            local bx, by, bz
            local ch = lp and lp.Character
            local hrp = ch and ch:FindFirstChild("HumanoidRootPart")
            if hrp then
                local p = hrp.Position
                bx, by, bz = p.X, p.Y, p.Z
            elseif Camera then
                local p = Camera.CFrame.Position
                bx, by, bz = p.X, p.Y, p.Z
            else
                return
            end
            if doMaintain then
                maintain(bx, by, bz)
            end
            local list, n = actorsProvider()
            for ai = 1, n do
                local a = list[ai]
                local st = _pstates[a.key]
                if not st then
                    st = { last = 0, wet = 0, side = 1, prevVy = 0 }
                    _pstates[a.key] = st
                end
                local sp = a.speed
                local pud = nil
                for _, P in _plist do
                    if math.abs(a.fy - P.top) < 4.5 and inPuddle(P, a.fx, a.fz) then
                        pud = P
                        break
                    end
                end
                local interval = math.clamp(3.6 / math.max(sp, 1), 0.15, 0.32)   -- stride cadence
                if pud and a.grounded and sp > 2.2 then
                    local landing = st.prevVy < -25   -- was falling last tick, grounded now
                    if landing or now - st.last >= interval then
                        st.last = now
                        st.side = -st.side
                        local fs = sp
                        if landing then
                            fs = 36
                        end
                        fireSplash(a.fx, pud.top, a.fz, fs)
                        st.wet = now + 1.7
                    end
                elseif not pud and now < st.wet and a.grounded and sp > 2.2 then
                    if now - st.last >= interval then
                        st.last = now
                        st.side = -st.side
                        dropPrint(a.fx, a.fy, a.fz, math.atan2(-a.vx, -a.vz), st.side)
                    end
                end
                st.prevVy = a.vy
            end
        end

        local function startPuddles()
            if _pud.conn then
                return
            end
            local folder = Instance.new("Folder")
            folder.Name = "_wxPuddles"
            folder:SetAttribute("WX_Custom", true)
            folder.Parent = getFolder()
            _pud.folder = folder
            _plist = {}
            _pmap = {}
            table.clear(_pfail)
            -- reseeded per session so one map never has a permanently dry corner;
            -- stable WITHIN a session, so walking away and back returns the same flood
            _seed = math.random() * 512
            _homeI, _homeJ = 1e9, 1e9
            refreshParams()
            _units, _uIdx = {}, 1
            for i = 1, 8 do
                _units[i] = mkSplashUnit(folder)
            end
            _prints, _pIdx = {}, 1
            for i = 1, 12 do
                local p = mkFlatPart(Vec(0.6, 0.02, 1.0), CFrame.new(0, -900, 0), folder)
                p.Material = Enum.Material.SmoothPlastic
                p.Color = Color3.fromRGB(14, 18, 25)
                p.Reflectance = 0.18
                _prints[i] = p
            end
            local accum = 0
            local mTick = 0
            _pud.conn = RunService.Heartbeat:Connect(function(dt)
                accum = accum + dt
                if accum < 0.05 then
                    return
                end
                accum = 0
                if not pudActive() then
                    return   -- refresh() owns teardown; just idle on stale frames
                end
                mTick = mTick + 1
                local doMaintain = false
                if mTick >= 14 then   -- ~0.7s: placement, recycle, rate sync
                    mTick = 0
                    doMaintain = true
                end
                pcall(tick20, os.clock(), doMaintain)
            end)
        end
        function stopPuddles(hard)
            if _pud.conn then
                _pud.conn:Disconnect()
                _pud.conn = nil
            end
            table.clear(_pstates)
            local folder = _pud.folder
            _pud.folder = nil
            _plist = {}
            _pmap = {}
            table.clear(_pfail)
            _units = {}
            _prints = {}
            _rebuild = false
            if not folder then
                return
            end
            if hard then
                folder:Destroy()
                return
            end
            -- soft fade: water sheets tween clear, emitters stop, live rings drain out
            for _, m in folder:GetDescendants() do
                if m:IsA("ParticleEmitter") then
                    m.Rate = 0
                elseif m:IsA("Part") and m.Transparency < 1 then
                    TweenService:Create(m, TweenInfo.new(1.4, Enum.EasingStyle.Sine), { Transparency = 1 }):Play()
                end
            end
            -- folder is detached from state above — always safe to reap after the drain
            task.delay(2.2, function()
                pcall(function() folder:Destroy() end)
            end)
        end
        function puddlesRefresh()
            if pudActive() then
                startPuddles()
            else
                stopPuddles(false)
            end
        end
        -- slider: coalesce a drag into ONE reflood on the next maintain tick
        function puddlesIntensity()
            if not _pud.conn then
                return
            end
            _fieldToken = _fieldToken + 1
            local token = _fieldToken
            task.delay(0.25, function()
                if token ~= _fieldToken then
                    return
                end
                _rebuild = true
            end)
        end
    end)()

    -- ── TIME-OF-DAY DIAL (own toggle; only drives ClockTime while the Visuals master is OFF so it
    --    never fights a lighting preset). Cheap 1Hz drift-guarded loop. ──
    local function startClock()
        if _clockConn then return end
        local accum = 0
        _clockConn = RunService.Heartbeat:Connect(function(dt)
            accum = accum + dt
            if accum < 1 then return end
            accum = 0
            if not Config.Weather or not cfg("WeatherClockDial", false) then return end
            if Config.Visuals then return end   -- don't fight a preset
            local cycle = math.max(cfg("WeatherClockCycleMin", 8), 1) * 60
            local t = ((tick() % cycle) / cycle) * 24
            if math.abs(Lighting.ClockTime - t) > 0.02 then
                Lighting.ClockTime = t
            end
        end)
    end
    local function stopClock()
        if _clockConn then _clockConn:Disconnect(); _clockConn = nil end
    end

    -- saved configs may hold a removed type name — fall back to Rain instead of a dead layer
    local function normType(name)
        if name == "Rain" or name == "BloodMoon" or PRESETS[name] then
            return name
        end
        return "Rain"
    end
    local function applyType(name)
        name = normType(name)
        local prev = _curType
        stopRain(); clearSpecial()
        if #_partLayers > 0 then
            beginLayerFade()   -- outgoing particle layers drain instead of hard-cut
        end
        _curType = name
        if name == "Rain" then
            buildRain(); startRain()
        elseif name == "BloodMoon" then
            buildBloodMoon(); startSpecialAnim()
        else
            local ramp = PRESETS[prev] ~= nil   -- particle→particle switch crossfades in
            local ir = IR[name] or IR_DEF
            for _, spec in PRESETS[name] do
                mkPartLayer(spec, ir, ramp)
            end
        end
        if name == "Fireflies" then startFireflies() else stopFireflies() end
        applyMood(name)   -- persistent CC lerps between grades (no pop)
        startAmbient(SOUND[name])
        puddlesRefresh()  -- follows the type: Rain (or Storm on) = wet ground
    end

    function Weather.setType(name)
        name = normType(name)
        Config.WeatherType = name
        if Config.Weather then applyType(name) end
    end
    local _intensityToken = 0
    -- Rate is only one of the axes now. Everything below is a LIVE property write on an
    -- already-running emitter, so a drag never rebuilds a layer; the Rate write still obeys
    -- the rateTween precedence rule, but it re-targets through L.rateMul so a slider move
    -- during a blizzard swell / gust surge / leaf-devil duck keeps the surged LEVEL instead
    -- of snapping the effect out.
    function Weather.setIntensity(v)
        Config.WeatherIntensity = math.clamp(v, 0.15, 2)
        if not Config.Weather then return end
        puddlesIntensity()   -- coverage + water depth ride the slider: one debounced reflood
        local I = Config.WeatherIntensity
        for _, L in _partLayers do
            if L.emitter then
                pcall(tuneLayer, L, I)
                if L.rateTween then L.rateTween:Cancel(); L.rateTween = nil end
                pcall(function() L.emitter.Rate = layerRate(L, I) * L.rateMul end)
            end
        end
        if _special.kind == "BloodMoon" then pcall(tuneBloodMoon, I) end
        if _rays then
            pcall(function()
                _rays.Intensity = 0.10 + 0.18 * I
                _rays.Spread = 1.1 - 0.2 * I
            end)
        end
        -- the rest is per-drag-settle work: the rain pool grows/trims (up to RAIN_MAX
        -- instances) and the mood grade re-tweens over 3s. Coalesce a drag into ONE pass.
        _intensityToken = _intensityToken + 1
        local token = _intensityToken
        task.delay(0.12, function()
            if token ~= _intensityToken then return end
            if not Config.Weather then return end
            if _curType == "Rain" then pcall(tuneRain) end
            if _curType then pcall(applyMood, _curType) end
            pcall(refreshFireflies)
        end)
    end
    function Weather.setSoundVolume(v)
        Config.WeatherSoundVolume = v
        if _ambient then pcall(function() _ambient.Volume = v end) end   -- live: no retoggle needed
    end
    function Weather.toggleStorm(on)
        Config.WeatherStorm = on
        if not Config.Weather then return end
        if on then startStorm() else stopStormLoop() end
        puddlesRefresh()   -- storm alone also wets the ground
    end
    -- Sky-event FREQUENCY setters. Each writes Config (so a save round-trips) and pulls the
    -- pending wait into the new window, so a drag is felt without re-toggling the feature.
    function Weather.setStormMin(v)
        Config.WeatherStormMin = math.clamp(v, 1, 30)
        rescheduleStorm()
    end
    function Weather.setStormVar(v)
        Config.WeatherStormVar = math.clamp(v, 0, 30)
        rescheduleStorm()
    end
    function Weather.setMeteorRate(v)
        Config.WeatherMeteorRate = math.clamp(v, 0.25, 3)
        rescheduleMeteors()
    end
    function Weather.setStarRate(v)
        Config.WeatherStarRate = math.clamp(v, 0.25, 3)
        rescheduleStars()
    end
    function Weather.togglePuddles(on)
        Config.WeatherPuddles = on
        puddlesRefresh()
    end
    function Weather.toggleMood(on)
        Config.WeatherMood = on
        if Config.Weather and _curType then
            if on then applyMood(_curType) else clearMood() end
        end
    end

    function Weather.toggleMeteors(on)
        Config.WeatherMeteors = on
        if Config.Weather then if on then startMeteors() else stopMeteors() end end
    end
    function Weather.toggleShootingStars(on)
        Config.WeatherShootingStars = on
        if Config.Weather then if on then startStars() else stopStars() end end
    end
    function Weather.toggleClock(on)
        Config.WeatherClockDial = on
        if Config.Weather then if on then startClock() else stopClock() end end
    end

    function Weather.enableWeather()
        Config.Weather = true
        getFolder(); startFollow()
        applyType(cfg("WeatherType", "Rain"))
        if cfg("WeatherStorm", false) then startStorm() end
        if cfg("WeatherMeteors", false) then startMeteors() end
        if cfg("WeatherShootingStars", false) then startStars() end
        if cfg("WeatherClockDial", false) then startClock() end
    end
    function Weather.disableWeather()
        Config.Weather = false
        stopFollow(); stopStorm(); stopRain(); clearPartLayers(); killFade(); clearMood()
        clearSpecial(); stopMeteors(); stopStars(); stopFireflies(); stopLeafDevil(); stopClock()
        stopPuddles(true)
        if _ambient then pcall(function() _ambient:Destroy() end); _ambient = nil end
        _curType = nil
    end

    -- ── SKYBOX ──────────────────────────────────────────────────────────────────
    local SKY = {
        Space     = { Bk="rbxassetid://159454299", Dn="rbxassetid://159454296", Ft="rbxassetid://159454293", Lf="rbxassetid://159454286", Rt="rbxassetid://159454300", Up="rbxassetid://159454288" },
        Sunset    = { Bk="rbxassetid://264908339", Dn="rbxassetid://264907909", Ft="rbxassetid://264909420", Lf="rbxassetid://264909758", Rt="rbxassetid://264908886", Up="rbxassetid://264907379" },
        Clouds    = { Bk="rbxassetid://570557514", Dn="rbxassetid://570557775", Ft="rbxassetid://570557559", Lf="rbxassetid://570557620", Rt="rbxassetid://570557672", Up="rbxassetid://570557727" },
        Storm     = { Bk="rbxassetid://255027929", Dn="rbxassetid://255027967", Ft="rbxassetid://255027923", Lf="rbxassetid://255027938", Rt="rbxassetid://255027946", Up="rbxassetid://255027960" },
        Winter    = { Bk="rbxassetid://402229526", Dn="rbxassetid://402229596", Ft="rbxassetid://402229293", Lf="rbxassetid://402229368", Rt="rbxassetid://402229417", Up="rbxassetid://402229564" },
        Vaporwave = { Bk="rbxassetid://1417494030", Dn="rbxassetid://1417494146", Ft="rbxassetid://1417494253", Lf="rbxassetid://1417494402", Rt="rbxassetid://1417494499", Up="rbxassetid://1417494643" },
    }
    Weather.SkyboxOrder = { "Off", "Space", "Sunset", "Clouds", "Storm", "Winter", "Vaporwave" }
    local _sky, _skyConn = nil, nil

    local function buildSky(preset)
        local set = SKY[preset]; if not set then return end
        local s = Instance.new("Sky")
        s.Name = "_wxSky"; s:SetAttribute("WX_Custom", true)
        s.SkyboxBk, s.SkyboxDn, s.SkyboxFt = set.Bk, set.Dn, set.Ft
        s.SkyboxLf, s.SkyboxRt, s.SkyboxUp = set.Lf, set.Rt, set.Up
        if cfg("SkyboxHideCelestial", false) then
            s.SunAngularSize = 0; s.MoonAngularSize = 0; s.StarCount = 0
        end
        s.Parent = Lighting
        _sky = s
    end
    local function startSkyGuard()
        if _skyConn then return end
        _skyConn = Lighting.ChildAdded:Connect(function(c)
            if c:IsA("Sky") and not c:GetAttribute("WX_Custom") and Config.SkyboxPreset and Config.SkyboxPreset ~= "Off" then
                pcall(function() c:Destroy() end)
            end
        end)
    end
    local function stopSkyGuard()
        if _skyConn then _skyConn:Disconnect(); _skyConn = nil end
    end
    local function clearSky()
        if _sky then pcall(function() _sky:Destroy() end); _sky = nil end
    end
    function Weather.setSkybox(preset)
        if preset and not SKY[preset] then preset = "Off" end   -- removed/unknown saved preset
        Config.SkyboxPreset = preset
        clearSky()
        if preset == "Off" or preset == nil then stopSkyGuard(); return end
        buildSky(preset); startSkyGuard()
    end
    function Weather.toggleCelestial(hide)
        Config.SkyboxHideCelestial = hide
        if _sky then
            if hide then _sky.SunAngularSize = 0; _sky.MoonAngularSize = 0; _sky.StarCount = 0
            else _sky.SunAngularSize = 11; _sky.MoonAngularSize = 11; _sky.StarCount = 3000 end
        end
    end

    -- ── RAINBOW (independent toggle; pairs with rain). A real DOUBLE bow, Studio-verified vs
    --    the bright day sky 2026-07-14. Primary = 7 concentric ROYGBIV bands (red outermost)
    --    that all share ONE span so the leg tips CONVERGE (the old staggered spans combed the
    --    ends into frayed strands), over a soft bright-inside glow gapped INSIDE the violet
    --    band (overlapping it diluted the purples to lavender). Plus a faint SECONDARY bow
    --    above, colour order REVERSED (red faces red, real optics). Bands are UNtextured
    --    alpha-leaning beams: LOW transparency + emission 0.42 keeps the pigment saturated —
    --    the old 0.28-0.5 transp @ 0.55-0.65 emission washed to invisible pastel. Pulled to
    --    430 out / -20 up (was 500/-60: legs buried behind map geometry + hazed out).
    --    FaceCamera keeps full band width along the arch; parallax-locked + yaw-billboarded
    --    via the follow loop. ──
    local _rainbow = { host = nil, conn = nil }
    local RB_BANDS = {   -- { colour, mid transparency, width } — narrow dim violets, real-bow feel
        { Color3.fromRGB(255, 40, 40),  0.10, 24 },
        { Color3.fromRGB(255, 130, 20), 0.12, 24 },
        { Color3.fromRGB(255, 225, 40), 0.14, 24 },
        { Color3.fromRGB(60, 210, 70),  0.16, 24 },
        { Color3.fromRGB(40, 130, 255), 0.18, 24 },
        { Color3.fromRGB(85, 60, 235),  0.26, 18 },
        { Color3.fromRGB(165, 65, 230), 0.30, 16 },
    }
    local RB_DIST, RB_OY = 430, -20
    local function clearRainbow()
        if _rainbow.conn then _rainbow.conn:Disconnect(); _rainbow.conn = nil end
        if _rainbow.host then pcall(function() _rainbow.host:Destroy() end); _rainbow.host = nil end
    end
    local function buildRainbow()
        local host = Instance.new("Part")
        host.Name = "_wxRainbow"; host.Anchored = true; host.CanCollide = false; host.CanQuery = false
        host.CanTouch = false; host.CastShadow = false; host.Massless = true; host.Transparency = 1
        host.Size = Vec(1, 1, 1); host:SetAttribute("WX_Custom", true)
        local rot = CFrame.Angles(0, 0, math.rad(90))   -- turn attachment Right(+X)→+Y so CurveSize bows UP
        local function band(sp, rr, w, col, midT, emit, soft)
            local a0 = Instance.new("Attachment"); a0.CFrame = CFrame.new(-sp, 0, 0) * rot; a0.Parent = host
            local a1 = Instance.new("Attachment"); a1.CFrame = CFrame.new( sp, 0, 0) * rot; a1.Parent = host
            local b = Instance.new("Beam")
            b.Attachment0 = a0; b.Attachment1 = a1; b.Segments = 46; b.FaceCamera = true
            if soft then b.Texture = TX_SOFT; b.TextureMode = Enum.TextureMode.Stretch; b.TextureLength = 1 end
            b.Width0 = w; b.Width1 = w; b.CurveSize0 = rr; b.CurveSize1 = -rr
            b.LightEmission = emit; b.LightInfluence = 0
            b.Color = ColorSequence.new(col)
            local edge = math.min(midT + 0.25, 1)   -- legs dissolve toward the horizon
            b.Transparency = NumberSequence.new({
                NSK(0, 1), NSK(0.08, edge), NSK(0.45, midT), NSK(0.55, midT), NSK(0.92, edge), NSK(1, 1) })
            b.Parent = host
        end
        for i = 1, #RB_BANDS do
            local B = RB_BANDS[i]
            band(340, 320 - (i - 1) * 15, B[3], B[1], B[2], 0.42, false)
        end
        band(340, 165, 52, Color3.fromRGB(242, 248, 255), 0.78, 0.6, true)   -- bright-inside glow
        for i = 1, #RB_BANDS do   -- faint SECONDARY bow above, colours reversed (red faces red)
            band(400, 355 + (i - 1) * 12, 20, RB_BANDS[i][1], 0.64 + i * 0.012, 0.42, false)
        end
        host.Parent = getFolder()
        _rainbow.host = host
    end
    local function startRainbowFollow()
        if _rainbow.conn then return end
        local accum = 0
        _rainbow.conn = RunService.Heartbeat:Connect(function(dt)
            if not cfg("WeatherRainbow", false) then return end
            accum = accum + dt; if accum < 0.08 then return end   -- ~12Hz is plenty for a sky element
            accum = 0
            local cam = Camera; if not cam then return end
            local host = _rainbow.host; if not host or not host.Parent then return end
            local p = cam.CFrame.Position
            local hp = Vec(p.X, p.Y + RB_OY, p.Z - RB_DIST)        -- fixed world azimuth, XZ-parallax
            host.CFrame = CFrame.lookAt(hp, Vec(p.X, hp.Y, p.Z))   -- yaw-billboard: always a face-on arch
        end)
    end
    function Weather.toggleRainbow(on)
        Config.WeatherRainbow = on
        clearRainbow()
        if not on then return end
        buildRainbow(); startRainbowFollow()
    end

    -- ── GOD RAYS (independent toggle, skybox-style lifecycle): volumetric shafts off the sun.
    --    A single SunRaysEffect — no Atmosphere sidecar, so it never fights BloodMoon/mood. ──
    function Weather.toggleGodRays(on)
        Config.WeatherGodRays = on
        if _rays then pcall(function() _rays:Destroy() end); _rays = nil end
        if not on then return end
        local r = Instance.new("SunRaysEffect")
        r.Name = "_wxRays"; r:SetAttribute("WX_Custom", true)
        -- anchored at the approved 0.28 / 0.9 for I = 1; strong intensity = brighter and
        -- TIGHTER shafts, weak = a faint broad wash. Broad soft shafts, not a laser star.
        local I = iAmt()
        r.Intensity = 0.10 + 0.18 * I
        r.Spread = 1.1 - 0.2 * I
        r.Parent = Lighting
        _rays = r
    end

    -- ── lifecycle ──
    function Weather.init()
        if Config.Weather then pcall(Weather.enableWeather) end
        if Config.SkyboxPreset and Config.SkyboxPreset ~= "Off" then pcall(Weather.setSkybox, Config.SkyboxPreset) end
        if Config.WeatherGodRays then pcall(Weather.toggleGodRays, true) end
        if Config.WeatherRainbow then pcall(Weather.toggleRainbow, true) end
    end
    function Weather.unload()
        pcall(Weather.disableWeather)
        stopSkyGuard(); clearSky(); clearRainbow()
        if _rays then pcall(function() _rays:Destroy() end); _rays = nil end
        if _folder then pcall(function() _folder:Destroy() end); _folder = nil end
        _lightFolder = nil
    end
end)()

-- ── HUD+ · INSTRUMENT CLUSTER ───────────────────────────────────────────────
-- Tactical readouts that Rivals' own UI does NOT provide: a top-center bearing COMPASS (enemy
-- pips + off-window threat pill), a crosshair THREAT ARC pointing at the nearest enemy bearing,
-- and a nearest-enemy RANGE readout. All default-OFF, all riding the HUD master (Config.HUD) +
-- their own key. ONE 30Hz RenderStepped updater, ONE module-local pool built once (radar pattern),
-- numeric hot loops, integer-snapped writes to feed the shim skip-write. Draws on the "fx" layer.
-- Internals live in an IIFE so the block-locals never touch the main-chunk register budget.
-- (Ammo plate + session card were REMOVED: the ammo plate duped Rivals' own ammo UI, and the
--  session card duped the watermark's shots/hits/acc line.)
local HUDPlus = {}
;(function()
    local hasDrawing = screenDraw ~= nil

    -- SIGNAL palette (design §1.1). Shapes carry state; text stays neutral; gold is earned.
    local C_TEXT1  = Color3.fromRGB(243, 246, 250)  -- #F3F6FA values
    local C_TEXT2  = Color3.fromRGB(174, 185, 197)  -- #AEB9C5 labels/data
    local C_GOLD   = Color3.fromRGB(255, 194, 75)   -- #FFC24B north / accent
    local C_ENEMY  = Color3.fromRGB(255, 59, 78)    -- #FF3B4E enemy pips
    local C_EDGE   = Color3.fromRGB(155, 232, 255)  -- #9BE8FF transient motion (caret glint)
    local C_BLACK  = Color3.new(0, 0, 0)

    local PI  = math.pi
    local TAU = PI * 2
    local RAD60 = math.rad(60)

    -- ── pools (built ONCE) ──────────────────────────────────────────────────
    local _alloc = false
    local _cRail, _cRailCas = nil, nil
    local _cTicks = nil                              -- 10 tick Lines
    local _cCard  = nil                              -- 4 cardinal Texts (N/E/S/W)
    local _cCaret = nil                              -- center caret Triangle
    local _cPips  = nil                              -- 8 enemy pip Circles
    local _cPill  = nil                              -- off-window threat count "▲N"
    local _taSeg, _taSegCas = nil, nil               -- threat arc: 9 seg Lines (+ casing)
    local _rng = nil                                 -- range readout Text

    -- ── lifecycle state ─────────────────────────────────────────────────────
    local _conn = nil
    local _started = false
    local _thrT = 0                                  -- 30Hz throttle
    -- reused scratch (no per-frame alloc)
    local _pip = {}                                  -- {rel, dist} rows for compass pips

    local function nd(kind, props)
        if not hasDrawing then return nil end
        local ok, d = pcall(screenDraw, kind, "fx")
        if not ok or not d then return nil end
        d.Visible = false
        if props then for k, v in pairs(props) do pcall(function() d[k] = v end) end end
        return d
    end

    local function alloc()
        if _alloc then return end
        _alloc = true
        -- COMPASS (casing rail created before the bright rail = under)
        _cRailCas = nd("Line", { Color = C_BLACK, Thickness = 3 })
        _cRail    = nd("Line", { Color = C_TEXT2, Thickness = 1 })
        _cTicks = {}
        for i = 1, 10 do _cTicks[i] = nd("Line", { Thickness = 1 }) end
        _cCard = {}
        for i = 1, 4 do _cCard[i] = nd("Text", { Center = true, Outline = true, Font = 3, Size = 11 }) end
        _cCaret = nd("Triangle")
        _cPips = {}
        for i = 1, 8 do _cPips[i] = nd("Circle", { Filled = true, NumSides = 12 }) end
        _cPill = nd("Text", { Center = true, Outline = true, Font = 3, Size = 11, Color = C_ENEMY })
        -- THREAT ARC (casing first = under)
        _taSegCas = {}
        for i = 1, 9 do _taSegCas[i] = nd("Line", { Color = C_BLACK, Thickness = 4 }) end
        _taSeg = {}
        for i = 1, 9 do _taSeg[i] = nd("Line", { Thickness = 2 }) end
        -- RANGE readout
        _rng = nd("Text", { Center = true, Outline = true, Font = 3, Size = 13, Color = C_TEXT1 })
    end

    -- ── per-feature hide helpers (idle features cost nothing after one hide) ──
    local function hideCompass()
        if _cRail then _cRail.Visible = false; _cRailCas.Visible = false end
        if _cTicks then for i = 1, #_cTicks do _cTicks[i].Visible = false end end
        if _cCard then for i = 1, 4 do _cCard[i].Visible = false end end
        if _cCaret then _cCaret.Visible = false end
        if _cPips then for i = 1, #_cPips do _cPips[i].Visible = false end end
        if _cPill then _cPill.Visible = false end
    end
    local function hideArc()
        if _taSeg then for i = 1, 9 do _taSeg[i].Visible = false; _taSegCas[i].Visible = false end end
    end
    local function hideRange() if _rng then _rng.Visible = false end end
    local function hideAll()
        hideCompass(); hideArc(); hideRange()
    end

    local CARDINALS = {
        { "N", 0, 0, -1, true  },   -- world -Z, gold
        { "E", 1, 0,  0, false },
        { "S", 0, 0,  1, false },
        { "W", -1, 0, 0, false },
    }

    local function wrap(a)
        a = (a + PI) % TAU
        if a < 0 then a = a + TAU end
        return a - PI
    end

    -- ── COMPASS ──────────────────────────────────────────────────────────────
    -- camYaw from the flattened look heading; every element's bearing is (worldBearing - camYaw)
    -- so ticks, cardinals and pips all share ONE reference — they can never disagree.
    local function drawCompass(vp, camYaw, y)
        local railW = math.floor(math.max(Config.HUDCompassWidth or 380, 120))
        local half = railW * 0.5
        local cxp = math.floor(vp.X * 0.5)
        -- rail + casing
        local lx, rx = cxp - half, cxp + half
        _cRailCas.From = Vector2.new(lx, y); _cRailCas.To = Vector2.new(rx, y)
        _cRailCas.Transparency = 0.5; _cRailCas.Visible = true
        _cRail.From = Vector2.new(lx, y); _cRail.To = Vector2.new(rx, y)
        _cRail.Transparency = 0.85; _cRail.Visible = true
        -- 15° ticks across the window; taller at 45° multiples
        local base = math.floor(math.deg(camYaw) / 15 + 0.5) * 15
        local shown = 0
        for k = -4, 4 do
            local degA = base + k * 15
            local rel = wrap(math.rad(degA) - camYaw)
            if rel >= -RAD60 and rel <= RAD60 then
                shown = shown + 1
                local L = _cTicks[shown]
                if L then
                    local tx = math.floor(cxp + (rel / RAD60) * half)
                    local tall = (degA % 45 == 0) and 7 or 4
                    L.From = Vector2.new(tx, y - tall); L.To = Vector2.new(tx, y)
                    L.Color = C_TEXT2; L.Transparency = 0.45; L.Visible = true
                end
            end
        end
        for i = shown + 1, #_cTicks do _cTicks[i].Visible = false end
        -- cardinal labels
        for i = 1, 4 do
            local c = CARDINALS[i]
            local bearing = math.atan2(c[2], c[4])
            local rel = wrap(bearing - camYaw)
            local T = _cCard[i]
            if T then
                if rel >= -RAD60 and rel <= RAD60 then
                    local tx = math.floor(cxp + (rel / RAD60) * half)
                    T.Position = Vector2.new(tx, y - 22)
                    T.Text = c[1]; T.Color = c[5] and C_GOLD or C_TEXT2
                    T.Transparency = 1; T.Visible = true
                else T.Visible = false end
            end
        end
        -- center caret with an EDGE glint (transient accent — pulses subtly at 1.6Hz)
        if _cCaret then
            local glint = 0.5 + 0.5 * math.sin((tick()) * 1.6 * TAU)
            _cCaret.Filled = true
            _cCaret.PointA = Vector2.new(cxp, y + 7)
            _cCaret.PointB = Vector2.new(cxp - 5, y + 1)
            _cCaret.PointC = Vector2.new(cxp + 5, y + 1)
            _cCaret.Color = C_EDGE
            _cCaret.Transparency = 0.65 + 0.35 * glint
            _cCaret.Visible = true
        end
    end

    -- ── the ONE updater ──────────────────────────────────────────────────────
    local function update()
        if not (Config.HUD and hasDrawing) then hideAll(); return end
        local now = tick()
        if now - _thrT < 0.033 then return end   -- 30Hz
        _thrT = now
        local cam = Camera; if not cam then return end
        local vp = cam.ViewportSize
        local cx, cy = vp.X * 0.5, vp.Y * 0.5

        local compassOn = Config.HUDCompass
        local arcOn     = Config.HUDThreatArc
        local rangeOn   = Config.HUDRangeReadout

        -- shared enemy sweep (compass pips + nearest for arc/range) — ONE pass over players
        local myC = lp.Character
        local myR = myC and myC:FindFirstChild("HumanoidRootPart")
        local myPos = myR and myR.Position
        local camPos = cam.CFrame.Position
        local look = cam.CFrame.LookVector
        local camYaw = math.atan2(look.X, look.Z)

        local nearD, nearRel = math.huge, 0
        local pipN, offCount = 0, 0
        if compassOn or arcOn or rangeOn then
            local players = getSafePlayers()
            for i = 1, #players do
                local pl = players[i]
                if pl ~= lp and not isTeammate(pl) and isAlive(pl) then
                    local ch = pl.Character
                    local r = ch and ch:FindFirstChild("HumanoidRootPart")
                    if r then
                        local pos = r.Position
                        local dx, dz = pos.X - camPos.X, pos.Z - camPos.Z
                        local rel = wrap(math.atan2(dx, dz) - camYaw)
                        local d = myPos and (myPos - pos).Magnitude
                            or math.sqrt(dx * dx + dz * dz)
                        if d < nearD then nearD = d; nearRel = rel end
                        if rel < -RAD60 or rel > RAD60 then offCount = offCount + 1 end
                        if compassOn and pipN < 8 then
                            pipN = pipN + 1
                            local e = _pip[pipN]; if not e then e = {}; _pip[pipN] = e end
                            e.rel = rel; e.dist = d
                        end
                    end
                end
            end
        end

        -- COMPASS
        if compassOn then
            local y = 26
            drawCompass(vp, camYaw, y)
            local half = math.max(Config.HUDCompassWidth or 380, 120) * 0.5
            local cxp = math.floor(cx)
            local maxD = math.max(Config.ESPMaxDistance or 1200, 1)
            local drawPips = Config.HUDCompassPips and pipN or 0
            for i = 1, drawPips do
                local e = _pip[i]
                local P = _cPips[i]
                if P then
                    local rel = e.rel
                    local clamped = rel
                    if clamped < -RAD60 then clamped = -RAD60 elseif clamped > RAD60 then clamped = RAD60 end
                    local edge = (rel < -RAD60 or rel > RAD60)   -- behind us: edge-clamp + dim
                    local px = math.floor(cxp + (clamped / RAD60) * half)
                    local a = 1 - 0.6 * math.clamp(e.dist / maxD, 0, 1)
                    if edge then a = a * 0.4 end
                    P.Radius = 2; P.Position = Vector2.new(px, 26)
                    P.Color = C_ENEMY; P.Transparency = a; P.Visible = true
                end
            end
            for i = drawPips + 1, #_cPips do _cPips[i].Visible = false end
            -- off-window threat pill at the rail's right end
            if _cPill then
                if offCount > 0 then
                    _cPill.Text = "\226\150\178" .. offCount   -- ▲N
                    _cPill.Position = Vector2.new(math.floor(cxp + half + 16), 20)
                    _cPill.Transparency = 1; _cPill.Visible = true
                else _cPill.Visible = false end
            end
        else hideCompass() end

        -- THREAT ARC around the crosshair (nearest enemy bearing)
        if arcOn and _taSeg and nearD < math.huge then
            local R = 42
            local width = math.rad(30)
            local step = width / 9
            local th0 = nearRel - width * 0.5
            -- GOLD close -> TEXT-2 far; alpha 1 -> 0.25 with distance
            local dfrac = math.clamp((nearD - 25) / (300 - 25), 0, 1)
            local col = C_GOLD:Lerp(C_TEXT2, dfrac)
            local a = 1 - 0.75 * dfrac
            for j = 1, 9 do
                local t1 = th0 + step * (j - 1)
                local t2 = t1 + step * 0.72
                local p1 = Vector2.new(cx + math.sin(t1) * R, cy - math.cos(t1) * R)
                local p2 = Vector2.new(cx + math.sin(t2) * R, cy - math.cos(t2) * R)
                local cs = _taSegCas[j]
                if cs then cs.From = p1; cs.To = p2; cs.Transparency = a * 0.8; cs.Visible = true end
                local L = _taSeg[j]
                if L then L.From = p1; L.To = p2; L.Color = col; L.Transparency = a; L.Visible = true end
            end
        else hideArc() end

        -- RANGE readout under the crosshair (nearest enemy)
        if rangeOn and _rng then
            if nearD < math.huge then
                _rng.Text = ("%dm"):format(math.floor(nearD + 0.5))
                _rng.Position = Vector2.new(math.floor(cx), math.floor(cy + 64))
                _rng.Transparency = 1; _rng.Visible = true
            else _rng.Visible = false end
        else hideRange() end
    end

    function HUDPlus.start()
        if _started then return end
        _started = true
        alloc()
        if not _conn then
            _conn = RunService.RenderStepped:Connect(function() if _started then pcall(update) end end)
        end
    end
    function HUDPlus.stop()
        if not _started then return end
        _started = false
        if _conn then _conn:Disconnect(); _conn = nil end
        hideAll()
    end
    function HUDPlus.init()
        -- rides the HUD master: only spin up the loop when the HUD subsystem is on.
        if Config.HUD then pcall(HUDPlus.start) end
    end
    function HUDPlus.unload()
        pcall(HUDPlus.stop)
    end
end)()

-- ════════════════ PART 2 ════════════════
-- Rage · Gun Hook · Aimbot · ESP · GUI · Init
-- ══════════════════════════════════════════

-- ── RAGE INTERNALS ─────────────────────────────────────────────────────────
local Rage = {}
-- Rage internals run in their OWN function proto, not the main chunk. The executor prepends a ~25-local
-- API-shim preamble to the MAIN chunk that would push it past Luau's 200-local/proto cap, so we keep these
-- locals in a fresh IIFE proto budget. `Rage` (the export table) stays in the outer scope. This IIFE spans
-- BOTH 07a and 07b — it OPENS here (`;(function()`) and CLOSES in 07b (`end)()`); everything between is one
-- Luau proto sharing the 190-local budget. Runs immediately.
;(function()
    local _tgtConn  = nil   -- target-acquisition loop (fills RageTrueVelocityMap; sets State.Target/RageTarget)
    local _labConn  = nil   -- Lab-gated enemy-damage poll (feeds State.RageDealtTotal); only alive while RageLab

    -- per-life character token (guards the desync restore from yanking a freshly-respawned body)
    local function bump(p) State.RageCharTokens[p] = (State.RageCharTokens[p] or 0) + 1 end

    -- kill floor: the lowest Y we clamp a point-blank park to (above the fallen-parts sweep + our buffer)
    local function killFloor()
        return workspace.FallenPartsDestroyHeight + Config.RageKillPlaneBuffer
    end

    -- Clear LOS from a forged-eye position to the resolved head (private; used by flankPoint + the natural
    -- encoder). Exclude filter FAILS SAFE — a false block only skips a frame; a false clear can't happen.
    local function hasLOS(fromPos, toPos, ignore)
        local rp = RaycastParams.new()
        rp.FilterType = Enum.RaycastFilterType.Exclude
        rp.FilterDescendantsInstances = ignore
        local res = workspace:Raycast(fromPos, toPos - fromPos, rp)
        if not res then return true end
        return (res.Position - toPos).Magnitude < 3
    end

    -- ── SHARED SHOT GEOMETRY ────────────────────────────────────────────────
    -- Build the char0/char1/char3 packet fields into camData for a shot at `aimWorldPos` on `hitPart`, from
    -- eye `eyeCF` / muzzle `muzzleCF`. `jitter` (MANDATORY): when true, the char3 object-space offset is
    -- randomized within the clamp band so it varies per shot (a real Rivals shot is never char3=0 static);
    -- `clampFrac`/`jitterFrac` set the band. char2 (the live hitPart instance) is set by the caller. Returns
    -- the encoded fields via camData; the caller owns the utf8-keyed envelope.
    local function buildShotFields(camData, eyeCF, muzzleCF, hitPart, aimWorldPos, jitter, clampFrac, jitterFrac)
        local objSpace = hitPart.CFrame:PointToObjectSpace(aimWorldPos)
        local hs = hitPart.Size * (clampFrac or 0.45)
        objSpace = Vector3.new(
            math.clamp(objSpace.X, -hs.X, hs.X),
            math.clamp(objSpace.Y, -hs.Y, hs.Y),
            math.clamp(objSpace.Z, -hs.Z, hs.Z)
        )
        if jitter then
            local jf = jitterFrac or 1.0
            objSpace = Vector3.new(
                math.clamp(objSpace.X + (math.random() - 0.5) * hs.X * jf, -hs.X, hs.X),
                math.clamp(objSpace.Y + (math.random() - 0.5) * hs.Y * jf, -hs.Y, hs.Y),
                math.clamp(objSpace.Z + (math.random() - 0.5) * hs.Z * jf, -hs.Z, hs.Z)
            )
        end
        local hitWorld   = hitPart.CFrame:PointToWorldSpace(objSpace)
        local objSpaceCF = hitPart.CFrame:ToObjectSpace(CFrame.new(hitWorld))
        camData[utf8.char(0)] = Rivals.Util:EncodeCFrame(eyeCF)
        camData[utf8.char(1)] = Rivals.Util:EncodeCFrame(muzzleCF)
        camData[utf8.char(2)] = hitPart
        camData[utf8.char(3)] = Rivals.Util:EncodeCFrame(objSpaceCF)
    end

    -- Shot encoder (silent-aim / aimbot path — the 08 hook calls Rage._encodeShot). Far eye => the ±45%
    -- clamp + full-half-extent jitter are sub-degree, always on the part. jitter follows Config.SilentAimJitter.
    local function encodeShot(camData, hitPart, targetChar, fromCamPos)
        if not hitPart or not camData or not Rivals.Util then return false end
        local lead      = calculateLead(targetChar, fromCamPos)
        local leadedPos = hitPart.Position + lead
        local eyeCF     = CFrame.new(fromCamPos, leadedPos)
        local muzzlePos = fromCamPos + (eyeCF.RightVector * 0.2) + Vector3.new(0, -Config.RageEyeMuzzleSep, 0)
        local muzzleCF  = CFrame.new(muzzlePos, leadedPos)
        buildShotFields(camData, eyeCF, muzzleCF, hitPart, leadedPos, Config.SilentAimJitter, 0.45, 1.0)
        return true
    end
    Rage._encodeShot = encodeShot

    -- ── WEAPON SWITCH / reload economy ──────────────────────────────────────
    local PICK_SLOT = { Primary = 1, Secondary = 2, Melee = 3 }
    ;(function()
        -- Sub-IIFE (register budget): the ~12 economy locals live in their own proto so they don't count
        -- against the shared rage-IIFE 190-cap. Exports go on the Rage table.
        local function fighterItems()
            local lf = Rivals.Fighter and Rivals.Fighter.LocalFighter
            return lf and lf.Items or nil
        end
        local function itemIsMelee(item)
            if not item then return false end
            local okN, nm = pcall(function() return item:Get("Name") or item.Name end)
            if okN and type(nm) == "string" and nm:lower():find("melee", 1, true) then return true end
            local okT, t = pcall(function() return item:Get("ItemType") or item:Get("Type") or item:Get("Category") end)
            if okT and type(t) == "string" and t:lower():find("melee", 1, true) then return true end
            return false
        end
        Rage._itemIsMelee = itemIsMelee
        local function itemAmmo(item)
            if not item then return nil end
            local ok, a = pcall(function() return item:Get("Ammo") or item:Get("CurrentAmmo") end)
            if ok and type(a) == "number" then return a end
            return nil
        end
        local function slotItemByIndex(idx)
            local items = fighterItems(); if not items then return nil end
            if items[idx] then return items[idx] end
            if items[tostring(idx)] then return items[tostring(idx)] end
            for key, it in pairs(items) do
                if it and typeof(it) == "table" then
                    local okS, s = pcall(function() return tonumber(it:Get("Slot") or it:Get("Index") or it:Get("ItemSlot") or key) end)
                    if okS and s == idx then return it end
                    local okT, t = pcall(function() return it:Get("ItemType") or it:Get("Type") end)
                    if okT then
                        if idx == 1 and (t == "Primary" or t == 1 or t == "1") then return it end
                        if idx == 2 and (t == "Secondary" or t == 2 or t == "2") then return it end
                        if idx == 3 and (t == "Melee" or (type(t) == "string" and t:lower():find("melee", 1, true))) then return it end
                    end
                end
            end
            return nil
        end
        local function whichSlotNow()
            local cur = getEquippedItem(); if not cur then return nil end
            local okId, oid = pcall(function() return cur:Get("ObjectID") end)
            if okId and oid then
                for idx = 1, 3 do
                    local it = slotItemByIndex(idx)
                    if it then
                        local okI, iid = pcall(function() return it:Get("ObjectID") end)
                        if okI and iid == oid then return idx end
                    end
                end
            end
            if itemIsMelee(cur) then return 3 end
            return nil
        end
        local function slotUsable(idx)
            local it = slotItemByIndex(idx); if not it then return false end
            if itemIsMelee(it) then return Config.RageSwitchMelee == true end
            local a = itemAmmo(it)
            return a == nil or a > 0
        end
        local function slotOrder()
            local pref = PICK_SLOT[Config.RageWeaponPick] or 1
            local order = { pref }
            for _, s in ipairs({ 1, 2, 3 }) do
                if s ~= pref then order[#order + 1] = s end
            end
            if not Config.RageSwitchMelee then
                local filtered = {}
                for _, s in ipairs(order) do if s ~= 3 then filtered[#filtered + 1] = s end end
                order = filtered
            end
            return order
        end
        local function nextUsableSlot(excludeIdx)
            for _, s in ipairs(slotOrder()) do
                if s ~= excludeIdx and slotUsable(s) then return s end
            end
            return nil
        end
        -- ONE equip action per switch, delivered through the game's OWN path. The speculative
        -- {Equip,Switch,Select,EquipItem,ChangeItem} UseItem:FireServer loop is GONE: those enum names were
        -- never verified, the payload was nil where a real shot sends a table, and no legitimate client emits
        -- five item actions for one ObjectID inside a frame — pure garbage on the same replication remote our
        -- shots ride. The synthetic hotkey is now a FALLBACK ONLY (firing it alongside lf:EquipItem replicated
        -- the same equip twice at up to 16 Hz — a cadence no human input path can produce).
        local function equipSlot(idx)
            if not idx then return false end
            if whichSlotNow() == idx then return true end
            local now = tick()
            if now - (State.RageSwitchLast or 0) < (Config.RageSwitchRateLimit or 0.06) then return false end
            State.RageSwitchLast = now
            local lf = Rivals.Fighter and Rivals.Fighter.LocalFighter
            local equipped = false
            if lf and lf.EquipItem then
                equipped = pcall(function() lf:EquipItem(idx) end)
            end
            if not equipped then
                pcall(function()
                    local kc = ({ Enum.KeyCode.One, Enum.KeyCode.Two, Enum.KeyCode.Three })[idx]
                    if kc then
                        VirtualInputMgr:SendKeyEvent(true, kc, false, game)
                        VirtualInputMgr:SendKeyEvent(false, kc, false, game)
                    end
                end)
            end
            return true
        end
        Rage._equipSlot = equipSlot
        Rage._nextUsableSlot = nextUsableSlot
        Rage._whichSlotNow = whichSlotNow   -- APEX economy bridge (fire-then-swap slot resolution)
    end)()

    -- NATURAL-delivery re-encoder (08 hook calls Rage._encodeRageShot on every Gun.StartShooting while
    -- Config.Rage & not RageDirectFire). The engine stores THIS shot's solution in State.RageFire*; here we
    -- rebuild char0-3 the SAME way the forged path does (the proven 7-0 shape: char0==char1 straight-up
    -- on-head eye, char2=live head, char3=0), and — because the game's StartShooting pipeline re-enters us at
    -- ITS own cadence, upstream of the
    -- engine's fire choke — we RE-ENFORCE the identical ammo/reload/equip/rate gate here so a natural shot
    -- can never fire in an impossible weapon state or over-rate. Re-validation means a stale/deferred read
    -- can only bail, never emit an illegal packet. Increments State.Shots exactly once per emitted packet.
    local function encodeRageShot(camData)
        local hitPart = State.RageFireHitPart
        local eyePos  = State.RageFireFromPos
        local aimPos  = State.RageFireAimPos
        if not (hitPart and hitPart.Parent and eyePos and aimPos) then return false end
        if not isSanePos(eyePos) or not isSanePos(hitPart.Position) then return false end
        if (hitPart.Position - eyePos).Magnitude > (400 - 5) then return false end
        local ignore = { lp.Character }
        local t = State.RageTarget or State.Target
        if t and t.Character then ignore[2] = t.Character end
        if not hasLOS(eyePos, hitPart.Position, ignore) then return false end

        -- IMPOSSIBLE-STATE + RATE GATE (natural path is upstream of the engine choke, so gate HERE too).
        local item = getEquippedItem(); if not item then return false end
        local okE, equipping = pcall(function() return item:IsEquipping() end)
        if okE and equipping then return false end
        if not (Rage._itemIsMelee and Rage._itemIsMelee(item)) then
            local okR, reloading = pcall(function() return (item._reload_cooldown or 0) > tick() end)
            if okR and reloading then return false end
            local okA, ammo = pcall(function() return item:Get("Ammo") end)
            if not okA or type(ammo) ~= "number" or ammo <= 0 then return false end
        end
        local now = tick()
        local interval = Rage._fireInterval and Rage._fireInterval() or 0.07
        if now - (State.RageLastFireTime or 0) < interval then return false end
        State.RageLastFireTime = now

        -- BUILD the packet — the proven 7-0 shape mirrored from the forged path: char0 == char1 == enc (the eye
        -- is STRAIGHT UP on the head, so no RightVector is read — a vertical look NaNs it), char2 = live head,
        -- char3 = EncodeCFrame(CFrame.new(0,0,0)). No separate muzzle => the ray-origin stays exactly on the head.
        local enc = Rivals.Util:EncodeCFrame(CFrame.new(eyePos, aimPos))
        camData[utf8.char(0)] = enc
        camData[utf8.char(1)] = enc
        camData[utf8.char(2)] = hitPart
        camData[utf8.char(3)] = Rivals.Util:EncodeCFrame(CFrame.new(0, 0, 0))

        State.Shots = State.Shots + 1
        return true
    end

    -- ── TARGET SYSTEM ───────────────────────────────────────────────────────
    local function findTarget()
        local myChar = lp.Character
        local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
        local cands  = {}
        for _, p in ipairs(getSafePlayers()) do
            if isValidTarget(p, Config.RageVisCheck, true) then   -- keepDeflect: rage flanks, never drops
                local hum = p.Character:FindFirstChildOfClass("Humanoid")
                local hrp = p.Character:FindFirstChild("HumanoidRootPart")
                local d   = myRoot and hrp and (hrp.Position - myRoot.Position).Magnitude or 9999
                table.insert(cands, { p = p, hp = hum.Health, d = d })
            end
        end
        if #cands == 0 then return nil end
        if Config.RageHPPriority then
            table.sort(cands, function(a, b)
                if math.abs(a.hp - b.hp) > 10 then return a.hp < b.hp end
                return a.d < b.d
            end)
        else
            table.sort(cands, function(a, b) return a.d < b.d end)
        end
        return cands[1].p
    end
    Rage._findTarget = findTarget

    -- APEX scorer: findTarget + a per-candidate head-sane term + a head-sane-first sort key. Head-surfaced
    -- enemies sort ahead of voided ones unconditionally (swing to whoever we can actually hit right now), then
    -- HP/distance as usual. isValidTarget (05_utils) is left UNCHANGED — the head term is scoped to APEX here so
    -- Polar's validity gate never drifts.
    local function findTargetHeadSane()
        local myChar = lp.Character
        local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
        local cands  = {}
        for _, p in ipairs(getSafePlayers()) do
            if isValidTarget(p, Config.RageVisCheck, true) then   -- keepDeflect: rage flanks, never drops
                local hum = p.Character:FindFirstChildOfClass("Humanoid")
                local hrp = p.Character:FindFirstChild("HumanoidRootPart")
                local hh  = p.Character:FindFirstChild("HitboxHead") or p.Character:FindFirstChild("Head")
                local headSane = false
                if hh and isSanePos(hh.Position) then headSane = true end
                local d = 9999
                if myRoot and hrp then d = (hrp.Position - myRoot.Position).Magnitude end
                table.insert(cands, { p = p, hp = hum.Health, d = d, s = headSane })
            end
        end
        if #cands == 0 then return nil end
        table.sort(cands, function(a, b)
            if a.s ~= b.s then return a.s end   -- head-sane candidates first, unconditionally
            if Config.RageHPPriority and math.abs(a.hp - b.hp) > 10 then return a.hp < b.hp end
            return a.d < b.d
        end)
        return cands[1].p
    end
    Rage._findTargetHeadSane = findTargetHeadSane

    -- Target-acquisition loop. KEEP: it fills State.RageTrueVelocityMap (calculateLead's fallback at
    -- 05_utils) as a side effect, so deleting it would feed stale lead data. Also maintains State.Target.
    local function startTargetLoop()
        if _tgtConn then _tgtConn:Disconnect() end
        local lastPosMap, lastTimeMap = {}, {}
        _tgtConn = RunService.Heartbeat:Connect(function()
            if not Config.Rage then return end
            local nowTime = tick()
            local safe = getSafePlayers() or {}
            for i = 1, #safe do
                local p = safe[i]
                if p ~= lp and p.Character then
                    local pRoot = p.Character:FindFirstChild("HumanoidRootPart")
                    if pRoot then
                        local currentPos = pRoot.Position
                        if lastPosMap[p] and lastTimeMap[p] then
                            local dt = nowTime - lastTimeMap[p]
                            if dt > 0 then State.RageTrueVelocityMap[p] = (currentPos - lastPosMap[p]) / dt end
                        end
                        lastPosMap[p] = currentPos
                        lastTimeMap[p] = nowTime
                    end
                end
            end
            if Config.RageFastTargetSwitch and State.Target and not isValidTarget(State.Target, false, true) then
                State.Target = nil
            end
            State.Target = findTarget()
        end)
    end
    Rage._startTargetLoop = startTargetLoop

    -- LAB-gated enemy-damage poll (feeds State.RageDealtTotal for the Lab trade readout). HealthChanged
    -- misses on desynced/streamed enemies, so we diff Humanoid.Health each frame. Costs nothing when the
    -- Lab is off (early return). 1v1-accurate; attributes any enemy HP loss to us.
    local function startLabPoll()
        if _labConn then return end
        local lastHP = {}
        _labConn = RunService.Heartbeat:Connect(function()
            if not Config.RageLab then return end
            local safe = getSafePlayers() or {}
            for i = 1, #safe do
                local p = safe[i]
                if p ~= lp then
                    local c = p.Character
                    local hum = c and c:FindFirstChildOfClass("Humanoid")
                    if hum then
                        local h, last = hum.Health, lastHP[p]
                        if last and h < last - 0.5 then
                            State.RageDealtTotal = (State.RageDealtTotal or 0) + (last - h)
                            if p == State.Target then State.Hits = State.Hits + 1 end
                        end
                        lastHP[p] = h
                    else
                        lastHP[p] = nil
                    end
                end
            end
        end)
    end
    Rage._startLabPoll = startLabPoll

    -- Counter-cheat detector was ripped in the clean rewrite; the Lab still reads Rage._isCheater, so keep
    -- an inert stub (never flags anyone).
    Rage._isCheater = function() return false end

    -- ── LIFECYCLE ───────────────────────────────────────────────────────────
    function Rage.init()
        for _, p in ipairs(getSafePlayers()) do bump(p) end
        Players.PlayerAdded:Connect(function(p)
            bump(p)
            p.CharacterAdded:Connect(function() bump(p) end)
        end)
        for _, p in ipairs(getSafePlayers()) do
            p.CharacterAdded:Connect(function() bump(p) end)
        end
        Players.PlayerRemoving:Connect(function(p)
            State.RageCharTokens[p] = nil
        end)
        lp.CharacterAdded:Connect(function()
            bump(lp)
            State.RageRealCF = nil; State.RageRealChar = nil
            State.RageTarget = nil
        end)
        bump(lp)
        startLabPoll()
    end

    -- ════════════════════════════════════════════════════════════════════════
    -- POLAR RAGE — the LIVE-PROVEN Kicia-beater (7-0, re-confirmed 5-0). ONE behavior, no modes, no fire
    -- handcuffs. Every Heartbeat: teleport the replicated body POINT-BLANK onto the enemy's live HitboxHead
    -- (no lead, no offset — LOS + range + eye-body consistency come free from the point-blank teleport) and
    -- FIRE a burst of taps at the live head; when the enemy voids their head we restore our real pos and fire
    -- a proactive first-strike burst (blanks while voided, LANDS the frame they surface = the edge that wins).
    -- Deep-void hide ONLY when there is nobody to shoot (no target / low HP / reloading). Undetection is
    -- handled separately — this engine keeps the proven POWER: SIMPLER beat complex.
    --   PACKET SHAPE (the proven 7-0 recipe): char0 == char1 == EncodeCFrame(CFrame.new(eyePos, headPos)) with
    --   the eye STRAIGHT UP on the head (no separate muzzle => no vertical-look RightVector NaN, ray-origin
    --   exactly on the head), char2 = the live head instance, char3 = EncodeCFrame(CFrame.new(0,0,0)).
    -- MOVEMENT MODEL (path-separation): the desync is CFrame-ONLY, position-ONLY. The void/park is written at
    -- Heartbeat (post-physics, the LAST write before the engine network-samples the RootPart -> that is what
    -- replicates the desync). The real pose is restored TWICE per frame, both strictly UPSTREAM of the next
    -- Heartbeat park so neither can weaken a shot: at the Camera-1 render step (own screen renders normal
    -- every frame incl. firing) and at Stepped (pre-physics backstop — physics then integrates from the real
    -- spot = native movement, and a throttled/dropped render frame can never let the Heartbeat-top capture
    -- latch the park as our "real" pose). Both restores are UNCONDITIONAL: no skip-while-firing, no overlay,
    -- no input lock, no takeover. Velocity + every BodyMover + the slide-jump impulse are NEVER touched — we
    -- only write hrp.CFrame's position (keeping the game's rotation).
    local _rageConn, _rageCharConn = nil, nil
    local _rageStepConn = nil               -- pre-physics (Stepped) restore backstop
    local _rageRestoreName = "__lh_rage_restore"
    local _rageDiedConn = nil               -- per-life Humanoid.Died hook (knife-death -> suggest Orbit hint)
    local _shootEnum = nil                 -- cached StartShooting enum (lazy; the game isn't Ready at IIFE-init)
    local RAGE_MELEE_INTERVAL = 0.35       -- melee has no gun ShootCooldown; the natural-path floor interval
    -- Polar tap counts live inline (POLAR_TAPS_SANE=6 / POLAR_TAPS_VOID=4); APEX reads Config.RageTaps/RageTapsVoid.

    -- FIRE INTERVAL — kept for the NATURAL delivery path (07a encodeRageShot) only; Polar's forged path fires
    -- every frame with no rate gate. READ the gun's own ShootCooldown (READ-ONLY; never mutate the shared Info
    -- table). Manual override or the melee floor short-circuit the read.
    local function fireInterval()
        if (Config.RageFireRateOverride or 0) > 0 then return Config.RageFireRateOverride end
        local item = getEquippedItem(); if not item then return 0.07 end
        if Rage._itemIsMelee and Rage._itemIsMelee(item) then
            local ok, cd = pcall(function() return item.Info and item.Info.ShootCooldown end)
            if ok and type(cd) == "number" and cd > 0 then return math.max(cd, RAGE_MELEE_INTERVAL) end
            return RAGE_MELEE_INTERVAL
        end
        local ok, cd = pcall(function() return item.Info and item.Info.ShootCooldown end)
        if ok and type(cd) == "number" and cd > 0 then return cd end
        return 0.07
    end
    Rage._fireInterval = fireInterval

    -- PRECISION-SAFE deep void (magnitude 110k-140k). Y is POSITIVE-ONLY: above FallenPartsDestroyHeight's
    -- +50000 clamp so the hidden body is never swept by the fallen-parts kill plane (underground = terrain
    -- blocks LOS = a tell). Non-sane (isSanePos false) so the sane-guard never latches it, still float32
    -- sub-stud so physics integrates cleanly (why not +-2e9). HELD 4-8s per spot (Δpos~0 = a believable, rare
    -- reposition; a per-frame re-roll = an impossible position-vs-velocity delta = the movement flag). Cleared
    -- on respawn.
    local function voidCFrame()
        local now = tick()
        if (not State.RageVoidCF) or now >= (State.RageVoidNext or 0) then
            local function axis() return math.random(110000, 140000) * (math.random(0, 1) == 0 and -1 or 1) end
            State.RageVoidCF   = CFrame.new(axis(), math.random(110000, 140000), axis())
            State.RageVoidNext = now + 4 + math.random() * 4
        end
        return State.RageVoidCF
    end

    -- WEAPON READY (drives the void-vs-fire branch — the sole ammo/reload gate on the Polar path). Melee is
    -- always ready. Non-melee: not equipping, not reloading, ammo > 0.
    local function weaponReady(it)
        if not it then return false end
        local okE, equipping = pcall(function() return it:IsEquipping() end)
        if okE and equipping then return false end
        if Rage._itemIsMelee and Rage._itemIsMelee(it) then return true end
        if (it._reload_cooldown or 0) > tick() then return false end
        local okA, ammo = pcall(function() return it:Get("Ammo") end)
        return okA and type(ammo) == "number" and ammo > 0
    end

    -- Behind-the-front-arc flank point vs a katana (deflect) or riot shield (block) — both only guard the
    -- FRONT, so we get behind them. ORBIT/APEX ONLY: Polar always parks on the head (standalone geometry).
    -- These are the ONLY remaining rage raycasts and are intentional (they protect the backstab geometry, not
    -- gate the shot). Returns a behind-target point with verified LOS to the head, or nil (=> point-blank).
    local function flankPoint(tgt, hh)
        if not tgt or not tgt.Character or not hh or not isSanePos(hh.Position) then return nil end
        local katana = Config.AvoidDeflect and isKatana(tgt)
        local shield = Config.RageShieldBackstab and isRiotShield(tgt)
        if not (katana or shield) then return nil end
        local thrp = tgt.Character:FindFirstChild("HumanoidRootPart")
        if not thrp then return nil end
        -- the server's deflect/block arc tests the target's REPLICATED HRP CFrame (Katana:StartAiming sends
        -- no camData) = the CFrame we read here, so -LookVector is genuinely their front. Flatten to
        -- horizontal and bail on a degenerate near-vertical facing.
        local look = thrp.CFrame.LookVector
        look = Vector3.new(look.X, 0, look.Z)
        if look.Magnitude < 1e-3 then return nil end
        local inv    = -look.Unit
        local anchor = hh.Position   -- anchor on the LIVE head (a desyncing katana user voids HRP; head stays live)
        local kf     = killFloor()
        local dist   = shield and 2.5 or 3.0
        local ignore = { tgt.Character, lp.Character }
        local rp = RaycastParams.new(); rp.FilterType = Enum.RaycastFilterType.Exclude
        rp.FilterDescendantsInstances = ignore
        local flank = anchor + inv * dist
        local wr = workspace:Raycast(anchor, inv * dist, rp)   -- don't clip through a wall behind them
        if wr then flank = wr.Position - inv * 0.5 end
        flank = Vector3.new(flank.X, math.max(flank.Y, kf + 3), flank.Z)
        if not hasLOS(flank, hh.Position, ignore) then return nil end
        return flank
    end

    -- Point-blank park directly on the head (kill-floor-clamped).
    local function pointBlank(hh)
        local hp = hh.Position
        return Vector3.new(hp.X, math.max(hp.Y, killFloor() + 3), hp.Z)
    end

    -- ── POLAR FIRE ───────────────────────────────────────────────────────────
    -- Emit a burst of `taps` shots at the live head. FORGED path (default = the live-verified Polar): build +
    -- FireServer the UseItem packet `taps` times directly — the proven multi-tap burst (the server expands
    -- shotgun pellets from the single char2 entry; extra taps beyond the tuned 6/4 get rejected, so raising
    -- them doesn't help). char0 == char1 == enc (eye straight up on the head => ray-origin on the head, no
    -- RightVector NaN), char2 = live head, char3 = EncodeCFrame(CFrame.new(0,0,0)). NATURAL path (fallback,
    -- RageDirectFire=false): store this shot's solution and drive the game's own StartShooting once (07a
    -- encodeRageShot re-encodes + re-gates). This runs in an un-pcall'd Heartbeat, so every emit is pcall-wrapped.
    local function polarFire(eyePos, aimPos, hh, taps)
        local it = getEquippedItem()
        if not it then return end
        if Config.RageDirectFire then
            -- zero this weapon's PER-INSTANCE client cooldowns so the forged burst is never throttled
            -- client-side (the proven Polar recipe). FORGED-only: the natural path leaves the cooldowns
            -- intact so the real Gun.StartShooting self-throttles lf:Input to the weapon's legal ShootCooldown.
            -- Info.* is the SHARED table and is left untouched.
            pcall(function() it._shoot_cooldown = 0; it._shoot_cooldown_no_ammo = 0; it._last_shot = tick() - 1 end)
            if not (Rivals.Ready and Rivals.Enums and Rivals.Util) then return end
            if _shootEnum == nil then pcall(function() _shootEnum = Rivals.Enums:ToEnum("StartShooting") end) end
            if _shootEnum == nil then return end
            local okId, objId = pcall(function() return it:Get("ObjectID") end)
            if not okId or not objId then return end
            pcall(function()
                local enc = Rivals.Util:EncodeCFrame(CFrame.new(eyePos, aimPos))
                local c3  = Rivals.Util:EncodeCFrame(CFrame.new(0, 0, 0))
                local data = { [utf8.char(1)] = {
                    [utf8.char(0)] = enc, [utf8.char(1)] = enc,
                    [utf8.char(2)] = hh,  [utf8.char(3)] = c3,
                } }
                local remote = ReplicatedStorage.Remotes.Replication.Fighter.UseItem
                for _ = 1, taps do
                    remote:FireServer(objId, _shootEnum, data, nil)
                end
            end)
            State.Shots = State.Shots + taps
        else
            -- NATURAL: hand this shot's solution to encodeRageShot (07a), then drive the game's own StartShooting
            -- on an identity-2 thread. The natural pipeline fires one shot per its own cadence, so `taps` is not
            -- replayed here; encodeRageShot re-encodes + re-gates + counts State.Shots.
            -- PRE-GATE UPSTREAM (single-variable hygiene): only drive lf:Input when THIS shot is currently
            -- encodable AND the dial interval has elapsed — mirroring the 07a gates BEFORE the real
            -- Gun.StartShooting runs. oldStart runs first and the overwrite is in-place-or-nothing, so if we
            -- drove Input for a shot 07a will bail on, the game would ship its OWN camData (real camera pose vs
            -- the parked body = an eye-body-mismatched, likely-no-LOS packet — the suspected ban vector). The
            -- non-sane-head first-strike call (voided target) fails isSanePos here, so it never drives Input.
            -- 07a stays the backstop (it re-reads + re-gates + is the sole writer of State.RageLastFireTime).
            if not weaponReady(it) then return end
            if not hh or not hh.Parent or not isSanePos(hh.Position) then return end
            local hpos = hh.Position
            if (hpos - eyePos).Magnitude > (400 - 5) then return end
            local ignore = { lp.Character }
            local tgt = State.RageTarget or State.Target
            if tgt and tgt.Character then ignore[2] = tgt.Character end
            if not hasLOS(eyePos, hpos, ignore) then return end
            local interval = fireInterval()
            if tick() - (State.RageLastFireTime or 0) < interval then return end
            -- >legal increment phase: defeat the gun cooldown so the dial interval binds; LEGAL (override=0)
            -- leaves it to self-throttle. The gun's equip/reload/ammo throttle is re-checked by 07a on emit.
            if (Config.RageFireRateOverride or 0) > 0 then
                pcall(function() it._shoot_cooldown = 0; it._shoot_cooldown_no_ammo = 0; it._last_shot = tick() - 1 end)
            end
            State.RageFireFromPos = eyePos
            State.RageFireAimPos  = aimPos
            State.RageFireHitPart = hh
            local lf = Rivals.Fighter and Rivals.Fighter.LocalFighter
            if not lf then return end
            task.spawn(function()
                if setthreadidentity then setthreadidentity(2) elseif setidentity then setidentity(2) end
                pcall(function() lf:Input("StartShooting") end)
                if setthreadidentity then setthreadidentity(8) elseif setidentity then setidentity(8) end
            end)
        end
    end

    -- ── DEEP-VOID HIDE ───────────────────────────────────────────────────────
    -- We deep-void ONLY when there is nobody to shoot (no/low-HP/reloading). RageVoidHide=false keeps us at our
    -- REAL position between engagements (the render restore holds us home = a normal-looking player);
    -- =true teleports to the deep void. We NEVER self-void during a live engagement.
    local function hide(hrp, status)
        State.RageStatus = status
        State.RageFiring = false
        if Config.RageVoidHide then
            State.RageVoidActive = true
            pcall(function() hrp.CFrame = voidCFrame() end)
        else
            State.RageVoidActive = false
        end
    end

    -- ── MODE TICKS (register-relief sub-IIFE) ────────────────────────────────
    -- The three mode ticks + Apex layer helpers live in their OWN proto so they don't add to the outer rage
    -- IIFE's 190-local budget. They read the shared park/fire/void helpers above as upvalues (upvalues don't
    -- count against this proto's budget) and export the dispatcher onto Rage._rageTick for the Heartbeat driver.
    -- All modes fire ONLY through polarFire (the RageDirectFire forged/natural swap seam) — never remote:FireServer.
    ;(function()
        -- ── POLAR ENGAGE (proven Kicia-beater, byte-identical to the shipped behavior) ──
        -- Deep-void ONLY when we can't/shouldn't fire; the instant we have a live target we teleport point-blank
        -- onto its head and open fire — no warmup, no rate gate, no self-void mid-engagement. When the enemy voids
        -- their head we surface at real and fire a proactive first-strike burst so a shot lands the frame they
        -- resurface (char2 self-resolves server-side). This is the DEFAULT mode and must stay unchanged.
        local function polarTick(ch, hrp, tgt)
            -- (a) self-protect (low HP)
            local hum = ch:FindFirstChildOfClass("Humanoid")
            if Config.RageSelfProtectHP and Config.RageSelfProtectHP > 0 and hum
               and hum.Health < hum.MaxHealth * Config.RageSelfProtectHP then
                return hide(hrp, "Self-protect")
            end
            -- (b) no target
            if not tgt or not tgt.Character then return hide(hrp, "No target") end
            -- (c) immune/protected target (honors RageSkipImmune, matching isValidTarget @ 05_utils)
            if Config.RageSkipImmune and isProtected(tgt) then return hide(hrp, "Hiding") end
            -- (d) weapon not ready -> economy (switch / reload) then hide
            local it = getEquippedItem()
            if not weaponReady(it) then
                if Config.RageOutOfAmmo == "Switch" then
                    local slot = Rage._nextUsableSlot and Rage._nextUsableSlot(nil)
                    if slot and Rage._equipSlot then Rage._equipSlot(slot) end
                else
                    -- reload once per empty->reload transition (rate-limited; a per-frame Input was a spam tell)
                    local now = tick()
                    if now - (State.RageReloadLast or 0) > 0.5 then
                        State.RageReloadLast = now
                        local lf = Rivals.Fighter and Rivals.Fighter.LocalFighter
                        if lf then pcall(function() lf:Input("StartReloading") end) end
                    end
                end
                return hide(hrp, "Reloading")
            end
            local hh = tgt.Character:FindFirstChild("HitboxHead") or tgt.Character:FindFirstChild("Head")
            if not hh then return hide(hrp, "Hiding") end
            State.RageFiring = true
            State.RageVoidActive = false
            local hpos = hh.Position
            if isSanePos(hpos) then
                -- HEAD SANE: teleport POINT-BLANK onto the head + fire 6 taps. NO lead, NO offset and NO flank
                -- (the proven standalone always parks on the head) — the point-blank teleport gives LOS, range
                -- and eye-body consistency for free. Orbit/Apex keep flankPoint; Polar's geometry stays proven.
                local park = pointBlank(hh)
                if posIsOOB(hpos) or posIsOOB(park) or not isSanePos(park) then
                    -- kill-plane bait guard (protects US, not a shot gate): a head/park in a kill volume => hide.
                    State.RageFiring = false
                    return hide(hrp, "Hiding")
                end
                pcall(function() hrp.CFrame = CFrame.new(park) end)   -- replicate point-blank on the enemy
                -- eye STRAIGHT UP on the park (char0==char1), aim AT the head. char2 = live hh, char3 = 0.
                polarFire(park + Vector3.new(0, 2.5, 0), hpos, hh, 6)   -- 6 taps onto a visible head (tuned; extra rejected)
                State.RageStatus = "Attacking"
                pcall(Visuals.notifyTarget, tgt)
            else
                -- HEAD VOIDED: fire a proactive first-strike burst (4 taps) from our CURRENT surfaced body pos.
                -- Both restores ran upstream this frame, so hrp.Position is the REAL pose (+ physics delta) —
                -- the same pose the standalone fires its void branch from. Blanks while the enemy is voided;
                -- LANDS the frame they surface (char2 = the live hh instance self-resolves server-side) = the
                -- edge that wins. We do NOT self-void during the engagement. The DECLARED aim is our own
                -- forward look (NOT the non-sane hpos — a CFrame.new(eye, nonSanePos) lookAt degenerates).
                State.RageVoidActive = false
                local myPos = hrp.Position
                polarFire(myPos + Vector3.new(0, 2, 0), myPos + hrp.CFrame.LookVector * 8, hh, 4)   -- 4-tap first-strike
                State.RageStatus = "First-strike"
                pcall(Visuals.notifyTarget, tgt)
            end
        end

        -- ── ORBIT (stepped side-vantage rager, effective vs normal/legit players) ──
        -- Holds one LOS-clear vantage for a short dwell so replication settles + the burst lands, then jumps a
        -- golden angle the enemy isn't pre-aiming. NEVER fires from the void; no legal vantage => deep-void hide
        -- (unhittable). Fires through polarFire (honors RageDirectFire). Knife enemies push the ring farther out.
        local function orbitVantage(aimPos, ignore, kf, knife)
            State.OrbitAngle = ((State.OrbitAngle or 0) + 2.39996) % (math.pi * 2)   -- golden-angle advance per repick
            local base = Config.RageCombatOrbitRadius or 60
            local radii
            if knife then
                radii = { math.max(base, 75), 95 }       -- knife enemy: push the ring out past their walk-back reach
            else
                radii = { base, base * 0.6, base * 1.5 }
            end
            for _, r in ipairs(radii) do
                for i = 0, 5 do
                    local ang    = State.OrbitAngle + i * (math.pi / 3)
                    local jitter = 0
                    if Config.RageCombatOrbitJitter then jitter = (math.random() - 0.5) * 14 end
                    local h = (Config.RageCombatOrbitHeight or 8) + jitter
                    local pos = Vector3.new(aimPos.X + math.cos(ang) * r,
                        math.max(aimPos.Y + h, kf + 6),
                        aimPos.Z + math.sin(ang) * r)
                    if isSanePos(pos) and not posIsOOB(pos) and hasLOS(pos, aimPos, ignore) then
                        return pos
                    end
                end
            end
            return nil
        end

        local function orbitTick(ch, hrp, tgt)
            local kf = killFloor()
            -- guard ladder: every "can't fire" case deep-voids (unhittable), never exposes at real
            if not tgt or not tgt.Character then
                State.RageStatus = "No target"; State.OrbitVantage = nil
                pcall(function() hrp.CFrame = voidCFrame() end)
                return
            end
            local tc   = tgt.Character
            local thrp = tc:FindFirstChild("HumanoidRootPart")
            local hh   = tc:FindFirstChild("HitboxHead") or tc:FindFirstChild("Head")
            if not hh or not isSanePos(hh.Position) then
                State.RageStatus = "Hiding"; State.OrbitVantage = nil
                pcall(function() hrp.CFrame = voidCFrame() end)
                return
            end
            if not weaponReady(getEquippedItem()) then
                State.RageStatus = "Reloading"; State.OrbitVantage = nil
                pcall(function() hrp.CFrame = voidCFrame() end)
                return
            end
            local aimPos = hh.Position
            if posIsOOB(aimPos) or aimPos.Y < kf + 1 then   -- kill-plane bait guard
                State.RageStatus = "Hiding"; State.OrbitVantage = nil
                pcall(function() hrp.CFrame = voidCFrame() end)
                return
            end
            local ignore = { tc, lp.Character }
            local vantage, status = nil, nil

            -- melee flank (katana/riot-shield) or our-knife backstab
            local flank = flankPoint(tgt, hh)
            if not flank and thrp and Config.RageKnifeBackstab and isLocalKnife() then
                local inv = -thrp.CFrame.LookVector
                local rp = RaycastParams.new()
                rp.FilterType = Enum.RaycastFilterType.Exclude
                rp.FilterDescendantsInstances = ignore
                local f  = thrp.Position + inv * 3.0
                local wr = workspace:Raycast(thrp.Position, inv * 3.0, rp)
                if wr then f = wr.Position - inv * 0.5 end
                local cand = Vector3.new(f.X, math.max(f.Y, kf + 3), f.Z)
                if isSanePos(cand) and not posIsOOB(cand) and hasLOS(cand, hh.Position, ignore) then
                    flank = cand
                end
            end

            if flank then
                vantage = flank
                State.OrbitVantage = nil
                if isRiotShield(tgt) then
                    status = "Anti-riot"
                elseif isKatana(tgt) then
                    status = "Katana flank"
                else
                    status = "Backstab"
                end
            else
                local knife = isEnemyKnife(tgt)
                local held  = State.OrbitVantage
                if (not held) or tick() >= (State.OrbitVantageUntil or 0) or not hasLOS(held, aimPos, ignore) then
                    local v = orbitVantage(aimPos, ignore, kf, knife)
                    if v then
                        State.OrbitVantage = v
                        State.OrbitVantageUntil = tick() + (Config.RageOrbitDwell or 0.09)
                        held = v
                    elseif held and hasLOS(held, aimPos, ignore) then
                        State.OrbitVantageUntil = tick() + (Config.RageOrbitDwell or 0.09)
                    else
                        held = nil
                    end
                end
                if held then
                    vantage = held
                    if knife then status = "Orbit (kept dist)" else status = "Orbit" end
                end
            end

            if not vantage or not isSanePos(vantage) or posIsOOB(vantage) then
                State.RageStatus = "Orbit (hiding)"
                State.OrbitVantage = nil; State.OrbitVantageUntil = 0
                pcall(function() hrp.CFrame = voidCFrame() end)
                return
            end
            State.RageStatus = status or "Orbit"
            State.RageFiring = true; State.RageVoidActive = false
            pcall(function() hrp.CFrame = CFrame.new(vantage) end)
            local eye = vantage + Vector3.new(0, Config.RagePBEyeUp or 3, 0)
            polarFire(eye, aimPos, hh, 6)   -- taps cosmetic on the forged path; honors RageDirectFire
            pcall(Visuals.notifyTarget, tgt)
        end

        -- ── APEX hittability primitive (reused by apexTick; NOT prediction) ──

        -- Track the target's last SANE head position (the decoupled fire-eye anchor). Reset the cache on a target
        -- switch so a stale enemy's head never anchors a new engagement. Returns the cached sane pos or nil.
        local function updateSaneHead(tgt, hh)
            if State.RageLastSaneHeadTgt ~= tgt then
                State.RageLastSaneHeadTgt = tgt
                State.RageLastSaneHeadPos = nil
            end
            if hh and isSanePos(hh.Position) then
                State.RageLastSaneHeadPos = hh.Position
            end
            return State.RageLastSaneHeadPos
        end

        -- Rate-limited physical reload (0.5s gate; shared with the Polar inline gate but factored here for APEX).
        local function reloadRateLimited()
            local now = tick()
            if now - (State.RageReloadLast or 0) <= 0.5 then return end
            State.RageReloadLast = now
            local lf = Rivals.Fighter and Rivals.Fighter.LocalFighter
            if lf then pcall(function() lf:Input("StartReloading") end) end
        end

        -- 5-arg fire wrapper: the ghost canary (Iron Law, machine-checked) then forward to the forged/natural
        -- seam. polarFire never sees hrp today, so we thread it in to police fire-from-void by construction.
        local function emitTaps(eyePos, aimPos, hh, taps, hrp)
            if hrp and not isSanePos(hrp.Position) then
                State.RageGhostCanary = (State.RageGhostCanary or 0) + 1   -- MUST read 0 every soak, forever
                return
            end
            polarFire(eyePos, aimPos, hh, taps)   -- polarFire unchanged: forged multi-tap OR natural seam + LOS choke
        end
        Rage._emitTaps = emitTaps

        -- ── APEX helpers (dedicated sub-IIFE: own 190-local proto budget, register-cap relief) ──
        -- liveHead / apexEconomy / apexAcquire. apexAcquire keeps a persistent voidFrames upvalue (closure).
        ;(function()
            -- liveHead(tgt) -> hh, headSane. Grace cache across a transient FindFirstChild miss.
            local cacheInst, cacheMiss = nil, 0
            local function liveHead(tgt)
                local c  = tgt.Character
                local hh = c and (c:FindFirstChild("HitboxHead") or c:FindFirstChild("Head"))
                if hh then
                    cacheInst = hh
                    cacheMiss = 0
                elseif cacheInst and cacheInst.Parent ~= nil and cacheMiss < (Config.RageHeadMissGrace or 2) then
                    hh = cacheInst
                    cacheMiss = cacheMiss + 1
                else
                    cacheInst = nil
                    return nil, false
                end
                local sane = hh.Parent ~= nil and isSanePos(hh.Position)
                return hh, sane
            end
            Rage._liveHead = liveHead

            -- apexEconomy(headSane) -> "fire" | "hotswap" | "reload_now" | "reload_lull". One equipped-item read
            -- per tick. Pre-emptive fire-then-swap queues State.RagePendingSwap at the ammo watermark (equipped
            -- AFTER this frame's taps in apexTick). Out of ammo: switch (same-frame recovery), else reload.
            local function apexEconomy(headSane)
                local it = getEquippedItem()
                if weaponReady(it) then
                    local a = nil
                    pcall(function() a = it:Get("Ammo") end)
                    if type(a) == "number" and a <= (Config.RageSwapAmmoWatermark or 0) and a > 0 then
                        local cur  = nil
                        if Rage._whichSlotNow then cur = Rage._whichSlotNow() end
                        local slot = Rage._nextUsableSlot and Rage._nextUsableSlot(cur)
                        if slot then State.RagePendingSwap = slot end
                    end
                    return "fire"
                end
                local cur  = nil
                if Rage._whichSlotNow then cur = Rage._whichSlotNow() end
                local slot = Rage._nextUsableSlot and Rage._nextUsableSlot(cur)
                if Config.RageOutOfAmmo == "Switch" and slot then
                    if Rage._equipSlot then Rage._equipSlot(slot) end
                    local it2 = getEquippedItem()
                    if weaponReady(it2) then return "fire" end   -- same-frame recovery: zero gap
                    return "hotswap"
                end
                if headSane then return "reload_now" end   -- dry + target sane: eat the gap (hard-strand fallback)
                return "reload_lull"                       -- target voided: the free lull
            end
            Rage._apexEconomy = apexEconomy

            -- apexAcquire: sticky (Polar-identical hard-invalidity), head-sane swing under a dwell guard.
            local apexAcquire = (function()
                local voidFrames = 0
                return function()
                    local cur = State.RageTarget
                    if cur and (not cur.Parent or not cur.Character or not isAlive(cur)
                        or (Config.TeamCheck and isTeammate(cur))) then
                        cur = nil
                    end
                    local curSane = false
                    if cur then
                        local hh = liveHead(cur)
                        if hh and isSanePos(hh.Position) then curSane = true end
                        if curSane then voidFrames = 0 else voidFrames = voidFrames + 1 end
                    end
                    if Config.RageHeadSaneFirst and cur and not curSane
                       and voidFrames > (Config.RageTargetDwell or 8) then
                        local alt = Rage._findTargetHeadSane()
                        if alt and alt ~= cur then
                            cur = alt
                            voidFrames = 0
                        end
                    end
                    if not cur then
                        cur = Rage._findTargetHeadSane()
                        voidFrames = 0
                    end
                    State.RageTarget = cur
                    return cur
                end
            end)()
            Rage._apexAcquire = apexAcquire
        end)()

        -- ── APEX ENGAGE (strict-superset Polar) ──
        -- Polar's point-blank multi-tap engine + three REAL speed knobs: in-engine same-frame acquire (driver),
        -- head-sane target swing (apexAcquire/findTargetHeadSane), and fire-then-swap economy (apexEconomy). No
        -- prediction, no duty-cycled void ghost: every fire leaves from a surfaced, LOS-gated body and the ghost
        -- canary (emitTaps) machine-checks the Iron Law. Helpers live in the Rage._apex* sub-IIFE (register relief).
        local function apexTick(ch, hrp, tgt)
            -- (a) self-protect (low HP)
            local hum = ch:FindFirstChildOfClass("Humanoid")
            if Config.RageSelfProtectHP and Config.RageSelfProtectHP > 0 and hum
               and hum.Health < hum.MaxHealth * Config.RageSelfProtectHP then
                return hide(hrp, "Self-protect")
            end
            if not tgt or not tgt.Character then return hide(hrp, "No target") end
            if Config.RageSkipImmune and isProtected(tgt) then return hide(hrp, "Hiding") end

            -- head instance with grace cache (a transient FindFirstChild miss must not blank the frame)
            local hh, headSane = Rage._liveHead(tgt)   -- headSane = hh live + isSanePos(hh.Position) THIS frame
            -- ECONOMY: "fire" | "hotswap" | "reload_now" | "reload_lull"; may queue State.RagePendingSwap
            local econ = Rage._apexEconomy(headSane)
            if econ == "hotswap" then return hide(hrp, "Hot-swap") end
            if econ == "reload_now" then
                reloadRateLimited()
                return hide(hrp, "Reloading")
            end
            if econ == "reload_lull" then
                if not Config.RageReloadOnVoidOnly then reloadRateLimited() end   -- else defer to a void lull
                return hide(hrp, "Reloading")
            end
            if not hh then return hide(hrp, "Hiding") end   -- grace exhausted, no head instance

            State.ParkGen = (State.ParkGen or 0) + 1   -- generation token (tap-spread guard scaffold; cheap)
            State.RageFiring = true
            State.RageVoidActive = false
            updateSaneHead(tgt, hh)
            local hpos = hh.Position

            if headSane then
                local flank = flankPoint(tgt, hh)
                local park  = flank or pointBlank(hh)
                if posIsOOB(hpos) then
                    return hide(hrp, "Hiding")   -- their head in a kill volume: protect us, skip
                end
                if posIsOOB(park) or not isSanePos(park) then
                    -- OOB-park recovery: don't blank; fire eye-origin from the REAL surfaced body, LOS-gated.
                    -- A coverage/exposure trade vs Polar's hide — the body is sane, so Iron-Law legal.
                    local eye = hrp.Position + Vector3.new(0, 2, 0)
                    Rage._emitTaps(eye, hpos, hh, Config.RageTaps, hrp)
                    State.RageStatus = "OOB-fire"
                else
                    pcall(function() hrp.CFrame = CFrame.new(park) end)   -- SURFACED point-blank
                    local eye = park + Vector3.new(0, 2.5, 0)
                    Rage._emitTaps(eye, hpos, hh, Config.RageTaps, hrp)
                    if not flank then
                        State.RageStatus = "Apex"
                    elseif isRiotShield(tgt) then
                        State.RageStatus = "Anti-riot"
                    else
                        State.RageStatus = "Katana flank"
                    end
                end
                pcall(Visuals.notifyTarget, tgt)
            else
                -- FIRST-STRIKE, ghost-proof by construction: re-establish the real surfaced pose, then fire from
                -- hrp.Position (never a stale voided pose). char2=hh self-resolves the frame they surface. The
                -- driver's no-anchor guard keeps State.RageRealCF non-nil here; the CFrame write is pcall'd anyway.
                pcall(function() hrp.CFrame = CFrame.new(State.RageRealCF.Position) * hrp.CFrame.Rotation end)
                local myPos = hrp.Position
                local eye   = myPos + Vector3.new(0, 2, 0)
                local aim   = myPos + hrp.CFrame.LookVector * 8
                Rage._emitTaps(eye, aim, hh, Config.RageTapsVoid, hrp)   -- K, not 4 (inert if the sweep gives K=4)
                State.RageStatus = "First-strike"
                pcall(Visuals.notifyTarget, tgt)
            end

            -- fire-then-swap: kick the queued equip AFTER this frame's taps (equipping first flips IsEquipping -> blanks)
            if State.RagePendingSwap then
                if Rage._equipSlot then Rage._equipSlot(State.RagePendingSwap) end
                State.RagePendingSwap = nil
            end
        end

        -- ── DISPATCH ──
        local function rageTick(ch, hrp, tgt)
            local mode = Config.RageMode or "Polar"
            if mode == "Apex" then
                apexTick(ch, hrp, tgt)
            elseif mode == "Orbit" then
                orbitTick(ch, hrp, tgt)
            else
                -- Polar AND Phantom: identical fire path (Phantom adds a SEPARATE detach step at the driver,
                -- never inside a tick). polarTick runs unmodified so Polar stays byte-identical.
                polarTick(ch, hrp, tgt)
            end
        end
        Rage._rageTick = rageTick
    end)()

    -- ── PHANTOM DETACHED-HITBOX (own sub-IIFE: fresh proto budget, register-cap relief) ──────────
    -- Keep HRP + visible rig point-blank (Polar fire, eye-body gate passes) while our OWN EntityHitbox parts
    -- are weld-severed and CFrame-flung >400 studs out, jittered, every Heartbeat -> enemy damage rays
    -- (MAX_RAYCAST=400) miss us. Sever via weld.Enabled=false (NEVER BreakJoints -> ragdoll). cloneref the
    -- cached JOINTS (long-held cross-frame refs), collect PARTS as raw refs. Iron-Law canary: never fling from
    -- a non-sane HRP (that recreates the fire-from-void symmetric-death). pcall every joint/part write.
    ;(function()
        local HITBOX_NAMES    = { "HitboxHead", "HitboxHeadSmall", "HitboxBody", "HitboxBodySmall", "PhysicalHitboxHead" }
        local BODY_ONLY_NAMES = { "HitboxBody", "HitboxBodySmall" }

        -- Resolve our own hitbox parts as RAW refs (NOT cloneref'd — cloneref'ing the part then comparing
        -- identity with weld.Part0/Part1 was the probe bug that severed zero). Returns an array of BaseParts.
        local function ownHitboxParts(ch)
            local names = HITBOX_NAMES
            if Config.RageDetachBodyOnly then
                names = BODY_ONLY_NAMES
            end
            local parts = {}
            for _, name in ipairs(names) do
                local p = ch:FindFirstChild(name)
                if p and p:IsA("BasePart") then
                    parts[#parts + 1] = p
                end
            end
            return parts
        end
        Rage._ownHitboxParts = ownHitboxParts

        -- Sever ONCE per life: each part's child WeldConstraint (Part0=HRP, Part1=hitbox) -> Enabled=false;
        -- cache { joint=cloneref(weld), wasEnabled=<prior>, part=<raw> }. Idempotent via the RageDetachChar token.
        local function ensureSevered(ch, hrp)
            if State.RageDetachWelds and State.RageDetachChar == ch then
                return
            end
            local cache = {}
            for _, part in ipairs(ownHitboxParts(ch)) do
                for _, j in ipairs(part:GetChildren()) do
                    if j:IsA("WeldConstraint") and (j.Part0 == hrp or j.Part1 == hrp) then
                        local was = j.Enabled
                        pcall(function() j.Enabled = false end)
                        cache[#cache + 1] = { joint = cloneref(j), wasEnabled = was, part = part }
                    end
                end
            end
            State.RageDetachWelds = cache
            State.RageDetachChar  = ch
        end

        -- Per-Heartbeat fling. Iron-Law canary FIRST: a non-sane HRP means we would fire-from-void -> ++ and bail.
        -- Gate (mode==Phantom AND engaged: firing or a live target) is enforced at the call site.
        local function detachStep(ch, hrp)
            if not isSanePos(hrp.Position) then
                State.RageDetachCanary = State.RageDetachCanary + 1   -- MUST read 0 every soak, forever
                return
            end
            ensureSevered(ch, hrp)
            local cache = State.RageDetachWelds
            if not cache then
                return
            end
            local base = hrp.Position
            local dist = Config.RageDetachDist or 500
            for _, ent in ipairs(cache) do
                local part = ent.part
                if part and part.Parent then
                    local jx, jz = 0, 0
                    if Config.RageDetachJitter then
                        jx = (math.random() - 0.5) * 8
                        jz = (math.random() - 0.5) * 8
                    end
                    -- positive-Y-biased, sub-map-plane-safe, always >= dist from HRP on every axis
                    local off = Vector3.new(dist + jx, dist + 200, dist + jz)
                    pcall(function() part.CFrame = CFrame.new(base + off) end)
                end
            end
        end
        Rage._detachStep = detachStep

        -- Idempotent restore: re-enable cached welds (Enabled = wasEnabled), snap parts home to HRP, null cache.
        local function detachRestore(ch)
            local cache = State.RageDetachWelds
            if cache then
                local hrp = ch and ch:FindFirstChild("HumanoidRootPart")
                for _, ent in ipairs(cache) do
                    pcall(function() ent.joint.Enabled = ent.wasEnabled end)
                    if ent.part and ent.part.Parent and hrp and hrp.Parent then
                        pcall(function() ent.part.CFrame = hrp.CFrame end)   -- weld re-seats it next physics step
                    end
                end
            end
            State.RageDetachWelds = nil
            State.RageDetachChar  = nil
        end
        Rage._detachRestore = detachRestore
    end)()

    -- KNIFE-DEATH HINT: on our own death, if a knife/melee player killed us, suggest switching to Orbit (our
    -- knife counter). No attacker handle exists, so infer: engaged RageTarget is a knifer, else a knifer sits
    -- within melee range of our death pose. Rate-limited (>=90s) + skipped when already in Orbit. The whole body
    -- is pcall'd — it runs off Humanoid.Died (an un-pcall'd Roblox signal), and it FindFirstChilds into a
    -- possibly-dying character.
    local function onLocalDied()
        pcall(function()
            if not Config.Rage then
                return
            end
            if Config.RageMode == "Orbit" then   -- already using our knife counter
                return
            end
            if tick() - (State.RageKnifeHintLast or 0) < 90 then
                return
            end
            local knifed = false
            local tgt = State.RageTarget
            if tgt and tgt.Parent and isEnemyKnife(tgt) then
                knifed = true
            else
                -- prefer the captured real pose (RageRealCF): the live HRP may be parked on the enemy this frame
                local dpos = nil
                local real = State.RageRealCF
                if real then
                    dpos = real.Position
                else
                    local ch = lp.Character
                    local hrp = ch and ch:FindFirstChild("HumanoidRootPart")
                    if hrp then
                        dpos = hrp.Position
                    end
                end
                if dpos then
                    for _, plr in ipairs(getSafePlayers()) do
                        if plr ~= lp and not (Config.TeamCheck and isTeammate(plr)) then
                            local r = plr.Character and plr.Character:FindFirstChild("HumanoidRootPart")
                            if r and (r.Position - dpos).Magnitude <= 18 and isEnemyKnife(plr) then
                                knifed = true
                                break
                            end
                        end
                    end
                end
            end
            if not knifed then
                return
            end
            State.RageKnifeHintLast = tick()
            local lib = _G["\76\72"]   -- the Linoria Library, published at load (13_gui)
            if lib and lib.Notify then
                pcall(function() lib:Notify("Knifed by a melee player — switch Rage mode to Orbit (our knife counter)", 5) end)
            end
        end)
    end

    local function hookDied(char)
        if _rageDiedConn then
            _rageDiedConn:Disconnect()
            _rageDiedConn = nil
        end
        if not char then
            return
        end
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum then
            _rageDiedConn = hum.Died:Connect(onLocalDied)
        end
    end

    -- ── THE ONE RESTORE PRIMITIVE ────────────────────────────────────────────
    -- Position-only homing write, shared by the render bind, the Stepped backstop and EVERY teardown path.
    -- CFrame ONLY: the game applies slide-jump as a BodyVelocity impulse, so writing AssemblyLinearVelocity /
    -- Velocity / RotVelocity (or touching a BodyMover) eats that impulse — a real shipped movement bug. The
    -- per-life token (RageRealChar) stops a freshly respawned body being yanked to a dead body's pose.
    -- Rotation is the game's, never ours. `pin` un-sticks a GROUNDED Freefall so jump/slide state can't churn.
    local function restoreHome(pin)
        local real = State.RageRealCF
        local ch   = lp.Character
        if not real or not ch or State.RageRealChar ~= ch then return end
        local hrp = ch:FindFirstChild("HumanoidRootPart")
        if not hrp or not hrp.Parent then return end
        pcall(function()
            hrp.CFrame = CFrame.new(real.Position) * hrp.CFrame.Rotation
            if pin then
                local hu = ch:FindFirstChildOfClass("Humanoid")
                if hu and hu:GetState() == Enum.HumanoidStateType.Freefall and hu.FloorMaterial ~= Enum.Material.Air then
                    hu:ChangeState(Enum.HumanoidStateType.Running)
                end
            end
        end)
    end

    local function startRage()
        if _rageConn then return end
        hookDied(lp.Character)   -- knife-death Orbit hint (per-life; re-hooked on respawn below)
        -- CLEAR-ON-RESPAWN: drop the captured pose (+ its per-life token) so a freshly-spawned body is never
        -- yanked to a stale pre-death pose. The no-anchor guard in the driver holds us until this life
        -- re-captures a sane pose at the top of the next Heartbeat.
        if not _rageCharConn then
            _rageCharConn = lp.CharacterAdded:Connect(function()
                State.RageRealCF = nil; State.RageRealChar = nil
                State.RageVoidCF = nil
                -- PHANTOM: null the stale weld cache (new body has fresh welds; the old joints died with the old
                -- body). Do NOT restore (the cached joints are gone) and do NOT reset RageDetachCanary (cumulative).
                State.RageDetachWelds = nil; State.RageDetachChar = nil
                hookDied(lp.Character)   -- re-hook the knife-death Orbit hint on the fresh Humanoid
            end)
        end
        -- CLIENT-CLEAN RESTORE (ALL modes), purely client-side and UNCONDITIONAL on both anchors: no
        -- skip-while-firing, no overlay, no input lock, no takeover. The Heartbeat park is the last write
        -- before the network sample, so it always re-parks after these — they cannot cost a single shot.
        local function rageRestoreActive()
            if not Config.Rage then return false end
            if not State.RageInMatch then return false end
            if not State.RageRealCF or State.RageRealChar ~= lp.Character then return false end
            return true
        end
        -- (1) RENDER, at Camera-1 so the camera update at Camera reads the RESTORED pose (binding after Camera
        -- would show a one-frame snap to the park). The fixed bind name lets a re-executed script kill a stale
        -- bind from a previous instance by name — do NOT randomize it.
        pcall(function() RunService:UnbindFromRenderStep(_rageRestoreName) end)
        RunService:BindToRenderStep(_rageRestoreName, Enum.RenderPriority.Camera.Value - 1, function()
            if rageRestoreActive() then restoreHome(true) end
        end)
        -- (2) PRE-PHYSICS BACKSTOP. Stepped is 1:1 with Heartbeat; RENDER is the one phase that can be
        -- throttled or dropped, and a single skipped render frame would let the Heartbeat-top capture latch
        -- the PARK as our real pose (a park on a head IS sane, so the sane-guard would not catch it) and
        -- "real" would permanently become the enemy. Restoring here also guarantees physics integrates from
        -- the real pose = native movement. Strictly UPSTREAM of the same frame's Heartbeat park.
        if not _rageStepConn then
            _rageStepConn = RunService.Stepped:Connect(function()
                if rageRestoreActive() then restoreHome(true) end
            end)
        end
        -- HEARTBEAT DRIVER (post-physics: this is what the engine network-samples -> what replicates).
        _rageConn = RunService.Heartbeat:Connect(function()
            if not Config.Rage then
                -- restore-before-teardown (a direct Config.Rage=false that skips Rage.disable): welds back
                -- first, then home the body, THEN drop every anchor — never strand at a park or in the void.
                if lp.Character then Rage._detachRestore(lp.Character) end   -- PHANTOM teardown on a direct Config.Rage=false
                restoreHome(false)
                pcall(function() RunService:UnbindFromRenderStep(_rageRestoreName) end)
                if _rageConn then _rageConn:Disconnect(); _rageConn = nil end
                if _rageStepConn then _rageStepConn:Disconnect(); _rageStepConn = nil end
                if _rageCharConn then _rageCharConn:Disconnect(); _rageCharConn = nil end
                if _rageDiedConn then _rageDiedConn:Disconnect(); _rageDiedConn = nil end
                State.RageRealCF = nil; State.RageRealChar = nil
                State.RageTarget = nil; State.RageVoidCF = nil
                State.RageInMatch = false
                State.OrbitVantage = nil; State.OrbitVantageUntil = 0
                State.RageLastSaneHeadPos = nil; State.RageLastSaneHeadTgt = nil
                State.RagePendingSwap = nil; State.ParkGen = 0
                return
            end
            local ch  = lp.Character
            local hrp = ch and ch:FindFirstChild("HumanoidRootPart")
            if not hrp or not hrp.Parent then return end
            -- capture the real pose (adopts WASD/dash/legit teleports) every sane frame. This runs at the TOP of
            -- the Heartbeat, BEFORE rageTick parks the body, so capture reads the real pose the client restore
            -- last wrote — never the park. Position only; velocity is never captured or restored.
            if isSanePos(hrp.Position) then
                State.RageRealCF = hrp.CFrame
                State.RageRealChar = ch
            end
            -- lobby gate: do nothing outside a match (no teleport onto people in the hub). Home ONLY from a
            -- non-sane pose — that can only be our own void write, so this can never fight a legitimate
            -- game-side teleport to the lobby.
            State.RageInMatch = inMatch()
            if not State.RageInMatch then
                if not isSanePos(hrp.Position) then restoreHome(false) end
                State.RageTarget = nil
                State.RageStatus = "Lobby"
                State.RageFiring = false
                return
            end
            -- no-anchor guard: never displace unless THIS life has a captured real pose to restore from.
            if not State.RageRealCF or State.RageRealChar ~= ch then
                State.RageStatus = "Waiting"
                State.RageFiring = false
                return
            end
            -- corpse guard: never park/void/fire while dead. A dead fighter cannot legally send UseItem and a
            -- point-blank teleport of a ragdolling body replicates as impossible movement. Home from the void.
            local hum0 = ch:FindFirstChildOfClass("Humanoid")
            if hum0 and hum0.Health <= 0 then
                if not isSanePos(hrp.Position) then restoreHome(false) end
                State.RageFiring = false
                State.RageVoidActive = false
                State.RageStatus = "Dead"
                return
            end
            -- ACQUIRE (APEX-scoped in-engine, same Heartbeat as the fire = no cross-loop stale). Polar/Orbit keep
            -- the sticky path untouched so Polar stays byte-identical; startTargetLoop still fills
            -- RageTrueVelocityMap + State.Target (the silent-aim lead fallback) regardless.
            local tgt = State.RageTarget
            if Config.RageMode == "Apex" and Config.RageInEngineAcquire then
                tgt = Rage._apexAcquire()   -- sticky + head-sane swing + dwell; sets State.RageTarget internally
            else
                if tgt and (not tgt.Parent or not tgt.Character or not isAlive(tgt)
                    or (Config.TeamCheck and isTeammate(tgt))) then
                    tgt = nil
                end
                if not tgt then tgt = Rage._findTarget() end
                State.RageTarget = tgt
            end
            Rage._rageTick(ch, hrp, tgt)   -- writes park/void at Heartbeat; the client restore renders between bursts
            -- PHANTOM: detach + fling our own hitbox parts alongside Polar's fire. The canary inside detachStep
            -- polices fire-from-void by construction. KillFast (default) flings ONLY during the lethal burst
            -- (RageFiring) and re-attaches between bursts, minimising the window we're un-hittable; with KillFast
            -- OFF we hold the fling through any grounded engaged frame (a live target), including the void lull.
            if Config.RageMode == "Phantom" then
                local engaged = State.RageFiring or (not Config.RageDetachKillFast and tgt ~= nil)
                if engaged then
                    Rage._detachStep(ch, hrp)
                elseif State.RageDetachWelds then
                    Rage._detachRestore(ch)
                end
            end
        end)
    end

    function Rage.enable()
        Config.Rage = true
        if Rage._startTargetLoop then Rage._startTargetLoop() end
        startRage()
    end

    function Rage.disable()
        Config.Rage = false
        pcall(function() RunService:UnbindFromRenderStep(_rageRestoreName) end)
        if _rageConn then _rageConn:Disconnect(); _rageConn = nil end
        if _rageStepConn then _rageStepConn:Disconnect(); _rageStepConn = nil end
        if _rageCharConn then _rageCharConn:Disconnect(); _rageCharConn = nil end
        if _rageDiedConn then _rageDiedConn:Disconnect(); _rageDiedConn = nil end
        if lp.Character then Rage._detachRestore(lp.Character) end   -- PHANTOM: re-enable welds + snap home BEFORE the HRP restore
        restoreHome(false)   -- home on EVERY exit (per-life token gated), never stranded at a park or in the void
        State.RageRealCF = nil; State.RageRealChar = nil
        State.RageTarget = nil; State.RageVoidCF = nil
        State.RageInMatch = false
        State.OrbitVantage = nil; State.OrbitVantageUntil = 0
        State.RageLastSaneHeadPos = nil; State.RageLastSaneHeadTgt = nil
        State.RagePendingSwap = nil; State.ParkGen = 0
        State.RageFiring = false
        State.RageVoidActive = false
        State.RageStatus = "Idle"
    end

    function Rage.unload()
        Rage.disable()
        -- _tgtConn / _labConn are 07a locals (same IIFE) — tear them down here so no standing loop survives.
        if _tgtConn then pcall(function() _tgtConn:Disconnect() end); _tgtConn = nil end
        if _labConn then pcall(function() _labConn:Disconnect() end); _labConn = nil end
    end

    Rage._encodeRageShot = encodeRageShot
end)()

-- ── GUN HOOK ───────────────────────────────────────────────────────────────

-- SILENT-AIM MULTIPOINT + TORSO FALLBACK: the server re-raycasts eye(camPos)->head, so a
-- head with no clear LOS silently misses. Instead of encoding a single clamped point, probe
-- a ring of candidate points around the chosen part; if none of the head's points are visible,
-- fall back to the other head hitboxes, then (optionally) the torso parts. Returns the first
-- BasePart the server should accept, or the primary (closest-to-crosshair) part as a last resort.
-- Reuses isVisible + HEAD_PARTS/TORSO_PARTS from 05_utils; the actual shot is still encoded by
-- Rage._encodeShot (which clamps + jitters within whichever part we return here).
local function silentPartVisible(part)
    if not part or not part:IsA("BasePart") then return false end
    if not Config.SilentAimMultipoint then return isVisible(part.Position) end
    if isVisible(part.Position) then return true end
    local n = math.max(1, math.floor(Config.SilentAimMultipointCount or 5))
    local r = math.min(part.Size.X, part.Size.Y, part.Size.Z) * 0.4
    for i = 1, n do
        local a   = (i / n) * math.pi * 2
        local off = Vector3.new(math.cos(a) * r, ((i % 2 == 0) and 0.5 or -0.5) * r, math.sin(a) * r)
        if isVisible(part.Position + off) then return true end
    end
    return false
end
local function pickSilentPart(char, primary)
    if not char then return primary end
    if silentPartVisible(primary) then return primary end
    for _, name in ipairs(HEAD_PARTS) do
        local p = char:FindFirstChild(name)
        if p and p ~= primary and silentPartVisible(p) then return p end
    end
    if Config.SilentAimTorsoFallback then
        for _, name in ipairs(TORSO_PARTS) do
            local p = char:FindFirstChild(name)
            if p and silentPartVisible(p) then return p end
        end
    end
    return primary   -- nothing clearly visible; fire the crosshair-closest part anyway
end

-- ── GUN HOOK ───────────────────────────────────────────────────────────────
-- NOTE: assigns the `local hookGunModule` forward-declared in 01_bootstrap.lua (do NOT re-`local` it,
-- or the CharacterAdded respawn re-hook there would capture a separate nil upvalue).
function hookGunModule()
    if not Rivals.Ready or not Rivals.Gun or shared._gunHooked == game then return end
    shared._gunHooked = game
    
    local oldStart = Rivals.Gun.StartShooting
    shared._LH_GunOrig = oldStart   -- save the TRUE original so Unload can restore it (prevents a stale hook
                                    -- from a re-executed/unloaded instance from breaking guns)
    local hookWrapper = function(self, ...)
        -- FORGED-RAGE M1 SUPPRESSION: in forged rage (Config.Rage AND Config.RageDirectFire) the engine
        -- delivers the whole burst itself via UseItemRemote:FireServer (07b). The player's physical M1 must
        -- be INERT here to match the no-gun-hook standalone — running oldStart would fire a REAL shot in the
        -- frozen-camera direction (client tracer + impact decals on the wall + a real server shot). We short-
        -- circuit BEFORE oldStart, returning `false` = the game's own "no shot fired" return (identical to its
        -- cooldown/no-ammo blocked paths, emitted every frame), so nothing is mutated and the gun is untouched.
        -- Restore is instant: rage OFF (or RageDirectFire=false natural path) skips this and the very next call
        -- runs oldStart normally. Guarded to the local player; wrapped in pcall so it can NEVER brick the gun.
        local suppress = false
        pcall(function()
            if Config.Rage and Config.RageDirectFire and self.ClientFighter and self.ClientFighter.IsLocalPlayer then
                suppress = true
            end
        end)
        if suppress then return false end
        local results = { oldStart(self, ...) }   -- ALWAYS run the real fire first and keep its results
        -- Our shot logic must NEVER break the game's firing. Run it in pcall and always return the original
        -- results, so ANY error here (or a stale hook left behind by a re-execute) can't stop guns working.
        pcall(function()
            if not self.ClientFighter or not self.ClientFighter.IsLocalPlayer then return end
            local camData = results[3]
            if not camData or typeof(camData) ~= "table" then return end

            local camPos = Camera.CFrame.Position
            State.CamPos = camPos

            -- Rage path — Kernal engine encoding
            if Config.Rage then
                if Rage._encodeRageShot(camData) then results[4] = true end
                return
            end

            -- Silent aim path
            if Config.SilentAim then
                local tgt, part = selectTarget({
                    fov        = Config.SilentAimFOV,
                    checkVis   = Config.SilentAimVisCheck,
                    partMode   = Config.SilentAimTargetPart,
                    stickyTarget = State.SilentLastTarget,
                    stickyBonus  = Config.SilentAimStickiness or 0.05,
                })
                if tgt and part then
                    State.SilentLastTarget = tgt
                    Visuals.notifyTarget(tgt)
                    local hitPart = pickSilentPart(tgt.Character, part)
                    if Rage._encodeShot(camData, hitPart, tgt.Character, camPos) then
                        results[4] = true
                        State.Shots = State.Shots + 1
                        State.Hits  = State.Hits  + 1
                    end
                    return
                end
            end

            -- Aimbot shot-override path
            if Config.Aimbot and Config.AimbotShotOverride
                and State.AimbotTarget and State.AimbotPart and State.AimbotKeyHeld then
                local tgt = State.AimbotTarget
                if tgt and tgt.Character and isValidTarget(tgt, false) then
                    Visuals.notifyTarget(tgt)
                    if Rage._encodeShot(camData, State.AimbotPart, tgt.Character, camPos) then
                        results[4] = true
                        State.Shots = State.Shots + 1
                        State.Hits  = State.Hits  + 1
                    end
                end
            end

            -- MANUAL fire (no assist target resolved above): still hologram whoever is under the crosshair,
            -- so the dot works on every shot at a target regardless of which features are on — gated only by
            -- the hologram toggle. (Rage returns earlier and shows its own dot from the engine.)
            if Config.HUD or Config.VisualsHolograms then
                local rp = RaycastParams.new()
                rp.FilterType = Enum.RaycastFilterType.Exclude
                rp.FilterDescendantsInstances = { lp.Character }
                local res = workspace:Raycast(camPos, Camera.CFrame.LookVector * (Config.MaxDistance or 1000), rp)
                if res and res.Instance then
                    local m  = res.Instance:FindFirstAncestorOfClass("Model")
                    local pl = m and Players:GetPlayerFromCharacter(m)
                    if pl and pl ~= lp then Visuals.notifyTarget(pl) end
                end
            end
        end)
        return unpack(results)
    end

    -- Give hookWrapper the GAME env so getfenv(Rivals.Gun.StartShooting).hookfunction == nil (closes a
    -- latent leak — no known probe scans StartShooting today, but cheap). NOT newcclosure: that would flip
    -- iscclosure false→true, anomalous for a module method. Upvalues are unaffected by setfenv; only free
    -- globals (typeof/pcall/unpack/Enum/RaycastParams) rebind, all present in the original method's env.
    if setfenv then pcall(setfenv, hookWrapper, getfenv(oldStart)) end
    if setreadonly then pcall(setreadonly, Rivals.Gun, false) end
    Rivals.Gun.StartShooting = hookWrapper
end

hookGunModule()

-- ── AIMBOT (mobile‑compatible) ─────────────────────────────────────────────
local Aimbot = {}
do
    local _rendBound    = false
    local _fovCircle, _lockIndicator = nil, nil
    local _mouseMoveFn  = mousemoverel

    local _aimCacheTarget, _aimCachePart = nil, nil
    local _aimCacheFrame, _aimFrameCount = 0, 0
    local AIM_CACHE_FRAMES = 3

    -- easing / humanization / auto-fire runtime state (file-local; no State-table changes)
    local _prevTarget   = nil
    local _acquireAt    = 0     -- tick() the current target was acquired (reaction delay + accel ramp)
    local _ramp         = 0     -- 0..1 acceleration ramp since acquire (EaseInOut)
    local _lastAutoFire = 0

    -- Convert the FOV setting to a pixel radius. "Degrees" mode uses the live camera FieldOfView
    -- so a given angular cone maps to the correct on-screen radius at any zoom.
    local function currentFovPx()
        if Config.AimbotFOVMode == "Degrees" then
            local vp   = Camera.ViewportSize
            local half = math.tan(math.rad(Config.AimbotFOVDegrees or 8) * 0.5)
                       / math.tan(math.rad(Camera.FieldOfView) * 0.5)
            return half * (vp.Y * 0.5)
        end
        return Config.AimbotFOV
    end

    -- Crosshair-pixel distance (and part) of a player's aim part — for switch-threshold hysteresis.
    local function screenDistOf(player, mode)
        local char = player and player.Character
        if not char then return math.huge, nil end
        local part = pickPart(char, mode)
        if not part then return math.huge, nil end
        local sp, on = Camera:WorldToViewportPoint(part.Position)
        if not on or sp.Z <= 0 then return math.huge, part end
        local vp = Camera.ViewportSize
        return (Vector2.new(sp.X, sp.Y) - Vector2.new(vp.X * 0.5, vp.Y * 0.5)).Magnitude, part
    end

    -- Optional auto-fire/trigger: click when locked within a pixel threshold. Off by default.
    local function tryAutoFire(pixelDist)
        if not Config.AimbotAutoFire then return end
        if pixelDist > (Config.AimbotAutoFireFOV or 8) then return end
        local now = tick()
        if (now - _lastAutoFire) * 1000 < (Config.AimbotAutoFireDelay or 0) then return end
        _lastAutoFire = now
        pcall(function()
            if mouse1click then mouse1click()
            elseif mouse1press and mouse1release then mouse1press(); mouse1release() end
        end)
    end

    local function ensureDrawings()
        if _fovCircle and _lockIndicator then return end
        pcall(function()
            if not _fovCircle then
                local c = screenDraw("Circle")
                c.Thickness = 1; c.NumSides = 64
                c.Color = Color3.fromRGB(180, 180, 180)
                c.Transparency = 0.6; c.Filled = false; c.Visible = false
                _fovCircle = c
            end
            if not _lockIndicator then
                local c = screenDraw("Circle")
                c.Thickness = 2; c.NumSides = 32
                c.Color = Color3.fromRGB(255, 80, 100)
                c.Transparency = 0.9; c.Filled = false; c.Radius = 6; c.Visible = false
                _lockIndicator = c
            end
        end)
    end

    local function updateDrawings()
        if not Config.AimbotShowFOV and not Config.AimbotShowLock then
            if _fovCircle    then _fovCircle.Visible    = false end
            if _lockIndicator then _lockIndicator.Visible = false end
            return
        end
        ensureDrawings()
        if _fovCircle then
            if Config.Aimbot and Config.AimbotShowFOV then
                local vp = Camera.ViewportSize
                _fovCircle.Position = Vector2.new(vp.X * 0.5, vp.Y * 0.5)
                _fovCircle.Radius   = currentFovPx()
                _fovCircle.Visible  = true
            else
                _fovCircle.Visible = false
            end
        end
        if _lockIndicator then
            if Config.Aimbot and Config.AimbotShowLock and _aimCachePart then
                local sp, on = Camera:WorldToViewportPoint(_aimCachePart.Position)
                if on and sp.Z > 0 then
                    _lockIndicator.Position = Vector2.new(sp.X, sp.Y)
                    _lockIndicator.Visible  = true
                else
                    _lockIndicator.Visible = false
                end
            else
                _lockIndicator.Visible = false
            end
        end
    end

    -- Mobile‑friendly aiming: use camera CFrame manipulation instead of mouse events
    local function applyCameraAim(deltaX, deltaY)
        local cam = Camera
        local pitch, yaw = cam.CFrame:toEulerAnglesYXZ()
        local sensitivity = 0.002 * (Config.AimbotSensMultiplier or 1)
        local newYaw   = yaw   - deltaX * sensitivity
        local newPitch = pitch - deltaY * sensitivity
        newPitch = math.clamp(newPitch, -math.rad(80), math.rad(80))
        cam.CFrame = CFrame.new(cam.CFrame.Position) * CFrame.fromEulerAnglesYXZ(newPitch, newYaw, 0)
    end

    local function step(dt)
        _aimFrameCount = _aimFrameCount + 1
        updateDrawings()

        if not Config.Aimbot then
            State.AimbotTarget  = nil; State.AimbotPart = nil
            State.AimbotKeyHeld = false
            _aimCacheTarget     = nil; _aimCachePart   = nil
            _prevTarget         = nil
            return
        end
        if State.RageFiring then return end

        local keyDown = aimbotKeyDown()
        State.AimbotKeyHeld = keyDown
        if not keyDown then
            State.AimbotTarget  = nil; State.AimbotPart = nil
            State.AimbotLastTarget = nil
            _aimCacheTarget     = nil; _aimCachePart   = nil
            _prevTarget         = nil
            return
        end

        local fovPx    = currentFovPx()
        local hardLock = Config.AimbotHardLock

        -- HARD LOCK reselects every frame (cacheLen 1); otherwise reuse the cached target for
        -- AIM_CACHE_FRAMES while stickiness is on, to save selection cost.
        local cacheLen = (hardLock or (Config.AimbotStickiness or 0) <= 0) and 1 or AIM_CACHE_FRAMES
        local needRefresh = (_aimFrameCount - _aimCacheFrame >= cacheLen)
            or not _aimCacheTarget
            or not isValidTarget(_aimCacheTarget, false)

        if needRefresh then
            local now    = tick()
            local sticky = State.AimbotLastTarget
            if sticky and (now - State.AimbotLastTargetTime) > 2 then sticky = nil end
            local newTgt, newPart = selectTarget({
                fov          = fovPx,
                checkVis     = Config.AimbotVisCheck,
                partMode     = Config.AimbotTargetPart,
                stickyTarget = sticky,
                stickyBonus  = Config.AimbotStickiness,
            })
            -- SWITCH-THRESHOLD hysteresis: only abandon a still-valid current target if the new
            -- candidate is meaningfully closer to the crosshair (px). Prevents target flicker.
            local cur = _aimCacheTarget
            if not hardLock and cur and cur ~= newTgt and isValidTarget(cur, false) then
                local curD = screenDistOf(cur, Config.AimbotTargetPart)
                if curD <= fovPx then
                    local newD = newTgt and screenDistOf(newTgt, Config.AimbotTargetPart) or math.huge
                    if (curD - newD) < (Config.AimbotSwitchThreshold or 0) then
                        local _, curPart = screenDistOf(cur, Config.AimbotTargetPart)
                        newTgt, newPart = cur, curPart
                    end
                end
            end
            _aimCacheTarget, _aimCachePart = newTgt, newPart
            _aimCacheFrame = _aimFrameCount
        end

        local tgt, part = _aimCacheTarget, _aimCachePart
        if not tgt or not part then
            State.AimbotTarget = nil; State.AimbotPart = nil; _prevTarget = nil; return
        end

        -- Target acquisition bookkeeping (reaction delay + accel ramp reset on a NEW target).
        if tgt ~= _prevTarget then
            _acquireAt  = tick()
            _ramp       = 0
            _prevTarget = tgt
        end

        State.AimbotTarget     = tgt
        State.AimbotPart       = part
        State.AimbotLastTarget = tgt
        State.AimbotLastTargetTime = tick()

        local lead        = calculateLead(tgt.Character, Camera.CFrame.Position, Config.AimbotPrediction)
        local worldTarget = part.Position + lead
        local screenPos, onScreen = Camera:WorldToViewportPoint(worldTarget)
        if not onScreen or screenPos.Z < 0 then return end

        local vp        = Camera.ViewportSize
        local center    = Vector2.new(vp.X * 0.5, vp.Y * 0.5)
        local delta     = Vector2.new(screenPos.X, screenPos.Y) - center
        local mul       = Config.AimbotSensMultiplier or 1
        local pixelDist = delta.Magnitude

        -- HARD LOCK: snap the FULL crosshair→target delta every frame. On PC this MUST be done via raw
        -- MOUSE movement (mousemoverel) — Rivals runs its own camera controller that overwrites
        -- Camera.CFrame every frame, so setting the camera CFrame does NOTHING (that was the bug). Only
        -- moving the mouse actually aims. Camera-CFrame is used ONLY as the mobile fallback (no
        -- mousemoverel there). Target is reselected every frame above (cacheLen=1), so moving the full
        -- delta keeps us hard-locked on.
        if hardLock then
            if isMobile or not _mouseMoveFn then
                applyCameraAim(delta.X * mul, delta.Y * mul)
            else
                pcall(_mouseMoveFn, delta.X * mul, delta.Y * mul)
            end
            tryAutoFire(pixelDist)
            return
        end

        if pixelDist < (Config.AimbotDeadzone or 0) then return end

        -- REACTION DELAY (ms since acquire) before we start tracking a fresh target.
        if (Config.AimbotReactionMs or 0) > 0
            and (tick() - _acquireAt) * 1000 < Config.AimbotReactionMs then
            return
        end

        -- TIME-BASED (dt-normalized) smoothing: convert the per-60fps move fraction to this frame's
        -- dt so tracking speed is FPS-independent.
        dt = dt or (1 / 60)
        local ref   = dt * 60
        local s     = math.clamp(Config.AimbotSmooth or 0, 0, 0.99)
        local base  = math.clamp(1 - s, 0.01, 1)
        local alpha = 1 - (1 - base) ^ ref

        -- Advance the acceleration ramp (used by EaseInOut).
        _ramp = math.min(1, _ramp + dt / math.max(0.001, Config.AimbotAccelTime or 0.12))

        -- REAL easing curves that genuinely differ:
        --   Linear    = constant approach fraction.
        --   EaseOut   = decelerate as the crosshair nears the target (soft settle, no ramp-in).
        --   EaseInOut = accelerate from acquire (ramp) AND decelerate on arrival.
        local easing = Config.AimbotEasing or "Linear"
        local ease
        if easing == "Linear" then
            ease = 1
        elseif easing == "EaseOut" then
            ease = math.clamp(pixelDist / math.max(1, fovPx), 0.15, 1)
        else -- EaseInOut (also the fallback for the legacy "Exponential" value)
            local t  = _ramp
            local io = (t < 0.5) and (2 * t * t) or (1 - ((-2 * t + 2) ^ 2) / 2)
            local dz = math.clamp(pixelDist / math.max(1, fovPx), 0.15, 1)
            ease = io * dz
        end
        local moveFactor = math.clamp(alpha * ease, 0.01, 1)

        -- OVERSHOOT-AND-SETTLE: brief mid-approach over-correction, decaying to 0. Default 0 = rigid.
        local os = Config.AimbotOvershoot or 0
        if os > 0 then
            moveFactor = moveFactor * (1 + math.sin(_ramp * math.pi) * os)
        end

        local moveX = delta.X * moveFactor * mul
        local moveY = delta.Y * moveFactor * mul

        -- SUB-PIXEL NOISE humanization (dt-scaled). Default 0 = perfectly rigid.
        local nz = Config.AimbotNoise or 0
        if nz > 0 then
            moveX = moveX + (math.random() - 0.5) * 2 * nz * ref
            moveY = moveY + (math.random() - 0.5) * 2 * nz * ref
        end

        local cap = Config.AimbotSpeedCap or 0
        if cap > 0 then
            local mag = math.sqrt(moveX * moveX + moveY * moveY)
            if mag > cap then
                local k = cap / mag
                moveX = moveX * k; moveY = moveY * k
            end
        end

        if isMobile or not _mouseMoveFn then
            applyCameraAim(moveX, moveY)
        else
            pcall(_mouseMoveFn, moveX, moveY)
        end
        tryAutoFire(pixelDist)
    end

    -- FPS: the aim RenderStep is bound ONLY while aimbot is enabled (it used to run every frame even
    -- when off). Toggling on binds it; toggling off unbinds it => literally 0 cost per frame when off.
    function Aimbot.enable()
        Config.Aimbot = true
        if not _rendBound then
            RunService:BindToRenderStep("LuaHook_Aimbot", Enum.RenderPriority.Camera.Value + 1, step)
            _rendBound = true
        end
    end
    function Aimbot.disable()
        Config.Aimbot = false
        State.AimbotTarget = nil; State.AimbotPart = nil; State.AimbotKeyHeld = false
        _aimCacheTarget = nil; _aimCachePart = nil
        if _rendBound then
            pcall(function() RunService:UnbindFromRenderStep("LuaHook_Aimbot") end)
            _rendBound = false
        end
        if _fovCircle     then _fovCircle.Visible     = false end
        if _lockIndicator then _lockIndicator.Visible = false end
    end
    function Aimbot.init()
        if Config.Aimbot then Aimbot.enable() end
    end
    function Aimbot.unload()
        if _rendBound then
            pcall(function() RunService:UnbindFromRenderStep("LuaHook_Aimbot") end)
            _rendBound = false
        end
        -- Tear down the shared velocity tracker (05_utils) on full unload.
        if shared._LH_velConn then
            pcall(function() shared._LH_velConn:Disconnect() end)
            shared._LH_velConn = nil
        end
        if _fovCircle     then pcall(function() _fovCircle:Remove()     end); _fovCircle     = nil end
        if _lockIndicator then pcall(function() _lockIndicator:Remove() end); _lockIndicator = nil end
    end
    function Aimbot.hasMouseMove() return _mouseMoveFn ~= nil end
end

-- ── ESP ────────────────────────────────────────────────────────────────────
local ESP = {}
do
    local hasDrawing  = screenDraw ~= nil
    local _renderConn = nil
    local _espFrame   = 0
    local _lastRenderT = 0
    local _dcLastT    = 0                                 -- declutter throttle timestamp (E2.4)
    local _bboxCache  = {}
    local _bboxFrameN = {}
    local _pts = {}; for _i = 1, 8 do _pts[_i] = {} end   -- reused bracket buffer (no per-frame alloc)
    local _ctx = {}                                       -- reused per-frame shared context (no per-player alloc)
    local _renderArr = {}                                 -- reused nearest-N cap buffer
    local _dcArr     = {}                                 -- reused declutter overlap buffer
    local _radarO    = { drawings = {} }                  -- module-local radar drawing pool
    local _module    = { drawings = {} }                  -- module-local HUD drawing pool (threat count)

    local BLACK = Color3.new(0, 0, 0)
    local WHITE = Color3.new(1, 1, 1)   -- gradient-box base (UIGradient shows true against white)
    local HPBG  = Color3.fromRGB(11, 15, 22)   -- SIGNAL v3 · HP track token (#0B0F16, esp-v3 §04)
    local cam   = Camera

    -- Drawing font name -> id
    local FONTS = { UI = 0, System = 1, Plex = 2, Monospace = 3 }

    -- SIGNAL v2 · 5-stop health ramp — HOISTED to 05_utils (SIGNAL v4) as the shared `hpRamp`
    -- so the 06_visuals FX target-info panel uses the exact same stops. Callers below unchanged.

    -- SIGNAL v2 · animated gradient outline color (rank 1). Triangle-wave ping-pong A<->B so the
    -- drift reads as one continuous sweep; `offset` gives each corner a distinct phase. nil => caller falls back.
    local function boxGradColor(now, offset)
        local a, b = Config.ESPBoxGradientA, Config.ESPBoxGradientB
        if not a or not b then return nil end
        local spd = Config.ESPBoxGradientSpeed or 0.12
        local t = ((now or 0) * spd + (offset or 0)) % 1
        if t > 0.5 then t = 1 - t end
        return a:Lerp(b, math.clamp(t * 2, 0, 1))
    end

    -- SIGNAL v3 · full-box gradient outline: a real UIStroke UIGradient (esp-v3 §02) whose Rotation the
    -- caller spins per frame. The A→B→A ColorSequence is cached (rebuilt only when the config colors
    -- change) so no ColorSequence is allocated per frame — the gradient write skip-writes after frame 1.
    local _boxGradSeq, _boxSeqA, _boxSeqB = nil, nil, nil
    local function boxGradSeq()
        local a, b = Config.ESPBoxGradientA, Config.ESPBoxGradientB
        if not a or not b then return nil end
        if _boxGradSeq and a == _boxSeqA and b == _boxSeqB then return _boxGradSeq end
        _boxSeqA, _boxSeqB = a, b
        _boxGradSeq = ColorSequence.new({
            ColorSequenceKeypoint.new(0, a),
            ColorSequenceKeypoint.new(0.5, b),
            ColorSequenceKeypoint.new(1, a),
        })
        return _boxGradSeq
    end

    -- SIGNAL v2 · target-lock hysteresis (rank 4). A challenger must be >=10% closer AND held 0.5s
    -- before the lock switches; identical target keeps the lock. Prevents flicker between equidistant players.
    local _lock = { obj = nil, cand = nil, candT = 0 }
    local function resolveLock(bestO, bestD, lockO, lockD, now)
        local lk = _lock
        if bestO == nil then lk.obj = nil; lk.cand = nil; return nil end
        local prev = lk.obj
        if lk.obj == nil or lockO == nil then
            -- no current lock, or the locked object left the render set -> adopt the nearest immediately
            lk.obj = bestO; lk.cand = nil
        elseif bestO == lk.obj then
            lk.cand = nil
        elseif bestD <= (lockD or math.huge) * 0.9 then
            -- nearest differs from the lock and is >=10% closer: hold the challenger 0.5s before switching
            if lk.cand == bestO then
                if (now - (lk.candT or now)) >= 0.5 then lk.obj = bestO; lk.cand = nil end
            else
                lk.cand = bestO; lk.candT = now
            end
        else
            lk.cand = nil
        end
        -- stamp _primT only when the lock genuinely changes (drives the 140ms corner-grow / chevron drop-in)
        if lk.obj and lk.obj ~= prev then lk.obj._primT = now end
        return lk.obj
    end

    -- Rig-aware bone maps (name pairs). R15 = default; R6 detected at build.
    local SKEL_R15 = {
        {"Head","UpperTorso"},{"UpperTorso","LowerTorso"},
        {"UpperTorso","LeftUpperArm"},{"LeftUpperArm","LeftLowerArm"},{"LeftLowerArm","LeftHand"},
        {"UpperTorso","RightUpperArm"},{"RightUpperArm","RightLowerArm"},{"RightLowerArm","RightHand"},
        {"LowerTorso","LeftUpperLeg"},{"LeftUpperLeg","LeftLowerLeg"},{"LeftLowerLeg","LeftFoot"},
        {"LowerTorso","RightUpperLeg"},{"RightUpperLeg","RightLowerLeg"},{"RightLowerLeg","RightFoot"},
    }
    local SKEL_R6 = {
        {"Head","Torso"},
        {"Torso","Left Arm"},{"Torso","Right Arm"},
        {"Torso","Left Leg"},{"Torso","Right Leg"},
    }

    -- bug#1 rank1 (allocation storm): entering a match / every respawn wave makes N characters appear on the
    -- SAME frame, and ESP draws are lazy-created on first render (each build() does Instance.new + child
    -- UIStroke/UICorner/Triangle-frames). Hundreds of synchronous Instance.new + reparents into the ScreenGui
    -- in one frame forces a GUI layout flush = the multi-hundred-ms-to-second hitch that reads as a freeze.
    -- Fix: cap NEW primitive creations per frame; elements that don't get created this frame simply render
    -- next frame (progressive fill-in over a few frames). Deferred creates DO NOT cache (o[key] stays nil) so
    -- they retry — only genuine failures cache `false`.
    local _createBudget = 0
    local CREATE_BUDGET_PER_FRAME = 24

    local function newDraw(t)
        if not hasDrawing then return nil end
        local ok, d = pcall(screenDraw, t)
        if not ok then return nil end
        d.Visible = false; return d
    end

    -- Lazy single-primitive: create on first use, cache after (false = create failed).
    local function nd(o, key, dtype)
        local d = o[key]
        if d == nil then
            if _createBudget <= 0 then return nil end   -- defer to a later frame; leave uncached so it retries
            _createBudget = _createBudget - 1
            d = newDraw(dtype) or false
            o[key] = d
            if d then o.drawings[#o.drawings + 1] = d end
        end
        return d or nil
    end

    -- Lazy array of primitives grown to n on demand.
    local function ndArr(o, key, dtype, n)
        local arr = o[key]
        if not arr then arr = {}; o[key] = arr end
        for i = #arr + 1, n do
            if _createBudget <= 0 then break end        -- defer remaining elements to a later frame
            local d = newDraw(dtype)
            if not d then break end
            _createBudget = _createBudget - 1
            arr[i] = d
            o.drawings[#o.drawings + 1] = d
        end
        return arr
    end

    -- Square setter (filled or outlined).
    local function sq(d, x, y, w, h, filled, color, transp, thick)
        d.Filled = filled
        if not filled then d.Thickness = thick or 1 end
        d.Position = Vector2.new(x, y)
        d.Size     = Vector2.new(w, h)
        d.Color    = color
        d.Transparency = transp
        d.Visible  = true
    end

    local function ensureCham(o, char)
        local hl = o.cham
        if hl == nil then
            hl = Instance.new("Highlight")
            hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
            hl.Enabled   = false
            hl.Parent    = char
            o.cham = hl
        elseif hl and hl.Parent ~= char then
            pcall(function() hl.Parent = char end)
        end
        return o.cham
    end

    local function cleanESP(p)
        local o = State.ESPObjects[p]; if not o then return end
        for _, d in ipairs(o.drawings or {}) do
            pcall(function() if d.Remove then d:Remove() end end)
        end
        if o.cham and o.cham.Parent then pcall(function() o.cham:Destroy() end) end
        _bboxCache[p]  = nil
        _bboxFrameN[p] = nil
        State.ESPObjects[p] = nil
    end

    local function buildESP(player)
        if player == lp then return end
        cleanESP(player)
        local char = player.Character; if not char then return end
        local root = char:FindFirstChild("HumanoidRootPart"); if not root then return end

        -- Rig detection + cached bone-part refs (invalidated because buildESP re-runs on CharacterAdded).
        local hum  = char:FindFirstChildOfClass("Humanoid")
        local isR6 = (hum and hum.RigType == Enum.HumanoidRigType.R6) or (char:FindFirstChild("Torso") ~= nil)
        local rig  = isR6 and SKEL_R6 or SKEL_R15
        local bones = {}
        for i, pair in ipairs(rig) do
            bones[i] = { a = char:FindFirstChild(pair[1]), b = char:FindFirstChild(pair[2]) }
        end

        -- Drawings are LAZY-CREATED per enabled element (see nd/ndArr) — no upfront ~42-primitive alloc.
        -- SIGNAL v4 · _fadeT stamps the spawn-fade origin (fresh build / respawn fades in over 180ms).
        local o = { drawings = {}, root = root, rig = rig, bones = bones, _fadeT = tick() }
        State.ESPObjects[player] = o
    end

    local function bbox(char, player)
        local frame = _bboxFrameN[player] or -999
        if (_espFrame - frame) < 1 then
            local c = _bboxCache[player]
            if c then return c[1], c[2], c[3], c[4] end   -- `and unpack(c)` truncates to 1 value; return all 4
            return nil
        end
        local minX, minY, maxX, maxY
        if Config.ESPBoxMode == "Static" then
            local root = char:FindFirstChild("HumanoidRootPart") or char:FindFirstChild("UpperTorso")
                or char:FindFirstChild("Torso") or char:FindFirstChild("Head")
            if not root then _bboxFrameN[player] = _espFrame; _bboxCache[player] = nil; return nil end
            local pos   = root.Position
            local topSp = cam:WorldToViewportPoint(pos + Vector3.new(0, 2.9, 0))
            local botSp = cam:WorldToViewportPoint(pos - Vector3.new(0, 3.1, 0))
            if topSp.Z <= 0 and botSp.Z <= 0 then _bboxFrameN[player] = _espFrame; _bboxCache[player] = nil; return nil end
            -- SIGNAL v2 · FIXED ASPECT: width is always 0.5*height so the silhouette can never re-fit the
            -- animation pose (the "cannot breathe" tenet). The old right-vector projection is gone — it added
            -- 2 WorldToViewportPoint calls/player/frame and reintroduced per-frame width jitter.
            local h  = math.abs(topSp.Y - botSp.Y)
            local w  = h * 0.5
            if w < 1 then w = h * 0.5 end   -- degenerate guard (kept per contract)
            local cx = (topSp.X + botSp.X) * 0.5
            minX = cx - w * 0.5; maxX = cx + w * 0.5
            minY = math.min(topSp.Y, botSp.Y); maxY = math.max(topSp.Y, botSp.Y)
        else
            local ok, cf, sz = pcall(function() return char:GetBoundingBox() end)
            if not ok then return nil end
            local anyPt = false
            minX, minY, maxX, maxY = math.huge, math.huge, -math.huge, -math.huge
            for x = -1, 1, 2 do for y = -1, 1, 2 do for z = -1, 1, 2 do
                local pt = (cf * CFrame.new(sz.X*0.5*x, sz.Y*0.5*y, sz.Z*0.5*z)).Position
                local sp = cam:WorldToViewportPoint(pt)
                if sp.Z > 0 then
                    anyPt = true
                    if sp.X < minX then minX = sp.X end; if sp.Y < minY then minY = sp.Y end
                    if sp.X > maxX then maxX = sp.X end; if sp.Y > maxY then maxY = sp.Y end
                end
            end end end
            if not anyPt then _bboxFrameN[player] = _espFrame; _bboxCache[player] = nil; return nil end
        end
        _bboxFrameN[player] = _espFrame
        _bboxCache[player] = { minX, minY, maxX, maxY }
        return minX, minY, maxX, maxY
    end

    local function hideDrawings(o)
        for _, d in ipairs(o.drawings) do d.Visible = false end
    end

    -- Hide the on-screen 2D primitives but keep the off-screen arrow (still valid when off-screen).
    -- SIGNAL v4 · the arrow distance label rides the arrow's lifecycle, so it is exempt too.
    local function hide2D(o)
        for _, d in ipairs(o.drawings) do
            if d ~= o.arrow and d ~= o.arrowOutline and d ~= o.arrowDist then d.Visible = false end
        end
    end

    local function hideAll(o, force)
        if o._allHidden and not force then return end
        o._allHidden = true
        hideDrawings(o)
        if o.cham then o.cham.Enabled = false end
    end

    function ESP.rebuildAll()
        for p in pairs(State.ESPObjects) do cleanESP(p) end
        if not Config.ESP then return end
        for _, p in ipairs(getSafePlayers()) do
            if p ~= lp and p.Character then buildESP(p) end
        end
    end

    -- Weapon name with a small TTL cache (getWeaponName is owned by 05_utils; we just throttle calls).
    local function espWeapon(o, player)
        local now = tick()
        if not o._wepT or (now - o._wepT) > 0.5 then
            o._wep  = getWeaponName(player)
            o._wepT = now
        end
        return o._wep or "?"
    end

    -- Per-player render body. Hoisted out of the loop so pcall(renderPlayer, ...) allocates
    -- no closure per player per frame. Shared frame data comes from the reused _ctx table.
    local function renderPlayer(player, o)
        local ctx  = _ctx
        local char = player.Character
        if not char or not o.root or not o.root.Parent then cleanESP(player); return end
        if not o.root:IsA("BasePart") then cleanESP(player); return end
        local rootPos = o.root.Position
        if not rootPos then cleanESP(player); return end

        local vp     = ctx.vp
        local myRoot = ctx.myRoot
        local dist   = o._dist
        if not dist then
            dist = (myRoot and myRoot.Position and rootPos)
                and (myRoot.Position - rootPos).Magnitude or 0
        end
        local oor = dist > Config.ESPMaxDistance

        -- TEAM / DEAD gate — one place hides everything (incl. cham + arrow) for a teammate
        -- (team-check on) or a dead player.
        if (Config.ESPTeamCheck and isTeammate(player)) or (not isAlive(player)) then
            hideAll(o); return
        end

        -- visibility raycast: only needed for the Visibility color mode; skip OOR; throttle to every 3rd frame + cache.
        local vis = o._vis or false
        if (Config.ESPColorMode == "Visibility" or Config.ESPRadar or Config.ESPChamsVisSplit or Config.ESPPeekAlert
                or (Config.ESPChams and Config.ESPChamsStyle == "Ghost"))
            and not oor and (_espFrame - (o._visFrame or -99)) >= 3 then
            pcall(function() o._vis = isVisible(rootPos) end)
            o._visFrame = _espFrame
            vis = o._vis or false
        end

        -- dynamic mode color (nil => Static: each element uses its own picker; else accent for all)
        local dyn
        local cmode = Config.ESPColorMode
        if cmode == "Rainbow" then dyn = ctx.rb
        elseif cmode == "Visibility" then dyn = vis and Config.ColorVisible or Config.ColorEnemy
        elseif cmode == "Team" then dyn = isTeammate(player) and Config.ColorTeam or Config.ColorEnemy
        elseif cmode == "Distance" then
            dyn = Config.ESPDistNearColor:Lerp(Config.ESPDistFarColor,
                math.clamp(dist / math.max(Config.ESPMaxDistance, 1), 0, 1))
        end

        if oor or not ctx.hasDrawing then
            -- CHAMS obey range even when 2D is off
            if o.cham then o.cham.Enabled = false end
            hideDrawings(o); o._allHidden = true
            return
        end
        -- SIGNAL v4 · spawn fade-in: re-stamp the fade origin on the hidden->shown transition
        -- (re-entered range / post-respawn) so elements ease in over 180ms instead of popping.
        if o._allHidden and Config.ESPFadeIn then o._fadeT = ctx.now end
        o._allHidden = false
        local fadeMul = 1
        if Config.ESPFadeIn and o._fadeT then
            fadeMul = math.clamp((ctx.now - o._fadeT) / 0.18, 0, 1)
        end

        local outline = ctx.outline
        local textOutline = ctx.textOutline
        if textOutline == nil then textOutline = true end
        local oAlpha  = ctx.outlineAlpha
        local tscale  = 1
        if Config.ESPTextScaling then
            tscale = math.clamp(1 - (dist / math.max(Config.ESPMaxDistance, 1)) * 0.5, 0.55, 1)
        end
        -- far-legibility: floor the distance-SCALED text at this px (capped per-element to the
        -- element's own size, so a near target at tscale=1 keeps its exact slider size — the floor
        -- only stops shrink at range, it never enlarges).
        local tfloor = Config.ESPTextMinSize or 10
        local font = FONTS[Config.ESPFont] or 2

        local minX, minY, maxX, maxY = bbox(char, player)
        if minX then
            -- SIGNAL v2 · SPLIT-EMA box: smooth CENTER fast (aim matters) and HALF-SIZE slow (pose noise
            -- lands here) so the silhouette stops "breathing". _sbox stores {cx, cy, hx, hy}.
            -- SNAP (no lerp) when the center jumps >150px (respawn/teleport).
            if Config.ESPSmoothing then
                local sb = o._sbox
                local ccx, ccy = (minX + maxX) * 0.5, (minY + maxY) * 0.5
                local chx, chy = (maxX - minX) * 0.5, (maxY - minY) * 0.5
                if not sb then
                    sb = { ccx, ccy, chx, chy }; o._sbox = sb
                else
                    local dcx = ccx - sb[1]
                    local dcy = ccy - sb[2]
                    if (dcx * dcx + dcy * dcy) > (150 * 150) then
                        sb[1] = ccx; sb[2] = ccy; sb[3] = chx; sb[4] = chy
                    else
                        local dt = ctx.dt or 0
                        local aC = 1 - math.exp(-22 * dt)   -- center: half-life ~31ms
                        local aS = 1 - math.exp(-12 * dt)   -- size: half-life ~58ms (calmer)
                        sb[1] = sb[1] + (ccx - sb[1]) * aC
                        sb[2] = sb[2] + (ccy - sb[2]) * aC
                        sb[3] = sb[3] + (chx - sb[3]) * aS
                        sb[4] = sb[4] + (chy - sb[4]) * aS
                    end
                end
                minX, maxX = sb[1] - sb[3], sb[1] + sb[3]
                minY, maxY = sb[2] - sb[4], sb[2] + sb[4]
            end
            -- SIGNAL v3 · integer-snap the box AFTER split-EMA so a still target writes value-equal
            -- endpoints => the shim skip-write stage discards them (0 target writes on a static box).
            minX = math.floor(minX); minY = math.floor(minY)
            maxX = math.floor(maxX); maxY = math.floor(maxY)
            local w, h = maxX - minX, maxY - minY

            -- far-legibility: floor the on-screen box height (width grows by the same ratio, so the
            -- aspect holds) — far targets never collapse to an unreadable speck. Integer growth keeps
            -- a still target's writes value-equal for the shim skip-write.
            local minH = Config.ESPMinBoxHeight or 0
            if h > 0 and h < minH then
                local gw = math.floor(w * (minH / h - 1) * 0.5 + 0.5)
                local gh = math.ceil((minH - h) * 0.5)
                minX = minX - gw; maxX = maxX + gw
                minY = minY - gh; maxY = maxY + gh
                w, h = maxX - minX, maxY - minY
            end

            -- shared HP transition state (ghost / healtick / low-HP emphasis)
            local _prevHP = o._lastHP
            local _curHP, _curMH, _curFrac
            if Config.ESPHealth or Config.ESPHealthGhost or Config.ESPHealTick then
                pcall(function() _curHP, _curMH = getHealth(player) end)
                if _curHP ~= nil then
                    _curFrac = math.clamp((_curMH or 0) > 0 and _curHP / _curMH or 0, 0, 1)
                end
            end
            if Config.ESPHealthGhost and _prevHP and _curFrac and _curFrac < _prevHP - 0.005 then
                if not o._ghostT or (ctx.now - o._ghostT) >= 0.35 then
                    o._ghostFrac = _prevHP
                else
                    o._ghostFrac = math.max(o._ghostFrac or _prevHP, _prevHP)
                end
                o._ghostT = ctx.now
            end

            -- ── BOX ──────────────────────────────────────────────────────────
            if Config.ESPBox then
                local bc     = dyn or Config.ESPBoxColor
                local bAlpha = (1 - Config.ESPBoxTransparency) * fadeMul
                local thick  = Config.ESPBoxThickness
                -- SIGNAL v2 · lock flourish (rank 4): +1px thickness and corners grow x1.0->x1.3 over
                -- 140ms outQuad from the moment this target became primary (_primT set in render()).
                local cornerMul = 1
                if Config.ESPPrimaryEmphasis and o._primary then
                    thick = thick + 1; bAlpha = fadeMul   -- full alpha (× spawn fade)
                    local pt = o._primT and math.clamp((ctx.now - o._primT) / 0.14, 0, 1) or 1
                    local e  = 1 - (1 - pt) * (1 - pt)   -- outQuad
                    cornerMul = 1 + 0.3 * e
                end
                -- SIGNAL v2 · animated gradient outline (rank 1): when ESPBoxGradient, each line takes a
                -- drifting A<->B color; nil-safe fallback to the solid box color bc.
                local grad = Config.ESPBoxGradient and boxGradColor(ctx.now, 0) ~= nil

                if Config.ESPBoxFill then
                    local bf = nd(o, "boxFill", "Square")
                    if bf then
                        sq(bf, minX, minY, w, h, true, dyn or Config.ESPBoxFillColor, 1 - Config.ESPBoxFillTransparency)
                        bf.Thickness = 0
                    end
                elseif o.boxFill then o.boxFill.Visible = false end

                if Config.ESPBoxBrackets then
                    if o.box then o.box.Visible = false end
                    if o.boxShadow then o.boxShadow.Visible = false end
                    local bl  = math.min(w, h) * math.clamp(Config.ESPCornerLength, 0.05, 0.5) * cornerMul
                    local pts = _pts
                    pts[1][1] = Vector2.new(minX,minY);      pts[1][2] = Vector2.new(minX+bl,minY)
                    pts[2][1] = Vector2.new(minX,minY);      pts[2][2] = Vector2.new(minX,minY+bl)
                    pts[3][1] = Vector2.new(maxX,minY);      pts[3][2] = Vector2.new(maxX-bl,minY)
                    pts[4][1] = Vector2.new(maxX,minY);      pts[4][2] = Vector2.new(maxX,minY+bl)
                    pts[5][1] = Vector2.new(minX,maxY);      pts[5][2] = Vector2.new(minX+bl,maxY)
                    pts[6][1] = Vector2.new(minX,maxY);      pts[6][2] = Vector2.new(minX,maxY-bl)
                    pts[7][1] = Vector2.new(maxX,maxY);      pts[7][2] = Vector2.new(maxX-bl,maxY)
                    pts[8][1] = Vector2.new(maxX,maxY);      pts[8][2] = Vector2.new(maxX,maxY-bl)
                    local brk   = ndArr(o, "brackets", "Line", 8)
                    local bshad = outline and ndArr(o, "bracketShadows", "Line", 8) or o.bracketShadows
                    for i = 1, 8 do
                        local L = brk[i]
                        if L then
                            local f, t = pts[i][1], pts[i][2]
                            local S = bshad and bshad[i]
                            if S then
                                if outline then S.From=f; S.To=t; S.Thickness=thick+2; S.Color=BLACK; S.Transparency=oAlpha; S.Visible=true
                                else S.Visible=false end
                            end
                            local lc = grad and boxGradColor(ctx.now, (math.ceil(i / 2) - 1) / 4) or bc
                            L.From=f; L.To=t; L.Color=lc; L.Thickness=thick; L.Transparency=bAlpha; L.Visible=true
                        end
                    end
                else
                    if o.brackets then for _, l in ipairs(o.brackets) do l.Visible=false end end
                    if o.bracketShadows then for _, l in ipairs(o.bracketShadows) do l.Visible=false end end
                    local box    = nd(o, "box", "Square")
                    local shadow = outline and nd(o, "boxShadow", "Square") or o.boxShadow
                    if shadow then
                        if outline then sq(shadow, minX-1, minY-1, w+2, h+2, false, BLACK, oAlpha, thick+2)
                        else shadow.Visible=false end
                    end
                    -- full-box: opt-in real UIStroke UIGradient (white base so the gradient shows true) spun
                    -- per frame; else the plain box color. Explicit Gradient=false keeps a pooled-reused box clean.
                    if box then
                        local seq = grad and boxGradSeq()
                        if seq then
                            sq(box, minX, minY, w, h, false, WHITE, bAlpha, thick)
                            box.Gradient = seq
                            box.GradientRotation = (ctx.now * (Config.ESPBoxGradientSpeed or 0.12) * 360) % 360
                        else
                            sq(box, minX, minY, w, h, false, bc, bAlpha, thick)
                            box.Gradient = false
                        end
                    end
                end
            else
                if o.box then o.box.Visible=false end
                if o.boxShadow then o.boxShadow.Visible=false end
                if o.boxFill then o.boxFill.Visible=false end
                if o.brackets then for i = 1, #o.brackets do o.brackets[i].Visible=false end end
                if o.bracketShadows then for i = 1, #o.bracketShadows do o.bracketShadows[i].Visible=false end end
            end

            -- ── SKELETON (rig-aware, cached bone refs) ───────────────────────
            if Config.ESPSkeleton and o.rig then
                local sc     = dyn or Config.ESPSkeletonColor
                local sAlpha = (1 - Config.ESPSkeletonTransparency) * fadeMul
                local thick  = Config.ESPSkeletonThickness
                local n      = #o.rig
                local lines  = ndArr(o, "skeleton", "Line", n)
                local shad   = outline and ndArr(o, "skeletonShadow", "Line", n) or o.skeletonShadow
                -- SIGNAL v3 · project the 2*n bone endpoints only every 2nd frame; cache the screen points
                -- in o._skelPts ({f=Vector2|false, t=Vector2|false} per bone). Off-frames re-feed the cached
                -- From/To (the shim skip-write makes those value-equal writes free). Halves WVP/player.
                local pts = o._skelPts
                if (not pts) or (_espFrame - (o._skelFrame or -9)) >= 2 then
                    if not pts then pts = {}; o._skelPts = pts end
                    for i = 1, n do
                        local bp = o.bones[i]
                        local a  = bp.a; if not a or not a.Parent then a = char:FindFirstChild(o.rig[i][1]); bp.a = a end
                        local b  = bp.b; if not b or not b.Parent then b = char:FindFirstChild(o.rig[i][2]); bp.b = b end
                        local pe = pts[i]; if not pe then pe = {}; pts[i] = pe end
                        if a and b then
                            local sa, oA = cam:WorldToViewportPoint(a.Position)
                            local sb, oB = cam:WorldToViewportPoint(b.Position)
                            if oA and oB and sa.Z > 0 and sb.Z > 0 then
                                pe.f = Vector2.new(sa.X, sa.Y); pe.t = Vector2.new(sb.X, sb.Y)
                            else
                                pe.f = false; pe.t = false
                            end
                        else
                            pe.f = false; pe.t = false
                        end
                    end
                    o._skelFrame = _espFrame
                end
                for i = 1, n do
                    local L = lines[i]
                    if L then
                        local pe = pts[i]
                        local S  = shad and shad[i]
                        if pe and pe.f then
                            local f, t = pe.f, pe.t
                            if S then
                                if outline then S.From=f; S.To=t; S.Thickness=thick+2; S.Color=BLACK; S.Transparency=oAlpha; S.Visible=true
                                else S.Visible=false end
                            end
                            L.From=f; L.To=t; L.Color=sc; L.Thickness=thick; L.Transparency=sAlpha; L.Visible=true
                        else
                            L.Visible=false; if S then S.Visible=false end
                        end
                    end
                end
            else
                if o.skeleton then for _, l in ipairs(o.skeleton) do l.Visible=false end end
                if o.skeletonShadow then for _, l in ipairs(o.skeletonShadow) do l.Visible=false end end
            end

            -- ── HEALTH BAR (outlined gradient; vertical/horizontal; never overlaps box/name) ──
            local infoY = maxY + 3
            if Config.ESPHealth then
                local hp, mh = _curHP, _curMH
                if hp == nil then pcall(function() hp, mh = getHealth(player) end) end
                local pct    = _curFrac or math.clamp((mh or 0) > 0 and hp / mh or 0, 0, 1)
                local fillPct = pct
                if Config.ESPHealthSmooth then
                    local s = o._sHP
                    if s == nil then s = pct
                    else s = s + (pct - s) * math.clamp((ctx.dt or 0) * 18, 0, 1) end
                    o._sHP = s; fillPct = s
                end
                local barCol = Config.ESPHealthGradient and hpRamp(fillPct)
                    or (dyn or Config.ESPHealthColor)
                local hAlpha = (1 - Config.ESPHealthTransparency) * fadeMul
                local horizontal = Config.ESPHealthOrientation == "Horizontal"
                local fillTop
                if h > 2 then
                    if horizontal then
                        local barH = 4
                        local by   = maxY + 4
                        local bw   = w
                        if outline and nd(o, "hpOutline", "Square") then sq(o.hpOutline, minX-1, by-1, bw+2, barH+2, false, BLACK, oAlpha, 1)
                        elseif o.hpOutline then o.hpOutline.Visible = false end
                        if nd(o, "hpBg", "Square") then sq(o.hpBg, minX, by, bw, barH, true, HPBG, 0.86) end   -- SIGNAL v3 · solid dark track @86% (esp-v3 §04)
                        local fw = math.max(bw * fillPct, 1)
                        if nd(o, "hpFill", "Square") then sq(o.hpFill, minX, by, fw, barH, true, barCol, hAlpha) end
                        if Config.ESPHealthGhost then
                            local g = nd(o, "hpGhost", "Square")
                            if g and o._ghostT and o._ghostFrac and o._ghostFrac > fillPct and (ctx.now - o._ghostT) < 0.35 then
                                local gx = minX + bw * fillPct
                                local gw = math.max(bw * (o._ghostFrac - fillPct), 1)
                                local gt = 0.55 * (1 - math.clamp((ctx.now - o._ghostT) / 0.35, 0, 1))
                                sq(g, gx, by, gw, barH, true, Color3.new(1, 1, 1), gt)
                            elseif g then g.Visible = false end
                        elseif o.hpGhost then o.hpGhost.Visible = false end
                        if o.hpTicks then for _, l in ipairs(o.hpTicks) do l.Visible = false end end   -- ticks are vertical-only
                        infoY   = by + barH + 3
                        fillTop = by
                    else
                        local barW = 4
                        local bx   = minX - 8
                        local by   = minY
                        local bh   = h
                        if outline and nd(o, "hpOutline", "Square") then sq(o.hpOutline, bx-1, by-1, barW+2, bh+2, false, BLACK, oAlpha, 1)
                        elseif o.hpOutline then o.hpOutline.Visible = false end
                        if nd(o, "hpBg", "Square") then sq(o.hpBg, bx, by, barW, bh, true, HPBG, 0.86) end   -- SIGNAL v3 · solid dark track @86% (esp-v3 §04)
                        local fh = math.max(bh * fillPct, 1)
                        fillTop  = by + (bh - fh)
                        if nd(o, "hpFill", "Square") then sq(o.hpFill, bx, fillTop, barW, fh, true, barCol, hAlpha) end
                        -- SIGNAL v3 · optional 25/50/75% dark seg ticks across the vertical HP bar (esp-v3 §04).
                        if Config.ESPHealthSegTicks then
                            local tk = ndArr(o, "hpTicks", "Line", 3)
                            for ti = 1, 3 do
                                local L = tk[ti]
                                if L then
                                    local ty = by + bh * (ti * 0.25)
                                    L.From = Vector2.new(bx, ty); L.To = Vector2.new(bx + barW, ty)
                                    L.Thickness = 1; L.Color = BLACK; L.Transparency = 0.55; L.Visible = true
                                end
                            end
                        elseif o.hpTicks then for _, l in ipairs(o.hpTicks) do l.Visible = false end end
                        if Config.ESPHealthGhost then
                            local g = nd(o, "hpGhost", "Square")
                            if g and o._ghostT and o._ghostFrac and o._ghostFrac > fillPct and (ctx.now - o._ghostT) < 0.35 then
                                local gTop = by + (bh - bh * o._ghostFrac)
                                local gh   = math.max(fillTop - gTop, 1)
                                local gt   = 0.55 * (1 - math.clamp((ctx.now - o._ghostT) / 0.35, 0, 1))
                                sq(g, bx, gTop, barW, gh, true, Color3.new(1, 1, 1), gt)
                            elseif g then g.Visible = false end
                        elseif o.hpGhost then o.hpGhost.Visible = false end
                    end
                    local numMode = Config.ESPHealthNumberMode
                    local showNum = numMode == "Always" or (numMode == "OnDamage" and pct < 1)
                        or (Config.ESPHealthGhost and _curFrac ~= nil and _curFrac <= 0.25)
                    if showNum then
                        local ht = nd(o, "hpText", "Text")
                        if ht then
                            ht.Font = font
                            ht.Size = math.max(math.min(tfloor, Config.ESPHealthTextSize), math.floor(Config.ESPHealthTextSize * tscale))
                            ht.Outline = textOutline
                            ht.Color = WHITE
                            ht.Transparency = hAlpha
                            ht.Text = tostring(math.floor(hp or 0))
                            if horizontal then
                                ht.Center = false
                                ht.Position = Vector2.new(math.floor(minX + w + 4), math.floor(fillTop - 1))
                            else
                                ht.Center = true
                                ht.Position = Vector2.new(math.floor(minX - 8 + 2), math.floor(fillTop - ht.Size - 1))
                            end
                            ht.Visible = true
                        end
                    elseif o.hpText then o.hpText.Visible = false end
                else
                    if o.hpOutline then o.hpOutline.Visible = false end
                    if o.hpBg then o.hpBg.Visible = false end
                    if o.hpFill then o.hpFill.Visible = false end
                    if o.hpGhost then o.hpGhost.Visible = false end
                    if o.hpText then o.hpText.Visible = false end
                    if o.hpTicks then for ti = 1, #o.hpTicks do o.hpTicks[ti].Visible = false end end
                end
            else
                if o.hpOutline then o.hpOutline.Visible = false end
                if o.hpBg then o.hpBg.Visible = false end
                if o.hpFill then o.hpFill.Visible = false end
                if o.hpGhost then o.hpGhost.Visible = false end
                if o.hpText then o.hpText.Visible = false end
                if o.hpTicks then for ti = 1, #o.hpTicks do o.hpTicks[ti].Visible = false end end
            end

            -- ── NAME (above box) ─────────────────────────────────────────────
            if Config.ESPName and not (Config.ESPDeclutter and o._declutter) then
                local nt = nd(o, "nameText", "Text")
                if nt then
                    nt.Font = font
                    nt.Size = math.max(math.min(tfloor, Config.ESPTextSize), math.floor(Config.ESPTextSize * tscale))   -- legibility floor (no hide)
                    nt.Center = true; nt.Outline = textOutline
                    nt.Position = Vector2.new(math.floor(minX + w * 0.5), math.floor(minY - nt.Size - 3))
                    -- SIGNAL v2 · name source (§02 row 5): Display default, @username opt-in.
                    nt.Text  = (Config.ESPNameMode == "Username") and player.Name or player.DisplayName
                    -- SIGNAL v2 · tenet 01: text is always TEXT-1 — the name keeps its own color in every
                    -- color mode (shapes carry state color, identity survives). No `dyn` on text.
                    nt.Color = Config.ESPNameColor
                    nt.Transparency = (1 - Config.ESPNameTransparency) * fadeMul
                    nt.Visible = true

                    -- SIGNAL v2 · health-tinted name underline (rank 3): 2px ramp-colored bar under the
                    -- name, length = TextBounds.X * HP%. Rides the smoothed fill; casing 1px. pcall-isolated.
                    if Config.ESPNameHealthUnderline then
                        pcall(function()
                            local frac = o._sHP or _curFrac
                            if frac == nil then
                                local hp2, mh2; pcall(function() hp2, mh2 = getHealth(player) end)
                                if hp2 then frac = math.clamp((mh2 or 0) > 0 and hp2 / mh2 or 0, 0, 1) end
                            end
                            local un = nd(o, "nameUnder", "Line")
                            if un and frac then
                                local tb    = nt.TextBounds
                                local fullW = (tb and tb.X) or 0
                                local uy    = nt.Position.Y + nt.Size + 2
                                local lx    = nt.Position.X - fullW * 0.5
                                local uw    = fullW * math.clamp(frac, 0, 1)
                                local unS   = outline and nd(o, "nameUnderShadow", "Line") or o.nameUnderShadow
                                if unS then
                                    if outline then unS.From=Vector2.new(lx-1, uy); unS.To=Vector2.new(lx+uw+1, uy); unS.Thickness=3; unS.Color=BLACK; unS.Transparency=oAlpha; unS.Visible=true
                                    else unS.Visible=false end
                                end
                                un.From=Vector2.new(lx, uy); un.To=Vector2.new(lx+uw, uy)
                                un.Thickness=2; un.Color=hpRamp(frac); un.Transparency=1; un.Visible = uw > 0.5
                            elseif un then un.Visible = false end
                        end)
                    elseif o.nameUnder then o.nameUnder.Visible = false; if o.nameUnderShadow then o.nameUnderShadow.Visible = false end end
                end
            elseif o.nameText then
                o.nameText.Visible = false
                if o.nameUnder then o.nameUnder.Visible = false end
                if o.nameUnderShadow then o.nameUnderShadow.Visible = false end
            end

            -- ── INFO: distance / weapon (below box, below any horizontal HP bar) ──
            if (Config.ESPDistance or Config.ESPWeapon) and not (Config.ESPDeclutter and o._declutter) then
                local it = nd(o, "infoText", "Text")
                if it then
                    local txt
                    if Config.ESPDistance then txt = ("%dm"):format(dist) end
                    if Config.ESPWeapon then
                        local wep = espWeapon(o, player)
                        txt = txt and (txt .. " · " .. wep) or wep
                    end
                    it.Font = font
                    it.Size = math.max(math.min(tfloor, Config.ESPInfoTextSize), math.floor(Config.ESPInfoTextSize * tscale))
                    it.Center = true; it.Outline = textOutline
                    it.Position = Vector2.new(math.floor(minX + w * 0.5), math.floor(infoY))
                    it.Text  = txt or ""
                    it.Color = Config.ESPInfoColor   -- SIGNAL v3 · footer is TEXT-2 (a tier below the TEXT-1 name); still no `dyn` state tint
                    it.Transparency = (1 - Config.ESPNameTransparency) * fadeMul
                    it.Visible = txt ~= nil
                end
            elseif o.infoText then o.infoText.Visible = false end

            -- ── TRACER ───────────────────────────────────────────────────────
            if Config.ESPTracers then
                -- SIGNAL v2 · origin set gains a Mouse option (rank 2).
                local origin = Vector2.new(vp.X * 0.5, vp.Y)
                local m = Config.ESPTracerOrigin
                if m == "Top" then origin = Vector2.new(vp.X * 0.5, 0)
                elseif m == "Middle" then origin = Vector2.new(vp.X * 0.5, vp.Y * 0.5)
                elseif m == "Mouse" then
                    local mp; pcall(function() mp = UserInputService:GetMouseLocation() end)
                    origin = mp or Vector2.new(vp.X * 0.5, vp.Y * 0.5)
                end
                local to    = Vector2.new(minX + w * 0.5, maxY)
                local thick = Config.ESPTracerThickness
                local tcol  = dyn or Config.ESPTracerColor
                local baseA = 1 - Config.ESPTracerTransparency
                local trS   = outline and nd(o, "tracerShadow", "Line") or o.tracerShadow

                if Config.ESPTracerGradient then
                    -- SIGNAL v2 · gradient snapline (rank 2): Drawing.Line has no per-vertex alpha, so we stack
                    -- 3 short segments with alpha ramped 0.15 -> 1.0 origin->target. Casing is one full line.
                    if o.tracer then o.tracer.Visible = false end
                    if trS then
                        if outline then trS.From=origin; trS.To=to; trS.Thickness=thick+2; trS.Color=BLACK; trS.Transparency=oAlpha; trS.Visible=true
                        else trS.Visible=false end
                    end
                    local segs = ndArr(o, "tracerSeg", "Line", 3)
                    for k = 1, 3 do
                        local L = segs[k]
                        if L then
                            local f = origin:Lerp(to, (k - 1) / 3)
                            local t = origin:Lerp(to, k / 3)
                            local a = (0.15 + 0.85 * (k / 3)) * baseA
                            L.From=f; L.To=t; L.Color=tcol; L.Thickness=thick; L.Transparency=math.clamp(a, 0, 1); L.Visible=true
                        end
                    end
                else
                    if o.tracerSeg then for _, l in ipairs(o.tracerSeg) do l.Visible=false end end
                    local tr = nd(o, "tracer", "Line")
                    if tr then
                        if trS then
                            if outline then trS.From=origin; trS.To=to; trS.Thickness=thick+2; trS.Color=BLACK; trS.Transparency=oAlpha; trS.Visible=true
                            else trS.Visible=false end
                        end
                        tr.From=origin; tr.To=to; tr.Color=tcol
                        tr.Thickness=thick; tr.Transparency=baseA; tr.Visible=true
                    end
                end
            elseif o.tracer then
                o.tracer.Visible = false
                if o.tracerShadow then o.tracerShadow.Visible = false end
                if o.tracerSeg then for _, l in ipairs(o.tracerSeg) do l.Visible=false end end
            end

            -- ── HEAD DOT ─────────────────────────────────────────────────────
            if Config.ESPHeadDot then
                local hd  = nd(o, "headDot", "Circle")
                local hdS = outline and nd(o, "headDotShadow", "Circle") or o.headDotShadow
                if hd then
                    local head = char:FindFirstChild("Head")
                    if head then
                        local hsp, inv = cam:WorldToViewportPoint(head.Position)
                        if inv and hsp.Z > 0 then
                            local p = Vector2.new(hsp.X, hsp.Y)
                            local r = math.clamp(Config.ESPHeadDotSize, 1, 20)
                            if hdS then
                                if outline then hdS.Filled=true; hdS.NumSides=16; hdS.Radius=r+1; hdS.Position=p; hdS.Color=BLACK; hdS.Transparency=oAlpha; hdS.Visible=true
                                else hdS.Visible=false end
                            end
                            hd.Filled=true; hd.NumSides=16; hd.Radius=r; hd.Position=p
                            hd.Color=dyn or Config.ESPHeadDotColor; hd.Transparency=1 - Config.ESPHeadDotTransparency; hd.Visible=true
                        else
                            hd.Visible=false; if hdS then hdS.Visible=false end
                        end
                    else
                        hd.Visible=false; if hdS then hdS.Visible=false end
                    end
                end
            elseif o.headDot then
                o.headDot.Visible = false
                if o.headDotShadow then o.headDotShadow.Visible = false end
            end

            -- ── NET-NEW OVERLAYS (all default OFF, each isolated by pcall) ───
            -- SIGNAL v2 · lock chevron (rank 4): a gold triangle drops in above the name of the primary
            -- (locked) target. Drop-in uses _primT (140ms outQuad). pcall-isolated; hidden otherwise.
            if Config.ESPLockChevron and o._primary then
                pcall(function()
                    local cv = nd(o, "lockChev", "Triangle")
                    if cv then
                        local pt  = o._primT and math.clamp((ctx.now - o._primT) / 0.14, 0, 1) or 1
                        local e   = 1 - (1 - pt) * (1 - pt)   -- outQuad
                        local cxp = minX + w * 0.5
                        local yTop = (minY - 24) + (1 - e) * 6   -- slides down into place
                        cv.Filled = true
                        cv.PointA = Vector2.new(cxp, yTop + 6)     -- tip points down at the box
                        cv.PointB = Vector2.new(cxp - 5, yTop)
                        cv.PointC = Vector2.new(cxp + 5, yTop)
                        cv.Color  = Color3.fromRGB(255, 194, 75)
                        cv.Transparency = e
                        cv.Visible = true
                    end
                end)
            elseif o.lockChev then o.lockChev.Visible = false end

            if Config.ESPPeekAlert then
                pcall(function()
                    local v = o._vis and true or false
                    if v and not o._peekWasVis then o._peekT = ctx.now end
                    o._peekWasVis = v
                    local pb = nd(o, "peekBox", "Square")
                    if pb then
                        if o._peekT and (ctx.now - o._peekT) < 0.12 then
                            sq(pb, minX - 4, minY - 4, w + 8, h + 8, false, dyn or Config.ESPBoxColor, 1, 2)
                        else pb.Visible = false end
                    end
                end)
            elseif o.peekBox then o.peekBox.Visible = false end

            if Config.ESPFacingIndicator then
                pcall(function()
                    local myPos = ctx.myRoot and ctx.myRoot.Position
                    local fp = nd(o, "facePip", "Circle")
                    if fp and myPos and o.root then
                        local look = o.root.CFrame.LookVector
                        local to   = myPos - o.root.Position
                        local m    = to.Magnitude
                        if m > 1 then
                            to = to / m
                            local dp = look.X * to.X + look.Y * to.Y + look.Z * to.Z
                            if dp > 0.985 then
                                fp.Filled = true; fp.NumSides = 12; fp.Radius = 3
                                fp.Position = Vector2.new(math.floor(minX + w * 0.5), math.floor(minY - 12))
                                fp.Color = Config.ColorEnemy; fp.Transparency = 1; fp.Visible = true
                            else fp.Visible = false end
                        else fp.Visible = false end
                    end
                end)
            elseif o.facePip then o.facePip.Visible = false end

            if Config.ESPHealTick then
                pcall(function()
                    if _prevHP and _curFrac and _curFrac > _prevHP + 0.005 then o._healT = ctx.now end
                    local he = nd(o, "healEdge", "Square")
                    if he then
                        if o._healT and (ctx.now - o._healT) < 0.2 then
                            sq(he, minX - 2, minY - 2, w + 4, h + 4, false, Color3.fromRGB(61, 224, 122), 1, 2)  -- SIGNAL visible-green edge flash #3DE07A (stays green even if ColorVisible is cyan)
                        else he.Visible = false end
                    end
                end)
            elseif o.healEdge then o.healEdge.Visible = false end

            -- SIGNAL v4 · look-direction line: short segment from the head along their aim direction.
            -- Enemy-red when it sweeps toward us (same dot-product gate as the facing pip), TEXT-2 grey
            -- otherwise. The 2 WVPs reuse the skeleton's every-2nd-frame throttle pattern (o._lookFrame);
            -- off-frames re-feed the cached endpoints (value-equal writes are shim skip-write free).
            if Config.ESPLookLine then
                pcall(function()
                    local ll = nd(o, "lookLine", "Line")
                    if not ll then return end
                    local llS = outline and nd(o, "lookLineShadow", "Line") or o.lookLineShadow
                    if (not o._lookPts) or (_espFrame - (o._lookFrame or -9)) >= 2 then
                        o._lookFrame = _espFrame
                        local pts = o._lookPts; if not pts then pts = {}; o._lookPts = pts end
                        local head = char:FindFirstChild("Head")
                        if head then
                            local look = head.CFrame.LookVector
                            local p1   = head.Position
                            local p2   = p1 + look * math.clamp(Config.ESPLookLineLength or 8, 2, 32)
                            local s1, v1 = cam:WorldToViewportPoint(p1)
                            local s2, v2 = cam:WorldToViewportPoint(p2)
                            if v1 and v2 and s1.Z > 0 and s2.Z > 0 then
                                pts.f = Vector2.new(s1.X, s1.Y); pts.t = Vector2.new(s2.X, s2.Y)
                                local toward = false
                                local myPos = ctx.myRoot and ctx.myRoot.Position
                                if myPos then
                                    local to = myPos - p1
                                    local m  = to.Magnitude
                                    if m > 1 then
                                        to = to / m
                                        toward = (look.X * to.X + look.Y * to.Y + look.Z * to.Z) > 0.985
                                    end
                                end
                                pts.hot = toward
                            else
                                pts.f = false
                            end
                        else
                            pts.f = false
                        end
                    end
                    local pts = o._lookPts
                    if pts and pts.f then
                        if llS then
                            if outline then llS.From=pts.f; llS.To=pts.t; llS.Thickness=3; llS.Color=BLACK; llS.Transparency=oAlpha; llS.Visible=true
                            else llS.Visible=false end
                        end
                        ll.From = pts.f; ll.To = pts.t; ll.Thickness = 1
                        ll.Color = pts.hot and Config.ColorEnemy or Config.ESPInfoColor
                        ll.Transparency = fadeMul; ll.Visible = true
                    else
                        ll.Visible = false; if llS then llS.Visible = false end
                    end
                end)
            elseif o.lookLine then
                o.lookLine.Visible = false
                if o.lookLineShadow then o.lookLineShadow.Visible = false end
            end

            if _curFrac ~= nil then o._lastHP = _curFrac end
        else
            hide2D(o)   -- can't project the box this frame; keep arrow live below
        end

        -- ── THREAT TALLY (own pass; no raycast, decoupled from arrows) ───────
        if Config.ESPThreatCount then
            local tsp = cam:WorldToViewportPoint(rootPos)
            local tOn = tsp.Z > 0 and tsp.X >= 0 and tsp.X <= vp.X and tsp.Y >= 0 and tsp.Y <= vp.Y
            if not tOn then ctx.threat = (ctx.threat or 0) + 1 end
        end

        -- ── OFF-SCREEN ARROW (own pass; projection AFTER the range cull) ──────
        if Config.ESPArrows then
            local ar  = nd(o, "arrow", "Triangle")
            -- SIGNAL v3 · the shim now self-cases the unfilled caret (dark casing legs + apex stem), so the
            -- separate arrowOutline triangle is retired — kept permanently hidden, never re-created.
            local arO = o.arrowOutline
            if ar then
                local asp = cam:WorldToViewportPoint(rootPos)
                local onScreen = asp.Z > 0 and asp.X >= 0 and asp.X <= vp.X and asp.Y >= 0 and asp.Y <= vp.Y
                if not onScreen then
                    local cen  = Vector2.new(vp.X * 0.5, vp.Y * 0.5)
                    local adir = Vector2.new(asp.X - cen.X, asp.Y - cen.Y)
                    if asp.Z <= 0 then adir = Vector2.new(-adir.X, -adir.Y) end
                    if adir.Magnitude < 1 then adir = Vector2.new(0, -1) end
                    adir = adir.Unit
                    local edge = cen + adir * (math.min(vp.X, vp.Y) * 0.38)
                    local perp = Vector2.new(-adir.Y, adir.X)
                    local ac   = dyn or Config.ESPBoxColor
                    if arO then arO.Visible = false end
                    -- SIGNAL v4 · distance fade: nearby threats scream (alpha 1), far ones whisper (0.35)
                    local aAlpha = 1 - Config.ESPBoxTransparency
                    if Config.ESPArrowDistFade then
                        aAlpha = aAlpha * (1 - 0.65 * math.clamp(dist / math.max(Config.ESPMaxDistance, 1), 0, 1))
                    end
                    ar.Filled = false   -- cased caret (self-cased in the shim); write Filled before PointC (geom bind)
                    ar.PointA = edge + adir * 20
                    ar.PointB = (edge - adir * 10) + perp * 8
                    ar.PointC = (edge - adir * 10) - perp * 8
                    ar.Color = ac; ar.Transparency = aAlpha; ar.Visible = true
                    -- SIGNAL v4 · optional distance label just inside the arrow
                    if Config.ESPArrowDistLabel then
                        local dl = nd(o, "arrowDist", "Text")
                        if dl then
                            local lpos = edge - adir * 24
                            dl.Font = FONTS[Config.ESPFont] or 2
                            dl.Size = 10; dl.Center = true; dl.Outline = true
                            dl.Color = Config.ESPInfoColor   -- TEXT-2
                            dl.Text = ("%dm"):format(dist)
                            dl.Position = Vector2.new(math.floor(lpos.X), math.floor(lpos.Y - 5))
                            dl.Transparency = aAlpha
                            dl.Visible = true
                        end
                    elseif o.arrowDist then o.arrowDist.Visible = false end
                else
                    ar.Visible = false
                    if arO then arO.Visible = false end
                    if o.arrowDist then o.arrowDist.Visible = false end
                end
            end
        elseif o.arrow then
            o.arrow.Visible = false
            if o.arrowOutline then o.arrowOutline.Visible = false end
            if o.arrowDist then o.arrowDist.Visible = false end
        end

        -- ── CHAMS (independent of screen projection) ─────────────────────────
        if Config.ESPChams then
            local hl = ensureCham(o, char)
            if hl then
                -- vis-split color (when on) is shared by every preset; else the raw cham colors.
                -- SIGNAL vis/occ matrix: visible = full team/enemy hue; occluded = same hue pulled toward grey.
                local vc = Config.ESPChamsVisSplit
                    and (o._vis and (isTeammate(player) and Config.ColorTeam or Config.ColorEnemy)
                                 or (isTeammate(player) and Config.ColorTeamOcc or Config.ColorEnemyOcc))
                    or nil
                local state = vc or dyn or Config.ESPChamsFillColor
                -- SIGNAL v2 · silhouette style presets (rank 5). Unknown/nil style -> raw-slider fallback.
                local style = Config.ESPChamsStyle
                if style == "Shade" then
                    -- SIGNAL v2 rank 5 · Shade (readable default): state fill @ .62 transp, crisp white edge.
                    hl.FillColor           = state
                    hl.OutlineColor        = Color3.new(1, 1, 1)
                    hl.FillTransparency    = 0.62
                    hl.OutlineTransparency = 0
                    hl.Enabled             = true
                elseif style == "Neon" then
                    -- SIGNAL v2 rank 5 · Neon (statement): faint fill @ .88, the read lives in the glowing edge.
                    hl.FillColor           = state
                    hl.OutlineColor        = vc or dyn or Config.ESPChamsOutlineColor
                    hl.FillTransparency    = 0.88
                    hl.OutlineTransparency = 0
                    hl.Enabled             = true
                elseif style == "Ghost" then
                    hl.FillColor           = state
                    hl.OutlineColor        = Color3.new(1, 1, 1)
                    hl.FillTransparency    = 1
                    hl.OutlineTransparency = 0.25
                    hl.Enabled             = not o._vis   -- occluded-only (needs the vis raycast; gated below)
                else
                    -- raw-slider fallback (original behavior)
                    if vc then
                        hl.FillColor = vc; hl.OutlineColor = vc
                    else
                        hl.FillColor = dyn or Config.ESPChamsFillColor
                        hl.OutlineColor = dyn or Config.ESPChamsOutlineColor
                    end
                    hl.FillTransparency    = Config.ESPChamsFillTransparency
                    hl.OutlineTransparency = Config.ESPChamsOutlineTransparency
                    hl.Enabled             = true
                end
            end
        elseif o.cham then o.cham.Enabled = false end
    end

    -- NaN-safe (bug#1 rank5): ragdolling/dying enemies fling parts to the float limit → NaN distances
    -- make a raw `a.d < b.d` an inconsistent comparator, which Luau throws on ("invalid order function"),
    -- and pcall cannot rescue a C-level non-return. Push NaN to the end deterministically.
    local function _byDist(a, b)
        local ad, bd = a.d, b.d
        if ad ~= ad then return false end
        if bd ~= bd then return true end
        return ad < bd
    end

    -- Declutter: flag farther boxes >60% covered by a nearer box (uses last frame's cached rects).
    local function doDeclutter(ctx)
        local myPos = ctx.myRoot and ctx.myRoot.Position
        local n = 0
        for player, o in pairs(State.ESPObjects) do
            o._declutter = false
            local c = _bboxCache[player]
            if c and o.root and o.root.Parent then
                n = n + 1
                local e = _dcArr[n]; if not e then e = {}; _dcArr[n] = e end
                e.o = o; e.c = c
                local rp = o.root.Position
                e.d = (myPos and rp) and (myPos - rp).Magnitude or math.huge
            end
        end
        for i = #_dcArr, n + 1, -1 do _dcArr[i] = nil end
        table.sort(_dcArr, _byDist)
        for i = 1, n - 1 do
            local A = _dcArr[i]; local ca = A.c
            local aMinX, aMinY, aMaxX, aMaxY = ca[1], ca[2], ca[3], ca[4]
            for j = i + 1, n do
                local B = _dcArr[j]
                if not B.o._declutter then
                    local cb = B.c
                    local ix = math.min(aMaxX, cb[3]) - math.max(aMinX, cb[1])
                    local iy = math.min(aMaxY, cb[4]) - math.max(aMinY, cb[2])
                    if ix > 0 and iy > 0 then
                        local barea = (cb[3] - cb[1]) * (cb[4] - cb[2])
                        if barea > 0 and (ix * iy) / barea > 0.6 then B.o._declutter = true end
                    end
                end
            end
        end
    end

    -- Radar: top-right disc; reused backdrop/rim; pooled dots/chevrons by index. No new raycasts (reuses o._vis).
    local function renderRadar(ctx)
        if not Config.ESPRadar then
            for _, d in ipairs(_radarO.drawings) do d.Visible = false end
            return
        end
        if (_espFrame % 2) ~= 0 then return end   -- SIGNAL v3 · ~60Hz radar (was %4) — fixes dot/pulse stutter
        local vp = ctx.vp; if not vp then return end
        local R     = math.max((Config.ESPRadarSize or 180) * 0.5, 10)
        local inset = Config.ESPRadarInset or 20
        local cx    = vp.X - inset - R
        local cy    = inset + R

        local disc = nd(_radarO, "disc", "Circle")
        if disc then
            disc.Filled = true; disc.NumSides = 32; disc.Radius = R
            disc.Position = Vector2.new(cx, cy)
            disc.Color = Color3.fromRGB(13, 18, 25)   -- SIGNAL FILL token (#0D1219 @ ~62%)
            disc.Transparency = 0.62; disc.Visible = true
        end
        -- SIGNAL v2 · inner range ring at 0.5R (faint white).
        local rring = nd(_radarO, "rangeRing", "Circle")
        if rring then
            rring.Filled = false; rring.Thickness = 1; rring.NumSides = 28; rring.Radius = R * 0.5
            rring.Position = Vector2.new(cx, cy); rring.Color = Color3.new(1, 1, 1)
            rring.Transparency = 0.10; rring.Visible = true
        end
        -- SIGNAL v2 · self-pip + rim are drawn AFTER the contacts (see below) so they sit on TOP
        -- of the dots per the design's bottom->top layer order (…dots -> chevrons -> self -> rim).

        local myPos = ctx.myRoot and ctx.myRoot.Position
        if not myPos then return end
        local rot = 0
        if Config.ESPRadarRotate then
            local look = cam.CFrame.LookVector
            rot = -math.atan2(look.X, look.Z)
        end
        local cosR, sinR = math.cos(rot), math.sin(rot)
        local range = math.max(Config.ESPRadarRange or 250, 1)

        -- SIGNAL v2 · 4 cardinal rim ticks; north gold, others faint white; they turn with the radar.
        local ticks = ndArr(_radarO, "ticks", "Line", 4)
        for i = 0, 3 do
            local L = ticks[i + 1]
            if L then
                local ang = i * (math.pi * 0.5) + rot
                local dxT, dyT = math.sin(ang), -math.cos(ang)
                L.From = Vector2.new(cx + dxT * (R - 6), cy + dyT * (R - 6))
                L.To   = Vector2.new(cx + dxT * R, cy + dyT * R)
                L.Thickness = 1
                if i == 0 then L.Color = Color3.fromRGB(255, 194, 75); L.Transparency = 0.9
                else L.Color = Color3.new(1, 1, 1); L.Transparency = 0.35 end
                L.Visible = true
            end
        end

        -- SIGNAL v4 · faint cross grid through the center (turns with the radar, matches the 0.5R ring)
        if Config.ESPRadarGrid then
            local grid = ndArr(_radarO, "grid", "Line", 2)
            for gi = 1, 2 do
                local L = grid[gi]
                if L then
                    local ang = rot + (gi - 1) * (math.pi * 0.5)
                    local gx, gy = math.sin(ang), -math.cos(ang)
                    L.From = Vector2.new(cx - gx * R, cy - gy * R)
                    L.To   = Vector2.new(cx + gx * R, cy + gy * R)
                    L.Thickness = 1
                    L.Color = Color3.fromRGB(168, 180, 192)   -- SIGNAL NEUTRAL #A8B4C0
                    L.Transparency = 0.10
                    L.Visible = true
                end
            end
        elseif _radarO.grid then
            for _, L in ipairs(_radarO.grid) do L.Visible = false end
        end

        -- SIGNAL v4 · sonar sweep: 3 short center→rim blades at ang / ang-0.12 / ang-0.24 rad with
        -- fading alpha (0.35/0.18/0.08) — one revolution per 4s. Pure ambiance; 3 pooled Lines.
        if Config.ESPRadarSweep then
            local sw = ndArr(_radarO, "sweep", "Line", 3)
            local baseAng = ((ctx.now or 0) % 4) / 4 * (math.pi * 2) + rot
            for si = 1, 3 do
                local L = sw[si]
                if L then
                    local ang = baseAng - (si - 1) * 0.12
                    local sx, sy = math.sin(ang), -math.cos(ang)
                    L.From = Vector2.new(cx, cy)
                    L.To   = Vector2.new(cx + sx * (R - 2), cy + sy * (R - 2))
                    L.Thickness = si == 1 and 2 or 1
                    L.Color = Color3.fromRGB(255, 194, 75)    -- SIGNAL GOLD blade
                    L.Transparency = si == 1 and 0.35 or (si == 2 and 0.18 or 0.08)
                    L.Visible = true
                end
            end
        elseif _radarO.sweep then
            for _, L in ipairs(_radarO.sweep) do L.Visible = false end
        end

        local idx = 0
        local closestIdx, closestH = nil, math.huge
        local closestPx, closestPy, closestDotR = 0, 0, 0
        for player, o in pairs(State.ESPObjects) do
            local root = o.root
            if root and root.Parent then
                local rp = root.Position
                local dx = rp.X - myPos.X
                local dz = rp.Z - myPos.Z
                local sx = (dx / range) * R
                local sz = (dz / range) * R
                local rx = sx * cosR - sz * sinR
                local rz = sx * sinR + sz * cosR
                local mag = math.sqrt(rx * rx + rz * rz)
                if mag > R then rx = rx / mag * R; rz = rz / mag * R end
                idx = idx + 1
                local px, py = cx + rx, cy + rz
                local hdist  = math.sqrt(dx * dx + dz * dz)
                local dotR   = math.clamp(5 - (hdist / range) * 2, 3, 5)
                local team   = isTeammate(player)
                local filled = (not Config.ESPRadarVisSplit) or (o._vis and true or false)
                -- SIGNAL radar matrix: visible = filled full hue; occluded = hollow + greyed hue.
                local col
                if Config.ESPRadarVisSplit and not o._vis then
                    col = team and Config.ColorTeamOcc or Config.ColorEnemyOcc
                else
                    col = team and Config.ColorTeam or Config.ColorEnemy
                end
                if hdist < closestH then closestH = hdist; closestIdx = idx; closestPx = px; closestPy = py; closestDotR = dotR end
                -- SIGNAL v2 · black shadow disc under each dot (allocated before the dot for lower z-order).
                local dsh = ndArr(_radarO, "dotShadow", "Circle", idx)
                local sD = dsh[idx]
                if sD then
                    sD.Filled = true; sD.NumSides = 12; sD.Radius = dotR + 1
                    sD.Position = Vector2.new(px, py); sD.Color = BLACK
                    sD.Transparency = 0.6; sD.Visible = true
                end
                local dots = ndArr(_radarO, "dots", "Circle", idx)
                local d = dots[idx]
                if d then
                    d.Filled = filled
                    if not filled then d.Thickness = 1 end
                    d.NumSides = 12; d.Radius = dotR
                    d.Position = Vector2.new(px, py); d.Color = col
                    d.Transparency = 1; d.Visible = true
                end
                local chevs = ndArr(_radarO, "chev", "Triangle", idx)
                local cv = chevs[idx]
                if cv then
                    local dy = rp.Y - myPos.Y
                    if dy > 6 or dy < -6 then
                        local dir = dy > 6 and -1 or 1
                        cv.Filled = false   -- SIGNAL v3 · cased caret (shim self-cases); write before PointC (geom bind)
                        cv.PointA = Vector2.new(px, py + dir * (dotR + 4))
                        cv.PointB = Vector2.new(px - 3, py + dir * (dotR + 1))
                        cv.PointC = Vector2.new(px + 3, py + dir * (dotR + 1))
                        cv.Color = col; cv.Transparency = 1; cv.Visible = true
                    else cv.Visible = false end
                end
            end
        end
        -- SIGNAL v3 · gold RING around the nearest contact (esp-v3 §08: a hollow #FFC24B hoop around the
        -- dot, pulses r ×1.28 @1.6Hz) — not a dot-size pulse. The contact dot keeps its true size/hue and
        -- a gold ring breathes around it, so "nearest" reads without distorting the range cue.
        local nearRing = nd(_radarO, "nearRing", "Circle")
        if nearRing then
            if closestIdx then
                local pulse = 1 + 0.28 * (0.5 + 0.5 * math.sin((ctx.now or 0) * 1.6 * math.pi * 2))
                nearRing.Filled = false; nearRing.Thickness = 1.2; nearRing.NumSides = 24
                nearRing.Radius = (closestDotR + 3) * pulse
                nearRing.Position = Vector2.new(closestPx, closestPy)
                nearRing.Color = Color3.fromRGB(255, 194, 75)   -- SIGNAL GOLD #FFC24B
                nearRing.Transparency = 1; nearRing.Visible = true
            else
                nearRing.Visible = false
            end
        end
        if _radarO.dots      then for i = idx + 1, #_radarO.dots do _radarO.dots[i].Visible = false end end
        if _radarO.dotShadow then for i = idx + 1, #_radarO.dotShadow do _radarO.dotShadow[i].Visible = false end end
        if _radarO.chev      then for i = idx + 1, #_radarO.chev do _radarO.chev[i].Visible = false end end

        -- SIGNAL v2 · self-pip then rim, allocated LAST so they render on TOP of the contacts
        -- (design layer order …dots -> chevrons -> self -> rim). Self = gold arrow; rim = neutral chrome.
        local selfTri = nd(_radarO, "self", "Triangle")
        if selfTri then
            selfTri.Filled = true
            selfTri.PointA = Vector2.new(cx, cy - 5)
            selfTri.PointB = Vector2.new(cx - 4, cy + 4)
            selfTri.PointC = Vector2.new(cx + 4, cy + 4)
            selfTri.Color = Color3.fromRGB(255, 194, 75); selfTri.Transparency = 1; selfTri.Visible = true   -- SIGNAL GOLD (#FFC24B)
        end
        local rimShadow = nd(_radarO, "rimShadow", "Circle")
        if rimShadow then
            rimShadow.Filled = false; rimShadow.Thickness = 2; rimShadow.NumSides = 32; rimShadow.Radius = R
            rimShadow.Position = Vector2.new(cx, cy); rimShadow.Color = BLACK
            rimShadow.Transparency = 0.55; rimShadow.Visible = true   -- grounding casing under the rim
        end
        local rim = nd(_radarO, "rim", "Circle")
        if rim then
            rim.Filled = false; rim.Thickness = 1; rim.NumSides = 32; rim.Radius = R
            rim.Position = Vector2.new(cx, cy)
            rim.Color = Color3.fromRGB(168, 180, 192)   -- SIGNAL NEUTRAL token (#A8B4C0)
            rim.Transparency = 0.45; rim.Visible = true
        end
    end

    local function render()
        local _now = tick()
        local _dt  = _now - _lastRenderT
        if _now - _lastRenderT < 0.0083 then return end   -- cap ESP work ~120Hz (high-refresh FPS saver)
        _lastRenderT = _now
        _espFrame = _espFrame + 1
        _createBudget = CREATE_BUDGET_PER_FRAME   -- bug#1 rank1: cap NEW draw-primitive allocations per frame

        -- One-time full pre-warm can't happen here (budget-gated), but the radar module + HUD share the same
        -- shim pool via cleanESP, so the FIRST populated match fills in over ~a few frames instead of a spike.

        if not Config.ESP then
            State.PrimaryTarget = nil   -- SIGNAL v4 · target-info panel goes dark with the ESP
            for _, o in pairs(State.ESPObjects) do hideAll(o, true) end
            for _, d in ipairs(_radarO.drawings) do d.Visible = false end
            for _, d in ipairs(_module.drawings) do d.Visible = false end
            return
        end

        State.RainbowHue = (State.RainbowHue + 0.002) % 1
        local ctx = _ctx
        ctx.vp     = cam.ViewportSize
        ctx.myRoot = lp.Character and lp.Character:FindFirstChild("HumanoidRootPart")
        ctx.rb     = (Config.ESPColorMode == "Rainbow") and Color3.fromHSV(State.RainbowHue, 1, 1) or nil
        ctx.outline      = Config.ESPOutline
        ctx.outlineAlpha = 1 - Config.ESPOutlineTransparency
        ctx.hasDrawing   = hasDrawing
        ctx.dt          = math.clamp(_dt, 0, 0.1)
        ctx.now         = _now
        ctx.textOutline = Config.ESPTextOutline ~= false
        ctx.threat      = 0
        -- SIGNAL v3 · throttle the O(n^2)+sort declutter pass to ~20Hz; _declutter flags persist between
        -- runs (only cleared inside doDeclutter) so throttling doesn't flicker the name/info hide.
        if Config.ESPDeclutter and (ctx.now - _dcLastT) >= 0.05 then
            _dcLastT = ctx.now; pcall(doDeclutter, ctx)
        end

        local cap = Config.ESPMaxPlayers or 0
        if cap and cap > 0 then
            -- nearest-N: gather + sort by distance, render the closest `cap`, hide the rest.
            local arr = _renderArr
            local n = 0
            local myPos = ctx.myRoot and ctx.myRoot.Position
            for player, o in pairs(State.ESPObjects) do
                n = n + 1
                local e = arr[n]; if not e then e = {}; arr[n] = e end
                e.p = player; e.o = o
                local rp = o.root and o.root.Parent and o.root.Position
                e.d = (myPos and rp) and (myPos - rp).Magnitude or math.huge
            end
            for i = #arr, n + 1, -1 do arr[i] = nil end
            pcall(table.sort, arr, _byDist)
            if Config.ESPPrimaryEmphasis or Config.ESPLockChevron or Config.FXTargetInfo then
                -- SIGNAL v2 · hysteresis lock instead of a raw nearest pick (rank 4). Runs for the box
                -- flourish, the chevron OR the FX target-info panel (SIGNAL v4) so _primary is maintained
                -- whenever any of them wants it.
                for i = 1, n do arr[i].o._primary = false end
                local bestO = arr[1] and arr[1].o
                local bestD = (arr[1] and arr[1].d) or math.huge
                local lockO, lockD
                for i = 1, n do if arr[i].o == _lock.obj then lockO = arr[i].o; lockD = arr[i].d; break end end
                local locked = resolveLock(bestO, bestD, lockO, lockD, ctx.now)
                State.PrimaryTarget = nil   -- SIGNAL v4 · stamp the locked PLAYER for the FX target panel
                if locked then
                    locked._primary = true
                    for i = 1, n do if arr[i].o == locked then State.PrimaryTarget = arr[i].p break end end
                end
            else
                State.PrimaryTarget = nil
            end
            for i = 1, n do
                local e = arr[i]
                if i <= cap then
                    e.o._dist = e.d
                    pcall(renderPlayer, e.p, e.o)
                else
                    hideAll(e.o)
                end
            end
        else
            if Config.ESPPrimaryEmphasis or Config.ESPLockChevron or Config.FXTargetInfo then
                local myPos = ctx.myRoot and ctx.myRoot.Position
                local bestD, bestO = math.huge, nil
                local lockO, lockD
                for player, o in pairs(State.ESPObjects) do
                    o._primary = false
                    local rp = o.root and o.root.Parent and o.root.Position
                    local d = (myPos and rp) and (myPos - rp).Magnitude or math.huge
                    if d < bestD then bestD = d; bestO = o end
                    if o == _lock.obj then lockO = o; lockD = d end
                end
                -- SIGNAL v2 · hysteresis lock (rank 4).
                local locked = resolveLock(bestO, bestD, lockO, lockD, ctx.now)
                State.PrimaryTarget = nil   -- SIGNAL v4 · stamp the locked PLAYER for the FX target panel
                if locked then
                    locked._primary = true
                    for player, o in pairs(State.ESPObjects) do
                        if o == locked then State.PrimaryTarget = player break end
                    end
                end
            else
                State.PrimaryTarget = nil
            end
            for player, o in pairs(State.ESPObjects) do
                o._dist = nil
                pcall(renderPlayer, player, o)
            end
        end

        pcall(renderRadar, ctx)
        if Config.ESPThreatCount then
            pcall(function()
                local t = nd(_module, "threat", "Text")
                if t then
                    local cnt = ctx.threat or 0
                    if cnt > 0 then
                        t.Font = FONTS[Config.ESPFont] or 2
                        t.Size = 14; t.Center = true; t.Outline = true
                        t.Color = Config.ColorEnemy
                        t.Text = cnt .. " <"
                        t.Position = Vector2.new(math.floor(ctx.vp.X * 0.5), math.floor(ctx.vp.Y * 0.5 + 28))
                        t.Transparency = 1; t.Visible = true
                    else t.Visible = false end
                end
            end)
        elseif _module.threat then _module.threat.Visible = false end
    end

    function ESP.init()
        for _, p in ipairs(getSafePlayers()) do
            if p ~= lp then
                p.CharacterAdded:Connect(function()
                    task.wait(0.5); if Config.ESP then buildESP(p) end
                end)
            end
        end
        Players.PlayerAdded:Connect(function(p)
            p.CharacterAdded:Connect(function()
                task.wait(0.5); if Config.ESP then buildESP(p) end
            end)
        end)
        Players.PlayerRemoving:Connect(function(p) cleanESP(p) end)
        -- FPS: only connect the per-frame render loop while ESP is actually enabled.
        if Config.ESP and not _renderConn then
            _renderConn = RunService.RenderStepped:Connect(render)
            ESP.rebuildAll()   -- build for players already present if ESP is on at init (e.g. saved config)
        end
    end

    function ESP.enable()
        Config.ESP = true; ESP.rebuildAll()
        if not _renderConn then _renderConn = RunService.RenderStepped:Connect(render) end
    end
    function ESP.disable()
        Config.ESP = false; ESP.rebuildAll()
        State.PrimaryTarget = nil   -- SIGNAL v4 · don't leave the FX target panel a stale lock
        if _renderConn then _renderConn:Disconnect(); _renderConn = nil end
    end
    function ESP.unload()
        if _renderConn then _renderConn:Disconnect(); _renderConn = nil end
        State.PrimaryTarget = nil
        for p in pairs(State.ESPObjects) do cleanESP(p) end
        for _, pool in ipairs({ _radarO, _module }) do
            for _, d in ipairs(pool.drawings) do
                pcall(function() if d.Remove then d:Remove() end end)
            end
            pool.drawings = {}
            pool.disc, pool.rim, pool.self, pool.dots, pool.chev, pool.threat = nil, nil, nil, nil, nil, nil
            -- SIGNAL v2 · new pooled radar layers must also be dropped so re-init re-creates them.
            pool.rangeRing, pool.ticks, pool.dotShadow, pool.rimShadow, pool.nearRing = nil, nil, nil, nil, nil
            -- SIGNAL v4 · grid + sonar sweep pools
            pool.grid, pool.sweep = nil, nil
        end
    end
end

-- ── UTILITY ESP ─────────────────────────────────────────────────────────────
-- World-object ESP for thrown gadgets (grenades / molotovs / flashbangs / tripmines / satchels /
-- warpstones). Detection is EVENT-DRIVEN: one Workspace.DescendantAdded listener does a cheap
-- lowercase substring match on the instance Name (thrown entities are always NEW instances, and
-- every skinned variant keeps its base word — "Soul Grenade", "DIY Tripmine", "Potion Satchel").
-- NO per-item connections and NO per-frame workspace scans (the classic utility-ESP mistakes):
-- one central ~30Hz render pass walks the small live-item table, culls by distance, projects and
-- feeds pooled screenDraw primitives (dot + casing + pulse ring + label). Items self-expire when
-- unparented. Draw objects go back to the shim pool via :Remove() on expiry.
local UtilESP = {}
do
    local hasDrawing = screenDraw ~= nil

    local BLACK = Color3.new(0, 0, 0)
    -- SIGNAL tokens: explosives read as ENEMY red, disorient as GOLD, mobility as VISIBLE cyan.
    local C_RED  = Color3.fromRGB(255, 59, 78)     -- #FF3B4E explosive
    local C_GOLD = Color3.fromRGB(255, 194, 75)    -- #FFC24B disorient/trap
    local C_CYAN  = Color3.fromRGB(53, 215, 199)   -- #35D7C7 mobility
    local C_TEXT1 = Color3.fromRGB(243, 246, 250)  -- #F3F6FA label text (shapes carry the state color, text stays neutral)

    -- keyword -> display + class color + danger pulse. Matched against lowercase names.
    local KINDS = {
        { k = "tripmine",  label = "TRIPMINE",  color = C_GOLD, danger = true  },
        { k = "grenade",   label = "GRENADE",   color = C_RED,  danger = true  },
        { k = "molotov",   label = "MOLOTOV",   color = C_RED,  danger = true  },
        { k = "satchel",   label = "SATCHEL",   color = C_RED,  danger = true  },
        { k = "flashbang", label = "FLASHBANG", color = C_GOLD, danger = true  },
        { k = "warpstone", label = "WARPSTONE", color = C_CYAN, danger = false },
    }

    local _items = {}          -- [Instance] = { root=BasePart, kind=KINDS entry, dot, dotB, ring, label, dist }
    local _addConn, _renderConn = nil, nil
    local _lastT = 0

    local function matchKind(name)
        local n = string.lower(name)
        for _, kd in ipairs(KINDS) do
            if string.find(n, kd.k, 1, true) then return kd end
        end
        return nil
    end

    -- our OWN characters/viewmodels also carry gadget-named tools — only track LOOSE world models
    -- (not descendants of any player character or the ViewModels rig)
    local function isWorldItem(inst)
        local a = inst.Parent
        while a and a ~= Workspace do
            local nm = a.Name
            if nm == "ViewModels" then return false end
            if Players:GetPlayerFromCharacter(a) then return false end
            a = a.Parent
        end
        return a == Workspace
    end

    local function rootOf(inst)
        if inst:IsA("BasePart") then return inst end
        if inst:IsA("Model") then
            return inst.PrimaryPart or inst:FindFirstChildWhichIsA("BasePart")
        end
        return nil
    end

    local function drop(inst)
        local it = _items[inst]
        if not it then return end
        for _, key in ipairs({ "dot", "dotB", "ring", "label" }) do
            local d = it[key]
            if d then pcall(function() d:Remove() end) end
        end
        _items[inst] = nil
    end

    local function track(inst)
        if _items[inst] then return end
        if not (inst:IsA("Model") or inst:IsA("BasePart")) then return end
        local kd = matchKind(inst.Name)
        if not kd then return end
        if not isWorldItem(inst) then return end
        local root = rootOf(inst)
        if not root then
            -- model streamed in before its parts: retry once shortly after
            task.delay(0.1, function()
                if inst.Parent and not _items[inst] then
                    local r2 = rootOf(inst)
                    if r2 then _items[inst] = { root = r2, kind = kd } end
                end
            end)
            return
        end
        _items[inst] = { root = root, kind = kd }
    end

    local function hideItem(it)
        if it.dot   then it.dot.Visible   = false end
        if it.dotB  then it.dotB.Visible  = false end
        if it.ring  then it.ring.Visible  = false end
        if it.label then it.label.Visible = false end
    end

    local function render()
        local now = tick()
        if now - _lastT < 0.033 then return end   -- ~30Hz is plenty for slow-moving world items
        _lastT = now
        if not Config.UtilityESP or not hasDrawing then return end
        local cam = Camera; if not cam then return end
        local myC = lp.Character
        local myR = myC and myC:FindFirstChild("HumanoidRootPart")
        local maxD = Config.UtilityESPMaxDistance or 250
        for inst, it in pairs(_items) do
            if not inst.Parent or not it.root or not it.root.Parent then
                drop(inst)
            else
                local pos = it.root.Position
                local d = myR and (myR.Position - pos).Magnitude
                    or (cam.CFrame.Position - pos).Magnitude
                local sp, on = cam:WorldToViewportPoint(pos)
                if d <= maxD and on and sp.Z > 0 then
                    local col = it.kind.color
                    local p = Vector2.new(sp.X, sp.Y)
                    -- distance-scaled marker (4px near -> 2px far)
                    local r = math.clamp(4 - (d / maxD) * 2, 2, 4)
                    if not it.dotB then it.dotB = screenDraw("Circle") end
                    if it.dotB then
                        it.dotB.Filled = true; it.dotB.NumSides = 12; it.dotB.Radius = r + 1
                        it.dotB.Position = p; it.dotB.Color = BLACK
                        it.dotB.Transparency = 0.7; it.dotB.Visible = true
                    end
                    if not it.dot then it.dot = screenDraw("Circle") end
                    if it.dot then
                        it.dot.Filled = true; it.dot.NumSides = 12; it.dot.Radius = r
                        it.dot.Position = p; it.dot.Color = col
                        it.dot.Transparency = 1; it.dot.Visible = true
                    end
                    -- danger pulse ring (grenades/mines/etc.): breathes r+4 -> r+9 @1.6Hz
                    if it.kind.danger and Config.UtilityESPRing then
                        if not it.ring then it.ring = screenDraw("Circle") end
                        if it.ring then
                            local pulse = 0.5 + 0.5 * math.sin(now * 1.6 * math.pi * 2)
                            it.ring.Filled = false; it.ring.NumSides = 20
                            it.ring.Thickness = 1.5
                            it.ring.Radius = r + 4 + 5 * pulse
                            it.ring.Position = p; it.ring.Color = col
                            it.ring.Transparency = 1 - 0.55 * pulse
                            it.ring.Visible = true
                        end
                    elseif it.ring then it.ring.Visible = false end
                    if Config.UtilityESPLabels then
                        if not it.label then
                            it.label = screenDraw("Text")
                            if it.label then
                                it.label.Center = true; it.label.Outline = true
                                it.label.Font = 2; it.label.Size = 12
                            end
                        end
                        if it.label then
                            it.label.Text = it.kind.label .. " · " .. math.floor(d + 0.5) .. "m"
                            it.label.Position = Vector2.new(math.floor(sp.X), math.floor(sp.Y - 22))
                            it.label.Color = C_TEXT1   -- one-glance rule: dot/ring carry the class hue
                            it.label.Transparency = 1
                            it.label.Visible = true
                        end
                    elseif it.label then it.label.Visible = false end
                else
                    hideItem(it)
                end
            end
        end
    end

    local function connect()
        if not _addConn then
            -- hot path: this fires for EVERY new workspace instance (bullets, fx, debris) — do the
            -- cheap class + name rejection inline and only defer for actual gadget matches.
            _addConn = Workspace.DescendantAdded:Connect(function(inst)
                if not Config.UtilityESP then return end
                if not (inst:IsA("Model") or inst:IsA("BasePart")) then return end
                if not matchKind(inst.Name) then return end
                task.defer(track, inst)
            end)
        end
        if not _renderConn then
            _renderConn = RunService.RenderStepped:Connect(function() pcall(render) end)
        end
    end
    local function disconnect()
        if _addConn then _addConn:Disconnect(); _addConn = nil end
        if _renderConn then _renderConn:Disconnect(); _renderConn = nil end
    end

    local function sweepExisting()
        -- one-time catch-up scan on enable (items already mid-air / planted before we listened)
        for _, inst in ipairs(Workspace:GetDescendants()) do
            if inst:IsA("Model") or inst:IsA("BasePart") then
                if matchKind(inst.Name) then pcall(track, inst) end
            end
        end
    end

    function UtilESP.enable()
        Config.UtilityESP = true
        connect()
        task.spawn(sweepExisting)
    end
    function UtilESP.disable()
        Config.UtilityESP = false
        disconnect()
        for inst in pairs(_items) do drop(inst) end
    end
    function UtilESP.init()
        if Config.UtilityESP then pcall(UtilESP.enable) end
    end
    function UtilESP.unload()
        pcall(UtilESP.disable)
    end
end

-- ── LAB (in-game HvH telemetry — measure, don't guess) ──────────────────────
local Lab = {}
do
    local _gui, _label, _conn, _hpConn, _caConn
    local s = {
        lastT = 0, lastShots = 0, lastEvents = 0,
        taken = 0, takenAtk = 0, takenHide = 0, takenReload = 0,
        sps = 0, eps = 0,
    }

    local function combatState()
        if Config.RageVoidPhase and State.RageVoidActive then return "HIDE" end
        if State.RageFiring then return "ATTACK" end
        local item = getEquippedItem()
        if item then
            if (item._reload_cooldown or 0) > tick() then return "RELOAD" end
            local okA, ammo = pcall(function() return item:Get("Ammo") end)
            if okA and type(ammo) == "number" and ammo <= 0 then return "RELOAD" end
        end
        return "OUT"
    end

    -- avg studs the current target's hitbox jumps per frame (their desync strength)
    local function enemyDesync()
        local t = State.Target;        if not t then return 0 end
        local buf = State.RageBacktrackBuf[t]; if not buf or #buf < 3 then return 0 end
        local sum, n = 0, 0
        for i = math.max(2, #buf - 8), #buf do
            sum = sum + (buf[i].pos - buf[i - 1].pos).Magnitude; n = n + 1
        end
        return n > 0 and sum / n or 0
    end

    -- how far our replicated body currently sits from our camera (are we desynced now?)
    local function selfDesync()
        local c = lp.Character
        local root = c and c:FindFirstChild("HumanoidRootPart")
        return root and (root.Position - Camera.CFrame.Position).Magnitude or 0
    end

    local function pct(p, w) return w > 0 and math.floor(p / w * 100 + 0.5) or 0 end

    local function hookSelfHP()
        local c = lp.Character
        local h = c and c:FindFirstChildOfClass("Humanoid"); if not h then return end
        if _hpConn then _hpConn:Disconnect() end
        local last = h.Health
        _hpConn = h.HealthChanged:Connect(function(new)
            if new < last - 0.5 then
                local d, st = last - new, combatState()
                s.taken = s.taken + d
                if st == "ATTACK" then s.takenAtk = s.takenAtk + d
                elseif st == "RELOAD" then s.takenReload = s.takenReload + d
                else s.takenHide = s.takenHide + d end
            end
            last = new
        end)
    end

    function Lab.reset()
        s.taken, s.takenAtk, s.takenHide, s.takenReload = 0, 0, 0, 0
        State.RageDealtTotal = 0
    end

    function Lab.enable()
        Config.RageLab = true
        if not _gui then
            _gui = Instance.new("ScreenGui")
            _gui.Name = "_lh_lab"; _gui.ResetOnSpawn = false; _gui.IgnoreGuiInset = true
            local ok = pcall(function() _gui.Parent = (gethui and gethui()) or game:GetService("CoreGui") end)
            if not ok then _gui.Parent = lp:WaitForChild("PlayerGui") end
            _label = Instance.new("TextLabel")
            _label.Position = UDim2.fromOffset(14, 120)
            _label.Size = UDim2.fromOffset(360, 168)
            _label.BackgroundColor3 = Color3.fromRGB(8, 10, 16)
            _label.BackgroundTransparency = 0.2
            _label.TextColor3 = Color3.fromRGB(0, 230, 160)
            _label.Font = Enum.Font.Code; _label.TextSize = 14
            _label.TextXAlignment = Enum.TextXAlignment.Left
            _label.TextYAlignment = Enum.TextYAlignment.Top
            _label.Text = "LAB"; _label.Parent = _gui
        end
        _gui.Enabled = true
        s.lastT = tick(); s.lastShots = State.Shots; s.lastEvents = State.Hits
        hookSelfHP()
        if _caConn then _caConn:Disconnect() end
        _caConn = lp.CharacterAdded:Connect(function() task.wait(0.3); if Config.RageLab then hookSelfHP() end end)
        if _conn then _conn:Disconnect() end
        _conn = RunService.Heartbeat:Connect(function()
            if not Config.RageLab then return end
            local now = tick(); local dt = now - s.lastT
            if dt < 0.25 then return end
            s.lastT = now
            s.sps = math.floor((State.Shots - s.lastShots) / dt)
            s.eps = (State.Hits - s.lastEvents) / dt
            s.lastShots = State.Shots; s.lastEvents = State.Hits
            local dealt = math.floor(State.RageDealtTotal or 0)
            local taken = math.floor(s.taken)
            local ratio = taken > 0 and (dealt / taken) or (dealt > 0 and 999 or 0)
            local sd = selfDesync()
            local sdStr = isSanePos((lp.Character and lp.Character:FindFirstChild("HumanoidRootPart") and lp.Character.HumanoidRootPart.Position) or Vector3.zero)
                and string.format("%d studs", math.floor(sd)) or "VOID!! (flung)"
            -- counter-cheat readout: flag the current target + count detected cheaters
            local cheaters = 0
            for _, pl in ipairs(getSafePlayers()) do
                if pl ~= lp and Rage._isCheater and Rage._isCheater(pl) then cheaters = cheaters + 1 end
            end
            local tgtTag = (State.Target and Rage._isCheater and Rage._isCheater(State.Target)) and "  <<CHEATER>>" or ""
            _label.Text = string.format(
                "── LUAHOOK LAB ──\n"
                .. "state: %-6s   self-desync: %s\n"
                .. "enemy desync: %.1f studs/frame%s\n"
                .. "cheaters detected: %d\n"
                .. "fire: %d shots/s   hit-events/s: %.1f\n"
                .. "DEALT: %d   TAKEN: %d   trade x%.2f\n"
                .. "we die while:\n"
                .. "  ATTACK %d (%d%%)  HIDE %d (%d%%)  RELOAD %d (%d%%)",
                combatState(), sdStr, enemyDesync(), tgtTag, cheaters,
                s.sps, s.eps, dealt, taken, ratio,
                math.floor(s.takenAtk), pct(s.takenAtk, s.taken),
                math.floor(s.takenHide), pct(s.takenHide, s.taken),
                math.floor(s.takenReload), pct(s.takenReload, s.taken))
        end)
    end

    function Lab.disable()
        Config.RageLab = false
        if _gui then _gui.Enabled = false end
        if _conn then _conn:Disconnect(); _conn = nil end
        if _hpConn then _hpConn:Disconnect(); _hpConn = nil end
        if _caConn then _caConn:Disconnect(); _caConn = nil end
    end

    function Lab.init() end
end

-- ── INIT ───────────────────────────────────────────────────────────────────
-- isolate each so one module failing to init can't abort the whole load (which could leave hooks
-- half-applied). The Gun hook is already crash-isolated separately.
pcall(Visuals.init); pcall(Rage.init); pcall(Aimbot.init); pcall(ESP.init); pcall(Weather.init); pcall(UtilESP.init)

-- HUD is an independent master (decoupled from Visuals). Start it if on by default/saved.
if Config.HUD then pcall(Visuals.enableHUD) end
pcall(HUDPlus.init)   -- rides the HUD master (starts its updater only when Config.HUD is on)

-- ── AC BYPASS  Channel A neutralization — MOVED to 00b_ac_emitter_scrub.lua ──────────────────
-- The old `ACReportBypass` (rawset shared.r numeric slots -> no-op) and `ChannelBKill` (getgc
-- hookfunction on TakeTheL/ban/kick reporters) blocks that lived HERE were REMOVED 2026-07-12.
-- WHY: both TAMPERED the egress (silenced handlers / no-op'd reporters). The wire oracle + the
-- independent review showed egress-tampering is itself a liability: it cannot silence the
-- keepalive (Channel B is continuous; its ABSENCE is a flag), and shadowing/no-op'ing report
-- handlers is a distinguishable client state. They are REPLACED by the SCRUB-AT-EMITTER-INPUT
-- design in 00b_ac_emitter_scrub.lua, which lets the genuine clean pipeline run over {1,2,3,4,5}
-- filler so the frame that leaves is byte-identical to a real clean report BY CONSTRUCTION, and
-- leaves Channel B untouched. The setmetatable hang (Layer 1, 01_bootstrap) remains the
-- load-bearing bypass. See C:\Kernaled\ac-re\BYPASS_UPDATED_DESIGN.txt for the full rationale.
-- IMPORTANT: purge any STALE saved LinoriaLib config that force-enables the retired
-- Config.ACReportBypass / Config.ChannelBKill flags (they are now dead toggles).

-- ── LINORIA GUI ────────────────────────────────────────────────────────────
-- LINORIA GUI (MOBILE OPTIMISED – conditional)
local repo = 'https://raw.githubusercontent.com/mstudio45/LinoriaLib/main/'
local Library, ThemeManager, SaveManager
local ok, err = pcall(function()
    Library      = loadstring(game:HttpGet(repo .. 'Library.lua'))()
    ThemeManager = loadstring(game:HttpGet(repo .. 'addons/ThemeManager.lua'))()
    SaveManager  = loadstring(game:HttpGet(repo .. 'addons/SaveManager.lua'))()
end)
if not ok or not Library then warn("[Engine] Linoria load failed:", err); return end

-- Only apply mobile flags when actually on a touch device
Library.IsMobile = isMobile
-- CURSOR FIX (2nd-monitor offset): always use the native OS cursor. The Linoria custom cursor is a
-- Drawing-library Triangle that renders in physical/desktop-pixel space, so on a secondary or DPI-scaled
-- monitor the drawn pointer sits shifted from where Roblox actually hit-tests GuiButtons ("clicks land off").
-- This is the same root cause we fixed for ESP by moving to the IgnoreGuiInset ScreenGui shim. Forcing
-- false makes MouseIconEnabled=true (the fork's cursor loop) so pointer + click always coincide on every
-- monitor/DPI. Touch already hit-tests natively, so mobile loses nothing. (Was: = isMobile — a touchscreen
-- laptop driving an external monitor reports TouchEnabled=true and silently got the offset drawn cursor.)
Library.ShowCustomCursor = false
Library.ShowToggleFrameInKeybinds = isMobile       -- toggle frame only on mobile

-- Refined default palette = the out-of-box look (a saved theme from the Settings tab overrides it on load).
-- Neutral near-black panels + a single cool accent; wrapped in pcall so an API drift can't break the load.
pcall(function()
    Library.MainColor       = Color3.fromRGB(26, 27, 31)
    Library.BackgroundColor = Color3.fromRGB(17, 18, 21)
    Library.AccentColor     = Color3.fromRGB(96, 165, 250)
    Library.OutlineColor    = Color3.fromRGB(43, 45, 52)
    Library.FontColor       = Color3.fromRGB(239, 241, 245)
end)

local windowOptions = {
    Title = 'LuaHook',
    Center = true,
    AutoShow = false,
    TabPadding = 8,
    MenuFadeTime = 0.2,
    NotifySide = 'Right',          -- notifications slide in from the right (out of the way of the tab rail)
    Resizable = true,              -- PC + mobile can resize/rescale the window
    UnlockMouseWhileOpen = true,   -- free OS cursor while the menu is open (re-locks for gameplay on close)
}

local Window = Library:CreateWindow(windowOptions)

-- Lucide tab icons (mstudio45 fork: AddTab(Name, Icon)) for a cleaner, quicker-to-scan tab rail.
local Tabs = {
    Combat    = Window:AddTab('Combat',   'crosshair'),
    Rage      = Window:AddTab('Rage',     'swords'),
    ESP       = Window:AddTab('ESP',      'eye'),
    Visuals   = Window:AddTab('Visuals',  'sparkles'),
    HUD       = Window:AddTab('HUD',      'activity'),
    Settings  = Window:AddTab('Settings', 'settings'),
}
local Options = Library.Options or {}
local Toggles = Library.Toggles or {}
Library.Options = Options
Library.Toggles = Toggles

-- Sync background blur with window visibility
do
    local function findBlur()
        if Library.Window and Library.Window.Parent then
            for _, child in ipairs(Library.Window.Parent:GetChildren()) do
                if child ~= Library.Window and child:IsA("Frame") and child:FindFirstChildOfClass("BlurEffect") then
                    return child
                end
            end
        end
    end
    local blur = findBlur()
    if blur then
        blur.Visible = Library.Window.Visible
        Library.Window:GetPropertyChangedSignal("Visible"):Connect(function()
            blur.Visible = Library.Window.Visible
        end)
    end
end

-- ── COMBAT TAB ─────────────────────────────────────────────────────────────
do
    local L = Tabs.Combat:AddLeftGroupbox('Silent Aim')
    L:AddToggle('SilentAim', { Text='Silent Aim', Default=Config.SilentAim,
        Callback=function(v) Config.SilentAim = v end })
        :AddKeyPicker('SilentAimKey', { Default='None', Mode='Toggle', SyncToggleState=true, Text='Silent Aim' })
    L:AddToggle('SilentAimVisCheck', { Text='Visibility check', Default=Config.SilentAimVisCheck,
        Callback=function(v) Config.SilentAimVisCheck = v end })
    L:AddToggle('SilentAimJitter', { Text='Hit randomization', Default=Config.SilentAimJitter,
        Callback=function(v) Config.SilentAimJitter = v end })
    L:AddSlider('SilentAimFOV', { Text='FOV radius', Default=Config.SilentAimFOV,
        Min=20, Max=2000, Rounding=0, Callback=function(v) Config.SilentAimFOV = v end })
    L:AddDropdown('SilentAimTargetPart', { Values={'Head','Torso','Closest'},
        Default=Config.SilentAimTargetPart, Text='Target bone',
        Callback=function(v) Config.SilentAimTargetPart = v end })
    L:AddSlider('SilentAimStickiness', { Text='Stickiness', Default=Config.SilentAimStickiness,
        Min=0, Max=0.5, Rounding=2,
        Callback=function(v) Config.SilentAimStickiness = v end })
    L:AddToggle('SilentAimMultipoint', { Text='Multipoint', Default=Config.SilentAimMultipoint,
        Callback=function(v) Config.SilentAimMultipoint = v end })
    local smpDep = L:AddDependencyBox()
    smpDep:AddSlider('SilentAimMultipointCount', { Text='Multipoint samples',
        Default=Config.SilentAimMultipointCount, Min=1, Max=12, Rounding=0,
        Callback=function(v) Config.SilentAimMultipointCount = math.floor(v) end })
    smpDep:SetupDependencies({ { Toggles.SilentAimMultipoint, true } })
    L:AddToggle('SilentAimTorsoFallback', { Text='Torso fallback', Default=Config.SilentAimTorsoFallback,
        Callback=function(v) Config.SilentAimTorsoFallback = v end })

    local R = Tabs.Combat:AddRightGroupbox('Aimbot')
    R:AddToggle('Aimbot', { Text='Aimbot', Default=Config.Aimbot,
        Callback=function(v) if v then Aimbot.enable() else Aimbot.disable() end end })
    -- SINGLE activation control = the actual run-gate. aimbotKeyDown() -> isInputActive(Config.AimbotKey)
    -- (05_utils). 'Always' = active whenever the aimbot is enabled (matches the visible toggle); MB2/MB1/key
    -- = hold-to-aim. This is the ONLY gate now — the old separate "Hold key" dropdown that silently forced
    -- M2-hold-to-aim even with the toggle ON is gone (default is 'Always', bug #1). The keypicker above is
    -- purely the enable/disable bind, consistent with every other subsystem.
    R:AddDropdown('AimbotKey', {
        Values={'Always','MB2','MB1','C','E','F','Q','V','X','LeftShift','LeftAlt','LeftControl'},
        Default=Config.AimbotKey, Text='Activation',
        Callback=function(v) Config.AimbotKey = v end })
    R:AddToggle('AimbotHardLock', { Text='Hard lock',
        Default=Config.AimbotHardLock,
        Callback=function(v) Config.AimbotHardLock = v end })
    R:AddToggle('AimbotShotOverride', { Text='Shot override',
        Default=Config.AimbotShotOverride,
        Callback=function(v) Config.AimbotShotOverride = v end })
    R:AddToggle('AimbotVisCheck', { Text='Visibility check', Default=Config.AimbotVisCheck,
        Callback=function(v) Config.AimbotVisCheck = v end })
    R:AddToggle('AimbotPrediction', { Text='Prediction',
        Default=Config.AimbotPrediction,
        Callback=function(v) Config.AimbotPrediction = v end })
    R:AddToggle('AimbotShowLock', { Text='Show lock indicator', Default=Config.AimbotShowLock,
        Callback=function(v) Config.AimbotShowLock = v end })
    -- Demoted per bug #4: the styled aim ring lives on the HUD tab (Aim Field · FOV ring). This is the
    -- plain debug circle; leave it off unless you want a raw radius readout alongside the styled ring.
    R:AddToggle('AimbotShowFOV', { Text='Debug FOV circle', Default=Config.AimbotShowFOV,
        Callback=function(v) Config.AimbotShowFOV = v end })
    R:AddSlider('AimbotSensMultiplier', { Text='Sensitivity multiplier',
        Default=Config.AimbotSensMultiplier, Min=0.5, Max=3.0, Rounding=2,
        Callback=function(v) Config.AimbotSensMultiplier = v end })
    R:AddSlider('AimbotSmooth', { Text='Smoothness',
        Default=Config.AimbotSmooth, Min=0, Max=0.99, Rounding=2,
        Callback=function(v) Config.AimbotSmooth = v end })
    R:AddSlider('AimbotFOV', { Text='FOV radius (px)', Default=Config.AimbotFOV,
        Min=20, Max=500, Rounding=0, Callback=function(v) Config.AimbotFOV = v end })
    R:AddSlider('AimbotDeadzone', { Text='Deadzone (px)', Default=Config.AimbotDeadzone,
        Min=0, Max=10, Rounding=1, Callback=function(v) Config.AimbotDeadzone = v end })
    R:AddSlider('AimbotSpeedCap', { Text='Speed cap', Default=Config.AimbotSpeedCap,
        Min=0, Max=500, Rounding=0, Callback=function(v) Config.AimbotSpeedCap = v end })
    R:AddSlider('AimbotStickiness', { Text='Stickiness', Default=Config.AimbotStickiness,
        Min=0, Max=0.5, Rounding=2, Callback=function(v) Config.AimbotStickiness = v end })
    R:AddDropdown('AimbotEasing', { Values={'Linear','EaseOut','EaseInOut'}, Default=Config.AimbotEasing,
        Text='Easing curve', Callback=function(v) Config.AimbotEasing = v end })
    R:AddDropdown('AimbotTargetPart', { Values={'Head','Torso','Closest'},
        Default=Config.AimbotTargetPart, Text='Target bone',
        Callback=function(v) Config.AimbotTargetPart = v end })
    R:AddDropdown('AimbotFOVMode', { Values={'Pixels','Degrees'}, Default=Config.AimbotFOVMode,
        Text='FOV mode', Callback=function(v) Config.AimbotFOVMode = v end })
    local fovDegDep = R:AddDependencyBox()
    fovDegDep:AddSlider('AimbotFOVDegrees', { Text='FOV (degrees)', Default=Config.AimbotFOVDegrees,
        Min=1, Max=45, Rounding=1, Callback=function(v) Config.AimbotFOVDegrees = v end })
    fovDegDep:SetupDependencies({ { Options.AimbotFOVMode, 'Degrees' } })
    R:AddSlider('AimbotAccelTime', { Text='Accel time (s)', Default=Config.AimbotAccelTime,
        Min=0.01, Max=0.6, Rounding=2,
        Callback=function(v) Config.AimbotAccelTime = v end })
    R:AddSlider('AimbotReactionMs', { Text='Reaction delay (ms)', Default=Config.AimbotReactionMs,
        Min=0, Max=300, Rounding=0, Callback=function(v) Config.AimbotReactionMs = v end })
    R:AddSlider('AimbotNoise', { Text='Aim noise (px)', Default=Config.AimbotNoise,
        Min=0, Max=10, Rounding=1, Callback=function(v) Config.AimbotNoise = v end })
    R:AddSlider('AimbotOvershoot', { Text='Overshoot', Default=Config.AimbotOvershoot,
        Min=0, Max=1, Rounding=2, Callback=function(v) Config.AimbotOvershoot = v end })
    R:AddSlider('AimbotSwitchThreshold', { Text='Switch threshold (px)', Default=Config.AimbotSwitchThreshold,
        Min=0, Max=200, Rounding=0,
        Callback=function(v) Config.AimbotSwitchThreshold = v end })
    R:AddToggle('AimbotAutoFire', { Text='Auto fire', Default=Config.AimbotAutoFire,
        Callback=function(v) Config.AimbotAutoFire = v end })
    local afDep = R:AddDependencyBox()
    afDep:AddSlider('AimbotAutoFireFOV', { Text='Auto-fire FOV (px)', Default=Config.AimbotAutoFireFOV,
        Min=1, Max=100, Rounding=0, Callback=function(v) Config.AimbotAutoFireFOV = v end })
    afDep:AddSlider('AimbotAutoFireDelay', { Text='Auto-fire delay (ms)', Default=Config.AimbotAutoFireDelay,
        Min=0, Max=500, Rounding=0, Callback=function(v) Config.AimbotAutoFireDelay = v end })
    afDep:SetupDependencies({ { Toggles.AimbotAutoFire, true } })

    local L2 = Tabs.Combat:AddLeftGroupbox('Targeting')
    L2:AddToggle('TeamCheck', { Text='Team check', Default=Config.TeamCheck,
        Callback=function(v) Config.TeamCheck = v end })
    L2:AddToggle('AvoidDeflect', { Text='Avoid deflecting katanas', Default=Config.AvoidDeflect,
        Callback=function(v) Config.AvoidDeflect = v end })
    L2:AddToggle('PredictiveLead', { Text='Predictive lead', Default=Config.PredictiveLead,
        Callback=function(v) Config.PredictiveLead = v end })
    L2:AddSlider('MaxDistance', { Text='Max distance', Default=Config.MaxDistance,
        Min=200, Max=3000, Rounding=0, Callback=function(v) Config.MaxDistance = v end })
    L2:AddSlider('LeadCap', { Text='Lead cap', Default=Config.LeadCap,
        Min=1, Max=50, Rounding=0, Callback=function(v) Config.LeadCap = v end })
    L2:AddToggle('ProjectileLead', { Text='Projectile lead', Default=Config.ProjectileLead,
        Callback=function(v) Config.ProjectileLead = v end })
    local projDep = L2:AddDependencyBox()
    projDep:AddSlider('ProjectileSpeed', { Text='Projectile speed', Default=Config.ProjectileSpeed,
        Min=50, Max=2000, Rounding=0, Callback=function(v) Config.ProjectileSpeed = v end })
    projDep:SetupDependencies({ { Toggles.ProjectileLead, true } })
    L2:AddSlider('ServerProcessingMs', { Text='Server processing (ms)', Default=Config.ServerProcessingMs,
        Min=0, Max=200, Rounding=0,
        Callback=function(v) Config.ServerProcessingMs = v end })
end

-- ── RAGE TAB ───────────────────────────────────────────────────────────────
-- 3-mode rage: Polar (proven Kicia-killer, point-blank multi-tap), Apex (Polar + opt-in desync layers, all
-- default off), Orbit (stepped side-vantage vs normal players). The mode dropdown selects the tick; per-mode
-- dependency boxes show only the active mode's knobs. IIFE-wrapped so the tab locals get their own proto.
do (function()
    local L = Tabs.Rage:AddLeftGroupbox('Master')
    L:AddToggle('Rage', { Text='Enable Rage', Default=Config.Rage,
        Callback=function(v) if v then Rage.enable() else Rage.disable() end end })
        :AddKeyPicker('RageToggleKey', { Default='None', Mode='Toggle', SyncToggleState=true, Text='Rage' })
    L:AddDropdown('RageMode', { Values={'Polar','Apex','Orbit','Phantom'},
        Default=Config.RageMode, Text='Rage mode',
        Callback=function(v) Config.RageMode = v end })
    L:AddToggle('RageDirectFire', { Text='Forged multi-tap fire (off = natural, weaker)',
        Default=Config.RageDirectFire,
        Callback=function(v) Config.RageDirectFire = v end })
    L:AddToggle('RageSkipImmune', { Text='Skip immune targets', Default=Config.RageSkipImmune,
        Callback=function(v) Config.RageSkipImmune = v end })
    L:AddToggle('RageHPPriority', { Text='HP priority targeting', Default=Config.RageHPPriority,
        Callback=function(v) Config.RageHPPriority = v end })
    L:AddToggle('RageFastTargetSwitch', { Text='Fast target switch',
        Default=Config.RageFastTargetSwitch,
        Callback=function(v) Config.RageFastTargetSwitch = v end })

    -- APEX dep box (shown only when RageMode == 'Apex'). Strict-superset Polar: same-frame acquire + head-sane
    -- swing + fire-through-economy + tap constants. No prediction/void-duty-cycle layers.
    local apexDep = L:AddDependencyBox()
    apexDep:AddToggle('RageInEngineAcquire', { Text='In-engine acquire (same-frame)',
        Default=Config.RageInEngineAcquire, Callback=function(v) Config.RageInEngineAcquire = v end })
    apexDep:AddToggle('RageHeadSaneFirst', { Text='Swing to surfaced target (lobby)',
        Default=Config.RageHeadSaneFirst, Callback=function(v) Config.RageHeadSaneFirst = v end })
    apexDep:AddSlider('RageTargetDwell', { Text='Target dwell (frames)',
        Default=Config.RageTargetDwell, Min=1, Max=30, Rounding=0,
        Callback=function(v) Config.RageTargetDwell = math.floor(v) end })
    apexDep:AddSlider('RageSwapAmmoWatermark', { Text='Pre-swap ammo watermark',
        Default=Config.RageSwapAmmoWatermark, Min=0, Max=10, Rounding=0,
        Callback=function(v) Config.RageSwapAmmoWatermark = math.floor(v) end })
    apexDep:AddToggle('RageReloadOnVoidOnly', { Text='Defer reload to void lull',
        Default=Config.RageReloadOnVoidOnly, Callback=function(v) Config.RageReloadOnVoidOnly = v end })
    apexDep:AddSlider('RageHeadMissGrace', { Text='Head-miss grace (frames)',
        Default=Config.RageHeadMissGrace, Min=0, Max=5, Rounding=0,
        Callback=function(v) Config.RageHeadMissGrace = math.floor(v) end })
    apexDep:AddSlider('RageTaps', { Text='Taps / sane frame (K)',
        Default=Config.RageTaps, Min=1, Max=12, Rounding=0,
        Callback=function(v) Config.RageTaps = math.floor(v) end })
    apexDep:AddSlider('RageTapsVoid', { Text='Taps / void frame (K)',
        Default=Config.RageTapsVoid, Min=1, Max=12, Rounding=0,
        Callback=function(v) Config.RageTapsVoid = math.floor(v) end })
    apexDep:SetupDependencies({ { Options.RageMode, 'Apex' } })

    -- ORBIT dep box (shown only when RageMode == 'Orbit')
    local orbitDep = L:AddDependencyBox()
    orbitDep:AddSlider('RageCombatOrbitRadius', { Text='Orbit radius',
        Default=Config.RageCombatOrbitRadius, Min=20, Max=380, Rounding=0,
        Callback=function(v) Config.RageCombatOrbitRadius = v end })
    orbitDep:AddSlider('RageOrbitDwell', { Text='Vantage time',
        Default=Config.RageOrbitDwell, Min=0.04, Max=0.2, Rounding=2,
        Callback=function(v) Config.RageOrbitDwell = v end })
    orbitDep:AddSlider('RageCombatOrbitHeight', { Text='Orbit height',
        Default=Config.RageCombatOrbitHeight, Min=0, Max=40, Rounding=0,
        Callback=function(v) Config.RageCombatOrbitHeight = v end })
    orbitDep:AddToggle('RageCombatOrbitJitter', { Text='Vertical jitter',
        Default=Config.RageCombatOrbitJitter,
        Callback=function(v) Config.RageCombatOrbitJitter = v end })
    orbitDep:AddSlider('RagePBEyeUp', { Text='Eye height',
        Default=Config.RagePBEyeUp, Min=1, Max=12, Rounding=0,
        Callback=function(v) Config.RagePBEyeUp = v end })
    orbitDep:SetupDependencies({ { Options.RageMode, 'Orbit' } })

    -- PHANTOM dep box (shown only when RageMode == 'Phantom'). Polar fire + detached-hitbox defense: our own
    -- hitbox parts are weld-severed and flung past MAX_RAYCAST so enemy shots miss while we keep killing.
    local phantomDep = L:AddDependencyBox()
    phantomDep:AddSlider('RageDetachDist', { Text='Detach distance (studs)',
        Default=Config.RageDetachDist, Min=400, Max=2000, Rounding=0,
        Callback=function(v) Config.RageDetachDist = math.floor(v) end })
    phantomDep:AddToggle('RageDetachJitter', { Text='Fling jitter',
        Default=Config.RageDetachJitter, Callback=function(v) Config.RageDetachJitter = v end })
    phantomDep:AddToggle('RageDetachKillFast', { Text='Kill-fast (fling only during burst)',
        Default=Config.RageDetachKillFast, Callback=function(v) Config.RageDetachKillFast = v end })
    phantomDep:AddToggle('RageDetachBodyOnly', { Text='Body-only (stay head-shottable)',
        Default=Config.RageDetachBodyOnly, Callback=function(v) Config.RageDetachBodyOnly = v end })
    phantomDep:SetupDependencies({ { Options.RageMode, 'Phantom' } })

    local F = Tabs.Rage:AddLeftGroupbox('Fire')
    F:AddSlider('RageFireRateOverride', { Text='Fire-rate (s, 0 = LEGAL self-throttle; >0 = force interval)',
        Default=Config.RageFireRateOverride, Min=0, Max=0.5, Rounding=3,
        Callback=function(v) Config.RageFireRateOverride = v end })
    F:AddSlider('RageEyeMuzzleSep', { Text='Eye/muzzle separation',
        Default=Config.RageEyeMuzzleSep, Min=0, Max=0.3, Rounding=2,
        Callback=function(v) Config.RageEyeMuzzleSep = v end })

    local W = Tabs.Rage:AddLeftGroupbox('Weapon')
    W:AddDropdown('RageWeaponPick', { Values={'Primary','Secondary','Melee'},
        Default=Config.RageWeaponPick, Text='Main weapon',
        Callback=function(v) Config.RageWeaponPick = v end })
    W:AddDropdown('RageOutOfAmmo', { Values={'Switch','Reload'},
        Default=Config.RageOutOfAmmo, Text='When out of ammo',
        Callback=function(v) Config.RageOutOfAmmo = v end })
    local swDep = W:AddDependencyBox()
    swDep:AddToggle('RageSwitchMelee', { Text='Include melee in switch',
        Default=Config.RageSwitchMelee,
        Callback=function(v) Config.RageSwitchMelee = v end })
    swDep:SetupDependencies({ { Options.RageOutOfAmmo, 'Switch' } })

    local B = Tabs.Rage:AddRightGroupbox('Melee')
    B:AddToggle('RageShieldBackstab', { Text='Riot shield bypass', Default=Config.RageShieldBackstab,
        Callback=function(v) Config.RageShieldBackstab = v end })
    B:AddToggle('RageKnifeBackstab', { Text='Force knife backstabs', Default=Config.RageKnifeBackstab,
        Callback=function(v) Config.RageKnifeBackstab = v end })

    local S = Tabs.Rage:AddRightGroupbox('Safety')
    S:AddSlider('RageSelfProtectHP', { Text='Self-protect HP %',
        Default=Config.RageSelfProtectHP, Min=0, Max=1, Rounding=2,
        Callback=function(v) Config.RageSelfProtectHP = v end })
    S:AddDivider()
    S:AddToggle('RageLab', { Text='Lab overlay', Default=Config.RageLab,
        Callback=function(v) if v then Lab.enable() else Lab.disable() end end })
    S:AddButton({ Text='Reset lab stats', Func=function() pcall(Lab.reset) end })
end)() end

-- ── ESP TAB ────────────────────────────────────────────────────────────────
do
    local L = Tabs.ESP:AddLeftGroupbox('ESP')

    -- Master toggle must be created first so dependency boxes can reference Toggles.ESP
    L:AddToggle('ESP', { Text='Enable ESP', Default=Config.ESP,
        Callback=function(v) if v then ESP.enable() else ESP.disable() end end })
        :AddKeyPicker('ESPToggleKey', { Default='None', Mode='Toggle', SyncToggleState=true, Text='ESP' })

    -- All ESP options are nested under the master ESP toggle
    local espDep = L:AddDependencyBox()

    espDep:AddToggle('ESPTeamCheck', { Text='Team check', Default=Config.ESPTeamCheck,
        Callback=function(v) Config.ESPTeamCheck = v end })

    espDep:AddToggle('ESPBox', { Text='Box', Default=Config.ESPBox,
        Callback=function(v) Config.ESPBox = v end })

    -- Box sub-options: only visible when Box is on
    local boxDep = espDep:AddDependencyBox()
    boxDep:AddDropdown('ESPBoxMode', { Values={'Dynamic','Static'}, Default=Config.ESPBoxMode,
        Text='Box source',
        Callback=function(v) Config.ESPBoxMode = v end })
    boxDep:AddToggle('ESPBoxBrackets', { Text='Corner style', Default=Config.ESPBoxBrackets,
        Callback=function(v) Config.ESPBoxBrackets = v end })
    boxDep:AddSlider('ESPCornerLength', { Text='Corner length %', Default=Config.ESPCornerLength,
        Min=0.05, Max=0.5, Rounding=2, Callback=function(v) Config.ESPCornerLength = v end })
    boxDep:AddToggle('ESPBoxFill', { Text='Box fill', Default=Config.ESPBoxFill,
        Callback=function(v) Config.ESPBoxFill = v end })
    boxDep:AddSlider('ESPBoxThickness', { Text='Box thickness', Default=Config.ESPBoxThickness,
        Min=1, Max=5, Rounding=0, Callback=function(v) Config.ESPBoxThickness = math.floor(v) end })
    boxDep:AddToggle('ESPBoxGradient', { Text='Gradient outline', Default=Config.ESPBoxGradient,
        Callback=function(v) Config.ESPBoxGradient = v end })
    boxDep:SetupDependencies({ { Toggles.ESPBox, true } })

    -- Gradient-outline palette: shown only when Box AND Gradient outline are both on.
    local boxGradDep = espDep:AddDependencyBox()
    boxGradDep:AddLabel('Grad A'):AddColorPicker('ESPBoxGradientA', { Default=Config.ESPBoxGradientA,
        Callback=function(v) Config.ESPBoxGradientA = v end })
    boxGradDep:AddLabel('Grad B'):AddColorPicker('ESPBoxGradientB', { Default=Config.ESPBoxGradientB,
        Callback=function(v) Config.ESPBoxGradientB = v end })
    boxGradDep:AddSlider('ESPBoxGradientSpeed', { Text='Grad speed', Default=Config.ESPBoxGradientSpeed,
        Min=0.02, Max=0.5, Rounding=2, Callback=function(v) Config.ESPBoxGradientSpeed = v end })
    boxGradDep:SetupDependencies({ { Toggles.ESPBox, true }, { Toggles.ESPBoxGradient, true } })

    espDep:AddToggle('ESPName', { Text='Name', Default=Config.ESPName,
        Callback=function(v) Config.ESPName = v end })
    espDep:AddToggle('ESPNameHealthUnderline', { Text='Health underline', Default=Config.ESPNameHealthUnderline,
        Callback=function(v) Config.ESPNameHealthUnderline = v end })
    espDep:AddToggle('ESPDistance', { Text='Distance', Default=Config.ESPDistance,
        Callback=function(v) Config.ESPDistance = v end })
    espDep:AddToggle('ESPWeapon', { Text='Weapon', Default=Config.ESPWeapon,
        Callback=function(v) Config.ESPWeapon = v end })

    espDep:AddToggle('ESPHealth', { Text='Health bar', Default=Config.ESPHealth,
        Callback=function(v) Config.ESPHealth = v end })

    -- Health sub-options: only visible when Health bar is on
    local hpDep = espDep:AddDependencyBox()
    hpDep:AddDropdown('ESPHealthOrientation', { Values={'Vertical','Horizontal'},
        Default=Config.ESPHealthOrientation, Text='Orientation',
        Callback=function(v) Config.ESPHealthOrientation = v end })
    hpDep:AddDropdown('ESPHealthNumberMode', { Values={'Off','OnDamage','Always'},
        Default=Config.ESPHealthNumberMode, Text='Number',
        Callback=function(v) Config.ESPHealthNumberMode = v end })
    hpDep:AddToggle('ESPHealthGradient', { Text='Health gradient', Default=Config.ESPHealthGradient,
        Callback=function(v) Config.ESPHealthGradient = v end })
    hpDep:AddToggle('ESPHealthSegTicks', { Text='Seg ticks', Default=Config.ESPHealthSegTicks,
        Callback=function(v) Config.ESPHealthSegTicks = v end })
    hpDep:SetupDependencies({ { Toggles.ESPHealth, true } })

    espDep:AddToggle('ESPSkeleton', { Text='Skeleton', Default=Config.ESPSkeleton,
        Callback=function(v) Config.ESPSkeleton = v end })
    espDep:AddToggle('ESPChams', { Text='Chams', Default=Config.ESPChams,
        Callback=function(v) Config.ESPChams = v end })
    local chamsDep = espDep:AddDependencyBox()
    chamsDep:AddDropdown('ESPChamsStyle', { Values={'Shade','Neon','Ghost'}, Default=Config.ESPChamsStyle,
        Text='Cham style', Callback=function(v) Config.ESPChamsStyle = v end })
    chamsDep:SetupDependencies({ { Toggles.ESPChams, true } })
    espDep:AddToggle('ESPTracers', { Text='Tracers', Default=Config.ESPTracers,
        Callback=function(v) Config.ESPTracers = v end })
    espDep:AddToggle('ESPHeadDot', { Text='Head dot', Default=Config.ESPHeadDot,
        Callback=function(v) Config.ESPHeadDot = v end })
    espDep:AddToggle('ESPArrows', { Text='Off-screen arrows', Default=Config.ESPArrows,
        Callback=function(v) Config.ESPArrows = v end })
    -- SIGNAL v4 · off-screen arrow polish (nested under the arrows toggle)
    local arrDep = espDep:AddDependencyBox()
    arrDep:AddToggle('ESPArrowDistFade', { Text='Distance fade', Default=Config.ESPArrowDistFade,
        Callback=function(v) Config.ESPArrowDistFade = v end })
    arrDep:AddToggle('ESPArrowDistLabel', { Text='Distance label', Default=Config.ESPArrowDistLabel,
        Callback=function(v) Config.ESPArrowDistLabel = v end })
    arrDep:SetupDependencies({ { Toggles.ESPArrows, true } })
    -- SIGNAL v4 · look-direction line (aim direction from the head; red when it sweeps toward you)
    espDep:AddToggle('ESPLookLine', { Text='Look line', Default=Config.ESPLookLine,
        Callback=function(v) Config.ESPLookLine = v end })
    espDep:AddToggle('ESPOutline', { Text='Outlines', Default=Config.ESPOutline,
        Callback=function(v) Config.ESPOutline = v end })

    espDep:SetupDependencies({ { Toggles.ESP, true } })

    -- Tuning: color mode, fonts, sizes, thickness, distance/cap
    local L2 = Tabs.ESP:AddLeftGroupbox('Tuning')
    L2:AddDropdown('ESPColorMode', { Values={'Static','Team','Visibility','Distance','Rainbow'},
        Default=Config.ESPColorMode, Text='Color mode',
        Callback=function(v) Config.ESPColorMode = v end })
    L2:AddDropdown('ESPFont', { Values={'UI','System','Plex','Monospace'}, Default=Config.ESPFont,
        Text='Font', Callback=function(v) Config.ESPFont = v end })
    L2:AddToggle('ESPTextScaling', { Text='Distance text scaling', Default=Config.ESPTextScaling,
        Callback=function(v) Config.ESPTextScaling = v end })
    L2:AddSlider('ESPTextSize', { Text='Name size', Default=Config.ESPTextSize,
        Min=8, Max=24, Rounding=0, Callback=function(v) Config.ESPTextSize = math.floor(v) end })
    L2:AddSlider('ESPInfoTextSize', { Text='Info size', Default=Config.ESPInfoTextSize,
        Min=8, Max=24, Rounding=0, Callback=function(v) Config.ESPInfoTextSize = math.floor(v) end })
    L2:AddSlider('ESPHealthTextSize', { Text='Health number size', Default=Config.ESPHealthTextSize,
        Min=8, Max=24, Rounding=0, Callback=function(v) Config.ESPHealthTextSize = math.floor(v) end })
    L2:AddSlider('ESPSkeletonThickness', { Text='Skeleton thickness', Default=Config.ESPSkeletonThickness,
        Min=1, Max=5, Rounding=0, Callback=function(v) Config.ESPSkeletonThickness = math.floor(v) end })
    L2:AddSlider('ESPTracerThickness', { Text='Tracer thickness', Default=Config.ESPTracerThickness,
        Min=1, Max=5, Rounding=0, Callback=function(v) Config.ESPTracerThickness = math.floor(v) end })
    L2:AddSlider('ESPHeadDotSize', { Text='Head dot size', Default=Config.ESPHeadDotSize,
        Min=1, Max=12, Rounding=0, Callback=function(v) Config.ESPHeadDotSize = math.floor(v) end })
    L2:AddDropdown('ESPNameMode', { Values={'Display','Username'}, Default=Config.ESPNameMode,
        Text='Name source', Callback=function(v) Config.ESPNameMode = v end })
    L2:AddDropdown('ESPTracerOrigin', { Values={'Top','Middle','Bottom','Mouse'},
        Default=Config.ESPTracerOrigin, Text='Tracer origin',
        Callback=function(v) Config.ESPTracerOrigin = v end })
    L2:AddToggle('ESPTracerGradient', { Text='Tracer gradient', Default=Config.ESPTracerGradient,
        Callback=function(v) Config.ESPTracerGradient = v end })
    L2:AddSlider('ESPMaxDistance', { Text='Max distance', Default=Config.ESPMaxDistance,
        Min=100, Max=3000, Rounding=0, Callback=function(v) Config.ESPMaxDistance = v end })
    L2:AddSlider('ESPMaxPlayers', { Text='Max players', Default=Config.ESPMaxPlayers,
        Min=0, Max=32, Rounding=0, Callback=function(v) Config.ESPMaxPlayers = math.floor(v) end })
    L2:AddSlider('ESPOutlineTransparency', { Text='Outline transp.', Default=Config.ESPOutlineTransparency,
        Min=0, Max=1, Rounding=2, Callback=function(v) Config.ESPOutlineTransparency = v end })
    L2:AddSlider('ESPLookLineLength', { Text='Look line length', Default=Config.ESPLookLineLength,
        Min=4, Max=16, Rounding=0, Callback=function(v) Config.ESPLookLineLength = math.floor(v) end })
    L2:AddDivider()
    L2:AddToggle('ESPSmoothing', { Text='Smoothing', Default=Config.ESPSmoothing,
        Callback=function(v) Config.ESPSmoothing = v end })
    L2:AddToggle('ESPFadeIn', { Text='Spawn fade', Default=Config.ESPFadeIn,
        Callback=function(v) Config.ESPFadeIn = v end })
    -- far-target legibility floors (0 / low = off)
    L2:AddSlider('ESPMinBoxHeight', { Text='Min box height (px)', Default=Config.ESPMinBoxHeight,
        Min=0, Max=60, Rounding=0, Callback=function(v) Config.ESPMinBoxHeight = math.floor(v) end })
    L2:AddSlider('ESPTextMinSize', { Text='Min text size (px)', Default=Config.ESPTextMinSize,
        Min=8, Max=18, Rounding=0, Callback=function(v) Config.ESPTextMinSize = math.floor(v) end })
    L2:AddToggle('ESPHealthSmooth', { Text='Smooth health', Default=Config.ESPHealthSmooth,
        Callback=function(v) Config.ESPHealthSmooth = v end })
    L2:AddToggle('ESPHealthGhost', { Text='Damage ghost', Default=Config.ESPHealthGhost,
        Callback=function(v) Config.ESPHealthGhost = v end })
    L2:AddToggle('ESPDeclutter', { Text='Declutter', Default=Config.ESPDeclutter,
        Callback=function(v) Config.ESPDeclutter = v end })
    L2:AddToggle('ESPTextOutline', { Text='Text outline', Default=Config.ESPTextOutline,
        Callback=function(v) Config.ESPTextOutline = v end })
    L2:AddToggle('ESPChamsVisSplit', { Text='Chams by visibility', Default=Config.ESPChamsVisSplit,
        Callback=function(v) Config.ESPChamsVisSplit = v end })

    -- Per-element colors (base in Static mode)
    local R = Tabs.ESP:AddRightGroupbox('Colors')
    R:AddLabel('Box'):AddColorPicker('ESPBoxColor', { Default=Config.ESPBoxColor,
        Callback=function(v) Config.ESPBoxColor = v end })
    R:AddLabel('Box fill'):AddColorPicker('ESPBoxFillColor', { Default=Config.ESPBoxFillColor,
        Callback=function(v) Config.ESPBoxFillColor = v end })
    R:AddLabel('Name'):AddColorPicker('ESPNameColor', { Default=Config.ESPNameColor,
        Callback=function(v) Config.ESPNameColor = v end })
    R:AddLabel('Health'):AddColorPicker('ESPHealthColor', { Default=Config.ESPHealthColor,
        Callback=function(v) Config.ESPHealthColor = v end })
    R:AddLabel('Skeleton'):AddColorPicker('ESPSkeletonColor', { Default=Config.ESPSkeletonColor,
        Callback=function(v) Config.ESPSkeletonColor = v end })
    R:AddLabel('Tracer'):AddColorPicker('ESPTracerColor', { Default=Config.ESPTracerColor,
        Callback=function(v) Config.ESPTracerColor = v end })
    R:AddLabel('Head dot'):AddColorPicker('ESPHeadDotColor', { Default=Config.ESPHeadDotColor,
        Callback=function(v) Config.ESPHeadDotColor = v end })
    R:AddLabel('Chams fill'):AddColorPicker('ESPChamsFillColor', { Default=Config.ESPChamsFillColor,
        Callback=function(v) Config.ESPChamsFillColor = v end })
    R:AddLabel('Chams outline'):AddColorPicker('ESPChamsOutlineColor', { Default=Config.ESPChamsOutlineColor,
        Callback=function(v) Config.ESPChamsOutlineColor = v end })

    -- Color-mode palette + per-element transparency
    local R2 = Tabs.ESP:AddRightGroupbox('Mode colors')
    R2:AddLabel('Enemy'):AddColorPicker('ColorEnemy', { Default=Config.ColorEnemy,
        Callback=function(v) Config.ColorEnemy = v end })
    R2:AddLabel('Team'):AddColorPicker('ColorTeam', { Default=Config.ColorTeam,
        Callback=function(v) Config.ColorTeam = v end })
    R2:AddLabel('Enemy occluded'):AddColorPicker('ColorEnemyOcc', { Default=Config.ColorEnemyOcc,
        Callback=function(v) Config.ColorEnemyOcc = v end })
    R2:AddLabel('Team occluded'):AddColorPicker('ColorTeamOcc', { Default=Config.ColorTeamOcc,
        Callback=function(v) Config.ColorTeamOcc = v end })
    R2:AddLabel('Visible'):AddColorPicker('ColorVisible', { Default=Config.ColorVisible,
        Callback=function(v) Config.ColorVisible = v end })
    R2:AddLabel('Distance near'):AddColorPicker('ESPDistNearColor', { Default=Config.ESPDistNearColor,
        Callback=function(v) Config.ESPDistNearColor = v end })
    R2:AddLabel('Distance far'):AddColorPicker('ESPDistFarColor', { Default=Config.ESPDistFarColor,
        Callback=function(v) Config.ESPDistFarColor = v end })
    R2:AddSlider('ESPBoxTransparency', { Text='Box transp.', Default=Config.ESPBoxTransparency,
        Min=0, Max=1, Rounding=2, Callback=function(v) Config.ESPBoxTransparency = v end })
    R2:AddSlider('ESPBoxFillTransparency', { Text='Box fill transp.', Default=Config.ESPBoxFillTransparency,
        Min=0, Max=1, Rounding=2, Callback=function(v) Config.ESPBoxFillTransparency = v end })
    R2:AddSlider('ESPNameTransparency', { Text='Text transp.', Default=Config.ESPNameTransparency,
        Min=0, Max=1, Rounding=2, Callback=function(v) Config.ESPNameTransparency = v end })
    R2:AddSlider('ESPHealthTransparency', { Text='Health transp.', Default=Config.ESPHealthTransparency,
        Min=0, Max=1, Rounding=2, Callback=function(v) Config.ESPHealthTransparency = v end })
    R2:AddSlider('ESPSkeletonTransparency', { Text='Skeleton transp.', Default=Config.ESPSkeletonTransparency,
        Min=0, Max=1, Rounding=2, Callback=function(v) Config.ESPSkeletonTransparency = v end })
    R2:AddSlider('ESPTracerTransparency', { Text='Tracer transp.', Default=Config.ESPTracerTransparency,
        Min=0, Max=1, Rounding=2, Callback=function(v) Config.ESPTracerTransparency = v end })
    R2:AddSlider('ESPHeadDotTransparency', { Text='Head dot transp.', Default=Config.ESPHeadDotTransparency,
        Min=0, Max=1, Rounding=2, Callback=function(v) Config.ESPHeadDotTransparency = v end })
    R2:AddSlider('ESPChamsFillTransparency', { Text='Chams fill transp.', Default=Config.ESPChamsFillTransparency,
        Min=0, Max=1, Rounding=2, Callback=function(v) Config.ESPChamsFillTransparency = v end })
    R2:AddSlider('ESPChamsOutlineTransparency', { Text='Chams outline transp.', Default=Config.ESPChamsOutlineTransparency,
        Min=0, Max=1, Rounding=2, Callback=function(v) Config.ESPChamsOutlineTransparency = v end })
    R2:AddButton({ Text='Rebuild all ESP', Func=function() pcall(ESP.rebuildAll) end })

    -- Radar
    local R3 = Tabs.ESP:AddRightGroupbox('Radar')
    R3:AddToggle('ESPRadar', { Text='Enable radar', Default=Config.ESPRadar,
        Callback=function(v) Config.ESPRadar = v end })
    local radarDep = R3:AddDependencyBox()
    radarDep:AddSlider('ESPRadarSize', { Text='Size', Default=Config.ESPRadarSize,
        Min=140, Max=280, Rounding=0, Callback=function(v) Config.ESPRadarSize = math.floor(v) end })
    radarDep:AddSlider('ESPRadarRange', { Text='Range', Default=Config.ESPRadarRange,
        Min=50, Max=400, Rounding=0, Callback=function(v) Config.ESPRadarRange = math.floor(v) end })
    radarDep:AddSlider('ESPRadarInset', { Text='Inset', Default=Config.ESPRadarInset,
        Min=0, Max=80, Rounding=0, Callback=function(v) Config.ESPRadarInset = math.floor(v) end })
    radarDep:AddToggle('ESPRadarRotate', { Text='Rotate with camera', Default=Config.ESPRadarRotate,
        Callback=function(v) Config.ESPRadarRotate = v end })
    radarDep:AddToggle('ESPRadarVisSplit', { Text='Hollow = occluded', Default=Config.ESPRadarVisSplit,
        Callback=function(v) Config.ESPRadarVisSplit = v end })
    -- SIGNAL v4 · grid + sonar sweep
    radarDep:AddToggle('ESPRadarGrid', { Text='Grid', Default=Config.ESPRadarGrid,
        Callback=function(v) Config.ESPRadarGrid = v end })
    radarDep:AddToggle('ESPRadarSweep', { Text='Sweep', Default=Config.ESPRadarSweep,
        Callback=function(v) Config.ESPRadarSweep = v end })
    radarDep:SetupDependencies({ { Toggles.ESPRadar, true } })

    -- Alerts
    local R4 = Tabs.ESP:AddRightGroupbox('Alerts')
    R4:AddToggle('ESPPeekAlert', { Text='Peek alert', Default=Config.ESPPeekAlert,
        Callback=function(v) Config.ESPPeekAlert = v end })
    R4:AddToggle('ESPFacingIndicator', { Text='Facing', Default=Config.ESPFacingIndicator,
        Callback=function(v) Config.ESPFacingIndicator = v end })
    R4:AddToggle('ESPThreatCount', { Text='Off-screen count', Default=Config.ESPThreatCount,
        Callback=function(v) Config.ESPThreatCount = v end })
    R4:AddToggle('ESPPrimaryEmphasis', { Text='Primary target', Default=Config.ESPPrimaryEmphasis,
        Callback=function(v) Config.ESPPrimaryEmphasis = v end })
    R4:AddToggle('ESPLockChevron', { Text='Lock chevron', Default=Config.ESPLockChevron,
        Callback=function(v) Config.ESPLockChevron = v end })
    R4:AddToggle('ESPHealTick', { Text='Heal tick', Default=Config.ESPHealTick,
        Callback=function(v) Config.ESPHealTick = v end })

    -- ── Utility ESP: thrown gadgets (10b_utility_esp.lua) ──
    local R5 = Tabs.ESP:AddRightGroupbox('Utility ESP')
    R5:AddToggle('UtilityESP', { Text='Thrown gadgets', Default=Config.UtilityESP,
        Callback=function(v) if v then UtilESP.enable() else UtilESP.disable() end end })
    R5:AddSlider('UtilityESPMaxDistance', { Text='Max distance', Default=Config.UtilityESPMaxDistance,
        Min=50, Max=600, Rounding=0, Callback=function(v) Config.UtilityESPMaxDistance = v end })
    R5:AddToggle('UtilityESPRing', { Text='Danger pulse ring', Default=Config.UtilityESPRing,
        Callback=function(v) Config.UtilityESPRing = v end })
    R5:AddToggle('UtilityESPLabels', { Text='Name + distance tag', Default=Config.UtilityESPLabels,
        Callback=function(v) Config.UtilityESPLabels = v end })
end

-- ── VISUALS TAB ────────────────────────────────────────────────────────────
do
    local L = Tabs.Visuals:AddLeftGroupbox('World & Lighting')
    L:AddToggle('Visuals', { Text='Enable Visuals', Default=Config.Visuals,
        Callback=function(v) if v then Visuals.enable() else Visuals.disable() end end })
        :AddKeyPicker('VisualsToggleKey', { Default='None', Mode='Toggle', SyncToggleState=true, Text='Visuals' })
    L:AddDropdown('VisualsPreset', { Values=Visuals.PresetOrder,
        Default=Config.VisualsPreset, Text='Preset',
        Callback=function(v) Visuals.setPreset(v) end })
    L:AddDivider()
    L:AddDropdown('VisualsGrade', { Values={'None','Crisp','Cold','Warm','Comp'},
        Default=Config.VisualsGrade, Text='Grade',
        Callback=function(v) if Visuals.setGrade then Visuals.setGrade(v) else Config.VisualsGrade = v end end })
    L:AddSlider('VisualsGradeStrength', { Text='Grade strength', Default=Config.VisualsGradeStrength,
        Min=0, Max=1, Rounding=2,
        Callback=function(v) if Visuals.setGradeStrength then Visuals.setGradeStrength(v) else Config.VisualsGradeStrength = v end end })
    L:AddToggle('VisualsBloom', { Text='Bloom', Default=Config.VisualsBloom,
        Callback=function(v) Visuals.setBloom(v) end })
    L:AddSlider('VisualsBloomIntensity', { Text='Bloom intensity', Default=Config.VisualsBloomIntensity,
        Min=0, Max=3, Rounding=1,
        Callback=function(v) Visuals.setBloomIntensity(v) end })
    L:AddDivider()
    L:AddToggle('VisualsFullbright', { Text='Fullbright', Default=Config.VisualsFullbright,
        Callback=function(v) Visuals.toggleFullbright(v) end })
    L:AddToggle('VisualsNoFog', { Text='No fog', Default=Config.VisualsNoFog,
        Callback=function(v) Visuals.toggleNoFog(v) end })
    L:AddToggle('VisualsRainbowMap', { Text='Rainbow map', Default=Config.VisualsRainbowMap,
        Callback=function(v) Visuals.toggleRainbowMap(v) end })
    L:AddSlider('VisualsRainbowMapSpeed', { Text='Rainbow speed',
        Default=Config.VisualsRainbowMapSpeed, Min=0.01, Max=1, Rounding=2,
        Callback=function(v) Config.VisualsRainbowMapSpeed = v end })
    L:AddDivider()
    L:AddToggle('VisualsPerformanceMode', { Text='Performance mode',
        Default=Config.VisualsPerformanceMode,
        Callback=function(v) Visuals.togglePerf(v) end })

    local LH = Tabs.Visuals:AddLeftGroupbox('Holograms')
    LH:AddToggle('VisualsHolograms', { Text='On-hit holograms', Default=Config.VisualsHolograms,
        Callback=function(v) Visuals.toggleHolograms(v) end })
    LH:AddSlider('VisualsHologramDuration', { Text='Duration',
        Default=Config.VisualsHologramDuration, Min=0.5, Max=10, Rounding=1,
        Callback=function(v) Config.VisualsHologramDuration = v end })
    LH:AddSlider('VisualsHologramRange', { Text='Range',
        Default=Config.VisualsHologramRange, Min=50, Max=1000, Rounding=0,
        Callback=function(v) Config.VisualsHologramRange = v end })
    LH:AddSlider('VisualsHologramVisibility', { Text='Visibility',
        Default=Config.VisualsHologramVisibility, Min=0.2, Max=2, Rounding=1,
        Callback=function(v) Config.VisualsHologramVisibility = v end })
    LH:AddLabel('Core color'):AddColorPicker('VisualsHologramColor', {
        Default=Config.VisualsHologramColor,
        Callback=function(v) Config.VisualsHologramColor = v end })
    LH:AddLabel('Halo color'):AddColorPicker('VisualsHologramAccent', {
        Default=Config.VisualsHologramAccent,
        Callback=function(v) Config.VisualsHologramAccent = v end })
    LH:AddDropdown('VisualsHologramStyle', { Values={'Orb','Skeleton','Wraith'},
        Default=Config.VisualsHologramStyle, Text='Style',
        Callback=function(v) if Visuals.setHologramStyle then Visuals.setHologramStyle(v) else Config.VisualsHologramStyle = v end end })
    LH:AddToggle('VisualsHologramLethal', { Text='Gold on kill', Default=Config.VisualsHologramLethal,
        Callback=function(v) Config.VisualsHologramLethal = v end })
    LH:AddLabel('Kill color'):AddColorPicker('VisualsHologramLethalColor', {
        Default=Config.VisualsHologramLethalColor,
        Callback=function(v) Config.VisualsHologramLethalColor = v end })

    local R = Tabs.Visuals:AddRightGroupbox('Camera')
    R:AddSlider('VisualsStretch', { Text='Screen stretch', Default=Config.VisualsStretch,
        Min=Config.VisualsStretchMin, Max=Config.VisualsStretchMax, Rounding=2,
        Callback=function(v) Visuals.setStretch(v) end })
	R:AddButton({ Text='Reset stretch to 1.0', Func=function()
		Visuals.setStretch(1.0)
		if Library.Options and Library.Options.VisualsStretch then
			Library.Options.VisualsStretch:SetValue(1.0)
		end
	end })
    R:AddToggle('VisualsCameraSway', { Text='Camera sway', Default=Config.VisualsCameraSway,
        Callback=function(v) Config.VisualsCameraSway = v end })
    R:AddSlider('VisualsCameraSwayAmount', { Text='Sway amount', Default=Config.VisualsCameraSwayAmount,
        Min=0, Max=1, Rounding=2, Callback=function(v) Config.VisualsCameraSwayAmount = v end })
    R:AddDivider()
    R:AddToggle('VisualsVignette', { Text='Cinematic vignette', Default=Config.VisualsVignette,
        Callback=function(v) Visuals.setVignette(v) end })
    R:AddSlider('VisualsVignetteStrength', { Text='Vignette strength',
        Default=Config.VisualsVignetteStrength, Min=0, Max=1, Rounding=2,
        Callback=function(v) Visuals.setVignetteStrength(v) end })
    R:AddToggle('VisualsLetterbox', { Text='Letterbox', Default=Config.VisualsLetterbox,
        Callback=function(v) Visuals.setLetterbox(v) end })
    R:AddSlider('VisualsLetterboxSize', { Text='Bar size', Default=Config.VisualsLetterboxSize,
        Min=0.04, Max=0.18, Rounding=2,
        Callback=function(v) Visuals.setLetterboxSize(v) end })
    R:AddDivider()
    R:AddToggle('VisualsDOF', { Text='Depth of field', Default=Config.VisualsDOF,
        Callback=function(v) Visuals.setDOF(v) end })
    R:AddSlider('VisualsDOFDistance', { Text='Focus distance', Default=Config.VisualsDOFDistance,
        Min=5, Max=100, Rounding=0,
        Callback=function(v) Visuals.setDOFDistance(v) end })
    R:AddSlider('VisualsDOFBlur', { Text='Blur strength', Default=Config.VisualsDOFBlur,
        Min=0, Max=1, Rounding=2,
        Callback=function(v) Visuals.setDOFBlur(v) end })

    -- ── Environment (06b_weather.lua): weather core · sky events · sky ──
    local R2 = Tabs.Visuals:AddRightGroupbox('Environment')
    R2:AddToggle('Weather', { Text='Weather', Default=Config.Weather,
        Callback=function(v) if v then Weather.enableWeather() else Weather.disableWeather() end end })
        :AddKeyPicker('WeatherToggleKey', { Default='None', Mode='Toggle', SyncToggleState=true, Text='Weather' })
    R2:AddDropdown('WeatherType', { Values=Weather.TypeOrder, Default=Config.WeatherType, Text='Type',
        Callback=function(v) Weather.setType(v) end })
    R2:AddSlider('WeatherIntensity', { Text='Intensity', Default=Config.WeatherIntensity,
        Tooltip='Ambient density only (how thick the snow / how hard the rain). Sky events have their own rate dials.',
        Min=0.15, Max=2, Rounding=2, Callback=function(v) Weather.setIntensity(v) end })
    R2:AddToggle('WeatherMood', { Text='Mood tint', Default=Config.WeatherMood,
        Callback=function(v) Weather.toggleMood(v) end })
    R2:AddSlider('WeatherSoundVolume', { Text='Ambient volume', Default=Config.WeatherSoundVolume,
        Min=0, Max=1, Rounding=2, Callback=function(v) Weather.setSoundVolume(v) end })
    R2:AddDivider()
    -- sky events (independent schedulers under the Weather master, all default OFF)
    R2:AddToggle('WeatherStorm', { Text='Storm (lightning)', Default=Config.WeatherStorm,
        Callback=function(v) Weather.toggleStorm(v) end })
    R2:AddToggle('WeatherStormFlash', { Text='  ↳ sky flash', Default=Config.WeatherStormFlash,
        Callback=function(v) Config.WeatherStormFlash = v end })
    -- storm frequency = its existing gap sliders (seconds, so the direction reads off the label)
    local stormDep = R2:AddDependencyBox()
    stormDep:AddSlider('WeatherStormMin', { Text='Bolt gap (s)', Default=Config.WeatherStormMin,
        Min=1, Max=30, Rounding=0, Callback=function(v) Weather.setStormMin(v) end })
    stormDep:AddSlider('WeatherStormVar', { Text='  ↳ random extra (s)', Default=Config.WeatherStormVar,
        Min=0, Max=30, Rounding=0, Callback=function(v) Weather.setStormVar(v) end })
    stormDep:SetupDependencies({ { Toggles.WeatherStorm, true } })
    R2:AddToggle('WeatherMeteors', { Text='Meteor shower', Default=Config.WeatherMeteors,
        Callback=function(v) Weather.toggleMeteors(v) end })
    local metDep = R2:AddDependencyBox()
    metDep:AddSlider('WeatherMeteorRate', { Text='Meteor rate', Default=Config.WeatherMeteorRate,
        Tooltip='How often meteors fall — independent of Intensity. 1 = stock (~8.5s far / ~19s near).',
        Min=0.25, Max=3, Rounding=2, Callback=function(v) Weather.setMeteorRate(v) end })
    metDep:SetupDependencies({ { Toggles.WeatherMeteors, true } })
    R2:AddToggle('WeatherShootingStars', { Text='Shooting stars', Default=Config.WeatherShootingStars,
        Callback=function(v) Weather.toggleShootingStars(v) end })
    local starDep = R2:AddDependencyBox()
    starDep:AddSlider('WeatherStarRate', { Text='Star rate', Default=Config.WeatherStarRate,
        Tooltip='How often streaks appear — independent of Intensity. 1 = stock (~8s between events).',
        Min=0.25, Max=3, Rounding=2, Callback=function(v) Weather.setStarRate(v) end })
    starDep:SetupDependencies({ { Toggles.WeatherShootingStars, true } })
    R2:AddToggle('WeatherPuddles', { Text='Interactive puddles', Default=Config.WeatherPuddles,
        Tooltip='Wet ground patches that ripple with the rain and splash when someone runs through (Rain type or Storm).',
        Callback=function(v) Weather.togglePuddles(v) end })
    R2:AddDivider()
    -- sky: skybox swap + celestial + rays + rainbow + day/night dial
    R2:AddDropdown('SkyboxPreset', { Values=Weather.SkyboxOrder, Default=Config.SkyboxPreset, Text='Skybox',
        Callback=function(v) Weather.setSkybox(v) end })
    R2:AddToggle('SkyboxHideCelestial', { Text='Hide sun/moon/stars', Default=Config.SkyboxHideCelestial,
        Callback=function(v) Weather.toggleCelestial(v) end })
    R2:AddToggle('WeatherGodRays', { Text='God rays', Default=Config.WeatherGodRays,
        Callback=function(v) Weather.toggleGodRays(v) end })
    R2:AddToggle('WeatherRainbow', { Text='Rainbow', Default=Config.WeatherRainbow,
        Callback=function(v) Weather.toggleRainbow(v) end })
    R2:AddToggle('WeatherClockDial', { Text='Day/night cycle', Default=Config.WeatherClockDial,
        Tooltip='Drives ClockTime while the Visuals lighting master is OFF.',
        Callback=function(v) Weather.toggleClock(v) end })
    local clkDep = R2:AddDependencyBox()
    clkDep:AddSlider('WeatherClockCycleMin', { Text='Cycle (min)', Default=Config.WeatherClockCycleMin,
        Min=1, Max=30, Rounding=0, Callback=function(v) Config.WeatherClockCycleMin = math.floor(v) end })
    clkDep:SetupDependencies({ { Toggles.WeatherClockDial, true } })
end

-- ── HUD TAB ────────────────────────────────────────────────────────────────
-- HUD is an independent master (decoupled from the Visuals/World master). Its toggle
-- drives Config.HUD via Visuals.enableHUD/disableHUD; every feature groupbox nests under it.
do
    -- Master toggle first. It drives ONLY Visuals.enableHUD/disableHUD (subsystem on/off).
    -- Feature settings live directly in their groupboxes so they stay visible/configurable
    -- regardless of master state (only genuine per-feature options nest under their own toggle).
    -- NOTE: no SyncToggleState on the keybind — with a default-on master + a 'None' key it would
    -- force the master OFF at build time and collapse every setting (the original HUD-tab bug).
    local M = Tabs.HUD:AddLeftGroupbox('Master')
    M:AddToggle('HUD', { Text='Enable HUD', Default=Config.HUD,
        Callback=function(v)
            if v then Visuals.enableHUD(); pcall(HUDPlus.start)
            else Visuals.disableHUD(); pcall(HUDPlus.stop) end
        end })
        :AddKeyPicker('HUDToggleKey', { Default='None', Mode='Toggle', Text='HUD' })
    -- Bug #2: explicitly wire the HUD hotkey. SyncToggleState is intentionally omitted (with the default-on
    -- master + a 'None' key it would force the master OFF at build — the original HUD-tab regression), so
    -- nothing was polling the keypicker and the bind was dead. Flip the master TOGGLE on the bound key; its
    -- callback runs enableHUD/disableHUD, keeping Toggle + Config in sync (no double-toggle).
    UserInputService.InputBegan:Connect(function(input, gpe)
        if gpe then return end
        local kp = Options.HUDToggleKey
        if kp and kp.Value and kp.Value ~= 'None'
           and Enum.KeyCode[kp.Value] and input.KeyCode == Enum.KeyCode[kp.Value] then
            if Toggles.HUD then pcall(function() Toggles.HUD:SetValue(not Toggles.HUD.Value) end) end
        end
    end)
    -- SIGNAL v4 · watermark pill + active-features list
    M:AddToggle('HUDWatermark', { Text='Watermark', Default=Config.HUDWatermark,
        Callback=function(v) Config.HUDWatermark = v end })
    local wmDep = M:AddDependencyBox()
    wmDep:AddToggle('HUDWatermarkStats', { Text='Show fps/ping/kills', Default=Config.HUDWatermarkStats,
        Callback=function(v) Config.HUDWatermarkStats = v end })
    wmDep:SetupDependencies({ { Toggles.HUDWatermark, true } })
    M:AddToggle('HUDBindList', { Text='Active features list', Default=Config.HUDBindList,
        Callback=function(v) Config.HUDBindList = v end })
    local blDep = M:AddDependencyBox()
    blDep:AddDropdown('HUDBindListSide', { Values={'Left','Right'}, Default=Config.HUDBindListSide,
        Text='Side', Callback=function(v) Config.HUDBindListSide = v end })
    blDep:SetupDependencies({ { Toggles.HUDBindList, true } })

    -- SIGNAL v4 · custom crosshair
    local XH = Tabs.HUD:AddLeftGroupbox('Crosshair')
    XH:AddToggle('FXCrosshair', { Text='Custom crosshair', Default=Config.FXCrosshair,
        Callback=function(v) Config.FXCrosshair = v end })
    local xhDep = XH:AddDependencyBox()
    xhDep:AddDropdown('FXCrosshairStyle', { Values={'Cross','X','T','Dot','Chevron','TriDot'}, Default=Config.FXCrosshairStyle,
        Text='Style', Callback=function(v) Config.FXCrosshairStyle = v end })
    xhDep:AddLabel('Color'):AddColorPicker('FXCrosshairColor', { Default=Config.FXCrosshairColor,
        Callback=function(v) Config.FXCrosshairColor = v end })
    xhDep:AddToggle('FXCrosshairDot', { Text='Center dot', Default=Config.FXCrosshairDot,
        Callback=function(v) Config.FXCrosshairDot = v end })
    xhDep:AddToggle('FXCrosshairBloom', { Text='Fire bloom', Default=Config.FXCrosshairBloom,
        Callback=function(v) Config.FXCrosshairBloom = v end })
    xhDep:AddToggle('FXCrosshairOutline', { Text='Outline', Default=Config.FXCrosshairOutline,
        Callback=function(v) Config.FXCrosshairOutline = v end })
    xhDep:AddToggle('FXCrosshairHitPop', { Text='Hit pop', Default=Config.FXCrosshairHitPop,
        Callback=function(v) Config.FXCrosshairHitPop = v end })
    xhDep:AddSlider('FXCrosshairGap', { Text='Gap', Default=Config.FXCrosshairGap,
        Min=0, Max=20, Rounding=0, Callback=function(v) Config.FXCrosshairGap = math.floor(v) end })
    xhDep:AddSlider('FXCrosshairLen', { Text='Length', Default=Config.FXCrosshairLen,
        Min=2, Max=24, Rounding=0, Callback=function(v) Config.FXCrosshairLen = math.floor(v) end })
    xhDep:AddSlider('FXCrosshairThickness', { Text='Thickness', Default=Config.FXCrosshairThickness,
        Min=1, Max=4, Rounding=0, Callback=function(v) Config.FXCrosshairThickness = math.floor(v) end })
    xhDep:SetupDependencies({ { Toggles.FXCrosshair, true } })

    local L = Tabs.HUD:AddLeftGroupbox('Hit Feedback')
    L:AddToggle('FXHitMarker', { Text='Hit marker', Default=Config.FXHitMarker,
        Callback=function(v) Config.FXHitMarker = v end })
    local hmDep = L:AddDependencyBox()
    hmDep:AddDropdown('FXHitMarkerStyle', { Values={'X','Plus','Ring','Dot'}, Default=Config.FXHitMarkerStyle,
        Text='Style', Callback=function(v) Config.FXHitMarkerStyle = v end })
    hmDep:AddLabel('Marker'):AddColorPicker('FXHitMarkerColor', { Default=Config.FXHitMarkerColor,
        Callback=function(v) Config.FXHitMarkerColor = v end })
    hmDep:AddLabel('Crit'):AddColorPicker('FXHitMarkerCritColor', { Default=Config.FXHitMarkerCritColor,
        Callback=function(v) Config.FXHitMarkerCritColor = v end })
    hmDep:AddLabel('Lethal'):AddColorPicker('FXHitMarkerLethalColor', { Default=Config.FXHitMarkerLethalColor,
        Callback=function(v) Config.FXHitMarkerLethalColor = v end })
    hmDep:AddSlider('FXHitMarkerGap', { Text='Gap', Default=Config.FXHitMarkerGap,
        Min=0, Max=20, Rounding=0, Callback=function(v) Config.FXHitMarkerGap = math.floor(v) end })
    hmDep:AddSlider('FXHitMarkerLen', { Text='Length', Default=Config.FXHitMarkerLen,
        Min=2, Max=30, Rounding=0, Callback=function(v) Config.FXHitMarkerLen = math.floor(v) end })
    hmDep:AddSlider('FXHitMarkerThickness', { Text='Thickness', Default=Config.FXHitMarkerThickness,
        Min=1, Max=6, Rounding=0, Callback=function(v) Config.FXHitMarkerThickness = math.floor(v) end })
    hmDep:SetupDependencies({ { Toggles.FXHitMarker, true } })

    L:AddToggle('FXHitSound', { Text='Hit sound', Default=Config.FXHitSound,
        Callback=function(v) Config.FXHitSound = v end })
    local hsDep = L:AddDependencyBox()
    hsDep:AddSlider('FXHitSoundVolume', { Text='Volume', Default=Config.FXHitSoundVolume,
        Min=0, Max=1, Rounding=2, Callback=function(v) Config.FXHitSoundVolume = v end })
    hsDep:AddInput('FXHitSoundId', { Text='Hit sound id', Default=Config.FXHitSoundId,
        Callback=function(v) Config.FXHitSoundId = v end })
    hsDep:AddInput('FXKillSoundId', { Text='Kill sound id', Default=Config.FXKillSoundId,
        Callback=function(v) Config.FXKillSoundId = v end })
    hsDep:SetupDependencies({ { Toggles.FXHitSound, true } })

    L:AddSlider('FXCritDamage', { Text='Crit threshold', Default=Config.FXCritDamage,
        Min=1, Max=200, Rounding=0, Callback=function(v) Config.FXCritDamage = math.floor(v) end })

    local R = Tabs.HUD:AddRightGroupbox('Combat Text')
    R:AddToggle('FXDamageNumbers', { Text='Damage numbers', Default=Config.FXDamageNumbers,
        Callback=function(v) Config.FXDamageNumbers = v end })
    R:AddSlider('FXDamageAccumWindow', { Text='Accumulate window', Default=Config.FXDamageAccumWindow,
        Min=0, Max=2, Rounding=2, Callback=function(v) Config.FXDamageAccumWindow = v end })
    R:AddToggle('FXKillBanner', { Text='Kill banner', Default=Config.FXKillBanner,
        Callback=function(v) Config.FXKillBanner = v end })
    R:AddLabel('Banner'):AddColorPicker('FXKillBannerColor', { Default=Config.FXKillBannerColor,
        Callback=function(v) Config.FXKillBannerColor = v end })
    R:AddToggle('FXKillFeed', { Text='Kill feed', Default=Config.FXKillFeed,
        Callback=function(v) Config.FXKillFeed = v end })
    R:AddToggle('FXHeadshotSpark', { Text='Headshot spark', Default=Config.FXHeadshotSpark,
        Callback=function(v) Config.FXHeadshotSpark = v end })

    local R2 = Tabs.HUD:AddRightGroupbox('Threat Cues')
    R2:AddToggle('FXDamageDirection', { Text='Damage direction', Default=Config.FXDamageDirection,
        Callback=function(v) Config.FXDamageDirection = v end })
    R2:AddToggle('FXLowHPVignette', { Text='Low HP vignette', Default=Config.FXLowHPVignette,
        Callback=function(v) Config.FXLowHPVignette = v end })
    R2:AddSlider('FXLowHPThreshold', { Text='Low HP threshold', Default=Config.FXLowHPThreshold,
        Min=0.05, Max=0.6, Rounding=2, Callback=function(v) Config.FXLowHPThreshold = v end })
    R2:AddToggle('FXHitFlash', { Text='Hit flash', Default=Config.FXHitFlash,
        Callback=function(v) Config.FXHitFlash = v end })

    -- SIGNAL v4 · target info panel. Reads State.PrimaryTarget, which is stamped ONLY by the ESP
    -- render loop (10_esp) and nil'd when the ESP master is off — so this panel REQUIRES ESP on.
    local TH = Tabs.HUD:AddRightGroupbox('Target HUD')
    TH:AddToggle('FXTargetInfo', { Text='Target info panel', Default=Config.FXTargetInfo,
        Tooltip='Requires the ESP master toggle ON (reads the ESP primary target).',
        Callback=function(v) Config.FXTargetInfo = v end })
    local tiDep = TH:AddDependencyBox()
    tiDep:AddSlider('FXTargetInfoOffset', { Text='Offset', Default=Config.FXTargetInfoOffset,
        Min=60, Max=300, Rounding=0, Callback=function(v) Config.FXTargetInfoOffset = math.floor(v) end })
    tiDep:SetupDependencies({ { Toggles.FXTargetInfo, true } })

    -- §07 Gradient FOV ring (screen-center aim field). Radius reads Config.AimbotFOV.
    local AF = Tabs.HUD:AddRightGroupbox('Aim Field')
    AF:AddToggle('FXFovRing', { Text='FOV ring', Default=Config.FXFovRing,
        Callback=function(v) Config.FXFovRing = v end })
    local fovDep = AF:AddDependencyBox()
    fovDep:AddSlider('FXFovThickness', { Text='Thickness', Default=Config.FXFovThickness,
        Min=0.5, Max=4, Rounding=1, Callback=function(v) Config.FXFovThickness = v end })
    fovDep:AddSlider('FXFovDriftSpeed', { Text='Drift speed', Default=Config.FXFovDriftSpeed,
        Min=0, Max=1, Rounding=2, Callback=function(v) Config.FXFovDriftSpeed = v end })
    fovDep:AddToggle('FXFovFill', { Text='Fill', Default=Config.FXFovFill,
        Callback=function(v) Config.FXFovFill = v end })
    fovDep:AddToggle('FXFovRotate', { Text='Rotate', Default=Config.FXFovRotate,
        Callback=function(v) Config.FXFovRotate = v end })
    fovDep:AddToggle('FXFovCasing', { Text='Ring casing', Default=Config.FXFovCasing,
        Callback=function(v) Config.FXFovCasing = v end })
    fovDep:AddLabel('Color A'):AddColorPicker('FXFovColorA', { Default=Config.FXFovColorA,
        Callback=function(v) Config.FXFovColorA = v end })
    fovDep:AddLabel('Color B'):AddColorPicker('FXFovColorB', { Default=Config.FXFovColorB,
        Callback=function(v) Config.FXFovColorB = v end })
    fovDep:SetupDependencies({ { Toggles.FXFovRing, true } })

    -- §07 World bullet tracer + hit/kill sparks.
    local WF = Tabs.HUD:AddRightGroupbox('World FX')
    WF:AddToggle('FXBeamTracer', { Text='Bullet tracer', Default=Config.FXBeamTracer,
        Callback=function(v) Config.FXBeamTracer = v end })
    local beamDep = WF:AddDependencyBox()
    beamDep:AddDropdown('FXBeamStyle', { Values={'Line','Glow','Prism','Arc'}, Default=Config.FXBeamStyle,
        Text='Style', Callback=function(v) Config.FXBeamStyle = v end })
    beamDep:AddLabel('Color'):AddColorPicker('FXBeamHitColor', { Default=Config.FXBeamHitColor,
        Callback=function(v) Config.FXBeamHitColor = v end })
    -- (the dead 'Miss' picker was removed — onHit always lands, FXBeamMissColor is a stale-save stub)
    beamDep:AddSlider('FXBeamWidth0', { Text='Muzzle width', Default=Config.FXBeamWidth0,
        Min=0.05, Max=0.4, Rounding=2, Callback=function(v) Config.FXBeamWidth0 = v end })
    beamDep:AddSlider('FXBeamWidth1', { Text='Impact width', Default=Config.FXBeamWidth1,
        Min=0.02, Max=0.2, Rounding=2, Callback=function(v) Config.FXBeamWidth1 = v end })
    beamDep:AddSlider('FXBeamDur', { Text='Beam duration', Default=Config.FXBeamDur,
        Min=0.2, Max=1.0, Rounding=2, Callback=function(v) Config.FXBeamDur = v end })
    beamDep:AddToggle('FXBeamGlowLight', { Text='Glow light', Default=Config.FXBeamGlowLight,
        Callback=function(v) Config.FXBeamGlowLight = v end })
    beamDep:AddToggle('FXBeamTravel', { Text='Travel animation', Default=Config.FXBeamTravel,
        Callback=function(v) Config.FXBeamTravel = v end })
    beamDep:AddSlider('FXBeamTravelSpeed', { Text='Travel speed', Default=Config.FXBeamTravelSpeed,
        Min=200, Max=4000, Rounding=0, Callback=function(v) Config.FXBeamTravelSpeed = v end })
    beamDep:AddToggle('FXBeamImpact', { Text='Impact flash', Default=Config.FXBeamImpact,
        Callback=function(v) Config.FXBeamImpact = v end })
    beamDep:SetupDependencies({ { Toggles.FXBeamTracer, true } })
    WF:AddToggle('FXWorldSpark', { Text='World sparks', Default=Config.FXWorldSpark,
        Callback=function(v) Config.FXWorldSpark = v end })
    WF:AddToggle('FXWorldSparkBloom', { Text='Spark bloom', Default=Config.FXWorldSparkBloom,
        Callback=function(v) Config.FXWorldSparkBloom = v end })

    -- FABLE v6 · Kill flourish (06_visuals kill FX — world-anchored lethal effects, default OFF)
    local KF = Tabs.HUD:AddRightGroupbox('Kill Flourish')
    KF:AddToggle('FXKillPillar', { Text='Light pillar', Default=Config.FXKillPillar,
        Callback=function(v) Config.FXKillPillar = v end })
    local kfpDep = KF:AddDependencyBox()
    kfpDep:AddLabel('Pillar'):AddColorPicker('FXKillPillarColor', { Default=Config.FXKillPillarColor,
        Callback=function(v) Config.FXKillPillarColor = v end })
    kfpDep:SetupDependencies({ { Toggles.FXKillPillar, true } })
    KF:AddToggle('FXKillShards', { Text='Shatter burst', Default=Config.FXKillShards,
        Callback=function(v) Config.FXKillShards = v end })
    local kfsDep = KF:AddDependencyBox()
    kfsDep:AddLabel('Shards'):AddColorPicker('FXKillShardsColor', { Default=Config.FXKillShardsColor,
        Callback=function(v) Config.FXKillShardsColor = v end })
    kfsDep:SetupDependencies({ { Toggles.FXKillShards, true } })
    KF:AddToggle('FXKillPulse', { Text='Impact-frame pulse', Default=Config.FXKillPulse,
        Callback=function(v) Config.FXKillPulse = v end })
    local kfuDep = KF:AddDependencyBox()
    kfuDep:AddSlider('FXKillPulseAmount', { Text='Pulse amount', Default=Config.FXKillPulseAmount,
        Min=0.2, Max=1, Rounding=2, Callback=function(v) Config.FXKillPulseAmount = v end })
    kfuDep:SetupDependencies({ { Toggles.FXKillPulse, true } })

    -- Instrument cluster (06c_hudplus). Rides the HUD master; the updater self-gates.
    local IC = Tabs.HUD:AddLeftGroupbox('Instrument Cluster')
    IC:AddToggle('HUDCompass', { Text='Compass', Default=Config.HUDCompass,
        Callback=function(v) Config.HUDCompass = v end })
    local cmpDep = IC:AddDependencyBox()
    cmpDep:AddSlider('HUDCompassWidth', { Text='Width', Default=Config.HUDCompassWidth,
        Min=200, Max=600, Rounding=0, Callback=function(v) Config.HUDCompassWidth = math.floor(v) end })
    cmpDep:AddToggle('HUDCompassPips', { Text='Enemy pips', Default=Config.HUDCompassPips,
        Callback=function(v) Config.HUDCompassPips = v end })
    cmpDep:SetupDependencies({ { Toggles.HUDCompass, true } })
    IC:AddToggle('HUDThreatArc', { Text='Threat arc', Default=Config.HUDThreatArc,
        Callback=function(v) Config.HUDThreatArc = v end })
    IC:AddToggle('HUDRangeReadout', { Text='Range readout', Default=Config.HUDRangeReadout,
        Callback=function(v) Config.HUDRangeReadout = v end })
end

-- ── SETTINGS TAB ───────────────────────────────────────────────────────────
do
    local L = Tabs.Settings:AddLeftGroupbox('Menu')

    L:AddDropdown('GUIToggleKey', {
        Values = {'RightShift','LeftShift','RightControl','LeftControl','RightAlt','LeftAlt',
                  'F1','F2','F3','F4','F5','F6','F7','F8','F9','F10','F11','F12',
                  'Insert','Delete','Home','End','PageUp','PageDown','CapsLock','Tab'},
        Default = Config.GUIToggleKey,
        Text = 'GUI toggle key',
        Callback = function(v) Config.GUIToggleKey = v end
    })

    -- Default OFF: the SIGNAL v4 FX watermark pill (06_visuals, HUDWatermark) is the primary
    -- brand watermark and also renders top-left. Two default-on watermarks collided out of the
    -- box; the Linoria library watermark is now opt-in via this toggle.
    L:AddToggle('ShowWatermark', { Text = 'Show watermark', Default = false,
        Callback = function(v)
            pcall(function() Library:SetWatermarkVisibility(v) end)
        end })

    -- SIGNAL v4 · one-click cohesive overlay palettes. Applied via Options[key]:SetValueRGB so the
    -- pickers, Config AND the save system all stay in sync. Pure GUI code — zero runtime cost.
    local OVERLAY_PALETTES = {
        ['Default'] = {
            ESPBoxColor = Color3.fromRGB(235, 235, 245), ColorEnemy = Color3.fromRGB(255, 59, 78),
            ColorTeam = Color3.fromRGB(53, 215, 199), ColorEnemyOcc = Color3.fromRGB(168, 85, 96),
            ColorTeamOcc = Color3.fromRGB(92, 153, 147), ESPTracerColor = Color3.fromRGB(255, 59, 78),
            ESPBoxGradientA = Color3.fromRGB(255, 59, 78), ESPBoxGradientB = Color3.fromRGB(255, 194, 75),
            FXFovColorA = Color3.fromRGB(53, 215, 199), FXFovColorB = Color3.fromRGB(255, 194, 75),
            FXKillBannerColor = Color3.fromRGB(255, 194, 75),
            VisualsHologramColor = Color3.fromRGB(0, 220, 255), VisualsHologramAccent = Color3.fromRGB(255, 60, 200),
        },
        ['Ice'] = {
            ESPBoxColor = Color3.fromRGB(232, 242, 250), ColorEnemy = Color3.fromRGB(77, 163, 255),
            ColorTeam = Color3.fromRGB(127, 255, 224), ColorEnemyOcc = Color3.fromRGB(90, 122, 153),
            ColorTeamOcc = Color3.fromRGB(110, 158, 150), ESPTracerColor = Color3.fromRGB(77, 163, 255),
            ESPBoxGradientA = Color3.fromRGB(77, 163, 255), ESPBoxGradientB = Color3.fromRGB(176, 224, 255),
            FXFovColorA = Color3.fromRGB(127, 219, 255), FXFovColorB = Color3.fromRGB(255, 255, 255),
            FXKillBannerColor = Color3.fromRGB(176, 224, 255),
            VisualsHologramColor = Color3.fromRGB(102, 204, 255), VisualsHologramAccent = Color3.fromRGB(255, 255, 255),
        },
        ['Crimson'] = {
            ESPBoxColor = Color3.fromRGB(245, 230, 232), ColorEnemy = Color3.fromRGB(255, 45, 85),
            ColorTeam = Color3.fromRGB(255, 209, 102), ColorEnemyOcc = Color3.fromRGB(138, 74, 85),
            ColorTeamOcc = Color3.fromRGB(153, 128, 77), ESPTracerColor = Color3.fromRGB(255, 45, 85),
            ESPBoxGradientA = Color3.fromRGB(255, 45, 85), ESPBoxGradientB = Color3.fromRGB(255, 122, 61),
            FXFovColorA = Color3.fromRGB(255, 45, 85), FXFovColorB = Color3.fromRGB(255, 194, 75),
            FXKillBannerColor = Color3.fromRGB(255, 77, 109),
            VisualsHologramColor = Color3.fromRGB(255, 51, 85), VisualsHologramAccent = Color3.fromRGB(255, 194, 75),
        },
        ['Mono'] = {
            ESPBoxColor = Color3.fromRGB(240, 240, 240), ColorEnemy = Color3.fromRGB(255, 255, 255),
            ColorTeam = Color3.fromRGB(138, 138, 138), ColorEnemyOcc = Color3.fromRGB(122, 122, 122),
            ColorTeamOcc = Color3.fromRGB(85, 85, 85), ESPTracerColor = Color3.fromRGB(221, 221, 221),
            ESPBoxGradientA = Color3.fromRGB(255, 255, 255), ESPBoxGradientB = Color3.fromRGB(102, 102, 102),
            FXFovColorA = Color3.fromRGB(255, 255, 255), FXFovColorB = Color3.fromRGB(153, 153, 153),
            FXKillBannerColor = Color3.fromRGB(255, 255, 255),
            VisualsHologramColor = Color3.fromRGB(221, 221, 221), VisualsHologramAccent = Color3.fromRGB(136, 136, 136),
        },
        ['Vapor'] = {
            ESPBoxColor = Color3.fromRGB(243, 232, 255), ColorEnemy = Color3.fromRGB(255, 113, 206),
            ColorTeam = Color3.fromRGB(1, 205, 254), ColorEnemyOcc = Color3.fromRGB(153, 85, 127),
            ColorTeamOcc = Color3.fromRGB(77, 127, 153), ESPTracerColor = Color3.fromRGB(255, 113, 206),
            ESPBoxGradientA = Color3.fromRGB(255, 113, 206), ESPBoxGradientB = Color3.fromRGB(1, 205, 254),
            FXFovColorA = Color3.fromRGB(185, 103, 255), FXFovColorB = Color3.fromRGB(5, 255, 161),
            FXKillBannerColor = Color3.fromRGB(255, 113, 206),
            VisualsHologramColor = Color3.fromRGB(1, 205, 254), VisualsHologramAccent = Color3.fromRGB(255, 113, 206),
        },
        -- EDGE/VIOLET-forward cohesive scheme (the brand painted across the overlay)
        ['Prism'] = {
            ESPBoxColor = Color3.fromRGB(155, 232, 255), ColorEnemy = Color3.fromRGB(255, 59, 78),
            ColorTeam = Color3.fromRGB(53, 215, 199), ColorEnemyOcc = Color3.fromRGB(150, 90, 110),
            ColorTeamOcc = Color3.fromRGB(80, 130, 128), ESPTracerColor = Color3.fromRGB(155, 232, 255),
            ESPBoxGradientA = Color3.fromRGB(139, 124, 255), ESPBoxGradientB = Color3.fromRGB(41, 224, 255),
            FXFovColorA = Color3.fromRGB(139, 124, 255), FXFovColorB = Color3.fromRGB(155, 232, 255),
            FXKillBannerColor = Color3.fromRGB(255, 194, 75),
            VisualsHologramColor = Color3.fromRGB(41, 224, 255), VisualsHologramAccent = Color3.fromRGB(139, 124, 255),
        },
        -- colorblind-safe (deuteranopia): enemy amber / ally blue (max luminance separation)
        ['Deuteranopia'] = {
            ESPBoxColor = Color3.fromRGB(240, 240, 245), ColorEnemy = Color3.fromRGB(255, 138, 0),
            ColorTeam = Color3.fromRGB(77, 163, 255), ColorEnemyOcc = Color3.fromRGB(153, 100, 40),
            ColorTeamOcc = Color3.fromRGB(70, 105, 150), ESPTracerColor = Color3.fromRGB(255, 138, 0),
            ESPBoxGradientA = Color3.fromRGB(255, 138, 0), ESPBoxGradientB = Color3.fromRGB(255, 214, 0),
            FXFovColorA = Color3.fromRGB(77, 163, 255), FXFovColorB = Color3.fromRGB(255, 214, 0),
            FXKillBannerColor = Color3.fromRGB(255, 214, 0),
            VisualsHologramColor = Color3.fromRGB(77, 163, 255), VisualsHologramAccent = Color3.fromRGB(255, 138, 0),
        },
    }
    local _selPalette = 'Default'
    L:AddDropdown('OverlayPalette', { Values={'Default','Ice','Crimson','Mono','Vapor','Prism','Deuteranopia'},
        Default=_selPalette, Text='Overlay palette',
        Callback=function(v) _selPalette = v end })
    L:AddButton({ Text='Apply palette', Func=function()
        local pal = OVERLAY_PALETTES[_selPalette]; if not pal then return end
        for key, col in pairs(pal) do
            pcall(function()
                if Options[key] and Options[key].SetValueRGB then
                    Options[key]:SetValueRGB(col)        -- fires the picker callback -> Config stays in sync
                else
                    Config[key] = col                    -- key has no picker (defensive fallback)
                end
            end)
        end
    end })

    L:AddDivider()
    L:AddButton({ Text = 'UNLOAD LuaHook', DoubleClick = true, Func = function()
        pcall(function() ESP.unload() end)
        pcall(function() Aimbot.unload() end)
        pcall(function() Rage.unload() end)
        pcall(function() Visuals.unload() end)
        pcall(function() Weather.unload() end)
        pcall(function() UtilESP.unload() end)
        pcall(function() HUDPlus.unload() end)
        Library:Unload()
        _G["\76\72"] = nil
    end })

    UserInputService.InputBegan:Connect(function(input, gpe)
        if gpe then return end
        local key = Config.GUIToggleKey
        if key and Enum.KeyCode[key] and input.KeyCode == Enum.KeyCode[key] then
            pcall(function() Library:Toggle() end)
        end
    end)
end

task.spawn(function()
    pcall(function()
        ThemeManager:SetLibrary(Library)
        SaveManager:SetLibrary(Library)
        SaveManager:IgnoreThemeSettings()
        SaveManager:SetIgnoreIndexes({ 'MenuKeybind', 'LH_ConfigName' })
        ThemeManager:SetFolder('LuaHook')
        SaveManager:SetFolder('LuaHook/configs')
        SaveManager:BuildConfigSection(Tabs.Settings)
        ThemeManager:ApplyToTab(Tabs.Settings)

        local origNotify = Library.Notify
        Library.Notify = function() end
        SaveManager:LoadAutoloadConfig()
        Library.Notify = origNotify

        pcall(function() Library:Toggle() end)
    end)
end)

-- ── WATERMARK ──────────────────────────────────────────────────────────────
if not Library.SetWatermarkVisibility then
    Library.SetWatermarkVisibility = function(self, bool)
        if self.Watermark then self.Watermark.Visible = bool end
    end
end

local _wmAlive   = true
local _origUnload = Library.Unload
Library.Unload = function(self, ...)
    _wmAlive = false
    -- RESTORE the original Gun.StartShooting and clear the hook flags so a later re-execute installs a
    -- fresh hook instead of leaving this (now-unloaded) instance's hook live — the cause of "no guns".
    pcall(function()
        if shared._LH_GunOrig and Rivals.Gun then
            if setreadonly then pcall(setreadonly, Rivals.Gun, false) end
            Rivals.Gun.StartShooting = shared._LH_GunOrig
        end
        shared._gunHooked  = nil
        shared._LH_GunOrig = nil
        -- RESTORE the Katana._StartDeflecting deflect-detection hook (same sentinel pattern as the gun hook)
        if shared._LH_KatanaMod and shared._LH_KatanaDeflOrig then
            if setreadonly then pcall(setreadonly, shared._LH_KatanaMod, false) end
            shared._LH_KatanaMod._StartDeflecting = shared._LH_KatanaDeflOrig
        end
        shared._LH_KatanaMod      = nil
        shared._LH_KatanaDeflOrig = nil
        -- clear the Layer-1 sentinel so a re-execute re-installs the setmetatable hang (on executors that
        -- reset hookfunction state on re-inject but persist getgenv, a stale sentinel would skip re-hooking).
        if getgenv then getgenv().__LH_SetmtBP = nil end
    end)
    return _origUnload(self, ...)
end

pcall(function() Library:SetWatermark('LuaHook v1 beta') end)
-- Match the default-OFF ShowWatermark toggle so the Linoria watermark doesn't clash with the
-- SIGNAL v4 FX pill on load (the toggle re-shows it on demand).
pcall(function() Library:SetWatermarkVisibility(false) end)
task.spawn(function()
    while _wmAlive do
        pcall(function()
            if not Library.Watermark then return end
            local hits, shots = State.Hits, State.Shots
            local acc = shots > 0 and math.floor((hits / shots) * 100) or 0
            local text = string.format(
                'LuaHook v1 beta  ·  %s  ·  Shots: %d  ·  Hits: %d  ·  Acc: %d%%',
                lp.DisplayName, shots, hits, acc
            )
            if Config.Rage then
                text = text .. '  ·  Rage: ' .. (State.RageStatus or 'Idle')
            end
            Library.Watermark.Text = text
        end)
        task.wait(0.5)
    end
end)

Library:Notify('LuaHook v1 beta loaded', 4)
_G["\76\72"] = Library

-- ════════════════ SCRIPT COMPLETE ════════════════