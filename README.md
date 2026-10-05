-- Lock-On híbrido
-- StarterPlayer > StarterPlayerScripts > LocalScript

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

local player = Players.LocalPlayer
local camera = workspace.CurrentCamera

local lockedTarget = nil
local enabled = false

local MAX_DISTANCE = 250
local SCREEN_RADIUS = 140
local CAMERA_SMOOTHNESS = 0.18

-- Crear botón
local gui = Instance.new("ScreenGui")
gui.Name = "LockOnGui"
gui.ResetOnSpawn = false
gui.Parent = player:WaitForChild("PlayerGui")

local button = Instance.new("TextButton")
button.Name = "LockOnButton"
button.Size = UDim2.fromOffset(120, 55)
button.Position = UDim2.new(1, -140, 0.72, 0)
button.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
button.BackgroundTransparency = 0.15
button.TextColor3 = Color3.new(1, 1, 1)
button.Text = "LOCK"
button.TextScaled = true
button.Font = Enum.Font.GothamBold
button.Parent = gui

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 12)
corner.Parent = button

local stroke = Instance.new("UIStroke")
stroke.Thickness = 2
stroke.Color = Color3.fromRGB(255, 255, 255)
stroke.Parent = button


local function getCharacter(plr)
	if not plr.Character then
		return nil
	end

	local humanoid = plr.Character:FindFirstChildOfClass("Humanoid")
	local root = plr.Character:FindFirstChild("HumanoidRootPart")

	if humanoid and humanoid.Health > 0 and root then
		return plr.Character
	end

	return nil
end


-- Busca jugadores que estén dentro de la pantalla
local function getPlayerOnScreen()
	local bestPlayer = nil
	local bestDistance = SCREEN_RADIUS

	local center = Vector2.new(
		camera.ViewportSize.X / 2,
		camera.ViewportSize.Y / 2
	)

	for _, otherPlayer in ipairs(Players:GetPlayers()) do
		if otherPlayer ~= player then
			local character = getCharacter(otherPlayer)

			if character then
				local root = character:FindFirstChild("HumanoidRootPart")

				local screenPosition, visible =
					camera:WorldToViewportPoint(root.Position)

				if visible and screenPosition.Z > 0 then
					local position2D = Vector2.new(
						screenPosition.X,
						screenPosition.Y
					)

					local distanceFromCenter =
						(position2D - center).Magnitude

					if distanceFromCenter < bestDistance then
						bestDistance = distanceFromCenter
						bestPlayer = otherPlayer
					end
				end
			end
		end
	end

	return bestPlayer
end


-- Si no hay nadie apuntado, busca el jugador más cercano
local function getNearestPlayer()
	local character = player.Character
	if not character then
		return nil
	end

	local myRoot = character:FindFirstChild("HumanoidRootPart")
	if not myRoot then
		return nil
	end

	local closest = nil
	local closestDistance = MAX_DISTANCE

	for _, otherPlayer in ipairs(Players:GetPlayers()) do
		if otherPlayer ~= player then
			local otherCharacter = getCharacter(otherPlayer)

			if otherCharacter then
				local root =
					otherCharacter:FindFirstChild("HumanoidRootPart")

				local distance =
					(root.Position - myRoot.Position).Magnitude

				if distance < closestDistance then
					closestDistance = distance
					closest = otherPlayer
				end
			end
		end
	end

	return closest
end


local function selectTarget()
	-- Primero intenta encontrar al jugador al que estás apuntando
	local target = getPlayerOnScreen()

	-- Si no hay ninguno, usa el más cercano
	if not target then
		target = getNearestPlayer()
	end

	return target
end


local function lockOn()
	if not enabled then
		return
	end

	if not lockedTarget or not getCharacter(lockedTarget) then
		lockedTarget = selectTarget()
	end

	if not lockedTarget then
		return
	end

	local character = getCharacter(lockedTarget)
	local myCharacter = player.Character

	if not character or not myCharacter then
		lockedTarget = nil
		return
	end

	local targetRoot = character:FindFirstChild("HumanoidRootPart")
	local myRoot = myCharacter:FindFirstChild("HumanoidRootPart")

	if not targetRoot or not myRoot then
		lockedTarget = nil
		return
	end

	-- Girar el personaje hacia el objetivo
	local targetPosition = targetRoot.Position
	local myPosition = myRoot.Position

	myRoot.CFrame = CFrame.lookAt(
		myPosition,
		Vector3.new(
			targetPosition.X,
			myPosition.Y,
			targetPosition.Z
		)
	)

	-- Mantener al objetivo en el centro de la cámara
	local cameraPosition = camera.CFrame.Position

	local desiredCamera =
		CFrame.lookAt(cameraPosition, targetPosition)

	camera.CFrame = camera.CFrame:Lerp(
		desiredCamera,
		CAMERA_SMOOTHNESS
	)
end


button.Activated:Connect(function()
	enabled = not enabled

	if enabled then
		button.Text = "LOCK ON"
		button.BackgroundColor3 = Color3.fromRGB(45, 150, 75)

		lockedTarget = selectTarget()
	else
		button.Text = "LOCK"
		button.BackgroundColor3 = Color3.fromRGB(35, 35, 35)

		lockedTarget = nil
	end
end)


RunService.RenderStepped:Connect(function()
	lockOn()
end)


-- Tecla opcional para PC
UserInputService.InputBegan:Connect(function(input, processed)
	if processed then
		return
	end

	if input.KeyCode == Enum.KeyCode.Q then
		button:Activate()
	end
end)
