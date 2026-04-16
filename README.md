-- DrivingEmpire.lua
-- Standalone Auto Arrest Script

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local VirtualInputManager = game:GetService("VirtualInputManager")
local localPlayer = Players.LocalPlayer

-- ═══════════════════════════════════════════════════════════
--  UI Library Wrapper
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
   Name = "Auto Arrest Only",
   LoadingTitle = "Loading Script...",
   LoadingSubtitle = "Mobile Supported",
   ConfigurationSaving = { Enabled = false }
})

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
        
        function sectionObj:Spacing() end
        return sectionObj
    end
    callback(tabBuilder)
end

-- ═══════════════════════════════════════════════════════════
--  Original Script Memory & Offsets
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

local function write_fvector3(address, offset, vec)
    if not memory_write then return end
    memory_write("float", address + offset,       vec.X)
    memory_write("float", address + offset + 0x4, vec.Y)
    memory_write("float", address + offset + 0x8, vec.Z)
end

local function read_rotation(address)
    local r = {}
    if not memory_read then return r end
    for i, key in ipairs({"r00","r01","r02","r10","r11","r12","r20","r21","r22"}) do
        r[key] = memory_read("float", address + (i - 1) * 0x4)
    end
    return r
end

local function write_rotation(address, rotation)
    if not memory_write then return end
    local keys = {"r00","r01","r02","r10","r11","r12","r20","r21","r22"}
    for i, key in ipairs(keys) do
        memory_write("float", address + (i - 1) * 0x4, rotation[key])
    end
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

local function write_cframe(part, cframe)
    if not part or not memory_write then return end
    local prim = memory_read("uintptr", part.Address + offsets.base_part.primitive)
    if prim == 0 then return end
    write_rotation(prim + offsets.primitive.cframe, cframe.rot)
    write_fvector3(prim, offsets.primitive.cframe + 0x24, cframe.pos)
end

local function cancel_velocity(part)
    if not memory_read or not memory_write then return end
    local prim = memory_read("uintptr", part.Address + offsets.base_part.primitive)
    if prim == 0 then return end
    write_fvector3(prim, offsets.primitive.velocity, Vector3.new(0, 0, 0))
end

-- ═══════════════════════════════════════════════════════════
--  Mobile Helpers
-- ═══════════════════════════════════════════════════════════
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

local function isOutlaw(player)
    local team = player.Team
    return team and string.lower(team.Name) == "outlaw" or false
end

local function isSecurity(player)
    local team = player.Team
    return team and string.lower(team.Name) == "security" or false
end

local function secEnabled() return UI.GetValue("oh_enabled") end
local function getYOffset() return UI.GetValue("oh_offset") or 20 end

local function teleportTo(targetPosition)
    local character = localPlayer.Character
    if not character then return end
    local rootPart = character:FindFirstChild("HumanoidRootPart")
    if not rootPart then return end
    
    if memory_read and memory_write then
        local sourceCFrame = read_cframe(rootPart)
        if not sourceCFrame then return end
        sourceCFrame.pos = targetPosition + Vector3.new(0, 3 + getYOffset(), 0)
        write_cframe(rootPart, sourceCFrame)
        cancel_velocity(rootPart)
    else
        rootPart.CFrame = CFrame.new(targetPosition + Vector3.new(0, 3 + getYOffset(), 0))
        rootPart.Velocity = Vector3.zero
    end
end

-- ═══════════════════════════════════════════════════════════
--  UI Building
-- ═══════════════════════════════════════════════════════════
UI.AddTab("Auto Arrest", function(tab)
    local ctrl = tab:Section("Controls", "Left")
    ctrl:Toggle("oh_enabled", "Enable Script", false)
    ctrl:Spacing()
    ctrl:Button("Get Security Team", function()
        task.spawn(joinSecurityTeam)
    end)

    local settings = tab:Section("Settings", "Right")
    settings:SliderInt("oh_offset", "Y Offset (studs)", -50, 50, 20)
    settings:Tip("Adjusts the vertical teleport offset when tracking an outlaw")
end)

-- ═══════════════════════════════════════════════════════════
--  Pad Join Logic
-- ═══════════════════════════════════════════════════════════
local function joinSecurityPad()
    local PAD_AREA = Vector3.new(-109.757, 25.080, -956.793)
    
    local function rawTeleport(pos)
        local char = localPlayer.Character or Workspace:FindFirstChild(localPlayer.Name)
        if char and char:FindFirstChild("HumanoidRootPart") then
            char.HumanoidRootPart.Position = pos
            char.HumanoidRootPart.Velocity = Vector3.new(0, 0, 0)
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
--  Outlaw Tracking Logic
-- ═══════════════════════════════════════════════════════════
local function getOutlaws()
    local outlaws = {}
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= localPlayer and isOutlaw(player) then
            table.insert(outlaws, player)
        end
    end
    return outlaws
end

local function getPositionFromMemory(player)
    if not memory_read then return nil end
    local char = player.Character
    if not char then return nil end
    for _, name in ipairs({"HumanoidRootPart", "UpperTorso", "LowerTorso", "Head"}) do
        local part = char:FindFirstChild(name)
        if part and part.Address and part.Address ~= 0 then
            local prim = memory_read("uintptr", part.Address + offsets.base_part.primitive)
            if prim and prim ~= 0 then
                local x = memory_read("float", prim + offsets.primitive.cframe + 0x24)
                local y = memory_read("float", prim + offsets.primitive.cframe + 0x28)
                local z = memory_read("float", prim + offsets.primitive.cframe + 0x2C)
                if x and y and z then return Vector3.new(x, y, z) end
            end
        end
    end
    return nil
end

local function getClosestOutlaw()
    local myPos = localPlayer.Character and localPlayer.Character:FindFirstChild("HumanoidRootPart") and localPlayer.Character.HumanoidRootPart.Position
    local closest, closestDist = nil, math.huge
    for _, outlaw in ipairs(getOutlaws()) do
        local pos = getPositionFromMemory(outlaw) or (outlaw.Character and outlaw.Character:FindFirstChild("HumanoidRootPart") and outlaw.Character.HumanoidRootPart.Position)
        if pos then
            if myPos then
                local dx, dy, dz = myPos.X - pos.X, myPos.Y - pos.Y, myPos.Z - pos.Z
                local dist = math.sqrt(dx*dx + dy*dy + dz*dz)
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

        local outlaws = getOutlaws()
        if #outlaws == 0 then
            while #getOutlaws() == 0 do task.wait(1) end
            continue
        end

        local target = getClosestOutlaw()
        if not target then task.wait(2) continue end

        local memPos = getPositionFromMemory(target)
        if memPos then teleportTo(memPos) end

        local rootPart = target.Character and target.Character:FindFirstChild("HumanoidRootPart")
        if not rootPart then
            task.wait(1)
            continue
        end

        startTracking(target)

        local waitTime, timeout = 0, 30
        while isOutlaw(target) and waitTime < timeout do
            if not secEnabled() then break end
            task.wait(0.1)
            waitTime = waitTime + 0.1
        end

        stopTracking()
        task.wait(3)
    end
end)
