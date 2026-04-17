-- DrivingEmpire.lua
-- Standalone Auto Arrest Script (Physics Constraint Tethering for Arrest Bar)

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local VirtualInputManager = game:GetService("VirtualInputManager")
local localPlayer = Players.LocalPlayer

-- ═══════════════════════════════════════════════════════════
--  UI Library Wrapper & Auto-Save Setup
-- ═══════════════════════════════════════════════════════════
local UI = {}
local uiValues = {}
local uiElements = {}

UI.GetValue = function(id) return uiValues[id] end
UI.SetValue = function(id, val) 
    uiValues[id] = val 
    if uiElements[id] and uiElements[id].Set then
        pcall(function() uiElements[id]:Set(val) end)
    end
end

local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()
local Window = Rayfield:CreateWindow({
   Name = "Auto Arrest Pro",
   LoadingTitle = "Loading Script...",
   LoadingSubtitle = "Physics Tethering Active",
   ConfigurationSaving = { 
       Enabled = true,
       FolderName = "AutoArrestDE",
       FileName = "ArrestSettings"
   }
})

local manualSkipFlag = false

function UI.AddTab(tabName, callback)
    local tab = Window:CreateTab(tabName)
    local tabBuilder = {}
    
    function tabBuilder:Section(secName, side)
        local sectionObj = {}
        tab:CreateSection(secName)
        
        function sectionObj:Toggle(id, name, default)
            uiValues[id] = default
            uiElements[id] = tab:CreateToggle({
                Name = name,
                CurrentValue = default,
                Flag = id,
                Callback = function(val) uiValues[id] = val end
            })
        end
        
        function sectionObj:Button(name, cb)
            tab:CreateButton({ Name = name, Callback = cb })
        end
        
        function sectionObj:SliderInt(id, name, min, max, default)
            uiValues[id] = default
            uiElements[id] = tab:CreateSlider({
                Name = name,
                Range = {min, max},
                Increment = 1,
                CurrentValue = default,
                Flag = id,
                Callback = function(val) uiValues[id] = val end
            })
        end
        
        function sectionObj:Tip(text)
            tab:CreateLabel(text)
        end
        
        return sectionObj
    end
    callback(tabBuilder)
end

-- ═══════════════════════════════════════════════════════════
--  Memory Helpers (For Finding Outlaws Only)
-- ═══════════════════════════════════════════════════════════
local offsets = {
    base_part = { primitive = 0x148 },
    primitive  = { cframe = 0xC0, velocity = 0xF0 },
}

local function read_fvector3(address, offset)
    if not memory_read then return Vector3.new() end
    return Vector3.new(
        memory_read("float", address + offset),
        memory_read("float", address + offset + 0x4),
        memory_read("float", address + offset + 0x8)
    )
end

local function read_rotation(address)
    local r = {}
    if not memory_read then return r end
    for i, key in ipairs({"r00","r01","r02","r10","r11","r12","r20","r21","r22"}) do
        r[key] = memory_read("float", address + (i - 1) * 0x4)
    end
    return r
end

local function read_cframe(part)
    if not part or not memory_read then return nil end
    local prim = memory_read("uintptr", part.Address + offsets.base_part.primitive)
    if prim == 0 then return nil end
    return {
        rot = read_rotation(prim + offsets.primitive.cframe),
        pos = read_fvector3(prim, offsets.primitive.cframe + 0x24),
    }
end

local function clickButton(button)
    if getconnections then
        for _, conn in ipairs(getconnections(button.MouseButton1Click)) do
            pcall(function() conn:Fire() end)
        end
        for _, conn in ipairs(getconnections(button.Activated)) do
            pcall(function() conn:Fire() end)
        end
    else
        local absPos = button.AbsolutePosition
        local absSize = button.AbsoluteSize
        local x = absPos.X + absSize.X / 2
        local y = absPos.Y + absSize.Y / 2
        VirtualInputManager:SendMouseButtonEvent(x, y, 0, true, game, 1)
        task.wait(0.1)
        VirtualInputManager:SendMouseButtonEvent(x, y, 0, false, game, 1)
    end
end

-- ═══════════════════════════════════════════════════════════
--  Team & Core Helpers
-- ═══════════════════════════════════════════════════════════
local joinSecurityTeam
local targetBlacklist = {}

local function isOutlaw(player)
    local team = player.Team
    return team and string.lower(team.Name) == "outlaw" or false
end

local function isSecurity(player)
    local team = player.Team
    return team and string.lower(team.Name) == "security" or false
end

local function secEnabled() return UI.GetValue("oh_enabled") end
local function getYOffset() return UI.GetValue("oh_offset") or -2 end
local function getMinBounty() return UI.GetValue("oh_min_bounty") or 0 end

local function getPlayerBounty(player)
    local bounty = 0
    pcall(function()
        if player:FindFirstChild("leaderstats") and player.leaderstats:FindFirstChild("Bounty") then
            bounty = tonumber(player.leaderstats.Bounty.Value) or 0
        else
            local b = player:FindFirstChild("Bounty", true)
            if b and (b:IsA("IntValue") or b:IsA("NumberValue")) then
                bounty = tonumber(b.Value) or 0
            end
        end
    end)
    return bounty
end

local function disableCollisions()
    local char = localPlayer.Character
    if char then
        for _, part in ipairs(char:GetDescendants()) do
            if part:IsA("BasePart") then
                part.CanCollide = false
            end
        end
    end
end

-- ═══════════════════════════════════════════════════════════
--  UI Building
-- ═══════════════════════════════════════════════════════════
UI.AddTab("Auto Arrest", function(tab)
    local ctrl = tab:Section("Controls", "Left")
    ctrl:Toggle("oh_enabled", "Enable Script", false)
    
    ctrl:Button("Skip Current Target", function()
        manualSkipFlag = true
    end)
    
    ctrl:Button("Get Security Team", function()
        task.spawn(joinSecurityTeam)
    end)

    local settings = tab:Section("Settings", "Right")
    settings:SliderInt("oh_min_bounty", "Min Bounty", 0, 50000, 0)
    settings:SliderInt("oh_offset", "Static Y Offset", -10, 10, -2)
    settings:Tip("Uses physics tethers to ensure the arrest bar fills smoothly.")
end)

-- ═══════════════════════════════════════════════════════════
--  Pad Join Logic
-- ═══════════════════════════════════════════════════════════
local function joinSecurityPad()
    local PAD_AREA = Vector3.new(-109.757, 25.080, -956.793)
    
    local function rawTeleport(pos)
        local char = localPlayer.Character or Workspace:FindFirstChild(localPlayer.Name)
        if char and char:FindFirstChild("HumanoidRootPart") then
            char:PivotTo(CFrame.new(pos))
            char.HumanoidRootPart.AssemblyLinearVelocity = Vector3.zero
        end
    end

    Rayfield:Notify({Title = "Security", Content = "Teleporting to Security Pad...", Duration = 3})
    
    local renderStart = os.clock()
    local jobContainer = nil
    while os.clock() - renderStart < 3 do
        rawTeleport(PAD_AREA)
        task.wait(0.1)
        if not jobContainer then
            jobContainer = game.Workspace.Game.Jobs:FindFirstChild("JobPadContainer")
        end
    end

    if not jobContainer then return false end
    local pad = jobContainer:FindFirstChild("SecurityPad")
    if not pad then return false end

    local padPart = pad:IsA("BasePart") and pad or pad:FindFirstChildWhichIsA("BasePart", true)
    if not padPart then return false end
    
    local holdTime = 0
    while holdTime < 2 do
        rawTeleport(padPart.Position + Vector3.new(0, 0.5, 0))
        task.wait(0.1)
        holdTime = holdTime + 0.1
    end

    task.wait(1)

    local clickTime = 0
    while not isSecurity(localPlayer) and clickTime < 10 do
        local promptUI = localPlayer.PlayerGui:FindFirstChild("PromptUI")
        if promptUI then
            local v2 = promptUI:FindFirstChild("PromptV2")
            local confirmBtn = v2 and v2:FindFirstChild("ButtonsFrame") and v2.ButtonsFrame:FindFirstChild("Confirm")
            if confirmBtn then
                clickButton(confirmBtn)
            else
                rawTeleport(padPart.Position + Vector3.new(0, 0.5, 0))
            end
        end
        task.wait(0.5)
        clickTime = clickTime + 0.5
    end

    if isSecurity(localPlayer) then
        Rayfield:Notify({Title = "Security", Content = "Joined Security team!", Duration = 4})
        return true
    end
    return false
end

local isJoiningSecurity = false
joinSecurityTeam = function()
    if isSecurity(localPlayer) then return true end
    if isJoiningSecurity then return false end
    isJoiningSecurity = true
    local ok = joinSecurityPad()
    isJoiningSecurity = false
    return ok
end

-- ═══════════════════════════════════════════════════════════
--  Physics Constraint Tracking System
-- ═══════════════════════════════════════════════════════════
local activeConstraints = {}

local function clearConstraints()
    for _, obj in ipairs(activeConstraints) do
        if obj and obj.Parent then obj:Destroy() end
    end
    table.clear(activeConstraints)
end

local function applyPhysicsTether(targetPart)
    local char = localPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if not root then return end

    clearConstraints()

    -- 1. My Attachment
    local att0 = Instance.new("Attachment")
    att0.Name = "ArrestAtt0"
    att0.Parent = root
    table.insert(activeConstraints, att0)

    -- 2. Their Attachment (with the offset)
    local att1 = Instance.new("Attachment")
    att1.Name = "ArrestAtt1"
    att1.Position = Vector3.new(0, getYOffset(), 0)
    att1.Parent = targetPart
    table.insert(activeConstraints, att1)

    -- 3. The physics glue that pulls us to them smoothly
    local alignPos = Instance.new("AlignPosition")
    alignPos.Attachment0 = att0
    alignPos.Attachment1 = att1
    alignPos.RigidityEnabled = true -- This forces instant physics snapping
    alignPos.Parent = root
    table.insert(activeConstraints, alignPos)

    -- 4. Keep our rotation matching theirs so we don't spin wildly
    local alignOri = Instance.new("AlignOrientation")
    alignOri.Attachment0 = att0
    alignOri.Attachment1 = att1
    alignOri.RigidityEnabled = true
    alignOri.Parent = root
    table.insert(activeConstraints, alignOri)
end

-- ═══════════════════════════════════════════════════════════
--  Outlaw Finding Logic
-- ═══════════════════════════════════════════════════════════
local function getOutlaws()
    local outlaws = {}
    local minBounty = math.max(1, getMinBounty()) 
    local currentTime = os.clock()
    
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= localPlayer and isOutlaw(player) then
            if targetBlacklist[player.UserId] and currentTime < targetBlacklist[player.UserId] then
                continue 
            end
            if getPlayerBounty(player) >= minBounty then
                table.insert(outlaws, player)
            end
        end
    end
    return outlaws
end

local function getClosestOutlaw()
    local myPos = localPlayer.Character and localPlayer.Character:FindFirstChild("HumanoidRootPart") and localPlayer.Character.HumanoidRootPart.Position
    local closest, closestDist = nil, math.huge
    for _, outlaw in ipairs(getOutlaws()) do
        local pos = (outlaw.Character and outlaw.Character:FindFirstChild("HumanoidRootPart") and outlaw.Character.HumanoidRootPart.Position)
        if pos then
            if myPos then
                local dist = (myPos - pos).Magnitude
                if dist < closestDist then
                    closestDist = dist
                    closest = outlaw
                end
            else
                return outlaw
            end
        end
    end
    return closest
end

-- ═══════════════════════════════════════════════════════════
--  Tracking Controller
-- ═══════════════════════════════════════════════════════════
local trackingTarget, trackingActive = nil, false
local trackingConnection = nil
local currentTetheredPart = nil

local function startTracking(target)
    trackingTarget = target
    trackingActive = true
    
    -- Teleport close first to avoid physics fling
    if target.Character and target.Character:FindFirstChild("HumanoidRootPart") and localPlayer.Character then
        localPlayer.Character:PivotTo(target.Character.HumanoidRootPart.CFrame)
    end
    
    trackingConnection = RunService.Heartbeat:Connect(function()
        if not trackingActive or not isOutlaw(trackingTarget) or not secEnabled() then
            trackingActive = false
            clearConstraints()
            if trackingConnection then 
                trackingConnection:Disconnect() 
                trackingConnection = nil
            end
            return
        end
        
        disableCollisions() -- MUST keep off or you will bounce off their car
        
        if trackingTarget.Character then
            local rp = trackingTarget.Character:FindFirstChild("HumanoidRootPart")
            local humanoid = trackingTarget.Character:FindFirstChild("Humanoid")
            
            local targetPartToStickTo = rp
            
            -- If they are in a car, we must tether to the car's primary part, not them directly.
            if humanoid and humanoid.SeatPart then
                local vehicle = humanoid.SeatPart:FindFirstAncestorWhichIsA("Model")
                if vehicle and vehicle.PrimaryPart then
                    targetPartToStickTo = vehicle.PrimaryPart
                end
            end
            
            -- If they switch vehicles or respawn, re-apply the tethers to the new part
            if targetPartToStickTo and targetPartToStickTo ~= currentTetheredPart then
                currentTetheredPart = targetPartToStickTo
                applyPhysicsTether(currentTetheredPart)
            end
            
            -- Dynamic offset adjustment if slider changes while tracking
            for _, obj in ipairs(activeConstraints) do
                if obj.Name == "ArrestAtt1" then
                    obj.Position = Vector3.new(0, getYOffset(), 0)
                end
            end
        end
    end)
end

local function stopTracking()
    trackingActive = false
    trackingTarget = nil
    currentTetheredPart = nil
    clearConstraints()
    if trackingConnection then
        trackingConnection:Disconnect()
        trackingConnection = nil
    end
end

-- ═══════════════════════════════════════════════════════════
--  Main Execution Loop
-- ═══════════════════════════════════════════════════════════
task.spawn(function()
    while true do
        task.wait(0.5)
        if secEnabled() and not isSecurity(localPlayer) and not isJoiningSecurity then
            joinSecurityTeam()
        end
    end
end)

task.spawn(function()
    while true do
        if not secEnabled() then
            stopTracking()
            task.wait(0.2)
            continue
        end

        stopTracking()
        manualSkipFlag = false

        local outlaws = getOutlaws()
        if #outlaws == 0 then
            while #getOutlaws() == 0 do task.wait(1) end
            continue
        end

        local target = getClosestOutlaw()
        if not target then task.wait(0.5) continue end

        local targetChar = target.Character
        local rootPart = targetChar and targetChar:FindFirstChild("HumanoidRootPart")
        if not rootPart then
            task.wait(0.5)
            continue
        end

        startTracking(target)

        while isOutlaw(target) and target.Character == targetChar do
            if not secEnabled() then break end
            if manualSkipFlag then break end
            
            local currentBounty = getPlayerBounty(target)
            if currentBounty == 0 or currentBounty < getMinBounty() then
                break
            end
            
            task.wait(0.1)
        end

        if manualSkipFlag then
            if target and isOutlaw(target) then
                Rayfield:Notify({Title = "Auto Arrest", Content = "Manually Skipped " .. target.Name, Duration = 3})
                targetBlacklist[target.UserId] = os.clock() + 30 
            end
        end

        manualSkipFlag = false
        stopTracking()
        task.wait(0.1)
    end
end)
