local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local StarterGui = game:GetService("StarterGui")
local Players = game:GetService("Players")
local player = Players.LocalPlayer
local Camera = workspace.CurrentCamera

local freecamEnabled = false
local camSpeed = 50

local moveForward, moveBackward, moveLeft, moveRight, moveUp, moveDown = false, false, false, false, false, false

local yaw, pitch = 0, 0
local targetYaw, targetPitch = 0, 0
local sensitivity = 0.65

local humanoid = nil
local oldWalkSpeed, oldJumpPower

local camPosition = nil
local defaultFOV = 70
local targetFOV = defaultFOV

local guiStates = {}

-- One-time start notification
do
    local playerGui = player:WaitForChild("PlayerGui")

    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "FreeCamStartNotification"
    screenGui.ResetOnSpawn = false
    screenGui.Parent = playerGui

    local textLabel = Instance.new("TextLabel")
    textLabel.Size = UDim2.new(0, 400, 0, 60)
    textLabel.Position = UDim2.new(0.5, -200, 0.1, 0)
    textLabel.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
    textLabel.BackgroundTransparency = 0.3
    textLabel.TextColor3 = Color3.new(1, 1, 1)
    textLabel.Font = Enum.Font.GothamBold
    textLabel.TextSize = 24
    textLabel.Text = "Hello " .. player.Name .. "! Press F To Enable Free Cam!"
    textLabel.Parent = screenGui
    textLabel.TextStrokeTransparency = 0.7

    task.delay(5, function()
        for i = 1, 20 do
            textLabel.BackgroundTransparency = textLabel.BackgroundTransparency + 0.05
            textLabel.TextTransparency = textLabel.TextTransparency + 0.05
            textLabel.TextStrokeTransparency = textLabel.TextStrokeTransparency + 0.05
            task.wait(0.05)
        end
        screenGui:Destroy()
    end)
end

local function getHumanoid()
    local char = player.Character
    if char then
        return char:FindFirstChildOfClass("Humanoid")
    end
    return nil
end

local function freezePlayer()
    humanoid = getHumanoid()
    if humanoid then
        oldWalkSpeed = humanoid.WalkSpeed
        oldJumpPower = humanoid.JumpPower
        humanoid.WalkSpeed = 0
        humanoid.JumpPower = 0
        humanoid:SetStateEnabled(Enum.HumanoidStateType.Jumping, false)
    end
end

local function unfreezePlayer()
    if humanoid then
        humanoid.WalkSpeed = oldWalkSpeed or 16
        humanoid.JumpPower = oldJumpPower or 50
        humanoid:SetStateEnabled(Enum.HumanoidStateType.Jumping, true)
    end
end

local function disableGui()
    StarterGui:SetCoreGuiEnabled(Enum.CoreGuiType.All, false)
    StarterGui:SetCore("ResetButtonCallback", false)
    local playerGui = player:FindFirstChildOfClass("PlayerGui")
    if playerGui then
        guiStates = {}
        for _, gui in pairs(playerGui:GetChildren()) do
            if gui:IsA("ScreenGui") then
                guiStates[gui] = gui.Enabled
                gui.Enabled = false
            end
        end
    end
end

local function enableGui()
    StarterGui:SetCoreGuiEnabled(Enum.CoreGuiType.All, true)
    StarterGui:SetCore("ResetButtonCallback", true)
    for gui, wasEnabled in pairs(guiStates) do
        if gui and gui.Parent then
            gui.Enabled = wasEnabled
        end
    end
    guiStates = {}
end

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end

    if input.KeyCode == Enum.KeyCode.F then
        freecamEnabled = not freecamEnabled
        if freecamEnabled then
            camPosition = Camera.CFrame.Position
            freezePlayer()
            disableGui()
            UserInputService.MouseBehavior = Enum.MouseBehavior.LockCenter
            UserInputService.MouseIconEnabled = false

            local lookVector = Camera.CFrame.LookVector
            targetYaw = math.atan2(lookVector.X, lookVector.Z)
            targetPitch = math.asin(lookVector.Y)
            yaw = targetYaw
            pitch = targetPitch
            targetFOV = Camera.FieldOfView or defaultFOV
        else
            unfreezePlayer()
            enableGui()
            UserInputService.MouseBehavior = Enum.MouseBehavior.Default
            UserInputService.MouseIconEnabled = true
            Camera.CameraType = Enum.CameraType.Custom
            Camera.FieldOfView = defaultFOV
            camPosition = nil
            moveForward, moveBackward, moveLeft, moveRight, moveUp, moveDown = false, false, false, false, false, false
        end
    end

    if freecamEnabled and input.UserInputType == Enum.UserInputType.Keyboard then
        if input.KeyCode == Enum.KeyCode.W then moveForward = true end
        if input.KeyCode == Enum.KeyCode.S then moveBackward = true end
        if input.KeyCode == Enum.KeyCode.A then moveLeft = true end
        if input.KeyCode == Enum.KeyCode.D then moveRight = true end
        if input.KeyCode == Enum.KeyCode.E then moveUp = true end
        if input.KeyCode == Enum.KeyCode.Q then moveDown = true end
    end
end)

UserInputService.InputEnded:Connect(function(input, gameProcessed)
    if gameProcessed then return end

    if freecamEnabled and input.UserInputType == Enum.UserInputType.Keyboard then
        if input.KeyCode == Enum.KeyCode.W then moveForward = false end
        if input.KeyCode == Enum.KeyCode.S then moveBackward = false end
        if input.KeyCode == Enum.KeyCode.A then moveLeft = false end
        if input.KeyCode == Enum.KeyCode.D then moveRight = false end
        if input.KeyCode == Enum.KeyCode.E then moveUp = false end
        if input.KeyCode == Enum.KeyCode.Q then moveDown = false end
    end
end)

UserInputService.InputChanged:Connect(function(input, gameProcessed)
    if freecamEnabled and input.UserInputType == Enum.UserInputType.MouseMovement then
        local delta = input.Delta
        targetYaw = targetYaw - delta.X * sensitivity * 0.0025
        targetPitch = math.clamp(targetPitch - delta.Y * sensitivity * 0.0025, -math.pi/2 + 0.01, math.pi/2 - 0.01)
    elseif freecamEnabled and input.UserInputType == Enum.UserInputType.MouseWheel then
        targetFOV = math.clamp(targetFOV - input.Position.Z * 5, 20, 120)
    end
end)

RunService.RenderStepped:Connect(function(dt)
    if freecamEnabled and camPosition then
        yaw = yaw + (targetYaw - yaw) * 0.15
        pitch = pitch + (targetPitch - pitch) * 0.15

        local yawRotation = CFrame.Angles(0, yaw, 0)
        local pitchRotation = CFrame.Angles(pitch, 0, 0)
        local camRotation = yawRotation * pitchRotation

        local lookVector = camRotation.LookVector
        local rightVector = camRotation.RightVector
        local upVector = Vector3.new(0,1,0)

        local moveVec = Vector3.new()
        if moveForward then moveVec = moveVec + lookVector end
        if moveBackward then moveVec = moveVec - lookVector end
        if moveLeft then moveVec = moveVec - rightVector end
        if moveRight then moveVec = moveVec + rightVector end
        if moveUp then moveVec = moveVec + upVector end
        if moveDown then moveVec = moveVec - upVector end

        if moveVec.Magnitude > 0 then
            moveVec = moveVec.Unit * camSpeed * dt
            camPosition = camPosition + moveVec
        end

        local targetCFrame = CFrame.new(camPosition) * camRotation

        Camera.CFrame = Camera.CFrame:Lerp(targetCFrame, 0.15)
        Camera.FieldOfView = Camera.FieldOfView + (targetFOV - Camera.FieldOfView) * 0.15
        Camera.CameraType = Enum.CameraType.Scriptable
    else
        if Camera.CameraType == Enum.CameraType.Scriptable then
            Camera.CameraType = Enum.CameraType.Custom
            Camera.FieldOfView = defaultFOV
        end
    end
end)
