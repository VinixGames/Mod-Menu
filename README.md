
		-- ================== [ LAYANAN & VARIABEL UTAMA ] ==================
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local PhysicsService = game:GetService("PhysicsService")
local CoreGui = game:GetService("CoreGui")
local HttpService = game:GetService("HttpService")
local TweenService = game:GetService("TweenService")
local ContextActionService = game:GetService("ContextActionService")

local player = Players.LocalPlayer

-- ================== [ SISTEM KEYBINDING KUSTOM ] ==================
local customKeybinds = {}          -- Tabel untuk menyimpan keybinds, e.g., { [Enum.KeyCode.F] = "Fly" }
local featureToggleFunctions = {}  -- Tabel untuk menyimpan fungsi toggle setiap fitur, e.g., { ["Fly"] = function() ... end }
local featureCtmButtons = {}       -- Tabel untuk menyimpan referensi tombol [CTM]

local isListeningForKeybind = false
local featureToBind = nil

-- ================== [ FLAGS & PENGATURAN AWAL ] ==================
local flags = {
	SpeedMove = false,
	Fly = false,
	Swim = false,
	Vehicle = false,
}

local DEFAULTS = {
	SpeedMove = 16,
	Fly = 50,
	Vehicle = 50,
	Swim = 50,
}

-- ================== [ FUNGSI UTILITAS: GESER ] ==================
local function makeDraggable(trigger, target)
	local dragging = false
	local dragInput, dragStart, startPos
	trigger.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
			dragging, dragStart, startPos = true, input.Position, target.Position
			input.Changed:Connect(function()
				if input.UserInputState == Enum.UserInputState.End then dragging = false end
			end)
		end
	end)
	trigger.InputChanged:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then dragInput = input end
	end)
	UserInputService.InputChanged:Connect(function(input)
		if input == dragInput and dragging then
			target.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + (input.Position.X - dragStart.X), startPos.Y.Scale, startPos.Y.Offset + (input.Position.Y - dragStart.Y))
		end
	end)
end

-- ================== [ SETUP GUI UTAMA ] ==================
local gui = Instance.new("ScreenGui", player:WaitForChild("PlayerGui"))
gui.Name = "ARVnnxzyz_Menu_"..math.random(1000, 9999)
gui.ResetOnSpawn = false
gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

local logoToggle = Instance.new("TextButton", gui)
logoToggle.Size, logoToggle.Position = UDim2.new(0, 50, 0, 50), UDim2.new(0, 20, 0, 20)
logoToggle.BackgroundColor3, logoToggle.Text = Color3.fromRGB(15, 15, 15), "AR"
logoToggle.Font, logoToggle.TextColor3, logoToggle.TextScaled, logoToggle.ZIndex = Enum.Font.SourceSansBold, Color3.fromRGB(255, 255, 255), true, 3
local logoCorner = Instance.new("UICorner", logoToggle); logoCorner.CornerRadius = UDim.new(0.5, 0)
local logoStroke = Instance.new("UIStroke", logoToggle); logoStroke.Thickness, logoStroke.ApplyStrokeMode = 2, Enum.ApplyStrokeMode.Border

local menu = Instance.new("ScrollingFrame", gui)
menu.Visible, menu.Active = false, true
menu.Size, menu.Position = UDim2.new(0, 260, 0, 350), UDim2.new(0, 20, 0, 80)
menu.BackgroundColor3, menu.BorderSizePixel, menu.ScrollBarThickness = Color3.fromRGB(30, 30, 30), 0, 6
menu.CanvasSize, menu.ZIndex = UDim2.new(0, 0, 0, 0), 2
local menuCorner = Instance.new("UICorner", menu); menuCorner.CornerRadius = UDim.new(0, 8)
local menuStroke = Instance.new("UIStroke", menu); menuStroke.Thickness, menuStroke.ApplyStrokeMode = 2, Enum.ApplyStrokeMode.Border

local titleBar = Instance.new("TextLabel", menu)
titleBar.Size, titleBar.Position = UDim2.new(1, 0, 0, 30), UDim2.new(0, 0, 0, 0)
titleBar.BackgroundColor3, titleBar.Text = Color3.fromRGB(20, 20, 20), "AR Vnnxzyz Menu"
titleBar.Font, titleBar.TextScaled, titleBar.TextColor3, titleBar.ZIndex = Enum.Font.SourceSansBold, true, Color3.fromRGB(255, 255, 255), 3

local layout = Instance.new("UIListLayout", menu)
layout.Padding, layout.SortOrder, layout.HorizontalAlignment = UDim.new(0, 5), Enum.SortOrder.LayoutOrder, Enum.HorizontalAlignment.Center
local topPadding = Instance.new("UIPadding", layout); topPadding.PaddingTop = UDim.new(0, 40)
layout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
	menu.CanvasSize = UDim2.new(0, 0, 0, layout.AbsoluteContentSize.Y + topPadding.PaddingTop.Offset + 10)
end)

logoToggle.MouseButton1Click:Connect(function() menu.Visible = not menu.Visible end)
makeDraggable(logoToggle, logoToggle); makeDraggable(titleBar, menu)

local rgbLoopConnection = RunService.Heartbeat:Connect(function()
	local rgbColor = Color3.fromHSV(tick() % 6 / 6, 0.8, 1)
	if logoToggle and logoToggle.Parent then logoToggle.TextColor3, logoStroke.Color = rgbColor, rgbColor end
	if menu and menu.Parent then titleBar.TextColor3, menuStroke.Color = rgbColor, rgbColor end
end)

-- ================== [ FUNGSI KREATOR UI INTI ] ==================
local currentLayoutOrder = 1
local function updateButtonColor(button, isOn)
	if button and button:IsA("TextButton") then
		button.BackgroundColor3 = isOn and Color3.fromRGB(0, 150, 70) or Color3.fromRGB(70, 70, 70)
	end
end

-- FUNGSI UNTUK MEMBUAT OPSI (DENGAN TOMBOL KEYBIND KUSTOM)
local function createOption(name, toggleCallback)
	local speed = DEFAULTS[name]
	local conn = nil

	local frame = Instance.new("Frame", menu); frame.Name, frame.Size = name, UDim2.new(1, -10, 0, 80)
	frame.BackgroundTransparency, frame.LayoutOrder = 1, currentLayoutOrder; currentLayoutOrder = currentLayoutOrder + 1

	local button = Instance.new("TextButton", frame); button.Size, button.Text = UDim2.new(0.8, -5, 0, 30), name .. ": OFF"
	button.BackgroundColor3, button.TextColor3, button.BorderSizePixel = Color3.fromRGB(70, 70, 70), Color3.fromRGB(255, 255, 255), 0

	local ctmButton = Instance.new("TextButton", frame); ctmButton.Size, ctmButton.Position = UDim2.new(0.2, 0, 0, 30), UDim2.new(0.8, 5, 0, 0)
	ctmButton.Text, ctmButton.BackgroundColor3, ctmButton.TextColor3 = "[CTM]", Color3.fromRGB(50, 50, 80), Color3.fromRGB(200, 200, 255)
	ctmButton.BorderSizePixel, ctmButton.Font = 0, Enum.Font.SourceSansBold
	featureCtmButtons[name] = ctmButton

	local box = Instance.new("TextBox", frame); box.Name, box.Size, box.Position = "TextBox", UDim2.new(0.5, -5, 0, 25), UDim2.new(0, 0, 0, 35)
	box.Text, box.BackgroundColor3, box.TextColor3 = tostring(speed), Color3.fromRGB(40, 40, 40), Color3.fromRGB(255, 255, 255)
	box.ClearTextOnFocus, box.TextScaled, box.Font = false, true, Enum.Font.SourceSans

	local reset = Instance.new("TextButton", frame); reset.Size, reset.Position = UDim2.new(0.5, -5, 0, 25), UDim2.new(0.5, 5, 0, 35)
	reset.Text, reset.BackgroundColor3, reset.TextColor3 = "Reset", Color3.fromRGB(80, 80, 80), Color3.fromRGB(255, 255, 255)
	reset.TextScaled, reset.Font = true, Enum.Font.SourceSans

	box.FocusLost:Connect(function()
		local v = tonumber(box.Text)
		speed = v and math.clamp(v, 1, 2000) or speed
		box.Text = tostring(speed)
		if flags[name] and toggleCallback.OnSpeedChange then toggleCallback.OnSpeedChange(speed) end
	end)
	reset.MouseButton1Click:Connect(function()
		speed = DEFAULTS[name]; box.Text = tostring(speed)
		if flags[name] and toggleCallback.OnSpeedChange then toggleCallback.OnSpeedChange(speed) end
	end)

	local function toggleFeature()
		flags[name] = not flags[name]; button.Text = name .. ": " .. (flags[name] and "ON" or "OFF"); updateButtonColor(button, flags[name])
		if flags[name] then conn = toggleCallback.OnEnable(speed)
		else if conn and typeof(conn) == "RBXScriptConnection" then conn:Disconnect(); conn = nil end
			if toggleCallback.OnDisable then toggleCallback.OnDisable() end
		end
	end
	button.MouseButton1Click:Connect(toggleFeature); featureToggleFunctions[name] = toggleFeature

	ctmButton.MouseButton1Click:Connect(function()
		isListeningForKeybind, featureToBind = true, name
		ctmButton.Text, ctmButton.BackgroundColor3 = "Waiting...", Color3.fromRGB(150, 150, 50)
	end)
end

-- FUNGSI BARU UNTUK MEMBUAT TOMBOL ON/OFF (DENGAN KEYBIND KUSTOM)
local function createToggleButton(name, callback)
	local isEnabled = false

	local frame = Instance.new("Frame", menu); frame.Name, frame.Size = name, UDim2.new(1, -10, 0, 40)
	frame.BackgroundTransparency, frame.LayoutOrder = 1, currentLayoutOrder; currentLayoutOrder = currentLayoutOrder + 1

	local button = Instance.new("TextButton", frame); button.Size, button.Text = UDim2.new(0.8, -5, 1, 0), name .. ": OFF"
	button.BackgroundColor3, button.TextColor3 = Color3.fromRGB(70, 70, 70), Color3.fromRGB(255, 255, 255)

	local ctmButton = Instance.new("TextButton", frame); ctmButton.Size, ctmButton.Position = UDim2.new(0.2, 0, 1, 0), UDim2.new(0.8, 5, 0, 0)
	ctmButton.Text, ctmButton.BackgroundColor3, ctmButton.TextColor3 = "[CTM]", Color3.fromRGB(50, 50, 80), Color3.fromRGB(200, 200, 255)
	ctmButton.Font = Enum.Font.SourceSansBold; featureCtmButtons[name] = ctmButton

	local function toggleFeature()
		isEnabled = not isEnabled; button.Text = name .. ": " .. (isEnabled and "ON" or "OFF")
		updateButtonColor(button, isEnabled); if callback then callback(isEnabled) end
	end

	button.MouseButton1Click:Connect(toggleFeature); featureToggleFunctions[name] = toggleFeature

	ctmButton.MouseButton1Click:Connect(function()
		isListeningForKeybind, featureToBind = true, name
		ctmButton.Text, ctmButton.BackgroundColor3 = "Waiting...", Color3.fromRGB(150, 150, 50)
	end)
end

-- ================== [ IMPLEMENTASI FITUR UTAMA ] ==================
-- SpeedMove
createOption("SpeedMove", {
	OnEnable = function(speed)
		local function apply() if player.Character and player.Character:FindFirstChildOfClass("Humanoid") then player.Character.Humanoid.WalkSpeed = speed end end
		player.CharacterAdded:Connect(function() task.wait(0.5); if flags.SpeedMove then apply() end end); apply()
	end,
	OnSpeedChange = function(speed) if player.Character and player.Character:FindFirstChildOfClass("Humanoid") then player.Character.Humanoid.WalkSpeed = speed end end,
	OnDisable = function() if player.Character and player.Character:FindFirstChildOfClass("Humanoid") then player.Character.Humanoid.WalkSpeed = DEFAULTS.SpeedMove end end
})

-- Fly (Placeholder, akan di-override)
createOption("Fly", { OnEnable = function() end, OnDisable = function() end, OnSpeedChange = function() end})

-- Swim Speed
createOption("Swim", {
	OnEnable = function(speed)
		return RunService.Heartbeat:Connect(function()
			local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
			if hum then
				if hum:GetState() == Enum.HumanoidStateType.Swimming then hum.WalkSpeed = speed
				else
					local speedMoveFrame = menu:FindFirstChild("SpeedMove", true)
					local speedMoveValue = speedMoveFrame and tonumber(speedMoveFrame:FindFirstChild("TextBox").Text) or DEFAULTS.SpeedMove
					hum.WalkSpeed = flags.SpeedMove and speedMoveValue or DEFAULTS.SpeedMove
				end
			end
		end)
	end,
	OnDisable = function()
		local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
		if hum then
			local speedMoveFrame = menu:FindFirstChild("SpeedMove", true)
			local speedMoveValue = speedMoveFrame and tonumber(speedMoveFrame:FindFirstChild("TextBox").Text) or DEFAULTS.SpeedMove
			hum.WalkSpeed = flags.SpeedMove and speedMoveValue or DEFAULTS.SpeedMove
		end
	end
})

-- ================== [ IMPLEMENTASI FITUR TOGGLE LAINNYA ] ==================
local jumpConn, multiFloorHumanoidConn
local multiJumpEnabled, multiFloorEnabled, onlyCharacter, platformInvisible = false, false, false, false
local multiFloorParts = {}

-- === FUNGSI PLATFORM (UNTUK MULTIJUMP/FLOOR) ===
local function spawnPlatform()
	if not multiFloorEnabled then return end
	local char = player.Character; local hrp = char and char:FindFirstChild("HumanoidRootPart")
	if not hrp then return end
	for _, part in ipairs(multiFloorParts) do if (part.Position - hrp.Position).Magnitude < 15 then return end end
	if #multiFloorParts >= 3 then local oldest = table.remove(multiFloorParts, 1); if oldest then oldest:Destroy() end end
	
	local platform = Instance.new("Part"); platform.Name, platform.Size = "MultiFloorPlatform", Vector3.new(1000, 0.5, 1000)
	platform.Anchored, platform.CanCollide, platform.Material = true, true, Enum.Material.SmoothPlastic
	platform.Color, platform.Position = Color3.fromRGB(200, 200, 255), hrp.Position - Vector3.new(0, 3, 0)
	platform.Transparency, platform.Parent = platformInvisible and 1 or 0.5, workspace; table.insert(multiFloorParts, platform)
	if onlyCharacter then pcall(function()
			if not PhysicsService:IsCollisionGroupRegistered("PlayerOnlyPlatform") then PhysicsService:CreateCollisionGroup("PlayerOnlyPlatform"); PhysicsService:CollisionGroupSetCollidable("PlayerOnlyPlatform", "Default", false) end
			PhysicsService:SetPartCollisionGroup(platform, "PlayerOnlyPlatform")
			for _, part in ipairs(player.Character:GetDescendants()) do if part:IsA("BasePart") then PhysicsService:SetPartCollisionGroup(part, "PlayerOnlyPlatform") end end
	end) end
	local touched = false; platform.Touched:Connect(function(hit) if hit and hit.Parent == player.Character then touched = true end end)
	task.delay(3, function() if not touched and platform.Parent then platform:Destroy(); for i, p in ipairs(multiFloorParts) do if p == platform then table.remove(multiFloorParts, i); break end end end end)
end

createToggleButton("MultiJump", function(isEnabled)
	multiJumpEnabled = isEnabled
	if isEnabled then jumpConn = UserInputService.JumpRequest:Connect(function()
			local char = player.Character; local hum = char and char:FindFirstChildOfClass("Humanoid")
			if hum and hum:GetState() == Enum.HumanoidStateType.Freefall then
				local hrp = char:FindFirstChild("HumanoidRootPart"); if hrp then hrp.Velocity = Vector3.new(hrp.Velocity.X, 50, hrp.Velocity.Z) end
				spawnPlatform()
			end
	end) else if jumpConn then jumpConn:Disconnect(); jumpConn = nil end end
end)

createToggleButton("MultiFloor", function(isEnabled)
	multiFloorEnabled = isEnabled
	if isEnabled then local function monitorFall()
			local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid"); if not hum then return end
			if multiFloorHumanoidConn then multiFloorHumanoidConn:Disconnect() end
			multiFloorHumanoidConn = hum.StateChanged:Connect(function(_, state) if multiFloorEnabled and state == Enum.HumanoidStateType.Freefall then spawnPlatform() end end)
		end
		player.CharacterAdded:Connect(monitorFall); monitorFall()
	else if multiFloorHumanoidConn then multiFloorHumanoidConn:Disconnect(); multiFloorHumanoidConn = nil end
		for _, part in ipairs(multiFloorParts) do if part and part.Parent then part:Destroy() end end; multiFloorParts = {}
	end
end)

-- Tombol ini tidak menggunakan createToggleButton karena memiliki teks kustom
local onlyCharBtn = Instance.new("TextButton", menu)
onlyCharBtn.Size, onlyCharBtn.Text, onlyCharBtn.BackgroundColor3, onlyCharBtn.TextColor3, onlyCharBtn.LayoutOrder = UDim2.new(1, -10, 0, 40), "Platform: For All", Color3.fromRGB(70, 70, 70), Color3.fromRGB(255, 255, 255), currentLayoutOrder; currentLayoutOrder = currentLayoutOrder + 1
onlyCharBtn.MouseButton1Click:Connect(function() onlyCharacter = not onlyCharacter; onlyCharBtn.Text = "Platform: " .. (onlyCharacter and "For Player" or "For All"); updateButtonColor(onlyCharBtn, onlyCharacter) end)

local invisBtn = Instance.new("TextButton", menu)
invisBtn.Size, invisBtn.Text, invisBtn.BackgroundColor3, invisBtn.TextColor3, invisBtn.LayoutOrder = UDim2.new(1, -10, 0, 40), "InvisPlatform: OFF", Color3.fromRGB(70, 70, 70), Color3.fromRGB(255, 255, 255), currentLayoutOrder; currentLayoutOrder = currentLayoutOrder + 1
invisBtn.MouseButton1Click:Connect(function() platformInvisible = not platformInvisible; invisBtn.Text = "InvisPlatform: " .. (platformInvisible and "ON" or "OFF"); updateButtonColor(invisBtn, platformInvisible); for _, part in ipairs(multiFloorParts) do if part and part.Parent then part.Transparency = platformInvisible and 1 or 0.5 end end end)

-- ESP & AIMLOCK
local espTeamEnabled, espEnemyEnabled, espNameEnabled, rightClickAimLockEnabled = false, false, false, false
local espObjects, targetEspPlayers, supportEspPlayers = {}, {}, {}
local espLoopConnection = RunService.RenderStepped:Connect(function()
	if not (gui and gui.Parent) then espLoopConnection:Disconnect(); return end
	for target, _ in pairs(targetEspPlayers) do if not (target and target:IsDescendantOf(Players)) then targetEspPlayers[target] = nil end end
	for target, _ in pairs(supportEspPlayers) do if not (target and target:IsDescendantOf(Players)) then supportEspPlayers[target] = nil end end
	for target, espObj in pairs(espObjects) do if not (target and target.Parent and target.Character and target.Character:FindFirstChild("Head")) then if espObj.Box and espObj.Box.Parent then espObj.Box:Destroy() end; if espObj.Name and espObj.Name.Parent then espObj.Name:Destroy() end; espObjects[target] = nil end end

	for _, target in ipairs(Players:GetPlayers()) do
		if target ~= player and target.Character and target.Character:FindFirstChild("Head") then
			local head, sameTeam = target.Character.Head, (player.Team and target.Team == player.Team and player.TeamColor ~= BrickColor.new("White"))
			local isEnemy, isTeam, isTargeted, isSupported = not sameTeam, sameTeam, targetEspPlayers[target], supportEspPlayers[target]
			local showEspBox = (espEnemyEnabled and isEnemy) or (espTeamEnabled and isTeam) or isTargeted or isSupported
			if not espObjects[target] then espObjects[target] = {} end; local esp = espObjects[target]
			if showEspBox then
				local color = isTargeted and Color3.fromRGB(255, 255, 0) or isSupported and Color3.fromRGB(0, 255, 0) or isTeam and Color3.fromRGB(0, 150, 255) or Color3.fromRGB(255, 0, 0)
				if not (esp.Box and esp.Box.Parent) then local b = Instance.new("BillboardGui", head); b.Name, b.Adornee, b.AlwaysOnTop, b.Size, b.StudsOffset = "ESPBox", head, true, UDim2.new(4, 0, 4, 0), Vector3.new(0, 0.5, 0); local f = Instance.new("Frame", b); f.Size, f.BackgroundColor3, f.BackgroundTransparency, f.BorderSizePixel, f.BorderColor3 = UDim2.fromScale(1, 1), color, 0.5, 2, color; esp.Box = b else esp.Box.Frame.BackgroundColor3, esp.Box.Frame.BorderColor3 = color, color end
				if espNameEnabled then
					if not (esp.Name and esp.Name.Parent) then local n = Instance.new("BillboardGui", head); n.Name, n.Adornee, n.Size, n.StudsOffset, n.AlwaysOnTop = "ESPNameLabel", head, UDim2.new(0, 100, 0, 12), Vector3.new(0, 2.5, 0), true; local t = Instance.new("TextLabel", n); t.Size, t.BackgroundTransparency, t.Text, t.TextColor3, t.Font, t.TextSize = UDim2.fromScale(1, 1), 1, target.Name, Color3.new(1, 1, 1), Enum.Font.SourceSans, 12; local s = Instance.new("UIStroke", t); s.Thickness, s.Color = 1, color; esp.Name = n else if esp.Name.TextLabel:FindFirstChild("UIStroke") then esp.Name.TextLabel.UIStroke.Color = color end end
				elseif esp.Name and esp.Name.Parent then esp.Name:Destroy(); esp.Name = nil end
			else if esp.Box and esp.Box.Parent then esp.Box:Destroy(); esp.Box = nil end; if esp.Name and esp.Name.Parent then esp.Name:Destroy(); esp.Name = nil end end
		else if espObjects[target] then if espObjects[target].Box then espObjects[target].Box:Destroy() end; if espObjects[target].Name then espObjects[target].Name:Destroy() end; espObjects[target] = nil end end
	end
end)
createToggleButton("ESP Enemy", function(isEnabled) espEnemyEnabled = isEnabled end)
createToggleButton("ESP Team", function(isEnabled) espTeamEnabled = isEnabled end)
createToggleButton("ESP Name", function(isEnabled) espNameEnabled = isEnabled end)
createToggleButton("RightClick AimLock", function(isEnabled) rightClickAimLockEnabled = isEnabled end)

local function getClosestESPHead()
	local mousePos, cam, closestHead, shortestDist = UserInputService:GetMouseLocation(), workspace.CurrentCamera, nil, math.huge
	for target, espObj in pairs(espObjects) do
		if target and target.Character and target.Character:FindFirstChild("Head") and espObj and espObj.Box then
			local head = target.Character.Head; local screenPos, onScreen = cam:WorldToViewportPoint(head.Position)
			if onScreen then local dist = (Vector2.new(mousePos.X, mousePos.Y) - Vector2.new(screenPos.X, screenPos.Y)).Magnitude
				if dist < shortestDist and dist < 250 then shortestDist, closestHead = dist, head end
			end
		end
	end
	return closestHead
end
UserInputService.InputBegan:Connect(function(input, processed)
	if input.UserInputType == Enum.UserInputType.MouseButton2 and not processed and rightClickAimLockEnabled then
		local targetHead = getClosestESPHead()
		if targetHead then RunService:BindToRenderStep("RightClickAimLock", Enum.RenderPriority.Camera.Value + 1, function() if targetHead and targetHead.Parent and workspace.CurrentCamera then workspace.CurrentCamera.CFrame = CFrame.lookAt(workspace.CurrentCamera.CFrame.Position, targetHead.Position) else RunService:UnbindFromRenderStep("RightClickAimLock") end end) end
	end
end)
UserInputService.InputEnded:Connect(function(input) if input.UserInputType == Enum.UserInputType.MouseButton2 then pcall(function() RunService:UnbindFromRenderStep("RightClickAimLock") end) end end)

-- NOCLIP
do local noClipConnection; local function applyNoClip(state) local char = player.Character; if not char then return end; if state then noClipConnection = RunService.Stepped:Connect(function() if not (player.Character and player.Character.Parent) then return end; for _, part in ipairs(player.Character:GetDescendants()) do if part:IsA("BasePart") and part.CanCollide then part.CanCollide = false end end end) else if noClipConnection then noClipConnection:Disconnect(); noClipConnection = nil end; if player.Character and player.Character.Parent then for _, part in ipairs(player.Character:GetDescendants()) do if part:IsA("BasePart") then part.CanCollide = true end end end end end; createToggleButton("NoClip", function(isEnabled) applyNoClip(isEnabled) end) end

-- ANTI AFK
do local antiAfkEnabled = false; local function manageAntiAfkLoop() task.spawn(function() while antiAfkEnabled do task.wait(120); if antiAfkEnabled then pcall(function() local v = game:GetService("VirtualUser"); v:CaptureController(); v:SetVirtualGamepadState(true); v:ButtonB(Enum.UserInputState.Begin); task.wait(0.1); v:ButtonB(Enum.UserInputState.End); v:SetVirtualGamepadState(false) end) end end end) end; createToggleButton("Anti AFK", function(isEnabled) antiAfkEnabled = isEnabled; if isEnabled then manageAntiAfkLoop() end end) end

-- NO DEATH (Dibiarkan seperti semula karena kompleksitas metatable)
do local noDeathEnabled, connections, originalMetatable, activeForceField = false, {}, nil, nil; local function setGodMode(character, state) if not (character and character:FindFirstChildOfClass("Humanoid")) then return end; local humanoid = character:FindFirstChildOfClass("Humanoid"); if state then if not (activeForceField and activeForceField.Parent) then activeForceField = Instance.new("ForceField"); activeForceField.Visible, activeForceField.Parent = false, character end; if connections["HealthLock"] then connections["HealthLock"]:Disconnect() end; local maxHealth = humanoid.MaxHealth; humanoid.Health = maxHealth; connections["HealthLock"] = humanoid:GetPropertyChangedSignal("Health"):Connect(function() if humanoid.Health < maxHealth then humanoid.Health = maxHealth end end); if not originalMetatable then originalMetatable = getrawmetatable(humanoid) end; local newMetatable = {__index = originalMetatable.__index, __newindex = originalMetatable.__newindex}; local oldNamecall = originalMetatable.__namecall; setmetatable(humanoid, newMetatable); newMetatable.__namecall = function(self, ...) local method = getnamecallmethod():lower(); if method == "takedamage" then return end; return oldNamecall(self, ...) end else if activeForceField and activeForceField.Parent then activeForceField:Destroy(); activeForceField = nil end; if connections["HealthLock"] then connections["HealthLock"]:Disconnect(); connections["HealthLock"] = nil end; if originalMetatable then setmetatable(humanoid, originalMetatable) end end end; local noDeathBtn = Instance.new("TextButton", menu); noDeathBtn.Name, noDeathBtn.Size, noDeathBtn.Text, noDeathBtn.BackgroundColor3, noDeathBtn.TextColor3, noDeathBtn.LayoutOrder = "NoDeathButton", UDim2.new(1, -10, 0, 40), "No Death: OFF", Color3.fromRGB(70, 70, 70), Color3.fromRGB(255, 255, 255), currentLayoutOrder; currentLayoutOrder = currentLayoutOrder + 1; noDeathBtn.MouseButton1Click:Connect(function() noDeathEnabled = not noDeathEnabled; noDeathBtn.Text, noDeathBtn.BackgroundColor3 = "No Death: " .. (noDeathEnabled and "ON" or "OFF"), noDeathEnabled and Color3.fromRGB(0, 150, 70) or Color3.fromRGB(70, 70, 70); setGodMode(player.Character, noDeathEnabled) end); connections["CharacterAdded"] = player.CharacterAdded:Connect(function(character) originalMetatable = nil; task.wait(0.5); if noDeathEnabled then setGodMode(character, true) end end); local destroyButton = menu:FindFirstChild("Destroy GUI"); if destroyButton then destroyButton.MouseButton1Click:Connect(function() if noDeathEnabled then setGodMode(player.Character, false) end; for _, conn in pairs(connections) do if conn then conn:Disconnect() end end end) end end

-- WALK ON AIR
do local walkOnAirConnection; local function manageWalkOnAir(state) if state then walkOnAirConnection = RunService.Heartbeat:Connect(function() local char = player.Character; local humanoid = char and char:FindFirstChildOfClass("Humanoid"); local hrp = char and char:FindFirstChild("HumanoidRootPart"); if not (humanoid and hrp and humanoid.Health > 0 and humanoid.FloorMaterial == Enum.Material.Air) then return end; hrp.Velocity = Vector3.new(hrp.Velocity.X, 0, hrp.Velocity.Z) end) else if walkOnAirConnection then walkOnAirConnection:Disconnect(); walkOnAirConnection = nil end end end; createToggleButton("Walk On Air", function(isEnabled) manageWalkOnAir(isEnabled) end) end

-- ================== [ FUNGSI GUI LANJUTAN, LIST, DLL. ] ==================
-- ... (Kode GUI Lanjutan, Skala, Player List, dll. diletakkan di sini tanpa perubahan) ...
-- FUNGSI GUI LANJUTAN
local showUpState = false; local guiName = gui.Name or ("Gui_" .. HttpService:GenerateGUID(false):gsub("-", "")); local showUpBtn = Instance.new("TextButton", menu); showUpBtn.Size, showUpBtn.Text, showUpBtn.BackgroundColor3, showUpBtn.TextColor3, showUpBtn.LayoutOrder = UDim2.new(1, -10, 0, 40), "Interface Mode: OFF", Color3.fromRGB(70, 70, 70), Color3.fromRGB(255, 255, 255), currentLayoutOrder; currentLayoutOrder = currentLayoutOrder + 1
local function toggleSafeGui(state) showUpState = state; showUpBtn.Text = "Interface Mode: " .. (state and "ON" or "OFF"); updateButtonColor(showUpBtn, state); pcall(function() gui.Name, gui.IgnoreGuiInset, gui.DisplayOrder, gui.Parent = guiName, state, state and 999999 or 1, state and CoreGui or player:WaitForChild("PlayerGui") end) end
showUpBtn.MouseButton1Click:Connect(function() toggleSafeGui(not showUpState) end)

-- FUNGSI SKALA MENU
local menuScale, baseSize, minScale, maxScale = 1.0, menu.Size, 0.7, 1.5; local function updateMenuScale() menuScale = math.clamp(menuScale, minScale, maxScale); menu.Size = UDim2.new(baseSize.X.Scale, baseSize.X.Offset * menuScale, baseSize.Y.Scale, baseSize.Y.Offset * menuScale) end
local scaleUpBtn = Instance.new("TextButton", titleBar); scaleUpBtn.Name, scaleUpBtn.Size, scaleUpBtn.AnchorPoint, scaleUpBtn.Position, scaleUpBtn.BackgroundColor3, scaleUpBtn.Text, scaleUpBtn.Font, scaleUpBtn.TextColor3, scaleUpBtn.TextSize, scaleUpBtn.ZIndex = "ScaleUpButton", UDim2.new(0, 22, 0, 22), Vector2.new(1, 0.5), UDim2.new(1, -8, 0.5, 0), Color3.fromRGB(40, 200, 80), "+", Enum.Font.SourceSansBold, Color3.new(1,1,1), 24, 4; local cu=Instance.new("UICorner", scaleUpBtn); cu.CornerRadius=UDim.new(0.5,0); local su=Instance.new("UIStroke",scaleUpBtn); su.Thickness,su.Color=1,Color3.new(1,1,1)
local scaleDownBtn = Instance.new("TextButton", titleBar); scaleDownBtn.Name, scaleDownBtn.Size, scaleDownBtn.AnchorPoint, scaleDownBtn.Position, scaleDownBtn.BackgroundColor3, scaleDownBtn.Text, scaleDownBtn.Font, scaleDownBtn.TextColor3, scaleDownBtn.TextSize, scaleDownBtn.ZIndex = "ScaleDownButton", UDim2.new(0, 22, 0, 22), Vector2.new(1, 0.5), UDim2.new(1, -36, 0.5, 0), Color3.fromRGB(220, 40, 40), "-", Enum.Font.SourceSansBold, Color3.new(1,1,1), 24, 4; local cd=Instance.new("UICorner", scaleDownBtn); cd.CornerRadius=UDim.new(0.5,0); local sd=Instance.new("UIStroke",scaleDownBtn); sd.Thickness,sd.Color=1,Color3.new(1,1,1)
scaleUpBtn.MouseButton1Click:Connect(function() menuScale = menuScale + 0.1; updateMenuScale() end); scaleDownBtn.MouseButton1Click:Connect(function() menuScale = menuScale - 0.1; updateMenuScale() end)

-- TP PLAYER LIST
local tpListEnabled, spectatingTarget, playerListEntries, playerListUpdateConn = false, nil, {}, nil
local playerListGui = Instance.new("ScrollingFrame", gui); playerListGui.Name, playerListGui.Visible, playerListGui.Active, playerListGui.Size, playerListGui.Position, playerListGui.BackgroundColor3, playerListGui.BorderSizePixel, playerListGui.ScrollBarThickness, playerListGui.CanvasSize, playerListGui.ZIndex = "ARVnnxzyz_PlayerList", false, true, UDim2.new(0, 300, 0, 250), UDim2.new(0, 300, 0, 80), Color3.fromRGB(30, 30, 30), 0, 6, UDim2.new(0,0,0,0), 5; local lc=Instance.new("UICorner", playerListGui); lc.CornerRadius=UDim.new(0,8); local ls=Instance.new("UIStroke", playerListGui); ls.Thickness,ls.ApplyStrokeMode,ls.Color=2,Enum.ApplyStrokeMode.Border,Color3.fromRGB(0,150,255)
local listTitleBar = Instance.new("TextLabel", playerListGui); listTitleBar.Size, listTitleBar.BackgroundColor3, listTitleBar.Text, listTitleBar.Font, listTitleBar.TextScaled, listTitleBar.TextColor3, listTitleBar.ZIndex = UDim2.new(1,0,0,30), Color3.fromRGB(20,20,20), "Player List", Enum.Font.SourceSansBold, true, Color3.new(1,1,1), 6
local listLayout = Instance.new("UIListLayout", playerListGui); listLayout.Padding, listLayout.SortOrder, listLayout.HorizontalAlignment = UDim.new(0,5), Enum.SortOrder.LayoutOrder, Enum.HorizontalAlignment.Center; local ltp=Instance.new("UIPadding", listLayout); ltp.PaddingTop=UDim.new(0,35)
listLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function() playerListGui.CanvasSize = UDim2.new(0,0,0,listLayout.AbsoluteContentSize.Y + ltp.PaddingTop.Offset + 5) end); makeDraggable(listTitleBar, playerListGui)
local function stopSpectating() if not spectatingTarget then return end; pcall(function() workspace.CurrentCamera.CameraSubject, workspace.CurrentCamera.CameraType = player.Character:FindFirstChildOfClass("Humanoid"), Enum.CameraType.Custom end); if playerListEntries[spectatingTarget] and playerListEntries[spectatingTarget].Frame and playerListEntries[spectatingTarget].Frame.Parent then playerListEntries[spectatingTarget].SpectateButton.Text, playerListEntries[spectatingTarget].SpectateButton.BackgroundColor3 = "[X]", Color3.fromRGB(180,20,20) end; spectatingTarget = nil end
local function createPlayerEntry(targetPlayer) if not targetPlayer or playerListEntries[targetPlayer] then return end; local entry={}; local f=Instance.new("Frame",playerListGui); f.Name,f.Size,f.BackgroundColor3,f.BorderSizePixel=targetPlayer.Name,UDim2.new(1,-10,0,25),Color3.fromRGB(45,45,45),0; local fc=Instance.new("UICorner",f); fc.CornerRadius=UDim.new(0,4); local n=Instance.new("TextButton",f); n.Size,n.Text,n.Font,n.TextSize,n.TextColor3,n.BackgroundTransparency,n.TextXAlignment=UDim2.new(0.4,0,1,0),targetPlayer.Name,Enum.Font.SourceSans,14,Color3.new(1,1,1),1,Enum.TextXAlignment.Left; n.MouseButton1Click:Connect(function() local lc,tc=player.Character,targetPlayer.Character; if lc and tc and lc:FindFirstChild("HumanoidRootPart") and tc:FindFirstChild("HumanoidRootPart") then lc.HumanoidRootPart.CFrame = tc.HumanoidRootPart.CFrame * CFrame.new(0,3,0) end end); local h=Instance.new("TextLabel",f); h.Size,h.Position,h.Text,h.Font,h.TextSize,h.TextColor3,h.BackgroundTransparency=UDim2.new(0.15,0,1,0),UDim2.new(0.4,0,0,0),"100%",Enum.Font.SourceSans,14,Color3.new(0,1,0.5),1; local d=Instance.new("TextLabel",f); d.Size,d.Position,d.Text,d.Font,d.TextSize,d.TextColor3,d.BackgroundTransparency=UDim2.new(0.15,0,1,0),UDim2.new(0.55,0,0,0),"0m",Enum.Font.SourceSans,14,Color3.new(1,1,1),1; local s=Instance.new("TextButton",f); s.Size,s.Position,s.AnchorPoint,s.Text,s.Font,s.TextSize,s.TextColor3,s.BackgroundColor3=UDim2.new(0.15,-5,1,-4),UDim2.new(0.7,5,0.5,0),Vector2.new(0,0.5),"[X]",Enum.Font.SourceSansBold,14,Color3.new(1,1,1),Color3.fromRGB(180,20,20); local sc=Instance.new("UICorner",s); sc.CornerRadius=UDim.new(0,4); local t=Instance.new("TextButton",f); t.Size,t.Position,t.AnchorPoint,t.Text,t.Font,t.TextSize,t.TextColor3,t.BackgroundColor3=UDim2.new(0.15,-5,1,-4),UDim2.new(0.85,5,0.5,0),Vector2.new(0,0.5),"[T]",Enum.Font.SourceSansBold,14,Color3.new(1,1,1),Color3.fromRGB(70,70,70); local tc=Instance.new("UICorner",t); tc.CornerRadius=UDim.new(0,4); local sp=Instance.new("TextButton",f); sp.AnchorPoint,sp.Text,sp.Font,sp.TextSize,sp.TextColor3,sp.BackgroundColor3=Vector2.new(0,0.5),"[S]",Enum.Font.SourceSansBold,14,Color3.new(1,1,1),Color3.fromRGB(70,70,70); local spc=Instance.new("UICorner",sp); spc.CornerRadius=UDim.new(0,4); s.MouseButton1Click:Connect(function() if not targetPlayer.Character or not targetPlayer.Character:FindFirstChildOfClass("Humanoid") then return end; if spectatingTarget==targetPlayer then stopSpectating() else stopSpectating(); spectatingTarget=targetPlayer; pcall(function() workspace.CurrentCamera.CameraSubject,workspace.CurrentCamera.CameraType=targetPlayer.Character.Humanoid,Enum.CameraType.Follow end); s.Text,s.BackgroundColor3="[O]",Color3.fromRGB(20,180,20) end end); t.MouseButton1Click:Connect(function() targetEspPlayers[targetPlayer]=not targetEspPlayers[targetPlayer] end); sp.MouseButton1Click:Connect(function() supportEspPlayers[targetPlayer]=not supportEspPlayers[targetPlayer] end); entry.Frame,entry.HealthLabel,entry.DistanceLabel,entry.SpectateButton,entry.TargetButton,entry.SupportButton=f,h,d,s,t,sp; playerListEntries[targetPlayer]=entry end
local function updatePlayerList() local localChar,localRoot=player.Character,player.Character and player.Character:FindFirstChild("HumanoidRootPart"); for p,entry in pairs(playerListEntries) do if not p or not p:IsDescendantOf(Players) then if spectatingTarget==p then stopSpectating() end; entry.Frame:Destroy(); playerListEntries[p]=nil end end; for _,p in ipairs(Players:GetPlayers()) do if p~=player then if not playerListEntries[p] then createPlayerEntry(p) end; local entry=playerListEntries[p]; local targetChar,targetHum=p.Character,p.Character and p.Character:FindFirstChildOfClass("Humanoid"); if entry and targetChar and targetHum and targetHum.Health>0 and localRoot then entry.Frame.Visible=true; local health=math.floor((targetHum.Health/targetHum.MaxHealth)*100); entry.HealthLabel.Text,entry.HealthLabel.TextColor3=tostring(health).."%",Color3.fromHSV(0.33*(health/100),0.9,1); local targetRoot=targetChar:FindFirstChild("HumanoidRootPart"); if targetRoot then entry.DistanceLabel.Text=tostring(math.floor((localRoot.Position-targetRoot.Position).Magnitude)).."m" end; entry.TargetButton.BackgroundColor3,entry.TargetButton.TextColor3=targetEspPlayers[p] and Color3.fromRGB(255,255,0) or Color3.fromRGB(70,70,70), targetEspPlayers[p] and Color3.new(0,0,0) or Color3.new(1,1,1); entry.SupportButton.BackgroundColor3,entry.SupportButton.TextColor3=supportEspPlayers[p] and Color3.fromRGB(0,255,0) or Color3.fromRGB(70,70,70), supportEspPlayers[p] and Color3.new(0,0,0) or Color3.new(1,1,1) elseif entry then entry.Frame.Visible=false; if spectatingTarget==p then stopSpectating() end end end end end
local tpListFrame = Instance.new("Frame", menu); tpListFrame.Size,tpListFrame.BackgroundTransparency,tpListFrame.LayoutOrder=UDim2.new(1,-10,0,40),1,currentLayoutOrder; currentLayoutOrder=currentLayoutOrder+1; local tpll=Instance.new("UIListLayout",tpListFrame); tpll.FillDirection,tpll.HorizontalAlignment,tpll.VerticalAlignment,tpll.Padding=Enum.FillDirection.Horizontal,Enum.HorizontalAlignment.Left,Enum.VerticalAlignment.Center,UDim.new(0,5); local hlb=Instance.new("TextButton",tpListFrame); hlb.Name,hlb.Size,hlb.Text,hlb.BackgroundColor3,hlb.TextColor3,hlb.Font,hlb.TextSize="HidePlayerListButton",UDim2.new(0,30,0,30),"[H]",Color3.fromRGB(20,20,150),Color3.new(1,1,1),Enum.Font.SourceSansBold,16; hlb.MouseButton1Click:Connect(function() if tpListEnabled then playerListGui.Visible=not playerListGui.Visible end end); local tpListBtn=Instance.new("TextButton",tpListFrame); tpListBtn.Name,tpListBtn.Size,tpListBtn.Text,tpListBtn.BackgroundColor3,tpListBtn.TextColor3="TPPlayerListToggle",UDim2.new(1,-45,1,0),"TP Player List: OFF",Color3.fromRGB(70,70,70),Color3.new(1,1,1)
tpListBtn.MouseButton1Click:Connect(function() tpListEnabled=not tpListEnabled; tpListBtn.Text="TP Player List: "..(tpListEnabled and "ON" or "OFF"); updateButtonColor(tpListBtn,tpListEnabled); playerListGui.Visible=tpListEnabled; if tpListEnabled then updatePlayerList(); playerListUpdateConn=RunService.Heartbeat:Connect(updatePlayerList) else if playerListUpdateConn then playerListUpdateConn:Disconnect(); playerListUpdateConn=nil end; stopSpectating(); for _,entry in pairs(playerListEntries) do if entry.Frame then entry.Frame:Destroy() end end; playerListEntries={} end end)

-- INFO PEMAIN DI DALAM LIST
do local o_cpE, o_uPL = createPlayerEntry, updatePlayerList; createPlayerEntry = function(tP) o_cpE(tP); local e = playerListEntries[tP]; if not (e and e.Frame) then return end; local sL = Instance.new("TextLabel",e.Frame); sL.Name,sL.BackgroundTransparency,sL.Font,sL.TextSize,sL.TextXAlignment = "StatusLabel",1,Enum.Font.SourceSansBold,14,Enum.TextXAlignment.Center; e.StatusLabel=sL; local nB,hL,dL,sB,tB,spB=e.Frame:FindFirstChild(tP.Name),e.HealthLabel,e.DistanceLabel,e.SpectateButton,e.TargetButton,e.SupportButton; if nB then nB.Size=UDim2.new(0.35,0,1,0) end; sL.Size,sL.Position=UDim2.new(0.15,0,1,0),UDim2.new(0.35,0,0,0); hL.Size,hL.Position=UDim2.new(0.15,0,1,0),UDim2.new(0.5,0,0,0); dL.Size,dL.Position=UDim2.new(0.1,0,1,0),UDim2.new(0.65,0,0,0); local bS,bY,bX,bSp=UDim2.new(0.08,-4,1,-4),0.5,0.75,0.085; sB.Size,sB.Position=bS,UDim2.new(bX,4,bY,0); tB.Size,tB.Position=bS,UDim2.new(bX+bSp,4,bY,0); spB.Size,spB.Position=bS,UDim2.new(bX+(bSp*2),4,bY,0) end; updatePlayerList = function() o_uPL(); for p,e in pairs(playerListEntries) do if e.Frame and e.Frame.Visible and e.StatusLabel then local tP=p; local iT=(player.Team and tP.Team==player.Team and player.TeamColor~=BrickColor.new("White")); local iTg,iSp=targetEspPlayers[tP],supportEspPlayers[tP]; if iT then e.StatusLabel.Text,e.StatusLabel.TextColor3 = "Team", (iTg or iSp) and Color3.fromRGB(0,255,0) or Color3.fromRGB(0,170,255) else e.StatusLabel.Text,e.StatusLabel.TextColor3 = "Enemy", iTg and Color3.fromRGB(255,255,0) or iSp and Color3.fromRGB(0,255,0) or Color3.fromRGB(255,40,40) end end end end end

-- SPAM TELEPORT
do local isSpamTping, spamLoopConnection = false, nil; local spamTpBtn = Instance.new("TextButton",playerListGui); spamTpBtn.Name, spamTpBtn.Size, spamTpBtn.LayoutOrder, spamTpBtn.Text, spamTpBtn.Font, spamTpBtn.TextScaled, spamTpBtn.BackgroundColor3, spamTpBtn.TextColor3 = "SpamTpAllButton", UDim2.new(1,-10,0,35), 9999, "Spam TP All: OFF", Enum.Font.SourceSansBold, true, Color3.fromRGB(200,40,40), Color3.new(1,1,1); local c=Instance.new("UICorner",spamTpBtn); c.CornerRadius=UDim.new(0,6); local s=Instance.new("UIStroke",spamTpBtn); s.Thickness,s.Color=1.5,Color3.fromRGB(10,10,10); local function startSpamLoop() task.spawn(function() while isSpamTping do local lc=player.Character; local lh=lc and lc:FindFirstChild("HumanoidRootPart"); if not lh then break end; for _,tp in ipairs(Players:GetPlayers()) do if not isSpamTping then break end; if tp~=player then local tc=tp.Character; local th=tc and tc:FindFirstChild("HumanoidRootPart"); if th then pcall(function() lh.CFrame=th.CFrame end); task.wait() end end end; task.wait() end end) end; spamTpBtn.MouseButton1Click:Connect(function() isSpamTping=not isSpamTping; if isSpamTping then spamTpBtn.BackgroundColor3,spamTpBtn.Text=Color3.fromRGB(40,200,40),"Spam TP All: ON"; startSpamLoop() else spamTpBtn.BackgroundColor3,spamTpBtn.Text=Color3.fromRGB(200,40,40),"Spam TP All: OFF" end end); local db=menu:FindFirstChild("Destroy GUI"); if db then db.MouseButton1Click:Connect(function() isSpamTping=false; if spamLoopConnection then spamLoopConnection:Disconnect(); spamLoopConnection=nil end end) end end

-- ADVANCE HIDE GUI & ZOOM CAMERA
do local isGuiVisible = true; local mainGui = gui; UserInputService.InputBegan:Connect(function(input, gpe) if gpe then return end; if input.KeyCode==Enum.KeyCode.Return or input.KeyCode==Enum.KeyCode.KeypadEnter then isGuiVisible=not isGuiVisible; if mainGui and mainGui.Parent then mainGui.Enabled=isGuiVisible end end end); local function showNotif() local nF=Instance.new("Frame",mainGui); nF.Name,nF.AnchorPoint,nF.Position,nF.Size,nF.BackgroundColor3,nF.BackgroundTransparency,nF.BorderSizePixel,nF.ZIndex="CustomNotification",Vector2.new(1,1),UDim2.new(1,-20,1,-20),UDim2.new(0,260,0,75),Color3.fromRGB(35,35,45),1,0,99999; local c=Instance.new("UICorner",nF); c.CornerRadius=UDim.new(0,8); local s=Instance.new("UIStroke",nF); s.Thickness,s.Color,s.Transparency=1.5,Color3.fromRGB(120,120,150),1; local tL=Instance.new("TextLabel",nF); tL.Size,tL.Position,tL.AnchorPoint,tL.BackgroundTransparency,tL.Font,tL.TextColor3,tL.Text,tL.TextSize,tL.TextTransparency=UDim2.new(1,-20,0,25),UDim2.new(0.5,0,0,10),Vector2.new(0.5,0),1,Enum.Font.SourceSansBold,Color3.new(1,1,1),"Advance hide Gui",18,1; local txL=Instance.new("TextLabel",nF); txL.Size,txL.Position,txL.AnchorPoint,txL.BackgroundTransparency,txL.Font,txL.TextColor3,txL.Text,txL.TextSize,txL.TextTransparency=UDim2.new(1,-20,0,30),UDim2.new(0.5,0,0,35),Vector2.new(0.5,0),1,Enum.Font.SourceSans,Color3.fromRGB(200,200,210),"Klik Enter For Show/Hide Menu",14,1; local ti=TweenInfo.new(0.5,Enum.EasingStyle.Quad); local fIF,fIS,fIT,fIST=TweenService:Create(nF,ti,{BackgroundTransparency=0.15}),TweenService:Create(s,ti,{Transparency=0.3}),TweenService:Create(tL,ti,{TextTransparency=0}),TweenService:Create(txL,ti,{TextTransparency=0}); fIF:Play();fIS:Play();fIT:Play();fIST:Play(); task.wait(5); local fOF,fOS,fOT,fOST=TweenService:Create(nF,ti,{BackgroundTransparency=1}),TweenService:Create(s,ti,{Transparency=1}),TweenService:Create(tL,ti,{TextTransparency=1}),TweenService:Create(txL,ti,{TextTransparency=1}); fOF:Play();fOS:Play();fOT:Play();fOST:Play(); fOF.Completed:Wait(); if nF and nF.Parent then nF:Destroy() end end; if mainGui and mainGui.Parent then showNotif() end
local ZOOM_ACTION_ID="DisableArrowKeyMovementAndEnableZoom"; local zLC=nil; local MIN_ZOOM,MAX_ZOOM=2,75; local function sinkInput() return Enum.ContextActionResult.Sink end; local function manageZoom() if zLC then zLC:Disconnect();zLC=nil end; zLC=RunService.RenderStepped:Connect(function() local cam,char=workspace.CurrentCamera,player.Character; if not(cam and cam.Parent and char and char:FindFirstChild("HumanoidRootPart")) then return end; local rp,zS=char.HumanoidRootPart,2; local cD=(cam.CFrame.Position-rp.Position).Magnitude; if UserInputService:IsKeyDown(Enum.KeyCode.Down) and cD>MIN_ZOOM then cam.CFrame*=CFrame.new(0,0,-zS) end; if UserInputService:IsKeyDown(Enum.KeyCode.Up) and cD<MAX_ZOOM then cam.CFrame*=CFrame.new(0,0,zS) end end) end; ContextActionService:BindAction(ZOOM_ACTION_ID,sinkInput,false,Enum.KeyCode.Up,Enum.KeyCode.Down); manageZoom(); local db=menu:FindFirstChild("Destroy GUI"); if db then db.MouseButton1Click:Connect(function() if zLC then zLC:Disconnect() end; pcall(function() ContextActionService:UnbindAction(ZOOM_ACTION_ID) end) end) end end

-- ================== [ KREDIT & HANCURKAN GUI ] ==================
local flyBV, flyGyro, flyConn, flyInputConn, flySpeed, followCursor; local credit = Instance.new("TextLabel", menu); credit.Size, credit.Text, credit.BackgroundTransparency, credit.TextColor3, credit.TextScaled, credit.Font, credit.LayoutOrder = UDim2.new(1,-10,0,30), "Created by AR Vnnxzyz", 1, Color3.new(1,1,1), true, Enum.Font.SourceSans, currentLayoutOrder; currentLayoutOrder = currentLayoutOrder + 1
local destroy = Instance.new("TextButton", menu); destroy.Size, destroy.Text, destroy.BackgroundColor3, destroy.TextColor3, destroy.BorderSizePixel, destroy.LayoutOrder = UDim2.new(1,-10,0,40), "Destroy GUI", Color3.fromRGB(200,0,0), Color3.new(1,1,1), 0, currentLayoutOrder; currentLayoutOrder = currentLayoutOrder + 1
destroy.MouseButton1Click:Connect(function() pcall(function() gui:Destroy() end) end)
gui.Destroying:Connect(function() if jumpConn then jumpConn:Disconnect() end; if multiFloorHumanoidConn then multiFloorHumanoidConn:Disconnect() end; if flyConn then flyConn:Disconnect() end; if playerListUpdateConn then playerListUpdateConn:Disconnect() end; if flyInputConn then flyInputConn:Disconnect() end; if rgbLoopConnection then rgbLoopConnection:Disconnect() end; if espLoopConnection then espLoopConnection:Disconnect() end; pcall(function() RunService:UnbindFromRenderStep("RightClickAimLock") end); if flyBV then flyBV:Destroy() end; if flyGyro then flyGyro:Destroy() end; for _,p in ipairs(multiFloorParts) do if p and p.Parent then p:Destroy() end end; multiFloorParts={}; for _,e in pairs(espObjects) do if e.Box and e.Box.Parent then e.Box:Destroy() end; if e.Name and e.Name.Parent then e.Name:Destroy() end end; espObjects={}; supportEspPlayers={}; stopSpectating() end)

-- ================== [ MODIFIKASI AKHIR: FITUR FLY LENGKAP & HOTKEYS (DIPERBAIKI) ] ==================
do
	local flyOptionFrame = menu:FindFirstChild("Fly"); if not flyOptionFrame then warn("[MOD GAGAL]: Opsi 'Fly' tidak ditemukan."); return end
	local originalLayoutOrder = flyOptionFrame.LayoutOrder; flyOptionFrame:Destroy()
	local initialFlyCFrame,SPEED_INC,MIN_SPEED,MAX_SPEED=nil,5,5,500
	local onFlyEnable,onFlyDisable,onFlySpeedChange; onFlyEnable = function(current_speed)
		flySpeed = current_speed; local char=player.Character or player.CharacterAdded:Wait(); local hum=char and char:FindFirstChildOfClass("Humanoid"); if not hum then return end
		local root; if hum.SeatPart and hum.SeatPart:IsA("VehicleSeat") then local vM=hum.SeatPart:FindFirstAncestorOfClass("Model"); if vM and vM.PrimaryPart then root=vM.PrimaryPart else root=char:FindFirstChild("HumanoidRootPart") end else root=char:FindFirstChild("HumanoidRootPart") end; if not root then return end
		if hum then hum:ChangeState(Enum.HumanoidStateType.Running) end
		flyBV = Instance.new("BodyVelocity"); flyBV.MaxForce,flyBV.P,flyBV.Velocity,flyBV.Parent=Vector3.new(math.huge,math.huge,math.huge),50000,Vector3.zero,root
		flyGyro = Instance.new("BodyGyro"); flyGyro.MaxTorque,flyGyro.P,flyGyro.CFrame,flyGyro.Parent=Vector3.new(math.huge,math.huge,math.huge),50000,root.CFrame,root
		initialFlyCFrame = root.CFrame; local flyFrame=menu:FindFirstChild("Fly",true); local flyTextBox=flyFrame and flyFrame:FindFirstChild("TextBox")
		flyInputConn=UserInputService.InputBegan:Connect(function(input,gpe) if gpe then return end; if input.KeyCode==Enum.KeyCode.KeypadZero then if root and root.Parent and flyGyro and flyGyro.Parent and initialFlyCFrame then local _,y,p=initialFlyCFrame:ToEulerAnglesYXZ(); flyGyro.CFrame=CFrame.new(root.Position)*CFrame.Angles(p,y,0) end elseif input.KeyCode==Enum.KeyCode.KeypadOne then flySpeed=math.clamp(flySpeed+SPEED_INC,MIN_SPEED,MAX_SPEED); if flyTextBox then flyTextBox.Text=tostring(flySpeed) end elseif input.KeyCode==Enum.KeyCode.KeypadTwo then flySpeed=math.clamp(flySpeed-SPEED_INC,MIN_SPEED,MAX_SPEED); if flyTextBox then flyTextBox.Text=tostring(flySpeed) end end end)
		local mouse=player:GetMouse(); flyConn=RunService.RenderStepped:Connect(function() 
			if not(root and root.Parent and flyBV and flyBV.Parent) then 
				if flags.Fly then onFlyDisable() end; return 
			end
			local move=Vector3.zero
			if UserInputService:IsKeyDown(Enum.KeyCode.W) then move+=root.CFrame.LookVector end
			if UserInputService:IsKeyDown(Enum.KeyCode.S) then move-=root.CFrame.LookVector end
			if UserInputService:IsKeyDown(Enum.KeyCode.A) then move-=root.CFrame.RightVector end
			if UserInputService:IsKeyDown(Enum.KeyCode.D) then move+=root.CFrame.RightVector end
			if UserInputService:IsKeyDown(Enum.KeyCode.Q) then move+=Vector3.new(0,1,0) end
			if UserInputService:IsKeyDown(Enum.KeyCode.E) then move-=Vector3.new(0,1,0) end
			if UserInputService:IsKeyDown(Enum.KeyCode.KeypadSeven) then move+=root.CFrame.UpVector end
			if UserInputService:IsKeyDown(Enum.KeyCode.KeypadNine) then move-=root.CFrame.UpVector end
			flyBV.Velocity=move.Magnitude>0 and move.Unit*flySpeed or Vector3.zero
			local rS=3
			if UserInputService:IsKeyDown(Enum.KeyCode.KeypadFour) then flyGyro.CFrame*=CFrame.Angles(0,math.rad(rS),0) end
			if UserInputService:IsKeyDown(Enum.KeyCode.KeypadSix) then flyGyro.CFrame*=CFrame.Angles(0,math.rad(-rS),0) end
			if UserInputService:IsKeyDown(Enum.KeyCode.KeypadEight) then flyGyro.CFrame*=CFrame.Angles(math.rad(rS),0,0) end
			-- FIX DI SINI: UserInputSrvice -> UserInputService
			if UserInputService:IsKeyDown(Enum.KeyCode.KeypadFive) then flyGyro.CFrame*=CFrame.Angles(math.rad(-rS),0,0) end
			if followCursor and UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton1) and mouse and mouse.Hit then 
				flyGyro.CFrame=CFrame.lookAt(root.Position,mouse.Hit.p) 
			end 
		end)
		return flyConn
	end
	onFlyDisable = function() if flyConn then flyConn:Disconnect();flyConn=nil end; if flyInputConn then flyInputConn:Disconnect();flyInputConn=nil end; if flyBV then flyBV:Destroy();flyBV=nil end; if flyGyro then flyGyro:Destroy();flyGyro=nil end; local hum=player.Character and player.Character:FindFirstChildOfClass("Humanoid"); if hum then hum:ChangeState(Enum.HumanoidStateType.Running) end end
	onFlySpeedChange = function(newSpeed) flySpeed=newSpeed end
	local flyCallbacks={OnEnable=onFlyEnable,OnDisable=onFlyDisable,OnSpeedChange=onFlySpeedChange}
	createOption("Fly",flyCallbacks)
	local newFlyOptionFrame=menu:FindFirstChild("Fly")
	if newFlyOptionFrame then newFlyOptionFrame.LayoutOrder=originalLayoutOrder end
end

-- ================== [ MANAJEMEN HOTKEY & KEYBINDING TERPUSAT (DIPERBARUI) ] ==================
UserInputService.InputBegan:Connect(function(input, gameProcessedEvent)
	if gameProcessedEvent then return end

	-- Jika sedang dalam mode menunggu keybind...
	if isListeningForKeybind then
		local ctmButton = featureCtmButtons[featureToBind]

		-- Logika pembatalan dengan tombol tengah mouse
		if input.UserInputType == Enum.UserInputType.MouseButton3 then
			isListeningForKeybind, featureToBind = false, nil
			ctmButton.Text, ctmButton.BackgroundColor3 = "[CTM]", Color3.fromRGB(50, 50, 80)
			return
		end
		
		-- Hanya terima input keyboard
		if input.UserInputType ~= Enum.UserInputType.Keyboard then return end
		local newKeyCode = input.KeyCode

		-- Hapus keybind lama untuk fitur ini atau jika tombol sudah digunakan
		for keyCode, featureName in pairs(customKeybinds) do
			if featureName == featureToBind or keyCode == newKeyCode then
				customKeybinds[keyCode] = nil
			end
		end

		-- Atur keybind baru
		customKeybinds[newKeyCode] = featureToBind
		ctmButton.Text = "[" .. newKeyCode.Name:gsub("Keypad", "N") .. "]"
		ctmButton.BackgroundColor3 = Color3.fromRGB(50, 50, 80)
		isListeningForKeybind, featureToBind = false, nil
		
	-- Jika tidak, cek apakah tombol yang ditekan adalah hotkey kustom
	else
		local featureName = customKeybinds[input.KeyCode]
		if featureName and featureToggleFunctions[featureName] then
			-- Jalankan fungsi toggle yang sesuai
			featureToggleFunctions[featureName]()
		end
	end
end)
