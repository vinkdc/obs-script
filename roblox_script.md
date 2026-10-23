--// =========================================================
--// NAMETAGS + BOMB TRACKER + TEAM ROUND DETECTOR (no HTTP)
--// =========================================================

local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local TeamsService = game:GetService("Teams")
local LocalPlayer = Players.LocalPlayer

local AnnounceEvent = game:GetService("ReplicatedStorage"):WaitForChild("Events"):WaitForChild("RemoteEvent")

local SCRIPT_VERSION = "1.7.1" -- bump on every edit

--// SETTINGS
local PLAYING_TEAM_NAME = "Playing"
local TEAM_A_NAME = "Black"
local TEAM_B_NAME = "White"
-- first to 7 (up to 3v3) or 11 (4v4 and 5v5), win by 2; the match end is decided by Rules.isMatchOver

--// =========================================================
--// SCRIM SETTINGS (fill these in)
--// =========================================================
local API_URL = "https://clip-hosting-o361.onrender.com" -- your clip-hosting site (it relays to the bot through Mongo), no trailing slash
local API_SECRET = "71fefba29e4dadfc22e9af5d690ae7f82e0687eada0d53702519c6cb653efaf4" -- the ROBLOX_WEBHOOK_SECRET stored in Mongo (config > api_keys) or in clip-hosting's env
local DRY_RUN = false                           -- true = print every remote instead of firing it (safe test)
local LOBBY_CAP = 10                            -- timer stops resetting once this many verified players are in
local PRACTICE_MIN_PLAYERS = 2                  -- while the lobby waits for the real match, this many verified players get practice rounds
local PRACTICE_GAP = 8                          -- seconds between practice rounds
local PRACTICE_MAX_ROUND = 180                  -- a practice round that runs longer than this is stopped
local ANONYMOUS_ON_ARGS = { "Anonymous" }        -- remote fired when the captain rounds start
local ANONYMOUS_OFF_ARGS = { "Anonymous" }       -- fired when they end (same call: it toggles anonymous mode; change it if the game uses a different one)
local ANONYMOUS_SETTLE = 4                          -- seconds to let anonymous mode take effect before the first captain round starts
local KICK_DURATION = 5                         -- 3rd arg of KickPlayer = lockout minutes
local VOTEKICK_DURATION = 10                    -- lockout minutes for a player voted out of a match
local FORFEIT_SECONDS = 180                    -- a team with nobody in the server this long forfeits the live match
local ACTION_GAP = 1.5                          -- seconds between ANY two game remotes (kick, timeout, announce, start/stop)
local ACTION_JITTER = 0.5                        -- extra random 0..N seconds added to every gap (gap is never below ACTION_GAP)
local ANNOUNCE_WORD_SECONDS = 0.2                  -- the game types an announcement word by word; seconds per word (tune to match)
local ANNOUNCE_FADE = 3                            -- seconds it stays and fades after the last word (the game ignores a new one until then)
local ANNOUNCE_JITTER = 0.4
local MAX_ANNOUNCE_QUEUE = 20                       -- pending announcements cap (oldest dropped)
local function jitteredGap() return ACTION_GAP + math.random() * ACTION_JITTER end
local REMOVE_GAP = 1.5                             -- removal events (unkick, releasing a timeout) wait at least this long, plus 0..REMOVE_JITTER
local REMOVE_JITTER = 0.5
local function removalGap() return REMOVE_GAP + math.random() * REMOVE_JITTER end
local KICK_SPACING = 1.5                           -- min seconds between unverified-player kicks (plus 0..KICK_JITTER random)
local KICK_JITTER = 0.5
local UNKICK_DELAY_SECONDS = 30                  -- after the match ends and everyone is kicked, wait this long, then unkick them one by one
local MAX_QUEUE = 60                             -- pending remotes cap: a runaway loop can't grow the queue (and the sort per push) forever
local MAX_BOMB_HISTORY = 20                      -- bomb-holder chain cap
local FUSE_SECONDS = 10                          -- the bomb always explodes 10s after the fuse resets (every explosion)
local HOT_PASS_WINDOW = 2                        -- receiving the bomb with this many seconds or less left on the fuse and passing it to an enemy is a hot pass (late track)
local MAX_PENDING_FINALS = 10                    -- unacked match results kept while the bot is unreachable
local MAX_KICKS_PER_MINUTE = 25                 -- safety valve: a bug can never mass-kick the server
-- Staff / spectator accounts the script must never count, kick or time out: [userId] = true
local IGNORED_USER_IDS = {
	-- [1234567890] = true,
}

-- The game's own kick call is KickPlayer(userId, minutes); every kick sends the lockout minutes (a boolean 3rd arg is not what the game sends).
local function buildKickArgs(userId, minutes)
	return { "KickPlayer", userId, minutes or KICK_DURATION }
end

-- KickPlayer with false as the 3rd arg removes a player's kick timer (an unkick), so they can rejoin
local function buildUnkickArgs(userId)
	return { "KickPlayer", userId, false }
end

-- ChangeTimeout(userId, true) = sit out (stays in the server, cannot play); false = release. Persists across rejoin.
local function buildTimeoutArgs(userId, on)
	return { "ChangeTimeout", userId, on }
end

--// Rules (mirror of core/services/scrim_rules.py; vectors are self-checked below)
local Rules = { WIN_BY = 2, CAP_OVER = 3, MAX_PER_TEAM = 5, MAX_BIG_PER_TEAM = 10 } -- hard cap = points to win + CAP_OVER

function Rules.pointsToWin(perSide)
	return perSide >= 4 and 11 or 7
end

function Rules.isMatchOver(a, b, perSide)
	local hi, lo = math.max(a, b), math.min(a, b)
	local target = Rules.pointsToWin(perSide)
	if hi >= target + Rules.CAP_OVER then return true end
	return hi >= target and hi - lo >= Rules.WIN_BY
end

-- game point: the match is not over, but winning the next round would end it
function Rules.isGamePoint(a, b, perSide, team)
	if Rules.isMatchOver(a, b, perSide) then return false end
	if team == "A" then return Rules.isMatchOver(a + 1, b, perSide) end
	return Rules.isMatchOver(a, b + 1, perSide)
end

function Rules.teamSize(playerCount, minPlayers)
	local perSide = math.min(playerCount // 2, Rules.MAX_PER_TEAM)
	if perSide < 1 or perSide * 2 < minPlayers then return 0 end
	return perSide
end

-- returns sideA, sideB, surplus (arrays of ids)
function Rules.pickTeams(ids, perSide, rng)
	rng = rng or Random.new()
	local pool = table.clone(ids)
	for i = #pool, 2, -1 do
		local j = rng:NextInteger(1, i)
		pool[i], pool[j] = pool[j], pool[i]
	end
	local a, b, surplus = {}, {}, {}
	for i, id in ipairs(pool) do
		if i <= perSide then table.insert(a, id)
		elseif i <= perSide * 2 then table.insert(b, id)
		else table.insert(surplus, id) end
	end
	return a, b, surplus
end

assert(not Rules.isMatchOver(6, 0, 2) and Rules.isMatchOver(7, 0, 2) and Rules.isMatchOver(7, 5, 3))
assert(not Rules.isMatchOver(7, 6, 2) and Rules.isMatchOver(8, 6, 2) and not Rules.isMatchOver(8, 7, 3))
assert(not Rules.isMatchOver(9, 9, 2) and Rules.isMatchOver(10, 9, 2) and Rules.isMatchOver(9, 10, 3))
assert(not Rules.isMatchOver(10, 0, 4) and Rules.isMatchOver(11, 0, 4) and Rules.isMatchOver(11, 9, 5))
assert(not Rules.isMatchOver(11, 10, 4) and Rules.isMatchOver(12, 10, 4) and not Rules.isMatchOver(13, 13, 5) and Rules.isMatchOver(14, 13, 5))
assert(Rules.teamSize(5, 6) == 0 and Rules.teamSize(6, 6) == 3 and Rules.teamSize(9, 6) == 4 and Rules.teamSize(14, 6) == 5)

--// Action facade. Replaced by a rate-limited queue in the SCRIM AGENT section at the bottom.
local Act = {
	fire = function(...) AnnounceEvent:FireServer(...) end,
	fireFirst = function(...) AnnounceEvent:FireServer(...) end,
	announce = function(msg) AnnounceEvent:FireServer("SendMessage", msg) end,
	announceNow = function(msg) AnnounceEvent:FireServer("SendMessage", msg) end,
}
local Agent = { active = false }

-- Fill this in once you find the real "stop round" remote call (see MainScript).
-- Example: STOP_ROUND_ARGS = { "StopMinigame" }   (leave nil to disable)
local STOP_ROUND_ARGS = { "StopMinigame" }

local SeriesScore = { [TEAM_A_NAME] = 0, [TEAM_B_NAME] = 0 }
local SeriesActive = false
local Paused = false -- /scrim console Pause: no new round starts and the lobby, countdown and break timers stand still
local SeriesPerSide = 2 -- players per team in the live match (decides first to 7 or 11); set when the match goes live
local PlayerTeams = {}          -- { ["PlayerName"] = "Black" }

local MatchStats = {}
local RoundActive = false
local RoundDecided = false
local deadPlayersThisRound = {}
local roundKills = {}                  -- { ["PlayerName"] = enemy kills this round } (an ace = all of them)
local clutchSeed = {}                  -- { [team] = { name, enemies } } the moment a team was down to one player, and how many enemies were still alive
local deathClock = {}              -- { ["PlayerName"] = os.clock() } of this round's deaths
local roundParticipants = {}    -- { ["PlayerName"] = true } snapshot at round start
local BombHistory = {}          -- ordered list of bomb holders since last explosion
local LastBombHolderName = nil
local fuseStart = nil           -- os.clock() when the first holder after the last explosion got the bomb
local heldSince = nil           -- os.clock() when the current holder got the bomb
local roundFirstOut = false     -- true once someone has died this round
local Running = true

--// ROUND TIMELINE: sent with the final so the site can show each round (validated by the bot's clean_timeline).
--// Times are ms since the round started. Players are UserIds (names change); 0 = nobody.
local TIMELINE_MAX_ROUNDS, TIMELINE_MAX_DEATHS, TIMELINE_MAX_EVENTS = 40, 20, 60
local Timeline = nil        -- { v = 1, rounds = {}, events = {}, truncated = bool } while a match is live
local TimelineRound = nil   -- { n, t0, deaths } for the round being played
local timelineIds = {}      -- [name] = UserId, remembered so players who left still resolve

local function timelineReset(truncated)
	Timeline = { v = 1, rounds = {}, events = {}, truncated = truncated == true }
	TimelineRound, timelineIds = nil, {}
end

local function timelineId(name)
	if not name then return 0 end
	local p = Players:FindFirstChild(name)
	if p and p:IsA("Player") then timelineIds[name] = p.UserId end
	return timelineIds[name] or 0
end

local function timelineMs(clock)
	if not TimelineRound then return 0 end
	return math.max(0, math.floor(((clock or os.clock()) - TimelineRound.t0) * 1000))
end

local function timelineRoundStart()
	if not (Timeline and SeriesActive) then return end
	TimelineRound = { n = #Timeline.rounds + 1, t0 = os.clock(), deaths = {} }
	for name in pairs(roundParticipants) do timelineId(name) end -- remember ids now, in case someone leaves mid-round
end

-- one row per victim: { ms, { victimId }, killerId, assisterId, teamKill }
local function timelineDeath(victim, killer, assister, teamKill)
	if not (Timeline and TimelineRound) then return end
	if #TimelineRound.deaths >= TIMELINE_MAX_DEATHS then Timeline.truncated = true return end
	table.insert(TimelineRound.deaths, { timelineMs(deathClock[victim]), { timelineId(victim) }, timelineId(killer), timelineId(assister), teamKill and 1 or 0 })
end

local function timelineRoundEnd(winner, tiebreak)
	if not (Timeline and TimelineRound) then return end
	if #Timeline.rounds >= TIMELINE_MAX_ROUNDS then Timeline.truncated = true TimelineRound = nil return end
	local players = {}
	for name in pairs(roundParticipants) do
		local id = timelineId(name)
		if id > 0 then table.insert(players, id) end
	end
	table.insert(Timeline.rounds, {
		n = TimelineRound.n,
		winner = winner, -- nil = draw, replayed without a point
		score = { SeriesScore[TEAM_A_NAME], SeriesScore[TEAM_B_NAME] },
		ms = timelineMs(),
		tiebreak = tiebreak == true,
		players = players,
		deaths = TimelineRound.deaths,
	})
	TimelineRound = nil
end

-- kind: "leave" | "rejoin" | "votekick" | "assign"
local function timelineEvent(kind, userId, extra)
	if not Timeline or type(userId) ~= "number" or userId <= 0 then return end
	if #Timeline.events >= TIMELINE_MAX_EVENTS then Timeline.truncated = true return end
	local n = TimelineRound and TimelineRound.n or (#Timeline.rounds + 1)
	table.insert(Timeline.events, { n, timelineMs(), kind, userId, extra })
end

local function timelineExport()
	if not Timeline or (#Timeline.rounds == 0 and #Timeline.events == 0) then return nil end
	return { v = 1, rounds = Timeline.rounds, events = Timeline.events, truncated = Timeline.truncated }
end

-- RoundsPlayed / RoundsSurvived: decided rounds this player started, and was still alive at the end of.
-- HoldMs / Holds: total time holding the bomb, and how many times. FirstOuts: rounds where this player died first.
local STAT_KEYS = { "Kills", "Deaths", "Assists", "TeamKills", "Aces", "Clutches", "RoundsPlayed", "RoundsSurvived", "HoldMs", "Holds", "HotPasses", "FirstOuts" }

local function copyStat(s)
	local out = {}
	for _, k in ipairs(STAT_KEYS) do out[k] = (s and s[k]) or 0 end
	return out
end

local function ensureStats(name)
	if not MatchStats[name] then MatchStats[name] = copyStat(nil) end
	return MatchStats[name]
end

-- close the current holder's hold (on a pass or an explosion)
local function endHold(name, now)
	if not (SeriesActive and name and heldSince) then return end
	local s = ensureStats(name)
	s.HoldMs += math.floor((now - heldSince) * 1000 + 0.5)
	s.Holds += 1
end

-- a new bomb holder: closes the last holder's hold and counts a hot pass if they got it late and passed it to an enemy
local function noteHolder(name)
	local now = os.clock()
	local prev = BombHistory[#BombHistory]
	if prev then
		endHold(prev, now)
		local t1, t2 = PlayerTeams[prev], PlayerTeams[name]
		if SeriesActive and fuseStart and t1 and t2 and t1 ~= t2 and heldSince and FUSE_SECONDS - (heldSince - fuseStart) <= HOT_PASS_WINDOW then
			ensureStats(prev).HotPasses += 1
		end
	else
		fuseStart = now -- first holder since the explosion: the fuse just reset
	end
	heldSince = now
	table.insert(BombHistory, name)
	if #BombHistory > MAX_BOMB_HISTORY then table.remove(BombHistory, 1) end
end

local function printStats()
	print("========== MATCH STATS ==========")
	for name, s in pairs(MatchStats) do
		print(string.format("%s [%s]  K:%d  D:%d  A:%d", name, PlayerTeams[name] or "?", s.Kills, s.Deaths, s.Assists))
	end
	print("=================================")
end

local function updatePlayerNametag(player)
	if player.Character and player.Character:FindFirstChild("Head") then
		local tag = player.Character.Head:FindFirstChild("TrackerNametag")
		if tag and tag:FindFirstChild("TeamLabel") then
			local team = PlayerTeams[player.Name]
			if team then
				tag.TeamLabel.Text = "[" .. team .. "]"
				tag.TeamLabel.TextColor3 = (team == TEAM_A_NAME) and Color3.fromRGB(255, 100, 100) or Color3.fromRGB(100, 100, 255)
				tag.TeamLabel.Visible = true
			else
				tag.TeamLabel.Visible = false
			end
		end
	end
end

--// =========================================================
--// UI
--// =========================================================

local UI = Instance.new("ScreenGui")
UI.Name = "TrackerUI"
UI.ResetOnSpawn = false
UI.IgnoreGuiInset = true
pcall(function() UI.Parent = game:GetService("CoreGui") end)
if not UI.Parent then UI.Parent = LocalPlayer:WaitForChild("PlayerGui") end

-- every event connection goes through on() so Unload can disconnect them (a re-run must not leave the old copy's handlers running)
local Conns = {}
local function on(signal, fn)
	local c = signal:Connect(fn)
	if #Conns >= 500 then -- per-character connections die with the character: drop those so the list doesn't grow forever
		local live = {}
		for _, old in ipairs(Conns) do if old.Connected then table.insert(live, old) end end
		Conns = live
	end
	table.insert(Conns, c)
	return c
end

local Unload = function()
	Running = false
	for _, c in ipairs(Conns) do c:Disconnect() end
	table.clear(Conns)
	UI:Destroy()
end
do
	local genv = getgenv and getgenv() or _G
	if genv.ScrimUnload then pcall(genv.ScrimUnload) end -- stop an older copy before this one runs
	genv.ScrimUnload = Unload
end

local function makePanel(name, title, size, pos)
	local f = Instance.new("Frame")
	f.Name = name
	f.Size = size
	f.Position = pos
	f.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
	f.Active = true
	f.Draggable = true
	f.ZIndex = 60
	f.Parent = UI
	local t = Instance.new("TextLabel")
	t.Size = UDim2.new(1, 0, 0, 22)
	t.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
	t.Text = title
	t.TextColor3 = Color3.fromRGB(255, 255, 255)
	t.Font = Enum.Font.GothamBold
	t.TextSize = 12
	t.ZIndex = 61
	t.Parent = f
	return f
end

--// KILLFEED
local KillFeedContainer = Instance.new("Frame")
KillFeedContainer.Name = "KillFeedContainer"
KillFeedContainer.AnchorPoint = Vector2.new(1, 0)
KillFeedContainer.Position = UDim2.new(1, -16, 0, 60)
KillFeedContainer.Size = UDim2.fromOffset(380, 200)
KillFeedContainer.BackgroundTransparency = 1
KillFeedContainer.ZIndex = 50
KillFeedContainer.Parent = UI

local KillFeedLayout = Instance.new("UIListLayout")
KillFeedLayout.FillDirection = Enum.FillDirection.Vertical
KillFeedLayout.VerticalAlignment = Enum.VerticalAlignment.Top
KillFeedLayout.HorizontalAlignment = Enum.HorizontalAlignment.Right
KillFeedLayout.Padding = UDim.new(0, 4)
KillFeedLayout.SortOrder = Enum.SortOrder.LayoutOrder
KillFeedLayout.Parent = KillFeedContainer

local MAX_KILLFEED_ENTRIES = 5
local KILLFEED_DURATION = 5
local killfeedEntries = {}
local feedOrder = 0

local function getTeamColor(playerName)
	local team = PlayerTeams[playerName]
	if team == TEAM_A_NAME then return Color3.fromRGB(255, 90, 90)
	elseif team == TEAM_B_NAME then return Color3.fromRGB(90, 130, 255)
	else return Color3.fromRGB(200, 200, 200) end
end

local function makeLabel(parent, text, color, size, pos, align, textSize)
	local l = Instance.new("TextLabel")
	l.Size = size
	l.Position = pos
	l.BackgroundTransparency = 1
	l.Text = text
	l.TextColor3 = color
	l.Font = Enum.Font.GothamBold
	l.TextSize = textSize or 14
	l.TextXAlignment = align
	l.TextTruncate = Enum.TextTruncate.AtEnd
	l.ZIndex = 51
	l.Parent = parent
	return l
end

local function pushFeedEntry(bgColor, bgTransparency, build)
	if not Running or not KillFeedContainer.Parent then return end
	if #killfeedEntries >= MAX_KILLFEED_ENTRIES then
		local oldest = table.remove(killfeedEntries, 1)
		if oldest then oldest:Destroy() end
	end

	feedOrder += 1
	local entry = Instance.new("Frame")
	entry.Name = "KillFeedEntry"
	entry.Size = UDim2.fromOffset(370, 32)
	entry.BackgroundColor3 = bgColor
	entry.BackgroundTransparency = bgTransparency
	entry.BorderSizePixel = 0
	entry.LayoutOrder = feedOrder
	entry.ZIndex = 50
	entry.Parent = KillFeedContainer

	local corner = Instance.new("UICorner")
	corner.CornerRadius = UDim.new(0, 6)
	corner.Parent = entry
	local pad = Instance.new("UIPadding")
	pad.PaddingLeft = UDim.new(0, 8)
	pad.PaddingRight = UDim.new(0, 8)
	pad.Parent = entry

	build(entry)
	table.insert(killfeedEntries, entry)

	task.delay(KILLFEED_DURATION, function()
		if not entry or not entry.Parent then return end
		local info = TweenInfo.new(0.5, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
		TweenService:Create(entry, info, { BackgroundTransparency = 1 }):Play()
		for _, c in ipairs(entry:GetChildren()) do
			if c:IsA("TextLabel") then TweenService:Create(c, info, { TextTransparency = 1 }):Play() end
		end
		task.wait(0.6)
		local idx = table.find(killfeedEntries, entry)
		if idx then table.remove(killfeedEntries, idx) end
		entry:Destroy()
	end)
end

local function announceKill(victim, killer)
	if not killer or killer == victim then
		print("[KILLFEED] " .. victim .. " died / self-destructed")
		pushFeedEntry(Color3.fromRGB(10, 10, 10), 0.35, function(e)
			makeLabel(e, "" .. victim .. "  (self-destruct)", Color3.fromRGB(180, 180, 180),
				UDim2.new(1, 0, 1, 0), UDim2.new(), Enum.TextXAlignment.Right)
		end)
	else
		print("[KILLFEED] " .. killer .. " -> " .. victim)
		pushFeedEntry(Color3.fromRGB(10, 10, 10), 0.35, function(e)
			makeLabel(e, killer, getTeamColor(killer), UDim2.new(0.44, 0, 1, 0), UDim2.new(0, 0, 0, 0), Enum.TextXAlignment.Right)
			makeLabel(e, ">", Color3.fromRGB(255, 255, 255), UDim2.new(0.12, 0, 1, 0), UDim2.new(0.44, 0, 0, 0), Enum.TextXAlignment.Center, 16)
			makeLabel(e, victim, getTeamColor(victim), UDim2.new(0.44, 0, 1, 0), UDim2.new(0.56, 0, 0, 0), Enum.TextXAlignment.Left)
		end)
	end
end

--// =========================================================
--// GAME LOOP (FIRST TO 3)
--// =========================================================

local function getAliveCount(teamName)
	local n = 0
	for name in pairs(roundParticipants) do
		if PlayerTeams[name] == teamName and not deadPlayersThisRound[name] and Players:FindFirstChild(name) then
			n += 1
		end
	end
	return n
end

local function broadcastMessage(msg) -- round results: jump the queue so the score shows as soon as the round ends
	Act.announceNow(msg)
	print("[GAMELOOP] " .. msg)
end

local checkRoundState -- forward declaration

-- match round mode, set from Discord (/scrim config mode or the console): 0 = normal, 1 = shrink
local ROUND_MODES = { [0] = { "LeagueStandard", "Normal" }, [1] = { "Shrink", "Multi" } }
local GameMode = 0
local function roundArgs()
	local m = ROUND_MODES[GameMode] or ROUND_MODES[0]
	return "StartMinigame", "Timebomb", m[1], m[2]
end

local LastStopAt = -100 -- when we last stopped a round: team removals right after it are not deaths

local function stopRound()
	LastStopAt = os.clock()
	if STOP_ROUND_ARGS then
		Act.fireFirst(table.unpack(STOP_ROUND_ARGS)) -- ahead of any queued kicks
		print("[GAMELOOP] Stop round fired")
	end
end

local NextRoundPending = false -- a startNextRound is running (it may be waiting out a pause)

local function nextRound()
	-- one team is wiped, so end the current round before starting the next
	stopRound()
	local timeout = 10
	while RoundActive and timeout > 0 do
		task.wait(1)
		timeout -= 1
	end
	task.wait(2)
	if not SeriesActive then return end -- the match ended (forfeit, /scrim stop) while we waited: no new round
	while Paused and SeriesActive do task.wait(0.5) end -- paused by an admin: hold the next round until they resume
	if not SeriesActive then return end
	if Agent.beforeRound then Agent.beforeRound() end -- bench latecomers before the next bomb round
	if not SeriesActive then return end -- clearing the field can take up to 30s
	Act.fire(roundArgs())
	print("[GAMELOOP] Next round started")

	-- If the server never emptied the Playing team, force a fresh round state
	task.wait(3)
	if RoundDecided and SeriesActive then
		RoundActive = false
		checkRoundState()
	end
end

local function startNextRound()
	NextRoundPending = true
	local ok, err = pcall(nextRound)
	NextRoundPending = false
	if not ok then warn("[GAMELOOP] next round failed: " .. tostring(err)) end
end

local function checkRoundWinner()
	if not SeriesActive or RoundDecided then return end

	-- need at least one assigned player on each team, otherwise it's not a real match round
	local sizeA, sizeB = 0, 0
	for name in pairs(roundParticipants) do
		local t = PlayerTeams[name]
		if t == TEAM_A_NAME then sizeA += 1 elseif t == TEAM_B_NAME then sizeB += 1 end
	end
	if sizeA == 0 or sizeB == 0 then return end

	local aliveA = getAliveCount(TEAM_A_NAME)
	local aliveB = getAliveCount(TEAM_B_NAME)
	if aliveA > 0 and aliveB > 0 then return end

	local roundWinner = nil
	local tiebreak = false
	if aliveA == 0 and aliveB > 0 then
		roundWinner = TEAM_B_NAME
	elseif aliveB == 0 and aliveA > 0 then
		roundWinner = TEAM_A_NAME
	else
		-- both wiped at once: team that held the bomb last loses
		tiebreak = true
		if LastBombHolderName then
			local bombTeam = PlayerTeams[LastBombHolderName]
			if bombTeam == TEAM_A_NAME then roundWinner = TEAM_B_NAME
			elseif bombTeam == TEAM_B_NAME then roundWinner = TEAM_A_NAME end
		end
		if not roundWinner then
			broadcastMessage("Draw!")
			RoundDecided = true
			timelineRoundEnd(nil, false)
			task.spawn(startNextRound)
			return
		end
	end

	RoundDecided = true
	SeriesScore[roundWinner] += 1
	timelineRoundEnd(roundWinner, tiebreak) -- before onSeriesOver below, which sends the timeline
	for name in pairs(roundParticipants) do
		if PlayerTeams[name] then
			local s = ensureStats(name)
			s.RoundsPlayed += 1
			if not deadPlayersThisRound[name] and Players:FindFirstChild(name) then s.RoundsSurvived += 1 end
		end
	end
	-- moments the bot announces in Discord. Ace: one player killed a full other team's worth of players, i.e. 5 in 5v5 (5v5 only; enemies who left don't count).
	-- Clutch: the last player alive beat 3 or more enemies. Flawless: the team won without losing a player (both from 3v3 up).
	local participants = {}
	for name in pairs(roundParticipants) do
		local t = PlayerTeams[name]
		if t then participants[t] = (participants[t] or 0) + 1 end
	end
	local aced = {}
	if SeriesPerSide >= 5 then
		for killer, n in pairs(roundKills) do
			local team = PlayerTeams[killer]
			local enemy = team == TEAM_A_NAME and TEAM_B_NAME or TEAM_A_NAME
			if team == roundWinner and n >= SeriesPerSide and n >= (participants[enemy] or math.huge) then -- enemies who left don't shrink the target
				aced[killer] = true
				ensureStats(killer).Aces += 1
				if Agent.onMoment then Agent.onMoment("ace", killer, team) end
			end
		end
	end
	if SeriesPerSide >= 3 and Agent.onMoment then
		local seed = clutchSeed[roundWinner]
		if seed and seed.enemies >= 3 and not deadPlayersThisRound[seed.name] then
			ensureStats(seed.name).Clutches += 1
			-- 1 vs the whole enemy team (1v5 in 5v5) is an ace too: after an explosion the bomb respawns on a random player and
			-- enemies sac each other, so those deaths never credit the survivor's roundKills even though they outlasted all five
			local enemy = roundWinner == TEAM_A_NAME and TEAM_B_NAME or TEAM_A_NAME
			if SeriesPerSide >= 5 and seed.enemies >= SeriesPerSide and seed.enemies >= (participants[enemy] or math.huge) then
				if not aced[seed.name] then
					ensureStats(seed.name).Aces += 1
					Agent.onMoment("ace", seed.name, roundWinner)
				end
			else
				Agent.onMoment("clutch", seed.name, roundWinner, seed.enemies)
			end
		end
		if participants[roundWinner] and getAliveCount(roundWinner) == participants[roundWinner] then
			Agent.onMoment("flawless", nil, roundWinner)
		end
	end
	local scoreMsg = TEAM_A_NAME .. " " .. SeriesScore[TEAM_A_NAME] .. " - " .. SeriesScore[TEAM_B_NAME] .. " " .. TEAM_B_NAME
	local seriesOver = Rules.isMatchOver(SeriesScore[TEAM_A_NAME], SeriesScore[TEAM_B_NAME], SeriesPerSide)
	if seriesOver then
		scoreMsg = scoreMsg .. " GG"
	else
		local sa, sb = SeriesScore[TEAM_A_NAME], SeriesScore[TEAM_B_NAME]
		local gpA, gpB = Rules.isGamePoint(sa, sb, SeriesPerSide, "A"), Rules.isGamePoint(sa, sb, SeriesPerSide, "B")
		if gpA and gpB then scoreMsg = scoreMsg .. " | GP: both teams"
		elseif gpA then scoreMsg = scoreMsg .. " | GP: " .. TEAM_A_NAME
		elseif gpB then scoreMsg = scoreMsg .. " | GP: " .. TEAM_B_NAME end
	end
	broadcastMessage(scoreMsg)

	if seriesOver then
		stopRound()
		printStats()
		if Agent.onSeriesOver then Agent.onSeriesOver(roundWinner, SeriesScore, MatchStats) end
		SeriesScore = { [TEAM_A_NAME] = 0, [TEAM_B_NAME] = 0 }
		SeriesActive = false
		MatchStats = {}
	else
		task.spawn(startNextRound)
	end
end

--// =========================================================
--// DEATH + KILL CREDIT
--// =========================================================
-- How credit works:
--  * BombHistory is the ordered list of players who held the bomb since the last explosion.
--  * The moment the CURRENT bomb holder dies, we snapshot the chain and reset BombHistory,
--    so a new bomb / dead-player tool events can't pollute it.
--  * Killer = the player who passed the bomb last. If the victim is their teammate it is a team kill (TeamKills, not Kills).
--    No previous holder = self-kill (no kill credit).
--  * Assist = the third-last holder (who passed it to the killer), only if on the killer's team; nobody else.

--// KO TRACKING: the game's own KOs stat (leaderstats) says who got the kill. A KO that goes up is a kill;
--// the victim is whoever just left the Playing team (see hookPlayerTeam). Without a KOs stat we fall back to the bomb chain.
local KO_STAT_NAME = "KOs"
local KO_BEFORE = 2          -- a KO counts for a death if the stat rose up to this many seconds before the death
local koEvents = {}          -- { { name = "Killer", t = os.clock(), left = 1 } } KO increments not yet matched to a death
local koTracking = false     -- true once any player's KOs stat has been found

local function hookKOs(player)
	local ls = player:WaitForChild("leaderstats", 10)
	if not ls then return end
	local stat = ls:FindFirstChild(KO_STAT_NAME) or ls:WaitForChild(KO_STAT_NAME, 5)
	if not stat or not stat:IsA("ValueBase") then return end
	koTracking = true
	local last = stat.Value
	on(stat:GetPropertyChangedSignal("Value"), function()
		local v = stat.Value
		if type(v) == "number" and type(last) == "number" and v > last and RoundActive and SeriesActive then -- SeriesActive: not during the captain rounds
			table.insert(koEvents, { name = player.Name, t = os.clock(), left = v - last })
			print("[KO] " .. player.Name .. " KOs " .. last .. " -> " .. v)
		end
		last = v
	end)
end

-- the oldest unmatched KO that rose shortly before this death
local function takeKO(deathTime)
	for _, e in ipairs(koEvents) do
		if e.left > 0 and e.t >= deathTime - KO_BEFORE then
			e.left -= 1
			return e.name
		end
	end
	return nil
end

local function pruneKOs()
	local now = os.clock()
	for i = #koEvents, 1, -1 do
		if koEvents[i].left <= 0 or now - koEvents[i].t > 15 then table.remove(koEvents, i) end
	end
end

local pendingDeaths = {}
local pendingBomb = nil
local processScheduled = false

local function resolveBombCredit(history)
	local holder = history[#history]
	local victimTeam = PlayerTeams[holder]

	-- the player who passed the bomb last gets the kill, even when the victim is their own teammate
	local killer, teamKill = nil, false
	local prev = history[#history - 1]
	if prev and prev ~= holder then
		killer = prev
		teamKill = victimTeam ~= nil and PlayerTeams[prev] == victimTeam
	end

	-- assist: only the player who passed the bomb to the killer (third last in the chain), and only if a teammate of the killer
	local assisters = {}
	if killer and not teamKill then
		local passer = history[#history - 2]
		local killerTeam = PlayerTeams[killer]
		if passer and passer ~= killer and passer ~= holder and killerTeam and PlayerTeams[passer] == killerTeam then
			table.insert(assisters, passer)
		end
	end

	local tagged = {}
	for i, n in ipairs(history) do tagged[i] = n .. "(" .. tostring(PlayerTeams[n] or "?") .. ")" end
	print("[CREDIT] chain: " .. table.concat(tagged, " > ") .. " | killer: " .. tostring(killer) .. (teamKill and " (team kill)" or "") .. " | assists: " .. #assisters)
	return { holder = holder, killer = killer, assisters = assisters, teamKill = teamKill }
end

local function processPendingDeaths()
	if #pendingDeaths == 0 then return end
	local batch = pendingDeaths
	pendingDeaths = {}
	local bomb = pendingBomb
	pendingBomb = nil

	-- There are no ties: right after the final explosion the game teleports/resets the survivor, which looks like a
	-- second death. If this batch wipes out everyone still playing, only the bomb holder (and their teammates) died.
	if bomb and bomb.holder then
		local stillAlive = 0
		for name in pairs(roundParticipants) do
			if not deadPlayersThisRound[name] and Players:FindFirstChild(name) then stillAlive += 1 end
		end
		if stillAlive == 0 then
			local holderTeam = PlayerTeams[bomb.holder]
			local kept = {}
			for _, v in ipairs(batch) do
				if v == bomb.holder or (holderTeam and PlayerTeams[v] == holderTeam) then
					table.insert(kept, v)
				else
					deadPlayersThisRound[v] = nil
					deathClock[v] = nil
					print("[DEATH] " .. v .. " was reset after the final explosion; not a death")
				end
			end
			batch = kept
		end
	end

	if not roundFirstOut and batch[1] and SeriesActive then
		roundFirstOut = true
		ensureStats(batch[1]).FirstOuts += 1
	end

	local credited = {}
	local assistLogged = false -- the stats count the assist once per explosion; log it on the first row only
	for _, victim in ipairs(batch) do
		local killer
		if koTracking then
			killer = takeKO(deathClock[victim] or os.clock())
		else
			killer = bomb and bomb.killer or nil
		end
		if killer == victim then killer = nil end

		ensureStats(victim).Deaths += 1
		if killer then
			if PlayerTeams[killer] and PlayerTeams[victim] and PlayerTeams[killer] == PlayerTeams[victim] then
				ensureStats(killer).TeamKills += 1 -- a sac: no kill credit, counts against the passer
			else
				ensureStats(killer).Kills += 1
				if PlayerTeams[killer] and PlayerTeams[victim] then
					roundKills[killer] = (roundKills[killer] or 0) + 1
				end
			end
			credited[killer] = true
		end
		do -- same killer/assist rules as the stats above, so the timeline always matches the totals
			local teamKill = killer ~= nil and PlayerTeams[killer] ~= nil and PlayerTeams[killer] == PlayerTeams[victim]
			local assister = not assistLogged and bomb and killer and bomb.killer == killer and not bomb.teamKill and bomb.assisters[1] or nil
			if assister then assistLogged = true end
			timelineDeath(victim, killer, assister, teamKill)
		end
		print(string.format("[DEATH] %s(%s) killed by %s", victim, tostring(PlayerTeams[victim] or "?"), tostring(killer)))
		announceKill(victim, killer)
	end
	pruneKOs()

	for _, team in ipairs({ TEAM_A_NAME, TEAM_B_NAME }) do
		if not clutchSeed[team] then
			local lastName, left = nil, 0
			for name in pairs(roundParticipants) do
				if PlayerTeams[name] == team and not deadPlayersThisRound[name] and Players:FindFirstChild(name) then
					lastName, left = name, left + 1
				end
			end
			if left == 1 then
				clutchSeed[team] = { name = lastName, enemies = getAliveCount(team == TEAM_A_NAME and TEAM_B_NAME or TEAM_A_NAME) }
			end
		end
	end

	-- assist: the player who passed the bomb to the credited killer (third last in the chain, a teammate of the killer).
	-- Only when the killer is the one who passed it to the victim, and never for a team kill.
	if bomb and bomb.killer and not bomb.teamKill and credited[bomb.killer] then
		for _, a in ipairs(bomb.assisters) do
			ensureStats(a).Assists += 1
			print("[ASSIST] " .. a .. " (passed it to " .. bomb.killer .. ")")
		end
	end

	checkRoundWinner()
end

local function handleDeath(victimName)
	if not RoundActive or RoundDecided then return end
	if not roundParticipants[victimName] then return end
	if deadPlayersThisRound[victimName] then return end

	deadPlayersThisRound[victimName] = true
	deathClock[victimName] = os.clock()
	table.insert(pendingDeaths, victimName)

	-- snapshot the bomb chain at the exact moment of death
	if not pendingBomb and BombHistory[#BombHistory] == victimName then
		endHold(victimName, os.clock())
		heldSince, fuseStart = nil, nil
		pendingBomb = resolveBombCredit(BombHistory)
		LastBombHolderName = victimName
		BombHistory = {}
	end

	-- short window so every victim of the same explosion lands in one batch
	if not processScheduled then
		processScheduled = true
		task.delay(0.6, function() -- long enough for the KOs stat to replicate
			processScheduled = false
			processPendingDeaths()
		end)
	end
end

local function getHeldObject(character)
	if not character then return nil end
	local player = Players:GetPlayerFromCharacter(character)

	local function checkContainer(container)
		if not container then return nil end
		for _, child in ipairs(container:GetChildren()) do
			local name = string.lower(child.Name)
			if string.find(name, "bomb") or string.find(name, "juggernaut") then
				return child
			end
		end
		return nil
	end

	local obj = checkContainer(character)
	if obj then return obj end
	if player and player:FindFirstChild("Backpack") then
		return checkContainer(player.Backpack)
	end
	return nil
end

--// =========================================================
--// NAMETAGS & BOMB TRACKER
--// =========================================================

local function setupPlayerTracker(player)
	local function fireUpdate()
		if player.Character then
			local fn = player.Character:FindFirstChild("UpdateToolStatusBindable")
			if fn then fn:Fire() end
		end
	end

	local function hookBackpack(bp)
		if not bp then return end
		on(bp.ChildAdded, fireUpdate)
		on(bp.ChildRemoved, fireUpdate)
	end

	hookBackpack(player:FindFirstChild("Backpack"))
	on(player.ChildAdded, function(child)
		if child.Name == "Backpack" then
			hookBackpack(child)
			fireUpdate()
		end
	end)

	local function setupCharacter(character)
		local head = character:WaitForChild("Head", 5)
		local humanoid = character:WaitForChild("Humanoid", 5)
		if not humanoid then return end

		local ToolLabel = nil
		if player ~= LocalPlayer and head then
			local oldTag = head:FindFirstChild("TrackerNametag")
			if oldTag then oldTag:Destroy() end

			local Billboard = Instance.new("BillboardGui")
			Billboard.Name = "TrackerNametag"
			Billboard.Size = UDim2.fromOffset(220, 80)
			Billboard.StudsOffset = Vector3.new(0, 2.7, 0)
			Billboard.AlwaysOnTop = true
			Billboard.Parent = head

			local NameLabel = Instance.new("TextLabel")
			NameLabel.Size = UDim2.fromScale(1, 0.33)
			NameLabel.BackgroundTransparency = 1
			NameLabel.Text = player.DisplayName .. " (@" .. player.Name .. ")"
			NameLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
			NameLabel.Font = Enum.Font.GothamBold
			NameLabel.TextSize = 13
			NameLabel.Parent = Billboard

			local TeamLabel = Instance.new("TextLabel")
			TeamLabel.Name = "TeamLabel"
			TeamLabel.Size = UDim2.fromScale(1, 0.33)
			TeamLabel.Position = UDim2.fromScale(0, 0.33)
			TeamLabel.BackgroundTransparency = 1
			TeamLabel.Font = Enum.Font.GothamBlack
			TeamLabel.TextSize = 14
			TeamLabel.Visible = false
			TeamLabel.Parent = Billboard

			ToolLabel = Instance.new("TextLabel")
			ToolLabel.Name = "ToolLabel"
			ToolLabel.Size = UDim2.fromScale(1, 0.33)
			ToolLabel.Position = UDim2.fromScale(0, 0.66)
			ToolLabel.BackgroundTransparency = 1
			ToolLabel.TextColor3 = Color3.fromRGB(255, 85, 85)
			ToolLabel.Font = Enum.Font.GothamBold
			ToolLabel.TextSize = 12
			ToolLabel.Visible = false
			ToolLabel.Parent = Billboard

			updatePlayerNametag(player)
		end

		local function updateToolStatus()
			local heldObject = getHeldObject(character)
			if ToolLabel then
				if heldObject then
					ToolLabel.Text = "[Holding: " .. heldObject.Name .. "]"
					ToolLabel.Visible = true
				else
					ToolLabel.Text = ""
					ToolLabel.Visible = false
				end
			end

			-- Record bomb holders only: real bombs, alive players, active round.
			-- (Dead players still "holding" the tool for a moment used to pollute the chain.)
			if heldObject
				and string.find(string.lower(heldObject.Name), "bomb")
				and RoundActive
				and humanoid.Health > 0
				and not deadPlayersThisRound[player.Name]
				and BombHistory[#BombHistory] ~= player.Name then
				noteHolder(player.Name)
				print("[BOMB TRACKER] " .. player.Name .. " now holds the bomb (chain size " .. #BombHistory .. ")")
			end
		end

		local updateBindable = Instance.new("BindableEvent")
		updateBindable.Name = "UpdateToolStatusBindable"
		on(updateBindable.Event, updateToolStatus)
		updateBindable.Parent = character

		updateToolStatus()
		on(character.ChildAdded, updateToolStatus)
		on(character.ChildRemoved, updateToolStatus)

		on(humanoid.Died, function() handleDeath(player.Name) end)
		on(character.AncestryChanged, function(_, parent)
			-- a character removed while still alive is the game resetting the round, not a death
			if not parent and humanoid.Health <= 0 then handleDeath(player.Name) end
		end)
	end

	if player.Character then task.spawn(setupCharacter, player.Character) end
	on(player.CharacterAdded, setupCharacter)
end

for _, player in ipairs(Players:GetPlayers()) do setupPlayerTracker(player) end
on(Players.PlayerAdded, setupPlayerTracker)

--// =========================================================
--// ROUND DETECTOR
--// =========================================================

local function snapshotParticipants()
	for _, p in ipairs(Players:GetPlayers()) do
		if p.Team and p.Team.Name == PLAYING_TEAM_NAME then
			roundParticipants[p.Name] = true
		end
	end
end

local function getPlayingCount()
	local count = 0
	for _, p in ipairs(Players:GetPlayers()) do
		if p.Team and p.Team.Name == PLAYING_TEAM_NAME then count += 1 end
	end
	return count
end

local function beginRound()
	RoundActive = true
	RoundDecided = false
	deadPlayersThisRound = {}
	roundKills = {}
	clutchSeed = {}
	deathClock = {}
	roundParticipants = {}
	pendingDeaths = {}
	pendingBomb = nil
	BombHistory = {}
	LastBombHolderName = nil
	fuseStart, heldSince, roundFirstOut = nil, nil, false
	snapshotParticipants()
	timelineRoundStart()
	print("[ROUND START]")

	-- catch late team changes + the initial bomb holder
	task.delay(1, function()
		if not RoundActive then return end
		snapshotParticipants()
		for _, p in ipairs(Players:GetPlayers()) do
			if p.Character and roundParticipants[p.Name] then
				local held = getHeldObject(p.Character)
				if held and string.find(string.lower(held.Name), "bomb") and BombHistory[#BombHistory] ~= p.Name then
					noteHolder(p.Name)
					print("[BOMB TRACKER] Initial bomb holder: " .. p.Name)
				end
			end
		end
	end)
end

local isRoundEnding = false

checkRoundState = function()
	if getPlayingCount() <= 0 then
		if RoundActive and not isRoundEnding then
			isRoundEnding = true
			task.spawn(function()
				task.wait(1.5)
				isRoundEnding = false
				if getPlayingCount() > 0 then return end
				RoundActive = false
				BombHistory = {}
				print("[ROUND END]")
			end)
		end
		return
	end

	if not RoundActive then beginRound() end
end

local function isPlaying(player)
	return player.Team ~= nil and player.Team.Name == PLAYING_TEAM_NAME
end

local function hookPlayerTeam(player)
	local wasPlaying = isPlaying(player)
	local function onTeamChanged()
		local nowPlaying = isPlaying(player)
		-- A bomb death sends the player to Neutral while game.Teams.Playing still exists. The game destroys the whole team
		-- 1-2s after the last death, which also drops the survivors to Neutral: those removals find the team gone, so they
		-- are not deaths (a dead humanoid still counts). Removals right after we stop a round ourselves are ignored too.
		if wasPlaying and not nowPlaying and os.clock() - LastStopAt > 5 then
			local teamExists = TeamsService:FindFirstChild(PLAYING_TEAM_NAME) ~= nil
			local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
			if teamExists or (hum and hum.Health <= 0) then handleDeath(player.Name) end
		end
		wasPlaying = nowPlaying
		checkRoundState()
	end
	on(player:GetPropertyChangedSignal("Team"), onTeamChanged)
	on(player:GetPropertyChangedSignal("Neutral"), checkRoundState)
	task.spawn(hookKOs, player)
end

for _, player in ipairs(Players:GetPlayers()) do hookPlayerTeam(player) end
on(Players.PlayerAdded, function(player)
	hookPlayerTeam(player)
	task.defer(checkRoundState)
end)
on(Players.PlayerRemoving, function(player)
	if RoundActive then handleDeath(player.Name) end
	task.defer(checkRoundState)
end)

task.defer(checkRoundState)
--// =========================================================
--// SCRIM AGENT (lobby -> countdown -> match -> log -> kick all -> break -> repeat)
--// The Discord bot owns the session + verification data; this script is the real-time referee.
--// =========================================================

local HttpService = game:GetService("HttpService")
local httpRequest = (syn and syn.request) or (http and http.request) or http_request or (fluxus and fluxus.request) or request

-- the agent runs in its own function: as a plain do-block its ~110 locals shared the main chunk's 255-register limit
-- with the ~130 above, and going over makes the script fail to compile (the loader's loadstring then returns nil)
;(function()
	local agentEnabled = true
	if not httpRequest then
		warn("[SCRIM] This executor has no HTTP request function; the scrim agent is disabled.")
		agentEnabled = false
	elseif string.find(API_URL, "YOUR-", 1, true) or string.find(API_SECRET, "YOUR_", 1, true) then
		warn("[SCRIM] Set API_URL / API_SECRET at the top of the script; the scrim agent is disabled.")
		agentEnabled = false
	end

	--// ---- rate-limited action queue (every remote is >= ACTION_GAP apart) ----
	local queue, seq, lastFire = {}, 0, 0

	local function push(prio, label, fn, removal)
		if #queue >= MAX_QUEUE then
			warn("[SCRIM] action queue full (" .. MAX_QUEUE .. "); dropping: " .. label)
			return
		end
		seq += 1
		table.insert(queue, { prio = prio, seq = seq, fn = fn, label = label, removal = removal })
		table.sort(queue, function(x, y)
			if x.prio ~= y.prio then return x.prio < y.prio end
			return x.seq < y.seq
		end)
	end

	Act.fire = function(...)
		local args = table.pack(...)
		push(2, tostring(args[1]), function() AnnounceEvent:FireServer(table.unpack(args, 1, args.n)) end)
	end
	-- like Act.fire, but the remote only goes out if cond() is still true when its turn comes (the queue can be seconds behind)
	Act.fireIf = function(cond, ...)
		local args = table.pack(...)
		push(2, tostring(args[1]), function()
			if cond() then AnnounceEvent:FireServer(table.unpack(args, 1, args.n)) end
		end)
	end
	-- like Act.fire but ahead of kicks and unkicks (the captain-round remotes must not wait behind a kick backlog)
	Act.fireFirst = function(...)
		local args = table.pack(...)
		push(0, tostring(args[1]), function() AnnounceEvent:FireServer(table.unpack(args, 1, args.n)) end)
	end
	-- Announcements get their own lane: the game ignores a SendMessage fired while the last one is still fading,
	-- so each waits until the last has typed out and faded, and anything that piled up meanwhile is merged into one message.
	-- They also cover the screen, so they are held while a round is being played and go out once it is decided.
	local announceQ, announceBusyUntil, announceUrgent = {}, 0, false
	local function announceHeld() return RoundActive and not RoundDecided end
	local function announceDuration(text)
		local words = select(2, string.gsub(text, "%S+", ""))
		return words * ANNOUNCE_WORD_SECONDS + ANNOUNCE_FADE
	end
	-- true while an announcement is queued or still on screen
	local function announceBusy() return #announceQ > 0 or os.clock() < announceBusyUntil end
	Act.announce = function(msg)
		if #announceQ >= MAX_ANNOUNCE_QUEUE then table.remove(announceQ, 1) end
		table.insert(announceQ, string.sub(tostring(msg), 1, 200))
	end
	-- goes to the front of the announcements and holds the action queue until it is sent (still waits for the last one to fade)
	Act.announceNow = function(msg)
		if #announceQ >= MAX_ANNOUNCE_QUEUE then table.remove(announceQ) end
		table.insert(announceQ, 1, string.sub(tostring(msg), 1, 200))
		announceUrgent = true
	end
	local kickTimes = {}
	Act.kick = function(userId, minutes)
		local now = os.clock()
		for i = #kickTimes, 1, -1 do
			if now - kickTimes[i] > 60 then table.remove(kickTimes, i) end
		end
		if #kickTimes >= MAX_KICKS_PER_MINUTE then
			warn("[SCRIM] kick limit reached (" .. MAX_KICKS_PER_MINUTE .. "/min); skipping kick of " .. tostring(userId))
			return
		end
		table.insert(kickTimes, now)
		local args = buildKickArgs(userId, minutes)
		push(2, "kick " .. tostring(userId), function() AnnounceEvent:FireServer(table.unpack(args)) end)
	end
	Act.unkick = function(userId)
		local args = buildUnkickArgs(userId)
		push(2, "unkick " .. tostring(userId), function() AnnounceEvent:FireServer(table.unpack(args)) end, true)
	end
	Act.timeout = function(userId, on, urgent) -- urgent (benching a late joiner) jumps ahead of kicks and unkicks
		local args = buildTimeoutArgs(userId, on)
		push(urgent and 0 or 2, "timeout " .. tostring(userId) .. " " .. tostring(on), function() AnnounceEvent:FireServer(table.unpack(args)) end, not on)
	end
	Act.clear = function() table.clear(queue) table.clear(announceQ) announceUrgent = false end
	Act.drain = function()
		while Running and #queue > 0 do task.wait(0.25) end
		task.wait(jitteredGap())
	end
	-- wait only for the urgent remotes (they sit at the front); queued kicks keep going in the background
	Act.urgentPending = function() return queue[1] ~= nil and queue[1].prio == 0 end
	Act.drainUrgent = function()
		while Running and Act.urgentPending() do task.wait(0.25) end
		task.wait(jitteredGap())
	end

	task.spawn(function()
		while Running do
			local item = queue[1]
			if announceUrgent and not announceHeld() then item = nil end -- an urgent announcement goes out before the next remote
			-- a round only starts once the screen is clear, so no announcement is still showing when play begins
			if item and item.label == "StartMinigame" and announceBusy() then item = nil end
			if item then table.remove(queue, 1) end
			if not item then
				task.wait(0.1)
			else
				local pause = (item.removal and removalGap() or jitteredGap()) - (os.clock() - lastFire) -- removals get the longer gap
				if pause > 0 then task.wait(pause) end
				lastFire = os.clock()
				if DRY_RUN then
					print("[DRYRUN] " .. item.label)
				else
					local ok, err = pcall(item.fn)
					if not ok then warn("[SCRIM] action failed: " .. tostring(err)) end
				end
			end
		end
	end)

	task.spawn(function()
		while Running do
			if #announceQ == 0 or announceHeld() then
				task.wait(0.1)
			else
				local wait = math.max(announceBusyUntil + math.random() * ANNOUNCE_JITTER, lastFire + jitteredGap()) - os.clock()
				if wait > 0 then
					task.wait(wait)
				end
				if announceHeld() then continue end -- a round went live while we waited: keep everything queued until it ends
				-- merge everything that queued up while waiting (up to the 100 char limit), oldest first
				local text = table.remove(announceQ, 1) -- nil if Act.clear() ran while waiting
				announceUrgent = false
				if not text then continue end
				while announceQ[1] and #text + 3 + #announceQ[1] <= 100 do
					text = text .. " | " .. table.remove(announceQ, 1)
				end
				lastFire = os.clock()
				announceBusyUntil = lastFire + announceDuration(text)
				if DRY_RUN then
					print("[DRYRUN] announce: " .. text)
				else
					local ok, err = pcall(function() AnnounceEvent:FireServer("SendMessage", text) end)
					if not ok then warn("[SCRIM] announce failed: " .. tostring(err)) end
				end
			end
		end
	end)

	if not agentEnabled then return end

	--// ---- state ----
	local cfg = { min_players = 4, break_seconds = 180, verify_grace_seconds = 20, countdown_seconds = 30, captains = 1, pick_seconds = 30, mode = 0 }
	local phase = "IDLE"
	local sessionId = nil
	local countdownEnds, breakEnds = nil, nil
	local latched, locking = false, false
	local lobbyFloor = 0 -- after a failed lock, wait until MORE verified players are in before re-arming
	local pausedAt = 0       -- os.clock() when the admin paused; the frozen time is added to the timers on resume
	local forceStart = false -- /scrim console Force start: lock now with as few as 2 verified players
	local bigStart = false   -- /scrim console Big match: like Force start, random teams up to MAX_BIG_PER_TEAM a side, unrated
	local customTeams = {}   -- [userId] = team: set from the console in the lobby; holds the automatic start until Custom match or Clear teams
	local matchUnrated = false -- the current match is a big match: the bot logs it without touching ratings
	local announced = {}
	local timerResetBy = {} -- [userId] = true: in the lobby when this countdown began, or already reset its timer once
	local matchId = nil
	local liveSince, roundSeen, startRetries = 0, false, 0
	local matchRoster = {}   -- [userId] = { name, team } for the selected players of the current match
	local verifiedCache = {} -- [userId] = true | false (nil = not checked yet)
	local warnedAt = {}      -- [userId] = os.clock() when first seen unverified
	local kickPending = {}   -- [userId] = os.clock() when the kick was queued
	local kickAttempts = {}
	local timedOut = {}      -- [userId] = true: verified but not picked, sitting out until the match ends
	local releaseOnJoin = {} -- [userId] = true: released while absent; send the release again when they rejoin
	local lockedIds = {}     -- [userId] = true: everyone verified when the lobby locked; later joiners are kicked
	local leftIds = {}       -- [userId] = true: left after the lock and the rejoin grace ran out; cannot come back this match
	local anonymousOn = false -- anonymous mode is on during the captain rounds only
	local assignedAck = {}   -- [userId] = true: applied a Discord /scrim assign; sent until the bot stops resending it
	local adminAck = {}      -- [key] = true: ran a /scrim console command; sent until the bot stops resending it
	local function setAnonymous(on)
		if on == anonymousOn then return end
		anonymousOn = on
		Act.fireFirst(table.unpack(on and ANONYMOUS_ON_ARGS or ANONYMOUS_OFF_ARGS))
	end
	local kickedIds = {}     -- [userId] = true: kicked this session; unkicked one by one after the match
	local kickedUnverified = {} -- [userId] = os.clock(): kicked as unverified; unkicked as soon as the bot says they verified
	local absentSince, absentWarned = {}, {} -- [team] = os.clock() since nobody from the team was in the server / warned already
	local leftAt, voteKicked = {}, {}        -- [userId] = os.clock() when a locked player disconnected; they may rejoin any time until the match ends
	local nameById = {}      -- [userId] = name for the locked roster (survives the player leaving)
	local captains = {}      -- [team] = { id, name }
	local captainRound = 0
	local teamOrder = { [TEAM_A_NAME] = {}, [TEAM_B_NAME] = {} } -- ids in pick order, captain first
	local draftId, draftDone, draftApplied, draftPerSide = nil, false, {}, 0
	local pendingFinals = {} -- sent with every sync until the bot acks the match_id
	local pendingMoments = {} -- ace / clutch / flawless announcements, sent with every sync until the bot acks their key
	local syncNow, syncCount = false, 0
	local syncStatus = "starting"
	local rng = Random.new()

	local function say(msg) Act.announce(msg) end

	local function setPhase(p)
		phase = p
		Agent.active = (p ~= "IDLE")
		syncNow = true
		print("[SCRIM] phase -> " .. p)
	end

	local function presentPlayers()
		local list = {}
		for _, p in ipairs(Players:GetPlayers()) do
			if p ~= LocalPlayer and not IGNORED_USER_IDS[p.UserId] then table.insert(list, p) end
		end
		return list
	end

	local function kick(p, unverified, minutes)
		local id = p.UserId
		if kickPending[id] then return end
		kickedIds[id] = true
		if unverified then kickedUnverified[id] = os.clock() end
		kickAttempts[id] = (kickAttempts[id] or 0) + 1
		print(string.format("[SCRIM] kick %s (id=%d) attempt %d", p.Name, id, kickAttempts[id]))
		if kickAttempts[id] > 3 then warn("[SCRIM] " .. p.Name .. " is still in the server after 3 kicks; the KickPlayer remote may be failing") end
		kickPending[id] = os.clock()
		Act.kick(id, minutes)
	end

	local function lobbyRoster()
		local list = {}
		for _, p in ipairs(presentPlayers()) do
			if verifiedCache[p.UserId] == true and not kickPending[p.UserId] then table.insert(list, p) end
		end
		return list
	end

	local function copyStats(stats)
		local out = {}
		for name, s in pairs(stats) do out[name] = copyStat(s) end
		return out
	end

	local function releaseTimeout(id, urgent)
		Act.timeout(id, false, urgent)
		timedOut[id] = nil
		if not Players:GetPlayerByUserId(id) then releaseOnJoin[id] = true end
	end

	local function releaseAllTimeouts()
		local ids = {}
		for id in pairs(timedOut) do table.insert(ids, id) end
		for _, id in ipairs(ids) do releaseTimeout(id) end
	end

	local function resetAll()
		Act.clear()
		setAnonymous(false) -- never leave anonymous mode on
		releaseAllTimeouts() -- timeouts persist in the game, so never leave anyone stuck
		SeriesActive, Paused = false, false
		matchRoster, PlayerTeams, MatchStats = {}, {}, {}
		for _, p in ipairs(Players:GetPlayers()) do updatePlayerNametag(p) end
		countdownEnds, breakEnds, matchId = nil, nil, nil
		latched, locking, lobbyFloor, forceStart, bigStart, matchUnrated = false, false, 0, false, false, false
		customTeams = {}
		kickPending, kickAttempts, warnedAt = {}, {}, {}
		lockedIds, leftIds, nameById, captains, captainRound = {}, {}, {}, {}, 0
		leftAt, voteKicked = {}, {}
		teamOrder = { [TEAM_A_NAME] = {}, [TEAM_B_NAME] = {} }
		draftId, draftDone, draftApplied, draftPerSide = nil, false, {}, 0
	end

	--// ---- verification + joins ----
	local function onVerifiedJoin(id)
		if phase ~= "COUNTDOWN" or latched or locking then return end
		if timerResetBy[id] then
			-- already in the lobby (or already reset the timer) this countdown: leaving and rejoining to stall it gets them kicked
			local p = Players:GetPlayerByUserId(id)
			if p then kick(p, false, KICK_DURATION) end
			return
		end
		timerResetBy[id] = true
		countdownEnds = os.clock() + cfg.countdown_seconds
		announced = {}
		say(string.format("Timer reset: %ds (%d/%d)", cfg.countdown_seconds, #lobbyRoster(), LOBBY_CAP))
	end

	local function applyVerified(reportedIds, verifiedList)
		local set = {}
		for _, id in ipairs(verifiedList or {}) do set[id] = true end
		for _, id in ipairs(reportedIds) do
			local was = verifiedCache[id]
			verifiedCache[id] = set[id] == true
			if verifiedCache[id] and was ~= true then onVerifiedJoin(id) end
		end
	end

	local nextKickAt = 0 -- pacing for kicks

	local function enforce()
		local now = os.clock()
		local kickedThisTick, timeoutedThisTick = false, false
		for id, t in pairs(kickedUnverified) do
			if now - t > KICK_DURATION * 60 + 60 then kickedUnverified[id] = nil end -- the lockout is over; they rejoin on their own
		end
		for id, t in pairs(kickPending) do
			-- the remote may have silently failed; retry after 10s, then back off to once a minute (never give up)
			local wait = (kickAttempts[id] or 0) >= 3 and 60 or 10
			if now - t > wait and Players:GetPlayerByUserId(id) then kickPending[id] = nil end
		end
		for _, p in ipairs(presentPlayers()) do
			local id = p.UserId
			if kickPending[id] then continue end
			if phase == "CAPTAINS" or phase == "DRAFT" or phase == "LIVE" or phase == "POSTGAME" then
				-- joined after the lock, or left and came back
				if not lockedIds[id] or leftIds[id] then
					-- bench them the moment they appear so the game cannot put them in a round before their kick goes out.
					-- One new timeout per tick; the queue then spaces every remote by the normal gap, so a crowd of joiners is never fired at once.
					if not timedOut[id] and not timeoutedThisTick then
						timedOut[id] = true -- released with everyone else when the match ends
						Act.timeout(id, true, true)
						timeoutedThisTick = true
					end
					-- then the kick, paced like the unverified kicks (no longer lockout)
					if not kickedThisTick and now >= nextKickAt then
						kick(p)
						kickedThisTick = true
						nextKickAt = now + KICK_SPACING + math.random() * KICK_JITTER
					end
				end
			elseif verifiedCache[id] == true then
				warnedAt[id] = nil
			elseif verifiedCache[id] == false then
				if not warnedAt[id] then
					warnedAt[id] = now
					say(p.Name .. " is unverified")
				end
				-- per-player grace jitter (stable per user id) so simultaneous joiners don't expire together
				local grace = cfg.verify_grace_seconds + (id % 1000) / 1000 * KICK_JITTER
				if now - warnedAt[id] >= grace and not kickedThisTick and now >= nextKickAt then
					kick(p, true)
					kickedThisTick = true
					nextKickAt = now + KICK_SPACING + math.random() * KICK_JITTER
				end
			end
		end
	end

	-- Bench every present player who is not on the roster and is not timed out yet. Returns how many were queued.
	local function benchLateJoiners()
		local queued = 0
		for _, p in ipairs(presentPlayers()) do
			local id = p.UserId
			if (not lockedIds[id] or leftIds[id]) and not timedOut[id] then
				timedOut[id] = true
				Act.timeout(id, true, true)
				queued += 1
			end
		end
		return queued
	end

	-- Called right before a bomb round starts (the bomb detonates after 10s, so a stranger in the round ruins it): keep
	-- benching latecomers and let the queue drain, repeating until nobody new shows up (capped so a crowd cannot stall the match).
	local function clearTheField()
		local started = os.clock()
		while Running and os.clock() - started < 30 do
			local queued = benchLateJoiners()
			if queued == 0 and not Act.urgentPending() then break end -- nothing new and nothing urgent pending: start right away
			Act.drainUrgent()
		end
	end
	Agent.beforeRound = clearTheField

	--// ---- sync with the bot ----
	local function post(body)
		local ok, res = pcall(httpRequest, {
			Url = API_URL .. "/api/scrim/sync",
			Method = "POST",
			Headers = { ["Content-Type"] = "application/json", ["X-Roblox-Secret"] = API_SECRET },
			Body = HttpService:JSONEncode(body),
		})
		if not ok or not res then return nil, tostring(res) end
		if res.StatusCode == 401 then return nil, "HTTP 401: API_SECRET does not match ROBLOX_WEBHOOK_SECRET (Mongo config > api_keys, or clip-hosting env)" end
		if res.StatusCode == 404 then return nil, "HTTP 404: wrong API_URL, or clip-hosting has not been updated with /api/scrim/sync yet" end
		if res.StatusCode == 503 then return nil, "HTTP 503: clip-hosting cannot reach the database" end
		if res.StatusCode == 504 or res.StatusCode == 502 then return nil, "HTTP " .. tostring(res.StatusCode) .. ": the Discord bot did not answer (is it online with the scrim update deployed?)" end
		if res.StatusCode ~= 200 then return nil, "HTTP " .. tostring(res.StatusCode) end
		local okJson, data = pcall(HttpService.JSONDecode, HttpService, res.Body)
		if not okJson then return nil, "bad JSON" end
		return data
	end

	local function secondsLeft(untilClock)
		return math.max(0, math.ceil(untilClock - os.clock()))
	end

	local function buildPayload()
		local players, ids = {}, {}
		for _, p in ipairs(presentPlayers()) do
			if not kickPending[p.UserId] then
				local info = matchRoster[p.UserId]
				table.insert(players, { id = p.UserId, name = p.Name, display = p.DisplayName, team = info and info.team or customTeams[p.UserId] })
				table.insert(ids, p.UserId)
			end
		end
		local stats = {}
		if phase == "LIVE" then
			for id, info in pairs(matchRoster) do
				local row = copyStat(MatchStats[info.name])
				row.UserId, row.Name, row.Team = id, info.name, info.team
				table.insert(stats, row)
			end
		end
		local timed = {}
		for id in pairs(timedOut) do table.insert(timed, id) end
		for id in pairs(releaseOnJoin) do
			if not timedOut[id] then table.insert(timed, id) end
		end
		local rejoinList = {} -- roster players who disconnected during the live match and may still rejoin (the bot shows a Rejoin button)
		if phase == "LIVE" then
			for id in pairs(leftAt) do
				if matchRoster[id] then table.insert(rejoinList, id) end -- not someone who left during the draft and was never picked
			end
		end
		local assignedList = {}
		for id in pairs(assignedAck) do table.insert(assignedList, id) end
		local adminList = {}
		for key in pairs(adminAck) do table.insert(adminList, key) end
		local votekickedList = {}
		for id in pairs(voteKicked) do table.insert(votekickedList, id) end
		local kickedList = {}
		for id in pairs(kickedUnverified) do table.insert(kickedList, id) end
		local caps = nil
		if next(captains) ~= nil and (phase == "CAPTAINS" or phase == "DRAFT") then
			caps = {}
			for team, c in pairs(captains) do caps[team] = { id = c.id, name = c.name } end
		end
		local draftInfo = nil
		if phase == "DRAFT" and draftId then
			local pool = {}
			for _, p in ipairs(presentPlayers()) do
				local id = p.UserId
				if lockedIds[id] and not matchRoster[id] and not kickPending[id] and not leftIds[id] then
					table.insert(pool, { id = id, name = p.Name, display = p.DisplayName })
				end
			end
			draftInfo = { id = draftId, per_side = draftPerSide, pool = pool }
		end
		local phaseEnds = nil
		if phase == "COUNTDOWN" and countdownEnds then phaseEnds = os.time() + secondsLeft(countdownEnds) end
		if phase == "BREAK" and breakEnds then phaseEnds = os.time() + secondsLeft(breakEnds) end
		if Paused then phaseEnds = nil end -- the timer is frozen
		return {
			session_id = sessionId,
			phase = phase,
			phase_ends = phaseEnds,
			match_id = matchId,
			unrated = matchUnrated,
			paused = Paused,
			score = { [TEAM_A_NAME] = SeriesScore[TEAM_A_NAME], [TEAM_B_NAME] = SeriesScore[TEAM_B_NAME] },
			players = players,
			stats = stats,
			timed_out = timed,
			captains = caps,
			captain_round = (phase == "CAPTAINS") and captainRound or nil,
			draft = draftInfo,
			finals = pendingFinals,
			kicked_ids = kickedList,
			votekicked_ids = votekickedList,
			assigned_ids = assignedList,
			admin_acked = adminList,
			moments = pendingMoments,
			rejoin_ids = rejoinList,
		}, ids
	end

	local function nameOf(id) return nameById[id] or tostring(id) end

	local function assignPick(id, team, suffix)
		local name = nameOf(id)
		matchRoster[id] = { name = name, team = team }
		PlayerTeams[name] = team
		table.insert(teamOrder[team], id)
		releaseTimeout(id) -- everyone but the captains is timed out during the draft; picked players are let back in
		local pl = Players:GetPlayerByUserId(id)
		if pl then updatePlayerNametag(pl) end
		say(string.format("%s picks %s%s", team, name, suffix or ""))
	end

	local function setCaptain(team, id)
		local name = nameOf(id)
		captains[team] = { id = id, name = name }
		matchRoster[id] = { name = name, team = team }
		PlayerTeams[name] = team
		if teamOrder[team][1] then teamOrder[team][1] = id else table.insert(teamOrder[team], 1, id) end
		local pl = Players:GetPlayerByUserId(id)
		if pl then updatePlayerNametag(pl) end
	end

	-- the bot owns who has been picked; apply any pick we have not applied yet (idempotent by seq)
	local function applyDraft(d)
		if phase ~= "DRAFT" or d.id ~= draftId then return end
		for _, pick in ipairs(d.picks or {}) do
			if not draftApplied[pick.seq] then
				draftApplied[pick.seq] = true
				if lockedIds[pick.id] and not matchRoster[pick.id] then
					assignPick(pick.id, pick.team, pick.auto and " (auto)" or "")
				else
					warn(string.format("[SCRIM] draft pick #%d (%s -> %s) was NOT applied: locked=%s alreadyOnTeam=%s", pick.seq, tostring(pick.name or pick.id), tostring(pick.team), tostring(lockedIds[pick.id]), tostring(matchRoster[pick.id] ~= nil)))
				end
			end
		end
		if d.done then draftDone = true end
	end

	-- a captain who leaves is replaced by a random remaining player; returns false if nobody is left
	local function ensureCaptains()
		for _, team in ipairs({ TEAM_A_NAME, TEAM_B_NAME }) do
			local c = captains[team]
			if c and not Players:GetPlayerByUserId(c.id) then
				local pool = {}
				for _, p in ipairs(presentPlayers()) do
					local id = p.UserId
					if lockedIds[id] and not leftIds[id] and not matchRoster[id] and not kickPending[id] then table.insert(pool, id) end
				end
				if #pool == 0 then return false end
				matchRoster[c.id] = nil
				PlayerTeams[c.name] = nil
				local replacement = pool[rng:NextInteger(1, #pool)]
				setCaptain(team, replacement)
				releaseTimeout(replacement)
				say(string.format("New %s captain: %s", team, nameOf(replacement)))
			end
		end
		return true
	end

	-- used only if the bot stops answering during the draft
	local function localAutoFill(perSide)
		local pool = {}
		for _, p in ipairs(presentPlayers()) do
			local id = p.UserId
			if lockedIds[id] and not matchRoster[id] and not kickPending[id] and not leftIds[id] then table.insert(pool, id) end
		end
		while #pool > 0 and (#teamOrder[TEAM_A_NAME] < perSide or #teamOrder[TEAM_B_NAME] < perSide) do
			local id = table.remove(pool, rng:NextInteger(1, #pool))
			local team = (#teamOrder[TEAM_A_NAME] <= #teamOrder[TEAM_B_NAME]) and TEAM_A_NAME or TEAM_B_NAME
			assignPick(id, team, " (auto)")
		end
	end

	-- the script was restarted in the middle of a live match: pick the match up from the bot's last snapshot
	-- (score, teams, stats, who is benched) instead of starting from an empty lobby
	local function resumeMatch(r)
		resetAll()
		matchId = r.match_id
		matchUnrated = r.unrated == true
		SeriesScore = { [TEAM_A_NAME] = (r.score or {})[TEAM_A_NAME] or 0, [TEAM_B_NAME] = (r.score or {})[TEAM_B_NAME] or 0 }
		MatchStats = {}
		local perSide = 0
		for _, s in ipairs(r.stats or {}) do
			local id = s.UserId
			if type(id) == "number" and id > 0 and (s.Team == TEAM_A_NAME or s.Team == TEAM_B_NAME) then
				matchRoster[id] = { name = s.Name, team = s.Team }
				PlayerTeams[s.Name] = s.Team
				nameById[id], lockedIds[id] = s.Name, true
				table.insert(teamOrder[s.Team], id)
				MatchStats[s.Name] = copyStat(s)
				if s.Team == TEAM_A_NAME then perSide += 1 end
				local pl = Players:GetPlayerByUserId(id)
				if pl then updatePlayerNametag(pl) else leftAt[id] = os.clock() end -- not in the server right now: may rejoin
			end
		end
		for _, id in ipairs(r.timed_out or {}) do timedOut[id] = true end
		SeriesPerSide = math.max(perSide, 1)
		SeriesActive = true
		timelineReset(true) -- rounds before the restart were not recorded
		liveSince, roundSeen, startRetries = os.clock(), false, 0
		absentSince, absentWarned = {}, {}
		setPhase("LIVE")
		say(string.format("Resumed: %s %d - %d %s", TEAM_A_NAME, SeriesScore[TEAM_A_NAME], SeriesScore[TEAM_B_NAME], TEAM_B_NAME))
		task.spawn(startNextRound) -- the round that was running is not tracked: stop it and start a fresh one at the same score
	end

	local function applySession(info, staleTimeouts, resume)
		if not info then return end
		if info.active then
			if sessionId ~= info.id and phase == "IDLE" and type(resume) == "table" and resume.match_id then
				sessionId = info.id
				resumeMatch(resume)
			elseif sessionId ~= info.id then
				if phase == "LIVE" or phase == "CAPTAINS" then stopRound() end
				resetAll()
				sessionId = info.id
				for _, id in ipairs(staleTimeouts or {}) do releaseTimeout(id) end
				setPhase("LOBBY")
			end
		elseif phase ~= "IDLE" then
			-- /scrim stop: drop everything and wait for the next /scrim start
			if phase == "LIVE" or phase == "CAPTAINS" then stopRound() end
			resetAll()
			sessionId = nil
			setPhase("IDLE")
		end
	end

	-- a team voted a teammate out in Discord: kick them (long lockout), drop them from the round counts, and never let them back in this match
	local function applyVotekick(id, reason)
		if voteKicked[id] or phase ~= "LIVE" then return end
		local info = matchRoster[id]
		if not info then return end
		voteKicked[id] = true
		leftAt[id] = nil
		timelineEvent("votekick", id)
		leftIds[id] = true -- the enforce loop benches and kicks them again if they return before the match ends
		PlayerTeams[info.name] = nil
		say(info.name .. " " .. (reason or "vote-kicked"))
		local p = Players:GetPlayerByUserId(id)
		if p then kick(p, false, VOTEKICK_DURATION) end
		syncNow = true
	end

	local function applyResponse(data, reportedIds)
		for _, matchIdAcked in ipairs(data.acked or {}) do
			for i = #pendingFinals, 1, -1 do
				if pendingFinals[i].match_id == matchIdAcked then table.remove(pendingFinals, i) end
			end
		end
		for _, key in ipairs(data.acked_moments or {}) do
			for i = #pendingMoments, 1, -1 do
				if pendingMoments[i].key == key then table.remove(pendingMoments, i) end
			end
		end
		if type(data.config) == "table" then
			for key in pairs(cfg) do
				if type(data.config[key]) == "number" then cfg[key] = data.config[key] end
			end
			if cfg.mode ~= GameMode then
				GameMode = cfg.mode
				if phase ~= "IDLE" then say("Mode: " .. (GameMode == 1 and "Shrink" or "Normal") .. " (from the next round)") end
			end
		end
		applyVerified(reportedIds, data.verified)
		-- a player kicked as unverified has since verified in Discord: lift their kick timer (only between matches)
		if phase == "LOBBY" or phase == "COUNTDOWN" or phase == "BREAK" then
			for _, id in ipairs(data.verified_kicked or {}) do
				if kickedUnverified[id] then
					kickedUnverified[id] = nil
					kickedIds[id] = nil
					Act.unkick(id)
					print("[SCRIM] " .. (nameById[id] or tostring(id)) .. " verified after a kick; unkicking")
				end
			end
		end
		applySession(data.session, data.timed_out, data.resume)
		if type(data.draft) == "table" then applyDraft(data.draft) end
		for _, id in ipairs(data.votekicks or {}) do applyVotekick(id) end
		local resent = {}
		for _, a in ipairs(data.assigns or {}) do
			resent[a.id] = true
			if not assignedAck[a.id] then
				Agent.assign(a.id, a.team)
				assignedAck[a.id] = true -- acknowledged even if it could not be applied, so the bot does not resend it forever
			end
		end
		for id in pairs(assignedAck) do
			if not resent[id] then assignedAck[id] = nil end
		end
		local resentKeys = {}
		for _, cmd in ipairs(data.admin or {}) do
			if type(cmd) == "table" and cmd.key then
				resentKeys[cmd.key] = true
				if not adminAck[cmd.key] then
					adminAck[cmd.key] = true -- acknowledged even if it failed, so it never runs twice
					local ok, err = pcall(Agent.admin, cmd)
					if not ok then warn("[SCRIM] admin " .. tostring(cmd.action) .. " failed: " .. tostring(err)) end
				end
			end
		end
		for key in pairs(adminAck) do
			if not resentKeys[key] then adminAck[key] = nil end
		end
	end

	task.spawn(function()
		local failures = 0
		while Running do
			local payload, ids = buildPayload()
			local data, err = post(payload)
			syncCount += 1
			if data then
				failures = 0
				syncStatus = "ok"
				local ok, applyErr = pcall(applyResponse, data, ids)
				if not ok then warn("[SCRIM] bad sync response: " .. tostring(applyErr)) end
			else
				failures += 1
				syncStatus = "FAILED: " .. tostring(err)
				warn("[SCRIM] sync failed (" .. tostring(err) .. ")")
			end
			local pause = failures > 0 and math.min(2 * 2 ^ failures, 15) or 2
			local started = os.clock()
			while Running and os.clock() - started < pause and not syncNow do task.wait(0.2) end
			syncNow = false
		end
	end)

	function Agent.forceSync()
		local target = syncCount + 2 -- the in-flight request may predate the latest joins
		syncNow = true
		local started = os.clock()
		while Running and syncCount < target and os.clock() - started < 6 do task.wait(0.1) end
	end

	function Agent.shutdown()
		local payload = buildPayload()
		payload.phase = "OFFLINE"
		-- the queue worker stops with Running, so release timeouts directly (best effort) or they stay timed out
		Act.clear()
		for id in pairs(timedOut) do
			if not DRY_RUN then
				pcall(function() AnnounceEvent:FireServer(table.unpack(buildTimeoutArgs(id, false))) end)
			end
			task.wait(removalGap())
		end
		pcall(post, payload)
	end

	--// ---- match end ----
	-- everyone kicked this session gets their kick timer removed, one by one, so they can rejoin for the next lobby
	local function unkickAll()
		local waited = os.clock()
		while Running and os.clock() - waited < UNKICK_DELAY_SECONDS do task.wait(1) end
		if not Running then return end
		local ids = {}
		for id in pairs(kickedIds) do table.insert(ids, id) end
		kickedIds, kickedUnverified = {}, {}
		for _, id in ipairs(ids) do Act.unkick(id) end
		print("[SCRIM] unkicking " .. #ids .. " player(s) from this scrim")
	end

	local function conclude(winnerTeam, score, stats, forfeit)
		if phase ~= "LIVE" then return end
		local a, b = score[TEAM_A_NAME], score[TEAM_B_NAME]
		local final = {
			match_id = matchId,
			score = { [TEAM_A_NAME] = a, [TEAM_B_NAME] = b },
			winner = winnerTeam,
			forfeit = forfeit,
			unrated = matchUnrated,
			rounds = a + b,
			stats = {},
			timeline = timelineExport(),
		}
		for id, info in pairs(matchRoster) do
			local row = copyStat(stats[info.name])
			row.UserId, row.Name, row.Team = id, info.name, info.team
			row.Left, row.Kicked = leftAt[id] ~= nil and not voteKicked[id], voteKicked[id] == true
			table.insert(final.stats, row)
		end
		table.insert(pendingFinals, final)
		while #pendingFinals > MAX_PENDING_FINALS do table.remove(pendingFinals, 1) end
		Paused = false
		setPhase("POSTGAME")

		task.spawn(function()
			say(string.format("%s wins %d-%d%s!", winnerTeam, math.max(a, b), math.min(a, b), forfeit and " (forfeit)" or ""))
			task.wait(3)
			releaseAllTimeouts() -- timeouts persist across rejoin, so clear them before everyone is kicked
			for _, p in ipairs(presentPlayers()) do kick(p) end -- everyone leaves, one per ACTION_GAP
			Act.drain()
			if phase ~= "POSTGAME" then return end -- /scrim stop happened meanwhile
			matchRoster, PlayerTeams = {}, {}
			for _, p in ipairs(Players:GetPlayers()) do updatePlayerNametag(p) end
			matchId, matchUnrated = nil, false
			kickAttempts = {}
			lockedIds, leftIds, nameById, captains = {}, {}, {}, {}
			leftAt, voteKicked = {}, {}
			breakEnds = os.clock() + cfg.break_seconds
			setPhase("BREAK")
			task.spawn(unkickAll)
			while Running and phase == "BREAK" and (Paused or os.clock() < breakEnds) do task.wait(1) end
			if Running and phase == "BREAK" then
				countdownEnds, latched = nil, false
				setPhase("LOBBY")
			end
		end)
	end

	-- called from checkRoundWinner() when Rules.isMatchOver; copies are taken before the script resets its own state
	-- an ace, clutch or flawless round: the bot announces it in Discord (name is nil for a flawless round, which belongs to the team)
	function Agent.onMoment(kind, name, team, enemies)
		local playerId = 0
		if name then
			for id, info in pairs(matchRoster) do
				if info.name == name then playerId = id break end
			end
			if playerId == 0 then return end
		end
		local a, b = SeriesScore[TEAM_A_NAME], SeriesScore[TEAM_B_NAME]
		table.insert(pendingMoments, {
			key = tostring(matchId) .. ":" .. (a + b) .. ":" .. kind .. ":" .. playerId,
			kind = kind, id = playerId, name = name or "", team = team, enemies = enemies or 0,
			score = { [TEAM_A_NAME] = a, [TEAM_B_NAME] = b },
		})
		while #pendingMoments > 10 do table.remove(pendingMoments, 1) end
		syncNow = true
	end

	function Agent.onSeriesOver(winnerTeam, score, stats)
		conclude(winnerTeam, { [TEAM_A_NAME] = score[TEAM_A_NAME], [TEAM_B_NAME] = score[TEAM_B_NAME] }, copyStats(stats), false)
	end

	-- manual fix: put a player on a team during the draft or a live match. From the executor: ScrimAssign("Username", "Black");
	-- from Discord: /scrim assign (the bot sends the id and team with the next sync)
	function Agent.assign(name, team)
		local p = type(name) == "number" and Players:GetPlayerByUserId(name) or Players:FindFirstChild(tostring(name))
		if not p or not p:IsA("Player") then return warn("[SCRIM] assign: no player named " .. tostring(name)) end
		if team ~= TEAM_A_NAME and team ~= TEAM_B_NAME then return warn("[SCRIM] assign: team must be \"" .. TEAM_A_NAME .. "\" or \"" .. TEAM_B_NAME .. "\"") end
		local id = p.UserId
		if phase == "LOBBY" or phase == "COUNTDOWN" then
			-- a custom match is being set up: remember the team and stop the automatic start
			if locking then return end
			if verifiedCache[id] ~= true then return warn("[SCRIM] assign: " .. p.Name .. " is not verified") end
			customTeams[id] = team
			if phase == "COUNTDOWN" then
				countdownEnds, latched = nil, false
				setPhase("LOBBY")
			end
			say(string.format("%s set to %s (custom match)", p.Name, team))
			syncNow = true
			return
		end
		if phase ~= "LIVE" and phase ~= "DRAFT" then return warn("[SCRIM] assign: only before the match, during the draft or in a live match") end
		for _, t in ipairs({ TEAM_A_NAME, TEAM_B_NAME }) do
			for i = #teamOrder[t], 1, -1 do
				if teamOrder[t][i] == id then table.remove(teamOrder[t], i) end
			end
		end
		table.insert(teamOrder[team], id)
		lockedIds[id], leftIds[id], nameById[id] = true, nil, p.Name
		matchRoster[id] = { name = p.Name, team = team }
		PlayerTeams[p.Name] = team
		if phase == "LIVE" then timelineEvent("assign", id, team) end
		ensureStats(p.Name)
		updatePlayerNametag(p)
		releaseTimeout(id, true)
		say(string.format("%s joined %s", p.Name, team))
		syncNow = true
	end
	;(getgenv and getgenv() or _G).ScrimAssign = Agent.assign

	--// ---- lobby + live loops ----
	local function startCountdown(n)
		setPhase("COUNTDOWN")
		countdownEnds = os.clock() + cfg.countdown_seconds
		announced = {}
		timerResetBy = {}
		for _, p in ipairs(lobbyRoster()) do timerResetBy[p.UserId] = true end
		latched = n >= LOBBY_CAP
		say(string.format("Starting in %ds (%d/%d)", cfg.countdown_seconds, n, LOBBY_CAP))
	end

	local function newMatchId()
		local ok, guid = pcall(function() return HttpService:GenerateGUID(false) end)
		return ok and guid or (tostring(os.time()) .. tostring(rng:NextInteger(1000, 999999)))
	end

	local function beginLive()
		clearTheField() -- nobody outside the roster may be in the first round
		matchId = newMatchId()
		SeriesScore = { [TEAM_A_NAME] = 0, [TEAM_B_NAME] = 0 }
		SeriesActive = true
		MatchStats = {}
		timelineReset()
		for _, info in pairs(matchRoster) do ensureStats(info.name) end
		liveSince, roundSeen, startRetries = os.clock(), false, 0
		absentSince, absentWarned = {}, {}
		locking = false
		koEvents = {}
		local sizes = { [TEAM_A_NAME] = 0, [TEAM_B_NAME] = 0 }
		for _, info in pairs(matchRoster) do sizes[info.team] += 1 end
		SeriesPerSide = math.max(sizes[TEAM_A_NAME], sizes[TEAM_B_NAME], 1) -- uneven custom teams play to the bigger side's target
		setPhase("LIVE")
		say(TEAM_A_NAME .. " 0 - 0 " .. TEAM_B_NAME .. " | first to " .. Rules.pointsToWin(SeriesPerSide))
		Act.fire("ResetKOs") -- clear everyone's KOs (the captain rounds added some) before the first real round
		Act.fire(roundArgs())
	end

	local function abortToLobby(reason)
		say(reason)
		Act.clear()
		setAnonymous(false)
		releaseAllTimeouts()
		matchRoster, PlayerTeams, captains = {}, {}, {}
		for _, p in ipairs(Players:GetPlayers()) do updatePlayerNametag(p) end
		lockedIds, leftIds, nameById, captainRound = {}, {}, {}, 0
		leftAt, voteKicked = {}, {}
		teamOrder = { [TEAM_A_NAME] = {}, [TEAM_B_NAME] = {} }
		draftId, draftDone, draftApplied = nil, false, {}
		countdownEnds, latched, locking, lobbyFloor, forceStart, bigStart, matchUnrated = nil, false, false, 0, false, false, false
		Paused, customTeams = false, {}
		setPhase("LOBBY")
	end

	--// ---- random teams (captains disabled) ----
	local function runRandomTeams(roster, perSide)
		local ids, byId = {}, {}
		for _, p in ipairs(roster) do
			table.insert(ids, p.UserId)
			byId[p.UserId] = p
		end
		local blackIds, whiteIds = Rules.pickTeams(ids, perSide, rng)

		matchRoster, PlayerTeams = {}, {}
		local function assign(list, team)
			local names = {}
			for _, id in ipairs(list) do
				local p = byId[id]
				matchRoster[id] = { name = p.Name, team = team }
				PlayerTeams[p.Name] = team
				table.insert(names, p.Name)
				updatePlayerNametag(p)
			end
			return names
		end
		local blackNames = assign(blackIds, TEAM_A_NAME)
		local whiteNames = assign(whiteIds, TEAM_B_NAME)

		local notPicked = {}
		for _, p in ipairs(presentPlayers()) do
			local id = p.UserId
			if not matchRoster[id] and lockedIds[id] then
				timedOut[id] = true -- verified but not picked: sits out, released after the match
				Act.timeout(id, true)
				table.insert(notPicked, p.Name)
			end
		end
		say(TEAM_A_NAME .. ": " .. table.concat(blackNames, ", "))
		say(TEAM_B_NAME .. ": " .. table.concat(whiteNames, ", "))
		if #notPicked > 0 then say("Sitting out: " .. table.concat(notPicked, ", ")) end
		Act.drain() -- timeouts and kicks finish (one per ACTION_GAP) before the game starts
		beginLive()
	end

	--// ---- captain rounds: two free-for-all bomb rounds, the last player standing in each is a captain ----
	local function idForName(name)
		local p = Players:FindFirstChild(name)
		if p and p:IsA("Player") then return p.UserId end
		for id, n in pairs(nameById) do
			if n == name then return id end
		end
		return nil
	end

	local function aliveNames()
		local alive = {}
		for name in pairs(roundParticipants) do
			if not deadPlayersThisRound[name] and Players:FindFirstChild(name) then table.insert(alive, name) end
		end
		return alive
	end

	local function captainPhaseActive()
		return Running and phase == "CAPTAINS"
	end

	-- returns "win", userId | "tie", { userIds } | "norun"
	local function awaitCaptainResult()
		local started = os.clock()
		while captainPhaseActive() and not RoundActive and os.clock() - started < 60 do task.wait(0.5) end
		if not captainPhaseActive() or not RoundActive then return "norun" end
		local activeSince = os.clock()
		local lastStanding = nil -- the moment one player is left alive they have won; the game's reset afterwards can't undo that
		while captainPhaseActive() do
			if os.clock() - activeSince > 3 then
				local now = aliveNames()
				if #now == 1 then lastStanding = now[1] end
			end
			-- ignore the first seconds: the participant list is still filling in
			if os.clock() - activeSince > 3 and (#aliveNames() <= 1 or not RoundActive) then
				task.wait(1.5) -- a bomb can kill several players at once; let those deaths register
				local alive = aliveNames()
				if #alive == 1 then return "win", idForName(alive[1]) end
				if #alive == 0 and lastStanding and idForName(lastStanding) then return "win", idForName(lastStanding) end
				if #alive == 0 then
					local latest = 0
					for _, t in pairs(deathClock) do latest = math.max(latest, t) end
					local tied = {}
					for name, t in pairs(deathClock) do
						local id = roundParticipants[name] and latest - t <= 0.8 and idForName(name) or nil
						if id then table.insert(tied, id) end
					end
					return "tie", tied
				end
				return "norun"
			end
			task.wait(0.5)
		end
		return "norun"
	end

	local function endCaptainRound()
		stopRound()
		local started = os.clock()
		while Running and RoundActive and os.clock() - started < 10 do task.wait(0.5) end
		task.wait(2)
	end

	-- one captain round; ties replay twice, then a random pick among the tied. Returns a userId or nil (aborted).
	-- onlyIds limits the players for the captain duel (the random pick then only chooses between the two).
	local function playCaptainRound(excludeId, onlyIds)
		local eligible, pool = {}, lockedIds
		if onlyIds then
			pool = {}
			for _, id in ipairs(onlyIds) do pool[id] = true end
		end
		for id in pairs(pool) do
			if id ~= excludeId and not leftIds[id] and Players:GetPlayerByUserId(id) then table.insert(eligible, id) end
		end
		local tied = nil
		for attempt = 1, 3 do
			if not captainPhaseActive() then return nil end
			clearTheField()
			if not captainPhaseActive() then return nil end
			if #eligible > 14 then
				Act.fireFirst("StartMinigame", "Timebomb", "Shrink", "Multi") -- big lobby: faster multi-bomb shrink round
			else
				Act.fireFirst("StartMinigame", "Timebomb", "LeagueStandard", "Normal")
			end
			local kind, value = awaitCaptainResult()
			endCaptainRound()
			if not captainPhaseActive() then return nil end
			if kind == "win" and value then return value end
			if kind == "tie" then
				tied = value
			elseif attempt < 3 then
				say("Retrying round")
			end
		end
		local choices = (tied and #tied > 0) and tied or eligible
		if #choices == 0 then return nil end
		local chosen = choices[rng:NextInteger(1, #choices)]
		say("Random captain: " .. nameOf(chosen))
		return chosen
	end

	local function finalizeDraft()
		local function presentMembers(team)
			local list = {}
			for _, id in ipairs(teamOrder[team]) do
				-- someone who disconnected may rejoin (leftAt), so they still count: a brief disconnect must not bench their teammate
				if Players:GetPlayerByUserId(id) or leftAt[id] then table.insert(list, id) end
			end
			return list
		end
		-- teams must be even: the latest picks of the bigger side sit out
		while true do
			local a, b = presentMembers(TEAM_A_NAME), presentMembers(TEAM_B_NAME)
			if #a == #b then break end
			local team = (#a > #b) and TEAM_A_NAME or TEAM_B_NAME
			local list = (#a > #b) and a or b
			if #list <= 1 then break end -- never bench a captain
			local id = list[#list]
			local name = nameOf(id)
			matchRoster[id] = nil
			PlayerTeams[name] = nil
			for i = #teamOrder[team], 1, -1 do
				if teamOrder[team][i] == id then
					table.remove(teamOrder[team], i)
					break
				end
			end
			timedOut[id] = true
			Act.timeout(id, true)
			warn(string.format("[SCRIM] %s benched to even the teams (%d vs %d present)", name, #a, #b))
			say(name .. " sits out")
		end

		local benched = {}
		for id in pairs(lockedIds) do
			local pl = Players:GetPlayerByUserId(id)
			if pl and not matchRoster[id] then
				table.insert(benched, pl.Name)
				if not timedOut[id] then
					timedOut[id] = true
					Act.timeout(id, true)
				end
			end
		end
		if #benched > 0 then say("Sitting out: " .. table.concat(benched, ", ")) end
		Act.drain() -- timeouts/releases finish (one per ACTION_GAP) before the game starts
		if phase ~= "DRAFT" then return end
		beginLive()
	end

	local function runDraft(perSide, firstCaptain, secondCaptain)
		-- captain 1 sat out round 2, so let them back in; everyone else waits out the draft
		releaseTimeout(firstCaptain)
		for id in pairs(lockedIds) do
			if id ~= firstCaptain and id ~= secondCaptain and not timedOut[id] and Players:GetPlayerByUserId(id) then
				timedOut[id] = true
				Act.timeout(id, true)
			end
		end
		matchRoster, PlayerTeams = {}, {}
		teamOrder = { [TEAM_A_NAME] = {}, [TEAM_B_NAME] = {} }
		setCaptain(TEAM_A_NAME, firstCaptain)
		setCaptain(TEAM_B_NAME, secondCaptain)
		draftId, draftDone, draftApplied, draftPerSide = newMatchId(), false, {}, perSide
		setPhase("DRAFT")
		say(string.format("Draft started (%ds)", cfg.pick_seconds))

		local started = os.clock()
		local limit = cfg.pick_seconds * perSide * 2 + 120 -- only reached if the bot stops answering
		while Running and phase == "DRAFT" and not draftDone do
			if not ensureCaptains() then
				abortToLobby("Captain left. Back to lobby")
				return
			end
			if os.clock() - started > limit then break end
			task.wait(0.5)
		end
		if phase ~= "DRAFT" then return end
		if not draftDone then
			warn("[SCRIM] draft timed out waiting for the bot; filling teams randomly")
			localAutoFill(perSide)
		end
		finalizeDraft()
	end

	local function runCaptains(perSide)
		setPhase("CAPTAINS")
		matchRoster, PlayerTeams = {}, {}
		if RoundActive then endCaptainRound() end -- a practice round must be fully over before anonymous mode is toggled
		setAnonymous(true)
		say("Captain rounds")
		Act.drainUrgent() -- the toggle goes out first (ahead of any kick backlog)...
		task.wait(ANONYMOUS_SETTLE) -- ...and the game gets a moment to apply it before round 1 begins
		local winners = {}
		for round = 1, 2 do
			captainRound = round
			syncNow = true
			if round == 2 then
				timedOut[winners[1]] = true -- the first captain sits out so there are two different captains
				Act.timeout(winners[1], true, true)
			end
			Act.drainUrgent()
			-- round 2: bench players for the duel as they are eliminated, so the queue works through the sit-outs during the round
			-- instead of after it. Only while 2+ players are still alive, so nobody who could still tie for the win is benched.
			local benching = round == 2
			if benching then
				task.spawn(function()
					while benching and captainPhaseActive() do
						if RoundActive and #aliveNames() >= 2 then
							for name in pairs(deadPlayersThisRound) do
								local pid = idForName(name)
								if pid and lockedIds[pid] and not timedOut[pid] and Players:GetPlayerByUserId(pid) then
									timedOut[pid] = true
									Act.timeout(pid, true, true)
								end
							end
						end
						task.wait(0.5)
					end
				end)
			end
			local id = playCaptainRound(round == 2 and winners[1] or nil)
			benching = false
			if not id then
				if phase == "CAPTAINS" then abortToLobby("No captain. Back to lobby") end
				return
			end
			winners[round] = id
			local team = round == 1 and TEAM_A_NAME or TEAM_B_NAME
			captains[team] = { id = id, name = nameOf(id) }
			say(string.format("%s captain: %s", team, nameOf(id)))
		end
		if phase ~= "CAPTAINS" then
			setAnonymous(false) -- same as before the duel existed: anonymous mode never stays on after the rounds
			return
		end

		-- the two captains play a 1v1 round to decide who picks first (the winner is the first captain, Black)
		local a, b = winners[1], winners[2]
		releaseTimeout(a, true) -- the first captain sat out round 2
		for id in pairs(lockedIds) do
			if id ~= a and id ~= b and not timedOut[id] and Players:GetPlayerByUserId(id) then
				timedOut[id] = true
				Act.timeout(id, true, true)
			end
		end
		captainRound = 3
		syncNow = true
		say(string.format("Duel: %s vs %s", nameOf(a), nameOf(b)))
		Act.drainUrgent()
		local first = playCaptainRound(nil, { a, b })
		if not first then
			if phase == "CAPTAINS" then abortToLobby("No first pick. Back to lobby") end
			return
		end
		local second = first == a and b or a
		say(string.format("%s picks first", nameOf(first)))
		setAnonymous(false)
		if phase ~= "CAPTAINS" then return end
		runDraft(perSide, first, second)
	end

	local function lockAndStart()
		if locking then return end
		locking = true
		Agent.forceSync() -- fresh verification for everyone before picking
		if phase ~= "COUNTDOWN" then
			locking = false -- /scrim stop or a reset happened while syncing
			return
		end
		local roster = lobbyRoster()
		local big = bigStart
		local perSide = big and math.min(#roster // 2, Rules.MAX_BIG_PER_TEAM) or Rules.teamSize(#roster, forceStart and 2 or cfg.min_players)
		forceStart, bigStart = false, false
		if perSide == 0 then
			say(string.format("Waiting for players (%d/%d)", #roster, cfg.min_players))
			countdownEnds, latched, lobbyFloor = nil, false, #roster
			locking = false
			setPhase("LOBBY")
			return
		end
		lobbyFloor = 0

		-- from here on only this roster may be in the server; anyone else (unverified, or joining later) is kicked
		lockedIds, leftIds, nameById, captains = {}, {}, {}, {}
		leftAt, voteKicked = {}, {}
		for _, p in ipairs(roster) do
			lockedIds[p.UserId] = true
			nameById[p.UserId] = p.Name
		end
		for _, p in ipairs(presentPlayers()) do
			if not lockedIds[p.UserId] then kick(p, true) end
		end

		matchUnrated = big
		if big then
			say(string.format("Unrated big match: %dv%d", perSide, perSide))
			runRandomTeams(roster, perSide) -- no captain draft: the bot's draft is built for 5 a side at most
		elseif cfg.captains == 1 and perSide > 1 then
			runCaptains(perSide)
		else
			runRandomTeams(roster, perSide) -- captains disabled, or only one player per side (nothing to draft)
		end
	end

	-- Practice rounds while the lobby waits for the real match. They use the game's own round, but no series is active,
	-- so nothing is scored or sent to the bot. The countdown waits for a practice round in progress to finish.
	local practiceStartedAt, practiceSeen, practiceNextAt = nil, false, 0

	local function practiceTick()
		if phase ~= "LOBBY" or locking or Paused then
			practiceStartedAt = nil
			return
		end
		local now = os.clock()
		if practiceStartedAt then
			if RoundActive then practiceSeen = true end
			if practiceSeen and not RoundActive then -- the round ended on its own
				practiceStartedAt, practiceSeen, practiceNextAt = nil, false, now + PRACTICE_GAP
			elseif not practiceSeen and now - practiceStartedAt > 12 then -- the game never started a round
				practiceStartedAt, practiceNextAt = nil, now + PRACTICE_GAP * 2
			elseif now - practiceStartedAt > PRACTICE_MAX_ROUND then -- stuck
				stopRound()
				practiceStartedAt, practiceSeen, practiceNextAt = nil, false, now + PRACTICE_GAP
			end
			return
		end
		if RoundActive or now < practiceNextAt or #lobbyRoster() < PRACTICE_MIN_PLAYERS then return end
		practiceStartedAt, practiceSeen = now, false
		say("Practice round")
		-- the queue can be behind (kicks, timeouts): if the lobby has moved on by then, a late practice round must not start
		Act.fireIf(function() return phase == "LOBBY" and practiceStartedAt ~= nil end, roundArgs())
	end

	local function lobbyTick()
		if locking or Paused or next(customTeams) ~= nil then return end
		local n = #lobbyRoster()
		if phase == "LOBBY" then
			if n < lobbyFloor then lobbyFloor = n end
			-- the timer only starts (and is only announced) once the minimum number of verified players is in,
			-- (a practice round in progress is stopped first)
			if n >= cfg.min_players and n > lobbyFloor then
				if practiceStartedAt then -- enough players: end the practice round now instead of waiting for it
					stopRound()
					practiceStartedAt, practiceSeen, practiceNextAt = nil, false, os.clock() + PRACTICE_GAP
				end
				startCountdown(n)
			end
		elseif phase == "COUNTDOWN" then
			if n < (forceStart and 2 or cfg.min_players) then
				-- dropped below the minimum: cancel the timer quietly, it restarts when enough players are back
				countdownEnds, latched, forceStart = nil, false, false
				setPhase("LOBBY")
				return
			end
			if n >= LOBBY_CAP then latched = true end -- stop resetting; the current countdown just runs out
			local left = countdownEnds - os.clock()
			for _, t in ipairs({ 20, 10, 5 }) do
				if left <= t and not announced[t] and cfg.countdown_seconds > t then
					announced[t] = true
					say(string.format("Starting in %ds (%d/%d)", t, n, LOBBY_CAP))
				end
			end
			if left <= 0 then task.spawn(lockAndStart) end
		end
	end

	local function liveTick()
		if phase ~= "LIVE" or not SeriesActive then return end
		if Paused then -- no forfeit countdown and no round-start retry while paused
			absentSince, absentWarned = {}, {}
			liveSince = os.clock()
			return
		end
		if RoundActive then roundSeen = true end
		if not roundSeen and startRetries < 2 and os.clock() - liveSince > 45 then
			startRetries += 1
			liveSince = os.clock()
			warn("[SCRIM] no round started, firing StartMinigame again")
			Act.fire(roundArgs())
		end

		-- a team with nobody in the server (disconnected players may still rejoin) forfeits after FORFEIT_SECONDS
		local inServer = { [TEAM_A_NAME] = 0, [TEAM_B_NAME] = 0 }
		for id, info in pairs(matchRoster) do
			if Players:GetPlayerByUserId(id) then inServer[info.team] += 1 end
		end
		local now = os.clock()
		for _, team in ipairs({ TEAM_A_NAME, TEAM_B_NAME }) do
			if inServer[team] > 0 then
				absentSince[team], absentWarned[team] = nil, nil
			elseif not absentSince[team] then
				absentSince[team] = now
			end
		end
		local function absentFor(team) return absentSince[team] and now - absentSince[team] or 0 end
		for _, team in ipairs({ TEAM_A_NAME, TEAM_B_NAME }) do
			local waited = absentFor(team)
			if waited >= FORFEIT_SECONDS / 2 and waited < FORFEIT_SECONDS and not absentWarned[team] then
				absentWarned[team] = true
				say(string.format("%s empty. Forfeit in %ds", team, math.ceil(FORFEIT_SECONDS - waited)))
			end
		end
		local aGone, bGone = absentFor(TEAM_A_NAME) >= FORFEIT_SECONDS, absentFor(TEAM_B_NAME) >= FORFEIT_SECONDS
		if inServer[TEAM_A_NAME] == 0 and inServer[TEAM_B_NAME] == 0 then
			-- everyone left: end now, no grace period and no winner. The bot logs this as an aborted match.
			SeriesActive = false
			stopRound()
			resetAll()
			setPhase("LOBBY")
		elseif aGone or bGone then
			local winner = aGone and TEAM_B_NAME or TEAM_A_NAME
			SeriesActive = false
			stopRound()
			conclude(winner, { [TEAM_A_NAME] = SeriesScore[TEAM_A_NAME], [TEAM_B_NAME] = SeriesScore[TEAM_B_NAME] }, copyStats(MatchStats), true)
		end
	end

	-- /scrim console Custom match: lock with exactly the teams set in the lobby (any sizes, uneven allowed), unrated.
	-- Verified players without a team sit out; unverified ones are kicked, as at a normal lock.
	local function startCustom()
		if locking then return end
		locking = true
		Agent.forceSync() -- fresh verification before locking
		if phase ~= "LOBBY" and phase ~= "COUNTDOWN" then
			locking = false
			return
		end
		local present, counts = lobbyRoster(), { [TEAM_A_NAME] = 0, [TEAM_B_NAME] = 0 }
		for _, p in ipairs(present) do
			local team = customTeams[p.UserId]
			if team then counts[team] += 1 end
		end
		if counts[TEAM_A_NAME] == 0 or counts[TEAM_B_NAME] == 0 then
			say("Custom match needs a player on each team")
			locking = false
			return
		end
		countdownEnds, latched, lobbyFloor = nil, false, 0
		lockedIds, leftIds, nameById, captains = {}, {}, {}, {}
		leftAt, voteKicked = {}, {}
		matchRoster, PlayerTeams = {}, {}
		local names = { [TEAM_A_NAME] = {}, [TEAM_B_NAME] = {} }
		local notPicked = {}
		for _, p in ipairs(present) do
			local id, team = p.UserId, customTeams[p.UserId]
			lockedIds[id], nameById[id] = true, p.Name
			if team then
				matchRoster[id] = { name = p.Name, team = team }
				PlayerTeams[p.Name] = team
				table.insert(names[team], p.Name)
				updatePlayerNametag(p)
			else
				timedOut[id] = true -- verified but not on a team: sits out, released after the match
				Act.timeout(id, true)
				table.insert(notPicked, p.Name)
			end
		end
		for _, p in ipairs(presentPlayers()) do
			if not lockedIds[p.UserId] then kick(p, true) end
		end
		customTeams = {}
		matchUnrated = true
		say(string.format("Custom match (unrated): %dv%d", counts[TEAM_A_NAME], counts[TEAM_B_NAME]))
		say(TEAM_A_NAME .. ": " .. table.concat(names[TEAM_A_NAME], ", "))
		say(TEAM_B_NAME .. ": " .. table.concat(names[TEAM_B_NAME], ", "))
		if #notPicked > 0 then say("Sitting out: " .. table.concat(notPicked, ", ")) end
		Act.drain()
		beginLive()
	end

	-- from Discord: /scrim console. Each command arrives once (by key); the phase is checked again here because it may have moved on.
	-- Every remote it fires goes through the jittered action queue like everything else.
	function Agent.admin(cmd)
		local action = cmd.action
		local p = type(cmd.id) == "number" and Players:GetPlayerByUserId(cmd.id) or nil
		local function currentScore() return { [TEAM_A_NAME] = SeriesScore[TEAM_A_NAME], [TEAM_B_NAME] = SeriesScore[TEAM_B_NAME] } end
		print("[SCRIM] admin: " .. tostring(action) .. (p and (" " .. p.Name) or ""))
		if action == "force_start" or action == "big_start" then
			if (phase ~= "LOBBY" and phase ~= "COUNTDOWN") or locking then return end
			local n = #lobbyRoster()
			if n < 2 then return say("Force start needs 2 verified players") end
			forceStart, bigStart = true, action == "big_start"
			Paused, customTeams = false, {} -- starting now overrides a pause and any custom teams
			if phase == "LOBBY" then startCountdown(n) end
			countdownEnds = os.clock() -- lobbyTick locks on its next tick
		elseif action == "custom_start" then
			if (phase ~= "LOBBY" and phase ~= "COUNTDOWN") or locking then return end
			Paused = false
			task.spawn(startCustom) -- it waits for a sync and the action queue, so not on the sync loop
		elseif action == "clear_teams" then
			if next(customTeams) == nil then return end
			customTeams = {}
			say("Custom teams cleared")
		elseif action == "skip_break" then
			if phase == "BREAK" then breakEnds, Paused = os.clock(), false end
		elseif action == "pause" then
			if Paused or (phase ~= "LOBBY" and phase ~= "COUNTDOWN" and phase ~= "BREAK" and phase ~= "LIVE") or locking then return end
			Paused, pausedAt = true, os.clock()
			if phase == "LIVE" then
				stopRound() -- the round in progress is dropped (no point to anyone); the next one waits for resume
				say("Match paused by admin")
			else
				say("Paused by admin")
			end
		elseif action == "resume" then
			if not Paused then return end
			Paused = false
			local frozen = os.clock() - pausedAt -- timers pick up where they stopped
			if countdownEnds then countdownEnds += frozen end
			if breakEnds then breakEnds += frozen end
			say(phase == "LIVE" and "Match resumed" or "Resumed")
			if phase == "LIVE" and SeriesActive and not NextRoundPending then task.spawn(startNextRound) end
		elseif action == "add_time" then
			if phase == "COUNTDOWN" and countdownEnds then
				countdownEnds += 30
				announced = {}
			elseif phase == "BREAK" and breakEnds then
				breakEnds += 30
			else
				return
			end
			say("+30s added by admin")
		elseif action == "restart_round" then
			if phase ~= "LIVE" or not SeriesActive then return end
			say("Round restarted by admin")
			task.spawn(startNextRound)
		elseif action == "abort" then
			if phase ~= "COUNTDOWN" and phase ~= "CAPTAINS" and phase ~= "DRAFT" and phase ~= "LIVE" then return end
			if phase == "LIVE" or phase == "CAPTAINS" then
				SeriesActive = false
				stopRound()
			end
			abortToLobby("Match cancelled by admin") -- the bot logs a cancelled live match as aborted (no rating change)
		elseif action == "end" then
			if phase ~= "LIVE" or (cmd.team ~= TEAM_A_NAME and cmd.team ~= TEAM_B_NAME) then return end
			SeriesActive = false
			stopRound()
			conclude(cmd.team, currentScore(), copyStats(MatchStats), true)
		elseif action == "score" then
			if phase ~= "LIVE" or type(cmd.score) ~= "table" then return end
			local a, b = tonumber(cmd.score[TEAM_A_NAME]) or 0, tonumber(cmd.score[TEAM_B_NAME]) or 0
			SeriesScore[TEAM_A_NAME], SeriesScore[TEAM_B_NAME] = a, b
			say(string.format("Score set: %s %d - %d %s", TEAM_A_NAME, a, b, TEAM_B_NAME))
			if Rules.isMatchOver(a, b, SeriesPerSide) then
				SeriesActive = false
				stopRound()
				conclude(a > b and TEAM_A_NAME or TEAM_B_NAME, currentScore(), copyStats(MatchStats), false)
			end
		elseif action == "announce" then
			if type(cmd.text) == "string" then say(cmd.text) end
		elseif action == "kick" then
			if not p then return end
			local id = p.UserId
			if phase == "LIVE" and matchRoster[id] then
				applyVotekick(id, "kicked by admin") -- off the team and kept out until the match ends
			else
				if lockedIds[id] then leftIds[id] = true end -- a locked player cannot come back this match
				kick(p, false, KICK_DURATION)
			end
		elseif action == "bench" then
			if not p or timedOut[p.UserId] then return end
			timedOut[p.UserId] = true -- released with everyone else when the match ends
			Act.timeout(p.UserId, true, true)
		elseif action == "unbench" then
			if p and timedOut[p.UserId] then releaseTimeout(p.UserId, true) end
		end
		syncNow = true
	end

	local function safe(fn)
		local ok, err = pcall(fn)
		if not ok then warn("[SCRIM] tick error: " .. tostring(err)) end
	end

	task.spawn(function()
		while Running do
			if phase ~= "IDLE" then
				safe(enforce)
				safe(lobbyTick)
				safe(practiceTick)
				safe(liveTick)
			end
			task.wait(0.5)
		end
	end)

	on(Players.PlayerAdded, function(p)
		verifiedCache[p.UserId] = nil -- re-check; they may have verified since last time
		if leftAt[p.UserId] and phase == "LIVE" then timelineEvent("rejoin", p.UserId) end
		leftAt[p.UserId] = nil -- back within the grace period
		kickedUnverified[p.UserId] = nil
		if releaseOnJoin[p.UserId] then
			releaseOnJoin[p.UserId] = nil
			Act.timeout(p.UserId, false)
		end
		syncNow = true
	end)
	on(Players.PlayerRemoving, function(p)
		if lockedIds[p.UserId] and (phase == "CAPTAINS" or phase == "DRAFT" or phase == "LIVE" or phase == "POSTGAME") then
			if not voteKicked[p.UserId] then
				leftAt[p.UserId] = os.clock()
				if phase == "LIVE" then timelineEvent("leave", p.UserId) end
				print("[SCRIM] " .. p.Name .. " disconnected; they can rejoin until the match ends")
			end
		end
		kickPending[p.UserId] = nil
		verifiedCache[p.UserId] = nil
		kickAttempts[p.UserId] = nil
		warnedAt[p.UserId] = nil
		syncNow = true
	end)

	do
		local panel = makePanel("ScrimAgentUI", "Scrim Agent v" .. SCRIPT_VERSION .. " (drag)", UDim2.fromOffset(280, 84), UDim2.fromOffset(340, 280))
		local status = Instance.new("TextLabel")
		status.Size = UDim2.new(1, -6, 0, 52)
		status.Position = UDim2.new(0, 3, 0, 24)
		status.BackgroundTransparency = 1
		status.TextColor3 = Color3.fromRGB(255, 255, 255)
		status.Font = Enum.Font.Gotham
		status.TextSize = 12
		status.TextXAlignment = Enum.TextXAlignment.Left
		status.TextYAlignment = Enum.TextYAlignment.Top
		status.TextWrapped = true
		status.ZIndex = 61
		status.Parent = panel

		task.spawn(function()
			while Running do
				local timedCount = 0
				for _ in pairs(timedOut) do timedCount += 1 end
				status.Text = string.format(
					"phase: %s%s\nplayers: %d | queue: %d | timed out: %d\nsync: %s",
					phase, DRY_RUN and " (DRY RUN)" or "", #presentPlayers(), #queue, timedCount, syncStatus
				)
				task.wait(1)
			end
		end)
	end

	print(string.format("[SCRIM] API %s | remote gap %.1fs | kick lockout %s | DRY_RUN %s", API_URL, ACTION_GAP, tostring(KICK_DURATION), tostring(DRY_RUN)))
	print("[SCRIM] Agent ready. Waiting for /scrim start" .. (DRY_RUN and " (DRY RUN: no remotes are fired)" or ""))
end)()

print("[BOMB TRACKER] Loaded.")
