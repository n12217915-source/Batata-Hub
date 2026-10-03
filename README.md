local Players            = game:GetService("Players")
local UserInputService   = game:GetService("UserInputService")
local TweenService       = game:GetService("TweenService")
local ReplicatedStorage  = game:GetService("ReplicatedStorage")
local MarketplaceService = game:GetService("MarketplaceService")
local SoundService       = game:GetService("SoundService")
local Debris             = game:GetService("Debris")
local RunService         = game:GetService("RunService")

local player    = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

local old = playerGui:FindFirstChild("BatataHub")
if old then old:Destroy() end
local oldApi = ReplicatedStorage:FindFirstChild("BatataHub_RegisterTab")
if oldApi then oldApi:Destroy() end

local ACCENT      = Color3.fromRGB(255, 200, 20)
local ACCENT_DARK = Color3.fromRGB(150, 105, 0)
local BLACK       = Color3.fromRGB(10, 10, 10)
local PANEL       = Color3.fromRGB(18, 18, 18)
local CARD        = Color3.fromRGB(25, 25, 25)
local TEXT        = Color3.fromRGB(245, 245, 245)
local SUBTEXT     = Color3.fromRGB(150, 150, 150)

local PANEL_CLOSED = UDim2.fromOffset(308, 198)
local PANEL_OPEN   = UDim2.fromOffset(341, 220)
local BUTTON_SIZE  = 56

local CONFIG = {
	ExternalPassword = "Batata001",
	VIPGamePassId    = 0,
	ClickSoundId     = "rbxassetid://86847045401690",
	HomeIconId          = 117525739056427,
	PortalIconId        = 128039132946840,
	RickHeadIconId      = 131775579293831,
	OmegaDeviceIconId   = 96858175598695,
	PortalAppearSoundId = "rbxassetid://71852278135255",
	RickAppearSoundId   = "rbxassetid://134634382299700",
	PanicKey = Enum.KeyCode.RightControl,
	DiscordLink = "https://discord.gg/guySypaQAp",
}

local SaveConfig = (function()
	local FOLDER    = "Batata Central"
	local SUBFOLDER = "Arquivos Secretos"
	local FILENAME  = "BatataHub_Config.json"
	local FILE_PATH = FOLDER .. "/" .. SUBFOLDER .. "/" .. FILENAME

	local data, defaults = {}, {}
	local saveQueued = false
	local FS_OK     = (type(writefile) == "function" and type(readfile) == "function" and type(isfile) == "function")
	local FOLDER_OK = (type(makefolder) == "function")

	if FS_OK and FOLDER_OK then
		pcall(function()
			pcall(makefolder, FOLDER)
			pcall(makefolder, FOLDER .. "/" .. SUBFOLDER)
		end)
	end

	local function jsonEncode(t)
		local function esc(s)
			return s:gsub('[%c"\\]', function(c)
				if c == '"' then return '\\"' end
				if c == '\\' then return '\\\\' end
				if c == '\n' then return '\\n' end
				if c == '\r' then return '\\r' end
				if c == '\t' then return '\\t' end
				return string.format('\\u%04x', c:byte())
			end)
		end
		local function enc(v)
			local ty = type(v)
			if ty == "nil" then return "null"
			elseif ty == "boolean" then return tostring(v)
			elseif ty == "number" then return tostring(v)
			elseif ty == "string" then return '"'..esc(v)..'"'
			elseif ty == "table" then
				local isArray = #v > 0
				local parts = {}
				if isArray then
					for _, x in ipairs(v) do parts[#parts+1] = enc(x) end
					return "["..table.concat(parts, ",").."]"
				else
					for k, x in pairs(v) do
						parts[#parts+1] = '"'..esc(tostring(k))..'":'..enc(x)
					end
					return "{"..table.concat(parts, ",").."}"
				end
			end
			return "null"
		end
		return enc(t)
	end

	local function jsonDecode(s)
		local pos = 1
		local function skip() while pos <= #s and s:sub(pos,pos):match("%s") do pos = pos + 1 end end
		local function parse()
			skip()
			local c = s:sub(pos,pos)
			if c == '"' then
				pos = pos + 1
				local out = {}
				while pos <= #s do
					local ch = s:sub(pos,pos)
					if ch == '\\' then
						local nxt = s:sub(pos+1,pos+1)
						if nxt == 'n' then out[#out+1] = '\n'
						elseif nxt == 't' then out[#out+1] = '\t'
						elseif nxt == 'r' then out[#out+1] = '\r'
						else out[#out+1] = nxt end
						pos = pos + 2
					elseif ch == '"' then pos = pos + 1 break
					else out[#out+1] = ch pos = pos + 1 end
				end
				return table.concat(out)
			elseif c == '{' then
				pos = pos + 1
				local obj = {}
				skip()
				if s:sub(pos,pos) == '}' then pos = pos + 1 return obj end
				while true do
					skip()
					local k = parse()
					skip()
					pos = pos + 1
					obj[k] = parse()
					skip()
					local n = s:sub(pos,pos)
					pos = pos + 1
					if n == '}' then break end
				end
				return obj
			elseif c == '[' then
				pos = pos + 1
				local arr = {}
				skip()
				if s:sub(pos,pos) == ']' then pos = pos + 1 return arr end
				while true do
					arr[#arr+1] = parse()
					skip()
					local n = s:sub(pos,pos)
					pos = pos + 1
					if n == ']' then break end
				end
				return arr
			elseif c == 't' then pos = pos + 4 return true
			elseif c == 'f' then pos = pos + 5 return false
			elseif c == 'n' then pos = pos + 4 return nil
			else
				local num = s:match("^-?%d+%.?%d*[eE]?[-+]?%d*", pos)
				if not num then return nil end
				pos = pos + #num
				return tonumber(num)
			end
		end
		local ok, res = pcall(parse)
		return ok and res or {}
	end

	local function load()
		if not FS_OK then return end
		local exists = false
		pcall(function() exists = isfile(FILE_PATH) end)
		if not exists then return end
		local ok, content = pcall(readfile, FILE_PATH)
		if not ok or not content or content == "" then return end
		local ok2, parsed = pcall(jsonDecode, content)
		if ok2 and type(parsed) == "table" then
			for k, v in pairs(parsed) do data[k] = v end
		end
	end

	local function save()
		if not FS_OK then return end
		local ok = pcall(writefile, FILE_PATH, jsonEncode(data))
		if not ok and FOLDER_OK then
			pcall(makefolder, FOLDER)
			pcall(makefolder, FOLDER .. "/" .. SUBFOLDER)
			pcall(writefile, FILE_PATH, jsonEncode(data))
		end
	end

	local function queueSave()
		if saveQueued then return end
		saveQueued = true
		task.delay(0.6, function()
			saveQueued = false
			save()
		end)
	end

	load()

	return {
		register = function(key, default)
			defaults[key] = default
			if data[key] == nil then data[key] = default end
			return data[key]
		end,
		get = function(key) return data[key] end,
		set = function(key, value)
			data[key] = value
			queueSave()
		end,
		saveNow = save,
		loadNow = load,
		reset = function()
			data = {}
			for k, v in pairs(defaults) do data[k] = v end
			save()
		end,
		FS_OK = FS_OK,
		PATH  = FILE_PATH,
	}
end)()
local clickSound = Instance.new("Sound")
clickSound.Name    = "BatataClick"
clickSound.SoundId = CONFIG.ClickSoundId
clickSound.Volume  = 0.5
clickSound.Parent  = SoundService

local function playClick() clickSound:Play() end

local function playSound(id, volume)
	if not id or id == "" then return end
	local s = Instance.new("Sound")
	s.SoundId = id
	s.Volume  = volume or 0.7
	s.Parent  = SoundService
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

local function clampToViewport(posOffset, objSize)
	local vp = workspace.CurrentCamera.ViewportSize
	local x = math.clamp(posOffset.X, -objSize.X + 40, vp.X - 40)
	local y = math.clamp(posOffset.Y, -objSize.Y + 40, vp.Y - 40)
	return Vector2.new(x, y)
end

local function isMostlyOffscreen(posOffset, objSize, threshold)
	threshold = threshold or 0.85
	local vp = workspace.CurrentCamera.ViewportSize
	local totalPx = objSize.X * objSize.Y
	if totalPx <= 0 then return false end
	local left   = math.max(posOffset.X, 0)
	local top    = math.max(posOffset.Y, 0)
	local right  = math.min(posOffset.X + objSize.X, vp.X)
	local bottom = math.min(posOffset.Y + objSize.Y, vp.Y)
	local visibleW = math.max(0, right - left)
	local visibleH = math.max(0, bottom - top)
	local visiblePx = visibleW * visibleH
	return (visiblePx / totalPx) <= (1 - threshold)
end

local VirtualUser = game:GetService("VirtualUser")
local AntiAFK = (function()
	local active = true
	local interval = 60

	task.spawn(function()
		while active do
			task.wait(interval)
			pcall(function()
				VirtualUser:CaptureController()
				VirtualUser:ClickButton2(Vector2.new())
			end)
		end
	end)

	return {
		toggle = function(v) active = v end,
		isActive = function() return active end,
		setInterval = function(n) interval = math.clamp(n, 10, 300) end,
	}
end)()

local AntiCheatDetector = (function()
	local KNOWN_INSTANCES = {
		"Rayfield", "OrionLib", "Kavo", "Fluent", "Linoria",
		"WindUI", "Sirius", "FakeKavo", "XenoUI", "Vape",
		"Rspy", "Dex", "SimpleSpy", "InfiniteYield", "IY",
	}
	local SUSPICIOUS_HOOKS = {
		"hookmetamethod", "hookfunction", "getrawmetatable",
		"setreadonly", "getgenv", "getrenv", "getreg",
	}

	local detected = false
	local listeners = {}

	local function report(name)
		if detected then return end
		detected = true
		for _, cb in ipairs(listeners) do
			task.spawn(cb, name)
		end
	end

	local function watchContainer(container)
		container.DescendantAdded:Connect(function(inst)
			local name = inst.Name
			for _, sus in ipairs(KNOWN_INSTANCES) do
				if name:lower():find(sus:lower(), 1, true) then
					report("Instância suspeita: " .. name)
					return
				end
			end
			if inst:IsA("LocalScript") then
				local src = ""
				pcall(function()
					src = inst.Source or ""
				end)
				for _, hk in ipairs(SUSPICIOUS_HOOKS) do
					if src:find(hk, 1, true) then
						report("Hook suspeito: " .. hk .. " em " .. name)
						return
					end
				end
			end
		end)
	end

	pcall(function() watchContainer(playerGui) end)
	pcall(function() watchContainer(game:GetService("CoreGui")) end)

	return {
		isDetected = function() return detected end,
		onDetect = function(cb) table.insert(listeners, cb) end,
	}
end)()

local gui = Instance.new("ScreenGui")
gui.Name            = "BatataHub"
gui.ResetOnSpawn    = false
gui.IgnoreGuiInset  = true
gui.ZIndexBehavior  = Enum.ZIndexBehavior.Sibling
gui.Parent          = playerGui
local notifyContainer = Instance.new("Frame")
notifyContainer.Name = "Notifications"
notifyContainer.AnchorPoint = Vector2.new(1, 1)
notifyContainer.Position    = UDim2.new(1, -14, 1, -14)
notifyContainer.Size        = UDim2.new(0, 210, 1, -28)
notifyContainer.BackgroundTransparency = 1
notifyContainer.ZIndex = 300
notifyContainer.Parent = gui

local notifyLayout = Instance.new("UIListLayout")
notifyLayout.FillDirection       = Enum.FillDirection.Vertical
notifyLayout.VerticalAlignment   = Enum.VerticalAlignment.Bottom
notifyLayout.HorizontalAlignment = Enum.HorizontalAlignment.Right
notifyLayout.Padding             = UDim.new(0, 6)
notifyLayout.SortOrder           = Enum.SortOrder.LayoutOrder
notifyLayout.Parent              = notifyContainer

local function Notify(text, duration)
	duration = math.max(1, math.floor(tonumber(duration) or 3))

	local box = Instance.new("Frame")
	box.Size = UDim2.new(1, 0, 0, 34)
	box.BackgroundColor3 = Color3.fromRGB(0,0,0)
	box.BackgroundTransparency = 1
	box.ClipsDescendants = true
	box.ZIndex = 300
	box.Parent = notifyContainer
	Instance.new("UICorner", box).CornerRadius = UDim.new(0, 4)

	local label = Instance.new("TextLabel")
	label.BackgroundTransparency = 1
	label.Position = UDim2.fromOffset(8, 0)
	label.Size     = UDim2.new(1, -36, 1, 0)
	label.Font     = Enum.Font.Gotham
	label.TextSize = 10
	label.TextColor3 = TEXT
	label.TextXAlignment = Enum.TextXAlignment.Left
	label.TextWrapped = true
	label.TextTransparency = 1
	label.Text = text
	label.ZIndex = 301
	label.Parent = box

	local timerLabel = Instance.new("TextLabel")
	timerLabel.BackgroundTransparency = 1
	timerLabel.Position = UDim2.new(1, -28, 0, 4)
	timerLabel.Size = UDim2.fromOffset(20, 14)
	timerLabel.Font = Enum.Font.GothamBold
	timerLabel.TextSize = 9
	timerLabel.TextColor3 = SUBTEXT
	timerLabel.TextXAlignment = Enum.TextXAlignment.Right
	timerLabel.TextTransparency = 1
	timerLabel.Text = tostring(duration) .. "s"
	timerLabel.ZIndex = 301
	timerLabel.Parent = box

	box.Position = UDim2.new(0, 40, 0, 0)

	TweenService:Create(box, TweenInfo.new(0.28, Enum.EasingStyle.Sine, Enum.EasingDirection.Out), {
		Position = UDim2.new(0, 0, 0, 0),
		BackgroundTransparency = 0.2,
	}):Play()
	TweenService:Create(label, TweenInfo.new(0.24), {TextTransparency = 0}):Play()
	TweenService:Create(timerLabel, TweenInfo.new(0.24), {TextTransparency = 0}):Play()

	task.spawn(function()
		local remaining = duration
		while remaining > 0 do
			task.wait(1)
			remaining -= 1
			timerLabel.Text = tostring(math.max(remaining, 0)) .. "s"
		end
		TweenService:Create(box, TweenInfo.new(0.22, Enum.EasingStyle.Sine, Enum.EasingDirection.In), {
			Position = UDim2.new(0, 40, 0, 0),
			BackgroundTransparency = 1,
		}):Play()
		TweenService:Create(label, TweenInfo.new(0.18), {TextTransparency = 1}):Play()
		TweenService:Create(timerLabel, TweenInfo.new(0.18), {TextTransparency = 1}):Play()
		task.wait(0.28)
		box:Destroy()
	end)
end

local function AskConfirm(text, onYes, onNo, duration)
	duration = duration or 10

	local askGui = Instance.new("ScreenGui")
	askGui.Name = "BatataAsk"
	askGui.ResetOnSpawn = false
	askGui.IgnoreGuiInset = true
	askGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
	askGui.Parent = playerGui

	local box = Instance.new("Frame")
	box.AnchorPoint = Vector2.new(0, 1)
	box.Position    = UDim2.new(0, 14, 1, -14)
	box.Size        = UDim2.fromOffset(240, 68)
	box.BackgroundColor3 = Color3.fromRGB(0,0,0)
	box.BackgroundTransparency = 0.2
	box.ZIndex = 500
	box.Parent = askGui
	Instance.new("UICorner", box).CornerRadius = UDim.new(0, 6)

	local stroke = Instance.new("UIStroke", box)
	stroke.Color = ACCENT
	stroke.Thickness = 1
	stroke.Transparency = 0.5

	local label = Instance.new("TextLabel")
	label.BackgroundTransparency = 1
	label.Position = UDim2.fromOffset(10, 8)
	label.Size     = UDim2.new(1, -20, 0, 28)
	label.Font     = Enum.Font.Gotham
	label.TextSize = 11
	label.TextColor3 = TEXT
	label.TextWrapped = true
	label.TextXAlignment = Enum.TextXAlignment.Left
	label.TextYAlignment = Enum.TextYAlignment.Top
	label.Text = text
	label.ZIndex = 501
	label.Parent = box

	local yes = Instance.new("TextButton")
	yes.Size = UDim2.fromOffset(80, 22)
	yes.Position = UDim2.fromOffset(10, 40)
	yes.BackgroundColor3 = ACCENT
	yes.Text = "Sim"
	yes.Font = Enum.Font.GothamBold
	yes.TextSize = 11
	yes.TextColor3 = BLACK
	yes.AutoButtonColor = false
	yes.ZIndex = 501
	yes.Parent = box
	Instance.new("UICorner", yes).CornerRadius = UDim.new(0, 5)

	local no = Instance.new("TextButton")
	no.Size = UDim2.fromOffset(80, 22)
	no.Position = UDim2.fromOffset(96, 40)
	no.BackgroundColor3 = CARD
	no.Text = "Não"
	no.Font = Enum.Font.GothamBold
	no.TextSize = 11
	no.TextColor3 = TEXT
	no.AutoButtonColor = false
	no.ZIndex = 501
	no.Parent = box
	Instance.new("UICorner", no).CornerRadius = UDim.new(0, 5)

	local bar = Instance.new("Frame")
	bar.Position = UDim2.new(0, 0, 1, -3)
	bar.Size = UDim2.new(1, 0, 0, 3)
	bar.BackgroundColor3 = ACCENT
	bar.BorderSizePixel = 0
	bar.ZIndex = 501
	bar.Parent = box
	Instance.new("UICorner", bar).CornerRadius = UDim.new(1, 0)

	local done = false
	local function finish(result)
		if done then return end
		done = true
		TweenService:Create(box, TweenInfo.new(0.2, Enum.EasingStyle.Sine, Enum.EasingDirection.In), {
			Position = UDim2.new(0, -300, 1, -14),
			BackgroundTransparency = 1,
		}):Play()
		TweenService:Create(label, TweenInfo.new(0.18), {TextTransparency = 1}):Play()
		TweenService:Create(yes, TweenInfo.new(0.18), {BackgroundTransparency = 1, TextTransparency = 1}):Play()
		TweenService:Create(no,  TweenInfo.new(0.18), {BackgroundTransparency = 1, TextTransparency = 1}):Play()
		TweenService:Create(bar, TweenInfo.new(0.18), {BackgroundTransparency = 1}):Play()
		task.wait(0.25)
		askGui:Destroy()
		if result == "yes" and onYes then onYes() end
		if result == "no"  and onNo  then onNo()  end
	end

	yes.Activated:Connect(function() playClick(); finish("yes") end)
	no.Activated:Connect(function()  playClick(); finish("no")  end)

	task.spawn(function()
		local tw = TweenService:Create(bar, TweenInfo.new(duration, Enum.EasingStyle.Linear), {Size = UDim2.new(0, 0, 0, 3)})
		tw:Play()
		tw.Completed:Wait()
		finish("no")
	end)
end
local floating = Instance.new("ImageButton")
floating.Name = "BatataButton"
floating.Size = UDim2.fromOffset(BUTTON_SIZE, BUTTON_SIZE)

local savedBtnX = SaveConfig.register("ui.btnX", 14)
local savedBtnY = SaveConfig.register("ui.btnY", -28)
floating.Position = UDim2.new(0, savedBtnX, 0.5, savedBtnY)
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
	local up   = TweenService:Create(floatingScale, TweenInfo.new(0.28, Enum.EasingStyle.Elastic, Enum.EasingDirection.Out), {Scale = 1})
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
	ColorSequenceKeypoint.new(0, Color3.fromRGB(16,16,16)),
	ColorSequenceKeypoint.new(1, Color3.fromRGB(8,8,8)),
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
bgImage.Size = UDim2.fromScale(1,1)
bgImage.ImageTransparency = 0.8
bgImage.ScaleType = Enum.ScaleType.Crop
bgImage.ZIndex = 1
bgImage.Image = ""
bgImage.Parent = panel

task.spawn(function()
	local bg = SaveConfig.get("ui.bg")
	if bg then
		bgImage.Image = "rbxassetid://" .. tostring(bg)
		bgImage.ImageTransparency = 0.8
	end
end)

local header = Instance.new("Frame")
header.Name = "Header"
header.Size = UDim2.new(1,0,0,34)
header.BackgroundColor3 = PANEL
header.BorderSizePixel = 0
header.ZIndex = 11
header.Parent = panel
Instance.new("UICorner", header).CornerRadius = UDim.new(0, 8)

local headerDivider = Instance.new("Frame")
headerDivider.Size = UDim2.new(1,0,0,1)
headerDivider.Position = UDim2.new(0,0,1,-1)
headerDivider.BackgroundColor3 = ACCENT
headerDivider.BackgroundTransparency = 0.7
headerDivider.BorderSizePixel = 0
headerDivider.ZIndex = 11
headerDivider.Parent = header

local avatar = Instance.new("ImageLabel")
avatar.Size = UDim2.fromOffset(23,23)
avatar.Position = UDim2.fromOffset(7,6)
avatar.BackgroundColor3 = CARD
avatar.ZIndex = 12
avatar.Parent = header
Instance.new("UICorner", avatar).CornerRadius = UDim.new(1,0)

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
title.Position = UDim2.fromOffset(36,4)
title.Size = UDim2.fromOffset(140,14)
title.Font = Enum.Font.GothamBlack
title.Text = "BATATA HUB"
title.TextSize = 12
title.TextColor3 = TEXT
title.TextXAlignment = Enum.TextXAlignment.Left
title.ZIndex = 12
title.Parent = header

local username = Instance.new("TextLabel")
username.BackgroundTransparency = 1
username.Position = UDim2.fromOffset(36,18)
username.Size = UDim2.fromOffset(120,11)
username.Font = Enum.Font.Gotham
username.Text = "@" .. player.Name
username.TextSize = 8
username.TextColor3 = ACCENT
username.TextXAlignment = Enum.TextXAlignment.Left
username.ZIndex = 12
username.Parent = header
local saveBtn = Instance.new("TextButton")
saveBtn.Size = UDim2.fromOffset(20,20)
saveBtn.Position = UDim2.new(1, -99, 0, 7)
saveBtn.BackgroundColor3 = CARD
saveBtn.Text = "💾"
saveBtn.TextSize = 11
saveBtn.Font = Enum.Font.GothamBold
saveBtn.TextColor3 = SUBTEXT
saveBtn.AutoButtonColor = false
saveBtn.ZIndex = 13
saveBtn.Parent = header
Instance.new("UICorner", saveBtn).CornerRadius = UDim.new(0,5)
addHover(saveBtn, CARD, Color3.fromRGB(38,38,38))

local resetBtn = Instance.new("TextButton")
resetBtn.Size = UDim2.fromOffset(20,20)
resetBtn.Position = UDim2.new(1, -75, 0, 7)
resetBtn.BackgroundColor3 = CARD
resetBtn.Text = "⟳"
resetBtn.TextSize = 12
resetBtn.Font = Enum.Font.GothamBold
resetBtn.TextColor3 = SUBTEXT
resetBtn.AutoButtonColor = false
resetBtn.ZIndex = 13
resetBtn.Parent = header
Instance.new("UICorner", resetBtn).CornerRadius = UDim.new(0,5)
addHover(resetBtn, CARD, Color3.fromRGB(38,38,38))

local gear = Instance.new("TextButton")
gear.Size = UDim2.fromOffset(20,20)
gear.Position = UDim2.new(1, -51, 0, 7)
gear.BackgroundColor3 = CARD
gear.Text = "⚙"
gear.TextSize = 12
gear.Font = Enum.Font.GothamBold
gear.TextColor3 = SUBTEXT
gear.AutoButtonColor = false
gear.ZIndex = 13
gear.Parent = header
Instance.new("UICorner", gear).CornerRadius = UDim.new(0,5)
addHover(gear, CARD, Color3.fromRGB(38,38,38))

local close = Instance.new("TextButton")
close.Size = UDim2.fromOffset(20,20)
close.Position = UDim2.new(1, -27, 0, 7)
close.BackgroundColor3 = CARD
close.Text = "×"
close.TextSize = 15
close.Font = Enum.Font.GothamBold
close.TextColor3 = TEXT
close.AutoButtonColor = false
close.ZIndex = 13
close.Parent = header
Instance.new("UICorner", close).CornerRadius = UDim.new(0,5)
addHover(close, CARD, Color3.fromRGB(60,30,30))

local tabs = Instance.new("Frame")
tabs.Name = "Tabs"
tabs.Size = UDim2.new(1,-14,0,22)
tabs.Position = UDim2.fromOffset(7,40)
tabs.BackgroundTransparency = 1
tabs.ZIndex = 11
tabs.Parent = panel

local tabLayout = Instance.new("UIListLayout")
tabLayout.FillDirection = Enum.FillDirection.Horizontal
tabLayout.Padding = UDim.new(0,5)
tabLayout.VerticalAlignment = Enum.VerticalAlignment.Center
tabLayout.Parent = tabs

local content = Instance.new("Frame")
content.Name = "Content"
content.Size = UDim2.new(1,-14,1,-68)
content.Position = UDim2.fromOffset(7,64)
content.BackgroundColor3 = PANEL
content.BorderSizePixel = 0
content.ZIndex = 11
content.Parent = panel
Instance.new("UICorner", content).CornerRadius = UDim.new(0,6)

local activePage = nil
local tabButtons = {}
local TabRegistry = {}

local ctx = {
	player = player,
	config = CONFIG,
	colors = {
		ACCENT = ACCENT, ACCENT_DARK = ACCENT_DARK,
		BLACK = BLACK, PANEL = PANEL, CARD = CARD,
		TEXT = TEXT, SUBTEXT = SUBTEXT,
	},
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

	TweenService:Create(page, TweenInfo.new(0.35, Enum.EasingStyle.Quint, Enum.EasingDirection.Out), {
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
		icon.Size = UDim2.fromOffset(12,12)
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

local function selectTab(name)
	local buildFn = TabRegistry[name]
	if not buildFn then return end
	for tabName, data in pairs(tabButtons) do
		if tabName == name then
			TweenService:Create(data.Button, TweenInfo.new(0.18, Enum.EasingStyle.Sine), {BackgroundColor3 = ACCENT}):Play()
			TweenService:Create(data.Label,  TweenInfo.new(0.18, Enum.EasingStyle.Sine), {TextColor3 = BLACK}):Play()
		else
			TweenService:Create(data.Button, TweenInfo.new(0.18, Enum.EasingStyle.Sine), {BackgroundColor3 = CARD}):Play()
			TweenService:Create(data.Label,  TweenInfo.new(0.18, Enum.EasingStyle.Sine), {TextColor3 = SUBTEXT}):Play()
		end
	end
	local page = createPage()
	buildFn(page, ctx)
end
local function makeToggle(parent, text, y, initial, onChange)
	local btn = Instance.new("TextButton")
	btn.Position = UDim2.fromOffset(9, y)
	btn.Size = UDim2.new(1, -18, 0, 22)
	btn.BackgroundColor3 = CARD
	btn.Text = ""
	btn.AutoButtonColor = false
	btn.ZIndex = 12
	btn.Parent = parent
	Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 5)

	local lbl = Instance.new("TextLabel")
	lbl.BackgroundTransparency = 1
	lbl.Position = UDim2.fromOffset(8, 0)
	lbl.Size = UDim2.new(1, -50, 1, 0)
	lbl.Font = Enum.Font.Gotham
	lbl.TextSize = 11
	lbl.TextColor3 = TEXT
	lbl.TextXAlignment = Enum.TextXAlignment.Left
	lbl.Text = text
	lbl.ZIndex = 13
	lbl.Parent = btn

	local st = Instance.new("TextLabel")
	st.BackgroundTransparency = 1
	st.Position = UDim2.new(1, -45, 0, 0)
	st.Size = UDim2.fromOffset(40, 22)
	st.Font = Enum.Font.GothamBold
	st.TextSize = 11
	st.TextColor3 = initial and Color3.fromRGB(90,220,120) or Color3.fromRGB(220,90,90)
	st.Text = initial and "ON" or "OFF"
	st.ZIndex = 13
	st.Parent = btn

	btn.MouseButton1Click:Connect(function()
		initial = not initial
		st.Text = initial and "ON" or "OFF"
		st.TextColor3 = initial and Color3.fromRGB(90,220,120) or Color3.fromRGB(220,90,90)
		onChange(initial)
	end)
end

local function makeSlider(parent, text, y, min, max, initial, onChange)
	local frame = Instance.new("Frame")
	frame.Position = UDim2.fromOffset(9, y)
	frame.Size = UDim2.new(1, -18, 0, 30)
	frame.BackgroundTransparency = 1
	frame.ZIndex = 12
	frame.Parent = parent

	local lbl = Instance.new("TextLabel")
	lbl.BackgroundTransparency = 1
	lbl.Size = UDim2.new(1, 0, 0, 14)
	lbl.Font = Enum.Font.Gotham
	lbl.TextSize = 11
	lbl.TextColor3 = SUBTEXT
	lbl.TextXAlignment = Enum.TextXAlignment.Left
	lbl.Text = text .. ": " .. tostring(initial)
	lbl.ZIndex = 13
	lbl.Parent = frame

	local bar = Instance.new("Frame")
	bar.Position = UDim2.fromOffset(0, 18)
	bar.Size = UDim2.new(1, 0, 0, 6)
	bar.BackgroundColor3 = Color3.fromRGB(50,50,50)
	bar.BorderSizePixel = 0
	bar.ZIndex = 13
	bar.Parent = frame
	Instance.new("UICorner", bar).CornerRadius = UDim.new(1, 0)

	local fill = Instance.new("Frame")
	fill.Size = UDim2.new((initial - min) / (max - min), 0, 1, 0)
	fill.BackgroundColor3 = ACCENT
	fill.BorderSizePixel = 0
	fill.ZIndex = 14
	fill.Parent = bar
	Instance.new("UICorner", fill).CornerRadius = UDim.new(1, 0)

	local dragging = false
	local function update(input)
		local pos = math.clamp((input.Position.X - bar.AbsolutePosition.X) / bar.AbsoluteSize.X, 0, 1)
		fill.Size = UDim2.new(pos, 0, 1, 0)
		local val = math.floor(min + (max - min) * pos + 0.5)
		lbl.Text = text .. ": " .. tostring(val)
		onChange(val)
	end
	bar.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1
		or input.UserInputType == Enum.UserInputType.Touch then
			dragging = true
			update(input)
		end
	end)
	UserInputService.InputChanged:Connect(function(input)
		if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement
		or input.UserInputType == Enum.UserInputType.Touch) then
			update(input)
		end
	end)
	UserInputService.InputEnded:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1
		or input.UserInputType == Enum.UserInputType.Touch then
			dragging = false
		end
	end)
end

local function makeScroll(parent, canvasHeight)
	local scroll = Instance.new("ScrollingFrame")
	scroll.Position = UDim2.fromOffset(0, 32)
	scroll.Size = UDim2.new(1, 0, 1, -32)
	scroll.BackgroundTransparency = 1
	scroll.BorderSizePixel = 0
	scroll.ScrollBarThickness = 4
	scroll.CanvasSize = UDim2.new(0, 0, 0, canvasHeight or 420)
	scroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
	scroll.ScrollingDirection = Enum.ScrollingDirection.Y
	scroll.ElasticBehavior = Enum.ElasticBehavior.Never
	scroll.ScrollBarImageColor3 = ACCENT
	scroll.ZIndex = 12
	scroll.Parent = parent

	-- Limita o scroll pra parar exatamente no último item
	scroll:GetPropertyChangedSignal("AbsoluteWindowSize"):Connect(function()
		local windowH = scroll.AbsoluteWindowSize.Y
		local contentH = scroll.AbsoluteCanvasSize.Y
		if contentH <= windowH then
			scroll.CanvasPosition = Vector2.new(0, 0)
		else
			local maxY = contentH - windowH
			if scroll.CanvasPosition.Y > maxY then
				scroll.CanvasPosition = Vector2.new(0, maxY)
			end
		end
	end)

	scroll:GetPropertyChangedSignal("CanvasPosition"):Connect(function()
		local windowH = scroll.AbsoluteWindowSize.Y
		local contentH = scroll.AbsoluteCanvasSize.Y
		local maxY = math.max(0, contentH - windowH)
		if scroll.CanvasPosition.Y > maxY then
			scroll.CanvasPosition = Vector2.new(0, maxY)
		elseif scroll.CanvasPosition.Y < 0 then
			scroll.CanvasPosition = Vector2.new(0, 0)
		end
	end)

	return scroll
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

	local desc = Instance.new("TextLabel")
	desc.BackgroundTransparency = 1
	desc.Position = UDim2.fromOffset(9, 26)
	desc.Size = UDim2.new(1, -19, 0, 16)
	desc.Font = Enum.Font.Gotham
	desc.Text = SaveConfig.FS_OK and "Configs salvas automaticamente" or "Configs apenas em memória"
	desc.TextSize = 9
	desc.TextColor3 = ctx.colors.SUBTEXT
	desc.TextXAlignment = Enum.TextXAlignment.Left
	desc.Parent = page

	local function makeCard(titleText, initialText, xPos)
		local card = Instance.new("Frame")
		card.Size = UDim2.new(0.48, 0, 0, 47)
		card.Position = UDim2.new(xPos, 0, 0, 50)
		card.BackgroundColor3 = ctx.colors.CARD
		card.Parent = page
		Instance.new("UICorner", card).CornerRadius = UDim.new(0, 5)
		local tt = Instance.new("TextLabel")
		tt.BackgroundTransparency = 1
		tt.Position = UDim2.fromOffset(7, 6)
		tt.Size = UDim2.new(1, -13, 0, 11)
		tt.Font = Enum.Font.GothamBold
		tt.Text = titleText
		tt.TextSize = 9
		tt.TextColor3 = ctx.colors.ACCENT
		tt.TextXAlignment = Enum.TextXAlignment.Left
		tt.Parent = card
		local vv = Instance.new("TextLabel")
		vv.BackgroundTransparency = 1
		vv.Position = UDim2.fromOffset(7, 18)
		vv.Size = UDim2.new(1, -13, 0, 17)
		vv.Font = Enum.Font.GothamBlack
		vv.Text = initialText
		vv.TextSize = 14
		vv.TextColor3 = ctx.colors.TEXT
		vv.TextXAlignment = Enum.TextXAlignment.Left
		vv.Parent = card
		return vv
	end

	local pingValue   = makeCard("PING", "...", 0)
	local serverValue = makeCard("NO SERVIDOR", tostring(#Players:GetPlayers()), 0.52)

	task.spawn(function()
		while page.Parent do
			local start = os.clock()
			task.wait()
			local ms = math.floor((os.clock() - start) * 1000)
			if ms < 1 then ms = math.random(20, 60) end
			pingValue.Text = ms .. " ms"
			serverValue.Text = tostring(#Players:GetPlayers())
			task.wait(1)
		end
	end)
end
TabRegistry["AIM LOCK"] = (function()
	local state = {
		Enabled      = SaveConfig.register("aim.enabled",    false),
		AutoLock     = SaveConfig.register("aim.autolock",   true),
		FOV          = SaveConfig.register("aim.fov",        120),
		Smooth       = SaveConfig.register("aim.smooth",     0.25),
		AimPart      = SaveConfig.register("aim.part",       "Head"),
		Prediction   = SaveConfig.register("aim.prediction", 0.15),
		ShowFOV      = SaveConfig.register("aim.showfov",    true),
		Highlight    = SaveConfig.register("aim.highlight",  true),
		TeamCheck    = SaveConfig.register("aim.teamcheck",  true),
		DragOutDist  = SaveConfig.register("aim.dragout",    40),
		BindPC       = Enum.KeyCode.E,
		ToggleBindPC = Enum.KeyCode.RightAlt,
	}
	local StickyTarget, HighlightObj = nil, nil
	local runtimeInited = false
	local fovFrameRef = nil

	local function isAlive(plr)
		local char = plr.Character
		if not char then return false end
		local hum = char:FindFirstChildOfClass("Humanoid")
		return hum and hum.Health > 0
	end
	local function isEnemy(plr)
		if plr == player then return false end
		if not state.TeamCheck then return true end
		return plr.Team ~= player.Team
	end
	local function getAimPart(plr)
		local char = plr.Character
		if not char then return nil end
		if state.AimPart == "Head" then return char:FindFirstChild("Head")
		elseif state.AimPart == "Torso" then return char:FindFirstChild("Torso") or char:FindFirstChild("UpperTorso")
		else return char:FindFirstChild("HumanoidRootPart") end
	end
	local function worldToScreen(pos)
		local sp, onScreen = workspace.CurrentCamera:WorldToViewportPoint(pos)
		return Vector2.new(sp.X, sp.Y), onScreen
	end
	local function getScreenCenter()
		local vp = workspace.CurrentCamera.ViewportSize
		return Vector2.new(vp.X/2, vp.Y/2)
	end
	local function isVisible(part)
		local origin = workspace.CurrentCamera.CFrame.Position
		local dir = part.Position - origin
		local params = RaycastParams.new()
		params.FilterType = Enum.RaycastFilterType.Exclude
		params.FilterDescendantsInstances = { player.Character, part.Parent, workspace.CurrentCamera }
		params.IgnoreWater = true
		local result = workspace:Raycast(origin, dir, params)
		return result == nil or result.Instance:IsDescendantOf(part.Parent)
	end
	local function getClosestTarget()
		local center = getScreenCenter()
		local best, bestDist = nil, state.FOV
		for _, plr in ipairs(Players:GetPlayers()) do
			if isAlive(plr) and isEnemy(plr) then
				local part = getAimPart(plr)
				if part then
					local screenPos, onScreen = worldToScreen(part.Position)
					if onScreen then
						local dist = (screenPos - center).Magnitude
						if dist < bestDist and isVisible(part) then
							bestDist = dist; best = plr
						end
					end
				end
			end
		end
		return best
	end
	local function predictPosition(part)
		if state.Prediction <= 0 then return part.Position end
		return part.Position + part.AssemblyLinearVelocity * state.Prediction
	end
	local function clearHighlight()
		if HighlightObj then HighlightObj:Destroy(); HighlightObj = nil end
	end
	local function applyHighlight(plr)
		clearHighlight()
		if not state.Highlight or not plr or not plr.Character then return end
		local h = Instance.new("Highlight")
		h.FillColor = Color3.fromRGB(255,60,60)
		h.OutlineColor = Color3.new(1,1,1)
		h.FillTransparency = 0.6
		h.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
		h.Adornee = plr.Character
		h.Parent = plr.Character
		HighlightObj = h
	end
	local function shouldReleaseFromTarget(plr)
		if not plr or not plr.Character then return true end
		local part = getAimPart(plr)
		if not part then return true end
		local screenPos, onScreen = worldToScreen(part.Position)
		if not onScreen then return true end
		return (UserInputService:GetMouseLocation() - screenPos).Magnitude > state.DragOutDist
	end

	local function initRuntime()
		if runtimeInited then return end
		runtimeInited = true

		local fovGui = Instance.new("ScreenGui")
		fovGui.Name = "AimLockFOV"
		fovGui.ResetOnSpawn = false
		fovGui.IgnoreGuiInset = true
		fovGui.Parent = playerGui

		local fovFrame = Instance.new("Frame")
		fovFrame.AnchorPoint = Vector2.new(0.5, 0.5)
		fovFrame.Position = UDim2.fromScale(0.5, 0.5)
		fovFrame.BackgroundTransparency = 1
		fovFrame.Size = UDim2.fromOffset(state.FOV*2, state.FOV*2)
		fovFrame.Visible = state.ShowFOV
		fovFrame.Parent = fovGui
		fovFrameRef = fovFrame

		local s = Instance.new("UIStroke", fovFrame)
		s.Thickness = 1; s.Color = Color3.new(1,1,1); s.Transparency = 0.5
		Instance.new("UICorner", fovFrame).CornerRadius = UDim.new(1,0)

		RunService.RenderStepped:Connect(function()
			if not state.Enabled then
				StickyTarget = nil; clearHighlight(); return
			end
			local wantsLock = state.AutoLock or UserInputService:IsKeyDown(state.BindPC)
			if not wantsLock then
				StickyTarget = nil
			else
				if StickyTarget and (not isAlive(StickyTarget) or shouldReleaseFromTarget(StickyTarget)) then
					StickyTarget = nil
				end
				if not StickyTarget then StickyTarget = getClosestTarget() end
			end
			if not StickyTarget or not isAlive(StickyTarget) then clearHighlight(); return end
			local part = getAimPart(StickyTarget)
			if not part or not isVisible(part) then
				StickyTarget = nil; clearHighlight(); return
			end
			applyHighlight(StickyTarget)
			local targetPos = predictPosition(part)
			local desired = CFrame.new(workspace.CurrentCamera.CFrame.Position, targetPos)
			workspace.CurrentCamera.CFrame = workspace.CurrentCamera.CFrame:Lerp(desired, math.clamp(1 - state.Smooth, 0, 1))
		end)

		UserInputService.InputBegan:Connect(function(input, gpe)
			if gpe then return end
			if input.KeyCode == state.ToggleBindPC then
				state.Enabled = not state.Enabled
				SaveConfig.set("aim.enabled", state.Enabled)
			end
		end)

		if UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled then
			local btn = Instance.new("TextButton")
			btn.Size = UDim2.fromOffset(60,60)
			btn.Position = UDim2.new(1,-80,0.5,-30)
			btn.BackgroundColor3 = state.Enabled and Color3.fromRGB(40,160,70) or Color3.fromRGB(30,30,30)
			btn.BackgroundTransparency = 0.25
			btn.TextColor3 = Color3.new(1,1,1)
			btn.Font = Enum.Font.GothamBold
			btn.TextSize = 11
			btn.Text = state.Enabled and "AIM\nON" or "AIM\nOFF"
			btn.BorderSizePixel = 0
			btn.Parent = fovGui
			Instance.new("UICorner", btn).CornerRadius = UDim.new(1,0)
			btn.MouseButton1Click:Connect(function()
				state.Enabled = not state.Enabled
				SaveConfig.set("aim.enabled", state.Enabled)
				btn.Text = state.Enabled and "AIM\nON" or "AIM\nOFF"
				btn.BackgroundColor3 = state.Enabled and Color3.fromRGB(40,160,70) or Color3.fromRGB(30,30,30)
			end)
		end
	end

	return function(page, ctx)
		initRuntime()
		local t = Instance.new("TextLabel")
		t.BackgroundTransparency = 1
		t.Position = UDim2.fromOffset(9,8)
		t.Size = UDim2.new(1,-18,0,17)
		t.Font = Enum.Font.GothamBlack
		t.Text = "AIM LOCK"
		t.TextSize = 13
		t.TextColor3 = TEXT
		t.TextXAlignment = Enum.TextXAlignment.Left
		t.Parent = page

		local scroll = makeScroll(page, 420)
		makeToggle(scroll, "Ativado", 8, state.Enabled, function(v) state.Enabled = v; SaveConfig.set("aim.enabled", v) end)
		makeToggle(scroll, "Auto Lock", 36, state.AutoLock, function(v) state.AutoLock = v; SaveConfig.set("aim.autolock", v) end)
		makeToggle(scroll, "Prediction", 64, state.Prediction > 0, function(v)
			state.Prediction = v and 0.15 or 0
			SaveConfig.set("aim.prediction", state.Prediction)
		end)
		makeToggle(scroll, "Mostrar FOV", 92, state.ShowFOV, function(v)
			state.ShowFOV = v; SaveConfig.set("aim.showfov", v)
			if fovFrameRef then fovFrameRef.Visible = v end
		end)
		makeToggle(scroll, "Highlight", 120, state.Highlight, function(v) state.Highlight = v; SaveConfig.set("aim.highlight", v) end)
		makeToggle(scroll, "Checar Time", 148, state.TeamCheck, function(v) state.TeamCheck = v; SaveConfig.set("aim.teamcheck", v) end)
		makeSlider(scroll, "FOV", 184, 20, 500, state.FOV, function(v)
			state.FOV = v; SaveConfig.set("aim.fov", v)
			if fovFrameRef then fovFrameRef.Size = UDim2.fromOffset(v*2, v*2) end
		end)
		makeSlider(scroll, "Smooth x100", 224, 0, 95, math.floor(state.Smooth*100), function(v)
			state.Smooth = v/100; SaveConfig.set("aim.smooth", state.Smooth)
		end)
		makeSlider(scroll, "Prediction x100", 264, 0, 80, math.floor(state.Prediction*100), function(v)
			state.Prediction = v/100; SaveConfig.set("aim.prediction", state.Prediction)
		end)
		makeSlider(scroll, "DragOut px", 304, 5, 150, state.DragOutDist, function(v)
			state.DragOutDist = v; SaveConfig.set("aim.dragout", v)
		end)
	end
end)()
TabRegistry["ESP"] = (function()
	local state = {
		Enabled    = SaveConfig.register("esp.enabled",    false),
		ShowName   = SaveConfig.register("esp.showname",   true),
		ShowDist   = SaveConfig.register("esp.showdist",   true),
		ShowHealth = SaveConfig.register("esp.showhealth", true),
		MaxDist    = SaveConfig.register("esp.maxdist",    500),
	}
	local espFolder = Instance.new("Folder")
	espFolder.Name = "BatataESP"
	espFolder.Parent = playerGui
	local active = {}
	local runtimeInited = false

	local function clearAll()
		for _, data in pairs(active) do
			if data.highlight then data.highlight:Destroy() end
			if data.billboard then data.billboard:Destroy() end
		end
		active = {}
	end

	local function buildFor(plr)
		if plr == player then return end
		local char = plr.Character
		if not char then return end
		local hrp = char:FindFirstChild("HumanoidRootPart") or char:FindFirstChild("Torso") or char:FindFirstChild("UpperTorso")
		if not hrp then return end
		local hum = char:FindFirstChildOfClass("Humanoid")

		local hl = Instance.new("Highlight")
		hl.FillColor = Color3.fromRGB(255,80,80)
		hl.OutlineColor = Color3.new(1,1,1)
		hl.FillTransparency = 0.65
		hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
		hl.Adornee = char
		hl.Parent = espFolder

		local bb = Instance.new("BillboardGui")
		bb.Size = UDim2.fromOffset(200,50)
		bb.StudsOffsetWorldSpace = Vector3.new(0,3.2,0)
		bb.AlwaysOnTop = true
		bb.Adornee = hrp
		bb.Parent = espFolder

		local label = Instance.new("TextLabel")
		label.BackgroundTransparency = 1
		label.Size = UDim2.new(1,0,1,0)
		label.Font = Enum.Font.GothamBold
		label.TextSize = 12
		label.TextColor3 = Color3.fromRGB(255,80,80)
		label.TextStrokeTransparency = 0
		label.Text = plr.Name
		label.Parent = bb

		active[plr] = { highlight=hl, billboard=bb, label=label, hum=hum, hrp=hrp }
	end

	local function updateLoop()
		clearAll()
		if not state.Enabled then return end
		for _, plr in ipairs(Players:GetPlayers()) do
			if plr ~= player then pcall(buildFor, plr) end
		end
		local start = tick()
		while tick() - start < 0.1 and state.Enabled do
			for plr, data in pairs(active) do
				if data.label and data.hrp and data.hrp.Parent then
					local parts = {}
					if state.ShowName then parts[#parts+1] = plr.Name end
					if state.ShowDist then
						local d = math.floor((workspace.CurrentCamera.CFrame.Position - data.hrp.Position).Magnitude)
						parts[#parts+1] = "["..d.."m]"
					end
					if state.ShowHealth and data.hum then
						parts[#parts+1] = math.floor(data.hum.Health).."hp"
					end
					data.label.Text = table.concat(parts, " ")
					local dist = (workspace.CurrentCamera.CFrame.Position - data.hrp.Position).Magnitude
					data.label.Visible = dist <= state.MaxDist
					data.highlight.Enabled = dist <= state.MaxDist
				end
			end
			task.wait(0.03)
		end
	end

	local function initRuntime()
		if runtimeInited then return end
		runtimeInited = true
		task.spawn(function()
			while espFolder.Parent do
				task.wait(0.1)
				if state.Enabled then updateLoop() end
			end
		end)
	end

	return function(page, ctx)
		initRuntime()
		local t = Instance.new("TextLabel")
		t.BackgroundTransparency = 1
		t.Position = UDim2.fromOffset(9,8)
		t.Size = UDim2.new(1,-18,0,17)
		t.Font = Enum.Font.GothamBlack
		t.Text = "ESP"
		t.TextSize = 13
		t.TextColor3 = TEXT
		t.TextXAlignment = Enum.TextXAlignment.Left
		t.Parent = page

		local scroll = makeScroll(page, 260)
		makeToggle(scroll, "Ativado", 8, state.Enabled, function(v)
			state.Enabled = v; SaveConfig.set("esp.enabled", v)
			if not v then clearAll() end
		end)
		makeToggle(scroll, "Nome", 36, state.ShowName, function(v) state.ShowName = v; SaveConfig.set("esp.showname", v) end)
		makeToggle(scroll, "Distância", 64, state.ShowDist, function(v) state.ShowDist = v; SaveConfig.set("esp.showdist", v) end)
		makeToggle(scroll, "Vida", 92, state.ShowHealth, function(v) state.ShowHealth = v; SaveConfig.set("esp.showhealth", v) end)
		makeSlider(scroll, "Dist. máx", 128, 50, 2000, state.MaxDist, function(v) state.MaxDist = v; SaveConfig.set("esp.maxdist", v) end)
	end
end)()
TabRegistry["PLAYER"] = (function()
	local state = {
		Float       = SaveConfig.register("plr.float",   false),
		FloatHeight = SaveConfig.register("plr.floatH",  5),
		Speed       = SaveConfig.register("plr.speed",   false),
		SpeedValue  = SaveConfig.register("plr.speedV",  32),
		Jump        = SaveConfig.register("plr.jump",    false),
		JumpValue   = SaveConfig.register("plr.jumpV",   100),
		InfJump     = SaveConfig.register("plr.infJump", false),
	}
	local floatBody, floatAttach
	local runtimeInited = false

	local function getHum()
		return player.Character and player.Character:FindFirstChildOfClass("Humanoid")
	end
	local function setupFloat()
		local char = player.Character
		if not char then return end
		local hrp = char:FindFirstChild("HumanoidRootPart")
		if not hrp then return end
		floatAttach = Instance.new("Attachment", hrp)
		floatBody = Instance.new("BodyPosition")
		floatBody.MaxForce = Vector3.new(1e5,1e5,1e5)
		floatBody.D = 600
		floatBody.P = 12000
		floatBody.Position = hrp.Position
		floatBody.Parent = hrp
	end
	local function destroyFloat()
		if floatBody then floatBody:Destroy(); floatBody=nil end
		if floatAttach then floatAttach:Destroy(); floatAttach=nil end
	end

	local function initRuntime()
		if runtimeInited then return end
		runtimeInited = true

		RunService.Heartbeat:Connect(function()
			if state.Float then
				if not floatBody then setupFloat() end
				if floatBody then
					local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
					if hrp then
						local target = hrp.Position
						target = Vector3.new(target.X, target.Y + state.FloatHeight*0.02, target.Z)
						floatBody.Position = target
					end
				end
			else
				destroyFloat()
			end
			local hum = getHum()
			if hum then
				hum.WalkSpeed = state.Speed and state.SpeedValue or 16
				hum.JumpPower = state.Jump and state.JumpValue or 50
				hum.UseJumpPower = true
			end
		end)

		UserInputService.JumpRequest:Connect(function()
			if state.InfJump then
				local hum = getHum()
				if hum then hum:ChangeState(Enum.HumanoidStateType.Jumping) end
			end
		end)
	end

	return function(page, ctx)
		initRuntime()
		local t = Instance.new("TextLabel")
		t.BackgroundTransparency = 1
		t.Position = UDim2.fromOffset(9,8)
		t.Size = UDim2.new(1,-18,0,17)
		t.Font = Enum.Font.GothamBlack
		t.Text = "PLAYER"
		t.TextSize = 13
		t.TextColor3 = TEXT
		t.TextXAlignment = Enum.TextXAlignment.Left
		t.Parent = page

		local scroll = makeScroll(page, 420)
		makeToggle(scroll, "Float", 8, state.Float, function(v) state.Float = v; SaveConfig.set("plr.float", v) end)
		makeSlider(scroll, "Float H", 36, 1, 20, state.FloatHeight, function(v) state.FloatHeight = v; SaveConfig.set("plr.floatH", v) end)
		makeToggle(scroll, "Speed", 76, state.Speed, function(v) state.Speed = v; SaveConfig.set("plr.speed", v) end)
		makeSlider(scroll, "Speed", 104, 16, 300, state.SpeedValue, function(v) state.SpeedValue = v; SaveConfig.set("plr.speedV", v) end)
		makeToggle(scroll, "Jump", 144, state.Jump, function(v) state.Jump = v; SaveConfig.set("plr.jump", v) end)
		makeSlider(scroll, "JumpPower", 172, 50, 500, state.JumpValue, function(v) state.JumpValue = v; SaveConfig.set("plr.jumpV", v) end)
		makeToggle(scroll, "InfJump", 212, state.InfJump, function(v) state.InfJump = v; SaveConfig.set("plr.infJump", v) end)
	end
end)()

TabRegistry["HITBOX"] = (function()
	local state = {
		Enabled      = SaveConfig.register("hb.enabled", false),
		Size         = SaveConfig.register("hb.size",    5),
		Transparency = SaveConfig.register("hb.transp",  1),
		NoLag        = SaveConfig.register("hb.nolag",   true),
		NoLagRange   = SaveConfig.register("hb.range",   100),
	}
	local originals = {}
	local runtimeInited = false

	local function expand(part)
		if not originals[part] then
			originals[part] = {
				size = part.Size, transparency = part.Transparency,
				cancollide = part.CanCollide, cantouch = part.CanTouch,
			}
		end
		local base = originals[part].size
		part.Size = Vector3.new(base.X+state.Size, base.Y+state.Size, base.Z+state.Size)
		part.Transparency = state.Transparency
		part.CanCollide = false
		part.CanTouch = true
	end
	local function restore(part)
		if originals[part] then
			part.Size = originals[part].size
			part.Transparency = originals[part].transparency
			part.CanCollide = originals[part].cancollide
			part.CanTouch = originals[part].cantouch
			originals[part] = nil
		end
	end
	local function restoreAll()
		for part in pairs(originals) do
			if part and part.Parent then restore(part) end
		end
		originals = {}
	end

	local function initRuntime()
		if runtimeInited then return end
		runtimeInited = true

		RunService.Heartbeat:Connect(function()
			if not state.Enabled then
				if next(originals) then restoreAll() end
				return
			end
			local alvo = nil
			if state.NoLag then
				local best, bestDist = nil, state.NoLagRange
				local myHrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
				if myHrp then
					for _, plr in ipairs(Players:GetPlayers()) do
						if plr ~= player and plr.Character then
							local hrp = plr.Character:FindFirstChild("HumanoidRootPart")
							local hum = plr.Character:FindFirstChildOfClass("Humanoid")
							if hrp and hum and hum.Health > 0 then
								local d = (hrp.Position - myHrp.Position).Magnitude
								if d < bestDist then bestDist = d; best = plr end
							end
						end
					end
				end
				alvo = best
			end
			for _, plr in ipairs(Players:GetPlayers()) do
				if plr ~= player and plr.Character then
					local hum = plr.Character:FindFirstChildOfClass("Humanoid")
					local hrp = plr.Character:FindFirstChild("HumanoidRootPart")
					if hum and hum.Health > 0 and hrp then
						local aplicar = (not state.NoLag) or (plr == alvo)
						if aplicar then
							expand(hrp)
							local head = plr.Character:FindFirstChild("Head")
							if head then expand(head) end
						else
							restore(hrp)
							local head = plr.Character:FindFirstChild("Head")
							if head then restore(head) end
						end
					end
				end
			end
		end)
	end

	return function(page, ctx)
		initRuntime()
		local t = Instance.new("TextLabel")
		t.BackgroundTransparency = 1
		t.Position = UDim2.fromOffset(9,8)
		t.Size = UDim2.new(1,-18,0,17)
		t.Font = Enum.Font.GothamBlack
		t.Text = "HITBOX EXPANDER"
		t.TextSize = 13
		t.TextColor3 = TEXT
		t.TextXAlignment = Enum.TextXAlignment.Left
		t.Parent = page

		local scroll = makeScroll(page, 340)
		makeToggle(scroll, "Ativado", 8, state.Enabled, function(v)
			state.Enabled = v; SaveConfig.set("hb.enabled", v)
			if not v then restoreAll() end
		end)
		makeToggle(scroll, "Modo No-Lag", 36, state.NoLag, function(v) state.NoLag = v; SaveConfig.set("hb.nolag", v) end)
		makeSlider(scroll, "Tamanho", 72, 1, 30, state.Size, function(v) state.Size = v; SaveConfig.set("hb.size", v) end)
		makeSlider(scroll, "Transp x100", 112, 0, 100, math.floor(state.Transparency*100), function(v)
			state.Transparency = v/100; SaveConfig.set("hb.transp", state.Transparency)
		end)
		makeSlider(scroll, "Range No-Lag", 152, 10, 500, state.NoLagRange, function(v) state.NoLagRange = v; SaveConfig.set("hb.range", v) end)
	end
end)()
local TAB_ORDER = { "HOME", "AIM LOCK", "ESP", "PLAYER", "HITBOX" }
local LOAD_DELAY = 2.5

local function makeTab(name, iconId)
	local btn = createTab(name, iconId)
	btn.Activated:Connect(function() playClick(); selectTab(name) end)
	addHover(btn, CARD, Color3.fromRGB(38, 38, 38))
	return btn
end

task.spawn(function()
	for i, name in ipairs(TAB_ORDER) do
		local iconId = (name == "HOME") and CONFIG.HomeIconId or nil
		makeTab(name, iconId)
		Notify(name .. " carregada (" .. i .. "/" .. #TAB_ORDER .. ")", 2)
		if i < #TAB_ORDER then
			task.wait(LOAD_DELAY)
		end
	end
	task.wait(0.4)
	selectTab("HOME")
	Notify("Batata Hub carregado!", 3)
end)

AntiCheatDetector.onDetect(function(reason)
	Notify("Anti-Cheat: " .. reason, 5)
end)

local settingsOpen = false
local function showBackgroundSettings()
	if settingsOpen then return end
	settingsOpen = true
	local page = createPage()

	local titleLbl = Instance.new("TextLabel")
	titleLbl.BackgroundTransparency = 1
	titleLbl.Position = UDim2.fromOffset(9, 8)
	titleLbl.Size = UDim2.new(1, -18, 0, 17)
	titleLbl.Font = Enum.Font.GothamBlack
	titleLbl.Text = "CONFIGURAÇÕES"
	titleLbl.TextSize = 13
	titleLbl.TextColor3 = TEXT
	titleLbl.TextXAlignment = Enum.TextXAlignment.Left
	titleLbl.Parent = page

	local input = Instance.new("TextBox")
	input.Size = UDim2.new(1, -18, 0, 26)
	input.Position = UDim2.fromOffset(9, 36)
	input.BackgroundColor3 = CARD
	input.Text = ""
	input.PlaceholderText = "ID do adesivo (plano de fundo)"
	input.Font = Enum.Font.Gotham
	input.TextSize = 11
	input.TextColor3 = TEXT
	input.ClearTextOnFocus = false
	input.Parent = page
	Instance.new("UICorner", input).CornerRadius = UDim.new(0, 5)

	local confirmBtn = Instance.new("TextButton")
	confirmBtn.Size = UDim2.new(1, -18, 0, 26)
	confirmBtn.Position = UDim2.fromOffset(9, 70)
	confirmBtn.BackgroundColor3 = ACCENT
	confirmBtn.Text = "Salvar plano de fundo"
	confirmBtn.Font = Enum.Font.GothamBold
	confirmBtn.TextSize = 11
	confirmBtn.TextColor3 = BLACK
	confirmBtn.AutoButtonColor = false
	confirmBtn.Parent = page
	Instance.new("UICorner", confirmBtn).CornerRadius = UDim.new(0, 5)
	addHover(confirmBtn, ACCENT, Color3.fromRGB(255, 215, 80))

	confirmBtn.Activated:Connect(function()
		playClick()
		local id = input.Text:match("%d+")
		if id then
			bgImage.Image = "rbxassetid://" .. id
			SaveConfig.set("ui.bg", id)
			TweenService:Create(bgImage, TweenInfo.new(0.4, Enum.EasingStyle.Sine), {ImageTransparency = 0.8}):Play()
			Notify("Plano de fundo salvo!", 3)
		else
			Notify("ID inválido.", 2)
		end
		settingsOpen = false
		selectTab("HOME")
	end)

	local discordBtn = Instance.new("TextButton")
	discordBtn.Size = UDim2.new(1, -18, 0, 26)
	discordBtn.Position = UDim2.fromOffset(9, 104)
	discordBtn.BackgroundColor3 = Color3.fromRGB(88, 101, 242)
	discordBtn.Text = "Discord — clique para copiar"
	discordBtn.Font = Enum.Font.GothamBold
	discordBtn.TextSize = 11
	discordBtn.TextColor3 = TEXT
	discordBtn.AutoButtonColor = false
	discordBtn.Parent = page
	Instance.new("UICorner", discordBtn).CornerRadius = UDim.new(0, 5)
	addHover(discordBtn, Color3.fromRGB(88, 101, 242), Color3.fromRGB(115, 128, 255))

	discordBtn.Activated:Connect(function()
		playClick()
		local ok = pcall(function()
			if type(setclipboard) == "function" then
				setclipboard(CONFIG.DiscordLink)
			end
		end)
		if ok then
			Notify("Link do Discord copiado!", 3)
		else
			Notify("Não foi possível copiar. Link: " .. CONFIG.DiscordLink, 5)
		end
	end)

	local restoreBtn = Instance.new("TextButton")
	restoreBtn.Size = UDim2.new(1, -18, 0, 26)
	restoreBtn.Position = UDim2.fromOffset(9, 138)
	restoreBtn.BackgroundColor3 = CARD
	restoreBtn.Text = "Restaurar posições"
	restoreBtn.Font = Enum.Font.GothamBold
	restoreBtn.TextSize = 11
	restoreBtn.TextColor3 = TEXT
	restoreBtn.AutoButtonColor = false
	restoreBtn.Parent = page
	Instance.new("UICorner", restoreBtn).CornerRadius = UDim.new(0, 5)
	addHover(restoreBtn, CARD, Color3.fromRGB(38,38,38))

	restoreBtn.Activated:Connect(function()
		playClick()
		panel.Position = UDim2.new(0.5, 0, 0.5, 0)
		floating.Position = UDim2.new(0, 14, 0.5, -28)
		SaveConfig.set("ui.panX", 0); SaveConfig.set("ui.panY", 0)
		SaveConfig.set("ui.btnX", 14); SaveConfig.set("ui.btnY", -28)
		Notify("Posições restauradas.", 3)
		settingsOpen = false
		selectTab("HOME")
	end)
end

gear.Activated:Connect(function() playClick(); if not settingsOpen then showBackgroundSettings() end end)

saveBtn.Activated:Connect(function()
	playClick()
	SaveConfig.saveNow()
	Notify("Configs salvas!", 2)
end)

resetBtn.Activated:Connect(function()
	playClick()
	SaveConfig.reset()
	Notify("Configs resetadas! Reinicie o script.", 4)
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

-- DRAG BOTÃO: usa delta absoluto com âncora fixa e clampa em cada frame
local draggingButton = false
local dragStart = nil
local buttonStart = nil

floating.InputBegan:Connect(function(input)
	if input.UserInputType ~= Enum.UserInputType.MouseButton1 and input.UserInputType ~= Enum.UserInputType.Touch then return end
	draggingButton = true
	dragStart = Vector2.new(input.Position.X, input.Position.Y)
	buttonStart = Vector2.new(floating.AbsolutePosition.X, floating.AbsolutePosition.Y)
end)

UserInputService.InputChanged:Connect(function(input)
	if not draggingButton then return end
	if input.UserInputType ~= Enum.UserInputType.MouseMovement and input.UserInputType ~= Enum.UserInputType.Touch then return end
	if not dragStart or not buttonStart then return end

	local current = Vector2.new(input.Position.X, input.Position.Y)
	local delta = current - dragStart
	local raw = buttonStart + delta
	local vp = workspace.CurrentCamera.ViewportSize
	local clamped = Vector2.new(
		math.clamp(raw.X, 0, vp.X - BUTTON_SIZE),
		math.clamp(raw.Y, 0, vp.Y - BUTTON_SIZE)
	)
	floating.Position = UDim2.fromOffset(clamped.X, clamped.Y)
end)

UserInputService.InputEnded:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		if draggingButton then
			draggingButton = false
			SaveConfig.set("ui.btnX", floating.Position.X.Offset)
			SaveConfig.set("ui.btnY", floating.Position.Y.Offset)
		end
	end
end)

-- DRAG PAINEL
local draggingPanel = false
local panelDragStart = nil
local panelStart = nil

header.InputBegan:Connect(function(input)
	if input.UserInputType ~= Enum.UserInputType.MouseButton1 and input.UserInputType ~= Enum.UserInputType.Touch then return end
	draggingPanel = true
	panelDragStart = Vector2.new(input.Position.X, input.Position.Y)
	panelStart = Vector2.new(panel.AbsolutePosition.X, panel.AbsolutePosition.Y)
end)

UserInputService.InputChanged:Connect(function(input)
	if not draggingPanel then return end
	if input.UserInputType ~= Enum.UserInputType.MouseMovement and input.UserInputType ~= Enum.UserInputType.Touch then return end
	if not panelDragStart or not panelStart then return end

	local current = Vector2.new(input.Position.X, input.Position.Y)
	local delta = current - panelDragStart
	local raw = panelStart + delta
	local vp = workspace.CurrentCamera.ViewportSize
	local size = panel.AbsoluteSize
	local clamped = Vector2.new(
		math.clamp(raw.X, -size.X + 40, vp.X - 40),
		math.clamp(raw.Y, 0, vp.Y - 40)
	)
	panel.Position = UDim2.fromOffset(clamped.X, clamped.Y)
end)

UserInputService.InputEnded:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		if draggingPanel then
			draggingPanel = false
			SaveConfig.set("ui.panX", panel.Position.X.Offset)
			SaveConfig.set("ui.panY", panel.Position.Y.Offset)
		end
	end
end)

task.spawn(function()
	local asked = false
	while gui.Parent do
		task.wait(0.5)
		local btnPos = Vector2.new(floating.AbsolutePosition.X, floating.AbsolutePosition.Y)
		local btnOff = isMostlyOffscreen(btnPos, floating.AbsoluteSize, 0.85)
		local panelOff = false
		if panel.Visible then
			local panPos = Vector2.new(panel.AbsolutePosition.X, panel.AbsolutePosition.Y)
			panelOff = isMostlyOffscreen(panPos, panel.AbsoluteSize, 0.85)
		end
		if (btnOff or panelOff) and not asked then
			asked = true
			AskConfirm("Deseja voltar a interface?", function()
				panel.Position = UDim2.new(0.5, 0, 0.5, 0)
				floating.Position = UDim2.new(0, 14, 0.5, -28)
				SaveConfig.set("ui.panX", 0); SaveConfig.set("ui.panY", 0)
				SaveConfig.set("ui.btnX", 14); SaveConfig.set("ui.btnY", -28)
				Notify("Interface restaurada!", 2)
				asked = false
			end, function()
				asked = false
			end, 10)
		end
	end
end)

UserInputService.InputBegan:Connect(function(input, gpe)
	if gpe then return end
	if input.KeyCode == CONFIG.PanicKey then
		SaveConfig.set("aim.enabled", false)
		SaveConfig.set("esp.enabled", false)
		SaveConfig.set("plr.float",   false)
		SaveConfig.set("plr.speed",   false)
		SaveConfig.set("plr.jump",    false)
		SaveConfig.set("plr.infJump", false)
		SaveConfig.set("hb.enabled",  false)
		Notify("Panic: features desativadas.", 3)
		SaveConfig.saveNow()
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
	flash.BackgroundColor3 = Color3.fromRGB(255,255,255)
	flash.BorderSizePixel = 0
	flash.ZIndex = 205
	flash.Parent = gui
	Instance.new("UICorner", flash).CornerRadius = UDim.new(1,0)
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
	Instance.new("UICorner", shockwave).CornerRadius = UDim.new(1,0)
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
		spark.BackgroundColor3 = (i % 2 == 0) and ACCENT or Color3.fromRGB(255,255,255)
		spark.BorderSizePixel = 0
		spark.ZIndex = 204
		spark.Parent = gui
		Instance.new("UICorner", spark).CornerRadius = UDim.new(1,0)
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
	floating.Rotation = -8
	TweenService:Create(floatingScale, TweenInfo.new(0.5, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Scale = 1}):Play()
	tweenAsync(floating, TweenInfo.new(0.18, Enum.EasingStyle.Sine, Enum.EasingDirection.Out), {Rotation = 8})
	tweenAsync(floating, TweenInfo.new(0.16, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {Rotation = -4})
	tweenAsync(floating, TweenInfo.new(0.14, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {Rotation = 0})

	Notify("Batata Hub carregado!", 3)
end

task.spawn(PlayIntroSequence)

print("[Batata Hub] Carregado.")
