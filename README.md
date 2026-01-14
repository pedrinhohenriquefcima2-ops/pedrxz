local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Camera = workspace.CurrentCamera

local LocalPlayer = Players.LocalPlayer

local espEnabled = false
local aimEnabled = false

local AIM_FOV = 9999
local PREDICTION = 0.12

local espCache = {}

local gui = Instance.new("ScreenGui")
gui.Name = "PedroHub"
gui.ResetOnSpawn = false
gui.Parent = LocalPlayer:WaitForChild("PlayerGui")

local frame = Instance.new("Frame", gui)
frame.Size = UDim2.new(0, 200, 0, 160)
frame.Position = UDim2.new(0, 20, 0.5, -80)
frame.BackgroundColor3 = Color3.fromRGB(20,20,20)
frame.BorderSizePixel = 0

local title = Instance.new("TextLabel", frame)
title.Size = UDim2.new(1,0,0,30)
title.BackgroundTransparency = 1
title.Text = "Pedro Hub"
title.TextColor3 = Color3.new(1,1,1)
title.Font = Enum.Font.SourceSansBold
title.TextSize = 22

local function makeButton(text, y)
	local b = Instance.new("TextButton", frame)
	b.Size = UDim2.new(1,-20,0,40)
	b.Position = UDim2.new(0,10,0,y)
	b.BackgroundColor3 = Color3.fromRGB(40,40,40)
	b.TextColor3 = Color3.new(1,1,1)
	b.Font = Enum.Font.SourceSansBold
	b.TextSize = 18
	b.Text = text .. ": OFF"
	return b
end

local espButton = makeButton("ESP", 40)
local aimButton = makeButton("AimLock", 90)

local function clearESP(player)
	if espCache[player] then
		espCache[player]:Destroy()
		espCache[player] = nil
	end
end

local function applyESP(player)
	if player == LocalPlayer then return end
	if player.Team == LocalPlayer.Team then return end
	if not espEnabled then return end
	if not player.Character or not player.Character:FindFirstChild("Head") then return end

	clearESP(player)

	local head = player.Character.Head

	local highlight = Instance.new("Highlight")
	highlight.FillColor = Color3.fromRGB(255,0,0)
	highlight.OutlineColor = Color3.new(1,1,1)
	highlight.Adornee = player.Character
	highlight.Parent = player.Character

	local billboard = Instance.new("BillboardGui")
	billboard.Size = UDim2.new(0,200,0,40)
	billboard.StudsOffset = Vector3.new(0,2.5,0)
	billboard.AlwaysOnTop = true
	billboard.Adornee = head

	local text = Instance.new("TextLabel", billboard)
	text.Size = UDim2.new(1,0,1,0)
	text.BackgroundTransparency = 1
	text.TextColor3 = Color3.new(1,1,1)
	text.TextStrokeTransparency = 0
	text.Font = Enum.Font.SourceSansBold
	text.TextScaled = true

	billboard.Parent = player.Character
	espCache[player] = billboard

	RunService.RenderStepped:Connect(function()
		if not espEnabled or not LocalPlayer.Character then return end
		local hrp = LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
		if not hrp or not head.Parent then return end
		local dist = math.floor((hrp.Position - head.Position).Magnitude)
		text.Text = player.Name .. " | " .. dist .. "m"
	end)
end

local function refreshESP()
	for _,p in pairs(Players:GetPlayers()) do
		if espEnabled then
			applyESP(p)
		else
			clearESP(p)
		end
	end
end

Players.PlayerAdded:Connect(function(p)
	p.CharacterAdded:Connect(function()
		if espEnabled then
			task.wait(1)
			applyESP(p)
		end
	end)
end)

local aiming = false

local function getClosestEnemy()
	local closest, dist = nil, AIM_FOV
	for _,p in pairs(Players:GetPlayers()) do
		if p ~= LocalPlayer and p.Team ~= LocalPlayer.Team
		and p.Character and p.Character:FindFirstChild("Head")
		and p.Character:FindFirstChild("Humanoid")
		and p.Character.Humanoid.Health > 0 then
			local pos, vis = Camera:WorldToViewportPoint(p.Character.Head.Position)
			if vis then
				local d = (Vector2.new(pos.X,pos.Y) -
					Vector2.new(Camera.ViewportSize.X/2,Camera.ViewportSize.Y/2)).Magnitude
				if d < dist then
					dist = d
					closest = p.Character.Head
				end
			end
		end
	end
	return closest
end

UserInputService.InputBegan:Connect(function(i,gp)
	if gp then return end
	if i.UserInputType == Enum.UserInputType.MouseButton2 then
		aiming = true
	end
end)

UserInputService.InputEnded:Connect(function(i)
	if i.UserInputType == Enum.UserInputType.MouseButton2 then
		aiming = false
	end
end)

RunService.RenderStepped:Connect(function()
	if not aimEnabled or not aiming then return end
	local target = getClosestEnemy()
	if target then
		local vel = target.AssemblyLinearVelocity or Vector3.zero
		Camera.CFrame = CFrame.new(
			Camera.CFrame.Position,
			target.Position + vel * PREDICTION
		)
	end
end)

espButton.MouseButton1Click:Connect(function()
	espEnabled = not espEnabled
	espButton.Text = "ESP: " .. (espEnabled and "ON" or "OFF")
	refreshESP()
end)

aimButton.MouseButton1Click:Connect(function()
	aimEnabled = not aimEnabled
	aimButton.Text = "AimLock: " .. (aimEnabled and "ON" or "OFF")
end)
