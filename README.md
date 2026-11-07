--[[
	TacticalMapClient
	Free top-down camera movement.
	Island numbers are hidden.
	Selecting a target keeps the map open.
	Right or middle mouse drag to pan.
	WASD or arrow keys to move.
	Shift to move faster.
	Mouse wheel or zoom buttons to zoom.
--]]

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local LocalPlayer = Players.LocalPlayer
local playerGui = LocalPlayer:WaitForChild("PlayerGui")

while not workspace.CurrentCamera do
	workspace:GetPropertyChangedSignal("CurrentCamera"):Wait()
end

local camera = workspace.CurrentCamera
local mapRemote = nil

local ISLAND_Y = 38
local ZOOM_PRESETS = { 900, 480, 200 }
local ZOOM_LABELS = { "X1", "X5", "X10" }

local MIN_HEIGHT = 150
local MAX_HEIGHT = 1200
local DRAG_THRESHOLD = 6

local COLOR_PANEL = Color3.fromRGB(20, 28, 42)
local COLOR_ACCENT = Color3.fromRGB(0, 170, 255)
local COLOR_ISLAND = Color3.fromRGB(70, 130, 200)
local COLOR_SELF = Color3.fromRGB(60, 220, 120)
local COLOR_OTHER = Color3.fromRGB(235, 80, 80)
local COLOR_SHIELD = Color3.fromRGB(80, 180, 255)
local COLOR_BASE = Color3.fromRGB(255, 200, 60)
local COLOR_TARGET = Color3.fromRGB(255, 70, 70)

local mapOpen = false
local transitioning = false
local dragging = false
local dragDist = 0

local camHeight = ZOOM_PRESETS[1]
local camCenter = Vector2.zero
local mapCenterDefault = Vector2.zero

local savedCamType = Enum.CameraType.Custom
local savedFieldOfView = 70
local savedCamCFrame = camera.CFrame

local selectedTargetName = nil
local remoteTargets = {}
local markers = {}
local entitySignature = ""

local cameraTween = nil
local transitionVersion = 0
local toastVersion = 0
local toastTween = nil

local openMap
local closeMap
local refreshMarkers
local setMapHeight

local controls = nil
local controlsDisabledByMap = false

task.spawn(function()
	local playerScripts = LocalPlayer:WaitForChild("PlayerScripts")
	local playerModule = playerScripts:WaitForChild("PlayerModule", 10)

	if not playerModule then
		return
	end

	local success, module = pcall(require, playerModule)

	if success and module then
		controls = module:GetControls()

		if mapOpen then
			controls:Disable()
			controlsDisabledByMap = true
		end
	end
end)

local function setMapControls(active)
	if not controls then
		return
	end

	if active then
		controls:Disable()
		controlsDisabledByMap = true
	elseif controlsDisabledByMap then
		controls:Enable()
		controlsDisabledByMap = false
	end
end

local function new(className, props, children)
	local instance = Instance.new(className)

	for key, value in pairs(props or {}) do
		instance[key] = value
	end

	for _, child in ipairs(children or {}) do
		child.Parent = instance
	end

	return instance
end

local function makeCorner(parent, radius)
	return new("UICorner", {
		CornerRadius = UDim.new(0, radius),
		Parent = parent,
	})
end

local function makeStroke(parent, color, thickness, transparency)
	return new("UIStroke", {
		Color = color or Color3.new(1, 1, 1),
		Thickness = thickness or 2,
		Transparency = transparency or 0,
		Parent = parent,
	})
end

----------------------------------------------------------------
-- ISLANDS
----------------------------------------------------------------

local FALLBACK_ISLANDS = {
	{ 1, Vector3.new(-400, 38, -400) },
	{ 2, Vector3.new(21, 38, 586) },
	{ 3, Vector3.new(425, 38, -400) },
	{ 4, Vector3.new(16, 38, -569) },
	{ 5, Vector3.new(425, 38, 425) },
	{ 6, Vector3.new(-390, 38, 420) },
	{ 7, Vector3.new(-677, 38, 11) },
	{ 8, Vector3.new(714, 38, 6) },
}

local function discoverIslands()
	local sums = {}

	for _, object in ipairs(workspace:GetDescendants()) do
		if object:IsA("BasePart") then
			local matchedId = object.Name:match("^Island(%d+)_Border")

			if matchedId then
				local id = tonumber(matchedId)

				if not sums[id] then
					sums[id] = {
						sum = Vector3.zero,
						count = 0,
					}
				end

				sums[id].sum += object.Position
				sums[id].count += 1
			end
		end
	end

	local found = {}

	for id, entry in pairs(sums) do
		found[id] = {
			name = "Island " .. id,
			position = entry.sum / entry.count,
		}
	end

	if next(found) == nil then
		for _, data in ipairs(FALLBACK_ISLANDS) do
			found[data[1]] = {
				name = "Island " .. data[1],
				position = data[2],
			}
		end
	end

	return found
end

local islands = discoverIslands()

do
	local minX, maxX = math.huge, -math.huge
	local minZ, maxZ = math.huge, -math.huge

	for _, island in pairs(islands) do
		local position = island.position

		minX = math.min(minX, position.X)
		maxX = math.max(maxX, position.X)
		minZ = math.min(minZ, position.Z)
		maxZ = math.max(maxZ, position.Z)
	end

	mapCenterDefault = Vector2.new(
		(minX + maxX) / 2,
		(minZ + maxZ) / 2
	)

	camCenter = mapCenterDefault
end

----------------------------------------------------------------
-- UI
----------------------------------------------------------------

local mapGui = new("ScreenGui", {
	Name = "TacticalMapGui",
	ResetOnSpawn = false,
	IgnoreGuiInset = true,
	DisplayOrder = 5,
	Enabled = false,
	ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
	Parent = playerGui,
})

local buttonGui = new("ScreenGui", {
	Name = "TacticalMapButtonGui",
	ResetOnSpawn = false,
	DisplayOrder = 10,
	Parent = playerGui,
})

local toggleBtn = new("TextButton", {
	Name = "MapToggleButton",
	Text = "MAP",
	Font = Enum.Font.GothamBold,
	TextSize = 16,
	TextColor3 = Color3.new(1, 1, 1),
	BackgroundColor3 = COLOR_PANEL,
	Size = UDim2.fromOffset(96, 40),
	Position = UDim2.new(1, -18, 1, -18),
	AnchorPoint = Vector2.new(1, 1),
	AutoButtonColor = true,
	Parent = buttonGui,
})

makeCorner(toggleBtn, 10)
makeStroke(toggleBtn, COLOR_ACCENT, 2, 0.4)

local overlay = new("Frame", {
	Name = "Overlay",
	BackgroundTransparency = 1,
	Size = UDim2.fromScale(1, 1),
	Parent = mapGui,
})

local markerLayer = new("Frame", {
	Name = "MarkerLayer",
	BackgroundTransparency = 1,
	Size = UDim2.fromScale(1, 1),
	ZIndex = 1,
	Parent = overlay,
})

local topBar = new("Frame", {
	Name = "TopBar",
	BackgroundColor3 = COLOR_PANEL,
	BackgroundTransparency = 0.15,
	Size = UDim2.new(1, 0, 0, 74),
	ZIndex = 5,
	Parent = overlay,
})

makeStroke(topBar, COLOR_ACCENT, 1, 0.6)

new("TextLabel", {
	Name = "Title",
	Text = "TACTICAL STRATEGY MAP",
	Font = Enum.Font.GothamBold,
	TextSize = 18,
	TextColor3 = COLOR_ACCENT,
	BackgroundTransparency = 1,
	Size = UDim2.new(1, -90, 0, 34),
	Position = UDim2.fromOffset(16, 4),
	TextXAlignment = Enum.TextXAlignment.Left,
	ZIndex = 6,
	Parent = topBar,
})

local statusLabel = new("TextLabel", {
	Name = "Status",
	Text = "Right-drag / WASD: move | Scroll: zoom | Click island: target",
	Font = Enum.Font.Gotham,
	TextSize = 13,
	TextColor3 = Color3.fromRGB(170, 185, 205),
	BackgroundTransparency = 1,
	Size = UDim2.new(1, -90, 0, 30),
	Position = UDim2.fromOffset(16, 36),
	TextXAlignment = Enum.TextXAlignment.Left,
	TextWrapped = true,
	ZIndex = 6,
	Parent = topBar,
})

local closeBtn = new("TextButton", {
	Name = "CloseButton",
	Text = "X",
	Font = Enum.Font.GothamBold,
	TextSize = 18,
	TextColor3 = Color3.new(1, 1, 1),
	BackgroundColor3 = Color3.fromRGB(170, 45, 45),
	Size = UDim2.fromOffset(38, 38),
	Position = UDim2.new(1, -16, 0.5, 0),
	AnchorPoint = Vector2.new(1, 0.5),
	ZIndex = 6,
	Parent = topBar,
})

makeCorner(closeBtn, 8)

local zoomPanel = new("Frame", {
	Name = "ZoomPanel",
	BackgroundColor3 = COLOR_PANEL,
	BackgroundTransparency = 0.15,
	Size = UDim2.fromOffset(64, 44 + 46 * #ZOOM_PRESETS),
	Position = UDim2.new(1, -16, 0.5, 0),
	AnchorPoint = Vector2.new(1, 0.5),
	ZIndex = 5,
	Parent = overlay,
})

makeCorner(zoomPanel, 10)
makeStroke(zoomPanel, COLOR_ACCENT, 1, 0.6)

new("TextLabel", {
	Name = "ZoomTitle",
	Text = "ZOOM",
	Font = Enum.Font.GothamBold,
	TextSize = 12,
	TextColor3 = COLOR_ACCENT,
	BackgroundTransparency = 1,
	Size = UDim2.new(1, 0, 0, 26),
	ZIndex = 6,
	Parent = zoomPanel,
})

local zoomButtons = {}

for index, height in ipairs(ZOOM_PRESETS) do
	local button = new("TextButton", {
		Name = ZOOM_LABELS[index],
		Text = ZOOM_LABELS[index],
		Font = Enum.Font.GothamBold,
		TextSize = 14,
		TextColor3 = Color3.new(1, 1, 1),
		BackgroundColor3 = Color3.fromRGB(30, 42, 62),
		Size = UDim2.fromOffset(48, 36),
		Position = UDim2.new(0.5, 0, 0, 34 + (index - 1) * 46),
		AnchorPoint = Vector2.new(0.5, 0),
		ZIndex = 6,
		Parent = zoomPanel,
	})

	makeCorner(button, 8)

	zoomButtons[index] = {
		button = button,
		height = height,
	}

	button.Activated:Connect(function()
		setMapHeight(height)
	end)
end

local toast = new("TextLabel", {
	Name = "Toast",
	Text = "",
	Font = Enum.Font.GothamBold,
	TextSize = 16,
	TextColor3 = Color3.new(1, 1, 1),
	BackgroundColor3 = COLOR_TARGET,
	BackgroundTransparency = 0.1,
	Size = UDim2.new(0.7, 0, 0, 44),
	Position = UDim2.new(0.5, 0, 0, 90),
	AnchorPoint = Vector2.new(0.5, 0),
	Visible = false,
	ZIndex = 7,
	Parent = overlay,
})

makeCorner(toast, 10)

----------------------------------------------------------------
-- MARKERS
----------------------------------------------------------------

local function getHrpPosition(character)
	local root = character and character:FindFirstChild("HumanoidRootPart")

	if root then
		return root.Position
	end

	return nil
end

local function makeMarker(config)
	local button = new("TextButton", {
		Name = config.name,
		Text = "",
		BackgroundColor3 = config.color,
		BackgroundTransparency = config.transparency or 0,
		Size = UDim2.fromOffset(config.size, config.size),
		AnchorPoint = Vector2.new(0.5, 0.5),
		AutoButtonColor = false,
		Visible = false,
		ZIndex = 2,
		Parent = markerLayer,
	})

	new("UICorner", {
		CornerRadius = config.square
			and UDim.new(0, math.floor(config.size * 0.18))
			or UDim.new(1, 0),
		Parent = button,
	})

	makeStroke(
		button,
		config.strokeColor or Color3.new(1, 1, 1),
		config.strokeThickness or 2,
		0.2
	)

	if config.label then
		new("TextLabel", {
			Name = "Label",
			Text = config.label,
			Font = Enum.Font.Gotham,
			TextSize = 12,
			TextColor3 = Color3.fromRGB(225, 235, 245),
			BackgroundTransparency = 1,
			Size = UDim2.fromOffset(160, 18),
			Position = UDim2.new(0.5, 0, 1, 4),
			AnchorPoint = Vector2.new(0.5, 0),
			TextStrokeTransparency = 0.4,
			ZIndex = 3,
			Parent = button,
		})
	end

	if config.onClick then
		button.Activated:Connect(function()
			if mapOpen and not transitioning
				and dragDist < DRAG_THRESHOLD then
				config.onClick()
			end
		end)
	end

	return {
		gui = button,
		posFn = config.posFn,
	}
end

local function selectIslandTarget(islandName)
	selectedTargetName = islandName
	statusLabel.Text = "TARGET SET: " .. islandName

	toastVersion += 1
	local version = toastVersion

	if toastTween then
		toastTween:Cancel()
		toastTween = nil
	end

	toast.Text = "TARGET SET: " .. string.upper(islandName)
	toast.Visible = true
	toast.BackgroundTransparency = 0.1

	task.delay(1.6, function()
		if version ~= toastVersion then
			return
		end

		local fade = TweenService:Create(
			toast,
			TweenInfo.new(0.4),
			{ BackgroundTransparency = 1 }
		)

		toastTween = fade

		fade.Completed:Once(function()
			if version == toastVersion then
				toast.Visible = false
				toastTween = nil
			end
		end)

		fade:Play()
	end)

	if mapRemote and mapRemote.Parent then
		mapRemote:FireServer(islandName)
	end

	entitySignature = ""
	refreshMarkers()
end

refreshMarkers = function()
	local entities = {}

	for id, island in pairs(islands) do
		local isTarget = island.name == selectedTargetName
		local remoteCount = 0

		for playerName, targetName in pairs(remoteTargets) do
			if playerName ~= LocalPlayer.Name
				and targetName == island.name then
				remoteCount += 1
			end
		end

		local label = island.name

		if isTarget then
			label ..= " [TARGET]"
		elseif remoteCount > 0 then
			label ..= " [" .. remoteCount .. " LOCKED]"
		end

		local highlighted = isTarget or remoteCount > 0

		table.insert(entities, {
			key = "island:" .. id,
			config = {
				name = island.name,
				size = 30,
				color = highlighted and COLOR_TARGET or COLOR_ISLAND,
				label = label,
				strokeColor = highlighted
					and COLOR_TARGET or Color3.new(1, 1, 1),
				strokeThickness = highlighted and 3 or 2,
				posFn = function()
					return island.position
				end,
				onClick = function()
					selectIslandTarget(island.name)
				end,
			},
		})
	end

	for _, player in ipairs(Players:GetPlayers()) do
		if player.Character then
			local isSelf = player == LocalPlayer

			table.insert(entities, {
				key = "player:" .. player.Name,
				config = {
					name = "Player_" .. player.Name,
					size = 16,
					color = isSelf and COLOR_SELF or COLOR_OTHER,
					label = player.DisplayName,
					posFn = function()
						return getHrpPosition(player.Character)
					end,
				},
			})
		end
	end

	for _, object in ipairs(workspace:GetDescendants()) do
		if object:IsA("BasePart") then
			local lowerName = string.lower(object.Name)
			local isShield = lowerName:find("shield") ~= nil
			local isBase = lowerName:find("base") ~= nil
				and lowerName ~= "baseplate"

			if isShield or isBase then
				table.insert(entities, {
					key = (isShield and "shield:" or "base:")
						.. object:GetFullName(),
					config = {
						name = object.Name,
						size = isShield and 20 or 22,
						color = isShield and COLOR_SHIELD or COLOR_BASE,
						transparency = isShield and 0.35 or 0,
						label = isShield and "SHIELD" or "BASE",
						square = not isShield,
						posFn = function()
							if object:IsDescendantOf(workspace) then
								return object.Position
							end

							return nil
						end,
					},
				})
			end
		end
	end

	table.sort(entities, function(a, b)
		return a.key < b.key
	end)

	local signatureParts = {}

	for _, entity in ipairs(entities) do
		table.insert(
			signatureParts,
			entity.key .. ":" .. (entity.config.label or "")
		)
	end

	local signature = table.concat(signatureParts, "|")

	if signature == entitySignature then
		return
	end

	entitySignature = signature
	markerLayer:ClearAllChildren()
	markers = {}

	for _, entity in ipairs(entities) do
		table.insert(markers, makeMarker(entity.config))
	end
end

----------------------------------------------------------------
-- CAMERA
----------------------------------------------------------------

local function computeMapCFrame()
	local position = Vector3.new(
		camCenter.X,
		ISLAND_Y + camHeight,
		camCenter.Y
	)

	return CFrame.new(position)
		* CFrame.Angles(-math.pi / 2, 0, 0)
end

setMapHeight = function(height)
	camHeight = math.clamp(height, MIN_HEIGHT, MAX_HEIGHT)

	for _, entry in ipairs(zoomButtons) do
		local active = math.abs(entry.height - camHeight) < 1

		entry.button.BackgroundColor3 = active
			and COLOR_ACCENT
			or Color3.fromRGB(30, 42, 62)

		entry.button.TextColor3 = active
			and Color3.new(1, 1, 1)
			or Color3.fromRGB(200, 210, 225)
	end
end

local function updateMarkerPositions()
	for _, marker in ipairs(markers) do
		local position = marker.posFn and marker.posFn()

		if position then
			local screenPoint, onScreen =
				camera:WorldToViewportPoint(position)

			marker.gui.Visible = onScreen

			if onScreen then
				marker.gui.Position = UDim2.fromOffset(
					screenPoint.X,
					screenPoint.Y
				)
			end
		else
			marker.gui.Visible = false
		end
	end
end

local function cancelCameraTween()
	transitionVersion += 1

	if cameraTween then
		cameraTween:Cancel()
		cameraTween = nil
	end
end

openMap = function()
	if mapOpen or transitioning then
		return
	end

	camera = workspace.CurrentCamera

	if not camera then
		return
	end

	cancelCameraTween()

	savedCamType = camera.CameraType
	savedFieldOfView = camera.FieldOfView
	savedCamCFrame = camera.CFrame

	mapOpen = true
	transitioning = true
	dragging = false
	dragDist = 0

	setMapControls(true)
	setMapHeight(ZOOM_PRESETS[1])

	camCenter = mapCenterDefault
	mapGui.Enabled = true

	entitySignature = ""
	refreshMarkers()

	toggleBtn.Text = "CLOSE"
	toggleBtn.BackgroundColor3 = Color3.fromRGB(150, 40, 40)

	camera.CameraType = Enum.CameraType.Scriptable
	camera.FieldOfView = 50

	local version = transitionVersion

	cameraTween = TweenService:Create(
		camera,
		TweenInfo.new(
			0.9,
			Enum.EasingStyle.Quint,
			Enum.EasingDirection.Out
		),
		{ CFrame = computeMapCFrame() }
	)

	cameraTween.Completed:Once(function(state)
		if version ~= transitionVersion then
			return
		end

		if state == Enum.PlaybackState.Completed then
			transitioning = false
			cameraTween = nil
		end
	end)

	cameraTween:Play()
end

closeMap = function()
	if not mapOpen then
		return
	end

	cancelCameraTween()

	mapOpen = false
	transitioning = true
	dragging = false

	toastVersion += 1

	if toastTween then
		toastTween:Cancel()
		toastTween = nil
	end

	toast.Visible = false
	mapGui.Enabled = false

	toggleBtn.Text = "MAP"
	toggleBtn.BackgroundColor3 = COLOR_PANEL

	local version = transitionVersion

	cameraTween = TweenService:Create(
		camera,
		TweenInfo.new(
			0.7,
			Enum.EasingStyle.Quint,
			Enum.EasingDirection.Out
		),
		{
			CFrame = savedCamCFrame,
			FieldOfView = savedFieldOfView,
		}
	)

	cameraTween.Completed:Once(function(state)
		if version ~= transitionVersion then
			return
		end

		if state == Enum.PlaybackState.Completed then
			camera.CameraType = savedCamType
			transitioning = false
			cameraTween = nil

			setMapControls(false)

			markerLayer:ClearAllChildren()
			markers = {}
			entitySignature = ""
		end
	end)

	cameraTween:Play()
end

toggleBtn.Activated:Connect(function()
	if mapOpen then
		closeMap()
	elseif not transitioning then
		openMap()
	end
end)

closeBtn.Activated:Connect(function()
	closeMap()
end)

----------------------------------------------------------------
-- INPUT
----------------------------------------------------------------

local function isPanInput(input)
	return input.UserInputType == Enum.UserInputType.MouseButton2
		or input.UserInputType == Enum.UserInputType.MouseButton3
end

UserInputService.InputBegan:Connect(function(input, gameProcessed)
	if not mapOpen or transitioning then
		return
	end

	if input.UserInputType == Enum.UserInputType.MouseButton1 then
		dragDist = 0

		if not gameProcessed then
			dragging = true
		end
	elseif isPanInput(input) then
		dragDist = 0
		dragging = true
	end
end)

UserInputService.InputChanged:Connect(function(input)
	if not mapOpen or transitioning then
		return
	end

	if input.UserInputType == Enum.UserInputType.MouseMovement
		and dragging then

		local delta = input.Delta
		dragDist += math.abs(delta.X) + math.abs(delta.Y)

		local viewportHeight = math.max(camera.ViewportSize.Y, 1)

		local studsPerPixel =
			(2 * math.tan(math.rad(camera.FieldOfView * 0.5))
				* camHeight) / viewportHeight

		camCenter -= Vector2.new(delta.X, delta.Y) * studsPerPixel

	elseif input.UserInputType == Enum.UserInputType.MouseWheel then
		setMapHeight(camHeight * (1 - input.Position.Z * 0.15))
	end
end)

UserInputService.InputEnded:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1
		or isPanInput(input) then
		dragging = false
	end
end)

UserInputService.WindowFocusReleased:Connect(function()
	dragging = false
end)

local function updateKeyboardMovement(dt)
	if UserInputService:GetFocusedTextBox() then
		return
	end

	local movement = Vector2.zero

	if UserInputService:IsKeyDown(Enum.KeyCode.W)
		or UserInputService:IsKeyDown(Enum.KeyCode.Up) then
		movement += Vector2.new(0, -1)
	end

	if UserInputService:IsKeyDown(Enum.KeyCode.S)
		or UserInputService:IsKeyDown(Enum.KeyCode.Down) then
		movement += Vector2.new(0, 1)
	end

	if UserInputService:IsKeyDown(Enum.KeyCode.A)
		or UserInputService:IsKeyDown(Enum.KeyCode.Left) then
		movement += Vector2.new(-1, 0)
	end

	if UserInputService:IsKeyDown(Enum.KeyCode.D)
		or UserInputService:IsKeyDown(Enum.KeyCode.Right) then
		movement += Vector2.new(1, 0)
	end

	if movement.Magnitude == 0 then
		return
	end

	local speed = camHeight * 0.8

	if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift)
		or UserInputService:IsKeyDown(Enum.KeyCode.RightShift) then
		speed *= 2
	end

	camCenter += movement.Unit * speed * dt
end

----------------------------------------------------------------
-- LIVE UPDATES
----------------------------------------------------------------

RunService.RenderStepped:Connect(function(dt)
	if mapOpen then
		if not transitioning then
			updateKeyboardMovement(dt)
			camera.CFrame = computeMapCFrame()
		end

		updateMarkerPositions()
	end
end)

task.spawn(function()
	while mapGui.Parent do
		if mapOpen and not transitioning then
			refreshMarkers()
		end

		task.wait(1)
	end
end)

LocalPlayer.CharacterAdded:Connect(function()
	if not mapOpen and not transitioning then
		return
	end

	cancelCameraTween()

	mapOpen = false
	transitioning = false
	dragging = false

	toastVersion += 1

	if toastTween then
		toastTween:Cancel()
		toastTween = nil
	end

	toast.Visible = false
	mapGui.Enabled = false

	markerLayer:ClearAllChildren()
	markers = {}
	entitySignature = ""

	toggleBtn.Text = "MAP"
	toggleBtn.BackgroundColor3 = COLOR_PANEL

	camera = workspace.CurrentCamera

	if camera then
		camera.FieldOfView = savedFieldOfView
		camera.CameraType = Enum.CameraType.Custom
	end

	setMapControls(false)
end)

Players.PlayerRemoving:Connect(function(player)
	remoteTargets[player.Name] = nil
	entitySignature = ""
end)

----------------------------------------------------------------
-- SERVER CONNECTION
----------------------------------------------------------------

local remoteConnection = nil

local function connectRemote(remote)
	if not remote:IsA("RemoteEvent") then
		return
	end

	if remoteConnection then
		remoteConnection:Disconnect()
	end

	mapRemote = remote

	remoteConnection = remote.OnClientEvent:Connect(function(
		playerName,
		targetName
	)
		if typeof(playerName) ~= "string" then
			return
		end

		if targetName ~= nil and typeof(targetName) ~= "string" then
			return
		end

		remoteTargets[playerName] = targetName
		entitySignature = ""

		if mapOpen then
			refreshMarkers()
		end
	end)

	if selectedTargetName then
		remote:FireServer(selectedTargetName)
	end
end

ReplicatedStorage.ChildAdded:Connect(function(child)
	if child.Name == "MapTargetSelected" then
		connectRemote(child)
	end
end)

local existingRemote =
	ReplicatedStorage:FindFirstChild("MapTargetSelected")

if existingRemote then
	connectRemote(existingRemote)
end
