-- Serviços
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Debris = game:GetService("Debris")
local TweenService = game:GetService("TweenService")
local StarterGui = game:GetService("StarterGui")

local player = Players.LocalPlayer
local mouse = player:GetMouse()
local character = player.Character or player.CharacterAdded:Wait()
local humanoid = character:WaitForChild("Humanoid")
local hrp = character:WaitForChild("HumanoidRootPart")

-- Estado
local powerEnabled = false
local vortexParts = {}
local afterImageConnection
local dragInput, dragStart, startPos
local aura, sound, tool

-- UI Botão móvel
local screenGui = Instance.new("ScreenGui", player:WaitForChild("PlayerGui"))
screenGui.ResetOnSpawn = false

local button = Instance.new("TextButton")
button.Text = "✅"
button.Size = UDim2.new(0, 60, 0, 60)
button.Position = UDim2.new(0.5, -30, 0.2, 0)
button.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
button.TextColor3 = Color3.new(1, 1, 1)
button.TextScaled = true
button.Parent = screenGui
button.Draggable = true

-- Drag (mobile/pc)
button.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.Touch then
		dragStart = input.Position
		startPos = button.Position

		input.Changed:Connect(function()
			if input.UserInputState == Enum.UserInputState.End then
				dragStart = nil
			end
		end)
	end
end)

button.InputChanged:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
		dragInput = input
	end
end)

RunService.RenderStepped:Connect(function()
	if dragStart and dragInput then
		local delta = dragInput.Position - dragStart
		button.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
	end
end)

-- Funções de efeitos
local function enablePower()
	powerEnabled = true
	button.Text = "❎"

	-- Imortalidade
	humanoid.MaxHealth = math.huge
	humanoid.Health = math.huge
	humanoid:GetPropertyChangedSignal("Health"):Connect(function()
		if powerEnabled then
			humanoid.Health = math.huge
		end
	end)

	-- Velocidade
	humanoid.WalkSpeed = 16 * 15

	-- Aura
	aura = Instance.new("ParticleEmitter")
	aura.Texture = "rbxassetid://243660364"
	aura.Color = ColorSequence.new(Color3.fromRGB(255, 0, 0))
	aura.Size = NumberSequence.new({NumberSequenceKeypoint.new(0, 3), NumberSequenceKeypoint.new(1, 6)})
	aura.Transparency = NumberSequence.new(0.5)
	aura.Lifetime = NumberRange.new(0.5)
	aura.Rate = 30
	aura.Speed = NumberRange.new(2)
	aura.SpreadAngle = Vector2.new(360, 360)
	aura.Parent = hrp

	-- Som
	sound = Instance.new("Sound", hrp)
	sound.SoundId = "rbxassetid://1843527674"
	sound.Volume = 2
	sound:Play()

	-- Vórtice
	for i = 1, 10 do
		local part = Instance.new("Part")
		part.Size = Vector3.new(0.3, 0.3, 0.3)
		part.Shape = Enum.PartType.Ball
		part.Anchored = true
		part.CanCollide = false
		part.Material = Enum.Material.Neon
		part.Color = Color3.fromRGB(0, 170, 255)
		part.Transparency = 0.2
		part.Parent = workspace

		local angleOffset = math.rad((360 / 10) * i)

		local conn = RunService.RenderStepped:Connect(function()
			if not powerEnabled then return end
			local time = tick()
			local radius = 6
			local angle = time * 5 + angleOffset
			local x = math.cos(angle) * radius
			local z = math.sin(angle) * radius
			local y = math.sin(time * 2 + angleOffset) * 2
			part.Position = hrp.Position + Vector3.new(x, y, z)
		end)

		table.insert(vortexParts, {part = part, conn = conn})
	end

	-- AfterImage vultos
	afterImageConnection = RunService.RenderStepped:Connect(function()
		local clone = character:Clone()
		for _, desc in ipairs(clone:GetDescendants()) do
			if desc:IsA("BasePart") then
				desc.Anchored = true
				desc.CanCollide = false
				desc.Transparency = 0.5
				desc.Material = Enum.Material.ForceField
			elseif desc:IsA("Decal") or desc:IsA("ParticleEmitter") then
				desc:Destroy()
			end
		end
		clone.Name = "AfterImage"
		clone.Parent = workspace
		clone:SetPrimaryPartCFrame(character:GetPrimaryPartCFrame())
		Debris:AddItem(clone, 0.2)
	end)

	-- Deslizar no chão
	RunService.Stepped:Connect(function()
		if not powerEnabled then return end
		for _, part in ipairs(character:GetChildren()) do
			if part:IsA("BasePart") then
				part.CustomPhysicalProperties = PhysicalProperties.new(0.1, 0, 0, 0, 0)
			end
		end
	end)

	-- Tool "tu slow"
	tool = Instance.new("Tool")
	tool.Name = "tu slow"
	tool.RequiresHandle = false
	tool.CanBeDropped = false

	tool.Activated:Connect(function()
		local pos = mouse.Hit.Position

		-- Teleportar
		hrp.CFrame = CFrame.new(pos + Vector3.new(0, 5, 0))

		-- Raio
		local ray = Instance.new("Part")
		ray.Anchored = true
		ray.CanCollide = false
		ray.Size = Vector3.new(0.3, 40, 0.3)
		ray.CFrame = CFrame.new(pos + Vector3.new(0, 20, 0))
		ray.Material = Enum.Material.Neon
		ray.Color = Color3.fromRGB(255, 255, 0)
		ray.Parent = workspace
		Debris:AddItem(ray, 0.5)
	end)

	tool.Parent = player:WaitForChild("Backpack")
end

local function disablePower()
	powerEnabled = false
	button.Text = "✅"

	-- Restaurar velocidade
	humanoid.WalkSpeed = 16

	-- Remover aura e som
	if aura then aura:Destroy() end
	if sound then sound:Destroy() end

	-- Remover vortices
	for _, v in ipairs(vortexParts) do
		if v.conn then v.conn:Disconnect() end
		if v.part then v.part:Destroy() end
	end
	vortexParts = {}

	-- Desconectar vultos
	if afterImageConnection then afterImageConnection:Disconnect() end

	-- Remover tool
	if tool and tool.Parent then
		tool:Destroy()
	end
end

-- Botão alterna o estado
button.MouseButton1Click:Connect(function()
	if powerEnabled then
		disablePower()
	else
		enablePower()
	end
end)
