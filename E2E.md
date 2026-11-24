-- Auto Farm v3.1 (Fluent UI)
-- Base: v3. Foco desta versão: robustez (não travar, não duplicar loops,
-- não varrer o mapa inteiro sem necessidade, limpar tudo ao fechar).
-- Nenhum nome de remote, quest, argumento ou caminho do jogo foi alterado:
-- tudo continua vindo da CONFIGURAÇÃO abaixo.
-- Biblioteca: https://github.com/dawid-scripts/Fluent

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local player = Players.LocalPlayer

-- Executar de novo derruba a instância anterior (evita dois farms ao mesmo tempo)
local env = (getgenv and getgenv()) or _G
if env.KLAutoFarmStop then
	pcall(env.KLAutoFarmStop)
	env.KLAutoFarmStop = nil
end

----------------------------------------------------------------
-- Dependências (falha cedo, com mensagem clara)
----------------------------------------------------------------
local function waitChain(root, names, timeout)
	local node = root
	for _, name in ipairs(names) do
		node = node:WaitForChild(name, timeout)
		if not node then
			return nil, name
		end
	end
	return node
end

local Functions, missingName = waitChain(ReplicatedStorage, { "Chest", "Remotes", "Functions" }, 10)
if not Functions then
	warn("[AutoFarm] Não encontrei '" .. tostring(missingName)
		.. "' em ReplicatedStorage.Chest.Remotes.Functions. O jogo mudou ou ainda não carregou.")
	return
end

local QuestRemote = Functions:WaitForChild("Quest", 10)
local SkillRemote = Functions:WaitForChild("SkillAction", 10)
if not (QuestRemote and SkillRemote) then
	warn("[AutoFarm] Remote 'Quest' ou 'SkillAction' não encontrado em Functions.")
	return
end
if not (QuestRemote:IsA("RemoteFunction") and SkillRemote:IsA("RemoteFunction")) then
	warn("[AutoFarm] 'Quest' e 'SkillAction' deveriam ser RemoteFunction (o script usa InvokeServer). Classes: "
		.. QuestRemote.ClassName .. " / " .. SkillRemote.ClassName)
	return
end

local okFluent, Fluent = pcall(function()
	return loadstring(game:HttpGet("https://github.com/dawid-scripts/Fluent/releases/latest/download/main.lua"))()
end)
if not okFluent or not Fluent then
	warn("[AutoFarm] Falha ao carregar a Fluent: " .. tostring(Fluent))
	return
end

----------------------------------------------------------------
-- CONFIGURAÇÃO
----------------------------------------------------------------
local M1Args = {"FS_None_M1"}

-- Npc      = trecho do nome do monstro (ignora espaços e maiúsculas)
-- MinLevel = level necessário. Se deixar vazio (nil), o script tenta
--            descobrir pelo "[Lv. N]" no nome do monstro dentro do jogo.
--            Se não conseguir, ele AVISA e não troca para essa quest.
local QUESTS = {
	{ Name = "Kill 4 Soldiers",      MinLevel = 1,   Npc = "Soldier",      Args = {"take", "Kill 4 Soldiers"} },
	{ Name = "Kill 5 Clown Pirates", MinLevel = nil, Npc = "Clown Pirate", Args = {"take", "Kill 5 Clown Pirates"} },
	{ Name = "Kill 1 Smoky",         MinLevel = nil, Npc = "Smoky",        Args = {"take", "Kill 1 Smoky"} },
}

local STAT_NAMES = {"Melee", "Defense", "Sword", "Fruit"}

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

-- Ajustes de robustez (valores NÃO calibrados no jogo, mexa se precisar)
local MOB_FOLDER_NAME = "Monster" -- pasta dos monstros em workspace (a mesma da v3)
local MOB_TIMEOUT = 25            -- segundos máximos num mesmo mob antes de pular
local MOB_SKIP_TIME = 15          -- quanto tempo um mob pulado fica ignorado
local QUEST_TIMEOUT = 3           -- espera máxima pela resposta do remote de quest
local M1_STALE_AFTER = 2          -- se o M1 não responder em N s, libera o próximo

----------------------------------------------------------------
-- Estado
----------------------------------------------------------------
local State = {
	Alive = true, -- false = todos os loops encerram
	Farming = false,
	AutoQuest = true,
	AutoByLevel = true,
	ManualQuest = QUESTS[1].Name,
	AttackDelay = 0.15,
	Distance = 3,
	Kills = 0,
	Deaths = 0,

	AutoStats = false,
	StatAmount = 10,
	StatInterval = 5,
	Stats = {},
	LastInfo = nil,
}

local currentQuest = nil

----------------------------------------------------------------
-- Utilidades
----------------------------------------------------------------
local function normalize(name)
	return (name:gsub("%s", ""):lower())
end

-- Chave normalizada calculada uma vez por quest (antes era recalculada por objeto)
for _, q in ipairs(QUESTS) do
	q.NpcKey = normalize(q.Npc)
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

local mobFolder
local warnedMobFolder = false

local function getMobFolder()
	if mobFolder and mobFolder.Parent then
		return mobFolder
	end
	local folder = workspace:FindFirstChild(MOB_FOLDER_NAME)
	if folder then
		mobFolder = folder
		return folder
	end
	if not warnedMobFolder then
		warnedMobFolder = true
		warn("[AutoFarm] workspace." .. MOB_FOLDER_NAME
			.. " não encontrada: varrendo o workspace inteiro (mais pesado).")
	end
	return workspace
end

----------------------------------------------------------------
-- Level do jogador
----------------------------------------------------------------
local LEVEL_NAMES = { level = true, lv = true, lvl = true, levels = true }
local cachedLevelObj
local lastLevelScan = -math.huge

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

	for name, value in pairs(player:GetAttributes()) do
		if LEVEL_NAMES[name:lower()] and tonumber(value) then
			return tonumber(value)
		end
	end

	-- varredura de descendentes limitada a uma vez a cada 5 s
	if os.clock() - lastLevelScan < 5 then
		return nil
	end
	lastLevelScan = os.clock()

	for _, d in ipairs(player:GetDescendants()) do
		if (d:IsA("IntValue") or d:IsA("NumberValue"))
			and LEVEL_NAMES[d.Name:lower()] then
			cachedLevelObj = d
			return tonumber(d.Value)
		end
	end

	return nil
end

----------------------------------------------------------------
-- Monstros / level necessário das quests
----------------------------------------------------------------
local function isTarget(obj, key)
	if not obj:IsA("Model") then
		return false
	end
	local hum = obj:FindFirstChildOfClass("Humanoid")
	if not hum then
		return false
	end
	if normalize(obj.Name):find(key, 1, true) == nil
		and normalize(hum.DisplayName):find(key, 1, true) == nil then
		return false
	end
	return not Players:GetPlayerFromCharacter(obj)
end

local estimatedLevels = {}
local lastScan = {}

-- Descobre o level pelo "[Lv. N]" do monstro. Retorna nil se não achar.
local function estimateLevel(q)
	if estimatedLevels[q.Name] then
		return estimatedLevels[q.Name]
	end
	if os.clock() - (lastScan[q.Name] or -math.huge) < 5 then
		return nil
	end
	lastScan[q.Name] = os.clock()

	local lowest
	for _, obj in ipairs(getMobFolder():GetDescendants()) do
		if isTarget(obj, q.NpcKey) then
			local hum = obj:FindFirstChildOfClass("Humanoid")
			local text = obj.Name .. " " .. (hum and hum.DisplayName or "")
			local n = tonumber(text:match("[Ll][Vv]%.?%s*(%d+)"))
			if n and (not lowest or n < lowest) then
				lowest = n
			end
		end
	end

	if lowest then
		estimatedLevels[q.Name] = lowest
	end
	return lowest
end

local function requiredLevel(q)
	return q.MinLevel or estimateLevel(q)
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

-- Retorna: quest escolhida, próxima quest bloqueada, lista de quests sem level conhecido, level
local function pickQuest()
	if State.AutoByLevel then
		local level = getLevel()
		if level then
			local chosen, chosenReq = nil, -1
			local nextLocked
			local unknown = {}

			for _, q in ipairs(QUESTS) do
				local req = requiredLevel(q)
				if not req then
					table.insert(unknown, q)
				elseif level >= req then
					if req > chosenReq then
						chosen, chosenReq = q, req
					end
				elseif not nextLocked or req < nextLocked.req then
					nextLocked = { quest = q, req = req }
				end
			end

			return chosen or QUESTS[1], nextLocked, unknown, level
		end
	end
	return findQuestByName(State.ManualQuest), nil, {}, nil
end

-- Antes: InvokeServer direto no loop principal (se o servidor não respondesse,
-- o farm inteiro congelava). Agora espera a resposta, mas com limite de tempo.
local function takeQuest(q)
	local done = false
	task.spawn(function()
		pcall(QuestRemote.InvokeServer, QuestRemote, table.unpack(q.Args))
		done = true
	end)
	local startedAt = os.clock()
	while not done and State.Alive and os.clock() - startedAt < QUEST_TIMEOUT do
		task.wait()
	end
end

-- M1 sem bloquear o loop: no máximo uma chamada em andamento por vez
local m1Token = 0
local m1Pending = false
local m1StartedAt = 0

local function m1()
	if m1Pending and os.clock() - m1StartedAt < M1_STALE_AFTER then
		return
	end
	m1Token += 1
	local token = m1Token
	m1Pending = true
	m1StartedAt = os.clock()
	task.spawn(function()
		pcall(SkillRemote.InvokeServer, SkillRemote, table.unpack(M1Args))
		if token == m1Token then
			m1Pending = false
		end
	end)
end

-- mobs que estouraram o tempo ficam ignorados por um período
local skipped = setmetatable({}, { __mode = "k" })

local function findNearestMob(q)
	local root = getRoot()
	if not root then
		return nil
	end

	local now = os.clock()
	local best, bestDist
	for _, obj in ipairs(getMobFolder():GetDescendants()) do
		if isTarget(obj, q.NpcKey) and isAlive(obj) and (skipped[obj] or 0) < now then
			local dist = (getMobRoot(obj).Position - root.Position).Magnitude
			if not bestDist or dist < bestDist then
				best, bestDist = obj, dist
			end
		end
	end

	return best
end

-- Ataca até o mob morrer, ou até VOCÊ morrer / desligar o farm / estourar o tempo
local function killMob(mob)
	local hum = mob:FindFirstChildOfClass("Humanoid")
	local startedAt = os.clock()

	while State.Alive and State.Farming and isAlive(mob) do
		local root = getRoot()
		if not root then
			return false -- você morreu
		end

		if os.clock() - startedAt > MOB_TIMEOUT then
			skipped[mob] = os.clock() + MOB_SKIP_TIME
			return false
		end

		root.CFrame = getMobRoot(mob).CFrame * CFrame.new(0, 0, State.Distance)
		root.AssemblyLinearVelocity = Vector3.zero
		m1()
		task.wait(State.AttackDelay)
	end

	if hum and hum.Health <= 0 then
		State.Kills += 1
		return true
	end
	return false
end

----------------------------------------------------------------
-- Upar status
----------------------------------------------------------------
local function getStatsRemote()
	local ok, remote = pcall(function()
		return player.PlayerGui.MainGui.StarterFrame.StatsFrame.RemoteEvent
	end)
	if ok and remote and remote:IsA("RemoteEvent") then
		return remote
	end
	return nil
end

local function upgradeStat(stat, amount)
	local remote = getStatsRemote()
	if not remote then
		return false
	end
	pcall(function()
		remote:FireServer(stat, amount)
	end)
	return true
end

local function upgradeSelected()
	local any = false
	for _, stat in ipairs(STAT_NAMES) do
		if State.Stats[stat] then
			any = true
			if not upgradeStat(stat, State.StatAmount) then
				return false, "Remote dos status não encontrado (ou não é um RemoteEvent)."
			end
		end
	end
	if not any then
		return false, "Selecione ao menos um status."
	end
	return true
end

----------------------------------------------------------------
-- Diagnóstico: lista monstros e possíveis dados de quest no console
----------------------------------------------------------------
local function debugScan()
	local lines = 0
	local function out(...)
		if lines < 250 then
			print(...)
		end
		lines += 1
	end

	out("---- MONSTROS (nomes únicos) ----")
	local seen = {}
	for _, obj in ipairs(getMobFolder():GetDescendants()) do
		if obj:IsA("Model") and obj:FindFirstChildOfClass("Humanoid")
			and not Players:GetPlayerFromCharacter(obj) and not seen[obj.Name] then
			seen[obj.Name] = true
			out(obj:GetFullName())
		end
	end

	out("---- POSSÍVEIS DADOS DE QUEST ----")
	for _, root in ipairs({ ReplicatedStorage, workspace }) do
		for _, obj in ipairs(root:GetDescendants()) do
			if obj.Name:lower():find("quest") then
				out(obj.ClassName, obj:GetFullName())
			end
		end
	end

	if lines > 250 then
		print(("(... %d linhas omitidas)"):format(lines - 250))
	end

	Fluent:Notify({
		Title = "Scan",
		Content = "Resultado impresso no console do executor.",
		Duration = 5,
	})
end

----------------------------------------------------------------
-- Encerramento (único caminho de limpeza)
----------------------------------------------------------------
local function shutdown()
	State.Alive = false
	State.Farming = false
	State.AutoStats = false
	pcall(function()
		Fluent:Destroy()
	end)
	if env.KLAutoFarmStop == shutdown then
		env.KLAutoFarmStop = nil
	end
end
env.KLAutoFarmStop = shutdown

----------------------------------------------------------------
-- Interface
----------------------------------------------------------------
local Window = Fluent:CreateWindow({
	Title = "Auto Farm",
	SubTitle = "v3.1",
	TabWidth = 140,
	Size = UDim2.fromOffset(540, 460),
	Acrylic = false,
	Theme = "Dark",
	MinimizeKey = Enum.KeyCode.RightShift,
})

local Tabs = {
	Farm = Window:AddTab({ Title = "Farm", Icon = "swords" }),
	Stats = Window:AddTab({ Title = "Status", Icon = "bar-chart" }),
	Settings = Window:AddTab({ Title = "Configurações", Icon = "settings" }),
}

-- ===== Aba Farm =====
local StatusLabel = Tabs.Farm:AddParagraph({
	Title = "Status",
	Content = "Parado",
})

local function updateStatus(text)
	if not State.Alive then
		return
	end
	local level = getLevel()
	pcall(function()
		StatusLabel:SetDesc(string.format(
			"%s\nLevel: %s | Kills: %d | Mortes: %d\nQuest: %s",
			text,
			level and tostring(level) or "não encontrado",
			State.Kills,
			State.Deaths,
			currentQuest and currentQuest.Name or "-"
		))
	end)
end

Tabs.Farm:AddToggle("FarmToggle", {
	Title = "Auto Farm",
	Description = "Pega a quest, ataca e volta sozinho se morrer",
	Default = false,
}):OnChanged(function(value)
	State.Farming = value
	currentQuest = nil -- força pegar a quest ao ligar
	if not value then
		updateStatus("Parado")
	end
end)

Tabs.Farm:AddToggle("AutoByLevel", {
	Title = "Trocar de quest pelo level",
	Description = "Vai para a quest mais forte que seu level permite",
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

-- ===== Aba Status (upar stats) =====
Tabs.Stats:AddParagraph({
	Title = "Distribuir pontos",
	Content = "Marque os status que quer upar. Se tiver poucos pontos, marque só um.",
})

for _, stat in ipairs(STAT_NAMES) do
	Tabs.Stats:AddToggle("Stat_" .. stat, {
		Title = stat,
		Default = false,
	}):OnChanged(function(value)
		State.Stats[stat] = value
	end)
end

Tabs.Stats:AddSlider("StatAmount", {
	Title = "Pontos por vez",
	Default = 10,
	Min = 1,
	Max = 100,
	Rounding = 0,
	Callback = function(value)
		State.StatAmount = value
	end,
})

Tabs.Stats:AddButton({
	Title = "Upar agora",
	Callback = function()
		local ok, err = upgradeSelected()
		Fluent:Notify({
			Title = "Status",
			Content = ok and "Pontos enviados." or err,
			Duration = 3,
		})
	end,
})

Tabs.Stats:AddToggle("AutoStats", {
	Title = "Upar automaticamente",
	Description = "Repete o upar no intervalo abaixo",
	Default = false,
}):OnChanged(function(value)
	State.AutoStats = value
end)

Tabs.Stats:AddSlider("StatInterval", {
	Title = "Intervalo do auto (s)",
	Default = 5,
	Min = 1,
	Max = 60,
	Rounding = 0,
	Callback = function(value)
		State.StatInterval = value
	end,
})

-- ===== Aba Configurações =====
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
	Title = "Escanear quests/monstros (console)",
	Description = "Imprime no console o que o script encontra no jogo",
	Callback = debugScan,
})

Tabs.Settings:AddButton({
	Title = "Fechar menu e parar tudo",
	Callback = shutdown,
})

Window:SelectTab(1)

Fluent:Notify({
	Title = "Auto Farm v3.1",
	Content = "Carregado! RightShift minimiza o menu.",
	Duration = 5,
})

----------------------------------------------------------------
-- Loop do auto-upar status
----------------------------------------------------------------
task.spawn(function()
	local warned = false
	local nextRun = 0
	while State.Alive do
		task.wait(0.5)
		if State.AutoStats and os.clock() >= nextRun then
			local ok, err = upgradeSelected()
			if not ok and not warned then
				warned = true
				Fluent:Notify({ Title = "Status", Content = err, Duration = 5 })
			elseif ok then
				warned = false
			end
			nextRun = os.clock() + State.StatInterval
		end
	end
end)

----------------------------------------------------------------
-- Loop principal do farm
----------------------------------------------------------------
task.spawn(function()
	while State.Alive do
		task.wait()
		if State.Farming then
			-- Morreu? espera o respawn e recomeça
			if not aliveChar() then
				if player.Character then
					State.Deaths += 1 -- só conta se havia personagem (e não só carregando)
				end
				updateStatus("Morreu, aguardando respawn...")
				repeat
					task.wait(0.5)
				until aliveChar() or not State.Farming or not State.Alive

				if State.Farming and State.Alive then
					task.wait(2) -- deixa o personagem carregar
					currentQuest = nil -- repega a quest
				end
			end

			if State.Farming and State.Alive then
				local q, nextLocked, unknown, level = pickQuest()

				-- Avisos sobre quests bloqueadas / sem level conhecido
				local info = ""
				if nextLocked then
					info = string.format(
						"Próxima quest: %s (level %d). Você está no level %s, continuando o farm.",
						nextLocked.quest.Name, nextLocked.req, tostring(level)
					)
				end
				if #unknown > 0 then
					local list = {}
					for _, u in ipairs(unknown) do
						table.insert(list, u.Name)
					end
					info = info .. (info ~= "" and "\n" or "")
						.. "Não consegui descobrir o level de: " .. table.concat(list, ", ")
						.. ". Defina MinLevel no script."
				end
				if State.AutoByLevel and not level then
					info = "Level do jogador não encontrado, usando a quest manual."
				end
				if info ~= "" and info ~= State.LastInfo then
					State.LastInfo = info
					Fluent:Notify({ Title = "Quests", Content = info, Duration = 8 })
				end

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
