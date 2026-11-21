-- Auto Farm Soldier com menu (Fluent UI)
-- Biblioteca: https://github.com/dawid-scripts/Fluent

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local player = Players.LocalPlayer
local Functions = ReplicatedStorage.Chest.Remotes.Functions
local QuestRemote = Functions.Quest
local SkillRemote = Functions.SkillAction

local Fluent = loadstring(game:HttpGet("https://github.com/dawid-scripts/Fluent/releases/latest/download/main.lua"))()

----------------------------------------------------------------
-- Configurações / estado
----------------------------------------------------------------
local CONFIG = {
	NpcName = "Soldier [Lv. 1]",
	QuestArgs = {"take", "Kill 4 Soldiers"},
	M1Args = {"FS_None_M1"},
}

local State = {
	Farming = false,
	AutoQuest = true,
	AttackDelay = 0.15,
	Distance = 3,
	Kills = 0,
}

----------------------------------------------------------------
-- Funções do farm
----------------------------------------------------------------
local function getRoot()
	local char = player.Character or player.CharacterAdded:Wait()
	return char:WaitForChild("HumanoidRootPart")
end

local function takeQuest()
	pcall(function()
		QuestRemote:InvokeServer(table.unpack(CONFIG.QuestArgs))
	end)
end

local function m1()
	pcall(function()
		SkillRemote:InvokeServer(table.unpack(CONFIG.M1Args))
	end)
end

local function normalize(name)
	return (name:gsub("%s", ""):lower())
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

-- ignora espaços e maiúsculas: "Soldier [Lv. 1]" também funciona
local function isTarget(obj)
	if not obj:IsA("Model") or Players:GetPlayerFromCharacter(obj) then
		return false
	end
	local wanted = normalize(CONFIG.NpcName)
	local hum = obj:FindFirstChildOfClass("Humanoid")
	return normalize(obj.Name) == wanted
		or (hum ~= nil and normalize(hum.DisplayName) == wanted)
end

local function findNearestMob()
	local root = getRoot()
	local best, bestDist

	local monster = workspace:FindFirstChild("Monster")
	local folder = (monster and monster:FindFirstChild("Mon")) or workspace

	for _, obj in ipairs(folder:GetDescendants()) do
		if isTarget(obj) and isAlive(obj) then
			local dist = (getMobRoot(obj).Position - root.Position).Magnitude
			if not bestDist or dist < bestDist then
				best, bestDist = obj, dist
			end
		end
	end

	return best
end

local function killMob(mob)
	local root = getRoot()
	local hum = mob:FindFirstChildOfClass("Humanoid")

	while State.Farming and isAlive(mob) and hum.Health > 0 do
		root.CFrame = getMobRoot(mob).CFrame * CFrame.new(0, 0, State.Distance)
		root.AssemblyLinearVelocity = Vector3.zero
		m1()
		task.wait(State.AttackDelay)
	end

	if hum.Health <= 0 then
		State.Kills += 1
	end
end

----------------------------------------------------------------
-- Interface
----------------------------------------------------------------
local Window = Fluent:CreateWindow({
	Title = "Auto Farm",
	SubTitle = "Soldier [Lv. 1]",
	TabWidth = 140,
	Size = UDim2.fromOffset(520, 400),
	Acrylic = false,
	Theme = "Dark",
	MinimizeKey = Enum.KeyCode.RightShift,
})

local Tabs = {
	Farm = Window:AddTab({ Title = "Farm", Icon = "swords" }),
	Settings = Window:AddTab({ Title = "Configurações", Icon = "settings" }),
}

local KillsLabel = Tabs.Farm:AddParagraph({
	Title = "Status",
	Content = "Kills: 0",
})

local FarmToggle = Tabs.Farm:AddToggle("FarmToggle", {
	Title = "Auto Farm",
	Description = "Pega a quest e ataca o Soldier até matar",
	Default = false,
})

FarmToggle:OnChanged(function(value)
	State.Farming = value
	if value and State.AutoQuest then
		task.spawn(takeQuest)
	end
end)

local QuestToggle = Tabs.Farm:AddToggle("QuestToggle", {
	Title = "Pegar quest automaticamente",
	Description = "Pega a quest de novo a cada kill",
	Default = true,
})

QuestToggle:OnChanged(function(value)
	State.AutoQuest = value
end)

Tabs.Farm:AddButton({
	Title = "Pegar quest agora",
	Callback = function()
		takeQuest()
		Fluent:Notify({
			Title = "Quest",
			Content = "Quest solicitada.",
			Duration = 3,
		})
	end,
})

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
	Title = "Auto Farm",
	Content = "Carregado! RightShift minimiza o menu.",
	Duration = 5,
})

----------------------------------------------------------------
-- Loop principal
----------------------------------------------------------------
task.spawn(function()
	while task.wait() do
		if State.Farming then
			local mob = findNearestMob()

			if mob then
				killMob(mob)
				if State.Farming and State.AutoQuest then
					takeQuest()
				end
				KillsLabel:SetDesc("Kills: " .. State.Kills)
			else
				if os.clock() - (State.LastWarn or 0) > 10 then
					State.LastWarn = os.clock()
					Fluent:Notify({
						Title = "Auto Farm",
						Content = "Nenhum '" .. CONFIG.NpcName .. "' vivo encontrado.",
						Duration = 4,
					})
				end
				task.wait(0.5)
			end
		end
	end
end)
