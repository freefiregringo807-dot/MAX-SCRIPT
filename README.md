-- LocalScript (colocar em StarterPlayer > StarterPlayerScripts)
-- Fly com suporte para joystick/touch (mobile) + teclado
-- Autor: Assistente (exemplo educativo). Use apenas em jogos onde você tem permissão.

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local StarterGui = game:GetService("StarterGui")
local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")
local Camera = workspace.CurrentCamera

-- === CONFIGURAÇÃO ===
local FLY_SPEED = 80         -- velocidade base
local ACCEL = 6              -- aceleração suave
local JOYSTICK_SIZE = 120    -- tamanho externo do joystick (px)
local JOYSTICK_DEADZONE = 0.12 -- deadzone (0..1)
-- ======================

local flyEnabled = false
local flying = false
local moveVector = Vector3.new(0,0,0) -- x = right, z = forward
local vertical = 0 -- -1 descend, 0 none, 1 ascend
local targetVelocity = Vector3.new(0,0,0)
local currentVelocity = Vector3.new(0,0,0)

-- UI Creation (simple, no dependencies)
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "FlyJoystickUI"
screenGui.ResetOnSpawn = false
screenGui.Parent = PlayerGui

-- Toggle button
local toggleBtn = Instance.new("TextButton")
toggleBtn.Name = "FlyToggle"
toggleBtn.Size = UDim2.new(0,100,0,40)
toggleBtn.Position = UDim2.new(1,-110,0,10)
toggleBtn.AnchorPoint = Vector2.new(0,0)
toggleBtn.Text = "Fly: OFF"
toggleBtn.Parent = screenGui
toggleBtn.BackgroundTransparency = 0.35
toggleBtn.TextScaled = true

-- Up/Down buttons for mobile + keyboard fallback
local upBtn = Instance.new("TextButton")
upBtn.Name = "UpBtn"
upBtn.Size = UDim2.new(0,60,0,60)
upBtn.Position = UDim2.new(1,-80,1,-140)
upBtn.AnchorPoint = Vector2.new(0,0)
upBtn.Text = "↑"
upBtn.Parent = screenGui
upBtn.BackgroundTransparency = 0.5
upBtn.TextScaled = true

local downBtn = Instance.new("TextButton")
downBtn.Name = "DownBtn"
downBtn.Size = UDim2.new(0,60,0,60)
downBtn.Position = UDim2.new(1,-80,1,-70)
downBtn.AnchorPoint = Vector2.new(0,0)
downBtn.Text = "↓"
downBtn.Parent = screenGui
downBtn.BackgroundTransparency = 0.5
downBtn.TextScaled = true

-- Joystick (only shown on touch devices)
local joystickOuter, joystickInner
local joystickVector = Vector2.new(0,0) -- normalized (-1..1)
local touchId = nil

local function createJoystick()
    joystickOuter = Instance.new("Frame")
    joystickOuter.Name = "JoystickOuter"
    joystickOuter.Size = UDim2.new(0, JOYSTICK_SIZE, 0, JOYSTICK_SIZE)
    joystickOuter.Position = UDim2.new(0, 20, 1, -20 - JOYSTICK_SIZE)
    joystickOuter.AnchorPoint = Vector2.new(0,1)
    joystickOuter.BackgroundTransparency = 0.6
    joystickOuter.BorderSizePixel = 0
    joystickOuter.Parent = screenGui
    joystickOuter.ClipsDescendants = true
    joystickOuter.Visible = true

    joystickInner = Instance.new("Frame")
    joystickInner.Name = "JoystickInner"
    local innerSize = math.floor(JOYSTICK_SIZE*0.5)
    joystickInner.Size = UDim2.new(0, innerSize, 0, innerSize)
    joystickInner.Position = UDim2.new(0.5, -innerSize/2, 0.5, -innerSize/2)
    joystickInner.AnchorPoint = Vector2.new(0,0)
    joystickInner.BackgroundTransparency = 0.3
    joystickInner.BorderSizePixel = 0
    joystickInner.Parent = joystickOuter
end

if UserInputService.TouchEnabled then
    createJoystick()
end

-- Helpers
local function getCharacter()
    return LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
end

local function enableFly(enable)
    flyEnabled = enable
    toggleBtn.Text = "Fly: " .. (enable and "ON" or "OFF")
    local char = getCharacter()
    local hrp = char:FindFirstChild("HumanoidRootPart")
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    if enable then
        if humanoid then
            humanoid.PlatformStand = true
        end
        flying = true
    else
        if humanoid then
            humanoid.PlatformStand = false
        end
        flying = false
        -- reset velocities
        currentVelocity = Vector3.new(0,0,0)
        targetVelocity = Vector3.new(0,0,0)
    end
end

toggleBtn.MouseButton1Click:Connect(function()
    enableFly(not flyEnabled)
end)

-- Up/Down touch
upBtn.MouseButton1Down:Connect(function() vertical = 1 end)
upBtn.MouseButton1Up:Connect(function() if vertical==1 then vertical = 0 end end)
downBtn.MouseButton1Down:Connect(function() vertical = -1 end)
downBtn.MouseButton1Up:Connect(function() if vertical==-1 then vertical = 0 end end)

-- Keyboard/Game input
UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.UserInputType == Enum.UserInputType.Keyboard then
        if input.KeyCode == Enum.KeyCode.Space then vertical = 1 end
        if input.KeyCode == Enum.KeyCode.LeftShift or input.KeyCode == Enum.KeyCode.LeftControl then vertical = -1 end
        if input.KeyCode == Enum.KeyCode.F then -- tecla F para toggle
            enableFly(not flyEnabled)
        end
    end
end)
UserInputService.InputEnded:Connect(function(input, processed)
    if input.UserInputType == Enum.UserInputType.Keyboard then
        if input.KeyCode == Enum.KeyCode.Space and vertical==1 then vertical = 0 end
        if (input.KeyCode == Enum.KeyCode.LeftShift or input.KeyCode == Enum.KeyCode.LeftControl) and vertical==-1 then vertical = 0 end
    end
end)

-- Touch joystick handling
if joystickOuter then
    joystickOuter.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Touch then
            touchId = input.UserInputState
            -- capture this touch via input object
            input.Changed:Connect(function() end)
            -- start tracking this touch id in InputChanged
        end
    end)
    joystickOuter.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Touch then
            -- nothing here; we'll handle global InputChanged to read touch movement
        end
    end)
    UserInputService.TouchMoved:Connect(function(touch)
        -- only consider touches that started inside joystick area
        -- easier approach: if touch started near joystick center use it
        local pos = Vector2.new(touch.Position.X, touch.Position.Y)
        local outerPos = joystickOuter.AbsolutePosition + Vector2.new(joystickOuter.AbsoluteSize.X/2, joystickOuter.AbsoluteSize.Y/2)
        local delta = pos - outerPos
        local maxRadius = joystickOuter.AbsoluteSize.X/2
        local nv = delta / maxRadius
        if nv.Magnitude > 1 then nv = nv.Unit end
        joystickVector = Vector2.new(nv.X, -nv.Y) -- invert Y so forward is up on screen
        -- move inner handle visually
        if joystickInner then
            local innerOffset = Vector2.new(nv.X * (maxRadius*0.45), nv.Y * (maxRadius*0.45))
            joystickInner.Position = UDim2.new(0.5, innerOffset.X - joystickInner.AbsoluteSize.X/2, 0.5, innerOffset.Y - joystickInner.AbsoluteSize.Y/2)
        end
    end)
    -- When touch ends, reset joystick
    UserInputService.TouchEnded:Connect(function(touch)
        -- reset
        joystickVector = Vector2.new(0,0)
        if joystickInner then
            local innerSize = joystickInner.AbsoluteSize
            joystickInner.Position = UDim2.new(0.5, -innerSize.X/2, 0.5, -innerSize.Y/2)
        end
    end)
end

-- Mouse/keyboard movement (WASD/arrows)
local kbMove = Vector2.new(0,0)
UserInputService.InputChanged:Connect(function(input, processed)
    if input.UserInputType == Enum.UserInputType.MouseMovement then
        -- ignore
    elseif input.UserInputType == Enum.UserInputType.Gamepad1 then
        -- Gamepad thumbstick handling (if device sends it)
        if input.KeyCode == Enum.KeyCode.Thumbstick1 or input.KeyCode == Enum.KeyCode.Thumbstick2 then
            local v = input.Position -- Vector2
            -- Thumbstick Y is typically up/down; map to our kbMove
            kbMove = Vector2.new(v.X, v.Y)
        end
    elseif input.UserInputType == Enum.UserInputType.Touch then
        -- handled separately
    end
end)

-- Also monitor keyboard directional keys using GetKeysDown
local keysDown = {}
UserInputService.InputBegan:Connect(function(input, processed)
    if input.UserInputType == Enum.UserInputType.Keyboard then
        keysDown[input.KeyCode] = true
    end
end)
UserInputService.InputEnded:Connect(function(input, processed)
    if input.UserInputType == Enum.UserInputType.Keyboard then
        keysDown[input.KeyCode] = nil
    end
end)

local function getKBVector()
    local mv = Vector2.new(0,0)
    if keysDown[Enum.KeyCode.W] or keysDown[Enum.KeyCode.Up] then mv = mv + Vector2.new(0,1) end
    if keysDown[Enum.KeyCode.S] or keysDown[Enum.KeyCode.Down] then mv = mv + Vector2.new(0,-1) end
    if keysDown[Enum.KeyCode.A] or keysDown[Enum.KeyCode.Left] then mv = mv + Vector2.new(-1,0) end
    if keysDown[Enum.KeyCode.D] or keysDown[Enum.KeyCode.Right] then mv = mv + Vector2.new(1,0) end
    if mv.Magnitude > 0 then mv = mv.Unit end
    return mv
end

-- Main fly loop
RunService.RenderStepped:Connect(function(dt)
    if not flyEnabled then return end
    local char = LocalPlayer.Character
    if not char then return end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    if not hrp or not humanoid then return end

    -- compute moveVector: joystick (mobile) > gamepad thumb > keyboard
    local inputVec = Vector2.new(0,0)
    if joystickVector and joystickVector.Magnitude > JOYSTICK_DEADZONE then
        inputVec = joystickVector
    elseif kbMove and kbMove.Magnitude > JOYSTICK_DEADZONE then
        inputVec = kbMove
    else
        local k = getKBVector()
        if k.Magnitude > 0 then
            -- invert Y so up is forward
            inputVec = Vector2.new(k.X, k.Y)
        end
    end

    -- camera relative
    local camCFrame = Camera.CFrame
    local forward = Vector3.new(camCFrame.LookVector.X, 0, camCFrame.LookVector.Z).Unit
    local right = Vector3.new(camCFrame.RightVector.X, 0, camCFrame.RightVector.Z).Unit

    local horizontalVel = (right * inputVec.X + forward * inputVec.Y) * FLY_SPEED
    local verticalVel = Vector3.new(0, vertical * FLY_SPEED, 0)

    targetVelocity = horizontalVel + verticalVel

    -- smooth velocity
    currentVelocity = currentVelocity:Lerp(targetVelocity, math.clamp(ACCEL * dt, 0, 1))

    -- apply velocity directly
    -- Use AssemblyLinearVelocity to avoid physics warping; it's safer on client side for LocalScripts.
    hrp.AssemblyLinearVelocity = currentVelocity

    -- keep character upright / prevent falling animation weirdness
    humanoid.PlatformStand = true
end)

-- Cleanup on character respawn / leaving
LocalPlayer.CharacterAdded:Connect(function(char)
    wait(0.2)
    if not flyEnabled then return end
    local humanoid = char:FindFirstChildOfClass("Humanoid") or char:WaitForChild("Humanoid",5)
    if humanoid then humanoid.PlatformStand = true end
end)

-- Safety hint: when script disabled, ensure to reset PlatformStand after small delay
RunService.Heartbeat:Connect(function()
    if not flyEnabled then
        local char = LocalPlayer.Character
        if char then
            local humanoid = char:FindFirstChildOfClass("Humanoid")
            if humanoid and humanoid.PlatformStand then
                -- gentle reset
                humanoid.PlatformStand = false
            end
        end
    end
end)

-- Finished
print("[FlyJoystick] script loaded. Use button or 'F' to toggle fly.")
