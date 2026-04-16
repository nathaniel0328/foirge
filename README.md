-- DrivingEmpire.lua (Fully Standalone Mobile UI Version)

local Players   = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local CoreGui   = game:GetService("CoreGui")
local localPlayer = Players.LocalPlayer

local offsets = {
    base_part = { primitive = 0x148 },
    primitive  = { cframe = 0xC0, velocity = 0xF0 },
}

-- ─────────────────────────────────────────────
--  Global State (Replaces External UI Libraries)
-- ─────────────────────────────────────────────
local STATE = {
    auto_arrest = false,
    auto_rob = false,
    y_offset = 20,
    atm_hold = 7,
    atm_wait = 1.5,
    sell_threshold = 10
}

local function secEnabled() return STATE.auto_arrest end
local function atmEnabled() return STATE.auto_rob end
local function getYOffset() return STATE.y_offset end
local function getSellThreshold() return STATE.sell_threshold end

-- ─────────────────────────────────────────────
--  Mobile UI Generation
-- ─────────────────────────────────────────────
if CoreGui:FindFirstChild("DEMobileHub") then
    CoreGui.DEMobileHub:Destroy()
elseif localPlayer:WaitForChild("PlayerGui"):FindFirstChild("DEMobileHub") then
    localPlayer.PlayerGui.DEMobileHub:Destroy()
end

local mobileGui = Instance.new("ScreenGui")
mobileGui.Name = "DEMobileHub"
mobileGui.ResetOnSpawn = false
local success = pcall(function() mobileGui.Parent = CoreGui end)
if not success then mobileGui.Parent = localPlayer:WaitForChild("PlayerGui") end

-- Open/Close Button
local toggleBtn = Instance.new("TextButton", mobileGui)
toggleBtn.Size = UDim2.new(0, 110, 0, 40)
toggleBtn.Position = UDim2.new(0, 10, 0, 10)
toggleBtn.Text = "Open Menu"
toggleBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
toggleBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
toggleBtn.Font = Enum.Font.GothamBold
toggleBtn.TextSize = 14
toggleBtn.Active = true
toggleBtn.Draggable = true
toggleBtn.BorderSizePixel = 2
toggleBtn.BorderColor3 = Color3.fromRGB(50, 150, 255)

-- Main Frame
local mainFrame = Instance.new("Frame", mobileGui)
mainFrame.Size = UDim2.new(0, 300, 0, 350)
mainFrame.Position = UDim2.new(0.5, -150, 0.5, -175)
mainFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
mainFrame.BorderSizePixel = 2
mainFrame.BorderColor3 = Color3.fromRGB(50, 150, 255)
mainFrame.Active = true
mainFrame.Draggable = true
mainFrame.Visible = false

local titleBar = Instance.new("TextLabel", mainFrame)
titleBar.Size = UDim2.new(1, 0, 0, 30)
titleBar.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
titleBar.Text = " Driving Empire Hub"
titleBar.TextColor3 = Color3.fromRGB(255, 255, 255)
titleBar.Font = Enum.Font.GothamBold
titleBar.TextSize = 16
titleBar.TextXAlignment = Enum.TextXAlignment.Left

toggleBtn.Activated:Connect(function()
    mainFrame.Visible = not mainFrame.Visible
end)

-- Tab Buttons
local tabContainer = Instance.new("Frame", mainFrame)
tabContainer.Size = UDim2.new(1, 0, 0, 35)
tabContainer.Position = UDim2.new(0, 0, 0, 30)
tabContainer.BackgroundTransparency = 1

local btnTabArrest = Instance.new("TextButton", tabContainer)
btnTabArrest.Size = UDim2.new(0.5, 0, 1, 0)
btnTabArrest.Text = "Auto Arrest"
btnTabArrest.Font = Enum.Font.GothamBold
btnTabArrest.TextColor3 = Color3.new(1,1,1)
btnTabArrest.BackgroundColor3 = Color3.fromRGB(50, 50, 50)

local btnTabRob = Instance.new("TextButton", tabContainer)
btnTabRob.Size = UDim2.new(0.5, 0, 1, 0)
btnTabRob.Position = UDim2.new(0.5, 0, 0, 0)
btnTabRob.Text = "Auto Rob"
btnTabRob.Font = Enum.Font.GothamBold
btnTabRob.TextColor3 = Color3.new(1,1,1)
btnTabRob.BackgroundColor3 = Color3.fromRGB(30, 30, 30)

-- Pages
local pageArrest = Instance.new("ScrollingFrame", mainFrame)
pageArrest.Size = UDim2.new(1, 0, 1, -65)
pageArrest.Position = UDim2.new(0, 0, 0, 65)
pageArrest.BackgroundTransparency = 1
pageArrest.CanvasSize = UDim2.new(0, 0, 0, 300)
pageArrest.ScrollBarThickness = 4

local pageRob = Instance.new("ScrollingFrame", mainFrame)
pageRob.Size = UDim2.new(1, 0, 1, -65)
pageRob.Position = UDim2.new(0, 0, 0, 65)
pageRob.BackgroundTransparency = 1
pageRob.CanvasSize = UDim2.new(0, 0, 0, 400)
pageRob.ScrollBarThickness = 4
pageRob.Visible = false

btnTabArrest.Activated:Connect(function()
    pageArrest.Visible = true; pageRob.Visible = false
    btnTabArrest.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
    btnTabRob.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
end)

btnTabRob.Activated:Connect(function()
    pageArrest.Visible = false; pageRob.Visible = true
    btnTabArrest.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    btnTabRob.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
end)

-- UI Helper Functions
local uiYArrest = 10
local uiYRob = 10

local function createToggle(page, text, stateKey, startY)
    local btn = Instance.new("TextButton", page)
    btn.Size = UDim2.new(0.9, 0, 0, 40)
    btn.Position = UDim2.new(0.05, 0, 0, startY)
    btn.Text = text .. ": OFF"
    btn.Font = Enum.Font.GothamBold
    btn.TextColor3 = Color3.new(1,1,1)
    btn.BackgroundColor3 = Color3.fromRGB(150, 50, 50)
    
    btn.Activated:Connect(function()
        STATE[stateKey] = not STATE[stateKey]
        if STATE[stateKey] then
            btn.Text = text .. ": ON"
            btn.BackgroundColor3 = Color3.fromRGB(50, 150, 50)
        else
            btn.Text = text .. ": OFF"
            btn.BackgroundColor3 = Color3.fromRGB(150, 50, 50)
        end
    end)
    return startY + 50
end

local function createButton(page, text, callback, startY)
    local btn = Instance.new("TextButton", page)
    btn.Size = UDim2.new(0.9, 0, 0, 35)
    btn.Position = UDim2.new(0.05, 0, 0, startY)
    btn.Text = text
    btn.Font = Enum.Font.GothamBold
    btn.TextColor3 = Color3.new(1,1,1)
    btn.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
    btn.Activated:Connect(callback)
    return startY + 45
end

local function createAdjuster(page, text, stateKey, step, startY)
    local frame = Instance.new("Frame", page)
    frame.Size = UDim2.new(0.9, 0, 0, 40)
    frame.Position = UDim2.new(0.05, 0, 0, startY)
    frame.BackgroundTransparency = 1
    
    local lbl = Instance.new("TextLabel", frame)
    lbl.Size = UDim2.new(0.5, 0, 1, 0)
    lbl.Text = text
    lbl.TextColor3 = Color3.new(1,1,1)
    lbl.Font = Enum.Font.GothamBold
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.BackgroundTransparency = 1
    
    local valLbl = Instance.new("TextLabel", frame)
    valLbl.Size = UDim2.new(0.2, 0, 1, 0)
    valLbl.Position = UDim2.new(0.65, 0, 0, 0)
    valLbl.Text = tostring(STATE[stateKey])
    valLbl.TextColor3 = Color3.new(1,1,1)
    valLbl.Font = Enum.Font.GothamBold
    valLbl.BackgroundTransparency = 1
    
    local btnSub = Instance.new("TextButton", frame)
    btnSub.Size = UDim2.new(0.15, 0, 0.8, 0)
    btnSub.Position = UDim2.new(0.5, 0, 0.1, 0)
    btnSub.Text = "-"
    btnSub.Font = Enum.Font.GothamBold
    btnSub.BackgroundColor3 = Color3.fromRGB(50,50,50)
    btnSub.TextColor3 = Color3.new(1,1,1)
    
    local btnAdd = Instance.new("TextButton", frame)
    btnAdd.Size = UDim2.new(0.15, 0, 0.8, 0)
    btnAdd.Position = UDim2.new(0.85, 0, 0.1, 0)
    btnAdd.Text = "+"
    btnAdd.Font = Enum.Font.GothamBold
    btnAdd.BackgroundColor3 = Color3.fromRGB(50,50,50)
    btnAdd.TextColor3 = Color3.new(1,1,1)
    
    btnSub.Activated:Connect(function()
        STATE[stateKey] = STATE[stateKey] - step
        valLbl.Text = tostring(STATE[stateKey])
    end)
    btnAdd.Activated:Connect(function()
        STATE[stateKey] = STATE[stateKey] + step
        valLbl.Text = tostring(STATE[stateKey])
    end)
    
    return startY + 50
end

-- Forward Declarations
local joinSecurityTeam, joinOutlawTeam, atm_sell_bag

-- Populate Arrest Page
uiYArrest = createToggle(pageArrest, "Enable Auto Arrest", "auto_arrest", uiYArrest)
uiYArrest = createButton(pageArrest, "Get Security Team", function() task.spawn(joinSecurityTeam) end, uiYArrest)
uiYArrest = createAdjuster(pageArrest, "Y Offset (Studs)", "y_offset", 2, uiYArrest)

-- Populate Rob Page
uiYRob = createToggle(pageRob, "Enable Auto Rob", "auto_rob", uiYRob)
uiYRob = createButton(pageRob, "Get Outlaw Team", function() task.spawn(joinOutlawTeam) end, uiYRob)
uiYRob = createButton(pageRob, "Deposit Bags Now", function() task.spawn(atm_sell_bag) end, uiYRob)
uiYRob = createAdjuster(pageRob, "Sell Threshold (Bags)", "sell_threshold", 1, uiYRob)
uiYRob = createAdjuster(pageRob, "Hold Time (s)", "atm_hold", 1, uiYRob)
uiYRob = createAdjuster(pageRob, "Wait Time (s)", "atm_wait", 0.5, uiYRob)


-- ─────────────────────────────────────────────
--  ProximityPrompt offset 
-- ─────────────────────────────────────────────
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
--  Memory helpers  (Outlaw Hunter)
-- ═══════════════════════════════════════════════════════════
local function read_fvector3(address, offset)
    return Vector3.new(
        memory_read("float", address + offset),
        memory_read("float", address + offset + 0x4),
        memory_read("float", address + offset + 0x8)
    )
end

local function write_fvector3(address, offset, vec)
    memory_write("float", address + offset,       vec.X)
    memory_write("float", address + offset + 0x4, vec.Y)
    memory_write("float", address + offset + 0x8, vec.Z)
end

local function read_rotation(address)
    local r = {}
    for i, key in ipairs({"r00","r01","r02","r10","r11","r12","r20","r21","r22"}) do
        r[key] = memory_read("float", address + (i - 1) * 0x4)
    end
    return r
end

local function write_rotation(address, rotation)
    local keys = {"r00","r01","r02","r10","r11","r12","r20","r21","r22"}
    for i, key in ipairs(keys) do
        memory_write("float", address + (i - 1) * 0x4, rotation[key])
    end
end

local function read_cframe(part)
    if not part then return nil end
    local prim = memory_read("uintptr", part.Address + offsets.base_part.primitive)
    if prim == 0 then return nil end
    return {
        rot = read_rotation(prim + offsets.primitive.cframe),
        pos = read_fvector3(prim, offsets.primitive.cframe + 0x24),
    }
end

local function write_cframe(part, cframe)
    if not part then return end
    local prim = memory_read("uintptr", part.Address + offsets.base_part.primitive)
    if prim == 0 then return end
    write_rotation(prim + offsets.primitive.cframe, cframe.rot)
    write_fvector3(prim, offsets.primitive.cframe + 0x24, cframe.pos)
end

local function cancel_velocity(part)
    local prim = memory_read("uintptr", part.Address + offsets.base_part.primitive)
    if prim == 0 then return end
    write_fvector3(prim, offsets.primitive.velocity, Vector3.new(0, 0, 0))
end

local function teleportTo(targetPosition)
    local character = localPlayer.Character
    if not character then return end
    local rootPart = character:FindFirstChild("HumanoidRootPart")
    if not rootPart then return end
    local sourceCFrame = read_cframe(rootPart)
    if not sourceCFrame then return end
    sourceCFrame.pos = targetPosition + Vector3.new(0, 3 + getYOffset(), 0)
    write_cframe(rootPart, sourceCFrame)
    cancel_velocity(rootPart)
end

-- ═══════════════════════════════════════════════════════════
--  Team helpers & Shared Join Pad
-- ═══════════════════════════════════════════════════════════
local function isOutlaw(player)
    local team = player.Team
    if not team then return false end
    return string.lower(team.Name) == "outlaw"
end

local function isSecurity(player)
    local team = player.Team
    if not team then return false end
    return string.lower(team.Name) == "security"
end

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
        if not root then return end
        root.Position = pos
        root.Velocity  = Vector3.new(0, 0, 0)
    end

    local renderStart = os.clock()
    local jobContainer = nil
    while os.clock() - renderStart < 3 do
        rawTeleport(PAD_AREA)
        task.wait(0.1)
        if not jobContainer then
            jobContainer = game.Workspace.Game.Jobs:FindFirstChild("JobPadContainer")
        end
    end

    local extraAttempts = 0
    while not jobContainer and extraAttempts < 100 do
        rawTeleport(PAD_AREA)
        task.wait(0.1)
        jobContainer = game.Workspace.Game.Jobs:FindFirstChild("JobPadContainer")
        extraAttempts = extraAttempts + 1
    end

    if not jobContainer then return false end
    local pad = jobContainer:FindFirstChild(padName)
    if not pad then return false end

    local padPart = pad:IsA("BasePart") and pad or pad:FindFirstChildWhichIsA("BasePart", true)
    if not padPart then return false end
    local padPos = padPart.Position

    local function teleportToPad(pos)
        local char = localPlayer.Character or Workspace:FindFirstChild(localPlayer.Name)
        if not char then return end
        local root = char:FindFirstChild("HumanoidRootPart")
        if not root then return end
        root.Position = pos + Vector3.new(0, 0.5, 0)
        root.Velocity  = Vector3.new(0, 0, 0)
    end
    
    local holdTime = 0
    while holdTime < 2 do
        teleportToPad(padPos)
        task.wait(0.1)
        holdTime = holdTime + 0.1
    end
    task.wait(1)

    local clickTimeout = 10
    local clickTime = 0
    while not checkFn(localPlayer) and clickTime < clickTimeout do
        local promptUI = localPlayer.PlayerGui:FindFirstChild("PromptUI")
        if promptUI then
            local v2 = promptUI:FindFirstChild("PromptV2")
            local bf = v2 and v2:FindFirstChild("ButtonsFrame")
            local confirmBtn = bf and bf:FindFirstChild("Confirm")
            if confirmBtn then
                local fired = false
                if getconnections then
                    for _, conn in ipairs(getconnections(confirmBtn.Activated)) do conn:Fire(); fired = true end
                    for _, conn in ipairs(getconnections(confirmBtn.MouseButton1Click)) do conn:Fire(); fired = true end
                end
                
                if not fired then
                    local vim = game:GetService("VirtualInputManager")
                    local center = confirmBtn.AbsolutePosition + (confirmBtn.AbsoluteSize / 2)
                    vim:SendMouseButtonEvent(center.X, center.Y + 36, 0, true, game, 1)
                    task.wait(0.1)
                    vim:SendMouseButtonEvent(center.X, center.Y + 36, 0, false, game, 1)
                end
            else
                teleportToPad(padPos)
            end
        end

        task.wait(0.5)
        clickTime = clickTime + 0.5
    end

    if checkFn(localPlayer) then return true end
    return false
end

-- ═══════════════════════════════════════════════════════════
--  Outlaw Hunter logic
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

local function getMyPosition()
    local char = localPlayer.Character
    if not char then return nil end
    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return nil end
    local cf = read_cframe(root)
    return cf and cf.pos
end

local function distanceBetween(a, b)
    local dx, dy, dz = a.X - b.X, a.Y - b.Y, a.Z - b.Z
    return math.sqrt(dx*dx + dy*dy + dz*dz)
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

local function getClosestOutlaw()
    local myPos = getMyPosition()
    local closest, closestDist = nil, math.huge
    for _, outlaw in ipairs(getOutlaws()) do
        local pos = getPositionFromMemory(outlaw)
        if pos then
            if myPos then
                local dist = distanceBetween(myPos, pos)
                if dist < closestDist then closestDist = dist; closest = outlaw end
            else return outlaw end
        end
    end
    return closest
end

local function searchATMsForOutlaws(shouldRestartFn)
    local atmPositions = getATMPositions()
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
            if target then return target end
        end

        atmIndex = (atmIndex % #atmPositions) + 1
        searchTime = searchTime + 3
    end
    return nil
end

local trackingTarget, trackingActive = nil, false
local function startTracking(target)
    trackingTarget, trackingActive = target, true
    task.spawn(function()
        local lastPos = nil
        local oX, oY, oZ = 0, 0, 0
        while trackingActive and isOutlaw(trackingTarget) do
            if not secEnabled() then
                task.wait(0.2); lastPos = nil; oX, oY, oZ = 0, 0, 0; continue
            end
            if trackingTarget.Character then
                local rp = trackingTarget.Character:FindFirstChild("HumanoidRootPart")
                if rp then
                    local cf = read_cframe(rp)
                    if cf then
                        local cur = cf.pos
                        if lastPos then
                            local dx, dy, dz = cur.X-lastPos.X, cur.Y-lastPos.Y, cur.Z-lastPos.Z
                            local dist = math.sqrt(dx*dx + dy*dy + dz*dz)
                            if dist > 1 then
                                local spd = math.min(dist * 15, 50)
                                oX, oY, oZ = (dx/dist)*spd, (dy/dist)*spd, (dz/dist)*spd
                            elseif dist < 0.1 then oX, oY, oZ = 0, 0, 0 end
                        end
                        lastPos = cur
                        teleportTo(Vector3.new(cur.X+oX, cur.Y+oY, cur.Z+oZ))
                    end
                end
            end
            task.wait()
        end
    end)
end

local function stopTracking() trackingActive = false; trackingTarget = nil end

local isJoiningSecurity = false
joinSecurityTeam = function()
    if isSecurity(localPlayer) then return true end
    if isJoiningSecurity then return false end
    isJoiningSecurity = true
    local ok = joinViapad("SecurityPad", isSecurity, "Security")
    isJoiningSecurity = false
    return ok
end

-- ═══════════════════════════════════════════════════════════
--  Auto ATM logic
-- ═══════════════════════════════════════════════════════════
local ATM_SETTINGS = { EscapeDist = 500, MinTpDist = 3000 }
local PatrolPoints = {
    Vector3.new(0, 50, 0), Vector3.new(2000, 50, 0), Vector3.new(-2000, 50, 0),
    Vector3.new(0, 50, 2000), Vector3.new(0, 50, -2000), Vector3.new(1500, 50, 1500), Vector3.new(-1500, 50, -1500),
}
local robbed_atms_memory, atm_patrol_index, atm_last_patrol = {}, 1, 0

local function getBagCount()
    local count = 0
    local backpack = localPlayer:FindFirstChild("Backpack")
    if backpack then for _, tool in ipairs(backpack:GetChildren()) do if tool.Name == "CriminalMoneyBag" then count = count + 1 end end end
    local char = localPlayer.Character
    if char then for _, tool in ipairs(char:GetChildren()) do if tool:IsA("Tool") and tool.Name == "CriminalMoneyBag" then count = count + 1 end end end
    return count
end

local function atm_get_position(object)
    if not object then return nil end
    if typeof(object) == "Vector3" then return object end
    if typeof(object) == "CFrame"  then return object.Position end
    if object:IsA("BasePart")      then return object.Position end
    if object:IsA("Model") then
        if object.PrimaryPart then return object.PrimaryPart.Position end
        local p = object:FindFirstChildWhichIsA("BasePart", true)
        return p and p.Position
    end
    return nil
end

local function atm_enable_noclip()
    local char = localPlayer.Character
    if char then for _, part in ipairs(char:GetDescendants()) do if part:IsA("BasePart") then part.CanCollide = false end end end
end

local function atm_teleport(target_pos)
    local char = localPlayer.Character or Workspace:FindFirstChild(localPlayer.Name)
    if not char then return end
    local root = char:FindFirstChild("HumanoidRootPart")
    if root then
        if root.Position then root.Position = target_pos end
        if root.Velocity then root.Velocity = Vector3.new(0, 0, 0) end
    end
end

local function atm_is_broken(atm_model)
    local mesh = atm_model:FindFirstChild("ATM")
    if mesh and (mesh:FindFirstChild("BrokenSurfaceAppearance") or mesh:FindFirstChild("ATMElectricArc")) then return true end
    return false
end

local function atm_escape_pos(from_pos)
    local angle = math.random() * math.pi * 2
    local dist  = ATM_SETTINGS.MinTpDist + math.random(0, 1000)
    return Vector3.new(from_pos.X + math.cos(angle) * dist, from_pos.Y, from_pos.Z + math.sin(angle) * dist)
end

local function atm_scan(root_part)
    local jobs = (Workspace:FindFirstChild("Game") and Workspace.Game:FindFirstChild("Jobs")) or Workspace:FindFirstChild("Jobs")
    if not jobs or not root_part then return nil end

    local candidates = {}
    local spawners = jobs:FindFirstChild("CriminalATMSpawners")
    if spawners then candidates = spawners:GetDescendants() else candidates = jobs:GetDescendants() end

    local my_pos = root_part.Position
    local best, best_dist = nil, math.huge

    for _, obj in ipairs(candidates) do
        local atm_model, is_atm = nil, false
        if obj:IsA("ProximityPrompt") and obj.Enabled then
            atm_model = obj.Parent; is_atm = true
        elseif obj:IsA("BasePart") and string.find(obj.Name, "CriminalATMSpawner") then
            local inner = obj:FindFirstChild("CriminalATM")
            atm_model = inner or obj; is_atm = true
        end

        if is_atm and atm_model then
            if atm_model:IsA("Attachment") then atm_model = atm_model.Parent end
            if atm_model and atm_model.Name == "CriminalATM" then
                if atm_is_broken(atm_model) then atm_model = nil
                else
                    local mesh = atm_model:FindFirstChild("ATM")
                    local pos_part = atm_model:FindFirstChild("Position") or mesh
                    atm_model = pos_part or atm_model:FindFirstChildWhichIsA("BasePart", true) or atm_model
                end
            end

            if atm_model then
                local pos = atm_get_position(atm_model)
                if pos then
                    local id = math.floor(pos.X) .. "_" .. math.floor(pos.Z)
                    local last_t = robbed_atms_memory[id]
                    if not last_t or (os.clock() - last_t > 120) then
                        local dx, dy, dz = my_pos.X - pos.X, my_pos.Y - pos.Y, my_pos.Z - pos.Z
                        local dist = math.sqrt(dx*dx + dy*dy + dz*dz)
                        if type(dist) == "number" and dist < best_dist then
                            best_dist = dist; best = { pos = pos, obj = atm_model, id = id }
                        end
                    end
                end
            end
        end
    end
    return best
end

local SELL_PAD = Vector3.new(-2543.308, 12.393, 4030.319)
atm_sell_bag = function()
    local char = localPlayer.Character or Workspace:FindFirstChild(localPlayer.Name)
    if not char then return end
    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return end

    local start = os.clock()
    local dest = nil
    while os.clock() - start < 3 do
        root.Position = SELL_PAD + Vector3.new(0, 0.5, 0)
        root.Velocity  = Vector3.new(0, 0, 0)
        task.wait(0.1)
        if not dest then
            local jobs = (Workspace:FindFirstChild("Game") and Workspace.Game:FindFirstChild("Jobs")) or Workspace:FindFirstChild("Jobs")
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
        root.Velocity  = Vector3.new(0, 0, 0)
        task.wait(0.1)
    end
    task.wait(1)
end

local isJoiningOutlaw = false
joinOutlawTeam = function()
    if isOutlaw(localPlayer) then return true end
    if isJoiningOutlaw then return false end
    isJoiningOutlaw = true
    local ok, err = pcall(function() return joinViapad("CriminalPad", isOutlaw, "Outlaw") end)
    isJoiningOutlaw = false
    return ok
end

local _evading = false
task.spawn(function()
    while true do
        task.wait(0.3)
        if not atmEnabled() or _evading then continue end

        local char = localPlayer.Character or Workspace:FindFirstChild(localPlayer.Name)
        if not char then continue end
        local root = char:FindFirstChild("HumanoidRootPart")
        if not root or not root.Position then continue end
        local my_pos = root.Position

        local detected = false
        for _, other in ipairs(Players:GetPlayers()) do
            if other == localPlayer then continue end
            local team = other.Team
            if not team or team.Name ~= "Security" then continue end
            local oc = other.Character or Workspace:FindFirstChild(other.Name)
            if not oc then continue end
            local or_ = oc:FindFirstChild("HumanoidRootPart")
            if not or_ or not or_.Position then continue end
            local dx, dy, dz = my_pos.X - or_.Position.X, my_pos.Y - or_.Position.Y, my_pos.Z - or_.Position.Z
            if math.sqrt(dx*dx + dy*dy + dz*dz) < ATM_SETTINGS.EscapeDist then
                detected = true; break
            end
        end

        if detected then
            _evading = true
            local evadeStart = os.clock()
            while os.clock() - evadeStart < 2 do
                atm_teleport(atm_escape_pos(my_pos)); task.wait(0.1)
            end
            _evading = false
        end
    end
end)

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
        if secEnabled() and not isSecurity(localPlayer) and not isJoiningSecurity then
            joinSecurityTeam()
        end
    end
end)


-- ═══════════════════════════════════════════════════════════
--  Main Threads
-- ═══════════════════════════════════════════════════════════
task.spawn(function()
    while true do
        if not secEnabled() then
            stopTracking(); task.wait(0.2); continue
        end

        shouldRestart = false
        stopTracking()

        local outlaws = getOutlaws()
        if #outlaws == 0 then
            while #getOutlaws() == 0 do task.wait(1) end
            continue
        end

        local target = getClosestOutlaw()
        if not target then
            target = searchATMsForOutlaws(function() return shouldRestart or not secEnabled() end)
        end
        if not target then task.wait(2) continue end

        local memPos = getPositionFromMemory(target)
        if memPos then teleportTo(memPos) end
        if shouldRestart then continue end

        local rootPart = target.Character and target.Character:FindFirstChild("HumanoidRootPart")
        if not rootPart then
            local rootWait = 0
            while not rootPart and rootWait < 10 do
                if shouldRestart or not secEnabled() then break end
                task.wait(0.1); rootWait = rootWait + 0.1
                rootPart = target.Character and target.Character:FindFirstChild("HumanoidRootPart")
            end
        end
        if shouldRestart then continue end

        if not rootPart then task.wait(1); continue end

        startTracking(target)

        local waitTime, timeout = 0, 30
        while isOutlaw(target) and waitTime < timeout do
            if shouldRestart or not secEnabled() then break end
            task.wait(0.1); waitTime = waitTime + 0.1
        end

        stopTracking()
        if shouldRestart then continue end
        task.wait(3)
    end
end)

task.spawn(function()
    while true do
        if not atmEnabled() then task.wait(0.5); continue end

        local holdTime = STATE.atm_hold
        local waitTime = STATE.atm_wait
        local player = Players.LocalPlayer
        local char   = player.Character or Workspace:FindFirstChild(player.Name)

        local ok, err = pcall(function()
            atm_enable_noclip()
            if not char then return end
            local root = char:FindFirstChild("HumanoidRootPart")
            if not root then return end

            if not player.Team or player.Team.Name ~= "Outlaw" then
                task.wait(3); return
            end

            local bagCount = getBagCount()
            if bagCount >= getSellThreshold() then
                atm_sell_bag(); return
            end

            local target = atm_scan(root)

            if target then
                atm_last_patrol = os.clock()
                local dx, dy, dz = root.Position.X - target.pos.X, root.Position.Y - target.pos.Y, root.Position.Z - target.pos.Z
                local dist = math.sqrt(dx*dx + dy*dy + dz*dz)
                if type(dist) ~= "number" then dist = 999999 end

                if dist > 15 then
                    atm_teleport(target.pos)
                else
                    local stand_pos = target.pos
                    if target.obj and target.obj:IsA("BasePart") then
                        local s, lv = pcall(function() return target.obj.CFrame.LookVector end)
                        if s and lv then stand_pos = target.pos + (lv * 4) end
                    end

                    robbed_atms_memory[target.id] = os.clock()

                    local prompt = target.obj.Parent:FindFirstChildWhichIsA("ProximityPrompt", true) or target.obj:FindFirstChildWhichIsA("ProximityPrompt", true)
                    if prompt and fireproximityprompt then
                        fireproximityprompt(prompt, holdTime)
                    end

                    local start_rob = os.clock()
                    while os.clock() - start_rob < holdTime do
                        if not atmEnabled() then break end
                        pcall(function() root.Position = stand_pos end)
                        pcall(function() root.Velocity = Vector3.new(0, 0, 0) end)
                        atm_enable_noclip()
                        task.wait()
                    end
                    task.wait(waitTime)
                end
            else
                if os.clock() - atm_last_patrol > 4 then
                    atm_patrol_index = (atm_patrol_index % #PatrolPoints) + 1
                    atm_teleport(PatrolPoints[atm_patrol_index])
                    atm_last_patrol = os.clock()
                end
                if root.Velocity then root.Velocity = Vector3.new(0, 0, 0) end
                task.wait(0.2)
            end
        end)
        if not ok then task.wait(2) end
        task.wait()
    end
end)
