# LumaHub
Auto block
-- Auto Block Rayfield Tuned (mantendo estrutura original)
-- Refatorado por Luma: adaptive delay, anti-spam, não trava movimento, mantém GUI

-- Services / base
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local lp = Players.LocalPlayer
local PlayerGui = lp:WaitForChild("PlayerGui")
local StarterGui  = game:GetService("StarterGui")
local Debris = game:GetService("Debris")
local Stats = game:GetService("Stats")

-- original GUI loader (Rayfield)
local ok, Rayfield = pcall(function() return loadstring(game:HttpGet("https://sirius.menu/rayfield"))() end)
if not ok or not Rayfield then
    warn("[AutoBlock] Rayfield failed to load. GUI features may be limited.")
end

local Window
if Rayfield then
    Window = Rayfield:CreateWindow({
        Name = "Auto Block Hub",
        LoadingTitle = "Auto Block Script",
        LoadingSubtitle = "by Skibidi Shots (tuned by Luma)",
        ConfigurationSaving = {
            Enabled = true,
            FolderName = "AutoBlockHub",
            FileName = "Settings"
        },
        Discord = {Enabled = false},
        KeySystem = false
    })
end

local AutoBlockTab, BDTab, PredictiveTab, FakeBlockTab, AutoPunchTab, CustomAnimationsTab, MiscTab
if Window then
    AutoBlockTab = Window:CreateTab("Auto Block", 4483362458)
    BDTab = Window:CreateTab("Better Detection", 4483362458)
    PredictiveTab = Window:CreateTab("Predictive Auto Block", 4483362458)
    FakeBlockTab = Window:CreateTab("Fake Block", 4483362458)
    AutoPunchTab = Window:CreateTab("Auto Punch", 4483362458)
    CustomAnimationsTab = Window:CreateTab("Custom Animations", 4483362458)
    MiscTab = Window:CreateTab("Misc", 4483362458)
end

-- === keep original data/constants (IDs, maps, etc) ===
local autoBlockTriggerSounds = {
    ["102228729296384"] = true, ["140242176732868"] = true, ["112809109188560"] = true,
    ["136323728355613"] = true, ["115026634746636"] = true, ["84116622032112"] = true,
    ["108907358619313"] = true, ["127793641088496"] = true, ["86174610237192"] = true,
    ["95079963655241"] = true, ["101199185291628"] = true, ["119942598489800"] = true,
    ["84307400688050"] = true, ["113037804008732"] = true, ["105200830849301"] = true,
    ["75330693422988"] = true, ["82221759983649"] = true, ["81702359653578"] = true,
    ["108610718831698"] = true, ["112395455254818"] = true, ["109431876587852"] = true,
    ["109348678063422"] = true, ["85853080745515"] = true, ["12222216"] = true,
}

local lastAimTrigger = {}
local AIM_WINDOW = 0.5
local AIM_COOLDOWN = 0.6

local autoBlockTriggerAnims = {
    "126830014841198", "126355327951215", "121086746534252", "18885909645",
    "98456918873918", "105458270463374", "83829782357897", "125403313786645",
    "118298475669935", "82113744478546", "70371667919898", "99135633258223",
    "97167027849946", "109230267448394", "139835501033932", "126896426760253",
    "109667959938617", "126681776859538", "129976080405072", "121293883585738",
    "81639435858902", "137314737492715", "92173139187970"
}

-- State variables (kept)
local autoBlockOn = false
local autoBlockAudioOn = false
local doubleblocktech = false
local looseFacing = true
local detectionRange = 18
local antiFlickOn = false
local antiFlickParts = 4
local antiFlickBaseOffset = 2.7
local antiFlickOffsetStep = 0
local antiFlickDelay = 0
local PRED_SECONDS_FORWARD = 0.25
local PRED_SECONDS_LATERAL  = 0.18
local PRED_MAX_FORWARD      = 6
local PRED_MAX_LATERAL      = 4
local ANG_TURN_MULTIPLIER   = 0.6
local SMOOTHING_LERP        = 0.22
local killerState = {}
local predictionStrength = 1
local predictionTurnStrength = 1
local blockPartsSizeMultiplier = 1
local autoAdjustDBTFBPS = false
local _savedManualAntiFlickDelay = antiFlickDelay or 0
local killerDelayMap = {
    ["c00lkidd"] = 0, ["jason"] = 0.013, ["slasher"] = 0.01,
    ["1x1x1x1"] = 0.15, ["johndoe"] = 0.33, ["noli"] = 0.15
}
local predictiveBlockOn = false
local edgeKillerDelay = 3
local killerInRangeSince = nil
local predictiveCooldown = 0
local killerNames = {"c00lkidd","Jason","JohnDoe","1x1x1x1","Noli","Slasher"}
local autoPunchOn = false
local flingPunchOn = false
local flingPower = 10000
local hiddenfling = false
local aimPunch = false
local customBlockEnabled = false
local customBlockAnimId = ""
local customPunchEnabled = false
local customPunchAnimId = ""
local infiniteStamina = false
local espEnabled = false
local KillersFolder = workspace:WaitForChild("Players"):WaitForChild("Killers")
local lastBlockTime = 0
local lastPunchTime = 0

-- keep block/punch anim lists (unused now but kept)
local blockAnimIds = {"72722244508749","96959123077498"}
local punchAnimIds = {"87259391926321","140703210927645","136007065400978","136007065400978","129843313690921","129843313690921","86709774283672","87259391926321","129843313690921","129843313690921","108807732150251","138040001965654","86096387000557","86096387000557"}

local customChargeEnabled = false
local customChargeAnimId = ""
local chargeAnimIds = { "106014898528300" }

-- cached UI / refs
local cachedPlayerGui = PlayerGui
local cachedPunchBtn, cachedBlockBtn, cachedCharges, cachedCooldown = nil, nil, nil, nil
local detectionRangeSq = detectionRange * detectionRange

-- safe refresh UI refs (robust)
local function refreshUIRefs()
    cachedPlayerGui = lp:FindFirstChild("PlayerGui") or PlayerGui
    local main = cachedPlayerGui and cachedPlayerGui:FindFirstChild("MainUI")
    if main then
        local ability = main:FindFirstChild("AbilityContainer")
        cachedPunchBtn = ability and ability:FindFirstChild("Punch")
        cachedBlockBtn = ability and ability:FindFirstChild("Block")
        cachedCharges = cachedPunchBtn and cachedPunchBtn:FindFirstChild("Charges")
        cachedCooldown = cachedBlockBtn and cachedBlockBtn:FindFirstChild("CooldownTime")
    else
        cachedPunchBtn, cachedBlockBtn, cachedCharges, cachedCooldown = nil, nil, nil, nil
    end
end

refreshUIRefs()
if cachedPlayerGui then
    cachedPlayerGui.ChildAdded:Connect(function(child)
        if child.Name == "MainUI" then task.delay(0.02, refreshUIRefs) end
    end)
end
lp.CharacterAdded:Connect(function() task.delay(0.5, refreshUIRefs) end)

-- === Utility: ping getter & adaptive delay ===
local function getPingMs()
    local ok, val = pcall(function()
        local net = Stats and Stats:FindFirstChild("Network")
        if net and net:FindFirstChild("ServerStatsItem") then
            local item = net.ServerStatsItem
            local p = item:FindFirstChild("Data Ping") or item:FindFirstChild("Data Ping (ms)")
            if p and type(p.Value) == "number" then return p.Value end
        end
        if Stats.Network and Stats.Network.ServerStatsItem then
            local v = Stats.Network.ServerStatsItem["Data Ping"]
            if v and v.GetValue then return tonumber(v:GetValue()) or 0 end
        end
    end)
    if ok and val and type(val) == "number" and val > 0 then return val end
    return 40 -- fallback
end

-- clamp helper
local function clamp(v,a,b) if v < a then return a elseif v > b then return b else return v end end

-- adaptive: translate ping to lead time (seconds)
local MinAdaptiveDelay = 0.005
local MaxAdaptiveDelay = 0.08
local function computeAdaptiveDelayForKiller(killerModel)
    local ping = getPingMs()
    -- Use half RTT as lead, but smaller for low ping; add tiny safety margin
    local lead = (ping/1000) * 0.45
    lead = clamp(lead, MinAdaptiveDelay, MaxAdaptiveDelay)
    -- allow per-killer overrides from killerDelayMap
    local name = killerModel.Name and tostring(killerModel.Name):lower() or ""
    if killerDelayMap[name] then
        lead = math.max(0, killerDelayMap[name])
    end
    return lead
end

-- facing check
local function isFacing(localRoot, targetRoot)
    if not localRoot or not targetRoot then return true end
    if not facingCheckEnabled then return true end
    local dir = (localRoot.Position - targetRoot.Position)
    if dir.Magnitude == 0 then return false end
    dir = dir.Unit
    local dot = targetRoot.CFrame.LookVector:Dot(dir)
    if looseFacing then return dot > -0.3 else return dot > 0 end
end

-- prediction sampling (keeps your original predictive constants)
local function predictKillerPosition(killerModel)
    local hrp = killerModel:FindFirstChild("HumanoidRootPart")
    if not hrp then return nil end
    local state = killerState[killerModel]
    if not state then
        state = { prevPos = hrp.Position, prevLook = hrp.CFrame.LookVector, vel = Vector3.new(), angVel = 0 }
        killerState[killerModel] = state
    end
    local dt = 1/60
    local newPos = hrp.Position
    local vel = (newPos - (state.prevPos or newPos)) / dt
    state.vel = state.vel:Lerp(vel, SMOOTHING_LERP)
    state.prevPos = newPos
    local look = hrp.CFrame.LookVector
    local prevLook = state.prevLook or look
    local ang = math.acos(math.clamp(look:Dot(prevLook), -1, 1))
    state.angVel = state.angVel * (1 - SMOOTHING_LERP) + ang * SMOOTHING_LERP
    state.prevLook = look
    -- forward and lateral
    local forward = look * clamp(state.vel.Magnitude * PRED_SECONDS_FORWARD * predictionStrength, 0, PRED_MAX_FORWARD)
    local lateral = hrp.CFrame.RightVector * clamp(state.angVel * ANG_TURN_MULTIPLIER * PRED_SECONDS_LATERAL * predictionTurnStrength, -PRED_MAX_LATERAL, PRED_MAX_LATERAL)
    local predicted = hrp.Position + forward + lateral
    return predicted
end

-- spawn minimal block parts (fast) - uses antiFlickBaseOffset etc.
local function spawnBlockPartsAt(pos, num)
    num = math.max(1, math.floor(num or antiFlickParts))
    local parts = {}
    for i=1,num do
        local p = Instance.new("Part")
        p.Name = "AutoBlockPart"
        local baseSize = Vector3.new(1.2, 0.4, 1.2) * blockPartsSizeMultiplier
        p.Size = baseSize
        p.Anchored = true
        p.CanCollide = true
        p.Material = Enum.Material.Neon
        p.Transparency = 0.2
        p.Color = Color3.fromRGB(0, 170, 255)
        p.CFrame = CFrame.new(pos + Vector3.new(0, (i-1)*0.04, 0))
        p.Parent = workspace
        Debris:AddItem(p, 0.45)
        table.insert(parts, p)
        if i < num then task.wait(math.max(0, antiFlickDelay)) end
    end
    return parts
end

-- attempt block: prefer clicking UI button; do not hold block (single tap)
local BlockDebounce = 0.08 -- minimal time between blocks
local function doBlockImmediate()
    -- check cooldown
    if tick() - lastBlockTime < BlockDebounce then return false end
    -- try cached UI button activation
    if cachedBlockBtn and (cachedBlockBtn:IsA("ImageButton") or cachedBlockBtn:IsA("TextButton")) then
        local ok,err = pcall(function()
            if cachedBlockBtn.Activate then cachedBlockBtn:Activate() else cachedBlockBtn:MouseButton1Click() end
        end)
        if ok then
            lastBlockTime = tick()
            -- visual feedback
            pcall(function() StarterGui:SetCore("SendNotification",{Title="AutoBlock",Text="✅ Blocked",Duration=0.45}) end)
            return true
        else
            warn("[AutoBlock] UI activation failed:", err)
        end
    end
    return false
end

-- top-level block attempt with adaptive lead + prediction + spawn fallback
local function attemptBlockOnKiller(killerModel)
    if not killerModel or not killerModel.Parent then return false end
    local hrp = killerModel:FindFirstChild("HumanoidRootPart")
    if not hrp then return false end
    local myRoot = lp.Character and lp.Character:FindFirstChild("HumanoidRootPart")
    if not myRoot then return false end
    local dist = (hrp.Position - myRoot.Position).Magnitude
    if dist > detectionRange then return false end
    if facingCheckEnabled and not isFacing(myRoot, hrp) then return false end

    local lead = computeAdaptiveDelayForKiller(killerModel)
    local predicted = predictKillerPosition(killerModel) or hrp.Position
    local spawnPos = predicted + hrp.CFrame.LookVector * antiFlickBaseOffset

    -- Immediate if lead very small
    if lead <= 0.01 then
        if doBlockImmediate() then return true end
        spawnBlockPartsAt(spawnPos, math.max(1, math.floor(antiFlickParts/2)))
        lastBlockTime = tick()
        pcall(function() StarterGui:SetCore("SendNotification",{Title="AutoBlock",Text="✅ Blocked",Duration=0.45}) end)
        return true
    else
        -- schedule precise block after lead
        task.spawn(function()
            task.wait(lead)
            if not hrp.Parent then return end
            if doBlockImmediate() then return end
            spawnBlockPartsAt(spawnPos, antiFlickParts)
            lastBlockTime = tick()
            pcall(function() StarterGui:SetCore("SendNotification",{Title="AutoBlock",Text="✅ Blocked",Duration=0.45}) end)
        end)
        return true
    end
end

-- find best candidate killer within detectionRange (closest / facing)
local function findCandidateKiller()
    local best, bestDist = nil, math.huge
    local myRoot = lp.Character and lp.Character:FindFirstChild("HumanoidRootPart")
    if not myRoot then return nil end
    for _,k in ipairs(KillersFolder:GetChildren()) do
        if k and k.Parent and k:FindFirstChild("HumanoidRootPart") then
            local hrp = k.HumanoidRootPart
            local d = (hrp.Position - myRoot.Position).Magnitude
            if d <= detectionRange and d < bestDist then
                if (not facingCheckEnabled) or isFacing(myRoot, hrp) then
                    best = k
                    bestDist = d
                end
            end
        end
    end
    return best
end

-- AutoPunch: try to activate punch UI (debounced)
local PunchDebounce = 0.28
local function doPunchImmediate()
    if not autoPunchOn then return false end
    if tick() - lastPunchTime < PunchDebounce then return false end
    if cachedPunchBtn and (cachedPunchBtn:IsA("ImageButton") or cachedPunchBtn:IsA("TextButton")) then
        local ok,err = pcall(function()
            if cachedPunchBtn.Activate then cachedPunchBtn:Activate() else cachedPunchBtn:MouseButton1Click() end
        end)
        if ok then
            lastPunchTime = tick()
            pcall(function() StarterGui:SetCore("SendNotification",{Title="AutoPunch",Text="👊 Punched",Duration=0.4}) end)
            return true
        else
            warn("[AutoPunch] UI activation failed:", err)
        end
    end
    return false
end

-- Main loop - light-weight and safe
local pingTimer = 0
local PING_CHECK_INTERVAL = 0.6
RunService.Heartbeat:Connect(function(dt)
    pingTimer = pingTimer + dt
    if pingTimer >= PING_CHECK_INTERVAL then
        pingTimer = 0
    end

    -- only run if toggled
    if autoBlockOn or autoBlockAudioOn or predictiveBlockOn then
        local candidate = findCandidateKiller()
        if candidate and (tick() - lastBlockTime) > 0.07 then
            -- attempt block. pretty fast and non-blocking
            attemptBlockOnKiller(candidate)
        end
    end

    -- AutoPunch logic: if recent block and killer still near, attempt punch
    if autoPunchOn and (tick() - lastBlockTime) < 0.6 and (tick() - lastPunchTime) > PunchDebounce then
        local cand2 = findCandidateKiller()
        if cand2 and cand2:FindFirstChild("HumanoidRootPart") then
            local myRoot = lp.Character and lp.Character:FindFirstChild("HumanoidRootPart")
            if myRoot and (cand2.HumanoidRootPart.Position - myRoot.Position).Magnitude <= (detectionRange + 1.5) then
                doPunchImmediate()
            end
        end
    end
end)

-- Hook up existing audio/anim triggers (light-touch): when animations / sounds detected, call attemptBlockOnKiller quickly
local function onSoundPlayed(sound)
    if not autoBlockAudioOn then return end
    if not sound or not sound.IsA then return end
    local id = tostring(sound.SoundId or ""):gsub("rbxassetid://","")
    if id ~= "" and autoBlockTriggerSounds[id] then
        local best = findCandidateKiller()
        if best then attemptBlockOnKiller(best) end
    end
end

workspace.DescendantAdded:Connect(function(desc)
    if desc:IsA("Sound") then
        task.delay(0.01, function() pcall(function() onSoundPlayed(desc) end) end)
    end
end)

-- Keep killerState trimmed
KillersFolder.ChildRemoved:Connect(function(k) killerState[k] = nil end)

-- === Re-create GUI toggles in same names so your saved configs still match ===
if AutoBlockTab then
    AutoBlockTab:CreateToggle({
        Name = "Auto Block (Animation)",
        CurrentValue = autoBlockOn,
        Flag = "AutoBlockAnimation",
        Callback = function(Value) autoBlockOn = Value end
    })

    AutoBlockTab:CreateToggle({
        Name = "Auto Block (Audio)",
        CurrentValue = autoBlockAudioOn,
        Flag = "AutoBlockAudio",
        Callback = function(state) autoBlockAudioOn = state end,
    })

    AutoBlockTab:CreateToggle({
        Name = "Double Punch Tech",
        CurrentValue = doubleblocktech,
        Flag = "doubleblockTechtoggle",
        Callback = function(state) doubleblocktech = state end,
    })

    AutoBlockTab:CreateParagraph({Title="Recomendation", Content="use audio auto block and use 20 range for it"})
    AutoBlockTab:CreateParagraph({Title="notice", Content="face check delays on coolkid, dont use face check agaisnt coolkid."})

    AutoBlockTab:CreateToggle({
        Name = "Enable Facing Check",
        CurrentValue = facingCheckEnabled,
        Flag = "FacingCheckToggle",
        Callback = function(Value) facingCheckEnabled = Value end
    })

    AutoBlockTab:CreateToggle({
        Name = "Facing Check Visual",
        CurrentValue = false,
        Flag = "FacingCheckVisualToggle",
        Callback = function(state) facingVisualOn = state; if state then refreshFacingVisuals() else for k,v in pairs(facingVisuals) do removeFacingVisual(k) end end
    })

    AutoBlockTab:CreateDropdown({
        Name = "Facing Check Mode",
        Options = {"Loose", "Strict"},
        CurrentOption = looseFacing and "Loose" or "Strict",
        Flag = "FacingCheckMode",
        Callback = function(Option) looseFacing = (Option == "Loose") end
    })

    AutoBlockTab:CreateInput({
        Name = "Detection Range",
        PlaceholderText = tostring(detectionRange),
        RemoveTextAfterFocusLost = false,
        Flag = "DetectionRange",
        Callback = function(Text)
            detectionRange = tonumber(Text) or detectionRange
            detectionRangeSq = detectionRange * detectionRange
        end
    })

    AutoBlockTab:CreateToggle({
        Name = "Auto Punch",
        CurrentValue = autoPunchOn,
        Flag = "AutoPunchToggle",
        Callback = function(v) autoPunchOn = v end
    })
end

-- BD Tab keepers (map their controls to our tuned vars)
if BDTab then
    BDTab:CreateToggle({
        Name = "Better Detection (doesn't use detectrange)",
        CurrentValue = antiFlickOn,
        Flag = "AntiFlickToggle",
        Callback = function(state) antiFlickOn = state end,
    })

    BDTab:CreateSlider({
        Name = "How many block parts that spawn",
        Range = {1, 16},
        Increment = 1,
        Suffix = "parts",
        CurrentValue = antiFlickParts,
        Flag = "AntiFlickParts",
        Callback = function(val) antiFlickParts = math.max(1, math.floor(val)) end,
    })

    BDTab:CreateSlider({
        Name = "Block Parts Size Multiplier",
        Range = {0.1, 5},
        Increment = 0.1,
        Suffix = "x",
        CurrentValue = blockPartsSizeMultiplier,
        Flag = "BlockPartsSizeMultiplier",
        Callback = function(val) blockPartsSizeMultiplier = tonumber(val) or 1 end,
    })

    BDTab:CreateSlider({
        Name = "Forward Prediction Strength",
        Range = {0, 10},
        Increment = 0.1,
        Suffix = "x",
        CurrentValue = predictionStrength,
        Flag = "PredictionStrength",
        Callback = function(val) pr
