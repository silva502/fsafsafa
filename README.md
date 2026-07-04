local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local CoreGui = game:GetService("CoreGui")
local LP = Players.LocalPlayer

local bypassAimbotToggled = false
local bypassAimbotConn = nil
local prevAutoRotate = nil
local hitCD = false
local SWING_CD = 0.35
local HIT_DIST = 8
local isMinimized = false

local BAT_SLAP_LIST = {
    "Bat", "Slap", "Iron Slap", "Gold Slap", "Diamond Slap", 
    "Emerald Slap", "Ruby Slap", "Dark Matter Slap", "Flame Slap", 
    "Nuclear Slap", "Galaxy Slap", "Glitched Slap"
}

local function findBat()
    local char = LP.Character
    if not char then return nil end
    
    for _, name in ipairs(BAT_SLAP_LIST) do
        local t = char:FindFirstChild(name)
        if t and t:IsA("Tool") then return t end
    end
    
    local bp = LP:FindFirstChildOfClass("Backpack")
    if bp then
        for _, name in ipairs(BAT_SLAP_LIST) do
            local t = bp:FindFirstChild(name)
            if t and t:IsA("Tool") then
                local hum = char:FindFirstChildOfClass("Humanoid")
                if hum then pcall(function() hum:EquipTool(t) end) end
                return t
            end
        end
    end
    
    for _, ch in ipairs(char:GetChildren()) do
        if ch:IsA("Tool") and (ch.Name:lower():find("bat") or ch.Name:lower():find("slap")) then
            return ch
        end
    end
    
    return nil
end

local function trySwing()
    if hitCD then return end
    hitCD = true
    pcall(function()
        local char = LP.Character
        if char then
            local bat = findBat()
            if bat then
                if bat.Parent ~= char then
                    local hum = char:FindFirstChildOfClass("Humanoid")
                    if hum then pcall(function() hum:EquipTool(bat) end) end
                end
                pcall(function() bat:Activate() end)
            end
        end
    end)
    task.delay(SWING_CD, function() hitCD = false end)
end

local function getClosestTarget()
    local root = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
    if not root then return nil, math.huge end
    
    local closest, minDist = nil, math.huge
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LP and plr.Character then
            local tRoot = plr.Character:FindFirstChild("HumanoidRootPart")
            local hum = plr.Character:FindFirstChildOfClass("Humanoid")
            if tRoot and hum and hum.Health > 0 then
                local dist = (tRoot.Position - root.Position).Magnitude
                if dist < minDist then 
                    minDist = dist
                    closest = tRoot
                end
            end
        end
    end
    return closest, minDist
end

local function startBypassAimbot()
    if bypassAimbotConn then bypassAimbotConn:Disconnect() end
    
    local hum = LP.Character and LP.Character:FindFirstChildOfClass("Humanoid")
    if hum then
        if prevAutoRotate == nil then prevAutoRotate = hum.AutoRotate end
        hum.AutoRotate = false
    end
    
    bypassAimbotConn = RunService.RenderStepped:Connect(function()
        if not bypassAimbotToggled then return end
        
        local char = LP.Character
        if not char then return end
        
        local root = char:FindFirstChild("HumanoidRootPart")
        if not root then return end
        
        local hum = char:FindFirstChildOfClass("Humanoid")
        if not hum then return end
        
        if not char:FindFirstChildOfClass("Tool") then
            local bat = findBat()
            if bat then pcall(function() hum:EquipTool(bat) end) end
        end
        
        local target, targetDist = getClosestTarget()
        if not target then return end
        
        local myPos = root.Position
        local targetPos = target.Position
        
        local direction = targetPos - myPos
        local flatDir = Vector3.new(direction.X, 0, direction.Z)
        if flatDir.Magnitude > 0 then flatDir = flatDir.Unit else flatDir = Vector3.zero end
        
        local chaseSpeed = 58
        local desiredHeight = targetPos.Y + 3.7
        local yVel = (desiredHeight - myPos.Y) * 19.5
        if hum.FloorMaterial ~= Enum.Material.Air then yVel = math.max(yVel, 13) end
        yVel = math.clamp(yVel, -70, 110)
        
        local desiredVel = Vector3.new(flatDir.X * chaseSpeed, yVel, flatDir.Z * chaseSpeed)
        root.AssemblyLinearVelocity = root.AssemblyLinearVelocity:Lerp(desiredVel, 0.8)
        
        local toTarget = targetPos - myPos
        if toTarget.Magnitude > 0.1 then
            local goalCF = CFrame.lookAt(myPos, targetPos)
            local diffCF = root.CFrame:Inverse() * goalCF
            local rx, ry, rz = diffCF:ToEulerAnglesXYZ()
            rx = math.clamp(rx, -2.5, 2.5)
            ry = math.clamp(ry, -2.5, 2.5)
            rz = math.clamp(rz, -2.5, 2.5)
            root.AssemblyAngularVelocity = root.CFrame:VectorToWorldSpace(Vector3.new(rx * 42, ry * 42, rz * 42))
        end
        
        if targetDist <= HIT_DIST then trySwing() end
    end)
end

local function stopBypassAimbot()
    if bypassAimbotConn then
        bypassAimbotConn:Disconnect()
        bypassAimbotConn = nil
    end
    
    local c = LP.Character
    local root = c and c:FindFirstChild("HumanoidRootPart")
    if root then
        root.AssemblyLinearVelocity = Vector3.zero
        root.AssemblyAngularVelocity = Vector3.zero
    end
    
    local hum = c and c:FindFirstChildOfClass("Humanoid")
    if hum then
        hum.AutoRotate = (prevAutoRotate == nil) and true or prevAutoRotate
        hum.PlatformStand = false
        pcall(function() hum:ChangeState(Enum.HumanoidStateType.GettingUp) end)
    end
    
    hitCD = false
end

local function toggleBypassAimbot()
    bypassAimbotToggled = not bypassAimbotToggled
    if bypassAimbotToggled then
        startBypassAimbot()
    else
        stopBypassAimbot()
    end
    return bypassAimbotToggled
end

for _, v in ipairs(CoreGui:GetChildren()) do
    if v.Name == "BypassAimbot" then v:Destroy() end
end

local gui = Instance.new("ScreenGui")
gui.Name = "BypassAimbot"
gui.Parent = CoreGui

local main = Instance.new("Frame")
main.Name = "Main"
main.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
main.BackgroundTransparency = 0.15
main.BorderSizePixel = 0
main.Size = UDim2.new(0, 160, 0, 60)
main.Position = UDim2.new(0.5, -80, 0.5, -30)
main.Parent = gui
main.ClipsDescendants = true
main.Active = true
Instance.new("UICorner", main).CornerRadius = UDim.new(0, 10)

local stroke = Instance.new("UIStroke", main)
stroke.Color = Color3.fromRGB(255, 0, 0)
stroke.Thickness = 2
stroke.Transparency = 0.3

local header = Instance.new("Frame", main)
header.BackgroundTransparency = 1
header.Size = UDim2.new(1, 0, 0, 30)

local title = Instance.new("TextLabel", header)
title.BackgroundTransparency = 1
title.Position = UDim2.new(0, 10, 0, 5)
title.Size = UDim2.new(1, -40, 0, 20)
title.Font = Enum.Font.GothamBlack
title.Text = "BYPASS AIMBOT"
title.TextColor3 = Color3.fromRGB(255, 255, 255)
title.TextSize = 12
title.TextXAlignment = Enum.TextXAlignment.Left

local status = Instance.new("TextLabel", header)
status.Size = UDim2.new(0, 35, 0, 20)
status.Position = UDim2.new(1, -45, 0, 5)
status.BackgroundTransparency = 1
status.Text = "OFF"
status.TextColor3 = Color3.fromRGB(200, 0, 0)
status.Font = Enum.Font.GothamBold
status.TextSize = 11
status.TextXAlignment = Enum.TextXAlignment.Right

local sw = Instance.new("TextButton", main)
sw.Size = UDim2.new(1, -10, 0, 22)
sw.Position = UDim2.new(0, 5, 1, -28)
sw.BackgroundColor3 = Color3.fromRGB(15, 0, 0)
sw.BorderSizePixel = 0
sw.Text = ""
sw.ZIndex = 2
Instance.new("UICorner", sw).CornerRadius = UDim.new(1, 0)
local swStroke = Instance.new("UIStroke", sw)
swStroke.Color = Color3.fromRGB(255, 0, 0)
swStroke.Thickness = 1

local dot = Instance.new("Frame", sw)
dot.Size = UDim2.new(0, 16, 0, 16)
dot.Position = UDim2.new(0, 2, 0.5, -8)
dot.BackgroundColor3 = Color3.fromRGB(50, 0, 0)
dot.ZIndex = 2
Instance.new("UICorner", dot).CornerRadius = UDim.new(1, 0)

local dragging = false
local dragStart, startPos

main.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = main.Position
        local conn
        conn = input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
                conn:Disconnect()
            end
        end)
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
        local delta = input.Position - dragStart
        main.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
end)

local function updateUI()
    local pos = bypassAimbotToggled and 42 or 2
    TweenService:Create(dot, TweenInfo.new(0.2), {
        Position = UDim2.new(0, pos, 0.5, -8),
        BackgroundColor3 = bypassAimbotToggled and Color3.fromRGB(255, 0, 0) or Color3.fromRGB(50, 0, 0)
    }):Play()
    status.Text = bypassAimbotToggled and "ON" or "OFF"
    status.TextColor3 = bypassAimbotToggled and Color3.fromRGB(255, 0, 0) or Color3.fromRGB(200, 0, 0)
end

sw.MouseButton1Click:Connect(function()
    toggleBypassAimbot()
    updateUI()
end)

UserInputService.InputBegan:Connect(function(input, gp)
    if gp then return end
    if input.KeyCode == Enum.KeyCode.X then
        toggleBypassAimbot()
        updateUI()
    end
end)

LP.CharacterAdded:Connect(function()
    if bypassAimbotToggled then
        task.wait(0.5)
        startBypassAimbot()
    end
end)

updateUI()
