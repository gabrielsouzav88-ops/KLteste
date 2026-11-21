-- Auto Farm v2 (Fluent UI)
-- Novidades: volta pro farm ao morrer, detecta level e troca de quest sozinho.
-- Biblioteca: https://github.com/dawid-scripts/Fluent

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local player = Players.LocalPlayer
local Functions = ReplicatedStorage.Chest.Remotes.Functions
local QuestRemote = Functions.Quest
local SkillRemote = Functions.SkillAction

local Fluent = loadstring(game:HttpGet("https://github.com/dawid-scripts/Fluent/releases/latest/download/main.lua"))()

----------------------------------------------------------------
-- CONFIGURAÇÃO
----------------------------------------------------------------
local M1Args = {"FS_None_M1"}

-- Lista de quests em ordem de level. O script usa a ÚLTIMA cuja
-- MinLevel seja menor ou igual ao seu level.
-- Npc = trecho do nome do monstro (ignora espaços e maiúsculas).
-- >>> AJUSTE o MinLevel do Clown Pirates para o level real da quest <<<
local QUESTS = {
	{ Name = "Kill 4 Soldiers",       MinLevel = 1,  Npc = "Soldier",       Args = {"take", "Kill 4 Soldiers"} },
	{ Name = "Kill 5 Clown Pirates",  MinLevel = 10, Npc = "Clown Pirate",  Args = {"take", "Kill 5 Clown Pirates"} },
}

-- Se o script não achar seu level sozinho, coloque aqui o caminho, ex:
-- LEVEL_PATH = function() return player.Data.Level end
local LEVEL_PATH = function()
	-- Players.<seu nome>.PlayerStats.lvl
	local stats = player:FindFirstChild("PlayerStats")
	if stats and stats:FindFirstChild("lvl") then
		return stats.lvl
	end

	-- plano B: PlayerStats guardado em outro lugar, em uma pasta com o seu nome
	for _, root in ipairs({ ReplicatedStorage, workspace }) do
		local folder = root:FindFirstChild("PlayerStats")
		local mine = folder and folder:FindFirstChild(player.Name)
		if mine and mine:FindFirstChild("lvl") then
			return mine.lvl
		end
	end
end

----------------------------------------------------------------
-- Estado
----------------------------------------------------------------
local State = {
	Farming = false,
	AutoQuest = true,
	AutoByLevel = true,
	ManualQuest = QUESTS[1].Name,
	AttackDelay = 0.15,
	Distance = 3,
	Kills = 0,
	Deaths = 0,
}

local currentQuest = nil

----------------------------------------------------------------
-- Utilidades
----------------------------------------------------------------
local function normalize(name)
	return (name:gsub("%s", ""):lower())
end

local function aliveChar()
	local char = player.Character
	local hum = char and char:FindFirstChildOfClass("Humanoid")
	if hum and hum.Health > 0 and char:FindFirstChild("HumanoidRootPart") then
		return char
	end
end

local function getRoot()
	local char = aliveChar()
	return char and char.HumanoidRootPart
end

local function getMobRoot(mob)
	return mob:FindFirstChild("HumanoidRootPart")
		or mob.PrimaryPart
		or mob:FindFirstChildWhichIsA("BasePart", true)
end

local function isAlive(mob)
	local hum = mob and mob:FindFirstChildOfClass("Humanoid")
	return hum and hum.Health > 0 and getMobRoot(mob)
end

----------------------------------------------------------------
-- Level
----------------------------------------------------------------
local LEVEL_NAMES = { level = true, lv = true, lvl = true, levels = true }
local cachedLevelObj

local function getLevel()
	if LEVEL_PATH then
		local ok, v = pcall(LEVEL_PATH)
		if ok and v then
			return tonumber(typeof(v) == "Instance" and v.Value or v)
		end
	end

	if cachedLevelObj and cachedLevelObj.Parent then
		return tonumber(cachedLevelObj.Value)
	end

	for _, d in ipairs(player:GetDescendants()) do
		if (d:IsA("IntValue") or d:IsA("NumberValue"))
			and LEVEL_NAMES[d.Name:lower()] then
			cachedLevelObj = d
			return tonumber(d.Value)
		end
	end

	for name, value in pairs(player:GetAttributes()) do
		if LEVEL_NAMES[name:lower()] and tonumber(value) then
			return tonumber(value)
		end
	end

	return nil
end

----------------------------------------------------------------
-- Quest / ataque
----------------------------------------------------------------
local function findQuestByName(name)
	for _, q in ipairs(QUESTS) do
		if q.Name == name then
			return q
		end
	end
	return QUESTS[1]
end

local function pickQuest()
	if State.AutoByLevel then
		local level = getLevel()
		if level then
			local chosen = QUESTS[1]
			for _, q in ipairs(QUESTS) do
				if level >= q.MinLevel then
					chosen = q
				end
			end
			return chosen
		end
	end
	return findQuestByName(State.ManualQuest)
end

local function takeQuest(q)
	pcall(function()
		QuestRemote:InvokeServer(table.unpack(q.Args))
	end)
end

local function m1()
	pcall(function()
		SkillRemote:InvokeServer(table.unpack(M1Args))
	end)
end

local function isTarget(obj, npcPart)
	if not obj:IsA("Model") or Players:GetPlayerFromCharacter(obj) then
		return false
	end
	local wanted = normalize(npcPart)
	local hum = obj:FindFirstChildOfClass("Humanoid")
	if not hum then
		return false
	end
	return normalize(obj.Name):find(wanted, 1, true) ~= nil
		or normalize(hum.DisplayName):find(wanted, 1, true) ~= nil
end

local function findNearestMob(q)
	local root = getRoot()
	if not root then
		return nil
	end

	local monster = workspace:FindFirstChild("Monster")
	local folder = (monster and monster:FindFirstChild("Mon")) or workspace
	local best, bestDist

	for _, obj in ipairs(folder:GetDescendants()) do
		if isTarget(obj, q.Npc) and isAlive(obj) then
			local dist = (getMobRoot(obj).Position - root.Position).Magnitude
			if not bestDist or dist < bestDist then
				best, bestDist = obj, dist
			end
		end
	end

	return best
end

-- Ataca até o mob morrer, ou até VOCÊ morrer / desligar o farm
local function killMob(mob)
	local hum = mob:FindFirstChildOfClass("Humanoid")

	while State.Farming and isAlive(mob) and hum.Health > 0 do
		local root = getRoot()
		if not root then
			return false -- você morreu
		end

		root.CFrame = getMobRoot(mob).CFrame * CFrame.new(0, 0, State.Distance)
		root.AssemblyLinearVelocity = Vector3.zero
		m1()
		task.wait(State.AttackDelay)
	end

	if hum.Health <= 0 then
		State.Kills += 1
		return true
	end
	return false
end

----------------------------------------------------------------
-- Interface
----------------------------------------------------------------
local Window = Fluent:CreateWindow({
	Title = "Auto Farm",
	SubTitle = "v2",
	TabWidth = 140,
	Size = UDim2.fromOffset(520, 440),
	Acrylic = false,
	Theme = "Dark",
	MinimizeKey = Enum.KeyCode.RightShift,
})

local Tabs = {
	Farm = Window:AddTab({ Title = "Farm", Icon = "swords" }),
	Settings = Window:AddTab({ Title = "Configurações", Icon = "settings" }),
}

local StatusLabel = Tabs.Farm:AddParagraph({
	Title = "Status",
	Content = "Parado",
})

local function updateStatus(text)
	local level = getLevel()
	StatusLabel:SetDesc(string.format(
		"%s\nLevel: %s | Kills: %d | Mortes: %d\nQuest: %s",
		text,
		level and tostring(level) or "não encontrado",
		State.Kills,
		State.Deaths,
		currentQuest and currentQuest.Name or "-"
	))
end

local FarmToggle = Tabs.Farm:AddToggle("FarmToggle", {
	Title = "Auto Farm",
	Description = "Pega a quest, ataca e volta sozinho se morrer",
	Default = false,
})

FarmToggle:OnChanged(function(value)
	State.Farming = value
	currentQuest = nil -- força pegar a quest ao ligar
	if not value then
		updateStatus("Parado")
	end
end)

Tabs.Farm:AddToggle("AutoByLevel", {
	Title = "Trocar de quest pelo level",
	Description = "Escolhe a quest certa de acordo com seu level",
	Default = true,
}):OnChanged(function(value)
	State.AutoByLevel = value
end)

local names = {}
for _, q in ipairs(QUESTS) do
	table.insert(names, q.Name)
end

Tabs.Farm:AddDropdown("ManualQuest", {
	Title = "Quest manual",
	Description = "Usada se o level automático estiver desligado ou não for achado",
	Values = names,
	Multi = false,
	Default = 1,
}):OnChanged(function(value)
	State.ManualQuest = value
end)

Tabs.Farm:AddToggle("QuestToggle", {
	Title = "Pegar quest a cada kill",
	Default = true,
}):OnChanged(function(value)
	State.AutoQuest = value
end)

Tabs.Settings:AddSlider("AttackDelay", {
	Title = "Delay entre M1 (s)",
	Default = 0.15,
	Min = 0.05,
	Max = 1,
	Rounding = 2,
	Callback = function(value)
		State.AttackDelay = value
	end,
})

Tabs.Settings:AddSlider("Distance", {
	Title = "Distância do inimigo",
	Default = 3,
	Min = 1,
	Max = 10,
	Rounding = 1,
	Callback = function(value)
		State.Distance = value
	end,
})

Tabs.Settings:AddButton({
	Title = "Fechar menu e parar farm",
	Callback = function()
		State.Farming = false
		Fluent:Destroy()
	end,
})

Window:SelectTab(1)

Fluent:Notify({
	Title = "Auto Farm v2",
	Content = "Carregado! RightShift minimiza o menu.",
	Duration = 5,
})

----------------------------------------------------------------
-- Loop principal
----------------------------------------------------------------
task.spawn(function()
	while task.wait() do
		if State.Farming then
			-- Morreu? espera o respawn e recomeça
			if not aliveChar() then
				State.Deaths += 1
				updateStatus("Morreu, aguardando respawn...")
				repeat
					task.wait(0.5)
				until aliveChar() or not State.Farming

				if State.Farming then
					task.wait(2) -- deixa o personagem carregar
					currentQuest = nil -- repega a quest
				end
			end

			if State.Farming then
				local q = pickQuest()

				if q ~= currentQuest then
					currentQuest = q
					takeQuest(q)
					Fluent:Notify({
						Title = "Quest",
						Content = q.Name,
						Duration = 4,
					})
				end

				local mob = findNearestMob(q)

				if mob then
					updateStatus("Farmando: " .. q.Npc)
					local killed = killMob(mob)
					if killed and State.Farming and State.AutoQuest then
						takeQuest(q)
					end
				else
					updateStatus("Procurando '" .. q.Npc .. "'...")
					task.wait(0.5)
				end
			end
		end
	end
end)
