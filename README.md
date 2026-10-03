# GaddielHub local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local CollectionService = game:GetService("CollectionService")

--==================================================
-- CONFIGURAÇÃO
--==================================================

local MAX_EGG_DISTANCE = 30
local PULL_TIME = 0.35
local COOLDOWN = 0.5

--==================================================
-- REMOTES
--==================================================

local remotes = ReplicatedStorage:FindFirstChild("GameRemotes")

if not remotes then
	remotes = Instance.new("Folder")
	remotes.Name = "GameRemotes"
	remotes.Parent = ReplicatedStorage
end

local toggleRemote = remotes:FindFirstChild("ToggleFeature")

if not toggleRemote then
	toggleRemote = Instance.new("RemoteEvent")
	toggleRemote.Name = "ToggleFeature"
	toggleRemote.Parent = remotes
end

local pullRemote = remotes:FindFirstChild("PullEgg")

if not pullRemote then
	pullRemote = Instance.new("RemoteEvent")
	pullRemote.Name = "PullEgg"
	pullRemote.Parent = remotes
end

--==================================================
-- ESTADO DOS JOGADORES
--==================================================

local playerSettings = {}
local cooldowns = {}

local DEFAULT_SETTINGS = {
	GhostMode = false,
	SafeZone = false,
	ESP = false,
	PullEgg = true
}

local function getSettings(player)
	if not playerSettings[player] then
		playerSettings[player] = table.clone(DEFAULT_SETTINGS)
	end

	return playerSettings[player]
end

--==================================================
-- GHOST MODE
--==================================================

local function applyGhostMode(player)
	local settings = getSettings(player)

	if not player.Character then
		return
	end

	local character = player.Character

	character:SetAttribute("GhostMode", settings.GhostMode)

	-- CollisionGroup pode ser usado pelo sistema de NPC
	for _, object in ipairs(character:GetDescendants()) do
		if object:IsA("BasePart") then

			if settings.GhostMode then
				object:SetAttribute("OriginalCanCollide", object.CanCollide)
				object.CanCollide = false
			else
				local original = object:GetAttribute("OriginalCanCollide")

				if original ~= nil then
					object.CanCollide = original
				end
			end

		end
	end
end

--==================================================
-- TOGGLE
--==================================================

toggleRemote.OnServerEvent:Connect(function(player, feature, value)

	if typeof(feature) ~= "string" then
		return
	end

	if typeof(value) ~= "boolean" then
		return
	end

	local settings = getSettings(player)

	if settings[feature] == nil then
		return
	end

	settings[feature] = value

	if feature == "GhostMode" then
		applyGhostMode(player)
	end
end)

--==================================================
-- PUXAR OVO
--==================================================

local function isEgg(object)
	if not object then
		return false
	end

	if not object:IsDescendantOf(workspace) then
		return false
	end

	if object:GetAttribute("IsEgg") == true then
		return true
	end

	if CollectionService:HasTag(object, "Egg") then
		return true
	end

	return false
end

local function getEggPosition(egg)
	if egg:IsA("BasePart") then
		return egg.Position
	end

	if egg:IsA("Model") then
		local primary = egg.PrimaryPart

		if primary then
			return primary.Position
		end

		local part = egg:FindFirstChildWhichIsA("BasePart", true)

		if part then
			return part.Position
		end
	end

	return nil
end

local function moveEgg(egg, target)
	if egg:IsA("BasePart") then

		local start = egg.Position

		for i = 1, 20 do
			if not egg.Parent then
				return
			end

			local alpha = i / 20

			egg.Position = start:Lerp(
				target,
				alpha
			)

			task.wait(PULL_TIME / 20)
		end

	elseif egg:IsA("Model") then

		local start = egg:GetPivot()

		local targetCFrame = CFrame.new(target)

		for i = 1, 20 do
			if not egg.Parent then
				return
			end

			local alpha = i / 20

			egg:PivotTo(
				start:Lerp(targetCFrame, alpha)
			)

			task.wait(PULL_TIME / 20)
		end
	end
end

pullRemote.OnServerEvent:Connect(function(player, egg)

	local settings = getSettings(player)

	if not settings.PullEgg then
		return
	end

	if not isEgg(egg) then
		return
	end

	if not player.Character then
		return
	end

	local root =
		player.Character:FindFirstChild("HumanoidRootPart")

	if not root then
		return
	end

	-- Anti-spam
	local now = os.clock()

	if cooldowns[player] and
		now - cooldowns[player] < COOLDOWN then
		return
	end

	cooldowns[player] = now

	local eggPosition = getEggPosition(egg)

	if not eggPosition then
		return
	end

	-- Validação no servidor
	local distance =
		(root.Position - eggPosition).Magnitude

	if distance > MAX_EGG_DISTANCE then
		return
	end

	if egg:GetAttribute("Collected") then
		return
	end

	egg:SetAttribute("Collected", true)

	moveEgg(
		egg,
		root.Position + Vector3.new(0, 1.5, 0)
	)

	-- Aqui você pode colocar sua recompensa.
	local leaderstats = player:FindFirstChild("leaderstats")

	if leaderstats then
		local eggs = leaderstats:FindFirstChild("Eggs")

		if eggs and eggs:IsA("IntValue") then
			eggs.Value += 1
		end
	end

	if egg.Parent then
		egg:Destroy()
	end
end)

--==================================================
-- RESPAWN
--==================================================

Players.PlayerAdded:Connect(function(player)

	getSettings(player)

	player.CharacterAdded:Connect(function()
		task.wait(0.5)

		applyGhostMode(player)
	end)
end)

Players.PlayerRemoving:Connect(function(player)
	playerSettings[player] = nil
	cooldowns[player] = nil
end)
