# HITBOX 1
--// ============================================================
--// MÓDULO: Aura Divina + Pés de Vento + Ferramentas
--// Registra no PONTO G via _G.OverdriveUI.Register
--// ============================================================

-- Aguarda o Core estar pronto
local UI = _G.OverdriveUI
if not UI then
	warn("[Módulo Utility] Overdrive UI Core não encontrado. Rode o Core primeiro.")
	return
end

--// ATALHOS
local Players = UI.Users
local RunService = UI.RunService
local UserInput = UI.UserInput
local Http = UI.Http
local LocalPlayer = UI.Player

--// SAVE/LOAD (persistência local)
local CONFIG_FOLDER = "OverdriveConfig"
local CONFIG_FILE = "utility_settings.json"

local function getConfigPath()
	if not isfolder(CONFIG_FOLDER) then
		makefolder(CONFIG_FOLDER)
	end
	return CONFIG_FOLDER .. "/" .. CONFIG_FILE
end

local function saveConfig(data)
	pcall(function()
		writefile(getConfigPath(), Http:JSONEncode(data))
	end)
end

local function loadConfig()
	local path = getConfigPath()
	if isfile(path) then
		local ok, decoded = pcall(function()
			return Http:JSONDecode(readfile(path))
		end)
		if ok and type(decoded) == "table" then
			return decoded
		end
	end
	return nil
end

local savedData = loadConfig() or {}

--// ESTADO (tudo ATIVO por padrão)
local State = {
	CubeEnabled  = true,
	NoLagMode    = true,
	HeadSize     = savedData.HeadSize     or 50,
	Transparency = savedData.Transparency or 20,
	Material     = savedData.Material     or "Neon",
	Color        = savedData.Color        or "Really blue",
	SpeedEnabled = true,
	SpeedValue   = savedData.SpeedValue   or 60,
	InfJump      = true,
	JumpValue    = savedData.JumpValue    or 100,
}

local function persist()
	saveConfig({
		HeadSize     = State.HeadSize,
		Transparency = math.floor(State.Transparency * 100 + 0.5),
		Material     = State.Material,
		Color        = State.Color,
		SpeedValue   = State.SpeedValue,
		JumpValue    = State.JumpValue,
	})
end

--// NOTIFICAÇÕES
local function notifyState(key, state)
	local msgs = {
		CubeEnabled  = state and "Aura Divina ativada!" or "Aura Divina desativada!",
		NoLagMode    = state and "Modo Otimizado ativo!" or "Modo Otimizado desativado!",
		SpeedEnabled = state and "Pés de Vento ativos!" or "Pés de Vento desativados!",
		InfJump      = state and "Salto Infinito ativo!" or "Salto Infinito desativado!",
	}
	if msgs[key] then
		UI.Notify(msgs[key], state and "success" or "info")
	end
end

local function notifyValue(key, value)
	local msgs = {
		HeadSize     = "Tamanho da Aura: " .. tostring(value),
		Transparency = "Transparência: " .. tostring(value) .. "%",
		Material     = "Textura: " .. tostring(value),
		Color        = "Cor: " .. tostring(value),
		SpeedValue   = "Velocidade: " .. tostring(value),
		JumpValue    = "Força do Salto: " .. tostring(value),
	}
	if msgs[key] then
		UI.Notify(msgs[key], "info")
	end
end

--// HELPERS LOCAIS DE UI
local function createActionButton(parent, labelText, onClick)
	local btn = Instance.new("TextButton")
	btn.Size = UDim2.new(1, 0, 0, 36)
	btn.BackgroundColor3 = UI.Colors.PanelSecond
	btn.Text = labelText
	btn.Font = Enum.Font.GothamMedium
	btn.TextSize = 13
	btn.TextColor3 = UI.Colors.TextPrimary
	btn.AutoButtonColor = false
	btn.Parent = parent
	UI.Corner(btn, 8)
	UI.Stroke(btn, UI.Colors.BorderGray, 1)
	UI.AddPressFeedback(btn, 0.97)

	btn.MouseEnter:Connect(function()
		UI.Tween(btn, UI.TW_FAST, {BackgroundColor3 = UI.Colors.Hover})
	end)
	btn.MouseLeave:Connect(function()
		UI.Tween(btn, UI.TW_FAST, {BackgroundColor3 = UI.Colors.PanelSecond})
	end)
	btn.MouseButton1Click:Connect(function()
		UI.PlayClick()
		if onClick then onClick() end
	end)
	return btn
end

local function createDropdown(parent, labelText, options, default, onChanged)
	local wrap = Instance.new("Frame")
	wrap.Size = UDim2.new(1, 0, 0, 60)
	wrap.BackgroundTransparency = 1
	wrap.Parent = parent

	local label = Instance.new("TextLabel")
	label.BackgroundTransparency = 1
	label.Size = UDim2.new(1, 0, 0, 16)
	label.Font = Enum.Font.Gotham
	label.Text = labelText
	label.TextSize = 12
	label.TextColor3 = UI.Colors.TextPrimary
	label.TextXAlignment = Enum.TextXAlignment.Left
	label.Parent = wrap

	local btn = Instance.new("TextButton")
	btn.Position = UDim2.new(0, 0, 0, 20)
	btn.Size = UDim2.new(1, 0, 0, 32)
	btn.BackgroundColor3 = UI.Colors.PanelSecond
	btn.Text = "  " .. tostring(default) .. "  ▾"
	btn.Font = Enum.Font.Gotham
	btn.TextSize = 12
	btn.TextColor3 = UI.Colors.TextPrimary
	btn.TextXAlignment = Enum.TextXAlignment.Left
	btn.AutoButtonColor = false
	btn.Parent = wrap
	UI.Corner(btn, 6)
	UI.Stroke(btn, UI.Colors.BorderGray, 1)

	local current = default
	local expanded = false
	local optionFrame

	btn.MouseButton1Click:Connect(function()
		UI.PlayClick()
		expanded = not expanded
		if expanded then
			optionFrame = Instance.new("Frame")
			optionFrame.Size = UDim2.new(1, 0, 0, #options * 26 + 8)
			optionFrame.Position = UDim2.new(0, 0, 1, 2)
			optionFrame.BackgroundColor3 = UI.Colors.CardBg
			optionFrame.BorderSizePixel = 0
			optionFrame.ZIndex = 10
			optionFrame.Parent = wrap
			UI.Corner(optionFrame, 6)
			UI.Stroke(optionFrame, UI.Colors.BorderGray, 1)
			UI.Pad(optionFrame, 4, 4, 4, 4)

			local layout = Instance.new("UIListLayout")
			layout.Padding = UDim.new(0, 2)
			layout.Parent = optionFrame

			for _, opt in ipairs(options) do
				local ob = Instance.new("TextButton")
				ob.Size = UDim2.new(1, 0, 0, 22)
				ob.BackgroundTransparency = 1
				ob.Text = "  " .. opt
				ob.Font = Enum.Font.Gotham
				ob.TextSize = 11
				ob.TextColor3 = UI.Colors.TextPrimary
				ob.TextXAlignment = Enum.TextXAlignment.Left
				ob.AutoButtonColor = false
				ob.ZIndex = 11
				ob.Parent = optionFrame
				UI.Corner(ob, 4)

				ob.MouseEnter:Connect(function()
					UI.Tween(ob, UI.TW_FAST, {BackgroundTransparency = 0, BackgroundColor3 = UI.Colors.Hover})
				end)
				ob.MouseLeave:Connect(function()
					UI.Tween(ob, UI.TW_FAST, {BackgroundTransparency = 1})
				end)
				ob.MouseButton1Click:Connect(function()
					current = opt
					btn.Text = "  " .. opt .. "  ▾"
					expanded = false
					if optionFrame then optionFrame:Destroy() end
					UI.PlayClick()
					if onChanged then onChanged(opt) end
				end)
			end
		else
			if optionFrame then optionFrame:Destroy() end
		end
	end)

	return wrap
end

--// ============================================================
--// REGISTRO NO PONTO G — ABA 1: AURA DIVINA
--// ============================================================
UI.Register("Aura Divina", {
	id = "aura_divina",
	label = "Aura Divina",
	icon = "✦",
	build = function(page, api)
		local cubeCard = api.CreateCard(page, "Aura Divina", "Manifesta uma aura brilhante ao redor de outros jogadores. O alcance é igual ao tamanho da aura.", 1)
		cubeCard.Size = UDim2.new(1, 0, 0, 320)

		api.CreateToggle(cubeCard, "Ativar Aura", State.CubeEnabled, function(state)
			State.CubeEnabled = state
			notifyState("CubeEnabled", state)
		end)

		api.CreateToggle(cubeCard, "Modo Otimizado", State.NoLagMode, function(state)
			State.NoLagMode = state
			notifyState("NoLagMode", state)
		end)

		api.CreateSlider(cubeCard, "Tamanho da Aura", 1, 200, State.HeadSize, function(v)
			State.HeadSize = v
			persist()
		end)

		api.CreateSlider(cubeCard, "Transparência", 0, 100, State.Transparency, function(v)
			State.Transparency = v / 100
			persist()
		end)

		createDropdown(cubeCard, "Textura",
			{"Neon", "SmoothPlastic", "Metal", "Glass", "Wood", "Plastic", "ForceField"},
			State.Material, function(v)
				State.Material = v
				persist()
			end)

		createDropdown(cubeCard, "Cor",
			{"Really blue", "Really red", "Lime green", "New Yeller", "Bright orange", "Bright violet", "White", "Black", "Cyan", "Magenta"},
			State.Color, function(v)
				State.Color = v
				persist()
			end)
	end,
})

--// ============================================================
--// REGISTRO NO PONTO G — ABA 2: PÉS DE VENTO
--// ============================================================
UI.Register("Pés de Vento", {
	id = "pes_de_vento",
	label = "Pés de Vento",
	icon = "≫",
	build = function(page, api)
		local moveCard = api.CreateCard(page, "Pés de Vento", "Corra como o vento e salte sem limites.", 1)
		moveCard.Size = UDim2.new(1, 0, 0, 200)

		api.CreateToggle(moveCard, "Ativar Velocidade", State.SpeedEnabled, function(state)
			State.SpeedEnabled = state
			notifyState("SpeedEnabled", state)
		end)

		api.CreateSlider(moveCard, "Velocidade", 16, 500, State.SpeedValue, function(v)
			State.SpeedValue = v
			persist()
		end)

		api.CreateToggle(moveCard, "Salto Infinito", State.InfJump, function(state)
			State.InfJump = state
			notifyState("InfJump", state)
		end)

		api.CreateSlider(moveCard, "Força do Salto", 50, 500, State.JumpValue, function(v)
			State.JumpValue = v
			if LocalPlayer.Character then
				local h = LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
				if h then
					h.UseJumpPower = true
					h.JumpPower = v
				end
			end
			persist()
		end)
	end,
})

--// ============================================================
--// REGISTRO NO PONTO G — ABA 3: FERRAMENTAS
--// ============================================================
UI.Register("Ferramentas", {
	id = "ferramentas",
	label = "Ferramentas",
	icon = "⚒",
	build = function(page, api)
		local extraCard = api.CreateCard(page, "Caixa de Ferramentas", "Utilidades rápidas do dia a dia.", 1)
		extraCard.Size = UDim2.new(1, 0, 0, 200)

		createActionButton(extraCard, "Renascer Personagem", function()
			if LocalPlayer.Character then
				LocalPlayer.Character:BreakJoints()
			end
		end)

		local rjBtn = createActionButton(extraCard, "Reentrar no Servidor", function()
			game:GetService("TeleportService"):Teleport(game.PlaceId, LocalPlayer)
		end)
		rjBtn.Position = UDim2.new(0, 0, 0, 42)

		local cpBtn = createActionButton(extraCard, "Copiar Job ID", function()
			if setclipboard then
				setclipboard(game.JobId)
				UI.Notify("Job ID copiado!", "success")
			end
		end)
		cpBtn.Position = UDim2.new(0, 0, 0, 84)

		local fbBtn = createActionButton(extraCard, "Brilho Total (Fullbright)", function()
			game:GetService("Lighting").Brightness = 2
			game:GetService("Lighting").ClockTime = 12
			game:GetService("Lighting").FogEnd = 100000
			game:GetService("Lighting").GlobalShadows = false
			UI.Notify("Brilho Total ativado!", "success")
		end)
		fbBtn.Position = UDim2.new(0, 0, 0, 126)
	end,
})

--// ============================================================
--// LOOPS DE EXECUÇÃO
--// ============================================================
local function getLocalRoot()
	if LocalPlayer.Character then
		return LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
	end
end

local function resetCube(p)
	pcall(function()
		if p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
			local rp = p.Character.HumanoidRootPart
			rp.Size = Vector3.new(2, 2, 1)
			rp.Transparency = 1
			rp.Material = Enum.Material.Plastic
			rp.BrickColor = BrickColor.new("Medium stone grey")
		end
	end)
end

-- Loop da Aura Divina
RunService.RenderStepped:Connect(function()
	if not State.CubeEnabled then return end
	local myRoot = getLocalRoot()

	for _, v in next, Players:GetPlayers() do
		if v ~= LocalPlayer then
			if State.NoLagMode then
				if myRoot and v.Character and v.Character:FindFirstChild("HumanoidRootPart") then
					local targetRoot = v.Character.HumanoidRootPart
					local distance = (myRoot.Position - targetRoot.Position).Magnitude
					if distance <= State.HeadSize then
						pcall(function()
							targetRoot.Size = Vector3.new(State.HeadSize, State.HeadSize, State.HeadSize)
							targetRoot.Transparency = State.Transparency
							targetRoot.BrickColor = BrickColor.new(State.Color)
							targetRoot.Material = State.Material
							targetRoot.CanCollide = false
							targetRoot.CastShadow = true
						end)
					else
						resetCube(v)
					end
				end
			else
				pcall(function()
					if v.Character and v.Character:FindFirstChild("HumanoidRootPart") then
						local rp = v.Character.HumanoidRootPart
						rp.Size = Vector3.new(State.HeadSize, State.HeadSize, State.HeadSize)
						rp.Transparency = State.Transparency
						rp.BrickColor = BrickColor.new(State.Color)
						rp.Material = State.Material
						rp.CanCollide = false
						rp.CastShadow = true
					end
				end)
			end
		end
	end
end)

-- Reset ao desativar
local lastCubeState = false
RunService.Heartbeat:Connect(function()
	if lastCubeState and not State.CubeEnabled then
		for _, v in next, Players:GetPlayers() do
			if v ~= LocalPlayer then resetCube(v) end
		end
	end
	lastCubeState = State.CubeEnabled
end)

-- Loop Pés de Vento
RunService.Heartbeat:Connect(function()
	if State.SpeedEnabled and LocalPlayer.Character then
		local h = LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
		if h then h.WalkSpeed = State.SpeedValue end
	end
end)

-- Salto Infinito
UserInput.JumpRequest:Connect(function()
	if State.InfJump and LocalPlayer.Character then
		local h = LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
		if h then h:ChangeState(Enum.HumanoidStateType.Jumping) end
	end
end)

--// ============================================================
--// LOAD AUTOMÁTICO — 2 segundos entre cada notificação
--// ============================================================
task.spawn(function()
	task.wait(1.5)

	local loadOrder = {
		{key = "CubeEnabled",  type = "state"},
		{key = "NoLagMode",    type = "state"},
		{key = "HeadSize",     type = "value"},
		{key = "Transparency", type = "value"},
		{key = "Material",     type = "value"},
		{key = "Color",        type = "value"},
		{key = "SpeedEnabled", type = "state"},
		{key = "SpeedValue",   type = "value"},
		{key = "InfJump",      type = "state"},
		{key = "JumpValue",    type = "value"},
	}

	for _, entry in ipairs(loadOrder) do
		local value = State[entry.key]
		if value ~= nil then
			if entry.type == "state" then
				notifyState(entry.key, value)
			else
				notifyValue(entry.key, value)
			end
		end
		task.wait(2)
	end

	-- Vai direto para a primeira aba criada por este módulo
	UI.SelectTab("Aura Divina")
end)

-- Notificação inicial do módulo
task.wait(0.5)
UI.Notify("Módulo Utility carregado! (3 abas)", "success")
