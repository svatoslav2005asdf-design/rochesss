-- RoChess Aurora UI / Luau (Roblox), revision 13 / mindex. Author: @Lesxov.
-- Save this full file as rochess3.txt in executor workspace for teleport relaunch.
-- Move-arrow renderer copied from the supplied working rochess3.txt.
-- Native arrows by default; the original calibrated fallback is preserved.
-- Auto board detection serves the separate opponent marker only.
-- BAR: independent draggable/resizable HUD. HEX arrow colors and HUD settings save in profiles.
-- Opponent selection requires actually exposed data; LAST MOVE is the confirmed-move fallback.
-- + / = / keypad +: toggle panel. End: auto; Home: arrows.
-- Respawn: UI survives. Full reconnect: execute script again (or use executor autoexec).
-- CFG: named profile list, CREATE / RENAME / SAVE / LOAD / DELETE.
-- Appearance: Midnight / Graphite / Pearl, four accents, animation toggle.
-- Window: 640 x 480 (4:3); animated visibility, responsive scale and smooth dragging.
-- ULTRA BRAIN: longer bounded search, no deliberate errors; no per-match error guarantee.
-- Direct engine search; no heuristic reranking. Per-position cancellation and frozen search timer.
-- Stockfish Online is optional and sends FEN only when that source is selected.
-- Disk persistence requires readfile/writefile; otherwise profiles live in getgenv only.
--[[
	rocheese.luau - RoChess! engine panel + autopilot
	Luau, Real executor, client side.

	Uses the game's own modules instead of re-implementing chess:
		ReplicatedStorage.Chess.Engine       board, legal moves, UCI/SAN, FEN
		ReplicatedStorage.Chess.Bot          local search: analyse(fen,moves,sec,depth) / think(fen,moves,persona)
		ReplicatedStorage.Chess.VortexBot    server engine bridge: analyse / think
		ReplicatedStorage.ChessClient.App    live client state: session, board, onBoardMove, playMove

	Squares use Engine.algToIndex64 (bottom-up, a1 = 1, h8 = 64).
	Engine scores are always from WHITE's side of the board.

	Panel: top-right, drag it by the title.
	Arrows on the board:
		green  = best move      blue = second best      yellow = third best

	Presets:
		LEGIT  local engine, plays like a 1500 human (noise + blunders), 5-10s per move
		RAJ    server engine (Vortex), always the best move, plays fast

	Controls: STYLE, DELAY min/max, PLAY BEST/ELO, ENGINE LOCAL/VORTEX, AUTOPLAY,
	          PLAY NOW, COPY FEN, BOT GAME.
	Hotkeys: End auto | Home arrow | PageUp hide | PageDown source | Insert copy fen.
]]

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local LocalPlayer = Players.LocalPlayer
local Chess = ReplicatedStorage:WaitForChild("Chess")
local Client = ReplicatedStorage:WaitForChild("ChessClient")

local function req(mod)
	return require(mod)
end

local Engine = req(Chess:WaitForChild("Engine"))
local Bot = req(Chess:WaitForChild("Bot"))
local VortexBot = {}
do
    local module=Chess:FindFirstChild("VortexBot")
    if module then
        local ok,value=pcall(req,module)
        if ok and type(value)=="table" then VortexBot=value end
    end
end
local App = req(Client:WaitForChild("App"))

local GENV = type(getgenv) == "function" and getgenv() or _G
-- Teleport bootstrap: read the current saved script at the destination.
local teleportNotice = "Teleport: not configured"
local queueTeleport
if type(GENV.__RoCheeseDispose) == "function" then
    pcall(GENV.__RoCheeseDispose)
end
local TweenService = game:GetService("TweenService")
local HttpService = game:GetService("HttpService")
GENV.__RoCheeseGen = (GENV.__RoCheeseGen or 0) + 1
local GEN = GENV.__RoCheeseGen

local CFG = {
	W = 640,
	H = 480,
	MARGIN = 14,
	TOP = 84,
	DEPTH = 16,
	SECONDS = 0.8,
	CHILD_DEPTH = 8,
	CHILD_SECONDS = 0.06,
	VK_SHIFT = 160,
	PROMO = { q = 5, r = 4, b = 3, n = 2 },
	ARROW_1 = Color3.fromRGB(129, 182, 76),
	ARROW_2 = Color3.fromRGB(96, 156, 240),
	ARROW_3 = Color3.fromRGB(240, 200, 70),
	DELAY_MIN = 0,
	DELAY_MAX = 20,
	DELAY_STEP = 0.5,
}

local PRESETS = {
	legit = {
		label = "LEGIT",
		source = "local",
		mode = "elo",
		elo = 1500,
		noise = 0.32,
		blunder = 0.06,
		delayMin = 5,
		delayMax = 10,
	},
	raj = {
		label = "RAJ",
		source = "vortex",
		mode = "best",
		elo = 1800,
		noise = 0,
		blunder = 0,
		delayMin = 0.5,
		delayMax = 1.5,
	},
}

local RUNTIME = GENV.__RoCheese or {}
RUNTIME.SHOW_ARROW = RUNTIME.SHOW_ARROW ~= false
RUNTIME.AUTO = RUNTIME.AUTO == true
RUNTIME.HUMANIZE = RUNTIME.HUMANIZE ~= false
RUNTIME.SOURCE = (RUNTIME.SOURCE=="custom" or RUNTIME.SOURCE=="stockfish" or RUNTIME.SOURCE=="vortex") and RUNTIME.SOURCE or "local"
RUNTIME.MODE = (RUNTIME.MODE == "mindex" or RUNTIME.MODE == "vortex3" or RUNTIME.MODE == "vortex2" or RUNTIME.MODE == "vortex" or RUNTIME.MODE == "ultra" or RUNTIME.MODE == "best") and RUNTIME.MODE or "elo"
RUNTIME.ELO = tonumber(RUNTIME.ELO) or 1500
RUNTIME.NOISE = tonumber(RUNTIME.NOISE) or 0.32
RUNTIME.BLUNDER = tonumber(RUNTIME.BLUNDER) or 0.06
RUNTIME.DELAY_MIN = math.clamp(tonumber(RUNTIME.DELAY_MIN) or 5, CFG.DELAY_MIN, CFG.DELAY_MAX)
RUNTIME.DELAY_MAX = math.clamp(tonumber(RUNTIME.DELAY_MAX) or 10, CFG.DELAY_MIN, CFG.DELAY_MAX)
RUNTIME.STYLE = RUNTIME.STYLE == "raj" and "raj" or (RUNTIME.STYLE == "legit" and "legit" or "custom")
GENV.__RoCheese = RUNTIME

local ARROW_KEYS = { "rc1", "rc2", "rc3" }

local S = {
	ply = -1,
	sessionId = nil,
	best = nil,
	san = nil,
	score = 0,
	depth = 0,
	second = nil,
	third = nil,
	secondSan = nil,
	thirdSan = nil,
	busy = false,
	autoAt = 0,
	thinkAt = 0,
	status = "loading",
}
local cancelAuxiliary
local function cancelThread(thread)
    if thread and thread~=coroutine.running() and type(task.cancel)=="function" then pcall(task.cancel,thread) end
end
local function cancelMoveWork()
    local job=S.pickJob
    S.pickJob=nil;S.picking=false;S.playing=false
    if job then job.cancelled=true;cancelThread(job.thread) end
end
local function searchElapsed()
    if S.searchRunning and S.analysisStarted then return math.max(0,os.clock()-S.analysisStarted) end
    return math.max(0,S.analysisDuration or 0)
end
GENV.__RoCheeseState = S

local connections = {}
local disposers = {}
local disposed = false
local function alive()
    return not disposed and GENV.__RoCheeseGen == GEN
end
local function cleanup(fn)
    table.insert(disposers, fn)
end
local function connect(signal, callback)
    local connection = signal:Connect(callback)
    table.insert(connections, connection)
    return connection
end
GENV.__RoCheeseDispose = function()
    if disposed then return end
    disposed = true
    for _, connection in ipairs(connections) do connection:Disconnect() end
    for _, fn in ipairs(disposers) do pcall(fn) end
end

-- Versioned profiles: validate untrusted JSON before applying any values.
local DEFAULTS = { UI_SCALE = 1, ARROW_ALPHA = 0.12, ARROW_WIDTH = 1, ARROW_GLOW = true, ARROW_LABELS = true, ARROW_TINT = true, ARROW_PALETTE = "Classic", BOARD_FLIPPED = false, TELEPORT = true, THEME = "Midnight", ACCENT = "Blue", ANIMATIONS = true }
DEFAULTS.ARROW_COLOR_1, DEFAULTS.ARROW_COLOR_2, DEFAULTS.ARROW_COLOR_3 = "", "", ""
DEFAULTS.V2_SECONDS, DEFAULTS.STOCKFISH_DEPTH = 18, 15
DEFAULTS.V3_SECONDS=12
DEFAULTS.MINDEX_SECONDS=2
DEFAULTS.CUSTOM_MODEL=""
DEFAULTS.BAR, DEFAULTS.BAR_SCALE, DEFAULTS.BAR_X, DEFAULTS.BAR_Y = false, 1, -1, -1
DEFAULTS.OPPONENT_MOVE, DEFAULTS.AUTO_BOARD = false, true
DEFAULTS.AUTO_ORIENTATION = true
DEFAULTS.AUTO = false
DEFAULTS.SOURCE, DEFAULTS.MODE, DEFAULTS.STYLE = "local", "elo", "legit"
DEFAULTS.ELO, DEFAULTS.NOISE, DEFAULTS.BLUNDER = 1500, 0.32, 0.06
DEFAULTS.DELAY_MIN, DEFAULTS.DELAY_MAX = 5, 10
DEFAULTS.SHOW_ARROW, DEFAULTS.HUMANIZE = true, true
local CONFIG_PATH = "rocheese_profiles_v2.json"
local store = GENV.__RoCheeseProfiles or { version = 2, active = 1, profiles = {} }
local configNotice = "Profiles: session memory"
if type(readfile) == "function" then
    local ok, data = pcall(function() return HttpService:JSONDecode(readfile(CONFIG_PATH)) end)
    if ok and type(data) == "table" and data.version == 2 and type(data.profiles) == "table" then
        store = data
        configNotice = "Profiles restored from disk"
    end
end
store.active = tostring(store.active or "1")
store.names = type(store.names) == "table" and store.names or {}
for id, profile in pairs(store.profiles) do
    if type(id) ~= "string" or type(profile) ~= "table" then
        store.profiles[id] = nil
    elseif type(store.names[id]) ~= "string" then
        store.names[id] = "Profile " .. id
    end
end
GENV.__RoCheeseProfiles = store
local function validated(data)
    local result = table.clone(DEFAULTS)
    if type(data) ~= "table" then return result end
    for key, default in pairs(DEFAULTS) do
        local value = data[key]
        if type(value) == type(default) then
            if type(value) ~= "number" or (value == value and math.abs(value) < math.huge) then
                result[key] = value
            end
        end
    end
    result.MINDEX_SECONDS=math.clamp(result.MINDEX_SECONDS,.8,10)
    result.V3_SECONDS=math.clamp(result.V3_SECONDS,1,45)
    result.V2_SECONDS=math.clamp(result.V2_SECONDS,5,45)
    result.STOCKFISH_DEPTH=math.clamp(math.floor(result.STOCKFISH_DEPTH),8,15)
    result.BAR_SCALE = math.clamp(result.BAR_SCALE,0.55,1.8)
    result.BAR_X, result.BAR_Y = math.clamp(result.BAR_X,-1,20000), math.clamp(result.BAR_Y,-1,20000)
    for i=1,3 do
        local k="ARROW_COLOR_"..i
        local hex=result[k]:gsub("#",""):upper()
        result[k]=hex:match("^%x%x%x%x%x%x$") and hex or ""
    end
    result.UI_SCALE = math.clamp(result.UI_SCALE, 0.65, 1.25)
    result.ARROW_ALPHA = math.clamp(result.ARROW_ALPHA, 0, 0.8)
    result.ARROW_WIDTH = math.clamp(result.ARROW_WIDTH, 0.5, 2)
    if result.ARROW_PALETTE ~= "Classic" and result.ARROW_PALETTE ~= "Candy" and result.ARROW_PALETTE ~= "Ice" then result.ARROW_PALETTE = "Classic" end
    result.SOURCE = (result.SOURCE=="custom" or result.SOURCE=="stockfish" or result.SOURCE=="vortex") and result.SOURCE or "local"
    result.MODE = (result.MODE == "mindex" or result.MODE == "vortex3" or result.MODE == "vortex2" or result.MODE == "vortex" or result.MODE == "ultra" or result.MODE == "best") and result.MODE or "elo"
    result.STYLE = (result.STYLE == "legit" or result.STYLE == "raj") and result.STYLE or "custom"
    result.ELO = math.clamp(result.ELO, 100, 3200)
    result.NOISE = math.clamp(result.NOISE, 0, 1)
    result.BLUNDER = math.clamp(result.BLUNDER, 0, 1)
    result.DELAY_MIN = math.clamp(result.DELAY_MIN, 0, 19.5)
    result.DELAY_MAX = math.clamp(result.DELAY_MAX, result.DELAY_MIN + 0.5, 20)
    if result.THEME ~= "Midnight" and result.THEME ~= "Graphite" and result.THEME ~= "Pearl" then result.THEME = "Midnight" end
    if result.ACCENT ~= "Blue" and result.ACCENT ~= "Mint" and result.ACCENT ~= "Violet" and result.ACCENT ~= "Rose" then result.ACCENT = "Blue" end
    return result
end
local function applyConfig(data)
    for key, value in pairs(validated(data)) do RUNTIME[key] = value end
end
applyConfig(store.profiles[tostring(store.active)] or RUNTIME)
RUNTIME.SHOW_ARROW=true -- Visible on launch; the user can still toggle this off.
local function persistProfiles()
    if type(writefile) ~= "function" then return "Saved in session (no writefile)" end
    local ok = pcall(function() writefile(CONFIG_PATH, HttpService:JSONEncode(store)) end)
    return ok and "Saved to disk" or "Disk write failed; saved in session"
end

queueTeleport = function()
    if not RUNTIME.TELEPORT then teleportNotice = "Teleport: disabled"; return end
    local queue = queue_on_teleport or queueonteleport
        or (type(syn) == "table" and syn.queue_on_teleport)
        or (type(fluxus) == "table" and fluxus.queue_on_teleport)
    if type(queue) ~= "function" then teleportNotice = "Teleport: executor unsupported"; return end
    local source = GENV.__RoCheeseTeleportSource
    if type(source) ~= "string" or source == "" then
        local path = GENV.__RoCheeseScriptPath or "rochess3.txt"
        local ok, contents = pcall(function() return readfile(path) end)
        if not ok or type(contents) ~= "string" or contents == "" then
            teleportNotice = "Teleport: save rochess3.txt in executor workspace"; return
        end
        source = string.format("repeat task.wait() until game:IsLoaded(); loadstring(readfile(%q))()", path)
    end
    if GENV.__RoCheeseQueuedSource == source then teleportNotice = "Teleport: queued"; return end
    local ok, err = pcall(queue, source)
    if ok then GENV.__RoCheeseQueuedSource = source end
    teleportNotice = ok and "Teleport: queued" or ("Teleport failed: " .. tostring(err))
end
queueTeleport()
connect(LocalPlayer.OnTeleport, function(state)
    if state == Enum.TeleportState.Started then queueTeleport() end
end)

-- board ------------------------------------------------------------------------------------

local arrowLayer, boardRect, calibration, firstCorner
local detectedRect
local customArrows = {}
local boardProbeAt = 0
local palettes = {
    Classic = {Color3.fromRGB(104,235,170),Color3.fromRGB(105,174,255),Color3.fromRGB(255,205,105)},
    Candy = {Color3.fromRGB(255,126,181),Color3.fromRGB(176,143,255),Color3.fromRGB(108,235,224)},
    Ice = {Color3.fromRGB(120,235,255),Color3.fromRGB(169,190,255),Color3.fromRGB(231,245,255)},
}
local function boardFlipped()
    local color=App.session and App.session.myColor
    if RUNTIME.AUTO_ORIENTATION and (color==1 or color==-1 or color=="white" or color=="black" or color=="w" or color=="b") then return color==-1 or color=="black" or color=="b" end
    return RUNTIME.BOARD_FLIPPED
end
local function colorFor(rank)
    local hex=RUNTIME["ARROW_COLOR_"..rank]
    if type(hex)=="string" and hex:match("^%x%x%x%x%x%x$") then
        return Color3.fromRGB(tonumber(hex:sub(1,2),16),tonumber(hex:sub(3,4),16),tonumber(hex:sub(5,6),16))
    end
    return (palettes[RUNTIME.ARROW_PALETTE] or palettes.Classic)[rank] or palettes.Classic[1]
end
local function effectivelyVisible(object)
    local current=object
    while current and current~=LocalPlayer do
        if current:IsA("GuiObject") and not current.Visible then return false end
        if current:IsA("ScreenGui") and not current.Enabled then return false end
        current=current.Parent
    end
    return true
end
local function probeBoard()
    if calibration or not RUNTIME.AUTO_BOARD or os.clock()<boardProbeAt then return end
    boardProbeAt=os.clock()+1.5
    local playerGui=LocalPlayer:FindFirstChild("PlayerGui")
    if not playerGui then return end
    local winner, bestScore
    for _,object in ipairs(playerGui:GetDescendants()) do
        if object:IsA("GuiObject") and not object:FindFirstAncestor("RoCheesePanel") then
            local size=object.AbsoluteSize
            if size.X>=160 and math.abs(size.X-size.Y)<=math.max(3,size.X*.015) and effectivelyVisible(object) then
                local cell=size.X/8
                local occupied={}
                -- Require a complete 8x8 grid; avatars and square menu cards do not qualify.
                for _,tile in ipairs(object:GetDescendants()) do
                    if tile:IsA("GuiObject") then
                        local ts=tile.AbsoluteSize
                        if math.abs(ts.X-cell)<cell*.09 and math.abs(ts.Y-cell)<cell*.09 then
                            local offset=tile.AbsolutePosition-object.AbsolutePosition
                            local x,y=math.floor(offset.X/cell+.5),math.floor(offset.Y/cell+.5)
                            if x>=0 and x<8 and y>=0 and y<8 and math.abs(offset.X-x*cell)<cell*.09 and math.abs(offset.Y-y*cell)<cell*.09 then occupied[y*8+x]=true end
                        end
                    end
                end
                local count=0; for _ in pairs(occupied) do count=count+1 end
                if count==64 then
                    local score=size.X
                    if not bestScore or score>bestScore then winner,bestScore=object,score end
                end
            end
        end
    end
    if winner then
        detectedRect={origin=winner.AbsolutePosition,size=winner.AbsoluteSize.X,source=winner,viewport=workspace.CurrentCamera and workspace.CurrentCamera.ViewportSize}
    end
end
local function clearArrows()
    for _, item in pairs(customArrows) do item:Destroy() end
    customArrows = {}
	local board = App.board
	if not board or type(board.setArrow) ~= "function" then
		return
	end
	for _, key in ARROW_KEYS do
		pcall(function()
			board:setArrow(key, nil)
			board:setTint(key, nil)
		end)
	end
	-- older build of this script used a single arrow key
	pcall(function()
		board:setArrow("rocheese", nil)
		board:setTint("rocheese", nil)
	end)
end

-- Custom screen overlay uses two user-selected board corners: no private board UI assumptions.
local function arrowPart(parent, position, size, color, alpha, rotation, round)
    local f = Instance.new("Frame")
    f.AnchorPoint = Vector2.new(0.5,0.5)
    f.Position, f.Size = UDim2.fromOffset(position.X,position.Y), UDim2.fromOffset(size.X,size.Y)
    f.BackgroundColor3, f.BackgroundTransparency = color, alpha
    f.BorderSizePixel, f.Rotation, f.Parent = 0, rotation or 0, parent
    if round then local c=Instance.new("UICorner"); c.CornerRadius=UDim.new(1,0); c.Parent=f end
    return f
end
local function drawArrow(key, uci, color, tint)
    if type(uci) ~= "string" then return end
    local from, to = Engine.algToIndex64(uci:sub(1,2)), Engine.algToIndex64(uci:sub(3,4))
    if not from or not to then return end
    local rank = table.find(ARROW_KEYS,key) or 1
    color = colorFor(rank)
    if not boardRect or not arrowLayer then
        local board = App.board
        if board then pcall(function()
            board:setArrow(key,from,to,color,RUNTIME.ARROW_ALPHA)
            board:setTint(key,(tint and RUNTIME.ARROW_TINT) and to or nil,color,0.7)
        end) end
        return
    end
    local signature = table.concat({uci,RUNTIME.ARROW_PALETTE,tostring(RUNTIME.ARROW_WIDTH),tostring(RUNTIME.ARROW_ALPHA),tostring(RUNTIME.ARROW_GLOW),tostring(RUNTIME.ARROW_LABELS),tostring(RUNTIME.ARROW_TINT),tostring(RUNTIME.BOARD_FLIPPED)},":")
    if customArrows[key] and customArrows[key]:GetAttribute("Signature") == signature then return end
    if customArrows[key] then customArrows[key]:Destroy() end
    local root=Instance.new("Frame"); root.Size=UDim2.fromScale(1,1); root.BackgroundTransparency=1; root.Parent=arrowLayer
    root:SetAttribute("Signature",signature); customArrows[key]=root
    local cell=boardRect.size/8
    local function center(index)
        local file=(index-1)%8; local row=7-math.floor((index-1)/8)
        if RUNTIME.BOARD_FLIPPED then file=7-file; row=7-row end
        return boardRect.origin+Vector2.new((file+0.5)*cell,(row+0.5)*cell)
    end
    local a,b=center(from),center(to)
    local delta=b-a; if delta.Magnitude<1 then return end
    local unit=delta.Unit; local angle=math.deg(math.atan2(delta.Y,delta.X))
    local width=math.max(3,cell*0.10*RUNTIME.ARROW_WIDTH)
    local tip=b-unit*cell*0.12; local start=a+unit*cell*0.14
    local finish=tip-unit*width*1.2; local mid=(start+finish)/2
    local length=(finish-start).Magnitude
    if RUNTIME.ARROW_TINT then arrowPart(root,b,Vector2.new(cell*0.86,cell*0.86),color,0.86,0,true) end
    local function shaft(w,alpha)
        arrowPart(root,mid,Vector2.new(length,w),color,alpha,angle,true)
        for _,sign in ipairs({-1,1}) do
            local theta=math.rad(angle+180+sign*36)
            local arm=Vector2.new(math.cos(theta),math.sin(theta))*width*3
            arrowPart(root,tip+arm/2,Vector2.new(arm.Magnitude,w),color,alpha,angle+180+sign*36,true)
        end
    end
    if RUNTIME.ARROW_GLOW then shaft(width*2.8,0.92); shaft(width*1.8,0.80) end
    shaft(width,RUNTIME.ARROW_ALPHA)
    arrowPart(root,mid,Vector2.new(length,width*0.22),Color3.new(1,1,1),0.65,angle,true)
    local ring=arrowPart(root,a,Vector2.new(width*2.3,width*2.3),color,1,0,true)
    local stroke=Instance.new("UIStroke"); stroke.Color=color; stroke.Thickness=2; stroke.Transparency=RUNTIME.ARROW_ALPHA; stroke.Parent=ring
    if RUNTIME.ARROW_LABELS then
        local badge=arrowPart(root,a,Vector2.new(21,21),Color3.fromRGB(18,24,38),0.05,0,true)
        local text=Instance.new("TextLabel"); text.Size=UDim2.fromScale(1,1); text.BackgroundTransparency=1
        text.Text=tostring(rank); text.TextColor3=color; text.TextSize=12; text.Font=Enum.Font.GothamBold; text.Parent=badge
    end
end

-- engine ----------------------------------------------------------------------------------

local function buildEngine(startFen, moves)
	local engine = Engine.new(startFen)
	for _, uci in ipairs(moves) do
		local mv = engine:moveFromUci(uci)
		if not mv then
			return nil
		end
		engine:play(mv)
	end
	return engine
end

local function legalMoves(engine)
    local list,state,first=engine:legalMoves()
    if type(list)=="table" then return ipairs(list) end
    return list,state,first
end
local function normalizeResult(result,engine)
    if type(result)=="string" then result={best=result} end
    if type(result)~="table" then return nil end
    local move=result.best or result.bestMove or result.uci or result.move or (type(result.pv)=="table" and result.pv[1])
    if type(move)=="table" and engine then
        local ok,uci=pcall(engine.moveToUci,engine,move)
        move=ok and uci or nil
    end
    if type(move)~="string" then return nil end
    move=move:lower():match("^%s*([a-h][1-8][a-h][1-8][qrbn]?)%s*$")
    if not move then return nil end
    local pv={move}
    if type(result.pv)=="table" then
        pv={}
        for _,uci in ipairs(result.pv) do
            if type(uci)=="string" and uci:match("^[a-h][1-8][a-h][1-8][qrbn]?$") then pv[#pv+1]=uci else break end
        end
        if pv[1]~=move then pv={move} end
    end
    return {best=move,score=result.score,depth=result.depth,pv=pv}
end
local function sanOf(engine, uci)
	if not engine or type(uci)~="string" then
		return nil
	end
	local want = string.lower(uci)
	for _, mv in legalMoves(engine) do
		local u = engine:moveToUci(mv)
		if u and string.lower(u) == want then
			return engine:moveToSan(mv)
		end
	end
	return nil
end

-- Position recognition uses the game's FEN, never theme-dependent piece images.
local PIECE_VALUE={p=100,n=320,b=330,r=500,q=900,k=20000}
local PIECE_NAME={p="пешка",n="конь",b="слон",r="ладья",q="ферзь",k="король"}
local function recognizeFen(fen)
    if type(fen)~="string" then return nil end
    local placement,turn=fen:match("^(%S+)%s+([wb])")
    if not placement then return nil end
    local board,counts={}, {[1]={},[-1]={}}
    local rank,file,rows=7,0,1
    for char in placement:gmatch(".") do
        if char=="/" then
            if file~=8 or rank<=0 then return nil end
            rank,file,rows=rank-1,0,rows+1
        elseif char:match("^[1-8]$") then file=file+tonumber(char)
        elseif PIECE_VALUE[char:lower()] then
            if file>=8 then return nil end
            local kind,color=char:lower(),char==char:upper() and 1 or -1
            board[rank*8+file+1]={kind=kind,color=color}
            counts[color][kind]=(counts[color][kind] or 0)+1
            file=file+1
        else return nil end
        if file>8 then return nil end
    end
    if rows~=8 or rank~=0 or file~=8 then return nil end
    return {board=board,counts=counts,turn=turn=="w" and 1 or -1,fen=fen}
end
local function recognizeEngine(engine)
    local ok,fen=pcall(function() return engine:getFen() end)
    return ok and recognizeFen(fen) or nil
end
local function recognitionText(info)
    if not info then return "Доска не прочитана: FEN недоступен. Визуальное распознавание не заявляется." end
    local names={k="Кр",q="Ф",r="Л",b="С",n="К",p="п"}
    local rows={"Данные RoChess / ходят "..(info.turn==1 and "белые" or "чёрные")}
    for _,color in ipairs({1,-1}) do
        local pieces={}
        for square=1,64 do
            local piece=info.board[square]
            if piece and piece.color==color then
                pieces[#pieces+1]=names[piece.kind]..string.char(97+(square-1)%8)..(math.floor((square-1)/8)+1)
            end
        end
        rows[#rows+1]=(color==1 and "Белые: " or "Чёрные: ")..table.concat(pieces,"  ")
    end
    return table.concat(rows,"\n")
end
-- External engine is opt-in through Engine > STOCKFISH ONLINE.
-- This sends only the selected live FEN, not player/account identifiers.
local stockfishCache={}
local stockfishBackoff=0
local function engineFen(engine)
    local ok,fen=pcall(function()return engine:getFen()end)
    if ok and recognizeFen(fen) then return fen end
    return nil
end
local function positionSnapshot(session)
    if not session then return nil end
    local ok,rebuilt=pcall(buildEngine,session.startFen,table.clone(session.ucis or {}))
    local historyFen=ok and rebuilt and engineFen(rebuilt)
    local liveFen=session.engine and engineFen(session.engine)
    local viewed=tonumber(session.viewPly)
    local historical=viewed~=nil and viewed~=#(session.ucis or {})
    local useLive=liveFen~=nil and not historical
    local fen=useLive and liveFen or (historyFen or liveFen)
    if not fen then return nil end
    return {fen=fen,engine=useLive and session.engine or (historyFen and rebuilt or session.engine),
        source=useLive and "FEN игрового движка RoChess" or "история ходов RoChess",
        mismatch=historyFen and liveFen and historyFen:match("^([^ ]+ [^ ]+ [^ ]+ [^ ]+)")~=liveFen:match("^([^ ]+ [^ ]+ [^ ]+ [^ ]+)") or false,
        info=recognizeFen(fen)}
end
local function stockfishResult(data,depth)
    if type(data)~="table" or data.success~=true then return nil end
    local best=type(data.bestmove)=="string" and data.bestmove:match("bestmove%s+([a-h][1-8][a-h][1-8][qrbn]?)")
    if not best then return nil end
    local score
    if tonumber(data.mate) then
        local mate=tonumber(data.mate)
        score=(mate<0 and -1 or 1)*(30000-math.min(99,math.abs(mate))*100)
    elseif tonumber(data.evaluation) then score=tonumber(data.evaluation)*100 end
    if score and (score~=score or math.abs(score)==math.huge) then score=nil end
    local pv={}
    if type(data.continuation)=="string" then
        for uci in data.continuation:gmatch("%S+") do
            if uci:match("^[a-h][1-8][a-h][1-8][qrbn]?$") then pv[#pv+1]=uci else break end
        end
    end
    if pv[1]~=best then pv={best} end
    return {best=best,score=score,depth=tonumber(data.depth) or depth,pv=pv,provider="Stockfish Online"}
end
local function analyseStockfish(fen,depth,current)
    if os.clock()<stockfishBackoff or os.clock()<(S.externalSkipUntil or 0) then return nil,"service cooldown" end
    depth=math.clamp(math.floor(depth or 15),8,15)
    local key=fen.."|"..depth
    if stockfishCache[key] then return table.clone(stockfishCache[key]) end
    local url="https://stockfish.online/api/s/v2.php?fen="..HttpService:UrlEncode(fen).."&depth="..depth
    local requestFn=request or http_request or (type(syn)=="table" and syn.request) or (type(http)=="table" and http.request)
    local ok,body,status=pcall(function()
        if type(requestFn)=="function" then
            local response=requestFn({Url=url,Method="GET",Timeout=8,Headers={Accept="application/json"}})
            return response and (response.Body or response.body),response and (response.StatusCode or response.status_code)
        end
        return game:HttpGet(url),200
    end)
    if current and not current() then return nil,"obsolete" end
    if not ok or type(body)~="string" then return nil,"HTTP unavailable" end
    status=tonumber(status) or 200
    if status==429 then stockfishBackoff=os.clock()+60;return nil,"rate limited" end
    if status<200 or status>=300 then return nil,"HTTP "..status end
    local decoded,data=pcall(HttpService.JSONDecode,HttpService,body)
    local result=decoded and stockfishResult(data,depth)
    if not result then return nil,"invalid engine response" end
    local legalOk,isLegal=pcall(function()
        local engine=Engine.new(fen)
        for _,move in legalMoves(engine) do if engine:moveToUci(move)==result.best then return true end end
        return false
    end)
    if not legalOk or not isLegal then return nil,"response does not match live legal moves" end
    if current and not current() then return nil,"obsolete" end
    if (S.stockfishCacheCount or 0)>=128 then stockfishCache={};S.stockfishCacheCount=0 end
    stockfishCache[key]=table.clone(result);S.stockfishCacheCount=(S.stockfishCacheCount or 0)+1
    return result
end

-- Models share the same position/legality adapter. No UI state is changed by
-- sandbox engine calls, and no shallow evaluation can override an engine move.
local MODEL_IDS={"best","mindex","local","vortex","ultra","vortex2","vortex3","stockfish"}
local MODEL_LABELS={best="BEST MOVE / selected source",mindex="mindex / BEST MOVE+",["local"]="Local Best",vortex="RoChess Vortex",ultra="Ultra Brain / Local",vortex2="Vortex 2.0 / Vortex",vortex3="Vortex 3.0 / Vortex",stockfish="Stockfish Online"}
local CUSTOM={}
do
local CUSTOM_MODELS={}
local CUSTOM_MODEL_PATH="rocheese_custom_models_v1.json"
local modelsLocked=function()return false end
local MODEL_CHECK={running=false,token=0,status="Выбери модель и нажми CHECK MODEL"}
local function finiteNumber(value,lo,hi,default)
    local n=tonumber(value)
    if not n or n~=n or math.abs(n)==math.huge then return default end
    return math.clamp(n,lo,hi)
end
local function validateCustomModel(data)
    if type(data)~="table" then return nil,"Model definition must be a JSON object" end
    local name=type(data.name)=="string" and data.name:match("^%s*(.-)%s*$")
    if not name or #name<1 or #name>80 or name:find("[%c]") then return nil,"Name: 1–80 bytes, no control characters" end
    local provider=data.provider or "local"
    if provider~="local" and provider~="vortex" and provider~="stockfish" and provider~="http" and provider~="callback" then
        return nil,"provider: local, vortex, stockfish, http or callback"
    end
    local result={name=name,provider=provider,depth=math.floor(finiteNumber(data.depth,1,64,20)),
        maxSeconds=finiteNumber(data.maxSeconds,.1,50,50),timeout=finiteNumber(data.timeout,1,90,65)}
    if provider=="http" then
        if type(data.url)~="string" or #data.url>2048 or not data.url:match("^https?://") or data.url:find("%s") then return nil,"HTTP model needs an http:// or https:// URL" end
        result.url=data.url
        result.method=type(data.method)=="string" and data.method:upper() or "POST"
        if result.method~="GET" and result.method~="POST" then return nil,"HTTP method: GET or POST" end
        if result.method=="GET" and not result.url:find("{fen}",1,true) then return nil,"GET URL must contain {fen}" end
        result.request=type(data.request)=="table" and data.request or {fen="{fen}",seconds="{seconds}",depth="{depth}"}
        local requestOk,encoded=pcall(HttpService.JSONEncode,HttpService,result.request)
        if not requestOk or #encoded>65536 then return nil,"request must be JSON-compatible and under 64 KiB" end
        local decodeOk,requestCopy=pcall(HttpService.JSONDecode,HttpService,encoded)
        if not decodeOk then return nil,"Cannot validate request JSON" end
        result.request=requestCopy
        result.headers={}
        if data.headers~=nil and type(data.headers)~="table" then return nil,"headers must be an object" end
        for key,value in pairs(data.headers or {}) do
            if type(key)~="string" or type(value)~="string" or key:find("[\r\n]") or value:find("[\r\n]") then return nil,"Invalid HTTP header" end
            result.headers[key]=value
        end
        result.movePath=type(data.movePath)=="string" and data.movePath or "bestmove"
        result.scorePath=type(data.scorePath)=="string" and data.scorePath or "score"
        result.pvPath=type(data.pvPath)=="string" and data.pvPath or "pv"
        result.depthPath=type(data.depthPath)=="string" and data.depthPath or "depth"
        result.scoreUnit=data.scoreUnit=="pawns" and "pawns" or "cp"
    elseif provider=="callback" then
        if type(data.callback)~="string" or not data.callback:match("^[%w_%-]+$") then return nil,"callback must name a function in getgenv().__RoCheeseCustomCallbacks" end
        result.callback=data.callback
    end
    return result
end
local function refreshCustomModels()
    while #MODEL_IDS>8 do table.remove(MODEL_IDS) end
    local ids={}
    for id,definition in pairs(CUSTOM_MODELS) do MODEL_LABELS[id]=definition.name.." / custom";ids[#ids+1]=id end
    table.sort(ids,function(a,b)return CUSTOM_MODELS[a].name<CUSTOM_MODELS[b].name end)
    for _,id in ipairs(ids) do MODEL_IDS[#MODEL_IDS+1]=id end
end
local function persistCustomModels()
    GENV.__RoCheeseCustomModels=CUSTOM_MODELS
    if type(writefile)~="function" then return "Saved in session memory" end
    local ok=pcall(function()writefile(CUSTOM_MODEL_PATH,HttpService:JSONEncode({version=1,models=CUSTOM_MODELS}))end)
    return ok and "Saved to "..CUSTOM_MODEL_PATH or "Saved in memory; file write failed"
end
local function saveCustomModel(data,id)
    if modelsLocked() then return nil,"Stop Test model / model check before editing" end
    local definition,reason=validateCustomModel(data)
    if not definition then return nil,reason end
    local count=0
    for key,existing in pairs(CUSTOM_MODELS) do
        count=count+1
        if key~=id and existing.name==definition.name then return nil,"A model with that name already exists" end
    end
    if not id and count>=32 then return nil,"Maximum 32 custom models" end
    if id and not CUSTOM_MODELS[id] then return nil,"Selected model no longer exists" end
    id=id or ("custom_"..HttpService:GenerateGUID(false))
    CUSTOM_MODELS[id]=definition;refreshCustomModels()
    if RUNTIME.CUSTOM_MODEL==id then S.ply=-1 end
    return id,persistCustomModels()
end
local function deleteCustomModel(id)
    if modelsLocked() then return false,"Stop Test model / model check before editing" end
    if not CUSTOM_MODELS[id] then return false,"Select a custom model first" end
    CUSTOM_MODELS[id]=nil;MODEL_LABELS[id]=nil;refreshCustomModels()
    if RUNTIME.CUSTOM_MODEL==id then RUNTIME.CUSTOM_MODEL="";RUNTIME.SOURCE="local";S.ply=-1 end
    return true,persistCustomModels()
end
do
    local saved=GENV.__RoCheeseCustomModels
    if type(readfile)=="function" then
        local ok,data=pcall(function()return HttpService:JSONDecode(readfile(CUSTOM_MODEL_PATH))end)
        if ok and type(data)=="table" and data.version==1 then saved=data.models end
    end
    if type(saved)=="table" then
        local count=0
        for id,definition in pairs(saved) do
            if count>=32 then break end
            if type(id)=="string" and id:match("^custom_[%w_%-]+$") then
                local valid=validateCustomModel(definition)
                if valid then CUSTOM_MODELS[id]=valid;count=count+1 end
            end
        end
    end
    refreshCustomModels()
end
local function responseField(data,path)
    if path=="" then return data end
    local value=data
    for key in tostring(path):gmatch("[^%.]+") do
        if type(value)~="table" then return nil end
        value=value[key]~=nil and value[key] or value[tonumber(key)]
    end
    return value
end
local function modelTemplate(value,variables,level)
    level=level or 0
    if level>12 then error("Request template nesting exceeds 12 levels") end
    if type(value)=="string" then
        local token=value:match("^{([%w_]+)}$")
        if token and variables[token]~=nil then return variables[token] end
        return value:gsub("{([%w_]+)}",function(key)return tostring(variables[key] or ("{"..key.."}"))end)
    end
    if type(value)=="table" then
        local result={};for key,item in pairs(value) do result[key]=modelTemplate(item,variables,level+1) end;return result
    end
    return value
end
local function customResponse(definition,raw,engine)
    local move=definition.provider=="http" and responseField(raw,definition.movePath) or raw
    if type(move)=="string" then move=move:match("bestmove%s+([a-h][1-8][a-h][1-8][qrbn]?)") or move end
    if definition.provider~="http" then return normalizeResult(move,engine) end
    local res=normalizeResult(move)
    if not res then return nil,"No UCI move at response field '"..definition.movePath.."'" end
    local score=tonumber(responseField(raw,definition.scorePath))
    if score and score==score and math.abs(score)<math.huge then res.score=score*(definition.scoreUnit=="pawns" and 100 or 1) end
    res.depth=tonumber(responseField(raw,definition.depthPath)) or definition.depth
    local pv=responseField(raw,definition.pvPath)
    if type(pv)=="string" then local moves={};for uci in pv:gmatch("%S+") do moves[#moves+1]=uci end;pv=moves end
    if type(pv)=="table" then
        local normalized=normalizeResult({best=res.best,pv=pv})
        res.pv=normalized and normalized.pv or {res.best}
    end
    return res
end
local function runCustomModel(id,fen,seconds,current)
    local definition=CUSTOM_MODELS[id]
    if not definition then return nil,"Custom model not found; open My models" end
    local budget=math.min(seconds,definition.maxSeconds)
    local raw
    if definition.provider=="http" then
        if definition.url:find("your%-server%.example") then return nil,"Replace the example URL with your actual engine endpoint" end
        local vars={fen=fen,seconds=budget,depth=definition.depth}
        local urlVars={fen=HttpService:UrlEncode(fen),seconds=tostring(budget),depth=tostring(definition.depth)}
        local url=modelTemplate(definition.url,urlVars)
        local headers=table.clone(definition.headers)
        headers.Accept=headers.Accept or "application/json"
        local body
        if definition.method=="POST" then headers["Content-Type"]=headers["Content-Type"] or "application/json";body=HttpService:JSONEncode(modelTemplate(definition.request,vars)) end
        local requestFn=request or http_request or (type(syn)=="table" and syn.request) or (type(http)=="table" and http.request)
        local ok,response=pcall(function()
            if type(requestFn)=="function" then return requestFn({Url=url,Method=definition.method,Headers=headers,Body=body,Timeout=definition.timeout}) end
            if definition.method=="GET" then return {StatusCode=200,Body=game:HttpGet(url)} end
            error("POST requires an executor HTTP request function")
        end)
        if current and not current() then return nil,"obsolete" end
        if not ok or type(response)~="table" then return nil,"HTTP request failed; check executor HTTP support and endpoint" end
        local status=tonumber(response.StatusCode or response.status_code) or 200
        if status<200 or status>=300 then return nil,"HTTP "..status.." from custom model" end
        local decoded,value=pcall(HttpService.JSONDecode,HttpService,response.Body or response.body or "")
        if not decoded then return nil,"Custom endpoint did not return valid JSON" end
        raw=value
    elseif definition.provider=="callback" then
        local callbacks=GENV.__RoCheeseCustomCallbacks
        local fn=type(callbacks)=="table" and callbacks[definition.callback]
        if type(fn)~="function" then return nil,"Callback '"..definition.callback.."' is not registered" end
        local ok,value=pcall(fn,{fen=fen,seconds=budget,depth=definition.depth})
        if not ok then return nil,"Custom callback failed: "..tostring(value) end
        raw=value
    elseif definition.provider=="stockfish" then
        return analyseStockfish(fen,math.clamp(definition.depth,8,15),current)
    else
        local module=definition.provider=="vortex" and VortexBot or Bot
        if type(module.analyse)~="function" then return nil,definition.provider..".analyse is unavailable" end
        local ok,value=pcall(module.analyse,fen,{},budget,definition.depth)
        if not ok then return nil,definition.provider..": "..tostring(value) end
        raw=value
    end
    if current and not current() then return nil,"obsolete" end
    local engineOk,engine=pcall(Engine.new,fen)
    local result,reason=customResponse(definition,raw,engineOk and engine or nil)
    if not result then return nil,reason or "Custom model did not return a UCI best move" end
    return result
end


    CUSTOM.MODELS=CUSTOM_MODELS
    CUSTOM.CHECK=MODEL_CHECK
    CUSTOM.number=finiteNumber
    CUSTOM.validate=validateCustomModel
    CUSTOM.save=saveCustomModel
    CUSTOM.delete=deleteCustomModel
    CUSTOM.responseField=responseField
    CUSTOM.template=modelTemplate
    CUSTOM.parse=customResponse
    CUSTOM.run=runCustomModel
    CUSTOM.isLocked=function()return modelsLocked()end
    CUSTOM.setLocked=function(fn)modelsLocked=fn end
end

local function listLegal(engine)
    local list,set={},{}
    for _,move in legalMoves(engine) do
        local uci=engine:moveToUci(move)
        if type(uci)=="string" then list[#list+1]=uci;set[uci]=move end
    end
    return list,set
end
local function positionPhase(info)
    local weight={n=1,b=1,r=2,q=4}
    local phase,count=0,0
    for _,piece in pairs(info and info.board or {}) do phase=phase+(weight[piece.kind] or 0);count=count+1 end
    return math.min(24,phase),count
end
local function clockSeconds(session)
    if not session or not session.clocks then return nil end
    local color=session.myColor
    local numeric=(color==1 or color=="white" or color=="w") and 1 or ((color==-1 or color=="black" or color=="b") and -1 or nil)
    local ms=tonumber(session.clocks[color] or session.clocks[numeric])
    return ms and math.max(0,ms/1000) or nil
end
local function v3Budget(session,info,cap)
    cap=cap or 12
    local remaining=clockSeconds(session)
    if not remaining then return cap end
    local phase=positionPhase(info)
    local movesLeft=math.clamp(12+phase*1.2,12,40)
    local reserve=math.min(remaining*.25,math.max(1.5,remaining*.07))
    local increment=math.max(0,tonumber(session.incrementMs) or 0)/1000
    local budget=(math.max(0,remaining-reserve)/movesLeft)+increment*.65
    return math.max(.02,math.min(cap,budget,remaining*.18))
end
local function selectProvider(spec,startFen,moves,seconds,depth,current,snapshot,strict)
    local function valid()return alive() and (not current or current())end
    local lastError
    local function usable(raw)
        local res=normalizeResult(raw,snapshot and snapshot.engine)
        if not res then return nil end
        if snapshot and snapshot.engine then
            local ok,list,set=pcall(listLegal,snapshot.engine)
            if ok and not set[res.best] then lastError="engine returned an illegal move";return nil end
        end
        return res
    end
    if spec.source=="custom" then
        if not snapshot then return nil,nil,"Custom model requires a readable FEN" end
        local result,reason=CUSTOM.run(spec.customId,snapshot.fen,seconds,current)
        if not valid() then return nil,nil,"obsolete" end
        local res=result and usable(result)
        if res then return res,MODEL_LABELS[spec.customId] or "Custom model" end
        return nil,nil,reason or lastError or "Custom model failed validation"
    end
    if spec.source=="stockfish" and snapshot then
        local res,reason=analyseStockfish(snapshot.fen,spec.stockfishDepth or 15,current)
        if not valid() then return nil,nil,"obsolete" end
        if res then return res,"Stockfish Online" end
        lastError=reason
        if strict then return nil,nil,"Stockfish Online: "..tostring(reason) end
    end
    local preferVortex=spec.source=="vortex" or spec.source=="stockfish" or spec.mode=="vortex" or spec.mode=="vortex2" or spec.mode=="vortex3"
    if preferVortex then
        local ok,raw
        if type(VortexBot.analyse)=="function" then ok,raw=pcall(VortexBot.analyse,startFen,moves,seconds,depth) else ok,raw=false,"VortexBot.analyse is unavailable in this RoChess client" end
        if not valid() then return nil,nil,"obsolete" end
        local res=ok and usable(raw)
        if res then return res,"RoChess Vortex",lastError end
        lastError=lastError or (ok and "invalid Vortex result" or tostring(raw))
        if strict then return nil,nil,"Vortex: "..tostring(lastError) end
    end
    local ok,raw
    if type(Bot.analyse)=="function" then ok,raw=pcall(Bot.analyse,startFen,moves,seconds,depth) else ok,raw=false,"Bot.analyse is unavailable in this RoChess client" end
    if not valid() then return nil,nil,"obsolete" end
    local res=ok and usable(raw)
    if res then return res,"RoChess Local",lastError end
    return nil,nil,lastError or (ok and "no engine result" or tostring(raw))
end
-- mindex keeps the BEST MOVE provider and root position. Extra time is used
-- only to deepen that same root search; child heuristics never replace it.
local MINDEX={}
do
    local function complexity(info,engine)
        local _,pieces=positionPhase(info)
        local legal=listLegal(engine)
        local captures=0
        for _,uci in ipairs(legal) do
            local from,to=Engine.algToIndex64(uci:sub(1,2)),Engine.algToIndex64(uci:sub(3,4))
            local mover,target=info.board[from],info.board[to]
            if mover and target and mover.color~=target.color or #uci==5 then captures=captures+1 end
        end
        return pieces<=10 or #legal>=26 or captures>=2,#legal,pieces
    end
    function MINDEX.gameBudget(session,info,engine,cap)
        cap=cap or RUNTIME.MINDEX_SECONDS or 2
        local difficult=true
        if info and engine then local ok,value=pcall(complexity,info,engine);if ok then difficult=value end end
        local budget=difficult and cap or math.min(CFG.SECONDS,cap)
        local remaining=clockSeconds(session)
        if remaining then
            local phase=positionPhase(info)
            local movesLeft=math.clamp(12+phase*1.2,12,40)
            local allocation=remaining*.9/movesLeft
            if remaining>10 then allocation=math.max(math.min(CFG.SECONDS,cap),allocation) end
            budget=math.min(budget,allocation)
        end
        return math.max(.02,budget)
    end
    function MINDEX.run(spec,fen,seconds,current,strict,snapshot)
        local began=os.clock()
        local function valid()return alive() and (not current or current())end
        if not snapshot then
            local ok,engine=pcall(Engine.new,fen)
            if not ok then return nil,nil,tostring(engine) end
            snapshot={fen=fen,engine=engine,info=recognizeEngine(engine)}
        end
        if not snapshot.info then return nil,nil,"mindex: unreadable position" end
        local baseSpec=table.clone(spec);baseSpec.mode="best"
        local budget=math.max(.02,seconds or CFG.SECONDS)
        local baseline,provider,warning=selectProvider(baseSpec,fen,{},math.min(CFG.SECONDS,budget),CFG.DEPTH,current,snapshot,strict)
        if not valid() then return nil,nil,"obsolete" end
        if not baseline then return nil,nil,warning end
        local scanOk,difficult,legalCount,pieces=pcall(complexity,snapshot.info,snapshot.engine)
        if not scanOk then local phase,count=positionPhase(snapshot.info);difficult,legalCount,pieces=false,0,count end
        local meta={provider=provider,warning=warning,legalCount=legalCount,pieceCount=pieces,budget=budget,
            baselineDepth=tonumber(baseline.depth) or 0,extended=false,extensionAccepted=false}
        local remaining=budget-(os.clock()-began)
        local winningMate=(tonumber(baseline.score) or 0)*snapshot.info.turn>=29900
        if not difficult or winningMate or remaining<.10 or baseSpec.source=="stockfish" then return baseline,meta end
        meta.extended=true
        local ok,deeper,deeperProvider=pcall(selectProvider,baseSpec,fen,{},remaining,pieces<=10 and 24 or 22,current,snapshot,strict)
        if not valid() then return nil,nil,"obsolete" end
        -- A failed, shallower or different-provider extension leaves the fast
        -- BEST MOVE result intact. Scores at unequal depths are not compared.
        if ok and deeper and deeperProvider==provider and (tonumber(deeper.depth) or 0)>meta.baselineDepth then
            local checked,legalList,legalSet=pcall(listLegal,snapshot.engine)
            if checked and legalSet[deeper.best] then meta.extensionAccepted=true;return deeper,meta end
        end
        return baseline,meta
    end
end

local function analyseModel(spec,fen,seconds,current,strict)
    if spec.mode=="mindex" then return MINDEX.run(spec,fen,seconds,current,strict) end
    local function stage(n,text)if spec.onStage then spec.onStage(n,text) end end
    stage(1,"анализ фигур")
    local engine=Engine.new(fen)
    local info=recognizeEngine(engine)
    if not info then return nil,nil,"cannot read model position" end
    stage(2,"построение легальных ходов")
    local moves,set=listLegal(engine)
    if #moves==0 then return nil,{legalCount=0,info=info},"terminal" end
    local phase,pieces=positionPhase(info)
    local depth=spec.mode=="vortex3" and 32 or (spec.mode=="ultra" and 24 or (spec.mode=="best" and 16 or (spec.mode=="vortex" and 28 or 30)))
    stage(3,"поиск среди "..#moves.." легальных ходов")
    local res,provider,warning=selectProvider(spec,fen,{},seconds,depth,current,{fen=fen,engine=engine,info=info},strict)
    if current and not current() then return nil,nil,"obsolete" end
    if not res then return nil,nil,warning end
    -- Final validation uses this exact board, including checks, pins, castling,
    -- en passant and promotion as implemented by RoChess.Engine.
    if not set[res.best] then return nil,nil,"selected move is not legal in this position" end
    stage(4,"ход проверен; готов к отправке")
    return res,{provider=provider,legalCount=#moves,pieceCount=pieces,phase=phase,info=info,warning=warning,budget=seconds}
end
local function modelSpec(id)
    if id=="best" or id=="mindex" then
        return {id=id,source=RUNTIME.SOURCE,mode=id=="best" and "best" or "mindex",stockfishDepth=RUNTIME.STOCKFISH_DEPTH,customId=RUNTIME.CUSTOM_MODEL}
    end
    if CUSTOM.MODELS[id] then return {id=id,source="custom",customId=id,mode="custom"} end
    return {id=id,source=id=="stockfish" and "stockfish" or ((id=="local" or id=="ultra") and "local" or "vortex"),
        mode=id=="local" and "best" or id,stockfishDepth=RUNTIME.STOCKFISH_DEPTH}
end
local function analyseBest(startFen,moves,seconds,depth,current,snapshot)
    local spec={source=RUNTIME.SOURCE,mode=RUNTIME.MODE,stockfishDepth=RUNTIME.STOCKFISH_DEPTH,customId=RUNTIME.CUSTOM_MODEL}
    if RUNTIME.MODE=="mindex" then
        if not snapshot then return nil end
        local res,meta,err=MINDEX.run(spec,snapshot.fen,seconds,current,false,snapshot)
        if current and not current() then return nil end
        if not res then S.engineNotice=err;return nil end
        S.actualEngine=meta.provider;S.mindexMeta=meta
        return res
    end
    if RUNTIME.MODE=="vortex3" then
        if not snapshot then return nil end
        if spec.source=="stockfish" and (clockSeconds(App.session) or 100)<6 then
            spec.source="vortex";S.engineNotice="Short clock: native engine selected"
        end
        spec.onStage=function(n,text)
            if not current or current() then S.v3Step=n.." / "..text end
        end
        local res,meta,err=analyseModel(spec,snapshot.fen,seconds,current,false)
        if current and not current() then return nil end
        if not res then S.engineNotice=err;return nil end
        S.actualEngine,S.v3Meta=meta.provider,meta
        S.v3Step="4 / легальный ход выбран; ожидание отправки"
        return res
    end
    local res,provider,warning=selectProvider(spec,startFen,moves,seconds,depth,current,snapshot,false)
    if current and not current() then return nil end
    if provider then S.actualEngine=provider end
    S.engineNotice=warning
    return res
end

-- Private model laboratory. It only mutates its own Engine instance. It never
-- invokes App.onBoardMove, App.playMove, remotes for moves, or the live board UI.
local LAB={running=false,token=0,modelA="best",modelB="mindex",games=2,seconds=5,sameAsGame=true,grace=15,strictTime=false,maxPlies=160,lastError="",diagnostics={},
    status="Выбери две модели и нажми START",score={a=0,b=0,draws=0,errors=0,unfinished=0,overruns=0},records={},moves={},fen=nil,game=0}
local suspendAssistant
local START_FEN="rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1"
local function modelBudget(id,fen,ceiling,sameAsGame,spec)
    ceiling=math.clamp(tonumber(ceiling) or 5,.25,50)
    if not sameAsGame then return ceiling end
    local budget=ceiling
    if id=="best" or id=="local" then budget=CFG.SECONDS
    elseif id=="mindex" then
        local engine=Engine.new(fen)
        budget=MINDEX.gameBudget(nil,recognizeEngine(engine),engine,spec and spec.gameLimits and spec.gameLimits.mindex or RUNTIME.MINDEX_SECONDS)
    elseif id=="ultra" then budget=3
    elseif id=="vortex" then budget=4
    elseif id=="vortex2" then budget=spec and spec.gameLimits and spec.gameLimits.vortex2 or RUNTIME.V2_SECONDS
    elseif id=="vortex3" then budget=spec and spec.gameLimits and spec.gameLimits.vortex3 or RUNTIME.V3_SECONDS
    elseif CUSTOM.MODELS[id] then budget=CUSTOM.MODELS[id].maxSeconds end
    return math.min(ceiling,budget)
end
local function squareAttacked(info,target,color)
    local tx,ty=(target-1)%8,math.floor((target-1)/8)
    for square,piece in pairs(info.board) do
        if piece.color==color then
            local x,y=(square-1)%8,math.floor((square-1)/8)
            local dx,dy=tx-x,ty-y
            if piece.kind=="p" and dy==color and math.abs(dx)==1 then return true end
            if piece.kind=="n" and (math.abs(dx)==1 and math.abs(dy)==2 or math.abs(dx)==2 and math.abs(dy)==1) then return true end
            if piece.kind=="k" and math.max(math.abs(dx),math.abs(dy))==1 then return true end
            local diagonal=math.abs(dx)==math.abs(dy) and dx~=0
            local straight=(dx==0)~=(dy==0)
            if (piece.kind=="b" and diagonal) or (piece.kind=="r" and straight) or (piece.kind=="q" and (diagonal or straight)) then
                local sx,sy=dx==0 and 0 or (dx>0 and 1 or -1),dy==0 and 0 or (dy>0 and 1 or -1)
                local nx,ny=x+sx,y+sy
                local clear=true
                while nx~=tx or ny~=ty do if info.board[ny*8+nx+1] then clear=false;break end;nx,ny=nx+sx,ny+sy end
                if clear then return true end
            end
        end
    end
    return false
end
local function labTerminal(engine,fen,repetitions)
    local info=recognizeFen(fen)
    if not info then return {kind="error",reason="Unreadable FEN"} end
    local moves=listLegal(engine)
    if #moves==0 then
        local king
        for square,piece in pairs(info.board) do if piece.color==info.turn and piece.kind=="k" then king=square;break end end
        if king and squareAttacked(info,king,-info.turn) then return {kind="win",winner=-info.turn,reason="checkmate"} end
        return {kind="draw",reason="stalemate"}
    end
    if repetitions>=3 then return {kind="draw",reason="threefold repetition / automatic test claim"} end
    local half=tonumber(fen:match("^%S+%s+[wb]%s+%S+%s+%S+%s+(%d+)")) or 0
    if half>=100 then return {kind="draw",reason="50-move rule / automatic test claim"} end
    local minors,bishopColors={},{}
    local other=false
    for square,piece in pairs(info.board) do
        if piece.kind~="k" then
            minors[#minors+1]=piece.kind
            if piece.kind=="b" then bishopColors[((square-1)%8+math.floor((square-1)/8))%2]=true
            elseif piece.kind~="n" then other=true end
        end
    end
    if not other and (#minors==0 or #minors==1) then return {kind="draw",reason="insufficient material"} end
    local bishopsOnly=#minors>0
    for _,kind in ipairs(minors) do if kind~="b" then bishopsOnly=false end end
    if bishopsOnly and not (bishopColors[0] and bishopColors[1]) then return {kind="draw",reason="same-color bishops / dead position"} end
    return nil
end
local function repetitionKey(fen)return fen:match("^(%S+%s+%S+%s+%S+%s+%S+)") or fen end
local function labStop(reason)
    LAB.token=LAB.token+1
    LAB.running=false
    cancelThread(LAB.thread);LAB.thread=nil
    if LAB.currentRecord and not LAB.currentRecord.result then
        LAB.currentRecord.result="*";LAB.currentRecord.reason=reason or "stopped by user"
        LAB.score.unfinished=LAB.score.unfinished+1
    end
    LAB.status=reason or "Остановлено пользователем"
    S.ply=-1
end
local function labReport()
    local config=LAB.config or {a=LAB.modelA,b=LAB.modelB,seconds=LAB.seconds,maxPlies=LAB.maxPlies,games=LAB.games,grace=LAB.grace,strictTime=LAB.strictTime}
    local lines={"RoChess model test — local sandbox", "A: "..(config.aName or MODEL_LABELS[config.a] or tostring(config.a)),"B: "..(config.bName or MODEL_LABELS[config.b] or tostring(config.b)),
        string.format("Requested compute budget %.2fs; network/bridge grace %.2fs; strict wall time %s; max %d plies; requested games %d",config.seconds,config.grace or 15,tostring(config.strictTime or false),config.maxPlies,config.games),
        string.format("A wins %d; B wins %d; draws %d; errors %d; unfinished %d",LAB.score.a,LAB.score.b,LAB.score.draws,LAB.score.errors,LAB.score.unfinished),""}
    for _,record in ipairs(LAB.records) do
        lines[#lines+1]=string.format("[Game %d] [White %s] [Black %s] [Result %s]",record.game,(record.whiteSlot or "").." "..record.white,(record.blackSlot or "").." "..record.black,record.result or "*")
        lines[#lines+1]="Start FEN: "..record.startFen
        local sans={}
        local side=record.startFen:match("^%S+%s+([wb])") or "w"
        local number=tonumber(record.startFen:match("(%d+)%s*$")) or 1
        for _,move in ipairs(record.moves) do
            if side=="w" then sans[#sans+1]=number.."." else if #sans==0 then sans[#sans+1]=number.."..." end end
            sans[#sans+1]=move.san or move.uci
            if side=="b" then number=number+1 end
            side=side=="w" and "b" or "w"
        end
        lines[#lines+1]=table.concat(sans," ").." "..(record.result or "*")
        lines[#lines+1]="Reason: "..tostring(record.reason or "in progress")
        for ply,move in ipairs(record.moves) do
            lines[#lines+1]=string.format("ply=%d model=%s provider=%s move=%s think=%.3fs depth=%s legal=%s pieces=%s overrun=%s budget=%s baselineDepth=%s extended=%s FEN=%s",ply,move.model,move.provider,move.uci,move.seconds,tostring(move.depth or "?"),tostring(move.legalCount or "?"),tostring(move.pieceCount or "?"),tostring(move.overrun or false),tostring(move.budget or "?"),tostring(move.baselineDepth or "?"),tostring(move.extensionAccepted or false),move.before)
        end
        lines[#lines+1]=""
    end
    lines[#lines+1]="Test timing: "..(config.sameAsGame and "same search policy as live game (per-model budgets)" or "equal requested budget")
    lines[#lines+1]="BEST MOVE source: "..tostring(config.bestSource or RUNTIME.SOURCE)
    lines[#lines+1]="Accuracy is not estimated by this sandbox; compare actual reviewed games."
    lines[#lines+1]="Overruns (continued within grace): "..tostring(LAB.score.overruns or 0)
    for _,diagnostic in ipairs(LAB.diagnostics or {}) do lines[#lines+1]=diagnostic end
    return table.concat(lines,"\n")
end
local function labStart()
    if CUSTOM.isLocked() then LAB.status="Stop the current test/check first";return end
    if not MODEL_LABELS[LAB.modelA] or not MODEL_LABELS[LAB.modelB] then LAB.status="Selected model was removed; select A/B again";return end
    if suspendAssistant then suspendAssistant() end
    LAB.token=LAB.token+1;local token=LAB.token
    LAB.running=true;LAB.records={};LAB.score={a=0,b=0,draws=0,errors=0,unfinished=0,overruns=0};LAB.currentRecord=nil;LAB.lastError="";LAB.diagnostics={}
    local config={a=LAB.modelA,b=LAB.modelB,games=LAB.games,seconds=math.clamp(tonumber(LAB.seconds) or 5,.25,50),sameAsGame=LAB.sameAsGame,bestSource=RUNTIME.SOURCE,bestCustom=RUNTIME.CUSTOM_MODEL,grace=LAB.grace,strictTime=LAB.strictTime,maxPlies=LAB.maxPlies,depth=RUNTIME.STOCKFISH_DEPTH,startFen=LAB.startFen or START_FEN}
    config.gameLimits={mindex=RUNTIME.MINDEX_SECONDS,vortex2=RUNTIME.V2_SECONDS,vortex3=RUNTIME.V3_SECONDS}
    config.aName=MODEL_LABELS[config.a] or config.a;config.bName=MODEL_LABELS[config.b] or config.b
    LAB.config=config
    local function current()return alive() and LAB.running and LAB.token==token end
    LAB.thread=task.spawn(function()
        local ok,err=pcall(function()
            for gameNumber=1,config.games do
                if not current() then return end
                local aWhite=gameNumber%2==1
                local whiteId,blackId=aWhite and config.a or config.b,aWhite and config.b or config.a
                local record={game=gameNumber,white=MODEL_LABELS[whiteId],black=MODEL_LABELS[blackId],whiteSlot=aWhite and "A" or "B",blackSlot=aWhite and "B" or "A",moves={},startFen=config.startFen}
                LAB.records[#LAB.records+1]=record;LAB.currentRecord=record;LAB.game=gameNumber;LAB.moves=record.moves
                local engine=Engine.new(config.startFen)
                local repetitions={}
                local outcome
                for ply=0,config.maxPlies do
                    if not current() then return end
                    local fen=engine:getFen();LAB.fen=fen
                    local key=repetitionKey(fen);repetitions[key]=(repetitions[key] or 0)+1
                    outcome=labTerminal(engine,fen,repetitions[key])
                    if outcome then break end
                    if ply==config.maxPlies then outcome={kind="unfinished",reason="training ply limit; not a draw"};break end
                    local info=recognizeFen(fen)
                    local id=info.turn==1 and whiteId or blackId
                    LAB.status=string.format("Game %d/%d · ply %d · %s считает",gameNumber,config.games,ply+1,MODEL_LABELS[id])
                    local spec=modelSpec(id);spec.stockfishDepth=config.depth;spec.gameLimits=config.gameLimits
                    if id=="best" or id=="mindex" then spec.source=config.bestSource;spec.customId=config.bestCustom end
                    local budget=modelBudget(id,fen,config.seconds,config.sameAsGame,spec)
                    LAB.currentBudget=budget
                    local started=os.clock();LAB.moveStarted=started;LAB.moveDeadline=started+budget+(config.strictTime and math.max(.15,budget*.1) or config.grace)
                    local callOk,result,meta,why=pcall(analyseModel,spec,fen,budget,current,true)
                    LAB.moveStarted=nil;LAB.moveDeadline=nil
                    if not current() then return end
                    local elapsed=os.clock()-started
                    local limit=budget+(config.strictTime and math.max(.15,budget*.1) or config.grace)
                    if elapsed>limit then
                        outcome={kind="error",reason=string.format("%s: response timeout %.2fs > %.2fs",MODEL_LABELS[id],elapsed,limit)};break
                    end
                    if not callOk or not result then
                        outcome={kind="error",reason=MODEL_LABELS[id]..": "..tostring(callOk and why or result)};break
                    end
                    local overrun=elapsed>budget+math.max(.15,budget*.1)
                    if overrun then LAB.score.overruns=(LAB.score.overruns or 0)+1 end
                    local legal,set=listLegal(engine)
                    local move=set[result.best]
                    if not move then outcome={kind="error",reason="illegal model result: "..tostring(result.best)};break end
                    local sanOk,san=pcall(engine.moveToSan,engine,move)
                    record.moves[#record.moves+1]={uci=result.best,san=sanOk and san or result.best,model=MODEL_LABELS[id],provider=meta.provider,
                        seconds=os.clock()-started,depth=result.depth,before=fen,legalCount=meta.legalCount,pieceCount=meta.pieceCount,overrun=overrun,budget=budget,baselineDepth=meta.baselineDepth,extensionAccepted=meta.extensionAccepted}
                    engine:play(move)
                    LAB.fen=engine:getFen()
                    task.wait(.03)
                end
                if not current() then return end
                if outcome.kind=="win" then
                    record.result=outcome.winner==1 and "1-0" or "0-1"
                    local aWon=(outcome.winner==1)==aWhite
                    if aWon then LAB.score.a=LAB.score.a+1 else LAB.score.b=LAB.score.b+1 end
                elseif outcome.kind=="draw" then record.result="1/2-1/2";LAB.score.draws=LAB.score.draws+1
                elseif outcome.kind=="error" then record.result="*";LAB.score.errors=LAB.score.errors+1
                else record.result="*";LAB.score.unfinished=LAB.score.unfinished+1 end
                record.reason=outcome.reason
                if outcome.kind=="error" then
                    LAB.lastError=string.format("Game %d, ply %d: %s\nFEN: %s",gameNumber,#record.moves+1,outcome.reason,LAB.fen or "?")
                    LAB.diagnostics[#LAB.diagnostics+1]=LAB.lastError
                end
                LAB.status="Game "..gameNumber..": "..record.result.." / "..outcome.reason
                task.wait(.1)
            end
        end)
        if LAB.token==token then
            LAB.running=false;LAB.moveStarted=nil;LAB.moveDeadline=nil;S.ply=-1
            if not ok then
                LAB.score.errors=LAB.score.errors+1
                if LAB.currentRecord then LAB.currentRecord.result="*";LAB.currentRecord.reason=tostring(err) end
                LAB.lastError="Test setup/runtime: "..tostring(err);LAB.diagnostics[#LAB.diagnostics+1]=LAB.lastError
                LAB.status=LAB.lastError
            else LAB.status=LAB.score.errors>0 and ("Ошибка теста: "..LAB.lastError.." / COPY ERROR") or "Серия завершена; COPY REPORT — подробный результат" end
        end
    end)
end
cleanup(function()labStop("script unloaded")end)

local function stopModelCheck(reason)
    CUSTOM.CHECK.token=CUSTOM.CHECK.token+1;CUSTOM.CHECK.running=false
    cancelThread(CUSTOM.CHECK.thread);CUSTOM.CHECK.thread=nil;CUSTOM.CHECK.deadline=nil
    CUSTOM.CHECK.status=reason or "Проверка остановлена";S.ply=-1
end
local function checkOneModel(id,fen,seconds)
    if CUSTOM.isLocked() then return false,"Stop the current test/check first" end
    if not MODEL_LABELS[id] then return false,"Model not found" end
    if suspendAssistant then suspendAssistant() end
    CUSTOM.CHECK.token=CUSTOM.CHECK.token+1;local token=CUSTOM.CHECK.token
    CUSTOM.CHECK.running=true;CUSTOM.CHECK.status="Проверка: "..MODEL_LABELS[id];CUSTOM.CHECK.id=id
    local budget=modelBudget(id,fen or LAB.startFen or START_FEN,seconds or LAB.seconds,LAB.sameAsGame,modelSpec(id))
    CUSTOM.CHECK.deadline=os.clock()+budget+15
    local function current()return alive() and CUSTOM.CHECK.running and CUSTOM.CHECK.token==token end
    CUSTOM.CHECK.thread=task.spawn(function()
        local started=os.clock()
        local ok,result,meta,reason=pcall(analyseModel,modelSpec(id),fen or LAB.startFen or START_FEN,budget,current,true)
        if not current() then return end
        CUSTOM.CHECK.running=false;CUSTOM.CHECK.deadline=nil;S.ply=-1
        if ok and result then
            CUSTOM.CHECK.status=string.format("OK · %s · %s · %.2f s · %d legal moves",MODEL_LABELS[id],result.best,os.clock()-started,meta.legalCount)
        else CUSTOM.CHECK.status="FAILED · "..MODEL_LABELS[id].." · "..tostring(ok and reason or result) end
    end)
    return true,"Model check started"
end
CUSTOM.setLocked(function()return LAB.running or CUSTOM.CHECK.running end)
cleanup(function()stopModelCheck("script unloaded")end)

-- Rank sampled alternatives within a short budget; primary best move remains authoritative.
local function rankContinuations(startFen, moves, skipUci, current)
    local deadline=os.clock()+0.8
	local engine = buildEngine(startFen, moves)
	if not engine then
		return nil
	end
	local scored = {}
	for _, mv in legalMoves(engine) do
        if current and not current() then return nil end
        if os.clock()>deadline then break end
        task.wait()
		local uci = engine:moveToUci(mv)
		if not skipUci or string.lower(uci) ~= string.lower(skipUci) then
			local child = buildEngine(startFen, moves)
			local ok = pcall(function()
				child:playUci(uci)
			end)
			if ok then
				local success, cres = pcall(Bot.analyse, child:getFen(), {}, CFG.CHILD_SECONDS, CFG.CHILD_DEPTH)
                if success and type(cres) == "table" and tonumber(cres.score) then
				table.insert(scored, {
					uci = uci,
					san = engine:moveToSan(mv),
					score = tonumber(cres.score),
				})
                end
			end
		end
	end
	table.sort(scored, function(a, b)
		local whiteToMove = engine:getFen():match("%s(w)%s") ~= nil
        return (whiteToMove and a.score > b.score) or (not whiteToMove and a.score < b.score)
	end)
	return scored
end

local function pickMove(session)
    if RUNTIME.SOURCE=="custom" then return S.best end
	if RUNTIME.MODE == "elo" then
        local history = table.concat(session.ucis," ")
        if not alive() or App.session ~= session or table.concat(session.ucis," ") ~= history then return nil end
		local ok, uci = pcall(Bot.think, session.startFen, table.clone(session.ucis), {
			rating = RUNTIME.ELO,
			noise = RUNTIME.NOISE,
			blunder = RUNTIME.BLUNDER,
			time = 0.6,
			depth = 14,
		})
		if ok then
            local res=normalizeResult(uci,session.engine)
            if res then return res.best end
		end
	end
	return S.best
end

-- playing ---------------------------------------------------------------------------------

local function tapInput()
	if not RUNTIME.HUMANIZE then
		return
	end
	if type(keyclick) ~= "function" then
		return
	end
	if type(isrbxactive) == "function" and not isrbxactive() then
		return
	end
	pcall(keyclick, CFG.VK_SHIFT)
end

local function isLiveView(session)
    local view=session and tonumber(session.viewPly)
    return session~=nil and (view==nil or view==#(session.ucis or {}))
end
local function playerColor(session)
    local color=session and session.myColor
    if color==1 or color=="white" or color=="w" then return 1 end
    if color==-1 or color=="black" or color=="b" then return -1 end
    return nil
end
local function myTurn(session)
    if not session or session.over then return false end
    if type(session.isMyTurn)=="function" then
        local ok,result=pcall(session.isMyTurn,session)
        if ok and type(result)=="boolean" then return result end
    end
    local color=playerColor(session)
    local engine=session.engine
    if color and engine and type(engine.getFen)=="function" then
        local ok,fen=pcall(engine.getFen,engine)
        if ok and type(fen)=="string" then
            local side=fen:match("^%S+%s+(%a)")
            return (side=="w" and color==1) or (side=="b" and color==-1)
        end
    end
    return false
end
local function playUci(uci)
    local normalized=normalizeResult(uci)
    if not normalized then return false,"bad uci" end
    uci=normalized.best
    if CUSTOM.isLocked() then return false,"Model test/check running: live move sending paused" end
    local session=App.session
    if not session or session.over then return false,"no game" end
    if not myTurn(session) then return false,"not your turn" end
    if not isLiveView(session) then return false,"return to live board" end
    -- The native game move handler validates the engine-selected UCI.
    -- SAN generation is display-only and cannot block sending or drawing it.
    if RUNTIME.MODE=="vortex3" or RUNTIME.MODE=="mindex" then
        local snap=positionSnapshot(session)
        if not snap then return false,"Vortex 3.0: live board unavailable" end
        local _,legal=listLegal(snap.engine)
        if not legal[uci] then S.ply=-1;return false,"Vortex 3.0: stale/illegal move rejected; reanalysing" end
    end
    local history=table.concat(session.ucis," ")
    local same=S.pendingSession==session and S.pendingHistory==history
    if S.playing or (same and os.clock()<(S.pendingUntil or 0)) then return false,"awaiting game response" end
    if same and (S.deliveryAttempts or 0)>=3 then return false,"AutoPlay paused: game rejected moves; toggle AUTO to retry" end
    local from,to=Engine.algToIndex64(uci:sub(1,2)),Engine.algToIndex64(uci:sub(3,4))
    if not from or not to then return false,"bad uci" end
    S.playing=true
    local before=#session.ucis
    local requestJob=S.pickJob
    local attempts=same and (S.deliveryAttempts or 0) or 0
    local useFallback=attempts>0
    tapInput()
    local callOk,result=pcall(function()
        if useFallback then return App.playMove(uci) end
        return App.onBoardMove(from,to,CFG.PROMO[uci:sub(5,5)] or 0,nil)
    end)
    if not useFallback and #session.ucis==before and (not callOk or result==false) then
        callOk,result=pcall(App.playMove,uci)
    end
    S.playing=false
    if not alive() or App.session~=session or (requestJob and S.pickJob~=requestJob) then return false,"game changed" end
    local progressed=#session.ucis>before
    S.pendingSession,S.pendingHistory=session,history
    S.deliveryAttempts=attempts+1
    S.pendingUntil=os.clock()+(progressed and .12 or math.clamp((clockSeconds(session) or 60)*.05,.35,3))
    S.thinkAt=S.pendingUntil
    if progressed then if RUNTIME.MODE=="vortex3" then S.v3Step="4 / ход отправлен и применён" end;return true,uci end
    if callOk and result~=false then return true,"awaiting game response" end
    return false,"game rejected move / "..tostring(result)
end

local function humanDelay()
	local lo = math.min(RUNTIME.DELAY_MIN, RUNTIME.DELAY_MAX)
	local hi = math.max(RUNTIME.DELAY_MIN, RUNTIME.DELAY_MAX)
	if hi - lo < 0.2 then
		return lo
	end
	return lo + math.random() * (hi - lo)
end

local function moveDelay(session)
    local delay=humanDelay()
    local remaining=clockSeconds(session)
    if remaining and remaining<15 then return math.min(delay,.05) end
    if remaining and remaining<60 then return math.min(delay,.35) end
    if RUNTIME.MODE=="mindex" then return math.min(delay,.15) end
    if RUNTIME.MODE=="vortex3" then return math.min(delay,1) end
    return delay
end

local function positionKey(session)
    if not session then return "none" end
    local liveFen=""
    if isLiveView(session) and session.engine then liveFen=engineFen(session.engine) or "" end
    return table.concat({tostring(session.startFen),table.concat(session.ucis or {}," "),liveFen,RUNTIME.SOURCE,RUNTIME.MODE,
        tostring(RUNTIME.V2_SECONDS),tostring(RUNTIME.MINDEX_SECONDS),tostring(RUNTIME.V3_SECONDS),tostring(RUNTIME.CUSTOM_MODEL),tostring(RUNTIME.STOCKFISH_DEPTH),tostring(RUNTIME.ELO)},"|")
end
local function requestMove(session,automatic)
    if CUSTOM.isLocked() or S.picking or S.playing or S.busy or not S.best then return end
    if cancelAuxiliary then cancelAuxiliary() end
    local job={session=session,key=positionKey(session)}
    S.pickJob=job;S.picking=true
    local function current()
        return alive() and S.pickJob==job and not job.cancelled and App.session==session and positionKey(session)==job.key
    end
    job.thread=task.spawn(function()
        local ok,err=pcall(function()
            local uci=pickMove(session)
            if not current() or automatic and not RUNTIME.AUTO then return end
            local played,message=playUci(uci)
            if S.pickJob==job then S.status=played and ("played "..message) or message end
        end)
        if S.pickJob==job then
            S.pickJob=nil;S.picking=false
            if not ok then S.status="Move failed: "..tostring(err);S.thinkAt=os.clock()+1 end
        end
    end)
end
local function autopilotTick(session,ply)
    if CUSTOM.isLocked() or not RUNTIME.AUTO or S.picking or S.playing then return end
    if S.busy or not S.best then S.autoStatus=S.busy and "Waiting for engine" or "No engine move";return end
    if session.over or not myTurn(session) then S.autoStatus=session.over and "Game over" or "Opponent turn";return end
    if not isLiveView(session) then S.autoStatus="Return to live board";return end
    if os.clock()<S.thinkAt then S.autoStatus=string.format("Autoplay in %.1fs",S.thinkAt-os.clock());return end
    if S.key~=positionKey(session) then return end
    S.autoStatus="Sending move"
    requestMove(session,true)
end

-- Embedded chess sprites, rendered from the DejaVu Sans chess symbol outlines.
-- No external image service is required. Native shape icons remain available
-- if this executor cannot register a local PNG as a Roblox image asset.
local PIECE_ART={asset=nil,version=0,status="native shapes"}
do
    local PIECE_PNG_BASE64=[[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]]
    local columns={k=0,q=1,r=2,b=3,n=4,p=5}
    local function decode64(input)
        local alphabet="ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
        local map={};for i=1,#alphabet do map[alphabet:sub(i,i)]=i-1 end
        local output,buffer,bits={},0,0
        for i=1,#input do
            local value=map[input:sub(i,i)]
            if value then
                buffer=buffer*64+value;bits=bits+6
                if bits>=8 then
                    bits=bits-8
                    local divisor=2^bits
                    output[#output+1]=string.char(math.floor(buffer/divisor)%256)
                    buffer=buffer%divisor
                end
            end
        end
        return table.concat(output)
    end
    PIECE_ART.decode64=decode64
    function PIECE_ART.prepare()
        local register=getcustomasset or getsynasset or (type(syn)=="table" and syn.getcustomasset)
        if type(writefile)~="function" or type(register)~="function" then return end
        local ok,asset=pcall(function()
            local path="rocheese_piece_sheet_v1.png"
            writefile(path,decode64(PIECE_PNG_BASE64))
            return register(path)
        end)
        if alive() and ok and type(asset)=="string" and asset~="" then
            PIECE_ART.asset=asset;PIECE_ART.version=PIECE_ART.version+1;PIECE_ART.status="embedded PNG sprites"
        end
    end
    function PIECE_ART.create(parent)
        local fallback=Instance.new("Frame");fallback.BackgroundTransparency=1;fallback.Size=UDim2.fromScale(1,1);fallback.Parent=parent
        local image=Instance.new("ImageLabel");image.BackgroundTransparency=1;image.Position=UDim2.fromOffset(2,1)
        image.Size=UDim2.new(1,-4,1,-4);image.ScaleType=Enum.ScaleType.Fit;image.ImageColor3=Color3.new(1,1,1)
        image.Visible=false;image.ZIndex=2;image.Parent=parent
        return {image=image,fallback=fallback,key=""}
    end
    local function shape(parent,x,y,w,h,color,round,rotation)
        local frame=Instance.new("Frame");frame.BorderSizePixel=0
        frame.Position=UDim2.fromScale(x/48,y/48);frame.Size=UDim2.fromScale(w/48,h/48)
        frame.BackgroundColor3=color;frame.Rotation=rotation or 0;frame.Parent=parent
        if round then local corner=Instance.new("UICorner");corner.CornerRadius=UDim.new(1,0);corner.Parent=frame end
        return frame
    end
    local function drawFallback(parent,piece)
        for _,child in ipairs(parent:GetChildren()) do child:Destroy() end
        local fill=piece.color==1 and Color3.fromRGB(250,249,243) or Color3.fromRGB(24,31,43)
        local edge=piece.color==1 and Color3.fromRGB(22,30,42) or Color3.fromRGB(224,232,243)
        local function part(x,y,w,h,round,rotation)return shape(parent,x,y,w,h,fill,round,rotation)end
        local base=part(9,37,30,6,true)
        local stroke=Instance.new("UIStroke");stroke.Color=edge;stroke.Thickness=.8;stroke.Parent=base
        part(13,32,22,7,true);part(17,22,14,13,true)
        local kind=piece.kind
        if kind=="p" then part(17,8,14,14,true);part(15,23,18,4,true)
        elseif kind=="k" then part(14,16,20,11,true);part(22,4,4,15,false);part(17,8,14,4,false);part(14,27,20,4,true)
        elseif kind=="q" then
            part(12,16,24,11,true);for _,x in ipairs({10,19,28}) do part(x,8,10,9,true);part(x+3,14,4,10,false) end
            part(15,27,18,4,true)
        elseif kind=="r" then
            part(12,13,24,14,false);for _,x in ipairs({12,21,30}) do part(x,7,6,10,false) end
            part(15,27,18,4,true)
        elseif kind=="b" then
            part(17,9,14,17,true,-18);part(22,4,5,7,true);shape(parent,22,10,2,10,edge,false,25);part(14,27,20,4,true)
        elseif kind=="n" then
            part(15,11,18,16,true,-16);part(10,11,22,9,true,-18);part(15,4,6,13,false,-15)
            part(17,23,13,10,true);shape(parent,15,13,3,3,edge,true,0)
        end
    end
    function PIECE_ART.refresh(view)
        if view.key=="" then view.image.Visible=false;view.fallback.Visible=false;return end
        view.fallback.Visible=not (PIECE_ART.asset and view.image.IsLoaded)
    end
    function PIECE_ART.paint(view,piece)
        if not piece then view.key="";PIECE_ART.refresh(view);return end
        local key=piece.kind..piece.color
        if key~=view.key then drawFallback(view.fallback,piece);view.key=key end
        if PIECE_ART.asset then
            view.image.Image=PIECE_ART.asset
            view.image.ImageRectSize=Vector2.new(96,96)
            view.image.ImageRectOffset=Vector2.new((columns[piece.kind] or 0)*96,piece.color==1 and 0 or 96)
            view.image.Visible=true
        else view.image.Visible=false end
        PIECE_ART.refresh(view)
    end
end

-- gui -------------------------------------------------------------------------------------

-- UI: ordinary Frames keep text colors independent of background styling.
local gui, panel, dot, lastEval
local rows, btns = {}, {}
local dragged = false
local toggleAuto, playNow, copyPosition, refreshButtons, syncDelay, applyPreset, startBotGame
local setVisible, applyTheme, layoutPanel
local C_BG = Color3.fromRGB(17, 21, 32)
local C_IDLE = Color3.fromRGB(35, 42, 60)
local C_ON = Color3.fromRGB(39, 107, 101)
local C_RAJ = Color3.fromRGB(113, 66, 99)
local C_TXT = Color3.fromRGB(240, 244, 255)
local C_DIM = Color3.fromRGB(161, 173, 195)
local C_ACC = Color3.fromRGB(126, 177, 255)
local themes = {
    Midnight = {bg=Color3.fromRGB(17,21,32), card=Color3.fromRGB(25,31,46), button=Color3.fromRGB(35,43,62), text=Color3.fromRGB(240,244,255), dim=Color3.fromRGB(163,177,200)},
    Graphite = {bg=Color3.fromRGB(24,24,27), card=Color3.fromRGB(33,33,38), button=Color3.fromRGB(47,47,54), text=Color3.fromRGB(248,248,250), dim=Color3.fromRGB(184,184,196)},
    Pearl = {bg=Color3.fromRGB(236,240,247), card=Color3.fromRGB(250,252,255), button=Color3.fromRGB(222,229,240), text=Color3.fromRGB(26,36,53), dim=Color3.fromRGB(77,92,116)},
}
local accents = {Blue=Color3.fromRGB(105,166,255), Mint=Color3.fromRGB(65,194,159), Violet=Color3.fromRGB(164,127,245), Rose=Color3.fromRGB(235,117,157)}
local themed, activeTweens = {}, setmetatable({}, {__mode="k"})
local pages, nav = {}, {}
local currentPage = "Overview"
local opening = false
local function animate(object, props, seconds)
    local changed = false
    for key,value in pairs(props) do if object[key] ~= value then changed = true; break end end
    if not changed then return end
    if activeTweens[object] then activeTweens[object]:Cancel() end
    if not RUNTIME.ANIMATIONS then
        for key,value in pairs(props) do object[key] = value end
        activeTweens[object] = nil
        return
    end
    local tween = TweenService:Create(object, TweenInfo.new(seconds or 0.24, Enum.EasingStyle.Quint, Enum.EasingDirection.Out), props)
    activeTweens[object] = tween
    tween.Completed:Once(function()
        if activeTweens[object] == tween then activeTweens[object] = nil end
    end)
    tween:Play()
    return tween
end
cleanup(function()
    for _,tween in pairs(activeTweens) do tween:Cancel() end
end)
local function role(object, property, name)
    table.insert(themed, {object, property, name})
    return object
end
local function corner(object, radius)
    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0, radius or 10)
    c.Parent = object
end
local function frame(parent, x, y, w, h, colorRole)
    local f = Instance.new("Frame")
    f.BorderSizePixel = 0
    f.Position, f.Size = UDim2.fromOffset(x,y), UDim2.fromOffset(w,h)
    f.Parent = parent
    if colorRole then role(f,"BackgroundColor3",colorRole) else f.BackgroundTransparency = 1 end
    corner(f,12)
    if colorRole == "card" then
        local edge=Instance.new("UIStroke")
        edge.Thickness=1; edge.Transparency=0.91; edge.Parent=f
        role(edge,"Color","accent")
    end
    return f
end
local function label(parent,x,y,w,h,text,size,colorRole)
    local t = Instance.new("TextLabel")
    t.BackgroundTransparency = 1
    t.Position, t.Size = UDim2.fromOffset(x,y), UDim2.fromOffset(w,h)
    t.Text, t.TextSize = text, size or 13
    t.Font = Enum.Font.GothamMedium
    t.TextXAlignment = Enum.TextXAlignment.Left
    t.TextTruncate = Enum.TextTruncate.AtEnd
    t.Parent = parent
    role(t,"TextColor3",colorRole or "text")
    return t
end
local function button(parent,x,y,w,text,callback)
    local b = Instance.new("TextButton")
    b.Position, b.Size = UDim2.fromOffset(x,y), UDim2.fromOffset(w,34)
    b.Text, b.TextSize, b.Font = text, 12, Enum.Font.GothamBold
    b.TextTruncate = Enum.TextTruncate.AtEnd
    b.AutoButtonColor, b.BorderSizePixel = false, 0
    b.Parent = parent
    corner(b,9)
    role(b,"BackgroundColor3","button")
    role(b,"TextColor3","text")
    local stroke = Instance.new("UIStroke")
    stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    stroke.Transparency = 1
    stroke.Parent = b
    role(stroke,"Color","accent")
    b.MouseEnter:Connect(function() animate(stroke,{Transparency=0.2},0.14) end)
    b.MouseLeave:Connect(function() animate(stroke,{Transparency=1},0.22) end)
    local pressScale=Instance.new("UIScale"); pressScale.Parent=b
    b.MouseButton1Down:Connect(function() animate(pressScale,{Scale=0.96},0.10) end)
    b.MouseButton1Up:Connect(function() animate(pressScale,{Scale=1},0.20) end)
    b.MouseLeave:Connect(function() animate(pressScale,{Scale=1},0.20) end)
    b.Activated:Connect(callback)
    return b
end
local function setBtn(b,on,text,onColor)
    if not b then return end
    b.Text = text
    local theme = themes[RUNTIME.THEME] or themes.Midnight
    local target = on and (onColor or C_ON) or theme.button
    if b:GetAttribute("TargetColor") ~= target then
        b:SetAttribute("TargetColor",target)
        animate(b,{BackgroundColor3=target})
    end
    b.TextColor3 = on and Color3.fromRGB(255,255,255) or theme.text
end
local function page(name, subtitle)
    local p = Instance.new("ScrollingFrame")
    p.Position,p.Size=UDim2.fromOffset(160,76),UDim2.fromOffset(456,350)
    p.BackgroundTransparency,p.BorderSizePixel=1,0
    p.CanvasSize=UDim2.new(); p.AutomaticCanvasSize=Enum.AutomaticSize.Y
    p.ScrollBarThickness=3; p.Parent=panel
    p.Visible = false
    label(p,0,0,456,28,name,22)
    label(p,0,31,456,22,subtitle,11,"dim")
    pages[name] = p
    return p
end
local function showPage(name)
    if currentPage == name and pages[name] and pages[name].Visible then return end
    currentPage = name
    for key,p in pairs(pages) do
        if activeTweens[p] then activeTweens[p]:Cancel() end
        p.Visible = key == name
        if p.Visible then
            p.Position = UDim2.fromOffset(170,76)
            animate(p,{Position=UDim2.fromOffset(160,76)},0.24)
        end
    end
    for key,b in pairs(nav) do setBtn(b,key==name,b.Text) end
end
local function buildGui()
    local playerGui = LocalPlayer:WaitForChild("PlayerGui")
    local old = playerGui:FindFirstChild("RoCheesePanel")
    if old then old:Destroy() end
    gui = Instance.new("ScreenGui")
    gui.Name, gui.ResetOnSpawn, gui.IgnoreGuiInset = "RoCheesePanel", false, true
    gui.DisplayOrder = 999
    gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    gui.Parent = playerGui
    panel = Instance.new("CanvasGroup")
    panel.Position, panel.Size = UDim2.fromOffset(30,80), UDim2.fromOffset(640,480)
    panel.BorderSizePixel = 0
    panel.Parent = gui
    role(panel,"BackgroundColor3","bg")
    panel.Name, panel.Active = "Panel", true
    panel.ClipsDescendants = true
    local shadow = Instance.new("UIStroke")
    shadow.Color = Color3.fromRGB(3,7,15); shadow.Thickness = 4; shadow.Transparency = 0.76; shadow.Parent = panel
    local backdrop=frame(panel,140,0,500,68,"card")
    local gradient=Instance.new("UIGradient"); gradient.Rotation=20
    gradient.Transparency=NumberSequence.new({NumberSequenceKeypoint.new(0,0.15),NumberSequenceKeypoint.new(1,1)})
    gradient.Parent=backdrop
    arrowLayer=Instance.new("Frame"); arrowLayer.Size=UDim2.fromScale(1,1)
    arrowLayer.BackgroundTransparency=1; arrowLayer.Active=false; arrowLayer.Name="ArrowOverlay"; arrowLayer.ZIndex=0; arrowLayer.Parent=gui
    connect(UserInputService.InputBegan,function(input,processed)
        if not calibration or (input.UserInputType~=Enum.UserInputType.MouseButton1 and input.UserInputType~=Enum.UserInputType.Touch) then return end
        local point=Vector2.new(input.Position.X,input.Position.Y)
        if calibration==1 then firstCorner=point; calibration=2; S.status="Click bottom-right outer corner of board"
        else
            local size=point-firstCorner
            if size.X<80 or size.Y<80 or math.abs(size.X-size.Y)>math.max(size.X,size.Y)*0.08 then
                calibration=1; S.status="Board must be square; select top-left again"; return
            end
            clearArrows(); boardRect={origin=firstCorner,size=math.min(size.X,size.Y),viewport=workspace.CurrentCamera and workspace.CurrentCamera.ViewportSize}
            calibration=nil; S.status="Original calibrated arrows ready"
            setVisible(true)
        end
    end)
    corner(panel,18)
    local scale = Instance.new("UIScale")
    scale.Name, scale.Parent = "ResponsiveScale", panel
    local outline = Instance.new("UIStroke")
    outline.Transparency, outline.Parent = 0.55, panel
    role(outline,"Color","accent")
    -- No UIGradient on a CanvasGroup: it tinted every descendant in the old UI.
    local rail = frame(panel,0,0,140,480,"card")
    label(rail,18,18,108,28,"RO / CHESS",15,"accent")
    label(rail,18,44,108,18,"ENGINE STUDIO",9,"dim")
    for i,name in ipairs({"Overview","Engine","Automation","Appearance","Visuals","Test model","My models","Configs","Credits"}) do
        nav[name] = button(rail,10,78+(i-1)*35,120,name,function() showPage(name) end)
    end
    label(rail,18,408,110,18,"AURORA / 13",10,"accent")
    label(rail,18,430,110,30,"by @Lesxov",11,"dim")
    rows.title = label(panel,160,20,370,26,"ROCHESS ENGINE",15)
    rows.title.Active = true
    dot = frame(panel,548,28,8,8,"accent")
    corner(dot,8)
    button(panel,574,16,42,"×",function() setVisible(false) end)
    rows.status = label(panel,160,442,455,20,"Ready",11,"dim")
    local overview = page("Overview","Position, evaluation and quick actions")
    local evaluation = frame(overview,0,68,456,132,"card")
    label(evaluation,16,10,420,16,"LIVE POSITION",10,"accent")
    rows.eval = label(evaluation,16,34,420,30,"EVAL —",20)
    rows.best = label(evaluation,16,72,420,24,"1 —",15)
    rows.rest = label(evaluation,16,100,420,22,"",12,"dim")
    rows.game = label(overview,0,210,456,24,"No game",11,"dim")
    btns.play = button(overview,0,246,222,"PLAY NOW",function() playNow() end)
    btns.copy = button(overview,234,246,222,"COPY FEN",function() copyPosition() end)
    btns.botgame = button(overview,0,290,222,"BOT GAME / PRACTICE",function() startBotGame() end)
    btns.arrows = button(overview,234,290,222,"ARROWS",function()
        RUNTIME.SHOW_ARROW = not RUNTIME.SHOW_ARROW
        clearArrows(); refreshButtons()
    end)
    btns.OPPONENT_MOVE=button(overview,234,334,222,"The opponent's move",function()
        RUNTIME.OPPONENT_MOVE=not RUNTIME.OPPONENT_MOVE; refreshButtons()
    end)
    btns.OPPONENT_MOVE:SetAttribute("Title","The opponent's move")
    label(overview,0,378,456,52,"Выбор соперника — только если игра передаёт его клиенту.\nИначе отмечается последний подтверждённый ход.",11,"dim").TextWrapped=true
    local strategyCard=frame(overview,0,444,456,112,"card")
    label(strategyCard,14,8,428,20,"ЛИНИЯ ВЫБРАННОГО ДВИЖКА",10,"accent")
    rows.plan=label(strategyCard,14,34,428,68,"Включи Vortex 2.0 в Engine",12)
    rows.plan.TextWrapped=true; rows.plan.TextYAlignment=Enum.TextYAlignment.Top
    local engine = page("Engine","Presets, search source and move selection")
    label(engine,0,66,456,20,"PLAY STYLE",10,"accent")
    btns.legit = button(engine,0,92,222,"LEGIT",function() applyPreset("legit") end)
    btns.raj = button(engine,234,92,222,"RAJ",function() applyPreset("raj") end)
    label(engine,0,138,456,20,"ENGINE SOURCE",10,"accent")
    btns.engineLocal = button(engine,0,164,222,"LOCAL",function() RUNTIME.SOURCE="local"; RUNTIME.STYLE="custom"; S.ply=-1; refreshButtons() end)
    btns.engineVortex = button(engine,234,164,222,"VORTEX",function() RUNTIME.SOURCE="vortex"; RUNTIME.STYLE="custom"; S.ply=-1; refreshButtons() end)
    label(engine,0,210,456,20,"MOVE SELECTION",10,"accent")
    btns.best = button(engine,0,236,222,"BEST MOVE",function() RUNTIME.MODE="best"; RUNTIME.STYLE="custom"; S.ply=-1; refreshButtons() end)
    btns.elo = button(engine,234,236,222,"ELO",function() RUNTIME.MODE="elo"; RUNTIME.STYLE="custom"; S.ply=-1; refreshButtons() end)
    btns.mindex=button(engine,0,280,456,"mindex / BEST MOVE+",function()
        RUNTIME.MODE="mindex";RUNTIME.STYLE="custom";S.ply=-1;refreshButtons()
    end)
    btns.ultra = button(engine,0,324,456,"ULTRA BRAIN",function()
        RUNTIME.MODE="ultra"; RUNTIME.STYLE="custom"; S.ply=-1; refreshButtons()
    end)
    btns.vortex=button(engine,0,368,456,"VORTEX / DEEP MACHINE",function()
        RUNTIME.MODE="vortex"; RUNTIME.STYLE="custom"; S.ply=-1; refreshButtons()
    end)
    btns.vortex2=button(engine,0,412,456,"VORTEX 2.0 / DIRECT ENGINE",function()
        RUNTIME.MODE="vortex2"; RUNTIME.STYLE="custom"; S.ply=-1; refreshButtons()
    end)
    btns.vortex3=button(engine,0,456,456,"VORTEX 3.0 / VALIDATED PLAY",function()
        RUNTIME.MODE="vortex3";RUNTIME.STYLE="custom";S.ply=-1;refreshButtons()
    end)
    label(engine,0,502,456,36,"Прямой поиск движка, без переоценки ходов скриптом",11,"dim").TextWrapped=true
    local rating = label(engine,54,552,160,34,"",13)
    button(engine,0,552,42,"−",function() RUNTIME.ELO=math.max(100,RUNTIME.ELO-100); RUNTIME.STYLE="custom"; refreshButtons() end)
    button(engine,222,552,42,"+",function() RUNTIME.ELO=math.min(3200,RUNTIME.ELO+100); RUNTIME.STYLE="custom"; refreshButtons() end)
    rows.rating = rating
    rows.recognition=label(engine,0,600,456,168,"Распознавание фигур: ожидание доски",12,"dim")
    rows.recognition.TextWrapped=true
    label(engine,0,780,456,40,"Кр — король, Ф — ферзь, Л — ладья, С — слон, К — конь, п — пешка.\nПоказаны клетки фигур текущей позиции, а не их изображения.",11,"dim").TextWrapped=true
    local test=page("Test model","Private board • models play each other • live assistant pauses")
    btns.labA=button(test,0,66,222,"",function()
        if CUSTOM.isLocked() then return end
        local index=table.find(MODEL_IDS,LAB.modelA) or 1;LAB.modelA=MODEL_IDS[index%#MODEL_IDS+1]
    end)
    btns.labB=button(test,234,66,222,"",function()
        if CUSTOM.isLocked() then return end
        local index=table.find(MODEL_IDS,LAB.modelB) or 1;LAB.modelB=MODEL_IDS[index%#MODEL_IDS+1]
    end)
    label(test,0,108,456,42,"A / B: нажимай для выбора. Цвета меняются между партиями.\nБюджет расчёта + запас на ответ сервера. STRICT — жёсткий лимит.",11,"dim").TextWrapped=true
    btns.labSeconds=button(test,0,160,144,"",function()
        if CUSTOM.isLocked() then return end
        local choices={.25,.5,1,2,4,5,8,12,20,30,40,50};local nextValue=choices[1];for _,value in ipairs(choices) do if value>LAB.seconds then nextValue=value;break end end;LAB.seconds=nextValue
    end)
    btns.labGames=button(test,156,160,144,"",function()
        if CUSTOM.isLocked() then return end
        LAB.games=LAB.games==1 and 2 or (LAB.games==2 and 4 or 1)
    end)
    btns.labPlies=button(test,312,160,144,"",function()
        if CUSTOM.isLocked() then return end
        LAB.maxPlies=LAB.maxPlies==80 and 160 or (LAB.maxPlies==160 and 300 or 80)
    end)
    button(test,0,204,222,"START TEST",labStart)
    button(test,234,204,222,"STOP",function()labStop("Остановлено пользователем");stopModelCheck("Проверка остановлена")end)
    button(test,0,248,42,"−",function()if not CUSTOM.isLocked() then LAB.seconds=math.max(.25,LAB.seconds-1) end end)
    local timeInput=Instance.new("TextBox");timeInput.Position=UDim2.fromOffset(54,248);timeInput.Size=UDim2.fromOffset(142,34)
    timeInput.Text=tostring(LAB.seconds);timeInput.ClearTextOnFocus=false;timeInput.Font=Enum.Font.GothamBold;timeInput.TextSize=14
    timeInput.BorderSizePixel=0;timeInput.Parent=test;corner(timeInput,8);role(timeInput,"BackgroundColor3","button");role(timeInput,"TextColor3","text")
    rows.labTimeInput=timeInput
    timeInput.FocusLost:Connect(function()
        if not CUSTOM.isLocked() then LAB.seconds=CUSTOM.number(timeInput.Text,.25,50,LAB.seconds) end
        timeInput.Text=tostring(LAB.seconds)
    end)
    button(test,208,248,42,"+",function()if not CUSTOM.isLocked() then LAB.seconds=math.min(50,LAB.seconds+1) end end)
    label(test,264,248,192,34,"seconds · max / equal",12,"dim")
    btns.labStrict=button(test,0,292,222,"",function()if not CUSTOM.isLocked() then LAB.strictTime=not LAB.strictTime end end)
    button(test,234,292,222,"CHECK MODEL A",function()
        local ok,reason=checkOneModel(LAB.modelA,LAB.startFen or START_FEN,LAB.seconds)
        LAB.status=reason
    end)
    btns.labSpeed=button(test,0,336,456,"",function()
        if not CUSTOM.isLocked() then LAB.sameAsGame=not LAB.sameAsGame end
    end)
    rows.labStatus=label(test,0,384,456,38,"",11,"dim");rows.labStatus.TextWrapped=true
    local board=frame(test,0,430,312,312)
    rows.labSquares={}
    for row=0,7 do for file=0,7 do
        local index=(7-row)*8+file+1
        local tile=Instance.new("Frame");tile.BorderSizePixel=0
        tile.Position=UDim2.fromOffset(file*39,row*39);tile.Size=UDim2.fromOffset(39,39)
        tile.BackgroundColor3=(row+file)%2==0 and Color3.fromRGB(173,188,207) or Color3.fromRGB(60,79,105)
        tile.Parent=board
        rows.labSquares[index]=PIECE_ART.create(tile)
        local coordinate=Instance.new("TextLabel");coordinate.BackgroundTransparency=1
        coordinate.Position=UDim2.fromOffset(1,27);coordinate.Size=UDim2.fromOffset(37,11)
        coordinate.Text=string.char(97+file)..(8-row);coordinate.TextSize=8;coordinate.TextColor3=Color3.fromRGB(208,219,234)
        coordinate.ZIndex=3;coordinate.Font=Enum.Font.GothamMedium;coordinate.TextXAlignment=Enum.TextXAlignment.Left;coordinate.Parent=tile
    end end
    local score=frame(test,324,430,132,136,"card")
    label(score,10,8,112,20,"SERIES SCORE",9,"accent")
    rows.labScore=label(score,10,34,112,94,"",12);rows.labScore.TextWrapped=true;rows.labScore.TextYAlignment=Enum.TextYAlignment.Top
    rows.labClock=label(test,324,578,132,44,"",11,"accent");rows.labClock.TextWrapped=true
    rows.labMoves=label(test,324,634,132,108,"",10,"dim");rows.labMoves.TextWrapped=true;rows.labMoves.TextYAlignment=Enum.TextYAlignment.Top
    button(test,0,760,222,"COPY REPORT",function()
        local ok=type(setclipboard)=="function" and pcall(setclipboard,labReport())
        LAB.status=ok and "Отчёт скопирован: SAN, FEN, движок, время каждого хода" or "Clipboard unavailable"
    end)
    button(test,234,760,222,"COPY TEST FEN",function()
        local ok=type(setclipboard)=="function" and pcall(setclipboard,LAB.fen or LAB.startFen or START_FEN)
        LAB.status=ok and "Тестовый FEN скопирован" or "Clipboard unavailable"
    end)
    label(test,0,806,456,22,"START POSITION / FEN",10,"accent")
    local fenInput=Instance.new("TextBox");fenInput.Position=UDim2.fromOffset(0,836);fenInput.Size=UDim2.fromOffset(456,72)
    fenInput.Text=LAB.startFen or START_FEN;fenInput.ClearTextOnFocus=false;fenInput.MultiLine=true;fenInput.TextWrapped=true
    fenInput.Font=Enum.Font.GothamMedium;fenInput.TextSize=12;fenInput.BorderSizePixel=0;fenInput.Parent=test;corner(fenInput,8)
    role(fenInput,"BackgroundColor3","button");role(fenInput,"TextColor3","text")
    button(test,0,920,222,"APPLY START FEN",function()
        if LAB.running then return end
        local fen=fenInput.Text:match("^%s*(.-)%s*$")
        local ok,engine=pcall(Engine.new,fen)
        local info=ok and recognizeEngine(engine)
        if not info or info.counts[1].k~=1 or info.counts[-1].k~=1 then LAB.status="Invalid FEN: нужна корректная позиция с двумя королями";return end
        local legalOk=pcall(listLegal,engine)
        if not legalOk then LAB.status="Game engine rejected FEN";return end
        LAB.startFen=engine:getFen();LAB.fen=LAB.startFen;LAB.status="Начальная позиция применена; START начнёт новую серию"
    end)
    button(test,234,920,222,"RESET START POSITION",function()
        if LAB.running then return end
        LAB.startFen=START_FEN;LAB.fen=START_FEN;fenInput.Text=START_FEN;LAB.status="Стандартная начальная позиция"
    end)
    label(test,0,972,456,112,"Фигуры показаны картинками. GAME SPEED — скорость как в игре; EQUAL BUDGET — общий лимит на модель.\nЭто отдельный Engine, без ходов в текущей партии. Лимит ходов и сбои источника не считаются победой или ничьей. Сравнивай серию с обоими цветами. Это не измерение Elo. Часы текущей партии продолжают идти, пока помощник приостановлен.",11,"dim").TextWrapped=true
    button(test,0,1104,222,"COPY ERROR",function()
        local value=LAB.lastError~="" and LAB.lastError or CUSTOM.CHECK.status
        local ok=type(setclipboard)=="function" and pcall(setclipboard,value)
        LAB.status=ok and "Diagnostic copied" or "Clipboard unavailable"
    end)
    button(test,234,1104,222,"MY MODELS",function()showPage("My models")end)
    rows.labError=label(test,0,1152,456,130,"",11,"dim");rows.labError.TextWrapped=true;rows.labError.TextYAlignment=Enum.TextYAlignment.Top
    rows.labCheck=label(test,0,1292,456,80,"",11,"accent");rows.labCheck.TextWrapped=true
    local myModels=page("My models","Custom presets, HTTP engines and registered Luau callbacks")
    label(myModels,0,66,456,48,"Добавь JSON-конфигурацию своего движка. Модель появится в A/B теста и может использоваться в основной игре. Формат ответа — ход UCI, например e2e4 или a7a8q.",11,"dim").TextWrapped=true
    local selectedCustom
    local selector
    local editor=Instance.new("TextBox")
    editor.Position=UDim2.fromOffset(0,166);editor.Size=UDim2.fromOffset(456,260)
    editor.MultiLine=true;editor.TextWrapped=true;editor.ClearTextOnFocus=false
    editor.TextXAlignment=Enum.TextXAlignment.Left;editor.TextYAlignment=Enum.TextYAlignment.Top
    editor.Font=Enum.Font.Code;editor.TextSize=12;editor.BorderSizePixel=0;editor.Parent=myModels;corner(editor,8)
    role(editor,"BackgroundColor3","button");role(editor,"TextColor3","text")
    rows.customNotice=label(myModels,0,620,456,72,"Конфигурации сохраняются автоматически",11,"dim");rows.customNotice.TextWrapped=true
    local function setExample(kind)
        if CUSTOM.isLocked() then rows.customNotice.Text="Stop current test/check first";return end
        selectedCustom=nil
        local example
        if kind=="http" then
            example={name="My HTTP engine",provider="http",url="https://your-server.example/analyse",method="POST",depth=20,
                request={fen="{fen}",seconds="{seconds}",depth="{depth}"},movePath="bestmove",scorePath="score",pvPath="pv",scoreUnit="cp",headers={}}
        elseif kind=="callback" then example={name="My Luau engine",provider="callback",callback="my_engine",depth=20}
        else example={name="My Local engine",provider="local",depth=20,maxSeconds=50} end
        editor.Text=HttpService:JSONEncode(example)
        selector.Text="NEW MODEL / "..kind
    end
    selector=button(myModels,0,120,456,"NEW MODEL",function()
        if CUSTOM.isLocked() then return end
        local customIds={}
        for _,id in ipairs(MODEL_IDS) do if CUSTOM.MODELS[id] then customIds[#customIds+1]=id end end
        if #customIds==0 then rows.customNotice.Text="No saved models; edit the example and press SAVE MODEL";return end
        local index=table.find(customIds,selectedCustom) or 0
        selectedCustom=customIds[index%#customIds+1]
        editor.Text=HttpService:JSONEncode(CUSTOM.MODELS[selectedCustom]);selector.Text=MODEL_LABELS[selectedCustom]
    end)
    button(myModels,0,440,222,"SAVE MODEL",function()
        if #editor.Text>65536 then rows.customNotice.Text="Configuration is too large (max 64 KiB)";return end
        local ok,data=pcall(HttpService.JSONDecode,HttpService,editor.Text)
        if not ok then rows.customNotice.Text="Invalid JSON: check quotes, commas and braces";return end
        local id,reason=CUSTOM.save(data,selectedCustom)
        rows.customNotice.Text=tostring(reason)
        if id then selectedCustom=id;selector.Text=MODEL_LABELS[id];editor.Text=HttpService:JSONEncode(CUSTOM.MODELS[id]) end
    end)
    button(myModels,234,440,222,"CHECK MODEL",function()
        if not selectedCustom then rows.customNotice.Text="Save this model first";return end
        local ok,reason=checkOneModel(selectedCustom,LAB.startFen or START_FEN,LAB.seconds)
        rows.customNotice.Text=reason
    end)
    button(myModels,0,484,144,"LOCAL EXAMPLE",function()setExample("local")end)
    button(myModels,156,484,144,"HTTP EXAMPLE",function()setExample("http")end)
    button(myModels,312,484,144,"LUA EXAMPLE",function()setExample("callback")end)
    button(myModels,0,528,222,"USE IN LIVE GAME",function()
        if CUSTOM.isLocked() then rows.customNotice.Text="Stop current test/check first";return end
        if not selectedCustom or not CUSTOM.MODELS[selectedCustom] then rows.customNotice.Text="Save/select a model first";return end
        RUNTIME.SOURCE="custom";RUNTIME.CUSTOM_MODEL=selectedCustom;RUNTIME.MODE="vortex3";RUNTIME.STYLE="custom";S.ply=-1
        refreshButtons();rows.customNotice.Text="Live source: "..MODEL_LABELS[selectedCustom]
    end)
    button(myModels,234,528,222,"COPY MODEL JSON",function()
        local ok=type(setclipboard)=="function" and pcall(setclipboard,editor.Text)
        rows.customNotice.Text=ok and "JSON copied" or "Clipboard unavailable"
    end)
    button(myModels,0,572,222,"DELETE SELECTED",function()
        local id=selectedCustom;local ok,reason=CUSTOM.delete(id)
        rows.customNotice.Text=tostring(reason)
        if ok then
            if LAB.modelA==id then LAB.modelA="local" end
            if LAB.modelB==id then LAB.modelB="vortex" end
            selectedCustom=nil;selector.Text="NEW MODEL"
        end
    end)
    button(myModels,234,572,222,"STOP CHECK",function()stopModelCheck()end)
    rows.modelCheck=label(myModels,0,704,456,64,CUSTOM.CHECK.status,11,"accent");rows.modelCheck.TextWrapped=true
    label(myModels,0,782,456,236,"Providers: local / vortex — параметры встроенного движка; stockfish — Stockfish Online; http — свой сервер; callback — своя Luau-функция.\n\nHTTP POST отправляет JSON request. GET: URL должен содержать {fen}; он будет URL-encoded. movePath / scorePath / pvPath поддерживают вложенные поля: data.bestmove, result.pv. При выборе HTTP-модели FEN отправляется на указанный адрес.\n\nCallback ищется в getgenv().__RoCheeseCustomCallbacks.my_engine и получает таблицу {fen, seconds, depth}. Верни строку UCI или {best=..., score=..., pv={...}}. Скрипт не загружает и не исполняет код из JSON. Файлы нейросетей требуют своего совместимого движка/сервера.",11,"dim").TextWrapped=true
    setExample("local")
    local automation = page("Automation","Autoplay timing and input options")
    btns.auto = button(automation,0,70,456,"AUTOPLAY",function() toggleAuto() end)
    local delayValues = {}
    for i,key in ipairs({"DELAY_MIN","DELAY_MAX"}) do
        local y=128+(i-1)*56
        label(automation,0,y,170,34,i==1 and "Minimum delay" or "Maximum delay",13)
        button(automation,196,y,42,"−",function()
            if key=="DELAY_MIN" then RUNTIME[key]=math.max(0,RUNTIME[key]-0.5)
            else RUNTIME[key]=math.max(RUNTIME.DELAY_MIN+0.5,RUNTIME[key]-0.5) end
            RUNTIME.STYLE="custom"; syncDelay(); refreshButtons()
        end)
        delayValues[key]=label(automation,254,y,118,34,"",14)
        button(automation,414,y,42,"+",function()
            if key=="DELAY_MIN" then RUNTIME[key]=math.min(RUNTIME.DELAY_MAX-0.5,RUNTIME[key]+0.5)
            else RUNTIME[key]=math.min(20,RUNTIME[key]+0.5) end
            RUNTIME.STYLE="custom"; syncDelay(); refreshButtons()
        end)
    end
    syncDelay = function()
        for key,l in pairs(delayValues) do l.Text=string.format("%.1f sec",RUNTIME[key]) end
    end
    btns.humanize=button(automation,0,250,456,"HUMANIZE",function() RUNTIME.HUMANIZE=not RUNTIME.HUMANIZE; refreshButtons() end)
    label(automation,0,302,456,26,"End: autoplay  /  Home: arrows  /  +: panel",11,"dim")
    local appearance=page("Appearance","Personalize the panel; save your look in a config")
    label(appearance,0,66,456,20,"COLOR THEME",10,"accent")
    for i,name in ipairs({"Midnight","Graphite","Pearl"}) do
        button(appearance,(i-1)*156,94,144,name,function() RUNTIME.THEME=name; applyTheme() end)
    end
    label(appearance,0,142,456,20,"ACCENT COLOR",10,"accent")
    for i,name in ipairs({"Blue","Mint","Violet","Rose"}) do
        button(appearance,(i-1)*117,170,105,name,function() RUNTIME.ACCENT=name; applyTheme() end)
    end
    btns.animations=button(appearance,0,226,456,"ANIMATIONS",function() RUNTIME.ANIMATIONS=not RUNTIME.ANIMATIONS; refreshButtons() end)
    label(appearance,0,282,456,48,"640 × 480 • scales to your screen\nDrag the header to move the panel",12,"dim").TextWrapped=true
    local visuals=page("Visuals","Arrow studio • изменения видны сразу")
    local function option(y,key,title)
        local b
        b=button(visuals,0,y,456,"",function()
            RUNTIME[key]=not RUNTIME[key]; if key=="BOARD_FLIPPED" then RUNTIME.AUTO_ORIENTATION=false end; clearArrows(); refreshButtons()
        end)
        btns[key]=b; b:SetAttribute("Title",title)
    end
    button(visuals,0,66,280,"CALIBRATE / ORIGINAL ARROWS",function()
        clearArrows(); RUNTIME.AUTO_BOARD=false; boardRect=nil; calibration=1; setVisible(false)
        S.status="Click top-left outer corner, then bottom-right"
    end)
    button(visuals,292,66,164,"NATIVE ARROWS",function() clearArrows(); boardRect=nil; calibration=nil end)
    label(visuals,0,108,456,44,"Стрелки перенесены из рабочего файла.\nNATIVE — стрелки игры; CALIBRATE — исходный запасной способ.",11,"dim").TextWrapped=true
    for i,name in ipairs({"Classic","Candy","Ice"}) do
        button(visuals,(i-1)*156,160,144,name,function() RUNTIME.ARROW_PALETTE=name; clearArrows() end)
    end
    label(visuals,0,206,456,64,"ORIGINAL GAME ARROWS\nСтандартная форма без наложений и новой графики.",12,"dim").TextWrapped=true
    option(290,"ARROW_TINT","DESTINATION HIGHLIGHT")
    option(332,"BOARD_FLIPPED","BLACK AT BOTTOM / MANUAL")
    option(846,"AUTO_ORIENTATION","AUTO BOARD ORIENTATION")
    rows.arrowInfo=label(visuals,0,894,456,56,"Board detection: waiting",11,"dim"); rows.arrowInfo.TextWrapped=true
    local function stepper(parent,y,key,title,lo,hi,step)
        local value=label(parent,54,y,344,34,"",12)
        local function update(delta)
            RUNTIME[key]=math.clamp(math.floor((RUNTIME[key]+delta)*100+0.5)/100,lo,hi)
            clearArrows(); refreshButtons(); layoutPanel()
        end
        button(parent,0,y,42,"−",function() update(-step) end)
        button(parent,414,y,42,"+",function() update(step) end)
        rows[key]=value; value:SetAttribute("Title",title)
    end
    label(visuals,0,380,456,34,"Форму и толщину стрелок задаёт сама игра.",12,"dim")
    stepper(visuals,424,"ARROW_ALPHA","Transparency",0,0.8,0.1)
    label(visuals,0,474,456,22,"ARROW COLORS / HEX RGB",11,"accent")
    for i=1,3 do
        local rank=i
        local y=506+(i-1)*48
        label(visuals,0,y,90,34,"Arrow "..i,12)
        local input=Instance.new("TextBox")
        input.Position,input.Size=UDim2.fromOffset(96,y),UDim2.fromOffset(230,34)
        input.Text=RUNTIME["ARROW_COLOR_"..i]; input.PlaceholderText="RRGGBB (e.g. FF4060)"
        input.ClearTextOnFocus=false; input.Font=Enum.Font.GothamMedium; input.TextSize=12
        input.BorderSizePixel=0; input.Parent=visuals; corner(input,8)
        role(input,"BackgroundColor3","button"); role(input,"TextColor3","text"); role(input,"PlaceholderColor3","dim")
        rows["ARROW_COLOR_"..i]=input
        local function applyColor()
            local hex=input.Text:gsub("#",""):gsub("%s",""):upper()
            if hex~="" and not hex:match("^%x%x%x%x%x%x$") then S.status="Color: enter six HEX digits, e.g. FF4060"; return end
            RUNTIME["ARROW_COLOR_"..rank]=hex; input.Text=hex; clearArrows()
        end
        input.FocusLost:Connect(applyColor)
        button(visuals,338,y,118,"APPLY",applyColor)
    end
    button(visuals,0,654,456,"RESET CUSTOM COLORS",function()
        for i=1,3 do RUNTIME["ARROW_COLOR_"..i]=""; rows["ARROW_COLOR_"..i].Text="" end
        clearArrows()
    end)
    option(702,"BAR","BAR / PERFORMANCE HUD")
    stepper(visuals,750,"BAR_SCALE","Bar size",0.55,1.8,0.05)
    button(visuals,0,798,456,"RESET BAR POSITION",function() RUNTIME.BAR_X=-1; RUNTIME.BAR_Y=-1 end)
    stepper(engine,1104,"MINDEX_SECONDS","mindex max seconds",.8,10,.2)
    stepper(engine,1056,"V3_SECONDS","Vortex 3.0 max seconds",1,45,1)
    stepper(engine,840,"V2_SECONDS","Vortex 2.0 time / sec",5,45,1)
    btns.engineStockfish=button(engine,0,888,456,"STOCKFISH ONLINE / SELECT SOURCE",function()
        RUNTIME.SOURCE="stockfish"; RUNTIME.STYLE="custom"; S.ply=-1; refreshButtons()
    end)
    label(engine,0,932,456,64,"Внешний источник: при выборе FEN позиции отправляется на stockfish.online. Нужен доступ HTTP. При сбое используется движок RoChess; фактический источник виден в bar.",11,"dim").TextWrapped=true
    stepper(engine,1008,"STOCKFISH_DEPTH","Stockfish Online depth",8,15,1)
    stepper(appearance,338,"UI_SCALE","Interface scale",0.65,1.25,0.05)
    local credits=page("Credits","Автор • справка • сохранение")
    local author=frame(credits,0,66,456,72,"card")
    label(author,16,8,424,20,"ROCHESS / AURORA",11,"accent")
    label(author,16,32,424,28,"Автор — @Lesxov",22)
    local help={
        {"MINDEX","Основа — BEST MOVE с тем же источником. В спокойных позициях используется быстрый поиск. В сложных позициях добавляется поиск того же корня; результат меняется только при подтверждённо большей глубине того же движка. Цель 98–99% требует проверки в реальных партиях, скрипт не рисует фиктивную accuracy."},
        {"TEST MODEL SPEED","GAME SPEED по умолчанию использует те же бюджеты, что и игра: BEST MOVE около 0.8 с, mindex адаптивно до выбранного максимума. Поле времени задаёт потолок. EQUAL BUDGET даёт обеим моделям указанный лимит для сравнения при равном времени."},
        {"MY MODELS / TEST TIME","My models: добавь JSON-настройки локального, Vortex, Stockfish, HTTP-движка или зарегистрированного Luau callback. SAVE MODEL добавляет модель в A/B. CHECK MODEL проверяет один ход, USE IN LIVE GAME выбирает её для игры. Test model: бюджет 0.25–50 с, сетевой запас по умолчанию 15 с, ошибки доступны через COPY ERROR."},
        {"VORTEX 3.0","Читает фигуры и точную позицию, получает легальные ходы из Engine, запрашивает движок и проверяет выбранный ход повторно перед отправкой. Время распределяется на остаток партии с резервом. Это режим управления поиском, не новая обученная нейросеть."},
        {"TEST MODEL","Две выбранные модели играют на отдельной доске. Одинаковый бюджет на ход, смена цветов, FEN начальной позиции и COPY REPORT. Текущая игра не получает тестовых ходов; живой помощник на время теста приостанавливается. Сбои источника — ошибки теста, не победы."},
        {"VORTEX 2.0 / DIRECT ENGINE","Больше времени штатному движку RoChess; без переоценки лучшего хода скриптом. Обученная нейросеть не заявляется. Устаревшие расчёты отменяются. Stockfish Online выбирается отдельно в Engine."},
        {"VORTEX MODE","Запрос к игровому модулю Vortex, при недоступности — локальный поиск с несколькими проходами. Бюджет до 4 с с учётом часов. Реальную силу задаёт доступный движок."},
        {"BAR / COLORS / TOUCH","Visuals: BAR включает FPS, ping, плейс и время поиска справа. Перетащи заголовок bar; тяни ↘ для масштаба или используй Bar size. Цвета стрелок — HEX RRGGBB. Калибровка запасной отрисовки поддерживает касания. По умолчанию используются стрелки игры."},
        {"THE OPPONENT'S MOVE","Показывает выбранную соперником клетку только при наличии переданных игрой данных. Иначе LAST MOVE отмечает последний сделанный ход. Приватные клики другого игрока клиенту обычно недоступны."},
        {"ULTRA BRAIN","Углублённый поиск без намеренных ошибок. На ход выделяется до 3 секунд с учётом оставшегося времени. Число ошибок за партию не гарантируется."},
        {"BEST MOVE","Выбирает лучший ход по текущему поиску. PLAY NOW делает ход один раз; AUTOPLAY ходит автоматически. Оценка не гарантирует идеальную игру."},
        {"ELO / LEGIT / RAJ","ELO учитывает рейтинг, шум и ошибки. LEGIT: локальный движок, 1500, задержка 5–10 с. RAJ: Vortex, BEST, 0.5–1.5 с."},
        {"LOCAL / VORTEX","LOCAL считает на клиенте. VORTEX вызывает игровой модуль; при сбое анализ переключается на LOCAL. Вторая и третья стрелки считаются локально."},
        {"СТРЕЛКИ И ОЦЕНКА","1 — лучший ход; 2 и 3 — альтернативы из ограниченного поиска. Цвета меняются в Visuals. Плюс оценки — преимущество белых, минус — чёрных; M означает мат."},
        {"CONFIGS / ГДЕ СОХРАНЯЕТСЯ","Введи имя → CREATE. SAVE обновляет выбранный профиль; LOAD применяет его. Файл: rocheese_profiles_v2.json в workspace исполнителя. Без writefile — только память текущей сессии."},
        {"QUEUE ON TELEPORT","Сохрани этот полный скрипт как rochess3.txt в workspace исполнителя. Нужны queue_on_teleport, readfile и loadstring. Для другого пути: getgenv().__RoCheeseScriptPath. После сохранения нажми RETRY ниже."},
        {"ВИЗУАЛЬНЫЕ НАСТРОЙКИ","Appearance: тема, акцент, анимации, масштаб. Visuals: палитра, цвет и прозрачность стандартных стрелок; ориентация метки соперника. Калибровка действует только в текущей сессии."},
        {"КЛАВИШИ / БУФЕР","+ или PageUp — панель; End — авто; Home — стрелки; PageDown — движок; Insert — COPY FEN. COPY FEN копирует FEN и ходы SAN в буфер, а не в файл. HUMANIZE посылает Shift, если доступен keyclick."},
    }
    for i,item in ipairs(help) do
        local y=152+(i-1)*122
        label(credits,0,y,456,22,item[1],12,"accent")
        local body=label(credits,0,y+26,446,88,item[2],13,"dim")
        body.TextWrapped=true; body.TextTruncate=Enum.TextTruncate.None; body.TextYAlignment=Enum.TextYAlignment.Top
    end
    rows.teleport=label(credits,0,152+#help*122,456,44,teleportNotice,11,"dim")
    rows.teleport.TextWrapped=true
    button(credits,0,202+#help*122,456,"RETRY TELEPORT QUEUE",function() queueTeleport(); rows.teleport.Text=teleportNotice end)
    local configs=page("Configs","Select, create, rename and save named configurations")
    local list = Instance.new("ScrollingFrame")
    list.Position, list.Size = UDim2.fromOffset(0,68), UDim2.fromOffset(210,236)
    list.BackgroundTransparency, list.BorderSizePixel = 1, 0
    list.ScrollBarThickness = 3
    list.CanvasSize = UDim2.fromOffset(0,0)
    list.Parent = configs
    local input = Instance.new("TextBox")
    input.Position,input.Size=UDim2.fromOffset(226,68),UDim2.fromOffset(230,34)
    input.PlaceholderText="Config name…"
    input.Text=""
    input.ClearTextOnFocus=false
    input.TextSize,input.Font=12,Enum.Font.GothamMedium
    input.BorderSizePixel=0
    input.Parent=configs
    corner(input,8)
    role(input,"BackgroundColor3","button"); role(input,"TextColor3","text"); role(input,"PlaceholderColor3","dim")
    local notice=label(configs,0,312,456,30,configNotice,11,"dim")
    local selected=tostring(store.active)
    local profileButtons={}
    local rebuild
    local function cleanName()
        local name=input.Text:match("^%s*(.-)%s*$")
        if #name==0 or #name>48 or name:find("[%c]") then notice.Text="Use a name of 1–48 bytes"; return nil end
        for id,n in pairs(store.names) do
            if id~=selected and n==name then notice.Text="That name already exists"; return nil end
        end
        return name
    end
    rebuild=function()
        for _,b in ipairs(profileButtons) do b:Destroy() end
        profileButtons={}
        local ids={}
        for id in pairs(store.profiles) do table.insert(ids,id) end
        table.sort(ids,function(a,b) return (store.names[a] or a)<(store.names[b] or b) end)
        for i,id in ipairs(ids) do
            local b=button(list,0,(i-1)*42,200,store.names[id] or ("Profile "..id),function()
                selected=id; input.Text=store.names[id] or ("Profile "..id)
                notice.Text="Selected: "..input.Text.." — press LOAD"
                rebuild()
            end)
            -- The dynamic list is recolored directly; discard obsolete theme records.
            for _=1,3 do table.remove(themed) end
            setBtn(b,id==selected,b.Text)
            table.insert(profileButtons,b)
        end
        list.CanvasSize=UDim2.fromOffset(0,#ids*42)
    end
    button(configs,226,114,110,"CREATE",function()
        local name=cleanName(); if not name then return end
        for _,n in pairs(store.names) do if n==name then notice.Text="That name already exists"; return end end
        local id=HttpService:GenerateGUID(false)
        store.profiles[id],store.names[id]=validated(RUNTIME),name
        selected,store.active=id,id
        notice.Text=persistProfiles(); rebuild()
    end)
    button(configs,346,114,110,"RENAME",function()
        if not store.profiles[selected] then notice.Text="Select a config first"; return end
        local name=cleanName(); if not name then return end
        store.names[selected]=name; notice.Text=persistProfiles(); rebuild()
    end)
    button(configs,226,158,110,"SAVE",function()
        if not store.profiles[selected] then notice.Text="Create or select a config first"; return end
        store.profiles[selected],store.active=validated(RUNTIME),selected
        notice.Text=persistProfiles(); rebuild()
    end)
    button(configs,346,158,110,"LOAD",function()
        if not store.profiles[selected] then notice.Text="Select a config first"; return end
        applyConfig(store.profiles[selected]); store.active=selected
        for i=1,3 do rows["ARROW_COLOR_"..i].Text=RUNTIME["ARROW_COLOR_"..i] end
        S.ply=-1; clearArrows(); syncDelay(); applyTheme()
        notice.Text="Loaded: "..(store.names[selected] or selected).." / "..persistProfiles()
        rebuild()
    end)
    local pendingDelete
    button(configs,226,202,230,"DELETE SELECTED",function()
        if not store.profiles[selected] then notice.Text="Select a config first"; return end
        if pendingDelete~=selected then
            pendingDelete=selected; notice.Text="Press DELETE again to confirm"
            task.delay(3,function() pendingDelete=nil end); return
        end
        store.profiles[selected],store.names[selected]=nil,nil
        pendingDelete=nil
        if tostring(store.active)==selected then store.active="" end
        selected=""; input.Text=""; notice.Text=persistProfiles(); rebuild()
    end)
    button(configs,226,246,230,"RESET CURRENT SETTINGS",function()
        applyConfig(DEFAULTS); for i=1,3 do rows["ARROW_COLOR_"..i].Text="" end; S.ply=-1; clearArrows(); syncDelay(); applyTheme()
        notice.Text="Defaults applied; saved configs unchanged"; rebuild()
    end)
    applyTheme=function()
        local t=themes[RUNTIME.THEME] or themes.Midnight
        t.accent=accents[RUNTIME.ACCENT] or accents.Blue
        C_ACC,C_TXT,C_DIM,C_BG,C_IDLE=t.accent,t.text,t.dim,t.bg,t.button
        for _,item in ipairs(themed) do
            if item[1].Parent then
                item[1][item[2]]=t[item[3]]
                if item[2]=="BackgroundColor3" then item[1]:SetAttribute("TargetColor",nil) end
            end
        end
        refreshButtons(); showPage(currentPage); rebuild()
    end
    local reopen=button(gui,16,80,44,"+",function() setVisible(true) end)
    reopen.BackgroundColor3=C_BG; reopen.TextColor3=C_ACC
    local visibilityToken=0
    local restingPosition
    setVisible=function(value)
        visibilityToken+=1
        local token=visibilityToken
        RUNTIME.UI_VISIBLE=value
        if activeTweens[panel] then activeTweens[panel]:Cancel() end
        if opening and restingPosition then panel.Position=restingPosition end
        opening=false
        layoutPanel()
        local target=panel.Position
        restingPosition=target
        reopen.Visible=not value
        if value then
            panel.Visible=true
            panel.Position=target+UDim2.fromOffset(0,12)
            opening=true
            panel.GroupTransparency=1
            animate(panel,{Position=target,GroupTransparency=0},0.28)
        else
            opening=true
            animate(panel,{Position=target+UDim2.fromOffset(0,12),GroupTransparency=1},0.20)
        end
        task.delay(RUNTIME.ANIMATIONS and 0.29 or 0,function()
            if alive() and token==visibilityToken then panel.Visible=value; panel.Position=target; opening=false end
        end)
    end
    local dragging,dragInput,dragStart,origin=false,nil,nil,nil
    local dragTarget
    connect(game:GetService("RunService").RenderStepped,function(dt)
        if not panel or not panel.Parent or not dragTarget or opening then return end
        local a = RUNTIME.ANIMATIONS and (1-math.exp(-24*dt)) or 1
        panel.Position=panel.Position:Lerp(dragTarget,a)
        layoutPanel()
        if not dragging and (Vector2.new(panel.Position.X.Offset,panel.Position.Y.Offset)-Vector2.new(dragTarget.X.Offset,dragTarget.Y.Offset)).Magnitude<0.5 then dragTarget=nil end
    end)
    rows.title.InputBegan:Connect(function(inputObject)
        if inputObject.UserInputType==Enum.UserInputType.MouseButton1 or inputObject.UserInputType==Enum.UserInputType.Touch then
            if activeTweens[panel] then activeTweens[panel]:Cancel() end
            opening=false; visibilityToken+=1; panel.GroupTransparency=0
            dragging=true; dragInput=inputObject; dragged=true
            dragTarget=nil; dragStart=inputObject.Position; origin=panel.Position
        end
    end)
    connect(UserInputService.InputChanged,function(inputObject)
        if not dragging then return end
        if inputObject.UserInputType==Enum.UserInputType.MouseMovement or inputObject==dragInput then
            local delta=inputObject.Position-dragStart
            local vp=workspace.CurrentCamera and workspace.CurrentCamera.ViewportSize or Vector2.new(1280,720)
            local scale=panel.ResponsiveScale.Scale
            dragTarget=UDim2.fromOffset(math.clamp(origin.X.Offset+delta.X,0,math.max(0,vp.X-640*scale)),math.clamp(origin.Y.Offset+delta.Y,0,math.max(0,vp.Y-480*scale)))
        end
    end)
    connect(UserInputService.InputEnded,function(inputObject)
        if inputObject==dragInput or inputObject.UserInputType==Enum.UserInputType.MouseButton1 then dragging=false end
    end)
    syncDelay(); applyTheme(); setVisible(RUNTIME.UI_VISIBLE~=false)
    cleanup(function() if gui then gui:Destroy(); gui=nil; panel=nil end end)
end
refreshButtons=function()
    if not btns.auto then return end
    setBtn(btns.auto,RUNTIME.AUTO,RUNTIME.AUTO and "AUTOPLAY / ON" or "AUTOPLAY / OFF")
    setBtn(btns.legit,RUNTIME.STYLE=="legit","LEGIT / HUMAN",C_ON)
    setBtn(btns.raj,RUNTIME.STYLE=="raj","RAJ / FAST",C_RAJ)
    setBtn(btns.best,RUNTIME.MODE=="best","BEST MOVE")
    setBtn(btns.mindex,RUNTIME.MODE=="mindex","mindex / BEST MOVE+",accents.Mint)
    setBtn(btns.ultra,RUNTIME.MODE=="ultra","ULTRA BRAIN / DEEP SEARCH",accents.Violet)
    setBtn(btns.vortex,RUNTIME.MODE=="vortex","VORTEX / DEEP MACHINE",accents.Violet)
    setBtn(btns.vortex2,RUNTIME.MODE=="vortex2","VORTEX 2.0 / DIRECT ENGINE",accents.Violet)
    setBtn(btns.vortex3,RUNTIME.MODE=="vortex3","VORTEX 3.0 / VALIDATED PLAY",accents.Violet)
    setBtn(btns.elo,RUNTIME.MODE=="elo","ELO / "..RUNTIME.ELO)
    setBtn(btns.engineLocal,RUNTIME.SOURCE=="local","LOCAL")
    setBtn(btns.engineVortex,RUNTIME.SOURCE=="vortex","VORTEX")
    setBtn(btns.engineStockfish,RUNTIME.SOURCE=="stockfish","STOCKFISH ONLINE / SELECT SOURCE")
    setBtn(btns.arrows,RUNTIME.SHOW_ARROW,RUNTIME.SHOW_ARROW and "ARROWS / ON" or "ARROWS / OFF")
    setBtn(btns.humanize,RUNTIME.HUMANIZE,RUNTIME.HUMANIZE and "HUMANIZE INPUT / ON" or "HUMANIZE INPUT / OFF")
    setBtn(btns.animations,RUNTIME.ANIMATIONS,RUNTIME.ANIMATIONS and "ANIMATIONS / ON" or "ANIMATIONS / OFF")
    for _,key in ipairs({"ARROW_GLOW","ARROW_LABELS","ARROW_TINT","BOARD_FLIPPED","OPPONENT_MOVE","BAR","AUTO_ORIENTATION"}) do
        local b=btns[key]; if b then setBtn(b,RUNTIME[key],b:GetAttribute("Title") .. (RUNTIME[key] and " / ON" or " / OFF")) end
    end
    for _,key in ipairs({"ARROW_WIDTH","ARROW_ALPHA","UI_SCALE","BAR_SCALE","V2_SECONDS","V3_SECONDS","MINDEX_SECONDS","STOCKFISH_DEPTH"}) do
        local l=rows[key]; if l then l.Text=l:GetAttribute("Title") .. string.format("  %.2f",RUNTIME[key]) end
    end
    rows.rating.Text="Rating: "..RUNTIME.ELO
end
layoutPanel=function()
    if not panel or not panel.Parent or opening then return end
    local cam=workspace.CurrentCamera
    local vp=cam and cam.ViewportSize or Vector2.new(1280,720)
    local scale=math.max(0.1,math.min(RUNTIME.UI_SCALE,(vp.X-24)/640,(vp.Y-24)/480))
    panel.ResponsiveScale.Scale=scale
    local width,height=640*scale,480*scale
    local x=dragged and panel.Position.X.Offset or vp.X-width-24
    local y=dragged and panel.Position.Y.Offset or (vp.Y-height)/2
    panel.Position=UDim2.fromOffset(math.clamp(x,0,math.max(0,vp.X-width)),math.clamp(y,0,math.max(0,vp.Y-height)))
end
local function shiftPanel(dx,dy)
    if not panel then return end
    dragged=true
    panel.Position=panel.Position+UDim2.fromOffset(dx,dy)
    layoutPanel()
end

-- Floating telemetry and publicly available opponent information.
local hud, hudScale, hudTitle, hudTime, hudPlace, hudStatus, calibrationPrompt
local hudFps,hudPing,hudDepth,hudChecks,hudPhase,hudFill,hudPlan,hudState
local HUD_W,HUD_H=460,210
local placeName=game.Name~="" and game.Name or ("Place "..tostring(game.PlaceId))
local fps,frameCount,frameElapsed=0,0,0
local pingText="—"
local pingAt=0
local opponentMarker
local function opponentInfo(session)
    if not session then return nil,"No game" end
    -- Integration point for selection data actually exposed by the game.
    -- Never infer a remote player's private mouse/touch input.
    local selected=session.opponentSelectedSquare
    if type(GENV.__RoCheeseOpponentSelection)=="function" then
        local ok,value=pcall(GENV.__RoCheeseOpponentSelection,session)
        if ok then selected=value end
    end
    local square
    if type(selected)=="string" and selected:match("^[a-h][1-8]$") then square=Engine.algToIndex64(selected)
    elseif type(selected)=="number" and selected%1==0 and selected>=1 and selected<=64 then square=selected end
    if square then return square,"OPPONENT SELECTION / received" end
    local mine=playerColor(session)
    if not mine then return nil,"Opponent selection unavailable" end
    local whiteStarts=not tostring(session.startFen):match("^%S+%s+b%s")
    for i=#(session.ucis or {}),1,-1 do
        local mover=((i%2==1)==whiteStarts) and 1 or -1
        if mover~=mine then
            local uci=session.ucis[i]
            if type(uci)=="string" then
                return Engine.algToIndex64(uci:sub(3,4)),"LAST OPPONENT MOVE / "..uci
            end
        end
    end
    return nil,"Opponent selection unavailable / no confirmed move"
end
local function updateOpponent(session)
    local rect=boardRect or detectedRect
    if not RUNTIME.OPPONENT_MOVE or not session or not isLiveView(session) or calibration or not rect then
        if opponentMarker then opponentMarker:Destroy(); opponentMarker=nil end
        return
    end
    local square,description=opponentInfo(session)
    S.opponentStatus=description
    if not square then if opponentMarker then opponentMarker:Destroy(); opponentMarker=nil end; return end
    local cell=rect.size/8
    local file,row=(square-1)%8,7-math.floor((square-1)/8)
    if boardFlipped() then file,row=7-file,7-row end
    if not opponentMarker then
        opponentMarker=Instance.new("Frame"); opponentMarker.Name="OpponentPublicMove"
        opponentMarker.BackgroundColor3=Color3.fromRGB(255,117,142); opponentMarker.BackgroundTransparency=.80
        opponentMarker.BorderSizePixel=0; opponentMarker.ZIndex=29; opponentMarker.Parent=arrowLayer; corner(opponentMarker,8)
        local stroke=Instance.new("UIStroke"); stroke.Color=Color3.fromRGB(255,117,142); stroke.Thickness=2; stroke.Parent=opponentMarker
        local caption=label(opponentMarker,2,0,200,16,"",9,"text"); caption.Name="Caption"; caption.ZIndex=30
    end
    opponentMarker.Position=UDim2.fromOffset(rect.origin.X+file*cell+2,rect.origin.Y+row*cell+2)
    opponentMarker.Size=UDim2.fromOffset(cell-4,cell-4)
    opponentMarker.Caption.Size=UDim2.new(1,0,0,16)
    opponentMarker.Caption.Text=description:find("SELECTION",1,true) and "SELECTED" or "LAST MOVE"
    opponentMarker.Caption.TextColor3=Color3.fromRGB(255,230,236)
end
local function layoutHud()
    if not hud then return end
    local vp=workspace.CurrentCamera and workspace.CurrentCamera.ViewportSize or Vector2.new(1280,720)
    local scale=math.max(.25,math.min(RUNTIME.BAR_SCALE,(vp.X-16)/HUD_W,(vp.Y-16)/HUD_H))
    hudScale.Scale=scale
    local x=RUNTIME.BAR_X>=0 and RUNTIME.BAR_X or math.max(8,vp.X-HUD_W*scale-16)
    local y=RUNTIME.BAR_Y>=0 and RUNTIME.BAR_Y or 16
    hud.Position=UDim2.fromOffset(math.clamp(x,0,math.max(0,vp.X-HUD_W*scale)),math.clamp(y,0,math.max(0,vp.Y-HUD_H*scale)))
end
local function buildHud()
    hud=frame(gui,0,16,HUD_W,HUD_H,"bg"); hud.Name="PerformanceBar"; hud.ZIndex=40; hud.Active=true
    hudScale=Instance.new("UIScale"); hudScale.Parent=hud
    local shadow=Instance.new("UIStroke"); shadow.Color=Color3.fromRGB(3,7,15);shadow.Thickness=5;shadow.Transparency=.78;shadow.Parent=hud
    local edge=Instance.new("UIStroke"); edge.Thickness=1;edge.Transparency=.55;edge.Parent=hud;role(edge,"Color","accent")
    local header=frame(hud,0,0,HUD_W,56,"card")
    local sheen=Instance.new("UIGradient");sheen.Rotation=20
    sheen.Transparency=NumberSequence.new({NumberSequenceKeypoint.new(0,.08),NumberSequenceKeypoint.new(1,.85)})
    sheen.Parent=header
    frame(hud,0,15,3,27,"accent")
    hudTitle=label(hud,16,10,284,24,"VORTEX 2.0",15,"accent");hudTitle.Font=Enum.Font.GothamBold;hudTitle.Active=true
    hudState=label(hud,16,34,284,15,"ENGINE STUDIO / READY",9,"dim")
    local timeTitle=label(hud,314,8,130,13,"SEARCH TIME",9,"dim");timeTitle.TextXAlignment=Enum.TextXAlignment.Right
    hudTime=label(hud,304,22,140,30,"0.00 s",23);hudTime.Font=Enum.Font.GothamBold;hudTime.TextXAlignment=Enum.TextXAlignment.Right
    local function metric(x,w,title)
        local card=frame(hud,x,66,w,44,"card")
        label(card,10,4,w-20,12,title,8,"dim")
        local value=label(card,10,18,w-20,22,"—",16);value.Font=Enum.Font.GothamBold
        return value
    end
    hudFps=metric(16,100,"FRAME RATE")
    hudPing=metric(124,120,"NETWORK / MS")
    hudDepth=metric(252,90,"DEPTH")
    hudChecks=metric(350,94,"BOARD DATA")
    hudPlace=label(hud,16,117,428,18,placeName,11,"dim")
    hudPhase=label(hud,16,141,428,16,"Готов к анализу",10,"accent")
    local track=frame(hud,16,163,428,3,"button")
    hudFill=frame(track,0,0,0,3,"accent")
    hudPlan=label(hud,16,174,400,15,"Расчётная линия появится после поиска",10)
    hudStatus=label(hud,16,191,400,13,"Ready",9,"dim")
    local resize=button(hud,424,180,24,"↘",function() end);resize.Size=UDim2.fromOffset(24,24)
    for _,object in ipairs(hud:GetDescendants()) do if object:IsA("GuiObject") then object.ZIndex=41 end end
    hudFill.ZIndex=42
    local held,start,p0,s0,kind
    local function begin(input,action)
        if input.UserInputType~=Enum.UserInputType.MouseButton1 and input.UserInputType~=Enum.UserInputType.Touch then return end
        held,start,p0,s0,kind=input,input.Position,hud.Position,RUNTIME.BAR_SCALE,action
    end
    hudTitle.InputBegan:Connect(function(input) begin(input,"move") end)
    resize.InputBegan:Connect(function(input) begin(input,"size") end)
    connect(UserInputService.InputChanged,function(input)
        if not held or (input~=held and input.UserInputType~=Enum.UserInputType.MouseMovement) then return end
        local delta=input.Position-start
        if kind=="move" then
            RUNTIME.BAR_X,RUNTIME.BAR_Y=math.max(0,p0.X.Offset+delta.X),math.max(0,p0.Y.Offset+delta.Y)
        else RUNTIME.BAR_SCALE=math.clamp(s0+delta.X/HUD_W,0.55,1.8) end
        layoutHud()
    end)
    connect(UserInputService.InputEnded,function(input)
        if input==held then held=nil; if refreshButtons then refreshButtons() end end
    end)
    connect(game:GetService("RunService").RenderStepped,function(dt)
        frameCount,frameElapsed=frameCount+1,frameElapsed+dt
        if frameElapsed>=.5 then fps=frameCount/frameElapsed;frameCount,frameElapsed=0,0 end
    end)
    calibrationPrompt=label(gui,0,0,600,44,"",14,"accent")
    calibrationPrompt.Name="CalibrationPrompt"; calibrationPrompt.AnchorPoint=Vector2.new(.5,0)
    calibrationPrompt.Position=UDim2.new(.5,0,0,4); calibrationPrompt.ZIndex=60
    calibrationPrompt.TextXAlignment=Enum.TextXAlignment.Center
    calibrationPrompt.TextWrapped=true
    calibrationPrompt.BackgroundColor3=Color3.fromRGB(12,18,28); calibrationPrompt.BackgroundTransparency=.12; corner(calibrationPrompt,10)
    task.spawn(function()
        local ok,info=pcall(function() return game:GetService("MarketplaceService"):GetProductInfo(game.PlaceId) end)
        if alive() and ok and type(info)=="table" and type(info.Name)=="string" then placeName=info.Name end
    end)
    cleanup(function() if opponentMarker then opponentMarker:Destroy();opponentMarker=nil end end)
end
local function updateHud(session)
    if not hud then return end
    hud.Visible=RUNTIME.BAR
    calibrationPrompt.Visible=calibration~=nil
    local vp=workspace.CurrentCamera and workspace.CurrentCamera.ViewportSize
    if vp then calibrationPrompt.Size=UDim2.fromOffset(math.min(600,vp.X-12),44) end
    calibrationPrompt.Text=calibration==1 and "Калибровка: нажми внешний верхний левый угол доски" or "Калибровка: нажми внешний нижний правый угол доски"
    if not hud.Visible then return end
    layoutHud()
    if os.clock()>=pingAt then
        pingAt=os.clock()+1
        local ok,value=pcall(function() return game:GetService("Stats").Network.ServerStatsItem["Data Ping"]:GetValue() end)
        if not ok or type(value)~="number" then
            ok,value=pcall(function() return LocalPlayer:GetNetworkPing()*1000 end)
        end
        pingText=ok and type(value)=="number" and string.format("%.0f ms",value) or "unavailable"
    end
    local elapsed=searchElapsed()
    local mode=RUNTIME.MODE=="mindex" and "mindex" or (RUNTIME.MODE=="vortex3" and "VORTEX 3.0" or (RUNTIME.MODE=="vortex2" and "VORTEX 2.0" or string.upper(RUNTIME.MODE)))
    local busy=S.searchRunning==true
    hudTitle.Text=mode
    hudState.Text=not session and "ENGINE STUDIO / IDLE" or (busy and "ENGINE STUDIO / THINKING" or (S.best and "ENGINE STUDIO / READY" or "ENGINE STUDIO / WAIT"))
    hudTime.Text=string.format("%.2f s",elapsed)
    hudFps.Text=string.format("%.0f",fps)
    hudPing.Text=pingText:gsub(" ms","")
    hudDepth.Text=tostring(S.depth or 0)
    hudChecks.Text=S.recognition and "64 / FEN" or "—"
    hudPlace.Text=placeName.." / "..tostring(S.actualEngine or "engine pending")
    hudPhase.Text=CUSTOM.isLocked() and "Test model: live assistant paused" or (S.v3Step or (busy and "Поиск лучшего хода" or (S.best and "Ход найден / таймер остановлен" or "Ожидание позиции")))
    local progress=0
    if S.best and not busy then progress=1
    elseif busy then progress=S.searchProgress or 0;progress=math.max(progress,math.min(.94,elapsed/math.max(.1,S.searchBudget or 1))) end
    local target=UDim2.new(math.clamp(progress,0,1),0,0,3)
    if hudFill:GetAttribute("Progress")~=progress then
        hudFill:SetAttribute("Progress",progress);animate(hudFill,{Size=target},.18)
    end
    hudPlan.Text=S.planMoves and ("Линия: "..table.concat(S.planMoves," → ")) or (S.planText and S.planText:match("^[^\n]+") or (S.best and ("Ход: "..S.best) or "Расчётная линия появится после поиска"))
    hudStatus.Text=RUNTIME.OPPONENT_MOVE and (S.opponentStatus or "Opponent selection unavailable") or (RUNTIME.AUTO and (S.autoStatus or S.status) or S.status)

end

local function updateLab()
    if CUSTOM.CHECK.running and CUSTOM.CHECK.deadline and os.clock()>CUSTOM.CHECK.deadline then stopModelCheck("FAILED: engine response timeout") end
    if rows.modelCheck then rows.modelCheck.Text=CUSTOM.CHECK.status end
    if LAB.running and LAB.moveDeadline and os.clock()>LAB.moveDeadline then
        LAB.score.errors=LAB.score.errors+1
        if LAB.currentRecord then LAB.currentRecord.result="*";LAB.currentRecord.reason="engine response timeout" end
        LAB.lastError=LAB.status.." / engine response timeout / FEN: "..tostring(LAB.fen)
        LAB.diagnostics[#LAB.diagnostics+1]=LAB.lastError
        labStop("Тест прерван: движок не ответил вовремя")
    end
    if not rows.labStatus then return end
    btns.labA.Text="A · "..(MODEL_LABELS[LAB.modelA] or "model removed")
    btns.labB.Text="B · "..(MODEL_LABELS[LAB.modelB] or "model removed")
    btns.labSeconds.Text=string.format("%.2f s %s",LAB.seconds,LAB.sameAsGame and "ceiling" or "/ move")
    btns.labSpeed.Text=LAB.sameAsGame and "TIME: GAME SPEED / per-model budgets" or "TIME: EQUAL BUDGET / same limit for both"
    btns.labGames.Text=tostring(LAB.games).." games"
    btns.labPlies.Text=tostring(LAB.maxPlies).." max plies"
    rows.labStatus.Text=LAB.status
    btns.labStrict.Text=LAB.strictTime and "TIME: STRICT" or ("TIME: +"..LAB.grace.."s GRACE")
    if rows.labTimeInput and not rows.labTimeInput:IsFocused() then rows.labTimeInput.Text=tostring(LAB.seconds) end
    if rows.labError then rows.labError.Text=LAB.lastError~="" and LAB.lastError or "Последняя ошибка: нет" end
    if rows.labCheck then rows.labCheck.Text=CUSTOM.CHECK.status end
    rows.labScore.Text=string.format("A wins: %d\nB wins: %d\nDraws: %d\nErrors: %d\nUnfinished: %d",LAB.score.a,LAB.score.b,LAB.score.draws,LAB.score.errors,LAB.score.unfinished)
    rows.labClock.Text=LAB.running and string.format("Game %d/%d\nBudget %.2fs\nThink %.2fs",LAB.game,LAB.games,LAB.currentBudget or 0,LAB.moveStarted and (os.clock()-LAB.moveStarted) or 0) or "Test idle"
    local recent={}
    for i=math.max(1,#LAB.moves-7),#LAB.moves do recent[#recent+1]=i.." "..LAB.moves[i].uci end
    rows.labMoves.Text=table.concat(recent,"\n")
    local fen=LAB.fen or LAB.startFen or START_FEN
    if LAB.renderedFen~=fen or LAB.renderedArt~=PIECE_ART.version then
        local info=recognizeFen(fen)
        for square,view in pairs(rows.labSquares) do PIECE_ART.paint(view,info and info.board[square]) end
        LAB.renderedFen=fen;LAB.renderedArt=PIECE_ART.version
    end
    for _,view in pairs(rows.labSquares) do PIECE_ART.refresh(view) end

end

-- actions ---------------------------------------------------------------------------------

toggleAuto = function()
	RUNTIME.AUTO = not RUNTIME.AUTO
    S.deliveryAttempts,S.pendingUntil=0,0
	S.thinkAt = os.clock() + (RUNTIME.AUTO and moveDelay(App.session) or 0)
	S.status = RUNTIME.AUTO and "autopilot on" or "autopilot off"
	refreshButtons()
end

playNow = function()
    if CUSTOM.isLocked() then S.status="Model test/check running / live moves paused";return end
    local session=App.session
    if not session or session.over then S.status="no game running";return end
    if S.busy or S.picking or not S.best or S.key~=positionKey(session) then S.status="engine still thinking";return end
    if not myTurn(session) or not isLiveView(session) then S.status="return to your live turn";return end
    requestMove(session,false)
end

copyPosition = function()
	local session = App.session
	if not session then
		S.status = "no game running"
		return
	end
	local fen = session.engine:getFen()
	local sans = table.concat(session.sans or {}, " ")
	local ok = type(setclipboard) == "function" and pcall(setclipboard, fen .. "\n" .. sans)
	S.status = ok and "fen + moves copied" or "clipboard unavailable"
end

startBotGame = function()
	local ok, err = pcall(function()
		App.startBotGame("jasper", "white")
	end)
	S.status = ok and "practice game started" or ("bot game failed: " .. tostring(err))
end

applyPreset = function(name)
	local preset = PRESETS[name]
	if not preset then
		return
	end
	RUNTIME.STYLE = name
	RUNTIME.SOURCE = preset.source
	RUNTIME.MODE = preset.mode
	RUNTIME.ELO = preset.elo
	RUNTIME.NOISE = preset.noise
	RUNTIME.BLUNDER = preset.blunder
	RUNTIME.DELAY_MIN = preset.delayMin
	RUNTIME.DELAY_MAX = preset.delayMax
	S.ply = -1
	S.status = name .. " preset"
	if syncDelay then
		syncDelay()
	end
	refreshButtons()
end

local setDelay = function(lo, hi)
	lo = math.clamp(tonumber(lo) or RUNTIME.DELAY_MIN, CFG.DELAY_MIN, CFG.DELAY_MAX)
	hi = math.clamp(tonumber(hi) or RUNTIME.DELAY_MAX, CFG.DELAY_MIN, CFG.DELAY_MAX)
	if hi < lo then
		lo, hi = hi, lo
	end
	RUNTIME.DELAY_MIN = math.min(19.5, math.floor(lo / CFG.DELAY_STEP + 0.5) * CFG.DELAY_STEP)
	RUNTIME.DELAY_MAX = math.floor(hi / CFG.DELAY_STEP + 0.5) * CFG.DELAY_STEP
	if RUNTIME.DELAY_MAX - RUNTIME.DELAY_MIN < CFG.DELAY_STEP then
		RUNTIME.DELAY_MAX = RUNTIME.DELAY_MIN + CFG.DELAY_STEP
	end
	RUNTIME.STYLE = "custom"
	if syncDelay then
		syncDelay()
	end
	refreshButtons()
end

-- hotkeys ---------------------------------------------------------------------------------

local function onKey(input, processed)
	if processed or UserInputService:GetFocusedTextBox() then
		return
	end
	local code = input.KeyCode
	if code == Enum.KeyCode.End then
		toggleAuto()
	elseif code == Enum.KeyCode.Home then
		RUNTIME.SHOW_ARROW = not RUNTIME.SHOW_ARROW
		if not RUNTIME.SHOW_ARROW then
			clearArrows()
		end
		refreshButtons()
	elseif code == Enum.KeyCode.PageUp or code == Enum.KeyCode.Equals or code == Enum.KeyCode.KeypadPlus then
        if setVisible then setVisible(not (RUNTIME.UI_VISIBLE ~= false)) end
	elseif code == Enum.KeyCode.PageDown then
		RUNTIME.SOURCE = RUNTIME.SOURCE == "local" and "vortex" or "local"
		RUNTIME.STYLE = "custom"
		S.ply = -1
		refreshButtons()
	elseif code == Enum.KeyCode.Insert then
		copyPosition()
	end
end

local prevConn = GENV.__RoCheeseConn
if prevConn then
	pcall(function()
		prevConn:Disconnect()
	end)
end
local conn = connect(UserInputService.InputBegan, onKey)
GENV.__RoCheeseConn = conn
cleanup(function()
	conn:Disconnect()
end)

-- eval bar -------------------------------------------------------------------------------
-- the game already ships a chess-style eval bar next to the board, it is just never
-- switched on during a normal game. Feeding it our score wakes it up.
local GAME_KINDS = {
	bot = true,
	online = true,
	puzzle = true,
	rush = true,
}

local function pushEval()
	local session = App.session
	local kind = session and session.kind
	if not session or not kind or not GAME_KINDS[kind] then
		lastEval = nil
		return
	end
	if S.busy or not S.best then
		return
	end
	local score = S.score
	if lastEval == score then
		return
	end
	lastEval = score
	local mate = nil
	if math.abs(score) >= 29900 then
		mate = math.max(1, math.floor((30000 - math.abs(score)) / 100 + 0.5))
	end
	pcall(function()
		App.setEval(score, mate)
	end)
end

-- panel refresh ---------------------------------------------------------------------------

local function formatClock(ms)
	if type(ms) ~= "number" then
		return "-"
	end
	ms = math.max(0, ms)
	local total = math.floor(ms / 1000)
	return string.format("%d:%02d", math.floor(total / 60), total % 60)
end

local function evalText(score)
	local abs = math.abs(score)
	if abs >= 29900 then
		local moves = math.max(1, math.floor((30000 - abs) / 100 + 0.5))
		return string.format("M%d %s", moves, score > 0 and "white" or "black")
	end
	local pawns = score / 100
	return string.format("%s%.2f", pawns >= 0 and "+" or "", pawns)
end

local function refreshPanel(session, ply)
	if not panel or not panel.Parent then
		return
	end

    if rows.plan then rows.plan.Text=(S.v3Step and (S.v3Step.."\n") or "")..(S.planText or (S.busy and (S.searchPhase or "Движок считает...") or "Выбери режим в Engine")) end
    if rows.recognition then
        rows.recognition.Text=recognitionText(S.recognition)
            ..(S.recognitionSource and ("\nИсточник: "..S.recognitionSource) or "")
            ..(S.recognitionMismatch and " / FEN и история различаются; источник указан выше" or "")
    end
    if rows.arrowInfo then
        rows.arrowInfo.Text=(boardRect and "Arrows: original calibrated renderer" or "Arrows: original native renderer").." / "..(S.best or S.status)
    end
	rows.title.Text = "ROCHESS ENGINE"
	rows.status.Text = (RUNTIME.AUTO and S.autoStatus) or S.status
	rows.eval.Text = session
		and (S.busy and "EVAL ...      DEPTH ..." or string.format("EVAL %s      DEPTH %d", evalText(S.score), S.depth))
		or "EVAL -      DEPTH 0"
	if S.busy then
		rows.best.Text = "1 thinking..."
		rows.rest.Text = ""
	else
		rows.best.Text = S.best and string.format("1  %s   (%s)", S.san or S.best, S.best) or "1  -"
		local parts = {}
		if S.second then
			table.insert(parts, string.format("2  %s", S.secondSan or S.second))
		end
		if S.third then
			table.insert(parts, string.format("3  %s", S.thirdSan or S.third))
		end
		rows.rest.Text = table.concat(parts, "     ")
	end
	local myClock, oppClock = "-", "-"
	if session and session.clocks and playerColor(session) then
        local color=playerColor(session)
		myClock = formatClock(session.clocks[color] or session.clocks[session.myColor])
		oppClock = formatClock(session.clocks[-color])
	end
	rows.game.Text = session
		and string.format("%s | ply %d | clock %s/%s",
			playerColor(session) == 1 and (session.white and session.white.display or "white")
				or (session.black and session.black.display or "black"),
			ply,
			myClock,
			oppClock)
		or "no game"

	if session and session.kind == "idle" then
		rows.status.Text = "no game running - press BOT GAME or Play in the game"
	end

	if dot then
		if S.busy then
			dot.BackgroundColor3 = C_ACC
		elseif S.best then
			dot.BackgroundColor3 = CFG.ARROW_1
		else
			dot.BackgroundColor3 = Color3.fromRGB(200, 80, 70)
		end
	end

	pushEval()
end

-- main loop -------------------------------------------------------------------------------

buildGui()
task.spawn(PIECE_ART.prepare)
buildHud()
applyTheme()
refreshButtons()
S.status = "ready / + to toggle"

GENV.__RoCheese.api = {
    addModel=CUSTOM.save,
    deleteModel=CUSTOM.delete,
    checkModel=checkOneModel,
    models=function()return HttpService:JSONDecode(HttpService:JSONEncode(CUSTOM.MODELS))end,
	auto = toggleAuto,
	play = playNow,
	copy = copyPosition,
	move = shiftPanel,
	delay = setDelay,
	preset = applyPreset,
	state = function()
		return {
			ply = S.ply,
			best = S.best,
			san = S.san,
			second = S.second,
			third = S.third,
			score = S.score,
			depth = S.depth,
			status = S.status,
			auto = RUNTIME.AUTO,
			style = RUNTIME.STYLE,
			source = RUNTIME.SOURCE,
			mode = RUNTIME.MODE,
			delayMin = RUNTIME.DELAY_MIN,
			delayMax = RUNTIME.DELAY_MAX,
		}
	end,
}

-- Each search owns a position token. Cancelling old work never unlocks or
-- overwrites a newer job, even if a provider returns after cancellation.
local activeJob
local jobSequence=0
local function jobCurrent(job)
    return alive() and activeJob==job and not job.cancelled and App.session==job.session
        and positionKey(job.session)==job.key and S.key==job.key and S.ply==job.ply
end
cancelAuxiliary=function()
    if activeJob then
        activeJob.altCancelled=true
        cancelThread(activeJob.altThread)
        activeJob.altThread=nil
    end
end
local function abandonSearch()
    if activeJob then
        activeJob.cancelled=true
        cancelThread(activeJob.thread)
        cancelThread(activeJob.altThread)
    end
    activeJob=nil
    S.busy,S.engineBusy,S.searchRunning=false,false,false
end
local function updatePosition(session,ply)
    local key=positionKey(session)
    if S.session~=session or S.key~=key or S.ply~=ply then
        abandonSearch()
        cancelMoveWork()
        S.session,S.key,S.ply=session,key,ply
        S.best,S.san,S.second,S.third,S.secondSan,S.thirdSan=nil,nil,nil,nil,nil,nil
        S.score,S.depth=0,0
        S.analysisDone,S.analysisStarted,S.analysisDuration=false,nil,0
        S.retryAt,S.attempts=0,0
        S.planText,S.planMoves,S.searchProgress,S.searchPhase=nil,nil,0,"Ожидание движка"
        S.actualEngine,S.engineNotice,S.v3Meta,S.v3Step,S.mindexMeta=nil,nil,nil,nil,nil
        S.recognition,S.recognitionSource,S.recognitionMismatch=nil,nil,false
        S.thinkAt=os.clock()+moveDelay(session)
        lastEval=nil
        clearArrows()
    end
    if activeJob and S.searchRunning and os.clock()-activeJob.started>activeJob.timeout then
        S.analysisDuration=os.clock()-activeJob.started
        if RUNTIME.SOURCE=="stockfish" then S.externalSkipUntil=os.clock()+60 end
        abandonSearch()
        S.analysisDone=false;S.retryAt=os.clock()+.2
        S.status="Engine timeout; restarting latest position"
    end
    if CUSTOM.isLocked() or not session or session.over or S.analysisDone or activeJob and S.searchRunning
        or S.picking or os.clock()<(S.retryAt or 0) then return end
    local snapshot
    local snapOk,snap=pcall(positionSnapshot,session)
    if snapOk then snapshot=snap end
    if snapshot then
        S.recognition=snapshot.info;S.recognitionSource=snapshot.source;S.recognitionMismatch=snapshot.mismatch
    end
    local seconds,depth=CFG.SECONDS,CFG.DEPTH
    if RUNTIME.MODE=="mindex" then seconds,depth=MINDEX.gameBudget(session,snapshot and snapshot.info,snapshot and snapshot.engine,RUNTIME.MINDEX_SECONDS),CFG.DEPTH
    elseif RUNTIME.MODE=="vortex3" then seconds,depth=v3Budget(session,snapshot and snapshot.info,RUNTIME.V3_SECONDS),32
    elseif RUNTIME.MODE=="ultra" or RUNTIME.MODE=="vortex" or RUNTIME.MODE=="vortex2" then
        local cap=RUNTIME.MODE=="vortex2" and RUNTIME.V2_SECONDS or (RUNTIME.MODE=="vortex" and 4 or 3)
        seconds,depth=cap,RUNTIME.MODE=="ultra" and 24 or (RUNTIME.MODE=="vortex2" and 30 or 28)
        local color=playerColor(session)
        local remaining=session.clocks and tonumber(session.clocks[session.myColor] or session.clocks[color])
        if remaining then seconds=math.clamp(remaining/1000*(RUNTIME.MODE=="vortex2" and .08 or .035),.12,cap) end
    end
    jobSequence=jobSequence+1
    local timeout=seconds+10
    if RUNTIME.MODE=="vortex3" then timeout=math.max(.2,math.min(seconds+3,(clockSeconds(session) or 100)*.7)) end
    local job={id=jobSequence,session=session,key=key,ply=ply,started=os.clock(),timeout=timeout}
    activeJob=job
    S.busy,S.engineBusy,S.searchRunning=true,true,true
    S.analysisStarted,S.analysisDuration,S.searchBudget=job.started,0,seconds
    S.attempts=S.attempts+1
    local startFen=snapshot and snapshot.fen or session.startFen
    local moves=snapshot and {} or table.clone(session.ucis or {})
    local function current()return jobCurrent(job)end
    job.thread=task.spawn(function()
        local ok,err=pcall(function()
            local res=analyseBest(startFen,moves,seconds,depth,current,snapshot)
            if not current() then return end
            -- Freeze when the primary engine returns. SAN, alternatives,
            -- AutoPlay delay and move delivery cannot restart this timer.
            S.analysisDuration=os.clock()-job.started
            S.searchRunning,S.busy,S.engineBusy=false,false,false
            if type(res)~="table" or type(res.best)~="string" then
                local terminalOk,terminal=pcall(function()
                    if not snapshot or not snapshot.engine then return false end
                    for _ in legalMoves(snapshot.engine) do return false end
                    return true
                end)
                S.analysisDone=terminalOk and terminal or false
                S.retryAt=os.clock()+math.min(10,1+S.attempts*2)
                S.status=S.analysisDone and "Position has no legal moves" or "No engine result; retrying current position"
                return
            end
            S.best=res.best;S.score=tonumber(res.score) or 0;S.depth=tonumber(res.depth) or 0
            S.analysisDone=true;S.searchProgress=1
            S.status="Ready / "..tostring(S.actualEngine or "RoChess engine")
            local sanOk,san=pcall(function()return sanOf(snapshot and snapshot.engine or session.engine,res.best)end)
            if current() and sanOk then S.san=san end
            if res.pv and #res.pv>1 then
                S.planMoves=res.pv;S.planText=table.concat(res.pv," → ")
            else S.planMoves={res.best};S.planText="Ход: "..res.best..". Движок не вернул продолжение." end
            if not current() then return end
            -- Alternatives are optional and never lock the main search or
            -- change its selected move. Any new position cancels this task.
            if RUNTIME.MODE~="vortex2" and RUNTIME.MODE~="vortex3" and RUNTIME.MODE~="mindex" then
                job.altThread=task.spawn(function()
                    local function altCurrent()return current() and not job.altCancelled end
                    local rankedOk,ranked=pcall(rankContinuations,startFen,moves,res.best,altCurrent)
                    if rankedOk and ranked and altCurrent() then
                        S.second,S.secondSan=ranked[1] and ranked[1].uci,ranked[1] and ranked[1].san
                        S.third,S.thirdSan=ranked[2] and ranked[2].uci,ranked[2] and ranked[2].san
                    end
                end)
            end
        end)
        if activeJob==job then
            if S.searchRunning then S.analysisDuration=os.clock()-job.started end
            S.searchRunning,S.busy,S.engineBusy=false,false,false
            job.complete=true
            if not ok and current() then
                S.analysisDone=false;S.retryAt=os.clock()+math.min(10,1+S.attempts*2)
                S.status="Engine error: "..tostring(err)
            end
        end
    end)
end
suspendAssistant=function()
    abandonSearch();cancelMoveWork();S.ply=-1;clearArrows()
    S.status="Test model running / live assistant paused"
end
cleanup(function()abandonSearch();cancelMoveWork()end)

local function tick()
    updateLab()
    if gui and gui.Parent ~= LocalPlayer:FindFirstChild("PlayerGui") then
        local playerGui = LocalPlayer:FindFirstChild("PlayerGui")
        if playerGui then gui.Parent = playerGui end
    end
    if boardRect and workspace.CurrentCamera and boardRect.viewport ~= workspace.CurrentCamera.ViewportSize then
        clearArrows(); boardRect=nil; S.status="Viewport changed: recalibrate the optional board overlay"
    end
    if detectedRect and detectedRect.source then
        local object=detectedRect.source
        if object.Parent and effectivelyVisible(object) then
            detectedRect.origin,detectedRect.size=object.AbsolutePosition,object.AbsoluteSize.X
        else detectedRect=nil; boardProbeAt=0 end
    end
    local session = App.session
    local ply = session and #(session.ucis or {}) or -1
    updatePosition(session,ply)
    local live = session and RUNTIME.SHOW_ARROW and isLiveView(session) and not session.over
    if live then
        if S.best then drawArrow(ARROW_KEYS[1],S.best,CFG.ARROW_1,true) end
        if S.second then drawArrow(ARROW_KEYS[2],S.second,CFG.ARROW_2,false) end
        if S.third then drawArrow(ARROW_KEYS[3],S.third,CFG.ARROW_3,false) end
        autopilotTick(session,ply)
    else
        clearArrows()
        if session then autopilotTick(session,ply) end
    end
    if RUNTIME.OPPONENT_MOVE and not detectedRect and not boardRect then pcall(probeBoard) end
    pcall(updateOpponent,session)
    pcall(updateHud,session)
    refreshPanel(session,ply)
    layoutPanel()
end

task.spawn(function()
    while alive() do
        local ok,err = pcall(tick)
        if not ok then S.status="Update failed: "..tostring(err) end
        task.wait(0.10)
    end
end)

cleanup(clearArrows)

print("[rocheese] ready - STYLE LEGIT/RAJ, ULTRA BRAIN, DELAY min/max, AUTOPLAY; arrows: green best / blue second / yellow third")