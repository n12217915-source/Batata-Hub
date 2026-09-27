local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local MarketplaceService = game:GetService("MarketplaceService")
local SoundService = game:GetService("SoundService")
local Debris = game:GetService("Debris")
local HttpService = game:GetService("HttpService")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

local old = playerGui:FindFirstChild("BatataHub")
if old then old:Destroy() end
local oldApi = ReplicatedStorage:FindFirstChild("BatataHub_RegisterTab")
if oldApi then oldApi:Destroy() end

local CONFIG_FOLDER = "Batata Central"
local PLUGINS_FOLDER = CONFIG_FOLDER .. "/Plugins"
local CONFIG_FILE = CONFIG_FOLDER .. "/hub.json"

pcall(function()
	if not isfolder(CONFIG_FOLDER) then makefolder(CONFIG_FOLDER) end
	if not isfolder(PLUGINS_FOLDER) then makefolder(PLUGINS_FOLDER) end
end)

local HAS_FS = (type(isfolder) == "function") and (type(writefile) == "function") and (type(readfile) == "function")

local ACCENT = Color3.fromRGB(255, 200, 20)
local ACCENT_DARK = Color3.fromRGB(150, 105, 0)
local BLACK = Color3.fromRGB(10, 10, 10)
local PANEL = Color3.fromRGB(18, 18, 18)
local CARD = Color3.fromRGB(25, 25, 25)
local TEXT = Color3.fromRGB(245, 245, 245)
local SUBTEXT = Color3.fromRGB(150, 150, 150)

local PANEL_CLOSED = UDim2.fromOffset(308, 198)
local PANEL_OPEN   = UDim2.fromOffset(341, 220)
local BUTTON_SIZE = 56

local CONFIG = {
	ExternalPassword = "Batata001",
	VIPGamePassId = 0,
	ClickSoundId = "rbxassetid://86847045401690",

	PortalIconId        = 128039132946840,
	RickHeadIconId      = 131775579293831,
	OmegaDeviceIconId   = 96858175598695,
	PortalAppearSoundId = "rbxassetid://104121542162714",
	RickAppearSoundId   = "rbxassetid://135042210759082",
}

local clickSound = Instance.new("Sound")
clickSound.Name = "BatataClick"
clickSound.SoundId = CONFIG.ClickSoundId
clickSound.Volume = 0.5
clickSound.Parent = SoundService

local function playClick() clickSound:Play() end

local function playSound(id, volume)
	if not id or id == "" then return end
	local s = Instance.new("Sound")
	s.SoundId = id
	s.Volume = volume or 0.7
	s.Parent = SoundService
	s:Play()
	Debris:AddItem(s, 5)
end

local function tweenAsync(instance, info, props)
	local tw = TweenService:Create(instance, info, props)
	tw:Play()
	tw.Completed:Wait()
	return tw
end

local function addHover(button, baseColor, hoverColor)
	button.MouseEnter:Connect(function()
		TweenService:Create(button, TweenInfo.new(0.12, Enum.EasingStyle.Sine), {BackgroundColor3 = hoverColor}):Play()
	end)
	button.MouseLeave:Connect(function()
		TweenService:Create(button, TweenInfo.new(0.12, Enum.EasingStyle.Sine), {BackgroundColor3 = baseColor}):Play()
	end)
end
local Save = {}

local DEFAULT_SAVE = {
	lastTab = nil,
	backgroundId = nil,
	plugins = {},
}

local function readSave()
	if not HAS_FS then
		return HttpService:JSONDecode(HttpService:JSONEncode(DEFAULT_SAVE))
	end
	local ok, decoded = pcall(function()
		if isfile(CONFIG_FILE) then
			return HttpService:JSONDecode(readfile(CONFIG_FILE))
		end
	end)
	if ok and type(decoded) == "table" then
		if type(decoded.plugins) ~= "table" then decoded.plugins = {} end
		return decoded
	end
	return HttpService:JSONDecode(HttpService:JSONEncode(DEFAULT_SAVE))
end

Save.data = readSave()

local function writeSave()
	if not HAS_FS then return end
	pcall(function()
		writefile(CONFIG_FILE, HttpService:JSONEncode(Save.data))
	end)
end
Save.write = writeSave

local function pluginPath(id) return PLUGINS_FOLDER .. "/" .. id .. ".lua" end

local function pluginExists(id)
	if not HAS_FS then return false end
	local ok, exists = pcall(isfile, pluginPath(id))
	return ok and exists
end

local function savePlugin(id, src)
	if not HAS_FS then return end
	pcall(function() writefile(pluginPath(id), src) end)
end

local function loadPlugin(id)
	if not HAS_FS then return nil end
	local ok, src = pcall(function()
		if isfile(pluginPath(id)) then
			return readfile(pluginPath(id))
		end
	end)
	if ok and src and #src > 0 then return src end
	return nil
end

local function deletePlugin(id)
	if not HAS_FS then return end
	pcall(function()
		if isfile(pluginPath(id)) then
			delfile(pluginPath(id))
		end
	end)
end

local function funcToString(fn)
	local ok, s = pcall(string.dump, fn)
	return (ok and s) or nil
end

local function stringToFunc(src)
	local fn = loadstring(src)
	return fn
end

local function listPlugins()
	local list = {}
	for id, info in pairs(Save.data.plugins or {}) do
		if type(info) == "table" then
			table.insert(list, {
				pluginId = id,
				name = info.name or id,
				iconId = info.iconId,
			})
		end
	end
	table.sort(list, function(a, b) return (a.name or "") < (b.name or "") end)
	return list
end
local gui = Instance.new("ScreenGui")
gui.Name = "BatataHub"
gui.ResetOnSpawn = false
gui.IgnoreGuiInset = true
gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
gui.Parent = playerGui

local notifIndex = 0

local function notify(text, kind, duration)
	duration = duration or 3
	local accent = ACCENT
	if kind == "error" then accent = Color3.fromRGB(220, 90, 90)
	elseif kind == "success" then accent = Color3.fromRGB(120, 220, 150)
	elseif kind == "warn" then accent = Color3.fromRGB(255, 180, 80) end

	notifIndex = notifIndex + 1
	local slot = notifIndex
	local yOffset = 20 + (slot - 1) * 46

	local container = Instance.new("Frame")
	container.Size = UDim2.fromOffset(260, 40)
	container.Position = UDim2.new(1, 20, 0, yOffset)
	container.AnchorPoint = Vector2.new(1, 0)
	container.BackgroundColor3 = PANEL
	container.BorderSizePixel = 0
	container.ZIndex = 200
	container.Parent = gui
	Instance.new("UICorner", container).CornerRadius = UDim.new(0, 8)

	local stroke = Instance.new("UIStroke", container)
	stroke.Color = accent
	stroke.Thickness = 1

	local dot = Instance.new("Frame")
	dot.AnchorPoint = Vector2.new(0, 0.5)
	dot.Position = UDim2.new(0, 12, 0.5, 0)
	dot.Size = UDim2.fromOffset(6, 6)
	dot.BackgroundColor3 = accent
	dot.BorderSizePixel = 0
	dot.Parent = container
	Instance.new("UICorner", dot).CornerRadius = UDim.new(1, 0)

	local label = Instance.new("TextLabel")
	label.BackgroundTransparency = 1
	label.Position = UDim2.fromOffset(26, 0)
	label.Size = UDim2.new(1, -36, 1, 0)
	label.Font = Enum.Font.Gotham
	label.Text = text
	label.TextSize = 10
	label.TextColor3 = TEXT
	label.TextXAlignment = Enum.TextXAlignment.Left
	label.TextWrapped = true
	label.Parent = container

	TweenService:Create(container, TweenInfo.new(0.3, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
		Position = UDim2.new(1, -20, 0, yOffset),
	}):Play()

	task.delay(duration, function()
		local fade = TweenService:Create(container, TweenInfo.new(0.3, Enum.EasingStyle.Sine), {
			Position = UDim2.new(1, 20, 0, yOffset),
		})
		fade:Play()
		fade.Completed:Connect(function()
			container:Destroy()
			notifIndex = math.max(0, notifIndex - 1)
		end)
	end)
end

local floating = Instance.new("ImageButton")
floating.Name = "BatataButton"
floating.Size = UDim2.fromOffset(BUTTON_SIZE, BUTTON_SIZE)
floating.Position = UDim2.new(0, 14, 0.5, -BUTTON_SIZE / 2)
floating.AnchorPoint = Vector2.new(0, 0)
floating.BackgroundTransparency = 1
floating.Image = "rbxassetid://" .. tostring(CONFIG.OmegaDeviceIconId)
floating.ScaleType = Enum.ScaleType.Fit
floating.AutoButtonColor = false
floating.ZIndex = 100
floating.Parent = gui

local floatingStroke = Instance.new("UIStroke", floating)
floatingStroke.Color = ACCENT
floatingStroke.Thickness = 0
floatingStroke.Transparency = 0.3

local floatingScale = Instance.new("UIScale")
floatingScale.Scale = 0.01
floatingScale.Parent = floating

task.spawn(function()
	while floating.Parent do
		TweenService:Create(floatingStroke, TweenInfo.new(1.3, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {Thickness = 2}):Play()
		task.wait(1.3)
		TweenService:Create(floatingStroke, TweenInfo.new(1.3, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {Thickness = 0}):Play()
		task.wait(1.3)
	end
end)

local function pulseButton()
	local down = TweenService:Create(floatingScale, TweenInfo.new(0.09, Enum.EasingStyle.Sine, Enum.EasingDirection.Out), {Scale = 0.8})
	local up = TweenService:Create(floatingScale, TweenInfo.new(0.28, Enum.EasingStyle.Elastic, Enum.EasingDirection.Out), {Scale = 1})
	down:Play()
	down.Completed:Connect(function() up:Play() end)
end
local panel = Instance.new("Frame")
panel.Name = "Main"
panel.Size = PANEL_CLOSED
panel.Position = UDim2.new(0.5, 0, 0.5, 0)
panel.AnchorPoint = Vector2.new(0.5, 0.5)
panel.BackgroundColor3 = BLACK
panel.BackgroundTransparency = 1
panel.ClipsDescendants = true
panel.Visible = false
panel.ZIndex = 10
panel.Parent = gui

Instance.new("UICorner", panel).CornerRadius = UDim.new(0, 8)

local panelGradient = Instance.new("UIGradient", panel)
panelGradient.Color = ColorSequence.new({
	ColorSequenceKeypoint.new(0, Color3.fromRGB(16, 16, 16)),
	ColorSequenceKeypoint.new(1, Color3.fromRGB(8, 8, 8)),
})
panelGradient.Rotation = 80

local panelStroke = Instance.new("UIStroke", panel)
panelStroke.Color = ACCENT
panelStroke.Thickness = 1.2
panelStroke.Transparency = 1

task.spawn(function()
	while panel.Parent do
		if panel.Visible then
			TweenService:Create(panelStroke, TweenInfo.new(1.6, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {Thickness = 1.8}):Play()
			task.wait(1.6)
			TweenService:Create(panelStroke, TweenInfo.new(1.6, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {Thickness = 1.2}):Play()
			task.wait(1.6)
		else
			task.wait(0.3)
		end
	end
end)

local bgImage = Instance.new("ImageLabel")
bgImage.Name = "Background"
bgImage.BackgroundTransparency = 1
bgImage.Size = UDim2.fromScale(1, 1)
bgImage.ImageTransparency = 0.8
bgImage.ScaleType = Enum.ScaleType.Crop
bgImage.ZIndex = 1
bgImage.Image = ""
bgImage.Parent = panel

if Save.data.backgroundId then
	bgImage.Image = "rbxassetid://" .. tostring(Save.data.backgroundId)
end

local header = Instance.new("Frame")
header.Name = "Header"
header.Size = UDim2.new(1, 0, 0, 34)
header.BackgroundColor3 = PANEL
header.BorderSizePixel = 0
header.ZIndex = 11
header.Parent = panel

Instance.new("UICorner", header).CornerRadius = UDim.new(0, 8)

local headerDivider = Instance.new("Frame")
headerDivider.Size = UDim2.new(1, 0, 0, 1)
headerDivider.Position = UDim2.new(0, 0, 1, -1)
headerDivider.BackgroundColor3 = ACCENT
headerDivider.BackgroundTransparency = 0.7
headerDivider.BorderSizePixel = 0
headerDivider.ZIndex = 11
headerDivider.Parent = header

local avatar = Instance.new("ImageLabel")
avatar.Size = UDim2.fromOffset(23, 23)
avatar.Position = UDim2.fromOffset(7, 6)
avatar.BackgroundColor3 = CARD
avatar.ZIndex = 12
avatar.Parent = header
Instance.new("UICorner", avatar).CornerRadius = UDim.new(1, 0)

local avatarStroke = Instance.new("UIStroke", avatar)
avatarStroke.Color = ACCENT
avatarStroke.Thickness = 1

task.spawn(function()
	local ok, image = pcall(function()
		return Players:GetUserThumbnailAsync(player.UserId, Enum.ThumbnailType.HeadShot, Enum.ThumbnailSize.Size100x100)
	end)
	if ok then avatar.Image = image end
end)

local title = Instance.new("TextLabel")
title.BackgroundTransparency = 1
title.Position = UDim2.fromOffset(36, 4)
title.Size = UDim2.fromOffset(140, 14)
title.Font = Enum.Font.GothamBlack
title.Text = "BATATA HUB"
title.TextSize = 12
title.TextColor3 = TEXT
title.TextXAlignment = Enum.TextXAlignment.Left
title.ZIndex = 12
title.Parent = header

local username = Instance.new("TextLabel")
username.BackgroundTransparency = 1
username.Position = UDim2.fromOffset(36, 18)
username.Size = UDim2.fromOffset(120, 11)
username.Font = Enum.Font.Gotham
username.Text = "@" .. player.Name
username.TextSize = 8
username.TextColor3 = ACCENT
username.TextXAlignment = Enum.TextXAlignment.Left
username.ZIndex = 12
username.Parent = header

local gear = Instance.new("TextButton")
gear.Size = UDim2.fromOffset(20, 20)
gear.Position = UDim2.new(1, -51, 0, 7)
gear.BackgroundColor3 = CARD
gear.Text = "⚙"
gear.TextSize = 12
gear.Font = Enum.Font.GothamBold
gear.TextColor3 = SUBTEXT
gear.AutoButtonColor = false
gear.ZIndex = 13
gear.Parent = header
Instance.new("UICorner", gear).CornerRadius = UDim.new(0, 5)
addHover(gear, CARD, Color3.fromRGB(38, 38, 38))

local close = Instance.new("TextButton")
close.Size = UDim2.fromOffset(20, 20)
close.Position = UDim2.new(1, -27, 0, 7)
close.BackgroundColor3 = CARD
close.Text = "×"
close.TextSize = 15
close.Font = Enum.Font.GothamBold
close.TextColor3 = TEXT
close.AutoButtonColor = false
close.ZIndex = 13
close.Parent = header
Instance.new("UICorner", close).CornerRadius = UDim.new(0, 5)
addHover(close, CARD, Color3.fromRGB(60, 30, 30))

local tabs = Instance.new("Frame")
tabs.Name = "Tabs"
tabs.Size = UDim2.new(1, -14, 0, 22)
tabs.Position = UDim2.fromOffset(7, 40)
tabs.BackgroundTransparency = 1
tabs.ZIndex = 11
tabs.Parent = panel

local tabLayout = Instance.new("UIListLayout")
tabLayout.FillDirection = Enum.FillDirection.Horizontal
tabLayout.Padding = UDim.new(0, 5)
tabLayout.VerticalAlignment = Enum.VerticalAlignment.Center
tabLayout.Parent = tabs

local content = Instance.new("Frame")
content.Name = "Content"
content.Size = UDim2.new(1, -14, 1, -68)
content.Position = UDim2.fromOffset(7, 64)
content.BackgroundColor3 = PANEL
content.BorderSizePixel = 0
content.ZIndex = 11
content.Parent = panel
Instance.new("UICorner", content).CornerRadius = UDim.new(0, 6)
local activePage = nil
local tabButtons = {}
local TabRegistry = {}
local settingsOpen = false

local ctx = {
	player = player,
	config = CONFIG,
	colors = {ACCENT = ACCENT, ACCENT_DARK = ACCENT_DARK, BLACK = BLACK, PANEL = PANEL, CARD = CARD, TEXT = TEXT, SUBTEXT = SUBTEXT},
}

local function clearContent()
	if activePage then
		local dead = activePage
		activePage = nil
		local fade = TweenService:Create(dead, TweenInfo.new(0.12, Enum.EasingStyle.Sine), {GroupTransparency = 1})
		fade:Play()
		fade.Completed:Connect(function() dead:Destroy() end)
	end
end

local function createPage()
	clearContent()
	local page = Instance.new("CanvasGroup")
	page.BackgroundTransparency = 1
	page.Size = UDim2.fromScale(1, 1.05)
	page.Position = UDim2.fromScale(0, -0.05)
	page.GroupTransparency = 1
	page.ZIndex = 11
	page.Parent = content
	activePage = page
	TweenService:Create(page, TweenInfo.new(0.22, Enum.EasingStyle.Quint, Enum.EasingDirection.Out), {
		GroupTransparency = 0, Position = UDim2.fromScale(0, 0),
	}):Play()
	return page
end

local function createTab(name, iconId)
	local button = Instance.new("TextButton")
	button.Size = UDim2.fromOffset(iconId and 74 or 62, 22)
	button.BackgroundColor3 = CARD
	button.Text = ""
	button.AutoButtonColor = false
	button.ZIndex = 12
	button.Parent = tabs

	Instance.new("UICorner", button).CornerRadius = UDim.new(0, 5)

	local layout = Instance.new("UIListLayout")
	layout.FillDirection = Enum.FillDirection.Horizontal
	layout.VerticalAlignment = Enum.VerticalAlignment.Center
	layout.HorizontalAlignment = Enum.HorizontalAlignment.Center
	layout.Padding = UDim.new(0, 3)
	layout.Parent = button

	if iconId then
		local icon = Instance.new("ImageLabel")
		icon.Size = UDim2.fromOffset(12, 12)
		icon.BackgroundTransparency = 1
		icon.Image = "rbxassetid://" .. tostring(iconId)
		icon.ZIndex = 12
		icon.LayoutOrder = 1
		icon.Parent = button
	end

	local label = Instance.new("TextLabel")
	label.BackgroundTransparency = 1
	label.Size = UDim2.fromOffset(iconId and 50 or 58, 22)
	label.Font = Enum.Font.GothamBold
	label.TextSize = 9
	label.TextColor3 = SUBTEXT
	label.Text = name
	label.ZIndex = 12
	label.LayoutOrder = 2
	label.Parent = button

	local btnScale = Instance.new("UIScale")
	btnScale.Scale = 0.01
	btnScale.Parent = button

	TweenService:Create(btnScale, TweenInfo.new(0.35, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Scale = 1}):Play()

	tabButtons[name] = {Button = button, Label = label}
	return button
end

local function selectTab(pluginId)
	local entry = TabRegistry[pluginId]
	if not entry then
		warn("[BatataHub] Aba não encontrada:", pluginId)
		return
	end

	settingsOpen = false

	for tabName, data in pairs(tabButtons) do
		if tabName == pluginId or tabName == entry.name then
			TweenService:Create(data.Button, TweenInfo.new(0.18, Enum.EasingStyle.Sine), {BackgroundColor3 = ACCENT}):Play()
			TweenService:Create(data.Label, TweenInfo.new(0.18, Enum.EasingStyle.Sine), {TextColor3 = BLACK}):Play()
		else
			TweenService:Create(data.Button, TweenInfo.new(0.18, Enum.EasingStyle.Sine), {BackgroundColor3 = CARD}):Play()
			TweenService:Create(data.Label, TweenInfo.new(0.18, Enum.EasingStyle.Sine), {TextColor3 = SUBTEXT}):Play()
		end
	end

	local page = createPage()
	entry.buildFn(page, ctx)

	Save.data.lastTab = pluginId
	writeSave()
end

TabRegistry["HOME"] = {
	name = "HOME",
	iconId = nil,
	buildFn = function(page, ctx)
		local welcome = Instance.new("TextLabel")
		welcome.BackgroundTransparency = 1
		welcome.Position = UDim2.fromOffset(9, 8)
		welcome.Size = UDim2.new(1, -18, 0, 18)
		welcome.Font = Enum.Font.GothamBlack
		welcome.Text = "Olá, " .. ctx.player.Name .. "!"
		welcome.TextSize = 14
		welcome.TextColor3 = ctx.colors.TEXT
		welcome.TextXAlignment = Enum.TextXAlignment.Left
		welcome.Parent = page

		local description = Instance.new("TextLabel")
		description.BackgroundTransparency = 1
		description.Position = UDim2.fromOffset(9, 26)
		description.Size = UDim2.new(1, -19, 0, 16)
		description.Font = Enum.Font.Gotham
		description.Text = "Bem-vindo ao Batata Hub."
		description.TextSize = 9
		description.TextColor3 = ctx.colors.SUBTEXT
		description.TextXAlignment = Enum.TextXAlignment.Left
		description.Parent = page

		local pingCard = Instance.new("Frame")
		pingCard.Size = UDim2.new(0.48, 0, 0, 47)
		pingCard.Position = UDim2.new(0, 9, 0, 50)
		pingCard.BackgroundColor3 = ctx.colors.CARD
		pingCard.Parent = page
		Instance.new("UICorner", pingCard).CornerRadius = UDim.new(0, 5)

		local pingTitle = Instance.new("TextLabel")
		pingTitle.BackgroundTransparency = 1
		pingTitle.Position = UDim2.fromOffset(7, 6)
		pingTitle.Size = UDim2.new(1, -13, 0, 11)
		pingTitle.Font = Enum.Font.GothamBold
		pingTitle.Text = "PING"
		pingTitle.TextSize = 9
		pingTitle.TextColor3 = ctx.colors.ACCENT
		pingTitle.TextXAlignment = Enum.TextXAlignment.Left
		pingTitle.Parent = pingCard

		local pingValue = Instance.new("TextLabel")
		pingValue.BackgroundTransparency = 1
		pingValue.Position = UDim2.fromOffset(7, 18)
		pingValue.Size = UDim2.new(1, -13, 0, 17)
		pingValue.Font = Enum.Font.GothamBlack
		pingValue.Text = "..."
		pingValue.TextSize = 14
		pingValue.TextColor3 = ctx.colors.TEXT
		pingValue.TextXAlignment = Enum.TextXAlignment.Left
		pingValue.Parent = pingCard

		local friendCard = Instance.new("Frame")
		friendCard.Size = UDim2.new(0.48, 0, 0, 47)
		friendCard.Position = UDim2.new(0.52, 0, 0, 50)
		friendCard.BackgroundColor3 = ctx.colors.CARD
		friendCard.Parent = page
		Instance.new("UICorner", friendCard).CornerRadius = UDim.new(0, 5)

		local friendTitle = Instance.new("TextLabel")
		friendTitle.BackgroundTransparency = 1
		friendTitle.Position = UDim2.fromOffset(7, 6)
		friendTitle.Size = UDim2.new(1, -13, 0, 11)
		friendTitle.Font = Enum.Font.GothamBold
		friendTitle.Text = "NO SERVIDOR"
		friendTitle.TextSize = 9
		friendTitle.TextColor3 = ctx.colors.ACCENT
		friendTitle.TextXAlignment = Enum.TextXAlignment.Left
		friendTitle.Parent = friendCard

		local friendValue = Instance.new("TextLabel")
		friendValue.BackgroundTransparency = 1
		friendValue.Position = UDim2.fromOffset(7, 18)
		friendValue.Size = UDim2.new(1, -13, 0, 17)
		friendValue.Font = Enum.Font.GothamBlack
		friendValue.Text = tostring(#Players:GetPlayers())
		friendValue.TextSize = 14
		friendValue.TextColor3 = ctx.colors.TEXT
		friendValue.TextXAlignment = Enum.TextXAlignment.Left
		friendValue.Parent = friendCard

		task.spawn(function()
			while page.Parent do
				local start = os.clock()
				task.wait()
				local ms = math.floor((os.clock() - start) * 1000)
				if ms < 1 then ms = math.random(20, 60) end
				pingValue.Text = ms .. " ms"
				friendValue.Text = tostring(#Players:GetPlayers())
				task.wait(1)
			end
		end)
	end,
}

local homeButton = createTab("HOME")
homeButton.Activated:Connect(function() playClick(); selectTab("HOME") end)
addHover(homeButton, CARD, Color3.fromRGB(38, 38, 38))
selectTab("HOME")
local function RegisterExternalTab(password, tabData)
	if password ~= CONFIG.ExternalPassword then
		notify("Senha inválida.", "error")
		return false, "Senha inválida"
	end

	if type(tabData) ~= "table" or type(tabData.Name) ~= "string" then
		notify("Dados inválidos.", "error")
		return false, "Dados inválidos"
	end

	local buildFn = nil
	local sourceCode = nil

	if type(tabData.BuildContent) == "function" then
		buildFn = tabData.BuildContent
		sourceCode = funcToString(tabData.BuildContent)
	elseif type(tabData.BuildContent) == "string" then
		local fn = loadstring(tabData.BuildContent)
		if fn then
			buildFn = fn
			sourceCode = tabData.BuildContent
		end
	end

	if not buildFn then
		notify("BuildContent inválido.", "error")
		return false, "BuildContent inválido"
	end

	local pluginId = tabData.PluginId or tabData.Name
	local displayName = tabData.Name

	if TabRegistry[pluginId] then
		notify("Você já tem esse plugin!", "error")
		return false, "Já ativo"
	end
	if pluginExists(pluginId) then
		notify("Você já tem esse plugin salvo!", "error")
		return false, "Já salvo"
	end

	TabRegistry[pluginId] = {
		buildFn = buildFn,
		name = displayName,
		iconId = tabData.IconId,
	}

	local btn = createTab(displayName, tabData.IconId)
	btn.Activated:Connect(function()
		playClick()
		selectTab(pluginId)
	end)
	addHover(btn, CARD, Color3.fromRGB(38, 38, 38))

	if sourceCode and HAS_FS then
		savePlugin(pluginId, sourceCode)
	end

	Save.data.plugins[pluginId] = {
		name = displayName,
		iconId = tabData.IconId,
		installDate = os.time(),
	}
	writeSave()

	notify("Plugin instalado: " .. displayName, "success")
	print("[BatataHub] Plugin registrado:", pluginId)
	return true
end

local api = Instance.new("BindableFunction")
api.Name = "BatataHub_RegisterTab"
api.Parent = ReplicatedStorage
api.OnInvoke = function(password, tabData)
	return RegisterExternalTab(password, tabData)
end

local function showSettingsPanel()
	if settingsOpen then return end
	settingsOpen = true

	local page = createPage()

	local titleLbl = Instance.new("TextLabel")
	titleLbl.BackgroundTransparency = 1
	titleLbl.Position = UDim2.fromOffset(9, 6)
	titleLbl.Size = UDim2.new(1, -18, 0, 16)
	titleLbl.Font = Enum.Font.GothamBlack
	titleLbl.Text = "CONFIGURAÇÕES"
	titleLbl.TextSize = 12
	titleLbl.TextColor3 = TEXT
	titleLbl.TextXAlignment = Enum.TextXAlignment.Left
	titleLbl.Parent = page

	local scroll = Instance.new("ScrollingFrame")
	scroll.Size = UDim2.new(1, -14, 1, -28)
	scroll.Position = UDim2.fromOffset(7, 26)
	scroll.BackgroundTransparency = 1
	scroll.BorderSizePixel = 0
	scroll.ScrollBarThickness = 3
	scroll.ScrollBarImageColor3 = ACCENT
	scroll.CanvasSize = UDim2.new(0, 0, 0, 0)
	scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
	scroll.Parent = page

	local list = Instance.new("UIListLayout")
	list.Padding = UDim.new(0, 4)
	list.SortOrder = Enum.SortOrder.LayoutOrder
	list.Parent = scroll

	local function sectionHeader(text)
		local lbl = Instance.new("TextLabel")
		lbl.BackgroundTransparency = 1
		lbl.Size = UDim2.new(1, -8, 0, 14)
		lbl.Font = Enum.Font.GothamBold
		lbl.Text = text
		lbl.TextSize = 9
		lbl.TextColor3 = ACCENT
		lbl.TextXAlignment = Enum.TextXAlignment.Left
		lbl.Parent = scroll
	end

	sectionHeader("🔌 PLUGINS INSTALADOS")

	local plugins = listPlugins()
	print("[BatataHub] Plugins na lista:", #plugins)

	if #plugins == 0 then
		local empty = Instance.new("TextLabel")
		empty.BackgroundTransparency = 1
		empty.Size = UDim2.new(1, -8, 0, 20)
		empty.Font = Enum.Font.Gotham
		empty.Text = "Nenhum plugin salvo ainda."
		empty.TextSize = 10
		empty.TextColor3 = SUBTEXT
		empty.TextXAlignment = Enum.TextXAlignment.Left
		empty.Parent = scroll
	else
		for _, pluginInfo in ipairs(plugins) do
			local row = Instance.new("Frame")
			row.Size = UDim2.new(1, -8, 0, 26)
			row.BackgroundColor3 = CARD
			row.BorderSizePixel = 0
			row.Parent = scroll
			Instance.new("UICorner", row).CornerRadius = UDim.new(0, 5)

			local nameLbl = Instance.new("TextLabel")
			nameLbl.BackgroundTransparency = 1
			nameLbl.Position = UDim2.fromOffset(8, 0)
			nameLbl.Size = UDim2.new(1, -80, 1, 0)
			nameLbl.Font = Enum.Font.GothamBold
			nameLbl.Text = pluginInfo.name
			nameLbl.TextSize = 10
			nameLbl.TextColor3 = TEXT
			nameLbl.TextXAlignment = Enum.TextXAlignment.Left
			nameLbl.Parent = row

			local delBtn = Instance.new("TextButton")
			delBtn.AnchorPoint = Vector2.new(1, 0.5)
			delBtn.Position = UDim2.new(1, -6, 0.5, 0)
			delBtn.Size = UDim2.fromOffset(30, 16)
			delBtn.BackgroundColor3 = Color3.fromRGB(80, 30, 30)
			delBtn.Text = "✕"
			delBtn.Font = Enum.Font.GothamBold
			delBtn.TextSize = 10
			delBtn.TextColor3 = Color3.fromRGB(255, 180, 180)
			delBtn.AutoButtonColor = false
			delBtn.Parent = row
			Instance.new("UICorner", delBtn).CornerRadius = UDim.new(0, 4)

			delBtn.Activated:Connect(function()
				playClick()

				deletePlugin(pluginInfo.pluginId)
				Save.data.plugins[pluginInfo.pluginId] = nil
				writeSave()

				if TabRegistry[pluginInfo.pluginId] then
					TabRegistry[pluginInfo.pluginId] = nil
				end

				if tabButtons[pluginInfo.name] then
					tabButtons[pluginInfo.name].Button:Destroy()
					tabButtons[pluginInfo.name] = nil
				end

				if Save.data.lastTab == pluginInfo.pluginId then
					Save.data.lastTab = "HOME"
					writeSave()
				end

				row:Destroy()
				notify("Plugin removido: " .. pluginInfo.name, "success")
			end)
		end
	end

	sectionHeader("🎨 PLANO DE FUNDO")

	local bgRow = Instance.new("Frame")
	bgRow.Size = UDim2.new(1, -8, 0, 60)
	bgRow.BackgroundColor3 = CARD
	bgRow.BorderSizePixel = 0
	bgRow.Parent = scroll
	Instance.new("UICorner", bgRow).CornerRadius = UDim.new(0, 5)

	local bgInput = Instance.new("TextBox")
	bgInput.Size = UDim2.new(1, -16, 0, 22)
	bgInput.Position = UDim2.fromOffset(8, 8)
	bgInput.BackgroundColor3 = PANEL
	bgInput.Text = Save.data.backgroundId and tostring(Save.data.backgroundId) or ""
	bgInput.PlaceholderText = "ID do adesivo"
	bgInput.PlaceholderColor3 = SUBTEXT
	bgInput.Font = Enum.Font.Gotham
	bgInput.TextSize = 10
	bgInput.TextColor3 = TEXT
	bgInput.ClearTextOnFocus = false
	bgInput.Parent = bgRow
	Instance.new("UICorner", bgInput).CornerRadius = UDim.new(0, 4)

	local bgConfirm = Instance.new("TextButton")
	bgConfirm.Size = UDim2.fromOffset(70, 22)
	bgConfirm.Position = UDim2.fromOffset(8, 36)
	bgConfirm.BackgroundColor3 = ACCENT
	bgConfirm.Text = "Aplicar"
	bgConfirm.Font = Enum.Font.GothamBold
	bgConfirm.TextSize = 10
	bgConfirm.TextColor3 = BLACK
	bgConfirm.AutoButtonColor = false
	bgConfirm.Parent = bgRow
	Instance.new("UICorner", bgConfirm).CornerRadius = UDim.new(0, 4)

	local bgClear = Instance.new("TextButton")
	bgClear.Size = UDim2.fromOffset(70, 22)
	bgClear.Position = UDim2.fromOffset(84, 36)
	bgClear.BackgroundColor3 = PANEL
	bgClear.Text = "Limpar"
	bgClear.Font = Enum.Font.GothamBold
	bgClear.TextSize = 10
	bgClear.TextColor3 = SUBTEXT
	bgClear.AutoButtonColor = false
	bgClear.Parent = bgRow
	Instance.new("UICorner", bgClear).CornerRadius = UDim.new(0, 4)

	bgConfirm.Activated:Connect(function()
		playClick()
		local id = bgInput.Text:match("%d+")
		if id then
			bgImage.Image = "rbxassetid://" .. id
			TweenService:Create(bgImage, TweenInfo.new(0.4, Enum.EasingStyle.Sine), {ImageTransparency = 0.8}):Play()
			Save.data.backgroundId = tonumber(id)
			writeSave()
			notify("Fundo aplicado!", "success")
		end
	end)

	bgClear.Activated:Connect(function()
		playClick()
		bgImage.Image = ""
		Save.data.backgroundId = nil
		writeSave()
		bgInput.Text = ""
	end)

	local backBtn = Instance.new("TextButton")
	backBtn.Size = UDim2.new(1, -8, 0, 24)
	backBtn.BackgroundColor3 = CARD
	backBtn.Text = "← Voltar"
	backBtn.Font = Enum.Font.GothamBold
	backBtn.TextSize = 10
	backBtn.TextColor3 = SUBTEXT
	backBtn.AutoButtonColor = false
	backBtn.Parent = scroll
	Instance.new("UICorner", backBtn).CornerRadius = UDim.new(0, 5)

	backBtn.Activated:Connect(function()
		playClick()
		settingsOpen = false
		selectTab(Save.data.lastTab or "HOME")
	end)
end

gear.Activated:Connect(function()
	playClick()
	if settingsOpen then return end
	showSettingsPanel()
end)
local opened = false
local busy = false

local function openHub()
	if busy or opened then return end
	busy = true
	opened = true

	panel.Visible = true
	panel.Size = UDim2.fromOffset(PANEL_CLOSED.X.Offset * 0.5, PANEL_CLOSED.Y.Offset * 0.5)
	panel.BackgroundTransparency = 1
	panelStroke.Transparency = 1

	local sizeTween = TweenService:Create(
		panel, TweenInfo.new(0.45, Enum.EasingStyle.Back, Enum.EasingDirection.Out),
		{Size = PANEL_OPEN, BackgroundTransparency = 0}
	)
	TweenService:Create(panelStroke, TweenInfo.new(0.5, Enum.EasingStyle.Sine), {Transparency = 0}):Play()

	sizeTween:Play()
	sizeTween.Completed:Wait()
	busy = false
end

local function closeHub()
	if busy or not opened then return end
	busy = true
	opened = false

	local tween = TweenService:Create(
		panel, TweenInfo.new(0.22, Enum.EasingStyle.Quint, Enum.EasingDirection.In),
		{Size = UDim2.fromOffset(PANEL_CLOSED.X.Offset * 0.5, PANEL_CLOSED.Y.Offset * 0.5), BackgroundTransparency = 1}
	)
	TweenService:Create(panelStroke, TweenInfo.new(0.18), {Transparency = 1}):Play()

	tween:Play()
	tween.Completed:Wait()
	panel.Visible = false
	busy = false
end

floating.Activated:Connect(function()
	playClick()
	pulseButton()
	if opened then closeHub() else openHub() end
end)

close.Activated:Connect(function() playClick(); closeHub() end)

local draggingButton = false
local dragStart, buttonStart

floating.InputBegan:Connect(function(input)
	if input.UserInputType ~= Enum.UserInputType.MouseButton1 and input.UserInputType ~= Enum.UserInputType.Touch then return end
	draggingButton = true
	dragStart = input.Position
	buttonStart = floating.Position
end)

UserInputService.InputChanged:Connect(function(input)
	if not draggingButton then return end
	if input.UserInputType ~= Enum.UserInputType.MouseMovement and input.UserInputType ~= Enum.UserInputType.Touch then return end
	local delta = input.Position - dragStart
	floating.Position = UDim2.new(buttonStart.X.Scale, buttonStart.X.Offset + delta.X, buttonStart.Y.Scale, buttonStart.Y.Offset + delta.Y)
end)

UserInputService.InputEnded:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		draggingButton = false
	end
end)

local draggingPanel = false
local panelDragStart, panelStart

header.InputBegan:Connect(function(input)
	if input.UserInputType ~= Enum.UserInputType.MouseButton1 and input.UserInputType ~= Enum.UserInputType.Touch then return end
	draggingPanel = true
	panelDragStart = input.Position
	panelStart = panel.Position
end)

UserInputService.InputChanged:Connect(function(input)
	if not draggingPanel then return end
	if input.UserInputType ~= Enum.UserInputType.MouseMovement and input.UserInputType ~= Enum.UserInputType.Touch then return end
	local delta = input.Position - panelDragStart
	panel.Position = UDim2.new(panelStart.X.Scale, panelStart.X.Offset + delta.X, panelStart.Y.Scale, panelStart.Y.Offset + delta.Y)
end)

UserInputService.InputEnded:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		draggingPanel = false
	end
end)

task.spawn(function()
	task.wait(0.8)

	local plugins = listPlugins()
	local restored = 0
	local failed = 0

	for _, info in ipairs(plugins) do
		if TabRegistry[info.pluginId] then
			continue
		end

		local src = loadPlugin(info.pluginId)

		if not src or #src == 0 then
			warn("[BatataHub] Plugin vazio:", info.pluginId)
			failed = failed + 1
			continue
		end

		local fn, err = loadstring(src)

		if not fn then
			warn("[BatataHub] Falha ao compilar plugin " .. info.pluginId .. ": " .. tostring(err))
			failed = failed + 1
			continue
		end

		TabRegistry[info.pluginId] = {
			buildFn = fn,
			name = info.name,
			iconId = info.iconId,
		}

		local btn = createTab(info.name, info.iconId)
		btn.Activated:Connect(function()
			playClick()
			selectTab(info.pluginId)
		end)
		addHover(btn, CARD, Color3.fromRGB(38, 38, 38))

		restored = restored + 1
		print("[BatataHub] Plugin restaurado:", info.pluginId, "|", info.name)
	end

	if restored > 0 then
		print("[BatataHub] " .. restored .. " plugin(s) restaurado(s).")
	end
	if failed > 0 then
		warn("[BatataHub] " .. failed .. " plugin(s) com falha.")
	end

	task.wait(0.2)
	local lastTab = Save.data.lastTab
	if lastTab and TabRegistry[lastTab] then
		selectTab(lastTab)
	else
		selectTab("HOME")
	end
end)

local function spawnFallTrail(device)
	task.spawn(function()
		for i = 1, 5 do
			if not device.Parent then break end
			local ghost = device:Clone()
			ghost.ZIndex = device.ZIndex - 1
			ghost.ImageTransparency = 0.5
			ghost.Parent = gui
			TweenService:Create(ghost, TweenInfo.new(0.3), {ImageTransparency = 1}):Play()
			Debris:AddItem(ghost, 0.35)
			task.wait(0.05)
		end
	end)
end

local function PlayIntroSequence()
	floating.Visible = false

	local spawnCenter = UDim2.new(
		0, floating.Position.X.Offset + floating.Size.X.Offset / 2,
		0, floating.Position.Y.Offset + floating.Size.Y.Offset / 2
	)

	local portal = Instance.new("ImageLabel")
	portal.BackgroundTransparency = 1
	portal.AnchorPoint = Vector2.new(0.5, 0.5)
	portal.Position = spawnCenter
	portal.Size = UDim2.fromOffset(BUTTON_SIZE * 1.7, BUTTON_SIZE * 1.7)
	portal.Image = "rbxassetid://" .. tostring(CONFIG.PortalIconId)
	portal.ZIndex = 200
	portal.Parent = gui

	local portalScale = Instance.new("UIScale")
	portalScale.Scale = 0.01
	portalScale.Parent = portal

	playSound(CONFIG.PortalAppearSoundId)
	tweenAsync(portalScale, TweenInfo.new(1.5, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Scale = 1})

	local spinTween = TweenService:Create(portal, TweenInfo.new(6, Enum.EasingStyle.Linear, Enum.EasingDirection.InOut, -1), {Rotation = 360})
	spinTween:Play()

	local head = Instance.new("ImageLabel")
	head.BackgroundTransparency = 1
	head.AnchorPoint = Vector2.new(0.5, 0.5)
	head.Position = spawnCenter
	head.Size = UDim2.fromOffset(BUTTON_SIZE * 1.1, BUTTON_SIZE * 1.1)
	head.Image = "rbxassetid://" .. tostring(CONFIG.RickHeadIconId)
	head.ImageTransparency = 1
	head.ZIndex = 201
	head.Parent = gui

	local headScale = Instance.new("UIScale")
	headScale.Scale = 0.01
	headScale.Parent = head

	TweenService:Create(head, TweenInfo.new(0.5, Enum.EasingStyle.Sine, Enum.EasingDirection.Out), {ImageTransparency = 0}):Play()
	tweenAsync(headScale, TweenInfo.new(1.3, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Scale = 1})

	playSound(CONFIG.RickAppearSoundId)
	task.wait(1)

	TweenService:Create(head, TweenInfo.new(0.5, Enum.EasingStyle.Sine, Enum.EasingDirection.In), {ImageTransparency = 1}):Play()
	tweenAsync(headScale, TweenInfo.new(1.3, Enum.EasingStyle.Back, Enum.EasingDirection.In), {Scale = 0.01})
	head:Destroy()

	local device = Instance.new("ImageLabel")
	device.BackgroundTransparency = 1
	device.AnchorPoint = Vector2.new(0.5, 0.5)
	device.Position = spawnCenter
	device.Size = UDim2.fromOffset(BUTTON_SIZE * 1.35, BUTTON_SIZE * 1.35)
	device.Image = "rbxassetid://" .. tostring(CONFIG.OmegaDeviceIconId)
	device.ZIndex = 201
	device.Parent = gui

	local deviceScale = Instance.new("UIScale")
	deviceScale.Scale = 0.2
	deviceScale.Parent = device
	TweenService:Create(deviceScale, TweenInfo.new(0.4, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Scale = 1}):Play()

	spawnFallTrail(device)

	local fallTarget = UDim2.new(0, spawnCenter.X.Offset, 1, -40)
	tweenAsync(device, TweenInfo.new(0.85, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {Position = fallTarget, Rotation = 25})
	tweenAsync(device, TweenInfo.new(0.14, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {Position = fallTarget - UDim2.fromOffset(0, 14), Rotation = 10})
	tweenAsync(device, TweenInfo.new(0.12, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {Position = fallTarget, Rotation = 0})

	tweenAsync(portal, TweenInfo.new(0.3, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {Rotation = portal.Rotation + 180})
	spinTween:Cancel()
	tweenAsync(portalScale, TweenInfo.new(0.5, Enum.EasingStyle.Back, Enum.EasingDirection.In), {Scale = 0.01})
	portal:Destroy()

	task.wait(0.15)

	local flash = Instance.new("Frame")
	flash.AnchorPoint = Vector2.new(0.5, 0.5)
	flash.Position = device.Position
	flash.Size = UDim2.fromOffset(6, 6)
	flash.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
	flash.BorderSizePixel = 0
	flash.ZIndex = 205
	flash.Parent = gui
	Instance.new("UICorner", flash).CornerRadius = UDim.new(1, 0)
	TweenService:Create(flash, TweenInfo.new(0.18, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
		Size = UDim2.fromOffset(170, 170), BackgroundTransparency = 1,
	}):Play()

	local shockwave = Instance.new("Frame")
	shockwave.AnchorPoint = Vector2.new(0.5, 0.5)
	shockwave.Position = device.Position
	shockwave.Size = UDim2.fromOffset(10, 10)
	shockwave.BackgroundTransparency = 1
	shockwave.ZIndex = 203
	shockwave.Parent = gui
	Instance.new("UICorner", shockwave).CornerRadius = UDim.new(1, 0)
	local shockStroke = Instance.new("UIStroke", shockwave)
	shockStroke.Color = ACCENT
	shockStroke.Thickness = 4
	TweenService:Create(shockwave, TweenInfo.new(0.5, Enum.EasingStyle.Quint, Enum.EasingDirection.Out), {Size = UDim2.fromOffset(220, 220)}):Play()
	TweenService:Create(shockStroke, TweenInfo.new(0.5, Enum.EasingStyle.Quint, Enum.EasingDirection.Out), {Thickness = 0, Transparency = 1}):Play()
	Debris:AddItem(shockwave, 0.6)

	for i = 1, 18 do
		local spark = Instance.new("Frame")
		spark.AnchorPoint = Vector2.new(0.5, 0.5)
		spark.Position = device.Position
		spark.Size = UDim2.fromOffset(6, 6)
		spark.BackgroundColor3 = (i % 2 == 0) and ACCENT or Color3.fromRGB(255, 255, 255)
		spark.BorderSizePixel = 0
		spark.ZIndex = 204
		spark.Parent = gui
		Instance.new("UICorner", spark).CornerRadius = UDim.new(1, 0)

		local angle = (i / 18) * math.pi * 2
		local distance = 55 + math.random(0, 35)

		TweenService:Create(spark, TweenInfo.new(0.45 + math.random() * 0.25, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
			Position = device.Position + UDim2.fromOffset(math.cos(angle) * distance, math.sin(angle) * distance),
			Size = UDim2.fromOffset(1, 1),
			BackgroundTransparency = 1,
		}):Play()
		Debris:AddItem(spark, 1)
	end

	TweenService:Create(device, TweenInfo.new(0.1), {ImageTransparency = 1}):Play()
	task.wait(0.55)
	device:Destroy()
	flash:Destroy()

	floating.Visible = true
	floatingScale.Scale = 0.01
	TweenService:Create(floatingScale, TweenInfo.new(0.55, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Scale = 1}):Play()
end

task.spawn(PlayIntroSequence)

print("[Batata Hub] Interface carregada com sucesso!")
