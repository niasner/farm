local CONFIG = {
	BaseUrl  = "https://nbhh.pages.dev/",
	Images   = { "nbhh1.png", "nbhh2.png", "nbhh3.png" },
	Audio    = "nbhh.mp3",

	FlashDelay = 0.15,                     -- 每张图停留时间 (task.wait)
	Strobe     = false,                    -- true = 每次切换前插一帧全透明, 闪得更狠
	ScaleType  = Enum.ScaleType.Crop,      -- Crop=铺满不变形 / Stretch=拉伸铺满
	Pixelated  = false,                    -- true = 关闭平滑采样, 像素风更"硬"

	Loop   = true,                         -- 音频循环
	Volume = 1,                            -- 音量 (0~10)
	FallbackSoundId = "",                  -- 本地 MP3 播不了时兜底, 填 "rbxassetid://数字"

	Cache     = true,                      -- 本地已有文件就跳过下载 (改 false 强制重下)
	Retries   = 3,
	RetryWait = 1.5,
	HideTopBar = false,                    -- true = 连 Roblox 顶栏/聊天一起隐藏, 覆盖更彻底
	Verbose   = true,
}

local GENV = (type(getgenv) == "function" and getgenv()) or _G or {}

local function exfn(name)
	local fn = GENV[name]
	if type(fn) ~= "function" then
		local ok, g = pcall(function() return _G[name] end)
		if ok then fn = g end
	end
	if type(fn) == "function" then return fn end
	return nil
end

local STOPPED = false
local function isStopped()
	if STOPPED then return true end
	local ok, v = pcall(function() return GENV.NBHH_STOP end)
	return ok and v == true
end

local function log(...)
	if CONFIG.Verbose then print("[nbhh]", ...) end
end

local function warnLog(...)
	warn("[nbhh]", ...)
end

local writefile  = exfn("writefile")
local readfile   = exfn("readfile")
local isfile     = exfn("isfile")
local getcustomasset = exfn("getcustomasset") or exfn("getsynasset") or exfn("getasset")

local function rawHttpGet(url)
	local req = exfn("request") or exfn("http_request")
	if req then
		local ok, res = pcall(req, { Url = url, Method = "GET" })
		if ok and type(res) == "table" then
			local body = res.Body or res.body
			if type(body) == "string" and #body > 0 then
				return body, "request"
			end
		end
	end

	local ok, body = pcall(function() return game:HttpGet(url, true) end)
	if ok and type(body) == "string" and #body > 0 then
		return body, "game:HttpGet"
	end

	return nil, "无可用 HTTP 方法"
end

local function looksLikeError(body)
	if type(body) ~= "string" or #body == 0 then return true end
	if body:sub(1, 1) == "<" then return true end   -- HTML 错误页
	return false
end

local function isPng(body)
	return type(body) == "string" and #body > 8 and body:sub(1, 4) == "\137PNG"
end

local function isAudio(body)
	if type(body) ~= "string" or #body < 3 then return false end
	if body:sub(1, 3) == "ID3" then return true end         -- ID3 标签
	local b1, b2 = body:byte(1), body:byte(2)
	return b1 == 0xFF and b2 ~= nil and b2 >= 0xE0          -- MPEG 帧同步
end

local function ensureFile(name, kind)
	local cached = false
	if isfile and readfile then
		local ok, data = pcall(readfile, name)
		if ok and type(data) == "string" and #data > 0 then
			if kind == "png" then
				cached = isPng(data)
			else
				cached = not looksLikeError(data)
			end
		end
	end

	if cached and CONFIG.Cache then
		log("使用本地缓存: " .. name)
		return name
	end

	local url = CONFIG.BaseUrl .. name
	local lastErr = "未知错误"

	for attempt = 1, CONFIG.Retries do
		local body, via = rawHttpGet(url)
		if not body then
			lastErr = tostring(via)
		elseif kind == "png" and not isPng(body) then
			lastErr = "返回内容不是合法 PNG (" .. #body .. " 字节)"
		elseif kind == "audio" and looksLikeError(body) then
			lastErr = "返回内容不是音频 (" .. #body .. " 字节)"
		elseif not writefile then
			lastErr = "执行器不支持 writefile"
		else
			if kind == "audio" and not isAudio(body) then
				warnLog("警告: " .. name .. " 头部不像标准 MP3, 仍然继续尝试")
			end
			local ok, err = pcall(writefile, name, body)
			if ok then
				log(("下载完成: %s (%d 字节, via %s)"):format(name, #body, via))
				return name
			end
			lastErr = tostring(err)
		end

		warnLog(("下载失败 (%d/%d) %s : %s"):format(attempt, CONFIG.Retries, name, lastErr))
		task.wait(CONFIG.RetryWait)
	end

	if cached then
		warnLog("下载失败, 回退使用本地缓存: " .. name)
		return name
	end

	return nil, lastErr
end

local function toAssetId(path)
	if not getcustomasset then return nil end

	local candidates = { path }
	local getworkspace = exfn("getworkspace")
	if getworkspace then
		local ok, ws = pcall(getworkspace)
		if ok and type(ws) == "string" and #ws > 0 then
			table.insert(candidates, ws .. "/" .. path)
		end
	end

	for _, p in ipairs(candidates) do
		local ok, id = pcall(getcustomasset, p)
		if ok and type(id) == "string" and #id > 0 then
			return id
		end
	end
	return nil
end

--============================== 实例 ==============================--
local Players      = game:GetService("Players")
local SoundService = game:GetService("SoundService")
local LocalPlayer  = Players.LocalPlayer
local PlayerGui    = LocalPlayer:FindFirstChildOfClass("PlayerGui")
if not PlayerGui then
	PlayerGui = LocalPlayer:WaitForChild("PlayerGui", 30)
end

local State = {
	gui      = nil,
	labels   = {},
	imageIds = {},
	sound    = nil,
	soundId  = nil,
}

--===================== UI 挂载 =====================--
local function guiCandidates()
	local list = {}

	local gethui = exfn("gethui")
	if gethui then
		local ok, container = pcall(gethui)
		if ok and typeof(container) == "Instance" then
			table.insert(list, container)
		end
	end

	local ok, coreGui = pcall(function() return game:GetService("CoreGui") end)
	if ok and typeof(coreGui) == "Instance" then
		table.insert(list, coreGui)
	end

	if PlayerGui then
		table.insert(list, PlayerGui)
	end

	return list
end

local function mount(gui)
	for _, parent in ipairs(guiCandidates()) do
		local ok = pcall(function() gui.Parent = parent end)
		if ok and gui.Parent == parent then
			return parent
		end
	end
	return nil
end

local function buildGui()
	local gui = Instance.new("ScreenGui")
	gui.Name = "nbhh_flash_ui"
	gui.IgnoreGuiInset = true               
	gui.ResetOnSpawn = false                
	gui.DisplayOrder = 2147483647           
	gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
	gui.Enabled = true

	local labels = {}
	for index = 1, #State.imageIds do
		local label = Instance.new("ImageLabel")
		label.Name = "layer_" .. index
		label.AnchorPoint = Vector2.new(0, 0)
		label.Position = UDim2.fromScale(0, 0)
		label.Size = UDim2.fromScale(1, 1)    
		label.BackgroundTransparency = 1
		label.BorderSizePixel = 0
		label.Image = State.imageIds[index]
		label.ScaleType = CONFIG.ScaleType
		label.Visible = (index == 1)
		label.ZIndex = 1000 + index
		if CONFIG.Pixelated then
			pcall(function() label.ResampleMode = Enum.ResamplerMode.Pixelated end)
		end
		label.Parent = gui
		labels[index] = label
	end

	local parent = mount(gui)
	if not parent then
		pcall(function() gui:Destroy() end)
		warnLog("UI 挂载失败: 没有任何可用容器")
		return nil
	end

	State.gui = gui
	State.labels = labels
	log("UI 已挂载到: " .. parent:GetFullName() .. " (DisplayOrder=2147483647)")

	gui.AncestryChanged:Connect(function(_, newParent)
		if newParent == nil and State.gui == gui and not isStopped() then
			task.defer(function()
				if State.gui == gui and not gui.Parent then
					mount(gui)
				end
			end)
		end
	end)

	return gui
end

local function labelsAlive()
	if #State.labels == 0 then return false end
	for _, label in ipairs(State.labels) do
		if not label.Parent then return false end
	end
	return true
end

local function ensureGui()
	local gui = State.gui
	if gui and gui.Parent and labelsAlive() then
		if not gui.Enabled then
			pcall(function() gui.Enabled = true end)
		end
		return
	end

	if gui then
		if labelsAlive() and mount(gui) then
			return
		end
		pcall(function() gui:Destroy() end)
		State.gui = nil
		State.labels = {}
	end

	buildGui()
end

--======================== 闪烁主循环 ========================--
local function startFlashLoop()
	task.spawn(function()
		local index = 1
		while not isStopped() do
			local labels = State.labels
			local count = #labels
			if count > 0 then
				for i = 1, count do
					local label = labels[i]
					if label.Parent then
						label.Visible = (i == index)
					end
				end
				index = index % count + 1
			end

			task.wait(CONFIG.FlashDelay)      

			if CONFIG.Strobe then
				for _, label in ipairs(State.labels) do
					if label.Parent then label.Visible = false end
				end
				task.wait(CONFIG.FlashDelay * 0.4)
			end
		end
	end)
end

--=========================== 音频循环播放 ===========================--
local function resolveAudioId()
	local path, err = ensureFile(CONFIG.Audio, "audio")
	if path then
		local id = toAssetId(path)
		if id then
			log("音频已注册: " .. id)
			return id
		end
		warnLog("执行器不支持 getcustomasset, 本地 MP3 无法直接播放")
	else
		warnLog("MP3 下载失败: " .. tostring(err))
	end

	if CONFIG.FallbackSoundId ~= "" then
		warnLog("回退到备用音频: " .. CONFIG.FallbackSoundId)
		return CONFIG.FallbackSoundId
	end

	return nil
end

local function playSound()
	local sound = State.sound

	if not sound or not sound.Parent then
		sound = Instance.new("Sound")
		sound.Name = "nbhh_bgm"
		sound.SoundId = State.soundId
		sound.Looped = CONFIG.Loop
		sound.Volume = CONFIG.Volume
		sound.PlayOnRemove = false
		sound.Parent = SoundService      
		State.sound = sound

		sound.Ended:Connect(function()
			if not isStopped() then
				pcall(function() sound:Play() end)
			end
		end)
	end

	if not sound.Playing then
		pcall(function() sound:Play() end)
	end

	return sound
end

--============================= 主流程 =============================--
local function main()
	for _, name in ipairs(CONFIG.Images) do
		local path, err = ensureFile(name, "png")
		if path then
			local id = toAssetId(path)
			if id then
				table.insert(State.imageIds, id)
			else
				warnLog(name .. " 无法注册为 Roblox 资源 (缺少 getcustomasset)")
			end
		else
			warnLog(name .. " 下载失败: " .. tostring(err))
		end
	end

	if #State.imageIds == 0 then
		error("[nbhh] 三张图片全部加载失败, 脚本中止")
	end
	log(("图片就绪 %d 张, 开始全屏闪烁 (间隔 %.2fs)"):format(#State.imageIds, CONFIG.FlashDelay))

	buildGui()
	startFlashLoop()

	if CONFIG.HideTopBar then
		pcall(function()
			local StarterGui = game:GetService("StarterGui")
			StarterGui:SetCoreGuiEnabled(Enum.CoreGuiType.All, false)
		end)
	end

	task.spawn(function()
		local id = resolveAudioId()
		if not id then
			warnLog("音频未启动: 本地 MP3 注册失败且未配置 FallbackSoundId")
			return
		end
		State.soundId = id
		playSound()
		log("MP3 循环播放中 (Looped=" .. tostring(CONFIG.Loop) .. ", Volume=" .. CONFIG.Volume .. ")")

		task.wait(5)
		local sound = State.sound
		if sound and not sound.IsLoaded then
			warnLog("音频 5 秒内仍未加载完成, 如果听不到声音请设置 CONFIG.FallbackSoundId")
		end
	end)

	task.spawn(function()
		while not isStopped() do
			task.wait(0.5)
			pcall(function()
				ensureGui()
				if State.soundId then
					playSound()
				end
			end)
		end
	end)
end

--=========================== 停止开关 ===========================--
local function stopEverything()
	STOPPED = true
	pcall(function() GENV.NBHH_STOP = true end)
	if State.gui then pcall(function() State.gui:Destroy() end) end
	if State.sound then
		pcall(function()
			State.sound:Stop()
			State.sound:Destroy()
		end)
	end
	log("已停止")
end

pcall(function() GENV.NBHH_STOP = false end)
pcall(function() GENV.nbhh_stop = stopEverything end)

main()
