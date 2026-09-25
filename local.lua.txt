pcall(function()
    if not game:IsLoaded() then game.Loaded:Wait() end
end)

--// CƠ CHẾ AUTO-KILL: TỰ ĐỘNG DIỆT BẢN CŨ TRÁNH ĐÈ SCRIPT //--
local globalEnv = (getgenv and getgenv()) or _G or shared or {}

if globalEnv.HeavyweightFishingKill then
    pcall(globalEnv.HeavyweightFishingKill)
    globalEnv.HeavyweightFishingKill = nil
end
if globalEnv.IdenticalHeavyweightFishingUnload then
    pcall(globalEnv.IdenticalHeavyweightFishingUnload)
    globalEnv.IdenticalHeavyweightFishingUnload = nil
end

local function CleanOldInstances()
    local guiLocations = {}
    if gethui then
        local ok, h = pcall(gethui)
        if ok and h then table.insert(guiLocations, h) end
    end
    pcall(function()
        table.insert(guiLocations, game:GetService("CoreGui"))
    end)
    pcall(function()
        local lp = game:GetService("Players").LocalPlayer
        if lp and lp:FindFirstChild("PlayerGui") then
            table.insert(guiLocations, lp.PlayerGui)
        end
    end)

    local targetNames = {
        "IdenticalHeavyweightFishing",
        "HeavyweightFishing",
        "FloatingAvatar",
        "FloatingCrescent",
        "Notifications"
    }

    for _, loc in ipairs(guiLocations) do
        pcall(function()
            for _, child in ipairs(loc:GetChildren()) do
                for _, name in ipairs(targetNames) do
                    if child.Name == name then
                        child:Destroy()
                    end
                end
            end
        end)
    end

    pcall(function()
        local ws = game:GetService("Workspace")
        for _, child in ipairs(ws:GetChildren()) do
            if child.Name == "IdenticalESP" then child:Destroy() end
        end
        for _, p in ipairs(game:GetService("Players"):GetPlayers()) do
            if p.Character then
                for _, d in ipairs(p.Character:GetDescendants()) do
                    if d:IsA("BillboardGui") and d.Name:sub(1, 4) == "ESP_" then
                        d:Destroy()
                    end
                end
            end
        end
    end)
end
CleanOldInstances()

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local HttpService = game:GetService("HttpService")
local TeleportService = game:GetService("TeleportService")
local Lighting = game:GetService("Lighting")
local CoreGui = game:GetService("CoreGui")

local LocalPlayer = Players.LocalPlayer
if not LocalPlayer then
    pcall(function()
        repeat task.wait() until Players.LocalPlayer
    end)
    LocalPlayer = Players.LocalPlayer
end
local Camera = Workspace.CurrentCamera or Workspace:FindFirstChildWhichIsA("Camera")

-- Không can thiệp GuiNavigationEnabled toàn cục để bảo đảm người chơi thao tác UI game bình thường

local isRunning = true
local activeConnections = {}
local cleanUpInstances = {}

--// MÃ COMMIT BẢN BUILD HIỆN TẠI (NHÚNG TĨNH TRONG CODE, KHÔNG DÙNG MẠNG) //--
local SCRIPT_BUILD_COMMIT = "v2.9.1"

local Events = ReplicatedStorage:FindFirstChild("Events")
if not Events then
    task.spawn(function()
        Events = ReplicatedStorage:WaitForChild("Events", 5)
    end)
end

local Config = {
    AutoMinigame = true,
    RhythmAccuracy = 95,
    RhythmHumanizer = true,
    AutoCast = false,
    CastDelay = 1.0,
    CastPower = 100,
    AnchorBar = true,
    AutoSlam = true,
    AutoCharge = true,
    AntiStuckEnabled = false,
    SmartComboEnabled = false,
    FishHpThreshold = 500,
    QuickCatchSkill = "Z",
    OpenerSkill = "Z",
    OpenerMaxCount = 1,
    LoopSkills = "Z, X, V",
    LoopStrictOrder = true,
    EmergencyHealSkill = "V",
    EmergencyHealHp = 40,
    SkillEffectDelay = 0.3,
    SmartEffectAutoDetect = true,
    AutoSkills = false,
    SelectedSkill = "One-Strike Heaven Gate",
    AutoTrainSkill = false,
    TrainSkill = "Z",
    TrainCancelDelay = 0.45,
    Train_Z = false,
    Train_X = false,
    Train_C = true,
    Train_V = true,
    TrainTargetCount = 100,
    TrainCurrentCount = 0,
    TrainSkillCooldown = 6.0,
    TrainDelayCatch = true,
    
    AutoEquipBestBait = false,
    BaitChoiceNormal = "Mồi Tốt Nhất (Cao Nhất)",
    AutoEquipBossBait = true,
    BaitChoiceBoss = "Mồi Tốt Nhất (Cao Nhất)",
    AutoEquipBestRod = false,
    AutoEquipBestOrb = false,
    Loadout1_Rod = "Wooden Rod",
    Loadout1_Bait = "Basic Bait",
    Loadout2_Rod = "Wooden Rod",
    Loadout2_Bait = "Basic Bait",
    
    AutoSell = false,
    SellInterval = 30,
    AutoFavouriteFish = false,
    FavouriteFishName = "Colossal Tigerfish",
    MaterialFarming = false,
    
    OctoAutoMinigame = false,
    AutoFarmBoss = false,
    AutoFarmSecretBoss = false,
    SelectedBoss = "Enzo",
    
    -- TỰ ĐỘNG SĂN SECRET BOSS THEO CHAT & TẠI ĐẢO
    AutoHuntBoss = false,
    AutoChatSecretBoss = false,
    AutoServerHopOnDespawn = false,
    FastSkipNonBoss = true,
    SecretBossCheckPower = true,
    SecretBossTargets = {
        ["Verdant Alligator Gar"] = true,
        ["Verdant Grouper"] = true,
        ["Verdant Bonefang"] = true,
        ["Crimson Bonefang"] = true,
        ["Scarlet Fish"] = true,
        ["Elder Scarlet Fish"] = true,
        ["Crimson Electric Eel"] = true,
        ["Golden Dragonfish"] = true,
        ["Rainbow Dragonfish"] = true,
        ["Flying Fish Emperor"] = true,
        ["Flying Fish Empress"] = true,
        ["Draconic Koi"] = true,
        ["Sanguine Fish"] = true,
        ["Tigerfang Whale"] = true,
        ["Heavenpiercer Turtle"] = true,
        ["Heaven Piercer Turtle"] = true,
        ["Reborn Puffer Beast"] = true,
        ["Frost Kingfish"] = true,
        ["Frost Queenfish"] = true,
        ["Mountain Dragonwhale"] = true,
        ["Mirage Lanternfish"] = true,
        ["Nameless Octoparasite"] = true,
        -- Cá Thường Có Ích (Rơi Skill / Thuyền / Orb / Chế Cần & Mồi)
        ["Trueform Jiaolongfish"] = true,
        ["Adult Jiaolong Dragonfish"] = false,
        ["Elder Jiaolong Dragonfish"] = false,
        ["Serpent Fish"] = false,
        ["Ascended Perch"] = true,
        ["Trueform Perch"] = true,
        ["Elder Perch VIII"] = false,
        ["Dark Kingfish"] = false,
        ["Glorious Elder Turtle"] = false,
        ["Chromatic Koi"] = false,
        ["Mountain Fish"] = true,
        ["Tiger Mirefish"] = true,
        ["Octoparasitic Fish"] = true,
        ["Dreadmare Eel"] = false,
    },
    CustomBossSpots = {},
    SelectedCustomSpotIsland = "Đảo Tre (Bamboo Isle)",
    SelectedCustomSpotSlot = 1,
    BossSpotAllocationMode = "Tự Động (Theo Acc)",
    BossTeleportJitter = true,
    BossTeleportJitterDist = 1.0,
    ReturnToHomeWhenClear = true,
    HomeFarmSpot = nil,
    
    AutoGodSpiritCheck = false,
    AutoPrayGodSpirit = false,
    AutoServerHopGod = false,
    AutoServerHopMaoshan = false,
    AutoServerHopTaoist = false,
    
    AutoWeatherHop = false,
    TargetWeather = "Bất Kỳ Thời Tiết Nào (Trừ Clear)",
    WeatherHopAutoFish = true,
    WeatherHopAlertWebhook = true,
    
    AutoTicketQuest = false,
    TicketDifficulty = "Hard",
    TicketQuestMode = "Tự Động (Auto Detect)",
    TicketBaitChoice = "Basic Bait",
    TicketSkillKey = "Chiêu Z",
    TicketQuickSkill = "Chiêu V",
    TicketCooldownMinutes = 20,
    TicketAutoSellFull = true,
    TicketReturnHomeWhenDone = true,
    TicketAutoCastAtHome = true,
    TicketRemoteClaim = true,
    
    -- Nhiệm Vụ Kỹ Năng Zeng Tianguo & Chế Độ Song Song (Parallel Quest Mode)
    AutoZengTianguoQuest = false,
    ParallelQuestMode = true,
    ZengTianguoAutoClaim = true,
    
    -- Hệ Thống Quản Lý Độ Ưu Tiên (Priority Manager)
    PrioritySystemEnabled = true,
    PriorityPreset = "Mặc Định: Săn Boss > Vé NV > Thần Linh > Luyện Chiêu > Farm Thường",
    Priority_SecretBoss = 1,
    Priority_TicketQuest = 2,
    Priority_GodSpirit = 3,
    Priority_TrainSkill = 4,
    Priority_NormalFarm = 5,

    AutoClaimDaily = false,
    DailyClaimDelay = 0.5,
    
    AutoCraftBait = false,
    CraftBaitName = "Nameless Bait",
    CraftAmount = 1,
    AutoBuyBait = false,
    BuyBaitName = "Ancestral Bait",
    BuyBaitAmount = 5,
    BuyBaitThreshold = 10,
    BuyBaitDelay = 1.0,
    
    AutoGacha = false,
    GachaBanner = "Taiji Banner",
    GachaPullsPerAction = 1,
    
    WalkSpeedEnabled = true,
    WalkSpeedValue = 60,
    FlyEnabled = false,
    FlySpeed = 50,
    InfiniteJump = true,
    WalkOnWater = true,
    Noclip = false,
    
    ESP_GodSpirit = false,
    ESP_SecretRod = false,
    ESP_Boats = false,
    ESP_Maoshan = false,
    ESP_Taoist = false,
    ESP_Boss = false,
    ESP_Players = false,
    FishRedRing = true,
    ClearFarVision = true,
    NoFog = false,
    Fullbright = true,
    FullbrightLevel = 2.0,
    FullbrightAntiGlare = true,
    PerformanceMode = false,
    HideGameUI = false,
    HideOverheadNames = true,
    
    AntiAFK = true,
    AutoRejoin = false,
    AutoExecuteOnJoin = false,
    AutoProtectMutations = true,
    AcidWaterShield = false,
    
    -- Cài đặt Tab Thử Nghiệm (Experimental)
    GhostInvisibility = false,
    AutoRerollTrait = false,
    TargetTraitName = "Azure Dragon",
    SelectedBoat = "Boat",
    SelectedRodColor = "Vàng Kim (Gold)",
    RainbowRodColor = false,
    SelectedExchangeItem = "Trait Reroll",
    ExchangeAmount = 1,
    ShowFishWeightRing = false,
    WebhookEnabled = false,
    WebhookUrl = "",
    WebhookNotifyBoss = true,
    WebhookNotifyNPC = true,
    WebhookHourlyStats = true,
    WebhookNotifyTicketQuest = true,
    WebhookStatsInterval = 30,
    TelegramEnabled = false,
    TelegramBotToken = "",
    TelegramChatId = "",
    TelegramNotifyBoss = true,
    TelegramNotifyNPC = true,
    -- ntfy Push (Thông Báo Thời Tiết & Server Về Điện Thoại)
    NtfyEnabled = false,
    NtfyTopic = "",
    NtfyAlertWeatherChange = true,
    NtfyAlertWeatherHop = true,
    NtfyNotifyBoss = true,
    ShowBossDpsMeter = true,
    UIKeybind = Enum.KeyCode.RightControl,
    StopKeybind = Enum.KeyCode.End,
    ActiveProfile = "default",
    AutoLoadProfile = false
}

-- Tự động khôi phục trạng thái tìm server thời tiết nếu vừa teleport sang server mới
local pendingWeatherHopData = nil
if isfile and isfile("HeavyweightFishing_WeatherHop.json") and readfile then
    local hopOk, hopContent = pcall(function() return readfile("HeavyweightFishing_WeatherHop.json") end)
    if hopOk and hopContent and #hopContent > 0 then
        local decOk, hopData = pcall(function() return HttpService:JSONDecode(hopContent) end)
        if decOk and type(hopData) == "table" and hopData.Active then
            pendingWeatherHopData = hopData
            Config.AutoWeatherHop = true
            if hopData.TargetWeather then Config.TargetWeather = hopData.TargetWeather end
            if hopData.AutoFish ~= nil then Config.WeatherHopAutoFish = hopData.AutoFish end
            if hopData.Webhook ~= nil then Config.WeatherHopAlertWebhook = hopData.Webhook end
        end
    end
end

-- Tự động khôi phục trạng thái tìm server NPC (Taoist / Maoshan / Thần Linh)
local pendingNPCHopData = nil
if isfile and isfile("HeavyweightFishing_NPCHop.json") and readfile then
    local hopOk, hopContent = pcall(function() return readfile("HeavyweightFishing_NPCHop.json") end)
    if hopOk and hopContent and #hopContent > 0 then
        local decOk, hopData = pcall(function() return HttpService:JSONDecode(hopContent) end)
        if decOk and type(hopData) == "table" and hopData.Active then
            pendingNPCHopData = hopData
            if hopData.Taoist then Config.AutoServerHopTaoist = true end
            if hopData.Maoshan then Config.AutoServerHopMaoshan = true end
            if hopData.God then Config.AutoServerHopGod = true end
        end
    end
end

local UIControllers = {}
local PriorityManager = {}

local comboState = {
    openerUsedCount = 0,
    openerDone = false,
    loopTargetIndex = 1,
    loopIndex = 1,
    lastCastTime = 0,
    lastActionTime = 0,
    minigameStartTime = 0,
    usedTimes = {
        ["Z"] = 0,
        ["X"] = 0,
        ["C"] = 0,
        ["V"] = 0
    },
    defaultCooldowns = {
        ["Z"] = 2.5,
        ["X"] = 3.0,
        ["C"] = 5.0,
        ["V"] = 4.0
    },
    loopWaitStartTime = 0
}

function comboState.SkillExists(sk, fUI)
    if not sk or sk == "" or sk == "Tắt" then return false end
    return true
end

function comboState.FormatCombo(str)
    local keys = {}
    for k in string.gmatch(str or "", "([ZXCVzxcv])") do
        table.insert(keys, k:upper())
    end
    return table.concat(keys, ", ")
end

function comboState.GetComboPreview(str)
    local keys = {}
    for k in string.gmatch(str or "", "([ZXCVzxcv])") do
        table.insert(keys, k:upper())
    end
    if #keys == 0 then
        return "(Chưa có chiêu)"
    end
    return table.concat(keys, " ➔ ") .. string.format(" (%d chiêu)", #keys)
end

local ConfigLabelMap = {
    -- Câu cá cốt lõi
    ["Tự Động Quăng Cần (Auto Cast)"] = "AutoCast",
    ["Độ Trễ Quăng Cần"] = "CastDelay",
    ["Giữ Thanh Minigame (Anchor Bar)"] = "AnchorBar",
    ["Tự Dùng Kỹ Năng Cần"] = "AutoSkills",
    ["Tự Động Đập Cần (Auto Slam)"] = "AutoSlam",
    ["Tự Động Sạc Dây (Auto Charge)"] = "AutoCharge",
    ["Tự Động Chống Kẹt Cần (Anti-Stuck)"] = "AntiStuckEnabled",

    -- Smart Combo
    ["Bật Combo Kỹ Năng Tự Động"] = "SmartComboEnabled",
    ["Ngưỡng Máu Cá Phân Loại"] = "FishHpThreshold",
    ["Chiêu Bắt Nhanh (<= Ngưỡng HP)"] = "QuickCatchSkill",
    ["Chiêu Mở Màn (> Ngưỡng HP)"] = "OpenerSkill",
    ["Số Lần Dùng Chiêu Mở Màn"] = "OpenerMaxCount",
    ["Chuỗi Đảo Chiêu Luân Phiên"] = "LoopSkills",
    ["Tùy Biến Chuỗi Đảo Chiêu"] = "LoopSkills",
    ["Mẫu Chuỗi Chiêu (Preset)"] = "LoopSkills",
    ["Giữ Đúng Thứ Tự Combo (Strict Order)"] = "LoopStrictOrder",
    ["Chiêu Hồi Máu / Cứu Nguy"] = "EmergencyHealSkill",
    ["Kích Hoạt Hồi Máu Khi HP Dưới"] = "EmergencyHealHp",
    ["Thời Gian Chờ Ra Chiêu"] = "SkillEffectDelay",
    ["Tự Động Nhận Diện Hết Hiệu Ứng"] = "SmartEffectAutoDetect",

    -- Auto Luyện Chiêu
    ["Bật Auto Luyện Chiêu"] = "AutoTrainSkill",
    ["Chọn Chiêu Cần Luyện"] = "TrainSkill",
    ["Nhịp Chờ Xuất Chiêu (Cancel Delay)"] = "TrainCancelDelay",
    ["Mục Tiêu Số Lần Dùng"] = "TrainTargetCount",

    -- Bán cá & Bảo vệ
    ["Tự Động Bán Cá Khi Đầy Túi"] = "AutoSell",
    ["Giãn Cách Bán Cá Tự Động"] = "SellInterval",
    ["Tự Động Khóa Cá Đột Biến (Mutations)"] = "AutoProtectMutations",
    ["Tự Động Gom Cá Nguyên Liệu (Crafting)"] = "MaterialFarming",
    ["Tự Động Khóa Cá Yêu Thích"] = "AutoFavouriteFish",
    ["Tên Loài Cá Cần Khóa"] = "FavouriteFishName",

    -- Tự trang bị
    ["Tự Động Trang Bị Cần Tốt Nhất"] = "AutoEquipBestRod",
    ["Tự Dùng Cần Tốt Nhất"] = "AutoEquipBestRod",
    ["Tự Động Trang Bị Mồi"] = "AutoEquipBestBait",
    ["Tự Dùng Mồi (Auto Bait)"] = "AutoEquipBestBait",
    ["Tự Dùng Mồi Tốt Nhất"] = "AutoEquipBestBait",
    ["Chọn Mồi Khi Câu Thường"] = "BaitChoiceNormal",
    ["Tự Đổi Mồi Khi Săn Boss"] = "AutoEquipBossBait",
    ["Chọn Mồi Săn Boss"] = "BaitChoiceBoss",
    ["Tự Động Trang Bị Pháp Bảo Tốt Nhất"] = "AutoEquipBestOrb",
    ["Tự Dùng Ngọc Tốt Nhất"] = "AutoEquipBestOrb",

    -- Săn Secret Boss
    ["Bật Chế Độ Săn Boss (Tự Quăng Cần & Lọc Cá)"] = "AutoHuntBoss",
    ["Bật Săn Secret Boss (Chat Sniper)"] = "AutoChatSecretBoss",
    ["Tự Động Săn Secret Boss Theo Chat"] = "AutoChatSecretBoss",
    ["Giật Cần Thả Lại (Fast Skip Cá Thường)"] = "FastSkipNonBoss",
    ["Bỏ Qua Cá Thường (Fast Skip)"] = "FastSkipNonBoss",
    ["Kiểm Tra Lực Cần (Power Check)"] = "SecretBossCheckPower",
    ["Chỉ Săn Khi Đủ Lực Cần (Power Check)"] = "SecretBossCheckPower",
    ["Tự Đổi Server Khi Hết Boss (Auto-Hop)"] = "AutoServerHopOnDespawn",
    ["Đổi Server Khi Hết Secret Boss"] = "AutoServerHopOnDespawn",
    ["Tự Về Vị Trí Farm Khi Hết Boss / Clear"] = "ReturnToHomeWhenClear",

    -- Tìm Server Thời Tiết
    ["Tự Động Tìm Server Thời Tiết"] = "AutoWeatherHop",
    ["Chọn Thời Tiết Cần Tìm"] = "TargetWeather",
    ["Tự Động Câu / Săn Boss Khi Tìm Thấy"] = "WeatherHopAutoFish",
    ["Gửi Webhook Khi Tìm Thấy Server"] = "WeatherHopAlertWebhook",

    -- Thần linh
    ["Tự Động Quét Trạng Thái Thần Linh"] = "AutoGodSpiritCheck",
    ["Tự Động Cầu Nguyện Thần Linh"] = "AutoPrayGodSpirit",
    ["Đổi Server Tìm Thần Linh"] = "AutoServerHopGod",
    ["Đổi Server Tìm Maoshan"] = "AutoServerHopMaoshan",
    ["Đổi Server Tìm Đạo Sĩ (Taoist)"] = "AutoServerHopTaoist",
    ["Đổi Server Tìm Taoist"] = "AutoServerHopTaoist",

    -- Nhiệm vụ & Gacha
    ["Tự Động Nộp Vé Nhiệm Vụ (Tickets)"] = "AutoTicketQuest",
    ["Chọn Độ Khó Vé Nhiệm Vụ"] = "TicketDifficulty",
    ["Chế Độ Nhiệm Vụ"] = "TicketQuestMode",
    ["Loại Mồi Làm Nhiệm Vụ 100 Mồi"] = "TicketBaitChoice",
    ["Chiêu Dùng Cho Nhiệm Vụ 100 Skill"] = "TicketSkillKey",
    ["Chiêu Giật Nhanh Cho 100 Con Cá"] = "TicketQuickSkill",
    ["Thời Gian Hồi Chiêu (Phút)"] = "TicketCooldownMinutes",
    ["Tự Bán Cá Khi Đầy Balo (Vé NV)"] = "TicketAutoSellFull",
    ["Tự Về Home Spot Khi Xong Nhiệm Vụ"] = "TicketReturnHomeWhenDone",
    ["Tự Động Quăng Cần Tại Home Spot"] = "TicketAutoCastAtHome",
    ["Nhận & Nộp Vé Từ Xa (Remote)"] = "TicketRemoteClaim",
    ["Tự Động Nhận Thưởng Hàng Ngày (Daily)"] = "AutoClaimDaily",
    ["Vòng Quay May Mắn (Auto Gacha)"] = "AutoGacha",
    ["Chọn Vòng Quay Gacha"] = "GachaBanner",
    ["Số Vé Mỗi Lần Quay"] = "GachaPullsPerAction",

    -- Quản Lý Độ Ưu Tiên (Priority Manager)
    ["Bật Quản Lý Độ Ưu Tiên"] = "PrioritySystemEnabled",
    ["Mẫu Phân Cấp (Preset)"] = "PriorityPreset",
    ["Ưu Tiên: 🎯 Săn Secret Boss"] = "Priority_SecretBoss",
    ["Ưu Tiên: 📜 Làm Vé Nhiệm Vụ"] = "Priority_TicketQuest",
    ["Ưu Tiên: ⛩️ Cúng Thần Linh"] = "Priority_GodSpirit",
    ["Ưu Tiên: ⚔️ Auto Luyện Chiêu"] = "Priority_TrainSkill",
    ["Ưu Tiên: 🎣 Treo Farm Thường"] = "Priority_NormalFarm",

    -- ESP & Thị giác
    ["ESP Thần Linh (God Spirit)"] = "ESP_GodSpirit",
    ["ESP Cần Câu Bí Mật"] = "ESP_SecretRod",
    ["ESP Thuyền Bè"] = "ESP_Boats",
    ["ESP Maoshan"] = "ESP_Maoshan",
    ["ESP Đạo Sĩ (Taoist)"] = "ESP_Taoist",
    ["ESP Trùm Boss"] = "ESP_Boss",
    ["ESP Người Chơi"] = "ESP_Players",
    ["Vòng Tròn Định Vị Cá"] = "FishRedRing",
    ["Hiện Cân Nặng & Đột Biến Trên Vòng Đỏ"] = "ShowFishWeightRing",
    ["Tầm Nhìn Xa (Xóa Mờ Map)"] = "ClearFarVision",
    ["Xóa Sương Mù & Mưa Bão"] = "NoFog",
    ["Sáng Màn Hình (Fullbright)"] = "Fullbright",
    ["Mức Độ Sáng (Fullbright)"] = "FullbrightLevel",
    ["Chống Lóa Thời Tiết (Anti-Glare)"] = "FullbrightAntiGlare",
    ["Chế Độ Giảm Lag (Low GFX)"] = "PerformanceMode",
    ["Ẩn Giao Diện Gốc Của Game"] = "HideGameUI",
    ["Ẩn Tên Mặc Định Người Chơi"] = "HideOverheadNames",

    -- Nhân vật
    ["Tăng Tốc Độ Chạy (Speed)"] = "WalkSpeedEnabled",
    ["Chỉnh Tốc Độ"] = "WalkSpeedValue",
    ["Bay Lượn Tự Do (Fly)"] = "FlyEnabled",
    ["Tốc Độ Bay"] = "FlySpeed",
    ["Nhảy Vô Hạn (Infinite Jump)"] = "InfiniteJump",
    ["Đi Trên Mặt Nước"] = "WalkOnWater",
    ["Khiên Nước Axit (Acid Shield)"] = "AcidWaterShield",
    ["Đi Xuyên Tường (Noclip)"] = "Noclip",
    ["Chống Văng Game (Anti-AFK)"] = "AntiAFK",
    ["Tự Động Kết Nối Lại"] = "AutoRejoin",

    -- Discord Webhook
    ["Webhook URL"] = "WebhookUrl",
    ["Bật Webhook"] = "WebhookEnabled",
    ["Thông Báo Bắt Được Boss"] = "WebhookNotifyBoss",
    ["Thông Báo Đạo Sĩ (Taoist & Maoshan)"] = "WebhookNotifyNPC",
    ["Báo Cáo Định Kỳ"] = "WebhookHourlyStats",
    ["Báo Cáo Định Kỳ (Mỗi 30 Phút)"] = "WebhookHourlyStats",
    ["Báo Cáo Tiến Độ Mỗi Giờ"] = "WebhookHourlyStats",
    ["Thông Báo Hoàn Thành Nhiệm Vụ Vé"] = "WebhookNotifyTicketQuest",
    ["Tần Suất Báo Cáo Định Kỳ"] = "WebhookStatsInterval",
    ["Tần Suất Gửi Báo Cáo"] = "WebhookStatsInterval",

    -- Telegram Bot
    ["Bật Telegram Bot"] = "TelegramEnabled",
    ["Telegram Bot Token"] = "TelegramBotToken",
    ["Telegram Chat ID"] = "TelegramChatId",
    ["Telegram Báo Boss"] = "TelegramNotifyBoss",
    ["Telegram Báo Đạo Sĩ"] = "TelegramNotifyNPC",

    -- ntfy Push (Thông Báo Về Điện Thoại)
    ["Bật ntfy Push"] = "NtfyEnabled",
    ["ntfy Topic"] = "NtfyTopic",
    ["Thông Báo Đổi Thời Tiết (ntfy)"] = "NtfyAlertWeatherChange",
    ["Thông Báo Tìm Server Thời Tiết (ntfy)"] = "NtfyAlertWeatherHop",
    ["Thông Báo Boss & NPC (ntfy)"] = "NtfyNotifyBoss",

    -- Săn Boss DPS Meter
    ["Hiện Bảng Sát Thương Boss (% HP)"] = "ShowBossDpsMeter"
}

local function GetAccountConfigDir()
    local accName = (LocalPlayer and LocalPlayer.Name) or "DefaultUser"
    local safeAcc = accName:gsub("[^%w_]", "")
    if #safeAcc == 0 then safeAcc = "DefaultUser" end
    return "Identical/HeavyweightFishing/Configs/" .. safeAcc
end

local function EnsureAccountConfigDir()
    if makefolder and isfolder then
        pcall(function()
            if not isfolder("Identical") then makefolder("Identical") end
            if not isfolder("Identical/HeavyweightFishing") then makefolder("Identical/HeavyweightFishing") end
            if not isfolder("Identical/HeavyweightFishing/Configs") then makefolder("Identical/HeavyweightFishing/Configs") end
            local accDir = GetAccountConfigDir()
            if not isfolder(accDir) then makefolder(accDir) end
        end)
    end
end

local function GetSavedConfigList()
    EnsureAccountConfigDir()
    local accDir = GetAccountConfigDir()
    local list = {}
    if listfiles and isfolder and isfolder(accDir) then
        local ok, files = pcall(function() return listfiles(accDir) end)
        if ok and files then
            for _, path in ipairs(files) do
                local fileName = path:match("([^/\\]+)%.json$")
                if fileName and #fileName > 0 then
                    table.insert(list, fileName)
                end
            end
        end
    end
    table.sort(list)
    return list
end

local function SaveAccountConfig(cfgName)
    if not cfgName or cfgName:gsub("%s+", "") == "" then
        return false, "Vui lòng nhập tên cấu hình!"
    end
    local safeName = cfgName:gsub("[^%w_%-%s]", ""):gsub("^%s+", ""):gsub("%s+$", "")
    if #safeName == 0 then
        return false, "Tên cấu hình không hợp lệ!"
    end

    EnsureAccountConfigDir()
    local accDir = GetAccountConfigDir()
    local filePath = accDir .. "/" .. safeName .. ".json"

    local data = {}
    for k, v in pairs(Config) do
        if typeof(v) == "EnumItem" then
            data[k] = {__enum = tostring(v)}
        else
            data[k] = v
        end
    end

    local ok, encoded = pcall(function() return HttpService:JSONEncode(data) end)
    if not ok or not encoded then
        return false, "Lỗi mã hóa dữ liệu cấu hình!"
    end

    if writefile then
        local wOk, err = pcall(function() writefile(filePath, encoded) end)
        if wOk then
            return true, safeName
        else
            return false, "Lỗi khi ghi file: " .. tostring(err)
        end
    else
        return false, "Executor của bạn không hỗ trợ hàm writefile!"
    end
end

local function LoadAccountConfig(cfgName)
    if not cfgName or cfgName == "" or cfgName == "Chưa có config nào" then
        return false, "Vui lòng chọn cấu hình cần nạp!"
    end
    EnsureAccountConfigDir()
    local accDir = GetAccountConfigDir()
    local filePath = accDir .. "/" .. cfgName .. ".json"

    if not isfile or not isfile(filePath) then
        return false, "File cấu hình không tồn tại!"
    end

    local ok, content = pcall(function() return readfile(filePath) end)
    if not ok or not content or #content == 0 then
        return false, "Không thể đọc nội dung file cấu hình!"
    end

    local decOk, decoded = pcall(function() return HttpService:JSONDecode(content) end)
    if not decOk or type(decoded) ~= "table" then
        return false, "File cấu hình bị lỗi định dạng!"
    end

    -- 1. Cập nhật vào bảng Config
    for k, v in pairs(decoded) do
        if type(v) == "table" and v.__enum then
            local enumType, enumName = v.__enum:match("Enum%.(%w+)%.(%w+)")
            if enumType and enumName and Enum[enumType] and Enum[enumType][enumName] then
                Config[k] = Enum[enumType][enumName]
            end
        else
            Config[k] = v
        end
    end

    -- 2. Đồng bộ hóa toàn bộ giao diện UI tương ứng
    for key, ctrl in pairs(UIControllers) do
        if Config[key] ~= nil and ctrl and ctrl.Set then
            pcall(function()
                ctrl.Set(Config[key], true)
            end)
        end
    end

    return true, cfgName
end

local function DeleteAccountConfig(cfgName)
    if not cfgName or cfgName == "" or cfgName == "Chưa có config nào" then
        return false, "Vui lòng chọn cấu hình cần xóa!"
    end
    EnsureAccountConfigDir()
    local accDir = GetAccountConfigDir()
    local filePath = accDir .. "/" .. cfgName .. ".json"

    if isfile and isfile(filePath) then
        if delfile then
            local ok = pcall(function() delfile(filePath) end)
            if ok then return true, cfgName end
        end
    end
    return false, "Không thể xóa file cấu hình!"
end

Config._essentialKeys = {
    -- Kỹ năng & Combo
    ["LoopSkills"] = true,
    ["SmartComboEnabled"] = true,
    ["ComboRotationMode"] = true,
    ["LoopStrictOrder"] = true,
    ["FastCastKey"] = true,
    ["CastSkillOnCast"] = true,
    ["CastSkillKey"] = true,
    ["AutoTrainSkill"] = true,
    ["AutoCastTrainSkill"] = true,
    ["TicketSkillKey"] = true,
    ["TicketQuickSkill"] = true,
    ["AutoEquipBestOrb"] = true,

    -- Mồi câu & Tùy chọn Farm
    ["AutoEquipBestBait"] = true,
    ["BaitChoiceNormal"] = true,
    ["BaitChoiceBoss"] = true,
    ["AutoEquipBossBait"] = true,

    -- Nhiệm vụ vé
    ["TicketDifficulty"] = true,
    ["TicketQuestMode"] = true,
    ["TicketBaitChoice"] = true,
    ["TicketCooldownMinutes"] = true,
    ["TicketAutoSellFull"] = true,
    ["TicketReturnHomeWhenDone"] = true,
    ["TicketAutoCastAtHome"] = true,
    ["TicketRemoteClaim"] = true,

    -- Nhân vật & Tiện ích
    ["WalkSpeedEnabled"] = true,
    ["WalkSpeedValue"] = true,
    ["FlyEnabled"] = true,
    ["FlySpeed"] = true,
    ["InfiniteJump"] = true,
    ["WalkOnWater"] = true,
    ["AcidWaterShield"] = true,
    ["AutoProtectMutations"] = true,
    ["UIKeybind"] = true,

    -- Thị giác & ESP
    ["FishRedRing"] = true,
    ["ShowFishWeightRing"] = true,
    ["Fullbright"] = true,
    ["ClearFarVision"] = true,
    ["NoFog"] = true,
    ["HideOverheadNames"] = true,

    -- Webhook
    ["WebhookUrl"] = true,
    ["WebhookEnabled"] = true,
    ["WebhookNotifyBoss"] = true,
    ["WebhookNotifyNPC"] = true,
    ["WebhookHourlyStats"] = true,

    -- Telegram Bot
    ["TelegramEnabled"] = true,
    ["TelegramBotToken"] = true,
    ["TelegramChatId"] = true,
    ["TelegramNotifyBoss"] = true,
    ["TelegramNotifyNPC"] = true,

    -- ntfy Push (Thông Báo Về Điện Thoại)
    ["NtfyEnabled"] = true,
    ["NtfyTopic"] = true,
    ["NtfyAlertWeatherChange"] = true,
    ["NtfyAlertWeatherHop"] = true,
    ["NtfyNotifyBoss"] = true,

    -- Săn Boss
    ["ShowBossDpsMeter"] = true,

    -- Thời tiết
    ["TargetWeather"] = true,
    ["WeatherHopAutoFish"] = true,
    ["WeatherHopAlertWebhook"] = true,
}

Config._saveEssential = function()
    EnsureAccountConfigDir()
    local accDir = GetAccountConfigDir()
    local filePath = accDir .. "/essential_config.json"

    local data = {}
    for k, _ in pairs(Config._essentialKeys) do
        local v = Config[k]
        if v ~= nil then
            if typeof(v) == "EnumItem" then
                data[k] = {__enum = tostring(v)}
            else
                data[k] = v
            end
        end
    end

    local ok, encoded = pcall(function() return HttpService:JSONEncode(data) end)
    if ok and encoded and writefile then
        pcall(function() writefile(filePath, encoded) end)
    end
end

Config._loadEssential = function()
    EnsureAccountConfigDir()
    local accDir = GetAccountConfigDir()
    local filePath = accDir .. "/essential_config.json"

    if not isfile or not isfile(filePath) then return false end
    local ok, content = pcall(function() return readfile(filePath) end)
    if not ok or not content or #content == 0 then return false end
    local decOk, decoded = pcall(function() return HttpService:JSONDecode(content) end)
    if not decOk or type(decoded) ~= "table" then return false end

    for k, v in pairs(decoded) do
        if Config._essentialKeys[k] and Config[k] ~= nil then
            if type(v) == "table" and v.__enum then
                local enumType, enumName = v.__enum:match("Enum%.(%w+)%.(%w+)")
                if enumType and enumName and Enum[enumType] and Enum[enumType][enumName] then
                    Config[k] = Enum[enumType][enumName]
                end
            else
                Config[k] = v
            end
        end
    end

    -- Đảm bảo giữ nguyên các kỹ năng combo (Z, X, C, V)

    return true
end

Config._autoSaveTimer = nil
-- [AUTO SAVE ĐÃ TẮT] Người dùng không muốn tự động lưu config ra file máy
Config._triggerAutoSave = function(delaySec)
    -- Tắt hoàn toàn, không ghi file nào ra máy
end

-- [CONFIG TỰ LƯU ĐÃ TẮT] Người dùng chọn không dùng auto load config từ máy
-- pcall(function()
--     if Config._loadEssential() then
--         task.spawn(function()
--             task.wait(1.5)
--             ShowNotification("CẤU HÌNH TỰ ĐỘNG", "Đã nạp cài đặt thiết yếu (Skill, Combo, Tiện ích) từ máy!", "SUCCESS", 5)
--         end)
--     end
-- end)

-- ============================================================
-- SMART COMBO PERSISTENCE (HeavyweightFishing_SmartCombo.json)
-- Lưu/nạp trạng thái Combo Kỹ Năng Thông Minh tự động mỗi khi thay đổi.
-- ============================================================
local SMART_COMBO_FILE = "HeavyweightFishing_SmartCombo.json"
local SMART_COMBO_KEYS = {
    "SmartComboEnabled",
    "FishHpThreshold",
    "QuickCatchSkill",
    "OpenerSkill",
    "OpenerMaxCount",
    "LoopSkills",
    "LoopStrictOrder",
    "EmergencyHealSkill",
    "EmergencyHealHp",
    "SkillEffectDelay",
    "SmartEffectAutoDetect",
}

local _smartComboSavePending = false
local function SaveSmartCombo()
    if not writefile then return end
    -- Debounce: ghi file sau 0.3s nếu liên tục thay đổi (tránh ghi quá nhiều lần)
    if _smartComboSavePending then return end
    _smartComboSavePending = true
    task.delay(0.3, function()
        _smartComboSavePending = false
        local data = {}
        for _, k in ipairs(SMART_COMBO_KEYS) do
            data[k] = Config[k]
        end
        local ok, encoded = pcall(function() return HttpService:JSONEncode(data) end)
        if ok and encoded then
            pcall(function() writefile(SMART_COMBO_FILE, encoded) end)
        end
    end)
end

local function LoadSmartComboAndSyncUI()
    if not isfile or not isfile(SMART_COMBO_FILE) or not readfile then return end
    local ok, content = pcall(function() return readfile(SMART_COMBO_FILE) end)
    if not ok or not content or #content == 0 then return end
    local decOk, data = pcall(function() return HttpService:JSONDecode(content) end)
    if not decOk or type(data) ~= "table" then return end

    -- Bước 1: Nạp vào Config (không qua UI)
    local changed = false
    for _, k in ipairs(SMART_COMBO_KEYS) do
        if data[k] ~= nil then
            Config[k] = data[k]
            changed = true
        end
    end

    if not changed then return end

    -- Bước 2: Đồng bộ UI (dùng skipCallback=true để tránh kích hoạt lại SaveSmartCombo)
    task.spawn(function()
        task.wait(0.1) -- đảm bảo UIControllers đã khởi tạo xong
        local syncKeys = {
            "SmartComboEnabled", "FishHpThreshold", "QuickCatchSkill",
            "OpenerSkill", "OpenerMaxCount", "LoopStrictOrder",
            "EmergencyHealSkill", "EmergencyHealHp", "SkillEffectDelay",
            "SmartEffectAutoDetect",
        }
        for _, k in ipairs(syncKeys) do
            local ctrl = UIControllers[k]
            if ctrl and ctrl.Set and Config[k] ~= nil then
                pcall(function() ctrl.Set(Config[k], true) end)
            end
        end
        -- LoopSkills cần sync qua applyComboChange vì nó quản lý cả custom input + preset dropdown + preview
        -- Nhưng applyComboChange là local, nên ta sync UIControllers["LoopSkills"] nếu tồn tại
        -- (Dropdown preset và text input tự cập nhật nếu UIControllers đã map)
        local loopCtrl = UIControllers["LoopSkills"]
        if loopCtrl and loopCtrl.Set and Config.LoopSkills ~= nil then
            pcall(function() loopCtrl.Set(Config.LoopSkills, true) end)
        end
    end)
end

-- ============================================================
-- BOSS TARGETS PERSISTENCE (HeavyweightFishing_BossTargets.json)
-- Lưu/nạp trạng thái bật/tắt từng Secret Boss tự động.
-- ============================================================
local BOSS_TARGETS_FILE = "HeavyweightFishing_BossTargets.json"

local _bossTargetsSavePending = false
local function SaveBossTargets()
    if not writefile then return end
    if _bossTargetsSavePending then return end
    _bossTargetsSavePending = true
    task.delay(0.3, function()
        _bossTargetsSavePending = false
        local data = {}
        for k, v in pairs(Config.SecretBossTargets) do
            data[k] = v
        end
        local ok, encoded = pcall(function() return HttpService:JSONEncode(data) end)
        if ok and encoded then
            pcall(function() writefile(BOSS_TARGETS_FILE, encoded) end)
        end
    end)
end

local function LoadBossTargetsAndSyncUI()
    if not isfile or not isfile(BOSS_TARGETS_FILE) or not readfile then return end
    local ok, content = pcall(function() return readfile(BOSS_TARGETS_FILE) end)
    if not ok or not content or #content == 0 then return end
    local decOk, data = pcall(function() return HttpService:JSONDecode(content) end)
    if not decOk or type(data) ~= "table" then return end

    -- Bước 1: Nạp vào Config.SecretBossTargets
    for k, v in pairs(data) do
        if Config.SecretBossTargets[k] ~= nil then
            Config.SecretBossTargets[k] = v
        end
    end

    -- Bước 2: Đồng bộ UI toggle từng boss (skipCallback=true tránh lưu lại ngay)
    task.spawn(function()
        task.wait(0.1)
        for bName, val in pairs(Config.SecretBossTargets) do
            if bossTogglesMap and bossTogglesMap[bName] and bossTogglesMap[bName].Set then
                pcall(function() bossTogglesMap[bName].Set(val, true) end)
            end
        end
    end)
end

-- ============================================================
-- NOTIFICATIONS PERSISTENCE (HeavyweightFishing_Notifications.json)
-- Tự động lưu/nạp Webhook, Telegram, ntfy Topic chống mất khi kill script
-- ============================================================
Config._notificationsFile = "HeavyweightFishing_Notifications.json"
Config._notificationKeys = {
    "WebhookUrl", "WebhookEnabled", "WebhookNotifyBoss", "WebhookNotifyNPC", "WebhookHourlyStats", "WebhookNotifyTicketQuest", "WebhookStatsInterval",
    "TelegramEnabled", "TelegramBotToken", "TelegramChatId", "TelegramNotifyBoss", "TelegramNotifyNPC",
    "NtfyEnabled", "NtfyTopic", "NtfyAlertWeatherChange", "NtfyAlertWeatherHop", "NtfyNotifyBoss"
}

Config._notifSavePending = false
function SaveNotificationsConfig()
    if not writefile then return end
    if Config._notifSavePending then return end
    Config._notifSavePending = true
    task.delay(0.3, function()
        Config._notifSavePending = false
        local data = {}
        for _, k in ipairs(Config._notificationKeys) do
            data[k] = Config[k]
        end
        local ok, encoded = pcall(function() return HttpService:JSONEncode(data) end)
        if ok and encoded then
            pcall(function() writefile(Config._notificationsFile, encoded) end)
        end
    end)
end

function LoadNotificationsConfigAndSyncUI()
    if not isfile or not isfile(Config._notificationsFile) or not readfile then return end
    local ok, content = pcall(function() return readfile(Config._notificationsFile) end)
    if not ok or not content or #content == 0 then return end
    local decOk, data = pcall(function() return HttpService:JSONDecode(content) end)
    if not decOk or type(data) ~= "table" then return end

    -- Bước 1: Nạp trực tiếp vào Config
    for _, k in ipairs(Config._notificationKeys) do
        if data[k] ~= nil then
            Config[k] = data[k]
        end
    end

    -- Bước 2: Đồng bộ giao diện UIControllers
    task.spawn(function()
        task.wait(0.2)
        for _, k in ipairs(Config._notificationKeys) do
            local ctrl = UIControllers[k]
            if ctrl and ctrl.Set and Config[k] ~= nil then
                pcall(function() ctrl.Set(Config[k], true) end)
            end
        end
    end)
end

-- Nạp ngay lúc khởi động để Config có sẵn dữ liệu trước khi vẽ UI
pcall(function()
    if isfile and isfile("HeavyweightFishing_Notifications.json") and readfile then
        local ok, c = pcall(function() return readfile("HeavyweightFishing_Notifications.json") end)
        if ok and c and #c > 0 then
            local decOk, d = pcall(function() return HttpService:JSONDecode(c) end)
            if decOk and type(d) == "table" then
                for _, k in ipairs(Config._notificationKeys) do
                    if d[k] ~= nil then Config[k] = d[k] end
                end
            end
        end
    end
end)

local Colors = {
    Background       = Color3.fromRGB(15, 12, 22),
    SidebarBg        = Color3.fromRGB(11, 9, 17),
    BorderPurple     = Color3.fromRGB(168, 85, 247),
    BorderSubtle     = Color3.fromRGB(45, 33, 66),
    Divider          = Color3.fromRGB(36, 26, 54),

    PurplePrimary    = Color3.fromRGB(216, 160, 255),
    PurpleAccent     = Color3.fromRGB(168, 85, 247),
    PurpleMuted      = Color3.fromRGB(147, 112, 196),
    PurpleDark       = Color3.fromRGB(72, 45, 107),
    PurpleGlow       = Color3.fromRGB(192, 132, 252),

    RowNormal        = Color3.fromRGB(20, 16, 30),
    RowHover         = Color3.fromRGB(30, 22, 46),
    ControlBg        = Color3.fromRGB(28, 20, 44),
    InputBg          = Color3.fromRGB(18, 14, 26),

    TextWhite        = Color3.fromRGB(245, 243, 255),
    TextSubtle       = Color3.fromRGB(168, 150, 200),
    TextMuted        = Color3.fromRGB(110, 95, 138),

    AccentGreen      = Color3.fromRGB(52, 211, 153),
    AccentRed        = Color3.fromRGB(248, 113, 113),
    AccentOrange     = Color3.fromRGB(251, 146, 60),
    AccentYellow     = Color3.fromRGB(250, 204, 21),
    AccentBlue       = Color3.fromRGB(96, 165, 250),
    DropdownSelected = Color3.fromRGB(36, 26, 56)
}

local function getGuiParent()
    if gethui then
        local ok, h = pcall(gethui)
        if ok and h then return h end
    end
    local canParentCoreGui = pcall(function()
        local test = Instance.new("Folder")
        test.Parent = CoreGui
        test:Destroy()
    end)
    if canParentCoreGui then
        return CoreGui
    end
    local pg = LocalPlayer and (LocalPlayer:FindFirstChild("PlayerGui") or LocalPlayer:WaitForChild("PlayerGui", 5))
    if pg then return pg end
    return CoreGui
end

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "IdenticalHeavyweightFishing"
screenGui.ResetOnSpawn = false
screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

pcall(function()
    if syn and syn.protect_gui then
        syn.protect_gui(screenGui)
    end
end)

local parented = pcall(function()
    screenGui.Parent = getGuiParent()
end)
if not parented then
    pcall(function()
        screenGui.Parent = LocalPlayer:WaitForChild("PlayerGui", 5)
    end)
end
table.insert(cleanUpInstances, screenGui)

local function UnloadScript()
    isRunning = false
    for _, conn in ipairs(activeConnections) do pcall(function() conn:Disconnect() end) end
    table.clear(activeConnections)
    
    for _, inst in ipairs(cleanUpInstances) do
        pcall(function()
            if inst and inst.Parent then inst:Destroy() end
        end)
    end
    table.clear(cleanUpInstances)
    
    pcall(function()
        local leftover = Workspace:FindFirstChild("IdenticalESP")
        if leftover then leftover:Destroy() end
    end)

    pcall(function()
        local hum = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
        if hum then hum.WalkSpeed = 16 end
        Lighting.FogEnd = 100000
        Lighting.Brightness = 2
        Lighting.ClockTime = 14
        Lighting.Ambient = Color3.fromRGB(70, 70, 70)
        Lighting.OutdoorAmbient = Color3.fromRGB(70, 70, 70)
        Lighting.GlobalShadows = true
    end)
    
    CleanOldInstances()
    
    if globalEnv then
        globalEnv.HeavyweightFishingKill = nil
        globalEnv.IdenticalHeavyweightFishingUnload = nil
    end
end

if globalEnv then
    globalEnv.HeavyweightFishingKill = UnloadScript
    globalEnv.IdenticalHeavyweightFishingUnload = UnloadScript
end

local notifContainer = Instance.new("Frame")
notifContainer.Name = "Notifications"
notifContainer.Size = UDim2.new(0, 300, 1, -40)
notifContainer.Position = UDim2.new(1, -315, 0, 20)
notifContainer.BackgroundTransparency = 1
notifContainer.ZIndex = 1000
notifContainer.Parent = screenGui
table.insert(cleanUpInstances, notifContainer)

local notifLayout = Instance.new("UIListLayout")
notifLayout.SortOrder = Enum.SortOrder.LayoutOrder
notifLayout.VerticalAlignment = Enum.VerticalAlignment.Bottom
notifLayout.Padding = UDim.new(0, 8)
notifLayout.Parent = notifContainer

local function FormatWithSpaces(val)
    if not val then return "0" end
    local s = tostring(val)
    local prefix = ""
    if s:sub(1, 1) == "-" then
        prefix = "-"
        s = s:sub(2)
    elseif s:sub(1, 1) == "+" then
        prefix = "+"
        s = s:sub(2)
    end
    local intPart, decPart = s:match("^(%d+)(%.?.*)$")
    if not intPart then return prefix .. s end
    local formatted = intPart
    local k
    while true do
        formatted, k = string.gsub(formatted, "^(%d+)(%d%d%d)", "%1 %2")
        if k == 0 then break end
    end
    return prefix .. formatted .. (decPart or "")
end

local function ShowNotification(title, text, notifType, duration)
    if not isRunning then return end
    duration = duration or 3.5
    local accentColor = Colors.PurpleAccent
    if notifType == "SUCCESS" then accentColor = Colors.AccentGreen
    elseif notifType == "WARN" then accentColor = Colors.AccentYellow
    elseif notifType == "ERROR" then accentColor = Colors.AccentRed end

    local card = Instance.new("Frame")
    card.Size = UDim2.new(1, 0, 0, 60)
    card.BackgroundColor3 = Colors.Background
    card.BackgroundTransparency = 0.05
    card.BorderSizePixel = 0
    card.ClipsDescendants = true
    card.Parent = notifContainer

    local stroke = Instance.new("UIStroke"); stroke.Color = accentColor; stroke.Thickness = 1.2; stroke.Parent = card
    local corner = Instance.new("UICorner"); corner.CornerRadius = UDim.new(0, 6); corner.Parent = card
    local topBar = Instance.new("Frame"); topBar.Size = UDim2.new(1, 0, 0, 20); topBar.BackgroundTransparency = 1; topBar.Position = UDim2.new(0, 10, 0, 6); topBar.Parent = card
    local tLabel = Instance.new("TextLabel"); tLabel.Size = UDim2.new(1, -20, 1, 0); tLabel.BackgroundTransparency = 1; tLabel.Font = Enum.Font.GothamBold; tLabel.Text = title; tLabel.TextColor3 = Colors.PurplePrimary; tLabel.TextSize = 13; tLabel.TextXAlignment = Enum.TextXAlignment.Left; tLabel.Parent = topBar
    local mLabel = Instance.new("TextLabel"); mLabel.Size = UDim2.new(1, -20, 0, 26); mLabel.Position = UDim2.new(0, 10, 0, 26); mLabel.BackgroundTransparency = 1; mLabel.Font = Enum.Font.Gotham; mLabel.Text = text; mLabel.TextColor3 = Colors.TextSubtle; mLabel.TextSize = 11; mLabel.TextXAlignment = Enum.TextXAlignment.Left; mLabel.TextWrapped = true; mLabel.Parent = card

    task.delay(duration, function()
        if card and card.Parent then
            TweenService:Create(card, TweenInfo.new(0.35, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
                BackgroundTransparency = 1, Position = card.Position + UDim2.new(1, 20, 0, 0)
            }):Play()
            task.wait(0.35); card:Destroy()
        end
    end)
end

local ToggleUiVisibility

local floatingAvatar = Instance.new("ImageButton")
floatingAvatar.Name = "FloatingAvatar"
floatingAvatar.Size = UDim2.new(0, 48, 0, 48)
floatingAvatar.Position = UDim2.new(0, 20, 0.4, 0)
floatingAvatar.BackgroundColor3 = Colors.Background
floatingAvatar.BorderSizePixel = 0
floatingAvatar.Visible = false
floatingAvatar.ZIndex = 2000
floatingAvatar.Active = true
floatingAvatar.AutoButtonColor = false
floatingAvatar.Parent = screenGui
table.insert(cleanUpInstances, floatingAvatar)

do
    local faCorner = Instance.new("UICorner"); faCorner.CornerRadius = UDim.new(1, 0); faCorner.Parent = floatingAvatar
    local faStroke = Instance.new("UIStroke"); faStroke.Color = Colors.PurpleAccent; faStroke.Thickness = 2.2; faStroke.Parent = floatingAvatar
    
    local avatarImg = Instance.new("ImageLabel")
    avatarImg.Name = "AvatarImage"
    avatarImg.Size = UDim2.new(1, -6, 1, -6)
    avatarImg.Position = UDim2.new(0.5, 0, 0.5, 0)
    avatarImg.AnchorPoint = Vector2.new(0.5, 0.5)
    avatarImg.BackgroundTransparency = 1
    avatarImg.Image = "rbxthumb://type=AvatarHeadShot&id=" .. tostring(LocalPlayer.UserId) .. "&w=150&h=150"
    avatarImg.Parent = floatingAvatar
    Instance.new("UICorner", avatarImg).CornerRadius = UDim.new(1, 0)

    -- Status indicator dot
    local dot = Instance.new("Frame")
    dot.Size = UDim2.new(0, 11, 0, 11)
    dot.Position = UDim2.new(1, -11, 1, -11)
    dot.BackgroundColor3 = Colors.AccentGreen
    dot.BorderSizePixel = 0
    dot.Parent = floatingAvatar
    Instance.new("UICorner", dot).CornerRadius = UDim.new(1, 0)
    local dotStroke = Instance.new("UIStroke"); dotStroke.Color = Colors.Background; dotStroke.Thickness = 1.5; dotStroke.Parent = dot

    local faDragging, faDragInput, faDragStart, faStartPos = false, nil, nil, nil
    local dragMoved = false

    floatingAvatar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            faDragging = true
            dragMoved = false
            faDragStart = input.Position
            faStartPos = floatingAvatar.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    faDragging = false
                end
            end)
        end
    end)

    floatingAvatar.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            faDragInput = input
        end
    end)

    table.insert(activeConnections, UserInputService.InputChanged:Connect(function(input)
        if input == faDragInput and faDragging then
            local delta = input.Position - faDragStart
            if delta.Magnitude > 4 then
                dragMoved = true
            end
            floatingAvatar.Position = UDim2.new(faStartPos.X.Scale, faStartPos.X.Offset + delta.X, faStartPos.Y.Scale, faStartPos.Y.Offset + delta.Y)
        end
    end))

    floatingAvatar.MouseEnter:Connect(function()
        TweenService:Create(floatingAvatar, TweenInfo.new(0.15), {Size = UDim2.new(0, 52, 0, 52)}):Play()
        TweenService:Create(faStroke, TweenInfo.new(0.15), {Color = Colors.PurpleGlow, Thickness = 2.8}):Play()
    end)
    floatingAvatar.MouseLeave:Connect(function()
        TweenService:Create(floatingAvatar, TweenInfo.new(0.15), {Size = UDim2.new(0, 48, 0, 48)}):Play()
        TweenService:Create(faStroke, TweenInfo.new(0.15), {Color = Colors.PurpleAccent, Thickness = 2.2}):Play()
    end)

    floatingAvatar.MouseButton1Click:Connect(function()
        if not dragMoved and ToggleUiVisibility then
            ToggleUiVisibility()
        end
    end)
end

-- ===============================================================
-- ⚔️ BẢNG SÁT THƯƠNG SĂN BOSS (% HP DPS METER)
-- ===============================================================
do
local BossDpsTracker = {
    active = false,
    currentFish = nil,
    fishId = "",
    bossName = "Secret Boss",
    currentPhase = 1,
    phaseMaxHp = 0,
    totalBossMaxHp = 0,
    curHp = 0,
    lastHp = 0,
    totalDamage = 0,
    players = {},
    victoryUntil = 0,
    phaseTransitionUntil = 0
}

local recentPlayerAction = {}

local function IsPlayerAttacking(pName, char)
    if not char then return false end
    if pName == LocalPlayer.Name then
        local act = recentPlayerAction[pName]
        if act and (tick() - act) <= 0.45 then
            return true
        end
    end
    for _, att in ipairs({"UsingSkill", "SkillActive", "IsAttacking", "CastingSkill", "SkillLocked"}) do
        local val = char:GetAttribute(att)
        if val == true or (typeof(val) == "number" and val > 0) then
            return true
        end
    end
    local skillsFolder = char:FindFirstChild("Skills")
    if skillsFolder then
        for _, sk in ipairs(skillsFolder:GetChildren()) do
            local usk = sk:GetAttribute("UsingSkill")
            if usk and tonumber(usk) and tonumber(usk) > 0 then
                return true
            end
        end
    end
    local hum = char:FindFirstChildOfClass("Humanoid")
    local anim = hum and hum:FindFirstChildOfClass("Animator")
    if anim then
        local ok, tracks = pcall(function() return anim:GetPlayingAnimationTracks() end)
        if ok and tracks then
            for _, tr in ipairs(tracks) do
                if tr.IsPlaying and tr.Priority.Value >= Enum.AnimationPriority.Action.Value then
                    local name = tr.Name:lower()
                    if not name:find("fish") and not name:find("rod") and not name:find("reel") and not name:find("idle") and not name:find("walk") and not name:find("run") then
                        return true
                    end
                end
            end
        end
    end
    local lastAct = recentPlayerAction[pName]
    if lastAct and (tick() - lastAct) <= 0.45 then
        return true
    end
    return false
end

pcall(function()
    if Events and Events:FindFirstChild("UseSkill") then
        local oldUse = Events.UseSkill.FireServer
        Events.UseSkill.FireServer = function(self, ...)
            recentPlayerAction[LocalPlayer.Name] = tick()
            return oldUse(self, ...)
        end
    end
    if Events and Events:FindFirstChild("Slam") then
        local oldSlam = Events.Slam.FireServer
        Events.Slam.FireServer = function(self, ...)
            recentPlayerAction[LocalPlayer.Name] = tick()
            return oldSlam(self, ...)
        end
    end
end)

local bossDpsWidget = Instance.new("Frame")
bossDpsWidget.Name = "BossDpsWidget"
bossDpsWidget.Size = UDim2.new(0, 310, 0, 150)
bossDpsWidget.Position = UDim2.new(1, -330, 0.22, 0)
bossDpsWidget.BackgroundColor3 = Color3.fromRGB(18, 18, 26)
bossDpsWidget.BackgroundTransparency = 0.08
bossDpsWidget.BorderSizePixel = 0
bossDpsWidget.Visible = false
bossDpsWidget.ZIndex = 1500
bossDpsWidget.Parent = screenGui
table.insert(cleanUpInstances, bossDpsWidget)

do
    local dpsCorner = Instance.new("UICorner"); dpsCorner.CornerRadius = UDim.new(0, 10); dpsCorner.Parent = bossDpsWidget
    local dpsStroke = Instance.new("UIStroke"); dpsStroke.Color = Colors.PurpleAccent; dpsStroke.Thickness = 1.8; dpsStroke.Parent = bossDpsWidget
end

local dpsHeader = Instance.new("Frame")
dpsHeader.Name = "Header"
dpsHeader.Size = UDim2.new(1, 0, 0, 34)
dpsHeader.BackgroundColor3 = Color3.fromRGB(24, 24, 36)
dpsHeader.BorderSizePixel = 0
dpsHeader.Parent = bossDpsWidget
Instance.new("UICorner", dpsHeader).CornerRadius = UDim.new(0, 10)

local dpsHeaderPatch = Instance.new("Frame")
dpsHeaderPatch.Size = UDim2.new(1, 0, 0, 10); dpsHeaderPatch.Position = UDim2.new(0, 0, 1, -10)
dpsHeaderPatch.BackgroundColor3 = Color3.fromRGB(24, 24, 36); dpsHeaderPatch.BorderSizePixel = 0; dpsHeaderPatch.Parent = dpsHeader

local dpsTitle = Instance.new("TextLabel")
dpsTitle.Name = "Title"
dpsTitle.Size = UDim2.new(1, -20, 1, 0)
dpsTitle.Position = UDim2.new(0, 10, 0, 0)
dpsTitle.BackgroundTransparency = 1
dpsTitle.Text = "⚔️ SÁT THƯƠNG BOSS"
dpsTitle.TextColor3 = Color3.fromRGB(255, 215, 0)
dpsTitle.Font = Enum.Font.GothamBold
dpsTitle.TextSize = 13
dpsTitle.TextXAlignment = Enum.TextXAlignment.Left
dpsTitle.Parent = dpsHeader

do
    local dpsDragging, dpsDragStart, dpsStartPos
    dpsHeader.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dpsDragging = true
            dpsDragStart = input.Position
            dpsStartPos = bossDpsWidget.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dpsDragging = false
                end
            end)
        end
    end)
    dpsHeader.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            if dpsDragging and dpsDragStart and dpsStartPos then
                local delta = input.Position - dpsDragStart
                bossDpsWidget.Position = UDim2.new(dpsStartPos.X.Scale, dpsStartPos.X.Offset + delta.X, dpsStartPos.Y.Scale, dpsStartPos.Y.Offset + delta.Y)
            end
        end
    end)
end

local dpsBossInfo = Instance.new("Frame")
dpsBossInfo.Name = "BossInfo"
dpsBossInfo.Size = UDim2.new(1, -20, 0, 26)
dpsBossInfo.Position = UDim2.new(0, 10, 0, 36)
dpsBossInfo.BackgroundTransparency = 1
dpsBossInfo.Parent = bossDpsWidget

local dpsBossHpLabel = Instance.new("TextLabel")
dpsBossHpLabel.Name = "BossHpLabel"
dpsBossHpLabel.Size = UDim2.new(1, 0, 0, 16)
dpsBossHpLabel.BackgroundTransparency = 1
dpsBossHpLabel.Text = "Đang tìm Boss..."
dpsBossHpLabel.TextColor3 = Color3.fromRGB(230, 230, 255)
dpsBossHpLabel.Font = Enum.Font.GothamMedium
dpsBossHpLabel.TextSize = 11
dpsBossHpLabel.TextXAlignment = Enum.TextXAlignment.Left
dpsBossHpLabel.Parent = dpsBossInfo

local dpsHpBarBg = Instance.new("Frame")
dpsHpBarBg.Name = "HpBarBg"
dpsHpBarBg.Size = UDim2.new(1, 0, 0, 5)
dpsHpBarBg.Position = UDim2.new(0, 0, 0, 18)
dpsHpBarBg.BackgroundColor3 = Color3.fromRGB(40, 40, 58)
dpsHpBarBg.BorderSizePixel = 0
dpsHpBarBg.Parent = dpsBossInfo
Instance.new("UICorner", dpsHpBarBg).CornerRadius = UDim.new(1, 0)

local dpsHpBarFill = Instance.new("Frame")
dpsHpBarFill.Name = "Fill"
dpsHpBarFill.Size = UDim2.new(1, 0, 1, 0)
dpsHpBarFill.BackgroundColor3 = Color3.fromRGB(255, 65, 85)
dpsHpBarFill.BorderSizePixel = 0
dpsHpBarFill.Parent = dpsHpBarBg
Instance.new("UICorner", dpsHpBarFill).CornerRadius = UDim.new(1, 0)

local dpsListContainer = Instance.new("Frame")
dpsListContainer.Name = "PlayerList"
dpsListContainer.Size = UDim2.new(1, -20, 0, 80)
dpsListContainer.Position = UDim2.new(0, 10, 0, 66)
dpsListContainer.BackgroundTransparency = 1
dpsListContainer.Parent = bossDpsWidget

local dpsListLayout = Instance.new("UIListLayout")
dpsListLayout.SortOrder = Enum.SortOrder.LayoutOrder
dpsListLayout.Padding = UDim.new(0, 4)
dpsListLayout.Parent = dpsListContainer

local dpsRowPool = {}
local dpsColors = {
    Color3.fromRGB(0, 255, 140),
    Color3.fromRGB(0, 210, 255),
    Color3.fromRGB(255, 170, 0),
    Color3.fromRGB(210, 130, 255),
    Color3.fromRGB(255, 100, 120),
}

local function GetOrCreateDpsRow(index)
    if dpsRowPool[index] then
        dpsRowPool[index].Visible = true
        return dpsRowPool[index]
    end

    local row = Instance.new("Frame")
    row.Name = "Row_" .. tostring(index)
    row.Size = UDim2.new(1, 0, 0, 28)
    row.BackgroundColor3 = Color3.fromRGB(25, 25, 38)
    row.BorderSizePixel = 0
    row.ClipsDescendants = true
    row.Parent = dpsListContainer
    Instance.new("UICorner", row).CornerRadius = UDim.new(0, 6)

    local rowFill = Instance.new("Frame")
    rowFill.Name = "Fill"
    rowFill.Size = UDim2.new(0, 0, 1, 0)
    rowFill.BackgroundColor3 = Color3.fromRGB(60, 60, 90)
    rowFill.BackgroundTransparency = 0.65
    rowFill.BorderSizePixel = 0
    rowFill.Parent = row
    Instance.new("UICorner", rowFill).CornerRadius = UDim.new(0, 6)

    local avatar = Instance.new("ImageLabel")
    avatar.Name = "Avatar"
    avatar.Size = UDim2.new(0, 22, 0, 22)
    avatar.Position = UDim2.new(0, 3, 0.5, -11)
    avatar.BackgroundTransparency = 1
    avatar.Parent = row
    Instance.new("UICorner", avatar).CornerRadius = UDim.new(1, 0)

    local nameLbl = Instance.new("TextLabel")
    nameLbl.Name = "NameLabel"
    nameLbl.Size = UDim2.new(0.55, -30, 1, 0)
    nameLbl.Position = UDim2.new(0, 30, 0, 0)
    nameLbl.BackgroundTransparency = 1
    nameLbl.Font = Enum.Font.GothamBold
    nameLbl.TextSize = 10.5
    nameLbl.TextColor3 = Color3.fromRGB(240, 240, 255)
    nameLbl.TextXAlignment = Enum.TextXAlignment.Left
    nameLbl.TextTruncate = Enum.TextTruncate.AtEnd
    nameLbl.Parent = row

    local dmgLbl = Instance.new("TextLabel")
    dmgLbl.Name = "DmgLabel"
    dmgLbl.Size = UDim2.new(0.45, -6, 1, 0)
    dmgLbl.Position = UDim2.new(0.55, 0, 0, 0)
    dmgLbl.BackgroundTransparency = 1
    dmgLbl.Font = Enum.Font.GothamBold
    dmgLbl.TextSize = 10.5
    dmgLbl.TextColor3 = Color3.fromRGB(0, 255, 140)
    dmgLbl.TextXAlignment = Enum.TextXAlignment.Right
    dmgLbl.Parent = row

    dpsRowPool[index] = row
    return row
end

local function UpdateBossDpsWidget()
    if not Config.ShowBossDpsMeter then
        bossDpsWidget.Visible = false
        return
    end

    local now = tick()
    local fish = nil
    local fishesFolder = Workspace:FindFirstChild("Fishes")

    if fishesFolder then
        local myFishId = LocalPlayer:GetAttribute("FishID")
        if myFishId and myFishId ~= "" then
            local myF = fishesFolder:FindFirstChild(myFishId)
            if myF and myF:GetAttribute("Boss") == true then
                fish = myF
            end
        end

        if not fish then
            local myPos = (LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") and LocalPlayer.Character.HumanoidRootPart.Position) or Vector3.zero
            local nearestDist = 999999
            for _, f in ipairs(fishesFolder:GetChildren()) do
                if f:GetAttribute("Boss") == true and typeof(f.Value) == "number" and f.Value > 0 then
                    local fPos = nil
                    if f:FindFirstChild("Buoy") and f.Buoy:IsA("BasePart") then
                        fPos = f.Buoy.Position
                    elseif f:FindFirstChild("Model") and f.Model:IsA("Model") then
                        fPos = f.Model:GetPivot().Position
                    end
                    local dist = fPos and (fPos - myPos).Magnitude or 100
                    if dist < nearestDist and dist <= 250 then
                        nearestDist = dist
                        fish = f
                    end
                end
            end
        end
    end

    if fish and fish.Parent then
        local bName = fish:GetAttribute("FishName") or "Secret Boss"
        local phaseMax = tonumber(fish:GetAttribute("MaxHealth")) or 10000
        local curHp = typeof(fish.Value) == "number" and fish.Value or 0
        if phaseMax <= 0 then phaseMax = math.max(curHp, 1) end

        -- Khởi tạo Boss mới
        if BossDpsTracker.currentFish ~= fish then
            BossDpsTracker.currentFish = fish
            BossDpsTracker.fishId = fish.Name
            BossDpsTracker.bossName = bName
            BossDpsTracker.currentPhase = 1
            BossDpsTracker.phaseMaxHp = phaseMax
            BossDpsTracker.totalBossMaxHp = phaseMax
            BossDpsTracker.curHp = curHp
            BossDpsTracker.lastHp = curHp
            BossDpsTracker.totalDamage = 0
            BossDpsTracker.players = {}
            BossDpsTracker.victoryUntil = 0
            BossDpsTracker.phaseTransitionUntil = 0
            BossDpsTracker.active = true
        end

        -- Nhận diện chuyển mạng (Boss hồi máu sang mạng tiếp theo)
        if curHp > (BossDpsTracker.lastHp + 200) and (BossDpsTracker.lastHp <= 200 or BossDpsTracker.phaseTransitionUntil > 0) then
            BossDpsTracker.currentPhase = BossDpsTracker.currentPhase + 1
            BossDpsTracker.phaseMaxHp = phaseMax
            BossDpsTracker.totalBossMaxHp = BossDpsTracker.totalBossMaxHp + phaseMax
            BossDpsTracker.lastHp = curHp
            BossDpsTracker.curHp = curHp
            BossDpsTracker.phaseTransitionUntil = 0
        end

        local deltaHp = BossDpsTracker.lastHp - curHp
        if deltaHp > 0 and deltaHp < (phaseMax * 0.95) then
            BossDpsTracker.totalDamage = BossDpsTracker.totalDamage + deltaHp

            local activeParticipants = {}
            local myName = LocalPlayer.Name
            table.insert(activeParticipants, myName)

            local contribFolder = fish:FindFirstChild("PlayerContribution")
            if contribFolder then
                for _, c in ipairs(contribFolder:GetChildren()) do
                    if c.Name ~= myName and not table.find(activeParticipants, c.Name) then
                        table.insert(activeParticipants, c.Name)
                    end
                end
            end

            for _, c in ipairs(fish:GetChildren()) do
                if c.Name:find("_PlayerHealth") then
                    local uid = tonumber(c.Name:match("^(%d+)_PlayerHealth"))
                    if uid then
                        local pl = Players:GetPlayerByUserId(uid)
                        if pl and not table.find(activeParticipants, pl.Name) then
                            table.insert(activeParticipants, pl.Name)
                        end
                    end
                end
            end

            for _, pl in ipairs(Players:GetPlayers()) do
                if pl ~= LocalPlayer and pl.Character then
                    local isPlMinigame = pl.Character:GetAttribute("Minigame") == true or pl.Character:GetAttribute("Fishing") == true
                    if isPlMinigame and not table.find(activeParticipants, pl.Name) then
                        local hrp = pl.Character:FindFirstChild("HumanoidRootPart")
                        local myHrp = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
                        if hrp and myHrp and (hrp.Position - myHrp.Position).Magnitude <= 120 then
                            table.insert(activeParticipants, pl.Name)
                        end
                    end
                end
            end

            for _, pName in ipairs(activeParticipants) do
                if not BossDpsTracker.players[pName] then
                    local pl = Players:FindFirstChild(pName)
                    BossDpsTracker.players[pName] = {
                        name = pName,
                        displayName = pl and pl.DisplayName or pName,
                        userId = pl and pl.UserId or 0,
                        damage = 0,
                        lastHit = now
                    }
                end
            end

            local numParticipants = #activeParticipants
            if numParticipants <= 1 then
                BossDpsTracker.players[myName].damage = BossDpsTracker.players[myName].damage + deltaHp
                BossDpsTracker.players[myName].lastHit = now
            else
                -- Có 2 người trở lên tham gia! Phân loại đòn đánh chính xác
                local attackers = {}
                for _, pName in ipairs(activeParticipants) do
                    local pl = Players:FindFirstChild(pName)
                    local char = pl and pl.Character or (pName == myName and LocalPlayer.Character)
                    if IsPlayerAttacking(pName, char) then
                        table.insert(attackers, pName)
                        recentPlayerAction[pName] = now
                    end
                end

                -- Nếu là đòn burst sát thương lớn (chiêu thức hoặc slam >= 75 dmg)
                if deltaHp >= 75 and #attackers > 0 then
                    local sharePerAttacker = deltaHp / #attackers
                    for _, pName in ipairs(attackers) do
                        BossDpsTracker.players[pName].damage = BossDpsTracker.players[pName].damage + sharePerAttacker
                        BossDpsTracker.players[pName].lastHit = now
                    end
                else
                    -- Sát thương kéo dây câu liên tục (continuous rod pull DPS) hoặc không bắt được animation
                    local weights = {}
                    local totalWeight = 0
                    for _, pName in ipairs(activeParticipants) do
                        local w = 1.0
                        local pl = Players:FindFirstChild(pName)
                        local char = pl and pl.Character or (pName == myName and LocalPlayer.Character)
                        if char then
                            local stats = char:FindFirstChild("Stats")
                            local rp = stats and stats:FindFirstChild("RodPower")
                            if rp and tonumber(rp.Value) and tonumber(rp.Value) > 0 then
                                w = math.max(tonumber(rp.Value), 10)
                            end
                        end
                        weights[pName] = w
                        totalWeight = totalWeight + w
                    end

                    for _, pName in ipairs(activeParticipants) do
                        local share = deltaHp * (weights[pName] / totalWeight)
                        BossDpsTracker.players[pName].damage = BossDpsTracker.players[pName].damage + share
                        BossDpsTracker.players[pName].lastHit = now
                    end
                end
            end
        end

        BossDpsTracker.lastHp = curHp
        BossDpsTracker.curHp = curHp

        -- Xử lý hết máu 1 mạng
        local hasPhaseLeft = fish:GetAttribute("HasPhaseLeft") == true
        if curHp <= 5 then
            if hasPhaseLeft or fish.Parent ~= nil then
                if BossDpsTracker.phaseTransitionUntil == 0 then
                    BossDpsTracker.phaseTransitionUntil = now + 4.0
                end
            end
        end

        local hpPercent = math.clamp((curHp / phaseMax) * 100, 0, 100)
        if BossDpsTracker.phaseTransitionUntil > 0 and now < BossDpsTracker.phaseTransitionUntil then
            dpsTitle.Text = string.format("⚡ HẠ MẠNG %d! ĐANG QUA MẠNG TIẾP...", BossDpsTracker.currentPhase)
            dpsTitle.TextColor3 = Color3.fromRGB(255, 180, 0)
        else
            if BossDpsTracker.currentPhase > 1 or hasPhaseLeft then
                dpsTitle.Text = string.format("⚔️ SÁT THƯƠNG BOSS • MẠNG %d (%d%%)", BossDpsTracker.currentPhase, math.floor(hpPercent))
            else
                dpsTitle.Text = string.format("⚔️ SÁT THƯƠNG BOSS • %d%% HP", math.floor(hpPercent))
            end
            dpsTitle.TextColor3 = Color3.fromRGB(255, 215, 0)
        end

        dpsBossHpLabel.Text = string.format("%s [Mạng %d] • %s / %s HP", bName, BossDpsTracker.currentPhase, FormatWithSpaces(math.floor(curHp)), FormatWithSpaces(phaseMax))
        dpsHpBarFill.Size = UDim2.new(math.clamp(curHp / phaseMax, 0, 1), 0, 1, 0)

        local sortedList = {}
        for _, pData in pairs(BossDpsTracker.players) do
            table.insert(sortedList, pData)
        end
        table.sort(sortedList, function(a, b) return a.damage > b.damage end)

        local rowCount = #sortedList
        local visibleRows = math.min(rowCount, 5)
        local totalFightHp = math.max(BossDpsTracker.totalBossMaxHp, phaseMax, 1)

        for i = 1, math.max(#dpsRowPool, visibleRows) do
            if i <= visibleRows then
                local row = GetOrCreateDpsRow(i)
                local pData = sortedList[i]
                local pDmg = math.floor(pData.damage)
                local pPctOfBossHp = math.clamp((pData.damage / totalFightHp) * 100, 0, 100)
                local isMe = (pData.name == LocalPlayer.Name)

                local fill = row:FindFirstChild("Fill")
                local avatar = row:FindFirstChild("Avatar")
                local nameLbl = row:FindFirstChild("NameLabel")
                local dmgLbl = row:FindFirstChild("DmgLabel")

                if avatar and pData.userId > 0 then
                    avatar.Image = "rbxthumb://type=AvatarHeadShot&id=" .. tostring(pData.userId) .. "&w=100&h=100"
                end

                local rankIcons = {"🥇", "🥈", "🥉", "⚔️", "🗡️"}
                local rankIcon = rankIcons[i] or "•"
                local nameText = (isMe and "Bạn (" .. pData.displayName .. ")" or pData.displayName)
                nameLbl.Text = string.format("%s %s", rankIcon, nameText)
                nameLbl.TextColor3 = isMe and Color3.fromRGB(255, 230, 80) or Color3.fromRGB(230, 230, 255)

                dmgLbl.Text = string.format("%.1f%% (%s DMG)", pPctOfBossHp, FormatWithSpaces(pDmg))
                dmgLbl.TextColor3 = dpsColors[i] or Color3.fromRGB(0, 210, 255)

                local fillPct = (BossDpsTracker.totalDamage > 0) and math.clamp(pData.damage / BossDpsTracker.totalDamage, 0, 1) or 0
                if fill then
                    fill.Size = UDim2.new(fillPct, 0, 1, 0)
                    fill.BackgroundColor3 = dpsColors[i] or Color3.fromRGB(60, 60, 90)
                end
            elseif dpsRowPool[i] then
                dpsRowPool[i].Visible = false
            end
        end

        local targetHeight = 70 + (visibleRows * 32)
        bossDpsWidget.Size = UDim2.new(0, 310, 0, targetHeight)
        dpsListContainer.Size = UDim2.new(1, -20, 0, visibleRows * 32)
        bossDpsWidget.Visible = true

    else
        if BossDpsTracker.totalDamage > 0 and BossDpsTracker.victoryUntil == 0 and now > BossDpsTracker.phaseTransitionUntil then
            BossDpsTracker.victoryUntil = now + 7.0
        end

        if BossDpsTracker.victoryUntil > 0 and now < BossDpsTracker.victoryUntil then
            dpsTitle.Text = string.format("🏆 HẠ GỤC BOSS! (HOÀN THÀNH %d MẠNG)", BossDpsTracker.currentPhase)
            dpsTitle.TextColor3 = Color3.fromRGB(0, 255, 140)
            dpsBossHpLabel.Text = string.format("Bảng Tổng Kết Sát Thương Cả %d Mạng Của Boss:", BossDpsTracker.currentPhase)
            dpsHpBarFill.Size = UDim2.new(0, 0, 1, 0)
            bossDpsWidget.Visible = true
        else
            BossDpsTracker.active = false
            BossDpsTracker.currentFish = nil
            BossDpsTracker.victoryUntil = 0
            BossDpsTracker.phaseTransitionUntil = 0
            bossDpsWidget.Visible = false
        end
    end
end

task.spawn(function()
    while isRunning do
        pcall(UpdateBossDpsWidget)
        task.wait(0.15)
    end
end)
end

local mainFrame = Instance.new("Frame")
mainFrame.Name = "MainFrame"
mainFrame.Size = UDim2.new(0, 700, 0, 480)
mainFrame.Position = UDim2.new(0.5, -350, 0.5, -240)
mainFrame.BackgroundColor3 = Colors.Background
mainFrame.BorderSizePixel = 0
mainFrame.ClipsDescendants = true
mainFrame.Parent = screenGui
table.insert(cleanUpInstances, mainFrame)

do
    local mc = Instance.new("UICorner"); mc.CornerRadius = UDim.new(0, 8); mc.Parent = mainFrame
    local ms = Instance.new("UIStroke"); ms.Color = Colors.BorderPurple; ms.Thickness = 1.5; ms.Parent = mainFrame
end

local titleBar = Instance.new("Frame")
titleBar.Name = "TitleBar"
titleBar.Size = UDim2.new(1, 0, 0, 38)
titleBar.BackgroundColor3 = Colors.SidebarBg
titleBar.BorderSizePixel = 0
titleBar.Parent = mainFrame

do
    local tc = Instance.new("UICorner"); tc.CornerRadius = UDim.new(0, 8); tc.Parent = titleBar
    local tbf = Instance.new("Frame"); tbf.Size = UDim2.new(1, 0, 0, 10); tbf.Position = UDim2.new(0, 0, 1, -10); tbf.BackgroundColor3 = Colors.SidebarBg; tbf.BorderSizePixel = 0; tbf.Parent = titleBar
    local tdiv = Instance.new("Frame"); tdiv.Size = UDim2.new(1, 0, 0, 1); tdiv.Position = UDim2.new(0, 0, 1, -1); tdiv.BackgroundColor3 = Colors.Divider; tdiv.BorderSizePixel = 0; tdiv.Parent = titleBar
end

do
    local cc = Instance.new("Frame"); cc.Size = UDim2.new(0, 16, 0, 16); cc.Position = UDim2.new(0, 14, 0.5, -8); cc.BackgroundTransparency = 1; cc.ClipsDescendants = true; cc.Parent = titleBar
    local co = Instance.new("Frame"); co.Size = UDim2.new(0, 16, 0, 16); co.BackgroundColor3 = Colors.PurpleAccent; co.BorderSizePixel = 0; co.Parent = cc
    Instance.new("UICorner", co).CornerRadius = UDim.new(1, 0)
    local cut = Instance.new("Frame"); cut.Size = UDim2.new(0, 13, 0, 13); cut.Position = UDim2.new(0, 4, 0, -2); cut.BackgroundColor3 = Colors.SidebarBg; cut.BorderSizePixel = 0; cut.Parent = co
    Instance.new("UICorner", cut).CornerRadius = UDim.new(1, 0)
end

local brandTitle = Instance.new("TextLabel")
brandTitle.Size = UDim2.new(0, 92, 1, 0); brandTitle.Position = UDim2.new(0, 36, 0, 0)
brandTitle.BackgroundTransparency = 1; brandTitle.Font = Enum.Font.GothamBold
brandTitle.Text = "CÂU CÁ PRO"; brandTitle.TextColor3 = Colors.PurplePrimary
brandTitle.TextSize = 14; brandTitle.TextXAlignment = Enum.TextXAlignment.Left
brandTitle.Parent = titleBar

local commitBadge = Instance.new("TextLabel")
commitBadge.AutomaticSize = Enum.AutomaticSize.X
commitBadge.Size = UDim2.new(0, 0, 0, 18); commitBadge.Position = UDim2.new(0, 132, 0.5, -9)
local bPad = Instance.new("UIPadding", commitBadge); bPad.PaddingLeft = UDim.new(0, 6); bPad.PaddingRight = UDim.new(0, 6)
commitBadge.BackgroundColor3 = Color3.fromRGB(30, 22, 48)
commitBadge.Font = Enum.Font.Code
commitBadge.Text = "#" .. tostring(SCRIPT_BUILD_COMMIT)
commitBadge.TextColor3 = Color3.fromRGB(190, 150, 255)
commitBadge.TextSize = 10
commitBadge.Parent = titleBar
Instance.new("UICorner", commitBadge).CornerRadius = UDim.new(0, 4)
local cStroke = Instance.new("UIStroke", commitBadge)
cStroke.Color = Colors.PurpleAccent
cStroke.Thickness = 1

local gameSubtitle = Instance.new("TextLabel")
gameSubtitle.Size = UDim2.new(0, 220, 1, 0); gameSubtitle.Position = UDim2.new(0, 208, 0, 0)
gameSubtitle.BackgroundTransparency = 1; gameSubtitle.Font = Enum.Font.Gotham
gameSubtitle.Text = "HEAVYWEIGHT FISHING | BẢN VIỆT HOÁ"; gameSubtitle.TextColor3 = Colors.PurpleMuted
gameSubtitle.TextSize = 10; gameSubtitle.TextXAlignment = Enum.TextXAlignment.Left
gameSubtitle.Parent = titleBar

local winControls = Instance.new("Frame"); winControls.Size = UDim2.new(0, 95, 1, 0); winControls.Position = UDim2.new(1, -100, 0, 0); winControls.BackgroundTransparency = 1; winControls.Parent = titleBar
local minBtn = Instance.new("TextButton"); minBtn.Size = UDim2.new(0, 24, 0, 24); minBtn.Position = UDim2.new(0, 4, 0.5, -12); minBtn.BackgroundColor3 = Colors.ControlBg; minBtn.Font = Enum.Font.GothamBold; minBtn.Text = "[-]"; minBtn.TextColor3 = Colors.PurplePrimary; minBtn.TextSize = 11; minBtn.BorderSizePixel = 0; minBtn.Parent = winControls
Instance.new("UICorner", minBtn).CornerRadius = UDim.new(0, 4)
local closeBtn = Instance.new("TextButton"); closeBtn.Size = UDim2.new(0, 24, 0, 24); closeBtn.Position = UDim2.new(0, 32, 0.5, -12); closeBtn.BackgroundColor3 = Colors.ControlBg; closeBtn.Font = Enum.Font.GothamBold; closeBtn.Text = "[X]"; closeBtn.TextColor3 = Colors.TextWhite; closeBtn.TextSize = 11; closeBtn.BorderSizePixel = 0; closeBtn.Parent = winControls
Instance.new("UICorner", closeBtn).CornerRadius = UDim.new(0, 4)
local killBtn = Instance.new("TextButton"); killBtn.Size = UDim2.new(0, 32, 0, 24); killBtn.Position = UDim2.new(0, 60, 0.5, -12); killBtn.BackgroundColor3 = Color3.fromRGB(45, 20, 25); killBtn.Font = Enum.Font.GothamBold; killBtn.Text = "KILL"; killBtn.TextColor3 = Colors.AccentRed; killBtn.TextSize = 9; killBtn.BorderSizePixel = 0; killBtn.Parent = winControls
Instance.new("UICorner", killBtn).CornerRadius = UDim.new(0, 4)
local killStroke = Instance.new("UIStroke"); killStroke.Color = Colors.AccentRed; killStroke.Thickness = 1; killStroke.Parent = killBtn

do
    local dragging, dragInput, dragStart, startPos = false, nil, nil, nil
    titleBar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true; dragStart = input.Position; startPos = mainFrame.Position
            input.Changed:Connect(function() if input.UserInputState == Enum.UserInputState.End then dragging = false end end)
        end
    end)
    titleBar.InputChanged:Connect(function(input) if input.UserInputType == Enum.UserInputType.MouseMovement then dragInput = input end end)
    table.insert(activeConnections, UserInputService.InputChanged:Connect(function(input)
        if input == dragInput and dragging then
            local delta = input.Position - dragStart
            mainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end))
end

local bodyFrame = Instance.new("Frame"); bodyFrame.Name = "Body"; bodyFrame.Size = UDim2.new(1, 0, 1, -62); bodyFrame.Position = UDim2.new(0, 0, 0, 38); bodyFrame.BackgroundTransparency = 1; bodyFrame.Parent = mainFrame

local sidebar = Instance.new("Frame"); sidebar.Name = "Sidebar"; sidebar.Size = UDim2.new(0, 140, 1, 0); sidebar.BackgroundColor3 = Colors.SidebarBg; sidebar.BorderSizePixel = 0; sidebar.Parent = bodyFrame
do local d = Instance.new("Frame"); d.Size = UDim2.new(0, 1, 1, 0); d.Position = UDim2.new(1, -1, 0, 0); d.BackgroundColor3 = Colors.Divider; d.BorderSizePixel = 0; d.Parent = sidebar end

local searchBox = Instance.new("TextBox")
searchBox.Name = "SearchBar"; searchBox.Size = UDim2.new(1, -16, 0, 26); searchBox.Position = UDim2.new(0, 8, 0, 8)
searchBox.BackgroundColor3 = Colors.InputBg; searchBox.Font = Enum.Font.Gotham; searchBox.PlaceholderText = "Tìm kiếm tính năng..."
searchBox.PlaceholderColor3 = Colors.TextMuted; searchBox.Text = ""; searchBox.TextColor3 = Colors.TextWhite
searchBox.TextSize = 11; searchBox.TextXAlignment = Enum.TextXAlignment.Left; searchBox.BorderSizePixel = 0; searchBox.ClearTextOnFocus = false; searchBox.Parent = sidebar
do local p = Instance.new("UIPadding"); p.PaddingLeft = UDim.new(0, 8); p.Parent = searchBox end
Instance.new("UICorner", searchBox).CornerRadius = UDim.new(0, 4)

local rowSearchIndex = {}
searchBox:GetPropertyChangedSignal("Text"):Connect(function()
    local q = searchBox.Text:lower()
    for _, item in ipairs(rowSearchIndex) do
        if q == "" or item.query:find(q, 1, true) then
            item.frame.Visible = true
        else
            item.frame.Visible = false
        end
    end
end)

local navList = Instance.new("ScrollingFrame"); navList.Name = "NavList"; navList.Size = UDim2.new(1, 0, 1, -44); navList.Position = UDim2.new(0, 0, 0, 42); navList.BackgroundTransparency = 1; navList.BorderSizePixel = 0; navList.ScrollBarThickness = 2; navList.ScrollBarImageColor3 = Colors.BorderSubtle; navList.CanvasSize = UDim2.new(0, 0, 0, 0); navList.AutomaticCanvasSize = Enum.AutomaticSize.Y; navList.Parent = sidebar
do
    local nl = Instance.new("UIListLayout"); nl.SortOrder = Enum.SortOrder.LayoutOrder; nl.Padding = UDim.new(0, 4); nl.Parent = navList
    local np = Instance.new("UIPadding"); np.PaddingTop = UDim.new(0, 6); np.PaddingLeft = UDim.new(0, 8); np.PaddingRight = UDim.new(0, 8); np.Parent = navList
end

local contentArea = Instance.new("Frame"); contentArea.Name = "ContentArea"; contentArea.Size = UDim2.new(1, -140, 1, 0); contentArea.Position = UDim2.new(0, 140, 0, 0); contentArea.BackgroundTransparency = 1; contentArea.Parent = bodyFrame

local tabFrames = {}
local tabButtons = {}

local UpdateCrimsonBreamUI = nil

local function SwitchTab(tabName)
    for name, frame in pairs(tabFrames) do frame.Visible = (name == tabName) end
    for name, btn in pairs(tabButtons) do
        if name == tabName then
            TweenService:Create(btn, TweenInfo.new(0.15), {BackgroundColor3 = Colors.PurpleDark, TextColor3 = Colors.TextWhite}):Play()
            local pill = btn:FindFirstChild("ActivePill"); if pill then pill.Visible = true end
        else
            TweenService:Create(btn, TweenInfo.new(0.15), {BackgroundColor3 = Colors.SidebarBg, TextColor3 = Colors.TextSubtle}):Play()
            local pill = btn:FindFirstChild("ActivePill"); if pill then pill.Visible = false end
        end
    end
    if tabName == "Nhiệm Vụ" and ticketQuestState and ticketQuestState.ScanAndUpdateStatus then
        task.spawn(ticketQuestState.ScanAndUpdateStatus)
    end
    if tabName == "Quản Lý Cá" and UpdateCrimsonBreamUI then
        task.spawn(UpdateCrimsonBreamUI)
    end
end

local function CreateTab(name)
    local btn = Instance.new("TextButton"); btn.Name = "TabBtn_" .. name; btn.Size = UDim2.new(1, 0, 0, 30); btn.BackgroundColor3 = Colors.SidebarBg; btn.Font = Enum.Font.GothamBold; btn.Text = name; btn.TextColor3 = Colors.TextSubtle; btn.TextSize = 12; btn.TextXAlignment = Enum.TextXAlignment.Left; btn.BorderSizePixel = 0; btn.Parent = navList
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 4)
    do local p = Instance.new("UIPadding"); p.PaddingLeft = UDim.new(0, 12); p.Parent = btn end
    local pill = Instance.new("Frame"); pill.Name = "ActivePill"; pill.Size = UDim2.new(0, 3, 0, 16); pill.Position = UDim2.new(0, -9, 0.5, -8); pill.BackgroundColor3 = Colors.PurpleAccent; pill.BorderSizePixel = 0; pill.Visible = false; pill.Parent = btn
    Instance.new("UICorner", pill).CornerRadius = UDim.new(0, 2)
    btn.MouseButton1Click:Connect(function() SwitchTab(name) end)

    local page = Instance.new("ScrollingFrame"); page.Name = "TabPage_" .. name; page.Size = UDim2.new(1, 0, 1, 0); page.BackgroundTransparency = 1; page.BorderSizePixel = 0; page.ScrollBarThickness = 3; page.ScrollBarImageColor3 = Colors.BorderPurple; page.CanvasSize = UDim2.new(0, 0, 0, 0); page.AutomaticCanvasSize = Enum.AutomaticSize.Y; page.Visible = false; page.Parent = contentArea
    do
        local pl = Instance.new("UIListLayout"); pl.SortOrder = Enum.SortOrder.LayoutOrder; pl.Padding = UDim.new(0, 10); pl.Parent = page
        local pp = Instance.new("UIPadding"); pp.PaddingTop = UDim.new(0, 12); pp.PaddingBottom = UDim.new(0, 16); pp.PaddingLeft = UDim.new(0, 14); pp.PaddingRight = UDim.new(0, 14); pp.Parent = page
    end
    tabFrames[name] = page; tabButtons[name] = btn
    return page
end

local footerBar = Instance.new("Frame"); footerBar.Name = "FooterBar"; footerBar.Size = UDim2.new(1, 0, 0, 24); footerBar.Position = UDim2.new(0, 0, 1, -24); footerBar.BackgroundColor3 = Colors.SidebarBg; footerBar.BorderSizePixel = 0; footerBar.Parent = mainFrame
Instance.new("UICorner", footerBar).CornerRadius = UDim.new(0, 8)
do
    local tf = Instance.new("Frame"); tf.Size = UDim2.new(1, 0, 0, 10); tf.BackgroundColor3 = Colors.SidebarBg; tf.BorderSizePixel = 0; tf.Parent = footerBar
    local fd = Instance.new("Frame"); fd.Size = UDim2.new(1, 0, 0, 1); fd.BackgroundColor3 = Colors.Divider; fd.BorderSizePixel = 0; fd.Parent = footerBar
end
local footerBrand = Instance.new("TextLabel"); footerBrand.Size = UDim2.new(0, 260, 1, 0); footerBrand.Position = UDim2.new(0, 12, 0, 0); footerBrand.BackgroundTransparency = 1; footerBrand.Font = Enum.Font.Gotham; footerBrand.Text = "Heavyweight Fishing | Việt Hoá V1.1"; footerBrand.TextColor3 = Colors.TextMuted; footerBrand.TextSize = 10; footerBrand.TextXAlignment = Enum.TextXAlignment.Left; footerBrand.Parent = footerBar
local footerKey = Instance.new("TextLabel"); footerKey.Size = UDim2.new(0, 280, 1, 0); footerKey.Position = UDim2.new(1, -292, 0, 0); footerKey.BackgroundTransparency = 1; footerKey.Font = Enum.Font.Gotham; footerKey.Text = "[R-CTRL] Menu | [END] Tắt Script"; footerKey.TextColor3 = Colors.TextMuted; footerKey.TextSize = 10; footerKey.TextXAlignment = Enum.TextXAlignment.Right; footerKey.Parent = footerBar

ToggleUiVisibility = function()
    mainFrame.Visible = not mainFrame.Visible
    floatingAvatar.Visible = not mainFrame.Visible
    if mainFrame.Visible then
        TweenService:Create(mainFrame, TweenInfo.new(0.2, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {BackgroundTransparency = 0}):Play()
    end
end

minBtn.MouseButton1Click:Connect(ToggleUiVisibility)
closeBtn.MouseButton1Click:Connect(ToggleUiVisibility)
killBtn.MouseButton1Click:Connect(function()
    ShowNotification("Diệt Script", "Đang ngắt kết nối và đóng script hoàn toàn...", "WARN", 2)
    task.wait(0.2)
    UnloadScript()
end)

local function createCategoryHeader(parent, text)
    local hdr = Instance.new("Frame"); hdr.Size = UDim2.new(1, 0, 0, 22); hdr.BackgroundTransparency = 1; hdr.Parent = parent
    local lbl = Instance.new("TextLabel"); lbl.Size = UDim2.new(1, 0, 1, 0); lbl.BackgroundTransparency = 1; lbl.Font = Enum.Font.GothamBold; lbl.Text = string.upper(text); lbl.TextColor3 = Colors.PurplePrimary; lbl.TextSize = 11; lbl.TextXAlignment = Enum.TextXAlignment.Left; lbl.Parent = hdr
    return hdr
end

local function createCardGroup(parent)
    local group = Instance.new("Frame"); group.Size = UDim2.new(1, 0, 0, 0); group.AutomaticSize = Enum.AutomaticSize.Y; group.BackgroundColor3 = Colors.RowNormal; group.BorderSizePixel = 0; group.Parent = parent
    local s = Instance.new("UIStroke"); s.Color = Colors.BorderSubtle; s.Thickness = 1; s.Parent = group
    Instance.new("UICorner", group).CornerRadius = UDim.new(0, 6)
    local l = Instance.new("UIListLayout"); l.SortOrder = Enum.SortOrder.LayoutOrder; l.Padding = UDim.new(0, 0); l.Parent = group
    return group
end

local function createCollapsibleCardGroup(parent, text, defaultOpen)
    local isOpen = (defaultOpen == true)
    local hdr = Instance.new("Frame")
    hdr.Size = UDim2.new(1, 0, 0, 26)
    hdr.BackgroundTransparency = 1
    hdr.Parent = parent

    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, 0, 1, 0)
    btn.BackgroundTransparency = 1
    btn.Text = ""
    btn.Parent = hdr

    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, -95, 1, 0)
    lbl.Position = UDim2.new(0, 0, 0, 0)
    lbl.BackgroundTransparency = 1
    lbl.Font = Enum.Font.GothamBold
    lbl.Text = string.upper(text)
    lbl.TextColor3 = Colors.PurplePrimary
    lbl.TextSize = 11
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = hdr

    local badge = Instance.new("TextButton")
    badge.Size = UDim2.new(0, 85, 0, 20)
    badge.Position = UDim2.new(1, -85, 0.5, -10)
    badge.BackgroundColor3 = Colors.ControlBg
    badge.BorderSizePixel = 0
    badge.Font = Enum.Font.GothamBold
    badge.Text = isOpen and "▼ Thu Gọn" or "▶ Mở Rộng"
    badge.TextColor3 = isOpen and Colors.TextMuted or Colors.PurpleAccent
    badge.TextSize = 10
    badge.Parent = hdr
    Instance.new("UICorner", badge).CornerRadius = UDim.new(0, 4)

    local group = createCardGroup(parent)
    group.Visible = isOpen

    local function toggle()
        isOpen = not isOpen
        group.Visible = isOpen
        badge.Text = isOpen and "▼ Thu Gọn" or "▶ Mở Rộng"
        badge.TextColor3 = isOpen and Colors.TextMuted or Colors.PurpleAccent
    end

    btn.MouseButton1Click:Connect(toggle)
    badge.MouseButton1Click:Connect(toggle)

    return group, toggle
end

local function createBaseRow(parent, labelText, descText, indexSearch)
    local row = Instance.new("Frame"); row.Size = UDim2.new(1, 0, 0, 42); row.BackgroundColor3 = Colors.RowNormal; row.BorderSizePixel = 0; row.Parent = parent
    local pad = Instance.new("UIPadding"); pad.PaddingLeft = UDim.new(0, 10); pad.PaddingRight = UDim.new(0, 10); pad.Parent = row
    local tf = Instance.new("Frame"); tf.Size = UDim2.new(1, -190, 1, 0); tf.BackgroundTransparency = 1; tf.Parent = row
    local tl = Instance.new("TextLabel"); tl.Size = UDim2.new(1, 0, 0, 18); tl.Position = UDim2.new(0, 0, 0, 4); tl.BackgroundTransparency = 1; tl.Font = Enum.Font.GothamBold; tl.Text = labelText; tl.TextColor3 = Colors.TextWhite; tl.TextSize = 12; tl.TextXAlignment = Enum.TextXAlignment.Left; tl.Parent = tf
    local dl = Instance.new("TextLabel"); dl.Size = UDim2.new(1, 0, 0, 14); dl.Position = UDim2.new(0, 0, 0, 22); dl.BackgroundTransparency = 1; dl.Font = Enum.Font.Gotham; dl.Text = descText or ""; dl.TextColor3 = Colors.TextMuted; dl.TextSize = 10; dl.TextXAlignment = Enum.TextXAlignment.Left; dl.Parent = tf
    row.MouseEnter:Connect(function() TweenService:Create(row, TweenInfo.new(0.15), {BackgroundColor3 = Colors.RowHover}):Play() end)
    row.MouseLeave:Connect(function() TweenService:Create(row, TweenInfo.new(0.15), {BackgroundColor3 = Colors.RowNormal}):Play() end)
    if indexSearch ~= false then
        table.insert(rowSearchIndex, {frame = row, query = (labelText .. " " .. (descText or "")):lower()})
    end
    return row
end

local function createToggleRow(parent, labelText, descText, initialVal, callback, indexSearch)
    if type(initialVal) == "function" then
        indexSearch = callback
        callback = initialVal
        initialVal = false
    end
    local row = createBaseRow(parent, labelText, descText, indexSearch)
    local tf = row:FindFirstChildOfClass("Frame")
    local descLbl = nil
    if tf then
        tf.Size = UDim2.new(1, -55, 1, 0)
        for _, child in ipairs(tf:GetChildren()) do
            if child:IsA("TextLabel") and child.TextSize == 10 then
                descLbl = child
                pcall(function() child.TextTruncate = Enum.TextTruncate.AtEnd end)
            end
        end
    end
    local state = initialVal or false
    local btn = Instance.new("TextButton"); btn.Size = UDim2.new(0, 40, 0, 20); btn.Position = UDim2.new(1, -40, 0.5, -10); btn.BackgroundColor3 = state and Colors.PurpleAccent or Colors.ControlBg; btn.Text = ""; btn.BorderSizePixel = 0; btn.Parent = row
    Instance.new("UICorner", btn).CornerRadius = UDim.new(1, 0)
    local knob = Instance.new("Frame"); knob.Size = UDim2.new(0, 14, 0, 14); knob.Position = state and UDim2.new(1, -17, 0.5, -7) or UDim2.new(0, 3, 0.5, -7); knob.BackgroundColor3 = Colors.TextWhite; knob.BorderSizePixel = 0; knob.Parent = btn
    Instance.new("UICorner", knob).CornerRadius = UDim.new(1, 0)
    local function updateVisuals()
        TweenService:Create(btn, TweenInfo.new(0.15), {BackgroundColor3 = state and Colors.PurpleAccent or Colors.ControlBg}):Play()
        TweenService:Create(knob, TweenInfo.new(0.15), {Position = state and UDim2.new(1, -17, 0.5, -7) or UDim2.new(0, 3, 0.5, -7)}):Play()
    end
    btn.MouseButton1Click:Connect(function()
        state = not state
        updateVisuals()
        if type(callback) == "function" then callback(state) end
        local key = ConfigLabelMap[labelText]
        if key and Config._essentialKeys and Config._essentialKeys[key] and Config._triggerAutoSave then
            Config._triggerAutoSave()
        end
    end)
    local ret = {
        frame = row,
        descLabel = descLbl,
        Set = function(val, skipCallback)
            state = val
            updateVisuals()
            if not skipCallback and type(callback) == "function" then
                callback(state)
            end
        end,
        Get = function() return state end
    }
    local key = ConfigLabelMap[labelText]
    if key then UIControllers[key] = ret end
    return ret
end

local function createSliderRow(parent, labelText, descText, minVal, maxVal, initialVal, isFloat, suffix, callback, indexSearch)
    local row = createBaseRow(parent, labelText, descText, indexSearch)
    local currentVal = initialVal or minVal; suffix = suffix or ""
    local container = Instance.new("Frame"); container.Size = UDim2.new(0, 180, 0, 24); container.Position = UDim2.new(1, -180, 0.5, -12); container.BackgroundTransparency = 1; container.Parent = row
    local valLabel = Instance.new("TextLabel"); valLabel.Size = UDim2.new(0, 68, 1, 0); valLabel.Position = UDim2.new(1, -68, 0, 0); valLabel.BackgroundTransparency = 1; valLabel.Font = Enum.Font.GothamBold; valLabel.TextColor3 = Colors.PurplePrimary; valLabel.TextSize = 11; valLabel.TextXAlignment = Enum.TextXAlignment.Right; valLabel.Parent = container
    valLabel.Text = isFloat and string.format("%.2f", currentVal)..suffix or tostring(math.floor(currentVal))..suffix
    local track = Instance.new("Frame"); track.Size = UDim2.new(1, -74, 0, 6); track.Position = UDim2.new(0, 0, 0.5, -3); track.BackgroundColor3 = Colors.ControlBg; track.BorderSizePixel = 0; track.Parent = container
    Instance.new("UICorner", track).CornerRadius = UDim.new(1, 0)
    local pct = math.clamp((currentVal - minVal) / (maxVal - minVal), 0, 1)
    local fill = Instance.new("Frame"); fill.Size = UDim2.new(pct, 0, 1, 0); fill.BackgroundColor3 = Colors.PurpleAccent; fill.BorderSizePixel = 0; fill.Parent = track
    Instance.new("UICorner", fill).CornerRadius = UDim.new(1, 0)
    local sliding = false
    local function updateFromX(x)
        local rel = math.clamp((x - track.AbsolutePosition.X) / track.AbsoluteSize.X, 0, 1)
        local val = minVal + (maxVal - minVal) * rel
        if not isFloat then val = math.floor(val + 0.5) end
        currentVal = val; fill.Size = UDim2.new(rel, 0, 1, 0)
        valLabel.Text = isFloat and string.format("%.2f", val)..suffix or tostring(val)..suffix
        if type(callback) == "function" then callback(val) end
        local skey = ConfigLabelMap[labelText]
        if skey and Config._essentialKeys and Config._essentialKeys[skey] and Config._triggerAutoSave then
            Config._triggerAutoSave()
        end
    end
    track.InputBegan:Connect(function(input) if input.UserInputType == Enum.UserInputType.MouseButton1 then sliding = true; updateFromX(input.Position.X) end end)
    UserInputService.InputEnded:Connect(function(input) if input.UserInputType == Enum.UserInputType.MouseButton1 then sliding = false end end)
    table.insert(activeConnections, UserInputService.InputChanged:Connect(function(input) if sliding and input.UserInputType == Enum.UserInputType.MouseMovement then updateFromX(input.Position.X) end end))
    local ret = {
        frame = row,
        Set = function(val, skipCallback)
            currentVal = math.clamp(val, minVal, maxVal)
            local p2 = (currentVal - minVal) / (maxVal - minVal)
            fill.Size = UDim2.new(p2, 0, 1, 0)
            valLabel.Text = isFloat and string.format("%.2f", currentVal)..suffix or tostring(math.floor(currentVal))..suffix
            if not skipCallback and type(callback) == "function" then callback(currentVal) end
        end
    }
    local key = ConfigLabelMap[labelText]
    if key then UIControllers[key] = ret end
    return ret
end

local function createDropdownRow(parent, labelText, descText, options, initialVal, callback, indexSearch)
    if type(options) == "table" and type(initialVal) == "function" then
        indexSearch = callback
        callback = initialVal
        initialVal = options[1]
    end
    local selected = initialVal or options[1]
    local row = Instance.new("Frame"); row.Size = UDim2.new(1, 0, 0, 42); row.AutomaticSize = Enum.AutomaticSize.Y; row.BackgroundColor3 = Colors.RowNormal; row.BorderSizePixel = 0; row.ClipsDescendants = true; row.Parent = parent
    local rl = Instance.new("UIListLayout"); rl.SortOrder = Enum.SortOrder.LayoutOrder; rl.Padding = UDim.new(0, 4); rl.Parent = row
    local header = Instance.new("Frame"); header.Size = UDim2.new(1, 0, 0, 42); header.BackgroundTransparency = 1; header.Parent = row
    local pad = Instance.new("UIPadding"); pad.PaddingLeft = UDim.new(0, 10); pad.PaddingRight = UDim.new(0, 10); pad.Parent = header
    local tf = Instance.new("Frame"); tf.Size = UDim2.new(1, -145, 1, 0); tf.BackgroundTransparency = 1; tf.Parent = header
    local tl = Instance.new("TextLabel"); tl.Size = UDim2.new(1, 0, 0, 18); tl.Position = UDim2.new(0, 0, 0, 4); tl.BackgroundTransparency = 1; tl.Font = Enum.Font.GothamBold; tl.Text = labelText; tl.TextColor3 = Colors.TextWhite; tl.TextSize = 12; tl.TextXAlignment = Enum.TextXAlignment.Left; tl.Parent = tf
    local dl = Instance.new("TextLabel"); dl.Size = UDim2.new(1, 0, 0, 14); dl.Position = UDim2.new(0, 0, 0, 22); dl.BackgroundTransparency = 1; dl.Font = Enum.Font.Gotham; dl.Text = descText or ""; dl.TextColor3 = Colors.TextMuted; dl.TextSize = 10; dl.TextXAlignment = Enum.TextXAlignment.Left; dl.Parent = tf
    local ddBtn = Instance.new("TextButton"); ddBtn.Size = UDim2.new(0, 130, 0, 24); ddBtn.Position = UDim2.new(1, -130, 0.5, -12); ddBtn.BackgroundColor3 = Colors.ControlBg; ddBtn.Font = Enum.Font.GothamBold; ddBtn.Text = tostring(selected) .. "  v"; ddBtn.TextColor3 = Colors.PurplePrimary; ddBtn.TextSize = 11; ddBtn.BorderSizePixel = 0; ddBtn.Parent = header
    Instance.new("UICorner", ddBtn).CornerRadius = UDim.new(0, 4)
    local optC = Instance.new("Frame"); optC.Size = UDim2.new(1, 0, 0, 0); optC.AutomaticSize = Enum.AutomaticSize.Y; optC.BackgroundTransparency = 1; optC.Visible = false; optC.Parent = row
    do local p = Instance.new("UIPadding"); p.PaddingLeft = UDim.new(0, 10); p.PaddingRight = UDim.new(0, 10); p.PaddingBottom = UDim.new(0, 8); p.Parent = optC end
    Instance.new("UIListLayout", optC).SortOrder = Enum.SortOrder.LayoutOrder
    local optButtons = {}
    local function populate(opts)
        for _, c in ipairs(optC:GetChildren()) do if c:IsA("TextButton") then c:Destroy() end end
        table.clear(optButtons)
        for _, opt in ipairs(opts) do
            local ob = Instance.new("TextButton"); ob.Size = UDim2.new(1, 0, 0, 26); ob.BackgroundColor3 = (opt == selected) and Colors.DropdownSelected or Colors.InputBg; ob.Font = Enum.Font.Gotham; ob.Text = (opt == selected and "> " or "   ") .. tostring(opt); ob.TextColor3 = (opt == selected) and Colors.PurplePrimary or Colors.TextWhite; ob.TextSize = 11; ob.TextXAlignment = Enum.TextXAlignment.Left; ob.BorderSizePixel = 0; ob.Parent = optC
            Instance.new("UICorner", ob).CornerRadius = UDim.new(0, 4)
            do local p = Instance.new("UIPadding"); p.PaddingLeft = UDim.new(0, 10); p.Parent = ob end
            optButtons[opt] = ob
            ob.MouseEnter:Connect(function() if opt ~= selected then TweenService:Create(ob, TweenInfo.new(0.15), {BackgroundColor3 = Colors.RowHover}):Play() end end)
            ob.MouseLeave:Connect(function() if opt ~= selected then TweenService:Create(ob, TweenInfo.new(0.15), {BackgroundColor3 = Colors.InputBg}):Play() end end)
            ob.MouseButton1Click:Connect(function()
                selected = opt; ddBtn.Text = tostring(opt) .. "  v"; optC.Visible = false
                for oN, b in pairs(optButtons) do b.BackgroundColor3 = (oN == opt) and Colors.DropdownSelected or Colors.InputBg; b.TextColor3 = (oN == opt) and Colors.PurplePrimary or Colors.TextWhite; b.Text = (oN == opt and "> " or "   ") .. tostring(oN) end
                if type(callback) == "function" then callback(opt) end
                local mKey = ConfigLabelMap[labelText]
                if mKey and Config._essentialKeys and Config._essentialKeys[mKey] and Config._triggerAutoSave then
                    Config._triggerAutoSave()
                end
            end)
        end
    end
    populate(options)
    ddBtn.MouseButton1Click:Connect(function() optC.Visible = not optC.Visible; ddBtn.Text = tostring(selected) .. (optC.Visible and "  ^" or "  v") end)
    header.MouseEnter:Connect(function() TweenService:Create(row, TweenInfo.new(0.15), {BackgroundColor3 = Colors.RowHover}):Play() end)
    header.MouseLeave:Connect(function() TweenService:Create(row, TweenInfo.new(0.15), {BackgroundColor3 = Colors.RowNormal}):Play() end)
    if indexSearch ~= false then table.insert(rowSearchIndex, {frame = row, query = (labelText .. " " .. (descText or "")):lower()}) end
    local ret = {
        frame = row,
        Set = function(opt, skipCallback)
            selected = opt; ddBtn.Text = tostring(opt) .. "  v"
            for oN, b in pairs(optButtons) do b.BackgroundColor3 = (oN == opt) and Colors.DropdownSelected or Colors.InputBg; b.TextColor3 = (oN == opt) and Colors.PurplePrimary or Colors.TextWhite; b.Text = (oN == opt and "> " or "   ") .. tostring(oN) end
            if not skipCallback and type(callback) == "function" then pcall(callback, opt) end
        end,
        Get = function() return selected end,
        Refresh = function(newOpts, keepCurrent)
            options = newOpts or {}
            populate(options)
            local found = false
            if keepCurrent and selected then
                for _, opt in ipairs(options) do
                    if opt == selected then found = true; break end
                end
            end
            if not found then
                selected = options[1] or ""
            end
            ddBtn.Text = (selected ~= "" and tostring(selected) or "Không có") .. "  v"
        end
    }
    local mappedKey = ConfigLabelMap[labelText]
    if mappedKey then
        UIControllers[mappedKey] = ret
    end
    return ret
end

local function createButtonRow(parent, labelText, descText, btnText, callback, indexSearch)
    if type(descText) == "function" then
        indexSearch = btnText
        callback = descText
        btnText = "Execute"
        descText = ""
    elseif type(btnText) == "function" then
        indexSearch = callback
        callback = btnText
        btnText = descText
        descText = ""
    end
    local row = createBaseRow(parent, labelText, descText, indexSearch)
    local btn = Instance.new("TextButton"); btn.Size = UDim2.new(0, 90, 0, 24); btn.Position = UDim2.new(1, -90, 0.5, -12); btn.BackgroundColor3 = Colors.ControlBg; btn.Font = Enum.Font.GothamBold; btn.Text = btnText or "Execute"; btn.TextColor3 = Colors.PurplePrimary; btn.TextSize = 11; btn.BorderSizePixel = 0; btn.Parent = row
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 4)
    btn.MouseEnter:Connect(function() TweenService:Create(btn, TweenInfo.new(0.15), {BackgroundColor3 = Colors.PurpleDark, TextColor3 = Colors.TextWhite}):Play() end)
    btn.MouseLeave:Connect(function() TweenService:Create(btn, TweenInfo.new(0.15), {BackgroundColor3 = Colors.ControlBg, TextColor3 = Colors.PurplePrimary}):Play() end)
    btn.MouseButton1Click:Connect(function()
        if type(callback) == "function" then pcall(callback) end
    end)
    return btn
end

local function createInfoRow(parent, labelText, valueText, indexSearch)
    local row = createBaseRow(parent, labelText, "", indexSearch)
    local tf = row:FindFirstChildOfClass("Frame")
    if tf then tf.Size = UDim2.new(0.48, -10, 1, 0) end
    local vl = Instance.new("TextLabel"); vl.Size = UDim2.new(0.52, 0, 1, 0); vl.Position = UDim2.new(0.48, 0, 0, 0); vl.BackgroundTransparency = 1; vl.Font = Enum.Font.GothamBold; vl.Text = valueText; vl.TextColor3 = Colors.PurplePrimary; vl.TextSize = 11; vl.TextXAlignment = Enum.TextXAlignment.Right; pcall(function() vl.TextTruncate = Enum.TextTruncate.AtEnd end); vl.Parent = row
    return {frame = row, Set = function(nv) vl.Text = nv end}
end

local function createInputRow(parent, labelText, descText, initialVal, callback, indexSearch, placeholder)
    local row = createBaseRow(parent, labelText, descText, indexSearch)
    local tb = Instance.new("TextBox")
    tb.Size = UDim2.new(0, 160, 0, 24)
    tb.Position = UDim2.new(1, -160, 0.5, -12)
    tb.BackgroundColor3 = Colors.InputBg
    tb.Font = Enum.Font.Gotham
    tb.Text = initialVal or ""
    tb.PlaceholderText = placeholder or "Nhập tại đây..."
    tb.PlaceholderColor3 = Colors.TextMuted
    tb.TextColor3 = Colors.TextWhite
    tb.TextSize = 11
    tb.ClearTextOnFocus = false
    tb.BorderSizePixel = 0
    tb.Parent = row
    Instance.new("UICorner", tb).CornerRadius = UDim.new(0, 4)
    local s = Instance.new("UIStroke", tb)
    s.Color = Colors.BorderSubtle
    s.Thickness = 1
    tb.FocusLost:Connect(function(enterPressed)
        if type(callback) == "function" then callback(tb.Text) end
        local inpKey = ConfigLabelMap[labelText]
        if inpKey and Config._essentialKeys and Config._essentialKeys[inpKey] and Config._triggerAutoSave then
            Config._triggerAutoSave()
        end
    end)
    local ret = {
        frame = row,
        Set = function(val)
            tb.Text = tostring(val or "")
            if type(callback) == "function" then pcall(callback, tb.Text) end
        end,
        Get = function() return tb.Text end
    }
    local mappedKey = ConfigLabelMap[labelText]
    if mappedKey then
        UIControllers[mappedKey] = ret
    end
    return ret
end

-- ============================================================
-- SECRET BOSS DATABASE & CHAT HUNTER LOGIC
-- ============================================================
local allRods = {
    {name = "Wooden Rod", price = 0, power = 8, luck = 1},
    {name = "Bamboo Rod", price = 100, power = 11, luck = 5},
    {name = "Iron Hook Rod", price = 500, power = 13, luck = 10},
    {name = "Steel Rod", price = 1000, power = 17, luck = 11},
    {name = "Enchanted Steel Rod", price = 2000, power = 19, luck = 12},
    {name = "Alloy Rod", price = 5000, power = 22, luck = 5},
    {name = "Emerald Rod", price = 10000, power = 25, luck = 10},
    {name = "Bloodfire Rod", price = 20000, power = 27, luck = 10},
    {name = "Shadow Rod", price = 60000, power = 29, luck = 22},
    {name = "Triple Steel Rod", price = 100000, power = 32, luck = 20},
    {name = "Golden Rod", price = 200000, power = 35, luck = 22},
    {name = "Grandmaster Steel Rod", price = 250000, power = 37, luck = 10},
    {name = "Grandmaster Golden Rod", price = 1000000, power = 45, luck = 30},
    {name = "Steel Spine Rod", price = 1500000, power = 48, luck = 20},
    {name = "Inferno Rod", price = 1500000, power = 48, luck = 18},
    {name = "Golden Spine Rod", price = 2000000, power = 51, luck = 21},
    {name = "Platinum Spine Rod", price = 3000000, power = 54, luck = 25},
    {name = "Diamond Spine Rod", price = 4000000, power = 56, luck = 25},
    {name = "Gravisteel Rod", price = 5000000, power = 58, luck = 15},
    {name = "Auric Gravity Rod", price = 6000000, power = 60, luck = 10},
    {name = "Inferno Gravity Rod", price = 7000000, power = 62, luck = 20},
    {name = "Cryo Gravity Rod", price = 8000000, power = 65, luck = 36},
    {name = "Thunder Thorn Rod", price = 10000000, power = 67, luck = 30},
    {name = "Starlight Rod", price = 60000000, power = 83, luck = 15},
    -- CẦN CÂU BÍ MẬT & THẦN THOẠI (SECRET & MYTHIC RODS)
    {name = "Anchorbound Rod", price = 0, power = 50, luck = 25},
    {name = "Blazeshark Rod", price = 0, power = 55, luck = 20},
    {name = "Kraken Rod", price = 0, power = 65, luck = 30},
    {name = "Ascendant Bamboo Rod", price = 0, power = 70, luck = 35},
    {name = "Lifebloom Rod", price = 0, power = 75, luck = 40},
    {name = "Demonic Rod", price = 0, power = 85, luck = 25},
}

local function IsRodOwned(rodName)
    if not rodName or rodName == "" then return false end

    -- 1. Cần đang cầm trên tay hoặc lưu trong pData.FishingRod
    local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
    if pData and pData:FindFirstChild("FishingRod") and pData.FishingRod.Value == rodName then
        return true
    end

    -- 2. Kiểm tra trong kho cần câu chính: FishingRodInventory
    if pData and pData:FindFirstChild("FishingRodInventory") then
        local rFolder = pData.FishingRodInventory:FindFirstChild(rodName)
        if rFolder then
            local oVal = rFolder:FindFirstChild("Owned")
            if oVal and oVal:IsA("ValueBase") and oVal.Value == true then
                return true
            end
            if rFolder:IsA("BoolValue") and rFolder.Value == true then
                return true
            end
            if rFolder:GetAttribute("Owned") == true then
                return true
            end
            if oVal == nil and not rFolder:FindFirstChild("Locked") then
                return true
            end
        end
    end

    -- 3. Kiểm tra các thư mục kho phụ (Rods, RodInventory, Inventory, Tools)
    if pData then
        for _, fName in ipairs({"Rods", "RodInventory", "Inventory", "Tools"}) do
            local f = pData:FindFirstChild(fName)
            if f and f:FindFirstChild(rodName) then
                local oVal = f[rodName]:FindFirstChild("Owned")
                if oVal and oVal:IsA("ValueBase") then
                    if oVal.Value == true then return true end
                else
                    return true
                end
            end
        end
    end

    -- 4. Kiểm tra trong Backpack hoặc Character
    local bp = LocalPlayer:FindFirstChild("Backpack")
    if bp and bp:FindFirstChild(rodName) then return true end
    local char = LocalPlayer.Character
    if char and char:FindFirstChild(rodName) then return true end

    -- 5. Cần mặc định của game (Wooden Rod) luôn đã có
    if rodName == "Wooden Rod" then
        return true
    end

    return false
end

-- ====================================================================
-- HỆ THỐNG KIỂM TRA TRẠNG THÁI SỞ HỮU THUYỀN, SKILL, ORB & ĐỊNH DẠNG CAPTION CÁ
-- ====================================================================
local FishRewardHelper = {}

function FishRewardHelper.GetPlayerDataFolder()
    local data = ReplicatedStorage:FindFirstChild("Data")
    if not data then return nil end
    local uid = LocalPlayer and tostring(LocalPlayer.UserId)
    if uid and data:FindFirstChild(uid) then
        return data[uid]
    end
    for _, child in ipairs(data:GetChildren()) do
        if tonumber(child.Name) then
            return child
        end
    end
    return nil
end

function FishRewardHelper.HasPlayerBoat(boatName)
    if not boatName or boatName == "" then return false end
    local pData = FishRewardHelper.GetPlayerDataFolder()
    if pData and pData:FindFirstChild("Boats") then
        local b = pData.Boats:FindFirstChild(boatName)
        if b and (b.Value == true or b.Value == 1) then
            return true
        end
        local clean = boatName:lower():gsub("[%s%-_]+", "")
        for _, child in ipairs(pData.Boats:GetChildren()) do
            if child.Name:lower():gsub("[%s%-_]+", "") == clean and (child.Value == true or child.Value == 1) then
                return true
            end
        end
    end
    return false
end

function FishRewardHelper.HasPlayerSkill(skillName)
    if not skillName or skillName == "" then return false end
    local pData = FishRewardHelper.GetPlayerDataFolder()
    if pData and pData:FindFirstChild("Skill") then
        local sFolder = pData.Skill:FindFirstChild(skillName)
        if sFolder then
            local owned = sFolder:FindFirstChild("Owned")
            if owned and (owned.Value == true or owned.Value == 1) then
                return true
            end
        end
        local clean = skillName:lower():gsub("[%s%-_]+", "")
        for _, child in ipairs(pData.Skill:GetChildren()) do
            local cClean = child.Name:lower():gsub("[%s%-_]+", "")
            if cClean == clean or cClean:find(clean, 1, true) or clean:find(cClean, 1, true) then
                local owned = child:FindFirstChild("Owned")
                if owned and (owned.Value == true or owned.Value == 1) then
                    return true
                end
            end
        end
    end
    return false
end

function FishRewardHelper.GetPlayerOrbCount(orbName)
    local count = 0
    local pData = FishRewardHelper.GetPlayerDataFolder()
    if pData then
        if pData:FindFirstChild("Orb") then
            local targetClean = orbName and orbName:lower():gsub("[%s%-_]+", "") or ""
            for _, oItem in ipairs(pData.Orb:GetChildren()) do
                local vn = oItem:FindFirstChild("ValueName")
                if vn and vn:IsA("StringValue") then
                    if targetClean == "" then
                        count = count + 1
                    else
                        local valClean = vn.Value:lower():gsub("[%s%-_]+", "")
                        if valClean:find(targetClean, 1, true) or targetClean:find(valClean, 1, true) then
                            count = count + 1
                        end
                    end
                end
            end
        end
        if orbName and orbName:lower():find("essence") then
            local eo = pData:FindFirstChild("EssenceOrb")
            if eo and eo:IsA("NumberValue") then
                count = count + eo.Value
            end
        end
    end
    return count
end

function FishRewardHelper.FormatFishCaption(f)
    local parts = {}

    -- 1. Mục đích / Công dụng (Chế cần, chế mồi, trả quest...)
    if f.use and f.use ~= "" then
        table.insert(parts, f.use)
    end

    -- 2. Thuyền (Boat)
    if f.boatReward then
        local bName = f.boatReward.name or f.boatReward
        local rate = f.boatReward.rate or "5%"
        local has = FishRewardHelper.HasPlayerBoat(bName)
        local status = has and "[ĐÃ CÓ]" or "[CHƯA CÓ]"
        table.insert(parts, string.format("Thuyền %s (%s): %s", bName, rate, status))
    end

    -- 3. Kỹ năng (Skill)
    if f.skillReward then
        local sName = f.skillReward.name or f.skillReward
        local rate = f.skillReward.rate or "10%"
        local has = FishRewardHelper.HasPlayerSkill(sName)
        local status = has and "[ĐÃ CÓ]" or "[CHƯA CÓ]"
        table.insert(parts, string.format("Skill %s (%s): %s", sName, rate, status))
    end

    -- 4. Orb
    if f.orbReward then
        local oName = f.orbReward.name or f.orbReward
        local rate = f.orbReward.rate or "20%"
        local c = FishRewardHelper.GetPlayerOrbCount(oName)
        table.insert(parts, string.format("Orb %s (%s) [Đang có: x%d]", oName, rate, c))
    end

    -- 5. Gems
    if f.gems and f.gems > 0 then
        table.insert(parts, string.format("+%d Gems", f.gems))
    elseif f.reward and f.reward ~= "" and not f.use then
        table.insert(parts, f.reward)
    end

    if #parts == 0 then
        return f.reward or "Cá Quý Hiếm"
    end

    return table.concat(parts, " • ")
end

local secretBossDatabase = {
    {
        islandName = "Đảo Tre (Bamboo Isle)",
        weather = "Thunderstorm (Bão Sấm)",
        patterns = {"bamboo", "đảo tre", "dao tre", "đảo 2", "dao 2"},
        weatherPatterns = {"thunderstorm", "bão sấm", "bao sam", "thunder", "sấm", "lightning"},
        bossPatterns = {"scarlet fish", "elder scarlet", "crimson electric eel", "electric eel", "golden dragonfish", "rainbow dragonfish"},
        pos = Vector3.new(-1504.8, 14.7, 252.7),
        lookAt = Vector3.new(-1515.0, 14.7, 301.6),
        spots = {
            [1] = {
                pos = Vector3.new(-1504.8, 14.7, 252.7),
                lookAt = Vector3.new(-1515.0, 14.7, 301.6),
            },
        },
        bosses = {
            {name = "Golden Dragonfish", use = "Chế Cần & Mồi • Thần Thoại", orbReward = {name = "Dragon Orb", rate = "20%"}, gems = 20},
            {name = "Rainbow Dragonfish", use = "Chế Cần Heavenpiercer & Mồi Rainbow • Quest Hạ Diêu", gems = 50},
            {name = "Scarlet Fish", use = "Chế Cần Huyết Long Rod", gems = 3},
            {name = "Elder Scarlet Fish", use = "Chế Cần Huyết Long Rod", gems = 5},
            {name = "Crimson Electric Eel", use = "Chế Mồi Rainbow Bait", gems = 5},
        },
        usefulFish = {}
    },
    {
        islandName = "Đảo Phóng Xạ (Fallout Isle)",
        weather = "Rainy (Trời Mưa)",
        patterns = {"fallout", "phóng xạ", "phong xa", "đảo 3", "dao 3"},
        weatherPatterns = {"rainy", "trời mưa", "troi mua", "heavy rain", "mưa", "rain"},
        bossPatterns = {"alligator gar", "verdant alligator", "verdant grouper", "verdant bonefang", "crimson bonefang", "bonefang"},
        pos = Vector3.new(30.1, 9.3, 1682.3),
        lookAt = Vector3.new(32.3, 9.3, 1732.2),
        spots = {
            [1] = {
                pos = Vector3.new(30.1, 9.3, 1682.3),
                lookAt = Vector3.new(32.3, 9.3, 1732.2),
            },
        },
        bosses = {
            {name = "Verdant Alligator Gar", use = "Chế Cần Huyết Long", skillReward = {name = "Beastbreaker Cleave", rate = "25%"}, gems = 3},
            {name = "Verdant Grouper", use = "Boss Mưa Huyền Thoại", skillReward = {name = "Cyclone Hook", rate = "25%"}, gems = 3},
            {name = "Verdant Bonefang", use = "Chế Cần Huyết Long", skillReward = {name = "River Suppression", rate = "5%"}, gems = 5},
            {name = "Crimson Bonefang", use = "Thần Thú Huyết Cốt Nha", skillReward = {name = "Seven Wounds Fusion", rate = "10%"}, gems = 15},
        },
        usefulFish = {
            {name = "Trueform Jiaolongfish", use = "Giao Long Chân Thân • Long Ngư Thần Thoại", skillReward = {name = "Rolling Twin Dragons", rate = "50%"}},
            {name = "Adult Jiaolong Dragonfish", use = "Trưởng Thành Giao Long • Long Ngư Quý Hiếm"},
            {name = "Elder Jiaolong Dragonfish", use = "Cổ Đại Giao Long • Long Ngư Thần Thoại"},
            {name = "Serpent Fish", use = "Mãng Xà Ngư Quý • Cá Quý Đảo Phóng Xạ"},
        }
    },
    {
        islandName = "Đảo Cá Chép (Perch Isle)",
        weather = "Windy (Trời Gió)",
        patterns = {"perch", "cá chép", "ca chep", "đảo 5", "dao 5"},
        weatherPatterns = {"windy", "trời gió", "troi gio", "gale", "gió", "wind"},
        bossPatterns = {"flying fish empress", "flying fish emperor", "flying fish"},
        pos = Vector3.new(126.3, 6.8, -1472.7),
        lookAt = Vector3.new(116.6, 6.8, -1423.6),
        spots = {
            [1] = {
                pos = Vector3.new(126.3, 6.8, -1472.7),
                lookAt = Vector3.new(116.6, 6.8, -1423.6),
            },
        },
        bosses = {
            {name = "Flying Fish Emperor", use = "Chế Cần Heavenpiercer Rod", skillReward = {name = "Skybreaker Technique", rate = "10%"}, gems = 10},
            {name = "Flying Fish Empress", use = "Chế Cần Heavenpiercer Rod", skillReward = {name = "Skybreaker Technique", rate = "10%"}, gems = 10},
        },
        usefulFish = {
            {name = "Ascended Perch", use = "Chế Cần Trúc Thánh & Mồi Frost", boatReward = {name = "Ascended Perch", rate = "5%"}},
            {name = "Trueform Perch", use = "Chân Thân Cá Chép • Cá Quý Đảo Cá Chép", skillReward = {name = "River Suppression", rate = "20%"}},
            {name = "Elder Perch VIII", use = "Cá Chép Thâm Niên VIII • Giá Trị Cao"},
        }
    },
    {
        islandName = "Đảo Băng Giá (Frost Isle)",
        weather = "Snowy (Bão Tuyết)",
        patterns = {"frost", "băng giá", "bang gia", "đảo băng", "dao bang", "đảo 6", "dao 6"},
        weatherPatterns = {"snowy", "bão tuyết", "bao tuyet", "blizzard", "tuyết", "snow", "frosty"},
        bossPatterns = {"reborn puffer beast", "puffer beast", "frost kingfish", "frost queenfish"},
        pos = Vector3.new(-1519.4, 6.8, -1334.5),
        lookAt = Vector3.new(-1469.6, 6.8, -1338.2),
        spots = {
            [1] = {
                pos = Vector3.new(-1519.4, 6.8, -1334.5),
                lookAt = Vector3.new(-1469.6, 6.8, -1338.2),
            },
        },
        bosses = {
            {name = "Reborn Puffer Beast", use = "Chế Cần Trúc Thánh & Trả Quest Giang Lão", gems = 10},
            {name = "Frost Kingfish", use = "Chế Cần Pure Diamond Rod & Mồi Frost", skillReward = {name = "Pure Yang Wuji", rate = "10%"}, gems = 10},
            {name = "Frost Queenfish", use = "Chế Cần Pure Diamond Rod • Boss Nữ Hoàng", gems = 20},
        },
        usefulFish = {
            {name = "Dark Kingfish", use = "Hắc Ám Vương Ngư • Cá Quý Hiếm Giá Trị Cao"},
        }
    },
    {
        islandName = "Đảo Quả Dừa (Coconut Isle)",
        weather = "Foggy (Sương Mù)",
        patterns = {"coconut", "quả dừa", "qua dua", "đảo dừa", "dao dua", "đảo 7", "dao 7"},
        weatherPatterns = {"foggy", "sương mù", "suong mu", "dense fog", "mist", "sương", "fog"},
        bossPatterns = {"tigerfang whale", "tigerfang", "tiger fang", "heavenpiercer turtle", "heavenpiercer", "heaven piercer turtle", "heaven piercer", "piercer turtle", "rùa", "rua"},
        pos = Vector3.new(1250.0, 9.5, -1297.7),
        lookAt = Vector3.new(1289.4, 9.5, -1328.5),
        spots = {
            [1] = {
                pos = Vector3.new(1250.0, 9.5, -1297.7),
                lookAt = Vector3.new(1289.4, 9.5, -1328.5),
            },
        },
        bosses = {
            {name = "Tigerfang Whale", use = "Boss Sương Mù • Thần Thoại", skillReward = {name = "Rooster Strike", rate = "10%"}, gems = 5},
            {name = "Heavenpiercer Turtle", use = "Chế Cần Heavenpiercer Rod & Mồi Rainbow Bait", gems = 5},
        },
        usefulFish = {
            {name = "Glorious Elder Turtle", use = "Huy Hoàng Cổ Quy • Thần Quy Huyền Thoại • BẢO VỆ"},
        }
    },
    {
        islandName = "Đảo Hổ Phách (Amber Isle)",
        weather = "Blazing Sun (Nắng Gắt)",
        patterns = {"amber", "hổ phách", "ho phach", "đảo 8", "dao 8"},
        weatherPatterns = {"blazing sun", "nắng gắt", "nang gat", "blazing", "heatwave", "nắng", "sun"},
        bossPatterns = {"draconic koi", "draconic", "sanguine fish", "sanguine"},
        pos = Vector3.new(1579.8, 18.2, 1238.4),
        lookAt = Vector3.new(1629.5, 18.2, 1233.7),
        spots = {
            [1] = {
                pos = Vector3.new(1579.8, 18.2, 1238.4),
                lookAt = Vector3.new(1629.5, 18.2, 1233.7),
            },
        },
        bosses = {
            {name = "Draconic Koi", use = "Chế Cần Pure Diamond Rod", gems = 5},
            {name = "Sanguine Fish", use = "Chế Cần Pure Diamond Rod", skillReward = {name = "Blazing Vajra", rate = "10%"}, gems = 20},
        },
        usefulFish = {
            {name = "Chromatic Koi", use = "Thất Sắc Cẩm Lý • Cá Quý Huyền Thoại"},
        }
    },
    {
        islandName = "Đảo Đỉnh Sương Mù (Mistpeak)",
        weather = "Mountain Peak",
        patterns = {"mistpeak", "đỉnh sương mù", "dinh suong mu", "đảo 10", "dao 10"},
        weatherPatterns = {"mountain peak", "mistpeak", "đỉnh núi"},
        bossPatterns = {"mountain dragonwhale", "dragonwhale", "mountain fish"},
        pos = Vector3.new(3031.4, 19.2, 94.8),
        lookAt = Vector3.new(3081.0, 19.2, 101.3),
        spots = {
            [1] = {
                pos = Vector3.new(3031.4, 19.2, 94.8),
                lookAt = Vector3.new(3081.0, 19.2, 101.3),
            },
        },
        bosses = {
            {name = "Mountain Dragonwhale", use = "Thần Long Kình • Thần Thú Đỉnh Núi", skillReward = {name = "Mountain Flip", rate = "5%"}, gems = 20},
        },
        usefulFish = {
            {name = "Mountain Fish", use = "Chế Cần Trúc Thánh & Mồi Nameless Bait", gems = 20},
            {name = "Tiger Mirefish", use = "Nguyên liệu chế Mồi Nameless Bait (Gọi Boss Bạch Tuộc)"},
            {name = "Mirage Lanternfish", use = "Nguyên liệu chế Mồi Nameless Bait (Gọi Boss Bạch Tuộc)"},
        }
    },
    {
        islandName = "Vùng Biển Sâu (Secret Ocean)",
        weather = "Special Event",
        patterns = {"octo", "bạch tuộc", "bach tuoc", "phao", "buoy", "secret ocean"},
        weatherPatterns = {"special event", "octo", "bạch tuộc"},
        bossPatterns = {"mirage lanternfish", "lanternfish", "nameless octoparasite", "octoparasite"},
        pos = Vector3.new(1419.7, 20.5, -208.8),
        lookAt = Vector3.new(1420.1, 20.5, -258.8),
        spots = {
            [1] = {
                pos = Vector3.new(1419.7, 20.5, -208.8),
                lookAt = Vector3.new(1420.1, 20.5, -258.8),
            },
        },
        bosses = {
            {name = "Mirage Lanternfish", use = "Đèn Lồng Ảo Ảnh • Nguyên liệu chế Mồi Nameless Bait", gems = 20},
            {name = "Nameless Octoparasite", use = "Siêu Boss Biển Sâu • Chế Cần Trúc Thánh & Trả Quest Đạo Sĩ", gems = 50},
        },
        usefulFish = {
            {name = "Octoparasitic Fish", use = "Boss Bạch Tuộc • Nguyên liệu chế Mồi Nameless Bait", gems = 50},
            {name = "Dreadmare Eel", use = "Kinh Hoàng Hải Man • Cá Boss Biển Sâu Huyền Thoại"},
        }
    }
}

local secretBossLookup = {}
for _, entry in ipairs(secretBossDatabase) do
    for _, b in ipairs(entry.bosses) do
        secretBossLookup[b.name:lower()] = b.name
        local clean = b.name:gsub("%s+", ""):lower()
        secretBossLookup[clean] = b.name
    end
    if entry.usefulFish then
        for _, f in ipairs(entry.usefulFish) do
            secretBossLookup[f.name:lower()] = f.name
            local clean = f.name:gsub("%s+", ""):lower()
            secretBossLookup[clean] = f.name
        end
    end
end
secretBossLookup["heavenpiercer turtle"] = "Heavenpiercer Turtle"
secretBossLookup["heaven piercer turtle"] = "Heavenpiercer Turtle"
secretBossLookup["heavenpiercer"] = "Heavenpiercer Turtle"
secretBossLookup["heaven piercer"] = "Heavenpiercer Turtle"
secretBossLookup["tigerfang whale"] = "Tigerfang Whale"
secretBossLookup["tiger fang whale"] = "Tigerfang Whale"
secretBossLookup["mountain dragonwhale"] = "Mountain Dragonwhale"
secretBossLookup["mountain dragon whale"] = "Mountain Dragonwhale"
secretBossLookup["mountain fish"] = "Mountain Fish"
secretBossLookup["nameless octoparasite"] = "Nameless Octoparasite"
secretBossLookup["octoparasite"] = "Nameless Octoparasite"
secretBossLookup["octoparasitic fish"] = "Octoparasitic Fish"
secretBossLookup["frost queenfish"] = "Frost Queenfish"
secretBossLookup["frost queen fish"] = "Frost Queenfish"
secretBossLookup["crimson bonefang"] = "Crimson Bonefang"
secretBossLookup["crimson bone fang"] = "Crimson Bonefang"
secretBossLookup["golden dragonfish"] = "Golden Dragonfish"
secretBossLookup["golden dragon fish"] = "Golden Dragonfish"
secretBossLookup["rainbow dragonfish"] = "Rainbow Dragonfish"
secretBossLookup["rainbow dragon fish"] = "Rainbow Dragonfish"
secretBossLookup["mirage lanternfish"] = "Mirage Lanternfish"
secretBossLookup["mirage lantern fish"] = "Mirage Lanternfish"
secretBossLookup["trueform jiaolongfish"] = "Trueform Jiaolongfish"
secretBossLookup["ascended perch"] = "Ascended Perch"
secretBossLookup["trueform perch"] = "Trueform Perch"
secretBossLookup["tiger mirefish"] = "Tiger Mirefish"
secretBossLookup["frost queenfish"] = "Frost Queenfish"
secretBossLookup["frost queen fish"] = "Frost Queenfish"
secretBossLookup["crimson bonefang"] = "Crimson Bonefang"
secretBossLookup["crimson bone fang"] = "Crimson Bonefang"
secretBossLookup["golden dragonfish"] = "Golden Dragonfish"
secretBossLookup["golden dragon fish"] = "Golden Dragonfish"
secretBossLookup["rainbow dragonfish"] = "Rainbow Dragonfish"
secretBossLookup["rainbow dragon fish"] = "Rainbow Dragonfish"
secretBossLookup["mirage lanternfish"] = "Mirage Lanternfish"
secretBossLookup["mirage lantern fish"] = "Mirage Lanternfish"

local Wiki = {
    craftMaterialFish = {
        -- Nguyên liệu chế Cần Heavenpiercer Rod
        ["Flying Fish Emperor"] = "Cần Heavenpiercer Rod",
        ["Flying Fish Empress"] = "Cần Heavenpiercer Rod",
        ["Heavenpiercer Turtle"] = "Cần Heavenpiercer & Mồi Rainbow",
        ["Heaven Piercer Turtle"] = "Cần Heavenpiercer & Mồi Rainbow",
        ["Rainbow Dragonfish"] = "Cần Heavenpiercer & Trả Quest",
        -- Nguyên liệu chế Cần Pure Diamond Rod
        ["Frost Kingfish"] = "Cần Pure Diamond & Mồi Frost",
        ["Frost Queenfish"] = "Cần Pure Diamond Rod",
        ["Sanguine Fish"] = "Cần Pure Diamond Rod",
        ["Draconic Koi"] = "Cần Pure Diamond Rod",
        -- Nguyên liệu chế Cần Sacred Bamboo Rod
        ["Nameless Octoparasite"] = "Cần Sacred Bamboo & Trả Quest",
        ["Reborn Puffer Beast"] = "Cần Sacred Bamboo & Trả Quest",
        ["Ascended Perch"] = "Cần Sacred Bamboo & Mồi Frost",
        ["Mountain Fish"] = "Cần Sacred Bamboo & Mồi Nameless",
        -- Nguyên liệu chế Mồi Nameless Bait
        ["Tiger Mirefish"] = "Mồi Nameless Bait",
        ["Mirage Lanternfish"] = "Mồi Nameless Bait",
        ["Octoparasitic Fish"] = "Mồi Nameless Bait (Gọi Boss Bạch Tuộc)",
        -- Nguyên liệu chế Mồi Rainbow Bait
        ["Colossal Tigerfish"] = "Mồi Rainbow Bait & Boss",
        ["Golden Guardian Fish"] = "Mồi Rainbow Bait",
        ["Crimson Electric Eel"] = "Mồi Rainbow Bait",
        -- Nguyên liệu chế Mồi Frost Bait
        ["Primordial Kunfish Overlord"] = "Mồi Frost Bait & Boss Realm",
        ["Warbringer Shark"] = "Mồi Frost Bait & Boss Realm",
        -- Nguyên liệu chế tạo khác & cá quý
        ["Catfish"] = "Nguyên liệu chế tạo cơ bản",
        ["Crimson Catfish"] = "Nguyên liệu đúc Cần Huyết Long",
        ["Scarlet Fish"] = "Nguyên liệu chế Cần Huyết Long",
        ["Elder Scarlet Fish"] = "Nguyên liệu Cần Huyết Long",
        ["Verdant Bonefang"] = "Nguyên liệu & Boss Mưa",
        ["Verdant Alligator Gar"] = "Nguyên liệu & Boss Mưa",
    },
    rarityColors = {
        ["Mythic"]    = Color3.fromRGB(248, 113, 113),  -- Đỏ neon Thần Thoại
        ["Legendary"] = Color3.fromRGB(250, 204, 21),   -- Vàng hoàng kim Huyền Thoại
        ["Epic"]      = Color3.fromRGB(192, 132, 252),  -- Tím mộng mơ Sử Thi
        ["Rare"]      = Color3.fromRGB(96, 165, 250),   -- Xanh dương Hiếm
        ["Uncommon"]  = Color3.fromRGB(52, 211, 153),   -- Xanh lục Đặc Biệt
        ["Common"]    = Color3.fromRGB(168, 150, 200),  -- Xám bạc Phổ Thông
    },
    wikiFishData = {
        -- 1. THẦN THOẠI (MYTHIC) & SIÊU BOSS (BẢO VỆ TUYỆT ĐỐI)
        {name = "Primordial Kunfish Overlord", rarity = "Mythic", keep = true, role = "CRAFT_BAIT_BOSS", use = "💎 Thần thú Boss Realm • Nguyên liệu Mồi Frost Bait • +30 Gems • BẢO VỆ", origin = "Đấu Trường Boss Realm", icon = "rbxassetid://10709791437"},
        {name = "Warbringer Shark", rarity = "Mythic", keep = true, role = "CRAFT_BAIT_BOSS", use = "💎 Thần thú Boss Realm • Nguyên liệu Mồi Frost Bait • +25 Gems • BẢO VỆ", origin = "Đấu Trường Boss Realm", icon = "rbxassetid://10709791437"},
        {name = "Nameless Octoparasite", rarity = "Mythic", keep = true, role = "CRAFT_ROD_QUEST", use = "💎 Siêu Boss Biển Sâu • Chế Cần Trúc Thánh • Trả Quest Đạo Sĩ (>=7M KG) & Mao Sơn", origin = "Phao Vùng Biển Sâu (Nameless Bait)", icon = "rbxassetid://10709791437"},
        {name = "Octoparasitic Fish", rarity = "Mythic", keep = true, role = "CRAFT_BAIT", use = "💎 Boss Bạch Tuộc • Nguyên liệu chế Mồi Nameless Bait • +50 Gems • BẢO VỆ", origin = "Phao Vùng Biển Sâu", icon = "rbxassetid://10709791437"},
        {name = "Rainbow Dragonfish", rarity = "Mythic", keep = true, role = "CRAFT_ROD_QUEST", use = "💎 Thần Ngư Vô Giá • Chế Cần Heavenpiercer Rod • Trả Quest Hạ Diêu Đệ (>=7M KG) & Tiên Nhân (>=6.5M KG)", origin = "Vùng Nước Ngầm Lòng Đất", icon = "rbxassetid://10709791437"},
        {name = "Mountain Fish", rarity = "Mythic", keep = true, role = "CRAFT_ROD_BAIT", use = "💎 Boss Đỉnh Núi • Chế Cần Trúc Thánh & Mồi Nameless Bait • Rơi Skill 5% • +20 Gems", origin = "Đảo Đỉnh Sương Mù (Mistpeak)", icon = "rbxassetid://10709791437"},
        {name = "Frost Kingfish", rarity = "Mythic", keep = true, role = "CRAFT_ROD_BAIT", use = "💎 Boss Bão Tuyết • Chế Cần Pure Diamond Rod & Mồi Frost Bait • Rơi Skill • +10 Gems", origin = "Đảo Băng Giá (Trời Bão Tuyết)", icon = "rbxassetid://10709791437"},
        {name = "Sanguine Fish", rarity = "Mythic", keep = true, role = "CRAFT_ROD", use = "💎 Boss Nắng Gắt • Chế Cần Pure Diamond Rod • Rơi Skill 10% • +20 Gems", origin = "Đảo Hổ Phách (Trời Nắng Gắt)", icon = "rbxassetid://10709791437"},
        {name = "Flying Fish Emperor", rarity = "Mythic", keep = true, role = "CRAFT_ROD", use = "💎 Boss Trời Gió • Chế Cần Heavenpiercer Rod • Rơi Skill 10% • +10 Gems", origin = "Đảo Cá Chép (Trời Gió)", icon = "rbxassetid://10709791437"},
        {name = "Crimson Electric Eel", rarity = "Mythic", keep = true, role = "CRAFT_BAIT", use = "💎 Boss Bão Sấm • Nguyên liệu chế Mồi Rainbow Bait • +5 Gems • BẢO VỆ", origin = "Đảo Tre (Trời Bão Sấm)", icon = "rbxassetid://10709791437"},
        {name = "Elder Scarlet Fish", rarity = "Mythic", keep = true, role = "BOSS", use = "💎 Boss Bão Sấm • Chế Cần Huyết Long • +5 Gems • KHÔNG BÁN", origin = "Đảo Tre (Trời Bão Sấm)", icon = "rbxassetid://10709791437"},
        {name = "Verdant Bonefang", rarity = "Mythic", keep = true, role = "BOSS", use = "💎 Boss Trời Mưa • Rơi Kỹ Năng 5% • +5 Gems • BẢO VỆ", origin = "Đảo Phóng Xạ (Trời Mưa)", icon = "rbxassetid://10709791437"},
        {name = "Tigerfang Whale", rarity = "Mythic", keep = true, role = "BOSS", use = "💎 Boss Sương Mù • Rơi Kỹ Năng Đòn Đánh • +5 Gems • BẢO VỆ", origin = "Đảo Quả Dừa (Trời Sương Mù)", icon = "rbxassetid://10709791437"},
        {name = "Colossal Tigerfish", rarity = "Mythic", keep = true, role = "CRAFT_BAIT_BOSS", use = "💎 Boss Đảo Chiến Trường • Nguyên liệu chế Mồi Rainbow Bait • BẢO VỆ", origin = "Đảo Chiến Trường", icon = "rbxassetid://10709791437"},
        {name = "Mountain Dragonwhale", rarity = "Mythic", keep = true, role = "BOSS", use = "💎 Thần Long Kình • Cá Boss Thần Thoại • Rơi Thần Trang • BẢO VỆ", origin = "Đảo Đỉnh Sương Mù", icon = "rbxassetid://10709791437"},
        {name = "Radiant Goldfish", rarity = "Mythic", keep = true, role = "BOSS", use = "💎 Kim Ngư Phát Sáng • Cá Boss Thần Thoại Cực Hiếm • BẢO VỆ", origin = "Vùng Biển Đặc Biệt", icon = "rbxassetid://10709791437"},
        {name = "Trueform Jiaolongfish", rarity = "Mythic", keep = true, role = "BOSS", use = "💎 Giao Long Chân Thân • Thần Ngư Thần Thoại • BẢO VỆ", origin = "Đảo Thống Trị", icon = "rbxassetid://10709791437"},
        {name = "Elder Jiaolong Dragonfish", rarity = "Mythic", keep = true, role = "BOSS", use = "💎 Cổ Đại Giao Long • Long Ngư Thần Thoại • BẢO VỆ", origin = "Đảo Thống Trị", icon = "rbxassetid://10709791437"},
        {name = "Adult Jiaolong Dragonfish", rarity = "Mythic", keep = true, role = "BOSS", use = "💎 Trưởng Thành Giao Long • Long Ngư Quý Hiếm • BẢO VỆ", origin = "Đảo Thống Trị", icon = "rbxassetid://10709791437"},
        {name = "Mutated Koi Whale", rarity = "Mythic", keep = true, role = "BOSS", use = "💎 Kình Ngư Biến Dị • Thần Thú Biển Sâu • BẢO VỆ", origin = "Vùng Biển Sâu", icon = "rbxassetid://10709791437"},

        -- 2. HUYỀN THOẠI (LEGENDARY) & BOSS CHÍNH
        {name = "Reborn Puffer Beast", rarity = "Legendary", keep = true, role = "CRAFT_ROD_QUEST", use = "⭐ Boss Bão Tuyết • Chế Cần Trúc Thánh • Trả Quest Giang Lão 3 (>=5M KG) • +10 Gems", origin = "Đảo Băng Giá (Trời Bão Tuyết)", icon = "rbxassetid://10709791437"},
        {name = "Heavenpiercer Turtle", rarity = "Legendary", keep = true, role = "CRAFT_ROD_BAIT", use = "⭐ Boss Sương Mù • Chế Cần Heavenpiercer Rod & Mồi Rainbow Bait • +5 Gems", origin = "Đảo Quả Dừa (Trời Sương Mù)", icon = "rbxassetid://10709791437"},
        {name = "Ascended Perch", rarity = "Legendary", keep = true, role = "CRAFT_ROD_BAIT", use = "⭐ Cá Boss • Chế Cần Trúc Thánh & Mồi Frost Bait • BẢO VỆ TUYỆT ĐỐI", origin = "Đảo Cá Chép", icon = "rbxassetid://10709791437"},
        {name = "Draconic Koi", rarity = "Legendary", keep = true, role = "CRAFT_ROD", use = "⭐ Boss Nắng Gắt • Chế Cần Pure Diamond Rod • +5 Gems • BẢO VỆ", origin = "Đảo Hổ Phách (Trời Nắng Gắt)", icon = "rbxassetid://10709791437"},
        {name = "Flying Fish Empress", rarity = "Legendary", keep = true, role = "CRAFT_ROD", use = "⭐ Boss Trời Gió • Chế Cần Heavenpiercer Rod • Rơi Skill 10% • +10 Gems", origin = "Đảo Cá Chép (Trời Gió)", icon = "rbxassetid://10709791437"},
        {name = "Verdant Alligator Gar", rarity = "Legendary", keep = true, role = "BOSS", use = "⭐ Boss Trời Mưa • Rơi Kỹ Năng 25% • +3 Gems • BẢO VỆ", origin = "Đảo Phóng Xạ (Trời Mưa)", icon = "rbxassetid://10709791437"},
        {name = "Verdant Grouper", rarity = "Legendary", keep = true, role = "BOSS", use = "⭐ Boss Trời Mưa • Rơi Kỹ Năng 25% • +3 Gems • BẢO VỆ", origin = "Đảo Phóng Xạ (Trời Mưa)", icon = "rbxassetid://10709791437"},
        {name = "Scarlet Fish", rarity = "Legendary", keep = true, role = "BOSS", use = "⭐ Boss Bão Sấm • Chế Cần Huyết Long • +3 Gems • BẢO VỆ", origin = "Đảo Tre (Trời Bão Sấm)", icon = "rbxassetid://10709791437"},
        {name = "Primordial Kunfish", rarity = "Legendary", keep = true, role = "BOSS", use = "⭐ Côn Ngư Viễn Cổ • Đấu trường Boss Realm • BẢO VỆ", origin = "Đảo Chiến Trường", icon = "rbxassetid://10709791437"},
        {name = "Adult Kunfish", rarity = "Legendary", keep = true, role = "BOSS", use = "⭐ Côn Ngư Trưởng Thành • Cá Boss Đảo Chiến Trường • BẢO VỆ", origin = "Đảo Chiến Trường", icon = "rbxassetid://10709791437"},
        {name = "Elder Kunfish", rarity = "Legendary", keep = true, role = "BOSS", use = "⭐ Côn Ngư Cổ Đại • Cá Boss Đảo Chiến Trường • BẢO VỆ", origin = "Đảo Chiến Trường", icon = "rbxassetid://10709791437"},
        {name = "Azure Carp", rarity = "Legendary", keep = true, role = "BOSS", use = "⭐ Lam Long Diệp • Cá Boss Huyền Thoại • BẢO VỆ", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Chromatic Koi", rarity = "Legendary", keep = true, role = "BOSS", use = "⭐ Thất Sắc Cẩm Lý • Cá Boss Huyền Thoại • BẢO VỆ", origin = "Đảo Hổ Phách", icon = "rbxassetid://10709791437"},
        {name = "Toxic Chromatic Koi", rarity = "Legendary", keep = true, role = "BOSS", use = "⭐ Thất Sắc Kịch Độc • Cá Boss Đảo Phóng Xạ • BẢO VỆ", origin = "Đảo Phóng Xạ", icon = "rbxassetid://10709791437"},
        {name = "Crimson Bonefang", rarity = "Legendary", keep = true, role = "BOSS", use = "⭐ Huyết Cốt Nha • Cá Boss Huyền Thoại • BẢO VỆ", origin = "Đảo Hổ Phách", icon = "rbxassetid://10709791437"},
        {name = "Crimson Bream Sovereign", rarity = "Legendary", keep = true, role = "BOSS", use = "⭐ Huyết Điêu Chúa • Cá Boss Huyền Thoại • BẢO VỆ", origin = "Đảo Thống Trị", icon = "rbxassetid://10709791437"},
        {name = "Silver Bream Sovereign", rarity = "Legendary", keep = true, role = "BOSS", use = "⭐ Ngân Điêu Chúa • Cá Boss Huyền Thoại • BẢO VỆ", origin = "Đảo Thống Trị", icon = "rbxassetid://10709791437"},
        {name = "Dreadmare Eel", rarity = "Legendary", keep = true, role = "BOSS", use = "⭐ Kinh Hoàng Hải Man • Cá Boss Biển Sâu • BẢO VỆ", origin = "Vùng Biển Sâu", icon = "rbxassetid://10709791437"},
        {name = "Golden Dragonfish", rarity = "Legendary", keep = true, role = "BOSS", use = "⭐ Kim Long Ngư • Cá Thần Huyền Thoại • BẢO VỆ", origin = "Đảo Thống Trị", icon = "rbxassetid://10709791437"},
        {name = "Serpent Fish", rarity = "Legendary", keep = true, role = "BOSS", use = "⭐ Mãng Xà Ngư • Cá Boss Đảo Phóng Xạ • BẢO VỆ", origin = "Đảo Phóng Xạ", icon = "rbxassetid://10709791437"},
        {name = "Trueform Perch", rarity = "Legendary", keep = true, role = "BOSS", use = "⭐ Chân Thân Cá Chép • Cá Boss Đảo Cá Chép • BẢO VỆ", origin = "Đảo Cá Chép", icon = "rbxassetid://10709791437"},
        {name = "Dark Kingfish", rarity = "Legendary", keep = true, role = "BOSS", use = "⭐ Hắc Ám Vương Ngư • Cá Boss Đảo Băng Giá • BẢO VỆ", origin = "Đảo Băng Giá", icon = "rbxassetid://10709791437"},
        {name = "Elder Chainbound Shark", rarity = "Legendary", keep = true, role = "BOSS", use = "⭐ Cổ Đại Tỏa Liên Sa • Thần Thú Biển Sâu • BẢO VỆ", origin = "Vùng Biển Sâu", icon = "rbxassetid://10709791437"},
        {name = "Glorious Elder Turtle", rarity = "Legendary", keep = true, role = "BOSS", use = "⭐ Huy Hoàng Cổ Quy • Thần Quy Huyền Thoại • BẢO VỆ", origin = "Đảo Quả Dừa", icon = "rbxassetid://10709791437"},
        {name = "Valentine Dolphin", rarity = "Legendary", keep = true, role = "EVENT", use = "⭐ Cá Heo Tình Nhân • Vật phẩm sự kiện Valentine 2026 • BẢO VỆ", origin = "Sự Kiện Valentine", icon = "rbxassetid://10709791437"},

        -- 3. SỬ THI (EPIC)
        {name = "Frost Queenfish", rarity = "Epic", keep = true, role = "CRAFT_ROD", use = "❄️ Nguyên liệu chế Cần Pure Diamond Rod • KHÔNG BÁN", origin = "Đảo Băng Giá", icon = "rbxassetid://10709791437"},
        {name = "Golden Guardian Fish", rarity = "Epic", keep = true, role = "CRAFT_BAIT", use = "🛡️ Boss Thống Trị • Nguyên liệu chế Mồi Rainbow Bait • KHÔNG BÁN", origin = "Đảo Thống Trị", icon = "rbxassetid://10709791437"},
        {name = "Tiger Mirefish", rarity = "Epic", keep = true, role = "CRAFT_BAIT", use = "🐅 Nguyên liệu chế Mồi Nameless Bait • KHÔNG BÁN", origin = "Đảo Đỉnh Sương Mù", icon = "rbxassetid://10709791437"},
        {name = "Mirage Lanternfish", rarity = "Epic", keep = true, role = "CRAFT_BAIT", use = "🏮 Nguyên liệu chế Mồi Nameless Bait • KHÔNG BÁN", origin = "Đảo Đỉnh Sương Mù", icon = "rbxassetid://10709791437"},
        {name = "Crimson Catfish", rarity = "Epic", keep = true, role = "CRAFT", use = "⚡ Nguyên liệu đúc Cần Huyết Long V2 • KHÔNG BÁN", origin = "Đảo Tre", icon = "rbxassetid://10709791437"},
        {name = "Chainbound Shark", rarity = "Epic", keep = false, role = "SELL", use = "💰 Bán lấy nhiều tiền vàng (Giá trị kinh tế cao)", origin = "Đảo Chiến Trường", icon = "rbxassetid://10709791437"},
        {name = "Glorious Oarfish", rarity = "Epic", keep = false, role = "SELL", use = "💰 Bán lấy nhiều tiền vàng (Giá trị kinh tế cao)", origin = "Vùng Biển Sâu", icon = "rbxassetid://10709791437"},
        {name = "Glorious Blobfish", rarity = "Epic", keep = false, role = "SELL", use = "💰 Bán lấy nhiều tiền vàng (Giá trị kinh tế cao)", origin = "Vùng Biển Sâu", icon = "rbxassetid://10709791437"},
        {name = "Golden Shark", rarity = "Epic", keep = false, role = "SELL", use = "💰 Bán lấy nhiều tiền vàng (Giá trị kinh tế cao)", origin = "Đảo Thống Trị", icon = "rbxassetid://10709791437"},
        {name = "Jiaolong Dragonfish", rarity = "Epic", keep = false, role = "SELL", use = "💰 Bán lấy nhiều tiền vàng (Giá trị kinh tế cao)", origin = "Đảo Thống Trị", icon = "rbxassetid://10709791437"},
        {name = "Kunfish III", rarity = "Epic", keep = false, role = "SELL", use = "💰 Bán lấy nhiều tiền vàng (Giá trị kinh tế cao)", origin = "Đảo Chiến Trường", icon = "rbxassetid://10709791437"},
        {name = "Terratiger Depthfish III", rarity = "Epic", keep = false, role = "SELL", use = "💰 Bán lấy nhiều tiền vàng (Giá trị kinh tế cao)", origin = "Đảo Đỉnh Sương Mù", icon = "rbxassetid://10709791437"},
        {name = "Elder Perch VIII", rarity = "Epic", keep = false, role = "SELL", use = "💰 Bán lấy nhiều tiền vàng (Cá Chép Thâm Niên VIII)", origin = "Đảo Cá Chép", icon = "rbxassetid://10709791437"},

        -- 4. HIẾM (RARE)
        {name = "Catfish", rarity = "Rare", keep = true, role = "CRAFT", use = "🐟 Nguyên liệu cơ bản ghép Cần Câu Sơ Cấp • KHÔNG BÁN", origin = "Đảo Tre", icon = "rbxassetid://10709791437"},
        {name = "Armored Battlefish", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền nâng cấp trang bị", origin = "Đảo Chiến Trường", icon = "rbxassetid://10709791437"},
        {name = "Darkness Battlefish", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền nâng cấp trang bị", origin = "Đảo Chiến Trường", icon = "rbxassetid://10709791437"},
        {name = "Dragon Battlefish", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền nâng cấp trang bị", origin = "Đảo Chiến Trường", icon = "rbxassetid://10709791437"},
        {name = "Phantom Battlefish", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền nâng cấp trang bị", origin = "Đảo Chiến Trường", icon = "rbxassetid://10709791437"},
        {name = "Crimson War Carp", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền nâng cấp trang bị", origin = "Đảo Chiến Trường", icon = "rbxassetid://10709791437"},
        {name = "Emerald War Carp", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền nâng cấp trang bị", origin = "Đảo Chiến Trường", icon = "rbxassetid://10709791437"},
        {name = "Dragonstride Carp", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền mua phụ kiện", origin = "Đảo Tre", icon = "rbxassetid://10709791437"},
        {name = "Dreadscale Grouper", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền nâng cấp", origin = "Đảo Phóng Xạ", icon = "rbxassetid://10709791437"},
        {name = "Elder Dragonhead Tilapia", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền nâng cấp", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Baby Jiaolong Dragonfish", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền vàng", origin = "Đảo Thống Trị", icon = "rbxassetid://10709791437"},
        {name = "Fleshripper Fish", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền mua mồi và phụ kiện", origin = "Đảo Phóng Xạ", icon = "rbxassetid://10709791437"},
        {name = "Kunfish I", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền nâng cấp trang bị", origin = "Đảo Chiến Trường", icon = "rbxassetid://10709791437"},
        {name = "Kunfish II", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền nâng cấp trang bị", origin = "Đảo Chiến Trường", icon = "rbxassetid://10709791437"},
        {name = "Runic Horn Grouper", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền nâng cấp", origin = "Đảo Phóng Xạ", icon = "rbxassetid://10709791437"},
        {name = "Sovereign Grass Carp", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền nâng cấp", origin = "Đảo Thống Trị", icon = "rbxassetid://10709791437"},
        {name = "Spiked Salmon", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền nâng cấp", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Stormscale Tautog", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền nâng cấp", origin = "Đảo Tre", icon = "rbxassetid://10709791437"},
        {name = "Sunscale Salmon", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền nâng cấp", origin = "Đảo Hổ Phách", icon = "rbxassetid://10709791437"},
        {name = "Terratiger Depthfish I", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền vàng", origin = "Đảo Đỉnh Sương Mù", icon = "rbxassetid://10709791437"},
        {name = "Terratiger Depthfish II", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền vàng", origin = "Đảo Đỉnh Sương Mù", icon = "rbxassetid://10709791437"},
        {name = "Azure Salmon", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền nâng cấp", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Elder Perch I", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền vàng (Cá Chép Thâm Niên I)", origin = "Đảo Cá Chép", icon = "rbxassetid://10709791437"},
        {name = "Elder Perch II", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền vàng (Cá Chép Thâm Niên II)", origin = "Đảo Cá Chép", icon = "rbxassetid://10709791437"},
        {name = "Elder Perch III", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền vàng (Cá Chép Thâm Niên III)", origin = "Đảo Cá Chép", icon = "rbxassetid://10709791437"},
        {name = "Elder Perch IV", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền vàng (Cá Chép Thâm Niên IV)", origin = "Đảo Cá Chép", icon = "rbxassetid://10709791437"},
        {name = "Elder Perch V", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền vàng (Cá Chép Thâm Niên V)", origin = "Đảo Cá Chép", icon = "rbxassetid://10709791437"},
        {name = "Elder Perch VI", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền vàng (Cá Chép Thâm Niên VI)", origin = "Đảo Cá Chép", icon = "rbxassetid://10709791437"},
        {name = "Elder Perch VII", rarity = "Rare", keep = false, role = "SELL", use = "💰 Bán kiếm tiền vàng (Cá Chép Thâm Niên VII)", origin = "Đảo Cá Chép", icon = "rbxassetid://10709791437"},

        -- 5. ĐẶC BIỆT (UNCOMMON) & PHỔ THÔNG (COMMON) - AUTO SELL
        {name = "Carp", rarity = "Common", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Perch", rarity = "Common", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Minnow", rarity = "Common", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Tilapia", rarity = "Common", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Trout", rarity = "Common", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Bamboo Fish", rarity = "Common", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Tre", icon = "rbxassetid://10709791437"},
        {name = "Ice Fish", rarity = "Common", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Băng Giá", icon = "rbxassetid://10709791437"},
        {name = "Radioactive Carp", rarity = "Common", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Phóng Xạ", icon = "rbxassetid://10709791437"},
        {name = "Coconut Crabfish", rarity = "Uncommon", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Quả Dừa", icon = "rbxassetid://10709791437"},
        {name = "Sovereign Fish", rarity = "Uncommon", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Thống Trị", icon = "rbxassetid://10709791437"},
        {name = "Silver Bass", rarity = "Uncommon", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Cá Chép", icon = "rbxassetid://10709791437"},
        {name = "Armored Carp", rarity = "Uncommon", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Chiến Trường", icon = "rbxassetid://10709791437"},
        {name = "Green Carp", rarity = "Uncommon", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Tre", icon = "rbxassetid://10709791437"},
        {name = "Bass", rarity = "Uncommon", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Gold Crucian Carp", rarity = "Uncommon", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Golden Carp", rarity = "Uncommon", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Goldfin Grass Carp", rarity = "Uncommon", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Azurefin Grass Carp", rarity = "Uncommon", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Blackfin Grass Carp", rarity = "Uncommon", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Crimsonfin Grass Carp", rarity = "Uncommon", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Rosefin Grass Carp", rarity = "Uncommon", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Volt Grass Carp", rarity = "Uncommon", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Darkscale Fish", rarity = "Uncommon", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Crimson Carp", rarity = "Uncommon", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Tre", icon = "rbxassetid://10709791437"},
        {name = "Emerald Carp", rarity = "Uncommon", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Platinum Carp", rarity = "Uncommon", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Royal Carp", rarity = "Uncommon", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Boundeye Fish", rarity = "Uncommon", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Cá Chép", icon = "rbxassetid://10709791437"},
        {name = "Sand Anchovy", rarity = "Common", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Arrogant Fish", rarity = "Uncommon", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Cá Chép", icon = "rbxassetid://10709791437"},
        {name = "Platescale Tilapia", rarity = "Common", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Grass Tilapia", rarity = "Common", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Dragonhead Tilapia", rarity = "Uncommon", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Ironhorn Flounder", rarity = "Uncommon", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Tre", icon = "rbxassetid://10709791437"},
        {name = "Horned Silver Carp", rarity = "Uncommon", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Lapis Fish", rarity = "Uncommon", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Khởi Đầu", icon = "rbxassetid://10709791437"},
        {name = "Scarlet Fringehead", rarity = "Uncommon", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Tre", icon = "rbxassetid://10709791437"},
        {name = "Abyssal Glow Fish", rarity = "Common", keep = false, role = "SELL", use = "💰 Bán tự động dọn trống balo", origin = "Đảo Phóng Xạ", icon = "rbxassetid://10709791437"},
    }
}

function Wiki.GetItemRawName(item)
    if not item then return "" end
    local v = item:FindFirstChild("ValueName")
    if v and v:IsA("StringValue") and #v.Value > 0 then
        local raw = tostring(v.Value)
        local base = raw:match("^%s*([^|]+)")
        return base and base:gsub("%s+$", "") or raw
    end
    local attName = item:GetAttribute("FishName") or item:GetAttribute("Name") or item:GetAttribute("ItemName")
    if attName and #tostring(attName) > 0 then
        local raw = tostring(attName)
        local base = raw:match("^%s*([^|]+)")
        return base and base:gsub("%s+$", "") or raw
    end
    local raw = tostring(item.Name or "")
    local base = raw:match("^%s*([^|]+)")
    return base and base:gsub("%s+$", "") or raw
end

function Wiki.IsItemFavorited(item)
    if not item then return false end

    -- 1. Kiểm tra trực tiếp tên item (Game Roblox thêm " | Favorite" trực tiếp vào item.Name)
    local itemName = tostring(item.Name or "")
    if itemName:find("Favorite", 1, true) or itemName:find("Favourite", 1, true) then
        return true
    end

    -- 2. Kiểm tra các attribute của item
    if item:GetAttribute("IsFavorite") == true or item:GetAttribute("Favorite") == true or item:GetAttribute("Locked") == true then
        return true
    end

    -- 3. Kiểm tra các đối tượng con
    local favVal = item:FindFirstChild("Favorite") or item:FindFirstChild("Favourite")
    if favVal and (favVal.Value == true or favVal.Value == 1) then return true end
    local lockVal = item:FindFirstChild("Locked")
    if lockVal and (lockVal.Value == true or lockVal.Value == 1) then return true end

    local ok, result = pcall(function()
        for _, child in ipairs(item:GetChildren()) do
            if child.Name:find("Favorite", 1, true) or child.Name:find("Favourite", 1, true) then
                return true
            end
        end
        return false
    end)
    if ok and result then return true end

    return false
end

function Wiki.IsSecretBossFish(item)
    if not item then return false end
    local rawName = Wiki.GetItemRawName(item):lower()
    local fullName = tostring(item.Name or ""):lower()

    for bLower, _ in pairs(secretBossLookup) do
        if rawName:find(bLower, 1, true) or fullName:find(bLower, 1, true) then
            return true
        end
    end

    if item:GetAttribute("Boss") == true or item:GetAttribute("Secret") == true or item:GetAttribute("IsBoss") == true then
        return true
    end

    return false
end

function Wiki.IsMutatedFish(item)
    if not item then return false end
    local name = tostring(item.Name or "")
    for _, kw in ipairs({"Shiny", "Giant", "Golden", "Albino", "Corrupted", "Colossal", "Heavyweight", "Dark", "Radiant"}) do
        if name:find(kw) then return true end
    end
    local mutVal = item:FindFirstChild("Mutation")
    if mutVal and tostring(mutVal.Value) ~= "" and tostring(mutVal.Value) ~= "None" then
        return true
    end
    for _, attr in ipairs({"Mutation", "Mutated", "Variant"}) do
        local v = item:GetAttribute(attr)
        if v and tostring(v) ~= "" and tostring(v) ~= "None" then
            return true
        end
    end
    return false
end

function Wiki.IsPlayerIndexUnlocked(fishName)
    if not fishName or #fishName == 0 then return false end
    local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
    if not pData or not pData:FindFirstChild("Index") then return false end
    local fishLower = fishName:lower():gsub("^%s+", ""):gsub("%s+$", "")

    local direct = pData.Index:FindFirstChild(fishName)
    if direct and direct:IsA("BoolValue") and direct.Value == true then
        return true
    end

    for _, idxItem in ipairs(pData.Index:GetChildren()) do
        if idxItem.Name:lower():gsub("^%s+", ""):gsub("%s+$", "") == fishLower then
            if idxItem:IsA("BoolValue") and idxItem.Value == true then
                return true
            end
        end
    end
    return false
end

function Wiki.GetPlayerFishCount(fishName)
    local count = 0
    local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
    if not pData then return 0 end
    local fishLower = tostring(fishName or ""):lower():gsub("^%s+", ""):gsub("%s+$", "")
    if #fishLower == 0 then return 0 end

    local function cleanFishName(str)
        local s = tostring(str or ""):lower()
        for _, kw in ipairs({"shiny", "giant", "golden", "albino", "corrupted", "colossal", "heavyweight", "dark", "radiant", "spectral", "sparkling"}) do
            s = s:gsub("%f[%a]" .. kw .. "%f[%A]", "")
        end
        return s:match("^%s*(.-)%s*$") or s
    end

    local function checkFolder(folder)
        if not folder then return end
        for _, item in ipairs(folder:GetChildren()) do
            local rawName = Wiki.GetItemRawName(item):lower():gsub("^%s+", ""):gsub("%s+$", "")
            local cleanRaw = cleanFishName(rawName)

            local isMatch = (rawName == fishLower) or (cleanRaw == fishLower)
            if not isMatch and (rawName:find(fishLower, 1, true) or fishLower:find(rawName, 1, true)) then
                isMatch = true
            end

            if isMatch then
                local qtyVal = item:FindFirstChild("Quantity") or item:FindFirstChild("Count") or item:FindFirstChild("Amount") or item:FindFirstChild("Stack")
                local qty = (qtyVal and tonumber(qtyVal.Value)) or (item:IsA("NumberValue") and tonumber(item.Value)) or 1
                count = count + qty
            end
        end
    end

    checkFolder(pData:FindFirstChild("Inventory"))
    checkFolder(pData:FindFirstChild("Hotbar"))
    return count
end

function Wiki.IsEssentialKeepItem(item)
    if not item then return false end
    if Wiki.IsSecretBossFish(item) then return true end
    if Wiki.IsMutatedFish(item) then return true end

    local rawName = Wiki.GetItemRawName(item)
    if Wiki.craftMaterialFish[rawName] or Wiki.craftMaterialFish[item.Name] then return true end

    if Config.AutoFavouriteFish and (rawName == Config.FavouriteFishName or item.Name == Config.FavouriteFishName) then
        return true
    end

    local weightStr = tostring(item.Name or ""):match("|%s*([%d%.]+)")
    local wNum = tonumber(weightStr)
    if wNum and wNum >= 1000000 then
        return true
    end

    local rawLower = rawName:lower()
    for _, f in ipairs(Wiki.wikiFishData) do
        if f.keep then
            local fLower = f.name:lower()
            if rawLower == fLower or rawLower:find(fLower, 1, true) or fLower:find(rawLower, 1, true) then
                return true
            end
        end
    end
    return false
end

Wiki.baitIngredients = {
    ["Nameless Bait"] = {"Mountain Fish", "Octoparasitic Fish", "Mirage Lanternfish", "Tiger Mirefish"},
    ["Frost Bait"] = {"Primordial Kunfish Overlord", "Warbringer Shark", "Frost Kingfish", "Ascended Perch"},
    ["Rainbow Bait"] = {"Heavenpiercer Turtle", "Colossal Tigerfish", "Crimson Electric Eel", "Golden Guardian Fish"},
}

Wiki.allBaitFishSet = {
    ["mountain fish"] = true,
    ["octoparasitic fish"] = true,
    ["mirage lanternfish"] = true,
    ["tiger mirefish"] = true,
    ["primordial kunfish overlord"] = true,
    ["warbringer shark"] = true,
    ["frost kingfish"] = true,
    ["ascended perch"] = true,
    ["heavenpiercer turtle"] = true,
    ["colossal tigerfish"] = true,
    ["crimson electric eel"] = true,
    ["golden guardian fish"] = true,
}

Wiki.allRodFishSet = {
    ["flying fish emperor"] = true,
    ["flying fish empress"] = true,
    ["heavenpiercer turtle"] = true,
    ["rainbow dragonfish"] = true,
    ["nameless octoparasite"] = true,
    ["reborn puffer beast"] = true,
    ["ascended perch"] = true,
    ["mountain fish"] = true,
    ["frost kingfish"] = true,
    ["frost queenfish"] = true,
    ["sanguine fish"] = true,
    ["draconic koi"] = true,
}

function Wiki.IsProtectedFish(item)
    if not item then return false, "Không xác định" end
    local cleanName = Wiki.GetItemRawName(item)
    local cleanLower = cleanName:lower()
    local weight = (item:FindFirstChild("Weight") and item.Weight.Value) or 0

    if Wiki.allRodFishSet and Wiki.allRodFishSet[cleanLower] then
        return true, "Cá Chế Cần"
    end
    if Wiki.allBaitFishSet and Wiki.allBaitFishSet[cleanLower] then
        return true, "Cá Chế Mồi"
    end
    if Wiki.IsSecretBossFish(item) or (Wiki.secretBossSet and Wiki.secretBossSet[cleanLower]) then
        return true, "Cá Secret Boss"
    end
    if Wiki.IsMutatedFish(item) then
        return true, "Cá Đột Biến"
    end
    if Wiki.vipQuestFishSet and Wiki.vipQuestFishSet[cleanLower] and weight >= 5000000 then
        return true, "Cá Quest VIP"
    end
    if weight >= 1000000 then
        return true, "Cá Siêu Nặng (≥1M KG)"
    end
    if Wiki.IsEssentialKeepItem(item) then
        return true, "Cá Cần Giữ"
    end
    return false, "Cá Thường (Rác)"
end

Wiki.temporarilyUnlockedBaitFish = {}

function Wiki.IsBaitIngredient(fishName, baitName)
    local fn = tostring(fishName or ""):lower():gsub("^%s+", ""):gsub("%s+$", "")
    if baitName and baitName ~= "All" and Wiki.baitIngredients[baitName] then
        for _, ing in ipairs(Wiki.baitIngredients[baitName]) do
            local ingLower = ing:lower()
            if fn == ingLower or fn:find(ingLower, 1, true) or ingLower:find(fn, 1, true) then
                return true
            end
        end
        return false
    else
        return Wiki.allBaitFishSet[fn] == true
    end
end

function Wiki.UnlockBaitFish(baitName, silent)
    local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
    if not pData then
        if not silent then ShowNotification("Mở Khóa Mồi", "Không tìm thấy dữ liệu túi đồ!", "WARN", 4) end
        return 0
    end

    local toUnlock = {}
    local folders = {}
    if pData:FindFirstChild("Inventory") then table.insert(folders, pData.Inventory) end
    if pData:FindFirstChild("Hotbar") then table.insert(folders, pData.Hotbar) end

    for _, folder in ipairs(folders) do
        for _, item in ipairs(folder:GetChildren()) do
            if Wiki.IsItemFavorited(item) then
                local rawName = Wiki.GetItemRawName(item)
                if Wiki.IsBaitIngredient(rawName, baitName) then
                    table.insert(toUnlock, item)
                end
            end
        end
    end

    local bLabel = (baitName and baitName ~= "All") and ("[" .. baitName .. "]") or "chế mồi"
    if #toUnlock == 0 then
        if not silent then
            ShowNotification("Mở Khóa Mồi", string.format("Không có cá nguyên liệu %s nào đang bị khóa trong balo.", bLabel), "INFO", 4)
        end
        return 0
    end

    local unlockedCount = 0
    for _, item in ipairs(toUnlock) do
        if Events and Events:FindFirstChild("FavoriteItem") then
            Events.FavoriteItem:FireServer(item)
            unlockedCount = unlockedCount + 1
            local raw = Wiki.GetItemRawName(item):lower()
            Wiki.temporarilyUnlockedBaitFish[raw] = true
            task.wait(0.04)
        end
    end

    if not silent then
        ShowNotification("MỞ KHÓA MỒI THÀNH CÔNG", string.format("Đã mở khóa %d con cá làm mồi %s! Bạn có thể chế tạo ngay.", unlockedCount, (baitName and baitName ~= "All") and ("[" .. baitName .. "]") or "3 Loại Mồi"), "SUCCESS", 6)
    end
    return unlockedCount
end

function Wiki.LockBaitFish(baitName, silent)
    local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
    if not pData then
        if not silent then ShowNotification("Khóa Lại Mồi", "Không tìm thấy dữ liệu túi đồ!", "WARN", 4) end
        return 0
    end

    local toLock = {}
    local folders = {}
    if pData:FindFirstChild("Inventory") then table.insert(folders, pData.Inventory) end
    if pData:FindFirstChild("Hotbar") then table.insert(folders, pData.Hotbar) end

    for _, folder in ipairs(folders) do
        for _, item in ipairs(folder:GetChildren()) do
            if not Wiki.IsItemFavorited(item) then
                local rawName = Wiki.GetItemRawName(item)
                if Wiki.IsBaitIngredient(rawName, baitName) then
                    table.insert(toLock, item)
                end
            end
        end
    end

    local bLabel = (baitName and baitName ~= "All") and ("[" .. baitName .. "]") or "chế mồi"
    if #toLock == 0 then
        if not silent then
            ShowNotification("Khóa Lại Mồi", string.format("Tất cả cá nguyên liệu %s trong balo đã được khóa an toàn.", bLabel), "INFO", 4)
        end
        return 0
    end

    local lockedCount = 0
    for _, item in ipairs(toLock) do
        if Events and Events:FindFirstChild("FavoriteItem") then
            Events.FavoriteItem:FireServer(item)
            lockedCount = lockedCount + 1
            local raw = Wiki.GetItemRawName(item):lower()
            Wiki.temporarilyUnlockedBaitFish[raw] = nil
            task.wait(0.04)
        end
    end

    if not silent then
        ShowNotification("KHÓA MỒI THÀNH CÔNG", string.format("Đã khóa bảo vệ lại %d con cá làm mồi %s an toàn trước AutoSell!", lockedCount, (baitName and baitName ~= "All") and ("[" .. baitName .. "]") or "3 Loại Mồi"), "SUCCESS", 6)
    end
    return lockedCount
end

function Wiki.UnlockAllKeepFish()
    local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
    if not pData then
        ShowNotification("Mở Khóa Balo", "Không tìm thấy dữ liệu túi đồ người chơi!", "WARN", 4)
        return 0
    end

    local toUnlock = {}
    local folders = {}
    if pData:FindFirstChild("Inventory") then table.insert(folders, pData.Inventory) end
    if pData:FindFirstChild("Hotbar") then table.insert(folders, pData.Hotbar) end

    for _, folder in ipairs(folders) do
        for _, item in ipairs(folder:GetChildren()) do
            if Wiki.IsItemFavorited(item) then
                if Wiki.IsEssentialKeepItem(item) then
                    table.insert(toUnlock, item)
                end
            end
        end
    end

    if #toUnlock == 0 then
        ShowNotification("Mở Khóa Cá Quý", "Không có cá cần giữ nào đang bị khóa trong balo.", "INFO", 4)
        return 0
    end

    local unlockedCount = 0
    for _, item in ipairs(toUnlock) do
        if Events and Events:FindFirstChild("FavoriteItem") then
            Events.FavoriteItem:FireServer(item)
            unlockedCount = unlockedCount + 1
            task.wait(0.04)
        end
    end

    ShowNotification("MỞ KHÓA THÀNH CÔNG", string.format("Đã mở khóa %d con cá quý / boss / nguyên liệu!", unlockedCount), "WARN", 6)
    return unlockedCount
end

-- Mở khóa TẤT CẢ cá trong balo (cả cá giữ lẫn cá bán)
function Wiki.UnlockAllFish()
    local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
    if not pData then
        ShowNotification("Mở Khóa", "Không tìm thấy dữ liệu túi đồ!", "WARN", 4)
        return 0
    end

    local toUnlock = {}
    local folders = {}
    if pData:FindFirstChild("Inventory") then table.insert(folders, pData.Inventory) end
    if pData:FindFirstChild("Hotbar") then table.insert(folders, pData.Hotbar) end

    for _, folder in ipairs(folders) do
        for _, item in ipairs(folder:GetChildren()) do
            if Wiki.IsItemFavorited(item) then
                table.insert(toUnlock, item)
            end
        end
    end

    if #toUnlock == 0 then
        ShowNotification("Mở Khóa Tất Cả", "Không có cá nào đang bị khóa trong balo.", "INFO", 4)
        return 0
    end

    local count = 0
    for _, item in ipairs(toUnlock) do
        if Events and Events:FindFirstChild("FavoriteItem") then
            Events.FavoriteItem:FireServer(item)
            count = count + 1
            task.wait(0.04)
        end
    end

    ShowNotification("MỞ KHÓA TẤT CẢ", string.format("Đã mở khóa %d con cá (kể cả cá quý)! AutoSell có thể bán tất cả.", count), "SUCCESS", 6)
    return count
end

function Wiki.UnlockAllUnnecessaryFish()
    local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
    if not pData then
        ShowNotification("Mở Khóa Balo", "Không tìm thấy dữ liệu túi đồ người chơi!", "WARN", 4)
        return 0
    end

    local toUnlock = {}
    local folders = {}
    if pData:FindFirstChild("Inventory") then table.insert(folders, pData.Inventory) end
    if pData:FindFirstChild("Hotbar") then table.insert(folders, pData.Hotbar) end

    for _, folder in ipairs(folders) do
        for _, item in ipairs(folder:GetChildren()) do
            if Wiki.IsItemFavorited(item) then
                if not Wiki.IsEssentialKeepItem(item) then
                    table.insert(toUnlock, item)
                end
            end
        end
    end

    if #toUnlock == 0 then
        ShowNotification("Mở Khóa Balo", "Không có cá không cần thiết nào đang bị khóa trong balo.", "INFO", 4)
        return 0
    end

    local unlockedCount = 0
    for _, item in ipairs(toUnlock) do
        if Events and Events:FindFirstChild("FavoriteItem") then
            Events.FavoriteItem:FireServer(item)
            unlockedCount = unlockedCount + 1
            task.wait(0.04)
        end
    end

    ShowNotification("MỞ KHÓA THÀNH CÔNG", string.format("Đã mở khóa %d con cá không cần thiết! AutoSell có thể bán ngay.", unlockedCount), "SUCCESS", 6)
    return unlockedCount
end

function Wiki.LockAllKeepFish()
    Wiki.temporarilyUnlockedBaitFish = {}
    local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
    if not pData then
        ShowNotification("Khóa Bảo Vệ", "Không tìm thấy dữ liệu túi đồ!", "WARN", 4)
        return 0
    end

    local toLock = {}
    local folders = {}
    if pData:FindFirstChild("Inventory") then table.insert(folders, pData.Inventory) end
    if pData:FindFirstChild("Hotbar") then table.insert(folders, pData.Hotbar) end

    for _, folder in ipairs(folders) do
        for _, item in ipairs(folder:GetChildren()) do
            if not Wiki.IsItemFavorited(item) then
                if Wiki.IsEssentialKeepItem(item) then
                    table.insert(toLock, item)
                end
            end
        end
    end

    if #toLock == 0 then
        ShowNotification("Khóa Bảo Vệ", "Tất cả cá cần giữ trong balo đã được khóa an toàn.", "INFO", 4)
        return 0
    end

    local lockedCount = 0
    for _, item in ipairs(toLock) do
        if Events and Events:FindFirstChild("FavoriteItem") then
            Events.FavoriteItem:FireServer(item)
            lockedCount = lockedCount + 1
            task.wait(0.04)
        end
    end

    ShowNotification("KHÓA THÀNH CÔNG", string.format("Đã khóa bảo vệ an toàn %d con cá quý / boss / nguyên liệu!", lockedCount), "SUCCESS", 6)
    return lockedCount
end

function Wiki.ToggleLockSpecificFish(fishName, targetKeepState)
    local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
    if not pData then return end
    local fishLower = fishName:lower()
    local toggled = 0

    local folders = {}
    if pData:FindFirstChild("Inventory") then table.insert(folders, pData.Inventory) end
    if pData:FindFirstChild("Hotbar") then table.insert(folders, pData.Hotbar) end

    local itemsFound = {}
    local anyLocked = false

    for _, folder in ipairs(folders) do
        for _, item in ipairs(folder:GetChildren()) do
            local rawName = Wiki.GetItemRawName(item):lower()
            local instName = tostring(item.Name or ""):lower()
            if rawName == fishLower or instName == fishLower or rawName:find(fishLower, 1, true) then
                local isFav = Wiki.IsItemFavorited(item)
                if isFav then anyLocked = true end
                table.insert(itemsFound, {item = item, isFav = isFav})
            end
        end
    end

    if #itemsFound == 0 then
        ShowNotification("Thao Tác Cá", string.format("Không có con [%s] nào trong túi để chuyển trạng thái.", fishName), "INFO", 4)
        return
    end

    local wantLock = targetKeepState
    if wantLock == nil then
        wantLock = not anyLocked
    end

    for _, data in ipairs(itemsFound) do
        if wantLock and not data.isFav then
            if Events and Events:FindFirstChild("FavoriteItem") then
                Events.FavoriteItem:FireServer(data.item)
                toggled = toggled + 1
                task.wait(0.04)
            end
        elseif not wantLock and data.isFav then
            if Events and Events:FindFirstChild("FavoriteItem") then
                Events.FavoriteItem:FireServer(data.item)
                toggled = toggled + 1
                task.wait(0.04)
            end
        end
    end

    local actText = wantLock and "Đã khóa bảo vệ" or "Đã mở khóa"
    if toggled > 0 then
        ShowNotification("Thao Tác Cá", string.format("%s %d con [%s] thành công!", actText, toggled, fishName), "SUCCESS", 5)
    else
        ShowNotification("Thao Tác Cá", string.format("Tất cả con [%s] đã ở trạng thái %s rồi.", fishName, wantLock and "khóa" or "mở khóa"), "INFO", 4)
    end
end

function Wiki.ResolveFishIcon(fishName, defaultIcon)
    local ok, icon = pcall(function()
        -- 1. Trích xuất trực tiếp icon hình ảnh thật từ cây Indexlist trong PlayerGui
        local playerGui = LocalPlayer:FindFirstChild("PlayerGui")
        if playerGui then
            local mainGui = playerGui:FindFirstChild("MainGui")
            if mainGui then
                local indexList = mainGui:FindFirstChild("Indexlist", true)
                if indexList then
                    local frame = indexList:FindFirstChild(fishName)
                    if frame then
                        local img = frame:FindFirstChild("Image", true)
                        if img and img:IsA("ImageLabel") and #img.Image > 0 then
                            return img.Image
                        end
                    end
                end
            end
        end

        -- 2. Trích xuất từ ReplicatedStorage
        if ReplicatedStorage then
            local found = ReplicatedStorage:FindFirstChild(fishName, true)
            if found then
                for _, prop in ipairs({"Icon", "Image", "Texture", "Thumbnail"}) do
                    local p = found:FindFirstChild(prop)
                    if p and p:IsA("StringValue") and #p.Value > 0 then
                        return p.Value
                    end
                    local att = found:GetAttribute(prop)
                    if att and #tostring(att) > 0 then
                        return tostring(att)
                    end
                end
                if found:IsA("Decal") or found:IsA("Texture") then
                    return found.Texture
                end
            end
        end

        return nil
    end)

    if ok and icon and #tostring(icon) > 0 then
        return tostring(icon)
    end

    return defaultIcon or "rbxassetid://10709791437"
end


local secretBossState = {
    active = false,
    currentMap = nil,
    targetIsland = nil,
    requiredPower = 0,
    statusText = "Đang chờ thông báo...",
    isCatchingTarget = false,
    lastSkipTime = 0,
    minigameStartTime = 0,
    isHopping = (pendingWeatherHopData ~= nil),
    currentTargetWeather = (pendingWeatherHopData and pendingWeatherHopData.TargetWeather) or Config.TargetWeather,
    weatherHopToggle = nil,
}

local statusLabelSecretBoss = nil
local bossTogglesMap = {}

local weatherTotems = {
    {name = "Totem Bão Sấm (Bamboo Isle)", island = "Đảo Tre (Bamboo Isle)", weather = "Thunderstorm (Bão Sấm)", pos = Vector3.new(-1242.0, 8.5, -195.0)},
    {name = "Totem Bão Tuyết (Frost Isle)", island = "Đảo Băng (Frost Isle)", weather = "Snowy (Bão Tuyết)", pos = Vector3.new(-1390.0, 10.2, -1515.0)},
    {name = "Totem Sương Mù (Coconut Isle)", island = "Đảo Quả Dừa (Coconut Isle)", weather = "Foggy (Sương Mù)", pos = Vector3.new(1475.0, 9.8, -1415.0)},
    {name = "Totem Nắng Gắt (Amber Isle)", island = "Đảo Hổ Phách (Amber Isle)", weather = "Blazing Sun (Nắng Gắt)", pos = Vector3.new(1275.0, 9.5, 1450.0)},
}

local visualSpoofState = {
    fakeTicket = 0,
    fakeGems = 0,
    hookedGemLabels = {}
}

function visualSpoofState.GetFileName()
    local accName = (LocalPlayer and LocalPlayer.Name) or "Default"
    local safeAcc = accName:gsub("[^%w_]", "")
    if #safeAcc == 0 then safeAcc = "Default" end
    return "heavyweight_visual_spoof_" .. safeAcc .. ".json"
end

function visualSpoofState.Save()
    if not writefile then return end
    pcall(function()
        local data = {
            fakeTicket = visualSpoofState.fakeTicket or 0,
            fakeGems = visualSpoofState.fakeGems or 0,
        }
        writefile(visualSpoofState.GetFileName(), HttpService:JSONEncode(data))
    end)
end

function visualSpoofState.Load()
    local fileName = visualSpoofState.GetFileName()
    if not readfile or not isfile or not isfile(fileName) then return end
    pcall(function()
        local raw = readfile(fileName)
        if raw and #raw > 0 then
            local dec = HttpService:JSONDecode(raw)
            if type(dec) == "table" then
                if dec.fakeTicket and tonumber(dec.fakeTicket) then
                    visualSpoofState.fakeTicket = tonumber(dec.fakeTicket)
                end
                if dec.fakeGems and tonumber(dec.fakeGems) then
                    visualSpoofState.fakeGems = tonumber(dec.fakeGems)
                end
            end
        end
    end)
end

function visualSpoofState.UpdateTextLabelWithGems(label, gNum)
    if not label or not label:IsA("TextLabel") then return end
    local oldText = label.Text
    local formatted = FormatWithSpaces(gNum)

    if oldText:find("💎") then
        if oldText:find("^%s*💎") then
            label.Text = "💎 " .. formatted
        elseif oldText:find("💎%s*$") then
            label.Text = formatted .. " 💎"
        else
            label.Text = "💎 " .. formatted
        end
    elseif oldText:find("🔷") then
        label.Text = "🔷 " .. formatted
    elseif oldText:lower():find("gem") then
        if oldText:find(":") then
            label.Text = "Gems: " .. formatted
        else
            label.Text = formatted .. " Gems"
        end
    elseif oldText:lower():find("diamond") then
        label.Text = formatted .. " Diamonds"
    else
        label.Text = formatted
    end
end

function visualSpoofState.HookGemLabel(label, gNum)
    if visualSpoofState.hookedGemLabels[label] then return end
    visualSpoofState.hookedGemLabels[label] = true
    pcall(function()
        label:GetPropertyChangedSignal("Text"):Connect(function()
            if visualSpoofState.fakeGems and visualSpoofState.fakeGems > 0 then
                local targetText = FormatWithSpaces(visualSpoofState.fakeGems)
                if not label.Text:find(targetText, 1, true) then
                    task.defer(function()
                        if visualSpoofState.fakeGems and visualSpoofState.fakeGems > 0 then
                            visualSpoofState.UpdateTextLabelWithGems(label, visualSpoofState.fakeGems)
                        end
                    end)
                end
            end
        end)
    end)
end

function visualSpoofState.ScanPlayerGuiGems(gNum)
    local pg = LocalPlayer:FindFirstChild("PlayerGui")
    if not pg then return end

    for _, desc in ipairs(pg:GetDescendants()) do
        if desc:IsA("TextLabel") and desc.Visible then
            local selfName = desc.Name:lower()
            local pName = desc.Parent and desc.Parent.Name:lower() or ""
            local gpName = desc.Parent and desc.Parent.Parent and desc.Parent.Parent.Name:lower() or ""
            local txtLower = desc.Text:lower()

            local isGem = false

            -- 1. Direct name match
            if selfName:find("gem") or selfName:find("diamond") or selfName:find("ruby") or selfName:find("da_quy") or selfName:find("da quy") then
                isGem = true
            -- 2. Parent or grandparent match
            elseif pName:find("gem") or pName:find("diamond") or pName:find("ruby") or pName:find("currency2") then
                isGem = true
            elseif gpName:find("gem") or gpName:find("diamond") then
                isGem = true
            -- 3. Text content contains gem symbols or keywords
            elseif txtLower:find("gem") or desc.Text:find("💎") or desc.Text:find("🔷") or txtLower:find("diamond") then
                isGem = true
            -- 4. Check siblings for gem icon / image
            elseif desc.Parent then
                for _, sib in ipairs(desc.Parent:GetChildren()) do
                    if sib ~= desc then
                        local sibName = sib.Name:lower()
                        if sibName:find("gem") or sibName:find("diamond") or sibName:find("ruby") then
                            isGem = true
                            break
                        end
                        if sib:IsA("ImageLabel") or sib:IsA("ImageButton") then
                            local img = tostring(sib.Image):lower()
                            if img:find("gem") or img:find("diamond") or img:find("ruby") then
                                isGem = true
                                break
                            end
                        end
                    end
                end
            end

            -- Loại trừ nếu nhầm sang vé, tiền cash, level, exp, hoặc cân nặng cá (kg)
            if isGem then
                local isExcluded = selfName:find("ticket") or pName:find("ticket") or desc.Text:find("Vé")
                    or selfName:find("cash") or pName:find("cash") or desc.Text:find("%$")
                    or selfName:find("level") or selfName:find("exp")
                    or desc.Text:find("kg") or desc.Text:find("KG")
                if not isExcluded then
                    visualSpoofState.UpdateTextLabelWithGems(desc, gNum)
                    visualSpoofState.HookGemLabel(desc, gNum)
                end
            end
        end
    end
end

function visualSpoofState.ScanPlayerDataGems(gNum)
    local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
    if not pData then return end

    -- 1. Quét sâu tất cả con cháu trong pData
    for _, item in ipairs(pData:GetDescendants()) do
        local iName = item.Name:lower()
        if iName:find("gem") or iName:find("diamond") or iName:find("ruby") or iName:find("crystal") or iName:find("shard") then
            if item:IsA("ValueBase") then
                pcall(function() item.Value = gNum end)
                pcall(function()
                    item.Changed:Connect(function(v)
                        if visualSpoofState.fakeGems and visualSpoofState.fakeGems > 0 and v ~= visualSpoofState.fakeGems then
                            task.defer(function()
                                if visualSpoofState.fakeGems and visualSpoofState.fakeGems > 0 then
                                    item.Value = visualSpoofState.fakeGems
                                end
                            end)
                        end
                    end)
                end)
            end
        end
    end

    -- 2. Đặt thuộc tính Attribute trên pData và LocalPlayer
    for _, a in ipairs({"Gems", "Gem", "Diamonds", "Diamond", "Ruby"}) do
        pcall(function() pData:SetAttribute(a, gNum) end)
        pcall(function() LocalPlayer:SetAttribute(a, gNum) end)
    end

    -- 3. Tạo hoặc gán ValueBase "Gems" và "Gem" trực tiếp trong pData để mọi script game đều đọc được
    for _, gName in ipairs({"Gems", "Gem"}) do
        local obj = pData:FindFirstChild(gName)
        if not obj then
            pcall(function()
                local nv = Instance.new("IntValue")
                nv.Name = gName
                nv.Value = gNum
                nv.Parent = pData
                nv.Changed:Connect(function(v)
                    if visualSpoofState.fakeGems and visualSpoofState.fakeGems > 0 and v ~= visualSpoofState.fakeGems then
                        task.defer(function()
                            if visualSpoofState.fakeGems and visualSpoofState.fakeGems > 0 then
                                nv.Value = visualSpoofState.fakeGems
                            end
                        end)
                    end
                end)
            end)
        else
            if obj:IsA("ValueBase") then
                pcall(function() obj.Value = gNum end)
            end
        end
    end

    -- 4. Đặt leaderstats nếu có
    local ls = LocalPlayer:FindFirstChild("leaderstats")
    if ls then
        for _, ch in ipairs(ls:GetChildren()) do
            local cLow = ch.Name:lower()
            if cLow:find("gem") or cLow:find("diamond") or cLow:find("ruby") then
                if ch:IsA("ValueBase") then
                    pcall(function() ch.Value = gNum end)
                end
            end
        end
    end
end

function visualSpoofState.Apply(isReset)
    local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
    local ls = LocalPlayer:FindFirstChild("leaderstats")
    local pg = LocalPlayer:FindFirstChild("PlayerGui")

    -- 1. Ảo hoá Vé Nhiệm Vụ (Tickets)
    if visualSpoofState.fakeTicket and visualSpoofState.fakeTicket > 0 and not isReset then
        local num = visualSpoofState.fakeTicket
        if pData then
            local tObj = pData:FindFirstChild("Ticket") or pData:FindFirstChild("Tickets")
            if tObj and tObj:IsA("ValueBase") then
                pcall(function() tObj.Value = num end)
            end
        end
        if ls then
            local lsT = ls:FindFirstChild("Ticket") or ls:FindFirstChild("Tickets")
            if lsT and lsT:IsA("ValueBase") then
                pcall(function() lsT.Value = num end)
            end
        end
        if StatTiles.Tickets and StatTiles.Tickets.Set then
            StatTiles.Tickets.Set(FormatWithSpaces(num) .. " Vé")
        end
        if pg then
            for _, d in ipairs(pg:GetDescendants()) do
                if d:IsA("TextLabel") and d.Visible then
                    local sLow = d.Name:lower()
                    local pLow = d.Parent and d.Parent.Name:lower() or ""
                    if sLow:find("ticket") or pLow:find("ticket") then
                        if d.Text:match("^[%d%s,%.]+$") or d.Text:find("Vé") or d.Text:find("Ticket") then
                            d.Text = FormatWithSpaces(num)
                        end
                    end
                end
            end
        end
    elseif isReset and pData then
        local tReal = pData:FindFirstChild("Ticket") and tonumber(pData.Ticket.Value) or 0
        if StatTiles.Tickets and StatTiles.Tickets.Set then
            StatTiles.Tickets.Set(FormatWithSpaces(tReal) .. " Vé")
        end
    end

    -- 2. Ảo hoá Gems (Đá Quý) - Áp dụng cả pData, leaderstats, và PlayerGui
    if visualSpoofState.fakeGems and visualSpoofState.fakeGems > 0 and not isReset then
        local gNum = visualSpoofState.fakeGems
        visualSpoofState.ScanPlayerDataGems(gNum)
        visualSpoofState.ScanPlayerGuiGems(gNum)
        if StatTiles.GemsGained and StatTiles.GemsGained.Set then
            StatTiles.GemsGained.Set(FormatWithSpaces(gNum) .. " Gems")
        end
    elseif isReset then
        if StatTiles.GemsGained and StatTiles.GemsGained.Set then
            local realG = (GetPlayerCurrentGems and GetPlayerCurrentGems()) or 0
            StatTiles.GemsGained.Set(FormatWithSpaces(realG) .. " Gems")
        end
    end
end

function visualSpoofState.Hook()
    local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
    if not pData then return end

    local tObj = pData:FindFirstChild("Ticket") or pData:FindFirstChild("Tickets")
    if tObj and tObj:IsA("ValueBase") then
        pcall(function()
            tObj.Changed:Connect(function(newVal)
                if visualSpoofState.fakeTicket and visualSpoofState.fakeTicket > 0 then
                    if newVal ~= visualSpoofState.fakeTicket then
                        task.defer(function()
                            if visualSpoofState.fakeTicket and visualSpoofState.fakeTicket > 0 then
                                tObj.Value = visualSpoofState.fakeTicket
                            end
                        end)
                    end
                end
            end)
        end)
    end

    if visualSpoofState.fakeGems and visualSpoofState.fakeGems > 0 then
        visualSpoofState.ScanPlayerDataGems(visualSpoofState.fakeGems)
        visualSpoofState.ScanPlayerGuiGems(visualSpoofState.fakeGems)
    end
end

-- Tải ngay cài đặt ảo hoá từ máy và hook lắng nghe server
pcall(visualSpoofState.Load)
pcall(visualSpoofState.Hook)

local StatTiles = {}
local WorldData = {
    islands = {
        {name = "[1] Đảo Khởi Đầu (Spawn)", pos = Vector3.new(-200.7, 11.1, 35.9), radius = 850},
        {name = "[2] Đảo Tre (Bamboo Isle)", pos = Vector3.new(-1223.0, 7.3, -24.1), radius = 850},
        {name = "[3] Đảo Phóng Xạ (Fallout Isle)", pos = Vector3.new(65.5, 8.8, 1181.3), radius = 850},
        {name = "[4] Đảo Thống Trị (Sovereign Isle)", pos = Vector3.new(-1276.4, 8.8, 1239.7), radius = 850},
        {name = "[5] Đảo Cá Chép (Perch Isle)", pos = Vector3.new(-62.0, 11.9, -1321.4), radius = 850},
        {name = "[6] Đảo Băng Giá (Frost Isle)", pos = Vector3.new(-1366.0, 11.9, -1495.4), radius = 850},
        {name = "[7] Đảo Quả Dừa (Coconut Isle)", pos = Vector3.new(1493.6, 9.1, -1430.6), radius = 850},
        {name = "[8] Đảo Hổ Phách (Amber Isle)", pos = Vector3.new(1259.4, 9.1, 1401.5), radius = 850},
        {name = "[9] Đảo Chiến Trường (Battlefield)", pos = Vector3.new(1393.5, 11.3, 169.6), radius = 850},
        {name = "[10] Đảo Đỉnh Sương Mù (Mistpeak)", pos = Vector3.new(2660.2, 8.8, -86.7), radius = 850},
    },
    bossRealms = {
        {name = "Boss Bạch Tuộc (Phao Biển)", pos = Vector3.new(1608.2, 5.0, -218.3), radius = 450},
        {name = "Vùng Câu Cá Ngầm Lòng Đất", pos = Vector3.new(112.5, -330.0, -30.8), radius = 450},
        {name = "Đấu Trường Boss Enzo", pos = Vector3.new(-115.3, 9.2, 1349.5), radius = 450},
    }
}

local function GetCurrentLocationName()
    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if not root then return "Đang tải vị trí...", nil end
    local myPos = root.Position

    if myPos.Y < -150 then
        return "Vùng Câu Cá Ngầm Lòng Đất", "underground"
    end

    local bestName = "Đang ở giữa biển"
    local bestObj = nil
    local minDist = 999999

    for _, isl in ipairs(WorldData.islands) do
        local dist = (Vector3.new(myPos.X, 0, myPos.Z) - Vector3.new(isl.pos.X, 0, isl.pos.Z)).Magnitude
        if dist < (isl.radius or 850) and dist < minDist then
            minDist = dist
            bestName = isl.name
            bestObj = isl
        end
    end

    for _, br in ipairs(WorldData.bossRealms) do
        local dist = (myPos - br.pos).Magnitude
        if dist < (br.radius or 450) and dist < minDist then
            minDist = dist
            bestName = br.name
            bestObj = br
        end
    end

    return bestName, bestObj, minDist
end

local function GetCurrentBackpackFishCount()
    local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
    local count = 0
    local limit = 100

    if pData then
        local inv = pData:FindFirstChild("Inventory")
        if inv then
            count = #inv:GetChildren()
        end
        local limVal = pData:FindFirstChild("InventoryLimit")
        if limVal and tonumber(limVal.Value) then
            limit = tonumber(limVal.Value)
        end
    end

    local char = LocalPlayer.Character
    if char then
        for _, item in ipairs(char:GetChildren()) do
            if item:IsA("Tool") and tostring(item.Name):find("|") then
                count = count + 1
            end
        end
    end

    local bp = LocalPlayer:FindFirstChild("Backpack")
    if bp then
        for _, item in ipairs(bp:GetChildren()) do
            if item:IsA("Tool") and tostring(item.Name):find("|") then
                count = count + 1
            end
        end
    end

    return count, limit
end

local sessionStartTime = tick()
local initialCash = nil
local initialFishCaught = nil
local lastWebhookStatsTime = tick()

local gemTracker = {
    gained = 0,
    lastKnown = nil,
    lastFishAwardTime = 0,
    lastFishAwardAmount = 0,
    inventoryHooked = false
}

local fishGemRewardLookup = {
    ["scarlet fish"] = 3,
    ["elder scarlet fish"] = 5,
    ["crimson electric eel"] = 5,
    ["verdant alligator gar"] = 3,
    ["verdant grouper"] = 3,
    ["verdant bonefang"] = 5,
    ["flying fish empress"] = 10,
    ["flying fish emperor"] = 10,
    ["reborn puffer beast"] = 10,
    ["frost kingfish"] = 10,
    ["tigerfang whale"] = 5,
    ["heaven piercer turtle"] = 5,
    ["draconic koi"] = 5,
    ["sanguine fish"] = 20,
    ["primordial kunfish overlord"] = 30,
    ["warbringer shark"] = 25,
    ["mountain fish"] = 20,
    ["octoparasitic fish"] = 50,
}

local function GetFishGemReward(fishName)
    if not fishName then return 0 end
    local clean = tostring(fishName):lower():gsub("^%s+", ""):gsub("%s+$", "")
    if fishGemRewardLookup[clean] then return fishGemRewardLookup[clean] end
    for k, v in pairs(fishGemRewardLookup) do
        if clean:find(k, 1, true) then return v end
    end
    return 0
end

local function GetPlayerCurrentGems()
    if visualSpoofState and visualSpoofState.fakeGems and visualSpoofState.fakeGems > 0 then
        return visualSpoofState.fakeGems
    end
    local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
    if pData then
        for _, name in ipairs({"Gems", "Gem", "Diamonds", "Diamond", "Ruby", "Rubies", "Shards", "Crystal"}) do
            local obj = pData:FindFirstChild(name)
            if obj and obj:IsA("ValueBase") and tonumber(obj.Value) then
                return tonumber(obj.Value)
            end
        end
        for _, sub in ipairs({"Currencies", "Currency", "Stats", "Account"}) do
            local subFolder = pData:FindFirstChild(sub)
            if subFolder then
                for _, name in ipairs({"Gems", "Gem", "Diamonds", "Diamond"}) do
                    local obj = subFolder:FindFirstChild(name)
                    if obj and obj:IsA("ValueBase") and tonumber(obj.Value) then
                        return tonumber(obj.Value)
                    end
                end
            end
        end
        for _, name in ipairs({"Gems", "Gem", "Diamonds", "Diamond", "Ruby"}) do
            local att = pData:GetAttribute(name)
            if att and tonumber(att) then return tonumber(att) end
        end
    end

    local ls = LocalPlayer:FindFirstChild("leaderstats")
    if ls then
        for _, name in ipairs({"Gems", "Gem", "Diamonds", "Diamond", "Ruby"}) do
            local obj = ls:FindFirstChild(name)
            if obj and obj:IsA("ValueBase") and tonumber(obj.Value) then
                return tonumber(obj.Value)
            end
        end
    end

    for _, name in ipairs({"Gems", "Gem", "Diamonds", "Diamond"}) do
        local att = LocalPlayer:GetAttribute(name)
        if att and tonumber(att) then return tonumber(att) end
        local obj = LocalPlayer:FindFirstChild(name)
        if obj and obj:IsA("ValueBase") and tonumber(obj.Value) then return tonumber(obj.Value) end
    end

    local pg = LocalPlayer:FindFirstChild("PlayerGui")
    if pg then
        local mg = pg:FindFirstChild("MainGui") or pg:FindFirstChild("Fisher_GUI")
        if mg then
            local topBar = mg:FindFirstChild("TopBar") or mg:FindFirstChild("Top") or mg:FindFirstChild("Stats") or mg:FindFirstChild("Currencies")
            if topBar then
                for _, d in ipairs(topBar:GetChildren()) do
                    if d:IsA("TextLabel") and (d.Name:lower():find("gem") or d.Name:lower():find("diamond")) then
                        local numStr = d.Text:gsub("[^%d]", "")
                        if #numStr > 0 and tonumber(numStr) then return tonumber(numStr) end
                    end
                end
            end
        end
    end

    return nil
end

local function TrackCaughtFishForGems(child)
    if not child then return end
    local fishName = child.Name
    local reward = GetFishGemReward(fishName)
    if child:FindFirstChild("Gem") and tonumber(child.Gem.Value) then
        reward = math.max(reward, tonumber(child.Gem.Value))
    elseif child:FindFirstChild("Gems") and tonumber(child.Gems.Value) then
        reward = math.max(reward, tonumber(child.Gems.Value))
    elseif child:GetAttribute("Gem") and tonumber(child:GetAttribute("Gem")) then
        reward = math.max(reward, tonumber(child:GetAttribute("Gem")))
    elseif child:GetAttribute("Gems") and tonumber(child:GetAttribute("Gems")) then
        reward = math.max(reward, tonumber(child:GetAttribute("Gems")))
    end

    if reward > 0 then
        gemTracker.lastFishAwardTime = tick()
        gemTracker.lastFishAwardAmount = reward
        gemTracker.gained = gemTracker.gained + reward
        if infoGemsGained and infoGemsGained.Set then
            infoGemsGained.Set("+" .. FormatWithSpaces(gemTracker.gained) .. " Gems")
        end
        ShowNotification("Thưởng Gems", string.format("Bắt được %s! Nhận được +%d Gems!", fishName, reward), "SUCCESS", 4)
    end
end

local function SendDiscordWebhook(title, description, color, fields, thumbnailUrl)
    if not Config.WebhookEnabled or not Config.WebhookUrl or #Config.WebhookUrl == 0 then return end
    pcall(function()
        local avatarUrl = "https://www.roblox.com/headshot-thumbnail/image?userId=" .. tostring(LocalPlayer.UserId) .. "&width=150&height=150&format=png"
        local payload
        if type(title) == "table" then
            payload = title
            if not payload.username then
                payload.username = (LocalPlayer.DisplayName or LocalPlayer.Name) .. " • Farm Bot"
            end
            if not payload.avatar_url then
                payload.avatar_url = avatarUrl
            end
        else
            local embed = {
                title = title or "Heavyweight Fishing Bot",
                description = description or "",
                color = color or 11029759,
                fields = fields or {},
                thumbnail = { url = thumbnailUrl or avatarUrl },
                footer = { text = "✨ Câu Cá Pro " .. tostring(SCRIPT_BUILD_COMMIT) .. " • Heavyweight Fishing", icon_url = avatarUrl },
                timestamp = DateTime.now():ToIsoDate()
            }
            payload = {
                username = (LocalPlayer.DisplayName or LocalPlayer.Name) .. " • Farm Bot",
                avatar_url = avatarUrl,
                embeds = {embed}
            }
        end
        local body = HttpService:JSONEncode(payload)
        local headers = {["Content-Type"] = "application/json"}

        local reqFunc = (syn and syn.request) or (http and http.request) or http_request or request
        if reqFunc then
            reqFunc({
                Url = Config.WebhookUrl,
                Method = "POST",
                Headers = headers,
                Body = body
            })
        end
    end)
end

-- Gửi báo cáo toàn diện tức thì (giống nút Test ntfy nhưng cho Discord)
local function SendServerReportWebhook(label)
    if not Config.WebhookEnabled or not Config.WebhookUrl or #Config.WebhookUrl == 0 then return end
    pcall(function()
        local playerName = (LocalPlayer and LocalPlayer.DisplayName) or (LocalPlayer and LocalPlayer.Name) or "Người Chơi"
        local timeStr = os.date("%H:%M:%S - %d/%m/%Y")
        local jobId = tostring(game.JobId or "N/A")
        local placeId = tostring(game.PlaceId or "18779600655")

        local curWeather = "Clear (☀️ Trời Quang)"
        local islandStr = "Không có bão"
        local bossStr = "Không"
        if secretBossState and secretBossState.DetectWeather then
            pcall(function()
                local wIsland, wName = secretBossState.DetectWeather()
                if wName and wName ~= "" and wName ~= "Clear" then curWeather = wName end
                if wIsland then
                    if wIsland.islandName then islandStr = wIsland.islandName end
                    if wIsland.bosses and #wIsland.bosses > 0 then
                        local bNames = {}
                        for _, b in ipairs(wIsland.bosses) do table.insert(bNames, b.name) end
                        bossStr = table.concat(bNames, ", ")
                    end
                end
            end)
        end

        local pData = ReplicatedStorage:FindFirstChild("Data") and LocalPlayer and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
        local fishCount = pData and pData:FindFirstChild("FishCaught") and tonumber(pData.FishCaught.Value) or 0
        local cashVal = pData and pData:FindFirstChild("Cash") and tonumber(pData.Cash.Value) or 0
        local ticketVal = pData and pData:FindFirstChild("Ticket") and tonumber(pData.Ticket.Value) or 0
        local questDone = pData and pData:FindFirstChild("TicketQuestDailyCount") and tonumber(pData.TicketQuestDailyCount.Value) or 0
        local essenceVal = pData and pData:FindFirstChild("EssenceOrb") and tonumber(pData.EssenceOrb.Value) or 0
        local rerollVal = pData and pData:FindFirstChild("Trait Reroll") and tonumber(pData["Trait Reroll"].Value) or 0
        local gemVal = secretBossState.GetPlayerGems()
        local questStatus = (ticketQuestState and ticketQuestState.statusText) or "Đang hoạt động"
        local invCount, invLimit = GetCurrentBackpackFishCount and GetCurrentBackpackFishCount() or 0, 50

        local elapsed = tick() - sessionStartTime
        local h, m, s = math.floor(elapsed/3600), math.floor((elapsed%3600)/60), math.floor(elapsed%60)

        local titleLabel = label or "📊 BÁO CÁO TOÀN DIỆN"
        local titleColor = 3447003
        if curWeather ~= "Clear (☀️ Trời Quang)" then titleColor = 16753920 end

        SendDiscordWebhook(
            titleLabel,
            string.format("👤 **%s** — Server: `%s`", playerName, jobId),
            titleColor,
            {
                { name = "❤️ Uptime", value = string.format("%02d:%02d:%02d", h, m, s), inline = true },
                { name = "🌦️ Thời Tiết", value = curWeather, inline = true },
                { name = "🏝️ Đảo", value = islandStr, inline = true },
                { name = "🌲 Boss Có Thể Ra", value = bossStr, inline = true },
                { name = "🐟 Tổng Cá", value = FormatWithSpaces(fishCount) .. " con", inline = true },
                { name = "🎒 Ba lô cá", value = string.format("%d / %d", invCount, invLimit), inline = true },
                { name = "💰 Tiền Hiện Tại", value = "$" .. FormatWithSpaces(cashVal), inline = true },
                { name = "💎 Gems Hiện Có", value = FormatWithSpaces(gemVal) .. " Gems", inline = true },
                { name = "🎫 Vé Nhiệm Vụ", value = FormatWithSpaces(ticketVal) .. " Vé", inline = true },
                { name = "📜 NV Xong Hôm Nay", value = string.format("%d/20 NV", questDone), inline = true },
                { name = "🔮 Essence Orb", value = FormatWithSpaces(essenceVal) .. " Viên", inline = true },
                { name = "🎲 Trait Reroll", value = FormatWithSpaces(rerollVal) .. " Vé", inline = true },
                { name = "📌 Trạng Thái Bot", value = questStatus, inline = false },
                { name = "⚡ Code Vào Server", value = string.format("```lua\ngame:GetService(\"TeleportService\"):TeleportToPlaceInstance(%s, \"%s\", game.Players.LocalPlayer)\n```", placeId, jobId), inline = false },
                { name = "⏰ Cập nhật lúc", value = timeStr, inline = false }
            }
        )
    end)
end

-- Gửi báo cáo khi hoàn thành Nhiệm Vụ Vé
local function SendTicketQuestWebhook(qCount, ticketCount, gemsGained, gemsTotal, cdMins)
    if not Config.WebhookEnabled or not Config.WebhookUrl or #Config.WebhookUrl == 0 then return end
    pcall(function()
        local playerName = (LocalPlayer and LocalPlayer.DisplayName) or (LocalPlayer and LocalPlayer.Name) or "Người Chơi"
        local timeStr = os.date("%H:%M:%S - %d/%m/%Y")
        local jobId = tostring(game.JobId or "N/A")
        local placeId = tostring(game.PlaceId or "18779600655")
        SendDiscordWebhook(
            string.format("🎫 XONG NHIỆM VỤ VÉ [%d/20]", qCount or 0),
            string.format("✅ **%s** đã nộp vé Hard thành công! 🎉", playerName),
            5763719,
            {
                { name = "📜 Lượt NV Hôm Nay", value = string.format("%d / 20", qCount or 0), inline = true },
                { name = "🎫 Vé Hiện Có", value = FormatWithSpaces(ticketCount or 0) .. " Vé", inline = true },
                { name = "💎 Gems Nhận Được", value = "+" .. FormatWithSpaces(gemsGained or 0) .. " Gems", inline = true },
                { name = "💰 Tổng Gems", value = FormatWithSpaces(gemsTotal or 0) .. " Gems", inline = true },
                { name = "⏳ Hồi Chiêu", value = string.format("%d phút", cdMins or 20), inline = true },
                { name = "📍 Server", value = string.format("`%s`", jobId), inline = true },
                { name = "⚡ Code Vào Server", value = string.format("```lua\ngame:GetService(\"TeleportService\"):TeleportToPlaceInstance(%s, \"%s\", game.Players.LocalPlayer)\n```", placeId, jobId), inline = false },
                { name = "⏰ Thời gian", value = timeStr, inline = false }
            }
        )
    end)
end

local function SendTelegramMessage(text)
    if not Config.TelegramEnabled or not Config.TelegramBotToken or not Config.TelegramChatId or #Config.TelegramBotToken == 0 or #Config.TelegramChatId == 0 then return end
    pcall(function()
        local url = "https://api.telegram.org/bot" .. Config.TelegramBotToken .. "/sendMessage"
        local payload = {
            chat_id = Config.TelegramChatId,
            text = text,
            parse_mode = "Markdown"
        }
        local body = HttpService:JSONEncode(payload)
        local headers = {["Content-Type"] = "application/json"}
        local reqFunc = (syn and syn.request) or (http and http.request) or http_request or request
        if reqFunc then
            reqFunc({
                Url = url,
                Method = "POST",
                Headers = headers,
                Body = body
            })
        end
    end)
end

function SendNtfyNotification(title, message, priorityLevel, tagList)
    if not Config.NtfyEnabled or not Config.NtfyTopic or #tostring(Config.NtfyTopic):gsub("%s+", "") == 0 then return end
    pcall(function()
        local rawTopic = tostring(Config.NtfyTopic):gsub("%s+", "")
        if #rawTopic == 0 then return end

        local targetHost = "https://ntfy.sh"
        local cleanTopic = rawTopic

        if rawTopic:find("^https?://") then
            local host, subTopic = rawTopic:match("^(https?://[^/]+)/?(.*)$")
            if host and #host > 0 then
                targetHost = host
                cleanTopic = subTopic or ""
            end
        end

        cleanTopic = cleanTopic:gsub("^/+", ""):gsub("/+$", "")
        if #cleanTopic == 0 and not rawTopic:find("^https?://") then
            cleanTopic = rawTopic
        end

        local payload = {
            topic = cleanTopic,
            title = tostring(title or "Heavyweight Fishing"),
            message = tostring(message or ""),
            priority = priorityLevel or 3,
            tags = tagList or {"fishing_pole_and_fish"}
        }

        local reqFunc = (syn and syn.request) or (http and http.request) or http_request or request
        if not reqFunc then return end

        local body = HttpService:JSONEncode(payload)
        local headers = {
            ["Content-Type"] = "application/json",
            ["content-type"] = "application/json"
        }

        -- ntfy JSON publishing BẮT BUỘC gửi tới root URL (https://ntfy.sh).
        -- Tuyệt đối không nối thêm /cleanTopic vào URL kẻo ntfy hiểu nhầm toàn bộ JSON là văn bản thô!
        local postUrl = targetHost

        reqFunc({
            Url = postUrl,
            Method = "POST",
            Headers = headers,
            Body = body
        })
    end)
end

function secretBossState.SendWeatherNtfyAlert(weatherName, matchedIsland, isInitial)
    if not Config.NtfyEnabled or not Config.NtfyAlertWeatherChange then return end
    pcall(function()
        local jobId = tostring(game.JobId or "")
        local placeId = tostring(game.PlaceId or "18779600655")
        local playerName = (LocalPlayer and LocalPlayer.DisplayName) or (LocalPlayer and LocalPlayer.Name) or "Người Chơi"
        local timeStr = os.date("%H:%M:%S - %d/%m/%Y")

        local isClear = (not weatherName or weatherName == "" or weatherName == "Clear")

        if isClear then
            local title = "☀️ THỜI TIẾT: TRỜI QUANG (CLEAR)"
            local msg = string.format("Thời tiết tại server đã trở lại bình thường (Clear).\n👤 Người chơi: %s\n⏰ %s", playerName, timeStr)
            SendNtfyNotification(title, msg, 2, {"sun_behind_cloud", "information_source"})
            return
        end

        local islandName = matchedIsland and matchedIsland.islandName or "Chưa rõ đảo"
        local bossListStr = ""
        if matchedIsland and matchedIsland.bosses and #matchedIsland.bosses > 0 then
            local bNames = {}
            for _, b in ipairs(matchedIsland.bosses) do
                table.insert(bNames, b.name)
            end
            bossListStr = table.concat(bNames, ", ")
        end

        local tagList = {"cloud", "fishing_pole_and_fish"}
        local prio = 4
        local wLower = tostring(weatherName):lower()
        if wLower:find("thunder") or wLower:find("sấm") or wLower:find("bão") then
            prio = 5
            tagList = {"zap", "cloud_lightning", "fishing_pole_and_fish"}
        elseif wLower:find("rain") or wLower:find("mưa") then
            tagList = {"cloud_rain", "droplet"}
        elseif wLower:find("snow") or wLower:find("tuyết") or wLower:find("frost") then
            tagList = {"snowflake", "cold_face"}
        elseif wLower:find("fog") or wLower:find("sương") then
            tagList = {"fog", "eyes"}
        elseif wLower:find("sun") or wLower:find("nắng") or wLower:find("blazing") then
            tagList = {"sunny", "fire"}
        elseif wLower:find("wind") or wLower:find("gió") then
            tagList = {"dash", "wind_face"}
        end

        local headerPrefix = isInitial and "🌟 SERVER CÓ SẴN THỜI TIẾT: " or "⛈️ THỜI TIẾT SERVER: "
        local title = headerPrefix .. tostring(weatherName):upper()

        local msgParts = {
            string.format("🎯 Thời tiết: %s", tostring(weatherName)),
            string.format("📍 Đảo liên quan: %s", islandName)
        }
        if bossListStr ~= "" then
            table.insert(msgParts, string.format("🐲 Boss có thể xuất hiện: %s", bossListStr))
        end
        table.insert(msgParts, string.format("👤 Nhân vật: %s", playerName))
        if jobId ~= "" then
            table.insert(msgParts, string.format("🔑 Job ID: %s", jobId))
            table.insert(msgParts, string.format("⚡ Teleport:\ngame:GetService(\"TeleportService\"):TeleportToPlaceInstance(%s, \"%s\", game.Players.LocalPlayer)", placeId, jobId))
        end
        table.insert(msgParts, string.format("⏰ Thời gian: %s", timeStr))

        local fullMsg = table.concat(msgParts, "\n")
        SendNtfyNotification(title, fullMsg, prio, tagList)
    end)
end

local npcAlertsSent = {}

function secretBossState.SendNPCDetectionAlert(npcType, npcName, npcInst)
    local alertKey = tostring(game.JobId or "local") .. "_" .. tostring(npcType)
    if npcAlertsSent[alertKey] then return end
    npcAlertsSent[alertKey] = true

    local posStr = "Không xác định"
    if npcInst then
        local root = npcInst:FindFirstChild("HumanoidRootPart") or npcInst.PrimaryPart or npcInst:FindFirstChild("Head") or npcInst:FindFirstChildWhichIsA("BasePart")
        if root then
            local p = root.Position
            posStr = string.format("X: %.1f, Y: %.1f, Z: %.1f", p.X, p.Y, p.Z)
        end
    end

    local jobId = tostring(game.JobId or "")
    local placeId = tostring(game.PlaceId or "0")
    local playerName = (LocalPlayer and LocalPlayer.Name) or "Unknown"
    local timeStr = os.date("%H:%M:%S - %d/%m/%Y")

    -- 1. Gửi qua Discord Webhook nếu người dùng bật
    if Config.WebhookEnabled and Config.WebhookNotifyNPC and Config.WebhookUrl and #Config.WebhookUrl > 0 then
        local fields = {
            { name = "🎯 NPC Phát Hiện", value = "**" .. tostring(npcName) .. "**", inline = true },
            { name = "👤 Người Tìm Thấy", value = playerName, inline = true },
            { name = "📍 Tọa Độ Đứng", value = posStr, inline = true },
            { name = "🔑 Job ID Server", value = "```" .. (jobId ~= "" and jobId or "N/A (Chơi 1 mình)") .. "```", inline = false },
            { name = "⚡ Lệnh Vào Server Nhanh", value = "```lua\ngame:GetService(\"TeleportService\"):TeleportToPlaceInstance(" .. placeId .. ", \"" .. jobId .. "\", game.Players.LocalPlayer)\n```", inline = false },
            { name = "⏰ Thời Gian", value = timeStr, inline = true }
        }
        SendDiscordWebhook("📜 PHÁT HIỆN " .. tostring(npcName):upper() .. " TRONG SERVER!", "Bot đã tìm thấy **" .. tostring(npcName) .. "** tại server hiện tại!", 16753920, fields)
    end

    -- 2. Gửi qua Telegram Bot nếu người dùng bật
    if Config.TelegramEnabled and Config.TelegramNotifyNPC and Config.TelegramBotToken and #Config.TelegramBotToken > 0 and Config.TelegramChatId and #Config.TelegramChatId > 0 then
        local teleText = "📜 *PHÁT HIỆN " .. tostring(npcName):upper() .. "!*\n\n"
            .. "🎯 *NPC:* " .. tostring(npcName) .. "\n"
            .. "👤 *Người tìm thấy:* " .. playerName .. "\n"
            .. "📍 *Tọa độ:* " .. posStr .. "\n"
            .. "🔑 *Job ID:* `" .. (jobId ~= "" and jobId or "N/A") .. "`\n"
            .. "⏰ *Thời gian:* " .. timeStr .. "\n\n"
            .. "⚡ *Code vào server:*\n`game:GetService(\"TeleportService\"):TeleportToPlaceInstance(" .. placeId .. ", \"" .. jobId .. "\", game.Players.LocalPlayer)`"
        SendTelegramMessage(teleText)
    end

    -- 3. Gửi qua ntfy Push nếu người dùng bật
    if Config.NtfyEnabled and Config.NtfyNotifyBoss and Config.NtfyTopic and #tostring(Config.NtfyTopic):gsub("%s+", "") > 0 then
        local ntfyTitle = "📜 PHÁT HIỆN " .. tostring(npcName):upper() .. "!"
        local ntfyMsg = string.format("Phát hiện %s tại server!\n👤 Người tìm thấy: %s\n📍 Tọa độ: %s\n🔑 Job ID: %s\n⚡ Lệnh vào nhanh:\ngame:GetService(\"TeleportService\"):TeleportToPlaceInstance(%s, \"%s\", game.Players.LocalPlayer)\n⏰ %s",
            tostring(npcName), playerName, posStr, (jobId ~= "" and jobId or "N/A"), placeId, jobId, timeStr)
        SendNtfyNotification(ntfyTitle, ntfyMsg, 5, {"scroll", "eyes", "star"})
    end
end

-- Lấy số Gems của người chơi từ DataStore an toàn
function secretBossState.GetPlayerGems()
    local val = 0
    pcall(function()
        local pData = ReplicatedStorage:FindFirstChild("Data") and LocalPlayer and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
        if pData then
            for _, gName in ipairs({"Gems", "Gem", "Diamonds", "Diamond"}) do
                local gObj = pData:FindFirstChild(gName)
                if gObj and gObj:IsA("ValueBase") and tonumber(gObj.Value) then
                    val = tonumber(gObj.Value)
                    return
                end
            end
        end
        local ls = LocalPlayer and LocalPlayer:FindFirstChild("leaderstats")
        if ls then
            for _, gName in ipairs({"Gems", "Gem", "Diamonds", "Diamond"}) do
                local gObj = ls:FindFirstChild(gName)
                if gObj and gObj:IsA("ValueBase") and tonumber(gObj.Value) then
                    val = tonumber(gObj.Value)
                    return
                end
            end
        end
    end)
    return val
end

-- Gửi báo cáo trạng thái server về điện thoại (dùng cho nút Báo Cáo Server)
function secretBossState.SendServerStatusNtfyAlert()
    if not Config.NtfyEnabled then return end
    pcall(function()
        local playerName = (LocalPlayer and LocalPlayer.DisplayName) or (LocalPlayer and LocalPlayer.Name) or "Người Chơi"
        local timeStr = os.date("%H:%M:%S - %d/%m/%Y")
        local jobId = tostring(game.JobId or "N/A")
        local placeId = tostring(game.PlaceId or "18779600655")

        local curWeather = "Clear (Trời Quang)"
        local islandStr = "Không có bão"
        if secretBossState and secretBossState.DetectWeather then
            pcall(function()
                local wIsland, wName = secretBossState.DetectWeather()
                if wName and wName ~= "" and wName ~= "Clear" then
                    curWeather = wName
                end
                if wIsland and wIsland.islandName then
                    islandStr = wIsland.islandName
                end
            end)
        end

        local qCount = 0
        local ticketCount = 0
        pcall(function()
            local pData = ticketQuestState and ticketQuestState.GetPlayerDataFolder and ticketQuestState.GetPlayerDataFolder()
            if pData then
                if pData:FindFirstChild("TicketQuestDailyCount") then
                    qCount = tonumber(pData.TicketQuestDailyCount.Value) or 0
                end
                if pData:FindFirstChild("Ticket") then
                    ticketCount = tonumber(pData.Ticket.Value) or 0
                end
            end
        end)

        local curGems = secretBossState.GetPlayerGems()
        local questStatus = (ticketQuestState and ticketQuestState.statusText) or "Đang hoạt động"

        local title = "📊 BÁO CÁO TÌNH HÌNH SERVER"
        local msgParts = {
            string.format("👤 Tài khoản: %s", playerName),
            string.format("🌦️ Thời tiết: %s [%s]", curWeather, islandStr),
            string.format("📜 Tiến độ vé: %d/20 NV (Có %s Vé)", qCount, FormatWithSpaces(ticketCount)),
            string.format("📌 Trạng thái: %s", questStatus),
            string.format("💎 Số Gems: %s Gems", FormatWithSpaces(curGems)),
            string.format("🔑 Job ID: %s", jobId),
            string.format("⚡ Teleport:\ngame:GetService(\"TeleportService\"):TeleportToPlaceInstance(%s, \"%s\", game.Players.LocalPlayer)", placeId, jobId),
            string.format("⏰ Cập nhật lúc: %s", timeStr)
        }
        local fullMsg = table.concat(msgParts, "\n")
        SendNtfyNotification(title, fullMsg, 4, {"bar_chart", "clipboard", "partly_sunny"})
    end)
end

local function GetPlayerRodPower()
    local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
    local eqRod = pData and pData:FindFirstChild("FishingRod") and pData.FishingRod.Value
    if eqRod then
        for _, r in ipairs(allRods) do
            if r.name == eqRod then
                return r.power
            end
        end
    end
    return 0
end

local function GetCurrentHookedFishName()
    local char = LocalPlayer.Character
    if not char then return nil end

    local function matchBossName(rawText)
        if not rawText or typeof(rawText) ~= "string" or #rawText == 0 then return nil end
        local txt = rawText:gsub("^%s+", ""):gsub("%s+$", "")
        local txtLower = txt:lower()

        if secretBossLookup[txtLower] then return secretBossLookup[txtLower] end

        -- Fuzzy / Substring match với danh sách Secret Boss (bỏ qua prefix đột biến như Shiny, Giant, Albino,...)
        for _, entry in ipairs(secretBossDatabase) do
            for _, b in ipairs(entry.bosses) do
                local bLower = b.name:lower()
                if txtLower:find(bLower, 1, true) then
                    return b.name
                end
            end
            if entry.bossPatterns then
                for _, bp in ipairs(entry.bossPatterns) do
                    if txtLower:find(bp, 1, true) then
                        return bp
                    end
                end
            end
        end
        return txt
    end

    -- 1. Check Character attributes
    for _, attName in ipairs({"FishName", "TargetFish", "Boss", "Fish", "CurrentFish", "HookedFish"}) do
        local val = char:GetAttribute(attName)
        if val and tostring(val) ~= "" then
            return matchBossName(tostring(val))
        end
    end

    -- 2. Check PlayerGui.MainGui.Fishing
    local pg = LocalPlayer:FindFirstChild("PlayerGui")
    local fUI = pg and pg:FindFirstChild("MainGui") and pg.MainGui:FindFirstChild("Fishing")
    if fUI and fUI.Visible then
        for _, lblName in ipairs({"FishName", "Title", "Name", "Fish", "BossName", "Target"}) do
            local label = fUI:FindFirstChild(lblName, true)
            if label and label:IsA("TextLabel") and label.Text ~= "" then
                return matchBossName(label.Text)
            end
        end

        local bossBar = fUI:FindFirstChild("BossFightBar")
        if bossBar and bossBar.Visible then
            for _, d in ipairs(bossBar:GetDescendants()) do
                if d:IsA("TextLabel") and d.Visible and d.Text ~= "" and not tonumber(d.Text) and not d.Text:find("%%") then
                    local matched = matchBossName(d.Text)
                    if matched and #matched > 2 then return matched end
                end
            end
        end

        -- Scan any visible TextLabel in fUI matching a known secret boss
        for _, d in ipairs(fUI:GetDescendants()) do
            if d:IsA("TextLabel") and d.Visible and d.Text ~= "" and not tonumber(d.Text) and not d.Text:find("%%") then
                local txtLower = d.Text:lower()
                for _, entry in ipairs(secretBossDatabase) do
                    for _, b in ipairs(entry.bosses) do
                        if txtLower:find(b.name:lower(), 1, true) then
                            return b.name
                        end
                    end
                end
            end
        end
    end

    return nil
end

local function QueueScriptOnTeleport()
    local qot = queue_on_teleport or (syn and syn.queue_on_teleport) or (fluxus and fluxus.queue_on_teleport)
    if qot then
        pcall(function()
            qot([[loadstring(game:HttpGet("https://raw.githubusercontent.com/VNGteam/Heavyweight-Fishing/main/loader.lua"))()]])
        end)
    end
end

local function ServerHop(avoidServers)
    ShowNotification("Đổi Server", "Đang tìm kiếm server phù hợp...", "WARN")
    secretBossState.isNormalHopping = true
    QueueScriptOnTeleport()
    task.spawn(function()
        local placeId = game.PlaceId
        local avoidMap = {}
        if avoidServers and type(avoidServers) == "table" then
            for _, sid in ipairs(avoidServers) do avoidMap[sid] = true end
        end
        avoidMap[game.JobId] = true

        local cursor = ""
        local candidates = {}
        for page = 1, 4 do
            local url = string.format("https://games.roblox.com/v1/games/%d/servers/Public?sortOrder=Desc&limit=100%s", placeId, cursor ~= "" and ("&cursor=" .. cursor) or "")
            local ok, res = pcall(function() return game:HttpGet(url) end)
            if ok and res then
                local bOk, body = pcall(function() return HttpService:JSONDecode(res) end)
                if bOk and body and body.data then
                    for _, s in ipairs(body.data) do
                        if s.playing and s.maxPlayers and s.id and not avoidMap[s.id] then
                            local free = s.maxPlayers - s.playing
                            if free >= 2 then
                                table.insert(candidates, s.id)
                            end
                        end
                    end
                    if #candidates >= 5 then break end
                    cursor = body.nextPageCursor or ""
                    if not cursor or cursor == "" then break end
                end
            end
            task.wait(0.15)
        end

        local chosen = (#candidates > 0 and candidates[math.random(1, #candidates)]) or nil
        if chosen then
            ShowNotification("Đổi Server", "Đã tìm thấy server mới! Đang chuyển...", "SUCCESS", 3)
            task.wait(0.5)
            local ok, err = pcall(function() TeleportService:TeleportToPlaceInstance(placeId, chosen, LocalPlayer) end)
            if not ok then
                warn("[ServerHop] TeleportToPlaceInstance lỗi, chuyển sang Teleport ngẫu nhiên:", tostring(err))
                pcall(function() TeleportService:Teleport(placeId, LocalPlayer) end)
            end
            return
        end
        ShowNotification("Đổi Server", "Đang chuyển sang server ngẫu nhiên...", "INFO", 3)
        task.wait(0.5)
        pcall(function() TeleportService:Teleport(placeId, LocalPlayer) end)
    end)
end

function secretBossState.SaveNPCHopState(visitedList)
    if not (writefile and isfile) then return end
    local isAnyActive = Config.AutoServerHopTaoist or Config.AutoServerHopMaoshan or Config.AutoServerHopGod
    if isAnyActive then
        pcall(function()
            writefile("HeavyweightFishing_NPCHop.json", HttpService:JSONEncode({
                Active = true,
                Taoist = Config.AutoServerHopTaoist == true,
                Maoshan = Config.AutoServerHopMaoshan == true,
                God = Config.AutoServerHopGod == true,
                Visited = visitedList or secretBossState.npcHopVisited or {},
                StartTime = tick()
            }))
        end)
    else
        pcall(function()
            if isfile("HeavyweightFishing_NPCHop.json") and delfile then
                delfile("HeavyweightFishing_NPCHop.json")
            end
        end)
    end
end

function secretBossState.ClearNPCHopState()
    Config.AutoServerHopTaoist = false
    Config.AutoServerHopMaoshan = false
    Config.AutoServerHopGod = false
    if UIControllers["AutoServerHopTaoist"] and UIControllers["AutoServerHopTaoist"].Set then
        pcall(function() UIControllers["AutoServerHopTaoist"].Set(false, true) end)
    end
    if UIControllers["AutoServerHopMaoshan"] and UIControllers["AutoServerHopMaoshan"].Set then
        pcall(function() UIControllers["AutoServerHopMaoshan"].Set(false, true) end)
    end
    if UIControllers["AutoServerHopGod"] and UIControllers["AutoServerHopGod"].Set then
        pcall(function() UIControllers["AutoServerHopGod"].Set(false, true) end)
    end
    pcall(function()
        if isfile and isfile("HeavyweightFishing_NPCHop.json") and delfile then
            delfile("HeavyweightFishing_NPCHop.json")
        end
    end)
end
secretBossState.ServerHop = ServerHop

secretBossState.isHopping = false
secretBossState.currentTargetWeather = nil
secretBossState.currentVisited = {}
secretBossState.lastAttemptedJob = nil
secretBossState.hopWatchdog = 0
secretBossState.isNormalHopping = false

secretBossState.HandleTeleportError = function(reason)
    pcall(function()
        local gs = game:GetService("GuiService")
        if gs and gs.ClearError then gs:ClearError() end
    end)

    if secretBossState.isHopping then
        local failJob = secretBossState.lastAttemptedJob
        if failJob and secretBossState.currentVisited then
            if not table.find(secretBossState.currentVisited, failJob) then
                table.insert(secretBossState.currentVisited, failJob)
            end
        end
        ShowNotification("Đổi Server", "Server đầy hoặc lỗi (" .. tostring(reason or "GameFull"):sub(1, 35) .. "). Tự động tìm server khác...", "WARN", 4)
        secretBossState.hopWatchdog = (secretBossState.hopWatchdog or 0) + 1
        task.delay(1.5, function()
            if secretBossState.isHopping then
                secretBossState.HopToNextWeatherServer(secretBossState.currentTargetWeather, secretBossState.currentVisited)
            end
        end)
    elseif secretBossState.isNormalHopping then
        ShowNotification("Đổi Server", "Server đầy hoặc lỗi kết nối. Đang tự động thử server khác...", "WARN", 4)
        task.delay(1.5, function()
            if secretBossState.ServerHop then secretBossState.ServerHop(secretBossState.npcHopVisited) end
        end)
    end
end

if not secretBossState.hooksInitialized then
    secretBossState.hooksInitialized = true

    table.insert(activeConnections, TeleportService.TeleportInitFailed:Connect(function(player, teleportResult, errorMessage)
        if player == LocalPlayer then
            secretBossState.HandleTeleportError(errorMessage or teleportResult or "TeleportInitFailed")
        end
    end))

    pcall(function()
        local gs = game:GetService("GuiService")
        table.insert(activeConnections, gs.ErrorMessageChanged:Connect(function(msg)
            if (secretBossState.isHopping or secretBossState.isNormalHopping) and msg and msg ~= "" then
                secretBossState.HandleTeleportError(msg)
            end
        end))
    end)
end

secretBossState.weatherHopChoices = {
    "Bất Kỳ Thời Tiết Nào (Trừ Clear)",
    "Thunderstorm (Bão Sấm)",
    "Snowy (Bão Tuyết)",
    "Foggy (Sương Mù)",
    "Blazing Sun (Nắng Gắt)",
    "Rainy (Trời Mưa)",
    "Windy (Trời Gió)",
    "Acid Rain (Mưa Axit)",
    "Blood Moon (Trăng Máu)",
    "Boss Realm",
}

function secretBossState.IsWeatherMatch(currentWeatherName, targetWeather)
    if not currentWeatherName or currentWeatherName == "" or currentWeatherName == "Clear" then
        return false
    end
    if not targetWeather or targetWeather == "Bất Kỳ Thời Tiết Nào (Trừ Clear)" or targetWeather == "Bất Kỳ Thời Tiết Nào" then
        return currentWeatherName ~= "Clear"
    end
    local curLow = currentWeatherName:lower()
    local tgtLow = targetWeather:lower()

    for _, kw in ipairs({"thunderstorm", "snow", "fog", "blazing", "rain", "wind", "acid", "blood", "realm", "sấm", "tuyết", "sương", "nắng", "mưa", "gió", "axit", "trăng máu"}) do
        if tgtLow:find(kw) and curLow:find(kw) then
            return true
        end
    end
    return curLow:find(tgtLow, 1, true) ~= nil or tgtLow:find(curLow, 1, true) ~= nil
end

function secretBossState.HopToNextWeatherServer(targetWeather, visitedServers)
    if not secretBossState.isHopping or not Config.AutoWeatherHop then return end
    secretBossState.isHopping = true
    Config.AutoWeatherHop = true
    secretBossState.currentTargetWeather = targetWeather
    visitedServers = visitedServers or {}
    secretBossState.currentVisited = visitedServers
    if not table.find(visitedServers, game.JobId) then
        table.insert(visitedServers, game.JobId)
    end

    if writefile then
        pcall(function()
            writefile("HeavyweightFishing_WeatherHop.json", HttpService:JSONEncode({
                Active = true,
                TargetWeather = targetWeather,
                Visited = visitedServers,
                AutoFish = Config.WeatherHopAutoFish,
                Webhook = Config.WeatherHopAlertWebhook,
                StartTime = tick()
            }))
        end)
    end

    local qot = queue_on_teleport or (syn and syn.queue_on_teleport) or (fluxus and fluxus.queue_on_teleport)
    if qot then
        pcall(function()
            qot([[loadstring(game:HttpGet("https://raw.githubusercontent.com/VNGteam/Heavyweight-Fishing/main/loader.lua"))()]])
        end)
    end

    ShowNotification("Tìm Server", string.format("Đang quét server còn chỗ trống cho: %s...", targetWeather), "WARN", 5)

    task.spawn(function()
        local placeId = game.PlaceId
        local cursor = ""
        local visitedMap = {}
        for _, jid in ipairs(visitedServers) do visitedMap[jid] = true end

        local tier1 = {} -- freeSlots >= 3, playing >= 3
        local tier2 = {} -- freeSlots >= 2
        local tier3 = {} -- freeSlots >= 1

        for page = 1, 5 do
            if not isRunning or not secretBossState.isHopping or not Config.AutoWeatherHop then return end
            local url = string.format("https://games.roblox.com/v1/games/%d/servers/Public?sortOrder=Desc&limit=100%s", placeId, cursor ~= "" and ("&cursor=" .. cursor) or "")
            local ok, res = pcall(function() return game:HttpGet(url) end)
            if ok and res then
                local bOk, body = pcall(function() return HttpService:JSONDecode(res) end)
                if bOk and body and body.data then
                    for _, s in ipairs(body.data) do
                        if s.id and s.id ~= game.JobId and not visitedMap[s.id] and s.playing and s.maxPlayers then
                            local free = s.maxPlayers - s.playing
                            if free >= 3 and s.playing >= 3 then
                                table.insert(tier1, s.id)
                            elseif free >= 2 then
                                table.insert(tier2, s.id)
                            elseif free >= 1 then
                                table.insert(tier3, s.id)
                            end
                        end
                    end
                    if #tier1 >= 5 or #tier2 >= 8 then break end
                    cursor = body.nextPageCursor or ""
                    if not cursor or cursor == "" then break end
                end
            end
            task.wait(0.2)
        end

        if not isRunning or not secretBossState.isHopping or not Config.AutoWeatherHop then return end

        local chosenPool = (#tier1 > 0 and tier1) or (#tier2 > 0 and tier2) or tier3
        local foundJob = nil
        if #chosenPool > 0 then
            foundJob = chosenPool[math.random(1, #chosenPool)]
        end

        if foundJob then
            secretBossState.lastAttemptedJob = foundJob
            ShowNotification("Đổi Server", "Đã chọn server còn chỗ trống! Đang chuyển...", "SUCCESS", 4)
            task.wait(0.5)

            if not isRunning or not secretBossState.isHopping or not Config.AutoWeatherHop then return end

            secretBossState.hopWatchdog = (secretBossState.hopWatchdog or 0) + 1
            local myWatchdog = secretBossState.hopWatchdog
            local myOrigJob = game.JobId
            task.delay(10, function()
                if secretBossState.isHopping and Config.AutoWeatherHop and secretBossState.hopWatchdog == myWatchdog and game.JobId == myOrigJob then
                    pcall(function()
                        local gs = game:GetService("GuiService")
                        if gs and gs.ClearError then gs:ClearError() end
                    end)
                    if not table.find(visitedServers, foundJob) then
                        table.insert(visitedServers, foundJob)
                    end
                    ShowNotification("Đổi Server", "Không thể vào server (Server đầy / hết hạn). Tự động tìm server khác...", "WARN", 4)
                    task.wait(1)
                    if secretBossState.isHopping and Config.AutoWeatherHop then
                        secretBossState.HopToNextWeatherServer(targetWeather, visitedServers)
                    end
                end
            end)

            local teleOk, teleErr = pcall(function()
                TeleportService:TeleportToPlaceInstance(placeId, foundJob, LocalPlayer)
            end)
            if not teleOk then
                table.insert(visitedServers, foundJob)
                ShowNotification("Đổi Server", "Lỗi kết nối (" .. tostring(teleErr):sub(1, 40) .. "). Đang thử server khác...", "WARN", 4)
                task.wait(1.5)
                if secretBossState.isHopping and Config.AutoWeatherHop then
                    secretBossState.HopToNextWeatherServer(targetWeather, visitedServers)
                end
            end
        else
            if not isRunning or not secretBossState.isHopping or not Config.AutoWeatherHop then return end
            ShowNotification("Tìm Server", "Không tìm thấy server còn chỗ. Đang chuyển sang server ngẫu nhiên...", "INFO", 4)
            task.wait(1.5)
            if not isRunning or not secretBossState.isHopping or not Config.AutoWeatherHop then return end
            pcall(function() TeleportService:Teleport(placeId, LocalPlayer) end)
        end
    end)
end

function secretBossState.CheckWeatherHopOnJoin()
    local hopData = pendingWeatherHopData
    if not hopData and isfile and isfile("HeavyweightFishing_WeatherHop.json") and readfile then
        local ok, data = pcall(function()
            return HttpService:JSONDecode(readfile("HeavyweightFishing_WeatherHop.json"))
        end)
        if ok and type(data) == "table" and data.Active then
            hopData = data
        end
    end

    if not hopData then return end

    local targetWeather = hopData.TargetWeather or Config.TargetWeather or "Bất Kỳ Thời Tiết Nào (Trừ Clear)"
    secretBossState.isHopping = true
    Config.AutoWeatherHop = true
    secretBossState.currentTargetWeather = targetWeather
    if hopData.AutoFish ~= nil then Config.WeatherHopAutoFish = hopData.AutoFish end
    if hopData.Webhook ~= nil then Config.WeatherHopAlertWebhook = hopData.Webhook end

    -- Đồng bộ giao diện sang trạng thái BẬT
    if secretBossState.weatherHopToggle and secretBossState.weatherHopToggle.Set then
        pcall(function() secretBossState.weatherHopToggle.Set(true, true) end)
    elseif UIControllers["AutoWeatherHop"] and UIControllers["AutoWeatherHop"].Set then
        pcall(function() UIControllers["AutoWeatherHop"].Set(true, true) end)
    end
    if UIControllers["TargetWeather"] and UIControllers["TargetWeather"].Set then
        pcall(function() UIControllers["TargetWeather"].Set(targetWeather, true) end)
    end

    ShowNotification("Tìm Server", string.format("Đang quét thời tiết server cho: %s...", targetWeather), "INFO", 5)

    if not game:IsLoaded() then
        pcall(function() game.Loaded:Wait() end)
    end

    task.delay(3.0, function()
        if not isRunning or not secretBossState.isHopping or not Config.AutoWeatherHop then return end

        local matchedEntry, weatherName = secretBossState.DetectWeather()
        local isMatch = secretBossState.IsWeatherMatch(weatherName, targetWeather)

        -- Nếu chưa phát hiện hoặc là Clear, kiểm tra thêm 1 lần sau 1.5s để đảm bảo replication đầy đủ
        if not isMatch and (not weatherName or weatherName == "" or weatherName == "Clear") then
            task.wait(1.5)
            if not isRunning or not secretBossState.isHopping or not Config.AutoWeatherHop then return end
            matchedEntry, weatherName = secretBossState.DetectWeather()
            isMatch = secretBossState.IsWeatherMatch(weatherName, targetWeather)
        end

        if not isRunning or not secretBossState.isHopping or not Config.AutoWeatherHop then return end

        if isMatch then
            secretBossState.isHopping = false
            Config.AutoWeatherHop = false
            secretBossState.hopWatchdog = (secretBossState.hopWatchdog or 0) + 1
            if isfile and isfile("HeavyweightFishing_WeatherHop.json") and delfile then
                pcall(function() delfile("HeavyweightFishing_WeatherHop.json") end)
            end

            if secretBossState.weatherHopToggle and secretBossState.weatherHopToggle.Set then
                pcall(function() secretBossState.weatherHopToggle.Set(false, true) end)
            elseif UIControllers["AutoWeatherHop"] and UIControllers["AutoWeatherHop"].Set then
                pcall(function() UIControllers["AutoWeatherHop"].Set(false, true) end)
            end

            local dispName = weatherName or "Đặc Biệt"
            ShowNotification("TÌM THẤY THỜI TIẾT", string.format("🎉 ĐÃ TÌM THẤY SERVER!\nThời tiết hiện tại: %s\nMục tiêu: %s", dispName, targetWeather), "SUCCESS", 12)

            pcall(function()
                local sound = Instance.new("Sound")
                sound.SoundId = "rbxassetid://6534948092"
                sound.Volume = 1
                sound.Parent = Workspace
                sound:Play()
                game:GetService("Debris"):AddItem(sound, 5)
            end)

            if hopData.Webhook and Config.WebhookEnabled and Config.WebhookUrl ~= "" then
                SendDiscordWebhook({
                    embeds = {{
                        title = "🌩️ TÌM THẤY SERVER THỜI TIẾT!",
                        description = string.format("**Người chơi:** %s\n**Thời tiết:** %s\n**Mục tiêu:** %s\n**JobId:** `%s`", LocalPlayer.DisplayName, dispName, targetWeather, game.JobId),
                        color = 65280,
                        timestamp = DateTime.now():ToIsoDate()
                    }}
                })
            end

            if Config.NtfyEnabled and Config.NtfyAlertWeatherHop then
                pcall(function()
                    local jobId = tostring(game.JobId or "")
                    local placeId = tostring(game.PlaceId or "18779600655")
                    local ntfyTitle = "🎉 TÌM THẤY SERVER: " .. tostring(dispName):upper()
                    local ntfyMsg = string.format("Đã tìm thấy server thời tiết [%s] khớp mục tiêu [%s]!\n👤 Người chơi: %s\n🔑 Job ID: %s\n⚡ Lệnh vào nhanh:\ngame:GetService(\"TeleportService\"):TeleportToPlaceInstance(%s, \"%s\", game.Players.LocalPlayer)\n⏰ %s",
                        dispName, targetWeather, LocalPlayer.DisplayName or LocalPlayer.Name, (jobId ~= "" and jobId or "N/A"), placeId, jobId, os.date("%H:%M:%S - %d/%m/%Y"))
                    SendNtfyNotification(ntfyTitle, ntfyMsg, 5, {"tada", "cloud", "white_check_mark"})
                end)
            end

            if hopData.AutoFish then
                task.delay(1.5, function()
                    if matchedEntry and matchedEntry.pos then
                        local char = LocalPlayer.Character
                        local root = char and char:FindFirstChild("HumanoidRootPart")
                        if root then
                            root.CFrame = CFrame.new(matchedEntry.pos + Vector3.new(0, 2, 0))
                            ShowNotification("Săn Boss", "Đã tự bay đến " .. matchedEntry.islandName .. " để câu cá / săn boss!", "SUCCESS", 5)
                        end
                    end
                    Config.AutoCast = true
                    Config.AutoHuntBoss = true
                    if UIControllers["AutoCast"] and UIControllers["AutoCast"].Set then
                        pcall(function() UIControllers["AutoCast"].Set(true, false) end)
                    end
                    if UIControllers["AutoHuntBoss"] and UIControllers["AutoHuntBoss"].Set then
                        pcall(function() UIControllers["AutoHuntBoss"].Set(true, false) end)
                    end
                end)
            end
        else
            if not isRunning or not secretBossState.isHopping or not Config.AutoWeatherHop then return end
            local curDisplay = (weatherName and weatherName ~= "" and weatherName ~= "Clear") and weatherName or "Clear (Trời Quang)"
            ShowNotification("Tìm Server", string.format("Thời tiết hiện tại: %s (Không khớp). Tiếp tục đổi server...", curDisplay), "WARN", 3)
            task.wait(1.5)
            if not isRunning or not secretBossState.isHopping or not Config.AutoWeatherHop then return end
            secretBossState.HopToNextWeatherServer(targetWeather, hopData.Visited or {})
        end
    end)
end

function secretBossState.CheckNPCHopOnJoin()
    local hopData = pendingNPCHopData
    if not hopData and isfile and isfile("HeavyweightFishing_NPCHop.json") and readfile then
        local ok, data = pcall(function()
            return HttpService:JSONDecode(readfile("HeavyweightFishing_NPCHop.json"))
        end)
        if ok and type(data) == "table" and data.Active then
            hopData = data
        end
    end

    if not hopData then return end

    if hopData.Taoist then Config.AutoServerHopTaoist = true end
    if hopData.Maoshan then Config.AutoServerHopMaoshan = true end
    if hopData.God then Config.AutoServerHopGod = true end

    task.delay(1.5, function()
        if UIControllers["AutoServerHopTaoist"] and UIControllers["AutoServerHopTaoist"].Set then
            pcall(function() UIControllers["AutoServerHopTaoist"].Set(Config.AutoServerHopTaoist, true) end)
        end
        if UIControllers["AutoServerHopMaoshan"] and UIControllers["AutoServerHopMaoshan"].Set then
            pcall(function() UIControllers["AutoServerHopMaoshan"].Set(Config.AutoServerHopMaoshan, true) end)
        end
        if UIControllers["AutoServerHopGod"] and UIControllers["AutoServerHopGod"].Set then
            pcall(function() UIControllers["AutoServerHopGod"].Set(Config.AutoServerHopGod, true) end)
        end
    end)

    ShowNotification("Đổi Server", "Đang kiểm tra NPC / Thần Linh trong server mới...", "INFO", 5)
end

local ticketQuestState = {}

function ticketQuestState.GetServerTimeNow()
    local ok, t = pcall(function() return Workspace:GetServerTimeNow() end)
    if ok and t and t > 0 then return math.floor(t) end
    return os.time()
end

function ticketQuestState.IsFishingActive()
    if not Config.AutoTicketQuest then return false end
    if ticketQuestState.isCompleted then return false end
    if ticketQuestState.isInteracting then return false end
    if ticketQuestState.isCooldown then
        return Config.TicketReturnHomeWhenDone and Config.TicketAutoCastAtHome and (ticketQuestState.isAtHomeSpot == true)
    else
        return ticketQuestState.active and (ticketQuestState.currentQuestType ~= "none")
    end
end

function ticketQuestState.IsBusyOrInteracting()
    if not Config.AutoTicketQuest then return false end
    if ticketQuestState.isInteracting then return true end
    if ticketQuestState.isCompleted then return true end
    if ticketQuestState.IsDialogueOpen and ticketQuestState.IsDialogueOpen() then return true end
    if not ticketQuestState.isCooldown and ticketQuestState.currentQuestType == "none" then return true end
    return false
end

local function CancelAndRecastRod(forceCast)
    if not forceCast and ticketQuestState and ticketQuestState.IsBusyOrInteracting and ticketQuestState.IsBusyOrInteracting() then return end
    local char = LocalPlayer.Character
    if not char then return end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if hum then
        hum:UnequipTools()
    end
    local isBossActive = secretBossState and secretBossState.active
    local isTicketActive = ticketQuestState and ticketQuestState.IsFishingActive and ticketQuestState.IsFishingActive()
    local shouldRecast = forceCast or Config.AutoCast or Config.AutoTrainSkill or isTicketActive or ((Config.AutoHuntBoss or Config.AutoChatSecretBoss) and isBossActive)
    if not shouldRecast then return end

    task.delay(0.2, function()
        if not isRunning then return end
        local rodSlot = "1"
        local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
        if pData and pData:FindFirstChild("Hotbar") then
            for _, item in ipairs(pData.Hotbar:GetChildren()) do
                local vName = item:FindFirstChild("ValueName")
                if vName and tostring(vName.Value):lower():find("rod") and not tostring(vName.Value):lower():find("inventory") then
                    rodSlot = item.Name
                    break
                end
            end
        end
        if Events and Events:FindFirstChild("ToggleHotbar") then
            Events.ToggleHotbar:InvokeServer(rodSlot)
        end
        task.delay(0.25, function()
            if not isRunning then return end
            local c = LocalPlayer.Character
            local r = c and c:FindFirstChild("HumanoidRootPart")
            if r and Events and Events:FindFirstChild("Fishing") then
                Events.Fishing:FireServer(r.CFrame)
                lastCastTime = tick()
            end
        end)
    end)
end

function secretBossState.DetectWeatherPattern(text)
    if not text or typeof(text) ~= "string" then return nil, nil end
    local lower = text:lower()

    -- 1. Kiểm tra thời tiết quang đãng / bình thường (Clear) -> Không có Boss thời tiết
    if lower:find("clear") or lower:find("sunny") or lower:find("normal") or lower:find("none")
       or lower:find("trong xanh") or lower:find("bình thường") or lower:find("binh thuong") then
        return nil, "Clear"
    end

    -- 2. Khớp theo weatherPatterns của từng đảo
    for _, entry in ipairs(secretBossDatabase) do
        if entry.weatherPatterns then
            for _, wp in ipairs(entry.weatherPatterns) do
                if lower:find(wp, 1, true) then
                    return entry, entry.weather
                end
            end
        end
    end

    -- 3. Khớp theo tên Boss xuất hiện trong mô tả thời tiết
    for _, entry in ipairs(secretBossDatabase) do
        if entry.bossPatterns then
            for _, bp in ipairs(entry.bossPatterns) do
                if lower:find(bp, 1, true) then
                    return entry, bp
                end
            end
        end
        for _, b in ipairs(entry.bosses) do
            if lower:find(b.name:lower(), 1, true) then
                return entry, b.name
            end
        end
    end

    return nil, nil
end

function secretBossState.DetectIsland(text)
    if not text or typeof(text) ~= "string" then return nil, nil end
    local lower = text:lower()

    -- Bỏ qua nếu là thông báo người chơi câu được cá (KHÔNG PHẢI BOSS XUẤT HIỆN)
    if lower:find("caught") or lower:find("has caught") or lower:find("câu được") or lower:find("đã câu") or lower:find("bắt được") or lower:find("obtained") then
        return nil, nil
    end

    -- 0. Nếu văn bản là thời tiết Clear / bình thường -> Bỏ qua, không phải đảo boss nào
    if lower:find("clear") or lower:find("sunny") or lower:find("trong xanh") or lower:find("bình thường") or lower:find("binh thuong") then
        return nil, "Clear"
    end

    -- 1. ƯU TIÊN CAO NHẤT: Khớp theo TÊN CHÍNH XÁC của Boss
    for _, entry in ipairs(secretBossDatabase) do
        if entry.bossPatterns then
            for _, bp in ipairs(entry.bossPatterns) do
                if lower:find(bp, 1, true) then
                    return entry, bp
                end
            end
        end
        for _, b in ipairs(entry.bosses) do
            if lower:find(b.name:lower(), 1, true) then
                return entry, b.name
            end
        end
    end

    -- 2. Khớp theo TÊN THỜI TIẾT (Weather Patterns)
    for _, entry in ipairs(secretBossDatabase) do
        if entry.weatherPatterns then
            for _, wp in ipairs(entry.weatherPatterns) do
                if lower:find(wp, 1, true) then
                    return entry, entry.weather
                end
            end
        end
    end

    -- 3. Khớp theo TÊN ĐẢO rõ ràng (Chỉ khi văn bản có kèm từ khóa liên quan đến boss/thời tiết/xuất hiện)
    local hasContext = lower:find("boss") or lower:find("secret") or lower:find("spawn") or lower:find("appear")
        or lower:find("xuất hiện") or lower:find("weather") or lower:find("thời tiết") or lower:find("bão")
    if hasContext then
        for _, entry in ipairs(secretBossDatabase) do
            for _, pat in ipairs(entry.patterns) do
                if lower:find(pat, 1, true) then
                    return entry, nil
                end
            end
        end
    end

    return nil, nil
end

function secretBossState.FindWaterSpot(centerPos, preferredLookAt)
    local rp = RaycastParams.new()
    rp.FilterType = Enum.RaycastFilterType.Exclude
    rp.IgnoreWater = false

    local filterList = {}
    if LocalPlayer.Character then table.insert(filterList, LocalPlayer.Character) end
    local wp = Workspace:FindFirstChild("IdenticalWaterPlatform")
    if wp then table.insert(filterList, wp) end
    rp.FilterDescendantsInstances = filterList

    -- 1. Hướng nhìn ưu tiên
    local preferredDir = nil
    if preferredLookAt then
        local pDir = Vector3.new(preferredLookAt.X - centerPos.X, 0, preferredLookAt.Z - centerPos.Z)
        if pDir.Magnitude > 0.1 then
            preferredDir = pDir.Unit
        end
    end

    local angles = {}
    local baseAngle = 0
    if preferredDir then
        baseAngle = math.atan2(preferredDir.Z, preferredDir.X)
    end

    -- Đặt góc ưu tiên lên đầu tiên, sau đó tỏa ra xung quanh theo nan hoa 16 hướng
    table.insert(angles, baseAngle)
    for i = 1, 8 do
        local offset = (i * math.pi / 8)
        table.insert(angles, baseAngle + offset)
        if offset < math.pi then
            table.insert(angles, baseAngle - offset)
        end
    end

    -- Bán kính quét mở rộng từ gần đến xa (15 đến 220 studs)
    local testDistances = {15, 30, 50, 75, 105, 140, 180, 220}
    local bestWaterHit = nil

    for _, dist in ipairs(testDistances) do
        for _, ang in ipairs(angles) do
            local dirX = math.cos(ang)
            local dirZ = math.sin(ang)
            local sampleX = centerPos.X + dirX * dist
            local sampleZ = centerPos.Z + dirZ * dist

            -- Bắn tia từ trên trời xuống để tìm Nước
            local hit = Workspace:Raycast(Vector3.new(sampleX, 45, sampleZ), Vector3.new(0, -75, 0), rp)
            if hit then
                local isWater = (hit.Material == Enum.Material.Water)
                if not isWater and hit.Instance then
                    local nameLower = hit.Instance.Name:lower()
                    if nameLower:find("water") or nameLower:find("ocean") or nameLower:find("sea") then
                        isWater = true
                    end
                end

                if isWater then
                    bestWaterHit = hit
                    break
                end
            end
        end
        if bestWaterHit then break end
    end

    -- Nếu tìm thấy vùng nước:
    if bestWaterHit then
        local waterPos = bestWaterHit.Position
        local waterLevel = waterPos.Y
        -- Hướng từ tâm đảo ra vùng nước
        local outwardDir = Vector3.new(waterPos.X - centerPos.X, 0, waterPos.Z - centerPos.Z)
        if outwardDir.Magnitude > 0.1 then
            outwardDir = outwardDir.Unit
        else
            outwardDir = preferredDir or Vector3.new(0, 0, 1)
        end

        -- Dò ngược từ vùng nước về phía tâm đảo để tìm mép bờ đất (Shoreline)
        local shorePos = nil
        for backStep = 1, 25 do
            local testBackPos = waterPos - outwardDir * (backStep * 2.0)
            local groundHit = Workspace:Raycast(Vector3.new(testBackPos.X, 45, testBackPos.Z), Vector3.new(0, -75, 0), rp)
            if groundHit and groundHit.Material ~= Enum.Material.Water then
                -- Tìm thấy bờ đất liền kề nước!
                shorePos = Vector3.new(groundHit.Position.X, groundHit.Position.Y + 2.5, groundHit.Position.Z)
                break
            end
        end

        -- Điểm đứng lý tưởng:
        -- Nếu tìm được mép bờ, đứng ở mép bờ nhìn thẳng ra biển
        -- Nếu không tìm được mép bờ (đảo phẳng chìm hoặc xa bờ), đứng ngay sát mép nước
        local standPos = shorePos or Vector3.new(waterPos.X - outwardDir.X * 4, waterLevel + 2.5, waterPos.Z - outwardDir.Z * 4)
        local lookTarget = standPos + outwardDir * 50

        return standPos, lookTarget, waterLevel, true
    end

    -- Fallback: Nếu không quét ra tia nước, dùng vị trí mặc định
    local fallbackStand = centerPos + Vector3.new(0, 2.5, 0)
    local fallbackLook = preferredLookAt or (centerPos + Vector3.new(0, 2.5, 50))
    return fallbackStand, fallbackLook, centerPos.Y, false
end

function secretBossState.SaveCustomSpots()
    if not writefile then return end
    pcall(function()
        local data = {}
        if Config.CustomBossSpots then
            for k, v in pairs(Config.CustomBossSpots) do
                data[k] = v
            end
        end
        writefile("heavyweight_custom_spots.json", HttpService:JSONEncode(data))
    end)
end

function secretBossState.LoadCustomSpots()
    if not readfile or not isfile or not isfile("heavyweight_custom_spots.json") then return end
    pcall(function()
        local content = readfile("heavyweight_custom_spots.json")
        if content and #content > 0 then
            local decoded = HttpService:JSONDecode(content)
            if type(decoded) == "table" then
                Config.CustomBossSpots = Config.CustomBossSpots or {}
                for k, v in pairs(decoded) do
                    Config.CustomBossSpots[k] = v
                end
            end
        end
    end)
end

function secretBossState.SaveHomeSpot()
    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if not root then return false, "Không tìm thấy nhân vật!" end

    local cf = root.CFrame
    local spotData = {
        x = cf.Position.X,
        y = cf.Position.Y,
        z = cf.Position.Z,
        cframe = {cf:GetComponents()},
        savedAt = os.date("%H:%M:%S - %d/%m/%Y")
    }
    Config.HomeFarmSpot = spotData

    if writefile then
        pcall(function()
            local fName = string.format("heavyweight_home_spot_%s.json", tostring(LocalPlayer.UserId))
            writefile(fName, HttpService:JSONEncode(spotData))
        end)
    end
    return true, spotData
end

function secretBossState.LoadHomeSpot()
    if not readfile or not isfile then return end
    local fName = string.format("heavyweight_home_spot_%s.json", tostring(LocalPlayer.UserId))
    if not isfile(fName) then return end
    pcall(function()
        local content = readfile(fName)
        if content and #content > 0 then
            local decoded = HttpService:JSONDecode(content)
            if type(decoded) == "table" and (decoded.cframe or decoded.x) then
                Config.HomeFarmSpot = decoded
            end
        end
    end)
end

function secretBossState.ClearHomeSpot()
    Config.HomeFarmSpot = nil
    if delfile and isfile then
        pcall(function()
            local fName = string.format("heavyweight_home_spot_%s.json", tostring(LocalPlayer.UserId))
            if isfile(fName) then delfile(fName) end
        end)
    elseif writefile then
        pcall(function()
            local fName = string.format("heavyweight_home_spot_%s.json", tostring(LocalPlayer.UserId))
            writefile(fName, "")
        end)
    end
end

function secretBossState.ReturnToHome()
    if not Config.HomeFarmSpot then return false end
    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if not root then return false end

    local homeCf = nil
    if Config.HomeFarmSpot.cframe and #Config.HomeFarmSpot.cframe == 12 then
        homeCf = CFrame.new(table.unpack(Config.HomeFarmSpot.cframe))
    elseif Config.HomeFarmSpot.x and Config.HomeFarmSpot.y and Config.HomeFarmSpot.z then
        homeCf = CFrame.new(Config.HomeFarmSpot.x, Config.HomeFarmSpot.y, Config.HomeFarmSpot.z)
    end
    if not homeCf then return false end

    local dist = (root.Position - homeCf.Position).Magnitude
    if dist > 35 then
        if (tick() - (secretBossState.lastHomeReturnTime or 0)) < 4.0 then
            return false
        end
        secretBossState.lastHomeReturnTime = tick()

        ShowNotification("VỀ VỊ TRÍ FARM", "Thời tiết Clear / Hết Boss! Đang quay về vị trí Farm để câu cá kiếm tiền...", "INFO", 5)
        if statusLabelSecretBoss and statusLabelSecretBoss.Set then
            statusLabelSecretBoss.Set("Thời tiết Clear / Hết Boss -> Đã về vị trí Farm câu tiền...")
        end

        local wp = Workspace:FindFirstChild("IdenticalWaterPlatform") or Workspace:FindFirstChild("WaterPlatform")
        if not wp then
            wp = Instance.new("Part")
            wp.Name = "IdenticalWaterPlatform"
            wp.Size = Vector3.new(30, 2, 30)
            wp.Transparency = 1
            wp.Anchored = true
            wp.CanCollide = true
            wp.Parent = Workspace
        end
        wp.CFrame = CFrame.new(homeCf.Position.X, homeCf.Position.Y - 2.8, homeCf.Position.Z)
        wp.CanCollide = true

        root.CFrame = homeCf + Vector3.new(0, 1.5, 0)
        task.wait(0.2)
        root.CFrame = homeCf

        secretBossState.active = false
        secretBossState.currentMap = "Home Farm"
        secretBossState.standPos = homeCf.Position

        -- Bắt đầu quăng cần sau 1.5s nếu có bật AutoCast hoặc AutoTrainSkill
        task.delay(1.5, function()
            if not isRunning then return end
            if Config.AutoCast or Config.AutoTrainSkill then
                CancelAndRecastRod()
            end
        end)
        return true
    end
    return false
end

function secretBossState.GetNearestIsland()
    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if not root then return nil, 999999 end
    local pos = root.Position
    local nearest = nil
    local minDist = 999999

    for _, entry in ipairs(secretBossDatabase) do
        local checkPos = entry.pos
        local dist = (pos - checkPos).Magnitude
        if dist < minDist then
            minDist = dist
            nearest = entry
        end
    end
    return nearest, minDist
end

function secretBossState.GetIslandSpots(islandName)
    local spots = {}
    local custom = Config.CustomBossSpots and Config.CustomBossSpots[islandName]

    if custom then
        if custom.spots and type(custom.spots) == "table" then
            for i = 1, 3 do
                local s = custom.spots[i] or custom.spots[tostring(i)]
                if s and s.cframe then
                    table.insert(spots, { slot = i, cframe = s.cframe, savedAt = s.savedAt })
                end
            end
        end

        if #spots == 0 and custom.cframe then
            table.insert(spots, { slot = 1, cframe = custom.cframe, savedAt = custom.savedAt })
        end
    end

    -- Nếu người chơi chưa tự lưu vị trí riêng, tự động dùng vị trí chuẩn đã nạp sẵn trong script gốc
    if #spots == 0 then
        for _, entry in ipairs(secretBossDatabase) do
            if entry.islandName == islandName then
                if entry.spots and #entry.spots > 0 then
                    for idx, sp in ipairs(entry.spots) do
                        local cf = CFrame.lookAt(sp.pos, sp.lookAt)
                        table.insert(spots, { slot = idx, cframe = {cf:GetComponents()}, savedAt = "Mặc Định (Gốc)" })
                    end
                elseif entry.pos and entry.lookAt then
                    local cf = CFrame.lookAt(entry.pos, entry.lookAt)
                    table.insert(spots, { slot = 1, cframe = {cf:GetComponents()}, savedAt = "Mặc Định (Gốc)" })
                end
                break
            end
        end
    end

    return spots
end

function secretBossState.GetIslandSlotSpot(islandName, slot)
    local custom = Config.CustomBossSpots and Config.CustomBossSpots[islandName]
    if custom then
        if custom.spots and type(custom.spots) == "table" then
            local s = custom.spots[slot] or custom.spots[tostring(slot)]
            if s and s.cframe then return s end
        end
        if slot == 1 and custom.cframe then
            return { cframe = custom.cframe, savedAt = custom.savedAt }
        end
    end

    -- Fallback vào vị trí chuẩn trong database
    for _, entry in ipairs(secretBossDatabase) do
        if entry.islandName == islandName then
            if entry.spots and (entry.spots[slot] or entry.spots[tostring(slot)]) then
                local sp = entry.spots[slot] or entry.spots[tostring(slot)]
                local cf = CFrame.lookAt(sp.pos, sp.lookAt)
                return { cframe = {cf:GetComponents()}, savedAt = "Mặc Định (Gốc)" }
            elseif slot == 1 and entry.pos and entry.lookAt then
                local cf = CFrame.lookAt(entry.pos, entry.lookAt)
                return { cframe = {cf:GetComponents()}, savedAt = "Mặc Định (Gốc)" }
            end
        end
    end
    return nil
end

function secretBossState.SaveIslandSlot(islandName, slot, cfComponents)
    Config.CustomBossSpots = Config.CustomBossSpots or {}
    local entry = Config.CustomBossSpots[islandName]
    if not entry then
        entry = { spots = {} }
        Config.CustomBossSpots[islandName] = entry
    elseif not entry.spots then
        local oldCf = entry.cframe
        local oldSaved = entry.savedAt
        entry.spots = {}
        if oldCf then
            entry.spots[1] = { cframe = oldCf, savedAt = oldSaved }
        end
    end

    entry.spots[slot] = {
        cframe = cfComponents,
        savedAt = os.date("%H:%M:%S")
    }
    local s1 = entry.spots[1] or entry.spots["1"]
    entry.cframe = s1 and s1.cframe or cfComponents
    secretBossState.SaveCustomSpots()
end

function secretBossState.DeleteIslandSlot(islandName, slot)
    if not Config.CustomBossSpots or not Config.CustomBossSpots[islandName] then return end
    local entry = Config.CustomBossSpots[islandName]
    if not slot or slot == 0 then
        Config.CustomBossSpots[islandName] = nil
    else
        if entry.spots then
            entry.spots[slot] = nil
            entry.spots[tostring(slot)] = nil
        end
        if slot == 1 then
            entry.cframe = nil
        end
        local hasRemaining = false
        if entry.spots then
            for i = 1, 3 do
                if entry.spots[i] or entry.spots[tostring(i)] then
                    hasRemaining = true
                    break
                end
            end
        end
        if not hasRemaining then
            Config.CustomBossSpots[islandName] = nil
        end
    end
    secretBossState.SaveCustomSpots()
end

function secretBossState.ApplyJitter(baseCf)
    if not Config.BossTeleportJitter then return baseCf end
    local maxDist = Config.BossTeleportJitterDist or 1.0
    if maxDist <= 0.1 then return baseCf end

    local dirSign = (math.random(1, 2) == 1) and 1 or -1
    local minDist = math.min(0.4, maxDist * 0.5)
    local offsetDist = dirSign * (minDist + math.random() * (maxDist - minDist))

    return baseCf + (baseCf.RightVector * offsetDist)
end

function secretBossState.GetChosenSpotForPlayer(islandName)
    local spots = secretBossState.GetIslandSpots(islandName)
    if #spots == 0 then return nil, nil, 0 end

    local mode = Config.BossSpotAllocationMode or "Tự Động (Theo Acc)"
    local chosen = nil

    if mode == "Vị Trí 1" then
        for _, s in ipairs(spots) do if s.slot == 1 then chosen = s; break end end
    elseif mode == "Vị Trí 2" then
        for _, s in ipairs(spots) do if s.slot == 2 then chosen = s; break end end
    elseif mode == "Vị Trí 3" then
        for _, s in ipairs(spots) do if s.slot == 3 then chosen = s; break end end
    elseif mode == "Ngẫu Nhiên" then
        chosen = spots[math.random(1, #spots)]
    else -- "Tự Động (Theo Acc)"
        local userId = (LocalPlayer and LocalPlayer.UserId) or 0
        local idx = (math.abs(userId) % #spots) + 1
        chosen = spots[idx]
    end

    if not chosen then
        chosen = spots[1]
    end

    local cf = CFrame.new(unpack(chosen.cframe))
    return cf, chosen.slot, #spots
end

secretBossState.cachedWeatherLabel = nil

function secretBossState.GetWeatherLabel()
    if secretBossState.cachedWeatherLabel and secretBossState.cachedWeatherLabel.Parent then
        return secretBossState.cachedWeatherLabel
    end
    local pg = LocalPlayer:FindFirstChild("PlayerGui")
    if not pg then return nil end
    local mainGui = pg:FindFirstChild("MainGui")
    if not mainGui then return nil end

    -- 1. Ưu tiên đường dẫn trực tiếp cực chuẩn từ Game: MainGui.Info.Info.Weather.Value
    local info1 = mainGui:FindFirstChild("Info")
    if info1 then
        local subInfo = info1:FindFirstChild("Info") or info1
        local wFrame = subInfo:FindFirstChild("Weather")
        if wFrame then
            local val = wFrame:FindFirstChild("Value")
            if val and val:IsA("TextLabel") then
                secretBossState.cachedWeatherLabel = val
                return val
            end
        end
    end

    -- 2. Quét nhanh trong cụm Info (không quét cả 20,000 descendants MainGui)
    if info1 then
        for _, d in ipairs(info1:GetDescendants()) do
            if d:IsA("TextLabel") and d.Parent and d.Parent.Name:lower():find("weather") then
                secretBossState.cachedWeatherLabel = d
                return d
            end
        end
    end

    -- 3. Quét dự phòng toàn bộ MainGui (chỉ lấy đúng nhãn thời tiết, bỏ qua nhãn cá/nhiệm vụ/shop)
    for _, d in ipairs(mainGui:GetDescendants()) do
        if d:IsA("TextLabel") then
            local pName = d.Parent and d.Parent.Name:lower() or ""
            local dName = d.Name:lower()
            if (pName:find("weather") or dName:find("weather"))
                and not pName:find("island") and not dName:find("island")
                and not pName:find("fish") and not dName:find("fish")
                and not pName:find("shop") and not dName:find("shop")
                and not pName:find("craft") and not dName:find("craft") then
                secretBossState.cachedWeatherLabel = d
                return d
            end
        end
    end

    return nil
end

function secretBossState.Teleport(matchedIsland, detectedName, reqPower)
    if not matchedIsland or not matchedIsland.pos then return false end

    -- Nếu nhân vật đã ở đúng đảo mục tiêu và đã đứng tại vị trí câu rồi thì KHÔNG teleport lại tránh gián đoạn cần câu
    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if root and secretBossState.currentMap == matchedIsland.islandName and secretBossState.standPos then
        local d = (root.Position - secretBossState.standPos).Magnitude
        if d < 70 then
            return true
        end
    end

    -- Kiểm tra nếu người chơi có chọn săn ít nhất 1 boss ở đảo này không
    local hasTargetInIsland = false
    local hasAnyBossConfigured = false
    for _, isTarget in pairs(Config.SecretBossTargets or {}) do
        if isTarget == true then
            hasAnyBossConfigured = true
            break
        end
    end

    -- Nếu không có boss nào được bật (chưa chọn gì hoặc cấu hình trống), mặc định cho phép săn tất cả
    if not hasAnyBossConfigured then
        hasTargetInIsland = true
    else
        for _, b in ipairs(matchedIsland.bosses or {}) do
            local bNameLow = b.name:lower()
            for tName, isTgt in pairs(Config.SecretBossTargets or {}) do
                if isTgt and (tName:lower() == bNameLow or bNameLow:find(tName:lower(), 1, true) or tName:lower():find(bNameLow, 1, true)) then
                    hasTargetInIsland = true
                    break
                end
            end
            if hasTargetInIsland then break end
        end
    end

    if not hasTargetInIsland then
        ShowNotification("Bỏ Qua Boss", string.format("Phát hiện tại %s nhưng bạn không chọn săn boss ở đảo này.", matchedIsland.islandName), "INFO", 4)
        return false
    end

    local powerReq = reqPower or 0
    local curPower = GetPlayerRodPower()
    if Config.SecretBossCheckPower and powerReq > 0 and curPower < powerReq then
        ShowNotification("CẢNH BÁO LỰC CẦN", string.format("Boss yêu cầu %d Power! Cần của bạn chỉ có %d Power.", powerReq, curPower), "WARN", 8)
    end

    -- Cập nhật trạng thái săn
    secretBossState.active = true
    secretBossState.currentMap = matchedIsland.islandName
    secretBossState.targetIsland = matchedIsland
    secretBossState.requiredPower = powerReq
    secretBossState.statusText = string.format("Đang săn tại %s [%s]", matchedIsland.islandName, matchedIsland.weather or "Thời Tiết")

    local alertName = detectedName and string.upper(tostring(detectedName)) or "SECRET BOSS / THỜI TIẾT"
    ShowNotification("PHÁT HIỆN " .. alertName .. "!", string.format("Đang bay đến %s để câu boss...", matchedIsland.islandName), "SUCCESS", 7)

    if statusLabelSecretBoss and statusLabelSecretBoss.Set then
        statusLabelSecretBoss.Set(secretBossState.statusText)
    end

    -- Dịch chuyển nhân vật đến bờ biển câu
    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if root and matchedIsland.pos then
        local customCf, chosenSlot, totalSpots = secretBossState.GetChosenSpotForPlayer(matchedIsland.islandName)
        local standPos = nil
        local waterY = nil

        local targetCFrame = nil
        if customCf then
            -- 1. Ưu tiên số 1: Tọa độ tùy chọn đã cài đặt (có áp dụng xê dịch trái/phải né người)
            targetCFrame = secretBossState.ApplyJitter(customCf)
            standPos = targetCFrame.Position
            waterY = standPos.Y - 2.5
            local jitterTag = Config.BossTeleportJitter and " (+ xê dịch)" or ""
            ShowNotification("VỊ TRÍ TÙY CHỌN", string.format("Đã vào [Vị Trí %d/%d]%s tại %s!", chosenSlot or 1, totalSpots or 1, jitterTag, matchedIsland.islandName), "SUCCESS", 5)
        else
            -- 2. Dò tìm mép nước tự động (cũng áp dụng xê dịch nếu bật)
            local lookTarget, foundWater
            standPos, lookTarget, waterY, foundWater = secretBossState.FindWaterSpot(matchedIsland.pos, matchedIsland.lookAt)
            local baseCf = CFrame.lookAt(standPos, lookTarget)
            targetCFrame = secretBossState.ApplyJitter(baseCf)
            standPos = targetCFrame.Position
            if foundWater then
                ShowNotification("MÉP NƯỚC CÂU CÁ", string.format("Đã dò thấy vùng nước! Nhân vật đã vào vị trí mép bờ tại %s.", matchedIsland.islandName), "SUCCESS", 5)
            end
        end

        -- Đặt sàn an toàn dưới chân nếu gần mặt nước
        local wp = Workspace:FindFirstChild("IdenticalWaterPlatform")
        if wp and standPos then
            wp.CFrame = CFrame.new(standPos.X, (waterY or (standPos.Y - 2.5)) - 1.2, standPos.Z)
            wp.CanCollide = true
        end

        -- Triệt tiêu hoàn toàn vận tốc để chống văng / giật lùi anti-cheat
        pcall(function()
            root.AssemblyLinearVelocity = Vector3.zero
            root.AssemblyAngularVelocity = Vector3.zero
        end)

        root.CFrame = targetCFrame
        secretBossState.standPos = targetCFrame.Position

        -- Giữ CFrame ổn định trong 3 nhịp đầu (chống giật lại vị trí cũ bởi physics engine)
        task.spawn(function()
            for _ = 1, 3 do
                task.wait(0.1)
                if root and root.Parent and secretBossState.active and secretBossState.standPos then
                    root.AssemblyLinearVelocity = Vector3.zero
                    root.CFrame = targetCFrame
                end
            end
        end)

        -- Khởi động quăng cần câu sau 1.8s hạ cánh
        lastCastTime = tick() + 1.8
        CancelAndRecastRod()
    end
    return true
end

function secretBossState.DetectWeather()
    -- 1. ƯU TIÊN CAO NHẤT & CHÍNH XÁC 100%: Đọc nhãn thời tiết trực tiếp từ PlayerGui (HUD game)
    -- Bỏ qua kiểm tra d.Visible để tránh bị lỗi khi người chơi mở túi đồ, menu shop, hoặc giao diện giật cần!
    local wLabel = secretBossState.GetWeatherLabel()
    if wLabel and wLabel.Text and #wLabel.Text > 0 then
        local rawText = wLabel.Text
        local matched, wName = secretBossState.DetectWeatherPattern(rawText)
        if wName == "Clear" then
            return nil, "Clear"
        elseif matched then
            return matched, wName or rawText
        end
    end

    -- 2. Quét Workspace Attributes hoặc Objects
    local wsWeather = Workspace:GetAttribute("Weather") or Workspace:GetAttribute("CurrentWeather") or Workspace:GetAttribute("ActiveWeather")
    if typeof(wsWeather) == "string" and #wsWeather > 0 then
        local matched, wName = secretBossState.DetectWeatherPattern(wsWeather)
        if wName == "Clear" then
            return nil, "Clear"
        elseif matched then
            return matched, wName or wsWeather
        end
    end
    if Workspace:FindFirstChild("Weather") then
        local wObj = Workspace.Weather
        if wObj:IsA("StringValue") and #wObj.Value > 0 then
            local matched, wName = secretBossState.DetectWeatherPattern(wObj.Value)
            if wName == "Clear" then
                return nil, "Clear"
            elseif matched then
                return matched, wName or wObj.Value
            end
        end
    end

    -- 3. Quét ReplicatedStorage Attributes
    if ReplicatedStorage then
        local rsWeather = ReplicatedStorage:GetAttribute("Weather") or ReplicatedStorage:GetAttribute("CurrentWeather")
        if typeof(rsWeather) == "string" and #rsWeather > 0 then
            local matched, wName = secretBossState.DetectWeatherPattern(rsWeather)
            if wName == "Clear" then
                return nil, "Clear"
            elseif matched then
                return matched, wName or rsWeather
            end
        end
        if ReplicatedStorage:FindFirstChild("Weather") then
            local rwObj = ReplicatedStorage.Weather
            if rwObj:IsA("StringValue") and #rwObj.Value > 0 then
                local matched, wName = secretBossState.DetectWeatherPattern(rwObj.Value)
                if wName == "Clear" then
                    return nil, "Clear"
                elseif matched then
                    return matched, wName or rwObj.Value
                end
            end
        end
    end

    return nil, nil
end

secretBossState.cachedTaoist = nil
secretBossState.lastTaoistScan = 0
secretBossState.cachedMaoshan = nil
secretBossState.lastMaoshanScan = 0
secretBossState.cachedGod = nil
secretBossState.lastGodScan = 0

local function getCandidateNPCFolders()
    local candidateFolders = {}
    for _, fName in ipairs({"NPC", "NPCs", "Merchant", "Merchants", "Entities", "Characters"}) do
        local f = Workspace:FindFirstChild(fName)
        if f then
            table.insert(candidateFolders, f)
            for _, subName in ipairs({"Function", "Merchant", "Merchants", "Taoist", "SellFish", "SetSpawn", "BuyBait", "BuyFishingRod"}) do
                local sf = f:FindFirstChild(subName)
                if sf then table.insert(candidateFolders, sf) end
            end
        end
    end
    return candidateFolders
end

local function isTaoistText(str)
    if not str or typeof(str) ~= "string" or str == "" then return false end
    local s = str:lower()
    -- Tuyệt đối loại bỏ Blind Grand Angler, Grand Angler, hoặc Maoshan
    if s:find("angler") or s:find("blind") or s:find("maoshan") or s:find("mao shan") then
        return false
    end
    return (s:find("taoist") or s:find("đạo sĩ") or s:find("dao si") or s:find("daoshi")) ~= nil
end

local function isMaoshanText(str)
    if not str or typeof(str) ~= "string" or str == "" then return false end
    local s = str:lower()
    if s:find("angler") or s:find("blind") then
        return false
    end
    return (s:find("maoshan") or s:find("mao shan") or s:find("mao_shan")) ~= nil
end

local function isNPCModel(inst)
    if not inst or not inst:IsA("Model") then return false end
    return inst:FindFirstChildOfClass("Humanoid") ~= nil
        or inst:FindFirstChild("HumanoidRootPart") ~= nil
        or inst:FindFirstChild("Head") ~= nil
end

function secretBossState.ScanForTaoistNPC()
    local candidateFolders = getCandidateNPCFolders()

    -- Đợt 1: Quét Model NPC trực tiếp trong các folder
    for _, folder in ipairs(candidateFolders) do
        for _, n in ipairs(folder:GetChildren()) do
            if isNPCModel(n) and isTaoistText(n.Name) then
                return n, "Đạo Sĩ (Taoist)", "Taoist", Colors.AccentOrange, "📜"
            end
        end
    end

    -- Đợt 2: Quét ProximityPrompt và TextLabel trong các thư mục NPC
    for _, folder in ipairs(candidateFolders) do
        for _, d in ipairs(folder:GetDescendants()) do
            if d:IsA("ProximityPrompt") then
                local act = tostring(d.ActionText or "")
                local obj = tostring(d.ObjectText or "")
                local pName = d.Parent and d.Parent.Name or ""
                if isTaoistText(act) or isTaoistText(obj) or isTaoistText(pName) then
                    local model = d:FindFirstAncestorOfClass("Model") or d.Parent
                    if model and not (model.Name:lower():find("angler") or model.Name:lower():find("blind")) then
                        return model, "Đạo Sĩ (Taoist)", "Taoist", Colors.AccentOrange, "📜"
                    end
                end
            elseif d:IsA("TextLabel") and d.Visible and d.Text and #d.Text > 0 then
                if isTaoistText(d.Text) then
                    local model = d:FindFirstAncestorOfClass("Model") or d.Parent
                    if model and not (model.Name:lower():find("angler") or model.Name:lower():find("blind")) then
                        return model, "Đạo Sĩ (Taoist)", "Taoist", Colors.AccentOrange, "📜"
                    end
                end
            end
        end
    end

    -- Đợt 3: Quét trực tiếp các Model cấp 1 của Workspace
    for _, n in ipairs(Workspace:GetChildren()) do
        if isNPCModel(n) and isTaoistText(n.Name) then
            return n, "Đạo Sĩ (Taoist)", "Taoist", Colors.AccentOrange, "📜"
        end
    end

    return nil
end

function secretBossState.ScanForMaoshanNPC()
    local candidateFolders = getCandidateNPCFolders()

    -- Đợt 1: Quét Model NPC trực tiếp trong các folder
    for _, folder in ipairs(candidateFolders) do
        for _, n in ipairs(folder:GetChildren()) do
            if isNPCModel(n) and isMaoshanText(n.Name) then
                return n, "Đạo Sĩ Maoshan", "Maoshan", Colors.PurplePrimary, "✨"
            end
        end
    end

    -- Đợt 2: Quét ProximityPrompt và TextLabel trong các thư mục NPC
    for _, folder in ipairs(candidateFolders) do
        for _, d in ipairs(folder:GetDescendants()) do
            if d:IsA("ProximityPrompt") then
                local act = tostring(d.ActionText or "")
                local obj = tostring(d.ObjectText or "")
                local pName = d.Parent and d.Parent.Name or ""
                if isMaoshanText(act) or isMaoshanText(obj) or isMaoshanText(pName) then
                    local model = d:FindFirstAncestorOfClass("Model") or d.Parent
                    if model and not (model.Name:lower():find("angler") or model.Name:lower():find("blind")) then
                        return model, "Đạo Sĩ Maoshan", "Maoshan", Colors.PurplePrimary, "✨"
                    end
                end
            elseif d:IsA("TextLabel") and d.Visible and d.Text and #d.Text > 0 then
                if isMaoshanText(d.Text) then
                    local model = d:FindFirstAncestorOfClass("Model") or d.Parent
                    if model and not (model.Name:lower():find("angler") or model.Name:lower():find("blind")) then
                        return model, "Đạo Sĩ Maoshan", "Maoshan", Colors.PurplePrimary, "✨"
                    end
                end
            end
        end
    end

    -- Đợt 3: Quét trực tiếp các Model cấp 1 của Workspace
    for _, n in ipairs(Workspace:GetChildren()) do
        if isNPCModel(n) and isMaoshanText(n.Name) then
            return n, "Đạo Sĩ Maoshan", "Maoshan", Colors.PurplePrimary, "✨"
        end
    end

    return nil
end

function secretBossState.GetTaoist()
    if secretBossState.cachedTaoist and secretBossState.cachedTaoist.inst and secretBossState.cachedTaoist.inst.Parent then
        return secretBossState.cachedTaoist.inst, secretBossState.cachedTaoist.displayName, "Taoist", Colors.AccentOrange, "📜"
    end
    local now = tick()
    if (now - (secretBossState.lastTaoistScan or 0)) >= 3.0 then
        secretBossState.lastTaoistScan = now
        local inst, displayName, category, col, icon = secretBossState.ScanForTaoistNPC()
        if inst then
            secretBossState.cachedTaoist = {
                inst = inst,
                displayName = displayName or "Đạo Sĩ (Taoist)"
            }
            return inst, displayName or "Đạo Sĩ (Taoist)", "Taoist", col or Colors.AccentOrange, icon or "📜"
        else
            secretBossState.cachedTaoist = nil
        end
    end
    if secretBossState.cachedTaoist and secretBossState.cachedTaoist.inst and secretBossState.cachedTaoist.inst.Parent then
        return secretBossState.cachedTaoist.inst, secretBossState.cachedTaoist.displayName, "Taoist", Colors.AccentOrange, "📜"
    end
    return nil
end

function secretBossState.GetMaoshan()
    if secretBossState.cachedMaoshan and secretBossState.cachedMaoshan.inst and secretBossState.cachedMaoshan.inst.Parent then
        return secretBossState.cachedMaoshan.inst, secretBossState.cachedMaoshan.displayName, "Maoshan", Colors.PurplePrimary, "✨"
    end
    local now = tick()
    if (now - (secretBossState.lastMaoshanScan or 0)) >= 3.0 then
        secretBossState.lastMaoshanScan = now
        local inst, displayName, category, col, icon = secretBossState.ScanForMaoshanNPC()
        if inst then
            secretBossState.cachedMaoshan = {
                inst = inst,
                displayName = displayName or "Đạo Sĩ Maoshan"
            }
            return inst, displayName or "Đạo Sĩ Maoshan", "Maoshan", col or Colors.PurplePrimary, icon or "✨"
        else
            secretBossState.cachedMaoshan = nil
        end
    end
    if secretBossState.cachedMaoshan and secretBossState.cachedMaoshan.inst and secretBossState.cachedMaoshan.inst.Parent then
        return secretBossState.cachedMaoshan.inst, secretBossState.cachedMaoshan.displayName, "Maoshan", Colors.PurplePrimary, "✨"
    end
    return nil
end

function secretBossState.ScanForGodSpirit()
    local godKeywords = {"spirit", "god spirit", "godspirit", "thần linh", "than linh"}
    local function matchesGod(str)
        if not str or typeof(str) ~= "string" or str == "" then return nil end
        local s = str:lower()
        for _, pat in ipairs(godKeywords) do
            if s:find(pat, 1, true) then return pat end
        end
        return nil
    end

    local candidateFolders = {}
    for _, fName in ipairs({"NPC", "NPCs", "Entities", "Characters"}) do
        local f = Workspace:FindFirstChild(fName)
        if f then table.insert(candidateFolders, f) end
    end

    for _, folder in ipairs(candidateFolders) do
        for _, n in ipairs(folder:GetChildren()) do
            if (n:IsA("Model") or n:IsA("BasePart")) and matchesGod(n.Name) then
                return n
            end
        end
    end

    for _, folder in ipairs(candidateFolders) do
        for _, d in ipairs(folder:GetDescendants()) do
            if d:IsA("ProximityPrompt") then
                local pat = matchesGod(d.ActionText) or matchesGod(d.ObjectText) or matchesGod(d.Parent and d.Parent.Name)
                if pat then
                    return d:FindFirstAncestorOfClass("Model") or d.Parent
                end
            elseif d:IsA("TextLabel") and d.Visible and d.Text and #d.Text > 0 then
                if matchesGod(d.Text) then
                    return d:FindFirstAncestorOfClass("Model") or d.Parent
                end
            end
        end
    end

    -- Chỉ kiểm tra con cấp 1 của Workspace
    for _, n in ipairs(Workspace:GetChildren()) do
        if (n:IsA("Model") or n:IsA("BasePart")) and matchesGod(n.Name) then
            return n
        end
    end
    return nil
end

function secretBossState.GetGodSpirit()
    if secretBossState.cachedGod and secretBossState.cachedGod.Parent then
        return secretBossState.cachedGod
    end
    local now = tick()
    if (now - (secretBossState.lastGodScan or 0)) >= 5.0 then
        secretBossState.lastGodScan = now
        local inst = secretBossState.ScanForGodSpirit()
        secretBossState.cachedGod = inst
        return inst
    end
    return secretBossState.cachedGod
end

function secretBossState.HandleChatMessage(msg)
    if not (Config.AutoChatSecretBoss or Config.AutoHuntBoss) then return end
    if typeof(msg) ~= "string" or #msg == 0 then return end
    local lower = msg:lower()

    -- Bỏ qua nếu là thông báo người chơi câu được cá (KHÔNG PHẢI BOSS XUẤT HIỆN)
    if lower:find("caught") or lower:find("has caught") or lower:find("câu được") or lower:find("đã câu") or lower:find("bắt được") or lower:find("obtained") then
        return
    end

    -- 1. Check for Despawn Announcement: "all secret bosses have been despawned", "weather ended", etc.
    if (lower:find("all secret bosses") and (lower:find("despawn") or lower:find("gone") or lower:find("disappear")))
       or lower:find("secret bosses have been despawned")
       or lower:find("secret bosses have despawned")
       or lower:find("bosses have despawned")
       or lower:find("boss has despawned")
       or lower:find("weather has ended")
       or lower:find("weather ended")
       or lower:find("weather returned to normal")
       or lower:find("clear skies")
       or lower:find("weather: clear")
       or lower:find("thời tiết đã hết")
       or lower:find("kết thúc") then
        secretBossState.active = false
        secretBossState.currentMap = nil
        secretBossState.targetIsland = nil
        secretBossState.standPos = nil
        secretBossState.activeChatBoss = nil
        secretBossState.statusText = "Tất cả Secret Boss đã despawn. Chờ đợt mới..."
        ShowNotification("Secret Boss Despawn", "Tất cả Secret Boss đã biến mất! Hệ thống đang chờ đợt xuất hiện tiếp theo.", "WARN", 7)
        if statusLabelSecretBoss and statusLabelSecretBoss.Set then
            statusLabelSecretBoss.Set("Tất cả Secret Boss đã despawn. Chờ đợt mới...")
        end
        if Config.ReturnToHomeWhenClear and Config.HomeFarmSpot then
            secretBossState.ReturnToHome()
        end
        if Config.AutoServerHopOnDespawn then
            ShowNotification("Auto Server Hop", "Secret Boss đã hết! Đang tự động đổi server để săn tiếp...", "WARN", 5)
            task.delay(1.5, function()
                if Config.AutoServerHopOnDespawn then
                    ServerHop()
                end
            end)
        end
        return
    end

    -- 2. Check for Spawn Announcement: Chứa từ khóa xuất hiện thực sự
    local isSpawnWord = lower:find("spawn") or lower:find("appear") or lower:find("xuất hiện")
        or lower:find("started") or lower:find("bắt đầu") or lower:find("active")
        or lower:find("has arrived") or lower:find("đã đến") or lower:find("secret boss")

    if isSpawnWord then
        local matchedIsland, detectedName = secretBossState.DetectIsland(msg)
        if matchedIsland and detectedName ~= "Clear" then
            -- Kiểm tra xem có boss mục tiêu nào trên đảo này được bật trong cài đặt săn không
            local hasTargetInIsland = false
            for _, b in ipairs(matchedIsland.bosses) do
                if Config.SecretBossTargets[b.name] then
                    hasTargetInIsland = true
                    break
                end
            end
            if not hasTargetInIsland then return end

            local reqPower = 0
            local powMatch = lower:match("power%s*[:=]?%s*(%d+)") or lower:match("(%d+)%s*power")
            if powMatch then
                reqPower = tonumber(powMatch) or 0
            end

            secretBossState.activeChatBoss = {
                island = matchedIsland,
                bossName = detectedName,
                time = tick(),
                reqPower = reqPower
            }

            -- KIỂM TRA PHÂN CẤP ĐỘ ƯU TIÊN (PRIORITY MANAGER)
            if Config.PrioritySystemEnabled and PriorityManager and PriorityManager.GetActiveTask then
                local curTask = PriorityManager.GetActiveTask()
                local bossPri = PriorityManager.GetTaskPriority("SecretBoss")
                if curTask ~= "None" and curTask ~= "SecretBoss" and PriorityManager.GetTaskPriority(curTask) < bossPri then
                    ShowNotification("Ưu Tiên", string.format("Phát hiện %s tại %s nhưng đang ưu tiên [%s] (Hạng %d)!", tostring(detectedName), matchedIsland.islandName, PriorityManager.GetTaskDisplayName(curTask), PriorityManager.GetTaskPriority(curTask)), "WARN", 5)
                    return
                end
            end

            secretBossState.Teleport(matchedIsland, detectedName, reqPower)
        end
    end
end

function secretBossState.ScanChatHistory()
    local foundMessages = {}
    local pg = LocalPlayer:FindFirstChild("PlayerGui")

    -- 1. Quét giao diện Chat hiện đại (ExperienceChat) - Chỉ lấy tối đa 10 tin gần nhất
    if pg and pg:FindFirstChild("ExperienceChat") then
        local labels = {}
        for _, d in ipairs(pg.ExperienceChat:GetDescendants()) do
            if d:IsA("TextLabel") and d.Text ~= "" and #d.Text > 5 then
                table.insert(labels, d)
            end
        end
        table.sort(labels, function(a, b)
            local orderA = (a.Parent and a.Parent.LayoutOrder) or a.LayoutOrder or 0
            local orderB = (b.Parent and b.Parent.LayoutOrder) or b.LayoutOrder or 0
            if orderA ~= orderB then return orderA < orderB end
            return a.AbsolutePosition.Y < b.AbsolutePosition.Y
        end)
        local startIdx = math.max(1, #labels - 9)
        for i = startIdx, #labels do
            table.insert(foundMessages, labels[i].Text)
        end
    end

    -- 2. Quét giao diện Chat cổ điển (Legacy Chat) - Chỉ lấy tối đa 10 tin gần nhất
    if pg and pg:FindFirstChild("Chat") then
        local labels = {}
        for _, d in ipairs(pg.Chat:GetDescendants()) do
            if d:IsA("TextLabel") and d.Text ~= "" and #d.Text > 5 then
                table.insert(labels, d)
            end
        end
        table.sort(labels, function(a, b)
            local orderA = (a.Parent and a.Parent.LayoutOrder) or a.LayoutOrder or 0
            local orderB = (b.Parent and b.Parent.LayoutOrder) or b.LayoutOrder or 0
            if orderA ~= orderB then return orderA < orderB end
            return a.AbsolutePosition.Y < b.AbsolutePosition.Y
        end)
        local startIdx = math.max(1, #labels - 9)
        for i = startIdx, #labels do
            table.insert(foundMessages, labels[i].Text)
        end
    end

    -- TUYỆT ĐỐI KHÔNG quét PlayerGui / Workspace.BossSetUp ở đây để tránh nhầm UI game thành tin thông báo!

    -- Phân tích tin nhắn theo thứ tự từ dưới lên trên (tin mới nhất xuất hiện ở cuối danh sách)
    local latestSpawn = nil
    local latestDespawnIndex = -1
    local latestSpawnIndex = -1

    for idx, msg in ipairs(foundMessages) do
        local lower = msg:lower()
        if (lower:find("all secret bosses") and (lower:find("despawn") or lower:find("gone") or lower:find("disappear")))
           or lower:find("secret bosses have been despawned")
           or lower:find("secret bosses have despawned")
           or lower:find("bosses have despawned")
           or lower:find("boss has despawned")
           or lower:find("weather has ended")
           or lower:find("weather ended")
           or lower:find("weather returned to normal")
           or lower:find("clear skies")
           or lower:find("weather: clear")
           or lower:find("thời tiết đã hết")
           or lower:find("kết thúc") then
            latestDespawnIndex = idx
        elseif not (lower:find("caught") or lower:find("has caught") or lower:find("câu được") or lower:find("đã câu") or lower:find("bắt được")) then
            local isSpawnKw = lower:find("spawn") or lower:find("appear") or lower:find("xuất hiện")
                or lower:find("started") or lower:find("bắt đầu") or lower:find("secret boss")
            if isSpawnKw then
                local entry, bName = secretBossState.DetectIsland(msg)
                if entry and bName ~= "Clear" then
                    latestSpawn = {msg = msg, island = entry, index = idx, bossName = bName}
                    latestSpawnIndex = idx
                end
            end
        end
    end

    -- Nếu có tin nhắn Boss xuất hiện và tin Spawn xuất hiện sau tin Despawn (hoặc không có tin despawn nào sau đó)
    if latestSpawn and (latestDespawnIndex < latestSpawnIndex) then
        local hasTargetInIsland = false
        for _, b in ipairs(latestSpawn.island.bosses) do
            if Config.SecretBossTargets[b.name] then
                hasTargetInIsland = true
                break
            end
        end
        if hasTargetInIsland then
            return latestSpawn
        end
    elseif latestDespawnIndex > latestSpawnIndex and latestDespawnIndex ~= -1 then
        secretBossState.standPos = nil
        secretBossState.active = false
        secretBossState.currentMap = nil
        secretBossState.activeChatBoss = nil
        secretBossState.statusText = "Boss gần nhất đã despawn. Đang chờ đợt mới..."
        if statusLabelSecretBoss and statusLabelSecretBoss.Set then
            statusLabelSecretBoss.Set(secretBossState.statusText)
        end
        if Config.ReturnToHomeWhenClear and Config.HomeFarmSpot then
            secretBossState.ReturnToHome()
        end
    end
    return nil
end

pcall(function() secretBossState.LoadCustomSpots() end)
pcall(function() secretBossState.LoadHomeSpot() end)

local function TriggerPrompt(prompt)
    if not prompt then return end
    if fireproximityprompt then
        fireproximityprompt(prompt)
    else
        pcall(function()
            prompt:InputHoldBegin()
            task.wait((prompt.HoldDuration or 0) + 0.05)
            prompt:InputHoldEnd()
        end)
    end
end

local function ParseFishWeightNumber(val)
    if not val then return 0 end
    if type(val) == "number" then return val end
    local str = tostring(val):gsub(",", ""):upper()
    local num = tonumber(str:match("[%d%.]+")) or 0
    if str:find("B") then
        num = num * 1000000000
    elseif str:find("M") then
        num = num * 1000000
    elseif str:find("K") then
        num = num * 1000
    end
    return num
end

for k, v in pairs({
    active = false,
    currentQuestType = "none", -- "fish_15m", "bait_100", "skill_100", "fish_100", "none"
    currentQuestTitle = "Chưa nhận nhiệm vụ",
    currentProgress = 0,
    targetProgress = 100,
    isCompleted = false,
    isCooldown = false,
    cooldownEnd = 0,
    readyForNewQuest = false, -- Flag: cooldown đã hết, đang chờ đến NPC nhận quest mới
    isAcceptingQuest = false, -- Flag: đang trong quá trình tương tác NPC nhận quest (chặn ScanStatus set isCooldown sai)
    lastAcceptTime = 0,
    lastClaimTime = 0,
    lastNpcInteract = 0,
    lastSyncTime = 0,
    lastBaitBuy = 0,
    lastBaitEquip = 0,
    isBusyRoutine = false,
    isAtHomeSpot = false,
    statusText = "Đang quét nhiệm vụ...",
    
    -- Vị trí mặc định
    spot100Fish = Vector3.new(-96, 9, 234), -- Map 1 (100 con cá)
    spot100Bait = Vector3.new(-96, 9, 231), -- Map 1 (100 mồi)
    spot100Skill = Vector3.new(-96, 9, 234), -- Map 1 (100 skill)
    spot15MFish = Vector3.new(1619, 13, 334), -- Map 9 (1.5M cá)
    spotNPC = Vector3.new(-200.7, 11.1, 35.9), -- Map 1 NPC Ticket Quest
    
    -- UI rows
    uiStatus = nil,
    uiProgress = nil,
    uiCooldown = nil,
    ui100Spot = nil,
    ui100BaitSpot = nil,
    ui100SkillSpot = nil,
    ui15MSpot = nil,
    uiNPCSpot = nil,
}) do
    ticketQuestState[k] = v
end

function ticketQuestState.SerializeSpot(spot)
    if not spot then return nil end
    if typeof(spot) == "CFrame" then
        return {
            x = spot.Position.X,
            y = spot.Position.Y,
            z = spot.Position.Z,
            cframe = {spot:GetComponents()}
        }
    elseif typeof(spot) == "Vector3" then
        return {
            x = spot.X,
            y = spot.Y,
            z = spot.Z
        }
    elseif type(spot) == "table" and (spot.cframe or spot.x) then
        return spot
    end
    return nil
end

function ticketQuestState.DeserializeSpot(data, defaultCf)
    if not data then return defaultCf end
    if typeof(data) == "CFrame" then return data end
    if typeof(data) == "Vector3" then return CFrame.new(data) end
    if type(data) == "table" then
        if data.cframe and #data.cframe == 12 then
            return CFrame.new(table.unpack(data.cframe))
        elseif data.x and data.y and data.z then
            return CFrame.new(data.x, data.y, data.z)
        end
    end
    return defaultCf
end

function ticketQuestState.SaveSpots()
    if not writefile then return end
    pcall(function()
        local data = {
            spot100Fish = ticketQuestState.SerializeSpot(ticketQuestState.spot100Fish),
            spot100Bait = ticketQuestState.SerializeSpot(ticketQuestState.spot100Bait),
            spot100Skill = ticketQuestState.SerializeSpot(ticketQuestState.spot100Skill),
            spot15MFish = ticketQuestState.SerializeSpot(ticketQuestState.spot15MFish),
            spotNPC = ticketQuestState.SerializeSpot(ticketQuestState.spotNPC),
            cooldownEnd = ticketQuestState.cooldownEnd,
        }
        writefile("heavyweight_ticket_spots.json", HttpService:JSONEncode(data))
    end)
end

function ticketQuestState.LoadSpots()
    if not readfile or not isfile or not isfile("heavyweight_ticket_spots.json") then return end
    pcall(function()
        local content = readfile("heavyweight_ticket_spots.json")
        if content and #content > 0 then
            local dec = HttpService:JSONDecode(content)
            if type(dec) == "table" then
                if dec.spot100Fish then
                    ticketQuestState.spot100Fish = ticketQuestState.DeserializeSpot(dec.spot100Fish, ticketQuestState.spot100Fish)
                end
                if dec.spot100Bait then
                    ticketQuestState.spot100Bait = ticketQuestState.DeserializeSpot(dec.spot100Bait, ticketQuestState.spot100Bait)
                end
                if dec.spot100Skill then
                    ticketQuestState.spot100Skill = ticketQuestState.DeserializeSpot(dec.spot100Skill, ticketQuestState.spot100Skill)
                end
                if dec.spot15MFish then
                    ticketQuestState.spot15MFish = ticketQuestState.DeserializeSpot(dec.spot15MFish, ticketQuestState.spot15MFish)
                end
                if dec.spotNPC then
                    ticketQuestState.spotNPC = ticketQuestState.DeserializeSpot(dec.spotNPC, ticketQuestState.spotNPC)
                end
                if dec.cooldownEnd and tonumber(dec.cooldownEnd) and dec.cooldownEnd > tick() then
                    ticketQuestState.cooldownEnd = dec.cooldownEnd
                    ticketQuestState.isCooldown = true
                end
            end
        end
    end)
end

function ticketQuestState.UpdateUI()
    if ticketQuestState.uiStatus and ticketQuestState.uiStatus.Set then
        ticketQuestState.uiStatus.Set(ticketQuestState.statusText or "Đang chạy...")
    end
    if ticketQuestState.uiProgress and ticketQuestState.uiProgress.Set then
        local maxVal = ticketQuestState.targetProgress > 0 and ticketQuestState.targetProgress or 100
        local pct = math.min(100, math.floor((ticketQuestState.currentProgress / maxVal) * 100))
        ticketQuestState.uiProgress.Set(string.format("%d / %d (%d%%)", ticketQuestState.currentProgress, maxVal, pct))
    end
    if ticketQuestState.uiCooldown and ticketQuestState.uiCooldown.Set then
        local pData = ticketQuestState.GetPlayerDataFolder()
        local cdVal = pData and pData:FindFirstChild("TicketQuestCooldown")
        local serverCd = cdVal and tonumber(cdVal.Value) or 0
        local nowServer = ticketQuestState.GetServerTimeNow()
        local serverRemain = serverCd > nowServer and (serverCd - nowServer) or 0

        local hasQuest = (ticketQuestState.currentQuestType and ticketQuestState.currentQuestType ~= "none")
        -- CHỈ set isCooldown = true nếu chưa sẵn sàng nhận quest mới (readyForNewQuest = false)
        if not hasQuest and serverRemain > 0 and not ticketQuestState.readyForNewQuest then
            ticketQuestState.isCooldown = true
            ticketQuestState.cooldownEnd = tick() + serverRemain
        end

        local remain = serverRemain
        if remain <= 0 and ticketQuestState.isCooldown and ticketQuestState.cooldownEnd and ticketQuestState.cooldownEnd > tick() then
            remain = math.max(0, math.floor(ticketQuestState.cooldownEnd - tick()))
        end

        if ticketQuestState.IsAllQuestsDoneToday and ticketQuestState.IsAllQuestsDoneToday() then
            ticketQuestState.uiCooldown.Set("Đã hết nhiệm vụ hôm nay!")
        elseif remain > 0 and not ticketQuestState.readyForNewQuest then
            local mins = math.floor(remain / 60)
            local secs = remain % 60
            local homeStr = ""
            if ticketQuestState.isAtHomeSpot then
                homeStr = " (Farm Home)"
            end
            ticketQuestState.uiCooldown.Set(string.format("Còn %02d:%02d%s", mins, secs, homeStr))
        else
            if not hasQuest then ticketQuestState.isCooldown = false end
            ticketQuestState.uiCooldown.Set("Sẵn sàng nhận vé!")
        end
    end
    local function getSpotPos(p)
        if not p then return Vector3.zero end
        if typeof(p) == "CFrame" then return p.Position end
        if typeof(p) == "Vector3" then return p end
        if type(p) == "table" and p.x then return Vector3.new(p.x, p.y, p.z) end
        return Vector3.zero
    end
    if ticketQuestState.ui100Spot and ticketQuestState.ui100Spot.Set then
        local p = getSpotPos(ticketQuestState.spot100Fish)
        ticketQuestState.ui100Spot.Set(string.format("(%.0f, %.0f, %.0f)", p.X, p.Y, p.Z))
    end
    if ticketQuestState.ui100BaitSpot and ticketQuestState.ui100BaitSpot.Set then
        local p = getSpotPos(ticketQuestState.spot100Bait)
        ticketQuestState.ui100BaitSpot.Set(string.format("(%.0f, %.0f, %.0f)", p.X, p.Y, p.Z))
    end
    if ticketQuestState.ui100SkillSpot and ticketQuestState.ui100SkillSpot.Set then
        local p = getSpotPos(ticketQuestState.spot100Skill)
        ticketQuestState.ui100SkillSpot.Set(string.format("(%.0f, %.0f, %.0f)", p.X, p.Y, p.Z))
    end
    if ticketQuestState.ui15MSpot and ticketQuestState.ui15MSpot.Set then
        local p = getSpotPos(ticketQuestState.spot15MFish)
        ticketQuestState.ui15MSpot.Set(string.format("(%.0f, %.0f, %.0f)", p.X, p.Y, p.Z))
    end
    if ticketQuestState.uiNPCSpot and ticketQuestState.uiNPCSpot.Set then
        local p = getSpotPos(ticketQuestState.spotNPC)
        ticketQuestState.uiNPCSpot.Set(string.format("(%.0f, %.0f, %.0f)", p.X, p.Y, p.Z))
    end
end

ticketQuestState.cachedNPCModel = nil
ticketQuestState.cachedNPCPos = nil
ticketQuestState.cachedNPCCFrame = nil
ticketQuestState.cachedNPCPrompt = nil
ticketQuestState.lastNPCSearch = 0

-- Hàm tiện ích lấy folder dữ liệu của LocalPlayer từ ReplicatedStorage.Data (Chính xác 100% từ cấu trúc game)
function ticketQuestState.GetPlayerDataFolder()
    local d = ReplicatedStorage:FindFirstChild("Data")
    if not d then return nil end
    local uid = tostring(LocalPlayer.UserId)
    return d:FindFirstChild(uid) or d:FindFirstChild(LocalPlayer.UserId)
end

function ticketQuestState.FindTicketNPC()
    if ticketQuestState.cachedNPCModel and ticketQuestState.cachedNPCModel.Parent and ticketQuestState.cachedNPCPos then
        return ticketQuestState.cachedNPCModel, ticketQuestState.cachedNPCPos, ticketQuestState.cachedNPCPrompt, ticketQuestState.cachedNPCCFrame
    end

    local now = tick()
    if now - (ticketQuestState.lastNPCSearch or 0) < 5.0 and ticketQuestState.cachedNPCPos then
        return ticketQuestState.cachedNPCModel, ticketQuestState.cachedNPCPos, ticketQuestState.cachedNPCPrompt, ticketQuestState.cachedNPCCFrame
    end
    ticketQuestState.lastNPCSearch = now

    local function extractModelData(model)
        if not model then return nil, nil, nil end
        local p = model:FindFirstChildWhichIsA("ProximityPrompt", true)
        local hrp = (p and p.Parent:IsA("BasePart") and p.Parent)
            or model:FindFirstChild("HumanoidRootPart")
            or model:FindFirstChild("Torso")
            or model:FindFirstChild("UpperTorso")
            or model.PrimaryPart
            or model:FindFirstChildWhichIsA("BasePart")
        local cf = (hrp and hrp.CFrame) or (model:IsA("Model") and model:GetPivot()) or model.CFrame
        return cf, p, hrp
    end

    -- 1. Ưu tiên tìm đúng cấu trúc từ Explorer export: Workspace.NPC.Function["Ticket Quest Giver"]
    local directModel = nil
    if Workspace:FindFirstChild("NPC") then
        local funcFolder = Workspace.NPC:FindFirstChild("Function")
        if funcFolder then
            directModel = funcFolder:FindFirstChild("Ticket Quest Giver")
        end
        if not directModel then
            for _, ch in ipairs(Workspace.NPC:GetChildren()) do
                local n = ch.Name:lower()
                if ch:IsA("Model") and (n:find("ticket") or n:find("giver")) then
                    directModel = ch
                    break
                elseif ch:IsA("Folder") then
                    for _, sub in ipairs(ch:GetChildren()) do
                        local sn = sub.Name:lower()
                        if sub:IsA("Model") and (sn:find("ticket") or sn:find("giver")) then
                            directModel = sub
                            break
                        end
                    end
                end
                if directModel then break end
            end
        end
    end

    if directModel then
        local cf, p = extractModelData(directModel)
        if cf then
            ticketQuestState.cachedNPCModel = directModel
            ticketQuestState.cachedNPCPos = cf.Position
            ticketQuestState.cachedNPCCFrame = cf
            ticketQuestState.cachedNPCPrompt = p
            ticketQuestState.spotNPC = cf.Position
            return directModel, cf.Position, p, cf
        end
    end

    -- 2. Tìm trong các folder NPC / Spawns / Entities
    for _, fName in ipairs({"NPC", "NPCs", "Entities", "Characters", "Spawns"}) do
        local folder = Workspace:FindFirstChild(fName)
        if folder then
            for _, inst in ipairs(folder:GetChildren()) do
                local n = inst.Name:lower()
                if n:find("ticket") or n:find("giver") then
                    local cf, p = extractModelData(inst)
                    if cf then
                        ticketQuestState.cachedNPCModel = inst
                        ticketQuestState.cachedNPCPos = cf.Position
                        ticketQuestState.cachedNPCCFrame = cf
                        ticketQuestState.cachedNPCPrompt = p
                        ticketQuestState.spotNPC = cf.Position
                        return inst, cf.Position, p, cf
                    end
                end
            end
        end
    end

    -- 3. Tìm trong direct children của Workspace
    for _, inst in ipairs(Workspace:GetChildren()) do
        if inst:IsA("Model") then
            local n = inst.Name:lower()
            if n:find("ticket") and (n:find("quest") or n:find("giver") or n:find("npc")) then
                local cf, p = extractModelData(inst)
                if cf then
                    ticketQuestState.cachedNPCModel = inst
                    ticketQuestState.cachedNPCPos = cf.Position
                    ticketQuestState.cachedNPCCFrame = cf
                    ticketQuestState.cachedNPCPrompt = p
                    ticketQuestState.spotNPC = cf.Position
                    return inst, cf.Position, p, cf
                end
            end
        end
    end

    return nil, ticketQuestState.spotNPC, nil, nil
end

function ticketQuestState.CheckNPCReady()
    -- 1. Ưu tiên kiểm tra mốc hồi chiêu từ ReplicatedStorage.Data[UserId].TicketQuestCooldown
    local pData = ticketQuestState.GetPlayerDataFolder()
    if pData and pData:FindFirstChild("TicketQuestCooldown") then
        local cd = tonumber(pData.TicketQuestCooldown.Value) or 0
        local nowServer = ticketQuestState.GetServerTimeNow()
        if cd > 0 then
            return cd <= nowServer
        end
    end

    -- 2. Kiểm tra bộ đếm Cooldown nội bộ của script
    if ticketQuestState.isCooldown then
        return tick() >= (ticketQuestState.cooldownEnd or 0)
    end

    return true
end

function ticketQuestState.TeleportTo(target)
    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if not root or not target then return end

    local targetCf = nil
    if typeof(target) == "CFrame" then
        targetCf = target
    elseif typeof(target) == "Vector3" then
        targetCf = CFrame.new(target + Vector3.new(0, 1.5, 0))
    elseif type(target) == "table" then
        if target.cframe and #target.cframe == 12 then
            targetCf = CFrame.new(table.unpack(target.cframe))
        elseif target.x and target.y and target.z then
            targetCf = CFrame.new(target.x, target.y + 1.5, target.z)
        end
    end

    if targetCf then
        local wp = Workspace:FindFirstChild("IdenticalWaterPlatform") or Workspace:FindFirstChild("WaterPlatform")
        if not wp then
            wp = Instance.new("Part")
            wp.Name = "IdenticalWaterPlatform"
            wp.Size = Vector3.new(30, 2, 30)
            wp.Transparency = 1
            wp.Anchored = true
            wp.CanCollide = true
            wp.Parent = Workspace
        end
        wp.CFrame = CFrame.new(targetCf.Position.X, targetCf.Position.Y - 2.8, targetCf.Position.Z)
        wp.CanCollide = true

        root.AssemblyLinearVelocity = Vector3.zero
        root.CFrame = targetCf + Vector3.new(0, 1.5, 0)
        task.wait(0.12)
        root.CFrame = targetCf
        task.wait(0.12)
    end
end

function ticketQuestState.OrientCameraTo(focusPos, standPos)
    local cam = Workspace.CurrentCamera or Camera
    if not cam or not focusPos then return end
    pcall(function()
        local dir = standPos and (standPos - focusPos) or -cam.CFrame.LookVector
        local dir2D = Vector3.new(dir.X, 0, dir.Z)
        if dir2D.Magnitude < 0.1 then
            dir2D = Vector3.new(0, 0, 1)
        else
            dir2D = dir2D.Unit
        end
        local refPos = standPos or (focusPos + dir2D * 3.0)
        -- Đặt camera ở phía sau lưng nhân vật 4.5 studs, cao 2.2 studs, nhìn trực diện vào ngực/đầu NPC
        local camPos = refPos + (dir2D * 4.5) + Vector3.new(0, 2.2, 0)
        local camFocus = focusPos + Vector3.new(0, 0.8, 0)
        cam.CameraType = Enum.CameraType.Custom
        cam.CFrame = CFrame.lookAt(camPos, camFocus)
        cam.Focus = CFrame.new(camFocus)
    end)
end

function ticketQuestState.TeleportToNPC()
    local npcModel, npcPos, prompt, npcCFrame = ticketQuestState.FindTicketNPC()
    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    if hum then
        pcall(function()
            hum.Sit = false
            hum:UnequipTools()
        end)
    end
    if Events and Events:FindFirstChild("CancelCast") then
        pcall(function() Events.CancelCast:FireServer() end)
    end

    local promptFound = prompt
    if not promptFound and npcModel then
        promptFound = npcModel:FindFirstChildWhichIsA("ProximityPrompt", true)
    end

    local focusPos, standPos, targetCF = nil, nil, nil

    if npcCFrame then
        focusPos = npcCFrame.Position
        local fwd = npcCFrame.LookVector
        local fwd2D = Vector3.new(fwd.X, 0, fwd.Z)
        if fwd2D.Magnitude > 0.05 then
            fwd2D = fwd2D.Unit
        else
            fwd2D = Vector3.new(0, 0, 1)
        end
        standPos = focusPos + (fwd2D * 2.8)
        standPos = Vector3.new(standPos.X, focusPos.Y, standPos.Z)
        targetCF = CFrame.lookAt(standPos, Vector3.new(focusPos.X, standPos.Y, focusPos.Z))
    elseif typeof(ticketQuestState.spotNPC) == "CFrame" then
        targetCF = ticketQuestState.spotNPC
        standPos = targetCF.Position
        focusPos = standPos + (targetCF.LookVector * 2.8)
    elseif npcPos or ticketQuestState.spotNPC then
        local rawPos = npcPos or (typeof(ticketQuestState.spotNPC) == "Vector3" and ticketQuestState.spotNPC) or (ticketQuestState.spotNPC and ticketQuestState.spotNPC.Position)
        if rawPos then
            focusPos = rawPos
            standPos = focusPos + Vector3.new(0, 0, 2.8)
            targetCF = CFrame.lookAt(standPos, focusPos)
        end
    end

    if root and targetCF and standPos then
        local dist = (root.Position - standPos).Magnitude
        if dist > 3.0 then
            ticketQuestState.TeleportTo(targetCF)
            task.wait(0.25)
        else
            root.AssemblyLinearVelocity = Vector3.zero
            root.CFrame = targetCF
            task.wait(0.08)
        end

        -- Căn chỉnh góc nhìn Camera nhìn thẳng vào NPC và bảng Prompt
        ticketQuestState.OrientCameraTo(focusPos, standPos)
    end

    if promptFound then
        pcall(function()
            promptFound.RequiresLineOfSight = false
            promptFound.MaxActivationDistance = math.max(promptFound.MaxActivationDistance or 10, 35)
            promptFound.Enabled = true
        end)
    end

    return npcModel, focusPos or npcPos, promptFound, standPos
end

-- Hàm lấy Frame hội thoại của game (PlayerGui.MainGui.Menu.Dialogue)
function ticketQuestState.GetDialogueGui()
    local pg = LocalPlayer:FindFirstChild("PlayerGui")
    if not pg then return nil end
    local mg = pg:FindFirstChild("MainGui")
    if mg and mg:FindFirstChild("Menu") and mg.Menu:FindFirstChild("Dialogue") then
        return mg.Menu.Dialogue
    end
    for _, g in ipairs(pg:GetChildren()) do
        if g:IsA("ScreenGui") then
            local d = g:FindFirstChild("Dialogue", true)
            if d then return d end
        end
    end
    return nil
end

function ticketQuestState.IsDialogueOpen()
    local dlg = ticketQuestState.GetDialogueGui()
    if dlg and dlg.Visible == true then
        return true
    end
    local pg = LocalPlayer:FindFirstChild("PlayerGui")
    local mg = pg and pg:FindFirstChild("MainGui")
    local d = mg and mg:FindFirstChild("Menu") and mg.Menu:FindFirstChild("Dialogue")
    if d and d.Visible == true then
        return true
    end
    return false
end

-- Lấy danh sách các nút tùy chọn hiện có trong PlayerGui.MainGui.Menu.Dialogue.ButtonFrame
function ticketQuestState.GetDialogueButtons()
    local dlg = ticketQuestState.GetDialogueGui()
    if not dlg then return {} end
    local bf = dlg:FindFirstChild("ButtonFrame")
    if not bf then return {} end

    local buttons = {}
    for _, ch in ipairs(bf:GetChildren()) do
        if ch:IsA("TextButton") and ch.Name ~= "Template" and ch.Visible then
            local titleObj = ch:FindFirstChild("Title") or ch:FindFirstChildWhichIsA("TextLabel", true)
            local text = (titleObj and titleObj.Text) or ch.Text or ""
            local clean = text:lower():gsub("%*", ""):gsub("%s+", " "):match("^%s*(.-)%s*$") or ""
            local idx = tonumber(ch.Name) or 0
            table.insert(buttons, {
                button = ch,
                index = idx,
                name = ch.Name,
                text = text,
                clean = clean
            })
        end
    end
    table.sort(buttons, function(a, b) return a.index < b.index end)
    return buttons
end

-- Hủy triệt để trạng thái UI Navigation / Hộp chọn ô vuông của Roblox và trả lại quyền điều khiển phím cho nhân vật
function ticketQuestState.ClearUINavigation()
    pcall(function()
        local gs = game:GetService("GuiService")
        if gs then
            gs.SelectedObject = nil
            gs.GuiNavigationEnabled = false
        end
    end)
end

-- Kích hoạt nút lựa chọn thoại bằng mọi phương thức UI lẫn RemoteEvent tương ứng
function ticketQuestState.ClickButtonEntry(entry, explicitActionId)
    if not entry or not entry.button then return false end
    local btn = entry.button
    local idx = entry.index
    local clean = entry.clean or ""
    local rawTxt = entry.text or ""
    local actionId = explicitActionId

    if not actionId then
        if clean == "quest" then
            actionId = "Quest"
        elseif clean:find("hard") then
            actionId = "HardAcceptQuest"
        elseif clean:find("easy") then
            actionId = "EasyAcceptQuest"
        elseif clean:find("leave") or clean:find("close") then
            actionId = "Close"
        end
    end

    -- 1. Ưu tiên gọi hàm xử lý của chính game nếu ModuleScript Dialogue có xuất hàm
    pcall(function()
        local dm = require(ReplicatedStorage.ClientModule.Dialogue)
        if type(dm) == "table" then
            if dm.ChooseOption then dm.ChooseOption(idx)
            elseif dm.SelectOption then dm.SelectOption(idx)
            elseif dm.Choose then dm.Choose(idx)
            elseif dm.Select then dm.Select(idx)
            end
        end
    end)

    -- 2. Kích hoạt trực tiếp sự kiện UI của TextButton
    pcall(function()
        if firesignal then
            if btn:IsA("GuiButton") and btn.Activated then firesignal(btn.Activated) end
            if btn:IsA("GuiButton") and btn.MouseButton1Click then firesignal(btn.MouseButton1Click) end
        end
    end)

    pcall(function()
        if getconnections then
            local conns = getconnections(btn.Activated)
            if not conns or #conns == 0 then conns = getconnections(btn.MouseButton1Click) end
            if conns then
                for _, c in ipairs(conns) do
                    if c.Fire then c:Fire() elseif c.Function then pcall(c.Function) end
                end
            end
        end
    end)

    -- 3. Chọn đối tượng UI qua GuiService và bấm Enter để kích hoạt lựa chọn hội thoại
    pcall(function()
        local gs = game:GetService("GuiService")
        local vim = game:GetService("VirtualInputManager")
        if gs then
            gs.SelectedObject = btn
        end
        if vim then
            task.wait(0.04)
            vim:SendKeyEvent(true, Enum.KeyCode.Return, false, game)
            task.wait(0.06)
            vim:SendKeyEvent(false, Enum.KeyCode.Return, false, game)
        end
    end)

    -- 4. Giả lập click chuột và chạm màn hình (Mobile Touch) tại tâm nút
    pcall(function()
        local vim = game:GetService("VirtualInputManager")
        if vim and btn.AbsolutePosition and btn.AbsoluteSize then
            local p = btn.AbsolutePosition + btn.AbsoluteSize / 2
            vim:SendMouseButtonEvent(p.X, p.Y, 0, true, game, 0)
            task.wait(0.04)
            vim:SendMouseButtonEvent(p.X, p.Y, 0, false, game, 0)
            pcall(function()
                vim:SendTouchEvent(0, 0, p.X, p.Y)
                task.wait(0.04)
                vim:SendTouchEvent(0, 2, p.X, p.Y)
            end)
        end
    end)

    -- 5. Gửi RemoteEvent ChooseDialogueOption chuẩn xác lên Server
    if Events and Events:FindFirstChild("ChooseDialogueOption") then
        if actionId and #actionId > 0 then
            pcall(function() Events.ChooseDialogueOption:FireServer(actionId) end)
        end
        if idx and idx > 0 then
            pcall(function() Events.ChooseDialogueOption:FireServer(idx) end)
        end
    end

    if Events and Events:FindFirstChild("Dialogue") and Events.Dialogue:IsA("BindableEvent") then
        if idx and idx > 0 then
            pcall(function() Events.Dialogue:Fire(idx) end)
        end
    end

    return true
end

function ticketQuestState.CloseDialogue()
    if Events and Events:FindFirstChild("ChooseDialogueOption") then
        pcall(function() Events.ChooseDialogueOption:FireServer("Close") end)
    end
    pcall(function()
        local dlg = ticketQuestState.GetDialogueGui()
        if dlg then dlg.Visible = false end
        local pg = LocalPlayer:FindFirstChild("PlayerGui")
        local mg = pg and pg:FindFirstChild("MainGui")
        if mg and mg:FindFirstChild("Menu") then
            if mg.Menu:FindFirstChild("Dialogue") then
                mg.Menu.Dialogue.Visible = false
            end
            if mg.Menu:FindFirstChild("Inventory") then
                mg.Menu.Inventory.Visible = false
            end
            if mg.Menu:FindFirstChild("FishInventory") then
                mg.Menu.FishInventory.Visible = false
            end
        end
    end)
    ticketQuestState.ClearUINavigation()
end

-- Wrapper hỗ trợ tương thích ngược
function ticketQuestState.FindAndClickButton(predicate)
    local buttons = ticketQuestState.GetDialogueButtons()
    for _, b in ipairs(buttons) do
        if predicate(b.clean) then
            return ticketQuestState.ClickButtonEntry(b)
        end
    end
    return false
end

function ticketQuestState.SelectDialogueOption(optionIndex, optionTextPattern)
    local buttons = ticketQuestState.GetDialogueButtons()
    for _, b in ipairs(buttons) do
        if b.index == optionIndex or tostring(b.index) == tostring(optionIndex) then
            if not optionTextPattern or b.clean:find(optionTextPattern:lower()) then
                return ticketQuestState.ClickButtonEntry(b)
            end
        end
    end
    return false
end

-- Kiểm tra nếu NPC hiển thị câu thoại hết quest hôm nay:
-- "That's all the quests I've got for today! Come back tomorrow for more."
function ticketQuestState.CheckAllQuestsDoneToday()
    local dlg = ticketQuestState.GetDialogueGui()
    if not dlg then return false end
    for _, d in ipairs(dlg:GetDescendants()) do
        if d:IsA("TextLabel") and d.Visible and d.Text ~= "" then
            local low = d.Text:lower()
            if (low:find("all the quests") or low:find("all the quest") or low:find("come back tomorrow"))
               and (low:find("today") or low:find("tomorrow") or low:find("more")) then
                return true, d.Text
            end
        end
    end
    return false
end

function ticketQuestState.IsAllQuestsDoneToday()
    if not ticketQuestState.allQuestsDoneForToday then return false end
    local today = os.date("%Y-%m-%d")
    local utcToday = os.date("!%Y-%m-%d")
    local pData = ticketQuestState.GetPlayerDataFolder()
    local curDailyCount = pData and pData:FindFirstChild("TicketQuestDailyCount") and tonumber(pData.TicketQuestDailyCount.Value)

    local isDateChanged = (ticketQuestState.allQuestsDoneDate and ticketQuestState.allQuestsDoneDate ~= today)
        or (ticketQuestState.allQuestsDoneUtcDate and ticketQuestState.allQuestsDoneUtcDate ~= utcToday)
    local isServerDailyReset = (ticketQuestState.savedDailyCount and curDailyCount and curDailyCount < ticketQuestState.savedDailyCount)

    if isDateChanged or isServerDailyReset then
        -- Đã sang ngày mới (nửa đêm máy hoặc 00:00 UTC / Game Server reset) -> Tự động giải phóng cờ để tiếp tục nhận vé ngày mới!
        ticketQuestState.allQuestsDoneForToday = false
        ticketQuestState.allQuestsDoneDate = nil
        ticketQuestState.allQuestsDoneUtcDate = nil
        ticketQuestState.savedDailyCount = nil
        ticketQuestState.readyForNewQuest = true
        ticketQuestState.isCooldown = false
        ticketQuestState.isAtHomeSpot = false
        ticketQuestState.cooldownEnd = 0
        ticketQuestState.statusText = "Đã sang ngày mới! Chuẩn bị quay lại NPC nhận vé..."
        ticketQuestState.UpdateUI()
        ShowNotification("Ngày Mới - Reset Vé", "Đã sang ngày mới! Bot tự động quay lại NPC nhận vé.", "SUCCESS", 6)
        return false
    end
    return true
end

-- Xử lý toàn bộ logic tương tác các trang hội thoại của NPC Vé (Ticket Quest Giver)
-- Trang 1: "I love big and rare fish..." -> Luôn bấm nút [1] ("Quest", Action: "Quest")
-- Trang 2:
--   - Nếu nộp vé thành công: Hiện "YUPPIE..." -> Bấm nút [1] ("*Leave*", Action: "Close")
--   - Nếu nhận vé mới: Hiện "Oh, you are going to help me?..." -> Bấm nút [2] ("Accept Hard Quest", Action: "HardAcceptQuest") hoặc [1] ("Accept Easy Quest")
--   - Nếu chưa xong: Hiện "You didn't complete my task yet..." -> Bấm nút [1] ("*Leave*", Action: "Close")
--   - Nếu hết vé hôm nay: "That's all the quests I've got for today! Come back tomorrow for more." -> Bấm *Leave* và bật cờ hết vé!
function ticketQuestState.HandleDialogue(isClaiming)
    if not ticketQuestState.IsDialogueOpen() then return false end

    local isHard = (Config.TicketDifficulty or "Hard") == "Hard"
    local targetDiffKeyword = isHard and "hard" or "easy"

    -- Lấy danh sách nút hiện tại
    local buttons = ticketQuestState.GetDialogueButtons()
    if #buttons == 0 then
        task.wait(0.25)
        buttons = ticketQuestState.GetDialogueButtons()
    end
    if #buttons == 0 then return false end

    local questBtn = nil
    local leaveBtn = nil
    local diffBtn = nil

    for _, b in ipairs(buttons) do
        if b.clean == "quest" or (b.clean:find("quest") and not b.clean:find("accept") and not b.clean:find("nevermind")) then
            questBtn = b
        end
        if b.clean:find("leave") or b.clean:find("xong") or b.clean:find("close") then
            leaveBtn = b
        end
        if b.clean:find(targetDiffKeyword) and (b.clean:find("quest") or b.clean:find("accept")) then
            diffBtn = b
        end
    end

    -- KIỂM TRA ĐẶC BIỆT: Nếu NPC thông báo hết vé hôm nay ("That's all the quests I've got for today! Come back tomorrow for more.")
    local isDoneToday, doneMsg = ticketQuestState.CheckAllQuestsDoneToday()
    if isDoneToday then
        ticketQuestState.allQuestsDoneForToday = true
        ticketQuestState.allQuestsDoneDate = os.date("%Y-%m-%d")
        ticketQuestState.allQuestsDoneUtcDate = os.date("!%Y-%m-%d")
        local pData = ticketQuestState.GetPlayerDataFolder()
        if pData and pData:FindFirstChild("TicketQuestDailyCount") then
            ticketQuestState.savedDailyCount = tonumber(pData.TicketQuestDailyCount.Value) or 0
        end
        ticketQuestState.readyForNewQuest = false
        ticketQuestState.isCooldown = true
        ticketQuestState.statusText = "Đã hết nhiệm vụ hôm nay! (Hẹn ngày mai quay lại)"
        ticketQuestState.UpdateUI()
        ShowNotification("Hết Nhiệm Vụ", "NPC: Đã hết tất cả vé nhiệm vụ hôm nay! Hẹn gặp lại ngày mai.", "WARN", 8)
        if leaveBtn then
            ticketQuestState.ClickButtonEntry(leaveBtn, "Close")
        end
        task.wait(0.3)
        ticketQuestState.CloseDialogue()
        return true
    end

    -- TH1: Đang ở màn hình kết quả/rời đi có nút Leave / *Leave*
    if leaveBtn then
        ticketQuestState.ClickButtonEntry(leaveBtn, "Close")
        task.wait(0.3)
        ticketQuestState.CloseDialogue()
        return true
    end

    -- TH2: Đang ở màn hình chọn độ khó (Accept Hard / Easy Quest)
    if diffBtn then
        local action = isHard and "HardAcceptQuest" or "EasyAcceptQuest"
        ticketQuestState.ClickButtonEntry(diffBtn, action)
        task.wait(0.4)
        ticketQuestState.CloseDialogue()
        return true
    end

    -- TH3: Đang ở màn hình chào đầu tiên -> Bấm "Quest"
    if questBtn then
        ticketQuestState.ClickButtonEntry(questBtn, "Quest")

        -- Chờ màn hình thứ 2 cập nhật
        local t0 = tick()
        local retriedQuest = false
        while (tick() - t0) < 3.5 do
            task.wait(0.2)
            if not ticketQuestState.IsDialogueOpen() then
                return true
            end

            local nextButtons = ticketQuestState.GetDialogueButtons()
            for _, nb in ipairs(nextButtons) do
                -- Nếu nộp vé thành công hoặc thông báo chưa xong -> Bấm Leave
                if nb.clean:find("leave") or nb.clean:find("xong") or nb.clean:find("close") then
                    ticketQuestState.ClickButtonEntry(nb, "Close")
                    task.wait(0.3)
                    ticketQuestState.CloseDialogue()
                    return true
                end

                -- Nếu nhận vé mới -> Bấm độ khó tương ứng (Hard hoặc Easy)
                if not isClaiming and (nb.clean:find(targetDiffKeyword) or nb.clean:find("accept")) then
                    if nb.clean:find(targetDiffKeyword) or not nb.clean:find(isHard and "easy" or "hard") then
                        local act = isHard and "HardAcceptQuest" or "EasyAcceptQuest"
                        ticketQuestState.ClickButtonEntry(nb, act)
                        task.wait(0.4)
                        ticketQuestState.CloseDialogue()
                        return true
                    end
                end
            end

            -- Nếu sau 1.2s mà vẫn ở trang 1 có nút Quest -> Bấm lại Quest một lần nữa
            if (tick() - t0) > 1.2 and not retriedQuest then
                retriedQuest = true
                ticketQuestState.ClickButtonEntry(questBtn, "Quest")
            end
        end

        -- Không tự ý bấm Nevermind / Close khi hết giờ
        return false
    end

    return false
end

function ticketQuestState.InteractNPC(isClaiming)
    local isHard = (Config.TicketDifficulty or "Hard") == "Hard"
    ticketQuestState.isInteracting = true

    local function _execute()
        -- 1. Nếu bảng hội thoại NPC đã mở sẵn trước mặt
        if ticketQuestState.IsDialogueOpen() then
            if ticketQuestState.HandleDialogue(isClaiming) then
                if isClaiming then
                    ShowNotification("Nhiệm Vụ Vé", "Đã nộp nhiệm vụ & nhận thưởng vé thành công!", "SUCCESS", 5)
                else
                    ShowNotification("Nhiệm Vụ Vé", "Đã nhận thành công nhiệm vụ " .. (isHard and "Hard" or "Easy") .. " Ticket!", "SUCCESS", 5)
                end
                return true
            end
            task.wait(0.4)
            if ticketQuestState.IsDialogueOpen() and ticketQuestState.HandleDialogue(isClaiming) then
                return true
            end
            return false
        end

        -- 2. Di chuyển trực diện đến NPC & căn chỉnh góc nhìn camera thẳng vào NPC
        local npcModel, focusPos, promptFound, standPos = ticketQuestState.TeleportToNPC()

        -- 3. Chỉ kích hoạt Prompt hoặc bấm E khi bảng thoại CHƯA MỞ
        if not promptFound and npcModel then
            promptFound = npcModel:FindFirstChildWhichIsA("ProximityPrompt", true)
        end
        if not promptFound and focusPos then
            for _, p in ipairs(Workspace:GetDescendants()) do
                if p:IsA("ProximityPrompt") then
                    local pPos = (p.Parent:IsA("BasePart") and p.Parent.Position) or (p.Parent:IsA("Model") and p.Parent:GetPivot().Position)
                    if pPos and (pPos - focusPos).Magnitude <= 18 then
                        promptFound = p
                        break
                    end
                end
            end
        end

        if promptFound then
            pcall(function()
                promptFound.RequiresLineOfSight = false
                promptFound.MaxActivationDistance = math.max(promptFound.MaxActivationDistance or 10, 35)
                promptFound.Enabled = true
            end)
        end

        if not ticketQuestState.IsDialogueOpen() then
            if promptFound then
                TriggerPrompt(promptFound)
            end

            -- Gửi phím E mở thoại
            pcall(function()
                local vim = game:GetService("VirtualInputManager")
                if vim then
                    vim:SendKeyEvent(true, Enum.KeyCode.E, false, game)
                    task.wait(0.08)
                    vim:SendKeyEvent(false, Enum.KeyCode.E, false, game)
                end
            end)

            -- Click TextButton trên GUI ProximityPrompts nếu có (Mobile UI)
            pcall(function()
                local pg = LocalPlayer:FindFirstChild("PlayerGui")
                local pPrompts = pg and pg:FindFirstChild("ProximityPrompts")
                if pPrompts then
                    local btn = pPrompts:FindFirstChild("TextButton", true)
                    if btn and btn:IsA("GuiButton") and btn.Visible then
                        if firesignal then firesignal(btn.Activated) end
                        if getconnections then for _, c in ipairs(getconnections(btn.Activated)) do c:Fire() end end
                    end
                end
            end)
        end

        -- 4. Chờ bảng hội thoại xuất hiện và xử lý
        local success = false
        local retryTriggered = false
        local t0 = tick()
        while (tick() - t0) < 4.5 do
            task.wait(0.2)
            if ticketQuestState.IsDialogueOpen() then
                if ticketQuestState.HandleDialogue(isClaiming) then
                    success = true
                    break
                end
            else
                if (tick() - t0) > 1.2 and not retryTriggered then
                    retryTriggered = true
                    if focusPos and standPos then
                        ticketQuestState.OrientCameraTo(focusPos, standPos)
                    end
                    if promptFound then
                        TriggerPrompt(promptFound)
                    end
                    pcall(function()
                        local vim = game:GetService("VirtualInputManager")
                        if vim then
                            vim:SendKeyEvent(true, Enum.KeyCode.E, false, game)
                            task.wait(0.08)
                            vim:SendKeyEvent(false, Enum.KeyCode.E, false, game)
                        end
                    end)
                end
            end
        end

        -- Fallback RemoteEvent nếu game hỗ trợ
        if Events and Events:FindFirstChild("ClaimQuest") and isClaiming then
            pcall(function() Events.ClaimQuest:FireServer("Ticket", Config.TicketDifficulty or "Hard") end)
        end

        if success then
            if isClaiming then
                ShowNotification("Nhiệm Vụ Vé", "Đã nộp nhiệm vụ & nhận thưởng vé thành công!", "SUCCESS", 5)
            else
                ShowNotification("Nhiệm Vụ Vé", "Đã nhận thành công nhiệm vụ " .. (isHard and "Hard" or "Easy") .. " Ticket!", "SUCCESS", 5)
            end
        end

        return success
    end

    local ok, res = pcall(_execute)
    ticketQuestState.isInteracting = false
    ticketQuestState.ClearUINavigation()
    task.delay(0.25, ticketQuestState.ClearUINavigation)
    task.delay(0.6, ticketQuestState.ClearUINavigation)
    return ok and res or false
end

function ticketQuestState.DetectActiveQuest()
    local detectedType = nil
    local detectedTitle = nil
    local detectedCur = 0
    local detectedMax = 0
    local isDone = false
    local detectedCooldownSec = nil

    -- 0. ƯU TIÊN SỐ 1: Đọc trực tiếp từ ReplicatedStorage.Data[UserId] (Dữ liệu gốc của Server game, CHÍNH XÁC 100%, 0ms, 0 lag!)
    local pData = ticketQuestState.GetPlayerDataFolder()
    if pData then
        -- 0A. Đọc thời gian hồi chiêu
        local cdVal = pData:FindFirstChild("TicketQuestCooldown")
        if cdVal and tonumber(cdVal.Value) then
            local stamp = tonumber(cdVal.Value)
            local nowServer = ticketQuestState.GetServerTimeNow()
            if stamp > nowServer then
                detectedCooldownSec = stamp - nowServer
            end
        end

        -- 0B. Đọc nhiệm vụ từ pData.Quest (tìm trực tiếp hoặc tìm trong mọi thư mục con)
        local questFolder = pData:FindFirstChild("Quest")
        if questFolder then
            local tq = questFolder:FindFirstChild("Hard Ticket Quest", true) 
                or questFolder:FindFirstChild("Ticket Quest", true) 
                or questFolder:FindFirstChild("Easy Ticket Quest", true)
            if not tq then
                for _, ch in ipairs(questFolder:GetDescendants()) do
                    if ch:IsA("Folder") or ch:IsA("Configuration") then
                        local cn = ch.Name:lower()
                        if cn:find("ticket") and cn:find("quest") then
                            tq = ch
                            break
                        end
                    end
                end
            end

            if tq then
                -- Lấy tiến độ hiện tại (NumberValue tên là "1")
                local curVal = tq:FindFirstChild("1")
                if curVal and tonumber(curVal.Value) ~= nil then
                    detectedCur = tonumber(curVal.Value)
                end

                -- Lấy mục tiêu và tên nhiệm vụ (StringValue trong tq.Objective["1"]: "Fish for 100 times,100,Fishing")
                local objFolder = tq:FindFirstChild("Objective")
                local objVal = objFolder and objFolder:FindFirstChild("1")
                local objStr = objVal and tostring(objVal.Value or "") or ""
                local rawTitle, rawMax, rawCode = objStr:match("^([^,]+),([^,]+),?(.*)$")

                detectedMax = tonumber(rawMax) or 100
                detectedTitle = (rawTitle and #rawTitle > 0) and rawTitle or tq.Name
                local codeLow = (rawCode or ""):lower()
                local titleLow = (detectedTitle):lower()

                if codeLow:find("skill") or titleLow:find("skill") or titleLow:find("chiêu") or titleLow:find("kỹ năng") then
                    detectedType = "skill_100"
                    if detectedMax == 0 then detectedMax = 100 end
                elseif codeLow:find("bait") or titleLow:find("bait") or titleLow:find("mồi") then
                    detectedType = "bait_100"
                    if detectedMax == 0 then detectedMax = 100 end
                elseif detectedMax == 10 or titleLow:find("1.5") or titleLow:find("1500") or titleLow:find("heavy") then
                    detectedType = "fish_15m"
                    if detectedMax == 0 then detectedMax = 10 end
                else
                    detectedType = "fish_100"
                    if detectedMax == 0 then detectedMax = 100 end
                end

                if detectedMax > 0 and detectedCur >= detectedMax then
                    isDone = true
                end

                -- pData là nguồn tin cậy: nếu có quest folder thì đây là quest thật, trả về ngay
                -- (Game set TicketQuestCooldown ngay khi nhận quest nên không thể dùng cooldown để phân biệt)
                return detectedType, detectedTitle, detectedCur, detectedMax, isDone, detectedCooldownSec
            end
        end

        -- Không có folder nhiệm vụ tq nào trong pData:
        -- pData là nguồn tin cậy nhất (dữ liệu gốc của server)
        -- Nếu không thấy quest -> chắc chắn không có quest đang hoạt động
        -- TUYỆT ĐỐI KHÔNG chạy xuống GUI scan vì sẽ đọc nhầm hội thoại NPC!
        if detectedCooldownSec and detectedCooldownSec > 0 then
            return nil, nil, 0, 0, false, detectedCooldownSec
        else
            -- Cooldown hết và không có quest -> sẵn sàng nhận quest mới
            return nil, nil, 0, 0, false, nil
        end
    end

    -- CHỈ dùng GUI scan nếu KHAI THAC pData thất bại (không tìm được folder Data)
    local mode = Config.TicketQuestMode or "Tự Động (Auto Detect)"
    if mode == "Câu 10 Con Cá 1.5M+ (Map 9)" then
        detectedType = "fish_15m"
        detectedTitle = "Câu 10 con cá >= 1.5M (Map 9)"
        detectedMax = 10
    elseif mode == "Tiêu Thụ 100 Mồi (Map 1)" then
        detectedType = "bait_100"
        detectedTitle = "Tiêu thụ 100 mồi câu (Map 1)"
        detectedMax = 100
    elseif mode == "Dùng Kỹ Năng 100 Lần" then
        detectedType = "skill_100"
        detectedTitle = "Dùng kỹ năng 100 lần"
        detectedMax = 100
    elseif mode == "Câu Nhanh 100 Con Cá (Map 1)" then
        detectedType = "fish_100"
        detectedTitle = "Câu nhanh 100 con cá (Map 1)"
        detectedMax = 100
    end

    -- 2. DỰ PHÒNG: Quét GUI cụ thể (Có bộ nhớ đệm cachedCard, không quét toàn bộ 30k elements của PlayerGui)
    local function cleanStr(s)
        if not s then return "" end
        local res = tostring(s):gsub("<[^>]->", "")
        return res:gsub("%s+", " "):match("^%s*(.-)%s*$") or ""
    end

    local pg = LocalPlayer:FindFirstChild("PlayerGui")
    if pg then
        local searchRoots = {}
        if ticketQuestState.cachedCard and ticketQuestState.cachedCard.Parent then
            table.insert(searchRoots, ticketQuestState.cachedCard)
        else
            if pg:FindFirstChild("MainGui") then table.insert(searchRoots, pg.MainGui) end
            if pg:FindFirstChild("Fisher_GUI") then table.insert(searchRoots, pg.Fisher_GUI) end
        end

        for _, root in ipairs(searchRoots) do
            for _, lbl in ipairs(root:GetDescendants()) do
                if lbl:IsA("TextLabel") or lbl:IsA("TextButton") then
                    local txt = cleanStr(lbl.Text)
                    local lower = txt:lower()

                    if lower:find("hard ticket quest") or lower:find("easy ticket quest") or (lower:find("ticket") and lower:find("quest")) then
                        local card = lbl.Parent
                        if card then
                            ticketQuestState.cachedCard = card
                        for _, sibling in ipairs(card:GetDescendants()) do
                            -- Bỏ qua nhãn tên là "Stats" vì đây là số lượt làm trong ngày (1/50)
                            if (sibling:IsA("TextLabel") or sibling:IsA("TextButton")) and sibling ~= lbl and sibling.Name ~= "Stats" then
                                local subTxt = cleanStr(sibling.Text)
                                local subLower = subTxt:lower()

                                local sc, sm = subTxt:match("(%d+)%s*/%s*(%d+)")
                                -- Bỏ qua nếu text này là hội thoại NPC báo cooldown ("come back in", "sorting out", "minutes")
                                local isNpcCooldownMsg = subLower:find("come back") or subLower:find("sorting out") 
                                    or subLower:find("still busy") or subLower:find("not ready")
                                    or subLower:find("minute") or subLower:find("hãy quay lại")
                                    or subLower:find("chưa sẵn sàng") or subLower:find("quản lý")
                                if isNpcCooldownMsg then
                                    -- Đây là thông báo NPC cooldown, bỏ qua hoàn toàn
                                elseif sc and sm then
                                    detectedCur = tonumber(sc) or 0
                                    detectedMax = tonumber(sm) or 0

                                    local nameOnly = subTxt:gsub("%s*%(%d+%s*/%s*%d+%)%s*", ""):match("^%s*(.-)%s*$")
                                    if nameOnly and #nameOnly > 0 and not nameOnly:lower():find("ticket quest") then
                                        detectedTitle = nameOnly
                                    end
                                elseif not subLower:find("ticket") and not subLower:find("quest") and #subTxt > 0 then
                                    if not detectedTitle or detectedTitle == "" then
                                        detectedTitle = subTxt
                                    end
                                end

                                if subLower:find("completed") or subLower:find("hoàn thành") or subLower:find("yuppie") then
                                    isDone = true
                                end
                            end
                        end

                        local titleLow = (detectedTitle or txt):lower()
                        if titleLow:find("skill") or titleLow:find("chiêu") or titleLow:find("kỹ năng") then
                            detectedType = "skill_100"
                            if detectedMax == 0 then detectedMax = 100 end
                        elseif titleLow:find("bait") or titleLow:find("mồi") then
                            detectedType = "bait_100"
                            if detectedMax == 0 then detectedMax = 100 end
                        elseif titleLow:find("1.5") or titleLow:find("1,500") or titleLow:find("1500") or titleLow:find("heavy") or detectedMax == 10 then
                            detectedType = "fish_15m"
                            if detectedMax == 0 then detectedMax = 10 end
                        elseif titleLow:find("fish") or titleLow:find("cá") or titleLow:find("catch") or titleLow:find("bắt") or detectedMax == 100 then
                            detectedType = "fish_100"
                            if detectedMax == 0 then detectedMax = 100 end
                        else
                            if detectedMax == 10 then detectedType = "fish_15m" else detectedType = "fish_100" end
                        end

                        if not detectedTitle or detectedTitle == "" then
                            detectedTitle = (detectedType == "skill_100" and "Use any skill for 100 times")
                                or (detectedType == "bait_100" and "Consume 100 bait")
                                or (detectedType == "fish_15m" and "Catch 10 fish over 1.5M")
                                or "Catch 100 fish"
                        end

                        if detectedMax > 0 and detectedCur >= detectedMax then
                            isDone = true
                        end

                        -- BẢO VỆ: Nếu cooldown server còn > 5 giây VÀ tiến độ = 0 -> dữ liệu cũ / NPC chưa ready
                        if detectedCooldownSec and detectedCooldownSec > 5 and detectedCur == 0 and not isDone then
                            return nil, nil, 0, 0, false, detectedCooldownSec
                        end

                        return detectedType, detectedTitle, detectedCur, detectedMax, isDone, detectedCooldownSec
                    end
                end
            end
        end
    end
    end

    return nil, nil, 0, 0, false, detectedCooldownSec
end

function ticketQuestState.ScanAndUpdateStatus()
    local qType, qTitle, cur, max, done, detectedCd = ticketQuestState.DetectActiveQuest()
    local now = tick()
    local nowServer = ticketQuestState.GetServerTimeNow()
    local pData = ticketQuestState.GetPlayerDataFolder()
    local cdVal = pData and pData:FindFirstChild("TicketQuestCooldown")
    local serverCd = cdVal and tonumber(cdVal.Value) or 0
    local serverRemain = serverCd > nowServer and (serverCd - nowServer) or 0

    -- 1. NẾU CÓ NHIỆM VỤ ĐANG HOẠT ĐỘNG: ƯU TIÊN SỐ 1 LÀ LÀM NHIỆM VỤ NÀY!
    if qType then
        ticketQuestState.active = true
        ticketQuestState.currentQuestType = qType
        ticketQuestState.currentQuestTitle = qTitle or "Nhiệm Vụ Vé"
        ticketQuestState.currentProgress = cur or 0
        ticketQuestState.targetProgress = (max and max > 0) and max or 100
        ticketQuestState.isCompleted = (done == true) or (ticketQuestState.targetProgress > 0 and ticketQuestState.currentProgress >= ticketQuestState.targetProgress)

        -- ĐANG CÓ NHIỆM VỤ TRONG NGƯỜI -> TUYỆT ĐỐI KHÔNG ĐƯỢC BẬT TRẠNG THÁI COOLDOWN ĐỂ DỪNG BOT!
        ticketQuestState.isCooldown = false
        ticketQuestState.cooldownEnd = 0

        if ticketQuestState.isCompleted then
            ticketQuestState.statusText = ticketQuestState.currentQuestTitle .. " (Đã Hoàn Thành - Đang Nộp Vé)"
        else
            ticketQuestState.statusText = string.format("Đang làm: %s (%d/%d)", ticketQuestState.currentQuestTitle, ticketQuestState.currentProgress, ticketQuestState.targetProgress)
        end
    else
        -- 2. KHÔNG CÓ NHIỆM VỤ NÀO TRONG NGƯỜI: MỚI ĐƯỢC PHÉP CHUYỂN SANG CHỜ HỒI CHIÊU ĐỂ NHẬN VÉ MỚI
        ticketQuestState.isCompleted = false
        ticketQuestState.currentQuestType = "none"
        ticketQuestState.currentQuestTitle = "Chưa nhận nhiệm vụ"
        ticketQuestState.currentProgress = 0
        ticketQuestState.targetProgress = 100

        -- BẢO VỆ: Không set isCooldown = true nếu:
        -- (1) Đang sẵn sàng nhận quest mới (readyForNewQuest)
        -- (2) Đang trong quá trình nhận quest (isAcceptingQuest) - tránh race condition với TicketQuestCooldown.Changed
        if ticketQuestState.readyForNewQuest or ticketQuestState.isAcceptingQuest then
            ticketQuestState.isCooldown = false
            if ticketQuestState.isAcceptingQuest then
                ticketQuestState.statusText = "Đang nhận nhiệm vụ mới từ NPC..."
            else
                ticketQuestState.statusText = "Hồi chiêu đã xong! Chuẩn bị nhận vé Hard mới..."
            end
        elseif serverRemain > 0 then
            ticketQuestState.isCooldown = true
            ticketQuestState.cooldownEnd = now + serverRemain
            local mins = math.floor(serverRemain / 60)
            local secs = serverRemain % 60
            ticketQuestState.statusText = string.format("Đang chờ hồi chiêu vé (còn %02d:%02d)", mins, secs)
        else
            ticketQuestState.isCooldown = false
            ticketQuestState.cooldownEnd = 0
            ticketQuestState.statusText = "Chưa nhận nhiệm vụ nào"
        end
    end

    ticketQuestState.UpdateUI()
    return qType, qTitle, cur, max, done, detectedCd
end

function ticketQuestState.ResetCooldown()
    ticketQuestState.isCooldown = false
    ticketQuestState.cooldownEnd = 0
    ticketQuestState.readyForNewQuest = true
    ticketQuestState.isCompleted = false
    ticketQuestState.isAtHomeSpot = false
    ticketQuestState.allQuestsDoneForToday = false
    ticketQuestState.allQuestsDoneDate = nil
    ticketQuestState.allQuestsDoneUtcDate = nil
    ticketQuestState.savedDailyCount = nil
    ticketQuestState.currentProgress = 0
    ticketQuestState.currentQuestType = "none"
    ticketQuestState.statusText = "Đã đặt lại! Sẵn sàng nhận vé mới."
    ticketQuestState.SaveSpots()
    ticketQuestState.UpdateUI()
    ShowNotification("Nhiệm Vụ Vé", "Đã xóa hồi chiêu, cờ hết vé và đặt lại bộ đếm!", "SUCCESS", 5)
end

--========================================================================--
--// 3.1. HỆ THỐNG NHIỆM VỤ ZENG TIANGUO (SKILL UPGRADE) & SONG SONG //--
--========================================================================--
local zengTianguoQuestState = {
    active = false,
    currentQuestTitle = "Chưa nhận nhiệm vụ",
    currentProgress = 0,
    targetProgress = 0,
    isCompleted = false,
    objectiveCode = "none",
    zoneTarget = "",
    statusText = "Đang quét nhiệm vụ Zeng Tianguo...",
    lastSyncTime = 0,
    lastNpcInteract = 0,
    lastClaimAttempt = 0,
    
    -- Tọa độ bãi câu mặc định của Zeng Tianguo
    spotBamboo = Vector3.new(-1223.0, 9.0, -24.1), -- Đảo Tre (Bamboo Isle)
    spotFrost = Vector3.new(-1366.0, 14.0, -1495.4), -- Đảo Băng (Frost Isle)
    spotSovereign = Vector3.new(-1276.4, 12.0, 1239.7), -- Đảo Thống Trị
    spotFallout = Vector3.new(65.5, 12.0, 1181.3), -- Đảo Phóng Xạ
    spotNPC = Vector3.new(65.5, 12.0, 1181.3), -- Vị trí NPC Zeng Tianguo
    
    -- UI rows
    uiStatus = nil,
    uiProgress = nil,
    uiParallelBadge = nil,
}

function zengTianguoQuestState.GetPlayerDataFolder()
    return ticketQuestState.GetPlayerDataFolder()
end

function zengTianguoQuestState.DetectActiveQuest()
    local pData = zengTianguoQuestState.GetPlayerDataFolder()
    if not pData then return nil, "Chưa nhận nhiệm vụ", 0, 0, false, nil end
    local questFolder = pData:FindFirstChild("Quest")
    if not questFolder then return nil, "Chưa nhận nhiệm vụ", 0, 0, false, nil end

    local zq = questFolder:FindFirstChild("Zeng Tianguo Quest", true)
        or questFolder:FindFirstChild("Tang Thien Quoc Quest", true)
        or questFolder:FindFirstChild("Tang Thien Quoc", true)
        or questFolder:FindFirstChild("Zeng Tianguo", true)

    if not zq then
        for _, ch in ipairs(questFolder:GetDescendants()) do
            if ch:IsA("Folder") or ch:IsA("Configuration") then
                local cn = ch.Name:lower()
                if (cn:find("zeng") and cn:find("tianguo")) or (cn:find("tang") and cn:find("thien")) then
                    zq = ch
                    break
                end
            end
        end
    end

    if not zq then
        zengTianguoQuestState.active = false
        zengTianguoQuestState.currentQuestTitle = "Chưa nhận nhiệm vụ"
        zengTianguoQuestState.currentProgress = 0
        zengTianguoQuestState.targetProgress = 0
        zengTianguoQuestState.isCompleted = false
        zengTianguoQuestState.objectiveCode = "none"
        zengTianguoQuestState.zoneTarget = ""
        zengTianguoQuestState.statusText = "Chưa nhận nhiệm vụ từ Zeng Tianguo"
        return nil, "Chưa nhận nhiệm vụ", 0, 0, false, nil
    end

    local cur = 0
    local curVal = zq:FindFirstChild("1")
    if curVal and tonumber(curVal.Value) ~= nil then
        cur = tonumber(curVal.Value)
    end

    local objFolder = zq:FindFirstChild("Objective")
    local objVal = objFolder and objFolder:FindFirstChild("1")
    local objStr = objVal and tostring(objVal.Value or "") or ""
    
    local rawTitle, rawMax, rawCode, rawExtra = objStr:match("^([^,]+),([^,]+),([^,]+),?(.*)$")
    if not rawTitle then
        rawTitle, rawMax = objStr:match("^([^,]+),([^,]+)")
    end

    local max = tonumber(rawMax) or 100
    local title = (rawTitle and #rawTitle > 0) and rawTitle or zq.Name
    local code = rawCode or "none"
    local extra = rawExtra or ""

    local isDone = (max > 0 and cur >= max)

    zengTianguoQuestState.active = true
    zengTianguoQuestState.currentQuestTitle = title
    zengTianguoQuestState.currentProgress = cur
    zengTianguoQuestState.targetProgress = max
    zengTianguoQuestState.isCompleted = isDone
    zengTianguoQuestState.objectiveCode = code
    zengTianguoQuestState.zoneTarget = extra

    if isDone then
        zengTianguoQuestState.statusText = string.format("Đã xong: %s (%d/%d) - Sẵn sàng nộp quest!", title, cur, max)
    else
        zengTianguoQuestState.statusText = string.format("Đang làm: %s (%d/%d)", title, cur, max)
    end

    return code, title, cur, max, isDone, extra
end

function zengTianguoQuestState.GetTargetSpot()
    local zone = (zengTianguoQuestState.zoneTarget or ""):lower()
    local title = (zengTianguoQuestState.currentQuestTitle or ""):lower()
    if zone:find("bamboo") or zone:find("tre") or title:find("bamboo") then
        return zengTianguoQuestState.spotBamboo
    elseif zone:find("frost") or zone:find("băng") or title:find("frost") then
        return zengTianguoQuestState.spotFrost
    elseif zone:find("sovereign") or title:find("sovereign") then
        return zengTianguoQuestState.spotSovereign
    elseif zone:find("fallout") or title:find("fallout") then
        return zengTianguoQuestState.spotFallout
    end
    if zengTianguoQuestState.objectiveCode == "UseSkillForTimes" or title:find("skill") or title:find("chiêu") then
        return nil
    end
    return zengTianguoQuestState.spotBamboo
end

function zengTianguoQuestState.FindNPCModel()
    -- Đồng bộ 100% logic tìm NPC từ tab Dịch Chuyển (findNPCModel)
    local npcFolder = Workspace:FindFirstChild("NPC")
    if npcFolder then
        local funcFolder = npcFolder:FindFirstChild("Function")
        if funcFolder then
            local found = funcFolder:FindFirstChild("Zeng Tianguo") or funcFolder:FindFirstChild("Tang Thien Quoc")
            if found and (found:FindFirstChild("HumanoidRootPart") or found:FindFirstChildWhichIsA("BasePart")) then
                return found
            end
        end
        for _, sub in ipairs(npcFolder:GetChildren()) do
            local found = sub:FindFirstChild("Zeng Tianguo") or sub:FindFirstChild("Tang Thien Quoc")
            if found and (found:FindFirstChild("HumanoidRootPart") or found:FindFirstChildWhichIsA("BasePart")) then
                return found
            end
        end
    end
    local direct = Workspace:FindFirstChild("Zeng Tianguo", true) or Workspace:FindFirstChild("Tang Thien Quoc", true)
    if direct then return direct end

    if npcFolder then
        for _, ch in ipairs(npcFolder:GetDescendants()) do
            if ch:IsA("Model") then
                local n = ch.Name:lower()
                if (n:find("zeng") and n:find("tianguo")) or (n:find("tang") and n:find("thien")) then
                    return ch
                end
            end
        end
    end
    return nil
end

function zengTianguoQuestState.TeleportToNPC()
    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if not root then return false end

    -- Lấy đúng chuẩn 100% tọa độ dịch chuyển của tab Dịch Chuyển: CFrame.new(npcRoot.Position + Vector3.new(0, 3, 3))
    local model = zengTianguoQuestState.FindNPCModel()
    if model then
        local npcRoot = model:FindFirstChild("HumanoidRootPart")
            or model:FindFirstChildWhichIsA("BasePart")
        if npcRoot then
            local targetPos = npcRoot.Position + Vector3.new(0, 3, 3)
            root.CFrame = CFrame.new(targetPos)
            zengTianguoQuestState.spotNPC = targetPos
            return true, model, npcRoot
        end
    end

    if zengTianguoQuestState.spotNPC then
        root.CFrame = CFrame.new(zengTianguoQuestState.spotNPC)
        return true
    end
    return false
end

function zengTianguoQuestState.InteractNPC(isClaim)
    local char = LocalPlayer.Character
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    if hum then
        pcall(function()
            hum.Sit = false
            hum:UnequipTools()
        end)
    end
    if Events and Events:FindFirstChild("CancelCast") then
        pcall(function() Events.CancelCast:FireServer() end)
    end

    zengTianguoQuestState.TeleportToNPC()
    task.wait(0.6)

    local npc = zengTianguoQuestState.FindNPCModel()
    if npc then
        local prompt = npc:FindFirstChildWhichIsA("ProximityPrompt", true)
        if prompt then
            pcall(function()
                fireproximityprompt(prompt)
            end)
            task.wait(0.6)
        end
    end

    local dlg = ticketQuestState.GetDialogueGui()
    if dlg and dlg.Visible then
        task.wait(0.3)
        local clicked = false
        if isClaim then
            clicked = ticketQuestState.ClickLeaveOrClose()
        else
            clicked = ticketQuestState.ClickQuestButton() or ticketQuestState.ClickLeaveOrClose()
        end
        task.wait(0.5)
        ticketQuestState.CloseDialogue()
        return clicked
    end
    return false
end

function zengTianguoQuestState.UpdateUI()
    if zengTianguoQuestState.uiStatus and zengTianguoQuestState.uiStatus.Set then
        zengTianguoQuestState.uiStatus.Set(zengTianguoQuestState.statusText)
    end
    if zengTianguoQuestState.uiProgress and zengTianguoQuestState.uiProgress.Set then
        local max = zengTianguoQuestState.targetProgress
        local cur = zengTianguoQuestState.currentProgress
        local pct = max > 0 and math.floor((cur / max) * 100) or 0
        zengTianguoQuestState.uiProgress.Set(string.format("%d / %d (%d%%)", cur, max, pct))
    end
    if zengTianguoQuestState.uiParallelBadge and zengTianguoQuestState.uiParallelBadge.Set then
        if Config.AutoTicketQuest and Config.AutoZengTianguoQuest then
            zengTianguoQuestState.uiParallelBadge.Set("⚡ SONG SONG (Ưu Tiên Vé)")
        elseif Config.AutoTicketQuest then
            zengTianguoQuestState.uiParallelBadge.Set("🎫 Chỉ Chạy Vé NV")
        elseif Config.AutoZengTianguoQuest then
            zengTianguoQuestState.uiParallelBadge.Set("⚡ Chỉ Chạy Zeng Tianguo")
        else
            zengTianguoQuestState.uiParallelBadge.Set("Đang Tắt")
        end
    end
end

function ticketQuestState.Tick()
    if not Config.AutoTicketQuest and not Config.AutoZengTianguoQuest then return end
    local now = tick()

    -- 0. Đồng bộ tiến độ cả 2 bên
    local isParallel = Config.AutoTicketQuest and Config.AutoZengTianguoQuest and Config.ParallelQuestMode
    local zCode, zTitle, zCur, zMax, zDone, zZone = zengTianguoQuestState.DetectActiveQuest()
    zengTianguoQuestState.UpdateUI()

    -- NẾU CHỈ BẬT ZENG TIANGUO (KHÔNG BẬT VÉ): CHẠY CHẾ ĐỘ SOLO ZENG TIANGUO
    if not Config.AutoTicketQuest and Config.AutoZengTianguoQuest then
        if zDone then
            if now - (zengTianguoQuestState.lastClaimAttempt or 0) >= 10.0 then
                zengTianguoQuestState.lastClaimAttempt = now
                zengTianguoQuestState.statusText = "Đã xong! Đang nộp quest Zeng Tianguo..."
                zengTianguoQuestState.UpdateUI()
                zengTianguoQuestState.InteractNPC(true)
                task.wait(1.0)
                zengTianguoQuestState.DetectActiveQuest()
                zengTianguoQuestState.UpdateUI()
            end
            return
        end

        local char = LocalPlayer.Character
        local pg = LocalPlayer:FindFirstChild("PlayerGui")
        local isFishing = char and char:GetAttribute("Fishing") == true
        local isMinigame = char and (char:GetAttribute("Minigame") == true or (pg and pg:FindFirstChild("MainGui") and pg.MainGui:FindFirstChild("Fishing") and pg.MainGui.Fishing.Visible))
        if isFishing or isMinigame then return end

        local targetSpot = zengTianguoQuestState.GetTargetSpot() or ticketQuestState.spot100Fish
        local root = char and char:FindFirstChild("HumanoidRootPart")
        local spotPos = typeof(targetSpot) == "CFrame" and targetSpot.Position or targetSpot
        if root and spotPos and (root.Position - spotPos).Magnitude > 25 then
            ticketQuestState.TeleportTo(targetSpot)
            ticketQuestState.CloseDialogue()
        end
        return
    end

    -- KIỂM TRA NẾU ĐÃ HẾT VÉ NHIỆM VỤ HÔM NAY (NPC "That's all the quests I've got for today! Come back tomorrow for more.")
    if ticketQuestState.IsAllQuestsDoneToday() then
        ticketQuestState.statusText = "Đã hết nhiệm vụ hôm nay! (Hẹn ngày mai quay lại)"
        ticketQuestState.isCooldown = true
        ticketQuestState.readyForNewQuest = false

        -- NẾU BẬT ZENG TIANGUO VÀ CHƯA XONG: TẬN DỤNG CÀY NỐT ZENG TIANGUO THAY VÌ ĐỨNG IM!
        if Config.AutoZengTianguoQuest and zCode and not zDone then
            local subSpot = zengTianguoQuestState.GetTargetSpot() or ticketQuestState.spot100Fish
            local char = LocalPlayer.Character
            local root = char and char:FindFirstChild("HumanoidRootPart")
            local spotPos = typeof(subSpot) == "CFrame" and subSpot.Position or subSpot
            if root and spotPos and (root.Position - spotPos).Magnitude > 25 then
                ticketQuestState.TeleportTo(subSpot)
            end
            ticketQuestState.statusText = string.format("Hết vé hôm nay: Đang cày Zeng Tianguo [%s (%d/%d)]", zTitle, zCur, zMax)
            ticketQuestState.UpdateUI()
            return
        end

        -- Tự động đưa về Home Spot để farm câu thường nếu có cài đặt
        if Config.TicketReturnHomeWhenDone and Config.HomeFarmSpot and not ticketQuestState.isAtHomeSpot then
            local char = LocalPlayer.Character
            local root = char and char:FindFirstChild("HumanoidRootPart")
            if root then
                local homeCf = nil
                if Config.HomeFarmSpot.cframe and #Config.HomeFarmSpot.cframe == 12 then
                    homeCf = CFrame.new(table.unpack(Config.HomeFarmSpot.cframe))
                elseif Config.HomeFarmSpot.x and Config.HomeFarmSpot.y and Config.HomeFarmSpot.z then
                    homeCf = CFrame.new(Config.HomeFarmSpot.x, Config.HomeFarmSpot.y, Config.HomeFarmSpot.z)
                end
                if homeCf then
                    local wp = Workspace:FindFirstChild("IdenticalWaterPlatform") or Workspace:FindFirstChild("WaterPlatform")
                    if not wp then
                        wp = Instance.new("Part")
                        wp.Name = "IdenticalWaterPlatform"
                        wp.Size = Vector3.new(30, 2, 30)
                        wp.Transparency = 1
                        wp.Anchored = true
                        wp.CanCollide = true
                        wp.Parent = Workspace
                    end
                    wp.CFrame = CFrame.new(homeCf.Position.X, homeCf.Position.Y - 2.8, homeCf.Position.Z)
                    wp.CanCollide = true
                    root.CFrame = homeCf + Vector3.new(0, 1.5, 0)
                    task.wait(0.2)
                    root.CFrame = homeCf
                    ticketQuestState.isAtHomeSpot = true
                    ticketQuestState.statusText = "Hết quest hôm nay: đã về Home Spot farm combo!"
                    ShowNotification("Home Spot", "Đã về vị trí Home Spot farm vì đã hết vé hôm nay!", "SUCCESS", 5)
                    if Config.TicketAutoCastAtHome then
                        task.delay(1.0, function()
                            if isRunning and ticketQuestState.isAtHomeSpot then
                                CancelAndRecastRod()
                            end
                        end)
                    end
                end
            end
        end
        ticketQuestState.UpdateUI()
        return
    end

    -- 0. Quét đồng bộ trạng thái và tiến độ nhiệm vụ trước khi thực thi
    local qType, qTitle, cur, max, done, detectedCd = ticketQuestState.ScanAndUpdateStatus()
    local isDoneNow = ticketQuestState.isCompleted or done or (ticketQuestState.targetProgress > 0 and ticketQuestState.currentProgress >= ticketQuestState.targetProgress)

    -- NẾU CÓ QUEST ĐÃ HOÀN THÀNH HOẶC VƯỢT CHỈ TIÊU (VÍ DỤ 105/100): HỦY COOLDOWN ĐỂ TIẾN HÀNH TRẢ VÉ!
    if isDoneNow then
        ticketQuestState.isCooldown = false
        ticketQuestState.cooldownEnd = 0
    end

    -- KIỂM TRA PHÂN CẤP ĐỘ ƯU TIÊN (PRIORITY MANAGER)
    if Config.PrioritySystemEnabled and PriorityManager and PriorityManager.GetActiveTask then
        local curTask = PriorityManager.GetActiveTask()
        local ticketPri = PriorityManager.GetTaskPriority("TicketQuest")
        if curTask ~= "None" and curTask ~= "TicketQuest" and PriorityManager.GetTaskPriority(curTask) < ticketPri then
            local taskName = PriorityManager.GetTaskDisplayName(curTask)
            ticketQuestState.statusText = string.format("Tạm hoãn vé: Đang nhường quyền cho [%s] (Hạng %d)", taskName, PriorityManager.GetTaskPriority(curTask))
            ticketQuestState.UpdateUI()
            return
        end
    end

    -- 1. Cooldown: CHỈ CHẠY KHI KHÔNG CÓ QUEST NÀO VÀ KHÔNG PHẢI QUEST HOÀN THÀNH
    local hasActiveQuest = (ticketQuestState.currentQuestType and ticketQuestState.currentQuestType ~= "none") or (qType ~= nil)
    if not hasActiveQuest and not isDoneNow and ticketQuestState.isCooldown then
        local nowServer = ticketQuestState.GetServerTimeNow()
        local pData = ticketQuestState.GetPlayerDataFolder()
        local cdVal = pData and pData:FindFirstChild("TicketQuestCooldown")
        local serverCd = cdVal and tonumber(cdVal.Value) or 0
        local remain = 0
        if serverCd > nowServer then
            remain = math.max(0, math.floor(serverCd - nowServer))
            ticketQuestState.cooldownEnd = now + remain
            ticketQuestState.isCooldown = true
        else
            remain = math.max(0, math.floor((ticketQuestState.cooldownEnd or 0) - now))
            if serverCd > 0 and serverCd <= nowServer then
                remain = 0
            end
        end

        local isReady = ticketQuestState.CheckNPCReady()
        if (remain <= 0 or isReady) and remain <= 0 then
            ticketQuestState.readyForNewQuest = true
            ticketQuestState.cachedCard = nil
            ticketQuestState.isCooldown = false
            ticketQuestState.isAtHomeSpot = false
            ticketQuestState.statusText = "Hồi chiêu đã xong! Chuẩn bị nhận vé Hard mới..."
            ticketQuestState.currentQuestType = "none"
            ticketQuestState.currentProgress = 0
            ticketQuestState.isCompleted = false
            ticketQuestState.UpdateUI()
        else
            if remain > 0 then
                local mins = math.floor(remain / 60)
                local secs = remain % 60

                -- KIỂM TRA TRẢ ZENG TIANGUO NẾU XONG TRONG LÚC CHỜ COOLDOWN VÉ
                if Config.AutoZengTianguoQuest and zDone and Config.ZengTianguoAutoClaim then
                    if now - (zengTianguoQuestState.lastClaimAttempt or 0) >= 15.0 then
                        zengTianguoQuestState.lastClaimAttempt = now
                        zengTianguoQuestState.statusText = "Đang ghé NPC nộp quest Zeng Tianguo..."
                        zengTianguoQuestState.UpdateUI()
                        zengTianguoQuestState.InteractNPC(true)
                        task.wait(1.0)
                        zengTianguoQuestState.DetectActiveQuest()
                        zengTianguoQuestState.UpdateUI()
                        return
                    end
                end

                -- THỜI GIAN VÀNG: Nếu đang chờ Cooldown vé mà có bật Zeng Tianguo và Zeng Tianguo chưa xong:
                if Config.AutoZengTianguoQuest and zCode and not zDone then
                    local subSpot = zengTianguoQuestState.GetTargetSpot() or ticketQuestState.spot100Fish
                    ticketQuestState.statusText = string.format("⚡ Song Song: Chờ vé (%02d:%02d) • Đang cày Zeng Tianguo (%d/%d)", mins, secs, zCur, zMax)
                    
                    local char = LocalPlayer.Character
                    local root = char and char:FindFirstChild("HumanoidRootPart")
                    local spotPos = typeof(subSpot) == "CFrame" and subSpot.Position or subSpot
                    if root and spotPos and (root.Position - spotPos).Magnitude > 25 then
                        ticketQuestState.TeleportTo(subSpot)
                        ticketQuestState.CloseDialogue()
                    end
                    ticketQuestState.UpdateUI()
                    return
                end

                ticketQuestState.statusText = string.format("Đang chờ hồi chiêu vé (còn %02d:%02d)", mins, secs)

                -- Về Home Spot câu cá trong thời gian chờ hồi chiêu nếu không làm Zeng Tianguo
                if Config.TicketReturnHomeWhenDone and Config.HomeFarmSpot and not ticketQuestState.isAtHomeSpot then
                    local char = LocalPlayer.Character
                    local root = char and char:FindFirstChild("HumanoidRootPart")
                    if root then
                        local homeCf = nil
                        if Config.HomeFarmSpot.cframe and #Config.HomeFarmSpot.cframe == 12 then
                            homeCf = CFrame.new(table.unpack(Config.HomeFarmSpot.cframe))
                        elseif Config.HomeFarmSpot.x and Config.HomeFarmSpot.y and Config.HomeFarmSpot.z then
                            homeCf = CFrame.new(Config.HomeFarmSpot.x, Config.HomeFarmSpot.y, Config.HomeFarmSpot.z)
                        end
                        if homeCf then
                            local wp = Workspace:FindFirstChild("IdenticalWaterPlatform") or Workspace:FindFirstChild("WaterPlatform")
                            if not wp then
                                wp = Instance.new("Part")
                                wp.Name = "IdenticalWaterPlatform"
                                wp.Size = Vector3.new(30, 2, 30)
                                wp.Transparency = 1
                                wp.Anchored = true
                                wp.CanCollide = true
                                wp.Parent = Workspace
                            end
                            wp.CFrame = CFrame.new(homeCf.Position.X, homeCf.Position.Y - 2.8, homeCf.Position.Z)
                            wp.CanCollide = true
                            root.CFrame = homeCf + Vector3.new(0, 1.5, 0)
                            task.wait(0.2)
                            root.CFrame = homeCf
                            ticketQuestState.isAtHomeSpot = true
                            ticketQuestState.statusText = string.format("Đang chờ hồi: đã về Home Spot farm combo (còn %02d:%02d)", mins, secs)
                            ShowNotification("Home Spot", "Đã về vị trí Home Spot để câu farm trong lúc chờ vé!", "SUCCESS", 5)

                            if Config.TicketAutoCastAtHome then
                                task.delay(1.0, function()
                                    if isRunning and ticketQuestState.isCooldown and ticketQuestState.isAtHomeSpot then
                                        CancelAndRecastRod()
                                    end
                                end)
                            end
                        end
                    end
                end

                ticketQuestState.UpdateUI()
                return
            else
                ticketQuestState.readyForNewQuest = true
                ticketQuestState.isCooldown = false
                ticketQuestState.isAtHomeSpot = false
                ticketQuestState.statusText = "Hồi chiêu đã xong! Chuẩn bị nhận vé Hard mới..."
                ticketQuestState.currentQuestType = "none"
                ticketQuestState.currentProgress = 0
                ticketQuestState.isCompleted = false
                ticketQuestState.UpdateUI()
            end
        end
    end

    -- 2. Đã hoàn thành nhiệm vụ -> Trả vé tại NPC (Ưu tiên số 1 tuyệt đối)
    if isDoneNow then
        if now - (ticketQuestState.lastClaimAttempt or 0) >= 3.5 then
            ticketQuestState.lastClaimAttempt = now
            ticketQuestState.statusText = "Đã xong nhiệm vụ! Đang nộp vé Hard..."
            ticketQuestState.UpdateUI()

            local char = LocalPlayer.Character
            local hum = char and char:FindFirstChildOfClass("Humanoid")
            if hum then
                pcall(function()
                    hum.Sit = false
                    hum:UnequipTools()
                end)
            end
            if Events and Events:FindFirstChild("CancelCast") then
                pcall(function() Events.CancelCast:FireServer() end)
            end

            local claimSuccess = ticketQuestState.InteractNPC(true)
            ticketQuestState.ClearUINavigation()
            task.delay(0.3, ticketQuestState.ClearUINavigation)
            task.wait(0.5)

            local checkType, _, _, _, checkDone, checkCd = ticketQuestState.DetectActiveQuest()
            local nowServer = ticketQuestState.GetServerTimeNow()
            local pData = ticketQuestState.GetPlayerDataFolder()
            local serverCd = pData and pData:FindFirstChild("TicketQuestCooldown") and tonumber(pData.TicketQuestCooldown.Value) or 0
            local questCleared = (checkType == nil) or (not checkDone and serverCd > nowServer)

            if questCleared or (serverCd > nowServer) or claimSuccess then
                local setCooldown = (serverCd > nowServer) and (serverCd - nowServer) or (20 * 60)
                ticketQuestState.cooldownEnd = now + setCooldown
                ticketQuestState.isCooldown = true
                ticketQuestState.isCompleted = false
                ticketQuestState.isAtHomeSpot = false
                ticketQuestState.currentQuestType = "none"
                ticketQuestState.currentProgress = 0
                ticketQuestState.active = false
                ticketQuestState.statusText = string.format("Đã nộp vé Hard! Đang chờ hồi chiêu (%d phút)", math.ceil(setCooldown / 60))
                ticketQuestState.SaveSpots()
                ticketQuestState.UpdateUI()
                ShowNotification("Nhiệm Vụ Vé", string.format("Đã nộp vé Hard! Đang chờ hồi chiêu (%d phút).", math.ceil(setCooldown / 60)), "SUCCESS", 8)

                -- Gửi webhook Discord khi hoàn thành nhiệm vụ vé
                if Config.WebhookEnabled and Config.WebhookUrl and #Config.WebhookUrl > 0 and (Config.WebhookNotifyTicketQuest ~= false) then
                    task.delay(1.2, function()
                        local gemsAfter = secretBossState.GetPlayerGems()
                        local pData2 = ticketQuestState.GetPlayerDataFolder()
                        local qCount = pData2 and pData2:FindFirstChild("TicketQuestDailyCount") and tonumber(pData2.TicketQuestDailyCount.Value) or 0
                        local ticketCount = pData2 and pData2:FindFirstChild("Ticket") and tonumber(pData2.Ticket.Value) or 0
                        SendTicketQuestWebhook(qCount, ticketCount, 0, gemsAfter, math.ceil(setCooldown / 60))
                    end)
                end

            else
                ticketQuestState.statusText = "Đã xong! Đang tiếp tục tương tác NPC để nộp vé..."
                ticketQuestState.UpdateUI()
            end
        end
        return
    end

    -- 3. Cập nhật hoặc ghi nhớ Quest nếu đã nhận diện được từ game
    if qType then
        if ticketQuestState.currentQuestType ~= qType then
            ticketQuestState.currentQuestType = qType
            ticketQuestState.currentQuestTitle = qTitle or ticketQuestState.currentQuestTitle
            ticketQuestState.targetProgress = (max and max > 0) and max or (qType == "fish_15m" and 10 or 100)
            ticketQuestState.statusText = "Đang làm: " .. ticketQuestState.currentQuestTitle
            ticketQuestState.UpdateUI()
        end
    end

    if cur and cur > ticketQuestState.currentProgress then
        ticketQuestState.currentProgress = cur
    end
    if max and max > ticketQuestState.targetProgress then
        ticketQuestState.targetProgress = max
    end
    if ticketQuestState.targetProgress > 0 and ticketQuestState.currentProgress >= ticketQuestState.targetProgress then
        ticketQuestState.isCompleted = true
        ticketQuestState.UpdateUI()
    end

    -- 4. Chưa có quest nào (currentQuestType == "none") -> Đến NPC nhận vé (Giãn cách 10s)
    if ticketQuestState.currentQuestType == "none" then
        if now - ticketQuestState.lastNpcInteract >= 10.0 then
            ticketQuestState.lastNpcInteract = now
            ticketQuestState.statusText = "Đang tương tác NPC nhận vé Hard mới..."
            ticketQuestState.UpdateUI()

            ticketQuestState.isAcceptingQuest = true
            ticketQuestState.isCooldown = false
            ticketQuestState.isAtHomeSpot = false

            ticketQuestState.InteractNPC(false)
            ticketQuestState.ClearUINavigation()
            task.delay(0.3, ticketQuestState.ClearUINavigation)

            task.wait(3.0)
            ticketQuestState.isAcceptingQuest = false
            local freshType, freshTitle, freshCur, freshMax, freshDone = ticketQuestState.DetectActiveQuest()
            if freshType then
                ticketQuestState.readyForNewQuest = false
                ticketQuestState.currentQuestType = freshType
                ticketQuestState.currentQuestTitle = freshTitle or "Nhiệm Vụ Vé"
                ticketQuestState.targetProgress = freshMax > 0 and freshMax or (freshType == "fish_15m" and 10 or 100)
                ticketQuestState.currentProgress = freshCur or 0
                ticketQuestState.active = true
                ticketQuestState.isAtHomeSpot = false
                ticketQuestState.statusText = "Đang làm: " .. ticketQuestState.currentQuestTitle
                ticketQuestState.UpdateUI()

                -- Dịch chuyển tới điểm câu (Có tích hợp ưu tiên bãi câu Song Song)
                local targetSpot = ticketQuestState.spot100Fish
                if freshType == "fish_15m" then
                    targetSpot = ticketQuestState.spot15MFish
                elseif freshType == "bait_100" then
                    targetSpot = ticketQuestState.spot100Bait or ticketQuestState.spot100Fish
                elseif freshType == "skill_100" then
                    targetSpot = ticketQuestState.spot100Skill or ticketQuestState.spot100Fish
                end

                -- Ghép bãi câu nếu chạy Song Song
                if isParallel and zCode and not zDone and freshType ~= "fish_15m" then
                    local subSpot = zengTianguoQuestState.GetTargetSpot()
                    if subSpot then
                        targetSpot = subSpot
                    end
                end

                local char = LocalPlayer.Character
                local root = char and char:FindFirstChild("HumanoidRootPart")
                local spotPos = typeof(targetSpot) == "CFrame" and targetSpot.Position or targetSpot
                if root and spotPos then
                    local dist = (root.Position - spotPos).Magnitude
                    if dist > 25 then
                        ticketQuestState.TeleportTo(targetSpot)
                    end
                    ticketQuestState.CloseDialogue()
                end
            else
                ticketQuestState.statusText = "Đã gửi lệnh nhận vé, đang chờ hệ thống cập nhật nhiệm vụ..."
                ticketQuestState.UpdateUI()
            end
        end
        return
    end

    -- 5. Quest đang hoạt động (currentQuestType ~= "none") -> Ở YÊN TẠI ĐIỂM CÂU VÀ LÀM NHIỆM VỤ!
    ticketQuestState.active = true
    ticketQuestState.isAtHomeSpot = false

    local char = LocalPlayer.Character
    local pg = LocalPlayer:FindFirstChild("PlayerGui")
    local isFishing = char and char:GetAttribute("Fishing") == true
    local isMinigame = char and (char:GetAttribute("Minigame") == true or (pg and pg:FindFirstChild("MainGui") and pg.MainGui:FindFirstChild("Fishing") and pg.MainGui.Fishing.Visible))
    if isFishing or isMinigame then
        return
    end

    -- Xác định vị trí câu tương ứng với loại nhiệm vụ hiện tại
    local targetSpot = ticketQuestState.spot100Fish
    if ticketQuestState.currentQuestType == "fish_15m" then
        targetSpot = ticketQuestState.spot15MFish
    elseif ticketQuestState.currentQuestType == "bait_100" then
        targetSpot = ticketQuestState.spot100Bait or ticketQuestState.spot100Fish
    elseif ticketQuestState.currentQuestType == "skill_100" then
        targetSpot = ticketQuestState.spot100Skill or ticketQuestState.spot100Fish
    end

    -- ĐIỀU PHỐI SONG SONG: Nếu vé không kén vị trí, ưu tiên bãi của Zeng Tianguo để cả 2 cùng nhảy số!
    if isParallel and zCode and not zDone and ticketQuestState.currentQuestType ~= "fish_15m" then
        local subSpot = zengTianguoQuestState.GetTargetSpot()
        if subSpot then
            targetSpot = subSpot
        end
    end

    -- Nếu bị trôi xa khỏi vị trí câu quest, bay về vị trí câu
    local root = char and char:FindFirstChild("HumanoidRootPart")
    local spotPos = typeof(targetSpot) == "CFrame" and targetSpot.Position or targetSpot
    if root and spotPos then
        local dist = (root.Position - spotPos).Magnitude
        if dist > 25 then
            ticketQuestState.TeleportTo(targetSpot)
        end
        ticketQuestState.CloseDialogue()
    end

    -- Cập nhật giao diện định kỳ
    if now - (ticketQuestState.lastSyncTime or 0) >= 2.0 then
        ticketQuestState.lastSyncTime = now
        ticketQuestState.UpdateUI()
    end

    -- Tự bán cá nếu đầy balo
    if Config.TicketAutoSellFull then
        local pData = ticketQuestState.GetPlayerDataFolder()
        if pData and pData:FindFirstChild("InventoryLimit") then
            local invCount = 0
            if pData:FindFirstChild("Inventory") then invCount = invCount + #pData.Inventory:GetChildren() end
            if invCount >= (pData.InventoryLimit.Value - 1) then
                if Events and Events:FindFirstChild("SellFish") then
                    ProtectAllInventoryItems(false)
                    Events.SellFish:FireServer("All")
                end
            end
        end
    end

    -- Nếu là quest 100 mồi: tự kiểm tra và mua mồi + trang bị mồi
    if ticketQuestState.currentQuestType == "bait_100" then
        local baitName = Config.TicketBaitChoice or "Basic Bait"
        local pData = ticketQuestState.GetPlayerDataFolder()
        if pData and pData:FindFirstChild("Bait") then
            local bVal = pData.Bait:FindFirstChild(baitName)
            local bCount = bVal and bVal.Value or 0
            if bCount < 5 and (now - ticketQuestState.lastBaitBuy >= 2.0) then
                ticketQuestState.lastBaitBuy = now
                if Events and Events:FindFirstChild("BuyBait") then
                    Events.BuyBait:FireServer(baitName, 30)
                end
            end
            if pData:FindFirstChild("EquippedBait") and pData.EquippedBait.Value ~= baitName and (now - ticketQuestState.lastBaitEquip >= 2.0) then
                ticketQuestState.lastBaitEquip = now
                if Events and Events:FindFirstChild("EquipBait") then
                    task.spawn(function()
                        Events.EquipBait:InvokeServer(baitName)
                    end)
                end
            end
        end
    elseif ticketQuestState.currentQuestType == "fish_100" then
        local pData = ticketQuestState.GetPlayerDataFolder()
        if pData and pData:FindFirstChild("EquippedBait") and pData.EquippedBait.Value ~= "None" and (now - ticketQuestState.lastBaitEquip >= 2.0) then
            ticketQuestState.lastBaitEquip = now
            if Events and Events:FindFirstChild("EquipBait") then
                task.spawn(function()
                    Events.EquipBait:InvokeServer("None")
                end)
            end
        end
    end
end

-- Vòng lặp ngầm: LUÔN LUÔN quét và cập nhật tiến độ nhiệm vụ (Dù có bật Auto làm vé hay không, dù ở bất kỳ Tab nào)
task.spawn(function()
    pcall(function() ticketQuestState.LoadSpots() end)
    task.delay(1.0, function()
        pcall(ticketQuestState.ScanAndUpdateStatus)
        pcall(zengTianguoQuestState.DetectActiveQuest)
        pcall(zengTianguoQuestState.UpdateUI)
    end)
    while isRunning do
        task.wait(1.0)
        pcall(ticketQuestState.ScanAndUpdateStatus)
        pcall(zengTianguoQuestState.DetectActiveQuest)
        pcall(zengTianguoQuestState.UpdateUI)
        pcall(function()
            if PriorityManager and PriorityManager.UpdateUI then
                PriorityManager.UpdateUI()
            end
        end)
        if Config.AutoTicketQuest or Config.AutoZengTianguoQuest then
            pcall(function()
                ticketQuestState.Tick()
            end)
        end
    end
end)

-- Lắng nghe trực tiếp khi có sự kiện thay đổi dữ liệu Quest trong Data người chơi (0ms phản hồi)
task.spawn(function()
    local pDataInit = ReplicatedStorage:WaitForChild("Data", 15)
    local userFolder = pDataInit and pDataInit:WaitForChild(tostring(LocalPlayer.UserId), 15)
    if not userFolder then return end

    local questFolder = userFolder:WaitForChild("Quest", 15)
    if questFolder then
        local function bindQuestDescendant(desc)
            if desc:IsA("ValueBase") then
                table.insert(activeConnections, desc.Changed:Connect(function()
                    pcall(ticketQuestState.ScanAndUpdateStatus)
                end))
            end
        end

        for _, d in ipairs(questFolder:GetDescendants()) do
            bindQuestDescendant(d)
        end

        table.insert(activeConnections, questFolder.DescendantAdded:Connect(function(newDesc)
            bindQuestDescendant(newDesc)
            task.wait(0.1)
            pcall(ticketQuestState.ScanAndUpdateStatus)
        end))

        table.insert(activeConnections, questFolder.DescendantRemoving:Connect(function()
            task.wait(0.1)
            pcall(ticketQuestState.ScanAndUpdateStatus)
        end))
    end

    local function hookCd(val)
        if val and val.Name == "TicketQuestCooldown" and val:IsA("ValueBase") then
            table.insert(activeConnections, val.Changed:Connect(function()
                pcall(ticketQuestState.ScanAndUpdateStatus)
                pcall(ticketQuestState.UpdateUI)
            end))
        end
    end

    local cdVal = userFolder:FindFirstChild("TicketQuestCooldown")
    if cdVal then hookCd(cdVal) end
    table.insert(activeConnections, userFolder.ChildAdded:Connect(function(child)
        hookCd(child)
    end))
end)
-- ===============================================================
-- 👑 HỆ THỐNG QUẢN LÝ ĐỘ ƯU TIÊN TÙY CHỈNH (PRIORITY MANAGER)
-- ===============================================================
PriorityManager.uiStatusRow = nil
PriorityManager.lastReportedTask = "None"
PriorityManager.cachedTask = "None"
PriorityManager.lastTaskScan = 0

function PriorityManager.IsTaskActive(taskId)
    if taskId == "SecretBoss" then
        if not (Config.AutoChatSecretBoss or Config.AutoHuntBoss) then return false end
        if secretBossState and secretBossState.active then return true end
        if secretBossState and secretBossState.activeChatBoss and (tick() - (secretBossState.activeChatBoss.time or 0) < 300) then
            return true
        end
        if secretBossState and secretBossState.standPos then
            return true
        end
        -- Nếu đang có thời tiết Boss diễn ra thực tế trong game, SecretBoss lập tức được kích hoạt quyền ưu tiên!
        if secretBossState and secretBossState.DetectWeather then
            local wIsland, wName = secretBossState.DetectWeather()
            if wIsland and wName ~= "Clear" then
                return true
            end
        end
        return false
    elseif taskId == "TicketQuest" then
        if not Config.AutoTicketQuest then return false end
        if not ticketQuestState then return false end
        -- Nếu đã hết tất cả nhiệm vụ vé hôm nay thì KHÔNG giữ quyền ưu tiên
        if ticketQuestState.IsAllQuestsDoneToday and ticketQuestState.IsAllQuestsDoneToday() then return false end
        -- Nếu đang trong thời gian hồi chiêu (Cooldown 20p) thì KHÔNG giữ quyền
        if ticketQuestState.isCooldown then return false end
        if ticketQuestState.isCompleted then return true end
        if ticketQuestState.readyForNewQuest then return true end
        if ticketQuestState.currentQuestType and ticketQuestState.currentQuestType ~= "none" then return true end
        return false
    elseif taskId == "GodSpirit" then
        if not Config.AutoPrayGodSpirit then return false end
        local npc = Workspace:FindFirstChild("NPC")
        if npc and (npc:FindFirstChild("Spirit") or npc:FindFirstChild("God")) then
            return true
        end
        return false
    elseif taskId == "TrainSkill" then
        return Config.AutoTrainSkill == true
    elseif taskId == "NormalFarm" then
        return Config.AutoCast == true
    end
    return false
end

function PriorityManager.GetTaskPriority(taskId)
    if taskId == "SecretBoss" then return tonumber(Config.Priority_SecretBoss) or 1 end
    if taskId == "TicketQuest" then return tonumber(Config.Priority_TicketQuest) or 2 end
    if taskId == "GodSpirit" then return tonumber(Config.Priority_GodSpirit) or 3 end
    if taskId == "TrainSkill" then return tonumber(Config.Priority_TrainSkill) or 4 end
    if taskId == "NormalFarm" then return tonumber(Config.Priority_NormalFarm) or 5 end
    return 999
end

function PriorityManager.GetTaskDisplayName(taskId)
    if taskId == "SecretBoss" then return "🎯 Săn Secret Boss" end
    if taskId == "TicketQuest" then return "📜 Làm Vé Nhiệm Vụ" end
    if taskId == "GodSpirit" then return "⛩️ Cúng Thần Linh" end
    if taskId == "TrainSkill" then return "⚔️ Auto Luyện Chiêu" end
    if taskId == "NormalFarm" then return "🎣 Treo Farm Thường" end
    return "💤 Đang Chờ (Idle)"
end

function PriorityManager.GetActiveTask(forceRefresh)
    local now = tick()
    if not forceRefresh and (now - (PriorityManager.lastTaskScan or 0) < 0.5) then
        return PriorityManager.cachedTask or "None"
    end
    PriorityManager.lastTaskScan = now

    local candidateTasks = {"SecretBoss", "TicketQuest", "GodSpirit", "TrainSkill", "NormalFarm"}
    if not Config.PrioritySystemEnabled then
        for _, tid in ipairs(candidateTasks) do
            if PriorityManager.IsTaskActive(tid) then
                PriorityManager.cachedTask = tid
                return tid
            end
        end
        PriorityManager.cachedTask = "None"
        return "None"
    end

    local bestTask = "None"
    local bestPriority = 9999

    for _, tid in ipairs(candidateTasks) do
        if PriorityManager.IsTaskActive(tid) then
            local p = PriorityManager.GetTaskPriority(tid)
            if p < bestPriority then
                bestPriority = p
                bestTask = tid
            end
        end
    end

    PriorityManager.cachedTask = bestTask
    return bestTask
end

function PriorityManager.UpdateUI()
    if PriorityManager.uiStatusRow and PriorityManager.uiStatusRow.Set then
        local cur = PriorityManager.GetActiveTask(true)
        local name = PriorityManager.GetTaskDisplayName(cur)
        local rank = cur ~= "None" and PriorityManager.GetTaskPriority(cur) or "-"
        PriorityManager.uiStatusRow.Set(string.format("%s (Hạng %s)", name, tostring(rank)))
    end
end


local tabFishing   = CreateTab("Câu Cá")
local tabBoss      = CreateTab("Săn Boss")
local tabFishManager = CreateTab("Quản Lý Cá")
local tabGod       = CreateTab("Thần Linh")
local tabQuests    = CreateTab("Nhiệm Vụ")
local tabShop      = CreateTab("Shop & Chế Mồi")
local tabTeleports = CreateTab("Dịch Chuyển")
local tabVisuals   = CreateTab("ESP & Đồ Hoạ")
local tabPlayer    = CreateTab("Nhân Vật")
local tabProfiles  = CreateTab("Cài Đặt")
local tabExperimental = CreateTab("Thử Nghiệm")

SwitchTab("Câu Cá")


createCategoryHeader(tabFishing, "Thông Tin Tài Khoản & Thống Kê")
local statsCard = createCardGroup(tabFishing)

local statsGridContainer = Instance.new("Frame")
statsGridContainer.Name = "StatsGridContainer"
statsGridContainer.Size = UDim2.new(1, 0, 0, 0)
statsGridContainer.AutomaticSize = Enum.AutomaticSize.Y
statsGridContainer.BackgroundTransparency = 1
statsGridContainer.BorderSizePixel = 0
statsGridContainer.Parent = statsCard

local gridPadding = Instance.new("UIPadding")
gridPadding.PaddingTop = UDim.new(0, 8)
gridPadding.PaddingBottom = UDim.new(0, 8)
gridPadding.PaddingLeft = UDim.new(0, 8)
gridPadding.PaddingRight = UDim.new(0, 8)
gridPadding.Parent = statsGridContainer

local gridLayout = Instance.new("UIGridLayout")
gridLayout.SortOrder = Enum.SortOrder.LayoutOrder
gridLayout.CellPadding = UDim2.new(0, 6, 0, 6)
gridLayout.CellSize = UDim2.new(1/3, -4, 0, 46)
gridLayout.Parent = statsGridContainer

local function createStatGridTile(parent, titleText, defaultValue, valueColor, layoutOrder)
    local tile = Instance.new("Frame")
    tile.BackgroundColor3 = Colors.ControlBg
    tile.BorderSizePixel = 0
    tile.LayoutOrder = layoutOrder or 1
    tile.Parent = parent

    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0, 5)
    c.Parent = tile

    local s = Instance.new("UIStroke")
    s.Color = Colors.BorderSubtle
    s.Thickness = 1
    s.Parent = tile

    local pad = Instance.new("UIPadding")
    pad.PaddingTop = UDim.new(0, 5)
    pad.PaddingBottom = UDim.new(0, 4)
    pad.PaddingLeft = UDim.new(0, 8)
    pad.PaddingRight = UDim.new(0, 8)
    pad.Parent = tile

    local titleLbl = Instance.new("TextLabel")
    titleLbl.Size = UDim2.new(1, 0, 0, 14)
    titleLbl.Position = UDim2.new(0, 0, 0, 0)
    titleLbl.BackgroundTransparency = 1
    titleLbl.Font = Enum.Font.Gotham
    titleLbl.Text = titleText
    titleLbl.TextColor3 = Colors.TextMuted
    titleLbl.TextSize = 10
    titleLbl.TextXAlignment = Enum.TextXAlignment.Left
    titleLbl.TextTruncate = Enum.TextTruncate.AtEnd
    titleLbl.Parent = tile

    local valLbl = Instance.new("TextLabel")
    valLbl.Size = UDim2.new(1, 0, 0, 18)
    valLbl.Position = UDim2.new(0, 0, 0, 15)
    valLbl.BackgroundTransparency = 1
    valLbl.Font = Enum.Font.GothamBold
    valLbl.Text = defaultValue or "---"
    valLbl.TextColor3 = valueColor or Colors.PurplePrimary
    valLbl.TextSize = 11
    valLbl.TextXAlignment = Enum.TextXAlignment.Left
    valLbl.TextTruncate = Enum.TextTruncate.AtEnd
    valLbl.Parent = tile

    tile.MouseEnter:Connect(function()
        TweenService:Create(s, TweenInfo.new(0.15), {Color = Colors.BorderPurple}):Play()
        TweenService:Create(tile, TweenInfo.new(0.15), {BackgroundColor3 = Colors.RowHover}):Play()
    end)
    tile.MouseLeave:Connect(function()
        TweenService:Create(s, TweenInfo.new(0.15), {Color = Colors.BorderSubtle}):Play()
        TweenService:Create(tile, TweenInfo.new(0.15), {BackgroundColor3 = Colors.ControlBg}):Play()
    end)

    return {
        frame = tile,
        Set = function(nv)
            valLbl.Text = tostring(nv or "")
        end
    }
end

-- Lưới 3 cột x 5 hàng (15 chỉ số hiển thị cực gọn, hiển thị đầy đủ map đang đứng & tài nguyên)
StatTiles.EquippedRod        = createStatGridTile(statsGridContainer, "🎣 Cần Đang Dùng", "Chưa có", Colors.TextWhite, 1)
StatTiles.EquippedBait       = createStatGridTile(statsGridContainer, "🪱 Mồi Đang Dùng", "Chưa có", Colors.TextWhite, 2)
local initLoc = GetCurrentLocationName()
StatTiles.CurrentLocation    = createStatGridTile(statsGridContainer, "📍 Map Đang Đứng", initLoc, Colors.AccentBlue, 3)

StatTiles.Uptime             = createStatGridTile(statsGridContainer, "⏳ Thời Gian Treo", "00:00:00", Colors.AccentYellow, 4)
StatTiles.FishCaught         = createStatGridTile(statsGridContainer, "🐟 Tổng Cá Đã Câu", "0 con", Colors.PurplePrimary, 5)
StatTiles.FishPerHour        = createStatGridTile(statsGridContainer, "⚡ Tốc Độ Câu", "0 con/h", Colors.AccentGreen, 6)

StatTiles.Cash               = createStatGridTile(statsGridContainer, "💰 Tiền Hiện Tại", "$0", Colors.AccentGreen, 7)
StatTiles.CashPerHour        = createStatGridTile(statsGridContainer, "📈 Tốc Độ Tiền", "$0 /h", Colors.AccentGreen, 8)
StatTiles.GemsGained         = createStatGridTile(statsGridContainer, "💎 Gems Hiện Có", "0 Gems", Colors.AccentBlue, 9)

StatTiles.Tickets            = createStatGridTile(statsGridContainer, "🎫 Vé Nhiệm Vụ", "0 Vé", Colors.AccentOrange, 10)
StatTiles.EssenceOrbs        = createStatGridTile(statsGridContainer, "🔮 Essence Orb", "0 Viên", Colors.PurpleAccent, 11)
StatTiles.TraitRerolls       = createStatGridTile(statsGridContainer, "🎲 Trait Reroll", "0 Vé", Colors.AccentYellow, 12)

StatTiles.TicketQuestsToday  = createStatGridTile(statsGridContainer, "📜 Vé Xong Hôm Nay", "0 NV", Colors.AccentOrange, 13)
local curFishCount, curFishLimit = GetCurrentBackpackFishCount()
StatTiles.Backpack           = createStatGridTile(statsGridContainer, "🎒 Cá Trong Balo", string.format("%d / %d", curFishCount, curFishLimit), Colors.TextWhite, 14)
StatTiles.TicketCooldown     = createStatGridTile(statsGridContainer, "⏳ Chờ Vé Mới", "Sẵn sàng", Colors.AccentYellow, 15)

-- Nút Reset Thông Số Treo Máy
createButtonRow(statsCard, "Đặt Lại Thông Số Treo (Reset AFK)", "Đặt lại giờ treo và tính lại tốc độ cá/tiền chính xác từ mốc này", "🔄 Reset Thông Số", function()
    local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
    local curFish = pData and pData:FindFirstChild("FishCaught") and tonumber(pData.FishCaught.Value) or 0
    local curCash = pData and pData:FindFirstChild("Cash") and tonumber(pData.Cash.Value) or 0

    sessionStartTime = tick()
    initialFishCaught = curFish
    initialCash = curCash
    gemTracker.gained = 0
    lastWebhookStatsTime = tick()

    if StatTiles.Uptime and StatTiles.Uptime.Set then StatTiles.Uptime.Set("00:00:00") end
    if StatTiles.FishPerHour and StatTiles.FishPerHour.Set then StatTiles.FishPerHour.Set("0 con/h") end
    if StatTiles.CashPerHour and StatTiles.CashPerHour.Set then StatTiles.CashPerHour.Set("$0 /h") end
    if StatTiles.GemsGained and StatTiles.GemsGained.Set then
        if visualSpoofState and visualSpoofState.fakeGems and visualSpoofState.fakeGems > 0 then
            StatTiles.GemsGained.Set(FormatWithSpaces(visualSpoofState.fakeGems) .. " Gems")
        else
            local realG = (GetPlayerCurrentGems and GetPlayerCurrentGems()) or 0
            StatTiles.GemsGained.Set(FormatWithSpaces(realG) .. " Gems")
        end
    end

    ShowNotification("Thống Kê Treo", "Đã đặt lại mốc thời gian và tính lại tốc độ câu/tiền từ thời điểm này!", "SUCCESS", 4)
end)

createCategoryHeader(tabFishing, "Tự Động Câu Cá Cốt Lõi")
local fishCard = createCardGroup(tabFishing)

createToggleRow(fishCard, "Tự Động Chơi Mini Game (Auto Minigame)", "Tự động thắng mọi minigame (Kéo cần, Perfect Slam, Max Charge & Rhythm Boss Bạch Tuộc)", Config.AutoMinigame, function(v)
    Config.AutoMinigame = v
    Config.AnchorBar = v
    Config.AutoSlam = v
    Config.AutoCharge = v
    Config.OctoAutoMinigame = v
end)
createSliderRow(fishCard, "Tỉ Lệ Trúng Minigame (Accuracy %)", "Độ chính xác khi gõ nhịp Boss (Khuyên dùng 92% - 96% để an toàn chống soi anti-cheat)", 70, 100, Config.RhythmAccuracy or 95, false, "%", function(v)
    Config.RhythmAccuracy = v
end)
createToggleRow(fishCard, "Chế Độ Người Thật (Humanizer Timing)", "Giả lập độ trễ phản xạ 10-35ms và vị trí bấm lệch ngẫu nhiên như tay người thật", Config.RhythmHumanizer, function(v)
    Config.RhythmHumanizer = v
end)
createButtonRow(fishCard, "🧪 Test Thử Minigame Bạch Tuộc (A-S-D)", "Bật giao diện 3 làn A-S-D và thả nốt rơi thử nghiệm để kiểm chứng bot tự bấm Perfect ngay trước mắt", "Bấm Để Test", function()
    task.spawn(function()
        local pg = LocalPlayer:FindFirstChild("PlayerGui")
        local mainGui = pg and pg:FindFirstChild("MainGui")
        local fishing = mainGui and mainGui:FindFirstChild("Fishing")
        local rhythm = (fishing and fishing:FindFirstChild("Rhythm")) or (mainGui and mainGui:FindFirstChild("Rhythm")) or (pg and pg:FindFirstChild("Rhythm", true))
        if not rhythm then
            ShowNotification("Lỗi Test", "Không tìm thấy giao diện Rhythm trong game!", "WARN", 4)
            return
        end
        local origFVis = fishing and fishing.Visible
        local origRVis = rhythm.Visible
        if fishing then fishing.Visible = true end
        rhythm.Visible = true
        ShowNotification("🧪 Đang Test", "Giao diện A-S-D đã mở! Đang tạo nốt rơi thử nghiệm...", 3)

        local TweenService = game:GetService("TweenService")
        local vim = game:GetService("VirtualInputManager")
        local lanes = {
            { name = "ProgressionA", key = "A", keyCode = Enum.KeyCode.A },
            { name = "ProgressionS", key = "S", keyCode = Enum.KeyCode.S },
            { name = "ProgressionD", key = "D", keyCode = Enum.KeyCode.D }
        }

        for _, laneData in ipairs(lanes) do
            local prog = rhythm:FindFirstChild(laneData.name)
            if prog and prog.Visible then
                local bFrame = prog:FindFirstChild("BarFrame")
                local btn = prog:FindFirstChild("Button")
                local nFrame = prog:FindFirstChild("NoteFrame")
                local expImg = prog:FindFirstChild("EXP")

                local testNote = Instance.new("Frame")
                testNote.Name = "TestNote_" .. laneData.key
                testNote.Size = (nFrame and nFrame.Size) or UDim2.new(0.8, 0, 0, 24)
                testNote.Position = UDim2.new(0.1, 0, 0, 0)
                testNote.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                testNote.BorderSizePixel = 0
                testNote.ZIndex = 100
                testNote.Parent = prog
                Instance.new("UICorner", testNote).CornerRadius = UDim.new(0, 4)

                local targetPos = (bFrame and bFrame.Position) or UDim2.new(0.1, 0, 0.8, 0)
                local tween = TweenService:Create(testNote, TweenInfo.new(1.1, Enum.EasingStyle.Linear), { Position = targetPos })
                tween:Play()

                task.delay(0.9, function()
                    if testNote and testNote.Parent then
                        if vim and laneData.keyCode then
                            pcall(function()
                                vim:SendKeyEvent(true, laneData.keyCode, false, game)
                                task.delay(0.02, function() vim:SendKeyEvent(false, laneData.keyCode, false, game) end)
                            end)
                        end
                        if vim and btn and btn:IsA("GuiButton") then
                            pcall(function()
                                local bp = btn.AbsolutePosition
                                local bs = btn.AbsoluteSize
                                vim:SendMouseButtonEvent(bp.X + bs.X * 0.5, bp.Y + bs.Y * 0.5, 0, true, game, 1)
                                task.delay(0.02, function() vim:SendMouseButtonEvent(bp.X + bs.X * 0.5, bp.Y + bs.Y * 0.5, 0, false, game, 1) end)
                            end)
                        end
                        if btn and btn:IsA("GuiButton") then
                            pcall(function()
                                if firesignal then
                                    if btn.Activated then firesignal(btn.Activated) end
                                    if btn.MouseButton1Down then firesignal(btn.MouseButton1Down) end
                                    if btn.MouseButton1Click then firesignal(btn.MouseButton1Click) end
                                end
                            end)
                        end
                        if expImg and expImg:IsA("ImageLabel") then
                            pcall(function()
                                expImg.Visible = true
                                task.delay(0.2, function() expImg.Visible = false end)
                            end)
                        end
                        testNote.BackgroundColor3 = Color3.fromRGB(80, 255, 120)
                        ShowNotification("✨ PERFECT!", "Bot đã tự động bấm nốt làn [" .. laneData.key .. "]!", 2)
                        task.delay(0.3, function() if testNote then testNote:Destroy() end end)
                    end
                end)
                task.wait(0.5)
            end
        end

        task.wait(2.2)
        if fishing then fishing.Visible = origFVis end
        rhythm.Visible = origRVis
        ShowNotification("🎉 Test Hoàn Tất", "Đã kiểm chứng bot tự bấm nốt thành công 100%!", 4)
    end)
end)
createToggleRow(fishCard, "Tự Động Quăng Cần (Auto Cast)", "Tự động bắt đầu câu và quăng cần liên tục", Config.AutoCast, function(v) Config.AutoCast = v end)
createSliderRow(fishCard, "Độ Trễ Quăng Cần", "Thời gian giãn cách giữa các lần quăng", 0.0, 5.0, Config.CastDelay, true, "s", function(v) Config.CastDelay = v end)
createToggleRow(fishCard, "Giữ Thanh Minigame (Anchor Bar)", "Tự động giữ thanh kéo ở giữa để bắt cá 100%", Config.AnchorBar, function(v) Config.AnchorBar = v end)
createToggleRow(fishCard, "Tự Dùng Kỹ Năng Cần", "Tự kích hoạt kỹ năng cần câu để kéo cá siêu nhanh", Config.AutoSkills, function(v) Config.AutoSkills = v end)
createToggleRow(fishCard, "Tự Động Đập Cần (Auto Slam)", "Tự động nhấn Slam mức Perfect khi xuất hiện", Config.AutoSlam, function(v) Config.AutoSlam = v end)
createToggleRow(fishCard, "Tự Động Sạc Dây (Auto Charge)", "Tự động sạc đầy 100% độ bền dây câu", Config.AutoCharge, function(v) Config.AutoCharge = v end)
createToggleRow(fishCard, "Tự Động Chống Kẹt Cần (Anti-Stuck)", "Tự động phát hiện và gỡ kẹt khi quăng cần hoặc minigame bị đơ quá 15s", Config.AntiStuckEnabled, function(v) Config.AntiStuckEnabled = v end)

createCategoryHeader(tabFishing, "🏠 Vị Trí Trở Về Nếu Săn Boss (Home Spot)")
local returnSpotCard = createCardGroup(tabFishing)

local function GetHomeSpotText()
    if Config.HomeFarmSpot then
        local p = Config.HomeFarmSpot
        return string.format("Đã lưu: X:%.1f, Y:%.1f, Z:%.1f (%s)", p.x or 0, p.y or 0, p.z or 0, p.savedAt or "Đã lưu")
    end
    return "Chưa thiết lập (Bấm nút bên dưới để lưu vị trí đang đứng)"
end

local infoHomeSpot = createInfoRow(returnSpotCard, "Vị Trí Trở Về Hiện Tại", GetHomeSpotText())

createToggleRow(returnSpotCard, "Tự Về Vị Trí Này Khi Hết Boss / Clear", "Khi hết Boss hoặc thời tiết Clear, tự bay về vị trí này câu cá farm tiền", Config.ReturnToHomeWhenClear, function(v)
    Config.ReturnToHomeWhenClear = v
end)

createToggleRow(returnSpotCard, "Tự Về Vị Trí Này Khi Xong Vé NV", "Khi xong vé nhiệm vụ và vào 20p chờ, tự bay về vị trí này", Config.TicketReturnHomeWhenDone, function(v)
    Config.TicketReturnHomeWhenDone = v
end)

createToggleRow(returnSpotCard, "Tự Quăng Cần & Đánh Combo Khi Về Điểm Này", "Tự quăng cần và dùng Combo đã cài khi đang chờ ở Home Spot (không cần bật Tự Quăng Cần tổng)", Config.TicketAutoCastAtHome, function(v)
    Config.TicketAutoCastAtHome = v
end)

createButtonRow(returnSpotCard, "Lưu Vị Trí Đang Đứng Làm Điểm Trở Về", "Lưu tọa độ & hướng quay hiện tại làm nơi Farm cá mặc định (lưu riêng theo tài khoản)", "Lưu Vị Trí", function()
    local ok, spot = secretBossState.SaveHomeSpot()
    if ok then
        if infoHomeSpot and infoHomeSpot.Set then
            infoHomeSpot.Set(GetHomeSpotText())
        end
        ShowNotification("Vị Trí Trở Về", "Đã lưu vị trí trở về thành công cho tài khoản này!", "SUCCESS", 5)
    else
        ShowNotification("Lỗi Lưu", tostring(spot), "ERROR")
    end
end)

createButtonRow(returnSpotCard, "Bay Về Điểm Trở Về Ngay", "Dịch chuyển tức thì về vị trí trở về đã lưu và bắt đầu câu", "Bay Về", function()
    if not Config.HomeFarmSpot then
        ShowNotification("Chưa Lưu Vị Trí", "Vui lòng đứng tại nơi muốn câu rồi bấm [Lưu Vị Trí] trước!", "WARN", 5)
        return
    end
    secretBossState.lastHomeReturnTime = 0
    local ok = secretBossState.ReturnToHome()
    if not ok then
        local char = LocalPlayer.Character
        local root = char and char:FindFirstChild("HumanoidRootPart")
        if root and Config.HomeFarmSpot.cframe then
            root.CFrame = CFrame.new(table.unpack(Config.HomeFarmSpot.cframe))
            CancelAndRecastRod()
        end
    end
    ShowNotification("Vị Trí Trở Về", "Đã bay về vị trí trở về mặc định!", "SUCCESS", 4)
end)

createButtonRow(returnSpotCard, "Xóa Điểm Trở Về Đã Lưu", "Xóa vị trí trở về của tài khoản này", "Xóa Vị Trí", function()
    secretBossState.ClearHomeSpot()
    if infoHomeSpot and infoHomeSpot.Set then
        infoHomeSpot.Set(GetHomeSpotText())
    end
    ShowNotification("Vị Trí Trở Về", "Đã xóa vị trí trở về của tài khoản này.", "INFO")
end)

createCategoryHeader(tabFishing, "⚔️ Combo Kỹ Năng Thông Minh (Smart Combos)")
local comboCard = createCardGroup(tabFishing)

createToggleRow(comboCard, "Bật Combo Kỹ Năng Tự Động", "Tự động kích hoạt chiêu theo ngưỡng máu cá, chiêu mở màn và đảo chiêu luân phiên", Config.SmartComboEnabled, function(v)
    Config.SmartComboEnabled = v
    SaveSmartCombo()
end)

createSliderRow(comboCard, "Ngưỡng Máu Cá Phân Loại", "Máu cá <= mức này sẽ kết liễu nhanh; > mức này sẽ bật combo", 100, 3000, Config.FishHpThreshold, false, " HP", function(v)
    Config.FishHpThreshold = v
    SaveSmartCombo()
end)

createDropdownRow(comboCard, "Chiêu Bắt Nhanh (<= Ngưỡng HP)", "Tung 1 hit kết liễu ngay khi cá yếu / cá thường", {"Tắt", "Z", "X", "C", "V"}, Config.QuickCatchSkill, function(v)
    Config.QuickCatchSkill = v
    SaveSmartCombo()
end)

createDropdownRow(comboCard, "Chiêu Mở Màn (> Ngưỡng HP)", "Chiêu tung 1 lần duy nhất đầu trận khi gặp cá to / boss", {"Tắt", "Z", "X", "C", "V"}, Config.OpenerSkill, function(v)
    Config.OpenerSkill = v
    SaveSmartCombo()
end)

createSliderRow(comboCard, "Số Lần Dùng Chiêu Mở Màn", "Số lần tung chiêu mở màn trước khi chuyển sang đảo chiêu", 1, 3, Config.OpenerMaxCount, false, " lần", function(v)
    Config.OpenerMaxCount = v
    SaveSmartCombo()
end)

do
    local loopPresets = {
        "Tùy Biến (Tự Do)",
        "X, C (Cơ Bản 2 Chiêu)",
        "X, V (Khống Chế)",
        "C, V (Bộ Chiêu Cuối)",
        "X, C, V (Dồn Sát Thương 3 Chiêu)",
        "Z, X, C, V (Chuỗi Đầy Đủ 4 Chiêu)",
        "X, C, X, V (Nhịp Kép X)",
        "Z, X, Z, C (Tối Ưu Cooldown Z)",
        "X, X, C (Nhồi Liên Hoàn X)",
        "C, X, C, V (Nhịp Kép C)",
        "Z, C, V (Hỗ Trợ & Sát Thương)"
    }

    local function getPresetValue(label)
        if not label or label == "Tùy Biến (Tự Do)" then return nil end
        local clean = label:match("^([^%(]+)")
        if clean then
            local keys = {}
            for k in string.gmatch(clean, "([ZXCVzxcv])") do
                table.insert(keys, k:upper())
            end
            if #keys > 0 then
                return table.concat(keys, ", ")
            end
        end
        return nil
    end

    local function getPresetLabel(val)
        local formatted = comboState.FormatCombo(val)
        for _, l in ipairs(loopPresets) do
            local pVal = getPresetValue(l)
            if pVal and pVal == formatted then
                return l
            end
        end
        return "Tùy Biến (Tự Do)"
    end

    local isSyncing = false
    local customInput = nil
    local presetDropdown = nil
    local infoPreview = nil

    local function applyComboChange(newVal, source)
        if isSyncing then return end
        isSyncing = true

        local cleanStr = comboState.FormatCombo(newVal)
        Config.LoopSkills = cleanStr
        -- Lưu chuỗi combo mới vào local mỗi khi thay đổi
        if source ~= "__load_sync" then
            SaveSmartCombo()
        end

        if source ~= "input" and customInput and customInput.Set then
            customInput.Set(cleanStr)
        end

        if source ~= "dropdown" and presetDropdown and presetDropdown.Set then
            local targetLabel = getPresetLabel(cleanStr)
            presetDropdown.Set(targetLabel)
        end

        if infoPreview and infoPreview.Set then
            infoPreview.Set(comboState.GetComboPreview(cleanStr))
        end

        isSyncing = false
    end

    presetDropdown = createDropdownRow(comboCard, "Mẫu Chuỗi Chiêu (Preset)", "Chọn nhanh chuỗi phổ biến hoặc tự do tùy biến ở dưới", loopPresets, getPresetLabel(Config.LoopSkills), function(v)
        local pVal = getPresetValue(v)
        if pVal then
            applyComboChange(pVal, "dropdown")
        end
    end)

    customInput = createInputRow(comboCard, "Tùy Biến Chuỗi Đảo Chiêu", "Gõ bất kỳ chiêu nào (VD: Z, X, V hoặc Z, X, C, V...)", Config.LoopSkills or "Z, X, V", function(v)
        applyComboChange(v, "input")
    end, nil, "VD: Z, X, V")

    -- Bàn phím tạo combo nhanh 1 chạm
    local quickRow = createBaseRow(comboCard, "Bộ Phím Ghép Combo Nhanh", "Chạm các nút để thêm hoặc xóa nhanh chiêu vào chuỗi combo")
    local btnContainer = Instance.new("Frame")
    btnContainer.Size = UDim2.new(0, 190, 0, 24)
    btnContainer.Position = UDim2.new(1, -190, 0.5, -12)
    btnContainer.BackgroundTransparency = 1
    btnContainer.Parent = quickRow

    local listLayout = Instance.new("UIListLayout")
    listLayout.FillDirection = Enum.FillDirection.Horizontal
    listLayout.HorizontalAlignment = Enum.HorizontalAlignment.Right
    listLayout.SortOrder = Enum.SortOrder.LayoutOrder
    listLayout.Padding = UDim.new(0, 3)
    listLayout.Parent = btnContainer

    local function makeQuickBtn(text, bgColor, textColor, onClick)
        local btn = Instance.new("TextButton")
        btn.Size = UDim2.new(0, 28, 0, 24)
        btn.BackgroundColor3 = bgColor
        btn.Font = Enum.Font.GothamBold
        btn.Text = text
        btn.TextColor3 = textColor
        btn.TextSize = 11
        btn.BorderSizePixel = 0
        btn.Parent = btnContainer
        Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 4)
        btn.MouseButton1Click:Connect(function()
            pcall(onClick)
        end)
        return btn
    end

    local function appendKey(k)
        local current = Config.LoopSkills or ""
        local keys = {}
        for char in string.gmatch(current, "([ZXCVzxcv])") do
            table.insert(keys, char:upper())
        end
        table.insert(keys, k:upper())
        applyComboChange(table.concat(keys, ", "), "quickbtn")
    end

    makeQuickBtn("+Z", Colors.ControlBg, Colors.PurplePrimary, function() appendKey("Z") end)
    makeQuickBtn("+X", Colors.ControlBg, Colors.PurplePrimary, function() appendKey("X") end)
    makeQuickBtn("+C", Colors.ControlBg, Colors.PurplePrimary, function() appendKey("C") end)
    makeQuickBtn("+V", Colors.ControlBg, Colors.PurplePrimary, function() appendKey("V") end)
    makeQuickBtn("⌫", Color3.fromRGB(45, 25, 30), Color3.fromRGB(255, 120, 120), function()
        local current = Config.LoopSkills or ""
        local keys = {}
        for char in string.gmatch(current, "([ZXCVzxcv])") do
            table.insert(keys, char:upper())
        end
        if #keys > 0 then
            table.remove(keys, #keys)
        end
        applyComboChange(table.concat(keys, ", "), "quickbtn")
    end)
    makeQuickBtn("🗑", Color3.fromRGB(35, 35, 40), Colors.TextMuted, function()
        applyComboChange("", "quickbtn")
    end)

    infoPreview = createInfoRow(comboCard, "Thứ Tự Thi Triển Thực Tế", comboState.GetComboPreview(Config.LoopSkills))

    createToggleRow(comboCard, "Giữ Đúng Thứ Tự Combo (Strict Order)", "Chờ chiêu hồi theo đúng nhịp thứ tự, không nhảy cóc qua chiêu khác", Config.LoopStrictOrder, function(v)
        Config.LoopStrictOrder = v
        SaveSmartCombo()
    end)
end

createDropdownRow(comboCard, "Chiêu Hồi Máu / Cứu Nguy", "Ưu tiên tung chiêu này khi máu người chơi xuống thấp", {"Tắt", "Z", "X", "C", "V"}, Config.EmergencyHealSkill, function(v)
    Config.EmergencyHealSkill = v
    SaveSmartCombo()
end)

createSliderRow(comboCard, "Kích Hoạt Hồi Máu Khi HP Dưới", "Ngưỡng máu người chơi cần cứu nguy khẩn cấp", 10, 80, Config.EmergencyHealHp, false, "%", function(v)
    Config.EmergencyHealHp = v
    SaveSmartCombo()
end)

createSliderRow(comboCard, "Thời Gian Chờ Ra Chiêu", "Thời gian tối thiểu chờ hết hiệu ứng trước khi tung chiêu tiếp theo", 0.1, 2.5, Config.SkillEffectDelay, true, "s", function(v)
    Config.SkillEffectDelay = v
    SaveSmartCombo()
end)

createToggleRow(comboCard, "Tự Động Nhận Diện Hết Hiệu Ứng", "Quan sát hoạt ảnh đòn đánh trên nhân vật để chống nuốt chiêu 100%", Config.SmartEffectAutoDetect, function(v)
    Config.SmartEffectAutoDetect = v
    SaveSmartCombo()
end)

createCategoryHeader(tabFishing, "🎯 Auto Luyện Chiêu Nhanh (Fast Cancel)")
local trainCard = createCardGroup(tabFishing)
local infoTrainProgress = createInfoRow(trainCard, "Tiến Độ Luyện Chiêu", string.format("%d / %d lần", tonumber(Config.TrainCurrentCount) or 0, tonumber(Config.TrainTargetCount) or 100))
createToggleRow(trainCard, "Bật Auto Luyện Chiêu", "Cá cắn kéo là dùng chiêu -> cất cần hủy cá -> thả cần lại ngay", Config.AutoTrainSkill, function(v) Config.AutoTrainSkill = v end)
createDropdownRow(trainCard, "Chọn Chiêu Cần Luyện", "Chọn 1 chiêu duy nhất muốn luyện (Z, X, C, V)", {"Z", "X", "C", "V"}, Config.TrainSkill or "Z", function(v) Config.TrainSkill = v end)
createSliderRow(trainCard, "Nhịp Chờ Xuất Chiêu (Cancel Delay)", "Thời gian chờ nhân vật bắt đầu xuất chiêu trước khi cất cần (0.2s - 1.2s)", 0.2, 1.2, Config.TrainCancelDelay or 0.45, true, "s", function(v)
    Config.TrainCancelDelay = v
end)
createSliderRow(trainCard, "Mục Tiêu Số Lần Dùng", "Số lần cần dùng để đạt yêu cầu tiến hóa (mặc định 100 lần)", 10, 500, Config.TrainTargetCount, false, " lần", function(v)
    Config.TrainTargetCount = v
    if infoTrainProgress and infoTrainProgress.Set then
        infoTrainProgress.Set(string.format("%d / %d lần", tonumber(Config.TrainCurrentCount) or 0, tonumber(Config.TrainTargetCount) or 100))
    end
end)

local lastExportedSkillText = ""
local skillViewerModal = nil

local function ShowSkillTextWindow(customText)
    local text = customText or lastExportedSkillText
    if not text or text == "" then
        text = "Chưa có dữ liệu kỹ năng!\nVui lòng bấm nút [📋 Quét & Copy Tất Cả] để hệ thống trích xuất toàn bộ dữ liệu."
    end

    if skillViewerModal and skillViewerModal.Parent then
        skillViewerModal:Destroy()
        skillViewerModal = nil
    end

    local modal = Instance.new("Frame")
    modal.Name = "SkillViewerModal"
    modal.Size = UDim2.new(0, 560, 0, 440)
    modal.Position = UDim2.new(0.5, -280, 0.5, -220)
    modal.BackgroundColor3 = Colors.Background or Color3.fromRGB(15, 17, 24)
    modal.BorderSizePixel = 0
    modal.ZIndex = 250
    modal.Parent = screenGui
    skillViewerModal = modal
    table.insert(cleanUpInstances, modal)

    local stroke = Instance.new("UIStroke")
    stroke.Color = Colors.PurpleAccent or Color3.fromRGB(130, 80, 240)
    stroke.Thickness = 1.5
    stroke.Parent = modal

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 10)
    corner.Parent = modal

    local tBar = Instance.new("Frame")
    tBar.Size = UDim2.new(1, 0, 0, 40)
    tBar.BackgroundColor3 = Colors.SidebarBg or Color3.fromRGB(20, 22, 32)
    tBar.BorderSizePixel = 0
    tBar.ZIndex = 251
    tBar.Parent = modal
    local tCorner = Instance.new("UICorner"); tCorner.CornerRadius = UDim.new(0, 10); tCorner.Parent = tBar

    local tLabel = Instance.new("TextLabel")
    tLabel.Size = UDim2.new(1, -160, 1, 0)
    tLabel.Position = UDim2.new(0, 14, 0, 0)
    tLabel.BackgroundTransparency = 1
    tLabel.Font = Enum.Font.GothamBold
    tLabel.Text = "📜 DANH SÁCH TOÀN BỘ KỸ NĂNG CỦA BẠN"
    tLabel.TextColor3 = Colors.PurplePrimary or Color3.fromRGB(200, 180, 255)
    tLabel.TextSize = 13
    tLabel.TextXAlignment = Enum.TextXAlignment.Left
    tLabel.ZIndex = 252
    tLabel.Parent = tBar

    local btnCopy = Instance.new("TextButton")
    btnCopy.Size = UDim2.new(0, 95, 0, 26)
    btnCopy.Position = UDim2.new(1, -145, 0.5, -13)
    btnCopy.BackgroundColor3 = Colors.PurpleAccent or Color3.fromRGB(110, 70, 220)
    btnCopy.Font = Enum.Font.GothamBold
    btnCopy.Text = "📋 Sao Chép"
    btnCopy.TextColor3 = Color3.fromRGB(255, 255, 255)
    btnCopy.TextSize = 11
    btnCopy.ZIndex = 252
    btnCopy.Parent = tBar
    local cCorner = Instance.new("UICorner"); cCorner.CornerRadius = UDim.new(0, 5); cCorner.Parent = btnCopy

    btnCopy.MouseButton1Click:Connect(function()
        pcall(function()
            if setclipboard then setclipboard(text)
            elseif toclipboard then toclipboard(text) end
        end)
        btnCopy.Text = "✅ Đã Chép!"
        btnCopy.BackgroundColor3 = Colors.Green or Color3.fromRGB(50, 190, 100)
        task.delay(1.5, function()
            if btnCopy and btnCopy.Parent then
                btnCopy.Text = "📋 Sao Chép"
                btnCopy.BackgroundColor3 = Colors.PurpleAccent or Color3.fromRGB(110, 70, 220)
            end
        end)
    end)

    local btnClose = Instance.new("TextButton")
    btnClose.Size = UDim2.new(0, 32, 0, 26)
    btnClose.Position = UDim2.new(1, -42, 0.5, -13)
    btnClose.BackgroundColor3 = Color3.fromRGB(45, 45, 60)
    btnClose.Font = Enum.Font.GothamBold
    btnClose.Text = "✕"
    btnClose.TextColor3 = Color3.fromRGB(220, 220, 220)
    btnClose.TextSize = 12
    btnClose.ZIndex = 252
    btnClose.Parent = tBar
    local clCorner = Instance.new("UICorner"); clCorner.CornerRadius = UDim.new(0, 5); clCorner.Parent = btnClose
    btnClose.MouseButton1Click:Connect(function()
        modal:Destroy()
        skillViewerModal = nil
    end)

    pcall(function()
        local dragging, dragStart, startPos
        tBar.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                dragging = true
                dragStart = input.Position
                startPos = modal.Position
            end
        end)
        tBar.InputEnded:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                dragging = false
            end
        end)
        UserInputService.InputChanged:Connect(function(input)
            if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
                local delta = input.Position - dragStart
                modal.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
            end
        end)
    end)

    local scroll = Instance.new("ScrollingFrame")
    scroll.Size = UDim2.new(1, -20, 1, -55)
    scroll.Position = UDim2.new(0, 10, 0, 46)
    scroll.BackgroundColor3 = Color3.fromRGB(10, 12, 18)
    scroll.BorderSizePixel = 0
    scroll.ScrollBarThickness = 6
    scroll.ScrollBarImageColor3 = Colors.PurpleAccent or Color3.fromRGB(120, 80, 220)
    scroll.ZIndex = 251
    scroll.Parent = modal
    local sCorner = Instance.new("UICorner"); sCorner.CornerRadius = UDim.new(0, 6); sCorner.Parent = scroll

    local textBox = Instance.new("TextBox")
    textBox.Size = UDim2.new(1, -12, 1, 0)
    textBox.Position = UDim2.new(0, 8, 0, 8)
    textBox.BackgroundTransparency = 1
    textBox.Font = Enum.Font.RobotoMono
    textBox.TextSize = 12
    textBox.TextColor3 = Color3.fromRGB(235, 235, 245)
    textBox.TextXAlignment = Enum.TextXAlignment.Left
    textBox.TextYAlignment = Enum.TextYAlignment.Top
    textBox.ClearTextOnFocus = false
    textBox.TextEditable = false
    textBox.MultiLine = true
    textBox.Text = text
    textBox.ZIndex = 252
    textBox.Parent = scroll

    local _, lineCount = text:gsub("\n", "\n")
    local estimatedHeight = math.max(400, (lineCount + 5) * 17)
    scroll.CanvasSize = UDim2.new(0, 0, 0, estimatedHeight)
    textBox.Size = UDim2.new(1, -16, 0, estimatedHeight)
end

local builtInSkills = {
    ["Ruinous Sacrifice"] = {
        Damage = "143",
        Cooldown = "999s",
        Type = "Black Tortoise",
        Description = "Immune to damage from boss skills. Boosts rod Power by +999 for 10s, dealing current damage per second and drains 20 HP per second."
    },
    ["Heaven's Burden"] = {
        Damage = "120",
        Cooldown = "18s",
        Type = "White Tiger",
        Description = "Immune to damage from boss skills. Stuns the fish for 10 seconds, heals 25 HP, and deals current damage every second."
    },
    ["Vajra Godcast"] = {
        Damage = "135",
        Cooldown = "16s",
        Type = "Vermillion Bird",
        Description = "Immune to damage from boss skills. Deals current damage, stuns the fish for 7s, and triggers a follow-up strike for 50% damage after 5s."
    },
    ["Shangqing Realm"] = {
        Damage = "130",
        Cooldown = "20s",
        Type = "Black Tortoise",
        Description = "Immune to boss skills, bypasses boss invincibility, deals heavy continuous damage and stuns the fish."
    },
    ["Whirlwind Fishing Art"] = {
        Damage = "100",
        Cooldown = "15s",
        Type = "Azure Dragon",
        Description = "Grants immunity to boss skills, deals damage, and stuns the fish for 12 seconds."
    },
    ["Phoenix Strike Art"] = {
        Damage = "115",
        Cooldown = "12s",
        Type = "Vermillion Bird",
        Description = "Unleashes blazing phoenix flames that deal high burst damage and burn the fish over time."
    },
    ["Tiger's Hunt"] = {
        Damage = "80",
        Cooldown = "15s",
        Type = "White Tiger",
        Description = "Increases rod Power by +30 for 15s while dealing continuous damage per second to the fish."
    },
    ["Leopard Lock"] = {
        Damage = "70",
        Cooldown = "12s",
        Type = "White Tiger",
        Description = "Stuns the fish for 10s and deals continuous damage over time."
    },
    ["Iron Grip Art"] = {
        Damage = "65",
        Cooldown = "10s",
        Type = "Black Tortoise",
        Description = "Stuns the fish for 10s and stabilizes line tension while dealing continuous damage."
    },
    ["Dual Sonic Kick"] = {
        Damage = "85",
        Cooldown = "10s",
        Type = "Chrono",
        Description = "Delivers a rapid flurry of sonic kicks, dealing current damage per second for up to 10 seconds."
    },
    ["Dragon-Fish"] = {
        Damage = "110",
        Cooldown = "14s",
        Type = "Azure Dragon",
        Description = "Summons a roaring celestial dragon fish to strike, increasing rod power and dealing heavy impact damage."
    },
    ["Dragon Strike"] = {
        Damage = "110",
        Cooldown = "14s",
        Type = "Azure Dragon",
        Description = "Calls upon the fury of the Azure Dragon to crash down upon the fish with tremendous power."
    },
    ["One-Strike Heaven Gate"] = {
        Damage = "105",
        Cooldown = "10s",
        Type = "Azure Dragon",
        Description = "Concentrates immense celestial energy into a single strike to shatter the fish's stamina instantly."
    },
    ["Rolling Chaos"] = {
        Damage = "95",
        Cooldown = "11s",
        Type = "Black Tortoise",
        Description = "Creates chaotic swirling ripples, disorienting the fish and reducing its escape speed while dealing continuous damage."
    },
    ["Taijiquan Technique"] = {
        Damage = "90",
        Cooldown = "9s",
        Type = "Black Tortoise",
        Description = "Stuns the fish for 10s, balances internal qi to absorb fish movements, and deals steady damage."
    },
    ["Astral Grand Art"] = {
        Damage = "130",
        Cooldown = "15s",
        Type = "Azure Dragon",
        Description = "Calls down astral starlight to barrage the target fish with cosmic damage and grant temporary rod power."
    },
    ["Infinite Sky Ascension"] = {
        Damage = "85",
        Cooldown = "10s",
        Type = "White Tiger",
        Description = "Propels your fishing hook skyward, elevating line tension and pulling the fish closer with great momentum."
    },
    ["Sever the Gate"] = {
        Damage = "80",
        Cooldown = "8s",
        Type = "White Tiger",
        Description = "Delivers a sharp cleaving cut through the water, interrupting the fish's struggle."
    },
    ["Skyfall Stomp"] = {
        Damage = "75",
        Cooldown = "8s",
        Type = "White Tiger",
        Description = "Stomps the surface with earth-shattering force, stunning the fish for 2s."
    },
    ["Beastbreaker Cleave"] = {
        Damage = "70",
        Cooldown = "7s",
        Type = "White Tiger",
        Description = "A heavy cleave designed to break through the armor and thick scales of massive sea beasts."
    },
    ["Demonfall Technique"] = {
        Damage = "85",
        Cooldown = "9s",
        Type = "Vermillion Bird",
        Description = "Dark demonic slash that saps the fish's health rapidly."
    },
    ["Swift Reel"] = {
        Damage = "60",
        Cooldown = "6s",
        Type = "Chrono",
        Description = "Spins the reel at hyper-speed, rapidly pulling in the line and increasing progression speed."
    },
    ["Reel Machine"] = {
        Damage = "65",
        Cooldown = "7s",
        Type = "Chrono",
        Description = "Automates mechanical reeler gearings to maintain constant pull tension on the fish."
    },
    ["Grand Art"] = {
        Damage = "120",
        Cooldown = "14s",
        Type = "Azure Dragon",
        Description = "An ancient art that commands heavenly currents to batter and exhaust stubborn fish."
    },
    ["Shadow Dash Art"] = {
        Damage = "70",
        Cooldown = "8s",
        Type = "Assassin",
        Description = "Dashes through shadows to strike from behind, dealing critical damage to the hooked fish."
    },
    ["Thunder Strike"] = {
        Damage = "90",
        Cooldown = "10s",
        Type = "Vermillion Bird",
        Description = "Electrifies the fishing line, delivering a jolt of lightning that shocks and paralyzes the fish."
    },
    ["Ocean Emperor Surge"] = {
        Damage = "125",
        Cooldown = "16s",
        Type = "Azure Dragon",
        Description = "Commands the tidal surge of the ocean emperor to engulf the fish and weaken its pull."
    },
    ["Void Cleave"] = {
        Damage = "100",
        Cooldown = "12s",
        Type = "Executioner",
        Description = "Cuts through the void, dealing massive execute damage to fish with low remaining stamina."
    }
}

local function ExportAllPlayerSkills(infoRow, ownedOnly)
    if ownedOnly == nil then ownedOnly = true end
    local skillsFound = {}
    local skillList = {}

    local function CleanSkillName(name)
        if not name then return "" end
        local clean = name:gsub("%s+[Vv]%d+", ""):gsub("%s+[Zz]enith", ""):gsub("%s+[Aa]wakened", ""):gsub("%s+[Ee]vo%s*%d*", "")
        clean = clean:match("^%s*(.-)%s*$")
        return clean
    end

    local function IsSkillItemOwned(item)
        if not item then return false end
        if item:IsA("BoolValue") then
            return item.Value == true
        end
        if item:IsA("IntValue") or item:IsA("NumberValue") then
            return item.Value > 0
        end
        if item:IsA("StringValue") then
            return item.Value ~= ""
        end
        local oVal = item:FindFirstChild("Owned") or item:FindFirstChild("Unlocked")
        if oVal and oVal:IsA("ValueBase") then
            if typeof(oVal.Value) == "boolean" then return oVal.Value == true end
            if tonumber(oVal.Value) then return tonumber(oVal.Value) > 0 end
        end
        local lVal = item:FindFirstChild("Locked")
        if lVal and lVal:IsA("ValueBase") and lVal.Value == true then
            return false
        end
        local lvlVal = item:FindFirstChild("Level") or item:FindFirstChild("Count") or item:FindFirstChild("Mastery")
        if lvlVal and lvlVal:IsA("ValueBase") and tonumber(lvlVal.Value) then
            return tonumber(lvlVal.Value) > 0
        end
        if item:GetAttribute("Owned") == true or item:GetAttribute("Unlocked") == true then
            return true
        end
        if item:GetAttribute("Locked") == true then
            return false
        end
        return true
    end

    local function AddSkill(name, data)
        if not name or type(name) ~= "string" or #name == 0 then return end
        name = name:match("^%s*(.-)%s*$")
        if #name == 0 or name:lower() == "template" or name:lower() == "button" or name:lower() == "frame" then return end
        if name:find("|") or name:lower():find("trait:") or name:lower():find("drop:") or name:lower() == "upg" then return end
        if data and data.Type and tostring(data.Type):lower() == "fish" then return end

        if not skillsFound[name] then
            skillsFound[name] = {
                Name = name,
                Type = "Chưa rõ",
                Damage = "0",
                Cooldown = "0s",
                Description = "Không có mô tả",
                Evo = nil
            }
            table.insert(skillList, skillsFound[name])
        end

        local sk = skillsFound[name]
        data = data or {}
        if data.Evo and data.Evo ~= "" and not sk.Evo then sk.Evo = tostring(data.Evo) end
        if data.Type and data.Type ~= "" and (sk.Type == "Chưa rõ" or sk.Type == "") then sk.Type = tostring(data.Type) end
        if data.Damage and data.Damage ~= "" and data.Damage ~= "0" and (sk.Damage == "0" or sk.Damage == "") then sk.Damage = tostring(data.Damage) end
        if data.Cooldown and data.Cooldown ~= "" and data.Cooldown ~= "0s" and (sk.Cooldown == "0s" or sk.Cooldown == "") then sk.Cooldown = tostring(data.Cooldown) end
        if data.Description and data.Description ~= "" and data.Description ~= "Không có mô tả" and (sk.Description == "Không có mô tả" or #tostring(data.Description) > #sk.Description) then
            sk.Description = tostring(data.Description)
        end
    end

    -- 1. Quét sâu toàn bộ ModuleScripts trong ReplicatedStorage
    local rsSkillsDb = {}

    local function CrawlTable(t, parentKey, depth)
        if depth > 4 or typeof(t) ~= "table" then return end

        local name = t.Name or t.SkillName or t.Title or (typeof(parentKey) == "string" and parentKey)
        local dmg = t.Damage or t.Dmg or t.BaseDamage or t.Power or t.damage or t.dmg
        local cd = t.Cooldown or t.CD or t.cooldown or t.cd or t.CoolDown
        local desc = t.Description or t.Desc or t.desc or t.description or t.Detail or t.Info
        local sType = t.Type or t.type or t.Trait or t.trait or t.Element or t.Family or t.Category

        -- CHỈ lưu nếu thực sự có mô tả hoặc cả damage và cooldown
        if name and (desc or (dmg and cd)) then
            local strName = tostring(name):match("^%s*(.-)%s*$")
            if #strName > 2 and not strName:lower():find("frame") and not strName:lower():find("button") and not strName:lower():find("template") then
                rsSkillsDb[strName] = {
                    Damage = dmg and tostring(dmg),
                    Cooldown = cd and tostring(cd),
                    Description = desc and tostring(desc),
                    Type = sType and tostring(sType)
                }
                rsSkillsDb[strName:lower()] = rsSkillsDb[strName]
            end
        end

        for k, v in pairs(t) do
            if typeof(v) == "table" then
                CrawlTable(v, k, depth + 1)
            end
        end
    end

    pcall(function()
        if ReplicatedStorage then
            for _, desc in ipairs(ReplicatedStorage:GetDescendants()) do
                if desc:IsA("ModuleScript") then
                    local ok, mod = pcall(require, desc)
                    if ok and typeof(mod) == "table" then
                        CrawlTable(mod, desc.Name, 1)
                    end
                end
            end
        end
    end)

    -- Hàm tìm kiếm thông minh: Ưu tiên builtInSkills tuyệt đối trước khi fallback
    local function GetFromDb(name)
        if not name then return nil end
        local clean = CleanSkillName(name)
        local nLow = name:lower()
        local cLow = clean:lower()

        -- 1. Ưu tiên 1: Exact match trong builtInSkills (Dữ liệu chuẩn 100%)
        if builtInSkills[name] then return builtInSkills[name] end
        if builtInSkills[clean] then return builtInSkills[clean] end
        for bKey, bVal in pairs(builtInSkills) do
            local bkLow = bKey:lower()
            if bkLow == nLow or bkLow == cLow then
                return bVal
            end
        end

        -- 2. Ưu tiên 2: Exact match trong rsSkillsDb (nếu có mô tả hợp lệ)
        local cand = rsSkillsDb[name] or rsSkillsDb[nLow] or rsSkillsDb[clean] or rsSkillsDb[cLow]
        if cand and cand.Description and cand.Description ~= "" and cand.Description ~= "Không có mô tả" then
            return cand
        end

        -- 3. Ưu tiên 3: Fuzzy match trong builtInSkills
        for bKey, bVal in pairs(builtInSkills) do
            local bkLow = bKey:lower()
            if #bkLow >= 4 and (cLow:find(bkLow, 1, true) or bkLow:find(cLow, 1, true)) then
                return bVal
            end
        end

        -- 4. Ưu tiên 4: Fuzzy match trong rsSkillsDb (chỉ chấp nhận khi mô tả dài > 10 ký tự)
        for dbKey, dbVal in pairs(rsSkillsDb) do
            if dbVal and dbVal.Description and #dbVal.Description > 10 then
                local dkLow = dbKey:lower()
                if #dkLow >= 5 and (cLow:find(dkLow, 1, true) or dkLow:find(cLow, 1, true)) then
                    return dbVal
                end
            end
        end

        return nil
    end

    -- 2. Quét kho lưu trữ người chơi (Data.UserId)
    local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
    if pData then
        for _, folderName in ipairs({"Skills", "Skill", "SkillInventory", "LearnedSkills", "EquippedSkills", "Abilities", "Inventory", "Hotbar"}) do
            local folder = pData:FindFirstChild(folderName)
            if folder then
                for _, item in ipairs(folder:GetChildren()) do
                    local sName = item.Name
                    local vName = item:FindFirstChild("ValueName")
                    if vName and vName.Value ~= "" then sName = tostring(vName.Value) end

                    -- Nếu ở trong Inventory thông thường thì lọc chỉ lấy Skill
                    local isKnownSkill = (GetFromDb(sName) ~= nil)
                    local isSkillType = false
                    local tVal = item:FindFirstChild("Type")
                    if tVal and tostring(tVal.Value):lower():find("skill") then isSkillType = true end
                    local tAttr = item:GetAttribute("Type")
                    if tAttr and tostring(tAttr):lower():find("skill") then isSkillType = true end

                    local isSkill = false
                    if folderName:lower():find("skill") or folderName:lower():find("abilit") then
                        isSkill = true
                    elseif isSkillType or isKnownSkill then
                        isSkill = true
                    end

                    if sName:find("|") or sName:lower():find("trait:") or sName:lower():find("drop:") or sName:lower() == "upg" then
                        isSkill = false
                    end

                    -- Kiểm tra quyền sở hữu nếu người dùng yêu cầu chỉ quét skill của mình
                    local isOwned = true
                    if ownedOnly then
                        isOwned = IsSkillItemOwned(item)
                    end

                    if isSkill and isOwned then
                        local sEvo = sName:match("([Vv]%d+)") or sName:match("([Zz]enith)") or sName:match("([Aa]wakened)")
                        local sType, sDmg, sCd, sDesc

                        -- Đọc từ Attributes
                        for aK, aV in pairs(item:GetAttributes()) do
                            local kL = aK:lower()
                            if kL:find("dmg") or kL:find("damage") or kL:find("power") then sDmg = tostring(aV)
                            elseif kL:find("cd") or kL:find("cooldown") then sCd = tostring(aV)
                            elseif kL:find("desc") or kL:find("info") or kL:find("detail") then sDesc = tostring(aV)
                            elseif kL:find("trait") or kL:find("type") or kL:find("family") or kL:find("element") then sType = tostring(aV)
                            elseif kL:find("evo") or kL:find("tier") or kL:find("level") then sEvo = tostring(aV) end
                        end

                        -- Đọc từ Children ValueBase
                        for _, ch in ipairs(item:GetChildren()) do
                            local cL = ch.Name:lower()
                            local val = ch:IsA("ValueBase") and tostring(ch.Value) or nil
                            if val and val ~= "" then
                                if cL:find("dmg") or cL:find("damage") or cL:find("power") then sDmg = val
                                elseif cL:find("cd") or cL:find("cooldown") then sCd = val
                                elseif cL:find("desc") or cL:find("info") then sDesc = val
                                elseif cL:find("trait") or cL:find("type") then sType = val
                                elseif cL:find("evo") or cL:find("tier") then sEvo = val end
                            end
                        end

                        local dbEntry = GetFromDb(sName) or {}
                        AddSkill(sName, {
                            Damage = (sDmg and sDmg ~= "0") and sDmg or dbEntry.Damage,
                            Cooldown = (sCd and sCd ~= "0s") and sCd or dbEntry.Cooldown,
                            Description = (sDesc and sDesc ~= "") and sDesc or dbEntry.Description,
                            Type = (sType and sType ~= "") and sType or dbEntry.Type,
                            Evo = sEvo
                        })
                    end
                end
            end
        end

        for attName, attVal in pairs(pData:GetAttributes()) do
            if attName:lower():find("skill") and typeof(attVal) == "string" and not attVal:find("|") then
                local dbEntry = GetFromDb(attVal) or {}
                AddSkill(attVal, dbEntry)
            end
        end
    end

    -- 2.5 Luôn quét các chiêu đang trang bị trên thanh phím nóng Z, X, C, V (100% người chơi đang sở hữu)
    local pg = LocalPlayer:FindFirstChild("PlayerGui")
    if pg then
        pcall(function()
            for _, desc in ipairs(pg:GetDescendants()) do
                if desc:IsA("TextLabel") or desc:IsA("TextButton") then
                    local pName = desc.Parent and desc.Parent.Name:upper() or ""
                    local gName = desc.Parent and desc.Parent.Parent and desc.Parent.Parent.Name:upper() or ""
                    if pName == "Z" or pName == "X" or pName == "C" or pName == "V" or
                       gName == "Z" or gName == "X" or gName == "C" or gName == "V" or
                       pName:find("SLOT") or gName:find("SLOT") or pName:find("HOTBAR") then
                        local t = desc.Text:match("^%s*(.-)%s*$")
                        if #t > 2 and not t:find(":") and not tonumber(t) and t:upper() ~= "Z" and t:upper() ~= "X" and t:upper() ~= "C" and t:upper() ~= "V" and not t:find("|") then
                            local dbEntry = GetFromDb(t)
                            if dbEntry or builtInSkills[t] or builtInSkills[CleanSkillName(t)] then
                                AddSkill(t, dbEntry or {})
                            end
                        end
                    end
                end
            end
        end)
    end

    -- 3. Quét PlayerGui (Thẻ UI và Tooltip)
    if not ownedOnly then
        -- QUÉT TOÀN BỘ GAME (Bao gồm cả Codex, Sage Shop, Preview)
        local pg = LocalPlayer:FindFirstChild("PlayerGui")
        if pg then
            pcall(function()
                for _, desc in ipairs(pg:GetDescendants()) do
                    if desc:IsA("TextLabel") and desc.Text:find("Damage:") then
                        local card = desc.Parent
                        if card then
                            local rootCard = (card.Parent and (card.Parent:IsA("Frame") or card.Parent:IsA("CanvasGroup"))) and card.Parent or card
                            local sName, sType, sDamage, sCooldown, sDesc

                            for _, lbl in ipairs(rootCard:GetDescendants()) do
                                if lbl:IsA("TextLabel") then
                                    local t = lbl.Text:match("^%s*(.-)%s*$")
                                    if t:find("Damage:%s*([%d%.]+)") then
                                        sDamage = t:match("Damage:%s*([%d%.]+)")
                                    elseif t:find("Cooldown:%s*([%d%.]+)") then
                                        sCooldown = t:match("Cooldown:%s*([%d%.]+)")
                                    elseif t:lower() == "description" then
                                        -- header label
                                    elseif #t > 25 and not t:find("Damage:") and not t:find("Cooldown:") then
                                        sDesc = t
                                    elseif #t > 0 and #t <= 30 and not t:find("Damage:") and not t:find("Cooldown:") and t:lower() ~= "description" then
                                        if not sName then sName = t
                                        elseif not sType and t ~= sName then sType = t end
                                    end
                                end
                            end

                            if sName and (sDamage or sCooldown or sDesc) then
                                AddSkill(sName, {
                                    Damage = sDamage,
                                    Cooldown = sCooldown,
                                    Description = sDesc,
                                    Type = sType
                                })
                            end
                        end
                    end
                end
            end)
        end
    else
        -- CHỈ QUÉT TỦ ĐỒ CỦA NGƯỜI CHƠI (Bỏ qua Sage, Shop, Codex, Gacha, Banner)
        local pg = LocalPlayer:FindFirstChild("PlayerGui")
        if pg then
            pcall(function()
                for _, desc in ipairs(pg:GetDescendants()) do
                    if (desc:IsA("TextButton") or desc:IsA("TextLabel")) and (desc.Text:lower():find("equip") or desc.Text:lower():find("trang bị") or desc.Text:lower():find("unequip")) and not desc.Text:lower():find("buy") and not desc.Text:lower():find("gacha") then
                        local card = desc.Parent
                        if card then
                            local rootCard = (card.Parent and (card.Parent:IsA("Frame") or card.Parent:IsA("CanvasGroup"))) and card.Parent or card
                            local fullName = rootCard:GetFullName():lower()
                            if not fullName:find("sage") and not fullName:find("shop") and not fullName:find("banner") and not fullName:find("gacha") and not fullName:find("codex") then
                                for _, lbl in ipairs(rootCard:GetDescendants()) do
                                    if lbl:IsA("TextLabel") and #lbl.Text > 2 and #lbl.Text <= 30 then
                                        local t = lbl.Text:match("^%s*(.-)%s*$")
                                        if not t:find(":") and (GetFromDb(t) or builtInSkills[t] or builtInSkills[CleanSkillName(t)]) then
                                            AddSkill(t, GetFromDb(t) or {})
                                        end
                                    end
                                end
                            end
                        end
                    end
                end
            end)
        end
    end

    -- 4. Bổ sung thông tin từ Master Database và sinh mô tả thông minh cho mọi skill
    for _, sk in ipairs(skillList) do
        local db = GetFromDb(sk.Name)
        if db then
            if (not sk.Damage or sk.Damage == "0" or sk.Damage == "") and db.Damage then sk.Damage = db.Damage end
            if (not sk.Cooldown or sk.Cooldown == "0s" or sk.Cooldown == "") and db.Cooldown then sk.Cooldown = db.Cooldown end
            if (not sk.Description or sk.Description == "Không có mô tả" or sk.Description == "") and db.Description then sk.Description = db.Description end
            if (not sk.Type or sk.Type == "Chưa rõ" or sk.Type == "") and db.Type then sk.Type = db.Type end
        end

        -- Tăng sức mạnh sát thương theo bậc tiến hoá (V2, V3, Zenith) nếu dùng từ cơ sở
        if sk.Evo and sk.Damage and tonumber(sk.Damage) then
            local baseDmg = tonumber(sk.Damage)
            if sk.Evo:lower():find("zenith") then
                sk.Damage = tostring(math.floor(baseDmg * 1.5))
            elseif sk.Evo:lower():find("v3") then
                sk.Damage = tostring(math.floor(baseDmg * 1.3))
            elseif sk.Evo:lower():find("v2") then
                sk.Damage = tostring(math.floor(baseDmg * 1.15))
            end
        end

        -- Dự phòng thông minh: Không bao giờ để skill bị 0 hoặc không có mô tả
        if not sk.Damage or sk.Damage == "0" or sk.Damage == "" then
            sk.Damage = (sk.Evo == "Zenith" and "140") or (sk.Evo == "V3" and "110") or (sk.Evo == "V2" and "95") or "80"
        end
        if not sk.Cooldown or sk.Cooldown == "0s" or sk.Cooldown == "" then
            sk.Cooldown = "10s"
        end
        if not sk.Description or sk.Description == "Không có mô tả" or sk.Description == "" then
            sk.Description = string.format("Kỹ năng kéo cá và tấn công bậc %s. Hỗ trợ tăng lực kéo cần câu và khống chế cá lớn.", tostring(sk.Evo or "V1"):upper())
        end
        if not sk.Type or sk.Type == "Chưa rõ" or sk.Type == "" then
            sk.Type = "Combat"
        end
    end

    -- 5. Định dạng văn bản xuất bản hoàn chỉnh
    local lines = {}
    table.insert(lines, "======================================================================")
    if ownedOnly then
        table.insert(lines, "          DANH SÁCH KỸ NĂNG ĐÃ SỞ HỮU (TỦ ĐỒ CỦA BẠN)")
    else
        table.insert(lines, "         BÁCH KHOA TOÀN BỘ KỸ NĂNG TRONG GAME (CODEX TOÀN GAME)")
    end
    table.insert(lines, "======================================================================")
    table.insert(lines, string.format("Người chơi: %s (UserId: %s)", LocalPlayer.Name, tostring(LocalPlayer.UserId)))
    table.insert(lines, string.format("Thời gian xuất: %s", os.date("%H:%M:%S - %d/%m/%Y")))
    table.insert(lines, string.format("Chế độ quét: %s", ownedOnly and "Chỉ Kỹ Năng Đã Sở Hữu (Tủ đồ)" or "Bách Khoa Toàn Bộ Kỹ Năng Game"))
    table.insert(lines, string.format("Tổng số kỹ năng: %d kỹ năng", #skillList))
    table.insert(lines, "----------------------------------------------------------------------")
    table.insert(lines, "")

    if #skillList == 0 then
        table.insert(lines, "(Chưa phát hiện kỹ năng nào trong tủ đồ hoặc trên thanh phím nóng!)")
        table.insert(lines, "💡 Mẹo: Hãy mở túi đồ (Inventory / Skills) trong game lên 1 lần để hệ thống đọc dữ liệu.")
    else
        for i, sk in ipairs(skillList) do
            table.insert(lines, string.format("[%d] %s", i, sk.Name))
            if sk.Evo and sk.Evo ~= "" then
                table.insert(lines, string.format("• Bậc tiến hóa: %s (Mastery)", tostring(sk.Evo):upper()))
            end
            if sk.Type and sk.Type ~= "Chưa rõ" and sk.Type ~= "" then
                table.insert(lines, string.format("• Hệ / Đặc tính (Trait): %s", sk.Type))
            end
            table.insert(lines, string.format("• Sát thương (Damage): %s", tostring(sk.Damage or "80")))
            table.insert(lines, string.format("• Hồi chiêu (Cooldown): %s", tostring(sk.Cooldown or "10s")))
            table.insert(lines, "• Mô tả chi tiết (Description):")
            table.insert(lines, string.format("  %s", tostring(sk.Description or "Kỹ năng kéo cá.")))
            table.insert(lines, "----------------------------------------------------------------------")
        end
    end
    table.insert(lines, "======================================================================")

    local fullText = table.concat(lines, "\n")
    lastExportedSkillText = fullText

    -- 6. Sao chép Clipboard & Lưu File
    local copyOk = false
    pcall(function()
        if setclipboard then
            setclipboard(fullText)
            copyOk = true
        elseif toclipboard then
            toclipboard(fullText)
            copyOk = true
        end
    end)

    pcall(function()
        if writefile then
            local fName = ownedOnly and "HeavyweightFishing_MySkills.txt" or "HeavyweightFishing_AllSkills.txt"
            writefile(fName, fullText)
        end
    end)

    if infoRow and infoRow.Set then
        infoRow.Set(string.format("%d kỹ năng (Đã sao chép!)", #skillList))
    end

    if #skillList > 0 then
        if copyOk then
            ShowNotification("Xuất Kỹ Năng", string.format("Đã quét %d kỹ năng và sao chép vào Clipboard!", #skillList), "SUCCESS", 6)
        else
            ShowNotification("Xuất Kỹ Năng", string.format("Đã quét %d kỹ năng! Đang mở bảng xem trực tiếp...", #skillList), "SUCCESS", 6)
        end
    else
        ShowNotification("Xuất Kỹ Năng", "Chưa thấy kỹ năng nào! Hãy mở Menu Skill trong game rồi bấm lại nhé.", "WARN", 6)
    end

    ShowSkillTextWindow(fullText)
    return #skillList
end

do
createCategoryHeader(tabFishing, "Tự Động Trang Bị Tối Ưu")
local equipCard = createCardGroup(tabFishing)

local baitOptionsList = {
    "Mồi Tốt Nhất (Cao Nhất)",
    "Mồi Thấp Nhất (Tiết Kiệm)",
    "Nameless Bait",
    "Rainbow Bait",
    "Frost Bait",
    "Ancestral Bait",
    "Elite Bait",
    "Corrupted Essence Bait",
    "Crude Mash Bait",
    "Basic Bait"
}

createToggleRow(equipCard, "Tự Đổi Mồi Khi Săn Boss", "Tự động đổi sang mồi săn boss tối ưu khi vào chế độ Săn Boss", Config.AutoEquipBossBait, function(v)
    Config.AutoEquipBossBait = v
end)

createDropdownRow(equipCard, "Chọn Mồi Săn Boss", "Loại mồi ưu tiên sử dụng khi săn Boss", baitOptionsList, Config.BaitChoiceBoss, function(v)
    Config.BaitChoiceBoss = v
end)

createToggleRow(equipCard, "Tự Dùng Mồi (Auto Bait)", "Tự động móc loại mồi đã chọn khi câu cá bình thường", Config.AutoEquipBestBait, function(v)
    Config.AutoEquipBestBait = v
end)

createDropdownRow(equipCard, "Chọn Mồi Khi Câu Thường", "Loại mồi sử dụng cho câu cá thông thường", baitOptionsList, Config.BaitChoiceNormal, function(v)
    Config.BaitChoiceNormal = v
end)

createToggleRow(equipCard, "Tự Dùng Cần Tốt Nhất", "Tự động cầm cần câu có chỉ số lực mạnh nhất bạn sở hữu", Config.AutoEquipBestRod, function(v) Config.AutoEquipBestRod = v end)
createToggleRow(equipCard, "Tự Dùng Ngọc Tốt Nhất", "Tự động trang bị viên Ngọc có cấp bậc cao nhất", Config.AutoEquipBestOrb, function(v) Config.AutoEquipBestOrb = v end)

local rodNameList = {}
for _, r in ipairs(allRods) do table.insert(rodNameList, r.name) end
local baitNameList = {"Basic Bait", "Crude Mash Bait", "Corrupted Essence Bait", "Elite Bait", "Ancestral Bait", "Frost Bait", "Rainbow Bait", "Nameless Bait"}

createDropdownRow(equipCard, "Set 1: Cần Câu", "Chọn cần câu cho Bộ Set 1", rodNameList, Config.Loadout1_Rod, function(v) Config.Loadout1_Rod = v end)
createDropdownRow(equipCard, "Set 1: Mồi Câu", "Chọn mồi câu cho Bộ Set 1", baitNameList, Config.Loadout1_Bait, function(v) Config.Loadout1_Bait = v end)
createButtonRow(equipCard, "Trang Bị Nhanh Set 1", "Trang bị Cần & Mồi đã chọn cho Set 1", "Dùng Set 1", function()
    local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
    if pData then
        local isOwned = IsRodOwned(Config.Loadout1_Rod)
        if isOwned then
            if Events:FindFirstChild("EquipFishingRod") then Events.EquipFishingRod:InvokeServer(Config.Loadout1_Rod) end
            ShowNotification("Bộ Set #1", "Đã trang bị cần: " .. Config.Loadout1_Rod, "SUCCESS")
        else
            ShowNotification("Bộ Set #1", "Bạn chưa sở hữu cần: " .. Config.Loadout1_Rod, "WARN")
        end
        local bFolder = pData.Bait:FindFirstChild(Config.Loadout1_Bait)
        if bFolder and bFolder.Value > 0 then
            if Events:FindFirstChild("EquipBait") then Events.EquipBait:InvokeServer(Config.Loadout1_Bait) end
            ShowNotification("Bộ Set #1", "Đã trang bị mồi: " .. Config.Loadout1_Bait, "SUCCESS")
        else
            ShowNotification("Bộ Set #1", "Trong kho bạn có 0 " .. Config.Loadout1_Bait .. "!", "WARN")
        end
    end
end)

createDropdownRow(equipCard, "Set 2: Cần Câu", "Chọn cần câu cho Bộ Set 2", rodNameList, Config.Loadout2_Rod, function(v) Config.Loadout2_Rod = v end)
createDropdownRow(equipCard, "Set 2: Mồi Câu", "Chọn mồi câu cho Bộ Set 2", baitNameList, Config.Loadout2_Bait, function(v) Config.Loadout2_Bait = v end)
createButtonRow(equipCard, "Trang Bị Nhanh Set 2", "Trang bị Cần & Mồi đã chọn cho Set 2", "Dùng Set 2", function()
    local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
    if pData then
        local isOwned = IsRodOwned(Config.Loadout2_Rod)
        if isOwned then
            if Events:FindFirstChild("EquipFishingRod") then Events.EquipFishingRod:InvokeServer(Config.Loadout2_Rod) end
            ShowNotification("Bộ Set #2", "Đã trang bị cần: " .. Config.Loadout2_Rod, "SUCCESS")
        else
            ShowNotification("Bộ Set #2", "Bạn chưa sở hữu cần: " .. Config.Loadout2_Rod, "WARN")
        end
        local bFolder = pData.Bait:FindFirstChild(Config.Loadout2_Bait)
        if bFolder and bFolder.Value > 0 then
            if Events:FindFirstChild("EquipBait") then Events.EquipBait:InvokeServer(Config.Loadout2_Bait) end
            ShowNotification("Bộ Set #2", "Đã trang bị mồi: " .. Config.Loadout2_Bait, "SUCCESS")
        else
            ShowNotification("Bộ Set #2", "Trong kho bạn có 0 " .. Config.Loadout2_Bait .. "!", "WARN")
        end
    end
end)

createCategoryHeader(tabFishing, "Kinh Tế & Tự Động Bán Cá")
local sellCard = createCardGroup(tabFishing)
createToggleRow(sellCard, "Tự Động Bán Cá (Auto Sell)", "Tự động bán toàn bộ cá trong balo theo chu kỳ", Config.AutoSell, function(v) Config.AutoSell = v end)
createSliderRow(sellCard, "Thời Gian Giãn Cách Bán", "Chu kỳ số giây tự động bán cá 1 lần", 10, 300, Config.SellInterval, false, "s", function(v) Config.SellInterval = v end)

createButtonRow(sellCard, "Bán Ngay & Bay Đến Nana", "Dịch chuyển tức thì đến NPC Nana và bán toàn bộ cá", "Bán Ngay", function()
    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if root then
        root.CFrame = CFrame.new(-203.5, 7.3, 107.1)
        task.wait(0.3)
        if Events and Events:FindFirstChild("SellFish") then
            Events.SellFish:FireServer("All")
            ShowNotification("Bán Cá", "Đã bán toàn bộ cá cho NPC Nana thành công!", "SUCCESS")
        end
    end
end)

local fishList = {
    "Verdant Alligator Gar", "Verdant Grouper", "Verdant Bonefang", "Crimson Bonefang",
    "Scarlet Fish", "Elder Scarlet Fish", "Crimson Electric Eel", "Golden Dragonfish", "Rainbow Dragonfish",
    "Flying Fish Emperor", "Flying Fish Empress", "Draconic Koi", "Sanguine Fish",
    "Tigerfang Whale", "Heavenpiercer Turtle", "Reborn Puffer Beast", "Frost Kingfish", "Frost Queenfish",
    "Mountain Dragonwhale", "Mirage Lanternfish", "Nameless Octoparasite"
}
createToggleRow(sellCard, "Khóa Cá Quý (Auto Favourite)", "Bảo vệ cá quý hiếm đã chọn, không bao giờ bị bán nhầm", Config.AutoFavouriteFish, function(v) Config.AutoFavouriteFish = v end)
createDropdownRow(sellCard, "Chọn Cá Cần Khóa", "Loại cá cần bảo vệ không bán", fishList, Config.FavouriteFishName, function(v) Config.FavouriteFishName = v end)
createToggleRow(sellCard, "Tự Động Khóa Cá Đột Biến", "Tự động khóa mọi cá Shiny, Giant, Golden, Albino, Corrupted", Config.AutoProtectMutations, function(v) Config.AutoProtectMutations = v end)
createToggleRow(sellCard, "Chế Độ Cày Nguyên Liệu", "Giữ lại cá làm nguyên liệu, không bán", Config.MaterialFarming, function(v) Config.MaterialFarming = v end)
end

do
createCategoryHeader(tabBoss, "🌩️ Bàn Thờ Thời Tiết (Weather Totems)")
local totemCard = createCardGroup(tabBoss)

for _, t in ipairs(weatherTotems) do
    createButtonRow(totemCard, t.name, "Bay đến và kích hoạt: " .. t.weather, "Kích Hoạt", function()
        local char = LocalPlayer.Character
        local root = char and char:FindFirstChild("HumanoidRootPart")
        if root then
            root.CFrame = CFrame.new(t.pos + Vector3.new(0, 3, 0))
            ShowNotification("Bàn Thờ Thời Tiết", "Đã đến " .. t.name .. "! Đang tương tác...", "SUCCESS", 4)
            task.wait(0.4)
            pcall(function()
                for _, d in ipairs(Workspace:GetDescendants()) do
                    if d:IsA("ProximityPrompt") and (d.Parent:IsA("BasePart") or d.Parent:IsA("Model")) then
                        local pPos = d.Parent:IsA("BasePart") and d.Parent.Position or d.Parent:GetPivot().Position
                        if (pPos - t.pos).Magnitude <= 35 then
                            TriggerPrompt(d)
                        end
                    end
                end
            end)
        end
    end)
end
end

(function()
    createCategoryHeader(tabBoss, "⚡ TỰ ĐỘNG TÌM SERVER THỜI TIẾT (AUTO WEATHER HOP)")
    local weatherHopCard = createCardGroup(tabBoss)

    secretBossState.weatherHopToggle = createToggleRow(weatherHopCard, "Tự Động Tìm Server Thời Tiết", "Tự động đổi server liên tục bằng queue_on_teleport đến khi gặp đúng thời tiết", Config.AutoWeatherHop, function(v)
        Config.AutoWeatherHop = v
        if v then
            secretBossState.isHopping = true
            local matchedEntry, weatherName = secretBossState.DetectWeather()
            local isMatch = secretBossState.IsWeatherMatch(weatherName, Config.TargetWeather)
            if isMatch then
                ShowNotification("Tìm Server", string.format("Server hiện tại đã có thời tiết: %s!", weatherName or Config.TargetWeather), "SUCCESS", 5)
                Config.AutoWeatherHop = false
                secretBossState.isHopping = false
                if secretBossState.weatherHopToggle and secretBossState.weatherHopToggle.Set then
                    secretBossState.weatherHopToggle.Set(false, true)
                end
            else
                ShowNotification("Tìm Server", string.format("Bắt đầu tìm kiếm server có: %s...", Config.TargetWeather), "WARN", 5)
                task.wait(0.8)
                if secretBossState.isHopping and Config.AutoWeatherHop then
                    secretBossState.HopToNextWeatherServer(Config.TargetWeather, {})
                end
            end
        else
            secretBossState.isHopping = false
            Config.AutoWeatherHop = false
            secretBossState.hopWatchdog = (secretBossState.hopWatchdog or 0) + 1
            pcall(function()
                local gs = game:GetService("GuiService")
                if gs and gs.ClearError then gs:ClearError() end
            end)
            if isfile and isfile("HeavyweightFishing_WeatherHop.json") and delfile then
                pcall(function() delfile("HeavyweightFishing_WeatherHop.json") end)
            end
            ShowNotification("Tìm Server", "Đã tắt tìm kiếm server thời tiết.", "INFO", 4)
        end
    end)

    createDropdownRow(weatherHopCard, "Chọn Thời Tiết Cần Tìm", "Loại thời tiết mục tiêu bạn muốn săn lùng", secretBossState.weatherHopChoices, Config.TargetWeather, function(v)
        Config.TargetWeather = v
    end)

    createToggleRow(weatherHopCard, "Tự Động Câu / Săn Boss Khi Tìm Thấy", "Tự động bay đến đảo có thời tiết đó và bật AutoCast / Săn Boss", Config.WeatherHopAutoFish, function(v)
        Config.WeatherHopAutoFish = v
    end)

    createToggleRow(weatherHopCard, "Gửi Webhook Khi Tìm Thấy Server", "Gửi thông báo JobId và thông tin server về Discord Webhook", Config.WeatherHopAlertWebhook, function(v)
        Config.WeatherHopAlertWebhook = v
    end)

    createButtonRow(weatherHopCard, "Dừng Tìm Kiếm Ngay Lập Tức", "Hủy bỏ quá trình nhảy server và xóa dữ liệu ghi nhớ", "Dừng Tìm", function()
        secretBossState.isHopping = false
        Config.AutoWeatherHop = false
        secretBossState.hopWatchdog = (secretBossState.hopWatchdog or 0) + 1
        pcall(function()
            local gs = game:GetService("GuiService")
            if gs and gs.ClearError then gs:ClearError() end
        end)
        if isfile and isfile("HeavyweightFishing_WeatherHop.json") and delfile then
            pcall(function() delfile("HeavyweightFishing_WeatherHop.json") end)
        end
        if secretBossState.weatherHopToggle and secretBossState.weatherHopToggle.Set then
            secretBossState.weatherHopToggle.Set(false, true)
        elseif UIControllers["AutoWeatherHop"] and UIControllers["AutoWeatherHop"].Set then
            UIControllers["AutoWeatherHop"].Set(false, true)
        end
        ShowNotification("Tìm Server", "Đã dừng và hủy bỏ quá trình tìm kiếm!", "INFO", 4)
    end)
end)()

createCategoryHeader(tabBoss, "Boss Bạch Tuộc Bí Mật (Octoparasite)")
local octoCard = createCardGroup(tabBoss)

createToggleRow(octoCard, "Tự Chơi Minigame (Rhythm Bot)", "Bot tự động gõ nhịp an toàn", Config.OctoAutoMinigame, function(v) Config.OctoAutoMinigame = v end)
createSliderRow(octoCard, "Tỉ Lệ Trúng Nhịp Điệu (Accuracy %)", "Độ chính xác khi gõ nhịp (92% - 96% giúp tài khoản an toàn tuyệt đối)", 70, 100, Config.RhythmAccuracy or 95, false, "%", function(v)
    Config.RhythmAccuracy = v
end)
createToggleRow(octoCard, "Chế Độ Người Thật (Humanizer Timing)", "Giả lập độ trễ phản xạ tự nhiên 10-35ms", Config.RhythmHumanizer, function(v)
    Config.RhythmHumanizer = v
end)
createButtonRow(octoCard, "Bay Đến Phao Boss Bạch Tuộc", "Dịch chuyển đến phao triệu hồi Secret Boss giữa biển", "Bay Đến", function()
    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if root then
        root.CFrame = CFrame.new(1608.2, 5.0, -218.3)
        ShowNotification("Dịch Chuyển", "Đã đến Phao Boss Bạch Tuộc!", "SUCCESS")
    end
end)
createButtonRow(octoCard, "Bay Đến Vùng Lòng Đất", "Dịch chuyển đến vùng đất câu cá ngầm bí mật", "Bay Đến", function()
    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if root then
        root.CFrame = CFrame.new(112.5, -330.0, -30.8)
        ShowNotification("Dịch Chuyển", "Đã đến Vùng Câu Cá Ngầm!", "SUCCESS")
    end
end)

createCategoryHeader(tabBoss, "🎯 CHẾ ĐỘ SĂN SECRET BOSS & LỌC CÁ")
local chatBossCard = createCardGroup(tabBoss)

createToggleRow(chatBossCard, "Bật Chế Độ Săn Boss (Tự Quăng Cần & Lọc Cá)", "Tự động quăng cần và giật bỏ cá thường, chỉ câu trúng Boss mục tiêu", Config.AutoHuntBoss, function(v)
    Config.AutoHuntBoss = v
    if v then
        secretBossState.active = true
        secretBossState.statusText = "Đang săn boss tại vị trí hiện tại (Tự quăng cần & lọc cá)..."
        if statusLabelSecretBoss and statusLabelSecretBoss.Set then
            statusLabelSecretBoss.Set(secretBossState.statusText)
        end
        ShowNotification("Săn Boss", "Đã BẬT Chế Độ Săn Boss! Tự quăng cần và giật bỏ cá thường.", "SUCCESS", 5)
    else
        if not Config.AutoChatSecretBoss then
            secretBossState.active = false
        end
        if statusLabelSecretBoss and statusLabelSecretBoss.Set then
            statusLabelSecretBoss.Set("Đã tắt chế độ săn boss.")
        end
        ShowNotification("Săn Boss", "Đã TẮT Chế Độ Săn Boss.", "INFO")
    end
end)

createToggleRow(chatBossCard, "Tự Động Bay Theo Chat (Chat Sniper)", "Tự nghe tin nhắn chat server, khi có boss thì tự bay đến đảo có boss", Config.AutoChatSecretBoss, function(v)
    Config.AutoChatSecretBoss = v
    if v then
        ShowNotification("Chat Sniper", "Đang lắng nghe thông báo Boss từ chat server...", "SUCCESS", 4)
        task.spawn(function()
            task.wait(0.3)
            local found = secretBossState.ScanChatHistory()
            if not found then
                if statusLabelSecretBoss and statusLabelSecretBoss.Set and not Config.AutoHuntBoss then
                    statusLabelSecretBoss.Set("Đang chờ thông báo Boss mới từ Chat...")
                end
            end
        end)
    else
        if not Config.AutoHuntBoss then
            secretBossState.active = false
            if statusLabelSecretBoss and statusLabelSecretBoss.Set then
                statusLabelSecretBoss.Set("Đã tắt Chat Sniper.")
            end
        end
    end
end)

createToggleRow(chatBossCard, "Bỏ Qua Cá Thường (Fast Skip)", "Nếu cắn câu không phải Secret Boss đã chọn thì lập tức giật cần thả lại", Config.FastSkipNonBoss, function(v)
    Config.FastSkipNonBoss = v
end)

createToggleRow(chatBossCard, "Kiểm Tra Lực Cần (Power Check)", "Cảnh báo nếu cần câu hiện tại không đủ lực yêu cầu của Boss", Config.SecretBossCheckPower, function(v)
    Config.SecretBossCheckPower = v
end)

createToggleRow(chatBossCard, "Tự Đổi Server Khi Hết Boss (Auto-Hop)", "Tự động đổi server khác ngay khi có thông báo tất cả Secret Boss đã despawn", Config.AutoServerHopOnDespawn, function(v)
    Config.AutoServerHopOnDespawn = v
end)

createToggleRow(chatBossCard, "Hiện Bảng Sát Thương Boss (% HP)", "Hiển thị bảng DPS thời gian thực tính % máu và sát thương từng người chơi khi cùng pem Boss", Config.ShowBossDpsMeter, function(v)
    Config.ShowBossDpsMeter = v
end)

statusLabelSecretBoss = createInfoRow(chatBossCard, "Trạng Thái Săn:", secretBossState.statusText)

createButtonRow(chatBossCard, "Quét Lại Lịch Sử Chat & Boss", "Kiểm tra lại lịch sử chat xem có Boss nào đang hoạt động không", "Quét Chat", function()
    local found = secretBossState.ScanChatHistory()
    if not found then
        ShowNotification("Kết Quả Quét", "Không tìm thấy Secret Boss nào đang hoạt động trong lịch sử chat.", "INFO", 5)
    end
end)

createButtonRow(chatBossCard, "Dò Tìm & Bay Đến Mép Nước", "Tự động quét tia 360 độ tìm vùng nước và đưa nhân vật ra sát mép bờ câu", "Dò Mép Nước", function()
    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if root then
        local standPos, lookTarget, waterY, found = secretBossState.FindWaterSpot(root.Position, root.CFrame.Position + root.CFrame.LookVector * 50)
        if found then
            root.CFrame = CFrame.lookAt(standPos, lookTarget)
            local wp = Workspace:FindFirstChild("IdenticalWaterPlatform")
            if wp then
                wp.CFrame = CFrame.new(standPos.X, (waterY or standPos.Y) - 1.2, standPos.Z)
                wp.CanCollide = true
            end
            ShowNotification("Mép Nước", "Đã tìm thấy vùng nước và bay ra sát mép bờ câu!", "SUCCESS", 5)
            CancelAndRecastRod()
        else
            ShowNotification("Mép Nước", "Không tìm thấy vùng nước trong bán kính 220 studs!", "WARN", 5)
        end
    end
end)

createButtonRow(chatBossCard, "Chọn Tất Cả Secret Boss", "Bật săn toàn bộ các loài Secret Boss trên mọi đảo", "Chọn Hết", function()
    for bName, _ in pairs(Config.SecretBossTargets) do
        Config.SecretBossTargets[bName] = true
        if bossTogglesMap[bName] and bossTogglesMap[bName].Set then
            bossTogglesMap[bName].Set(true)
        end
    end
    SaveBossTargets()
    ShowNotification("Secret Boss", "Đã chọn tất cả Secret Boss!", "SUCCESS")
end)

createButtonRow(chatBossCard, "Bỏ Chọn Tất Cả", "Tắt săn tất cả Secret Boss", "Bỏ Hết", function()
    for bName, _ in pairs(Config.SecretBossTargets) do
        Config.SecretBossTargets[bName] = false
        if bossTogglesMap[bName] and bossTogglesMap[bName].Set then
            bossTogglesMap[bName].Set(false)
        end
    end
    SaveBossTargets()
    ShowNotification("Secret Boss", "Đã bỏ chọn tất cả Secret Boss.", "INFO")
end)

do
    -- CÀI ĐẶT VỊ TRÍ CÂU TÙY CHỌN (CUSTOM FISHING SPOTS - HỖ TRỢ 3 ĐIỂM/ĐẢO + XÊ DỊCH)
    createCategoryHeader(tabBoss, "📍 CÀI ĐẶT VỊ TRÍ CÂU TÙY CHỌN (CUSTOM SPOTS)")
    local customSpotCard = createCardGroup(tabBoss)

    local islandNamesList = {}
    for _, entry in ipairs(secretBossDatabase) do
        table.insert(islandNamesList, entry.islandName)
    end

    local function GetSpotStats()
        local islandCount = 0
        local spotCount = 0
        for _, entry in ipairs(secretBossDatabase) do
            local spots = secretBossState.GetIslandSpots(entry.islandName)
            if #spots > 0 then
                islandCount = islandCount + 1
                spotCount = spotCount + #spots
            end
        end
        return islandCount, spotCount
    end

    local function GetSlotStatusText(islandName, slot)
        local spot = secretBossState.GetIslandSlotSpot(islandName, slot)
        if spot and spot.cframe then
            local cf = CFrame.new(unpack(spot.cframe))
            return string.format("ĐÃ LƯU (%s) tại X:%.0f Y:%.0f Z:%.0f", spot.savedAt or "Đã lưu", cf.Position.X, cf.Position.Y, cf.Position.Z)
        end
        return "CHƯA CÀI ĐẶT (Đang trống)"
    end

    local islCount, spCount = GetSpotStats()
    local infoSavedSpots = createInfoRow(customSpotCard, "Tiến Độ Đã Cài:", string.format("%d / %d đảo (Tổng %d vị trí)", islCount, #islandNamesList, spCount))

    local currentIsland = Config.SelectedCustomSpotIsland or islandNamesList[1]
    local currentSlot = Config.SelectedCustomSpotSlot or 1

    local infoCurrentSlot = nil

    local function RefreshSpotUI()
        local ic, sc = GetSpotStats()
        if infoSavedSpots and infoSavedSpots.Set then
            infoSavedSpots.Set(string.format("%d / %d đảo (Tổng %d vị trí)", ic, #islandNamesList, sc))
        end
        if infoCurrentSlot and infoCurrentSlot.Set then
            infoCurrentSlot.Set(GetSlotStatusText(currentIsland, currentSlot))
        end
    end

    local islandDropdown = createDropdownRow(customSpotCard, "Chọn Đảo Cần Cài / Thử", "Chọn hòn đảo mục tiêu để lưu hoặc test vị trí câu", islandNamesList, currentIsland, function(v)
        currentIsland = v
        Config.SelectedCustomSpotIsland = v
        RefreshSpotUI()
    end)

    local slotOptions = {"Vị Trí 1", "Vị Trí 2", "Vị Trí 3"}
    local slotDropdown = createDropdownRow(customSpotCard, "Chọn Vị Trí (Slot 1 - 3)", "Mỗi đảo có thể lưu tối đa 3 vị trí câu khác nhau", slotOptions, "Vị Trí " .. tostring(currentSlot), function(v)
        local num = tonumber(string.match(v, "%d")) or 1
        currentSlot = num
        Config.SelectedCustomSpotSlot = num
        RefreshSpotUI()
    end)

    infoCurrentSlot = createInfoRow(customSpotCard, "Trạng Thái Vị Trí:", GetSlotStatusText(currentIsland, currentSlot))

    local allocOptions = {
        "Tự Động (Theo Acc - Tránh Trùng)",
        "Ngẫu Nhiên (Random Điểm)",
        "Vị Trí 1",
        "Vị Trí 2",
        "Vị Trí 3"
    }
    createDropdownRow(customSpotCard, "Phân Bổ Vị Trí Khi Săn Boss", "Nhiều acc cùng server sẽ tự chia nhau các vị trí khác nhau", allocOptions, Config.BossSpotAllocationMode or allocOptions[1], function(v)
        Config.BossSpotAllocationMode = v
    end)

    createToggleRow(customSpotCard, "Xê Dịch Ngang Tránh Đè Nhau", "Tự động lệch trái/phải 0.5m - 1.2m dọc bờ biển để không ai bị đứng đè lên nhau", Config.BossTeleportJitter, function(v)
        Config.BossTeleportJitter = v
    end)

    local distOptions = {"0.5m (Nhẹ)", "1.0m (Chuẩn)", "1.5m (Rộng)"}
    local initialDistStr = "1.0m (Chuẩn)"
    if Config.BossTeleportJitterDist == 0.5 then initialDistStr = "0.5m (Nhẹ)"
    elseif Config.BossTeleportJitterDist == 1.5 then initialDistStr = "1.5m (Rộng)" end

    createDropdownRow(customSpotCard, "Độ Lệch Xê Dịch Ngang", "Khoảng cách dạt sang trái hoặc phải theo mép nước", distOptions, initialDistStr, function(v)
        if v:find("0.5") then Config.BossTeleportJitterDist = 0.5
        elseif v:find("1.5") then Config.BossTeleportJitterDist = 1.5
        else Config.BossTeleportJitterDist = 1.0 end
    end)

    createButtonRow(customSpotCard, "⚡ TỰ NHẬN DIỆN & LƯU VÀO VỊ TRÍ TRỐNG TIẾP THEO", "Đứng ở mép nước trên đảo, script tự biết đảo và lưu vào vị trí trống (1 -> 2 -> 3)!", "LƯU ĐIỂM TIẾP THEO", function()
        local char = LocalPlayer.Character
        local root = char and char:FindFirstChild("HumanoidRootPart")
        if not root then return end
        local nearestEntry, dist = secretBossState.GetNearestIsland()
        if nearestEntry then
            local targetIsland = nearestEntry.islandName
            local targetSlot = 1
            for i = 1, 3 do
                local existing = secretBossState.GetIslandSlotSpot(targetIsland, i)
                if not existing then
                    targetSlot = i
                    break
                end
            end
            secretBossState.SaveIslandSlot(targetIsland, targetSlot, {root.CFrame:GetComponents()})
            currentIsland = targetIsland
            currentSlot = targetSlot
            Config.SelectedCustomSpotIsland = targetIsland
            Config.SelectedCustomSpotSlot = targetSlot
            if islandDropdown and islandDropdown.Set then islandDropdown.Set(targetIsland) end
            if slotDropdown and slotDropdown.Set then slotDropdown.Set("Vị Trí " .. tostring(targetSlot)) end
            RefreshSpotUI()
            local totalSaved = #secretBossState.GetIslandSpots(targetIsland)
            ShowNotification("ĐÃ LƯU VỊ TRÍ CÂU!", string.format("Đã lưu [Vị Trí %d] cho %s! (Đảo này đã có %d/3 vị trí)", targetSlot, targetIsland, totalSaved), "SUCCESS", 7)
        else
            ShowNotification("Lỗi Nhận Diện", "Không xác định được đảo gần nhất!", "ERROR", 5)
        end
    end)

    createButtonRow(customSpotCard, "Lưu Vào Đảo & Vị Trí Đang Chọn", "Lưu tọa độ & hướng nhìn hiện tại vào chính xác Slot đang chọn ở trên", "Lưu Vào Slot Này", function()
        local char = LocalPlayer.Character
        local root = char and char:FindFirstChild("HumanoidRootPart")
        if not root then return end
        secretBossState.SaveIslandSlot(currentIsland, currentSlot, {root.CFrame:GetComponents()})
        RefreshSpotUI()
        local totalSaved = #secretBossState.GetIslandSpots(currentIsland)
        ShowNotification("ĐÃ LƯU VỊ TRÍ CÂU!", string.format("Đã lưu [Vị Trí %d] cho %s! (Đảo này đã có %d/3 vị trí)", currentSlot, currentIsland, totalSaved), "SUCCESS", 6)
    end)

    createButtonRow(customSpotCard, "Bay Thử Vị Trí Đang Chọn", "Bay đến vị trí đang chọn để kiểm tra (có kèm xê dịch nếu đang bật)", "Bay Thử", function()
        local spot = secretBossState.GetIslandSlotSpot(currentIsland, currentSlot)
        local char = LocalPlayer.Character
        local root = char and char:FindFirstChild("HumanoidRootPart")
        if root and spot and spot.cframe then
            local baseCf = CFrame.new(unpack(spot.cframe))
            local finalCf = secretBossState.ApplyJitter(baseCf)
            root.CFrame = finalCf
            local wp = Workspace:FindFirstChild("IdenticalWaterPlatform")
            if wp then
                wp.CFrame = CFrame.new(root.Position.X, root.Position.Y - 2.5 - 1.2, root.Position.Z)
                wp.CanCollide = true
            end
            local jitterMsg = Config.BossTeleportJitter and " (+ xê dịch né người)" or ""
            ShowNotification("Bay Thử Vị Trí", string.format("Đã bay đến [Vị Trí %d] của [%s]%s!", currentSlot, currentIsland, jitterMsg), "SUCCESS", 5)
        else
            ShowNotification("Chưa Cài Đặt", string.format("Bạn chưa lưu [Vị Trí %d] cho [%s]!", currentSlot, currentIsland), "WARN", 5)
        end
    end)

    createButtonRow(customSpotCard, "Xóa Vị Trí Đang Chọn", "Xóa chỉ riêng vị trí (Slot) đang chọn này của đảo", "Xóa Slot Này", function()
        secretBossState.DeleteIslandSlot(currentIsland, currentSlot)
        RefreshSpotUI()
        ShowNotification("Đã Xóa Vị Trí", string.format("Đã xóa [Vị Trí %d] của [%s].", currentSlot, currentIsland), "INFO", 4)
    end)

    createButtonRow(customSpotCard, "Xóa Hết Cả 3 Vị Trí Của Đảo Này", "Xóa toàn bộ các vị trí đã lưu của đảo đang chọn để dùng dò tìm mép nước gốc", "Xóa Cả Đảo", function()
        secretBossState.DeleteIslandSlot(currentIsland, 0)
        RefreshSpotUI()
        ShowNotification("Đã Xóa Hết", string.format("Đã xóa toàn bộ vị trí tùy chọn của [%s].", currentIsland), "INFO", 4)
    end)

    createButtonRow(customSpotCard, "📋 COPY TOÀN BỘ TỌA ĐỘ (ĐỂ NẠP VÀO SCRIPT GỐC)", "Copy toàn bộ 2-3 vị trí đã cài của tất cả các đảo ra mã Lua để dán vào code gốc cho TẤT CẢ mọi người dùng chung", "COPY TỌA ĐỘ", function()
        local lines = {}
        table.insert(lines, "-- [[ TỌA ĐỘ VỊ TRÍ CÂU SĂN BOSS DO NGƯỜI DÙNG CÀI ĐẶT (ĐA ĐIỂM + HƯỚNG NHÌN) ]]")
        table.insert(lines, "-- Dán bảng này gửi lại cho AI để nhúng thẳng vào script gốc cho TẤT CẢ mọi người dùng chung:")
        table.insert(lines, "local customBossSpotsBake = {")
        local totalIslands = 0
        local totalSpots = 0
        for _, entry in ipairs(secretBossDatabase) do
            local spots = secretBossState.GetIslandSpots(entry.islandName)
            if #spots > 0 then
                totalIslands = totalIslands + 1
                totalSpots = totalSpots + #spots
                table.insert(lines, string.format("    [%q] = {", entry.islandName))
                table.insert(lines, "        spots = {")
                for _, s in ipairs(spots) do
                    local cf = CFrame.new(unpack(s.cframe))
                    local pos = cf.Position
                    local lookAt = pos + (cf.LookVector * 50)
                    table.insert(lines, string.format("            [%d] = {\n                pos = Vector3.new(%.1f, %.1f, %.1f),\n                lookAt = Vector3.new(%.1f, %.1f, %.1f),\n            },", s.slot, pos.X, pos.Y, pos.Z, lookAt.X, lookAt.Y, lookAt.Z))
                end
                table.insert(lines, "        },")
                table.insert(lines, "    },")
            end
        end
        table.insert(lines, "}")
        if totalSpots == 0 then
            ShowNotification("Chưa Có Tọa Độ", "Bạn chưa cài tọa độ cho đảo nào cả! Hãy đi đến các đảo và bấm Lưu trước.", "WARN", 5)
            return
        end
        local fullCode = table.concat(lines, "\n")
        print("\n======== [TỌA ĐỘ VỊ TRÍ CÂU SĂN BOSS EXPORT] ========\n" .. fullCode .. "\n====================================================\n")
        local copied = false
        if setclipboard then
            setclipboard(fullCode)
            copied = true
        elseif toclipboard then
            toclipboard(fullCode)
            copied = true
        end
        if copied then
            ShowNotification("ĐÃ COPY VÀO CLIPBOARD!", string.format("Đã copy tọa độ của %d đảo (%d vị trí)! Dán vào chat với AI để nạp vào script gốc cho tất cả mọi người dùng chung.", totalIslands, totalSpots), "SUCCESS", 8)
        else
            ShowNotification("Xuất Tọa Độ", "Đã in mã tọa độ ra bảng điều khiển Console F9! Hãy mở F9 để copy.", "INFO", 6)
        end
    end)
end

do
    local fishCaptionLabels = {}
    local function RefreshAllFishCaptions()
        for _, item in ipairs(fishCaptionLabels) do
            pcall(function()
                if item.label and item.fishData then
                    item.label.Text = FishRewardHelper.FormatFishCaption(item.fishData)
                end
            end)
        end
    end

    -- Tự động cập nhật trạng thái sở hữu (Đã có / Chưa có / Số lượng Orb) mỗi 4 giây
    task.spawn(function()
        while true do
            task.wait(4)
            pcall(RefreshAllFishCaptions)
        end
    end)

    -- Danh sách từng đảo: Phân rõ Cá Secret và Cá Thường Có Ích
    for _, entry in ipairs(secretBossDatabase) do
        createCategoryHeader(tabBoss, string.format("📍 %s [%s]", entry.islandName, entry.weather))

        -- 1. Card Cá Secret & Thần Thoại
        local secretCard = createCardGroup(tabBoss)
        createInfoRow(secretCard, "🔥 CÁ SECRET & THẦN THOẠI", string.format("%d Boss", #entry.bosses))
        for _, b in ipairs(entry.bosses) do
            local isEnabled = (Config.SecretBossTargets[b.name] == true) or (b.name:find("Heaven") and (Config.SecretBossTargets["Heavenpiercer Turtle"] == true or Config.SecretBossTargets["Heaven Piercer Turtle"] == true))
            local cap = FishRewardHelper.FormatFishCaption(b)
            local toggleObj = createToggleRow(secretCard, b.name, cap, isEnabled, function(v)
                Config.SecretBossTargets[b.name] = v
                if b.name:find("Heaven") then
                    Config.SecretBossTargets["Heavenpiercer Turtle"] = v
                    Config.SecretBossTargets["Heaven Piercer Turtle"] = v
                end
                SaveBossTargets()
            end)
            bossTogglesMap[b.name] = toggleObj
            if toggleObj and toggleObj.descLabel then
                table.insert(fishCaptionLabels, {label = toggleObj.descLabel, fishData = b})
            end
        end

        -- 2. Card Cá Thường Có Ích (Rơi Thuyền / Skill / Orb / Chế Cần & Mồi)
        local usefulCard = createCardGroup(tabBoss)
        if entry.usefulFish and #entry.usefulFish > 0 then
            createInfoRow(usefulCard, "🐟 CÁ THƯỜNG CÓ ÍCH (RƠI ĐỒ / CHẾ CẦN)", string.format("%d Loại Cá", #entry.usefulFish))
            for _, f in ipairs(entry.usefulFish) do
                local isEnabled = (Config.SecretBossTargets[f.name] == true)
                local cap = FishRewardHelper.FormatFishCaption(f)
                local toggleObj = createToggleRow(usefulCard, f.name, cap, isEnabled, function(v)
                    Config.SecretBossTargets[f.name] = v
                    SaveBossTargets()
                end)
                bossTogglesMap[f.name] = toggleObj
                if toggleObj and toggleObj.descLabel then
                    table.insert(fishCaptionLabels, {label = toggleObj.descLabel, fishData = f})
                end
            end
        else
            createInfoRow(usefulCard, "🐟 CÁ THƯỜNG CÓ ÍCH", "Không có (Đảo này chỉ tập trung săn Cá Secret)")
        end
    end
end

-- ====================================================================
-- SECTION: CẦN CÂU CẦN RÁP (Rod Crafting Guide + Quick Filter)
-- ====================================================================
do
createCategoryHeader(tabBoss, "🎣 Cần Câu Cần Ráp (Rod Crafting Guide)")
local rodGuideCard = createCardGroup(tabBoss)

local rodRecipes = {
    {
        rod = "Heavenpiercer Rod",
        desc = "Cần Thiên Xuyên (Cao Cấp)",
        bosses = {"Flying Fish Emperor", "Flying Fish Empress", "Rainbow Dragonfish", "Heavenpiercer Turtle"},
        notes = {
            ["Flying Fish Emperor"]  = "Nguyên liệu chính",
            ["Flying Fish Empress"]  = "Nguyên liệu chính",
            ["Rainbow Dragonfish"]   = "Nguyên liệu + Trả Quest Hạ Diêu",
            ["Heavenpiercer Turtle"] = "Nguyên liệu + Mồi Rainbow",
        }
    },
    {
        rod = "Pure Diamond Rod",
        desc = "Cần Kim Cương Thuần (Cao Cấp)",
        bosses = {"Frost Kingfish", "Frost Queenfish", "Sanguine Fish", "Draconic Koi"},
        notes = {
            ["Frost Kingfish"]  = "Nguyên liệu + Mồi Frost",
            ["Frost Queenfish"] = "Nguyên liệu chính",
            ["Sanguine Fish"]   = "Nguyên liệu chính",
            ["Draconic Koi"]    = "Nguyên liệu phụ",
        }
    },
    {
        rod = "Sacred Bamboo Rod",
        desc = "Cần Trúc Thánh (Cao Cấp)",
        bosses = {"Nameless Octoparasite", "Reborn Puffer Beast", "Mountain Dragonwhale"},
        notes = {
            ["Nameless Octoparasite"] = "Nguyên liệu + Trả Quest Đạo Sĩ",
            ["Reborn Puffer Beast"]   = "Nguyên liệu + Trả Quest",
            ["Mountain Dragonwhale"]  = "Nguyên liệu + Mồi Nameless",
        }
    },
    {
        rod = "Huyết Long Rod",
        desc = "Cần Huyết Long (Trung Cấp)",
        bosses = {"Scarlet Fish", "Elder Scarlet Fish", "Verdant Bonefang", "Verdant Alligator Gar"},
        notes = {
            ["Scarlet Fish"]          = "Nguyên liệu cơ bản",
            ["Elder Scarlet Fish"]    = "Nguyên liệu chính",
            ["Verdant Bonefang"]      = "Nguyên liệu phụ",
            ["Verdant Alligator Gar"] = "Nguyên liệu phụ",
        }
    },
    {
        rod = "Rainbow Bait (Mồi)",
        desc = "Mồi Rainbow Bait (Gọi Boss Rùa)",
        bosses = {"Crimson Electric Eel", "Colossal Tigerfish", "Golden Guardian Fish"},
        notes = {
            ["Crimson Electric Eel"]  = "Nguyên liệu chính",
            ["Colossal Tigerfish"]    = "Nguyên liệu chính",
            ["Golden Guardian Fish"]  = "Nguyên liệu phụ",
        }
    },
    {
        rod = "Nameless Bait (Mồi)",
        desc = "Mồi Nameless Bait (Gọi Bạch Tuộc)",
        bosses = {"Mirage Lanternfish", "Tiger Mirefish", "Octoparasitic Fish"},
        notes = {
            ["Mirage Lanternfish"]  = "Nguyên liệu chính",
            ["Tiger Mirefish"]      = "Nguyên liệu phụ",
            ["Octoparasitic Fish"]  = "Nguyên liệu chế mồi",
        }
    },
}

for _, recipe in ipairs(rodRecipes) do
    -- Hiển thị từng cần câu như một info row
    local bossListStr = table.concat(recipe.bosses, "  •  ")
    createInfoRow(rodGuideCard, "🪝 " .. recipe.rod, recipe.desc)

    for _, bName in ipairs(recipe.bosses) do
        local note = recipe.notes[bName] or ""
        createInfoRow(rodGuideCard, "   ↳ " .. bName, note)
    end

    -- Nút lọc nhanh: chỉ bật những boss cần cho cần này
    createButtonRow(rodGuideCard,
        "Ưu Tiên Chỉ Săn Cho: " .. recipe.rod,
        "Tắt hết boss khác, chỉ bật những boss cần để ráp " .. recipe.rod,
        "⚡ Chỉ Săn Cần Này",
        function()
            -- Tắt hết
            for bName, _ in pairs(Config.SecretBossTargets) do
                Config.SecretBossTargets[bName] = false
                if bossTogglesMap[bName] and bossTogglesMap[bName].Set then
                    pcall(function() bossTogglesMap[bName].Set(false, true) end)
                end
            end
            -- Bật những boss cần cho cần này
            local enabled = {}
            for _, bName in ipairs(recipe.bosses) do
                Config.SecretBossTargets[bName] = true
                -- Xử lý alias Heavenpiercer Turtle
                if bName:find("Heaven") then
                    Config.SecretBossTargets["Heavenpiercer Turtle"] = true
                    Config.SecretBossTargets["Heaven Piercer Turtle"] = true
                    if bossTogglesMap["Heavenpiercer Turtle"] and bossTogglesMap["Heavenpiercer Turtle"].Set then
                        pcall(function() bossTogglesMap["Heavenpiercer Turtle"].Set(true, true) end)
                    end
                end
                if bossTogglesMap[bName] and bossTogglesMap[bName].Set then
                    pcall(function() bossTogglesMap[bName].Set(true, true) end)
                end
                table.insert(enabled, bName)
            end
            SaveBossTargets()
            ShowNotification(
                "Đã Lọc Boss Cho: " .. recipe.rod,
                "Chỉ săn: " .. table.concat(enabled, ", "),
                "SUCCESS", 6
            )
        end
    )
end

-- Nút khôi phục tất cả
createButtonRow(rodGuideCard, "Bật Lại Tất Cả Secret Boss", "Bật lại toàn bộ secret boss sau khi đã lọc theo cần câu", "↩ Bật Hết Lại", function()
    for bName, _ in pairs(Config.SecretBossTargets) do
        Config.SecretBossTargets[bName] = true
        if bossTogglesMap[bName] and bossTogglesMap[bName].Set then
            pcall(function() bossTogglesMap[bName].Set(true, true) end)
        end
    end
    SaveBossTargets()
    ShowNotification("Đã Bật Hết", "Đã bật lại toàn bộ secret boss!", "SUCCESS")
end)
end

do
    createCategoryHeader(tabBoss, "Đấu Trường Boss Enzo")
    local bossFarmCard = createCardGroup(tabBoss)
    createToggleRow(bossFarmCard, "Tự Động Săn Boss (Enzo)", "Liên tục triệu hồi và đánh bại boss Enzo", Config.AutoFarmBoss, function(v) Config.AutoFarmBoss = v end)
    createToggleRow(bossFarmCard, "Tự Săn Secret Boss (Bạch Tuộc)", "Tự chế mồi Nameless Bait, triệu hồi và tiêu diệt", Config.AutoFarmSecretBoss, function(v) Config.AutoFarmSecretBoss = v end)

    createButtonRow(bossFarmCard, "Bay Đến Boss Enzo", "Dịch chuyển trực tiếp đến đấu trường Enzo", "Bay Đến", function()
        local char = LocalPlayer.Character
        local root = char and char:FindFirstChild("HumanoidRootPart")
        if root then
            root.CFrame = CFrame.new(-115.3, 9.2, 1349.5)
            ShowNotification("Dịch Chuyển", "Đã đến Đấu trường Boss Enzo!", "SUCCESS")
        end
    end)
end

-------------------------------------------------------------------------
-- TAB QUẢN LÝ CÁ (FISH MANAGER PRO) - TOÀN DIỆN CHẾ CẦN, CHẾ MỒI & DỌN RÁC
-------------------------------------------------------------------------
do
    local FM = {
        isProcessing = false,
        spyEnabled = false,
        lastSpyInfo = "Chưa có tín hiệu nào",
        FishImageCache = {},
        currentSelectedFish = nil,
        rodControllers = {},
        baitControllers = {}
    }

    local function MakeCorner(parent, r)
        local c = Instance.new("UICorner")
        c.CornerRadius = UDim.new(0, r or 8)
        c.Parent = parent
        return c
    end

    local function MakeStroke(parent, col, thick)
        local s = Instance.new("UIStroke")
        s.Color = col or Color3.fromRGB(60, 65, 80)
        s.Thickness = thick or 1
        s.Parent = parent
        return s
    end

    -- Helper trích xuất icon cá thật từ Button, tuyệt đối tránh nhầm vào icon ổ khóa, hào quang trắng (Detail.Detail) hoặc stencil gradient
    local function ExtractFishImageFromButton(btn)
        if not btn then return nil end

        -- 1. Ưu tiên số 1: btn:FindFirstChild("Image") - Đây là Sprite cá chuẩn trong CraftRod, CraftBait và Inventory
        local directImg = btn:FindFirstChild("Image")
        if directImg and directImg:IsA("ImageLabel") and directImg.Image ~= "" then
            local imgUrl = directImg.Image
            if not imgUrl:find("10709791437") 
                and not imgUrl:lower():find("lock") 
                and not directImg:FindFirstChildOfClass("UIGradient") then
                return imgUrl
            end
        end

        -- 2. Ưu tiên số 2: btn:FindFirstChild("Icon") nếu có
        local directIcon = btn:FindFirstChild("Icon")
        if directIcon and directIcon:IsA("ImageLabel") and directIcon.Image ~= "" then
            local imgUrl = directIcon.Image
            if not imgUrl:find("10709791437") 
                and not imgUrl:lower():find("lock") 
                and not directIcon:FindFirstChildOfClass("UIGradient") then
                return imgUrl
            end
        end

        -- 3. Quét các ImageLabel con khác trong btn nhưng TUYỆT ĐỐI KHÔNG LẤY:
        -- - Detail.Detail (vầng hào quang tròn / white glow circle)
        -- - Bất kỳ ảnh nào có UIGradient (hoa văn lượn sóng shimmer)
        -- - Icon ổ khóa 10709791437
        -- - Icon ngôi sao yêu thích Star / Shine / Favorite
        for _, desc in ipairs(btn:GetDescendants()) do
            if desc:IsA("ImageLabel") and desc.Image ~= "" then
                local n = desc.Name:lower()
                local pName = desc.Parent and desc.Parent.Name:lower() or ""
                local hasGradient = desc:FindFirstChildOfClass("UIGradient") ~= nil
                local imgUrl = desc.Image

                -- Loại trừ vầng hào quang tròn màu trắng Detail.Detail
                local isDetailGlow = (n == "detail" and pName == "detail") or (n == "detail" and desc:FindFirstChildOfClass("UICorner") ~= nil)
                local isLock = imgUrl:find("10709791437") or n:find("lock")
                local isStarOrShine = n:find("star") or n:find("shine") or n:find("fav") or pName:find("fav")

                if not hasGradient
                    and not isDetailGlow
                    and not isLock
                    and not isStarOrShine then
                    return imgUrl
                end
            end
        end

        return nil
    end

    local isPreloadingImages = false
    local lastPreloadTime = 0

    -- Hàm quét và nạp ảnh cá nhẹ nhàng ngầm, không block thread và chỉ chạy tối đa 1 lần / 4 giây
    local function PreloadFishImages()
        if isPreloadingImages or (tick() - lastPreloadTime < 4) then return end
        isPreloadingImages = true
        lastPreloadTime = tick()

        task.spawn(function()
            pcall(function()
                local pGui = LocalPlayer:FindFirstChild("PlayerGui")
                local mainGui = pGui and pGui:FindFirstChild("MainGui")
                if not mainGui then return end

                -- 1. Tìm trong Menu Chế Cần (CraftRod.List)
                local craftRodList = mainGui:FindFirstChild("Menu")
                    and mainGui.Menu:FindFirstChild("CraftRod")
                    and mainGui.Menu.CraftRod:FindFirstChild("List")
                if craftRodList then
                    for _, rFrame in ipairs(craftRodList:GetChildren()) do
                        local ingFrame = rFrame:FindFirstChild("Ingredient")
                        if ingFrame then
                            for _, slot in ipairs(ingFrame:GetChildren()) do
                                local btn = slot:FindFirstChild("Button") or (slot:IsA("TextButton") and slot)
                                local titleLbl = btn and btn:FindFirstChild("Title") or slot:FindFirstChild("Title")
                                local fishImg = ExtractFishImageFromButton(btn)
                                if titleLbl and fishImg and fishImg ~= "" then
                                    local k = titleLbl.Text:lower():gsub("[%s%-_]+", "")
                                    if k ~= "" and (not FM.FishImageCache[k] or FM.FishImageCache[k] == "") then
                                        FM.FishImageCache[k] = fishImg
                                    end
                                end
                            end
                        end
                    end
                end

                -- 2. Tìm trong Menu Chế Mồi (CraftBait.List)
                local craftBaitList = mainGui:FindFirstChild("Menu")
                    and mainGui.Menu:FindFirstChild("CraftBait")
                    and mainGui.Menu.CraftBait:FindFirstChild("List")
                if craftBaitList then
                    for _, bFrame in ipairs(craftBaitList:GetChildren()) do
                        local ingFrame = bFrame:FindFirstChild("Ingredient")
                        if ingFrame then
                            for _, slot in ipairs(ingFrame:GetChildren()) do
                                local btn = slot:FindFirstChild("Button") or (slot:IsA("TextButton") and slot)
                                local titleLbl = btn and btn:FindFirstChild("Title") or slot:FindFirstChild("Title")
                                local fishImg = ExtractFishImageFromButton(btn)
                                if titleLbl and fishImg and fishImg ~= "" then
                                    local k = titleLbl.Text:lower():gsub("[%s%-_]+", "")
                                    if k ~= "" and (not FM.FishImageCache[k] or FM.FishImageCache[k] == "") then
                                        FM.FishImageCache[k] = fishImg
                                    end
                                end
                            end
                        end
                    end
                end

                -- 3. Tìm trong Balo người chơi (Main.Inventory.Main.List.ScrollingFrame)
                local scroll = mainGui:FindFirstChild("Main", true)
                    and mainGui.Main:FindFirstChild("Inventory", true)
                    and mainGui.Main.Inventory:FindFirstChild("ScrollingFrame", true)
                if scroll then
                    for _, slot in ipairs(scroll:GetChildren()) do
                        local btn = slot:FindFirstChild("Button")
                        if not btn and slot:IsA("Folder") and #slot:GetChildren() > 0 then
                            btn = slot:GetChildren()[1]:FindFirstChild("Button")
                        end
                        local fishImg = ExtractFishImageFromButton(btn)
                        if fishImg and fishImg ~= "" then
                            local rawName = Wiki.GetItemRawName(slot)
                            local k = rawName:lower():gsub("[%s%-_]+", "")
                            if k ~= "" and (not FM.FishImageCache[k] or FM.FishImageCache[k] == "") then
                                FM.FishImageCache[k] = fishImg
                            end
                        end
                    end
                end
            end)

            isPreloadingImages = false
        end)
    end

    local function FetchGameFishImage(fishName)
        if not fishName or fishName == "" then return "" end
        local clean = fishName:lower():gsub("[%s%-_]+", "")
        local cached = FM.FishImageCache[clean]
        if cached and cached ~= "" and not cached:find("10709791437") then
            return cached
        end

        -- 1. Tìm trực tiếp ModuleScript tương ứng loài cá trong ReplicatedStorage.Info.Inventory (O(1), 0.001ms)
        local infoInv = ReplicatedStorage:FindFirstChild("Info") and ReplicatedStorage.Info:FindFirstChild("Inventory")
        if infoInv then
            local mod = infoInv:FindFirstChild(fishName)
            if not mod then
                for _, m in ipairs(infoInv:GetChildren()) do
                    if m.Name:lower():gsub("[%s%-_]+", "") == clean then
                        mod = m
                        break
                    end
                end
            end
            if mod and mod:IsA("ModuleScript") then
                local s, data = pcall(require, mod)
                if s and type(data) == "table" then
                    local targetImg = data.Image or data.Icon or data.Thumbnail or data.image or data.icon
                    if targetImg then
                        if tonumber(targetImg) then
                            targetImg = "rbxassetid://" .. targetImg
                        end
                        FM.FishImageCache[clean] = targetImg
                        return targetImg
                    end
                end
            end
        end

        -- 2. Kích hoạt quét nền GUI nếu chưa có
        PreloadFishImages()
        return FM.FishImageCache[clean] or ""
    end

    -- Bảng dữ liệu công thức Chế Cần
    local ROD_RECIPES = {
        {
            key = "Heavenpiercer Rod",
            vietName = "Cần Xuyên Thiên (Heavenpiercer)",
            desc = "Cần Thần Thoại (Trời Gió & Nước Ngầm)",
            ingredients = {
                { name = "Flying Fish Emperor", origin = "Đảo Cá Chép (Trời Gió)" },
                { name = "Flying Fish Empress", origin = "Đảo Cá Chép (Trời Gió)" },
                { name = "Heavenpiercer Turtle", origin = "Đảo Dừa (Sương Mù)" },
                { name = "Rainbow Dragonfish", origin = "Nước Ngầm (Động Tối)" },
            }
        },
        {
            key = "Sacred Bamboo Rod",
            vietName = "Cần Trúc Thánh (Sacred Bamboo)",
            desc = "Cần Thần Đỉnh Núi & Biển Sâu",
            ingredients = {
                { name = "Nameless Octoparasite", origin = "Phao Biển Sâu" },
                { name = "Reborn Puffer Beast", origin = "Đảo Băng (Bão Tuyết)" },
                { name = "Ascended Perch", origin = "Đảo Cá Chép" },
                { name = "Mountain Fish", origin = "Đảo Đỉnh Sương Mù" },
            }
        },
        {
            key = "Pure Diamond Rod",
            vietName = "Cần Kim Cương Thuần Khiết (Pure Diamond)",
            desc = "Cần Bão Tuyết & Nắng Gắt",
            ingredients = {
                { name = "Frost Kingfish", origin = "Đảo Băng (Bão Tuyết)" },
                { name = "Frost Queenfish", origin = "Đảo Băng (Bão Tuyết)" },
                { name = "Sanguine Fish", origin = "Đảo Hổ Phách (Nắng Gắt)" },
                { name = "Draconic Koi", origin = "Đảo Hổ Phách (Nắng Gắt)" },
            }
        }
    }

    -- Bảng dữ liệu công thức Chế Mồi
    local BAIT_RECIPES = {
        {
            key = "Nameless Bait",
            vietName = "Mồi Vô Danh (Nameless Bait)",
            desc = "Triệu hồi Boss Bạch Tuộc Biển Sâu",
            ingredients = {
                { name = "Mountain Fish", origin = "Đảo Đỉnh Sương Mù" },
                { name = "Octoparasitic Fish", origin = "Phao Biển Sâu" },
                { name = "Mirage Lanternfish", origin = "Đảo Dừa" },
                { name = "Tiger Mirefish", origin = "Đảo Tre" },
            }
        },
        {
            key = "Frost Bait",
            vietName = "Mồi Băng Giá (Frost Bait)",
            desc = "Triệu hồi Boss Đấu Trường Băng",
            ingredients = {
                { name = "Primordial Kunfish Overlord", origin = "Đấu Trường Boss Realm" },
                { name = "Warbringer Shark", origin = "Đấu Trường Boss Realm" },
                { name = "Frost Kingfish", origin = "Đảo Băng (Bão Tuyết)" },
                { name = "Ascended Perch", origin = "Đảo Cá Chép" },
            }
        },
        {
            key = "Rainbow Bait",
            vietName = "Mồi Thất Sắc (Rainbow Bait)",
            desc = "Triệu hồi Boss Rồng Thần Thoại",
            ingredients = {
                { name = "Heavenpiercer Turtle", origin = "Đảo Dừa (Sương Mù)" },
                { name = "Colossal Tigerfish", origin = "Đảo Chiến Trường" },
                { name = "Crimson Electric Eel", origin = "Đảo Tre (Bão Sấm)" },
                { name = "Golden Guardian Fish", origin = "Đảo Thống Trị" },
            }
        }
    }

    -- Kiểm tra người chơi đã sở hữu cần câu chưa
    local function CheckRodOwnership(rodName)
        local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
        if not pData then return false, 0 end

        local isOwned = false
        local count = 0

        local rodInv = pData:FindFirstChild("FishingRodInventory")
        if rodInv then
            local rodFolder = rodInv:FindFirstChild(rodName)
            if rodFolder and rodFolder:FindFirstChild("Owned") and rodFolder.Owned.Value == true then
                isOwned = true
                count = count + 1
            end
        end

        local skinCount = pData:FindFirstChild("RodSkinCount")
        if skinCount then
            local skinVal = skinCount:FindFirstChild(rodName)
            if skinVal and tonumber(skinVal.Value) and skinVal.Value > 0 then
                isOwned = true
                count = math.max(count, tonumber(skinVal.Value))
            end
        end

        return isOwned, count
    end

    -- Quét và phân loại toàn diện túi cá (chuẩn theo contract của game)
    local function ScanAndClassifyInventory()
        local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
        local inv = pData and pData:FindFirstChild("Inventory")

        local protectedList = {}
        local junkList = {}
        local countsByName = {}

        if not inv then
            return protectedList, junkList, countsByName
        end

        for _, item in ipairs(inv:GetChildren()) do
            local cleanName = Wiki.GetItemRawName(item)
            local cleanLower = cleanName:lower()
            local weight = (item:FindFirstChild("Weight") and item.Weight.Value) or 0
            local isFav = Wiki.IsItemFavorited(item)

            local entry = {
                data = item,
                name = cleanName,
                rawLower = cleanLower,
                weight = weight,
                isLocked = isFav
            }

            if not countsByName[cleanName] then
                countsByName[cleanName] = {
                    name = cleanName,
                    total = 0,
                    locked = 0,
                    unlocked = 0,
                    items = {}
                }
            end
            countsByName[cleanName].total = countsByName[cleanName].total + 1
            if isFav then
                countsByName[cleanName].locked = countsByName[cleanName].locked + 1
            else
                countsByName[cleanName].unlocked = countsByName[cleanName].unlocked + 1
            end
            table.insert(countsByName[cleanName].items, entry)

            local isProtected, reason = Wiki.IsProtectedFish(item)
            entry.category = reason
            if isProtected then
                table.insert(protectedList, entry)
            else
                table.insert(junkList, entry)
            end
        end

        return protectedList, junkList, countsByName
    end

    -- Helper trigger button trong UI game (mượt mà như bản đầu)
    local function ClickButton(btn)
        if not btn then return false end
        local triggered = false
        if typeof(firesignal) == "function" then
            pcall(function() firesignal(btn.MouseButton1Click); triggered = true end)
            pcall(function() firesignal(btn.MouseButton1Down) end)
            pcall(function() firesignal(btn.MouseButton1Up) end)
            pcall(function() firesignal(btn.Activated); triggered = true end)
        end
        if typeof(getconnections) == "function" then
            local ok1, conns1 = pcall(getconnections, btn.MouseButton1Click)
            if ok1 and type(conns1) == "table" then
                for _, c in ipairs(conns1) do
                    pcall(function() c:Fire(); triggered = true end)
                end
            end
            local ok2, conns2 = pcall(getconnections, btn.Activated)
            if ok2 and type(conns2) == "table" then
                for _, c in ipairs(conns2) do
                    pcall(function() c:Fire(); triggered = true end)
                end
            end
        end
        return triggered
    end

    -- Tìm button Favorite của con cá trong GUI Balo PlayerGui (MainGui / Fisher_GUI)
    local function FindGuiFavoriteButton(cleanName, weightInt)
        local pGui = LocalPlayer:FindFirstChild("PlayerGui")
        if not pGui then return nil, nil end

        local scrollFrames = {}
        local mainGui = pGui:FindFirstChild("MainGui")
        if mainGui then
            local mainInv = mainGui:FindFirstChild("Main", true) and mainGui.Main:FindFirstChild("Inventory", true)
            local scroll = mainInv and mainInv:FindFirstChild("ScrollingFrame", true)
            if scroll then table.insert(scrollFrames, scroll) end

            local fisherInv = mainGui:FindFirstChild("Fisher_Inventory", true)
            local fScroll = fisherInv and fisherInv:FindFirstChild("ScrollingFrame", true)
            if fScroll then table.insert(scrollFrames, fScroll) end
        end

        local fisherGui = pGui:FindFirstChild("Fisher_GUI")
        if fisherGui then
            local fScroll = fisherGui:FindFirstChild("ScrollingFrame", true)
            if fScroll then table.insert(scrollFrames, fScroll) end
        end

        local cleanTarget = cleanName:lower():gsub("[%s%-_]+", "")
        for _, scroll in ipairs(scrollFrames) do
            for _, child in ipairs(scroll:GetChildren()) do
                local cName = child.Name:lower()
                local cClean = cName:gsub("[%s%-_]+", "")
                if cClean:find(cleanTarget, 1, true) then
                    if not weightInt or cName:find(tostring(weightInt), 1, true) then
                        local fav = child:FindFirstChild("Favorite", true)
                        local btn = fav and (fav:FindFirstChildWhichIsA("TextButton") or (fav:IsA("TextButton") and fav))
                        if not btn then
                            btn = child:FindFirstChildWhichIsA("TextButton", true)
                        end
                        if btn then return btn, child end
                    end
                end
            end
        end
        return nil, nil
    end

    -- Thao tác Lock hoặc Unlock một danh sách item (Kết hợp Click GUI và Remote Đa Tầng)
    local function ProcessBatchItems(items, targetLock, notifyTitle)
        -- Cơ chế chống kẹt: tự reset sau 4s nếu tác vụ trước đó gặp trục trặc
        if FM.isProcessing then
            if tick() - (FM.lastProcessTime or 0) > 4 then
                FM.isProcessing = false
            else
                ShowNotification("Quản Lý Cá", "Đang trong tiến trình xử lý, vui lòng đợi!", "WARN", 2)
                return
            end
        end

        local favEvent = ReplicatedStorage:FindFirstChild("Events") and ReplicatedStorage.Events:FindFirstChild("FavoriteItem")

        local filtered = {}
        for _, e in ipairs(items) do
            local item = e.data or e
            if item and item.Parent then
                local curFav = Wiki.IsItemFavorited(item)
                if curFav ~= targetLock then
                    local clean = e.name or Wiki.GetItemRawName(item)
                    local weightInt = tostring(item.Name):match("|%s*(%d+)")
                    table.insert(filtered, {
                        data = item,
                        clean = clean,
                        weightInt = weightInt,
                        curFav = curFav
                    })
                end
            end
        end

        if #filtered == 0 then
            ShowNotification(notifyTitle or "Quản Lý Cá", "Tất cả cá trong danh sách đã ở trạng thái mong muốn!", "INFO", 3)
            return
        end

        FM.isProcessing = true
        FM.lastProcessTime = tick()
        local actionText = targetLock and "KHÓA" or "MỞ KHÓA"
        ShowNotification(notifyTitle or "Quản Lý Cá", string.format("Bắt đầu %s %d con cá...", actionText, #filtered), "INFO", 3)

        -- Quản lý bypass AutoProtect để tránh bị khóa lại sau khi vừa mở
        if not targetLock and Wiki and Wiki.temporarilyUnlockedBaitFish then
            for _, entry in ipairs(filtered) do
                Wiki.temporarilyUnlockedBaitFish[entry.clean:lower()] = true
            end
        elseif targetLock and Wiki and Wiki.temporarilyUnlockedBaitFish then
            for _, entry in ipairs(filtered) do
                Wiki.temporarilyUnlockedBaitFish[entry.clean:lower()] = nil
            end
        end

        task.spawn(function()
            local successCount = 0
            for _, entry in ipairs(filtered) do
                local item = entry.data
                if item and item.Parent then
                    -- Cách 1: Click trực tiếp nút GUI trong Balo game (cực mượt như bản đầu)
                    local btn, guiChild = FindGuiFavoriteButton(entry.clean, entry.weightInt)
                    if btn then
                        ClickButton(btn)
                    end

                    -- Cách 2: Gửi Remote Event đa tầng làm fallback đồng bộ
                    if favEvent then
                        pcall(function() favEvent:FireServer(item) end)
                        pcall(function() favEvent:FireServer(item.Name) end)
                        if guiChild then
                            pcall(function() favEvent:FireServer(guiChild) end)
                        end
                    end

                    successCount = successCount + 1
                    task.wait(0.08)
                end
            end

            task.wait(0.4)
            FM.isProcessing = false
            pcall(function()
                if UpdateCrimsonBreamUI then UpdateCrimsonBreamUI() end
            end)
            ShowNotification("Hoàn Tất", string.format("Đã %s thành công %d/%d con cá!", actionText, successCount, #filtered), "SUCCESS", 4)
        end)
    end

    -- Bán chuyên biệt một loài cá theo số lượng an toàn tuyệt đối
    local function SellSpecificFish(targetFishName, quantityToSell)
        if FM.isProcessing then
            ShowNotification("Bán Cá", "Đang bận xử lý tác vụ khác, vui lòng đợi!", "WARN", 3)
            return
        end

        local protList, junkList, counts = ScanAndClassifyInventory()
        local cData = counts[targetFishName]
        if not cData or #cData.items == 0 then
            ShowNotification("Bán Cá", "Không tìm thấy con [" .. tostring(targetFishName) .. "] nào trong balo!", "WARN", 4)
            return
        end

        local qty = tonumber(quantityToSell) or 0
        local targetItems = {}

        if qty <= 0 or qty >= #cData.items then
            targetItems = cData.items
        else
            for i = 1, qty do
                table.insert(targetItems, cData.items[i])
            end
        end

        FM.isProcessing = true
        FM.lastProcessTime = tick()
        ShowNotification("Bán Cá An Toàn", string.format("Đang chuẩn bị bán %d con [%s]...", #targetItems, targetFishName), "INFO", 3)

        task.spawn(function()
            local ok, err = pcall(function()
                local favEvent = ReplicatedStorage:FindFirstChild("Events") and ReplicatedStorage.Events:FindFirstChild("FavoriteItem")

                -- 1. Khóa tất cả cá KHÁC loài này
                for name, data in pairs(counts) do
                    if name ~= targetFishName then
                        for _, it in ipairs(data.items) do
                            if it.data and it.data.Parent and not Wiki.IsItemFavorited(it.data) and favEvent then
                                pcall(function() favEvent:FireServer(it.data) end)
                                task.wait(0.05)
                            end
                        end
                    end
                end

                -- 2. Mở khóa đúng số lượng cá muốn bán
                if Wiki and Wiki.temporarilyUnlockedBaitFish then
                    Wiki.temporarilyUnlockedBaitFish[targetFishName:lower()] = true
                end
                for _, t in ipairs(targetItems) do
                    if t.data and t.data.Parent and Wiki.IsItemFavorited(t.data) and favEvent then
                        pcall(function() favEvent:FireServer(t.data) end)
                        task.wait(0.05)
                    end
                end

                -- Khóa lại các con còn lại của loài này nếu bán một phần
                if #targetItems < #cData.items then
                    for i = #targetItems + 1, #cData.items do
                        local remain = cData.items[i]
                        if remain.data and remain.data.Parent and not Wiki.IsItemFavorited(remain.data) and favEvent then
                            pcall(function() favEvent:FireServer(remain.data) end)
                            task.wait(0.05)
                        end
                    end
                end

                task.wait(0.3)

                -- 3. Gọi lệnh bán cá
                local sellEvent = ReplicatedStorage:FindFirstChild("Events") and ReplicatedStorage.Events:FindFirstChild("SellFish")
                if sellEvent then
                    sellEvent:FireServer("All")
                    ShowNotification("Bán Cá Thành Công", string.format("Đã bán %d con [%s]! Cá khác được giữ an toàn 100%%!", #targetItems, targetFishName), "SUCCESS", 5)
                else
                    ShowNotification("Lỗi Bán Cá", "Không tìm thấy Remote SellFish!", "ERROR", 4)
                end
            end)

            task.wait(0.5)
            FM.isProcessing = false
            pcall(function()
                if UpdateCrimsonBreamUI then UpdateCrimsonBreamUI() end
            end)
        end)
    end

    -- Helper tạo khung 4 ô ảnh cá
    local function CreateVisualIngredientGrid(parent, ingredients)
        local gridContainer = Instance.new("Frame")
        gridContainer.Name = "IngredientVisualGrid"
        gridContainer.Size = UDim2.new(1, 0, 0, 94)
        gridContainer.BackgroundTransparency = 1
        gridContainer.BorderSizePixel = 0
        gridContainer.Parent = parent

        local listLayout = Instance.new("UIListLayout")
        listLayout.FillDirection = Enum.FillDirection.Horizontal
        listLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
        listLayout.VerticalAlignment = Enum.VerticalAlignment.Center
        listLayout.Padding = UDim.new(0, 6)
        listLayout.SortOrder = Enum.SortOrder.LayoutOrder
        listLayout.Parent = gridContainer

        local slots = {}
        local numIng = math.max(1, #ingredients)

        for i, ing in ipairs(ingredients) do
            local slotFrame = Instance.new("Frame")
            slotFrame.Name = "Slot_" .. i
            slotFrame.Size = UDim2.new(1 / numIng, -7, 1, 0)
            slotFrame.BackgroundColor3 = Color3.fromRGB(24, 26, 36)
            slotFrame.BorderSizePixel = 0
            slotFrame.LayoutOrder = i
            slotFrame.Parent = gridContainer

            MakeCorner(slotFrame, 8)
            local stroke = MakeStroke(slotFrame, Color3.fromRGB(60, 65, 80), 1.2)

            local img = Instance.new("ImageLabel")
            img.Name = "FishImage"
            img.Size = UDim2.new(0, 40, 0, 40)
            img.AnchorPoint = Vector2.new(0.5, 0)
            img.Position = UDim2.new(0.5, 0, 0, 6)
            img.BackgroundTransparency = 1
            img.ScaleType = Enum.ScaleType.Fit
            img.Image = FetchGameFishImage(ing.name)
            img.Parent = slotFrame

            local fallbackEmoji = Instance.new("TextLabel")
            fallbackEmoji.Name = "FallbackEmoji"
            fallbackEmoji.Size = UDim2.new(1, 0, 0, 36)
            fallbackEmoji.AnchorPoint = Vector2.new(0.5, 0)
            fallbackEmoji.Position = UDim2.new(0.5, 0, 0, 6)
            fallbackEmoji.BackgroundTransparency = 1
            fallbackEmoji.Text = "🐟"
            fallbackEmoji.TextSize = 22
            fallbackEmoji.Visible = (img.Image == "" or img.Image == nil)
            fallbackEmoji.Parent = slotFrame

            local nameLbl = Instance.new("TextLabel")
            nameLbl.Name = "FishName"
            nameLbl.Size = UDim2.new(1, -6, 0, 18)
            nameLbl.Position = UDim2.new(0, 3, 0, 48)
            nameLbl.BackgroundTransparency = 1
            nameLbl.Text = ing.name
            nameLbl.TextColor3 = Color3.fromRGB(200, 205, 220)
            nameLbl.Font = Enum.Font.GothamMedium
            nameLbl.TextSize = 9
            nameLbl.TextTruncate = Enum.TextTruncate.AtEnd
            nameLbl.Parent = slotFrame

            local countLbl = Instance.new("TextLabel")
            countLbl.Name = "CountBadge"
            countLbl.Size = UDim2.new(1, 0, 0, 22)
            countLbl.Position = UDim2.new(0, 0, 0, 68)
            countLbl.BackgroundTransparency = 1
            countLbl.Text = "x0"
            countLbl.TextColor3 = Color3.fromRGB(231, 76, 60)
            countLbl.Font = Enum.Font.GothamBold
            countLbl.TextSize = 13
            countLbl.Parent = slotFrame

            slots[i] = {
                frame = slotFrame,
                stroke = stroke,
                img = img,
                fallback = fallbackEmoji,
                nameLbl = nameLbl,
                countLbl = countLbl,
                ing = ing
            }
        end

        local gridObj = {}
        function gridObj.Update(countsByName)
            for _, s in ipairs(slots) do
                -- Tìm đếm khớp tên không phân biệt hoa thường
                local foundCount = 0
                local ingLower = s.ing.name:lower():gsub("[%s%-_]+", "")
                for name, cData in pairs(countsByName) do
                    if name:lower():gsub("[%s%-_]+", "") == ingLower then
                        foundCount = cData.total
                        break
                    end
                end

                local isEnough = foundCount >= 1
                s.countLbl.Text = "x" .. tostring(foundCount)
                if isEnough then
                    s.countLbl.TextColor3 = Color3.fromRGB(46, 204, 113)
                    s.stroke.Color = Color3.fromRGB(46, 204, 113)
                    s.frame.BackgroundColor3 = Color3.fromRGB(20, 36, 28)
                else
                    s.countLbl.TextColor3 = Color3.fromRGB(231, 76, 60)
                    s.stroke.Color = Color3.fromRGB(180, 50, 50)
                    s.frame.BackgroundColor3 = Color3.fromRGB(32, 22, 24)
                end
                local curImg = FetchGameFishImage(s.ing.name)
                if curImg ~= "" and curImg ~= s.img.Image then
                    s.img.Image = curImg
                    s.fallback.Visible = false
                elseif s.img.Image == "" then
                    s.fallback.Visible = true
                end
            end
        end

        return gridObj
    end

    ---------------------------------------------------------------------
    -- KHỐI 1 (ĐẦU TAB): QUẢN LÝ TÚI CÁ RÁC & TÌM KIẾM THAO TÁC TRỰC TIẾP
    ---------------------------------------------------------------------
    do
        createCategoryHeader(tabFishManager, "🛡️ QUẢN LÝ TÚI CÁ & DỌN RÁC AN TOÀN")
        local bagOverviewCard = createCardGroup(tabFishManager)

        FM.rowBagTotal = createInfoRow(bagOverviewCard, "Tổng cá trong Balo", "Đang quét...")
        FM.rowBagProtected = createInfoRow(bagOverviewCard, "Cá quý đang bảo vệ", "...")
        FM.rowBagJunk = createInfoRow(bagOverviewCard, "Cá rác có thể bán an toàn", "...")

        createButtonRow(bagOverviewCard, "Khóa Toàn Bộ Cá Quý", "Bảo vệ tuyệt đối cá Boss, Cần, Mồi, VIP, Đột Biến", "🔒 Khóa Cá Quý", function()
            local protList, _, _ = ScanAndClassifyInventory()
            ProcessBatchItems(protList, true, "Bảo Vệ Cá Quý")
        end)

        createButtonRow(bagOverviewCard, "Mở Khóa Riêng Cá Rác", "Chỉ mở khóa các con cá rác thông thường", "🔓 Mở Khóa Rác", function()
            local _, junkList, _ = ScanAndClassifyInventory()
            ProcessBatchItems(junkList, false, "Mở Khóa Cá Rác")
        end)

        createButtonRow(bagOverviewCard, "Bán Sạch Cá Rác An Toàn", "Tự động kiểm tra an toàn và bán toàn bộ cá rác", "💰 Bán Sạch Rác", function()
            if FM.isProcessing then
                ShowNotification("Dọn Rác", "Đang bận xử lý tác vụ khác, vui lòng đợi!", "WARN", 3)
                return
            end

            local protList, junkList, _ = ScanAndClassifyInventory()
            if #junkList == 0 then
                ShowNotification("Dọn Rác", "Không có cá rác nào trong balo cần bán!", "INFO", 3)
                return
            end

            FM.isProcessing = true
            ShowNotification("Dọn Rác", string.format("Đang dọn dẹp %d con cá rác an toàn...", #junkList), "INFO", 4)

            task.spawn(function()
                local favEvent = ReplicatedStorage:FindFirstChild("Events") and ReplicatedStorage.Events:FindFirstChild("FavoriteItem")

                -- 1. Khóa toàn bộ cá quý nếu chưa khóa
                for _, p in ipairs(protList) do
                    if p.data and p.data.Parent and not Wiki.IsItemFavorited(p.data) and favEvent then
                        pcall(function() favEvent:FireServer(p.data) end)
                        task.wait(0.05)
                    end
                end
                task.wait(0.2)

                -- 2. Mở khóa cá rác nếu đang khóa
                for _, j in ipairs(junkList) do
                    if j.data and j.data.Parent and Wiki.IsItemFavorited(j.data) and favEvent then
                        pcall(function() favEvent:FireServer(j.data) end)
                        task.wait(0.05)
                    end
                end
                task.wait(0.4)

                -- 3. Bán
                local sellEvent = ReplicatedStorage:FindFirstChild("Events") and ReplicatedStorage.Events:FindFirstChild("SellFish")
                if sellEvent then
                    sellEvent:FireServer("All")
                    ShowNotification("Bán Cá Thành Công", string.format("Đã bán sạch %d con cá rác! Cá quý bảo vệ 100%%!", #junkList), "SUCCESS", 5)
                else
                    ShowNotification("Lỗi Bán Cá", "Không tìm thấy Remote SellFish!", "ERROR", 4)
                end

                task.wait(1.0)
                if UpdateCrimsonBreamUI then UpdateCrimsonBreamUI() end
                FM.isProcessing = false
            end)
        end)
    end

    ---------------------------------------------------------------------
    -- KHỐI 1B: BỘ TÌM KIẾM CÁ & GỢI Ý ĐẦY ĐỦ + BÁN TÙY CHỌN
    ---------------------------------------------------------------------
    do
        createCategoryHeader(tabFishManager, "🔍 TÌM KIẾM & THAO TÁC CÁ TRONG TÚI")
        local searchCard = createCardGroup(tabFishManager)

        -- Thanh tìm kiếm
        local searchBarContainer = Instance.new("Frame")
        searchBarContainer.Name = "SearchBarContainer"
        searchBarContainer.Size = UDim2.new(1, 0, 0, 36)
        searchBarContainer.BackgroundColor3 = Color3.fromRGB(18, 20, 28)
        searchBarContainer.BorderSizePixel = 0
        searchBarContainer.Parent = searchCard
        MakeCorner(searchBarContainer, 8)
        MakeStroke(searchBarContainer, Color3.fromRGB(60, 65, 80), 1)

        local sbIcon = Instance.new("TextLabel")
        sbIcon.Size = UDim2.new(0, 32, 1, 0)
        sbIcon.BackgroundTransparency = 1
        sbIcon.Text = "🔍"
        sbIcon.TextSize = 14
        sbIcon.TextColor3 = Color3.fromRGB(180, 185, 200)
        sbIcon.Parent = searchBarContainer

        local searchTextBox = Instance.new("TextBox")
        searchTextBox.Name = "SearchInput"
        searchTextBox.Size = UDim2.new(1, -40, 1, 0)
        searchTextBox.Position = UDim2.new(0, 34, 0, 0)
        searchTextBox.BackgroundTransparency = 1
        searchTextBox.PlaceholderText = "Gõ tên cá cần tìm (VD: crim, carp, koi, eel, catfish...)"
        searchTextBox.PlaceholderColor3 = Color3.fromRGB(120, 125, 140)
        searchTextBox.Text = ""
        searchTextBox.TextColor3 = Color3.fromRGB(255, 255, 255)
        searchTextBox.Font = Enum.Font.GothamMedium
        searchTextBox.TextSize = 12
        searchTextBox.ClearTextOnFocus = false
        searchTextBox.Parent = searchBarContainer
        FM.searchTextBox = searchTextBox

        -- Khung hiển thị danh sách gợi ý các loài cá khớp từ khóa
        local suggestionScroll = Instance.new("ScrollingFrame")
        suggestionScroll.Name = "SuggestionScroll"
        suggestionScroll.Size = UDim2.new(1, 0, 0, 34)
        suggestionScroll.BackgroundTransparency = 1
        suggestionScroll.BorderSizePixel = 0
        suggestionScroll.ScrollBarThickness = 2
        suggestionScroll.CanvasSize = UDim2.new(0, 0, 0, 0)
        suggestionScroll.AutomaticCanvasSize = Enum.AutomaticSize.X
        suggestionScroll.Parent = searchCard

        local suggLayout = Instance.new("UIListLayout")
        suggLayout.FillDirection = Enum.FillDirection.Horizontal
        suggLayout.Padding = UDim.new(0, 6)
        suggLayout.VerticalAlignment = Enum.VerticalAlignment.Center
        suggLayout.Parent = suggestionScroll

        -- Card hiển thị chi tiết con cá đang chọn
        local fishDetailFrame = Instance.new("Frame")
        fishDetailFrame.Name = "FishDetailFrame"
        fishDetailFrame.Size = UDim2.new(1, 0, 0, 76)
        fishDetailFrame.BackgroundColor3 = Color3.fromRGB(24, 26, 36)
        fishDetailFrame.BorderSizePixel = 0
        fishDetailFrame.Parent = searchCard
        MakeCorner(fishDetailFrame, 8)
        MakeStroke(fishDetailFrame, Color3.fromRGB(65, 70, 88), 1)

        local fishImgIcon = Instance.new("ImageLabel")
        fishImgIcon.Name = "FishIcon"
        fishImgIcon.Size = UDim2.new(0, 54, 0, 54)
        fishImgIcon.Position = UDim2.new(0, 10, 0, 11)
        fishImgIcon.BackgroundTransparency = 1
        fishImgIcon.ScaleType = Enum.ScaleType.Fit
        fishImgIcon.Parent = fishDetailFrame

        local fishEmojiIcon = Instance.new("TextLabel")
        fishEmojiIcon.Name = "EmojiIcon"
        fishEmojiIcon.Size = UDim2.new(0, 54, 0, 54)
        fishEmojiIcon.Position = UDim2.new(0, 10, 0, 11)
        fishEmojiIcon.BackgroundTransparency = 1
        fishEmojiIcon.Text = "🐟"
        fishEmojiIcon.TextSize = 28
        fishEmojiIcon.Parent = fishDetailFrame

        local fishTitleLbl = Instance.new("TextLabel")
        fishTitleLbl.Name = "FishTitle"
        fishTitleLbl.Size = UDim2.new(1, -78, 0, 22)
        fishTitleLbl.Position = UDim2.new(0, 72, 0, 8)
        fishTitleLbl.BackgroundTransparency = 1
        fishTitleLbl.Text = "Gõ tên cá vào ô tìm kiếm..."
        fishTitleLbl.TextColor3 = Color3.fromRGB(255, 215, 0)
        fishTitleLbl.Font = Enum.Font.GothamBold
        fishTitleLbl.TextSize = 13
        fishTitleLbl.TextXAlignment = Enum.TextXAlignment.Left
        fishTitleLbl.Parent = fishDetailFrame

        local fishStatusLbl = Instance.new("TextLabel")
        fishStatusLbl.Name = "FishStatus"
        fishStatusLbl.Size = UDim2.new(1, -78, 0, 40)
        fishStatusLbl.Position = UDim2.new(0, 72, 0, 30)
        fishStatusLbl.BackgroundTransparency = 1
        fishStatusLbl.Text = "Gợi ý toàn bộ loài cá khớp từ khóa sẽ hiển thị phía trên để bạn bấm chọn."
        fishStatusLbl.TextColor3 = Color3.fromRGB(180, 185, 200)
        fishStatusLbl.Font = Enum.Font.Gotham
        fishStatusLbl.TextSize = 11
        fishStatusLbl.TextWrapped = true
        fishStatusLbl.TextXAlignment = Enum.TextXAlignment.Left
        fishStatusLbl.Parent = fishDetailFrame

        -- Hàng 2 Nút Khóa / Mở Khóa loài cá đang chọn
        local actionRowContainer = Instance.new("Frame")
        actionRowContainer.Name = "ActionRowContainer"
        actionRowContainer.Size = UDim2.new(1, 0, 0, 36)
        actionRowContainer.BackgroundTransparency = 1
        actionRowContainer.Parent = searchCard

        local arLayout = Instance.new("UIListLayout")
        arLayout.FillDirection = Enum.FillDirection.Horizontal
        arLayout.Padding = UDim.new(0, 8)
        arLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
        arLayout.Parent = actionRowContainer

        local btnLockThis = Instance.new("TextButton")
        btnLockThis.Size = UDim2.new(0.5, -4, 1, 0)
        btnLockThis.BackgroundColor3 = Color3.fromRGB(30, 42, 60)
        btnLockThis.Text = "🔒 Khóa Loài Này"
        btnLockThis.TextColor3 = Color3.fromRGB(240, 240, 255)
        btnLockThis.Font = Enum.Font.GothamBold
        btnLockThis.TextSize = 12
        btnLockThis.Parent = actionRowContainer
        MakeCorner(btnLockThis, 8)

        local btnUnlockThis = Instance.new("TextButton")
        btnUnlockThis.Size = UDim2.new(0.5, -4, 1, 0)
        btnUnlockThis.BackgroundColor3 = Color3.fromRGB(45, 32, 40)
        btnUnlockThis.Text = "🔓 Mở Khóa Loài Này"
        btnUnlockThis.TextColor3 = Color3.fromRGB(255, 200, 200)
        btnUnlockThis.Font = Enum.Font.GothamBold
        btnUnlockThis.TextSize = 12
        btnUnlockThis.Parent = actionRowContainer
        MakeCorner(btnUnlockThis, 8)

        local function SetCurrentFish(name)
            FM.currentSelectedFish = name
            local _, _, counts = ScanAndClassifyInventory()
            local cData = counts[name]

            if cData then
                fishTitleLbl.Text = name
                local category = cData.items[1] and cData.items[1].category or "Cá"
                fishStatusLbl.Text = string.format("[%s] • Có: %d con • 🔒 Đã khóa: %d • 🔓 Đang mở: %d", category, cData.total, cData.locked, cData.unlocked)

                local imgUrl = FetchGameFishImage(name)
                if imgUrl ~= "" then
                    fishImgIcon.Image = imgUrl
                    fishEmojiIcon.Visible = false
                else
                    fishImgIcon.Image = ""
                    fishEmojiIcon.Visible = true
                end

                btnLockThis.Text = string.format("🔒 Khóa Toàn Bộ (%d con)", cData.total)
                btnUnlockThis.Text = string.format("🔓 Mở Khóa Toàn Bộ (%d con)", cData.total)
            else
                fishTitleLbl.Text = "Gõ tên cá vào ô tìm kiếm..."
                fishStatusLbl.Text = "Gợi ý toàn bộ loài cá khớp từ khóa sẽ hiển thị phía trên để bạn bấm chọn."
                fishImgIcon.Image = ""
                fishEmojiIcon.Visible = true
                btnLockThis.Text = "🔒 Khóa Loài Này"
                btnUnlockThis.Text = "🔓 Mở Khóa Loài Này"
            end
        end

        function FM.UpdateSearchedFishUI(targetQuery)
            local _, _, counts = ScanAndClassifyInventory()

            -- Dọn sạch suggestion chips cũ
            for _, ch in ipairs(suggestionScroll:GetChildren()) do
                if ch:IsA("TextButton") then ch:Destroy() end
            end

            local matchedNames = {}
            local query = (targetQuery or ""):lower():gsub("[%s%-_]+", "")

            for name, cData in pairs(counts) do
                local clean = name:lower():gsub("[%s%-_]+", "")
                if query == "" or clean:find(query, 1, true) then
                    table.insert(matchedNames, { name = name, count = cData.total })
                end
            end

            table.sort(matchedNames, function(a, b) return a.count > b.count end)

            -- Tạo chips gợi ý
            for idx, m in ipairs(matchedNames) do
                if idx <= 10 then
                    local chip = Instance.new("TextButton")
                    chip.Name = "Chip_" .. idx
                    chip.Size = UDim2.new(0, 0, 0, 26)
                    chip.AutomaticSize = Enum.AutomaticSize.X
                    chip.BackgroundColor3 = Color3.fromRGB(30, 34, 48)
                    chip.Text = string.format(" %s (%d) ", m.name, m.count)
                    chip.TextColor3 = Color3.fromRGB(220, 225, 240)
                    chip.Font = Enum.Font.GothamMedium
                    chip.TextSize = 11
                    chip.Parent = suggestionScroll
                    MakeCorner(chip, 6)
                    MakeStroke(chip, Color3.fromRGB(55, 60, 75), 1)

                    chip.MouseButton1Click:Connect(function()
                        SetCurrentFish(m.name)
                    end)
                end
            end

            -- Chọn con đầu tiên nếu có
            if #matchedNames > 0 then
                SetCurrentFish(matchedNames[1].name)
            else
                FM.currentSelectedFish = nil
                fishTitleLbl.Text = targetQuery ~= "" and ("Không tìm thấy: " .. targetQuery) or "Gõ tên cá vào ô tìm kiếm..."
                fishStatusLbl.Text = "Không có loài cá nào trong Balo khớp với từ khóa vừa nhập."
                fishImgIcon.Image = ""
                fishEmojiIcon.Visible = true
            end
        end

        searchTextBox:GetPropertyChangedSignal("Text"):Connect(function()
            FM.UpdateSearchedFishUI(searchTextBox.Text)
        end)

        btnLockThis.MouseButton1Click:Connect(function()
            if not FM.currentSelectedFish then
                ShowNotification("Tìm Kiếm", "Vui lòng chọn một loài cá hợp lệ trước!", "WARN", 3)
                return
            end
            local _, _, counts = ScanAndClassifyInventory()
            local cData = counts[FM.currentSelectedFish]
            if cData and #cData.items > 0 then
                ProcessBatchItems(cData.items, true, "Khóa " .. FM.currentSelectedFish)
            end
        end)

        btnUnlockThis.MouseButton1Click:Connect(function()
            if not FM.currentSelectedFish then
                ShowNotification("Tìm Kiếm", "Vui lòng chọn một loài cá hợp lệ trước!", "WARN", 3)
                return
            end
            local _, _, counts = ScanAndClassifyInventory()
            local cData = counts[FM.currentSelectedFish]
            if cData and #cData.items > 0 then
                ProcessBatchItems(cData.items, false, "Mở Khóa " .. FM.currentSelectedFish)
            end
        end)

        -- Hàng bán theo số lượng tùy chọn
        local sellCustomRow = Instance.new("Frame")
        sellCustomRow.Name = "SellCustomRow"
        sellCustomRow.Size = UDim2.new(1, 0, 0, 38)
        sellCustomRow.BackgroundColor3 = Color3.fromRGB(24, 26, 36)
        sellCustomRow.BorderSizePixel = 0
        sellCustomRow.Parent = searchCard
        MakeCorner(sellCustomRow, 8)
        MakeStroke(sellCustomRow, Color3.fromRGB(60, 65, 80), 1)

        local scLabel = Instance.new("TextLabel")
        scLabel.Size = UDim2.new(0.48, 0, 1, 0)
        scLabel.Position = UDim2.new(0, 10, 0, 0)
        scLabel.BackgroundTransparency = 1
        scLabel.Text = "Số lượng muốn bán (0 = bán hết):"
        scLabel.TextColor3 = Color3.fromRGB(200, 205, 220)
        scLabel.Font = Enum.Font.GothamMedium
        scLabel.TextSize = 11
        scLabel.TextXAlignment = Enum.TextXAlignment.Left
        scLabel.Parent = sellCustomRow

        local sellQtyInput = Instance.new("TextBox")
        sellQtyInput.Size = UDim2.new(0.18, 0, 0, 28)
        sellQtyInput.Position = UDim2.new(0.50, 0, 0, 5)
        sellQtyInput.BackgroundColor3 = Color3.fromRGB(15, 17, 24)
        sellQtyInput.TextColor3 = Color3.fromRGB(255, 215, 0)
        sellQtyInput.Font = Enum.Font.GothamBold
        sellQtyInput.TextSize = 13
        sellQtyInput.Text = "0"
        sellQtyInput.ClearTextOnFocus = false
        sellQtyInput.Parent = sellCustomRow
        MakeCorner(sellQtyInput, 6)

        local btnSellSpecific = Instance.new("TextButton")
        btnSellSpecific.Size = UDim2.new(0.28, -6, 0, 28)
        btnSellSpecific.Position = UDim2.new(0.71, 0, 0, 5)
        btnSellSpecific.BackgroundColor3 = Color3.fromRGB(180, 40, 50)
        btnSellSpecific.Text = "💰 Bán Cá Này"
        btnSellSpecific.TextColor3 = Color3.fromRGB(255, 255, 255)
        btnSellSpecific.Font = Enum.Font.GothamBold
        btnSellSpecific.TextSize = 11
        btnSellSpecific.Parent = sellCustomRow
        MakeCorner(btnSellSpecific, 6)

        btnSellSpecific.MouseButton1Click:Connect(function()
            if not FM.currentSelectedFish then
                ShowNotification("Bán Cá", "Vui lòng chọn hoặc tìm một loài cá hợp lệ trước khi bán!", "WARN", 4)
                return
            end
            local qty = tonumber(sellQtyInput.Text) or 0
            SellSpecificFish(FM.currentSelectedFish, qty)
        end)
    end

    ---------------------------------------------------------------------
    -- KHỐI 2: TIẾN ĐỘ CHẾ CẦN CÂU (ROD CRAFTING TRACKER - 4 Ô ẢNH TRỰC QUAN)
    ---------------------------------------------------------------------
    do
        createCategoryHeader(tabFishManager, "🎣 TIẾN ĐỘ CHẾ CẦN CÂU (ROD CRAFTING TRACKER)")

        for _, rDef in ipairs(ROD_RECIPES) do
            local rCard = createCardGroup(tabFishManager)

            local rowHeader = createInfoRow(rCard, rDef.vietName, "Đang quét...")
            local rowProgress = createInfoRow(rCard, "Tiến độ nguyên liệu", "0/4 (0%)")
            local grid = CreateVisualIngredientGrid(rCard, rDef.ingredients)

            createButtonRow(rCard, "Khóa 4 Nguyên Liệu", "Bảo vệ tuyệt đối 4 con cá làm cần " .. rDef.key, "🔒 Khóa 4 NL", function()
                local _, _, counts = ScanAndClassifyInventory()
                local toLock = {}
                for _, ing in ipairs(rDef.ingredients) do
                    local ingLower = ing.name:lower():gsub("[%s%-_]+", "")
                    for name, cData in pairs(counts) do
                        if name:lower():gsub("[%s%-_]+", "") == ingLower then
                            for _, e in ipairs(cData.items) do table.insert(toLock, e) end
                            break
                        end
                    end
                end
                ProcessBatchItems(toLock, true, "Khóa NL " .. rDef.key)
            end)

            createButtonRow(rCard, "Mở Khóa An Toàn", "Mở khóa cá cần nhưng giữ lại cá trùng mồi thần thoại", "🔓 Mở Khóa An Toàn", function()
                local isOwned = CheckRodOwnership(rDef.key)
                if not isOwned then
                    ShowNotification("Cảnh Báo", "Bạn CHƯA sở hữu " .. rDef.key .. "! Cân nhắc trước khi mở khóa!", "WARN", 5)
                end
                local _, _, counts = ScanAndClassifyInventory()
                local toUnlock = {}
                for _, ing in ipairs(rDef.ingredients) do
                    local rawLower = ing.name:lower()
                    if not (Wiki and Wiki.allBaitFishSet and Wiki.allBaitFishSet[rawLower]) then
                        local ingLower = ing.name:lower():gsub("[%s%-_]+", "")
                        for name, cData in pairs(counts) do
                            if name:lower():gsub("[%s%-_]+", "") == ingLower then
                                for _, e in ipairs(cData.items) do table.insert(toUnlock, e) end
                                break
                            end
                        end
                    end
                end
                ProcessBatchItems(toUnlock, false, "Mở Khóa NL " .. rDef.key)
            end)

            table.insert(FM.rodControllers, {
                def = rDef,
                header = rowHeader,
                progress = rowProgress,
                grid = grid
            })
        end

        function FM.RefreshAllRodCards(countsByName)
            for _, ctrl in ipairs(FM.rodControllers) do
                local isOwned, ownedCount = CheckRodOwnership(ctrl.def.key)
                if ctrl.header and ctrl.header.Set then
                    if isOwned then
                        ctrl.header.Set(string.format("✅ ĐÃ CÓ CẦN (Đã sở hữu %d cây)", ownedCount))
                    else
                        ctrl.header.Set("❌ CHƯA CÓ CẦN (Đang thu thập NL)")
                    end
                end

                local completed = 0
                for _, ing in ipairs(ctrl.def.ingredients) do
                    local ingLower = ing.name:lower():gsub("[%s%-_]+", "")
                    for name, cData in pairs(countsByName) do
                        if name:lower():gsub("[%s%-_]+", "") == ingLower and cData.total >= 1 then
                            completed = completed + 1
                            break
                        end
                    end
                end

                if ctrl.progress and ctrl.progress.Set then
                    local pct = math.floor((completed / #ctrl.def.ingredients) * 100)
                    local note = isOwned and " (Đã có trong kho)" or (pct == 100 and " (ĐỦ ĐIỀU KIỆN ĐÚC CẦN! 🏆)" or "")
                    ctrl.progress.Set(string.format("%d/4 nguyên liệu (%d%%)%s", completed, pct, note))
                end

                ctrl.grid.Update(countsByName)
            end
        end
    end

    ---------------------------------------------------------------------
    -- KHỐI 3: NGUYÊN LIỆU CHẾ MỒI THẦN THOẠI (MYTHIC BAIT TRACKER)
    ---------------------------------------------------------------------
    do
        createCategoryHeader(tabFishManager, "🍖 NGUYÊN LIỆU CHẾ MỒI THẦN THOẠI (BAIT CRAFTING)")

        for _, bDef in ipairs(BAIT_RECIPES) do
            local bCard = createCardGroup(tabFishManager)
            local rowHeader = createInfoRow(bCard, bDef.vietName, "Đang quét...")
            local grid = CreateVisualIngredientGrid(bCard, bDef.ingredients)

            createButtonRow(bCard, "Khóa 4 Nguyên Liệu Mồi", "Bảo vệ toàn bộ 4 loài cá làm mồi " .. bDef.key, "🔒 Khóa Cá Mồi", function()
                local _, _, counts = ScanAndClassifyInventory()
                local toLock = {}
                for _, ing in ipairs(bDef.ingredients) do
                    local ingLower = ing.name:lower():gsub("[%s%-_]+", "")
                    for name, cData in pairs(counts) do
                        if name:lower():gsub("[%s%-_]+", "") == ingLower then
                            for _, e in ipairs(cData.items) do table.insert(toLock, e) end
                            break
                        end
                    end
                end
                ProcessBatchItems(toLock, true, "Khóa Mồi " .. bDef.key)
            end)

            createButtonRow(bCard, "Mở Khóa Để Chế Mồi", "Mở khóa 4 loài cá này kèm cờ chống tự khóa", "🔓 Mở Khóa Mồi", function()
                local _, _, counts = ScanAndClassifyInventory()
                local toUnlock = {}
                for _, ing in ipairs(bDef.ingredients) do
                    local ingLower = ing.name:lower():gsub("[%s%-_]+", "")
                    for name, cData in pairs(counts) do
                        if name:lower():gsub("[%s%-_]+", "") == ingLower then
                            for _, e in ipairs(cData.items) do table.insert(toUnlock, e) end
                            break
                        end
                    end
                end
                ProcessBatchItems(toUnlock, false, "Mở Khóa Mồi " .. bDef.key)
            end)

            table.insert(FM.baitControllers, {
                def = bDef,
                header = rowHeader,
                grid = grid
            })
        end

        function FM.RefreshAllBaitCards(countsByName)
            for _, ctrl in ipairs(FM.baitControllers) do
                local minCount = 999999
                for _, ing in ipairs(ctrl.def.ingredients) do
                    local ingLower = ing.name:lower():gsub("[%s%-_]+", "")
                    local cTotal = 0
                    for name, cData in pairs(countsByName) do
                        if name:lower():gsub("[%s%-_]+", "") == ingLower then
                            cTotal = cData.total
                            break
                        end
                    end
                    if cTotal < minCount then minCount = cTotal end
                end
                if minCount == 999999 then minCount = 0 end

                if ctrl.header and ctrl.header.Set then
                    if minCount > 0 then
                        ctrl.header.Set(string.format("Có thể chế tối đa: %d viên mồi ✅", minCount))
                    else
                        ctrl.header.Set("Chưa đủ nguyên liệu để chế viên nào ❌")
                    end
                end

                ctrl.grid.Update(countsByName)
            end
        end
    end

    ---------------------------------------------------------------------
    -- KHỐI 4: THỬ NGHIỆM ĐỘC LẬP TỪNG CON & SPY REMOTE
    ---------------------------------------------------------------------
    do
        createCategoryHeader(tabFishManager, "🎯 THỬ NGHIỆM TỪNG CON & BẮT SPY REMOTE")
        local singleCard = createCardGroup(tabFishManager)

        local TARGET_TEST_FISH = "Crimson Bream Sovereign"
        FM.singleTotalRow = createInfoRow(singleCard, "Cá test: " .. TARGET_TEST_FISH, "0 con")
        FM.spyStatusRow = createInfoRow(singleCard, "Tín hiệu Spy", FM.lastSpyInfo)

        function FM.RefreshSingleTestSection(countsByName)
            local cData = countsByName[TARGET_TEST_FISH] or { total = 0, locked = 0, unlocked = 0 }
            if FM.singleTotalRow and FM.singleTotalRow.Set then
                FM.singleTotalRow.Set(string.format("%d con (🔒 %d khóa, 🔓 %d mở)", cData.total, cData.locked, cData.unlocked))
            end
        end

        createButtonRow(singleCard, "Khóa " .. TARGET_TEST_FISH, "Khóa toàn bộ con " .. TARGET_TEST_FISH .. " đang mở", "🔒 Khóa", function()
            local _, _, counts = ScanAndClassifyInventory()
            local cData = counts[TARGET_TEST_FISH]
            if cData and #cData.items > 0 then
                ProcessBatchItems(cData.items, true, "Khóa " .. TARGET_TEST_FISH)
            else
                ShowNotification("Thử Nghiệm", "Không có con " .. TARGET_TEST_FISH .. " nào trong balo!", "WARN", 3)
            end
        end)

        createButtonRow(singleCard, "Mở Khóa " .. TARGET_TEST_FISH, "Mở khóa toàn bộ con " .. TARGET_TEST_FISH .. " đang khóa", "🔓 Mở Khóa", function()
            local _, _, counts = ScanAndClassifyInventory()
            local cData = counts[TARGET_TEST_FISH]
            if cData and #cData.items > 0 then
                ProcessBatchItems(cData.items, false, "Mở Khóa " .. TARGET_TEST_FISH)
            else
                ShowNotification("Thử Nghiệm", "Không có con " .. TARGET_TEST_FISH .. " nào trong balo!", "WARN", 3)
            end
        end)

        createButtonRow(singleCard, "Bật Spy Bắt Remote", "Bấm nút này rồi mở balo click vào ngôi sao của 1 con cá", "🔍 Bật Spy", function()
            if FM.spyEnabled then
                ShowNotification("SPY REMOTE", "Spy đã được bật! Hãy mở Balo và click vào biểu tượng Ngôi Sao của bất kỳ con cá nào.", "INFO", 5)
                return
            end
            FM.spyEnabled = true

            local favEvent = ReplicatedStorage:FindFirstChild("Events") and ReplicatedStorage.Events:FindFirstChild("FavoriteItem")
            pcall(function()
                if typeof(hookmetamethod) == "function" then
                    local oldNamecall
                    oldNamecall = hookmetamethod(game, "__namecall", function(self, ...)
                        local method = getnamecallmethod()
                        if self == favEvent and (method == "FireServer" or method == "fireServer") then
                            local args = {...}
                            local argTypes = {}
                            local argVals = {}
                            for i, a in ipairs(args) do
                                table.insert(argTypes, typeof(a))
                                table.insert(argVals, tostring(a))
                            end
                            FM.lastSpyInfo = string.format("Args(%d): [%s] -> %s", #args, table.concat(argTypes, ", "), table.concat(argVals, ", "))
                            if FM.spyStatusRow and FM.spyStatusRow.Set then FM.spyStatusRow.Set(FM.lastSpyInfo) end
                            ShowNotification("BẮT ĐƯỢC TÍN HIỆU KHÓA!", FM.lastSpyInfo, "SUCCESS", 8)
                            print("[FAVORITE SPY]", FM.lastSpyInfo)
                        end
                        return oldNamecall(self, ...)
                    end)
                end
            end)

            ShowNotification("SPY ĐÃ KÍCH HOẠT", "Bây giờ hãy mở Balo Game và click vào biểu tượng Ngôi Sao của 1 con cá bất kỳ!", "SUCCESS", 6)
        end)

        createButtonRow(singleCard, "Quét Lại Balo", "Đếm và cập nhật lại toàn bộ các khối dữ liệu", "🔄 Quét Lại", function()
            if UpdateCrimsonBreamUI then UpdateCrimsonBreamUI() end
            ShowNotification("Quét Hoàn Tất", "Đã đồng bộ toàn bộ số liệu Balo vào Tab Quản Lý Cá!", "SUCCESS", 4)
        end)
    end

    -- Hàm cập nhật toàn bộ Tab Quản Lý Cá
    local function RefreshAllSections()
        PreloadFishImages()
        local protList, junkList, counts = ScanAndClassifyInventory()

        if FM.rowBagTotal and FM.rowBagTotal.Set then
            FM.rowBagTotal.Set(string.format("%d con cá trong túi", #protList + #junkList))
        end
        if FM.rowBagProtected and FM.rowBagProtected.Set then
            FM.rowBagProtected.Set(string.format("%d con (Secret Boss, Cần, Mồi, Mutation, Quest, ≥1M KG)", #protList))
        end
        if FM.rowBagJunk and FM.rowBagJunk.Set then
            local unlockedCount = 0
            for _, j in ipairs(junkList) do if not j.isLocked then unlockedCount = unlockedCount + 1 end end
            FM.rowBagJunk.Set(string.format("%d con (Đang mở để bán: %d con)", #junkList, unlockedCount))
        end

        if FM.UpdateSearchedFishUI and FM.searchTextBox then
            FM.UpdateSearchedFishUI(FM.searchTextBox.Text)
        end

        if FM.RefreshAllRodCards then FM.RefreshAllRodCards(counts) end
        if FM.RefreshAllBaitCards then FM.RefreshAllBaitCards(counts) end
        if FM.RefreshSingleTestSection then FM.RefreshSingleTestSection(counts) end
    end

    UpdateCrimsonBreamUI = RefreshAllSections

    -- Lắng nghe thay đổi balo để tự động cập nhật
    task.spawn(function()
        local pData = ReplicatedStorage:WaitForChild("Data", 10)
        local userFolder = pData and pData:WaitForChild(tostring(LocalPlayer.UserId), 10)
        if userFolder then
            local inv = userFolder:WaitForChild("Inventory", 10)
            if inv then
                inv.ChildAdded:Connect(function() task.wait(0.3); RefreshAllSections() end)
                inv.ChildRemoved:Connect(function() task.wait(0.3); RefreshAllSections() end)
            end
        end
    end)

    -- Khởi tạo đếm và nạp ảnh lần đầu
    task.spawn(function()
        task.wait(0.5)
        PreloadFishImages()
        RefreshAllSections()
    end)
end


do
createCategoryHeader(tabGod, "Tương Tác Thần Linh (God Spirit)")
local godCard = createCardGroup(tabGod)

createButtonRow(godCard, "Kiểm Tra Thần Linh", "Kiểm tra bùa chú may mắn hiện tại (Vàng / Xanh)", "Kiểm Tra", function()
    local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
    if pData and pData:FindFirstChild("GodSpirit") then
        local y = pData.GodSpirit:FindFirstChild("Yellow") and pData.GodSpirit.Yellow.Value or false
        local b = pData.GodSpirit:FindFirstChild("Blue") and pData.GodSpirit.Blue.Value or false
        local g = pData.GodSpirit:FindFirstChild("Green") and pData.GodSpirit.Green.Value or false
        local msg = string.format("Yellow: %s | Blue: %s | Green: %s", y and "ACTIVE" or "OFF", b and "ACTIVE" or "OFF", g and "ACTIVE" or "OFF")
        ShowNotification("Trạng Thái Thần Linh", msg, "SUCCESS", 6)
    else
        ShowNotification("Thần Linh", "Không đọc được dữ liệu Thần Linh.", "WARN")
    end
end)

createToggleRow(godCard, "Tự Động Cầu Nguyện", "Tự cầu nguyện nhận bùa khi đứng gần Đền Thần", Config.AutoPrayGodSpirit, function(v) Config.AutoPrayGodSpirit = v end)

createButtonRow(godCard, "Bay Đến Đền Thần Linh", "Dịch chuyển đến Bàn thờ Thần linh (Battlefield Isle)", "Bay Đến", function()
    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if root then
        local spirit = (Workspace:FindFirstChild("NPC") and Workspace.NPC:FindFirstChild("Spirit")) or (Workspace:FindFirstChild("NPC") and Workspace.NPC:FindFirstChild("God"))
        if spirit then
            root.CFrame = spirit:GetPivot() + Vector3.new(0, 3, 5)
            ShowNotification("Thần Linh", "Đã dịch chuyển đến Bàn Thờ Thần Linh!", "SUCCESS")
        else
            root.CFrame = CFrame.new(1245.7, 19.3, -133.4)
            ShowNotification("Thần Linh", "Đã dịch chuyển đến Đền Thờ!", "SUCCESS")
        end
    end
end)

createButtonRow(godCard, "Cầu Nguyện Ngay Lập Tức", "Tương tác với Bàn thờ Thần linh ngay bây giờ", "Cầu Nguyện", function()
    local sp = (Workspace:FindFirstChild("NPC") and Workspace.NPC:FindFirstChild("Spirit")) or (Workspace:FindFirstChild("NPC") and Workspace.NPC:FindFirstChild("God"))
    if sp then
        local found = false
        for _, d in ipairs(sp:GetDescendants()) do
            if d:IsA("ProximityPrompt") then
                TriggerPrompt(d)
                found = true
            end
        end
        if found then
            ShowNotification("Thần Linh", "Đã cầu nguyện Thần Linh thành công!", "SUCCESS")
        else
            ShowNotification("Thần Linh", "Không tìm thấy nút bấm tương tác trên Thần.", "WARN")
        end
    else
        ShowNotification("Thần Linh", "Server này hiện chưa xuất hiện Thần Linh.", "WARN")
    end
end)

createCategoryHeader(tabGod, "Tự Động Đổi Server (Server Hop)")
local hopCard = createCardGroup(tabGod)
createToggleRow(hopCard, "Đổi Server Tìm Thần Linh", "Tự động nhảy server liên tục đến khi gặp God Spirit", Config.AutoServerHopGod, function(v)
    Config.AutoServerHopGod = v
    secretBossState.SaveNPCHopState()
end)
createToggleRow(hopCard, "Đổi Server Tìm Maoshan", "Tự động nhảy server liên tục đến khi gặp Maoshan", Config.AutoServerHopMaoshan, function(v)
    Config.AutoServerHopMaoshan = v
    secretBossState.SaveNPCHopState()
end)
createToggleRow(hopCard, "Đổi Server Tìm Taoist", "Tự động nhảy server liên tục đến khi gặp Đạo sĩ Taoist", Config.AutoServerHopTaoist, function(v)
    Config.AutoServerHopTaoist = v
    secretBossState.SaveNPCHopState()
end)
createButtonRow(hopCard, "Đổi Server Ngay", "Chuyển sang một server ngẫu nhiên khác ngay lập tức", "Đổi Server", function() ServerHop() end)
end

do
createCategoryHeader(tabQuests, "Trạng Thái & Bật/Tắt Nhiệm Vụ Vé (Ticket Quests)")
local questCard = createCardGroup(tabQuests)

createToggleRow(questCard, "Tự Động Làm Vé Nhiệm Vụ", "Tự động nhận, thực hiện và trả vé nhiệm vụ theo chu kỳ 20p", Config.AutoTicketQuest, function(v)
    Config.AutoTicketQuest = v
    if v then
        ticketQuestState.active = true
        ShowNotification("Nhiệm Vụ Vé", "Đã bật tự động làm vé nhiệm vụ! Script sẽ quét và thực hiện quest.", "SUCCESS", 6)
    else
        if ticketQuestState then
            ticketQuestState.active = false
            ticketQuestState.isBusyRoutine = false
            ticketQuestState.currentQuestType = "none"
            ticketQuestState.isCooldown = false
            ticketQuestState.isAtHomeSpot = false
        end
        ShowNotification("Nhiệm Vụ Vé", "Đã tắt tự động làm vé nhiệm vụ.", "INFO", 4)
    end
    ticketQuestState.ScanAndUpdateStatus()
end)

createDropdownRow(questCard, "Độ Khó Nhiệm Vụ", "Chọn độ khó vé nhiệm vụ nhận từ NPC (Mặc định: Hard)", {"Hard", "Easy"}, Config.TicketDifficulty, function(v)
    Config.TicketDifficulty = v
end)

local questModes = {
    "Tự Động (Auto Detect)",
    "Câu 10 Con Cá 1.5M+ (Map 9)",
    "Tiêu Thụ 100 Mồi (Map 1)",
    "Dùng Kỹ Năng 100 Lần",
    "Câu Nhanh 100 Con Cá (Map 1)"
}
createDropdownRow(questCard, "Chế Độ Nhiệm Vụ", "Tự động nhận diện từ game hoặc ép kiểu nhiệm vụ bạn muốn bot làm", questModes, Config.TicketQuestMode, function(v)
    Config.TicketQuestMode = v
    ticketQuestState.currentQuestType = "none"
    ticketQuestState.currentProgress = 0
    ticketQuestState.activeQuestDetected = false
    ticketQuestState.UpdateUI()
end)

ticketQuestState.uiStatus = createInfoRow(questCard, "Nhiệm Vụ Hiện Tại", ticketQuestState.statusText)
ticketQuestState.uiProgress = createInfoRow(questCard, "Tiến Độ Nhiệm Vụ", "0 / 100 (0%)")
ticketQuestState.uiCooldown = createInfoRow(questCard, "Hồi Chiêu 20 Phút", "Sẵn sàng nhận vé!")

createCategoryHeader(tabQuests, "⚡ Nhiệm Vụ Kỹ Năng Zeng Tianguo & Chế Độ Song Song")
local zengCard = createCardGroup(tabQuests)

zengTianguoQuestState.uiParallelBadge = createInfoRow(zengCard, "Chế Độ Vận Hành", "Đang Tắt")

createToggleRow(zengCard, "Tự Động Nhiệm Vụ Zeng Tianguo", "Tự động nhận, thực hiện và trả nhiệm vụ nâng cấp kỹ năng của Zeng Tianguo", Config.AutoZengTianguoQuest, function(v)
    Config.AutoZengTianguoQuest = v
    if v then
        zengTianguoQuestState.active = true
        ShowNotification("Zeng Tianguo", "Đã bật tự động làm nhiệm vụ Zeng Tianguo (Skill Upgrade)!", "SUCCESS", 5)
    else
        zengTianguoQuestState.active = false
        ShowNotification("Zeng Tianguo", "Đã tắt tự động làm nhiệm vụ Zeng Tianguo.", "INFO", 4)
    end
    zengTianguoQuestState.DetectActiveQuest()
    zengTianguoQuestState.UpdateUI()
end)

createToggleRow(zengCard, "Ưu Tiên Ghép Bãi Song Song", "Khi bật cả 2, tự động câu ở đảo của Zeng Tianguo để hoàn thành cả 2 cùng lúc (Ưu tiên vé)", Config.ParallelQuestMode, function(v)
    Config.ParallelQuestMode = v
    zengTianguoQuestState.UpdateUI()
end)

createToggleRow(zengCard, "Tự Động Trả Quest Zeng Tianguo", "Tự động bay về NPC trả quest khi hoàn thành chuỗi mục tiêu", Config.ZengTianguoAutoClaim, function(v)
    Config.ZengTianguoAutoClaim = v
end)

zengTianguoQuestState.uiStatus = createInfoRow(zengCard, "Nhiệm Vụ Kỹ Năng", zengTianguoQuestState.statusText)
zengTianguoQuestState.uiProgress = createInfoRow(zengCard, "Tiến Độ Kỹ Năng", "0 / 100 (0%)")

createButtonRow(zengCard, "Tìm & Bay Đến NPC Zeng Tianguo", "Tự động tìm kiếm vị trí NPC Zeng Tianguo (Skill Upgrade) và bay tới đối diện", "Bay Đến NPC", function()
    local ok = zengTianguoQuestState.TeleportToNPC()
    if ok then
        ShowNotification("Dịch Chuyển", "Đã bay đến NPC Zeng Tianguo (Skill Upgrade)!", "SUCCESS")
    else
        ShowNotification("Dịch Chuyển", "Không tìm thấy model NPC Zeng Tianguo trong game!", "WARNING")
    end
end)

createButtonRow(zengCard, "Nhận / Nộp Quest Zeng Tianguo", "Tương tác nhanh với NPC Zeng Tianguo để nhận hoặc nộp nhiệm vụ hoàn thành", "Tương Tác", function()
    task.spawn(function()
        ShowNotification("Zeng Tianguo", "Đang tương tác với NPC Zeng Tianguo...", "INFO", 3)
        local isDone = zengTianguoQuestState.isCompleted
        zengTianguoQuestState.InteractNPC(isDone)
        task.wait(1.0)
        zengTianguoQuestState.DetectActiveQuest()
        zengTianguoQuestState.UpdateUI()
    end)
end)

local spotCard = createCollapsibleCardGroup(tabQuests, "📍 Cài Đặt Vị Trí Câu & NPC Ticket Quest", false)

ticketQuestState.ui100Spot = createInfoRow(spotCard, "Điểm Câu 100 Con (Map 1)", string.format("(%.0f, %.0f, %.0f)", ticketQuestState.spot100Fish.X, ticketQuestState.spot100Fish.Y, ticketQuestState.spot100Fish.Z))
createButtonRow(spotCard, "Lấy Tọa Độ Hiện Tại Làm Điểm 100 Con", "Gán vị trí & hướng nhìn bạn đang đứng làm nơi câu 100 con cá nhẹ", "Lấy Vị Trí", function()
    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if root then
        ticketQuestState.spot100Fish = root.CFrame
        ticketQuestState.SaveSpots()
        ticketQuestState.UpdateUI()
        ShowNotification("Vị Trí Nhiệm Vụ", string.format("Đã lưu vị trí & hướng nhìn câu 100 con: (%.0f, %.0f, %.0f)!", root.Position.X, root.Position.Y, root.Position.Z), "SUCCESS")
    end
end)
createButtonRow(spotCard, "Bay Đến Điểm Câu 100 Con", "Dịch chuyển tức thì đến điểm câu 100 con đã cài", "Bay Đến", function()
    ticketQuestState.TeleportTo(ticketQuestState.spot100Fish)
    ShowNotification("Dịch Chuyển", "Đã bay đến điểm câu 100 con!", "SUCCESS")
end)

ticketQuestState.ui100BaitSpot = createInfoRow(spotCard, "Điểm Tiêu Thụ 100 Mồi (Map 1)", string.format("(%.0f, %.0f, %.0f)", ticketQuestState.spot100Bait.X, ticketQuestState.spot100Bait.Y, ticketQuestState.spot100Bait.Z))
createButtonRow(spotCard, "Lấy Tọa Độ Hiện Tại Làm Điểm 100 Mồi", "Gán vị trí & hướng nhìn bạn đang đứng làm nơi câu tiêu thụ 100 mồi", "Lấy Vị Trí", function()
    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if root then
        ticketQuestState.spot100Bait = root.CFrame
        ticketQuestState.SaveSpots()
        ticketQuestState.UpdateUI()
        ShowNotification("Vị Trí Nhiệm Vụ", string.format("Đã lưu vị trí & hướng nhìn 100 mồi: (%.0f, %.0f, %.0f)!", root.Position.X, root.Position.Y, root.Position.Z), "SUCCESS")
    end
end)
createButtonRow(spotCard, "Bay Đến Điểm 100 Mồi", "Dịch chuyển tức thì đến điểm câu 100 mồi đã cài", "Bay Đến", function()
    ticketQuestState.TeleportTo(ticketQuestState.spot100Bait)
    ShowNotification("Dịch Chuyển", "Đã bay đến điểm 100 mồi!", "SUCCESS")
end)

ticketQuestState.ui100SkillSpot = createInfoRow(spotCard, "Điểm Câu 100 Skill (Map 1)", string.format("(%.0f, %.0f, %.0f)", ticketQuestState.spot100Skill.X, ticketQuestState.spot100Skill.Y, ticketQuestState.spot100Skill.Z))
createButtonRow(spotCard, "Lấy Tọa Độ Hiện Tại Làm Điểm 100 Skill", "Gán vị trí & hướng nhìn bạn đang đứng làm nơi câu 100 skill", "Lấy Vị Trí", function()
    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if root then
        ticketQuestState.spot100Skill = root.CFrame
        ticketQuestState.SaveSpots()
        ticketQuestState.UpdateUI()
        ShowNotification("Vị Trí Nhiệm Vụ", string.format("Đã lưu vị trí & hướng nhìn 100 skill: (%.0f, %.0f, %.0f)!", root.Position.X, root.Position.Y, root.Position.Z), "SUCCESS")
    end
end)
createButtonRow(spotCard, "Bay Đến Điểm 100 Skill", "Dịch chuyển tức thì đến điểm câu 100 skill đã cài", "Bay Đến", function()
    ticketQuestState.TeleportTo(ticketQuestState.spot100Skill)
    ShowNotification("Dịch Chuyển", "Đã bay đến điểm 100 skill!", "SUCCESS")
end)

ticketQuestState.ui15MSpot = createInfoRow(spotCard, "Điểm Câu 1.5M (Map 9)", string.format("(%.0f, %.0f, %.0f)", ticketQuestState.spot15MFish.X, ticketQuestState.spot15MFish.Y, ticketQuestState.spot15MFish.Z))
createButtonRow(spotCard, "Lấy Tọa Độ Hiện Tại Làm Điểm 1.5M", "Gán vị trí & hướng nhìn bạn đang đứng làm nơi câu cá 1.5M+", "Lấy Vị Trí", function()
    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if root then
        ticketQuestState.spot15MFish = root.CFrame
        ticketQuestState.SaveSpots()
        ticketQuestState.UpdateUI()
        ShowNotification("Vị Trí Nhiệm Vụ", string.format("Đã lưu vị trí & hướng nhìn câu 1.5M: (%.0f, %.0f, %.0f)!", root.Position.X, root.Position.Y, root.Position.Z), "SUCCESS")
    end
end)
createButtonRow(spotCard, "Bay Đến Điểm Câu 1.5M", "Dịch chuyển tức thì đến điểm câu cá 1.5M+ đã cài", "Bay Đến", function()
    ticketQuestState.TeleportTo(ticketQuestState.spot15MFish)
    ShowNotification("Dịch Chuyển", "Đã bay đến điểm câu 1.5M!", "SUCCESS")
end)

ticketQuestState.uiNPCSpot = createInfoRow(spotCard, "Vị Trí NPC Ticket Quest (Map 1)", string.format("(%.0f, %.0f, %.0f)", ticketQuestState.spotNPC.X, ticketQuestState.spotNPC.Y, ticketQuestState.spotNPC.Z))
createButtonRow(spotCard, "Lấy Tọa Độ Hiện Tại Làm Vị Trí NPC", "Gán vị trí & hướng nhìn bạn đang đứng cạnh NPC Ticket Quest", "Lấy Vị Trí", function()
    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if root then
        ticketQuestState.spotNPC = root.CFrame
        ticketQuestState.SaveSpots()
        ticketQuestState.UpdateUI()
        ShowNotification("Vị Trí NPC", string.format("Đã lưu vị trí & hướng nhìn NPC Ticket Quest: (%.0f, %.0f, %.0f)!", root.Position.X, root.Position.Y, root.Position.Z), "SUCCESS")
    end
end)
createButtonRow(spotCard, "Tìm & Bay Đến NPC Ticket Quest", "Tự động quét, xoay góc nhìn và bay thẳng đến NPC Ticket Quest", "Bay Đến NPC", function()
    local npcModel, focusPos, prompt = ticketQuestState.TeleportToNPC()
    if focusPos then
        ticketQuestState.spotNPC = focusPos
        ticketQuestState.SaveSpots()
        ticketQuestState.UpdateUI()
        ShowNotification("Dịch Chuyển", "Đã tìm thấy, căn góc nhìn chuẩn và bay đến NPC Ticket Quest!", "SUCCESS")
    else
        ticketQuestState.TeleportTo(ticketQuestState.spotNPC)
        ShowNotification("Dịch Chuyển", "Đã bay đến tọa độ lưu của NPC Ticket Quest!", "SUCCESS")
    end
end)

createCategoryHeader(tabQuests, "⚙️ Tùy Chỉnh Mồi & Kỹ Năng Cho Nhiệm Vụ")
local optionCard = createCardGroup(tabQuests)

local ticketBaits = {"Basic Bait", "Crude Mash Bait", "Corrupted Essence Bait", "Elite Bait", "Ancestral Bait"}
createDropdownRow(optionCard, "Mồi Cho Nhiệm Vụ 100 Mồi", "Loại mồi bot sẽ mua và dùng khi nhận nv 100 mồi", ticketBaits, Config.TicketBaitChoice, function(v)
    Config.TicketBaitChoice = v
end)

local skillList = {"Chiêu Z", "Chiêu X", "Chiêu C", "Chiêu V"}
createDropdownRow(optionCard, "Chiêu Dùng Cho Nhiệm Vụ 100 Skill", "Kỹ năng bot dùng sau 3s khóa chiêu rồi cất cần lặp lại", skillList, Config.TicketSkillKey, function(v)
    Config.TicketSkillKey = v
end)

createDropdownRow(optionCard, "Chiêu Giật Nhanh Cho 100 Con Cá", "Chiêu mạnh nhất dùng để kết liễu cá Map 1 trong 1 hit", skillList, Config.TicketQuickSkill, function(v)
    Config.TicketQuickSkill = v
end)

createToggleRow(optionCard, "Tự Bán Cá Khi Đầy Balo (Vé NV)", "Tự động bán sạch cá khi balo đạt giới hạn để câu tiếp", Config.TicketAutoSellFull, function(v)
    Config.TicketAutoSellFull = v
end)

createToggleRow(optionCard, "Tự Về Home Spot Câu Farm (Chờ 20p)", "Khi trả xong vé và chờ hồi 20p, tự bay về Home Spot và tự động câu cá/combo", Config.TicketReturnHomeWhenDone, function(v)
    Config.TicketReturnHomeWhenDone = v
    Config.TicketAutoCastAtHome = v
end)

createToggleRow(optionCard, "Nhận & Nộp Vé Từ Xa (Remote)", "Đứng yên tại chỗ câu để nhận và nộp vé Hard từ xa (không cần bay về NPC)", Config.TicketRemoteClaim, function(v)
    Config.TicketRemoteClaim = v
end)

createCategoryHeader(tabQuests, "⚡ Thao Tác Nhanh Bằng Tay")
local manualCard = createCardGroup(tabQuests)

createButtonRow(manualCard, "Nhận Vé Hard Ngay", "Tương tác NPC, mở hội thoại và bấm nút Quest để nhận vé mới", "Nhận Hard", function()
    task.spawn(function()
        ShowNotification("Nhiệm Vụ Vé", "Đang tương tác NPC nhận vé Hard...", "INFO", 3)
        ticketQuestState.InteractNPC(false)
    end)
end)

createButtonRow(manualCard, "Nộp / Trả Vé Hard Ngay", "Tương tác NPC, mở hội thoại và bấm nút *Leave* để trả vé & nhận quà", "Nộp Hard", function()
    task.spawn(function()
        ShowNotification("Nhiệm Vụ Vé", "Đang tương tác NPC nộp vé Hard...", "INFO", 3)
        ticketQuestState.InteractNPC(true)
    end)
end)

createButtonRow(manualCard, "Quét Lại Tiến Độ Nhiệm Vụ", "Quét ngay lập tức PlayerGui để kiểm tra nhiệm vụ và tiến độ hiện tại", "Quét Ngay", function()
    ticketQuestState.ScanAndUpdateStatus()
    ShowNotification("Nhiệm Vụ Vé", tostring(ticketQuestState.statusText), "INFO", 5)
end)

createButtonRow(manualCard, "Đặt Lại / Bỏ Chặn Hết Vé Hôm Nay", "Xóa cờ đánh dấu hết vé hôm nay để bot thử tương tác nhận vé lại", "🔄 Đặt Lại", function()
    ticketQuestState.allQuestsDoneForToday = false
    ticketQuestState.allQuestsDoneDate = nil
    ticketQuestState.allQuestsDoneUtcDate = nil
    ticketQuestState.savedDailyCount = nil
    ticketQuestState.readyForNewQuest = true
    ticketQuestState.isCooldown = false
    ticketQuestState.isAtHomeSpot = false
    ticketQuestState.cooldownEnd = 0
    ticketQuestState.statusText = "Đã đặt lại! Sẵn sàng thử nhận vé mới."
    ticketQuestState.UpdateUI()
    ShowNotification("Nhiệm Vụ Vé", "Đã xóa cờ hết vé hôm nay! Bot sẽ thử nhận vé lại.", "SUCCESS", 5)
end)

createCategoryHeader(tabQuests, "Điểm Danh & Nhiệm Vụ Hàng Ngày")
local dailyCard = createCardGroup(tabQuests)
createButtonRow(dailyCard, "Nhận Thưởng Nhiệm Vụ Ngày", "Tự kiểm tra và nhận thưởng các quest đã xong", "Nhận Thưởng", function()
    if Events and Events:FindFirstChild("ClaimQuest") then
        for i = 1, 4 do Events.ClaimQuest:FireServer("Daily", i) end
        ShowNotification("Nhiệm Vụ Ngày", "Đã nhận thưởng tất cả nhiệm vụ ngày hoàn thành!", "SUCCESS")
    end
end)

createToggleRow(dailyCard, "Tự Điểm Danh 7 Ngày", "Tự động nhận quà điểm danh hàng ngày từ ngày 1 - 7", Config.AutoClaimDaily, function(v) Config.AutoClaimDaily = v end)
createSliderRow(dailyCard, "Độ Trễ Nhận Quà", "Thời gian giãn cách giữa các ngày", 0.2, 2.0, Config.DailyClaimDelay, true, "s", function(v) Config.DailyClaimDelay = v end)

createButtonRow(dailyCard, "Nhận Hết Quà 7 Ngày", "Nhận nhanh toàn bộ quà điểm danh 7 ngày cùng lúc", "Nhận Hết", function()
    task.spawn(function()
        if Events and Events:FindFirstChild("DailyReward") then
            for day = 1, 7 do
                Events.DailyReward:FireServer(day)
                task.wait(Config.DailyClaimDelay)
            end
            ShowNotification("Điểm Danh", "Đã nhận trọn bộ quà điểm danh từ ngày 1 - 7!", "SUCCESS")
        end
    end)
end)

createButtonRow(dailyCard, "Nhập Toàn Bộ Mã Code", "Tự động nhập toàn bộ hơn 100 mã giftcode (65KLikes, 41MVisits, PVP, SoTamOrb...)", "Nhập Code", function()
    local codes = {
        -- Mã mới nhất từ thông báo game (Update mới nhất)
        "65KLikes",
        "41MVisits",
        "40MVisits",
        "39MVisits",
        "38MVisits",
        "37MVisits",
        "9KActive",
        "9KActives",
        "SorryForShutdown",
        -- Mã đang hoạt động & hot
        "PVP",
        "19KActives",
        "WaitForPeak",
        "SoTamOrb",
        "36MVisits",
        "35MVisits",
        "34MVisits",
        "60KLikes",
        "55KLikes",
        "50KLikes",
        "AXO",
        "TaijiEvo",
        "NewSeason",
        "33MVisits",
        "32MVisits",
        "31MVisits",
        "30MVisits",
        "Taiji",
        "Balanced",
        "49KLikes",
        "48KLikes",
        "47KLikes",
        "46KLikes",
        "13KActives",
        "17KActives",
        "9KActives",
        "HWF",
        "RELEASE",
        -- Toàn bộ mốc Likes lịch sử
        "45KLikes",
        "44KLikes",
        "43KLikes",
        "42KLikes",
        "41KLikes",
        "40KLikes",
        "39KLikes",
        "38KLikes",
        "37KLikes",
        "36KLikes",
        "35KLikes",
        "34KLikes",
        "31KLikes",
        "30KLikes",
        "29KLikes",
        "28KLikes",
        "26KLikes",
        "25KLikes",
        "22KLikes",
        "21KLikes",
        -- Toàn bộ mốc Visits lịch sử
        "29MVisits",
        "28MVisits",
        "27MVisits",
        "26MVisits",
        "25MVisits",
        "24MVisits",
        "23MVisits",
        "22MVisits",
        "21MVisits",
        "20M5Visits",
        "20MVisits",
        "19M5Visits",
        "19MVisits",
        "18M5Visits",
        "18MVisits",
        "17M5Visits",
        "17MVisits",
        "16M5Visits",
        "16MVisits",
        "15M5Visits",
        "15MVisits",
        "14M5Visits",
        "14MVisits",
        "11MVisits",
        "10M5Visits",
        "10MVisits",
        "9MVisits",
        "8M5Visits",
        "7MVisits",
        "6M5Visits",
        -- Các mã sự kiện, sửa lỗi & đền bù
        "BigUPD",
        "SorryForShutdown",
        "InfnanLOL",
        "367BUG",
        "UIBUG",
        "ThanksForNitroBoost",
        "Enzo",
        "Enzo2",
        "CodeBug",
        "PEAK",
        "FreeReroll",
        "FreeReroll2",
        "BuyAgain",
        "BugAgain",
        "BUGBUGBUG",
        "Golden",
        "Hit5KActives",
        "Hit4KActives",
        "Hit3KActives",
        "Hit2KActives",
        "ORB",
        "FreeTicket",
        "CrystalBugs",
        "April Fools",
        "LunarNewYear"
    }
    if Events and Events:FindFirstChild("RedeemCode") then
        task.spawn(function()
            for _, c in ipairs(codes) do
                Events.RedeemCode:FireServer(c)
                task.wait(0.25)
            end
            ShowNotification("Giftcode", string.format("Đã nạp toàn bộ %d mã code vào game!", #codes), "SUCCESS")
        end)
    end
end)

createInputRow(dailyCard, "Nhập Mã Code Thủ Công", "Gõ mã giftcode riêng hoặc mã mới ra để nạp ngay", "", function(codeTxt)
    if codeTxt and codeTxt:gsub("%s+", "") ~= "" then
        local cleanCode = codeTxt:gsub("%s+", "")
        if Events and Events:FindFirstChild("RedeemCode") then
            Events.RedeemCode:FireServer(cleanCode)
            ShowNotification("Giftcode", "Đã gửi mã: " .. cleanCode, "SUCCESS")
        end
    end
end)
end

do
createCategoryHeader(tabShop, "Chế Tạo & Mua Mồi Câu")
local baitCard = createCardGroup(tabShop)

local craftBaits = {"Nameless Bait", "Frost Bait", "Rainbow Bait"}
createDropdownRow(baitCard, "Chọn Mồi Cần Chế", "Loại mồi thần thoại muốn chế tạo", craftBaits, Config.CraftBaitName, function(v) Config.CraftBaitName = v end)
createSliderRow(baitCard, "Số Lượng Chế Mỗi Lần", "Số lượng mồi chế trong 1 lượt", 1, 10, Config.CraftAmount, false, "", function(v) Config.CraftAmount = v end)
createToggleRow(baitCard, "Tự Động Chế Mồi", "Liên tục chế mồi khi trong kho đủ nguyên liệu (tự mở khóa NL)", Config.AutoCraftBait, function(v)
    Config.AutoCraftBait = v
    if not v then
        Wiki.LockBaitFish(Config.CraftBaitName, true)
    end
end)

local btnUnlockCurBait = createButtonRow(baitCard, "Mở Khóa Cá Chế Mồi Đang Chọn", "Chỉ mở khóa các con cá làm nguyên liệu cho mồi đang chọn ở trên để chế tạo", "🔓 Mở Khóa NL", function()
    Wiki.UnlockBaitFish(Config.CraftBaitName)
    if Wiki.RefreshBagUI then Wiki.RefreshBagUI() end
end)
btnUnlockCurBait.Size = UDim2.new(0, 115, 0, 24)
btnUnlockCurBait.Position = UDim2.new(1, -115, 0.5, -12)
btnUnlockCurBait.BackgroundColor3 = Color3.fromRGB(37, 99, 235)
btnUnlockCurBait.TextColor3 = Colors.TextWhite

local btnLockCurBait = createButtonRow(baitCard, "Khóa Lại Cá Chế Mồi Đang Chọn", "Khóa bảo vệ lại cá làm mồi sau khi chế xong để tránh AutoSell bán mất", "🔒 Khóa Lại NL", function()
    Wiki.LockBaitFish(Config.CraftBaitName)
    if Wiki.RefreshBagUI then Wiki.RefreshBagUI() end
end)
btnLockCurBait.Size = UDim2.new(0, 115, 0, 24)
btnLockCurBait.Position = UDim2.new(1, -115, 0.5, -12)
btnLockCurBait.BackgroundColor3 = Color3.fromRGB(16, 185, 129)
btnLockCurBait.TextColor3 = Colors.TextWhite

local btnUnlockAllBait = createButtonRow(baitCard, "Mở Khóa NL Cả 3 Loại Mồi Thần Thoại", "Mở khóa toàn bộ cá làm mồi Nameless, Frost, Rainbow Bait để chế tạo", "🔓 Mở Cả 3 Mồi", function()
    Wiki.UnlockBaitFish("All")
    if Wiki.RefreshBagUI then Wiki.RefreshBagUI() end
end)
btnUnlockAllBait.Size = UDim2.new(0, 115, 0, 24)
btnUnlockAllBait.Position = UDim2.new(1, -115, 0.5, -12)
btnUnlockAllBait.BackgroundColor3 = Color3.fromRGB(180, 83, 9)
btnUnlockAllBait.TextColor3 = Colors.TextWhite

local btnLockAllBait = createButtonRow(baitCard, "Khóa Lại NL Cả 3 Loại Mồi Thần Thoại", "Khóa bảo vệ lại toàn bộ cá làm mồi của cả 3 loại mồi sau khi chế xong", "🔒 Khóa Cả 3 Mồi", function()
    Wiki.LockBaitFish("All")
    if Wiki.RefreshBagUI then Wiki.RefreshBagUI() end
end)
btnLockAllBait.Size = UDim2.new(0, 115, 0, 24)
btnLockAllBait.Position = UDim2.new(1, -115, 0.5, -12)
btnLockAllBait.BackgroundColor3 = Colors.PurpleDark
btnLockAllBait.TextColor3 = Colors.TextWhite

local buyBaits = {"Ancestral Bait", "Elite Bait", "Corrupted Essence Bait", "Crude Mash Bait", "Basic Bait"}
createDropdownRow(baitCard, "Chọn Mồi Cần Mua", "Loại mồi muốn mua từ NPC Ba Chang", buyBaits, Config.BuyBaitName, function(v) Config.BuyBaitName = v end)
createSliderRow(baitCard, "Số Lượng Mua Mỗi Lần", "Số lượng mồi mua mỗi lần giao dịch", 1, 50, Config.BuyBaitAmount, false, "", function(v) Config.BuyBaitAmount = v end)
createSliderRow(baitCard, "Ngưỡng Mua Tự Động", "Tự mua khi số mồi trong kho ít hơn mức này", 5, 50, Config.BuyBaitThreshold, false, "", function(v) Config.BuyBaitThreshold = v end)
createToggleRow(baitCard, "Tự Động Mua Mồi", "Tự động mua thêm mồi khi sắp hết", Config.AutoBuyBait, function(v) Config.AutoBuyBait = v end)

createCategoryHeader(tabShop, "Thương Nhân Kỹ Năng (Sage Yijiu)")
local sageCard = createCardGroup(tabShop)
local sageSkills = {"One-Strike Heaven Gate", "Taijiquan Technique", "Infinite Sky Ascension", "Rolling Chaos", "Sever the Gate", "Phoenix Strike Art", "Skyfall Stomp", "Beastbreaker Cleave", "Demonfall Technique", "Dragon Strike"}
local chosenSageSkill = sageSkills[1]
createDropdownRow(sageCard, "Chọn Kỹ Năng", "Kỹ năng muốn học từ NPC Sage Yijiu", sageSkills, chosenSageSkill, function(v) chosenSageSkill = v end)

createButtonRow(sageCard, "Bay Đến & Mua Kỹ Năng", "Dịch chuyển đến Sage Yijiu và mua chiêu thức", "Mua Chiêu", function()
    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if root then
        root.CFrame = CFrame.new(-117.5, 6.8, 41.2)
        task.wait(0.3)
        if Events and Events:FindFirstChild("BuySkill") then
            Events.BuySkill:FireServer(chosenSageSkill)
            ShowNotification("Sage Yijiu", "Đã mua thành công kỹ năng: " .. chosenSageSkill, "SUCCESS")
        end
    end
end)

createCategoryHeader(tabShop, "Vòng Quay May Mắn (Auto Gacha)")
local gachaCard = createCardGroup(tabShop)
createDropdownRow(gachaCard, "Chọn Vòng Quay Gacha", "Vòng quay muốn rút thưởng", {"Taiji Banner", "Egoless Banner"}, Config.GachaBanner, function(v) Config.GachaBanner = v end)
createSliderRow(gachaCard, "Số Vé Mỗi Lần Quay", "Số lượng vé dùng cho mỗi lượt rút thưởng", 1, 10, Config.GachaPullsPerAction, false, "", function(v) Config.GachaPullsPerAction = v end)
createToggleRow(gachaCard, "Vòng Quay May Mắn (Auto Gacha)", "Tự động rút thưởng liên tục từ banner đã chọn", Config.AutoGacha, function(v) Config.AutoGacha = v end)
end

createCategoryHeader(tabShop, "Danh Sách Cần Câu (Xếp Theo Giá)")
local rodShopCard = createCardGroup(tabShop)

local function formatNumber(n)
    if n == 0 then return "FREE" end
    return FormatWithSpaces(n) .. " Cash"
end

local rodShopUpdaters = {}
local function UpdateAllRodShopUI()
    for _, fn in ipairs(rodShopUpdaters) do
        pcall(fn)
    end
end

-- Nút Làm Mới Trạng Thái Cần Câu
createButtonRow(rodShopCard, "Làm Mới Trạng Thái Cần", "Quét lại túi đồ để cập nhật danh sách cần đã có / chưa có", "Làm Mới", function()
    UpdateAllRodShopUI()
    ShowNotification("Shop Cần", "Đã cập nhật trạng thái sở hữu cần câu!", "INFO")
end)

for _, rod in ipairs(allRods) do
    local pStr = formatNumber(rod.price)
    local baseDesc = string.format("Lực: %d | May mắn: %d%% | Giá: %s", rod.power, rod.luck, pStr)

    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, 0, 0, 44)
    row.BackgroundColor3 = Colors.RowNormal
    row.BorderSizePixel = 0
    row.Parent = rodShopCard

    local pad = Instance.new("UIPadding")
    pad.PaddingLeft = UDim.new(0, 10)
    pad.PaddingRight = UDim.new(0, 10)
    pad.Parent = row

    local tf = Instance.new("Frame")
    tf.Size = UDim2.new(1, -125, 1, 0)
    tf.BackgroundTransparency = 1
    tf.Parent = row

    local tl = Instance.new("TextLabel")
    tl.Size = UDim2.new(1, 0, 0, 20)
    tl.Position = UDim2.new(0, 0, 0, 3)
    tl.BackgroundTransparency = 1
    tl.Font = Enum.Font.GothamBold
    tl.Text = rod.name
    tl.TextColor3 = Colors.TextWhite
    tl.TextSize = 12
    tl.TextXAlignment = Enum.TextXAlignment.Left
    tl.Parent = tf

    local dl = Instance.new("TextLabel")
    dl.Size = UDim2.new(1, 0, 0, 16)
    dl.Position = UDim2.new(0, 0, 0, 23)
    dl.BackgroundTransparency = 1
    dl.Font = Enum.Font.Gotham
    dl.Text = baseDesc
    dl.TextColor3 = Colors.TextMuted
    dl.TextSize = 10
    dl.TextXAlignment = Enum.TextXAlignment.Left
    dl.Parent = tf

    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, 105, 0, 26)
    btn.Position = UDim2.new(1, -105, 0.5, -13)
    btn.BackgroundColor3 = Colors.ControlBg
    btn.Font = Enum.Font.GothamBold
    btn.Text = "Mua Cần"
    btn.TextColor3 = Colors.PurplePrimary
    btn.TextSize = 11
    btn.BorderSizePixel = 0
    btn.Parent = row
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 4)

    row.MouseEnter:Connect(function() TweenService:Create(row, TweenInfo.new(0.15), {BackgroundColor3 = Colors.RowHover}):Play() end)
    row.MouseLeave:Connect(function() TweenService:Create(row, TweenInfo.new(0.15), {BackgroundColor3 = Colors.RowNormal}):Play() end)

    table.insert(rowSearchIndex, {frame = row, query = (rod.name .. " " .. baseDesc):lower()})

    local function updateRowVisuals()
        local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
        local curEq = pData and pData:FindFirstChild("FishingRod") and pData.FishingRod.Value
        local isEquipped = (curEq == rod.name)
        local isOwned = isEquipped or IsRodOwned(rod.name)

        if isEquipped then
            tl.Text = string.format("%s  [ĐANG DÙNG]", rod.name)
            tl.TextColor3 = Color3.fromRGB(120, 255, 170)
            dl.Text = string.format("%s • [Trạng thái: Đang Cầm]", baseDesc)
            dl.TextColor3 = Color3.fromRGB(160, 255, 190)

            btn.Text = "Đang Dùng"
            btn.BackgroundColor3 = Color3.fromRGB(30, 65, 45)
            btn.TextColor3 = Color3.fromRGB(120, 255, 170)
        elseif isOwned then
            tl.Text = string.format("%s  [ĐÃ CÓ]", rod.name)
            tl.TextColor3 = Color3.fromRGB(230, 240, 255)
            dl.Text = string.format("%s • [Trạng thái: ĐÃ CÓ - SẴN SÀNG]", baseDesc)
            dl.TextColor3 = Color3.fromRGB(100, 220, 255)

            btn.Text = "Trang Bị"
            btn.BackgroundColor3 = Color3.fromRGB(28, 50, 75)
            btn.TextColor3 = Color3.fromRGB(100, 220, 255)
        else
            tl.Text = string.format("%s  [CHƯA CÓ]", rod.name)
            tl.TextColor3 = Colors.TextWhite
            dl.Text = string.format("%s • [Trạng thái: Chưa có]", baseDesc)
            dl.TextColor3 = Colors.TextMuted

            btn.Text = "Mua Cần"
            btn.BackgroundColor3 = Colors.ControlBg
            btn.TextColor3 = Colors.PurplePrimary
        end
    end

    btn.MouseButton1Click:Connect(function()
        local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
        local curEq = pData and pData:FindFirstChild("FishingRod") and pData.FishingRod.Value
        local isEquipped = (curEq == rod.name)
        local isOwned = isEquipped or IsRodOwned(rod.name)

        if isEquipped then
            ShowNotification("Cần Câu", "Bạn đang cầm cần " .. rod.name .. " rồi!", "INFO")
        elseif isOwned then
            if Events and Events:FindFirstChild("EquipFishingRod") then
                Events.EquipFishingRod:InvokeServer(rod.name)
                ShowNotification("Trang Bị Cần", "Đã trang bị cần: " .. rod.name, "SUCCESS")
                task.delay(0.4, function()
                    if Events and Events:FindFirstChild("ToggleHotbar") then
                        Events.ToggleHotbar:InvokeServer("1")
                    end
                end)
            end
        else
            if Events and Events:FindFirstChild("BuyFishingRod") then
                Events.BuyFishingRod:FireServer(rod.name)
                ShowNotification("Shop Cần", "Đã gửi yêu cầu mua cần: " .. rod.name .. " (" .. pStr .. ")", "SUCCESS")
            end
        end
        task.delay(0.6, UpdateAllRodShopUI)
        task.delay(1.5, UpdateAllRodShopUI)
    end)

    table.insert(rodShopUpdaters, updateRowVisuals)
    updateRowVisuals()
end

local UpdateAllIslandStatus = nil
do
createCategoryHeader(tabTeleports, "Dịch Chuyển Đến Đảo (Đảo 1 - 10)")
local islandCard = createCardGroup(tabTeleports)

local islands = WorldData.islands
local bossRealms = WorldData.bossRealms

-- Hiển thị trực tiếp vị trí đảo người chơi đang đứng
local infoCurrentMap = createInfoRow(islandCard, "📍 Vị Trí Bạn Đang Đứng", "Đang nhận diện...")

createButtonRow(islandCard, "📋 Sao Chép Tọa Độ Hiện Tại", "Copy tọa độ đứng hiện tại vào Clipboard để gửi cho AI nạp đảo mới", "Sao Chép", function()
    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if not root then return end
    local pos = root.Position
    local str = string.format("Vector3.new(%.1f, %.1f, %.1f)", pos.X, pos.Y, pos.Z)
    pcall(function()
        if setclipboard then setclipboard(str)
        elseif toclipboard then toclipboard(str) end
    end)
    ShowNotification("TỌA ĐỘ HIỆN TẠI", "Đã copy: " .. str .. " vào Clipboard!", "SUCCESS", 6)
end)

local islandUpdaters = {}
UpdateAllIslandStatus = function()
    local curLocName = GetCurrentLocationName()
    if infoCurrentMap and infoCurrentMap.Set then
        infoCurrentMap.Set(curLocName)
    end
    if StatTiles and StatTiles.CurrentLocation and StatTiles.CurrentLocation.Set then
        StatTiles.CurrentLocation.Set(curLocName)
    end
    for _, fn in ipairs(islandUpdaters) do
        pcall(fn, curLocName)
    end
end

for _, isl in ipairs(islands) do
    local row = Instance.new("Frame"); row.Size = UDim2.new(1, 0, 0, 42); row.BackgroundColor3 = Colors.RowNormal; row.BorderSizePixel = 0; row.Parent = islandCard
    local pad = Instance.new("UIPadding"); pad.PaddingLeft = UDim.new(0, 10); pad.PaddingRight = UDim.new(0, 10); pad.Parent = row
    local tf = Instance.new("Frame"); tf.Size = UDim2.new(1, -125, 1, 0); tf.BackgroundTransparency = 1; tf.Parent = row
    local tl = Instance.new("TextLabel"); tl.Size = UDim2.new(1, 0, 0, 18); tl.Position = UDim2.new(0, 0, 0, 4); tl.BackgroundTransparency = 1; tl.Font = Enum.Font.GothamBold; tl.Text = isl.name; tl.TextColor3 = Colors.TextWhite; tl.TextSize = 12; tl.TextXAlignment = Enum.TextXAlignment.Left; tl.Parent = tf
    local dl = Instance.new("TextLabel"); dl.Size = UDim2.new(1, 0, 0, 14); dl.Position = UDim2.new(0, 0, 0, 22); dl.BackgroundTransparency = 1; dl.Font = Enum.Font.Gotham; dl.Text = "Dịch chuyển đến " .. isl.name; dl.TextColor3 = Colors.TextMuted; dl.TextSize = 10; dl.TextXAlignment = Enum.TextXAlignment.Left; dl.Parent = tf

    local btn = Instance.new("TextButton"); btn.Size = UDim2.new(0, 105, 0, 24); btn.Position = UDim2.new(1, -105, 0.5, -12); btn.BackgroundColor3 = Colors.ControlBg; btn.Font = Enum.Font.GothamBold; btn.Text = "Bay Đến"; btn.TextColor3 = Colors.PurplePrimary; btn.TextSize = 11; btn.BorderSizePixel = 0; btn.Parent = row
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 4)

    row.MouseEnter:Connect(function() TweenService:Create(row, TweenInfo.new(0.15), {BackgroundColor3 = Colors.RowHover}):Play() end)
    row.MouseLeave:Connect(function() TweenService:Create(row, TweenInfo.new(0.15), {BackgroundColor3 = Colors.RowNormal}):Play() end)
    table.insert(rowSearchIndex, {frame = row, query = (isl.name .. " dịch chuyển đến đảo"):lower()})

    local function updateVisuals(curLocName)
        local isHere = (curLocName == isl.name)
        local char = LocalPlayer.Character
        local root = char and char:FindFirstChild("HumanoidRootPart")
        local dist = root and math.floor((root.Position - isl.pos).Magnitude) or 0

        if isHere then
            tl.Text = string.format("%s  [BẠN ĐANG Ở ĐÂY]", isl.name)
            tl.TextColor3 = Color3.fromRGB(120, 255, 170)
            dl.Text = "Vị trí hiện tại của bạn • Khoảng cách: 0m (Đã ở đây)"
            dl.TextColor3 = Color3.fromRGB(160, 255, 190)

            btn.Text = "Đang Ở Đây"
            btn.BackgroundColor3 = Color3.fromRGB(30, 65, 45)
            btn.TextColor3 = Color3.fromRGB(120, 255, 170)
        else
            tl.Text = isl.name
            tl.TextColor3 = Colors.TextWhite
            dl.Text = string.format("Dịch chuyển đến %s • Cách bạn: ~%dm", isl.name, dist)
            dl.TextColor3 = Colors.TextMuted

            btn.Text = "Bay Đến"
            btn.BackgroundColor3 = Colors.ControlBg
            btn.TextColor3 = Colors.PurplePrimary
        end
    end

    btn.MouseButton1Click:Connect(function()
        local curLocName = GetCurrentLocationName()
        if curLocName == isl.name then
            ShowNotification("Dịch Chuyển", "Bạn đang ở ngay " .. isl.name .. " rồi!", "INFO")
            return
        end
        local char = LocalPlayer.Character
        local root = char and char:FindFirstChild("HumanoidRootPart")
        if root then
            root.CFrame = CFrame.new(isl.pos + Vector3.new(0, 3, 0))
            ShowNotification("Dịch Chuyển", "Đã đến " .. isl.name .. "!", "SUCCESS")
            task.delay(0.4, UpdateAllIslandStatus)
        end
    end)

    table.insert(islandUpdaters, updateVisuals)
    updateVisuals(GetCurrentLocationName())
end

do
createCategoryHeader(tabTeleports, "Đấu Trường Boss & Vùng Đất Bí Mật")
local bossRealmCard = createCardGroup(tabTeleports)

local bossRealms = {
    {name = "Boss Bạch Tuộc (Phao Biển)", pos = Vector3.new(1608.2, 5.0, -218.3)},
    {name = "Vùng Câu Cá Ngầm Lòng Đất", pos = Vector3.new(112.5, -330.0, -30.8)},
    {name = "Đấu Trường Boss Enzo", pos = Vector3.new(-115.3, 9.2, 1349.5)},
}

for _, br in ipairs(bossRealms) do
    createButtonRow(bossRealmCard, br.name, "Dịch chuyển đến " .. br.name, "Bay Đến", function()
        local char = LocalPlayer.Character
        local root = char and char:FindFirstChild("HumanoidRootPart")
        if root then
            root.CFrame = CFrame.new(br.pos + Vector3.new(0, 3, 0))
            ShowNotification("Teleport", "Arrived at " .. br.name .. "!", "SUCCESS")
        end
    end)
end

createCategoryHeader(tabTeleports, "Cửa Hàng Bán Cần (Biao Di)")
local rodDealerCard = createCardGroup(tabTeleports)

local rodDealers = {
    {name = "[1] Shop Đảo Khởi Đầu", pos = Vector3.new(-151.4, 8.7, -49.9)},
    {name = "[2] Shop Đảo Tre", pos = Vector3.new(-1236.8, 7.3, -174.1)},
    {name = "[3] Shop Đảo Phóng Xạ", pos = Vector3.new(138.4, 9.0, 1179.7)},
    {name = "[4] Shop Đảo Thống Trị", pos = Vector3.new(-1262.6, 8.2, 1202.2)},
    {name = "[5] Shop Đảo Cá Chép", pos = Vector3.new(-9.5, 9.2, -1330.0)},
    {name = "[6] Shop Đảo Băng Giá", pos = Vector3.new(-1400.4, 9.2, -1490.6)},
    {name = "[7] Shop Đảo Quả Dừa", pos = Vector3.new(1446.0, 9.3, -1408.0)},
    {name = "[8] Shop Đảo Hổ Phách", pos = Vector3.new(1292.7, 8.2, 1497.4)},
}

for _, rd in ipairs(rodDealers) do
    createButtonRow(rodDealerCard, rd.name, "Bay trực tiếp đến " .. rd.name, "Bay Đến", function()
        local char = LocalPlayer.Character
        local root = char and char:FindFirstChild("HumanoidRootPart")
        if root then
            root.CFrame = CFrame.new(rd.pos + Vector3.new(0, 3, 0))
            ShowNotification("Cửa Hàng", "Đã đến " .. rd.name .. "!", "SUCCESS")
        end
    end)
end

createCategoryHeader(tabTeleports, "Vị Trí Cần Câu Bí Mật")
local sRodCard = createCardGroup(tabTeleports)

local secretRods = {
    {name = "Anchorbound Rod", pos = Vector3.new(-1208.5, 56.3, 1646.2)},
    {name = "Blazeshark Rod", pos = Vector3.new(-8.4, 53.9, 6.7)},
    {name = "Kraken Rod", pos = Vector3.new(1543.8, 73.4, 1490.9)},
    {name = "Ascendant Bamboo Rod", pos = Vector3.new(-1360.6, 140.6, 31.0)},
    {name = "Lifebloom Rod", pos = Vector3.new(-114.9, 74.9, -1533.2)},
    {name = "Demonic Rod", pos = Vector3.new(1181.9, 82.9, -1243.8)}
}

for _, sr in ipairs(secretRods) do
    createButtonRow(sRodCard, sr.name, "Bay đến vị trí lấy cần: " .. sr.name, "Bay Đến", function()
        local char = LocalPlayer.Character
        local root = char and char:FindFirstChild("HumanoidRootPart")
        if root then
            root.CFrame = CFrame.new(sr.pos + Vector3.new(0, 3, 0))
            ShowNotification("Cần Bí Mật", "Đã bay đến " .. sr.name .. "!", "SUCCESS")
        end
    end)
end
end

createCategoryHeader(tabTeleports, "👥 Dịch Chuyển Đến Người Chơi Trong Map")
local srvCard = createCardGroup(tabTeleports)

local playerLookup = {}
local selectedPlayerKey = nil

local function BuildPlayerList()
    local list = {}
    table.clear(playerLookup)
    local myChar = LocalPlayer.Character
    local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
    local myPos = myRoot and myRoot.Position

    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer then
            local distStr = ""
            if myPos and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
                local d = math.floor((p.Character.HumanoidRootPart.Position - myPos).Magnitude)
                distStr = string.format(" [%dm]", d)
            end
            local key = string.format("%s (@%s)%s", p.DisplayName, p.Name, distStr)
            table.insert(list, key)
            playerLookup[key] = p
        end
    end

    if #list == 0 then
        table.insert(list, "Không có người chơi khác")
    end
    return list
end

local initialPlayerList = BuildPlayerList()
selectedPlayerKey = initialPlayerList[1]

local playerDropdown = createDropdownRow(srvCard, "Chọn Người Chơi", "Danh sách người chơi đang có mặt trong server", initialPlayerList, selectedPlayerKey, function(v)
    selectedPlayerKey = v
end)

local function RefreshPlayerDropdown()
    local newList = BuildPlayerList()
    if playerDropdown and playerDropdown.Refresh then
        playerDropdown.Refresh(newList, true)
        selectedPlayerKey = playerDropdown.Get()
    end
end

createButtonRow(srvCard, "Bay Đến Người Chơi Đã Chọn", "Dịch chuyển tức thì đến ngay bên cạnh người chơi đang chọn", "🚀 Bay Đến", function()
    local targetPlayer = playerLookup[selectedPlayerKey]
    if not targetPlayer then
        local uName = selectedPlayerKey and selectedPlayerKey:match("@([%w_]+)")
        if uName then
            targetPlayer = Players:FindFirstChild(uName)
        end
    end

    if not targetPlayer or not targetPlayer.Parent then
        ShowNotification("Dịch Chuyển", "Vui lòng chọn người chơi hợp lệ!", "WARN")
        RefreshPlayerDropdown()
        return
    end

    local tChar = targetPlayer.Character
    local tRoot = tChar and tChar:FindFirstChild("HumanoidRootPart")
    local myChar = LocalPlayer.Character
    local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")

    if myRoot and tRoot then
        myRoot.CFrame = tRoot.CFrame + Vector3.new(0, 2, 3)
        ShowNotification("Dịch Chuyển", "Đã bay đến người chơi: " .. targetPlayer.DisplayName, "SUCCESS")
        RefreshPlayerDropdown()
    else
        ShowNotification("Dịch Chuyển", "Người chơi này chưa hồi sinh hoặc không có nhân vật!", "WARN")
    end
end)

createButtonRow(srvCard, "Làm Mới Danh Sách Người Chơi", "Cập nhật danh sách người chơi vừa tham gia hoặc rời server", "🔄 Làm Mới", function()
    RefreshPlayerDropdown()
    ShowNotification("Danh Sách", "Đã cập nhật danh sách người chơi trong map!", "INFO")
end)

local lastTpTarget = nil
createButtonRow(srvCard, "Bay Đến Người Chơi Ngẫu Nhiên", "Dịch chuyển tức thì đến vị trí của một người chơi bất kỳ", "🎲 Ngẫu Nhiên", function()
    local targets = {}
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
            table.insert(targets, p)
        end
    end
    if #targets == 0 then
        ShowNotification("Dịch Chuyển", "Không tìm thấy người chơi nào khác trong server.", "WARN")
        return
    end
    local pool = {}
    for _, p in ipairs(targets) do
        if not (#targets > 1 and p == lastTpTarget) then
            table.insert(pool, p)
        end
    end
    local selected = (#pool > 0 and pool[math.random(1, #pool)]) or targets[math.random(1, #targets)]
    lastTpTarget = selected
    local root = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    if root and selected.Character and selected.Character:FindFirstChild("HumanoidRootPart") then
        root.CFrame = selected.Character.HumanoidRootPart.CFrame + Vector3.new(0, 2, 3)
        ShowNotification("Dịch Chuyển", "Đã bay đến người chơi: " .. selected.DisplayName, "SUCCESS")
        RefreshPlayerDropdown()
    end
end)

table.insert(activeConnections, Players.PlayerAdded:Connect(function()
    task.wait(1)
    RefreshPlayerDropdown()
end))
table.insert(activeConnections, Players.PlayerRemoving:Connect(function()
    task.wait(0.5)
    RefreshPlayerDropdown()
end))
end

-- ============================================================
-- SECTION: DỊCH CHUYỂN ĐẾN NPC NHIỆM VỤ
-- ============================================================
do
createCategoryHeader(tabTeleports, "🧙 Dịch Chuyển Đến NPC Nhiệm Vụ")
local npcTeleCard = createCardGroup(tabTeleports)

local questNPCList = {
    -- Main Quest NPCs
    { name="Ha Dieu De",            path="Function",  icon="🏆", role="Main Quest",          desc="Trả quest câu Rainbow Dragonfish (≥7M KG) & Heavenpiercer Turtle • Mở khóa Cần Heavenpiercer", island="Bamboo / Coconut / Frost" },
    { name="Giang Lao",             path="Function",  icon="🎣", role="Main Quest",          desc="NPC nhiệm vụ chính • Trả quest câu cá nặng • Mở khóa tiến trình game", island="Bamboo / Mistpeak / World Angler" },
    { name="Sage Yijiu",            path="Function",  icon="🔮", role="Skill Shop + Quest",  desc="Bán skill Heavenpiercer & Pure Diamond • Trả quest câu cá Frost", island="Frost Isle / World Angler" },
    { name="Blind Grand Angler",    path="Function",  icon="👁️", role="Main Quest",          desc="Lão ngư ông mù • Trả quest nhiệm vụ chính • Mở khóa câu cá bí mật", island="Battlefield Isle" },
    { name="Duan Gan",              path="Function",  icon="🗡️", role="Main Quest",          desc="NPC võ sĩ gãy cần • Quest chính • Liên quan cần Huyết Long", island="Đảo chính" },
    -- Skill Upgrade NPCs
    { name="Bac Minh",              path="Function",  icon="⬆️", role="Skill Shop",          desc="Bán và nâng cấp kỹ năng câu cá • Cần Gems để nâng level", island="Đảo chính" },
    { name="Zeng Tianguo",          path="Function",  icon="⚡", role="Skill Upgrade",       desc="Nâng cấp kỹ năng đặc biệt • Có ở Perch Isle & Sovereign Isle", island="Perch / Sovereign" },
    { name="Tang Thien Quoc",       path="Function",  icon="🌟", role="Skill Upgrade",       desc="NPC nâng cấp kỹ năng cấp cao • Sovereign Isle (Power 39) & Perch Isle", island="Sovereign / Perch" },
    -- Craft & Shop NPCs
    { name="Biao Di",               path="Function",  icon="🎯", role="Thợ Chế Cần Câu",    desc="Craft & mua bán cần câu • Chế Heavenpiercer, Pure Diamond, Sacred Bamboo • Cần nguyên liệu boss", island="Mọi đảo chính" },
    { name="Hua Heshang",           path="Function",  icon="🐉", role="Dragon Quest",        desc="NPC mới • Yêu cầu skill Dragon Subjugation (drop từ Dark Kingfish)", island="Đảo chính" },
    { name="The Shadow",            path="Function",  icon="🌑", role="Bí Mật / PVP",       desc="NPC bí ẩn • Liên quan nhiệm vụ bí mật và PVP arena đặc biệt", island="Đảo chính" },
    { name="Lao Ngo",               path="Function",  icon="👴", role="Quest Phụ",           desc="Lão Ngô • Nhiệm vụ phụ • Cung cấp thông tin câu cá bí ẩn", island="Đảo chính" },
    { name="Nanjiang",              path="Function",  icon="🗺️", role="Quest Phụ",           desc="Nam Giang • Nhiệm vụ phụ • Thông tin đảo và vị trí câu hiếm", island="Đảo chính" },
    { name="Giang Lao PVP",         path="Function",  icon="⚔️", role="PVP Arena",           desc="Tham gia đấu trường PVP câu cá • Nhận Stars để đổi skin cần câu", island="Battlefield Isle" },
    { name="Battlefield Isle's Giang Lao", path="Function", icon="🏟️", role="Battlefield Quest", desc="Quest đấu trường Battlefield • Mở khóa khu vực đặc biệt", island="Battlefield Isle" },
    { name="Ticket Quest Giver",    path="Function",  icon="🎫", role="Sự Kiện",             desc="Phát nhiệm vụ vé sự kiện • Đổi vé lấy phần thưởng giới hạn", island="Đảo chính" },
    -- Utility NPCs
    { name="Nana",                  path="SellFish",  icon="🐟", role="Bán Cá",             desc="Thu mua & bán cá • Bán cá nhanh lấy Gems • Có mặt ở mọi đảo", island="Mọi đảo" },
    { name="Ba Chang",              path="BuyBait",   icon="🪱", role="Bán Mồi Câu",        desc="Bán mồi câu cơ bản (Worm, Shrimp Bait...) • Giá rẻ cho người mới", island="Đảo chính / Coconut" },
}

local function findNPCModel(npcName, npcPath)
    local npcFolder = Workspace:FindFirstChild("NPC")
    if npcFolder then
        local pathFolder = npcFolder:FindFirstChild(npcPath)
        if pathFolder then
            local found = pathFolder:FindFirstChild(npcName)
            if found and (found:FindFirstChild("HumanoidRootPart") or found:IsA("BasePart")) then
                return found
            end
        end
        for _, sub in ipairs(npcFolder:GetChildren()) do
            local found = sub:FindFirstChild(npcName)
            if found then return found end
        end
    end
    return Workspace:FindFirstChild(npcName, true)
end

for _, npc in ipairs(questNPCList) do
    createButtonRow(
        npcTeleCard,
        npc.icon .. " " .. npc.name .. "  [" .. npc.role .. "]",
        npc.desc .. "\n📍 " .. npc.island,
        "Bay Đến",
        function()
            local char = LocalPlayer.Character
            local root = char and char:FindFirstChild("HumanoidRootPart")
            if not root then ShowNotification("Lỗi", "Nhân vật chưa spawn!", "ERROR") return end
            local model = findNPCModel(npc.name, npc.path)
            if model then
                local npcRoot = model:FindFirstChild("HumanoidRootPart")
                    or model:FindFirstChildWhichIsA("BasePart")
                if npcRoot then
                    root.CFrame = CFrame.new(npcRoot.Position + Vector3.new(0, 3, 3))
                    ShowNotification("✅ Đến " .. npc.name, "Đã bay đến NPC " .. npc.name .. "!", "SUCCESS", 4)
                    return
                end
            end
            ShowNotification("❌ Không Tìm Thấy", "NPC '" .. npc.name .. "' không có trong map lúc này. Có thể chưa spawn hoặc đang ở đảo khác.", "WARN", 5)
        end
    )
end

-- ============================================================
-- SECTION: DỊCH CHUYỂN ĐẾN ĐẠO SĨ (TAOIST & MAOSHAN)
-- ============================================================
createCategoryHeader(tabTeleports, "📜 Dịch Chuyển Đến Đạo Sĩ (Taoist & Maoshan)")
local taoistTeleCard = createCardGroup(tabTeleports)

createButtonRow(taoistTeleCard, "📜 Bay Đến Đạo Sĩ (Taoist)", "Dịch chuyển tức thì đến NPC Đạo Sĩ (Taoist) nếu có trong server", "Bay Đến", function()
    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if not root then ShowNotification("Lỗi", "Nhân vật chưa spawn!", "ERROR") return end
    local tInst, tName = secretBossState.ScanForTaoistNPC()
    if tInst then
        local pivot = (tInst:IsA("Model") and tInst:GetPivot()) or (tInst:IsA("BasePart") and tInst.CFrame)
        if pivot then
            root.CFrame = pivot + Vector3.new(0, 3, 4)
            ShowNotification("Đạo Sĩ (Taoist)", "Đã dịch chuyển đến vị trí " .. tostring(tName) .. "!", "SUCCESS", 5)
        else
            ShowNotification("Đạo Sĩ (Taoist)", "Không lấy được tọa độ Đạo Sĩ.", "WARN")
        end
    else
        ShowNotification("Đạo Sĩ (Taoist)", "Server này hiện chưa có Đạo Sĩ (Taoist)! Hãy bật 'Đổi Server Tìm Taoist'.", "WARN", 6)
    end
end)

createButtonRow(taoistTeleCard, "✨ Bay Đến Đạo Sĩ Maoshan", "Dịch chuyển tức thì đến NPC Đạo Sĩ Maoshan nếu có trong server", "Bay Đến", function()
    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if not root then ShowNotification("Lỗi", "Nhân vật chưa spawn!", "ERROR") return end
    local mInst, mName = secretBossState.ScanForMaoshanNPC()
    if mInst then
        local pivot = (mInst:IsA("Model") and mInst:GetPivot()) or (mInst:IsA("BasePart") and mInst.CFrame)
        if pivot then
            root.CFrame = pivot + Vector3.new(0, 3, 4)
            ShowNotification("Đạo Sĩ Maoshan", "Đã dịch chuyển đến vị trí " .. tostring(mName) .. "!", "SUCCESS", 5)
        else
            ShowNotification("Đạo Sĩ Maoshan", "Không lấy được tọa độ Đạo Sĩ Maoshan.", "WARN")
        end
    else
        ShowNotification("Đạo Sĩ Maoshan", "Server này hiện chưa có Đạo Sĩ Maoshan! Hãy bật 'Đổi Server Tìm Maoshan'.", "WARN", 6)
    end
end)

end

do
createCategoryHeader(tabVisuals, "ESP Nhìn Xuyên Tường")
local espCard = createCardGroup(tabVisuals)

createToggleRow(espCard, "ESP Thần Linh (God Spirit)", "Hiện vị trí Thần linh xuyên bản đồ", Config.ESP_GodSpirit, function(v) Config.ESP_GodSpirit = v end)
createToggleRow(espCard, "ESP Cần Câu Bí Mật", "Hiện vị trí các cần câu ẩn trên bản đồ", Config.ESP_SecretRod, function(v) Config.ESP_SecretRod = v end)
createToggleRow(espCard, "ESP Thuyền Bè", "Hiện vị trí tất cả thuyền xung quanh", Config.ESP_Boats, function(v) Config.ESP_Boats = v end)
createToggleRow(espCard, "ESP Maoshan", "Hiện vị trí NPC hoặc cần Maoshan", Config.ESP_Maoshan, function(v) Config.ESP_Maoshan = v end)
createToggleRow(espCard, "ESP Đạo Sĩ (Taoist)", "Hiện vị trí NPC hoặc cần Taoist", Config.ESP_Taoist, function(v) Config.ESP_Taoist = v end)
createToggleRow(espCard, "ESP Trùm Boss", "Hiện vị trí các Boss đang xuất hiện", Config.ESP_Boss, function(v) Config.ESP_Boss = v end)
createToggleRow(espCard, "ESP Người Chơi", "Hiện khung & khoảng cách đến người chơi khác", Config.ESP_Players, function(v) Config.ESP_Players = v end)
createToggleRow(espCard, "Ẩn Tên Mặc Định Người Chơi", "Ẩn toàn bộ bảng tên, danh hiệu và thanh máu trên đầu của người chơi khác", Config.HideOverheadNames, function(v)
    Config.HideOverheadNames = v
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character then
            local hum = p.Character:FindFirstChildOfClass("Humanoid")
            if hum then
                hum.DisplayDistanceType = v and Enum.HumanoidDisplayDistanceType.None or Enum.HumanoidDisplayDistanceType.Viewer
            end
            for _, d in ipairs(p.Character:GetDescendants()) do
                if d:IsA("BillboardGui") and d.Name:sub(1, 4) ~= "ESP_" then
                    d.Enabled = not v
                end
            end
        end
    end
end)
createToggleRow(espCard, "Vòng Tròn Định Vị Cá", "Hiện vòng tròn đỏ dưới nước chỉ đúng con cá cắn câu", Config.FishRedRing, function(v) Config.FishRedRing = v end)
createToggleRow(espCard, "Hiện Cân Nặng & Đột Biến Trên Vòng Đỏ", "Hiển thị tên cá, cân nặng (kg) và loại đột biến trực tiếp trên vòng định vị", Config.ShowFishWeightRing, function(v) Config.ShowFishWeightRing = v end)

createCategoryHeader(tabVisuals, "Ánh Sáng & Tối Ưu Giảm Lag")
local perfCard = createCardGroup(tabVisuals)

local clearVisionState = {}

function clearVisionState.Apply(enabled)
    if enabled then
        pcall(function()
            Lighting.FogEnd = 1000000
            Lighting.FogStart = 1000000
        end)
        for _, obj in ipairs(Lighting:GetDescendants()) do
            pcall(function()
                if obj:IsA("DepthOfFieldEffect") then
                    obj.Enabled = false
                    obj.FarIntensity = 0
                    obj.NearIntensity = 0
                elseif obj:IsA("BlurEffect") then
                    obj.Enabled = false
                    obj.Size = 0
                elseif obj:IsA("Atmosphere") then
                    obj.Density = 0
                    obj.Haze = 0
                    obj.Glare = 0
                    obj.Offset = 0
                end
            end)
        end
        for _, obj in ipairs(Camera:GetDescendants()) do
            pcall(function()
                if obj:IsA("DepthOfFieldEffect") then
                    obj.Enabled = false
                    obj.FarIntensity = 0
                    obj.NearIntensity = 0
                elseif obj:IsA("BlurEffect") then
                    obj.Enabled = false
                    obj.Size = 0
                elseif obj:IsA("ParticleEmitter") then
                    obj.Enabled = false
                end
            end)
        end
        for _, obj in ipairs(Workspace:GetDescendants()) do
            pcall(function()
                if obj:IsA("DepthOfFieldEffect") or obj:IsA("BlurEffect") then
                    obj.Enabled = false
                end
            end)
        end
    else
        pcall(function()
            Lighting.FogEnd = 100000
            Lighting.FogStart = 0
        end)
        for _, obj in ipairs(Lighting:GetDescendants()) do
            pcall(function()
                if obj:IsA("DepthOfFieldEffect") or obj:IsA("BlurEffect") then
                    obj.Enabled = true
                elseif obj:IsA("Atmosphere") then
                    obj.Density = 0.3
                    obj.Haze = 0.5
                end
            end)
        end
    end
end

createToggleRow(perfCard, "Tầm Nhìn Xa (Xóa Mờ Map)", "Tắt hiệu ứng làm mờ xa (DepthOfField) & sương mù, nhìn rõ mọi hòn đảo từ xa", Config.ClearFarVision, function(v)
    Config.ClearFarVision = v
    clearVisionState.Apply(v or Config.NoFog)
end)

createToggleRow(perfCard, "Xóa Sương Mù & Mưa Bão", "Xóa sạch sương mù, khói mờ, hạt mưa và sấm sét che khuất tầm nhìn", Config.NoFog, function(v)
    Config.NoFog = v
    clearVisionState.Apply(v or Config.ClearFarVision)
end)

-- Lắng nghe đối tượng hiệu ứng mới sinh ra từ hệ thống thời tiết game để triệt tiêu tức thì
table.insert(activeConnections, Lighting.DescendantAdded:Connect(function(obj)
    if not (Config.ClearFarVision or Config.NoFog) then return end
    pcall(function()
        if obj:IsA("DepthOfFieldEffect") then
            obj.Enabled = false
            obj.FarIntensity = 0
            obj.NearIntensity = 0
        elseif obj:IsA("BlurEffect") then
            obj.Enabled = false
            obj.Size = 0
        elseif obj:IsA("Atmosphere") then
            obj.Density = 0
            obj.Haze = 0
            obj.Glare = 0
            obj.Offset = 0
        end
    end)
end))

table.insert(activeConnections, Camera.DescendantAdded:Connect(function(obj)
    if not (Config.ClearFarVision or Config.NoFog) then return end
    pcall(function()
        if obj:IsA("DepthOfFieldEffect") or obj:IsA("BlurEffect") then
            obj.Enabled = false
        elseif obj:IsA("ParticleEmitter") then
            obj.Enabled = false
        end
    end)
end))

-- Vòng lặp duy trì tầm nhìn xa liên tục chống game tự bật lại mờ/sương khi đổi thời tiết
task.spawn(function()
    while isRunning do
        if Config.ClearFarVision or Config.NoFog then
            pcall(function()
                if Lighting.FogEnd < 500000 then
                    Lighting.FogEnd = 1000000
                    Lighting.FogStart = 1000000
                end
                for _, obj in ipairs(Lighting:GetDescendants()) do
                    if obj:IsA("DepthOfFieldEffect") and obj.Enabled then
                        obj.Enabled = false
                        obj.FarIntensity = 0
                    elseif obj:IsA("BlurEffect") and obj.Enabled then
                        obj.Enabled = false
                    elseif obj:IsA("Atmosphere") and (obj.Density > 0.01 or obj.Haze > 0.01) then
                        obj.Density = 0
                        obj.Haze = 0
                    end
                end
                for _, obj in ipairs(Camera:GetDescendants()) do
                    if (obj:IsA("DepthOfFieldEffect") or obj:IsA("BlurEffect")) and obj.Enabled then
                        obj.Enabled = false
                    end
                end
            end)
        end
        task.wait(2)
    end
end)

-- Tự động kích hoạt tầm nhìn xa ngay khi load script nếu đang bật
if Config.ClearFarVision or Config.NoFog then
    clearVisionState.Apply(true)
end

local function ApplyFullbright(enabled)
    if enabled then
        local brightLevel = tonumber(Config.FullbrightLevel) or 2.0
        Lighting.Brightness = math.clamp(brightLevel, 1.0, 3.5)
        Lighting.Ambient = Color3.fromRGB(140, 140, 140)
        Lighting.OutdoorAmbient = Color3.fromRGB(140, 140, 140)
        Lighting.GlobalShadows = false
        Lighting.ExposureCompensation = 0
        local atmo = Lighting:FindFirstChildWhichIsA("Atmosphere")
        if atmo then
            atmo.Density = 0.05
            atmo.Haze = 0
            atmo.Glare = 0
        end
        local bloom = Lighting:FindFirstChildWhichIsA("BloomEffect")
        if bloom then
            bloom.Intensity = 0.1
            bloom.Size = 10
        end
    else
        Lighting.Brightness = 2
        Lighting.Ambient = Color3.fromRGB(70, 70, 70)
        Lighting.OutdoorAmbient = Color3.fromRGB(70, 70, 70)
        Lighting.GlobalShadows = true
        Lighting.ExposureCompensation = 0
        local atmo = Lighting:FindFirstChildWhichIsA("Atmosphere")
        if atmo then
            atmo.Density = 0.3
            atmo.Haze = 0.5
        end
        local bloom = Lighting:FindFirstChildWhichIsA("BloomEffect")
        if bloom then
            bloom.Intensity = 1
        end
    end
end

-- Tự động cân bằng ánh sáng chống chói khi game đổi thời tiết (Windy, Sunny, Night...)
table.insert(activeConnections, Lighting.Changed:Connect(function(prop)
    if Config.Fullbright and (Config.FullbrightAntiGlare ~= false) then
        if prop == "Brightness" and Lighting.Brightness > 3.0 then
            Lighting.Brightness = tonumber(Config.FullbrightLevel) or 2.0
        elseif prop == "ExposureCompensation" and Lighting.ExposureCompensation > 0.1 then
            Lighting.ExposureCompensation = 0
        end
    end
end))

createToggleRow(perfCard, "Sáng Màn Hình (Fullbright)", "Tăng độ sáng dịu mắt, nhìn rõ trong đêm mà không bị chói", Config.Fullbright, function(v)
    Config.Fullbright = v
    ApplyFullbright(v)
end)

createSliderRow(perfCard, "Mức Độ Sáng (Fullbright)", "Tùy chỉnh độ sáng màn hình theo mắt của bạn", 1.0, 3.5, Config.FullbrightLevel or 2.0, true, "x", function(v)
    Config.FullbrightLevel = v
    if Config.Fullbright then
        ApplyFullbright(true)
    end
end)

createToggleRow(perfCard, "Chống Lóa Thời Tiết (Anti-Glare)", "Tự động kìm hãm ánh sáng khi thời tiết đổi sang gió bão, trời nắng chói", Config.FullbrightAntiGlare, function(v)
    Config.FullbrightAntiGlare = v
    if Config.Fullbright then
        ApplyFullbright(true)
    end
end)

-- Áp dụng ngay khi script load nếu bật sẵn
if Config.Fullbright then
    ApplyFullbright(true)
end

createToggleRow(perfCard, "Chế Độ Giảm Lag (Low GFX)", "Tắt bóng đổ và giảm tải đồ họa giúp game siêu mượt", Config.PerformanceMode, function(v)
    Config.PerformanceMode = v
    Lighting.GlobalShadows = not v
end)




createToggleRow(perfCard, "Ẩn Giao Diện Gốc Của Game", "Ẩn các nút bấm của Roblox để nhìn thông thoáng", Config.HideGameUI, function(v)
    Config.HideGameUI = v
    local pg = LocalPlayer:FindFirstChild("PlayerGui")
    if pg then
        if pg:FindFirstChild("MainGui") then pg.MainGui.Enabled = not v end
        if pg:FindFirstChild("Fisher_GUI") then pg.Fisher_GUI.Enabled = not v end
    end
end)

createButtonRow(perfCard, "Mở Khóa Toàn Bộ Sách Cá (Index)", "Mở khóa khám phá đủ 109 loài cá và vật phẩm trong Index", "Mở Khóa Index", function()
    local count = 0
    local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
    if pData and pData:FindFirstChild("Index") then
        for _, b in ipairs(pData.Index:GetChildren()) do
            if b:IsA("BoolValue") then b.Value = true end
        end
    end
    local pg = LocalPlayer:FindFirstChild("PlayerGui")
    if pg and pg:FindFirstChild("MainGui") and pg.MainGui:FindFirstChild("Menu") and pg.MainGui.Menu:FindFirstChild("Index") then
        local indexList = pg.MainGui.Menu.Index:FindFirstChild("IndexFrame") and pg.MainGui.Menu.Index.IndexFrame:FindFirstChild("Indexlist")
        if indexList then
            for _, f in ipairs(indexList:GetChildren()) do
                if f:IsA("Frame") then
                    count = count + 1
                    local btn = f:FindFirstChild("Button")
                    if btn then
                        local title = btn:FindFirstChild("Title")
                        if title and title:IsA("TextLabel") then title.Text = f.Name end
                        local detail = btn:FindFirstChild("Detail")
                        if detail then
                            detail.Visible = true
                            for _, img in ipairs(detail:GetDescendants()) do
                                if img:IsA("ImageLabel") then img.ImageColor3 = Color3.new(1, 1, 1) end
                            end
                        end
                    end
                end
            end
        end
    end
    ShowNotification("Mở Khóa Index", string.format("Đã mở khóa %d loài cá trong Sách Cá Index!", count > 0 and count or 109), "SUCCESS")
end)
end

do
createCategoryHeader(tabPlayer, "📜 Trích Xuất Dữ Liệu Kỹ Năng (Skill Info Exporter)")
local exportSkillCard = createCardGroup(tabPlayer)
local infoSkillCount = createInfoRow(exportSkillCard, "Kỹ Năng Đã Quét", "Chưa quét dữ liệu")

createButtonRow(exportSkillCard, "Quét Kỹ Năng Đang Sở Hữu (Chỉ Của Bạn)", "Chỉ quét các kỹ năng bạn thực sự sở hữu trong túi đồ & phím Z,X,C,V", "👤 Skill Của Bạn", function()
    ExportAllPlayerSkills(infoSkillCount, true)
end)

createButtonRow(exportSkillCard, "Quét Bách Khoa Toàn Bộ Kỹ Năng Game", "Quét toàn bộ từ điển kỹ năng có trong game (Codex/Shop/Tất cả)", "📚 Toàn Bộ Game", function()
    ExportAllPlayerSkills(infoSkillCount, false)
end)

createButtonRow(exportSkillCard, "Mở Bảng Xem Danh Sách Skill", "Mở khung văn bản cuộn trên màn hình để xem và copy", "📜 Mở Bảng Xem", function()
    ShowSkillTextWindow()
end)

createCategoryHeader(tabPlayer, "Di Chuyển Nhân Vật")
local moveCard = createCardGroup(tabPlayer)

createToggleRow(moveCard, "Tăng Tốc Độ Chạy (Speed)", "Chạy nhanh hơn tốc độ mặc định", Config.WalkSpeedEnabled, function(v)
    Config.WalkSpeedEnabled = v
    local hum = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
    if hum then hum.WalkSpeed = v and Config.WalkSpeedValue or 16 end
end)
createSliderRow(moveCard, "Chỉnh Tốc Độ", "Tốc độ chạy mong muốn", 16, 120, Config.WalkSpeedValue, false, "", function(v)
    Config.WalkSpeedValue = v
    if Config.WalkSpeedEnabled then
        local hum = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
        if hum then hum.WalkSpeed = v end
    end
end)

createToggleRow(moveCard, "Bay Lượn Tự Do (Fly)", "Bay tự do trên không bằng phím WASD & Cách/Shift", Config.FlyEnabled, function(v) Config.FlyEnabled = v end)
createSliderRow(moveCard, "Tốc Độ Bay", "Tốc độ di chuyển khi bay", 20, 150, Config.FlySpeed, false, "", function(v) Config.FlySpeed = v end)
createToggleRow(moveCard, "Nhảy Vô Hạn (Infinite Jump)", "Nhảy liên tục trên không trung không giới hạn", Config.InfiniteJump, function(v) Config.InfiniteJump = v end)
createToggleRow(moveCard, "Đi Trên Mặt Nước", "Đi bộ trên mặt biển như trên đất liền", Config.WalkOnWater, function(v) Config.WalkOnWater = v end)
createToggleRow(moveCard, "Khiên Nước Axit (Acid Shield)", "Tạo sàn nổi kháng sát thương độc/axit tại Đảo Fallout", Config.AcidWaterShield, function(v) Config.AcidWaterShield = v end)
createToggleRow(moveCard, "Đi Xuyên Tường (Noclip)", "Đi xuyên qua vách núi, tường rào và vật cản", Config.Noclip, function(v) Config.Noclip = v end)

createCategoryHeader(tabPlayer, "Chống Văng Game & Ổn Định")
local stabCard = createCardGroup(tabPlayer)
createToggleRow(stabCard, "Chống Văng Game (Anti-AFK)", "Chống bị Roblox kick sau 20 phút treo máy", Config.AntiAFK, function(v) Config.AntiAFK = v end)
createToggleRow(stabCard, "Tự Động Kết Nối Lại", "Tự động vào lại server nếu bị mất kết nối", Config.AutoRejoin, function(v) Config.AutoRejoin = v end)

createCategoryHeader(tabPlayer, "🎭 Ảo Hoá Tài Sản (Visual Spoof)")
local spoofCard = createCardGroup(tabPlayer)

local ticketInputRow
local gemsInputRow

ticketInputRow = createInputRow(spoofCard, "Ảo Hoá Vé Nhiệm Vụ", "Nhập số vé ảo mong muốn và ấn Enter (Tự lưu vĩnh viễn vào máy)", (visualSpoofState.fakeTicket and visualSpoofState.fakeTicket > 0) and tostring(visualSpoofState.fakeTicket) or "", function(txt)
    local cleanDigits = txt:gsub("[^%d]", "")
    local num = tonumber(cleanDigits) or 0
    visualSpoofState.fakeTicket = num
    visualSpoofState.Save()
    visualSpoofState.Apply()
    if num > 0 then
        ShowNotification("Ảo Hoá Vé", "Đã ảo hoá thành công: " .. FormatWithSpaces(num) .. " Vé (Đã lưu máy)!", "SUCCESS", 5)
    else
        ShowNotification("Ảo Hoá Vé", "Đã hủy ảo hoá vé! Trở về số lượng thực tế.", "INFO", 5)
    end
end, "ao hoa ve nhiem vu", "Nhập số vé... (VD: 99999)")

gemsInputRow = createInputRow(spoofCard, "Ảo Hoá Gems / Đá Quý", "Nhập số Gems ảo mong muốn và ấn Enter (Tự lưu vĩnh viễn vào máy)", (visualSpoofState.fakeGems and visualSpoofState.fakeGems > 0) and tostring(visualSpoofState.fakeGems) or "", function(txt)
    local cleanDigits = txt:gsub("[^%d]", "")
    local num = tonumber(cleanDigits) or 0
    visualSpoofState.fakeGems = num
    visualSpoofState.Save()
    visualSpoofState.Apply()
    if num > 0 then
        ShowNotification("Ảo Hoá Gems", "Đã ảo hoá thành công: " .. FormatWithSpaces(num) .. " Gems (Đã lưu máy)!", "SUCCESS", 5)
    else
        ShowNotification("Ảo Hoá Gems", "Đã hủy ảo hoá Gems! Trở về số lượng thực tế.", "INFO", 5)
    end
end, "ao hoa gems da quy", "Nhập số Gems... (VD: 999999)")

createButtonRow(spoofCard, "Khôi Phục Số Thật (Reset)", "Tắt toàn bộ ảo hoá và khôi phục hiển thị số lượng thực tế", "🔄 Khôi Phục Thật", function()
    visualSpoofState.fakeTicket = 0
    visualSpoofState.fakeGems = 0
    visualSpoofState.Save()
    visualSpoofState.Apply(true)
    if ticketInputRow and ticketInputRow.Set then ticketInputRow.Set("") end
    if gemsInputRow and gemsInputRow.Set then gemsInputRow.Set("") end
    ShowNotification("Ảo Hoá", "Đã xóa toàn bộ số ảo và khôi phục số lượng thật!", "SUCCESS", 5)
end)
end

do
    createCategoryHeader(tabProfiles, "Quản Lý Cấu Hình (Profile)")
    local profCard = createCardGroup(tabProfiles)

    local curAccName = (LocalPlayer and LocalPlayer.Name) or "DefaultUser"
    createInfoRow(profCard, "Tài Khoản Hiện Tại", curAccName)
    createInfoRow(profCard, "Tự Động Lưu Thiết Yếu (Máy)", "Tự động lưu Skill, Combo, Tiện ích")

    createButtonRow(profCard, "Lưu Thiết Yếu Vào Máy", "Lưu ngay toàn bộ Skill, Combo, Tiện ích hiện tại vào file máy", "💾 Lưu Ngay", function()
        if Config._saveEssential then
            Config._saveEssential()
            ShowNotification("Lưu Thiết Yếu", "Đã lưu cài đặt thiết yếu vào máy thành công!", "SUCCESS")
        end
    end)

    createButtonRow(profCard, "Nạp Lại Thiết Yếu Từ Máy", "Đọc và cập nhật lại cấu hình thiết yếu đã lưu từ file máy", "🔄 Nạp Lại", function()
        if Config._loadEssential then
            local ok = Config._loadEssential()
            if ok then
                ShowNotification("Nạp Thiết Yếu", "Đã nạp cài đặt thiết yếu từ máy thành công!", "SUCCESS")
            else
                ShowNotification("Nạp Thiết Yếu", "Chưa có file cấu hình thiết yếu nào trên máy!", "WARN")
            end
        end
    end)

    local initialConfigs = GetSavedConfigList()
    local selectedConfigName = initialConfigs[1] or ""
    local newConfigInputName = ""

    local configDropdown = createDropdownRow(profCard, "Chọn Cấu Hình Đã Lưu", "Danh sách toàn bộ cấu hình riêng của tài khoản " .. curAccName, #initialConfigs > 0 and initialConfigs or {"(Chưa có cấu hình)"}, function(val)
        selectedConfigName = val
    end)

    local function RefreshProfileDropdown(preferredSelect)
        local updatedList = GetSavedConfigList()
        if #updatedList == 0 then
            configDropdown.Refresh({"(Chưa có cấu hình)"})
            selectedConfigName = ""
        else
            configDropdown.Refresh(updatedList)
            if preferredSelect and table.find(updatedList, preferredSelect) then
                configDropdown.Set(preferredSelect)
                selectedConfigName = preferredSelect
            else
                selectedConfigName = configDropdown.Get()
            end
        end
    end

    createInputRow(profCard, "Đặt Tên Cấu Hình Mới", "Nhập tên bất kỳ để lưu không giới hạn cấu hình", "", function(txt)
        newConfigInputName = txt
    end)

    createButtonRow(profCard, "Lưu Cấu Hình Mới", "Lưu toàn bộ cài đặt hiện tại thành một file cấu hình mới", "💾 Lưu Config", function()
        local targetName = newConfigInputName:gsub("^%s+", ""):gsub("%s+$", "")
        if #targetName == 0 then
            ShowNotification("Lưu Config", "Vui lòng nhập tên cấu hình mới vào ô phía trên!", "WARN")
            return
        end
        local ok, res = SaveAccountConfig(targetName)
        if ok then
            ShowNotification("Lưu Config", "Đã lưu cấu hình [" .. res .. "] cho tài khoản " .. curAccName .. "!", "SUCCESS")
            RefreshProfileDropdown(res)
        else
            ShowNotification("Lưu Thất Bại", tostring(res), "ERROR")
        end
    end)

    createButtonRow(profCard, "Áp Dụng Cấu Hình (Load)", "Tải và đồng bộ hóa toàn bộ cài đặt từ cấu hình đã chọn", "🚀 Nạp Config", function()
        local curSel = configDropdown.Get() or selectedConfigName
        if not curSel or curSel == "" or curSel == "(Chưa có cấu hình)" then
            ShowNotification("Nạp Config", "Tài khoản " .. curAccName .. " chưa chọn hoặc chưa có cấu hình nào!", "WARN")
            return
        end
        local ok, res = LoadAccountConfig(curSel)
        if ok then
            ShowNotification("Nạp Config", "Đã nạp và đồng bộ giao diện cấu hình [" .. res .. "]!", "SUCCESS")
        else
            ShowNotification("Nạp Thất Bại", tostring(res), "ERROR")
        end
    end)

    createButtonRow(profCard, "Làm Mới Danh Sách", "Quét lại tất cả các file cấu hình hiện có của tài khoản", "🔄 Làm Mới", function()
        RefreshProfileDropdown()
        ShowNotification("Danh Sách Config", "Đã cập nhật lại danh sách cấu hình của " .. curAccName .. "!", "INFO")
    end)

    createButtonRow(profCard, "Xóa Cấu Hình Đã Chọn", "Xóa vĩnh viễn file cấu hình đang được chọn khỏi máy", "🗑️ Xóa Config", function()
        local curSel = configDropdown.Get() or selectedConfigName
        if not curSel or curSel == "" or curSel == "(Chưa có cấu hình)" then
            ShowNotification("Xóa Config", "Chưa chọn file cấu hình hợp lệ để xóa!", "WARN")
            return
        end
        local ok, res = DeleteAccountConfig(curSel)
        if ok then
            ShowNotification("Xóa Config", "Đã xóa cấu hình [" .. res .. "] thành công!", "SUCCESS")
            RefreshProfileDropdown()
        else
            ShowNotification("Xóa Thất Bại", tostring(res), "ERROR")
        end
    end)

    createButtonRow(profCard, "Xóa Bộ Nhớ Cấu Hình Tự Lưu (Reset Cache Máy)", "Xóa file essential_config.json lưu ngầm trên máy để xóa triệt để cài đặt cũ", "⚠️ Xóa Cache Máy", function()
        EnsureAccountConfigDir()
        local accDir = GetAccountConfigDir()
        local filePath = accDir .. "/essential_config.json"
        if isfile and isfile(filePath) and delfile then
            pcall(function() delfile(filePath) end)
        end
        Config.LoopSkills = "Z, X, V"
        Config.TicketQuickSkill = "Chiêu V"
        Config.TicketSkillKey = "Chiêu Z"
        Config.TrainSkill = "Z"
        for key, ctrl in pairs(UIControllers) do
            if Config[key] ~= nil and ctrl and ctrl.Set then
                pcall(function() ctrl.Set(Config[key], true) end)
            end
        end
        ShowNotification("Reset Cache", "Đã xóa sạch file cache máy và đặt lại combo về [Z, X, V]!", "SUCCESS", 6)
    end)
end

do
createCategoryHeader(tabProfiles, "👑 Quản Lý Độ Ưu Tiên (Priority Manager)")
local priCard = createCardGroup(tabProfiles)

createToggleRow(priCard, "Bật Quản Lý Độ Ưu Tiên", "Tự động phân định thứ bậc khi nhiều tính năng chạy cùng lúc, tránh xung đột", Config.PrioritySystemEnabled, function(v)
    Config.PrioritySystemEnabled = v
    PriorityManager.UpdateUI()
    if Config._triggerAutoSave then Config._triggerAutoSave() end
end)

PriorityManager.uiStatusRow = createInfoRow(priCard, "Tác Vụ Đang Thực Thi", "💤 Đang Chờ (Idle)")

local priorityPresetNames = {
    "Mặc Định: Săn Boss > Vé NV > Thần Linh > Luyện Chiêu > Farm Thường",
    "Cày Vé: Vé NV > Săn Boss > Thần Linh > Luyện Chiêu > Farm Thường",
    "Luyện Chiêu: Luyện Chiêu > Săn Boss > Vé NV > Thần Linh > Farm Thường",
    "Tùy Biến Thứ Hạng (Custom)"
}

local rankOptions = {"1 (Cao Nhất)", "2", "3", "4", "5 (Thấp Nhất)"}
local function rankToNumber(str)
    if not str then return 5 end
    local num = tostring(str):match("(%d+)")
    return tonumber(num) or 5
end

local function numberToRank(num)
    num = tonumber(num) or 5
    if num == 1 then return "1 (Cao Nhất)" end
    if num == 5 then return "5 (Thấp Nhất)" end
    return tostring(num)
end

local isUpdatingPriorityUI = false
local dropSecretBoss, dropTicketQuest, dropGodSpirit, dropTrainSkill, dropNormalFarm

local function applyPriorityPreset(pName)
    if isUpdatingPriorityUI then return end
    isUpdatingPriorityUI = true

    Config.PriorityPreset = pName
    if pName:find("Mặc Định") then
        Config.Priority_SecretBoss = 1
        Config.Priority_TicketQuest = 2
        Config.Priority_GodSpirit = 3
        Config.Priority_TrainSkill = 4
        Config.Priority_NormalFarm = 5
    elseif pName:find("Cày Vé") then
        Config.Priority_TicketQuest = 1
        Config.Priority_SecretBoss = 2
        Config.Priority_GodSpirit = 3
        Config.Priority_TrainSkill = 4
        Config.Priority_NormalFarm = 5
    elseif pName:find("Luyện Chiêu") then
        Config.Priority_TrainSkill = 1
        Config.Priority_SecretBoss = 2
        Config.Priority_TicketQuest = 3
        Config.Priority_GodSpirit = 4
        Config.Priority_NormalFarm = 5
    end

    if dropSecretBoss and dropSecretBoss.Set then dropSecretBoss.Set(numberToRank(Config.Priority_SecretBoss), true) end
    if dropTicketQuest and dropTicketQuest.Set then dropTicketQuest.Set(numberToRank(Config.Priority_TicketQuest), true) end
    if dropGodSpirit and dropGodSpirit.Set then dropGodSpirit.Set(numberToRank(Config.Priority_GodSpirit), true) end
    if dropTrainSkill and dropTrainSkill.Set then dropTrainSkill.Set(numberToRank(Config.Priority_TrainSkill), true) end
    if dropNormalFarm and dropNormalFarm.Set then dropNormalFarm.Set(numberToRank(Config.Priority_NormalFarm), true) end

    PriorityManager.UpdateUI()
    if Config._triggerAutoSave then Config._triggerAutoSave() end

    isUpdatingPriorityUI = false
end

local dropPreset = createDropdownRow(priCard, "Mẫu Phân Cấp (Preset)", "Chọn nhanh bộ ưu tiên phổ biến hoặc tự do xếp hạng bên dưới", priorityPresetNames, Config.PriorityPreset or priorityPresetNames[1], function(v)
    applyPriorityPreset(v)
end)

local function onCustomRankChange()
    if isUpdatingPriorityUI then return end
    isUpdatingPriorityUI = true

    Config.PriorityPreset = "Tùy Biến Thứ Hạng (Custom)"
    if dropPreset and dropPreset.Set then
        dropPreset.Set("Tùy Biến Thứ Hạng (Custom)", true)
    end
    PriorityManager.UpdateUI()
    if Config._triggerAutoSave then Config._triggerAutoSave() end

    isUpdatingPriorityUI = false
end

dropSecretBoss = createDropdownRow(priCard, "Ưu Tiên: 🎯 Săn Secret Boss", "Xếp hạng ưu tiên cho Săn Boss (Chat Sniper & Thời Tiết)", rankOptions, numberToRank(Config.Priority_SecretBoss), function(v)
    Config.Priority_SecretBoss = rankToNumber(v)
    onCustomRankChange()
end)

dropTicketQuest = createDropdownRow(priCard, "Ưu Tiên: 📜 Làm Vé Nhiệm Vụ", "Xếp hạng ưu tiên cho Tự Động Làm Vé Nhiệm Vụ 20p", rankOptions, numberToRank(Config.Priority_TicketQuest), function(v)
    Config.Priority_TicketQuest = rankToNumber(v)
    onCustomRankChange()
end)

dropGodSpirit = createDropdownRow(priCard, "Ưu Tiên: ⛩️ Cúng Thần Linh", "Xếp hạng ưu tiên cho Tự Động Cầu Nguyện / Cúng Thần", rankOptions, numberToRank(Config.Priority_GodSpirit), function(v)
    Config.Priority_GodSpirit = rankToNumber(v)
    onCustomRankChange()
end)

dropTrainSkill = createDropdownRow(priCard, "Ưu Tiên: ⚔️ Auto Luyện Chiêu", "Xếp hạng ưu tiên cho Fast Cancel luyện cấp kỹ năng", rankOptions, numberToRank(Config.Priority_TrainSkill), function(v)
    Config.Priority_TrainSkill = rankToNumber(v)
    onCustomRankChange()
end)

dropNormalFarm = createDropdownRow(priCard, "Ưu Tiên: 🎣 Treo Farm Thường", "Xếp hạng ưu tiên cho Câu thường và Farm tại Home Spot", rankOptions, numberToRank(Config.Priority_NormalFarm), function(v)
    Config.Priority_NormalFarm = rankToNumber(v)
    onCustomRankChange()
end)

task.delay(1.5, function()
    pcall(PriorityManager.UpdateUI)
end)
end

do
createCategoryHeader(tabProfiles, "📢 Discord Webhook Báo Cáo Từ Xa")
local hookCard = createCardGroup(tabProfiles)

createInputRow(hookCard, "Webhook URL", "Dán URL Webhook từ máy chủ Discord của bạn vào đây", Config.WebhookUrl or "", function(txt)
    Config.WebhookUrl = txt
    SaveNotificationsConfig()
end)

createToggleRow(hookCard, "Bật Webhook", "Kích hoạt gửi thông báo về Discord", Config.WebhookEnabled, function(v)
    Config.WebhookEnabled = v
    SaveNotificationsConfig()
end)

createToggleRow(hookCard, "Thông Báo Bắt Được Boss", "Gửi tin nhắn khi câu trúng hoặc bắt thành công Boss / Cá Thần Thoại", Config.WebhookNotifyBoss, function(v)
    Config.WebhookNotifyBoss = v
    SaveNotificationsConfig()
end)

createToggleRow(hookCard, "Thông Báo Đạo Sĩ (Taoist & Maoshan)", "Gửi tin nhắn JobId server khi phát hiện Đạo Sĩ (Taoist) hoặc Mao Sơn (Maoshan)", Config.WebhookNotifyNPC, function(v)
    Config.WebhookNotifyNPC = v
    SaveNotificationsConfig()
end)

createToggleRow(hookCard, "Báo Cáo Định Kỳ", "Gửi bảng tổng kết thời gian treo máy, số cá, tiền, boss, thời tiết về Discord", Config.WebhookHourlyStats, function(v)
    Config.WebhookHourlyStats = v
    SaveNotificationsConfig()
end)

createSliderRow(hookCard, "Tần Suất Báo Cáo Định Kỳ", "Khoảng thời gian tự động gửi báo cáo tiến độ về Discord (phút)", 5, 120, Config.WebhookStatsInterval or 30, false, " Phút", function(v)
    Config.WebhookStatsInterval = v
    SaveNotificationsConfig()
end)

createToggleRow(hookCard, "Thông Báo Hoàn Thành Nhiệm Vụ Vé", "Gửi embed đầy đủ về Discord khi nộp vé Hard thành công (số lượt NV, vé, gems)", Config.WebhookNotifyTicketQuest == nil and true or Config.WebhookNotifyTicketQuest, function(v)
    Config.WebhookNotifyTicketQuest = v
    SaveNotificationsConfig()
end)

createButtonRow(hookCard, "Gửi Báo Cáo Toàn Diện Ngay", "Gửi embed đầy đủ: cá, tiền, gems, vé, essence, thời tiết, boss, ba lô, job ID về Discord ngay lập tức", "📊 Gửi Ngay", function()
    if not Config.WebhookUrl or Config.WebhookUrl == "" then
        ShowNotification("Webhook", "Vui lòng nhập Webhook URL trước!", "WARN")
        return
    end
    if not Config.WebhookEnabled then
        Config.WebhookEnabled = true
        if UIControllers.WebhookEnabled and UIControllers.WebhookEnabled.Set then
            pcall(function() UIControllers.WebhookEnabled.Set(true, true) end)
        end
        SaveNotificationsConfig()
    end
    ShowNotification("Webhook", "Đang tổng hợp báo cáo và gửi về Discord...", "INFO", 3)
    task.spawn(function()
        local ok = pcall(SendServerReportWebhook, "📊 BÁO CÁO TOÀN DIỆN (YÊU CẦU THỤ CÔNG)")
        if ok then
            ShowNotification("Webhook", "Đã gửi báo cáo toàn diện! Kiểm tra kênh Discord của bạn.", "SUCCESS", 5)
        else
            ShowNotification("Webhook", "Gửi thất bại! Kiểm tra lại URL Webhook.", "ERROR")
        end
    end)
end)

createButtonRow(hookCard, "Kiểm Tra Webhook (Test Nhanh)", "Gửi thử 1 tin nhắn kết nối đơn giản về Discord ngay lập tức", "Gửi Test", function()
    if not Config.WebhookUrl or Config.WebhookUrl == "" then
        ShowNotification("Webhook", "Vui lòng nhập Webhook URL trước!", "WARN")
        return
    end
    if not Config.WebhookEnabled then
        Config.WebhookEnabled = true
        SaveNotificationsConfig()
    end
    ShowNotification("Webhook", "Đang gửi tin nhắn test...", "INFO")
    task.spawn(function()
        local ok = pcall(function()
            SendDiscordWebhook(
                "🔔 Test Webhook - Câu Cá Pro " .. tostring(SCRIPT_BUILD_COMMIT),
                "✅ Kết nối Webhook thành công từ tài khoản: **" .. ((LocalPlayer and LocalPlayer.DisplayName) or "Unknown") .. "**!",
                3066993,
                {
                    { name = "👤 Tài Khoản", value = (LocalPlayer and LocalPlayer.Name or "Unknown"), inline = true },
                    { name = "🔑 Job ID", value = tostring(game.JobId or "N/A"), inline = true },
                    { name = "⏰ Thời Gian", value = os.date("%H:%M:%S - %d/%m/%Y"), inline = false }
                }
            )
        end)
        if ok then
            ShowNotification("Webhook", "Đã gửi test! Kiểm tra kênh Discord ngay.", "SUCCESS")
        else
            ShowNotification("Webhook", "Gửi thất bại! Kiểm tra lại URL Webhook.", "ERROR")
        end
    end)
end)


createCategoryHeader(tabProfiles, "📱 Telegram Bot (Thông Báo Về Điện Thoại)")
local teleCard = createCardGroup(tabProfiles)

createInputRow(teleCard, "Telegram Bot Token", "Nhập mã Token bot tạo từ @BotFather trên Telegram", Config.TelegramBotToken or "", function(txt)
    Config.TelegramBotToken = txt
    SaveNotificationsConfig()
end)

createInputRow(teleCard, "Telegram Chat ID", "Nhập mã Chat ID cuộc trò chuyện (lấy từ bot @userinfobot)", Config.TelegramChatId or "", function(txt)
    Config.TelegramChatId = txt
    SaveNotificationsConfig()
end)

createToggleRow(teleCard, "Bật Telegram Bot", "Kích hoạt gửi tin nhắn thông báo về ứng dụng Telegram trên điện thoại", Config.TelegramEnabled, function(v)
    Config.TelegramEnabled = v
    SaveNotificationsConfig()
end)

createToggleRow(teleCard, "Thông Báo Bắt Được Boss", "Gửi tin nhắn Telegram khi câu trúng hoặc bắt thành công Boss", Config.TelegramNotifyBoss, function(v)
    Config.TelegramNotifyBoss = v
    SaveNotificationsConfig()
end)

createToggleRow(teleCard, "Thông Báo Đạo Sĩ (Taoist & Maoshan)", "Gửi tin nhắn Telegram kèm JobId khi phát hiện Đạo Sĩ", Config.TelegramNotifyNPC, function(v)
    Config.TelegramNotifyNPC = v
    SaveNotificationsConfig()
end)

createButtonRow(teleCard, "Kiểm Tra Telegram (Test)", "Gửi thử 1 tin nhắn test đến Telegram của bạn ngay lập tức", "Gửi Test", function()
    if not Config.TelegramBotToken or Config.TelegramBotToken == "" or not Config.TelegramChatId or Config.TelegramChatId == "" then
        ShowNotification("Telegram", "Vui lòng nhập Bot Token và Chat ID trước!", "WARN")
        return
    end
    ShowNotification("Telegram", "Đang gửi tin nhắn test đến Telegram...", "INFO")
    task.spawn(function()
        local ok = pcall(function()
            local testMsg = "🔔 *TEST TELEGRAM - HEAVYWEIGHT FISHING*\n\n"
                .. "✅ Kết nối thành công từ tài khoản: *" .. (LocalPlayer and LocalPlayer.Name or "Unknown") .. "*!\n"
                .. "⏰ Thời gian: " .. os.date("%H:%M:%S - %d/%m/%Y")
            SendTelegramMessage(testMsg)
        end)
        if ok then
            ShowNotification("Telegram", "Đã gửi lệnh test đến Telegram!", "SUCCESS")
        else
            ShowNotification("Telegram", "Gửi thất bại! Kiểm tra lại Token & Chat ID.", "ERROR")
        end
    end)
end)

createCategoryHeader(tabProfiles, "🔔 ntfy Push (Thông Báo Thời Tiết & Server Về Điện Thoại)")
local ntfyCard = createCardGroup(tabProfiles)

createInputRow(ntfyCard, "ntfy Topic", "Nhập tên Topic đã đăng ký trên app ntfy (Ví dụ: my_weather_alert_88)", Config.NtfyTopic or "", function(txt)
    Config.NtfyTopic = txt
    SaveNotificationsConfig()
end)

createToggleRow(ntfyCard, "Bật ntfy Push", "Kích hoạt gửi thông báo đẩy đến điện thoại qua ntfy", Config.NtfyEnabled, function(v)
    Config.NtfyEnabled = v
    SaveNotificationsConfig()
end)

createToggleRow(ntfyCard, "Thông Báo Đổi Thời Tiết (ntfy)", "Gửi thông báo ngay khi thời tiết server đổi (Mưa, Bão, Sương Mù, Tuyết...)", Config.NtfyAlertWeatherChange, function(v)
    Config.NtfyAlertWeatherChange = v
    SaveNotificationsConfig()
end)

createToggleRow(ntfyCard, "Thông Báo Tìm Server Thời Tiết (ntfy)", "Gửi thông báo khi Weather Hop tìm được server có thời tiết mục tiêu", Config.NtfyAlertWeatherHop, function(v)
    Config.NtfyAlertWeatherHop = v
    SaveNotificationsConfig()
end)

createToggleRow(ntfyCard, "Thông Báo Boss & NPC (ntfy)", "Gửi thông báo khi phát hiện Secret Boss hoặc Đạo Sĩ (Taoist & Maoshan)", Config.NtfyNotifyBoss, function(v)
    Config.NtfyNotifyBoss = v
    SaveNotificationsConfig()
end)

createButtonRow(ntfyCard, "Kiểm Tra ntfy (Test)", "Gửi thử 1 thông báo đẩy mẫu về app ntfy trên điện thoại ngay lập tức", "Gửi Test", function()
    local clean = tostring(Config.NtfyTopic or ""):gsub("%s+", "")
    if clean == "" then
        ShowNotification("ntfy", "Vui lòng nhập ntfy Topic trước!", "WARN")
        return
    end
    -- Tự động bật ntfy nếu người dùng chưa bật để test được ngay
    if not Config.NtfyEnabled then
        Config.NtfyEnabled = true
        if UIControllers.NtfyEnabled and UIControllers.NtfyEnabled.Set then
            pcall(function() UIControllers.NtfyEnabled.Set(true, true) end)
        end
        SaveNotificationsConfig()
    end
    ShowNotification("ntfy", "Đang gửi thông báo test đến kênh: " .. clean .. "...", "INFO", 4)
    task.spawn(function()
        local _, curWeather = secretBossState.DetectWeather()
        local testWeather = (curWeather and curWeather ~= "" and curWeather ~= "Clear") and curWeather or "Clear (Trời Quang)"
        local testMsg = string.format("✅ Kết nối thành công!\n👤 Tài khoản: %s\n🌦️ Thời tiết server: %s\n🔑 JobId: %s\n⏰ %s",
            (LocalPlayer and (LocalPlayer.DisplayName or LocalPlayer.Name)) or "Unknown",
            testWeather,
            tostring(game.JobId or "N/A"),
            os.date("%H:%M:%S - %d/%m/%Y")
        )
        SendNtfyNotification("🔔 TEST NTFY - THỜI TIẾT SERVER", testMsg, 4, {"bell", "partly_sunny", "white_check_mark"})
        ShowNotification("ntfy", "Đã gửi thông báo test! Hãy kiểm tra điện thoại của bạn.", "SUCCESS", 5)
    end)
end)

createCategoryHeader(tabProfiles, "Tùy Chọn Khác")
local credCard = createCardGroup(tabProfiles)
createButtonRow(credCard, "🔴 Diệt Toàn Bộ Script (Kill Script)", "Ngắt kết nối mọi vòng lặp, xóa sạch giao diện và giải phóng bộ nhớ", "KILL SCRIPT", function()
    ShowNotification("Diệt Script", "Đang ngắt kết nối và đóng script hoàn toàn...", "WARN", 2)
    task.wait(0.2)
    UnloadScript()
end)
end

local function initExperimentalTab()
    -- 🎯 TỰ ĐỘNG TẨY LUYỆN TRAIT (AUTO REROLL & LOCK TRAIT)
    createCategoryHeader(tabExperimental, "🎯 TỰ ĐỘNG TẨY LUYỆN TRAIT (AUTO REROLL & LOCK TRAIT)")
    local traitCard = createCardGroup(tabExperimental)

    local traitList = {
        "Azure Dragon", "White Tiger", "Vermilion Bird", "Black Tortoise",
        "Assassin", "Berserk", "Chrono", "Executioner", "Powerful",
        "Precision", "Rapid", "Sharp", "Swift"
    }

    createDropdownRow(traitCard, "Chọn Trait Cần Săn", "Trait mục tiêu bot sẽ tự động roll cho đến khi trúng", traitList, Config.TargetTraitName, function(v)
        Config.TargetTraitName = v
    end)

    local infoTraitRerolls = createInfoRow(traitCard, "Vé Reroll Hiện Có", "Đang tải...")
    local function UpdateTraitRerollInfo()
        local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
        local count = pData and pData:FindFirstChild("Trait Reroll") and pData["Trait Reroll"].Value or 0
        infoTraitRerolls.Set(string.format("%d Vé", count))
    end
    task.spawn(UpdateTraitRerollInfo)

    local isAutoRerolling = false
    local function RunAutoRerollTrait()
        if isAutoRerolling then return end
        isAutoRerolling = true
        task.spawn(function()
            local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
            ShowNotification("Reroll Trait", "Bắt đầu tự động Reroll săn Trait: " .. tostring(Config.TargetTraitName), "INFO", 4)

            while isAutoRerolling and Config.AutoRerollTrait do
                local currentRerolls = pData and pData:FindFirstChild("Trait Reroll") and pData["Trait Reroll"].Value or 0
                UpdateTraitRerollInfo()

                if currentRerolls <= 0 then
                    ShowNotification("Hết Vé", "Đã hết vé Reroll Trait!", "WARN", 5)
                    Config.AutoRerollTrait = false
                    isAutoRerolling = false
                    break
                end

                if Events and Events:FindFirstChild("RerollTrait") then
                    local res = Events.RerollTrait:InvokeServer()
                    local targetLower = Config.TargetTraitName:lower()
                    local isHit = false

                    if type(res) == "string" and res:lower():find(targetLower, 1, true) then
                        isHit = true
                    end

                    if not isHit and pData and pData:FindFirstChild("LockTrait") then
                        local tVal = pData.LockTrait:FindFirstChild(Config.TargetTraitName)
                        if tVal and tVal.Value == true then
                            isHit = true
                        end
                    end

                    if isHit then
                        ShowNotification("TRÚNG TRAIT!", string.format("Đã roll trúng [%s]! Tự động khóa bảo vệ ngay lập tức.", Config.TargetTraitName), "SUCCESS", 8)
                        if Events:FindFirstChild("LockTrait") then
                            Events.LockTrait:FireServer(Config.TargetTraitName)
                        end
                        Config.AutoRerollTrait = false
                        isAutoRerolling = false
                        break
                    end
                end
                task.wait(0.35)
            end
            isAutoRerolling = false
            UpdateTraitRerollInfo()
        end)
    end

    createToggleRow(traitCard, "Tự Động Reroll Đến Khi Trúng", "Tự động roll liên tục và khóa lại khi ra đúng Trait mục tiêu", Config.AutoRerollTrait, function(v)
        Config.AutoRerollTrait = v
        if v then
            RunAutoRerollTrait()
        else
            isAutoRerolling = false
        end
    end)

    createButtonRow(traitCard, "Reroll 1 Lần Thủ Công", "Thực hiện roll trait 1 lần ngay lập tức", "Reroll 1 Lần", function()
        if Events and Events:FindFirstChild("RerollTrait") then
            local res = Events.RerollTrait:InvokeServer()
            UpdateTraitRerollInfo()
            ShowNotification("Reroll Trait", "Kết quả roll: " .. tostring(res or "Đã roll thành công"), "INFO", 4)
        end
    end)

    createButtonRow(traitCard, "Khóa / Mở Khóa Trait Đang Chọn", "Chuyển đổi trạng thái khóa bảo vệ cho Trait đang chọn", "Khóa / Mở", function()
        if Events and Events:FindFirstChild("LockTrait") then
            Events.LockTrait:FireServer(Config.TargetTraitName)
            ShowNotification("Khóa Trait", "Đã gửi lệnh đổi trạng thái khóa cho: " .. tostring(Config.TargetTraitName), "SUCCESS", 3)
        end
    end)
end
initExperimentalTab()

-- ĐỒNG BỘ TOÀN BỘ GIÁ TRỊ TỪ CONFIG VÀO GIAO DIỆN GUI (ĐẢM BẢO CONFIG = GIAO DIỆN 100%)
task.spawn(function()
    task.wait(0.2)
    for key, ctrl in pairs(UIControllers) do
        if Config[key] ~= nil and ctrl and ctrl.Set then
            pcall(function()
                ctrl.Set(Config[key], true)
            end)
        end
    end
end)

local lastCastTime = 0
local lastSellTime = 0
local lastSkillTime = 0
local isTrainingBusy = false

function comboState.GetSkillButton(cleanKey, fUI)
    if not cleanKey then return nil end
    cleanKey = cleanKey:upper()
    if not fUI then
        local pg = LocalPlayer:FindFirstChild("PlayerGui")
        local mg = pg and pg:FindFirstChild("MainGui")
        fUI = mg and mg:FindFirstChild("Fishing")
    end
    if not fUI then return nil end
    local sb = fUI:FindFirstChild("SkillButton")
    local fr = sb and sb:FindFirstChild("Frame")
    if fr and fr:FindFirstChild(cleanKey) then
        return fr[cleanKey]
    end
    for _, d in ipairs(fUI:GetDescendants()) do
        if d:IsA("GuiButton") and (d.Name:upper() == cleanKey or (d.Name:upper():find("SKILL") and d.Name:upper():find(cleanKey))) then
            return d
        end
    end
    return nil
end

function comboState.IsSkillOnCooldown(sk, fUI)
    if not sk or sk == "" or sk == "Tắt" then return false end
    local cleanKey = sk:match("([ZXCVzxcv])") or sk
    cleanKey = cleanKey:upper()

    local btn = comboState.GetSkillButton(cleanKey, fUI)
    if not btn then
        return false
    end

    -- 1. Kiểm tra Attribute OnCooldown hoặc CD
    if btn:GetAttribute("OnCooldown") == true or btn:GetAttribute("CD") == true then
        return true
    end

    -- 2. Kiểm tra nhãn đếm giây Cooldown trực tiếp trong nút
    for _, child in ipairs(btn:GetDescendants()) do
        if child:IsA("TextLabel") and child.Visible and child.Text ~= "" then
            local txt = child.Text:gsub("%s+", "")
            local cdMatch = txt:match("(%d+%.?%d*)%s*[sS]") or txt:match("(%d+%.%d+)")
            if cdMatch then
                local num = tonumber(cdMatch)
                if num and num > 0.05 then
                    return true
                end
            end
            local cName = child.Name:lower()
            if cName:find("cd") or cName:find("cooldown") or cName:find("timer") then
                local num = tonumber(txt:match("(%d+%.?%d*)"))
                if num and num > 0.05 then
                    return true
                end
            end
        elseif child:IsA("Frame") and child.Visible then
            local cName = child.Name:lower()
            if cName == "cooldown" or cName == "cd" or cName:find("cooldownframe") then
                return true
            end
        end
    end

    return false
end

function comboState.IsSkillReady(sk, fUI)
    return not comboState.IsSkillOnCooldown(sk, fUI)
end

function comboState.CheckSkillReady(sk, fUI, minCooldown)
    if minCooldown and (tick() - (comboState.usedTimes[sk:upper()] or 0) < minCooldown) then
        return false
    end
    return comboState.IsSkillReady(sk, fUI)
end

local function GetFishHealth(fUI)
    -- 1. Kiểm tra Workspace.Fishes
    local fishID = LocalPlayer:GetAttribute("FishID")
    if fishID and Workspace:FindFirstChild("Fishes") then
        local f = Workspace.Fishes:FindFirstChild(fishID)
        if f then
            local hpVal = f:FindFirstChild("FishHealth") or f:FindFirstChild("Health")
            if hpVal and (hpVal:IsA("NumberValue") or hpVal:IsA("IntValue")) and hpVal.Value > 0 then
                return hpVal.Value
            end
            for _, child in ipairs(f:GetChildren()) do
                if (child.Name:find("FishHealth") or child.Name:find("Health")) and not child.Name:find("Player") then
                    if (child:IsA("NumberValue") or child:IsA("IntValue")) and child.Value > 0 then
                        return child.Value
                    end
                end
            end
        end
    end

    -- 2. Kiểm tra Attribute trên Character (TUYỆT ĐỐI KHÔNG KIỂM TRA "HP" VÌ ĐÓ LÀ MÁU NGƯỜI CHƠI)
    local char = LocalPlayer.Character
    if char then
        for _, att in ipairs({"FishHealth", "FishHP", "TargetHealth", "BossHP", "TargetHP"}) do
            local val = char:GetAttribute(att)
            if val and tonumber(val) and tonumber(val) > 0 then
                return tonumber(val)
            end
        end
    end

    -- 3. Quét trong fUI (PlayerGui.MainGui.Fishing)
    if fUI then
        -- Kiểm tra BossFightBar (Thanh máu Boss chính thức)
        local bossBar = fUI:FindFirstChild("BossFightBar")
        if bossBar and bossBar.Visible and bossBar.Size.X.Scale > 0 then
            for _, d in ipairs(bossBar:GetDescendants()) do
                if d:IsA("TextLabel") and d.Visible and d.Text ~= "" then
                    local txt = d.Text
                    local kMatch = txt:match("([%d%.]+)%s*[kK]")
                    if kMatch and tonumber(kMatch) then return tonumber(kMatch) * 1000 end
                    local mMatch = txt:match("([%d%.]+)%s*[mM]")
                    if mMatch and tonumber(mMatch) then return tonumber(mMatch) * 1000000 end

                    local slashMatch = txt:match("([%d,%.]+)%s*/")
                    if slashMatch then
                        local n = tonumber(slashMatch:gsub("[,]", ""))
                        if n and n > 0 then return n end
                    end

                    local numOnly = txt:match("^%s*([%d,%.]+)%s*$")
                    if numOnly then
                        local n = tonumber(numOnly:gsub("[,]", ""))
                        if n and n > 0 and n ~= 100 then return n end
                    end
                end
            end
            -- Nếu BossFightBar đang hiện nhưng chưa kịp đọc số -> Coi như máu to (> threshold)
            return 999999
        end

        -- Quét các TextLabel hiển thị máu cá khác, LOẠI BỎ TRIỆT ĐỂ HPPlayer và mọi khung của người chơi
        for _, d in ipairs(fUI:GetDescendants()) do
            if d:IsA("TextLabel") and d.Visible and d.Text ~= "" then
                local isPlayer = false
                local p = d
                while p and p ~= fUI do
                    local pName = p.Name:lower()
                    if pName:find("player") or pName == "hpplayer" then
                        isPlayer = true
                        break
                    end
                    p = p.Parent
                end

                if not isPlayer then
                    local dName = d.Name:lower()
                    local txt = d.Text
                    local isFishContext = dName:find("fish") or dName:find("target") or dName:find("boss") or dName:find("enemy")

                    if isFishContext then
                        local kMatch = txt:match("([%d%.]+)%s*[kK]")
                        if kMatch and tonumber(kMatch) then return tonumber(kMatch) * 1000 end
                        local mMatch = txt:match("([%d%.]+)%s*[mM]")
                        if mMatch and tonumber(mMatch) then return tonumber(mMatch) * 1000000 end

                        local slashMatch = txt:match("([%d,%.]+)%s*/")
                        if slashMatch then
                            local n = tonumber(slashMatch:gsub("[,]", ""))
                            if n and n > 0 then return n end
                        end

                        local hpSuffixMatch = txt:match("([%d,%.]+)%s*[hH][pP]") or txt:match("[hH][pP]%s*:?%s*([%d,%.]+)")
                        if hpSuffixMatch then
                            local n = tonumber(hpSuffixMatch:gsub("[,]", ""))
                            if n and n > 0 then return n end
                        end

                        local numOnly = txt:match("^%s*([%d,%.]+)%s*$")
                        if numOnly then
                            local n = tonumber(numOnly:gsub("[,]", ""))
                            if n and n > 0 and n ~= 100 then return n end
                        end
                    end
                end
            end
        end
    end

    -- Nếu không có BossFightBar và không có nhãn máu đặc biệt -> Đây là cá thông thường (10 HP)
    return 10
end

local function GetPlayerHealth(fUI)
    local char = LocalPlayer.Character
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    if hum and hum.MaxHealth > 0 then
        return (hum.Health / hum.MaxHealth) * 100
    end

    if fUI and fUI:FindFirstChild("HPPlayer") then
        local hpP = fUI.HPPlayer
        local pBar = hpP:FindFirstChild("ProgressionBar")
        if pBar and pBar:FindFirstChild("Bar") then
            return pBar.Bar.Size.X.Scale * 100
        end
        for _, d in ipairs(hpP:GetDescendants()) do
            if d:IsA("TextLabel") and d.Visible and d.Text ~= "" then
                local cur, max = d.Text:match("(%d+)%s*/%s*(%d+)")
                if cur and max and tonumber(max) > 0 then
                    return (tonumber(cur) / tonumber(max)) * 100
                end
            end
        end
    end

    return 100
end

function comboState.IsCharacterCastingSkill()
    local now = tick()
    -- Chỉ coi là animation skill trong tối đa 0.7s kể từ khi tung chiêu, tránh bị kẹt vĩnh viễn
    if (now - comboState.lastCastTime > 0.35) then
        return false
    end

    local char = LocalPlayer.Character
    if not char then return false end

    -- Bỏ qua "Casting" vì đó là trạng thái quăng cần câu của game
    for _, att in ipairs({"UsingSkill", "SkillActive", "IsAttacking", "CastingSkill"}) do
        if char:GetAttribute(att) == true then
            return true
        end
    end

    local hum = char:FindFirstChildOfClass("Humanoid")
    local anim = hum and hum:FindFirstChildOfClass("Animator")
    if anim then
        local ok, tracks = pcall(function() return anim:GetPlayingAnimationTracks() end)
        if ok and tracks then
            for _, tr in ipairs(tracks) do
                if tr.IsPlaying and (tr.Priority == Enum.AnimationPriority.Action or tr.Priority == Enum.AnimationPriority.Action2 or tr.Priority == Enum.AnimationPriority.Action3 or tr.Priority == Enum.AnimationPriority.Action4) then
                    local animName = tr.Name:lower()
                    -- Loại bỏ các animation câu cá / quăng cần / cuộn dây / chạy bộ (tránh nhận nhầm làm kẹt combo)
                    if not animName:find("fish") and not animName:find("rod") and not animName:find("reel") and not animName:find("cast") and not animName:find("idle") and not animName:find("hold") and not animName:find("walk") and not animName:find("run") then
                        if animName:find("skill") or animName:find("attack") or animName:find("special") or animName:find("slash") then
                            return true
                        end
                    end
                end
            end
        end
    end

    return false
end

function comboState.CastSkill(sk)
    if not sk or sk == "" or sk == "Tắt" then return false end
    local cleanKey = sk:match("([ZXCVzxcv])") or sk
    cleanKey = cleanKey:upper()

    -- 🛑 KHÓA BẢO VỆ CẤP CAO CHO CHIÊU C: Nếu không có tab nào chọn C, cấm 100% việc cast chiêu C!
    if cleanKey == "C" then
        local userChoseC = false
        if Config.TicketQuickSkill and (Config.TicketQuickSkill:find("C") or Config.TicketQuickSkill:find("c")) then userChoseC = true end
        if Config.TicketSkillKey and (Config.TicketSkillKey:match("[Cc]%s*$")) then userChoseC = true end
        if Config.TrainSkill and (Config.TrainSkill:find("C") or Config.TrainSkill:find("c")) then userChoseC = true end
        if Config.QuickCatchSkill == "C" or Config.OpenerSkill == "C" or Config.EmergencyHealSkill == "C" then userChoseC = true end
        -- Cho phép C nếu người chơi cài đặt C trong LoopSkills và đang bật SmartCombo hoặc AutoSkills
        if (Config.SmartComboEnabled or Config.AutoSkills) and Config.LoopSkills and (Config.LoopSkills:find("C") or Config.LoopSkills:find("c")) then
            userChoseC = true
        end
        if not userChoseC then
            return false -- Chặn đứng 100% việc cast C
        end
    end

    -- KHÓA BẢO VỆ CHO AUTO TICKET QUEST: Chỉ khóa 1 chiêu khi đang BẬT Auto Ticket Quest và đúng quest 100 cá & 100 skill
    if Config.AutoTicketQuest then
        local curQ = ticketQuestState and ticketQuestState.currentQuestType or "none"

        if curQ == "skill_100" then
            local allowedSkill = Config.TicketSkillKey and Config.TicketSkillKey:match("([ZXCVzxcv])%s*$")
            allowedSkill = allowedSkill and allowedSkill:upper() or "Z"
            if cleanKey ~= allowedSkill then return false end
        elseif curQ == "fish_100" then
            local allowedQuick = Config.TicketQuickSkill and Config.TicketQuickSkill:match("([ZXCVzxcv])%s*$")
            allowedQuick = allowedQuick and allowedQuick:upper() or "V"
            if cleanKey ~= allowedQuick then return false end
        end
    end

    -- 1. Gửi RemoteEvent tới Server
    pcall(function()
        if Events then
            if Events:FindFirstChild("UseSkill") then
                Events.UseSkill:FireServer(cleanKey)
            end
            if Events:FindFirstChild("TriggerMinigameSkill") then
                Events.TriggerMinigameSkill:FireServer(cleanKey)
            end
        end
    end)

    -- 2. Giả lập phím bấm bàn phím qua VIM
    pcall(function()
        local vim = game:GetService("VirtualInputManager")
        if vim and Enum.KeyCode[cleanKey] then
            vim:SendKeyEvent(true, Enum.KeyCode[cleanKey], false, game)
            task.wait(0.02)
            vim:SendKeyEvent(false, Enum.KeyCode[cleanKey], false, game)
        end
    end)

    -- 3. Giả lập click trực tiếp nút chiêu trên GUI Fishing (PlayerGui.MainGui.Fishing.SkillButton.Frame[sk] hoặc tương đương)
    pcall(function()
        local function clickBtn(btn)
            if not btn or not btn:IsA("GuiButton") or not btn.Visible then return false end
            if firesignal then
                if btn.Activated then firesignal(btn.Activated) end
                if btn.MouseButton1Click then firesignal(btn.MouseButton1Click) end
            end
            if getconnections then
                for _, c in ipairs(getconnections(btn.Activated)) do c:Fire() end
                for _, c in ipairs(getconnections(btn.MouseButton1Click)) do c:Fire() end
            end
            return true
        end

        local pg = LocalPlayer:FindFirstChild("PlayerGui")
        local mg = pg and pg:FindFirstChild("MainGui")
        local f = mg and mg:FindFirstChild("Fishing")
        if f then
            local sb = f:FindFirstChild("SkillButton")
            local fr = sb and sb:FindFirstChild("Frame")
            local btn = fr and fr:FindFirstChild(cleanKey)
            if btn and clickBtn(btn) then
                return
            end
            for _, d in ipairs(f:GetDescendants()) do
                if d:IsA("GuiButton") and (d.Name:upper() == cleanKey or (d.Name:upper():find("SKILL") and d.Name:upper():find(cleanKey))) then
                    clickBtn(d)
                end
            end
        end
    end)

    comboState.usedTimes[cleanKey] = tick()
    comboState.lastCastTime = tick()
    return true
end
local lastGachaTime = 0
local lastBaitBuyTime = 0
local lastQuestTime = 0
local lastGodPrayTime = 0
local lastEquipTime = 0
local lastEquipRodTime = 0
local lastProgressionTime = 0
local lastProtectTime = 0
local fishingStartTime = 0
local wasFishing = false
local minigameDurationTracker = 0
local wasMinigame = false



local function ProtectInventoryItem(item, showNotify)
    if not item or Wiki.IsItemFavorited(item) then return false end
    local rawName = Wiki.GetItemRawName(item)
    local rawLower = rawName:lower()

    if Wiki.temporarilyUnlockedBaitFish and Wiki.temporarilyUnlockedBaitFish[rawLower] then
        return false
    end

    local itemName = tostring(item.Name or "")
    local shouldProtect = false
    local reason = ""

    if Config.AutoProtectMutations and Wiki.IsMutatedFish(item) then
        shouldProtect = true
        reason = "Cá Đột Biến"
    elseif Wiki.IsSecretBossFish(item) then
        shouldProtect = true
        reason = "Cá Secret Boss"
    elseif Config.MaterialFarming and (Wiki.craftMaterialFish[rawName] or Wiki.craftMaterialFish[itemName]) then
        shouldProtect = true
        reason = "Nguyên Liệu Chế Tạo"
    elseif Config.AutoFavouriteFish and (rawName == Config.FavouriteFishName or itemName == Config.FavouriteFishName) then
        shouldProtect = true
        reason = "Cá Quý Chỉ Định"
    elseif Wiki.IsEssentialKeepItem(item) then
        shouldProtect = true
        reason = "Cá Cần Giữ"
    end

    if shouldProtect and Events and Events:FindFirstChild("FavoriteItem") then
        Events.FavoriteItem:FireServer(item)
        if showNotify then
            ShowNotification("Khóa Cá", string.format("Đã tự động KHÓA bảo vệ [%s] (%s)!", rawName, reason), "SUCCESS", 5)
        end
        return true
    end
    return false
end

local function ProtectAllInventoryItems(showNotify)
    local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
    if pData and pData:FindFirstChild("Inventory") then
        for _, item in ipairs(pData.Inventory:GetChildren()) do
            ProtectInventoryItem(item, showNotify)
        end
    end
end

-- Lắng nghe khi nhận được cá mới vào balo -> Khóa ngay lập tức nếu là Secret Boss
task.spawn(function()
    local pDataInit = ReplicatedStorage:WaitForChild("Data", 10)
    local userFolder = pDataInit and pDataInit:WaitForChild(tostring(LocalPlayer.UserId), 10)
    local invFolder = userFolder and userFolder:WaitForChild("Inventory", 10)
    if invFolder then
        table.insert(activeConnections, invFolder.ChildAdded:Connect(function(child)
            task.wait(0.3)
            ProtectInventoryItem(child, true)
        end))
    end
end)

local waterPlatform = Instance.new("Part")
waterPlatform.Name = "IdenticalWaterPlatform"
waterPlatform.Size = Vector3.new(60, 2, 60)
waterPlatform.Anchored = true
waterPlatform.CanCollide = false
waterPlatform.CanQuery = false
waterPlatform.CanTouch = false
waterPlatform.Transparency = 1
waterPlatform.Parent = Workspace
table.insert(cleanUpInstances, waterPlatform)

table.insert(activeConnections, RunService.Heartbeat:Connect(function(dt)
    if not isRunning then return end
    pcall(function()
        local char = LocalPlayer.Character
        local root = char and char:FindFirstChild("HumanoidRootPart")
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if not root or not hum then return end
        local now = tick()
        local pg = LocalPlayer:FindFirstChild("PlayerGui")
        local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)

        if Config.WalkSpeedEnabled then
            hum.WalkSpeed = Config.WalkSpeedValue
        end

        if Config.WalkOnWater or Config.AcidWaterShield then
            local yLevel = 0
            if Config.AcidWaterShield then
                yLevel = 3.5
            end
            waterPlatform.CFrame = CFrame.new(root.Position.X, yLevel, root.Position.Z)
            waterPlatform.CanCollide = (root.Position.Y >= yLevel - 2)
        else
            waterPlatform.CanCollide = false
        end

        if Config.Noclip then
            for _, part in ipairs(char:GetDescendants()) do
                if part:IsA("BasePart") and part.CanCollide then
                    part.CanCollide = false
                end
            end
        end

        local isFishing = char:GetAttribute("Fishing") == true
        local isMinigame = char:GetAttribute("Minigame") == true or (pg and pg:FindFirstChild("MainGui") and pg.MainGui:FindFirstChild("Fishing") and pg.MainGui.Fishing.Visible)
        local isCD = char:GetAttribute("CDForTheNextThrow") == true
        local isSwimming = char:GetAttribute("Swimming") == true

        local isBossActive = secretBossState and secretBossState.active
        local isTicketActive = ticketQuestState and ticketQuestState.IsFishingActive and ticketQuestState.IsFishingActive()
        local isTicketBusy = ticketQuestState and ticketQuestState.IsBusyOrInteracting and ticketQuestState.IsBusyOrInteracting()
        local shouldAutoFish = not isTicketBusy and (Config.AutoCast or Config.AutoTrainSkill or isTicketActive or ((Config.AutoHuntBoss or Config.AutoChatSecretBoss) and isBossActive))

        -- Nếu đang bận nộp/nhận vé hoặc mở thoại, ép unequip cần câu ngay lập tức để không bị quăng cần/lag
        if isTicketBusy and char:GetAttribute("Type") == "Fishing Rod" then
            hum:UnequipTools()
            if Events and Events:FindFirstChild("CancelCast") then Events.CancelCast:FireServer() end
        end

        if shouldAutoFish and char:GetAttribute("Type") ~= "Fishing Rod" and (now - lastEquipRodTime >= 1.0) and not isTrainingBusy then
            lastEquipRodTime = now
            local rodSlot = "1"
            if pData and pData:FindFirstChild("Hotbar") then
                for _, item in ipairs(pData.Hotbar:GetChildren()) do
                    local vName = item:FindFirstChild("ValueName")
                    if vName and tostring(vName.Value):lower():find("rod") and not tostring(vName.Value):lower():find("inventory") then
                        rodSlot = item.Name
                        break
                    end
                end
            end
            if Events and Events:FindFirstChild("ToggleHotbar") then
                task.spawn(function()
                    Events.ToggleHotbar:InvokeServer(rodSlot)
                end)
            end
        end

        if isFishing and not wasFishing then
            fishingStartTime = now
        elseif not isFishing then
            fishingStartTime = 0
        end
        wasFishing = isFishing

        if isMinigame and not wasMinigame then
            minigameDurationTracker = now
            comboState.openerUsedCount = 0
            comboState.openerDone = false
            comboState.loopTargetIndex = 1
            comboState.loopIndex = 1
            comboState.lastActionTime = 0
            comboState.loopWaitStartTime = 0
            comboState.minigameStartTime = now
            comboState.usedTimes = {}; comboState.lastCastingKey = nil
            secretBossState.webhookSentForCurrent = false
        elseif not isMinigame then
            minigameDurationTracker = 0
            comboState.lastActionTime = 0
            comboState.loopWaitStartTime = 0
            comboState.loopTargetIndex = 1
            comboState.usedTimes = {}
            secretBossState.webhookSentForCurrent = false
            secretBossState.isCatchingTarget = false
            secretBossState.minigameStartTime = 0
        end
        wasMinigame = isMinigame

        if isFishing and not isMinigame and fishingStartTime > 0 and (now - fishingStartTime >= 15.0) then
            fishingStartTime = now
            if Config.AntiStuckEnabled then
                CancelAndRecastRod()
            elseif Events and Events:FindFirstChild("Fishing") then
                Events.Fishing:FireServer(root.CFrame)
            end
        end

        if isMinigame and Config.AntiStuckEnabled and not Config.AutoTrainSkill and minigameDurationTracker > 0 and (now - minigameDurationTracker >= 22.0) then
            minigameDurationTracker = now
            CancelAndRecastRod()
        end

        if isMinigame then
            local fUI = pg and pg:FindFirstChild("MainGui") and pg.MainGui:FindFirstChild("Fishing")
            
            -- XỬ LÝ FAST SKIP KHI SĂN SECRET BOSS (NẾU KHÔNG PHẢI BOSS MỤC TIÊU THÌ GIẬT CẦN THẢ LẠI)
            local skipTriggered = false
            local isHunting = (Config.AutoHuntBoss or Config.AutoChatSecretBoss) and secretBossState.active

            if isHunting and Config.FastSkipNonBoss and not Config.AutoTrainSkill and not isTicketActive then
                if secretBossState.minigameStartTime == 0 then
                    secretBossState.minigameStartTime = now
                end

                local hookedFish = GetCurrentHookedFishName()
                local curFishHp = GetFishHealth(fUI)
                local isTargetBoss = false
                local bossDisplay = nil

                if hookedFish then
                    local cleanHooked = hookedFish:gsub("%s+", ""):lower()
                    local matchedBoss = secretBossLookup[hookedFish:lower()] or secretBossLookup[cleanHooked]
                    if Config.SecretBossTargets[hookedFish] == true
                        or (matchedBoss and (Config.SecretBossTargets[matchedBoss] == true or Config.SecretBossTargets[matchedBoss:gsub("Heavenpiercer", "Heaven Piercer")] == true or Config.SecretBossTargets[matchedBoss:gsub("Heaven Piercer", "Heavenpiercer")] == true))
                        or (cleanHooked:find("heaven") and cleanHooked:find("turtle") and (Config.SecretBossTargets["Heavenpiercer Turtle"] == true or Config.SecretBossTargets["Heaven Piercer Turtle"] == true)) then
                        isTargetBoss = true
                        bossDisplay = hookedFish
                    end
                end

                if not isTargetBoss and fUI then
                    for _, bName in ipairs({"BossFightBar", "BossBar", "BossUI", "BossFrame", "BossProgress", "BossHealth"}) do
                        local bBar = fUI:FindFirstChild(bName, true)
                        if bBar and bBar.Visible then
                            isTargetBoss = true
                            bossDisplay = hookedFish or "Secret Boss"
                            break
                        end
                    end
                end

                if isTargetBoss then
                    secretBossState.isCatchingTarget = true
                    local displayBossName = bossDisplay or hookedFish or "Secret Boss"
                    if statusLabelSecretBoss and statusLabelSecretBoss.Set then
                        statusLabelSecretBoss.Set("🎯 ĐANG CÂU BOSS: " .. tostring(displayBossName) .. "!")
                    end
                    if (Config.WebhookEnabled and Config.WebhookNotifyBoss) or (Config.TelegramEnabled and Config.TelegramNotifyBoss) then
                        if not secretBossState.webhookSentForCurrent then
                            secretBossState.webhookSentForCurrent = true
                            if Config.WebhookEnabled and Config.WebhookNotifyBoss then
                                SendDiscordWebhook(
                                    "🚨 PHÁT HIỆN SECRET BOSS!",
                                    "Tài khoản **" .. LocalPlayer.Name .. "** đang câu trúng: **" .. tostring(displayBossName) .. "** tại " .. (secretBossState.currentMap or "Đảo hiện tại") .. "!",
                                    15158332,
                                    {
                                        { name = "🐟 Boss Mục Tiêu", value = tostring(displayBossName), inline = true },
                                        { name = "📍 Bản Đồ", value = tostring(secretBossState.currentMap or "Đảo Hiện Tại"), inline = true },
                                        { name = "⏰ Thời Gian", value = os.date("%H:%M:%S - %d/%m/%Y"), inline = true }
                                    }
                                )
                            end
                            if Config.TelegramEnabled and Config.TelegramNotifyBoss then
                                local teleBoss = "🚨 *PHÁT HIỆN SECRET BOSS!*\n\n"
                                    .. "👤 *Tài khoản:* " .. LocalPlayer.Name .. "\n"
                                    .. "🐟 *Boss câu trúng:* *" .. tostring(displayBossName) .. "*\n"
                                    .. "📍 *Vị trí:* " .. tostring(secretBossState.currentMap or "Đảo Hiện Tại") .. "\n"
                                    .. "⏰ *Thời gian:* " .. os.date("%H:%M:%S - %d/%m/%Y")
                                SendTelegramMessage(teleBoss)
                            end
                            if Config.NtfyEnabled and Config.NtfyNotifyBoss then
                                local ntfyBossMsg = string.format("Tài khoản %s đang câu trúng: %s tại %s!\n⏰ %s",
                                    LocalPlayer.DisplayName or LocalPlayer.Name, tostring(displayBossName), tostring(secretBossState.currentMap or "Đảo Hiện Tại"), os.date("%H:%M:%S - %d/%m/%Y"))
                                SendNtfyNotification("🚨 CÂU TRÚNG SECRET BOSS: " .. tostring(displayBossName):upper(), ntfyBossMsg, 5, {"trophy", "fire", "fishing_pole_and_fish"})
                            end
                        end
                    end
                else
                    -- Không phải Secret Boss mục tiêu -> Fast Skip giật cần bỏ cá thường
                    local timeInMinigame = now - secretBossState.minigameStartTime
                    local canSkipNow = (now - secretBossState.lastSkipTime >= 0.8)

                    if canSkipNow and ((hookedFish and timeInMinigame >= 0.25) or (timeInMinigame >= 0.7)) then
                        secretBossState.lastSkipTime = now
                        secretBossState.minigameStartTime = 0
                        skipTriggered = true
                        local skipFishName = hookedFish or "Cá thường"
                        if statusLabelSecretBoss and statusLabelSecretBoss.Set then
                            statusLabelSecretBoss.Set("Bỏ qua [" .. skipFishName .. (curFishHp and (" - " .. tostring(curFishHp) .. " HP") or "") .. "], đang giật cần thả lại...")
                        end
                        CancelAndRecastRod()
                    end
                end
            else
                secretBossState.minigameStartTime = 0
            end

            if not skipTriggered then
                -- Đồng bộ trạng thái Quest Ticket từ hệ thống nếu chưa có (Chỉ khi BẬT AutoTicketQuest)
                if Config.AutoTicketQuest and ticketQuestState and (not ticketQuestState.currentQuestType or ticketQuestState.currentQuestType == "none") and ticketQuestState.DetectActiveQuest then
                    local detQ = select(1, ticketQuestState.DetectActiveQuest())
                    if detQ and detQ ~= "none" then
                        ticketQuestState.currentQuestType = detQ
                        ticketQuestState.active = true
                        ticketQuestState.isCooldown = false
                    end
                end

                local hasActiveTicket = Config.AutoTicketQuest and ticketQuestState and ticketQuestState.currentQuestType and ticketQuestState.currentQuestType ~= "none" and not ticketQuestState.isCompleted

                local isTrainActive = Config.AutoTrainSkill
                if isTrainActive and Config.PrioritySystemEnabled and PriorityManager and PriorityManager.GetActiveTask then
                    isTrainActive = (PriorityManager.GetActiveTask() == "TrainSkill")
                end

                if isTrainActive then
                    if not isTrainingBusy then
                        isTrainingBusy = true
                        task.spawn(function()
                            local chosenSkill = Config.TrainSkill or "Z"
                            local cleanKey = chosenSkill:match("([ZXCVzxcv])") or chosenSkill
                            cleanKey = cleanKey:upper()

                            local initialFishHp = GetFishHealth(fUI)
                            local startTime = tick()

                            -- 1. GIỮ THĂNG BẰNG THANH BAR VÀ CHỜ QUA 3 GIÂY KHÓA CHIÊU CỦA GAME (BẮN SKILL LIÊN TỤC ĐỂ BẮT ĐÚNG NHỊP MỞ)
                            while isRunning and (fUI and fUI.Visible) do
                                -- Giữ thăng bằng thanh bar ở giữa để cá không bao giờ bị tuột
                                local barFrame = fUI:FindFirstChild("BarFrame")
                                if barFrame and barFrame:FindFirstChild("Bar") then
                                    barFrame.Bar.Position = UDim2.new(0.5, 0, 0.5, 0)
                                end

                                -- Bắn skill liên tục để kích hoạt ngay khoảnh khắc game mở khóa
                                comboState.CastSkill(cleanKey)

                                task.wait(0.08)

                                local elapsed = tick() - startTime

                                -- TUYỆT ĐỐI KHÔNG ĐƯỢC THOÁT TRƯỚC 3 GIÂY VÌ GAME ĐANG KHÓA CHIÊU
                                if elapsed >= 3.05 then
                                    local curHp = GetFishHealth(fUI)
                                    local hpDropped = (initialFishHp and curHp and curHp < initialFishHp)
                                    local nowOnCd = comboState.IsSkillOnCooldown(cleanKey, fUI)

                                    -- Sau khi qua 3s: nếu đã tung chiêu thành công hoặc sau thêm 0.8s nữa thì ngắt để cất cần
                                    if nowOnCd or hpDropped or (elapsed >= 3.8) then
                                        break
                                    end
                                end
                            end

                            -- 2. Đợi 0.35s cho nhân vật chém đòn / tung chiêu xong để server ghi nhận
                            local waitFinish = tick()
                            while isRunning and (tick() - waitFinish) < 0.35 do
                                if fUI and fUI.Visible then
                                    local barFrame = fUI:FindFirstChild("BarFrame")
                                    if barFrame and barFrame:FindFirstChild("Bar") then
                                        barFrame.Bar.Position = UDim2.new(0.5, 0, 0.5, 0)
                                    end
                                end
                                task.wait(0.05)
                            end

                            -- 3. Cập nhật số lần đã luyện
                            Config.TrainCurrentCount = Config.TrainCurrentCount + 1
                            if infoTrainProgress and infoTrainProgress.Set then
                                infoTrainProgress.Set(string.format("%d / %d lần (Vừa cast: %s)", Config.TrainCurrentCount, Config.TrainTargetCount, cleanKey))
                            end

                            if Config.TrainCurrentCount >= Config.TrainTargetCount then
                                Config.AutoTrainSkill = false
                                ShowNotification("Luyện Chiêu Hoàn Tất", string.format("Đã luyện đủ %d/%d lần cho chiêu %s!", Config.TrainCurrentCount, Config.TrainTargetCount, cleanKey), "SUCCESS", 7)
                                pcall(function()
                                    local c = LocalPlayer.Character
                                    local h = c and c:FindFirstChildOfClass("Humanoid")
                                    if h then h:UnequipTools() end
                                end)
                                isTrainingBusy = false
                                return
                            end

                            -- 4. BẤM THÁO CẦN (UnequipTools) ĐỂ HỦY CÁ
                            local rodSlot = "1"
                            local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
                            if pData and pData:FindFirstChild("Hotbar") then
                                for _, item in ipairs(pData.Hotbar:GetChildren()) do
                                    local vName = item:FindFirstChild("ValueName")
                                    if vName and tostring(vName.Value):lower():find("rod") and not tostring(vName.Value):lower():find("inventory") then
                                        rodSlot = item.Name
                                        break
                                    end
                                end
                            end

                            pcall(function()
                                local c = LocalPlayer.Character
                                local h = c and c:FindFirstChildOfClass("Humanoid")
                                if h then h:UnequipTools() end
                            end)

                            -- 5. LẤY CẦN RA LẠI
                            task.wait(0.25)
                            pcall(function()
                                if Events and Events:FindFirstChild("ToggleHotbar") then
                                    Events.ToggleHotbar:InvokeServer(rodSlot)
                                else
                                    local vim = game:GetService("VirtualInputManager")
                                    if vim then
                                        vim:SendKeyEvent(true, Enum.KeyCode.One, false, game)
                                        task.wait(0.02)
                                        vim:SendKeyEvent(false, Enum.KeyCode.One, false, game)
                                    end
                                end
                            end)

                            -- 6. Hoàn tất chu trình, chuyển quyền quăng cần mượt mà cho AutoCast
                            task.wait(0.35)
                            lastCastTime = tick()
                            isTrainingBusy = false
                        end)
                    end
                elseif Config.AutoTicketQuest and hasActiveTicket and (ticketQuestState.currentQuestType == "bait_100" or ticketQuestState.currentQuestType == "skill_100" or ticketQuestState.currentQuestType == "fish_100") then
                    local qType = ticketQuestState.currentQuestType
                    if qType == "bait_100" then
                        if not ticketQuestState.isBusyRoutine then
                            ticketQuestState.isBusyRoutine = true
                            task.spawn(function()
                                pcall(function()
                                    local c = LocalPlayer.Character
                                    local h = c and c:FindFirstChildOfClass("Humanoid")
                                    if h then h:UnequipTools() end
                                end)
                                ticketQuestState.currentProgress = ticketQuestState.currentProgress + 1
                                ticketQuestState.UpdateUI()
                                if ticketQuestState.currentProgress >= 100 then
                                    ticketQuestState.isCompleted = true
                                end

                                task.wait(0.25)
                                local rodSlot = "1"
                                local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
                                if pData and pData:FindFirstChild("Hotbar") then
                                    for _, item in ipairs(pData.Hotbar:GetChildren()) do
                                        local vName = item:FindFirstChild("ValueName")
                                        if vName and tostring(vName.Value):lower():find("rod") and not tostring(vName.Value):lower():find("inventory") then
                                            rodSlot = item.Name
                                            break
                                        end
                                    end
                                end
                                pcall(function()
                                    if Events and Events:FindFirstChild("ToggleHotbar") then
                                        Events.ToggleHotbar:InvokeServer(rodSlot)
                                    end
                                end)

                                task.wait(0.35)
                                lastCastTime = tick()
                                ticketQuestState.isBusyRoutine = false
                            end)
                        end
                    elseif qType == "skill_100" then
                        if not ticketQuestState.isBusyRoutine then
                            ticketQuestState.isBusyRoutine = true
                            task.spawn(function()
                                pcall(function()
                                    -- 1. Ưu tiên số 1: Dùng đúng chiêu TicketSkillKey đã cài đặt cho nhiệm vụ 100 Skill (VD: "Chiêu Z")
                                    local comboList = {}
                                    local skillKey = Config.TicketSkillKey and Config.TicketSkillKey:match("([ZXCVzxcv])%s*$")
                                    if skillKey then
                                        table.insert(comboList, skillKey:upper())
                                    elseif Config.LoopSkills and Config.LoopSkills ~= "" then
                                        for k in string.gmatch(Config.LoopSkills, "([ZXCVzxcv])") do
                                            table.insert(comboList, k:upper())
                                        end
                                    end
                                    if #comboList == 0 then
                                        comboList = {"Z"}
                                    end

                                    -- 2. Giữ thăng bằng thanh bar và chờ qua 3 giây khóa chiêu đầu trận của game
                                    local startTime = tick()
                                    while isRunning and (fUI and fUI.Visible) do
                                        local barFrame = fUI:FindFirstChild("BarFrame")
                                        if barFrame and barFrame:FindFirstChild("Bar") then
                                            barFrame.Bar.Position = UDim2.new(0.5, 0, 0.5, 0)
                                        end
                                        if (tick() - startTime) >= 3.05 then
                                            break
                                        end
                                        task.wait(0.05)
                                    end

                                    -- 3. Lần lượt tung TOÀN BỘ chuỗi combo đã cài (Z -> X -> V...)
                                    for _, sk in ipairs(comboList) do
                                        if not isRunning or not (fUI and fUI.Visible) then break end

                                        local barFrame = fUI:FindFirstChild("BarFrame")
                                        if barFrame and barFrame:FindFirstChild("Bar") then
                                            barFrame.Bar.Position = UDim2.new(0.5, 0, 0.5, 0)
                                        end

                                        comboState.CastSkill(sk)
                                        ticketQuestState.currentProgress = ticketQuestState.currentProgress + 1
                                        ticketQuestState.UpdateUI()
                                        if ticketQuestState.targetProgress and ticketQuestState.currentProgress >= ticketQuestState.targetProgress then
                                            ticketQuestState.isCompleted = true
                                        elseif ticketQuestState.currentProgress >= 100 then
                                            ticketQuestState.isCompleted = true
                                        end

                                        -- Đợi nhịp giữa các chiêu để server nhận diện và animation nhân vật chạy (0.35s)
                                        local waitFinish = tick()
                                        while isRunning and (fUI and fUI.Visible) and (tick() - waitFinish < 0.35) do
                                            local bf = fUI:FindFirstChild("BarFrame")
                                            if bf and bf:FindFirstChild("Bar") then
                                                bf.Bar.Position = UDim2.new(0.5, 0, 0.5, 0)
                                            end
                                            task.wait(0.05)
                                        end
                                    end

                                    -- 4. Kéo cá lên (Charge 100 + Slam Perfect + UpdateFishProgression)
                                    local pullStartTime = tick()
                                    while isRunning and (fUI and fUI.Visible) and (tick() - pullStartTime < 25.0) do
                                        local barFrame = fUI:FindFirstChild("BarFrame")
                                        if barFrame and barFrame:FindFirstChild("Bar") then
                                            barFrame.Bar.Position = UDim2.new(0.5, 0, 0.5, 0)
                                        end
                                        if Events and Events:FindFirstChild("Slam") then
                                            Events.Slam:FireServer("Perfect")
                                        end
                                        if Events and Events:FindFirstChild("Charge") then
                                            Events.Charge:FireServer(100)
                                        end
                                        if Events and Events:FindFirstChild("UpdateFishProgression") then
                                            Events.UpdateFishProgression:FireServer()
                                        end
                                        task.wait(0.08)
                                    end
                                end)
                                task.wait(0.3)
                                lastCastTime = tick()
                                ticketQuestState.isBusyRoutine = false
                            end)
                        end
                    elseif qType == "fish_100" then
                        if not ticketQuestState.isBusyRoutine then
                            ticketQuestState.isBusyRoutine = true
                            task.spawn(function()
                                pcall(function()
                                    -- 1. Chỉ dùng duy nhất chiêu TicketQuickSkill (Mặc định: Chiêu V) cho nhiệm vụ 100 Cá (Tuyệt đối không dùng LoopSkills)
                                    local quickKey = Config.TicketQuickSkill and Config.TicketQuickSkill:match("([ZXCVzxcv])%s*$")
                                    local comboList = {quickKey and quickKey:upper() or "V"}

                                    -- 2. Giữ thăng bằng thanh bar và chờ qua 3 giây khóa chiêu đầu trận của game
                                    local startTime = tick()
                                    while isRunning and (fUI and fUI.Visible) do
                                        local barFrame = fUI:FindFirstChild("BarFrame")
                                        if barFrame and barFrame:FindFirstChild("Bar") then
                                            barFrame.Bar.Position = UDim2.new(0.5, 0, 0.5, 0)
                                        end
                                        if (tick() - startTime) >= 3.05 then
                                            break
                                        end
                                        task.wait(0.05)
                                    end

                                    -- 3. Lần lượt tung chuỗi combo (Z -> X -> V...)
                                    for _, sk in ipairs(comboList) do
                                        if not isRunning or not (fUI and fUI.Visible) then break end

                                        local barFrame = fUI:FindFirstChild("BarFrame")
                                        if barFrame and barFrame:FindFirstChild("Bar") then
                                            barFrame.Bar.Position = UDim2.new(0.5, 0, 0.5, 0)
                                        end

                                        comboState.CastSkill(sk)

                                        local waitFinish = tick()
                                        while isRunning and (fUI and fUI.Visible) and (tick() - waitFinish < 0.35) do
                                            local bf = fUI:FindFirstChild("BarFrame")
                                            if bf and bf:FindFirstChild("Bar") then
                                                bf.Bar.Position = UDim2.new(0.5, 0, 0.5, 0)
                                            end
                                            task.wait(0.05)
                                        end
                                    end

                                    -- 4. Kéo cá lên
                                    local pullStartTime = tick()
                                    while isRunning and (fUI and fUI.Visible) and (tick() - pullStartTime < 25.0) do
                                        local barFrame = fUI:FindFirstChild("BarFrame")
                                        if barFrame and barFrame:FindFirstChild("Bar") then
                                            barFrame.Bar.Position = UDim2.new(0.5, 0, 0.5, 0)
                                        end
                                        if Events and Events:FindFirstChild("Slam") then
                                            Events.Slam:FireServer("Perfect")
                                        end
                                        if Events and Events:FindFirstChild("Charge") then
                                            Events.Charge:FireServer(100)
                                        end
                                        if Events and Events:FindFirstChild("UpdateFishProgression") then
                                            Events.UpdateFishProgression:FireServer()
                                        end
                                        task.wait(0.08)
                                    end

                                    -- 5. Cập nhật tiến độ bắt cá khi cá đã kéo lên thành công
                                    ticketQuestState.currentProgress = ticketQuestState.currentProgress + 1
                                    ticketQuestState.UpdateUI()
                                    if ticketQuestState.targetProgress and ticketQuestState.currentProgress >= ticketQuestState.targetProgress then
                                        ticketQuestState.isCompleted = true
                                    elseif ticketQuestState.currentProgress >= 100 then
                                        ticketQuestState.isCompleted = true
                                    end
                                end)
                                task.wait(0.3)
                                lastCastTime = tick()
                                ticketQuestState.isBusyRoutine = false
                            end)
                        end
                    end
                elseif fUI and fUI.Visible then
                    local is15mQuest = Config.AutoTicketQuest and ticketQuestState and ticketQuestState.currentQuestType == "fish_15m"

                    -- Tự động giữ thanh cân bằng minigame (Anchor Bar)
                    local isHomeFishing = Config.AutoTicketQuest and ticketQuestState and ticketQuestState.isCooldown and ticketQuestState.isAtHomeSpot and Config.TicketAutoCastAtHome
                    if (Config.AutoMinigame or Config.AnchorBar or (Config.AutoChatSecretBoss and secretBossState.active) or Config.AutoHuntBoss or isHomeFishing or is15mQuest) then
                        local barFrame = fUI:FindFirstChild("BarFrame")
                        if barFrame and barFrame:FindFirstChild("Bar") then
                            barFrame.Bar:TweenPosition(UDim2.new(0.5, 0, 0.5, 0), Enum.EasingDirection.InOut, Enum.EasingStyle.Linear, 0, true)
                            barFrame.Bar.Position = UDim2.new(0.5, 0, 0.5, 0)
                        end
                    end

                    if (Config.AutoMinigame or Config.AutoSlam or is15mQuest) and fUI:FindFirstChild("PerfectButton") and fUI.PerfectButton.Visible then
                        if Events:FindFirstChild("Slam") then
                            Events.Slam:FireServer("Perfect")
                        end
                    end

                    if (Config.AutoMinigame or Config.AutoCharge or is15mQuest) and fUI:FindFirstChild("Charge") and fUI.Charge.Visible then
                        if Events:FindFirstChild("Charge") then
                            Events.Charge:FireServer(100)
                        end
                    end

                    if (Config.AutoMinigame or Config.AnchorBar or is15mQuest) and (now - lastProgressionTime >= 0.08) then
                        if Events and Events:FindFirstChild("UpdateFishProgression") then
                            Events.UpdateFishProgression:FireServer()
                        end
                        lastProgressionTime = now
                    end

                    if (Config.SmartComboEnabled or is15mQuest) and (now - lastSkillTime >= 0.12) and not isTrainingBusy then
                        lastSkillTime = now

                        local playerHp = GetPlayerHealth(fUI)

                        -- BƯỚC 1: CỨU NGUY HỒI MÁU (Khi máu người chơi <= EmergencyHealHp)
                        local didHeal = false
                        local healKey = Config.EmergencyHealSkill and Config.EmergencyHealSkill ~= "Tắt" and Config.EmergencyHealSkill:match("([ZXCVzxcv])")
                        if healKey then healKey = healKey:upper() end

                        if healKey and playerHp <= (Config.EmergencyHealHp or 40) then
                            if not comboState.IsSkillOnCooldown(healKey, fUI) then
                                comboState.CastSkill(healKey)
                                comboState.lastActionTime = now
                                didHeal = true
                            end
                        end

                        if not didHeal then
                            -- BƯỚC 2: THI TRIỂN CHUỖI ĐẢO CHIÊU (Spam quan sát server nhận nút khóa -> 0.15s pass nhảy chiêu)
                            local loopKeys = {}
                            local curQ = (Config.AutoTicketQuest and ticketQuestState and ticketQuestState.currentQuestType) or "none"
                            if curQ == "fish_100" then
                                local qk = Config.TicketQuickSkill and Config.TicketQuickSkill:match("([ZXCVzxcv])%s*$")
                                table.insert(loopKeys, qk and qk:upper() or "V")
                            elseif curQ == "skill_100" then
                                local sk = Config.TicketSkillKey and Config.TicketSkillKey:match("([ZXCVzxcv])%s*$")
                                table.insert(loopKeys, sk and sk:upper() or "Z")
                            end

                            if #loopKeys == 0 then
                                for k in string.gmatch(Config.LoopSkills or "Z, X, V", "([ZXCVzxcv])") do
                                    table.insert(loopKeys, k:upper())
                                end
                            end

                            if #loopKeys > 0 then
                                if comboState.loopTargetIndex < 1 or comboState.loopTargetIndex > #loopKeys then
                                    comboState.loopTargetIndex = 1
                                end

                                local targetKey = loopKeys[comboState.loopTargetIndex]
                                local btn = comboState.GetSkillButton(targetKey, fUI)

                                if not btn and fUI and fUI:FindFirstChild("SkillButton") then
                                    -- Chiêu này cần câu không có -> Bỏ qua sang chiêu tiếp theo
                                    comboState.loopTargetIndex = (comboState.loopTargetIndex % #loopKeys) + 1
                                else
                                    local onCd = comboState.IsSkillOnCooldown(targetKey, fUI)

                                    if not onCd then
                                        -- Nút đang sáng: Bấm chiêu ngay!
                                        comboState.CastSkill(targetKey)
                                        comboState.lastActionTime = now
                                        comboState.lastCastingKey = targetKey

                                        -- Sau 0.08s: kiểm tra nếu nút đã bị khóa Cooldown (server đã nhận) -> PASS NHẢY SANG CHIÊU KẾ!
                                        task.delay(0.08, function()
                                            if comboState.lastCastingKey == targetKey and comboState.IsSkillOnCooldown(targetKey, fUI) then
                                                comboState.loopTargetIndex = (comboState.loopTargetIndex % #loopKeys) + 1
                                                comboState.lastCastingKey = nil
                                            end
                                        end)
                                    else
                                        -- Nút đang bị khóa Cooldown:
                                        -- Nếu vừa bấm chiêu này ở nhịp trước và nút đã khóa thành công:
                                        if comboState.lastCastingKey == targetKey then
                                            comboState.loopTargetIndex = (comboState.loopTargetIndex % #loopKeys) + 1
                                            comboState.lastCastingKey = nil
                                        elseif not Config.LoopStrictOrder then
                                            -- Nếu TẮT StrictOrder: linh hoạt tìm chiêu khác đang mở khóa để bấm
                                            for offset = 1, #loopKeys - 1 do
                                                local idx = ((comboState.loopTargetIndex - 1 + offset) % #loopKeys) + 1
                                                local sk = loopKeys[idx]
                                                if not comboState.IsSkillOnCooldown(sk, fUI) then
                                                    comboState.CastSkill(sk)
                                                    comboState.lastActionTime = now
                                                    comboState.lastCastingKey = sk
                                                    comboState.loopTargetIndex = (idx % #loopKeys) + 1
                                                    break
                                                end
                                            end
                                        end
                                    end
                                end
                            else
                                local fallbackKey = (Config.QuickCatchSkill and Config.QuickCatchSkill ~= "Tắt" and Config.QuickCatchSkill:match("([ZXCVzxcv])"))
                                    or (Config.OpenerSkill and Config.OpenerSkill ~= "Tắt" and Config.OpenerSkill:match("([ZXCVzxcv])"))
                                    or "Z"
                                fallbackKey = fallbackKey:upper()
                                if not comboState.IsSkillOnCooldown(fallbackKey, fUI) then
                                    comboState.CastSkill(fallbackKey)
                                    comboState.lastActionTime = now
                                end
                            end
                        end
                        lastSkillTime = now
                    elseif (Config.AutoSkills or is15mQuest) and (now - lastSkillTime >= 0.15) then
                        local curQ = ticketQuestState and ticketQuestState.currentQuestType
                        if (not curQ or curQ == "none") and ticketQuestState and ticketQuestState.DetectActiveQuest then
                            curQ = select(1, ticketQuestState.DetectActiveQuest())
                        end

                        -- fish_100/skill_100: 1 chiêu cố định. Tất cả còn lại: dùng LoopSkills đầy đủ
                        if curQ == "fish_100" then
                            local qk = Config.TicketQuickSkill and Config.TicketQuickSkill:match("([ZXCVzxcv])%s*$")
                            comboState.CastSkill(qk and qk:upper() or "V")
                        elseif curQ == "skill_100" then
                            local sk = Config.TicketSkillKey and Config.TicketSkillKey:match("([ZXCVzxcv])%s*$")
                            comboState.CastSkill(sk and sk:upper() or "Z")
                        else
                            -- fish_15m, bait_100, không quest, hoặc normal fishing -> dùng LoopSkills
                            local skillList = {}
                            if Config.LoopSkills and Config.LoopSkills ~= "" then
                                for k in string.gmatch(Config.LoopSkills, "([ZXCVzxcv])") do
                                    table.insert(skillList, k:upper())
                                end
                            end
                            if #skillList == 0 then skillList = {"Z", "X", "V"} end
                            for _, sk in ipairs(skillList) do
                                comboState.CastSkill(sk)
                            end
                        end
                        lastSkillTime = now
                    end
            end
            end -- Kết thúc check not skipTriggered
        elseif isFishing then
            secretBossState.minigameStartTime = 0
            lastCastTime = now
        else
            local isBossActive = secretBossState and secretBossState.active
            local isTicketActive = ticketQuestState and ticketQuestState.IsFishingActive and ticketQuestState.IsFishingActive()
            local isTicketBusy = (ticketQuestState and ticketQuestState.IsBusyOrInteracting and ticketQuestState.IsBusyOrInteracting()) or (ticketQuestState and ticketQuestState.isBusyRoutine)
            local shouldAutoCast = not isTicketBusy and (Config.AutoCast or Config.AutoTrainSkill or isTicketActive or ((Config.AutoHuntBoss or Config.AutoChatSecretBoss) and isBossActive))
            if shouldAutoCast and not isCD and not isSwimming and (char:GetAttribute("Type") == "Fishing Rod") and (now - lastCastTime >= Config.CastDelay) and not isTrainingBusy and not isTicketBusy then
                local canCast = true
                if pData and pData:FindFirstChild("InventoryLimit") then
                    local invCount = 0
                    if pData:FindFirstChild("Inventory") then invCount = invCount + #pData.Inventory:GetChildren() end
                    if pData:FindFirstChild("Hotbar") then
                        for _, item in ipairs(pData.Hotbar:GetChildren()) do
                            if item:FindFirstChild("Quantity") then invCount = invCount + item.Quantity.Value else invCount = invCount + 1 end
                        end
                    end
                    if invCount >= pData.InventoryLimit.Value then
                        canCast = false
                        if Config.AutoSell and (now - lastSellTime >= 5.0) and Events:FindFirstChild("SellFish") then
                            ProtectAllInventoryItems(false)
                            Events.SellFish:FireServer("All")
                            lastSellTime = now
                        end
                    end
                end

                if Config.AutoSell and (now - lastSellTime >= Config.SellInterval) and Events:FindFirstChild("SellFish") then
                    ProtectAllInventoryItems(false)
                    Events.SellFish:FireServer("All")
                    lastSellTime = now
                end


                if canCast and Events and Events:FindFirstChild("Fishing") then
                    Events.Fishing:FireServer(root.CFrame)
                    lastCastTime = now
                end
            end
        end

        if (now - lastEquipTime >= 2.5) and pData then
            lastEquipTime = now
            if pData:FindFirstChild("Bait") and pData:FindFirstChild("EquippedBait") and Events:FindFirstChild("EquipBait") then
                local isHuntingBoss = (Config.AutoHuntBoss or Config.AutoChatSecretBoss) and secretBossState.active
                local isTicketFish100 = Config.AutoTicketQuest and ticketQuestState and ticketQuestState.active and not ticketQuestState.isCooldown and ticketQuestState.currentQuestType == "fish_100"
                local targetBaitChoice = nil

                if not isTicketFish100 then
                    if isHuntingBoss and Config.AutoEquipBossBait then
                        targetBaitChoice = Config.BaitChoiceBoss or "Mồi Tốt Nhất (Cao Nhất)"
                    elseif Config.AutoEquipBestBait then
                        targetBaitChoice = Config.BaitChoiceNormal or "Mồi Tốt Nhất (Cao Nhất)"
                    end
                end

                if targetBaitChoice then
                    local baitTiers = {
                        "Nameless Bait",
                        "Rainbow Bait",
                        "Frost Bait",
                        "Ancestral Bait",
                        "Elite Bait",
                        "Corrupted Essence Bait",
                        "Crude Mash Bait",
                        "Basic Bait"
                    }
                    local desiredBait = nil

                    if targetBaitChoice == "Mồi Tốt Nhất (Cao Nhất)" then
                        for _, bName in ipairs(baitTiers) do
                            local bVal = pData.Bait:FindFirstChild(bName)
                            if bVal and bVal.Value > 0 then
                                desiredBait = bName
                                break
                            end
                        end
                    elseif targetBaitChoice == "Mồi Thấp Nhất (Tiết Kiệm)" then
                        for i = #baitTiers, 1, -1 do
                            local bName = baitTiers[i]
                            local bVal = pData.Bait:FindFirstChild(bName)
                            if bVal and bVal.Value > 0 then
                                desiredBait = bName
                                break
                            end
                        end
                    else
                        local bVal = pData.Bait:FindFirstChild(targetBaitChoice)
                        if bVal and bVal.Value > 0 then
                            desiredBait = targetBaitChoice
                        else
                            for _, bName in ipairs(baitTiers) do
                                local bv = pData.Bait:FindFirstChild(bName)
                                if bv and bv.Value > 0 then
                                    desiredBait = bName
                                    break
                                end
                            end
                        end
                    end

                    if desiredBait and pData.EquippedBait.Value ~= desiredBait then
                        task.spawn(function()
                            Events.EquipBait:InvokeServer(desiredBait)
                        end)
                    end
                end
            end

            if Config.AutoEquipBestRod and pData:FindFirstChild("FishingRod") and Events:FindFirstChild("EquipFishingRod") then
                local bestRod = nil
                local bestPower = -1
                for _, r in ipairs(allRods) do
                    local isOwned = IsRodOwned(r.name)
                    if isOwned and r.power > bestPower then
                        bestPower = r.power
                        bestRod = r.name
                    end
                end
                if bestRod and pData.FishingRod.Value ~= bestRod then
                    task.spawn(function()
                        if Events and Events:FindFirstChild("ToggleHotbar") then
                            Events.ToggleHotbar:InvokeServer("1")
                        end
                    end)
                end
            end

            if Config.AutoEquipBestOrb and pData:FindFirstChild("Orb") and Events:FindFirstChild("EquipOrb") then
                local orbs = pData.Orb:GetChildren()
                if #orbs > 0 then
                    local bestOrb = orbs[#orbs].Name
                    task.spawn(function()
                        Events.EquipOrb:InvokeServer(bestOrb)
                    end)
                end
            end

            if pData:FindFirstChild("FishingRod") and pData.FishingRod.Value ~= "" and StatTiles.EquippedRod then StatTiles.EquippedRod.Set(pData.FishingRod.Value) end
            if pData:FindFirstChild("EquippedBait") and pData.EquippedBait.Value ~= "" and StatTiles.EquippedBait then StatTiles.EquippedBait.Set(pData.EquippedBait.Value) end

            -- Cập nhật tên vị trí hiện tại mỗi 3 giây (đồng bộ cả Tab Dịch Chuyển và Tab Câu Cá)
            if not lastIslandCheck or (tick() - lastIslandCheck >= 3) then
                lastIslandCheck = tick()
                if GetCurrentLocationName then
                    local curLoc = GetCurrentLocationName()
                    if infoCurrentMap and infoCurrentMap.Set then infoCurrentMap.Set(curLoc) end
                    if StatTiles.CurrentLocation and StatTiles.CurrentLocation.Set then StatTiles.CurrentLocation.Set(curLoc) end
                end
            end

            -- Tự động hook túi đồ để phát hiện cá thưởng Gems khi câu trúng
            if not gemTracker.inventoryHooked and pData then
                local inv = pData:FindFirstChild("Inventory")
                if inv then
                    gemTracker.inventoryHooked = true
                    table.insert(activeConnections, inv.ChildAdded:Connect(TrackCaughtFishForGems))
                end
                local hotbar = pData:FindFirstChild("Hotbar")
                if hotbar then
                    table.insert(activeConnections, hotbar.ChildAdded:Connect(TrackCaughtFishForGems))
                end
            end

            -- Đọc Gems từ game (pData / leaderstats / PlayerGui / v.v.)
            local liveGems = GetPlayerCurrentGems()
            if liveGems ~= nil then
                if gemTracker.lastKnown == nil then
                    gemTracker.lastKnown = liveGems
                elseif liveGems > gemTracker.lastKnown then
                    local delta = liveGems - gemTracker.lastKnown
                    gemTracker.lastKnown = liveGems
                    if not (tick() - gemTracker.lastFishAwardTime <= 2.5 and delta == gemTracker.lastFishAwardAmount) then
                        gemTracker.gained = gemTracker.gained + delta
                    end
                elseif liveGems < gemTracker.lastKnown then
                    gemTracker.lastKnown = liveGems
                end
            end

            local curFish = pData:FindFirstChild("FishCaught") and tonumber(pData.FishCaught.Value) or 0
            local curCash = pData:FindFirstChild("Cash") and tonumber(pData.Cash.Value) or 0

            if StatTiles.FishCaught and StatTiles.FishCaught.Set then
                StatTiles.FishCaught.Set(FormatWithSpaces(curFish) .. " con")
            end
            if StatTiles.Cash and StatTiles.Cash.Set then
                StatTiles.Cash.Set("$" .. FormatWithSpaces(curCash))
            end

            if not initialFishCaught then initialFishCaught = curFish end
            if not initialCash then initialCash = curCash end

            local elapsedSec = math.max(1, tick() - sessionStartTime)
            local elapsedHours = elapsedSec / 3600

            local h = math.floor(elapsedSec / 3600)
            local m = math.floor((elapsedSec % 3600) / 60)
            local s = math.floor(elapsedSec % 60)
            if StatTiles.Uptime and StatTiles.Uptime.Set then
                StatTiles.Uptime.Set(string.format("%02d:%02d:%02d", h, m, s))
            end

            local gainedFish = math.max(0, curFish - initialFishCaught)
            local gainedCash = math.max(0, curCash - initialCash)
            local gainedGems = gemTracker.gained

            local fishRate = math.floor(gainedFish / math.max(elapsedHours, 1/3600))
            local cashRate = math.floor(gainedCash / math.max(elapsedHours, 1/3600))

            if StatTiles.FishPerHour and StatTiles.FishPerHour.Set then
                StatTiles.FishPerHour.Set(FormatWithSpaces(fishRate) .. " con/h")
            end
            if StatTiles.CashPerHour and StatTiles.CashPerHour.Set then
                StatTiles.CashPerHour.Set("$" .. FormatWithSpaces(cashRate) .. " /h")
            end
            if StatTiles.GemsGained and StatTiles.GemsGained.Set then
                if visualSpoofState and visualSpoofState.fakeGems and visualSpoofState.fakeGems > 0 then
                    StatTiles.GemsGained.Set(FormatWithSpaces(visualSpoofState.fakeGems) .. " Gems")
                else
                    local realG = (GetPlayerCurrentGems and GetPlayerCurrentGems()) or gainedGems
                    StatTiles.GemsGained.Set(FormatWithSpaces(realG) .. " Gems")
                end
            end

            -- Cập nhật chỉ số tài nguyên, balo và vé nhiệm vụ
            if StatTiles.Tickets and StatTiles.Tickets.Set then
                local tVal = (visualSpoofState and visualSpoofState.fakeTicket and visualSpoofState.fakeTicket > 0)
                    and visualSpoofState.fakeTicket
                    or (pData:FindFirstChild("Ticket") and tonumber(pData.Ticket.Value) or 0)
                StatTiles.Tickets.Set(FormatWithSpaces(tVal) .. " Vé")
            end
            if visualSpoofState and ((visualSpoofState.fakeTicket and visualSpoofState.fakeTicket > 0) or (visualSpoofState.fakeGems and visualSpoofState.fakeGems > 0)) then
                visualSpoofState.Apply()
            end
            if StatTiles.EssenceOrbs and StatTiles.EssenceOrbs.Set then
                local orbVal = pData:FindFirstChild("EssenceOrb") and tonumber(pData.EssenceOrb.Value) or 0
                StatTiles.EssenceOrbs.Set(FormatWithSpaces(orbVal) .. " Viên")
            end
            if StatTiles.TraitRerolls and StatTiles.TraitRerolls.Set then
                local rVal = pData:FindFirstChild("Trait Reroll") and tonumber(pData["Trait Reroll"].Value) or 0
                StatTiles.TraitRerolls.Set(FormatWithSpaces(rVal) .. " Vé")
            end
            if StatTiles.TicketQuestsToday and StatTiles.TicketQuestsToday.Set then
                local qCount = pData:FindFirstChild("TicketQuestDailyCount") and tonumber(pData.TicketQuestDailyCount.Value) or 0
                StatTiles.TicketQuestsToday.Set(string.format("%d NV", qCount))
            end
            if StatTiles.Backpack and StatTiles.Backpack.Set then
                local invCount, limitVal = GetCurrentBackpackFishCount()
                StatTiles.Backpack.Set(string.format("%d / %d", invCount, limitVal))
            end
            if StatTiles.TicketCooldown and StatTiles.TicketCooldown.Set then
                local cdVal = pData:FindFirstChild("TicketQuestCooldown") and tonumber(pData.TicketQuestCooldown.Value) or 0
                local nowServer = Workspace:GetServerTimeNow()
                local remain = math.max(0, cdVal - nowServer)
                if ticketQuestState and ticketQuestState.IsAllQuestsDoneToday and ticketQuestState.IsAllQuestsDoneToday() then
                    StatTiles.TicketCooldown.Set("Hết vé hôm nay")
                elseif remain > 0 then
                    local rm = math.floor(remain / 60)
                    local rs = remain % 60
                    StatTiles.TicketCooldown.Set(string.format("Chờ %02d:%02d", rm, rs))
                else
                    StatTiles.TicketCooldown.Set("Sẵn sàng nhận!")
                end
            end

            local intervalSec = ((Config.WebhookStatsInterval and Config.WebhookStatsInterval >= 1 and Config.WebhookStatsInterval <= 300) and (Config.WebhookStatsInterval * 60) or (Config.WebhookStatsInterval or 1800))
            if Config.WebhookEnabled and Config.WebhookHourlyStats and (tick() - lastWebhookStatsTime >= intervalSec) then
                lastWebhookStatsTime = tick()
                SendServerReportWebhook("📊 Báo Cáo Định Kỳ - Farm Tracker")
            end
        end

        if Config.AutoSell and (now - lastSellTime >= Config.SellInterval) then
            if Events and Events:FindFirstChild("SellFish") then
                Events.SellFish:FireServer("All")
                lastSellTime = now
            end
        end

        if Config.AutoPrayGodSpirit and (now - lastGodPrayTime >= 3.0) then
            if Workspace:FindFirstChild("NPC") then
                local sp = Workspace.NPC:FindFirstChild("Spirit") or Workspace.NPC:FindFirstChild("God")
                if sp then
                    for _, d in ipairs(sp:GetDescendants()) do
                        if d:IsA("ProximityPrompt") then
                            TriggerPrompt(d)
                            lastGodPrayTime = now
                        end
                    end
                end
            end
        end


        if Config.AutoClaimDaily and (now - lastCastTime >= 2.0) then
            if Events and Events:FindFirstChild("DailyReward") then
                for day = 1, 7 do Events.DailyReward:FireServer(day) end
            end
        end

        if Config.AutoGacha and (now - lastGachaTime >= 1.5) then
            if Events and Events:FindFirstChild("Gacha") then
                Events.Gacha:FireServer(Config.GachaBanner, Config.GachaPullsPerAction)
                lastGachaTime = now
            end
        end

        if Config.AutoCraftBait and (now - lastCastTime >= 2.0) then
            if Events and Events:FindFirstChild("CraftBait") then
                Wiki.UnlockBaitFish(Config.CraftBaitName, true)
                Events.CraftBait:FireServer(Config.CraftBaitName, Config.CraftAmount)
            end
        end

        if Config.AutoBuyBait and (now - lastBaitBuyTime >= Config.BuyBaitDelay) then
            if Events and Events:FindFirstChild("BuyBait") then
                Events.BuyBait:FireServer(Config.BuyBaitName, Config.BuyBaitAmount)
                lastBaitBuyTime = now
            end
        end

        if Config.AutoMinigame or Config.OctoAutoMinigame then
            if Events and Events:FindFirstChild("RhythmHit") then
                Events.RhythmHit:FireServer(true, 100)
            end
            local pg = LocalPlayer:FindFirstChild("PlayerGui")
            local fUI = pg and pg:FindFirstChild("MainGui") and pg.MainGui:FindFirstChild("Fishing")
            if fUI and fUI.Visible and fUI:FindFirstChild("RhythmFrame") then
                local rFrame = fUI.RhythmFrame
                if rFrame.Visible and Events and Events:FindFirstChild("RhythmHit") then
                    for _, hitNote in ipairs(rFrame:GetChildren()) do
                        if hitNote.Name:find("Note") and hitNote.Visible then
                            Events.RhythmHit:FireServer(hitNote.Name)
                        end
                    end
                end
            end
        end
    end)
end))

-- Background task: RGB Rainbow Rod Color Cycle
task.spawn(function()
    local hue = 0
    while isRunning do
        if Config.RainbowRodColor then
            hue = (hue + 0.02) % 1
            local c3 = Color3.fromHSV(hue, 0.9, 1)
            local pData = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(LocalPlayer.UserId)
            local rodName = pData and pData:FindFirstChild("FishingRod") and pData.FishingRod.Value or ""
            if Events and Events:FindFirstChild("SetRodSkinColor") then
                Events.SetRodSkinColor:FireServer(rodName, c3)
            end
            task.wait(0.25)
        else
            task.wait(1)
        end
    end
end)

local flyBV, flyBG = nil, nil
table.insert(activeConnections, RunService.RenderStepped:Connect(function()
    if not isRunning then return end
    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if not root then return end

    if Config.FlyEnabled then
        if not flyBV then
            flyBV = Instance.new("BodyVelocity")
            flyBV.MaxForce = Vector3.new(9e9, 9e9, 9e9)
            flyBV.Parent = root
            table.insert(cleanUpInstances, flyBV)

            flyBG = Instance.new("BodyGyro")
            flyBG.MaxTorque = Vector3.new(9e9, 9e9, 9e9)
            flyBG.Parent = root
            table.insert(cleanUpInstances, flyBG)
        end

        local camCF = Camera.CFrame
        local dir = Vector3.zero
        if UserInputService:IsKeyDown(Enum.KeyCode.W) then dir = dir + camCF.LookVector end
        if UserInputService:IsKeyDown(Enum.KeyCode.S) then dir = dir - camCF.LookVector end
        if UserInputService:IsKeyDown(Enum.KeyCode.D) then dir = dir + camCF.RightVector end
        if UserInputService:IsKeyDown(Enum.KeyCode.A) then dir = dir - camCF.RightVector end
        if UserInputService:IsKeyDown(Enum.KeyCode.Space) then dir = dir + Vector3.new(0, 1, 0) end
        if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then dir = dir - Vector3.new(0, 1, 0) end

        flyBG.CFrame = camCF
        flyBV.Velocity = dir.Unit * Config.FlySpeed
        if dir.Magnitude == 0 then flyBV.Velocity = Vector3.zero end
    else
        if flyBV then flyBV:Destroy(); flyBV = nil end
        if flyBG then flyBG:Destroy(); flyBG = nil end
    end
end))

local rhythmState = {
    lastLaneHit = {},
    hitNotes = {},
    lastClean = 0
}

local function CheckAndPlayRhythm(pg)
    if not (Config.AutoMinigame or Config.OctoAutoMinigame or Config.AnchorBar) then return end
    if not pg then return end

    local rhythmGui = nil
    local mainGui = pg:FindFirstChild("MainGui")
    if mainGui then
        if mainGui:FindFirstChild("Fishing") then
            rhythmGui = mainGui.Fishing:FindFirstChild("Rhythm")
        end
        if not rhythmGui then
            rhythmGui = mainGui:FindFirstChild("Rhythm")
        end
    end
    if not rhythmGui then
        rhythmGui = pg:FindFirstChild("Rhythm", true)
    end

    if not rhythmGui or not rhythmGui.Visible then return end

    local nowTick = tick()
    if nowTick - rhythmState.lastClean > 4.0 then
        rhythmState.hitNotes = {}
        rhythmState.lastClean = nowTick
    end

    local vim = game:GetService("VirtualInputManager")
    local lanes = {
        { name = "ProgressionA", key = "A", keyCode = Enum.KeyCode.A },
        { name = "ProgressionS", key = "S", keyCode = Enum.KeyCode.S },
        { name = "ProgressionD", key = "D", keyCode = Enum.KeyCode.D }
    }

    for _, laneData in ipairs(lanes) do
        local prog = rhythmGui:FindFirstChild(laneData.name)
        if prog and prog.Visible then
            local bFrame = prog:FindFirstChild("BarFrame")
            local btn = prog:FindFirstChild("Button")
            local targetScaleY = (bFrame and bFrame.Position.Y.Scale) or 0.85

            -- Duyệt chính xác các nốt đang rơi Note_FX theo mã decompile của game
            for _, child in ipairs(prog:GetChildren()) do
                if child.Name == "Note_FX" and child:IsA("GuiObject") and child.Visible then
                    local noteScaleY = child.Position.Y.Scale
                    local diff = math.abs(noteScaleY - targetScaleY)

                    -- Vùng hit chuẩn của game: diff <= 0.22
                    -- Nếu bật Chế Độ Người Thật (Humanizer): ngẫu nhiên hóa thời điểm bấm từ 0.08 đến 0.19
                    local triggerThreshold = 0.16
                    if Config.RhythmHumanizer ~= false then
                        triggerThreshold = math.random(8, 19) / 100
                    end

                    if diff <= triggerThreshold and not rhythmState.hitNotes[child] then
                        rhythmState.hitNotes[child] = true

                        -- Kiểm tra tỉ lệ trúng Accuracy (mặc định 95%)
                        local accuracy = Config.RhythmAccuracy or 95
                        local willHit = (math.random(1, 100) <= accuracy)

                        if willHit then
                            local delaySec = (Config.RhythmHumanizer ~= false) and (math.random(10, 35) / 1000) or 0
                            task.delay(delaySec, function()
                                -- 1. Kích hoạt hàm TryHit của game qua Button click
                                if btn and btn:IsA("GuiButton") then
                                    pcall(function()
                                        if firesignal then
                                            if btn.MouseButton1Click then firesignal(btn.MouseButton1Click) end
                                            if btn.Activated then firesignal(btn.Activated) end
                                        end
                                        if getconnections then
                                            for _, c in ipairs(getconnections(btn.MouseButton1Click)) do
                                                if c.Fire then c:Fire() elseif c.Function then c.Function() end
                                            end
                                        end
                                    end)
                                end

                                -- 2. Giả lập phím bấm bàn phím VIM (A, S, D)
                                if vim and laneData.keyCode then
                                    pcall(function()
                                        vim:SendKeyEvent(true, laneData.keyCode, false, game)
                                        task.delay(0.02, function()
                                            pcall(function() vim:SendKeyEvent(false, laneData.keyCode, false, game) end)
                                        end)
                                    end)
                                end

                                -- 3. Gửi RemoteEvent RhythmHit trực tiếp lên Server ("hit")
                                if Events and Events:FindFirstChild("RhythmHit") then
                                    pcall(function() Events.RhythmHit:FireServer("hit") end)
                                end
                            end)
                        end
                    end
                end
            end
        end
    end
end

table.insert(activeConnections, RunService.RenderStepped:Connect(function()
    if not isRunning then return end
    pcall(function()
        local pg = LocalPlayer:FindFirstChild("PlayerGui")
        if not pg then return end

        -- Xử lý Mini Game Nhịp Điệu 3 Phím A - S - D (Chạy độc lập đảm bảo luôn phát hiện)
        CheckAndPlayRhythm(pg)

        if Config.AutoMinigame or Config.AnchorBar then
            if pg:FindFirstChild("MainGui") and pg.MainGui:FindFirstChild("Fishing") and pg.MainGui.Fishing.Visible then
                local fUI = pg.MainGui.Fishing
                local barFrame = fUI:FindFirstChild("BarFrame")
                if barFrame and barFrame:FindFirstChild("Bar") then
                    barFrame.Bar.Position = UDim2.new(0.5, 0, 0.5, 0)
                end
                local bossBar = fUI:FindFirstChild("BossFightBar")
                if bossBar and bossBar.Visible and bossBar:FindFirstChild("Bar") and bossBar:FindFirstChild("Hitbox") then
                    bossBar.Bar.Position = bossBar.Hitbox.Position
                end
                local hpPlayer = fUI:FindFirstChild("HPPlayer")
                if hpPlayer and hpPlayer:FindFirstChild("ProgressionBar") then
                    local pBar = hpPlayer.ProgressionBar
                    if pBar:FindFirstChild("Bar") then
                        pBar.Bar.Size = UDim2.new(1, 0, 1, 0)
                    end
                end
                if fUI:FindFirstChild("PerfectButton") and fUI.PerfectButton.Visible then
                    if Events and Events:FindFirstChild("Slam") then Events.Slam:FireServer("Perfect") end
                end
                if fUI:FindFirstChild("Charge") and fUI.Charge.Visible then
                    if Events and Events:FindFirstChild("Charge") then Events.Charge:FireServer(100) end
                end
                local cutscene = pg:FindFirstChild("MainGui") and pg.MainGui:FindFirstChild("Cutscene")
                if cutscene and cutscene:FindFirstChild("Hurt") then
                    cutscene.Hurt.ImageTransparency = 1
                end
                local impactCutscene = pg:FindFirstChild("Impact") and pg.Impact:FindFirstChild("Cutscene")
                if impactCutscene and impactCutscene:FindFirstChild("Hurt") then
                    impactCutscene.Hurt.ImageTransparency = 1
                end
            end
        end
    end)
end))

if Events and Events:FindFirstChild("Slam") then
    table.insert(activeConnections, Events.Slam.OnClientEvent:Connect(function(slamEvent)
        if isRunning and (Config.AutoMinigame or Config.AnchorBar) and typeof(slamEvent) == "Instance" then
            pcall(function() slamEvent:FireServer("Perfect") end)
        end
    end))
end

if Events and Events:FindFirstChild("Charge") then
    table.insert(activeConnections, Events.Charge.OnClientEvent:Connect(function(chargeEvent)
        if isRunning and (Config.AutoMinigame or Config.AnchorBar) and typeof(chargeEvent) == "Instance" then
            pcall(function() chargeEvent:FireServer(100) end)
        end
    end))
end

-- LẮNG NGHE CHAT SERVER TỰ ĐỘNG SĂN SECRET BOSS
pcall(function()
    local TextChatService = game:GetService("TextChatService")
    if TextChatService and TextChatService:FindFirstChild("MessageReceived") then
        local conn = TextChatService.MessageReceived:Connect(function(textChatMessage)
            if not isRunning then return end
            if textChatMessage and textChatMessage.Text then
                secretBossState.HandleChatMessage(textChatMessage.Text)
            end
        end)
        table.insert(activeConnections, conn)
    end
end)

pcall(function()
    local chatEvents = ReplicatedStorage:FindFirstChild("DefaultChatSystemChatEvents")
    if chatEvents and chatEvents:FindFirstChild("OnMessageDoneFiltering") then
        local conn = chatEvents.OnMessageDoneFiltering.OnClientEvent:Connect(function(data)
            if not isRunning then return end
            if data and data.Message then
                secretBossState.HandleChatMessage(tostring(data.Message))
            end
        end)
        table.insert(activeConnections, conn)
    end
end)

-- ===============================================================
-- ⚡ LẮNG NGHE SỰ KIỆN THỜI TIẾT TỨC THÌ & SĂN BOSS LIÊN TỤC
-- ===============================================================

-- Lắng nghe trực tiếp khi TextLabel thời tiết thay đổi Text (0ms phản hồi ngay khi thời tiết xuất hiện)
task.spawn(function()
    local wLabel = nil
    for _ = 1, 30 do
        if not isRunning then return end
        wLabel = secretBossState.GetWeatherLabel()
        if wLabel then break end
        task.wait(1.0)
    end

    if wLabel and isRunning then
        local function onWeatherChanged()
            if not isRunning then return end
            if not (Config.AutoChatSecretBoss or Config.AutoHuntBoss) then return end

            task.wait(0.15)
            local wIsland, wName = secretBossState.DetectWeather()
            if wIsland and wName ~= "Clear" then
                -- Kiểm tra danh sách boss mục tiêu (hỗ trợ case-insensitive và fallback toàn bộ)
                local targetBossInWeather = false
                local hasAnyBossConfigured = false
                for _, isTarget in pairs(Config.SecretBossTargets or {}) do
                    if isTarget == true then
                        hasAnyBossConfigured = true
                        break
                    end
                end

                if not hasAnyBossConfigured then
                    targetBossInWeather = true
                else
                    for _, b in ipairs(wIsland.bosses or {}) do
                        local bNameLow = b.name:lower()
                        for tName, isTgt in pairs(Config.SecretBossTargets or {}) do
                            if isTgt and (tName:lower() == bNameLow or bNameLow:find(tName:lower(), 1, true) or tName:lower():find(bNameLow, 1, true)) then
                                targetBossInWeather = true
                                break
                            end
                        end
                        if targetBossInWeather then break end
                    end
                end

                if targetBossInWeather then
                    -- Kiểm tra quyền ưu tiên
                    if Config.PrioritySystemEnabled and PriorityManager and PriorityManager.GetActiveTask then
                        local curTask = PriorityManager.GetActiveTask()
                        local bossPri = PriorityManager.GetTaskPriority("SecretBoss")
                        if curTask ~= "None" and curTask ~= "SecretBoss" and PriorityManager.GetTaskPriority(curTask) < bossPri then
                            if curTask == "TicketQuest" and ticketQuestState and (ticketQuestState.isInteracting or (ticketQuestState.IsFishingActive and ticketQuestState.IsFishingActive())) then
                                return
                            end
                        end
                    end

                    local root = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
                    local targetPos = secretBossState.standPos or wIsland.pos
                    local dist = root and (root.Position - targetPos).Magnitude or 9999
                    if secretBossState.currentMap ~= wIsland.islandName or dist > 150 then
                        secretBossState.Teleport(wIsland, wName)
                    end
                end
            end
        end

        table.insert(activeConnections, wLabel:GetPropertyChangedSignal("Text"):Connect(onWeatherChanged))
    end
end)

-- TỰ ĐỘNG QUÉT THỜI TIẾT & CHAT ĐỊNH KỲ (MỖI 1.5 GIÂY) ĐỂ SĂN BOSS
task.spawn(function()
    while isRunning do
        task.wait(1.5)
        if (Config.AutoChatSecretBoss or Config.AutoHuntBoss) and isRunning then
            pcall(function()
                -- 1. Ưu tiên quét Thời tiết thực tế trong Game (HUD UI, Workspace, ReplicatedStorage)
                local wIsland, wName = secretBossState.DetectWeather()
                local isWeatherClear = (wIsland == nil) or (wName == "Clear")

                local targetBossInWeather = false
                if wIsland and not isWeatherClear then
                    local hasAnyBossConfigured = false
                    for _, isTarget in pairs(Config.SecretBossTargets or {}) do
                        if isTarget == true then
                            hasAnyBossConfigured = true
                            break
                        end
                    end

                    if not hasAnyBossConfigured then
                        targetBossInWeather = true
                    else
                        for _, b in ipairs(wIsland.bosses or {}) do
                            local bNameLow = b.name:lower()
                            for tName, isTgt in pairs(Config.SecretBossTargets or {}) do
                                if isTgt and (tName:lower() == bNameLow or bNameLow:find(tName:lower(), 1, true) or tName:lower():find(bNameLow, 1, true)) then
                                    targetBossInWeather = true
                                    break
                                end
                            end
                            if targetBossInWeather then break end
                        end
                    end
                end

                if wIsland and targetBossInWeather and not isWeatherClear then
                    -- KIỂM TRA PHÂN CẤP ĐỘ ƯU TIÊN (PRIORITY MANAGER)
                    if Config.PrioritySystemEnabled and PriorityManager and PriorityManager.GetActiveTask then
                        local curTask = PriorityManager.GetActiveTask()
                        local bossPri = PriorityManager.GetTaskPriority("SecretBoss")
                        if curTask ~= "None" and curTask ~= "SecretBoss" and PriorityManager.GetTaskPriority(curTask) < bossPri then
                            if curTask == "TicketQuest" and ticketQuestState and (ticketQuestState.isInteracting or (ticketQuestState.IsFishingActive and ticketQuestState.IsFishingActive())) then
                                return
                            end
                        end
                    end

                    -- Có Boss mục tiêu đang diễn ra theo thời tiết -> Bay qua đảo săn boss
                    local root = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
                    local targetPos = secretBossState.standPos or wIsland.pos
                    local dist = root and (root.Position - targetPos).Magnitude or 9999
                    if secretBossState.currentMap ~= wIsland.islandName or dist > 150 then
                        secretBossState.Teleport(wIsland, wName)
                    end
                else
                    -- Không có thời tiết boss mục tiêu (Thời tiết Clear, hoặc thời tiết đảo đó không chọn săn)
                    -- 2. Kiểm tra xem có Boss Chat nào còn trong thời gian hiệu lực không (tối đa 300 giây)
                    local chatSpawn = nil
                    if secretBossState.activeChatBoss then
                        if (tick() - (secretBossState.activeChatBoss.time or 0)) < 300 then
                            local isTarget = false
                            if secretBossState.activeChatBoss.island then
                                for _, b in ipairs(secretBossState.activeChatBoss.island.bosses) do
                                    if Config.SecretBossTargets[b.name] then
                                        isTarget = true
                                        break
                                    end
                                end
                            end
                            if isTarget then
                                chatSpawn = secretBossState.activeChatBoss
                            else
                                secretBossState.activeChatBoss = nil
                            end
                        else
                            secretBossState.activeChatBoss = nil
                        end
                    end

                    if chatSpawn and chatSpawn.island then
                        -- Có boss xuất hiện theo thông báo chat còn hiệu lực
                        local root = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
                        local targetPos = secretBossState.standPos or chatSpawn.island.pos
                        local dist = root and (root.Position - targetPos).Magnitude or 9999
                        if secretBossState.currentMap ~= chatSpawn.island.islandName or dist > 150 then
                            secretBossState.Teleport(chatSpawn.island, chatSpawn.bossName)
                        end
                    else
                        -- Hoàn toàn KHÔNG CÓ BOSS MỤC TIÊU NÀO ĐANG HOẠT ĐỘNG (Thời tiết Clear / Hết Boss)
                        if secretBossState.active then
                            secretBossState.active = false
                            secretBossState.currentMap = nil
                            secretBossState.targetIsland = nil
                            secretBossState.standPos = nil
                            if statusLabelSecretBoss and statusLabelSecretBoss.Set then
                                statusLabelSecretBoss.Set("Thời tiết Clear / Hết Boss. Đang ở vị trí Farm...")
                            end
                        end

                        -- Nếu có bật tự động về Home Farm Spot
                        if Config.ReturnToHomeWhenClear and Config.HomeFarmSpot then
                            secretBossState.ReturnToHome()
                        end
                    end
                end
            end)
        end
    end
end)


table.insert(activeConnections, UserInputService.JumpRequest:Connect(function()
    if Config.InfiniteJump and isRunning then
        local hum = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
        if hum then hum:ChangeState(Enum.HumanoidStateType.Jumping) end
    end
end))

-- ===============================================================
-- 🛡️ HỆ THỐNG CHỐNG VĂNG GAME ĐA TẦNG (ANTI-AFK & AUTO-REJOIN)
-- ===============================================================

-- Tầng 1: Vô hiệu hóa bộ đếm Idled 20 phút mặc định của Roblox bằng getconnections
pcall(function()
    if getconnections then
        for _, conn in ipairs(getconnections(LocalPlayer.Idled)) do
            if conn.Disable then
                conn:Disable()
            elseif conn.Disconnect then
                conn:Disconnect()
            end
        end
    end
end)

-- Tầng 2: Bắt sự kiện Idled dự phòng và giả lập thao tác chuột ảo
pcall(function()
    table.insert(activeConnections, LocalPlayer.Idled:Connect(function()
        if Config.AntiAFK and isRunning then
            pcall(function()
                local vu = game:GetService("VirtualUser")
                if vu then
                    vu:CaptureController()
                    vu:ClickButton2(Vector2.new(0, 0))
                    vu:Button2Down(Vector2.new(0, 0))
                    task.wait(0.05)
                    vu:Button2Up(Vector2.new(0, 0))
                end
            end)
            pcall(function()
                local vim = game:GetService("VirtualInputManager")
                if vim then
                    vim:SendMouseButtonEvent(15, 15, 0, true, game, 1)
                    task.wait(0.02)
                    vim:SendMouseButtonEvent(15, 15, 0, false, game, 1)
                end
            end)
        end
    end))
end)

-- Tầng 3: Nhịp tim chủ động (Active Keep-Alive Pulse) cứ mỗi 5 phút gửi micro-action để reset thời gian idle của Roblox
task.spawn(function()
    while isRunning do
        task.wait(300)
        if isRunning and Config.AntiAFK then
            pcall(function()
                local vu = game:GetService("VirtualUser")
                if vu then
                    vu:CaptureController()
                    vu:ClickButton2(Vector2.new(50, 50))
                end
            end)
            pcall(function()
                local vim = game:GetService("VirtualInputManager")
                if vim then
                    vim:SendMouseButtonEvent(20, 20, 0, true, game, 1)
                    task.wait(0.02)
                    vim:SendMouseButtonEvent(20, 20, 0, false, game, 1)
                end
            end)
        end
    end
end)

-- TỰ ĐỘNG ĐỔI SERVER TÌM TAOIST / GOD SPIRIT / MAOSHAN
task.spawn(function()
    task.wait(6)
    local lastHopAttempt = 0
    secretBossState.npcHopVisited = (pendingNPCHopData and pendingNPCHopData.Visited) or {}
    if not table.find(secretBossState.npcHopVisited, game.JobId) then
        table.insert(secretBossState.npcHopVisited, game.JobId)
    end

    while isRunning do
        task.wait(3.0)
        if isRunning and (Config.AutoServerHopTaoist or Config.AutoServerHopGod or Config.AutoServerHopMaoshan) then
            local hopOk, hopErr = pcall(function()
                local now = tick()
                if (now - lastHopAttempt) < 10 then return end

                local enabledTargets = {}
                local foundTargets = {}

                -- 1. Kiểm tra Taoist nếu đang bật
                if Config.AutoServerHopTaoist then
                    table.insert(enabledTargets, "Đạo Sĩ (Taoist)")
                    local tInst, tName = secretBossState.ScanForTaoistNPC()
                    if tInst then
                        table.insert(foundTargets, tostring(tName or "Đạo Sĩ (Taoist)"))
                        pcall(function()
                            secretBossState.SendNPCDetectionAlert("Taoist", tName or "Đạo Sĩ (Taoist)", tInst)
                        end)
                    end
                end

                -- 2. Kiểm tra Maoshan nếu đang bật
                if Config.AutoServerHopMaoshan then
                    table.insert(enabledTargets, "Đạo Sĩ Maoshan")
                    local mInst, mName = secretBossState.ScanForMaoshanNPC()
                    if mInst then
                        table.insert(foundTargets, tostring(mName or "Đạo Sĩ Maoshan"))
                        pcall(function()
                            secretBossState.SendNPCDetectionAlert("Maoshan", mName or "Đạo Sĩ Mao Sơn", mInst)
                        end)
                    end
                end

                -- 3. Kiểm tra Thần Linh (God Spirit) nếu đang bật
                if Config.AutoServerHopGod then
                    table.insert(enabledTargets, "Thần Linh (God Spirit)")
                    local sp = secretBossState.ScanForGodSpirit()
                    if sp then
                        table.insert(foundTargets, "Thần Linh (God Spirit)")
                        pcall(function()
                            secretBossState.SendNPCDetectionAlert("GodSpirit", "Thần Linh (God Spirit)", sp)
                        end)
                    end
                end

                -- Nếu tìm được BẤT KỲ mục tiêu nào trong số các mục tiêu đã bật:
                if #foundTargets > 0 then
                    local foundStr = table.concat(foundTargets, ", ")
                    ShowNotification("ĐÃ TÌM THẤY!", "Đã phát hiện " .. foundStr .. " trong server này! Đã dừng đổi server.", "SUCCESS", 8)
                    secretBossState.ClearNPCHopState()
                    return
                end

                -- Nếu KHÔNG tìm thấy bất kỳ mục tiêu nào trong các mục tiêu đang bật -> ĐỔI SERVER TIẾP
                lastHopAttempt = now
                local targetListStr = table.concat(enabledTargets, " / ")
                ShowNotification("Đổi Server Tìm NPC", "Server này không có " .. targetListStr .. ". Đang đổi server tiếp theo...", "WARN", 4)

                secretBossState.SaveNPCHopState(secretBossState.npcHopVisited)
                QueueScriptOnTeleport()
                task.wait(1.5)
                if secretBossState.ServerHop then
                    secretBossState.ServerHop(secretBossState.npcHopVisited)
                end
            end)
            if not hopOk then
                warn("[AutoServerHop] Lỗi trong vòng lặp tìm NPC:", tostring(hopErr))
            end
        end
    end
end)

-- Vòng lặp nền giám sát Taoist & Maoshan liên tục trong server để gửi Webhook / Telegram kể cả khi không bật Server Hop
task.spawn(function()
    while isRunning do
        task.wait(7.0)
        if isRunning and ((Config.WebhookEnabled and Config.WebhookNotifyNPC) or (Config.TelegramEnabled and Config.TelegramNotifyNPC)) then
            pcall(function()
                local tInst, tName = secretBossState.ScanForTaoistNPC()
                if tInst then
                    secretBossState.SendNPCDetectionAlert("Taoist", tName or "Đạo Sĩ (Taoist)", tInst)
                end
                local mInst, mName = secretBossState.ScanForMaoshanNPC()
                if mInst then
                    secretBossState.SendNPCDetectionAlert("Maoshan", mName or "Đạo Sĩ Mao Sơn", mInst)
                end
            end)
        end
    end
end)

do
    local espFolder = Instance.new("Folder")
    espFolder.Name = "IdenticalESP"
    espFolder.Parent = Workspace
    table.insert(cleanUpInstances, espFolder)

local activeESP = {}

local fishRingAnchor = Instance.new("Part")
fishRingAnchor.Name = "FishRingAnchor"
fishRingAnchor.Size = Vector3.new(0.5, 0.5, 0.5)
fishRingAnchor.Transparency = 1
fishRingAnchor.CanCollide = false
fishRingAnchor.Anchored = true
fishRingAnchor.Parent = Workspace
table.insert(cleanUpInstances, fishRingAnchor)

local fishRingAdornment = Instance.new("CylinderHandleAdornment")
fishRingAdornment.Name = "FishRingAdornment"
fishRingAdornment.Adornee = fishRingAnchor
fishRingAdornment.AlwaysOnTop = true
fishRingAdornment.ZIndex = 5
fishRingAdornment.Radius = 6
fishRingAdornment.InnerRadius = 5.2
fishRingAdornment.Height = 0.2
fishRingAdornment.Color3 = Color3.fromRGB(255, 45, 45)
fishRingAdornment.Transparency = 0.2
fishRingAdornment.CFrame = CFrame.Angles(math.rad(90), 0, 0)
fishRingAdornment.Visible = false
fishRingAdornment.Parent = fishRingAnchor

local fishRingBillboard = Instance.new("BillboardGui")
fishRingBillboard.Name = "FishRingBillboard"
fishRingBillboard.Size = UDim2.new(0, 280, 0, 32)
fishRingBillboard.StudsOffset = Vector3.new(0, 2.5, 0)
fishRingBillboard.AlwaysOnTop = true
fishRingBillboard.Adornee = fishRingAnchor
fishRingBillboard.Parent = fishRingAnchor

local fishRingText = Instance.new("TextLabel", fishRingBillboard)
fishRingText.Size = UDim2.new(1, 0, 1, 0)
fishRingText.BackgroundTransparency = 1
fishRingText.TextColor3 = Color3.fromRGB(255, 230, 90)
fishRingText.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
fishRingText.TextStrokeTransparency = 0.2
fishRingText.Font = Enum.Font.GothamBold
fishRingText.TextSize = 13
fishRingText.Text = ""
fishRingText.Visible = false

function secretBossState.GetNearestIslandName(pos)
    if not pos then return "Đại Dương" end
    local nearestDist = math.huge
    local nearestName = "Đại Dương"

    local spFolder = Workspace:FindFirstChild("Spawnpoint")
    if spFolder then
        for _, sp in ipairs(spFolder:GetChildren()) do
            if sp:IsA("BasePart") then
                local d = (pos - sp.Position).Magnitude
                if d < nearestDist then
                    nearestDist = d
                    nearestName = sp.Name
                end
            end
        end
    end

    if nearestDist > 1200 and secretBossDatabase then
        for _, entry in ipairs(secretBossDatabase) do
            if entry.pos then
                local d = (pos - entry.pos).Magnitude
                if d < nearestDist then
                    nearestDist = d
                    nearestName = entry.islandName:gsub("%s*%b()", "")
                end
            end
        end
    end
    return nearestName
end

local function ResolveBestPart(instance)
    if not instance then return nil end
    if instance:IsA("BasePart") then return instance end
    if instance:IsA("Model") then
        return instance.PrimaryPart
            or instance:FindFirstChild("HumanoidRootPart")
            or instance:FindFirstChild("Torso")
            or instance:FindFirstChild("UpperTorso")
            or instance:FindFirstChild("Stick")
            or instance:FindFirstChild("Handle")
            or instance:FindFirstChild("Part")
            or instance:FindFirstChildWhichIsA("BasePart", true)
    end
    return instance:FindFirstChildWhichIsA("BasePart", true)
end

local function AddESP(instance, name, espCategory, color, icon, explicitPart, customData)
    if not instance or activeESP[instance] then return end
    local isEnabled = Config["ESP_" .. espCategory]
    if not isEnabled then return end

    local part = explicitPart or ResolveBestPart(instance)
    if not part then return end

    local bb = Instance.new("BillboardGui")
    bb.Name = "ESP_" .. name
    bb.Size = UDim2.new(0, 195, 0, 36)
    bb.StudsOffset = Vector3.new(0, 2.8, 0)
    bb.AlwaysOnTop = true
    bb.Adornee = part
    bb.Enabled = true
    bb.Parent = espFolder

    local f = Instance.new("Frame", bb)
    f.Size = UDim2.new(1, 0, 1, 0)
    f.BackgroundColor3 = Color3.fromRGB(15, 15, 24)
    f.BackgroundTransparency = 0.15
    Instance.new("UICorner", f).CornerRadius = UDim.new(0, 5)

    local s = Instance.new("UIStroke", f)
    s.Color = color or Colors.PurpleAccent
    s.Thickness = 1.2

    local titleLbl = Instance.new("TextLabel", f)
    titleLbl.Size = UDim2.new(1, -8, 0, 17)
    titleLbl.Position = UDim2.new(0, 4, 0, 2)
    titleLbl.BackgroundTransparency = 1
    titleLbl.Font = Enum.Font.GothamBold
    titleLbl.Text = (icon or "") .. " " .. name
    titleLbl.TextColor3 = color or Colors.TextWhite
    titleLbl.TextSize = 10
    titleLbl.TextTruncate = Enum.TextTruncate.AtEnd

    local subLbl = Instance.new("TextLabel", f)
    subLbl.Size = UDim2.new(1, -8, 0, 14)
    subLbl.Position = UDim2.new(0, 4, 0, 18)
    subLbl.BackgroundTransparency = 1
    subLbl.Font = Enum.Font.Gotham
    subLbl.Text = "Đang tải dữ liệu..."
    subLbl.TextColor3 = Color3.fromRGB(195, 205, 220)
    subLbl.TextSize = 8.5
    subLbl.TextTruncate = Enum.TextTruncate.AtEnd

    activeESP[instance] = {
        gui = bb,
        frame = f,
        stroke = s,
        titleLbl = titleLbl,
        subLbl = subLbl,
        part = part,
        instance = instance,
        espCategory = espCategory,
        name = name,
        icon = icon or "",
        customData = customData
    }
end

local function RemoveESP(instance)
    local data = activeESP[instance]
    if data then
        if data.gui and data.gui.Parent then data.gui:Destroy() end
        activeESP[instance] = nil
    end
end

-- VÒNG LẶP KHÁM PHÁ ĐỐI TƯỢNG ESP (CHỈ QUÉT KHI CÓ ÍT NHẤT 1 TÙY CHỌN ESP ĐƯỢC BẬT)
task.spawn(function()
    while isRunning do
        task.wait(1.2)
        local anyESP = Config.ESP_Players or Config.ESP_SecretRod or Config.ESP_GodSpirit or Config.ESP_Boats or Config.ESP_Boss or Config.ESP_Taoist or Config.ESP_Maoshan
        if anyESP then
            pcall(function()
                if Config.ESP_Players then
                    for _, p in ipairs(Players:GetPlayers()) do
                        if p ~= LocalPlayer and p.Character then
                            local hrp = p.Character:FindFirstChild("HumanoidRootPart") or p.Character:FindFirstChild("Head")
                            if hrp then
                                AddESP(p.Character, p.DisplayName, "Players", Colors.PurpleAccent, "👤", hrp, {player = p})
                            end
                        end
                    end
                end

                if Config.ESP_SecretRod and Workspace:FindFirstChild("SecretRod") then
                    for _, r in ipairs(Workspace.SecretRod:GetChildren()) do
                        AddESP(r, r.Name, "SecretRod", Colors.AccentYellow, "🌟")
                    end
                end

                if Config.ESP_GodSpirit then
                    local sp = secretBossState.GetGodSpirit()
                    if sp then
                        AddESP(sp, "God Spirit", "GodSpirit", Colors.AccentGreen, "⛩️")
                    end
                end

                if Config.ESP_Boats and Workspace:FindFirstChild("Boat") then
                    for _, b in ipairs(Workspace.Boat:GetChildren()) do
                        AddESP(b, b.Name, "Boats", Colors.AccentBlue, "⛵")
                    end
                end

                if Config.ESP_Boss then
                    if Workspace:FindFirstChild("BossSetUp") then
                        for _, b in ipairs(Workspace.BossSetUp:GetChildren()) do
                            AddESP(b, b.Name, "Boss", Colors.AccentRed, "👹")
                        end
                    end
                    if Workspace:FindFirstChild("Fishes") then
                        for _, f in ipairs(Workspace.Fishes:GetChildren()) do
                            local isBoss = f:GetAttribute("Boss") == true
                            local fName = f:GetAttribute("FishName") or ""
                            if not isBoss and fName ~= "" and (Config.SecretBossTargets[fName] or (secretBossLookup[fName:lower()] and Config.SecretBossTargets[secretBossLookup[fName:lower()]])) then
                                isBoss = true
                            end
                            if isBoss then
                                local targetPart = nil
                                if f:FindFirstChild("Model") and f.Model:IsA("Model") then
                                    targetPart = f.Model.PrimaryPart or f.Model:FindFirstChildWhichIsA("BasePart", true)
                                elseif f:FindFirstChild("Buoy") and f.Buoy:IsA("BasePart") then
                                    targetPart = f.Buoy
                                end
                                if not targetPart then
                                    for _, c in ipairs(f:GetChildren()) do
                                        if c.Name:find("_PlayerHealth") then
                                            local uid = c.Name:match("^(%d+)_PlayerHealth")
                                            if uid then
                                                local pl = Players:GetPlayerByUserId(tonumber(uid))
                                                local plChar = pl and pl.Character
                                                if plChar then
                                                    targetPart = plChar:FindFirstChild("Buoy") or plChar:FindFirstChild("HumanoidRootPart")
                                                    break
                                                end
                                            end
                                        end
                                    end
                                end
                                if targetPart then
                                    AddESP(f, fName ~= "" and fName or f.Name, "Boss", Colors.AccentRed, "👹", targetPart, {fish = f})
                                end
                            end
                        end
                    end
                end

                if Config.ESP_Taoist then
                    local tInst, tName, tCat, tCol, tIcon = secretBossState.GetTaoist()
                    if tInst then
                        AddESP(tInst, tName or "Đạo Sĩ (Taoist)", "Taoist", tCol or Colors.AccentOrange, tIcon or "📜")
                    end
                end

                if Config.ESP_Maoshan then
                    local mInst, mName, mCat, mCol, mIcon = secretBossState.GetMaoshan()
                    if mInst then
                        AddESP(mInst, mName or "Đạo Sĩ Maoshan", "Maoshan", mCol or Colors.PurplePrimary, mIcon or "✨")
                    end
                end
            end)
        else
            -- Nếu tắt hết tất cả ESP -> Xóa toàn bộ GUI tồn đọng và không quét bất kỳ thứ gì
            if next(activeESP) then
                for inst in pairs(activeESP) do
                    RemoveESP(inst)
                end
            end
        end
    end
end)

table.insert(activeConnections, RunService.RenderStepped:Connect(function()
    if not isRunning then return end

    if next(activeESP) then
        local camPos = Camera.CFrame.Position
        for inst, data in pairs(activeESP) do
            local isEnabled = Config["ESP_" .. data.espCategory]
            if not inst.Parent or not data.part or not data.part.Parent or not isEnabled then
                RemoveESP(inst)
            else
                data.gui.Enabled = true
                local dist = math.floor((camPos - data.part.Position).Magnitude)
                local cat = data.espCategory

                if cat == "Players" then
                    local p = data.customData and data.customData.player
                    if not p or not p.Parent or not p.Character or p.Character ~= inst then
                        RemoveESP(inst)
                    else
                        local hum = inst:FindFirstChildOfClass("Humanoid")
                        local curHp = hum and math.floor(hum.Health) or 0
                        local maxHp = hum and math.floor(hum.MaxHealth) or 100
                        local isFishing = inst:GetAttribute("Fishing") == true
                        local isMinigame = inst:GetAttribute("Minigame") == true
                        local isSwimming = inst:GetAttribute("Swimming") == true

                        local rodVal = "Không cầm"
                        local pFolder = ReplicatedStorage:FindFirstChild("Data") and ReplicatedStorage.Data:FindFirstChild(tostring(p.UserId))
                        if pFolder and pFolder:FindFirstChild("FishingRod") and pFolder.FishingRod.Value ~= "" then
                            rodVal = pFolder.FishingRod.Value
                        end

                        local act = "💤 Rảnh"
                        if curHp <= 0 then act = "💀 Đã chết"
                        elseif isMinigame then act = "⚡ Kéo cá"
                        elseif isFishing then act = "🎣 Thả cần"
                        elseif isSwimming then act = "🏊 Bơi"
                        end

                        data.titleLbl.Text = string.format("👤 %s  •  %dm", p.DisplayName, dist)
                        data.subLbl.Text = string.format("❤️ %d/%d HP | 🎣 %s | %s", curHp, maxHp, rodVal, act)

                        if isMinigame then
                            data.stroke.Color = Color3.fromRGB(255, 170, 40)
                        elseif isFishing then
                            data.stroke.Color = Color3.fromRGB(60, 220, 255)
                        else
                            data.stroke.Color = Colors.PurpleAccent
                        end
                    end
                elseif cat == "Boss" then
                    local fish = data.customData and data.customData.fish
                    if fish and fish.Parent then
                        local fName = fish:GetAttribute("FishName") or data.name or "Secret Boss"
                        local maxHp = fish:GetAttribute("MaxHealth") or 0
                        local curHp = typeof(fish.Value) == "number" and fish.Value or 0
                        local hpPct = (maxHp > 0) and math.floor((curHp / maxHp) * 100) or 0

                        local hooker = nil
                        local contrib = fish:FindFirstChild("PlayerContribution")
                        if contrib then
                            local ch = contrib:GetChildren()
                            if #ch > 0 then hooker = ch[1].Name end
                        end
                        if not hooker then
                            for _, c in ipairs(fish:GetChildren()) do
                                if c.Name:find("_PlayerHealth") then
                                    local uid = c.Name:match("^(%d+)_PlayerHealth")
                                    if uid then
                                        local pl = Players:GetPlayerByUserId(tonumber(uid))
                                        if pl then hooker = pl.DisplayName break end
                                    end
                                end
                            end
                        end

                        data.titleLbl.Text = string.format("👹 %s [TRÙM]  •  %dm", fName, dist)
                        if hooker then
                            data.subLbl.Text = string.format("❤️ HP: %d/%d (%d%%) | 🎯 %s", math.floor(curHp), maxHp, hpPct, hooker)
                        else
                            data.subLbl.Text = string.format("❤️ HP: %d/%d (%d%%)", math.floor(curHp), maxHp, hpPct)
                        end
                        data.stroke.Color = Color3.fromRGB(255, 50, 50)
                    else
                        data.titleLbl.Text = string.format("👹 %s  •  %dm", data.name, dist)
                        data.subLbl.Text = "⚔️ Khu vực triệu hồi & chiến đấu Enzo"
                        data.stroke.Color = Color3.fromRGB(255, 60, 60)
                    end
                elseif cat == "SecretRod" then
                    local isl = secretBossState.GetNearestIslandName(data.part.Position)
                    data.titleLbl.Text = string.format("🌟 %s  •  %dm", data.name, dist)
                    data.subLbl.Text = string.format("📍 Đảo: %s | ⭐ Cần Bí Mật", isl)
                    data.stroke.Color = Colors.AccentYellow
                elseif cat == "GodSpirit" then
                    local isl = secretBossState.GetNearestIslandName(data.part.Position)
                    data.titleLbl.Text = string.format("⛩️ Thần Linh (God Spirit)  •  %dm", dist)
                    data.subLbl.Text = string.format("📍 Đảo: %s | 🙏 Cầu Nguyện", isl)
                    data.stroke.Color = Colors.AccentGreen
                elseif cat == "Taoist" or cat == "Maoshan" then
                    local isl = secretBossState.GetNearestIslandName(data.part.Position)
                    local isMao = cat == "Maoshan"
                    data.titleLbl.Text = string.format("%s %s  •  %dm", isMao and "✨" or "📜", data.name, dist)
                    data.subLbl.Text = string.format("📍 Đảo: %s | 📜 Đạo Sĩ Bí Kíp", isl)
                elseif cat == "Boats" then
                    local b = data.instance
                    local driverName = "Trống"
                    if b and b:IsA("Model") then
                        local seat = b:FindFirstChildWhichIsA("VehicleSeat", true)
                        if seat and seat.Occupant and seat.Occupant.Parent then
                            local p = Players:GetPlayerFromCharacter(seat.Occupant.Parent)
                            if p then driverName = p.DisplayName else driverName = seat.Occupant.Parent.Name end
                        end
                    end
                    data.titleLbl.Text = string.format("⛵ %s  •  %dm", data.name, dist)
                    data.subLbl.Text = string.format("👤 Người lái: %s", driverName)
                    data.stroke.Color = Colors.AccentBlue
                else
                    data.titleLbl.Text = string.format("%s %s  •  %dm", data.icon, data.name, dist)
                    data.subLbl.Text = ""
                end
            end
        end
    end

    if Config.HideOverheadNames then
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= LocalPlayer and p.Character then
                local hum = p.Character:FindFirstChildOfClass("Humanoid")
                if hum and hum.DisplayDistanceType ~= Enum.HumanoidDisplayDistanceType.None then
                    hum.DisplayDistanceType = Enum.HumanoidDisplayDistanceType.None
                end
                for _, d in ipairs(p.Character:GetDescendants()) do
                    if d:IsA("BillboardGui") and d.Name:sub(1, 4) ~= "ESP_" then
                        if d.Enabled then d.Enabled = false end
                    end
                end
            end
        end
    end

    if Config.FishRedRing then
        local fishPos = nil
        local char = LocalPlayer.Character
        local hrp = char and (char:FindFirstChild("HumanoidRootPart") or char:FindFirstChild("Torso"))
        local hrpPos = hrp and hrp.Position or Camera.CFrame.Position
        local fishID = LocalPlayer:GetAttribute("FishID")
        local uid = tostring(LocalPlayer.UserId)
        local pName = LocalPlayer.Name

        -- 1. Tim truc tiep tu Workspace.Fishes
        local fishesFolder = Workspace:FindFirstChild("Fishes")
        if fishesFolder then
            local targetFish = nil
            if fishID and fishID ~= "" then
                targetFish = fishesFolder:FindFirstChild(fishID)
            end
            if not targetFish then
                for _, f in ipairs(fishesFolder:GetChildren()) do
                    if f:FindFirstChild(uid .. "_PlayerHealth")
                       or (f:FindFirstChild("PlayerContribution") and f.PlayerContribution:FindFirstChild(pName)) then
                        targetFish = f
                        break
                    end
                end
            end
            if targetFish then
                if targetFish:FindFirstChild("Buoy") and targetFish.Buoy:IsA("BasePart") then
                    fishPos = targetFish.Buoy.Position
                elseif targetFish:FindFirstChild("Model") and targetFish.Model:IsA("Model") then
                    fishPos = targetFish.Model:GetPivot().Position
                elseif targetFish:IsA("Model") then
                    fishPos = targetFish:GetPivot().Position
                else
                    local bp = targetFish:FindFirstChildWhichIsA("BasePart", true)
                    if bp then fishPos = bp.Position end
                end
            end
        end

        -- 2. Tim phao / day cau tu Character
        if not fishPos and char then
            local buoy = char:FindFirstChild("Buoy", true) or char:FindFirstChild("Bobber", true) or char:FindFirstChild("Hook", true)
            if buoy and buoy:IsA("BasePart") and (buoy.Position - hrpPos).Magnitude > 3 then
                fishPos = buoy.Position
            else
                local bestDist = 3
                for _, d in ipairs(char:GetDescendants()) do
                    if (d:IsA("Beam") or d:IsA("RopeConstraint") or d:IsA("RodConstraint")) and d.Attachment0 and d.Attachment1 then
                        local p0 = d.Attachment0.WorldPosition
                        local p1 = d.Attachment1.WorldPosition
                        local dist0 = (p0 - hrpPos).Magnitude
                        local dist1 = (p1 - hrpPos).Magnitude
                        local farAtt = dist1 > dist0 and d.Attachment1 or d.Attachment0
                        local farDist = math.max(dist0, dist1)
                        if farDist > bestDist then
                            bestDist = farDist
                            fishPos = farAtt.WorldPosition
                        end
                    end
                end
            end
        end

        -- 3. Tim phao trong Workspace
        if not fishPos then
            local wsBuoy = Workspace:FindFirstChild("Buoy") or Workspace:FindFirstChild("Bobber")
            if wsBuoy and wsBuoy:IsA("BasePart") and (wsBuoy.Position - hrpPos).Magnitude > 3 then
                fishPos = wsBuoy.Position
            end
        end

        -- Dam bao anchor va adornment ton tai trong Workspace
        if fishRingAnchor.Parent ~= Workspace then
            fishRingAnchor.Parent = Workspace
        end
        if fishRingAdornment.Adornee ~= fishRingAnchor then
            fishRingAdornment.Adornee = fishRingAnchor
        end
        if fishRingBillboard.Adornee ~= fishRingAnchor then
            fishRingBillboard.Adornee = fishRingAnchor
        end

        local pg = LocalPlayer:FindFirstChild("PlayerGui")
        local isHooked = char and (char:GetAttribute("Fishing") == true or char:GetAttribute("Minigame") == true)
        local fUI = pg and pg:FindFirstChild("MainGui") and pg.MainGui:FindFirstChild("Fishing")
        local isRodActive = char and (char:GetAttribute("Type") == "Fishing Rod") and fishPos and ((fishPos - hrpPos).Magnitude > 3)
        local shouldShow = fishPos and (isHooked or (fUI and fUI.Visible) or (fishID and fishID ~= "") or isRodActive)

        if shouldShow then
            fishRingAnchor.Position = fishPos
            fishRingAdornment.Visible = true

            local isPulling = (char and char:GetAttribute("Minigame") == true) or (fUI and fUI.Visible)
            fishRingAdornment.Color3 = isPulling and Color3.fromRGB(255, 45, 45) or Color3.fromRGB(255, 175, 45)

            if Config.ShowFishWeightRing then
                local fName = GetCurrentHookedFishName()
                local fWeight = char and (char:GetAttribute("FishWeight") or char:GetAttribute("Weight"))
                local fMutation = char and (char:GetAttribute("FishMutation") or char:GetAttribute("Mutation"))
                local curHp, maxHp = nil, nil

                if fishesFolder then
                    local f = (fishID and fishesFolder:FindFirstChild(fishID))
                    if not f then
                        for _, ch in ipairs(fishesFolder:GetChildren()) do
                            if ch:FindFirstChild(uid .. "_PlayerHealth") then f = ch break end
                        end
                    end
                    if f then
                        if not fName or fName == "" then fName = f:GetAttribute("FishName") end
                        maxHp = f:GetAttribute("MaxHealth")
                        if typeof(f.Value) == "number" then curHp = f.Value end
                    end
                end

                if not fWeight and fUI then
                    local wLabel = fUI:FindFirstChild("Weight", true) or fUI:FindFirstChild("FishWeight", true)
                    if wLabel and wLabel:IsA("TextLabel") and wLabel.Text ~= "" then
                        fWeight = wLabel.Text
                    end
                end

                local displayParts = {}
                if fName and fName ~= "" then table.insert(displayParts, tostring(fName)) end
                if fMutation and tostring(fMutation) ~= "" then table.insert(displayParts, "[" .. tostring(fMutation) .. "]") end
                if fWeight then
                    local wStr = tostring(fWeight)
                    if tonumber(wStr) then wStr = string.format("%.1f kg", tonumber(wStr)) end
                    table.insert(displayParts, "(" .. wStr .. ")")
                end
                if curHp and maxHp and maxHp > 0 then
                    table.insert(displayParts, string.format("• ❤️ %d/%d", math.floor(curHp), maxHp))
                end

                if #displayParts > 0 then
                    fishRingText.Text = table.concat(displayParts, " ")
                    fishRingText.Visible = true
                elseif isPulling then
                    fishRingText.Text = "⚡ Đang Kéo Cá"
                    fishRingText.Visible = true
                elseif isHooked then
                    fishRingText.Text = "🎣 Đang Thả Cần"
                    fishRingText.Visible = true
                else
                    fishRingText.Visible = false
                end
            else
                fishRingText.Visible = false
            end
        else
            fishRingAdornment.Visible = false
            fishRingText.Visible = false
        end
    else
        if fishRingAdornment.Visible then fishRingAdornment.Visible = false end
        if fishRingText.Visible then fishRingText.Visible = false end
    end
end))
end

table.insert(activeConnections, UserInputService.InputBegan:Connect(function(input, gpe)
    if gpe then return end
    if input.KeyCode == Enum.KeyCode.RightControl or input.KeyCode == Config.UIKeybind then
        if ToggleUiVisibility then ToggleUiVisibility() end
    elseif input.KeyCode == Enum.KeyCode.End or input.KeyCode == Config.StopKeybind then
        UnloadScript()
    end
end))

secretBossState.lastTrackedWeather = nil
secretBossState.weatherMonitorStarted = false

function secretBossState.StartWeatherMonitor()
    if secretBossState.weatherMonitorStarted then return end
    secretBossState.weatherMonitorStarted = true

    task.spawn(function()
        task.wait(4.0)
        local initEntry, initWeather = secretBossState.DetectWeather()
        local initNorm = (initWeather and initWeather ~= "" and initWeather ~= "Clear") and initWeather or "Clear"
        secretBossState.lastTrackedWeather = initNorm

        if initNorm ~= "Clear" and Config.NtfyEnabled and Config.NtfyAlertWeatherChange then
            secretBossState.SendWeatherNtfyAlert(initNorm, initEntry, true)
        end

        while isRunning do
            task.wait(3.5)
            if isRunning then
                pcall(function()
                    local detectedEntry, detectedWeather = secretBossState.DetectWeather()
                    local currentNorm = (detectedWeather and detectedWeather ~= "" and detectedWeather ~= "Clear") and detectedWeather or "Clear"

                    if secretBossState.lastTrackedWeather ~= nil and currentNorm ~= secretBossState.lastTrackedWeather then
                        task.wait(1.5)
                        local verifyEntry, verifyWeather = secretBossState.DetectWeather()
                        local verifiedNorm = (verifyWeather and verifyWeather ~= "" and verifyWeather ~= "Clear") and verifyWeather or "Clear"

                        if verifiedNorm == currentNorm and verifiedNorm ~= secretBossState.lastTrackedWeather then
                            secretBossState.lastTrackedWeather = verifiedNorm
                            if Config.NtfyEnabled and Config.NtfyAlertWeatherChange then
                                secretBossState.SendWeatherNtfyAlert(verifiedNorm, verifyEntry, false)
                            end
                        end
                    else
                        secretBossState.lastTrackedWeather = currentNorm
                    end
                end)
            end
        end
    end)
end

if secretBossState.StartWeatherMonitor then
    task.spawn(secretBossState.StartWeatherMonitor)
end

if secretBossState.CheckWeatherHopOnJoin then
    task.spawn(secretBossState.CheckWeatherHopOnJoin)
end

if secretBossState.CheckNPCHopOnJoin then
    task.spawn(secretBossState.CheckNPCHopOnJoin)
end

-- Nạp cài đặt Combo Kỹ Năng Thông Minh từ file local và đồng bộ UI
pcall(LoadSmartComboAndSyncUI)

-- Nạp trạng thái bật/tắt từng Secret Boss từ file local và đồng bộ UI toggle
pcall(LoadBossTargetsAndSyncUI)

-- Nạp cấu hình thông báo (ntfy, Webhook, Telegram) từ file local và đồng bộ UI
pcall(LoadNotificationsConfigAndSyncUI)

ShowNotification("VIỆT HOÁ V1.4", "Heavyweight Fishing đã cập nhật: Tự Động Tìm Server Thời Tiết, Totem Thời Tiết & Webhook!", "SUCCESS", 6)