local Fluent = loadstring(game:HttpGet("https://github.com/dawid-scripts/Fluent/releases/latest/download/main.lua"))()

local Window = Fluent:CreateWindow({
    Title = "Guest 1337 | Hitbox Auto Counter",
    SubTitle = "Q (Block) -> Lock Aim -> R (Stun)",
    TabWidth = 160,
    Size = UDim2.fromOffset(540, 390),
    Acrylic = true,
    Theme = "Dark",
    MinimizeKey = Enum.KeyCode.RightControl
})

local Tabs = {
    Main = Window:AddTab({ Title = "Hitbox Parry", Icon = "rbxassetid://10723415648" }),
    Movement = Window:AddTab({ Title = "Tốc Độ", Icon = "rbxassetid://10734950309" })
}

-- BIẾN CẤU HÌNH
local comboActive = false
local speedActive = false
local currentSpeed = 150
local lastComboTime = 0

-- BẢNG DỮ LIỆU HITBOX & WINDUP THEO TỪNG KILLER (DỰA TRÊN BẢNG)
local KillerData = {
    ["Jason"] = { range = 8.5, windup = 0.25 },     -- Hitbox dài 7.5 + đẩy về trước
    ["John Doe"] = { range = 7.5, windup = 0.2 },  -- Hitbox sát người
    ["1x1x1x1"] = { range = 7.5, windup = 0.2 },   -- Hitbox sát người
    ["c00lkidd"] = { range = 5.5, windup = 0.1 },  -- Ngắn hơn 1-2 stud, Windup cực nhanh 0.1s
    ["Slasher"] = { range = 5.5, windup = 0.12 },  -- Tương đương c00lkidd, chạy nhanh[cite: 1]
    ["Default"] = { range = 7.5, windup = 0.15 }   -- Mặc định cho các Killer/Guest khác
}

-- TOGGLES & SLIDERS
local ComboToggle = Tabs.Main:AddToggle("ComboToggle", {Title = "⚔️️ Auto Counter Combo (Q -> Lock -> R)", Default = false})
ComboToggle:OnChanged(function(Value)
    comboActive = Value
end)

local SpeedToggle = Tabs.Movement:AddToggle("SpeedToggle", {Title = "⚡ Steal Super Speed", Default = false})
SpeedToggle:OnChanged(function(Value)
    speedActive = Value
    if not speedActive then
        local char = game.Players.LocalPlayer.Character
        if char and char:FindFirstChild("Humanoid") then
            char.Humanoid.WalkSpeed = 16
        end
    end
end)

local SpeedSlider = Tabs.Movement:AddSlider("SpeedSlider", {
    Title = "Tốc Độ Di Chuyển",
    Default = 150,
    Min = 16,
    Max = 300,
    Rounding = 0
})
SpeedSlider:OnChanged(function(Value)
    currentSpeed = Value
end)

-- HÀM GIẢ LẬP BẤM PHÍM
local VirtualInputManager = game:GetService("VirtualInputManager")

local function pressKey(keyCode)
    VirtualInputManager:SendKeyEvent(true, keyCode, false, game)
    task.wait(0.02)
    VirtualInputManager:SendKeyEvent(false, keyCode, false, game)
end

-- HÀM LOCK AIM CHÍNH XÁC VÀO KILLER
local function lockAimToTarget(targetChar)
    local camera = workspace.CurrentCamera
    local myChar = game.Players.LocalPlayer.Character
    if targetChar and targetChar:FindFirstChild("HumanoidRootPart") and myChar and myChar:FindFirstChild("HumanoidRootPart") then
        local enemyPos = targetChar.HumanoidRootPart.Position
        
        -- Quay nhân vật về đối phương
        myChar.HumanoidRootPart.CFrame = CFrame.new(myChar.HumanoidRootPart.Position, Vector3.new(enemyPos.X, myChar.HumanoidRootPart.Position.Y, enemyPos.Z))
        
        -- Quay Camera về đối phương
        camera.CFrame = CFrame.new(camera.CFrame.Position, enemyPos)
    end
end

-- HÀM XÁC ĐỊNH LOẠI KILLER ĐANG ÁP SÁT
local function getKillerInfo(enemyChar)
    local name = enemyChar.Name
    for killerName, config in pairs(KillerData) do
        if string.find(string.lower(name), string.lower(killerName)) then
            return config
        end
    end
    return KillerData["Default"]
end

-- VÒNG LẶP XỬ LÝ (RUNSERVICE)
local RunService = game:GetService("RunService")
RunService.Stepped:Connect(function()
    local player = game.Players.LocalPlayer
    local myChar = player.Character
    
    -- Xử lý Super Speed
    if speedActive and myChar and myChar:FindFirstChild("Humanoid") then
        myChar.Humanoid.WalkSpeed = currentSpeed
        myChar.Humanoid.CustomPhysicalProperties = PhysicalProperties.new(0.7, 0.3, 0.5, 1, 1)
    end

    -- Xử lý Auto Parry Combo dựa theo Hitbox
    if comboActive and myChar and myChar:FindFirstChild("HumanoidRootPart") then
        if tick() - lastComboTime < 0.7 then return end

        local myPos = myChar.HumanoidRootPart.Position
        
        for _, otherPlayer in pairs(game.Players:GetPlayers()) do
            if otherPlayer ~= player and otherPlayer.Character and otherPlayer.Character:FindFirstChild("HumanoidRootPart") then
                local enemyChar = otherPlayer.Character
                local enemyPos = enemyChar.HumanoidRootPart.Position
                local distance = (myPos - enemyPos).Magnitude

                -- Lấy dữ liệu Hitbox & Windup của Killer đó[cite: 1]
                local kConfig = getKillerInfo(enemyChar)

                -- Tự động kích hoạt khi bước vào phạm vi Hitbox chuẩn + 1 stud dự phòng[cite: 1]
                if distance <= (kConfig.range + 1) then
                    local enemyTool = enemyChar:FindFirstChildOfClass("Tool")
                    if enemyTool then
                        lastComboTime = tick()
                        
                        -- 1. Nhấn Q ngay lập tức để Block/Parry
                        pressKey(Enum.KeyCode.Q)
                        
                        -- 2. Lock Aim ngay về Killer
                        lockAimToTarget(enemyChar)
                        
                        -- 3. Delay theo tốc độ đòn đánh (Windup) của Killer để đấm Stun (R)[cite: 1]
                        task.delay(kConfig.windup, function()
                            lockAimToTarget(enemyChar)
                            pressKey(Enum.KeyCode.R)
                        end)
                        
                        break
                    end
                end
            end
        end
    end
end)

Window:SelectTab(1)

Fluent:Notify({
    Title = "Hitbox Counter Active!",
    Content = "Đã cập nhật dữ liệu Hitbox của Jason, c00lkidd, Slasher...",
    Duration = 4
})
