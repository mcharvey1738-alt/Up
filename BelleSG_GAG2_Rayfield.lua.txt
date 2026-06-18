--[[
=====================================================================
   360's GAG   -   Grow a Garden 2 hub
   Axon-style two-column UI, ruby-red accents.
   Right Shift toggles UI.  The X fully unloads.
=====================================================================
]]

--========================== SERVICES ==============================--
local Players          = game:GetService("Players")
local ReplicatedStorage= game:GetService("ReplicatedStorage")
local RunService       = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService     = game:GetService("TweenService")
local CollectionService= game:GetService("CollectionService")
local Workspace        = game:GetService("Workspace")
local LocalPlayer      = Players.LocalPlayer

--========================== GAME API ==============================--
local Net = (function() local ok,m = pcall(function() return require(ReplicatedStorage.SharedModules.Networking) end) return ok and m or nil end)()
local PSC = (function() local ok,m = pcall(function() return require(ReplicatedStorage.ClientModules.PlayerStateClient) end) return ok and m or nil end)()
if not Net then warn("[360's GAG] Networking module missing - aborting."); return end

local SeedData = (function() local ok,d = pcall(function() return require(ReplicatedStorage.SharedModules.SeedData) end) return ok and d or {} end)()
local SeedPrice = {}
for _, e in ipairs(SeedData) do
    if type(e) == "table" and e.SeedName then SeedPrice[e.SeedName] = tonumber(e.PurchasePrice) or math.huge end
end
local FruitValueCalc = (function() local ok,m = pcall(function() return require(ReplicatedStorage.SharedModules.FruitValueCalc) end) return (ok and type(m) == "function") and m or nil end)()
-- FruitValueCalc can't be called from spawned loop threads (executor capability),
-- so precompute each crop's base value here on the main thread and cache it.
local SeedBaseValue = {}
if FruitValueCalc then
    for _, e in ipairs(SeedData) do
        if type(e) == "table" and e.SeedName then
            local ok, v = pcall(FruitValueCalc, e.SeedName, 1, nil, LocalPlayer, nil)
            SeedBaseValue[e.SeedName] = (ok and type(v) == "number") and v or 0
        end
    end
end
local MUT_BONUS = 2.35  -- rough multiplier for any mutation (gold/rainbow/etc.)
local SIZE_EXP  = 2.65  -- FruitValueCalc scales value by size^2.65
local function sizeMul(sz) sz = tonumber(sz) or 1 return sz ^ SIZE_EXP end
local PetData = (function() local ok,m = pcall(function() return require(ReplicatedStorage.SharedData.PetData) end) return ok and m or {} end)()
local function getAnimalOptions()
    local list = {}
    for k, v in pairs(PetData) do if type(v) == "table" and type(k) == "string" then list[#list + 1] = k end end
    table.sort(list); return list
end

--========================== LIFECYCLE =============================--
local Hub = { running = true, conns = {} }
local genv = (getgenv and getgenv()) or _G
if genv.GAG360_unload then pcall(genv.GAG360_unload) end
local function track(conn) table.insert(Hub.conns, conn); return conn end
local function spawnLoop(interval, fn)
    task.spawn(function()
        while Hub.running do
            task.wait(interval)
            if not Hub.running then break end
            pcall(fn)
        end
    end)
end

--========================== STATE =================================--
local S = {
    autoBuySeed = false, buySeeds = {},
    autoPlant = false, plantSeeds = {}, plantReserve = 0, maxPerCycle = 40, plantDelay = 0.14, plantLoop = 1.2, smartReplant = false, autoExpand = false,
    plantPattern = "Fill", plantSource = "My Seeds", autoBuild = false, removeCrops = {},
    autoCollect = false, harvestCrops = {}, harvestMutsOnly = false, perFruitDelay = 0.05, harvestLoop = 1,
    autoSell = false, sellInterval = 20, sellOnFull = false,
    autoSteal = false, stealReturn = true, stealMult = 1,
    panicHarvest = false, retaliate = false,
    autoGrabPacks = false, grabRareOnly = true, packReturn = true, notifyRare = true,
    autoBuyGear = false, buyGears = {}, autoBuyCrate = false,
    autoEggs = false, autoCrates = false, autoPacks = false,
    autoTame = false, tameAnimals = {}, autoEquipPets = false, equipPets = {},
    walkSpeed = 16, jumpPower = 50, infJump = false, noclip = false, fly = false, flySpeed = 60,
    antiAfk = true, optimize = false, autoProgress = false,
    highlightReady = false, highlightRare = false, rareNotify = false,
    webhookUrl = "", whRareSeed = false, whBigHarvest = false, autoHopRare = false,
}

-- settings persistence: save S to disk, restore it on next load (toggles, sliders, picks)
local SAVE_FILE = "360_GAG_GrowAGarden2.json"
local HttpService = game:GetService("HttpService")
local function saveSettings()
    if not writefile then return end
    pcall(function() writefile(SAVE_FILE, HttpService:JSONEncode(S)) end)
end
local function loadSettings()
    if not (readfile and isfile) then return end
    local ok, raw = pcall(function() return isfile(SAVE_FILE) and readfile(SAVE_FILE) or nil end)
    if not (ok and raw) then return end
    local good, data = pcall(function() return HttpService:JSONDecode(raw) end)
    if not (good and type(data) == "table") then return end
    for k, v in pairs(data) do
        if S[k] ~= nil then
            if type(S[k]) == "table" and type(v) == "table" then
                table.clear(S[k]); for kk, vv in pairs(v) do S[k][kk] = vv end  -- keep the table reference the UI holds
            elseif type(S[k]) == type(v) then
                S[k] = v
            end
        end
    end
end
loadSettings()

--========================== HELPERS ===============================--
local function getReplica() if not PSC then return nil end local ok,r = pcall(function() return PSC:GetLocalReplica() end) return ok and r or nil end
local function getData()    local r = getReplica() return r and r.Data or nil end
local function getSheckles() local d = getData() return d and d.Sheckles or 0 end
local function myPlot()
    local g = Workspace:FindFirstChild("Gardens"); if not g then return nil end
    for _, plot in ipairs(g:GetChildren()) do if plot:GetAttribute("OwnerUserId") == LocalPlayer.UserId then return plot end end
end
local function isNight() local n = ReplicatedStorage:FindFirstChild("Night") return n and n.Value == true end
local function char()     return LocalPlayer.Character end
local function hrp()      local c = char() return c and c:FindFirstChild("HumanoidRootPart") end
local function humanoid() local c = char() return c and c:FindFirstChildOfClass("Humanoid") end
local function fire(pkt, ...) local a = {...} return pcall(function() return pkt:Fire(table.unpack(a)) end) end
local function teleportTo(pos) local r = hrp() if r and pos then r.CFrame = CFrame.new(pos + Vector3.new(0, 3, 0)) end end
-- notify/setStatus are forward-declared; Rayfield redefines them after load
local notify = function(t, title) pcall(function() Net.Notification:Fire("Belle.sg", t) end) end
local setStatus = function(t) end  -- no-op until Rayfield is ready
local C = { accent = Color3.fromRGB(167,119,227), green = Color3.fromRGB(80,220,130) }  -- Belle.sg purple

local function setCharCollide(on)
    local c = char(); if not c then return end
    for _, p in ipairs(c:GetDescendants()) do if p:IsA("BasePart") then pcall(function() p.CanCollide = on end) end end
end
local HOP = 70
local function reach(pos)
    local r = hrp(); if not (r and pos) then return end
    local target = pos + Vector3.new(0, 3, 0)
    setCharCollide(false)  -- noclip while teleporting so we never snag on fences/geometry
    for _ = 1, 60 do
        local cur = r.Position; local delta = target - cur
        if delta.Magnitude <= HOP then r.CFrame = CFrame.new(target); break end
        r.CFrame = CFrame.new(cur + delta.Unit * HOP); RunService.Heartbeat:Wait()
    end
    if not S.noclip then setCharCollide(true) end  -- restore unless permanent noclip is on
end
local function fruitValue(m)
    local base = SeedBaseValue[m:GetAttribute("CorePartName") or m:GetAttribute("SeedName")] or 0
    return base * sizeMul(m:GetAttribute("SizeMulti") or 1) * (m:GetAttribute("Mutation") and MUT_BONUS or 1)
end
-- a fruit/plant is ready when its Age has reached MaxAge (reliable + cheap);
-- fall back to the presence of a HarvestPrompt-tagged prompt inside it.
local function modelRipe(m)
    local age = tonumber(m:GetAttribute("Age")); local mx = tonumber(m:GetAttribute("MaxAge"))
    if age and mx then return age >= mx - 0.001 end
    for _, d in ipairs(m:GetDescendants()) do
        if d:IsA("ProximityPrompt") and CollectionService:HasTag(d, "HarvestPrompt") then return true end
    end
    return false
end
-- scan only MY plot (fast + reliable) instead of every tagged prompt on the server
local function ownHarvestTargets(respectFilter)
    local useCrop = respectFilter and next(S.harvestCrops) ~= nil
    local out = {}
    local plot = myPlot(); if not plot then return out end
    local plants = plot:FindFirstChild("Plants"); if not plants then return out end
    local function consider(m)
        if not m:GetAttribute("PlantId") then return end
        local crop = m:GetAttribute("CorePartName") or m:GetAttribute("SeedName")
        local mutOk = (not respectFilter) or (not S.harvestMutsOnly) or (m:GetAttribute("Mutation") ~= nil)
        if ((not useCrop) or (crop and S.harvestCrops[crop] == true)) and mutOk then out[#out + 1] = m end
    end
    for _, plant in ipairs(plants:GetChildren()) do
        local fr = plant:FindFirstChild("Fruits")
        local fruits = fr and fr:GetChildren() or {}
        if #fruits > 0 then
            for _, m in ipairs(fruits) do if modelRipe(m) then consider(m) end end  -- multi-fruit crops
        elseif modelRipe(plant) then
            consider(plant)  -- single-harvest crops (carrot/tulip/bamboo) - the plant is the unit
        end
    end
    return out
end
local function stealTargets()
    local out = {}
    for _, p in ipairs(CollectionService:GetTagged("StealPrompt")) do
        local m = p.Parent and p.Parent:FindFirstAncestorWhichIsA("Model")
        if m then
            local uid = tonumber(m:GetAttribute("UserId"))
            if uid and uid ~= LocalPlayer.UserId and m:GetAttribute("PlantId") then out[#out + 1] = { model = m, value = fruitValue(m) } end
        end
    end
    table.sort(out, function(a, b) return a.value > b.value end)
    return out
end
local function collectModel(m)
    if not m or not m.Parent then return end
    local pid = m:GetAttribute("PlantId"); if not pid then return end
    reach(m:GetPivot().Position); task.wait(S.perFruitDelay)
    fire(Net.Garden.CollectFruit, pid, m:GetAttribute("FruitId") or "")
end
-- bulk harvest: stand at plot centre once, then fire CollectFruit for every ripe
-- fruit (own crops sit within ~20 studs of centre) - no per-fruit teleporting
local function harvestAll(respectFilter)
    local plot = myPlot(); local ref = plot and plot:FindFirstChild("PlotSizeReference"); local r = hrp()
    if ref and r and (Vector3.new(r.Position.X,0,r.Position.Z) - Vector3.new(ref.Position.X,0,ref.Position.Z)).Magnitude > 16 then
        reach(ref.Position); task.wait(0.12)
    end
    local t = ownHarvestTargets(respectFilter); local n = 0
    for _, m in ipairs(t) do
        local pid = m:GetAttribute("PlantId")
        if pid then fire(Net.Garden.CollectFruit, pid, m:GetAttribute("FruitId") or ""); n = n + 1; task.wait(S.perFruitDelay) end
    end
    return n
end
local function stealModel(m, mult, skipReach)
    if not m or not m.Parent then return end
    local uid = tonumber(m:GetAttribute("UserId")); local pid = m:GetAttribute("PlantId")
    if not (uid and pid) then return end
    if not skipReach then reach(m:GetPivot().Position); task.wait(0.05) end
    fire(Net.Steal.BeginSteal, uid, pid, m:GetAttribute("FruitId") or "")
    -- you can carry multiple fruits per steal - fire CompleteSteal mult times
    for _ = 1, math.max(1, mult or 1) do fire(Net.Steal.CompleteSteal) end
end

local function stockItems(shop)
    local sv = ReplicatedStorage:FindFirstChild("StockValues"); sv = sv and sv:FindFirstChild(shop)
    return sv and sv:FindFirstChild("Items")
end
local function seedStockItems() return stockItems("SeedShop") end
local function gearStockItems() return stockItems("GearShop") end
local function stockOf(shop, name) local it = stockItems(shop); local v = it and it:FindFirstChild(name) return (v and v:IsA("ValueBase")) and v.Value or 0 end
local function seedStockOf(name) return stockOf("SeedShop", name) end
local function gearStockOf(name) return stockOf("GearShop", name) end
local function getGearOptions()
    local it = gearStockItems(); local list = {}
    if it then for _, sv in ipairs(it:GetChildren()) do list[#list + 1] = sv.Name end end
    table.sort(list); return list
end
local function getSeedOptions()
    local seen = {}
    for _, e in ipairs(SeedData) do if e.SeedName then seen[e.SeedName] = tonumber(e.SeedShopDisplayOrder) or 900 end end
    local it = seedStockItems(); if it then for _, sv in ipairs(it:GetChildren()) do if seen[sv.Name] == nil then seen[sv.Name] = 899 end end end
    local list = {} for name, ord in pairs(seen) do list[#list + 1] = { name, ord } end
    table.sort(list, function(a, b) if a[2] == b[2] then return a[1] < b[1] end return a[2] < b[2] end)
    local names = {} for _, x in ipairs(list) do names[#names + 1] = x[1] end
    return names
end
-- ONLY the seeds currently in your inventory (live-updates with the shop dropdown loop)
local function getOwnedSeedOptions()
    local d = getData(); local order = {}
    for _, e in ipairs(SeedData) do if e.SeedName then order[e.SeedName] = tonumber(e.SeedShopDisplayOrder) or 900 end end
    local list = {}
    if d and d.Inventory and d.Inventory.Seeds then
        for n, c in pairs(d.Inventory.Seeds) do if (c or 0) > 0 then list[#list + 1] = n end end
    end
    table.sort(list, function(a, b) local oa, ob = order[a] or 900, order[b] or 900 if oa == ob then return a < b end return oa < ob end)
    return list
end
-- distinct crop types currently PLANTED in your garden (for the remove picker)
local function getPlantedOptions()
    local plot = myPlot(); local seen = {}
    if plot then local plants = plot:FindFirstChild("Plants")
        if plants then for _, pl in ipairs(plants:GetChildren()) do local s = pl:GetAttribute("SeedName") or pl:GetAttribute("CorePartName") if s then seen[s] = true end end end
    end
    local list = {} for k in pairs(seen) do list[#list + 1] = k end table.sort(list); return list
end
local function getHarvestOptions()
    local seen = {}
    local plot = myPlot()
    if plot then local plants = plot:FindFirstChild("Plants") if plants then for _, pl in ipairs(plants:GetChildren()) do local s = pl:GetAttribute("SeedName") or pl:GetAttribute("CorePartName") if s then seen[s] = true end end end end
    local d = getData(); if d and d.Inventory and d.Inventory.Seeds then for n in pairs(d.Inventory.Seeds) do seen[n] = true end end
    local list = {} for k in pairs(seen) do list[#list + 1] = k end table.sort(list); return list
end
local function getPetOptions()
    local d = getData(); local seen = {}
    if d and d.Inventory and d.Inventory.Pets then
        for _, info in pairs(d.Inventory.Pets) do local nm = (type(info) == "table" and (info.PetType or info.Name)) or tostring(info) if nm and nm ~= "" then seen[nm] = true end end
    end
    local list = {} for k in pairs(seen) do list[#list + 1] = k end table.sort(list); return list
end
local function maxEquip() return tonumber(LocalPlayer:GetAttribute("MaxEquippedPets")) or 3 end

-- most valuable seed you currently own (uses cached base values)
local function bestOwnedSeed()
    local d = getData(); local seeds = d and d.Inventory and d.Inventory.Seeds; if not seeds then return nil end
    local best, bestV
    for name, count in pairs(seeds) do
        if (count or 0) > 0 then local v = SeedBaseValue[name] or 0 if not bestV or v > bestV then best, bestV = name, v end end
    end
    return best, bestV
end
-- estimated worth of harvested fruit in your backpack (cached base * size * mutation)
local function inventoryValue()
    local total, n = 0, 0
    local function scan(c) if not c then return end for _, t in ipairs(c:GetChildren()) do
        if t:IsA("Tool") and (t:GetAttribute("HarvestedFruit") or t:GetAttribute("Fruit")) then
            n = n + 1
            local base = SeedBaseValue[t:GetAttribute("Fruit") or t:GetAttribute("CorePartName")] or 0
            total = total + base * sizeMul(t:GetAttribute("SizeMultiplier") or t:GetAttribute("SizeMulti") or 1) * (t:GetAttribute("Mutation") and MUT_BONUS or 1)
        end
    end end
    scan(LocalPlayer:FindFirstChild("Backpack")); scan(char())
    return total, n
end
local function abbrev(n)
    n = tonumber(n) or 0
    if n >= 1e9 then return string.format("%.2fB", n/1e9) end
    if n >= 1e6 then return string.format("%.2fM", n/1e6) end
    if n >= 1e3 then return string.format("%.1fK", n/1e3) end
    return tostring(math.floor(n))
end

local EVENT_NAME = { Moon = "Moonlit", Bloodmoon = "Blood Moon", Goldmoon = "Gold Moon",
    ["Rainbow Moon"] = "Rainbow Moon", ["Chained Moon"] = "Chained Moon", ["Pizza Moon"] = "Pizza Moon", Sunset = "Sunset", Day = "Day" }
local EVENT_COLOR = {
    Day = Color3.fromRGB(255,214,90), Sunset = Color3.fromRGB(255,150,90), Moon = Color3.fromRGB(190,150,255),
    Bloodmoon = Color3.fromRGB(176,32,32), Goldmoon = Color3.fromRGB(255,205,70), ["Rainbow Moon"] = Color3.fromRGB(255,120,200),
    ["Chained Moon"] = Color3.fromRGB(150,150,162), ["Pizza Moon"] = Color3.fromRGB(232,120,60) }
local function eventColorOf(r) return EVENT_COLOR[r] or Color3.fromRGB(225,225,230) end
local function eventNameOf(r) return EVENT_NAME[r] or tostring(r or "-") end
local function currentEvent() return workspace:GetAttribute("ActiveWeather"), workspace:GetAttribute("ActivePhase"), tonumber(workspace:GetAttribute("PhaseDuration")) end
local function fmtClock(s) s = math.max(0, math.floor(s or 0)) return string.format("%d:%02d", s // 60, s % 60) end
local function restockIn(shop)
    local sv = ReplicatedStorage:FindFirstChild("StockValues"); sv = sv and sv:FindFirstChild(shop)
    local nx = sv and sv:FindFirstChild("UnixNextRestock")
    return nx and math.max(0, nx.Value - os.time()) or nil
end


--========================== RAYFIELD LOAD ========================--
local Rayfield = loadstring(game:HttpGet("https://sirius.menu/rayfield"))()

-- Override the forward-declared notify/setStatus now that Rayfield is live
notify = function(t, title) pcall(function() Rayfield:Notify({ Title = title or "Belle.sg", Content = t, Duration = 5 }) end) end
setStatus = function(t) pcall(function() Rayfield:Notify({ Title = "Belle.sg", Content = t, Duration = 3 }) end) end

--========================== FEATURE LOOPS =========================--
-- The actual plantable soil is the CollectionService "PlantArea" parts (two ~44x50
-- columns, centred ~12 studs off the PlotSizeReference centre). Grid over those, not a
-- guessed rectangle, so planting covers the WHOLE garden. Patterns sub-select cells.
local PLANT_PATTERNS = { "Fill", "Checkerboard", "Rows", "Columns", "Diagonal", "Spaced" }
local function patternKeep(pat, gx, gz)
    if pat == "Checkerboard" then return (gx + gz) % 2 == 0
    elseif pat == "Rows" then return gz % 2 == 0
    elseif pat == "Columns" then return gx % 2 == 0
    elseif pat == "Diagonal" then return (gx - gz) % 3 == 0
    elseif pat == "Spaced" then return gx % 2 == 0 and gz % 2 == 0 end
    return true  -- Fill
end
local function plantAreas(plot)
    local areas = {}
    for _, p in ipairs(CollectionService:GetTagged("PlantArea")) do
        if p:IsA("BasePart") and p:IsDescendantOf(plot) and p.Size.X * p.Size.Z > 400 then areas[#areas + 1] = p end
    end
    if #areas == 0 then local ref = plot:FindFirstChild("PlotSizeReference"); if ref then areas = { ref } end end
    return areas
end
local function plantPositions(plot)
    local pat = S.plantPattern or "Fill"
    local step = 6
    local seen, list = {}, {}
    for _, area in ipairs(plantAreas(plot)) do
        local cf, sz = area.CFrame, area.Size
        local topY = area.Position.Y + sz.Y/2 + 0.3
        local hx, hz = sz.X/2 - 3, sz.Z/2 - 3
        local nx, nz = math.floor((2*hx)/step), math.floor((2*hz)/step)
        for ix = 0, nx do for iz = 0, nz do
            local w = (cf * CFrame.new(-hx + ix*step, 0, -hz + iz*step)).Position
            local gx, gz = math.floor(w.X/step + 0.5), math.floor(w.Z/step + 0.5)
            if patternKeep(pat, gx, gz) then
                local key = math.floor(w.X/4 + 0.5) .. "," .. math.floor(w.Z/4 + 0.5)
                if not seen[key] then seen[key] = true; list[#list + 1] = Vector3.new(w.X, topY, w.Z) end
            end
        end end
    end
    return list
end
local function freePlantPositions(plot)
    local grid = plantPositions(plot); local plants = plot:FindFirstChild("Plants"); local occ = {}
    if plants then for _, pl in ipairs(plants:GetChildren()) do local ok, pv = pcall(function() return pl:GetPivot().Position end) if ok then occ[#occ+1] = pv end end end
    local free = {}
    for _, pos in ipairs(grid) do
        local clear = true
        for _, o in ipairs(occ) do if (Vector3.new(o.X,0,o.Z) - Vector3.new(pos.X,0,pos.Z)).Magnitude < 6 then clear = false break end end
        if clear then free[#free+1] = pos end
    end
    return free
end

--======================= GARDEN SNAPSHOTS ========================--
-- Capture another player's garden (which seeds + how many, and its buildings) to a named
-- snapshot, then replant the same seeds/amounts (and optionally rebuild the layout) on yours.
local SNAP_FILE = "360_GAG_GAG2_Snapshots.json"
local Snapshots = {}
local function saveSnapshots() if writefile then pcall(function() writefile(SNAP_FILE, HttpService:JSONEncode(Snapshots)) end) end end
do
    if readfile and isfile then
        local ok, raw = pcall(function() return isfile(SNAP_FILE) and readfile(SNAP_FILE) or nil end)
        if ok and raw then local g, d = pcall(function() return HttpService:JSONDecode(raw) end) if g and type(d) == "table" then Snapshots = d end end
    end
end
local function snapshotNames()
    local list = {} for n in pairs(Snapshots) do list[#list + 1] = n end table.sort(list); return list
end
-- the garden the player is standing in / nearest to
local function gardenNearPlayer()
    local g = Workspace:FindFirstChild("Gardens"); local r = hrp(); if not (g and r) then return nil end
    local best, bestD
    for _, plot in ipairs(g:GetChildren()) do
        local ref = plot:FindFirstChild("PlotSizeReference")
        if ref then local d = (Vector3.new(ref.Position.X,0,ref.Position.Z) - Vector3.new(r.Position.X,0,r.Position.Z)).Magnitude
            if not bestD or d < bestD then best, bestD = plot, d end end
    end
    return best
end
-- the building folders a plot can hold (placed props/sprinklers/pots/gnomes)
local BUILD_FOLDERS = { "Props", "Sprinklers", "Gnomes", "PottedPlants", "Pots", "Objects", "Decor" }
local function captureSnapshot(name)
    local plot = gardenNearPlayer(); if not plot then return false, "no garden nearby" end
    local ref = plot:FindFirstChild("PlotSizeReference"); local center = ref and ref.Position or Vector3.zero
    local snap = { seeds = {}, buildings = {}, owner = plot:GetAttribute("OwnerUserId") }
    -- plants -> seed counts
    local plants = plot:FindFirstChild("Plants")
    if plants then for _, pl in ipairs(plants:GetChildren()) do
        local s = pl:GetAttribute("SeedName") or pl:GetAttribute("CorePartName")
        if s then snap.seeds[s] = (snap.seeds[s] or 0) + 1 end
    end end
    -- buildings -> type + position relative to plot centre (best-effort; folders vary)
    for _, fname in ipairs(BUILD_FOLDERS) do
        local f = plot:FindFirstChild(fname)
        if f then for _, b in ipairs(f:GetChildren()) do
            local ok, piv = pcall(function() return b:GetPivot().Position end)
            if ok then
                local kind = b:GetAttribute("PropName") or b:GetAttribute("ItemName") or b:GetAttribute("Name") or b:GetAttribute("Type") or b.Name
                snap.buildings[#snap.buildings + 1] = { kind = tostring(kind), folder = fname,
                    rx = piv.X - center.X, ry = piv.Y - center.Y, rz = piv.Z - center.Z,
                    rot = (select(2, (b:GetPivot()):ToOrientation()) or 0) }
            end
        end end
    end
    Snapshots[name] = snap; saveSnapshots()
    local nSeeds = 0 for _ in pairs(snap.seeds) do nSeeds = nSeeds + 1 end
    return true, ("captured %d seed types, %d buildings"):format(nSeeds, #snap.buildings)
end

--====================== REMOVE / BUILD ===========================--
-- the shovel must be EQUIPPED and passed to UseShovel(plantId, fruitId, shovelAttr, shovelTool)
local function findShovel()
    local function scan(cont) if cont then for _, c in ipairs(cont:GetChildren()) do if c:IsA("Tool") and (c:GetAttribute("Shovel") ~= nil or c.Name:lower():find("shovel")) then return c end end end end
    return scan(char()) or scan(LocalPlayer:FindFirstChild("Backpack"))
end
local function equipShovel()
    local sh = findShovel(); if not sh then return nil end
    local h = humanoid()
    if h and sh.Parent ~= char() then pcall(function() h:EquipTool(sh) end); task.wait(0.3) end
    return sh
end
-- remove plants matching matchFn(cropName) (nil = remove everything)
local function removePlants(matchFn)
    local plot = myPlot(); if not plot then return 0 end
    local plants = plot:FindFirstChild("Plants"); if not plants then return 0 end
    local sh = equipShovel(); if not sh then setStatus("equip a shovel first"); return 0 end
    local sa = sh:GetAttribute("Shovel"); local n = 0; local lastPos
    for _, pl in ipairs(plants:GetChildren()) do
        local pid = pl:GetAttribute("PlantId")
        local crop = pl:GetAttribute("SeedName") or pl:GetAttribute("CorePartName")
        if pid and ((not matchFn) or matchFn(crop)) then
            local ok, pos = pcall(function() return pl:GetPivot().Position end)
            if ok and (not lastPos or (pos - lastPos).Magnitude > 10) then reach(pos); lastPos = pos end
            pcall(function() Net.Shovel.UseShovel:Fire(pid, "", sa, sh) end)
            n = n + 1; task.wait(0.05)
        end
    end
    return n
end
local function removeAllPlants() return removePlants(nil) end
local function removeSelectedPlants() return removePlants(function(crop) return crop and S.removeCrops[crop] == true end) end
local function removeAllBuildings()
    local plot = myPlot(); if not plot then return 0 end
    local n = 0
    for _, fname in ipairs(BUILD_FOLDERS) do
        local f = plot:FindFirstChild(fname)
        if f then for _, b in ipairs(f:GetChildren()) do
            pcall(function()
                if Net.Prop and Net.Prop.PickupProp then Net.Prop.PickupProp:Fire(b) end
                if Net.PotPlacement and Net.PotPlacement.PickUpPottedPlant then Net.PotPlacement.PickUpPottedPlant:Fire(b) end
                if fname == "Gnomes" and Net.Place and Net.Place.RemoveGnome then Net.Place.RemoveGnome:Fire(b) end
            end)
            n = n + 1; task.wait(0.06)
        end end
    end
    return n
end

spawnLoop(2, function()
    if not S.autoBuySeed then return end
    local it = seedStockItems(); if not it then return end
    local anySel = next(S.buySeeds) ~= nil  -- nothing picked = buy everything in stock
    for _, sv in ipairs(it:GetChildren()) do
        if sv:IsA("ValueBase") and sv.Value > 0 and ((not anySel) or S.buySeeds[sv.Name] == true) then
            if getSheckles() >= (SeedPrice[sv.Name] or 0) then fire(Net.SeedShop.PurchaseSeed, sv.Name); task.wait(0.08) end
        end
    end
end)

spawnLoop(0.6, function()
    if not S.autoPlant then return end
    task.wait(math.max(0, S.plantLoop - 0.6))
    if not S.autoPlant then return end
    local plot = myPlot(); if not plot then return end
    local d = getData(); local seeds = d and d.Inventory and d.Inventory.Seeds; if not seeds then return end
    local useFilter = next(S.plantSeeds) ~= nil
    local toPlant = {}
    local snap = (S.plantSource and S.plantSource ~= "My Seeds") and Snapshots[S.plantSource] or nil
    if snap then
        -- replant to match the snapshot's seed counts (capped by what you own)
        local have = {}
        local plf = plot:FindFirstChild("Plants")
        if plf then for _, pl in ipairs(plf:GetChildren()) do local s = pl:GetAttribute("SeedName") or pl:GetAttribute("CorePartName") if s then have[s] = (have[s] or 0) + 1 end end end
        for seed, target in pairs(snap.seeds) do
            local need = math.min((target or 0) - (have[seed] or 0), seeds[seed] or 0)
            for _ = 1, math.max(0, need) do toPlant[#toPlant + 1] = seed end
        end
    elseif S.smartReplant then
        local best = bestOwnedSeed()
        if best and ((not useFilter) or S.plantSeeds[best]) then
            local keep = S.plantReserve or 0
            for _ = 1, math.min(math.max(0, (seeds[best] or 0) - keep), 80) do toPlant[#toPlant + 1] = best end
        end
    else
        for name, count in pairs(seeds) do
            if (not useFilter) or S.plantSeeds[name] == true then
                local keep = S.plantReserve or 0
                for _ = 1, math.min(math.max(0, (count or 0) - keep), 40) do toPlant[#toPlant + 1] = name end
            end
        end
    end
    if #toPlant == 0 then return end
    local free = freePlantPositions(plot); if #free == 0 then return end
    local cap = math.min(#free, #toPlant, S.maxPerCycle); local planted = 0
    for i = 1, cap do
        fire(Net.Plant.PlantSeed, free[i], toPlant[i], plot); planted = planted + 1; task.wait(S.plantDelay)
    end
    if planted > 0 then setStatus("planted " .. planted) end
end)

-- auto-expand the garden (server gates on cost, so just fire when toggled)
spawnLoop(6, function()
    if not S.autoExpand then return end
    local plot = myPlot(); if not plot then return end
    local before = tonumber(plot:GetAttribute("GardenExpansion")) or 0
    fire(Net.Actions.ExpandGarden)
    task.wait(1)
    local after = tonumber(plot:GetAttribute("GardenExpansion")) or before
    if after > before then setStatus("garden expanded to size " .. after) end
end)

-- auto-build: recreate the selected snapshot's building layout on your plot (best-effort)
local function buildSnapshot()
    local snap = (S.plantSource and S.plantSource ~= "My Seeds") and Snapshots[S.plantSource] or nil
    if not (snap and snap.buildings and #snap.buildings > 0) then setStatus("pick a snapshot (with buildings) as the source") return 0 end
    local plot = myPlot(); if not plot then return 0 end
    local ref = plot:FindFirstChild("PlotSizeReference"); local center = ref and ref.Position or Vector3.zero
    local n = 0
    for _, b in ipairs(snap.buildings) do
        local pos = Vector3.new(center.X + (b.rx or 0), center.Y + (b.ry or 0), center.Z + (b.rz or 0))
        pcall(function() if Net.Prop and Net.Prop.PlaceProp then Net.Prop.PlaceProp:Fire(pos, b.kind, b.rot or 0, b.rot or 0) end end)
        n = n + 1; task.wait(0.15)
    end
    setStatus("auto-build: attempted " .. n .. " buildings")
    return n
end
spawnLoop(8, function()
    if not S.autoBuild then return end
    local snap = (S.plantSource and S.plantSource ~= "My Seeds") and Snapshots[S.plantSource] or nil
    if not (snap and snap.buildings and #snap.buildings > 0) then return end
    local plot = myPlot(); if not plot then return end
    local built = 0
    for _, fname in ipairs(BUILD_FOLDERS) do local f = plot:FindFirstChild(fname) if f then built = built + #f:GetChildren() end end
    if built < #snap.buildings then buildSnapshot() end
end)

spawnLoop(0.4, function()
    if not S.autoCollect then return end
    task.wait(math.max(0, S.harvestLoop - 0.4))
    if not S.autoCollect then return end
    local n = harvestAll(true)
    if n > 0 then setStatus("harvested " .. n) end
end)

spawnLoop(1, function()
    if S.sellOnFull then
        local fc = LocalPlayer:GetAttribute("FruitCount") or 0
        local mx = LocalPlayer:GetAttribute("MaxFruitCapacity") or 100
        if fc >= mx - 1 then fire(Net.NPCS.SellAll); setStatus("sold (backpack full)") end
    end
end)
do
    local acc = 0
    spawnLoop(1, function() acc = acc + 1 if S.autoSell and acc >= S.sellInterval then acc = 0 fire(Net.NPCS.SellAll) end end)
end

spawnLoop(0.8, function()
    if not S.autoSteal then return end
    if not isNight() then setStatus("steal: waiting for night") return end
    local home = hrp() and hrp().Position
    local t = stealTargets(); local n = 0; local lastPos
    for _, e in ipairs(t) do
        if not S.autoSteal or not isNight() then break end
        local m = e.model; local pos = (m and m.Parent) and m:GetPivot().Position or nil
        local skip = (lastPos and pos and (pos - lastPos).Magnitude <= 12) or false  -- same plant cluster -> don't re-teleport
        if pos and not skip then lastPos = pos end
        stealModel(m, S.stealMult, skip); n = n + 1
        setStatus(string.format("steal: %d/%d  (worth %d)", n, #t, math.floor(e.value))); task.wait(0.03)
    end
    if n > 0 then setStatus(("stole %d fruit this pass"):format(n)) end
    if S.stealReturn and home then reach(home - Vector3.new(0,3,0)) end
end)

-- event seeds: gold/rainbow seeds + seed packs randomly spawn around the map; you walk
-- to them and HOLD E (a server-added ProximityPrompt) to collect. We TP over + fire it.
local function packKind(loc)
    if loc:GetAttribute("GoldSeed") == true then return "Gold Seed" end
    if loc:GetAttribute("RainbowSeed") == true then return "Rainbow Seed" end
    if loc:GetAttribute("SeedPack") ~= nil then return tostring(loc:GetAttribute("SeedPack")) end
    return nil
end
local function isRarePack(loc)
    if loc:GetAttribute("GoldSeed") == true or loc:GetAttribute("RainbowSeed") == true then return true end
    local sp = loc:GetAttribute("SeedPack")
    return type(sp) == "string" and (sp:lower():find("gold") ~= nil or sp:lower():find("rainbow") ~= nil)
end
local function firePrompt(d)
    pcall(function()
        local hold = tonumber(d.HoldDuration) or 0
        if fireproximityprompt then
            if hold > 0 then fireproximityprompt(d, hold) else fireproximityprompt(d) end
        else
            d:InputHoldBegin(); task.wait(hold + 0.1); d:InputHoldEnd()
        end
    end)
end
local function packLocations()
    local map = Workspace:FindFirstChild("Map"); local f = map and map:FindFirstChild("SeedPackSpawnServerLocations")
    return f and f:GetChildren() or {}
end
-- hold every collect-prompt on / near a spawned seed (server adds the hold-E prompt)
local function holdSeedPrompts(pos)
    local map = Workspace:FindFirstChild("Map")
    for _, cont in ipairs({ map and map:FindFirstChild("SeedPackSpawnServerLocations"), map and map:FindFirstChild("SeedPackSpawnClient"), Workspace:FindFirstChild("Temporary") }) do
        if cont then for _, d in ipairs(cont:GetDescendants()) do
            if d:IsA("ProximityPrompt") then
                local p = d.Parent; local ok, pp = pcall(function() return p.Position end)
                if (not ok) or (pp - pos).Magnitude <= 35 then firePrompt(d) end
            end
        end end
    end
end
local function locPart(loc) return loc:IsA("BasePart") and loc or loc:FindFirstChildWhichIsA("BasePart", true) end
local function locPos(loc)
    if loc:IsA("BasePart") then return loc.Position end
    local ok, cf = pcall(function() return loc:GetPivot() end); if ok then return cf.Position end
    local bp = locPart(loc); return bp and bp.Position or nil
end
-- stand on the seed and collect it: fire its hold-E prompt, any nearby prompt, AND touch it
local function grabPack(loc)
    local landed = false
    for _ = 1, 90 do
        if not (loc and loc.Parent) then break end
        local pos = locPos(loc); if not pos then break end
        local r = hrp()
        if (not landed) or (r and (r.Position - pos).Magnitude > 6) then reach(pos); landed = true end
        for _, d in ipairs(loc:GetDescendants()) do if d:IsA("ProximityPrompt") then firePrompt(d) end end  -- prompt on the seed itself
        holdSeedPrompts(pos)                                                                                   -- + any prompt nearby (client visual)
        local part = locPart(loc)
        if firetouchinterest and part and hrp() then pcall(function() firetouchinterest(hrp(), part, 0); firetouchinterest(hrp(), part, 1) end) end  -- touch-to-collect fallback
        task.wait(0.12)
    end
end
do
    local grabbing = {}
    spawnLoop(0.6, function()
        if not S.autoGrabPacks then return end
        for _, loc in ipairs(packLocations()) do
            if loc.Parent and not grabbing[loc] then
                local rare = isRarePack(loc)
                if S.notifyRare and rare then local k = packKind(loc) or "Rare seed"; setStatus("EVENT: " .. k .. " spawned!"); notify(k .. " spawned on the map - grabbing it now!", "✦ Rare Seed Spawned", C.accent) end
                if (not S.grabRareOnly) or rare then
                    grabbing[loc] = true
                    task.spawn(function() grabPack(loc); grabbing[loc] = nil end)
                end
            end
        end
    end)
end
do
    local wasNight = false
    spawnLoop(1, function()
        local n = isNight()
        if S.packReturn and S.autoGrabPacks and wasNight and not n then
            local plot = myPlot(); local sp = plot and plot:FindFirstChild("SpawnPoint")
            if sp then reach(sp.Position); setStatus("event over - returned to garden") end
        end
        wasNight = n
    end)
end

do
    local wasNight = false
    spawnLoop(0.5, function()
        local n = isNight()
        if S.panicHarvest and n and not wasNight then
            setStatus("defense: panic harvesting")
            harvestAll(false)
        end
        wasNight = n
    end)
end
spawnLoop(0.6, function()
    if not S.retaliate then return end
    local plot = myPlot(); local ref = plot and plot:FindFirstChild("PlotSizeReference"); if not ref then return end
    local center, size = ref.Position, ref.Size
    for _, pl in ipairs(Players:GetPlayers()) do
        if pl ~= LocalPlayer and pl.Character then
            local r = pl.Character:FindFirstChild("HumanoidRootPart")
            if r and math.abs(r.Position.X - center.X) < size.X/2 + 4 and math.abs(r.Position.Z - center.Z) < size.Z/2 + 4 then fire(Net.Shovel.HitPlayer, pl.UserId) end
        end
    end
end)

spawnLoop(3, function()
    if not S.autoBuyCrate then return end
    local it = stockItems("CrateShop"); if not it then return end
    for _, sv in ipairs(it:GetChildren()) do if sv:IsA("ValueBase") and sv.Value > 0 then fire(Net.CrateShop.PurchaseCrate, sv.Name); task.wait(0.1) end end
end)
spawnLoop(3, function()
    if not S.autoBuyGear then return end
    local it = gearStockItems(); if not it then return end
    local anySel = next(S.buyGears) ~= nil
    for _, sv in ipairs(it:GetChildren()) do
        if sv:IsA("ValueBase") and sv.Value > 0 and ((not anySel) or S.buyGears[sv.Name] == true) then fire(Net.GearShop.PurchaseGear, sv.Name); task.wait(0.1) end
    end
end)

local function openAll(invKey, pkt, flag)
    spawnLoop(2.5, function()
        if not S[flag] then return end
        local d = getData(); local bag = d and d.Inventory and d.Inventory[invKey]; if not bag then return end
        for name, count in pairs(bag) do local n = (type(count) == "number") and count or 1 for _ = 1, n do task.spawn(function() fire(pkt, name) end) task.wait(0.15) end end
    end)
end
openAll("Eggs", Net.Egg.OpenEgg, "autoEggs")
openAll("Crates", Net.Crate.OpenCrate, "autoCrates")
openAll("SeedPacks", Net.SeedPack.OpenSeedPack, "autoPacks")

spawnLoop(1.2, function()
    if not S.autoTame then return end
    local map = Workspace:FindFirstChild("Map"); local refs = map and map:FindFirstChild("WildPetRef"); if not refs then return end
    local anySel = next(S.tameAnimals) ~= nil
    for _, pet in ipairs(refs:GetChildren()) do
        if not S.autoTame then break end
        local owner = tonumber(pet:GetAttribute("OwnerUserId")) or 0
        local species = pet:GetAttribute("PetName")
        if ((not anySel) or (species and S.tameAnimals[species] == true)) and (owner == 0 or owner == LocalPlayer.UserId) and pet:IsA("BasePart") then
            reach(pet.Position); setStatus("taming " .. tostring(species))
            for _ = 1, 6 do if not S.autoTame then break end pcall(function() Net.Pets.WildPetTame:Fire(pet) end) task.wait(0.08) end
        end
    end
end)
-- AUTO PROGRESS: hands-off progression. Harvest -> sell -> buy the best seeds you can
-- afford -> plant them everywhere -> tame valuable pets when they spawn. Snowballs coins.
local GOOD_PETS = {
    Raccoon = true, Dragonfly = true, ["Dragon Fly"] = true, Dragonling = true, Mimic = true,
    ["Disco Bee"] = true, ["Queen Bee"] = true, Kitsune = true, ["Red Fox"] = true, Fox = true,
    Owl = true, ["Night Owl"] = true, Bear = true, ["Polar Bear"] = true, Butterfly = true,
    ["Golden Lab"] = true, Cat = true, ["Red Giant Ant"] = true, Snail = true,
}
local function progressBuy()
    local it = seedStockItems(); if not it then return end
    local money = getSheckles(); local best, bestV
    for _, sv in ipairs(it:GetChildren()) do
        if sv:IsA("ValueBase") and sv.Value > 0 then
            local price, val = SeedPrice[sv.Name] or math.huge, SeedBaseValue[sv.Name] or 0
            if price <= money * 0.5 and (not bestV or val > bestV) then best, bestV = sv.Name, val end
        end
    end
    if best then for _ = 1, 6 do if getSheckles() < (SeedPrice[best] or 0) then break end fire(Net.SeedShop.PurchaseSeed, best); task.wait(0.1) end end
    return best
end
local function progressPlant()
    local plot = myPlot(); if not plot then return 0 end
    local d = getData(); local seeds = d and d.Inventory and d.Inventory.Seeds; if not seeds then return 0 end
    local toPlant = {}
    for name, count in pairs(seeds) do for _ = 1, math.min(count or 0, 30) do toPlant[#toPlant + 1] = name end end
    if #toPlant == 0 then return 0 end
    local free = freePlantPositions(plot); local cap = math.min(#free, #toPlant); local n = 0
    for i = 1, cap do fire(Net.Plant.PlantSeed, free[i], toPlant[i], plot); n = n + 1; task.wait(0.08) end
    return n
end
spawnLoop(4, function()
    if not S.autoProgress then return end
    local h = harvestAll(false)
    if (LocalPlayer:GetAttribute("FruitCount") or 0) > 0 then fire(Net.NPCS.SellAll); task.wait(0.2) end
    progressBuy()
    local p = progressPlant()
    setStatus(("auto progress: +%d harvest, +%d plant, %s"):format(h, p, money(getSheckles())))
end)
spawnLoop(1.5, function()
    if not S.autoProgress then return end
    local map = Workspace:FindFirstChild("Map"); local refs = map and map:FindFirstChild("WildPetRef"); if not refs then return end
    for _, pet in ipairs(refs:GetChildren()) do
        if not S.autoProgress then break end
        local species = pet:GetAttribute("PetName"); local owner = tonumber(pet:GetAttribute("OwnerUserId")) or 0
        if species and GOOD_PETS[species] and (owner == 0 or owner == LocalPlayer.UserId) and pet:IsA("BasePart") then
            reach(pet.Position); setStatus("auto progress: taming " .. species)
            for _ = 1, 6 do if not S.autoProgress then break end pcall(function() Net.Pets.WildPetTame:Fire(pet) end) task.wait(0.08) end
        end
    end
end)
spawnLoop(5, function()
    if not S.autoEquipPets then return end
    local n, mx = 0, maxEquip()
    for name in pairs(S.equipPets) do if n >= mx then break end fire(Net.Pets.RequestEquipByName, tostring(name)); n = n + 1; task.wait(0.15) end
end)

-- fly + movement
local flyBV, flyBG
local function stopFly()
    if flyBV then pcall(function() flyBV:Destroy() end) flyBV = nil end
    if flyBG then pcall(function() flyBG:Destroy() end) flyBG = nil end
    local h = humanoid(); if h then h.PlatformStand = false end
end
Hub.stopFly = stopFly
local function startFly()
    local r = hrp(); if not r then return end
    stopFly()
    flyBV = Instance.new("BodyVelocity"); flyBV.MaxForce = Vector3.new(1,1,1)*9e9; flyBV.Velocity = Vector3.zero; flyBV.Parent = r
    flyBG = Instance.new("BodyGyro"); flyBG.MaxTorque = Vector3.new(1,1,1)*9e9; flyBG.P = 1e5; flyBG.CFrame = r.CFrame; flyBG.Parent = r
end
track(RunService.Heartbeat:Connect(function()
    if not Hub.running then return end
    local h = humanoid()
    if h then
        if S.walkSpeed ~= 16 then h.WalkSpeed = S.walkSpeed end
        if S.jumpPower ~= 50 then h.UseJumpPower = true; h.JumpPower = S.jumpPower end
    end
    if S.noclip then local c = char() if c then for _, p in ipairs(c:GetDescendants()) do if p:IsA("BasePart") and p.CanCollide then p.CanCollide = false end end end end
    if S.fly then
        local r = hrp(); local cam = Workspace.CurrentCamera
        if r and cam then
            if not flyBV then startFly() end
            if h then h.PlatformStand = true end
            local d = Vector3.zero
            local function k(c) return UserInputService:IsKeyDown(c) end
            if k(Enum.KeyCode.W) then d = d + cam.CFrame.LookVector end
            if k(Enum.KeyCode.S) then d = d - cam.CFrame.LookVector end
            if k(Enum.KeyCode.D) then d = d + cam.CFrame.RightVector end
            if k(Enum.KeyCode.A) then d = d - cam.CFrame.RightVector end
            if k(Enum.KeyCode.Space) then d = d + Vector3.new(0,1,0) end
            if k(Enum.KeyCode.LeftControl) then d = d - Vector3.new(0,1,0) end
            if flyBV then flyBV.Velocity = (d.Magnitude > 0 and d.Unit or Vector3.zero) * S.flySpeed end
            if flyBG then flyBG.CFrame = cam.CFrame end
        end
    elseif flyBV then stopFly() end
end))
track(UserInputService.JumpRequest:Connect(function() if S.infJump then local h = humanoid() if h then h:ChangeState(Enum.HumanoidStateType.Jumping) end end end))

-- anti-afk: VirtualUser click on the Idled signal (fires just before the 20-min
-- idle kick). Non-disruptive - only acts when you are actually idle.
do
    local VU = game:GetService("VirtualUser")
    track(LocalPlayer.Idled:Connect(function()
        if not S.antiAfk then return end
        pcall(function()
            VU:CaptureController()
            VU:ClickButton2(Vector2.new())
        end)
    end))
end

-- webhook + server hopper
local HttpService = game:GetService("HttpService")
local TPS = game:GetService("TeleportService")
local httpRequest = (syn and syn.request) or (http and http.request) or (fluxus and fluxus.request) or (typeof(request) == "function" and request) or http_request
local function sendWebhook(content)
    if not (S.webhookUrl and S.webhookUrl ~= "" and httpRequest) then return false end
    task.spawn(function()
        pcall(function()
            httpRequest({ Url = S.webhookUrl, Method = "POST", Headers = { ["Content-Type"] = "application/json" },
                Body = HttpService:JSONEncode({ username = "360's GAG", content = content }) })
        end)
    end)
    return true
end
local function fetchServers()
    local ok, res = pcall(function()
        local raw = game:HttpGet("https://games.roblox.com/v1/games/" .. game.PlaceId .. "/servers/Public?sortOrder=Asc&limit=100")
        return HttpService:JSONDecode(raw)
    end)
    return (ok and res and res.data) or {}
end
local function serverHop(lowPop)
    setStatus("finding a server...")
    local servers = fetchServers(); local pick
    for _, s in ipairs(servers) do
        if s.id ~= game.JobId and s.playing and s.maxPlayers and s.playing < s.maxPlayers then
            if lowPop then if not pick or s.playing < pick.playing then pick = s end
            else pick = s; break end
        end
    end
    if pick then setStatus("hopping (" .. pick.playing .. " players)..."); pcall(function() TPS:TeleportToPlaceInstance(game.PlaceId, pick.id, LocalPlayer) end)
    else setStatus("no server found - retrying may help") end
end
local function rareSeedInStock()
    local it = seedStockItems(); if not it then return false end
    for _, sv in ipairs(it:GetChildren()) do if sv:IsA("ValueBase") and sv.Value > 0 and (SeedPrice[sv.Name] or 0) >= 5000 then return true, sv.Name end end
    return false
end
-- hop between servers until a rare seed is in stock
spawnLoop(20, function()
    if not S.autoHopRare then return end
    if not rareSeedInStock() then serverHop(false) end
end)

-- profit tracker (net sheckles, rolling 60s rate)
local Profit = { startS = nil, session = 0, perMin = 0, perHr = 0, win = {} }
spawnLoop(2, function()
    local s = getSheckles()
    if Profit.startS == nil then Profit.startS = s end
    Profit.session = s - Profit.startS
    table.insert(Profit.win, { t = os.clock(), s = s })
    while #Profit.win > 1 and (os.clock() - Profit.win[1].t) > 60 do table.remove(Profit.win, 1) end
    local f = Profit.win[1]; local dt = os.clock() - f.t
    if dt > 4 then Profit.perMin = (s - f.s)/dt*60; Profit.perHr = Profit.perMin*60 end
end)

-- highlight ESP (own ready crops + mutated fruit, distance-capped)
local hlFolder = Instance.new("Folder"); hlFolder.Name = "GAG_HL"; hlFolder.Parent = ScreenGui
local function clearHL() for _, h in ipairs(hlFolder:GetChildren()) do h:Destroy() end end
local function addHL(model, col)
    if not model or not model.Parent then return end
    local h = Instance.new("Highlight"); h.Adornee = model; h.FillColor = col; h.FillTransparency = 0.55
    h.OutlineColor = col; h.OutlineTransparency = 0; h.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop; h.Parent = hlFolder
end
spawnLoop(1, function()
    if not (S.highlightReady or S.highlightRare) then if #hlFolder:GetChildren() > 0 then clearHL() end return end
    clearHL()
    local root = hrp(); local rp = root and root.Position
    if S.highlightReady then for _, m in ipairs(ownHarvestTargets()) do addHL(m, C.accent) end end
    if S.highlightRare and rp then
        local count = 0
        for _, p in ipairs(CollectionService:GetTagged("StealPrompt")) do
            if count >= 50 then break end
            local m = p.Parent and p.Parent:FindFirstAncestorWhichIsA("Model")
            if m and m:GetAttribute("Mutation") then
                local ok, piv = pcall(function() return m:GetPivot().Position end)
                if ok and (piv - rp).Magnitude < 220 then addHL(m, Color3.fromRGB(255,205,70)); count = count + 1 end
            end
        end
    end
end)
table.insert(Hub.conns, { Disconnect = function() pcall(clearHL) pcall(function() hlFolder:Destroy() end) end })

-- rare seed restock notifier (fires once when an expensive seed appears in stock)
do
    local prev = {}
    spawnLoop(3, function()
        if not S.rareNotify then return end
        local it = seedStockItems(); if not it then return end
        for _, sv in ipairs(it:GetChildren()) do
            if sv:IsA("ValueBase") then
                local now = sv.Value > 0
                if now and not prev[sv.Name] and (SeedPrice[sv.Name] or 0) >= 5000 then
                    setStatus("RARE SEED IN STOCK: " .. sv.Name); notify(sv.Name .. " just restocked - " .. sv.Value .. "x available (" .. fmtPrice(SeedPrice[sv.Name]) .. ")", "✦ Rare Seed In Stock", C.green)
                    if S.whRareSeed then sendWebhook("**Rare seed in stock:** " .. sv.Name .. " (" .. sv.Value .. "x)  -  " .. LocalPlayer.Name) end
                end
                prev[sv.Name] = now
            end
        end
    end)
end

-- performance optimizer: flat textures, grey sky, no effects (FPS boost)
local Lighting = game:GetService("Lighting")
local optConns, optOrig
local function optimizeInstance(o)
    pcall(function()
        if o:IsA("BasePart") then
            o.Material = Enum.Material.SmoothPlastic; o.Reflectance = 0; o.CastShadow = false
        elseif o:IsA("Decal") or o:IsA("Texture") then
            o.Transparency = 1
        elseif o:IsA("ParticleEmitter") or o:IsA("Trail") or o:IsA("Beam") or o:IsA("Smoke") or o:IsA("Fire") or o:IsA("Sparkles") then
            o.Enabled = false
        elseif o:IsA("PostEffect") then
            o.Enabled = false
        end
    end)
end
local function setOptimize(on)
    if on then
        optOrig = optOrig or { gs = Lighting.GlobalShadows, fc = Lighting.FogColor, fs = Lighting.FogStart, fe = Lighting.FogEnd, br = Lighting.Brightness, oa = Lighting.OutdoorAmbient, am = Lighting.Ambient }
        pcall(function()
            Lighting.GlobalShadows = false
            Lighting.FogColor = Color3.fromRGB(131,133,139); Lighting.FogStart = 220; Lighting.FogEnd = 780  -- grey sky via fog
            Lighting.OutdoorAmbient = Color3.fromRGB(140,140,146); Lighting.Ambient = Color3.fromRGB(122,122,128)  -- neutralise colour tint
        end)
        for _, e in ipairs(Lighting:GetDescendants()) do
            if e:IsA("Atmosphere") or e:IsA("Clouds") or e:IsA("PostEffect") then pcall(function() e.Enabled = false end) end
            if e:IsA("Sky") then pcall(function() e.CelestialBodiesShown = false end) end
        end
        pcall(function() Workspace.Terrain.Decoration = false end)
        pcall(function() settings().Rendering.QualityLevel = Enum.QualityLevel.Level01 end)
        for _, o in ipairs(Workspace:GetDescendants()) do optimizeInstance(o) end
        -- keep optimizing ANYTHING that streams in later (new plants, players, effects, etc.)
        if optConns then for _, c in ipairs(optConns) do pcall(function() c:Disconnect() end) end end
        local function onAdd(o) if S.optimize then task.defer(optimizeInstance, o) end end
        optConns = { Workspace.DescendantAdded:Connect(onAdd), Lighting.DescendantAdded:Connect(onAdd) }
        for _, c in ipairs(optConns) do track(c) end
        setStatus("optimized - flat textures, grey sky, effects off")
    else
        if optConns then for _, c in ipairs(optConns) do pcall(function() c:Disconnect() end) end optConns = nil end
        if optOrig then pcall(function()
            Lighting.GlobalShadows = optOrig.gs; Lighting.FogColor = optOrig.fc; Lighting.FogStart = optOrig.fs; Lighting.FogEnd = optOrig.fe; Lighting.Brightness = optOrig.br
            Lighting.OutdoorAmbient = optOrig.oa; Lighting.Ambient = optOrig.am
        end) end
        for _, e in ipairs(Lighting:GetDescendants()) do
            if e:IsA("Atmosphere") or e:IsA("Clouds") or e:IsA("PostEffect") then pcall(function() e.Enabled = true end) end
            if e:IsA("Sky") then pcall(function() e.CelestialBodiesShown = true end) end
        end
        pcall(function() Workspace.Terrain.Decoration = true end)
        for _, o in ipairs(Workspace:GetDescendants()) do
            if o:IsA("ParticleEmitter") or o:IsA("Trail") or o:IsA("Beam") or o:IsA("Smoke") or o:IsA("Fire") or o:IsA("Sparkles") then pcall(function() o.Enabled = true end)
            elseif o:IsA("Decal") or o:IsA("Texture") then pcall(function() o.Transparency = 0 end) end
        end
        setStatus("optimizer off (rejoin to restore textures fully)")
    end
end


--========================== RAYFIELD UI ==========================--
-- Exact same features as original, Rayfield UI only
local Window = Rayfield:CreateWindow({
    Name = "Belle.sg  |  Grow a Garden 2",
    LoadingTitle = "Belle.sg",
    LoadingSubtitle = "Grow a Garden 2",
    Theme = "Default",
    ConfigurationSaving = { Enabled = false },
    Discord = { Enabled = false },
    KeySystem = false,
})

-- ─── FARM TAB ────────────────────────────────────────────────────
local FarmTab = Window:CreateTab("Farm", 4483362458)

FarmTab:CreateSection("Auto Plant")
FarmTab:CreateToggle({ Name = "Auto Plant", CurrentValue = S.autoPlant, Flag = "autoPlant",
    Callback = function(v) S.autoPlant = v; saveSettings() end })
FarmTab:CreateToggle({ Name = "Smart Replant (best seed only)", CurrentValue = S.smartReplant, Flag = "smartReplant",
    Callback = function(v) S.smartReplant = v; saveSettings() end })
FarmTab:CreateToggle({ Name = "Auto Expand Garden", CurrentValue = S.autoExpand, Flag = "autoExpand",
    Callback = function(v) S.autoExpand = v; saveSettings() end })
FarmTab:CreateSlider({ Name = "Max Plants Per Cycle", Range = {1,80}, Increment = 1, CurrentValue = S.maxPerCycle, Flag = "maxPerCycle",
    Callback = function(v) S.maxPerCycle = v; saveSettings() end })
FarmTab:CreateSlider({ Name = "Plant Delay (s)", Range = {0.05,1}, Increment = 0.01, Suffix = "s", CurrentValue = S.plantDelay, Flag = "plantDelay",
    Callback = function(v) S.plantDelay = v; saveSettings() end })
FarmTab:CreateSlider({ Name = "Loop Delay (s)", Range = {0.5,10}, Increment = 0.1, Suffix = "s", CurrentValue = S.plantLoop, Flag = "plantLoop",
    Callback = function(v) S.plantLoop = v; saveSettings() end })
FarmTab:CreateSlider({ Name = "Seed Reserve Per Type", Range = {0,25}, Increment = 1, CurrentValue = S.plantReserve, Flag = "plantReserve",
    Callback = function(v) S.plantReserve = v; saveSettings() end })
FarmTab:CreateButton({ Name = "Plant Once Now", Callback = function()
    local plot = myPlot(); if not plot then return end
    local d = getData(); local seeds = d and d.Inventory and d.Inventory.Seeds; if not seeds then return end
    local useF = next(S.plantSeeds) ~= nil; local tp = {}
    for n, c in pairs(seeds) do if (not useF) or S.plantSeeds[n] then for _=1,math.min(c or 0,40) do tp[#tp+1]=n end end end
    local free = freePlantPositions(plot)
    for i = 1, math.min(#free, #tp) do fire(Net.Plant.PlantSeed, free[i], tp[i], plot) task.wait(S.plantDelay) end
    setStatus("planted " .. math.min(#free, #tp))
end })
FarmTab:CreateButton({ Name = "Expand Garden Now", Callback = function()
    local plot = myPlot(); if not plot then return end
    local before = tonumber(plot:GetAttribute("GardenExpansion")) or 0
    fire(Net.Actions.ExpandGarden); task.wait(0.8)
    local after = tonumber(plot:GetAttribute("GardenExpansion")) or before
    setStatus(after > before and ("expanded to size "..after) or "can't expand (need more money or maxed)")
end })

FarmTab:CreateSection("Auto Harvest")
FarmTab:CreateToggle({ Name = "Auto Harvest", CurrentValue = S.autoCollect, Flag = "autoCollect",
    Callback = function(v) S.autoCollect = v; saveSettings() end })
FarmTab:CreateToggle({ Name = "Only Harvest Mutated Fruit", CurrentValue = S.harvestMutsOnly, Flag = "harvestMutsOnly",
    Callback = function(v) S.harvestMutsOnly = v; saveSettings() end })
FarmTab:CreateSlider({ Name = "Per-Fruit Delay (s)", Range = {0.02,0.5}, Increment = 0.01, Suffix = "s", CurrentValue = S.perFruitDelay, Flag = "perFruitDelay",
    Callback = function(v) S.perFruitDelay = v; saveSettings() end })
FarmTab:CreateSlider({ Name = "Harvest Loop Delay (s)", Range = {0.5,10}, Increment = 0.1, Suffix = "s", CurrentValue = S.harvestLoop, Flag = "harvestLoop",
    Callback = function(v) S.harvestLoop = v; saveSettings() end })
FarmTab:CreateButton({ Name = "Harvest Now (all ripe)", Callback = function()
    setStatus("harvested " .. harvestAll(false))
end })

FarmTab:CreateSection("Auto Sell")
FarmTab:CreateToggle({ Name = "Auto Sell (timed)", CurrentValue = S.autoSell, Flag = "autoSell",
    Callback = function(v) S.autoSell = v; saveSettings() end })
FarmTab:CreateToggle({ Name = "Sell When Backpack Full", CurrentValue = S.sellOnFull, Flag = "sellOnFull",
    Callback = function(v) S.sellOnFull = v; saveSettings() end })
FarmTab:CreateSlider({ Name = "Sell Interval (s)", Range = {5,120}, Increment = 1, Suffix = "s", CurrentValue = S.sellInterval, Flag = "sellInterval",
    Callback = function(v) S.sellInterval = v; saveSettings() end })
FarmTab:CreateButton({ Name = "Sell All Now", Callback = function()
    fire(Net.NPCS.SellAll); setStatus("sold all")
end })

-- ─── SHOP TAB ─────────────────────────────────────────────────────
local ShopTab = Window:CreateTab("Shop", 4483362458)

ShopTab:CreateSection("Seeds")
ShopTab:CreateToggle({ Name = "Auto Buy Seeds", CurrentValue = S.autoBuySeed, Flag = "autoBuySeed",
    Callback = function(v) S.autoBuySeed = v; saveSettings() end })
ShopTab:CreateButton({ Name = "Buy Seeds Now", Callback = function()
    local it = seedStockItems(); if not it then return end
    local anySel = next(S.buySeeds) ~= nil
    for _, sv in ipairs(it:GetChildren()) do
        if sv:IsA("ValueBase") and sv.Value > 0 and ((not anySel) or S.buySeeds[sv.Name] == true) then
            fire(Net.SeedShop.PurchaseSeed, sv.Name); task.wait(0.08)
        end
    end
    setStatus("bought seeds")
end })

ShopTab:CreateSection("Gear & Crates")
ShopTab:CreateToggle({ Name = "Auto Buy Gears", CurrentValue = S.autoBuyGear, Flag = "autoBuyGear",
    Callback = function(v) S.autoBuyGear = v; saveSettings() end })
ShopTab:CreateToggle({ Name = "Auto Buy Crates", CurrentValue = S.autoBuyCrate, Flag = "autoBuyCrate",
    Callback = function(v) S.autoBuyCrate = v; saveSettings() end })

-- ─── STEAL TAB ────────────────────────────────────────────────────
local StealTab = Window:CreateTab("Steal", 4483362458)

StealTab:CreateSection("Night Raiding")
StealTab:CreateToggle({ Name = "Auto Steal (night only)", CurrentValue = S.autoSteal, Flag = "autoSteal",
    Callback = function(v) S.autoSteal = v; saveSettings() end })
StealTab:CreateToggle({ Name = "Return Home After Each Pass", CurrentValue = S.stealReturn, Flag = "stealReturn",
    Callback = function(v) S.stealReturn = v; saveSettings() end })
StealTab:CreateSlider({ Name = "Fruits Per Steal", Range = {1,10}, Increment = 1, CurrentValue = S.stealMult, Flag = "stealMult",
    Callback = function(v) S.stealMult = v; saveSettings() end })
StealTab:CreateButton({ Name = "Steal Most Valuable Now", Callback = function()
    if not isNight() then setStatus("not night - cannot steal"); return end
    local t = stealTargets()
    if t[1] then stealModel(t[1].model, S.stealMult); setStatus("stole fruit worth " .. math.floor(t[1].value))
    else setStatus("nothing to steal") end
end })

-- ─── DEFENSE TAB ──────────────────────────────────────────────────
local DefenseTab = Window:CreateTab("Defense", 4483362458)

DefenseTab:CreateSection("Protect Your Garden")
DefenseTab:CreateToggle({ Name = "Panic Harvest At Night Start", CurrentValue = S.panicHarvest, Flag = "panicHarvest",
    Callback = function(v) S.panicHarvest = v; saveSettings() end })
DefenseTab:CreateToggle({ Name = "Retaliate (shovel intruders)", CurrentValue = S.retaliate, Flag = "retaliate",
    Callback = function(v) S.retaliate = v; saveSettings() end })
DefenseTab:CreateButton({ Name = "Emergency Harvest Now", Callback = function()
    setStatus("harvested " .. harvestAll(false))
end })

-- ─── EVENT TAB ────────────────────────────────────────────────────
local EventTab = Window:CreateTab("Event", 4483362458)

EventTab:CreateSection("Seed Pack Grabber")
EventTab:CreateToggle({ Name = "Auto Grab Seed Packs", CurrentValue = S.autoGrabPacks, Flag = "autoGrabPacks",
    Callback = function(v) S.autoGrabPacks = v; saveSettings() end })
EventTab:CreateToggle({ Name = "Rare Only (Gold / Rainbow)", CurrentValue = S.grabRareOnly, Flag = "grabRareOnly",
    Callback = function(v) S.grabRareOnly = v; saveSettings() end })
EventTab:CreateToggle({ Name = "Return When Event Ends", CurrentValue = S.packReturn, Flag = "packReturn",
    Callback = function(v) S.packReturn = v; saveSettings() end })
EventTab:CreateToggle({ Name = "Notify On Rare Pack Spawn", CurrentValue = S.notifyRare, Flag = "notifyRare",
    Callback = function(v) S.notifyRare = v; saveSettings() end })
EventTab:CreateButton({ Name = "Grab Nearest Pack Now", Callback = function()
    local root = hrp(); if not root then return end
    local map = Workspace:FindFirstChild("Map"); local locs = map and map:FindFirstChild("SeedPackSpawnServerLocations")
    if not locs or #locs:GetChildren() == 0 then setStatus("no pack spawned right now"); return end
    local best, bestD
    for _, loc in ipairs(locs:GetChildren()) do
        local d = (locPos(loc) and (locPos(loc) - root.Position).Magnitude) or math.huge
        if d < (bestD or math.huge) then best, bestD = loc, d end
    end
    if best then task.spawn(function() grabPack(best) end); setStatus("grabbing nearest pack") end
end })

-- ─── ITEMS TAB ────────────────────────────────────────────────────
local ItemsTab = Window:CreateTab("Items", 4483362458)

ItemsTab:CreateSection("Auto Open")
ItemsTab:CreateToggle({ Name = "Auto Open Eggs", CurrentValue = S.autoEggs, Flag = "autoEggs",
    Callback = function(v) S.autoEggs = v; saveSettings() end })
ItemsTab:CreateToggle({ Name = "Auto Open Crates", CurrentValue = S.autoCrates, Flag = "autoCrates",
    Callback = function(v) S.autoCrates = v; saveSettings() end })
ItemsTab:CreateToggle({ Name = "Auto Open Seed Packs", CurrentValue = S.autoPacks, Flag = "autoPacks",
    Callback = function(v) S.autoPacks = v; saveSettings() end })
ItemsTab:CreateButton({ Name = "Open All Eggs Now", Callback = function()
    local d = getData(); local b = d and d.Inventory and d.Inventory.Eggs
    if b then for n in pairs(b) do task.spawn(function() fire(Net.Egg.OpenEgg, n) end); task.wait(0.15) end end
    setStatus("opened eggs")
end })
ItemsTab:CreateButton({ Name = "Open All Crates Now", Callback = function()
    local d = getData(); local b = d and d.Inventory and d.Inventory.Crates
    if b then for n in pairs(b) do task.spawn(function() fire(Net.Crate.OpenCrate, n) end); task.wait(0.15) end end
    setStatus("opened crates")
end })
ItemsTab:CreateButton({ Name = "Open All Seed Packs Now", Callback = function()
    local d = getData(); local b = d and d.Inventory and d.Inventory.SeedPacks
    if b then for n in pairs(b) do task.spawn(function() fire(Net.SeedPack.OpenSeedPack, n) end); task.wait(0.15) end end
    setStatus("opened packs")
end })

ItemsTab:CreateSection("Garden Snapshots")
local _snapName = "Snapshot 1"
ItemsTab:CreateInput({ Name = "Snapshot Name", PlaceholderText = "Snapshot 1", RemoveTextAfterFocusLost = false,
    Callback = function(t) if t and t ~= "" then _snapName = t end end })
ItemsTab:CreateButton({ Name = "Snapshot This Garden", Callback = function()
    local ok, msg = captureSnapshot(_snapName)
    if ok then notify('Saved "'.._snapName..'" - '..msg, "Garden Snapshot")
    else setStatus(tostring(msg)) end
end })

ItemsTab:CreateSection("Cleanup")
ItemsTab:CreateToggle({ Name = "Auto Build Snapshot", CurrentValue = S.autoBuild, Flag = "autoBuild",
    Callback = function(v) S.autoBuild = v; saveSettings() end })
ItemsTab:CreateButton({ Name = "Build Snapshot Now", Callback = function() buildSnapshot() end })
ItemsTab:CreateButton({ Name = "Remove All Plants", Callback = function()
    setStatus("removing plants..."); task.spawn(function() local n = removeAllPlants(); setStatus("removed "..n.." plants") end)
end })
ItemsTab:CreateButton({ Name = "Remove All Buildings", Callback = function()
    setStatus("removing buildings..."); task.spawn(function() local n = removeAllBuildings(); setStatus("removed "..n.." buildings") end)
end })

-- ─── PETS TAB ─────────────────────────────────────────────────────
local PetsTab = Window:CreateTab("Pets", 4483362458)

PetsTab:CreateSection("Auto Tame")
PetsTab:CreateToggle({ Name = "Auto Tame Wild Animals", CurrentValue = S.autoTame, Flag = "autoTame",
    Callback = function(v) S.autoTame = v; saveSettings() end })

PetsTab:CreateSection("Auto Equip")
PetsTab:CreateToggle({ Name = "Auto Equip Pets", CurrentValue = S.autoEquipPets, Flag = "autoEquipPets",
    Callback = function(v) S.autoEquipPets = v; saveSettings() end })
PetsTab:CreateButton({ Name = "Equip Pets Now", Callback = function()
    local n, mx = 0, maxEquip()
    for name in pairs(S.equipPets) do if n >= mx then break end fire(Net.Pets.RequestEquipByName, tostring(name)); n = n + 1; task.wait(0.12) end
    setStatus("equipped "..n.." pets")
end })

-- ─── STATS TAB ────────────────────────────────────────────────────
local StatsTab = Window:CreateTab("Stats", 4483362458)

StatsTab:CreateSection("Profit Tracker")
local lPerMin  = StatsTab:CreateLabel("Per Minute:  -")
local lPerHr   = StatsTab:CreateLabel("Per Hour:  -")
local lSession = StatsTab:CreateLabel("Session Earned:  -")
StatsTab:CreateSection("Inventory")
local lInvVal  = StatsTab:CreateLabel("Backpack Value:  -")
local lInvCnt  = StatsTab:CreateLabel("Fruit Count:  -")
local lBest    = StatsTab:CreateLabel("Best Seed To Plant:  -")
StatsTab:CreateButton({ Name = "Rescan Inventory Now", Callback = function()
    local v, n = inventoryValue()
    lInvVal:Set("Backpack Value:  "..money(v))
    lInvCnt:Set("Fruit Count:  "..n.."x")
end })
spawnLoop(2, function()
    pcall(function()
        lPerMin:Set("Per Minute:  "..money(Profit.perMin))
        lPerHr:Set("Per Hour:  "..money(Profit.perHr))
        lSession:Set("Session Earned:  "..money(Profit.session))
        local v, n = inventoryValue()
        lInvVal:Set("Backpack Value:  "..money(v))
        lInvCnt:Set("Fruit Count:  "..n.."x")
        local best = bestOwnedSeed()
        local d2 = getData()
        local cnt = (best and d2 and d2.Inventory and d2.Inventory.Seeds and d2.Inventory.Seeds[best]) or 0
        lBest:Set("Best Seed:  "..(best and (best.."  x"..cnt) or "-"))
    end)
end)

-- ─── TELEPORT TAB ─────────────────────────────────────────────────
local TpTab = Window:CreateTab("Teleport", 4483362458)

TpTab:CreateSection("Shops & NPCs")
local function makeTpBtn(tab, label, pad)
    tab:CreateButton({ Name = "Teleport: "..label, Callback = function()
        local t = Workspace:FindFirstChild("Teleports"); local d = t and t:FindFirstChild(pad)
        if d and d:IsA("BasePart") then reach(d.Position); setStatus("teleported to "..label)
        else setStatus(label.." not found") end
    end })
end
makeTpBtn(TpTab, "Seed Shop", "Seeds")
makeTpBtn(TpTab, "Gear Shop", "Gears")
makeTpBtn(TpTab, "Sell NPC",  "Sell")
makeTpBtn(TpTab, "Props Shop","Props")
TpTab:CreateSection("Garden")
TpTab:CreateButton({ Name = "Go To My Garden", Callback = function()
    local plot = myPlot(); local sp = plot and plot:FindFirstChild("SpawnPoint")
    if sp then reach(sp.Position); setStatus("teleported home") end
end })

-- ─── VISUAL TAB ───────────────────────────────────────────────────
local VisualTab = Window:CreateTab("Visual", 4483362458)

VisualTab:CreateSection("ESP & Alerts")
VisualTab:CreateToggle({ Name = "Highlight Ready Crops", CurrentValue = S.highlightReady, Flag = "highlightReady",
    Callback = function(v) S.highlightReady = v; saveSettings() end })
VisualTab:CreateToggle({ Name = "Highlight Mutated Fruit (gold)", CurrentValue = S.highlightRare, Flag = "highlightRare",
    Callback = function(v) S.highlightRare = v; saveSettings() end })
VisualTab:CreateToggle({ Name = "Rare Seed Restock Alert", CurrentValue = S.rareNotify, Flag = "rareNotify",
    Callback = function(v) S.rareNotify = v; saveSettings() end })

-- ─── PLAYER TAB ───────────────────────────────────────────────────
local PlayerTab = Window:CreateTab("Player", 4483362458)

PlayerTab:CreateSection("Movement")
PlayerTab:CreateSlider({ Name = "Walk Speed", Range = {16,120}, Increment = 1, CurrentValue = S.walkSpeed, Flag = "walkSpeed",
    Callback = function(v) S.walkSpeed = v; saveSettings() end })
PlayerTab:CreateSlider({ Name = "Jump Power", Range = {50,250}, Increment = 1, CurrentValue = S.jumpPower, Flag = "jumpPower",
    Callback = function(v) S.jumpPower = v; saveSettings() end })
PlayerTab:CreateToggle({ Name = "Infinite Jump", CurrentValue = S.infJump, Flag = "infJump",
    Callback = function(v) S.infJump = v; saveSettings() end })
PlayerTab:CreateToggle({ Name = "Noclip", CurrentValue = S.noclip, Flag = "noclip",
    Callback = function(v)
        S.noclip = v
        if not v then local c = char() if c then for _, p in ipairs(c:GetDescendants()) do if p:IsA("BasePart") then p.CanCollide = true end end end end
        saveSettings()
    end })

PlayerTab:CreateSection("Fly")
PlayerTab:CreateToggle({ Name = "Fly (W/A/S/D + Space/Ctrl)", CurrentValue = S.fly, Flag = "fly",
    Callback = function(v) S.fly = v; if not v and Hub.stopFly then Hub.stopFly() end; saveSettings() end })
PlayerTab:CreateSlider({ Name = "Fly Speed", Range = {20,150}, Increment = 1, CurrentValue = S.flySpeed, Flag = "flySpeed",
    Callback = function(v) S.flySpeed = v; saveSettings() end })

-- ─── MISC TAB ─────────────────────────────────────────────────────
local MiscTab = Window:CreateTab("Misc", 4483362458)

MiscTab:CreateSection("Utility")
MiscTab:CreateToggle({ Name = "Auto Progress (harvest→sell→buy→plant)", CurrentValue = S.autoProgress, Flag = "autoProgress",
    Callback = function(v) S.autoProgress = v; saveSettings() end })
MiscTab:CreateToggle({ Name = "Anti-AFK", CurrentValue = S.antiAfk, Flag = "antiAfk",
    Callback = function(v) S.antiAfk = v; saveSettings() end })
MiscTab:CreateToggle({ Name = "Optimize (FPS Boost)", CurrentValue = S.optimize, Flag = "optimize",
    Callback = function(v) S.optimize = v; setOptimize(v); saveSettings() end })
MiscTab:CreateButton({ Name = "Rejoin Server", Callback = function()
    pcall(function() game:GetService("TeleportService"):Teleport(game.PlaceId, LocalPlayer) end)
end })
MiscTab:CreateButton({ Name = "Server Hop", Callback = function() serverHop(false) end })
MiscTab:CreateButton({ Name = "Low-Pop Hop", Callback = function() serverHop(true) end })
MiscTab:CreateToggle({ Name = "Auto-Hop Until Rare Seed", CurrentValue = S.autoHopRare, Flag = "autoHopRare",
    Callback = function(v) S.autoHopRare = v; saveSettings() end })

MiscTab:CreateSection("Webhook")
MiscTab:CreateInput({ Name = "Webhook URL", PlaceholderText = "https://discord.com/api/webhooks/...", RemoveTextAfterFocusLost = false,
    Callback = function(t) S.webhookUrl = t; saveSettings() end })
MiscTab:CreateToggle({ Name = "Notify: Rare Seed In Stock", CurrentValue = S.whRareSeed, Flag = "whRareSeed",
    Callback = function(v) S.whRareSeed = v; saveSettings() end })
MiscTab:CreateButton({ Name = "Send Test Webhook", Callback = function()
    if sendWebhook("Test from Belle.sg GAG2 - webhook working!") then setStatus("test sent")
    else setStatus("set a webhook URL first") end
end })

MiscTab:CreateSection("Info")
MiscTab:CreateLabel("Belle.sg  |  Grow a Garden 2")
MiscTab:CreateLabel("Right Shift to toggle UI")
MiscTab:CreateButton({ Name = "Unload Hub", Callback = function() Hub.unload() end })

--========================== ESP ===================================--
-- ESP highlights (exact same logic as original, purple accent for Belle.sg)
local ESPGui = Instance.new("ScreenGui"); ESPGui.Name = "BelleSG_ESP"; ESPGui.ResetOnSpawn = false
ESPGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling; ESPGui.IgnoreGuiInset = true
if syn and syn.protect_gui then pcall(syn.protect_gui, ESPGui) end
ESPGui.Parent = (function()
    if syn then return game:GetService("CoreGui") end
    local ok, r = pcall(function() return game:GetService("CoreGui") end)
    if ok then return r end
    return LocalPlayer:FindFirstChildOfClass("PlayerGui") or LocalPlayer:WaitForChild("PlayerGui")
end)()
local hlFolder = Instance.new("Folder"); hlFolder.Name = "BelleSG_HL"; hlFolder.Parent = ESPGui
local function clearHL() for _, h in ipairs(hlFolder:GetChildren()) do h:Destroy() end end
local function addHL(model, col)
    if not model or not model.Parent then return end
    local h = Instance.new("Highlight"); h.Adornee = model
    h.FillColor = col; h.FillTransparency = 0.55
    h.OutlineColor = col; h.OutlineTransparency = 0
    h.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop; h.Parent = hlFolder
end
spawnLoop(1, function()
    if not (S.highlightReady or S.highlightRare) then if #hlFolder:GetChildren() > 0 then clearHL() end return end
    clearHL()
    local root = hrp(); local rp = root and root.Position
    if S.highlightReady then
        for _, m in ipairs(ownHarvestTargets()) do addHL(m, Color3.fromRGB(167,119,227)) end  -- Belle.sg purple
    end
    if S.highlightRare and rp then
        local count = 0
        for _, p in ipairs(CollectionService:GetTagged("StealPrompt")) do
            if count >= 50 then break end
            local m = p.Parent and p.Parent:FindFirstAncestorWhichIsA("Model")
            if m and m:GetAttribute("Mutation") then
                local ok, piv = pcall(function() return m:GetPivot().Position end)
                if ok and (piv - rp).Magnitude < 220 then addHL(m, Color3.fromRGB(255,205,70)); count = count + 1 end
            end
        end
    end
end)
table.insert(Hub.conns, { Disconnect = function() pcall(clearHL); pcall(function() hlFolder:Destroy() end) end })

--========================== RARE SEED NOTIFIER ===================--
do
    local prev = {}
    spawnLoop(3, function()
        if not S.rareNotify then return end
        local it = seedStockItems(); if not it then return end
        for _, sv in ipairs(it:GetChildren()) do
            if sv:IsA("ValueBase") then
                local now = sv.Value > 0
                if now and not prev[sv.Name] and (SeedPrice[sv.Name] or 0) >= 5000 then
                    notify(sv.Name .. " just restocked - " .. sv.Value .. "x available (" .. fmtPrice(SeedPrice[sv.Name]) .. ")", "✦ Rare Seed In Stock")
                    if S.whRareSeed then sendWebhook("**Rare seed in stock:** " .. sv.Name .. " (" .. sv.Value .. "x)  -  " .. LocalPlayer.Name) end
                end
                prev[sv.Name] = now
            end
        end
    end)
end

--========================== UNLOAD ================================--
Hub.unload = function()
    if not Hub.running then return end
    saveSettings()
    Hub.running = false
    for _, c in ipairs(Hub.conns) do pcall(function() c:Disconnect() end) end
    Hub.conns = {}
    if Hub.stopFly then pcall(Hub.stopFly) end
    local h = humanoid()
    if h then h.WalkSpeed = 16; h.UseJumpPower = true; h.JumpPower = 50; h.PlatformStand = false end
    local c = char()
    if c then for _, p in ipairs(c:GetDescendants()) do if p:IsA("BasePart") then pcall(function() p.CanCollide = true end) end end end
    for k, v in pairs(S) do if type(v) == "boolean" then S[k] = false end end
    pcall(function() ESPGui:Destroy() end)
    pcall(function() Rayfield:Destroy() end)
    print("[Belle.sg] unloaded.")
end
genv.GAG360_unload = Hub.unload
genv.GAG360_notify = function(msg, title) notify(msg, title) end

--========================== RIGHT SHIFT TOGGLE ====================--
track(UserInputService.InputBegan:Connect(function(i, gpe)
    if gpe then return end
    if i.KeyCode == Enum.KeyCode.RightShift then
        Rayfield:Toggle()
    end
end))

--========================== INIT =================================--
print("[Belle.sg] Grow a Garden 2 loaded. Right Shift to toggle.")
