print("[DE-Auto] Starting full script...")

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local VirtualInputManager = game:GetService("VirtualInputManager")
local localPlayer = Players.LocalPlayer

-- ═══════════════════════════════════════════════════════════
--  UI Library Wrapper (Orion UI for Mobile Support)
-- ═══════════════════════════════════════════════════════════
print("[DE-Auto] Fetching Orion UI Library...")
local success, OrionLib = pcall(function()
    return loadstring(game:HttpGet(('https://raw.githubusercontent.com/shlexsoftware/Orion/main/source')))()
end)

if not success or not OrionLib then
    warn("[DE-Auto] CRITICAL ERROR: Your executor failed to load the UI library. It might be blocking GitHub raw links. Check your executor's settings.")
    return
end

print("[DE-Auto] UI Library loaded. Building window...")

local Window = OrionLib:MakeWindow({
    Name = "Driving Empire Auto | Mobile", 
    HidePremium = true, 
    SaveConfig = false, 
    IntroEnabled = false
})

local UI = {}
local uiValues = {}

UI.GetValue = function(id) return uiValues[id] end
UI.SetValue = function(id, val) uiValues[id] = val end

function UI.AddTab(tabName, callback)
    local tab = Window:MakeTab({
        Name = tabName,
        Icon = "rbxassetid://4483345998",
        PremiumOnly = false
    })
    
    local tabBuilder = {}
    
    function tabBuilder:Section(secName, side)
        tab:AddLabel("--- " .. secName .. " ---")
        local sectionObj = {}
        
        function sectionObj:Toggle(id, name, default)
            uiValues[id] = default
            tab:AddToggle({
                Name = name,
                Default = default,
                Callback = function(val) uiValues[id] = val end
            })
        end
        
        function sectionObj:Button(name, cb)
            tab:AddButton({Name = name, Callback = cb})
        end
        
        function sectionObj:SliderInt(id, name, min, max, default)
            uiValues[id] = default
            tab:AddSlider({
                Name = name, Min = min, Max = max, Default = default, Color = Color3.fromRGB(255,255,255), Increment = 1, ValueName = "",
                Callback = function(val) uiValues[id] = val end
            })
        end
        
        function sectionObj:SliderFloat(id, name, min, max, default, format)
            uiValues[id] = default
            tab:AddSlider({
                Name = name, Min = min, Max = max, Default = default, Color = Color3.fromRGB(255,255,255), Increment = 0.1, ValueName = "",
                Callback = function(val) uiValues[id] = val end
            })
        end
        
        function sectionObj:Tip(text)
            tab:AddLabel("💡 " .. text)
        end
        
        function sectionObj:Spacing() end
        
        return sectionObj
    end
    callback(tabBuilder)
end

local function notify(title, content)
    OrionLib:MakeNotification({Name = title, Content = content, Image = "rbxassetid://4483345998", Time = 3})
end

-- ═══════════════════════════════════════════════════════════
--  Original Script Offsets & Memory
-- ═══════════════════════════════════════════════════════════
local offsets = {
    base_part = { primitive = 0x148 },
    primitive  = { cframe = 0xC0, velocity = 0xF0 },
}

local promptHoldOffset = nil
task.spawn(function()
    local ok, res = pcall(function() return game:HttpGet("https://offsets.ntgetwritewatch.workers.dev/offsets.json") end)
    if ok and res then
        local HttpService = game:GetService("HttpService")
        local ok2, data = pcall(function() return HttpService:JSONDecode(res) end)
        if ok2 and data and data.ProximityPromptHoldDuraction then
            promptHoldOffset = tonumber(data.ProximityPromptHoldDuraction, 16)
        end
    end
end)

-- ═══════════════════════════════════════════════════════════
--  Mobile Bypasses
-- ═══════════════════════════════════════════════════════════
local function clickButton(button)
    if getconnections then
        for _, conn in ipairs(getconnections(button.MouseButton1Click)) do pcall(function() conn:Fire() end) end
        for _, conn in ipairs(getconnections(button.Activated)) do pcall(function() conn:Fire() end) end
    else
        local absPos, absSize = button.AbsolutePosition, button.AbsoluteSize
        local x, y = absPos.X + absSize.X / 2, absPos.Y + absSize.Y / 2
        VirtualInputManager:SendMouseButtonEvent(x, y, 0, true, game, 1)
        task.wait(0.1)
        VirtualInputManager:SendMouseButtonEvent(x, y, 0, false, game, 1)
    end
end

-- ═══════════════════════════════════════════════════════════
--  Forward declarations
-- ═══════════════════════════════════════════════════════════
local joinSecurityTeam
local joinOutlawTeam
local atm_sell_bag

-- ═══════════════════════════════════════════════════════════
--  UI Building
-- ═══════════════════════════════════════════════════════════
UI.AddTab("Auto Arrest", function(tab)
    local ctrl = tab:Section("Controls", "Left")
    ctrl:Toggle("oh_enabled", "Enable Auto Arrest", false)
    ctrl:Button("Get Security Team", function() task.spawn(joinSecurityTeam) end)

    local settings = tab:Section("Settings", "Right")
    settings:SliderInt("oh_offset", "Y Offset (studs)", -50, 50, 20)
    settings:Tip("Adjusts the vertical teleport offset")
end)

UI.AddTab("Auto Rob", function(tab)
    local ctrl = tab:Section("Controls", "Left")
    ctrl:Toggle("atm_enabled", "Enable Auto Rob", false)
    ctrl:Button("Deposit Bags", function() task.spawn(atm_sell_bag) end)
    ctrl:Button("Get Outlaw Team", function() task.spawn(joinOutlawTeam) end)

    local settings = tab:Section("Settings", "Right")
    settings:SliderInt("atm_hold",  "Hold Time (s)",  1, 20, 7)
    settings:SliderFloat("atm_wait", "Wait Time (s)", 0.5, 5.0, 1.5, "%.1f")
    settings:SliderInt("atm_sell_threshold", "Sell After (bags)", 1, 50, 10)
    settings:Tip("Deposits when this many bags are held")
end)

-- ═══════════════════════════════════════════════════════════
--  Shared Settings & Helpers
-- ═══════════════════════════════════════════════════════════
local function secEnabled() return UI.GetValue("oh_enabled") end
local function atmEnabled() return UI.GetValue("atm_enabled") end
local function getYOffset() return UI.GetValue("oh_offset") or 20 end
local function getSellThreshold() return UI.GetValue("atm_sell_threshold") or 10 end

local function read_fvector3(address, offset)
    if not memory_read then return Vector3.new() end
    return Vector3.new(
        memory_read("float", address + offset),
        memory_read("float", address + offset + 0x4),
        memory_read("float", address + offset + 0x8)
    )
end

local function read_cframe(part)
    if not part or not memory_read then return nil end
    local prim = memory_read("uintptr", part.Address + offsets.base_part.primitive)
    if prim == 0 then return nil end
    local r = {}
    for i, key in ipairs({"r00","r01","r02","r10","r11","r12","r20","r21","r22"}) do
        r[key] = memory_read("float", prim + offsets.primitive.cframe + (i - 1) * 0x4)
    end
    return { rot = r, pos = read_fvector3(prim, offsets.primitive.cframe + 0x24) }
end

local function teleportTo(targetPosition)
    local character = localPlayer.Character
    if not character then return end
    local rootPart = character:FindFirstChild("HumanoidRootPart")
    if not rootPart then return end
    
    rootPart.CFrame = CFrame.new(targetPosition + Vector3.new(0, 3 + getYOffset(), 0))
    rootPart.Velocity = Vector3.zero
end

local function distanceBetween(a, b)
    return (a - b).Magnitude
end

-- ═══════════════════════════════════════════════════════════
--  Team Helpers
-- ═══════════════════════════════════════════════════════════
local function isOutlaw(player) return player.Team and string.lower(player.Team.Name) == "outlaw" end
local function isSecurity(player) return player.Team and string.lower(player.Team.Name) == "security" end

local function joinViapad(padName, checkFn, label)
    local PAD_AREAS = {
        SecurityPad = Vector3.new(-109.757, 25.080, -956.793),
        CriminalPad = Vector3.new(-2525.589, 14.262, 4015.533),
    }
    local PAD_AREA = PAD_AREAS[padName] or Vector3.new(-109.757, 25.080, -956.793)
    
    local function rawTeleport(pos)
        local char = localPlayer.Character or Workspace:FindFirstChild(localPlayer.Name)
        if not char then return end
        local root = char:FindFirstChild("HumanoidRootPart")
        if root then root.Position = pos; root.Velocity = Vector3.zero end
    end

    notify(label, "Teleporting to pad area...")
    local renderStart = os.clock()
    local jobContainer = nil
    
    while os.clock() - renderStart < 3 and not jobContainer do
        rawTeleport(PAD_AREA)
        task.wait(0.1)
        jobContainer = Workspace:FindFirstChild("Game") and Workspace.Game:FindFirstChild("Jobs") and Workspace.Game.Jobs:FindFirstChild("JobPadContainer")
    end

    if not jobContainer then return false end
    local pad = jobContainer:FindFirstChild(padName)
    local padPart = pad and (pad:IsA("BasePart") and pad or pad:FindFirstChildWhichIsA("BasePart", true))
    if not padPart then return false end

    for i = 1, 20 do rawTeleport(padPart.Position + Vector3.new(0, 0.5, 0)); task.wait(0.1) end

    local clickTime = 0
    while not checkFn(localPlayer) and clickTime < 10 do
        local promptUI = localPlayer.PlayerGui:FindFirstChild("PromptUI")
        if promptUI then
            local confirmBtn = promptUI:FindFirstChild("PromptV2") and promptUI.PromptV2:FindFirstChild("ButtonsFrame") and promptUI.PromptV2.ButtonsFrame:FindFirstChild("Confirm")
            if confirmBtn then clickButton(confirmBtn) end
        else
            rawTeleport(padPart.Position + Vector3.new(0, 0.5, 0))
        end
        task.wait(0.5); clickTime = clickTime + 0.5
    end

    if checkFn(localPlayer) then notify(label, "Joined " .. label .. " team!"); return true end
    return false
end

joinSecurityTeam = function()
    if isSecurity(localPlayer) then return true end
    return joinViapad("SecurityPad", isSecurity, "Security")
end

joinOutlawTeam = function()
    if isOutlaw(localPlayer) then return true end
    return joinViapad("CriminalPad", isOutlaw, "Outlaw")
end

-- ═══════════════════════════════════════════════════════════
--  Outlaw Hunter (Full Logic Restored)
-- ═══════════════════════════════════════════════════════════
local function getOutlaws()
    local outlaws = {}
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= localPlayer and isOutlaw(player) then table.insert(outlaws, player) end
    end
    return outlaws
end

local function getOutlawNames()
    local set = {}
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= localPlayer and isOutlaw(player) then set[player.Name] = true end
    end
    return set
end

local function outlawsChanged(oldSet, newSet)
    for name in pairs(newSet) do if not oldSet[name] then return true end end
    return false
end

local function getPositionFromMemory(player)
    local char = player.Character
    if not char then return nil end
    local root = char:FindFirstChild("HumanoidRootPart") or char:FindFirstChild("UpperTorso")
    if memory_read and root and root.Address and root.Address ~= 0 then
        local prim = memory_read("uintptr", root.Address + offsets.base_part.primitive)
        if prim and prim ~= 0 then
            local x = memory_read("float", prim + offsets.primitive.cframe + 0x24)
            local y = memory_read("float", prim + offsets.primitive.cframe + 0x28)
            local z = memory_read("float", prim + offsets.primitive.cframe + 0x2C)
            if x and y and z then return Vector3.new(x, y, z) end
        end
    end
    -- Mobile fallback if no memory read
    return root and root.Position
end

local function getMyPosition()
    local char = localPlayer.Character
    if not char then return nil end
    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return nil end
    if memory_read then
        local cf = read_cframe(root)
        return cf and cf.pos
    end
    return root.Position
end

local function getClosestOutlaw()
    local myPos = getMyPosition()
    local closest, closestDist = nil, math.huge
    for _, outlaw in ipairs(getOutlaws()) do
        local pos = getPositionFromMemory(outlaw)
        if pos then
            if myPos then
                local dist = distanceBetween(myPos, pos)
                if dist < closestDist then closestDist = dist; closest = outlaw end
            else
                return outlaw
            end
        end
    end
    return closest
end

local function getATMPositions()
    return {
        Vector3.new(-2225.53, 11.89, 4091.53), Vector3.new(-1211.64, 11.88, 3697.89), Vector3.new(-2739.51, 12.39, 3874.07),
        Vector3.new(-859.39,  12.39, 3780.76), Vector3.new(-2062.00, 12.40, 3416.32), Vector3.new(-2648.79, 13.06, 4322.98),
        Vector3.new(-1346.86, 11.81, 4332.50), Vector3.new(-3126.92,  3.15, 2933.15), Vector3.new(-1600.78, 11.79, 2804.31),
        Vector3.new(-1415.05, 11.56, 2840.72), Vector3.new(-1203.72, 13.17, 2547.57), Vector3.new(-1130.09, 11.88, 3246.26),
        Vector3.new(-1895.66, 11.89, 2705.32), Vector3.new(-1249.56, 12.46, 2329.84), Vector3.new(-1992.79, 12.39, 2693.78),
        Vector3.new(-1896.50, 11.89, 2196.01), Vector3.new(-2081.52, 11.90, 2822.92), Vector3.new(-815.80,  12.51, 3097.14),
        Vector3.new(-2352.29,  7.85, 2264.55), Vector3.new(-2099.61, 12.27, 1961.71), Vector3.new(-1773.18, 13.13, 1977.24),
        Vector3.new(-2510.55,  3.15, 1858.11), Vector3.new(-2556.25, 27.11, 2477.17), Vector3.new(-1619.09, 12.04, 1825.72),
        Vector3.new(-2937.36, 27.61, 2049.01), Vector3.new(-2876.59, 27.61, 2115.64), Vector3.new(-2840.73,  3.15, 2465.16),
        Vector3.new(-2804.19,  9.01, 3059.35), Vector3.new(-2782.42,  5.85, 2828.41),
    }
end

local function searchATMsForOutlaws(shouldRestartFn)
    local atmPositions = getATMPositions()
    notify("Outlaw Hunter", "Searching ATMs for outlaws...")
    local atmIndex, searchTime = 1, 0
    while searchTime < 90 do
        if shouldRestartFn() then return nil end
        while not secEnabled() do task.wait(0.2) end

        for _ = 1, 10 do teleportTo(atmPositions[atmIndex]); task.wait(0.05) end

        local waitTime = 0
        while waitTime < 0.1 do
            if shouldRestartFn() then return nil end
            task.wait(0.1); waitTime = waitTime + 0.1
            local target = getClosestOutlaw()
            if target then
                notify("Outlaw Hunter", "Outlaw rendered at ATM " .. atmIndex .. "!")
                return target
            end
        end
        atmIndex = (atmIndex % #atmPositions) + 1
        searchTime = searchTime + 3
    end
    notify("Outlaw Hunter", "No outlaws found at ATMs.")
    return nil
end

local trackingTarget, trackingActive = nil, false

local function startTracking(target)
    trackingTarget = target
    trackingActive = true
    task.spawn(function()
        local lastPos = nil
        local oX, oY, oZ = 0, 0, 0
        while trackingActive and isOutlaw(trackingTarget) do
            if not secEnabled() then
                task.wait(0.2)
                lastPos = nil; oX, oY, oZ = 0, 0, 0
                continue
            end
            if trackingTarget.Character then
                local rp = trackingTarget.Character:FindFirstChild("HumanoidRootPart")
                if rp then
                    local cur = rp.Position
                    if memory_read then
                        local cf = read_cframe(rp)
                        if cf then cur = cf.pos end
                    end
                    
                    if lastPos then
                        local dx, dy, dz = cur.X-lastPos.X, cur.Y-lastPos.Y, cur.Z-lastPos.Z
                        local dist = math.sqrt(dx*dx + dy*dy + dz*dz)
                        if dist > 1 then
                            local spd = math.min(dist * 15, 50)
                            oX, oY, oZ = (dx/dist)*spd, (dy/dist)*spd, (dz/dist)*spd
                        elseif dist < 0.1 then
                            oX, oY, oZ = 0, 0, 0
                        end
                    end
                    lastPos = cur
                    teleportTo(Vector3.new(cur.X+oX, cur.Y+oY, cur.Z+oZ))
                end
            end
            task.wait()
        end
    end)
end

local function stopTracking()
    trackingActive = false
    trackingTarget = nil
end

-- ═══════════════════════════════════════════════════════════
--  Auto ATM (Full Logic Restored)
-- ═══════════════════════════════════════════════════════════
local ATM_SETTINGS = { HideDepth = -55, EscapeDist = 500, MinTpDist = 3000 }
local PatrolPoints = {
    Vector3.new(0, 50, 0), Vector3.new(2000, 50, 0), Vector3.new(-2000, 50, 0),
    Vector3.new(0, 50, 2000), Vector3.new(0, 50, -2000), Vector3.new(1500, 50, 1500), Vector3.new(-1500, 50, -1500)
}
local robbed_atms_memory = {}
local atm_patrol_index = 1
local atm_last_patrol = 0
local SELL_PAD = Vector3.new(-2543.308, 12.393, 4030.319)

local function getBagCount()
    local count = 0
    local bp = localPlayer:FindFirstChild("Backpack")
    if bp then for _, t in ipairs(bp:GetChildren()) do if t.Name == "CriminalMoneyBag" then count = count + 1 end end end
    local char = localPlayer.Character
    if char then for _, t in ipairs(char:GetChildren()) do if t:IsA("Tool") and t.Name == "CriminalMoneyBag" then count = count + 1 end end end
    return count
end

local function atm_teleport(target_pos)
    local char = localPlayer.Character or Workspace:FindFirstChild(localPlayer.Name)
    if not char then return end
    local root = char:FindFirstChild("HumanoidRootPart")
    if root then root.Position = target_pos; root.Velocity = Vector3.zero end
end

atm_sell_bag = function()
    local char = localPlayer.Character
    if not char or not char:FindFirstChild("HumanoidRootPart") then return end
    local root = char.HumanoidRootPart

    notify("Auto ATM", "Going to deposit...")
    local start = os.clock()
    local dest = nil
    
    while os.clock() - start < 3 do
        root.Position = SELL_PAD + Vector3.new(0, 0.5, 0)
        root.Velocity = Vector3.zero
        task.wait(0.1)
        if not dest then
            local jobs = Workspace:FindFirstChild("Game") and Workspace.Game:FindFirstChild("Jobs") or Workspace:FindFirstChild("Jobs")
            local spawner = jobs and jobs:FindFirstChild("CriminalDropOffSpawners")
            local target_part = spawner and spawner:FindFirstChild("CriminalDropOffSpawnerPermanent")
            if target_part then
                local zone = target_part:FindFirstChild("Zone")
                dest = zone and zone.Position or target_part.Position
            end
        end
    end
    
    local finalDest = dest or SELL_PAD
    local holdStart = os.clock()
    while os.clock() - holdStart < 2 do
        root.Position = finalDest + Vector3.new(0, 0.5, 0)
        root.Velocity  = Vector3.zero
        task.wait(0.1)
    end
end

local function atm_scan(root_part)
    local jobs = Workspace:FindFirstChild("Game") and Workspace.Game:FindFirstChild("Jobs") or Workspace:FindFirstChild("Jobs")
    if not jobs then return nil end

    local spawners = jobs:FindFirstChild("CriminalATMSpawners")
    local candidates = spawners and spawners:GetDescendants() or jobs:GetDescendants()
    local my_pos = root_part.Position
    local best, best_dist = nil, math.huge

    for _, obj in ipairs(candidates) do
        local atm_model, is_atm, atm_prompt = nil, false, nil

        if obj:IsA("ProximityPrompt") and obj.Enabled then
            atm_model = obj.Parent; is_atm = true; atm_prompt = obj
        elseif obj:IsA("BasePart") and string.find(obj.Name, "CriminalATMSpawner") then
            local inner = obj:FindFirstChild("CriminalATM")
            atm_model = inner or obj; is_atm = true
            atm_prompt = atm_model:FindFirstChildWhichIsA("ProximityPrompt", true)
        end

        if is_atm and atm_model then
            if atm_model:IsA("Attachment") then atm_model = atm_model.Parent end
            
            local mesh = atm_model:FindFirstChild("ATM")
            if not mesh or (not mesh:FindFirstChild("BrokenSurfaceAppearance") and not mesh:FindFirstChild("ATMElectricArc")) then
                local pos_part = atm_model:FindFirstChild("Position") or mesh or atm_model:FindFirstChildWhichIsA("BasePart", true) or atm_model
                local pos = pos_part and pos_part.Position or (atm_model:IsA("Model") and atm_model.PrimaryPart and atm_model.PrimaryPart.Position) or atm_model.Position

                if pos then
                    local id = math.floor(pos.X) .. "_" .. math.floor(pos.Z)
                    local last_t = robbed_atms_memory[id]
                    if not last_t or (os.clock() - last_t > 120) then
                        local dist = (my_pos - pos).Magnitude
                        if dist < best_dist then
                            best_dist = dist
                            best = { pos = pos, obj = pos_part, id = id, prompt = atm_prompt }
                        end
                    end
                end
            end
        end
    end
    return best
end

-- Security Evasion Thread
local _evading = false
task.spawn(function()
    while true do
        task.wait(0.3)
        if not atmEnabled() or _evading then continue end

        local char = localPlayer.Character
        if not char or not char:FindFirstChild("HumanoidRootPart") then continue end
        local my_pos = char.HumanoidRootPart.Position

        local detected = false
        for _, other in ipairs(Players:GetPlayers()) do
            if other == localPlayer then continue end
            if other.Team and other.Team.Name == "Security" then
                local oc = other.Character
                if oc and oc:FindFirstChild("HumanoidRootPart") then
                    if (my_pos - oc.HumanoidRootPart.Position).Magnitude < ATM_SETTINGS.EscapeDist then
                        detected = true; break
                    end
                end
            end
        end

        if detected then
            _evading = true
            notify("Auto ATM", "Security nearby! Evading...")
            local evadeStart = os.clock()
            local angle = math.random() * math.pi * 2
            local dist = ATM_SETTINGS.MinTpDist + math.random(0, 1000)
            local escapePos = Vector3.new(my_pos.X + math.cos(angle)*dist, my_pos.Y, my_pos.Z + math.sin(angle)*dist)
            
            while os.clock() - evadeStart < 2 do
                atm_teleport(escapePos)
                task.wait(0.1)
            end
            _evading = false
        end
    end
end)

-- ═══════════════════════════════════════════════════════════
--  Background Threads
-- ═══════════════════════════════════════════════════════════
local shouldRestart = false
local currentOutlawSet = getOutlawNames()

task.spawn(function()
    while true do
        task.wait(1)
        if secEnabled() then
            local newSet = getOutlawNames()
            if outlawsChanged(currentOutlawSet, newSet) then shouldRestart = true end
            currentOutlawSet = newSet
        end
    end
end)

task.spawn(function()
    while true do
        task.wait(0.5)
        if secEnabled() and not isSecurity(localPlayer) then joinSecurityTeam() end
    end
end)

-- ═══════════════════════════════════════════════════════════
--  Main Loop: Outlaw Hunter
-- ═══════════════════════════════════════════════════════════
task.spawn(function()
    while true do
        if not secEnabled() then stopTracking(); task.wait(0.2); continue end

        shouldRestart = false
        stopTracking()

        local outlaws = getOutlaws()
        if #outlaws == 0 then
            notify("Outlaw Hunter", "No outlaws found, waiting...")
            while #getOutlaws() == 0 do task.wait(1) end
            notify("Outlaw Hunter", "Outlaw detected! Starting hunt.")
            continue
        end

        local target = getClosestOutlaw()
        if not target then target = searchATMsForOutlaws(function() return shouldRestart or not secEnabled() end) end
        if not target then task.wait(2) continue end

        local memPos = getPositionFromMemory(target)
        if memPos then teleportTo(memPos) end
        if shouldRestart then continue end

        local rootPart = target.Character and target.Character:FindFirstChild("HumanoidRootPart")
        local rootWait = 0
        while not rootPart and rootWait < 10 do
            if shouldRestart or not secEnabled() then break end
            task.wait(0.1); rootWait = rootWait + 0.1
            rootPart = target.Character and target.Character:FindFirstChild("HumanoidRootPart")
        end
        if shouldRestart or not rootPart then task.wait(1); continue end

        notify("Outlaw Hunter", "Tracking " .. target.Name .. "...")
        startTracking(target)

        local waitTime = 0
        while isOutlaw(target) and waitTime < 30 do
            if shouldRestart or not secEnabled() then break end
            task.wait(0.1); waitTime = waitTime + 0.1
        end

        stopTracking()
        if shouldRestart then continue end
        task.wait(3)
    end
end)

-- ═══════════════════════════════════════════════════════════
--  Main Loop: Auto ATM
-- ═══════════════════════════════════════════════════════════
task.spawn(function()
    while true do
        if not atmEnabled() then task.wait(0.5) continue end

        local holdTime = UI.GetValue("atm_hold") or 7
        local waitTime = UI.GetValue("atm_wait") or 1.5
        local char = localPlayer.Character
        
        pcall(function()
            if char then for _, part in ipairs(char:GetDescendants()) do if part:IsA("BasePart") then part.CanCollide = false end end end
            if not char or not char:FindFirstChild("HumanoidRootPart") then return end
            local root = char.HumanoidRootPart

            if not localPlayer.Team or localPlayer.Team.Name ~= "Outlaw" then task.wait(3) return end

            if getBagCount() >= getSellThreshold() then
                atm_sell_bag()
                return
            end

            local target = atm_scan(root)
            if target then
                atm_last_patrol = os.clock()
                local dist = (root.Position - target.pos).Magnitude

                if dist > 15 then
                    atm_teleport(target.pos)
                else
                    local stand_pos = target.pos
                    if target.obj and target.obj:IsA("BasePart") then
                        local s, lv = pcall(function() return target.obj.CFrame.LookVector end)
                        if s and lv then stand_pos = target.pos + (lv * 4) end
                    end

                    robbed_atms_memory[target.id] = os.clock()
                    local start_rob = os.clock()
                    
                    -- Start Mobile Bypass Prompt
                    if target.prompt then pcall(function() target.prompt:InputHoldBegin() end) end

                    while os.clock() - start_rob < holdTime do
                        if not atmEnabled() then break end
                        pcall(function() root.Position = stand_pos; root.Velocity = Vector3.zero end)
                        
                        -- Secondary mobile fallback
                        if target.prompt and fireproximityprompt then fireproximityprompt(target.prompt, 1, 1) end
                        task.wait()
                    end
                    
                    -- End Mobile Bypass Prompt
                    if target.prompt then pcall(function() target.prompt:InputHoldEnd() end) end
                    task.wait(waitTime)
                end
            else
                if os.clock() - atm_last_patrol > 4 then
                    atm_patrol_index = (atm_patrol_index % #PatrolPoints) + 1
                    atm_teleport(PatrolPoints[atm_patrol_index])
                    atm_last_patrol = os.clock()
                end
                if root.Velocity then root.Velocity = Vector3.zero end
                task.wait(0.2)
            end
        end)
        task.wait()
    end
end)

print("[DE-Auto] Initialization complete! Finishing UI setup...")
OrionLib:Init()
print("[DE-Auto] Script running. The menu should be visible now.")
