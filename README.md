local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

local oldUI = playerGui:FindFirstChild("PlayerControls")
if oldUI then
	oldUI:Destroy()
end

-- UI SIZE
local expandedSize = UDim2.new(0, 300, 0, 730)
local minimizedSize = UDim2.new(0, 300, 0, 40)

local gui = Instance.new("ScreenGui")
gui.Name = "PlayerControls"
gui.ResetOnSpawn = false
gui.Parent = playerGui

local frame = Instance.new("Frame")
frame.Size = expandedSize
frame.Position = UDim2.new(0.5, -150, 0.5, -345)
frame.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
frame.BorderSizePixel = 0
frame.Parent = gui

-- STATES
local flying = false
local freecamEnabled = false
local invisible = false
local espEnabled = false
local noclipEnabled = false
local godmodeEnabled = false
local antiKnockbackEnabled = false
local aimbotEnabled = false

local targetPlayer = nil
local targetHighlight = nil

local flyConnection
local freecamConnection
local godmodeConnection
local targetConnection
local aimbotConnection

local bodyVelocity
local bodyGyro

local flySpeed = 80
local freecamSpeed = 1

local espHighlights = {}

--------------------------------------------------
-- TITLE
--------------------------------------------------

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, -115, 0, 40)
title.Position = UDim2.new(0, 5, 0, 0)
title.Text = "Player Controls made by: Hugo"
title.TextScaled = true
title.TextColor3 = Color3.new(1, 1, 1)
title.BackgroundTransparency = 1
title.Parent = frame

local minimized = false

local minimize = Instance.new("TextButton")
minimize.Size = UDim2.new(0, 35, 0, 35)
minimize.Position = UDim2.new(1, -75, 0, 3)
minimize.Text = "−"
minimize.TextScaled = true
minimize.Parent = frame

local close = Instance.new("TextButton")
close.Size = UDim2.new(0, 35, 0, 35)
close.Position = UDim2.new(1, -40, 0, 3)
close.Text = "X"
close.TextScaled = true
close.Parent = frame

--------------------------------------------------
-- PLAYER SPEED
--------------------------------------------------

local speedBox = Instance.new("TextBox")
speedBox.Size = UDim2.new(0, 250, 0, 35)
speedBox.Position = UDim2.new(0.5, -125, 0, 45)
speedBox.PlaceholderText = "Player Speed [amount]"
speedBox.Text = ""
speedBox.TextScaled = true
speedBox.ClearTextOnFocus = false
speedBox.Parent = frame

local speedButton = Instance.new("TextButton")
speedButton.Size = UDim2.new(0, 250, 0, 35)
speedButton.Position = UDim2.new(0.5, -125, 0, 85)
speedButton.Text = "Change Player Speed"
speedButton.TextScaled = true
speedButton.Parent = frame

speedButton.MouseButton1Click:Connect(function()
	local amount = tonumber(speedBox.Text)

	if amount then
		local character = player.Character
		local humanoid = character and character:FindFirstChildOfClass("Humanoid")

		if humanoid then
			humanoid.WalkSpeed = amount
		end
	end
end)

--------------------------------------------------
-- FLY SPEED
--------------------------------------------------

local flySpeedBox = Instance.new("TextBox")
flySpeedBox.Size = UDim2.new(0, 250, 0, 35)
flySpeedBox.Position = UDim2.new(0.5, -125, 0, 125)
flySpeedBox.PlaceholderText = "Fly Speed [amount]"
flySpeedBox.Text = ""
flySpeedBox.TextScaled = true
flySpeedBox.ClearTextOnFocus = false
flySpeedBox.Parent = frame

local flySpeedButton = Instance.new("TextButton")
flySpeedButton.Size = UDim2.new(0, 250, 0, 35)
flySpeedButton.Position = UDim2.new(0.5, -125, 0, 165)
flySpeedButton.Text = "Change Fly Speed"
flySpeedButton.TextScaled = true
flySpeedButton.Parent = frame

flySpeedButton.MouseButton1Click:Connect(function()
	local amount = tonumber(flySpeedBox.Text)

	if amount and amount > 0 then
		flySpeed = amount
	end
end)

--------------------------------------------------
-- FREECAM VARIABLES
--------------------------------------------------

local freecamPosition
local freecamYaw = 0
local freecamPitch = 0

--------------------------------------------------
-- GET PLAYER CONTROLS
--------------------------------------------------

local function getControls()
	local playerScripts = player:FindFirstChild("PlayerScripts")

	if not playerScripts then
		return nil
	end

	local playerModule = playerScripts:FindFirstChild("PlayerModule")

	if not playerModule then
		return nil
	end

	local module = require(playerModule)

	return module:GetControls()
end

--------------------------------------------------
-- FREECAM BUTTON
--------------------------------------------------

local freecamButton = Instance.new("TextButton")
freecamButton.Size = UDim2.new(0, 250, 0, 35)
freecamButton.Position = UDim2.new(0.5, -125, 0, 365)
freecamButton.Text = "Freecam: OFF"
freecamButton.TextScaled = true
freecamButton.Parent = frame

--------------------------------------------------
-- STOP FREECAM
--------------------------------------------------

local function stopFreecam()
	freecamEnabled = false
	freecamButton.Text = "Freecam: OFF"

	if freecamConnection then
		freecamConnection:Disconnect()
		freecamConnection = nil
	end

	local controls = getControls()

	if controls then
		controls:Enable()
	end

	UserInputService.MouseBehavior = Enum.MouseBehavior.Default
	UserInputService.MouseIconEnabled = true

	local camera = workspace.CurrentCamera

	if camera then
		camera.CameraType = Enum.CameraType.Custom

		local character = player.Character
		local humanoid = character and character:FindFirstChildOfClass("Humanoid")

		if humanoid then
			camera.CameraSubject = humanoid
		end
	end

	player.CameraMinZoomDistance = 0.5
	player.CameraMaxZoomDistance = 128
	player.CameraMode = Enum.CameraMode.Classic
end

--------------------------------------------------
-- FLY BUTTON
--------------------------------------------------

local flyButton = Instance.new("TextButton")
flyButton.Size = UDim2.new(0, 250, 0, 35)
flyButton.Position = UDim2.new(0.5, -125, 0, 205)
flyButton.Text = "Fly: OFF"
flyButton.TextScaled = true
flyButton.Parent = frame

--------------------------------------------------
-- STOP FLY
--------------------------------------------------

local function stopFly()
	flying = false
	flyButton.Text = "Fly: OFF"

	if flyConnection then
		flyConnection:Disconnect()
		flyConnection = nil
	end

	if bodyVelocity then
		bodyVelocity:Destroy()
		bodyVelocity = nil
	end

	if bodyGyro then
		bodyGyro:Destroy()
		bodyGyro = nil
	end

	local character = player.Character
	local humanoid = character and character:FindFirstChildOfClass("Humanoid")

	if humanoid then
		humanoid.PlatformStand = false
	end
end

--------------------------------------------------
-- START FLY
--------------------------------------------------

local function startFly()
	if freecamEnabled then
		stopFreecam()
	end

	local character = player.Character
	local humanoid = character and character:FindFirstChildOfClass("Humanoid")
	local root = character and character:FindFirstChild("HumanoidRootPart")

	if not humanoid or not root then
		return
	end

	flying = true
	flyButton.Text = "Fly: ON"

	bodyVelocity = Instance.new("BodyVelocity")
	bodyVelocity.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
	bodyVelocity.Velocity = Vector3.zero
	bodyVelocity.Parent = root

	bodyGyro = Instance.new("BodyGyro")
	bodyGyro.MaxTorque = Vector3.new(math.huge, math.huge, math.huge)
	bodyGyro.P = 9000
	bodyGyro.Parent = root

	humanoid.PlatformStand = true

	flyConnection = RunService.RenderStepped:Connect(function()
		if not flying or not root.Parent then
			stopFly()
			return
		end

		local camera = workspace.CurrentCamera
		local direction = Vector3.zero

		if UserInputService:IsKeyDown(Enum.KeyCode.W) then
			direction += camera.CFrame.LookVector
		end

		if UserInputService:IsKeyDown(Enum.KeyCode.S) then
			direction -= camera.CFrame.LookVector
		end

		if UserInputService:IsKeyDown(Enum.KeyCode.A) then
			direction -= camera.CFrame.RightVector
		end

		if UserInputService:IsKeyDown(Enum.KeyCode.D) then
			direction += camera.CFrame.RightVector
		end

		if UserInputService:IsKeyDown(Enum.KeyCode.Space) then
			direction += Vector3.new(0, 1, 0)
		end

		if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then
			direction -= Vector3.new(0, 1, 0)
		end

		if direction.Magnitude > 0 then
			direction = direction.Unit * flySpeed
		end

		bodyVelocity.Velocity = direction
		bodyGyro.CFrame = camera.CFrame
	end)
end

flyButton.MouseButton1Click:Connect(function()
	if flying then
		stopFly()
	else
		startFly()
	end
end)

--------------------------------------------------
-- INVISIBLE
--------------------------------------------------

local invisButton = Instance.new("TextButton")
invisButton.Size = UDim2.new(0, 250, 0, 35)
invisButton.Position = UDim2.new(0.5, -125, 0, 245)
invisButton.Text = "Invisible: OFF"
invisButton.TextScaled = true
invisButton.Parent = frame

local function setInvisible(state)
	local character = player.Character

	if not character then
		return
	end

	for _, object in ipairs(character:GetDescendants()) do
		if object:IsA("BasePart") or object:IsA("Decal") then
			object.Transparency = state and 1 or 0
		end
	end
end

invisButton.MouseButton1Click:Connect(function()
	invisible = not invisible

	invisButton.Text =
		invisible and "Invisible: ON"
		or "Invisible: OFF"

	setInvisible(invisible)
end)

--------------------------------------------------
-- ESP
--------------------------------------------------

local espButton = Instance.new("TextButton")
espButton.Size = UDim2.new(0, 250, 0, 35)
espButton.Position = UDim2.new(0.5, -125, 0, 285)
espButton.Text = "ESP: OFF"
espButton.TextScaled = true
espButton.Parent = frame

local function removeESP(target)
	if espHighlights[target] then
		espHighlights[target]:Destroy()
		espHighlights[target] = nil
	end

	local character = target.Character

	if character then
		local head = character:FindFirstChild("Head")

		if head then
			local username = head:FindFirstChild("PlayerUsernameESP")

			if username then
				username:Destroy()
			end
		end
	end
end

local function addESP(target)
	if target == player then
		return
	end

	local character = target.Character

	if not character then
		return
	end

	removeESP(target)

	local highlight = Instance.new("Highlight")
	highlight.Name = "PlayerESP"
	highlight.FillTransparency = 0.5
	highlight.OutlineTransparency = 0
	highlight.Adornee = character
	highlight.Parent = character

	espHighlights[target] = highlight

	local head = character:FindFirstChild("Head")

	if head then
		local billboard = Instance.new("BillboardGui")
		billboard.Name = "PlayerUsernameESP"
		billboard.Size = UDim2.new(0, 200, 0, 40)
		billboard.StudsOffset = Vector3.new(0, 2.5, 0)
		billboard.AlwaysOnTop = true
		billboard.MaxDistance = 1000
		billboard.Parent = head

		local username = Instance.new("TextLabel")
		username.Size = UDim2.new(1, 0, 1, 0)
		username.BackgroundTransparency = 1
		username.Text = target.Name
		username.TextColor3 = Color3.new(1, 1, 1)
		username.TextStrokeTransparency = 0
		username.TextScaled = true
		username.Font = Enum.Font.SourceSansBold
		username.Parent = billboard
	end
end

espButton.MouseButton1Click:Connect(function()
	espEnabled = not espEnabled

	espButton.Text =
		espEnabled and "ESP: ON"
		or "ESP: OFF"

	for _, target in ipairs(Players:GetPlayers()) do
		if espEnabled then
			addESP(target)
		else
			removeESP(target)
		end
	end
end)

--------------------------------------------------
-- NOCLIP
--------------------------------------------------

local noclipButton = Instance.new("TextButton")
noclipButton.Size = UDim2.new(0, 250, 0, 35)
noclipButton.Position = UDim2.new(0.5, -125, 0, 325)
noclipButton.Text = "Noclip: OFF"
noclipButton.TextScaled = true
noclipButton.Parent = frame

local function setNoclip(state)
	local character = player.Character

	if not character then
		return
	end

	for _, object in ipairs(character:GetDescendants()) do
		if object:IsA("BasePart") then
			object.CanCollide = not state
		end
	end
end

noclipButton.MouseButton1Click:Connect(function()
	noclipEnabled = not noclipEnabled

	noclipButton.Text =
		noclipEnabled and "Noclip: ON"
		or "Noclip: OFF"

	setNoclip(noclipEnabled)
end)

RunService.Stepped:Connect(function()
	if noclipEnabled then
		setNoclip(true)
	end
end)

--------------------------------------------------
-- FREECAM
--------------------------------------------------

local function startFreecam()
	if flying then
		stopFly()
	end

	local camera = workspace.CurrentCamera

	if not camera then
		return
	end

	freecamEnabled = true
	freecamButton.Text = "Freecam: ON"

	local controls = getControls()

	if controls then
		controls:Disable()
	end

	UserInputService.MouseBehavior = Enum.MouseBehavior.LockCenter
	UserInputService.MouseIconEnabled = false

	camera.CameraType = Enum.CameraType.Scriptable

	freecamPosition = camera.CFrame.Position

	local lookVector = camera.CFrame.LookVector

	freecamYaw = math.atan2(-lookVector.X, -lookVector.Z)
	freecamPitch = math.asin(math.clamp(lookVector.Y, -1, 1))

	freecamConnection = RunService.RenderStepped:Connect(function(deltaTime)
		if not freecamEnabled then
			return
		end

		local mouseDelta = UserInputService:GetMouseDelta()

		freecamYaw -= mouseDelta.X * 0.003
		freecamPitch -= mouseDelta.Y * 0.003

		freecamPitch = math.clamp(
			freecamPitch,
			math.rad(-89),
			math.rad(89)
		)

		local rotation = CFrame.fromEulerAnglesYXZ(
			freecamPitch,
			freecamYaw,
			0
		)

		local moveDirection = Vector3.zero

		if UserInputService:IsKeyDown(Enum.KeyCode.W) then
			moveDirection += rotation.LookVector
		end

		if UserInputService:IsKeyDown(Enum.KeyCode.S) then
			moveDirection -= rotation.LookVector
		end

		if UserInputService:IsKeyDown(Enum.KeyCode.A) then
			moveDirection -= rotation.RightVector
		end

		if UserInputService:IsKeyDown(Enum.KeyCode.D) then
			moveDirection += rotation.RightVector
		end

		if UserInputService:IsKeyDown(Enum.KeyCode.Space) then
			moveDirection += Vector3.new(0, 1, 0)
		end

		if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then
			moveDirection -= Vector3.new(0, 1, 0)
		end

		if moveDirection.Magnitude > 0 then
			moveDirection = moveDirection.Unit

			freecamPosition +=
				moveDirection
				* freecamSpeed
				* deltaTime
				* 60
		end

		camera.CFrame =
			CFrame.new(freecamPosition)
			* rotation
	end)
end

freecamButton.MouseButton1Click:Connect(function()
	if freecamEnabled then
		stopFreecam()
	else
		startFreecam()
	end
end)

--------------------------------------------------
-- GODMODE
--------------------------------------------------

local godmodeButton = Instance.new("TextButton")
godmodeButton.Size = UDim2.new(0, 250, 0, 35)
godmodeButton.Position = UDim2.new(0.5, -125, 0, 405)
godmodeButton.Text = "Godmode: OFF"
godmodeButton.TextScaled = true
godmodeButton.Parent = frame

local function setGodmode(state)
	godmodeEnabled = state

	if godmodeConnection then
		godmodeConnection:Disconnect()
		godmodeConnection = nil
	end

	local character = player.Character
	local humanoid = character and character:FindFirstChildOfClass("Humanoid")

	if not humanoid then
		godmodeButton.Text =
			state and "Godmode: ON"
			or "Godmode: OFF"

		return
	end

	if state then
		humanoid.MaxHealth = math.huge
		humanoid.Health = math.huge

		godmodeConnection = humanoid.HealthChanged:Connect(function()
			if godmodeEnabled and humanoid.Parent then
				humanoid.Health = math.huge
			end
		end)

		godmodeButton.Text = "Godmode: ON"
	else
		humanoid.MaxHealth = 100
		humanoid.Health = math.min(humanoid.Health, 100)

		godmodeButton.Text = "Godmode: OFF"
	end
end

godmodeButton.MouseButton1Click:Connect(function()
	setGodmode(not godmodeEnabled)
end)

--------------------------------------------------
-- TP TO PLAYER
--------------------------------------------------

local tpBox = Instance.new("TextBox")
tpBox.Size = UDim2.new(0, 250, 0, 35)
tpBox.Position = UDim2.new(0.5, -125, 0, 445)
tpBox.PlaceholderText = "Player name"
tpBox.Text = ""
tpBox.TextScaled = true
tpBox.ClearTextOnFocus = false
tpBox.Parent = frame

local tpButton = Instance.new("TextButton")
tpButton.Size = UDim2.new(0, 250, 0, 35)
tpButton.Position = UDim2.new(0.5, -125, 0, 485)
tpButton.Text = "TP to Player"
tpButton.TextScaled = true
tpButton.Parent = frame

local function findPlayer(name)
	name = name:lower():gsub("^%s*(.-)%s*$", "%1")

	if name == "" then
		return nil
	end

	for _, target in ipairs(Players:GetPlayers()) do
		if target.Name:lower() == name
			or target.DisplayName:lower() == name then
			return target
		end
	end

	for _, target in ipairs(Players:GetPlayers()) do
		if target.Name:lower():sub(1, #name) == name
			or target.DisplayName:lower():sub(1, #name) == name then
			return target
		end
	end

	return nil
end

tpButton.MouseButton1Click:Connect(function()
	local target = findPlayer(tpBox.Text)

	if not target then
		tpButton.Text = "Player not found!"

		task.wait(1.5)

		if tpButton.Parent then
			tpButton.Text = "TP to Player"
		end

		return
	end

	local character = player.Character
	local targetCharacter = target.Character

	local root =
		character
		and character:FindFirstChild("HumanoidRootPart")

	local targetRoot =
		targetCharacter
		and targetCharacter:FindFirstChild("HumanoidRootPart")

	if not root or not targetRoot then
		tpButton.Text = "Can't teleport!"

		task.wait(1.5)

		if tpButton.Parent then
			tpButton.Text = "TP to Player"
		end

		return
	end

	root.CFrame =
		targetRoot.CFrame
		+ Vector3.new(0, 3, 0)

	tpButton.Text = "Teleported!"

	task.wait(1)

	if tpButton.Parent then
		tpButton.Text = "TP to Player"
	end
end)

--------------------------------------------------
-- ANTI KNOCKBACK
--------------------------------------------------

local antiKnockbackButton = Instance.new("TextButton")
antiKnockbackButton.Size = UDim2.new(0, 250, 0, 35)
antiKnockbackButton.Position = UDim2.new(0.5, -125, 0, 525)
antiKnockbackButton.Text = "Anti Knockback: OFF"
antiKnockbackButton.TextScaled = true
antiKnockbackButton.Parent = frame

antiKnockbackButton.MouseButton1Click:Connect(function()
	antiKnockbackEnabled = not antiKnockbackEnabled

	antiKnockbackButton.Text =
		antiKnockbackEnabled
		and "Anti Knockback: ON"
		or "Anti Knockback: OFF"
end)

RunService.Heartbeat:Connect(function()
	if not antiKnockbackEnabled then
		return
	end

	local character = player.Character
	local root = character and character:FindFirstChild("HumanoidRootPart")

	if root then
		local velocity = root.AssemblyLinearVelocity

		root.AssemblyLinearVelocity = Vector3.new(
			0,
			velocity.Y,
			0
		)

		root.AssemblyAngularVelocity = Vector3.zero
	end
end)


--------------------------------------------------
-- AIMBOT
--------------------------------------------------

local aimbotButton = Instance.new("TextButton")
aimbotButton.Size = UDim2.new(0, 250, 0, 35)
aimbotButton.Position = UDim2.new(0.5, -125, 0, 560)
aimbotButton.Text = "Aimbot: OFF"
aimbotButton.TextScaled = true
aimbotButton.Parent = frame

local function getClosestPlayer()
	local character = player.Character
	local root = character and character:FindFirstChild("HumanoidRootPart")

	if not root then
		return nil
	end

	local closestPlayer = nil
	local closestDistance = math.huge

	for _, target in ipairs(Players:GetPlayers()) do
		if target ~= player and target.Character then
			local targetHumanoid =
				target.Character:FindFirstChildOfClass("Humanoid")

			local targetRoot =
				target.Character:FindFirstChild("HumanoidRootPart")

			local targetHead =
				target.Character:FindFirstChild("Head")

			if targetHumanoid
				and targetHumanoid.Health > 0
				and targetRoot
				and targetHead then

				local distance =
					(root.Position - targetRoot.Position).Magnitude

				if distance < closestDistance then
					closestDistance = distance
					closestPlayer = target
				end
			end
		end
	end

	return closestPlayer
end

local function stopAimbot()
	aimbotEnabled = false
	aimbotButton.Text = "Aimbot: OFF"

	if aimbotConnection then
		aimbotConnection:Disconnect()
		aimbotConnection = nil
	end
end

local function startAimbot()
	if aimbotConnection then
		aimbotConnection:Disconnect()
		aimbotConnection = nil
	end

	aimbotEnabled = true
	aimbotButton.Text = "Aimbot: ON"

	aimbotConnection = RunService.RenderStepped:Connect(function()
		if not aimbotEnabled then
			return
		end

		local character = player.Character
		local humanoid =
			character and character:FindFirstChildOfClass("Humanoid")

		local camera = workspace.CurrentCamera

		if not humanoid or humanoid.Health <= 0 or not camera then
			return
		end

		local target = getClosestPlayer()

		if not target or not target.Character then
			return
		end

		local targetHead = target.Character:FindFirstChild("Head")

		if targetHead then
			camera.CFrame = CFrame.lookAt(
				camera.CFrame.Position,
				targetHead.Position
			)
		end
	end)
end

aimbotButton.MouseButton1Click:Connect(function()
	if aimbotEnabled then
		stopAimbot()
	else
		startAimbot()
	end
end)

--------------------------------------------------
-- TARGET PLAYER SCROLL LIST
--------------------------------------------------


local targetLabel = Instance.new("TextLabel")
targetLabel.Size = UDim2.new(0, 250, 0, 30)
targetLabel.Position = UDim2.new(0.5, -125, 0, 600)
targetLabel.Text = "Select Target Player"
targetLabel.TextScaled = true
targetLabel.TextColor3 = Color3.new(1, 1, 1)
targetLabel.BackgroundTransparency = 1
targetLabel.Parent = frame

local targetList = Instance.new("ScrollingFrame")
targetList.Size = UDim2.new(0, 250, 0, 90)
targetList.Position = UDim2.new(0.5, -125, 0, 635)
targetList.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
targetList.BorderSizePixel = 0
targetList.ScrollBarThickness = 6
targetList.CanvasSize = UDim2.new(0, 0, 0, 0)
targetList.Parent = frame

local targetLayout = Instance.new("UIListLayout")
targetLayout.SortOrder = Enum.SortOrder.Name
targetLayout.Padding = UDim.new(0, 2)
targetLayout.Parent = targetList

--------------------------------------------------
-- TARGET HIGHLIGHT
--------------------------------------------------

local function createTargetHighlight(target)
	if targetHighlight then
		targetHighlight:Destroy()
		targetHighlight = nil
	end

	if not target then
		return
	end

	local character = target.Character

	if not character then
		return
	end

	targetHighlight = Instance.new("Highlight")
	targetHighlight.Name = "TargetHighlight"

	targetHighlight.FillColor = Color3.fromRGB(0, 255, 0)
	targetHighlight.OutlineColor = Color3.fromRGB(0, 255, 0)

	targetHighlight.FillTransparency = 0.35
	targetHighlight.OutlineTransparency = 0

	targetHighlight.Adornee = character
	targetHighlight.Parent = character
end

--------------------------------------------------
-- STOP TARGET
--------------------------------------------------

local function stopTarget()
	targetPlayer = nil

	if targetConnection then
		targetConnection:Disconnect()
		targetConnection = nil
	end

	if targetHighlight then
		targetHighlight:Destroy()
		targetHighlight = nil
	end

	for _, button in ipairs(targetList:GetChildren()) do
		if button:IsA("TextButton") then
			button.BackgroundColor3 = Color3.fromRGB(45, 45, 45)
		end
	end
end

--------------------------------------------------
-- START TARGET
--------------------------------------------------

local function startTarget(target, selectedButton)
	if not target or target == player then
		return
	end

	-- Clicking the current target turns it off
	if targetPlayer == target then
		stopTarget()
		return
	end

	stopTarget()

	targetPlayer = target

	if selectedButton then
		selectedButton.BackgroundColor3 = Color3.fromRGB(0, 170, 0)
	end

	createTargetHighlight(target)

	targetConnection = RunService.Heartbeat:Connect(function()
		if not targetPlayer or not targetPlayer.Parent then
			stopTarget()
			return
		end

		local character = player.Character
		local targetCharacter = targetPlayer.Character

		local root =
			character
			and character:FindFirstChild("HumanoidRootPart")

		local targetRoot =
			targetCharacter
			and targetCharacter:FindFirstChild("HumanoidRootPart")

		local humanoid =
			character
			and character:FindFirstChildOfClass("Humanoid")

		if not root or not targetRoot or not humanoid then
			return
		end

		local distance =
			(targetRoot.Position - root.Position).Magnitude

		if distance > 5 then
			humanoid:MoveTo(targetRoot.Position)
		else
			humanoid:MoveTo(root.Position)
		end
	end)
end

--------------------------------------------------
-- REFRESH TARGET LIST
--------------------------------------------------

local function refreshTargetList()
	for _, object in ipairs(targetList:GetChildren()) do
		if object:IsA("TextButton") then
			object:Destroy()
		end
	end

	local players = Players:GetPlayers()

	table.sort(players, function(a, b)
		return a.Name:lower() < b.Name:lower()
	end)

	for _, target in ipairs(players) do
		if target ~= player then
			local button = Instance.new("TextButton")

			button.Size = UDim2.new(1, -8, 0, 32)
			button.BackgroundColor3 = Color3.fromRGB(45, 45, 45)
			button.BorderSizePixel = 0

			if target.DisplayName ~= target.Name then
				button.Text =
					target.DisplayName
					.. "  @"
					.. target.Name
			else
				button.Text = target.Name
			end

			button.TextColor3 = Color3.new(1, 1, 1)
			button.TextScaled = true
			button.Parent = targetList

			button.MouseButton1Click:Connect(function()
				startTarget(target, button)
			end)

			if targetPlayer == target then
				button.BackgroundColor3 = Color3.fromRGB(0, 170, 0)
			end
		end
	end

	task.wait()

	targetList.CanvasSize = UDim2.new(
		0,
		0,
		0,
		targetLayout.AbsoluteContentSize.Y + 5
	)
end

targetLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
	targetList.CanvasSize = UDim2.new(
		0,
		0,
		0,
		targetLayout.AbsoluteContentSize.Y + 5
	)
end)

--------------------------------------------------
-- PLAYER JOIN / LEAVE
--------------------------------------------------

Players.PlayerAdded:Connect(function(target)
	target.CharacterAdded:Connect(function()
		if espEnabled then
			task.wait(0.5)
			addESP(target)
		end

		if targetPlayer == target then
			task.wait(0.5)
			createTargetHighlight(target)
		end
	end)

	task.wait(0.2)
	refreshTargetList()
end)

Players.PlayerRemoving:Connect(function(target)
	removeESP(target)

	if targetPlayer == target then
		stopTarget()
	end

	task.wait()
	refreshTargetList()
end)

--------------------------------------------------
-- INITIAL TARGET LIST
--------------------------------------------------

refreshTargetList()

--------------------------------------------------
-- MINIMIZE / MAXIMIZE
--------------------------------------------------

local tweenInfo = TweenInfo.new(
	0.7,
	Enum.EasingStyle.Quad,
	Enum.EasingDirection.Out
)

minimize.MouseButton1Click:Connect(function()
	minimized = not minimized

	if minimized then
		minimize.Text = "+"

		for _, object in ipairs(frame:GetChildren()) do
			if object ~= title
				and object ~= minimize
				and object ~= close then

				object.Visible = false
			end
		end

		TweenService:Create(
			frame,
			tweenInfo,
			{
				Size = minimizedSize
			}
		):Play()
	else
		minimize.Text = "−"

		for _, object in ipairs(frame:GetChildren()) do
			object.Visible = true
		end

		TweenService:Create(
			frame,
			tweenInfo,
			{
				Size = expandedSize
			}
		):Play()
	end
end)

--------------------------------------------------
-- DRAGGING
--------------------------------------------------

local dragging = false
local dragStart
local startPosition

title.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 then
		dragging = true
		dragStart = input.Position
		startPosition = frame.Position
	end
end)

title.InputEnded:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 then
		dragging = false
	end
end)

UserInputService.InputChanged:Connect(function(input)
	if dragging
		and input.UserInputType == Enum.UserInputType.MouseMovement then

		local delta = input.Position - dragStart

		frame.Position = UDim2.new(
			startPosition.X.Scale,
			startPosition.X.Offset + delta.X,
			startPosition.Y.Scale,
			startPosition.Y.Offset + delta.Y
		)
	end
end)

--------------------------------------------------
-- CLOSE
--------------------------------------------------

close.MouseButton1Click:Connect(function()
	if flying then
		stopFly()
	end

	if freecamEnabled then
		stopFreecam()
	end

	if targetPlayer then
		stopTarget()
	end

	if aimbotConnection then
		stopAimbot()
	end

	if godmodeConnection then
		godmodeConnection:Disconnect()
		godmodeConnection = nil
	end

	for target in pairs(espHighlights) do
		removeESP(target)
	end

	gui:Destroy()
end)

--------------------------------------------------
-- RESPAWN
--------------------------------------------------

player.CharacterAdded:Connect(function()
	if flying then
		stopFly()
	end

	if freecamEnabled then
		stopFreecam()
	end

	if targetPlayer then
		stopTarget()
	end

	task.wait(0.5)

	if invisible then
		setInvisible(true)
	end

	if noclipEnabled then
		setNoclip(true)
	end

	if godmodeEnabled then
		setGodmode(true)
	end

	if espEnabled then
		for _, target in ipairs(Players:GetPlayers()) do
			if target ~= player then
				addESP(target)
			end
		end
	end
end)
