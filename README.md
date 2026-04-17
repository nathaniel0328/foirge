-- DrivingEmpire.lua
-- Standalone Auto Arrest Script (Robust Root Finding & Error Prevention)

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
   LoadingSubtitle = "Robust Tracking Active",
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
        
        function sectionObj:SliderFloat(id, name, min, max, default, format)
            uiValues[id] = default
            uiElements[id] = tab:CreateSlider({
                Name = name,
                Range = {min, max},
                Increment = 0.05,
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
--  Memory & Helper Functions
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

-- Robust Root Part Finder (Fixes "No base root found" errors)
local function getRootPart(char)
    if not char then return nil end
    return char:FindFirstChild("HumanoidRootPart") 
        or char.PrimaryPart 
        or char:FindFirstChild("UpperTorso") 
        or char:FindFirstChild("Torso")
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
local function getYOffset() return UI.GetValue("oh_offset") or 4 end
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
    settings:SliderInt("oh_offset", "Y Offset (Height)", 0, 15, 4)
    settings:Tip("Keep this above 0 to prevent glitching inside cars.")
end)

-- ═══════════════════════════════════════════════════════════
--  Fixed Pad Join Logic
-- ═══════════════════════════════════════════════════════════
local function forceClickConfirm()
    local promptUI = localPlayer.PlayerGui:FindFirstChild("PromptUI")
    if promptUI then
        local confirmBtn = promptUI:FindFirstChild("Confirm", true)
        if confirmBtn and confirmBtn:IsA("GuiButton") then
            if getconnections then
                for _, conn in ipairs(getconnections(confirmBtn.MouseButton1Click)) do
                    pcall(function() conn:Fire() end)
                end
                for _, conn in ipairs(getconnections(confirmBtn.Activated)) do
                    pcall(function() conn:Fire() end)
                end
            end
            if firesignal then
                pcall(function() firesignal(confirmBtn.MouseButton1Click) end)
            end
            return true
        end
    end
    return false
end

local function joinSecurityPad()
    local PAD_AREA = CFrame.new(-109.757, 25.080, -956.793)
    
    local function tpToPad(cf)
        local char = localPlayer.Character
        if char then
            local root = getRootPart(char)
            if root then
                char:PivotTo(cf)
                root.AssemblyLinearVelocity = Vector3.zero
            end
        end
    end

    Rayfield:Notify({Title = "Security", Content = "Teleporting to Security Pad...", Duration = 3})
    
    local jobContainer = Workspace:FindFirstChild("JobPadContainer", true)
    local padPart = nil
    
    if jobContainer then
        local pad = jobContainer:FindFirstChild("SecurityPad")
        padPart = pad and (pad:IsA("BasePart") and pad or pad:FindFirstChildWhichIsA("BasePart", true))
    end

    local targetCF = padPart and padPart.CFrame + Vector3.new(0, 3, 0) or PAD_AREA
    
    local attempts = 0
    while not isSecurity(localPlayer) and attempts < 20 do
        tpToPad(targetCF)
        task.wait(0.2)
        
        local char = localPlayer.Character
        local root = getRootPart(char)
        if root then
            for _, prompt in ipairs(Workspace:GetDescendants()) do
                if prompt:IsA("ProximityPrompt") and prompt.Enabled then
                    if prompt.Parent and prompt.Parent:IsA("BasePart") then
                        local dist = (prompt.Parent.Position - root.Position).Magnitude
                        if dist < 10 then
                            pcall(function() fireproximityprompt(prompt) end)
                        end
                    end
                end
            end
        end
        
        forceClickConfirm()
        attempts = attempts + 1
    end

    if isSecurity(localPlayer) then
        Rayfield:Notify({Title = "Security", Content = "Successfully joined Security team!", Duration = 4})
        return true
    end
    
    Rayfield:Notify({Title = "Error", Content = "Failed to join Security.", Duration = 4})
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
    local myRoot = getRootPart(localPlayer.Character)
    local myPos = myRoot and myRoot.Position
    
    local closest, closestDist = nil, math.huge
    for _, outlaw in ipairs(getOutlaws()) do
        local outlawRoot = getRootPart(outlaw.Character)
        local pos = outlawRoot and outlawRoot.Position
        
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
--  Tracking Controller (Reliable CFrame Surf)
-- ═══════════════════════════════════════════════════════════
local trackingTarget, trackingActive = nil, false
local trackingConnection = nil

local function startTracking(target)
    trackingTarget = target
    trackingActive = true
    
    local char = localPlayer.Character
    local humanoid = char and char:FindFirstChild("Humanoid")
    if humanoid then
        humanoid.PlatformStand = true 
    end

    trackingConnection = RunService.Stepped:Connect(function()
        if not trackingActive or not trackingTarget or not trackingTarget.Parent or not isOutlaw(trackingTarget) or not secEnabled() then
            trackingActive = false
            if humanoid then pcall(function() humanoid.PlatformStand = false end) end
            if trackingConnection then 
                trackingConnection:Disconnect() 
                trackingConnection = nil
            end
            return
        end
        
        local root = getRootPart(char)
        if not root then return end
        
        -- Safely Disable Collisions
        for _, part in ipairs(char:GetDescendants()) do
            if part:IsA("BasePart") then
                part.CanCollide = false
            end
        end
        
        if trackingTarget.Character then
            local targetRP = getRootPart(trackingTarget.Character)
            local targetHum = trackingTarget.Character:FindFirstChild("Humanoid")
            
            local targetPartToFollow = targetRP
            local targetVelocity = Vector3.zero
            
            if targetHum and targetHum.SeatPart then
                local vehicle = targetHum.SeatPart:FindFirstAncestorWhichIsA("Model")
                if vehicle and vehicle.PrimaryPart then
                    targetPartToFollow = vehicle.PrimaryPart
                end
                targetVelocity = targetHum.SeatPart.AssemblyLinearVelocity
            elseif targetRP then
                targetVelocity = targetRP.AssemblyLinearVelocity
            end
            
            if targetPartToFollow then
                local targetPos = targetPartToFollow.Position
                
                if memory_read then
                    local cf = read_cframe(targetPartToFollow)
                    if cf then targetPos = cf.pos end
                end
                
                local offsetPos = targetPos + Vector3.new(0, math.max(getYOffset(), 2), 0)
                
                -- Pcall prevents crash if part gets destroyed mid-frame
                pcall(function()
                    root.CFrame = CFrame.new(offsetPos, offsetPos + targetPartToFollow.CFrame.LookVector)
                    root.AssemblyLinearVelocity = targetVelocity
                end)
            end
        end
    end)
end

local function stopTracking()
    trackingActive = false
    trackingTarget = nil
    if localPlayer.Character and localPlayer.Character:FindFirstChild("Humanoid") then
        pcall(function() localPlayer.Character.Humanoid.PlatformStand = false end)
    end
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
        local rootPart = getRootPart(targetChar)
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
