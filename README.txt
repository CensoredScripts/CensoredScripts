-- Sky Teleport Script with draggable GUI and auto-set target position at spawn

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")

local lp = Players.LocalPlayer
local char = lp.Character or lp.CharacterAdded:Wait()
local hrp = char:WaitForChild("HumanoidRootPart")

-- Save target position immediately at script load (spawn)
local targetPosition = hrp.Position

-- GUI Setup
local gui = Instance.new("ScreenGui")
gui.Name = "SkyTeleportGUI"
gui.ResetOnSpawn = false
gui.Parent = lp:WaitForChild("PlayerGui")

local mainCard = Instance.new("Frame")
mainCard.Size = UDim2.new(0, 240, 0, 220)
mainCard.Position = UDim2.new(0.5, 0, 0.5, 0)
mainCard.AnchorPoint = Vector2.new(0.5, 0.5)
mainCard.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
mainCard.BorderSizePixel = 0
mainCard.Active = true    -- for dragging on mobile
mainCard.Selectable = true -- for dragging on mobile
mainCard.Parent = gui

local UICorner = Instance.new("UICorner", mainCard)
UICorner.CornerRadius = UDim.new(0, 12)

local UIStroke = Instance.new("UIStroke", mainCard)
UIStroke.Color = Color3.fromRGB(70, 70, 70)
UIStroke.Thickness = 2

-- Dragging logic
local dragging, dragInput, dragStart, startPos

mainCard.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		dragging = true
		dragStart = input.Position
		startPos = mainCard.Position
		input.Changed:Connect(function()
			if input.UserInputState == Enum.UserInputState.End then
				dragging = false
			end
		end)
	end
end)

mainCard.InputChanged:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
		dragInput = input
	end
end)

UserInputService.InputChanged:Connect(function(input)
	if dragging and input == dragInput then
		local delta = input.Position - dragStart
		mainCard.Position = UDim2.new(
			startPos.X.Scale,
			startPos.X.Offset + delta.X,
			startPos.Y.Scale,
			startPos.Y.Offset + delta.Y
		)
	end
end)

-- Title
local title = Instance.new("TextLabel", mainCard)
title.Size = UDim2.new(1, 0, 0, 35)
title.Position = UDim2.new(0, 0, 0, 0)
title.BackgroundTransparency = 1
title.Text = "🚀 Sky Teleport"
title.Font = Enum.Font.GothamBold
title.TextSize = 22
title.TextColor3 = Color3.fromRGB(180, 240, 255)

-- Variables
local skyPart = nil
local hitBlock = nil
local movingToTarget = false
local movementConnection = nil
local speed = 100

-- Speed Label
local speedLabel = Instance.new("TextLabel", mainCard)
speedLabel.Size = UDim2.new(0.9, 0, 0, 20)
speedLabel.Position = UDim2.new(0.05, 0, 0, 45)
speedLabel.BackgroundTransparency = 1
speedLabel.Text = "Speed: 100"
speedLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
speedLabel.Font = Enum.Font.Gotham
speedLabel.TextSize = 14

-- Speed Slider
local sliderFrame = Instance.new("Frame", mainCard)
sliderFrame.Size = UDim2.new(0.9, 0, 0, 20)
sliderFrame.Position = UDim2.new(0.05, 0, 0, 70)
sliderFrame.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
sliderFrame.BorderSizePixel = 0
Instance.new("UICorner", sliderFrame).CornerRadius = UDim.new(0, 6)

local sliderFill = Instance.new("Frame", sliderFrame)
sliderFill.Size = UDim2.new(speed/500, 0, 1, 0)
sliderFill.BackgroundColor3 = Color3.fromRGB(0, 170, 255)
sliderFill.BorderSizePixel = 0
Instance.new("UICorner", sliderFill).CornerRadius = UDim.new(0, 6)

local sliderBtn = Instance.new("ImageButton", sliderFrame)
sliderBtn.Size = UDim2.new(0, 18, 0, 18)
sliderBtn.Position = UDim2.new(speed/500 - 0.018, 0, 0.5, -9)
sliderBtn.BackgroundColor3 = Color3.fromRGB(0, 120, 255)
sliderBtn.BorderSizePixel = 0
sliderBtn.Image = ""
Instance.new("UICorner", sliderBtn).CornerRadius = UDim.new(0, 9)

local draggingSlider = false

sliderBtn.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		draggingSlider = true
		input.Changed:Connect(function()
			if input.UserInputState == Enum.UserInputState.End then
				draggingSlider = false
			end
		end)
	end
end)

UserInputService.InputChanged:Connect(function(input)
	if draggingSlider and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
		local x = input.Position.X - sliderFrame.AbsolutePosition.X
		local scale = math.clamp(x / sliderFrame.AbsoluteSize.X, 0, 1)
		speed = math.floor(scale * 500)
		speedLabel.Text = "Speed: " .. speed
		sliderFill.Size = UDim2.new(scale, 0, 1, 0)
		sliderBtn.Position = UDim2.new(scale - 0.018, 0, 0.5, -9)
	end
end)

-- Create Sky Platform and Detector
local function createSkyBase()
	if skyPart then skyPart:Destroy() end
	if hitBlock then hitBlock:Destroy() end

	local skyY = math.max(hrp.Position.Y + 20, 100)
	skyPart = Instance.new("Part")
	skyPart.Size = Vector3.new(500, 5, 500)
	skyPart.Position = Vector3.new(hrp.Position.X, skyY, hrp.Position.Z)
	skyPart.Anchored = true
	skyPart.Transparency = 0.85
	skyPart.BrickColor = BrickColor.new("Lime green")
	skyPart.CanCollide = true
	skyPart.Name = "SkyBase"
	skyPart.Parent = workspace

	hitBlock = Instance.new("Part")
	hitBlock.Size = Vector3.new(5, 5, 5)
	hitBlock.Position = skyPart.Position + Vector3.new(0, 3, 0)
	hitBlock.Anchored = true
	hitBlock.CanCollide = false
	hitBlock.Transparency = 1
	hitBlock.Name = "SkyDetector"
	hitBlock.Parent = workspace

	hitBlock.Touched:Connect(function(hit)
		if hit:IsDescendantOf(char) then
			if skyPart then skyPart:Destroy() end
			if hitBlock then hitBlock:Destroy() end
			skyPart = nil
			hitBlock = nil
			movingToTarget = false
			if movementConnection then
				movementConnection:Disconnect()
				movementConnection = nil
			end
		end
	end)
end

-- Clean fall
local function fallDown()
	if movementConnection then movementConnection:Disconnect() movementConnection = nil end
	hrp.AssemblyLinearVelocity = Vector3.new(0, -150, 0)
end

-- Stop all movement & cleanup
local function stopMovement()
	if skyPart then skyPart:Destroy() end
	if hitBlock then hitBlock:Destroy() end
	skyPart = nil
	hitBlock = nil
	movingToTarget = false
	if movementConnection then
		movementConnection:Disconnect()
		movementConnection = nil
	end
	fallDown()
end

-- Tween to SkyBase
local function tweenToPlatform()
	if skyPart then
		local cf = CFrame.new(skyPart.Position + Vector3.new(0, 5, 0))
		local tween = TweenService:Create(hrp, TweenInfo.new(0.6, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {CFrame = cf})
		tween:Play()
		tween.Completed:Wait()
	end
end

-- Move to Target Pos smoothly
local function moveToTarget()
	if not targetPosition then return end
	movingToTarget = true
	local startY = hrp.Position.Y
	movementConnection = RunService.Heartbeat:Connect(function()
		if not movingToTarget then return end
		local currentPos = hrp.Position
		local flatTarget = Vector3.new(targetPosition.X, startY, targetPosition.Z)
		local dir = (flatTarget - currentPos)
		if dir.Magnitude < 3 then
			hrp.CFrame = CFrame.new(flatTarget)
			stopMovement()
			return
		end
		local vel = dir.Unit * speed
		hrp.AssemblyLinearVelocity = vel
	end)
end

-- Teleport Button
local tpBtn = Instance.new("TextButton", mainCard)
tpBtn.Size = UDim2.new(0.9, 0, 0, 40)
tpBtn.Position = UDim2.new(0.05, 0, 0, 110)
tpBtn.Text = "🛸 Teleport to SkyBase"
tpBtn.TextSize = 18
tpBtn.Font = Enum.Font.GothamBold
tpBtn.BackgroundColor3 = Color3.fromRGB(40, 120, 255)
tpBtn.TextColor3 = Color3.new(1, 1, 1)
Instance.new("UICorner", tpBtn).CornerRadius = UDim.new(0, 8)

tpBtn.MouseButton1Click:Connect(function()
	if not targetPosition then return end
	createSkyBase()
	task.wait(0.1)
	tweenToPlatform()
	moveToTarget()
end)

-- Stop Button
local stopBtn = Instance.new("TextButton", mainCard)
stopBtn.Size = UDim2.new(0.9, 0, 0, 40)
stopBtn.Position = UDim2.new(0.05, 0, 0, 160)
stopBtn.Text = "⏹ Stop"
stopBtn.TextSize = 18
stopBtn.Font = Enum.Font.GothamBold
stopBtn.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
stopBtn.TextColor3 = Color3.new(1, 1, 1)
Instance.new("UICorner", stopBtn).CornerRadius = UDim.new(0, 8)

stopBtn.MouseButton1Click:Connect(stopMovement)
