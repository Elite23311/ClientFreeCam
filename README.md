local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local StarterGui = game:GetService("StarterGui")
local Camera = workspace.CurrentCamera
local Players = game:GetService("Players")
local player = Players.LocalPlayer

local freecamEnabled = false
local camSpeed = 50

local moveForward, moveBackward, moveLeft, moveRight, moveUp, moveDown = false, false, false, false, false, false

local yaw, pitch = 0, 0
local targetYaw, targetPitch = 0, 0
local sensitivity = 0.35

local humanoid = nil
local oldWalkSpeed, oldJumpPower

local camTargetCFrame = nil
local defaultFOV = 70
local targetFOV = defaultFOV

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
end

local function enableGui()
	StarterGui:SetCoreGuiEnabled(Enum.CoreGuiType.All, true)
	StarterGui:SetCore("ResetButtonCallback", true)
end

UserInputService.InputBegan:Connect(function(input, gameProcessed)
	if gameProcessed then return end

	if input.KeyCode == Enum.KeyCode.F then
		freecamEnabled = not freecamEnabled
		if freecamEnabled then
			camTargetCFrame = Camera.CFrame
			freezePlayer()
			disableGui()
			UserInputService.MouseBehavior = Enum.MouseBehavior.LockCenter
			UserInputService.MouseIconEnabled = false
			local look = camTargetCFrame.LookVector
			yaw = math.atan2(-look.X, -look.Z)
			pitch = math.asin(look.Y)
			targetYaw, targetPitch = yaw, pitch
			targetFOV = defaultFOV
		else
			unfreezePlayer()
			enableGui()
			UserInputService.MouseBehavior = Enum.MouseBehavior.Default
			UserInputService.MouseIconEnabled = true
			Camera.CameraType = Enum.CameraType.Custom
			Camera.FieldOfView = defaultFOV
			camTargetCFrame = nil
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
	if freecamEnabled then
		if input.UserInputType == Enum.UserInputType.MouseMovement then
			local delta = input.Delta
			targetYaw = targetYaw - delta.X * sensitivity * 0.0025
			targetPitch = math.clamp(targetPitch - delta.Y * sensitivity * 0.0025, -math.pi/2, math.pi/2)
		elseif input.UserInputType == Enum.UserInputType.MouseWheel then
			targetFOV = math.clamp(targetFOV - input.Position.Z * 5, 20, 120)
		end
	end
end)

RunService.RenderStepped:Connect(function(dt)
	if freecamEnabled and camTargetCFrame then
		yaw = yaw + (targetYaw - yaw) * 0.1
		pitch = pitch + (targetPitch - pitch) * 0.1

		local moveVec = Vector3.new()
		if moveForward then moveVec = moveVec + Vector3.new(0, 0, -1) end
		if moveBackward then moveVec = moveVec + Vector3.new(0, 0, 1) end
		if moveLeft then moveVec = moveVec + Vector3.new(-1, 0, 0) end
		if moveRight then moveVec = moveVec + Vector3.new(1, 0, 0) end
		if moveUp then moveVec = moveVec + Vector3.new(0, 1, 0) end
		if moveDown then moveVec = moveVec + Vector3.new(0, -1, 0) end

		if moveVec.Magnitude > 0 then
			moveVec = moveVec.Unit * camSpeed * dt
			local yawCFrame = CFrame.Angles(0, yaw, 0)
			local worldMove = yawCFrame:VectorToWorldSpace(moveVec)
			camTargetCFrame = camTargetCFrame + worldMove
		end

		local yawCFrame = CFrame.Angles(0, yaw, 0)
		local pitchCFrame = CFrame.Angles(pitch, 0, 0)
		local targetCFrame = CFrame.new(camTargetCFrame.Position) * yawCFrame * pitchCFrame

		Camera.CFrame = Camera.CFrame:Lerp(targetCFrame, 0.1)
		Camera.CameraType = Enum.CameraType.Scriptable

		Camera.FieldOfView = Camera.FieldOfView + (targetFOV - Camera.FieldOfView) * 0.1
	else
		if Camera.CameraType == Enum.CameraType.Scriptable then
			Camera.CameraType = Enum.CameraType.Custom
			Camera.FieldOfView = defaultFOV
		end
	end
end)
