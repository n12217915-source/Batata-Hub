# Batata-Hub
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

local old = playerGui:FindFirstChild("BatataHub")
if old then old:Destroy() end

local oldApi = ReplicatedStorage:FindFirstChild("BatataHub_RegisterTab")
if oldApi then oldApi:Destroy() end

-- 🔧 PONTO G
local ACCENT = Color3.fromRGB(255, 200, 20)
local ACCENT_DARK = Color3.fromRGB(150, 105, 0)
local BLACK = Color3.fromRGB(10, 10, 10)
local PANEL = Color3.fromRGB(18, 18, 18)
local CARD = Color3.fromRGB(25, 25, 25)
local TEXT = Color3.fromRGB(245, 245, 245)
local SUBTEXT = Color3.fromRGB(150, 150, 150)

local PANEL_CLOSED = UDim2.fromOffset(308, 198)
local PANEL_OPEN   = UDim2.fromOffset(341, 220)

local CONFIG = {
	ExternalPassword = "Batata001", -- senha que Scripts externos usam pra registrar abas
}

local gui = Instance.new("ScreenGui")
gui.Name = "BatataHub"
gui.ResetOnSpawn = false
gui.IgnoreGuiInset = true
gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
gui.Parent = playerGui

local floating = Instance.new("TextButton")
floating.Name = "BatataButton"
floating.Size = UDim2.fromOffset(32, 32)
floating.Position = UDim2.new(0, 14, 0.5, -16)
floating.AnchorPoint = Vector2.new(0, 0)
floating.BackgroundColor3 = PANEL
floating.Text = "🥔"
floating.TextSize = 15
floating.Font = Enum.Font.GothamBold
floating.TextColor3 = TEXT
floating.AutoButtonColor = false
floating.ZIndex = 100
floating.Parent = gui

local floatingCorner = Instance.new("UICorner")
floatingCorner.CornerRadius = UDim.new(1, 0)
floatingCorner.Parent = floating

local floatingGradient = Instance.new("UIGradient")
floatingGradient.Color = ColorSequence.new(ACCENT_DARK, PANEL)
floatingGradient.Rotation = 90
floatingGradient.Parent = floating

local floatingStroke = Instance.new("UIStroke")
floatingStroke.Color = ACCENT
floatingStroke.Thickness = 1.5
floatingStroke.Parent = floating

local floatingScale = Instance.new("UIScale")
floatingScale.Scale = 0.01
floatingScale.Parent = floating

TweenService:Create(
	floatingScale,
	TweenInfo.new(0.5, Enum.EasingStyle.Back, Enum.EasingDirection.Out),
	{Scale = 1}
):Play()

local function pulseButton()
	local down = TweenService:Create(floatingScale, TweenInfo.new(0.08, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {Scale = 0.85})
	local up = TweenService:Create(floatingScale, TweenInfo.new(0.12, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Scale = 1})
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
panel.Visible = false
panel.ZIndex = 10
panel.Parent = gui

local panelCorner = Instance.new("UICorner")
panelCorner.CornerRadius = UDim.new(0, 8)
panelCorner.Parent = panel

local panelStroke = Instance.new("UIStroke")
panelStroke.Color = ACCENT
panelStroke.Thickness = 1.2
panelStroke.Transparency = 1
panelStroke.Parent = panel

local header = Instance.new("Frame")
header.Name = "Header"
header.Size = UDim2.new(1, 0, 0, 34)
header.BackgroundColor3 = PANEL
header.BorderSizePixel = 0
header.ZIndex = 11
header.Parent = panel

local headerCorner = Instance.new("UICorner")
headerCorner.CornerRadius = UDim.new(0, 8)
headerCorner.Parent = header

local headerDivider = Instance.new("Frame")
headerDivider.Size = UDim2.new(1, 0, 0, 1)
headerDivider.Position = UDim2.new(0, 0, 1, -1)
headerDivider.BackgroundColor3 = ACCENT
headerDivider.BackgroundTransparency = 0.75
headerDivider.BorderSizePixel = 0
headerDivider.ZIndex = 11
headerDivider.Parent = header

local avatar = Instance.new("ImageLabel")
avatar.Size = UDim2.fromOffset(23, 23)
avatar.Position = UDim2.fromOffset(7, 6)
avatar.BackgroundColor3 = CARD
avatar.ZIndex = 12
avatar.Parent = header

local avatarCorner = Instance.new("UICorner")
avatarCorner.CornerRadius = UDim.new(1, 0)
avatarCorner.Parent = avatar

local avatarStroke = Instance.new("UIStroke")
avatarStroke.Color = ACCENT
avatarStroke.Thickness = 1
avatarStroke.Parent = avatar

task.spawn(function()
	local success, image = pcall(function()
		return Players:GetUserThumbnailAsync(
			player.UserId,
			Enum.ThumbnailType.HeadShot,
			Enum.ThumbnailSize.Size100x100
		)
	end)
	if success then avatar.Image = image end
end)

local title = Instance.new("TextLabel")
title.BackgroundTransparency = 1
title.Position = UDim2.fromOffset(36, 4)
title.Size = UDim2.fromOffset(160, 14)
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
username.Size = UDim2.fromOffset(160, 11)
username.Font = Enum.Font.Gotham
username.Text = "@" .. player.Name
username.TextSize = 8
username.TextColor3 = ACCENT
username.TextXAlignment = Enum.TextXAlignment.Left
username.ZIndex = 12
username.Parent = header

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

local closeCorner = Instance.new("UICorner")
closeCorner.CornerRadius = UDim.new(0, 5)
closeCorner.Parent = close

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

local contentCorner = Instance.new("UICorner")
contentCorner.CornerRadius = UDim.new(0, 6)
contentCorner.Parent = content
local activePage = nil
local tabButtons = {}
local TabRegistry = {} -- Name -> function(container, ctx)

local ctx = {
	player = player,
	config = CONFIG,
	colors = {
		ACCENT = ACCENT, ACCENT_DARK = ACCENT_DARK, BLACK = BLACK,
		PANEL = PANEL, CARD = CARD, TEXT = TEXT, SUBTEXT = SUBTEXT,
	},
}

local function clearContent()
	if activePage then
		local dead = activePage
		activePage = nil
		local fade = TweenService:Create(dead, TweenInfo.new(0.1), {GroupTransparency = 1})
		fade:Play()
		fade.Completed:Connect(function() dead:Destroy() end)
	end
end

local function createPage()
	clearContent()
	local page = Instance.new("CanvasGroup")
	page.BackgroundTransparency = 1
	page.Size = UDim2.fromScale(1, 1)
	page.GroupTransparency = 1
	page.ZIndex = 11
	page.Parent = content
	activePage = page
	TweenService:Create(page, TweenInfo.new(0.16), {GroupTransparency = 0}):Play()
	return page
end

local function createTab(name)
	local button = Instance.new("TextButton")
	button.Size = UDim2.fromOffset(62, 22)
	button.BackgroundColor3 = CARD
	button.Text = name
	button.TextSize = 9
	button.Font = Enum.Font.GothamBold
	button.TextColor3 = SUBTEXT
	button.AutoButtonColor = false
	button.ZIndex = 12
	button.Parent = tabs

	local corner = Instance.new("UICorner")
	corner.CornerRadius = UDim.new(0, 5)
	corner.Parent = button

	tabButtons[name] = button
	return button
end

local function selectTab(name)
	local buildFn = TabRegistry[name]
	if not buildFn then
		warn("[BatataHub] Aba não encontrada:", name)
		return
	end

	for tabName, button in pairs(tabButtons) do
		if tabName == name then
			TweenService:Create(button, TweenInfo.new(0.15), {BackgroundColor3 = ACCENT, TextColor3 = BLACK}):Play()
		else
			TweenService:Create(button, TweenInfo.new(0.15), {BackgroundColor3 = CARD, TextColor3 = SUBTEXT}):Play()
		end
	end

	local page = createPage()
	buildFn(page, ctx)
end
TabRegistry["HOME"] = function(page, ctx)
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
	friendTitle.Text = "AMIGOS"
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
end

TabRegistry["PLAYER"] = function(page, ctx)
	local title = Instance.new("TextLabel")
	title.BackgroundTransparency = 1
	title.Position = UDim2.fromOffset(9, 8)
	title.Size = UDim2.new(1, -18, 0, 17)
	title.Font = Enum.Font.GothamBlack
	title.Text = "PLAYER"
	title.TextSize = 14
	title.TextColor3 = ctx.colors.TEXT
	title.TextXAlignment = Enum.TextXAlignment.Left
	title.Parent = page

	local card = Instance.new("Frame")
	card.Size = UDim2.new(1, -18, 0, 55)
	card.Position = UDim2.fromOffset(9, 32)
	card.BackgroundColor3 = ctx.colors.CARD
	card.Parent = page
	Instance.new("UICorner", card).CornerRadius = UDim.new(0, 5)

	local info = Instance.new("TextLabel")
	info.BackgroundTransparency = 1
	info.Position = UDim2.fromOffset(8, 7)
	info.Size = UDim2.new(1, -15, 1, -13)
	info.Font = Enum.Font.Gotham
	info.TextSize = 10
	info.TextColor3 = ctx.colors.TEXT
	info.TextXAlignment = Enum.TextXAlignment.Left
	info.TextYAlignment = Enum.TextYAlignment.Top
	info.Text = "Nome: " .. ctx.player.Name
		.. "\nDisplayName: " .. ctx.player.DisplayName
		.. "\nUserId: " .. ctx.player.UserId
	info.Parent = card
end

local homeButton = createTab("HOME")
homeButton.Activated:Connect(function() selectTab("HOME") end)

local playerButton = createTab("PLAYER")
playerButton.Activated:Connect(function() selectTab("PLAYER") end)

selectTab("HOME")

-- =========================================================
-- SISTEMA DE ABAS EXTERNAS V2
-- =========================================================

local function RegisterExternalTab(password, tabData)
	if password ~= CONFIG.ExternalPassword then
		warn("[BatataHub] Senha inválida para registrar aba externa.")
		return false, "Senha inválida"
	end

	if type(tabData) ~= "table"
		or type(tabData.Name) ~= "string"
		or type(tabData.BuildContent) ~= "function" then
		warn("[BatataHub] Dados de aba externa inválidos.")
		return false, "Dados inválidos"
	end

	if TabRegistry[tabData.Name] then
		warn("[BatataHub] Já existe uma aba com esse nome:", tabData.Name)
		return false, "Aba já existe"
	end

	TabRegistry[tabData.Name] = tabData.BuildContent

	local btn = createTab(tabData.Name)
	btn.Activated:Connect(function() selectTab(tabData.Name) end)

	print("[BatataHub] Aba externa registrada:", tabData.Name)
	return true
end

local api = Instance.new("BindableFunction")
api.Name = "BatataHub_RegisterTab"
api.Parent = ReplicatedStorage

api.OnInvoke = function(password, tabData)
	return RegisterExternalTab(password, tabData)
end
local opened = false
local busy = false

local function openHub()
	if busy or opened then return end
	busy = true
	opened = true

	panel.Visible = true
	panel.Size = UDim2.fromOffset(PANEL_CLOSED.X.Offset * 0.7, PANEL_CLOSED.Y.Offset * 0.7)
	panel.BackgroundTransparency = 1
	panelStroke.Transparency = 1

	local sizeTween = TweenService:Create(
		panel,
		TweenInfo.new(0.32, Enum.EasingStyle.Back, Enum.EasingDirection.Out),
		{Size = PANEL_OPEN, BackgroundTransparency = 0}
	)
	local strokeTween = TweenService:Create(panelStroke, TweenInfo.new(0.4), {Transparency = 0})

	sizeTween:Play()
	strokeTween:Play()
	sizeTween.Completed:Wait()

	busy = false
end

local function closeHub()
	if busy or not opened then return end
	busy = true
	opened = false

	local tween = TweenService:Create(
		panel,
		TweenInfo.new(0.16, Enum.EasingStyle.Quad, Enum.EasingDirection.In),
		{Size = UDim2.fromOffset(PANEL_CLOSED.X.Offset * 0.7, PANEL_CLOSED.Y.Offset * 0.7), BackgroundTransparency = 1}
	)
	TweenService:Create(panelStroke, TweenInfo.new(0.14), {Transparency = 1}):Play()

	tween:Play()
	tween.Completed:Wait()

	panel.Visible = false
	busy = false
end

floating.Activated:Connect(function()
	pulseButton()
	if opened then
		closeHub()
	else
		openHub()
	end
end)

close.Activated:Connect(function()
	closeHub()
end)

local draggingButton = false
local dragStart
local buttonStart

floating.InputBegan:Connect(function(input)
	if input.UserInputType ~= Enum.UserInputType.MouseButton1
		and input.UserInputType ~= Enum.UserInputType.Touch then
		return
	end
	draggingButton = true
	dragStart = input.Position
	buttonStart = floating.Position
end)

UserInputService.InputChanged:Connect(function(input)
	if not draggingButton then return end
	if input.UserInputType ~= Enum.UserInputType.MouseMovement
		and input.UserInputType ~= Enum.UserInputType.Touch then
		return
	end
	local delta = input.Position - dragStart
	floating.Position = UDim2.new(
		buttonStart.X.Scale, buttonStart.X.Offset + delta.X,
		buttonStart.Y.Scale, buttonStart.Y.Offset + delta.Y
	)
end)

UserInputService.InputEnded:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1
		or input.UserInputType == Enum.UserInputType.Touch then
		draggingButton = false
	end
end)

local draggingPanel = false
local panelDragStart
local panelStart

header.InputBegan:Connect(function(input)
	if input.UserInputType ~= Enum.UserInputType.MouseButton1
		and input.UserInputType ~= Enum.UserInputType.Touch then
		return
	end
	draggingPanel = true
	panelDragStart = input.Position
	panelStart = panel.Position
end)

UserInputService.InputChanged:Connect(function(input)
	if not draggingPanel then return end
	if input.UserInputType ~= Enum.UserInputType.MouseMovement
		and input.UserInputType ~= Enum.UserInputType.Touch then
		return
	end
	local delta = input.Position - panelDragStart
	panel.Position = UDim2.new(
		panelStart.X.Scale, panelStart.X.Offset + delta.X,
		panelStart.Y.Scale, panelStart.Y.Offset + delta.Y
	)
end)

UserInputService.InputEnded:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1
		or input.UserInputType == Enum.UserInputType.Touch then
		draggingPanel = false
	end
end)

print("[Batata Hub] Interface V2 carregada com sucesso!")
