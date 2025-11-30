-- HubServer (ServerScriptService/HubServer)
-- Server-side: xử lý RequestAttack và RequestTeleport (xác thực)
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local CollectionService = game:GetService("CollectionService")
local RunService = game:GetService("RunService")

-- Remotes folder + events
local remotesFolder = ReplicatedStorage:FindFirstChild("HubRemotes")
if not remotesFolder then
    remotesFolder = Instance.new("Folder", ReplicatedStorage)
    remotesFolder.Name = "HubRemotes"
end

local RequestAttack = remotesFolder:FindFirstChild("RequestAttack")
if not RequestAttack then
    RequestAttack = Instance.new("RemoteEvent", remotesFolder)
    RequestAttack.Name = "RequestAttack"
end

local RequestTeleport = remotesFolder:FindFirstChild("RequestTeleport")
if not RequestTeleport then
    RequestTeleport = Instance.new("RemoteEvent", remotesFolder)
    RequestTeleport.Name = "RequestTeleport"
end

-- Cấu hình server-side (tùy chỉnh)
local ATTACK_RANGE = 40            -- khoảng cách tối đa cho phép attack (studs)
local ATTACK_COOLDOWN = 0.4        -- cooldown server chấp nhận (giây)
local TELEPORT_MAX_DISTANCE = 2000 -- nếu muốn hạn chế TP theo khoảng cách
local TELEPORT_COOLDOWN = 2        -- cooldown giữa các request teleport (giây)

-- Theo dõi cooldown
local lastAttack = {}   -- userId -> tick()
local lastTeleport = {} -- userId -> tick()

-- Kiểm tra một Instance có phải enemy hợp lệ không
local function isValidEnemyInstance(inst)
    if not inst or not inst.Parent then return false end
    -- lấy model nếu gửi một part
    local model = inst
    if inst:IsA("BasePart") and inst.Parent and inst.Parent:FindFirstChildOfClass("Humanoid") then
        model = inst.Parent
    else
        -- nếu inst là Model, kiểm tra Humanoid
        if not inst:IsA("Model") then
            -- thử lấy parent model
            if inst.Parent and inst.Parent:IsA("Model") then
                model = inst.Parent
            end
        end
    end
    -- xác nhận model có Humanoid và được tag "Enemy"
    if model and model:IsA("Model") then
        local humanoid = model:FindFirstChildOfClass("Humanoid")
        if humanoid and CollectionService:HasTag(model, "Enemy") then
            return model, humanoid
        end
    end
    return nil, nil
end

-- Xử lý attack request từ client
RequestAttack.OnServerEvent:Connect(function(player, targetInstance)
    if typeof(player) ~= "Instance" or not player:IsA("Player") then return end
    if not targetInstance then return end

    local now = tick()
    local last = lastAttack[player.UserId] or 0
    if now - last < ATTACK_COOLDOWN then
        -- quá nhanh, ignore
        return
    end

    -- validate character & hrp
    local char = player.Character
    if not char or not char:FindFirstChild("HumanoidRootPart") then return end

    -- validate target
    local model, humanoid = isValidEnemyInstance(targetInstance)
    if not model or not humanoid then return end

    -- kiểm tra khoảng cách
    local hrp = char.HumanoidRootPart
    local primary = model:FindFirstChildWhichIsA("BasePart")
    if not primary then return end
    local dist = (hrp.Position - primary.Position).Magnitude
    if dist > ATTACK_RANGE then
        return
    end

    -- apply damage (tuỳ game, bạn có thể tách hệ thống damage riêng)
    local DAMAGE = 10 -- bạn có thể thay đổi
    local success, err = pcall(function()
        -- nếu humanoid tồn tại và còn sống
        if humanoid and humanoid.Health > 0 then
            humanoid:TakeDamage(DAMAGE)
        end
    end)
    if not success then
        warn("HubServer: lỗi khi damage:", err)
    end

    lastAttack[player.UserId] = now
end)

-- Xử lý teleport request
-- client gửi tên teleport point (string) hoặc Instance (Part). Server kiểm tra an toàn.
RequestTeleport.OnServerEvent:Connect(function(player, arg)
    if typeof(player) ~= "Instance" or not player:IsA("Player") then return end
    local now = tick()
    local last = lastTeleport[player.UserId] or 0
    if now - last < TELEPORT_COOLDOWN then return end

    if not player.Character or not player.Character:FindFirstChild("HumanoidRootPart") then return end

    local destCFrame = nil
    if typeof(arg) == "Instance" and arg:IsA("BasePart") and arg.Parent then
        destCFrame = arg.CFrame
    elseif typeof(arg) == "string" then
        local tpFolder = workspace:FindFirstChild("TeleportPoints")
        if tpFolder then
            local p = tpFolder:FindFirstChild(arg)
            if p and p:IsA("BasePart") then
                destCFrame = p.CFrame
            end
        end
    end

    if not destCFrame then return end

    -- optional distance check (so you don't allow teleport across whole game if undesired)
    local hrp = player.Character.HumanoidRootPart
    local dist = (hrp.Position - destCFrame.p).Magnitude
    if TELEPORT_MAX_DISTANCE and dist > TELEPORT_MAX_DISTANCE then
        -- deny if too far (optional)
        return
    end

    -- perform teleport server-side
    local success, err = pcall(function()
        if player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
            player.Character.HumanoidRootPart.CFrame = destCFrame + Vector3.new(0, 3, 0) -- đi lên 3 studs
        end
    end)
    if not success then
        warn("HubServer: Teleport thất bại:", err)
    else
        lastTeleport[player.UserId] = now
    end
end)

-- (Tùy chọn) Clean up: khi enemy die, remove tag / nothing to do here.
print("HubServer đã sẵn sàng.")

-- HubClient (StarterPlayerScripts/HubClient)
-- Tạo UI tiếng Việt + AutoFarm + ESP + Theo dõi sự kiện + Teleport request

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local CollectionService = game:GetService("CollectionService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

-- Remotes
local remotesFolder = ReplicatedStorage:WaitForChild("HubRemotes")
local RequestAttack = remotesFolder:WaitForChild("RequestAttack")
local RequestTeleport = remotesFolder:WaitForChild("RequestTeleport")

-- Config client-side (tùy chỉnh)
local AUTO_ATTACK_INTERVAL = 0.25 -- khoảng gửi request khi AutoFarm bật
local ESP_DISTANCE = 1000         -- tối đa hiển thị ESP
local AUTO_TARGET_RANGE = 60     -- client tìm mục tiêu trong range này

-- State
local enabledAutoFarm = false
local enabledESP = false
local enabledEventTracker = true

-- Data
local enemies = {}   -- instance -> {billboard = Instance}
local eventItems = {} -- id -> data

-- ========== UI tạo tự động ==========
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "Hub_Gui"
screenGui.ResetOnSpawn = false
screenGui.Parent = playerGui

-- Main window
local mainFrame = Instance.new("Frame")
mainFrame.Size = UDim2.new(0, 360, 0, 420)
mainFrame.Position = UDim2.new(0, 20, 0, 60)
mainFrame.BackgroundTransparency = 0.2
mainFrame.BackgroundColor3 = Color3.fromRGB(20,20,30)
mainFrame.BorderSizePixel = 0
mainFrame.Parent = screenGui

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, -10, 0, 36)
title.Position = UDim2.new(0, 5, 0, 6)
title.BackgroundTransparency = 1
title.Text = "HUB Hỗ Trợ - Tiếng Việt"
title.Font = Enum.Font.SourceSansBold
title.TextSize = 20
title.TextColor3 = Color3.new(1,1,1)
title.Parent = mainFrame

-- Tabs column (left)
local tabColumn = Instance.new("Frame")
tabColumn.Size = UDim2.new(0, 120, 1, -50)
tabColumn.Position = UDim2.new(0, 8, 0, 50)
tabColumn.BackgroundTransparency = 1
tabColumn.Parent = mainFrame

local contentFrame = Instance.new("Frame")
contentFrame.Size = UDim2.new(1, -140, 1, -60)
contentFrame.Position = UDim2.new(0, 132, 0, 50)
contentFrame.BackgroundTransparency = 1
contentFrame.Parent = mainFrame

local function makeTabButton(text, y)
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(1, -8, 0, 40)
    b.Position = UDim2.new(0, 4, 0, y)
    b.BackgroundTransparency = 0.3
    b.Text = text
    b.Font = Enum.Font.SourceSans
    b.TextSize = 16
    b.TextColor3 = Color3.new(1,1,1)
    b.Parent = tabColumn
    return b
end

-- content pages
local pages = {}
local currentPage = nil
local function createPage(name)
    local p = Instance.new("Frame")
    p.Size = UDim2.new(1,0,1,0)
    p.Position = UDim2.new(0,0,0,0)
    p.BackgroundTransparency = 1
    p.Visible = false
    p.Parent = contentFrame
    pages[name] = p
    return p
end

local function showPage(name)
    for k,v in pairs(pages) do v.Visible = false end
    if pages[name] then pages[name].Visible = true end
    currentPage = name
end

-- Tabs: AutoFarm, Sự Kiện, ESP, Dịch Chuyển
local btnAuto = makeTabButton("AutoFarm", 4)
local btnEvent = makeTabButton("Sự Kiện", 48)
local btnESP = makeTabButton("ESP", 92)
local btnTP = makeTabButton("Dịch Chuyển", 136)

local pageAuto = createPage("Auto")
local pageEvent = createPage("Event")
local pageESP = createPage("ESP")
local pageTP = createPage("TP")

-- Auto page elements
local lblAuto = Instance.new("TextLabel", pageAuto)
lblAuto.Size = UDim2.new(1,0,0,24)
lblAuto.Position = UDim2.new(0,0,0,0)
lblAuto.BackgroundTransparency = 1
lblAuto.Text = "Chức năng AutoFarm"
lblAuto.Font = Enum.Font.SourceSansBold
lblAuto.TextSize = 16
lblAuto.TextColor3 = Color3.new(1,1,1)

local toggleAuto = Instance.new("TextButton", pageAuto)
toggleAuto.Size = UDim2.new(0, 120, 0, 36)
toggleAuto.Position = UDim2.new(0, 0, 0, 36)
toggleAuto.Text = "Bật AutoFarm"
toggleAuto.Font = Enum.Font.SourceSans
toggleAuto.TextSize = 16

-- Event page
local lblEvent = Instance.new("TextLabel", pageEvent)
lblEvent.Size = UDim2.new(1,0,0,24)
lblEvent.Position = UDim2.new(0,0,0,0)
lblEvent.BackgroundTransparency = 1
lblEvent.Text = "Theo dõi sự kiện (EventItem, Boss)"
lblEvent.Font = Enum.Font.SourceSansBold
lblEvent.TextSize = 16
lblEvent.TextColor3 = Color3.new(1,1,1)

local eventList = Instance.new("ScrollingFrame", pageEvent)
eventList.Size = UDim2.new(1,0,1,-40)
eventList.Position = UDim2.new(0,0,0,36)
eventList.CanvasSize = UDim2.new(0,0)
eventList.BackgroundTransparency = 0.4

-- ESP page
local lblESP = Instance.new("TextLabel", pageESP)
lblESP.Size = UDim2.new(1,0,0,24)
lblESP.Position = UDim2.new(0,0,0,0)
lblESP.BackgroundTransparency = 1
lblESP.Text = "ESP: hiển thị quái / item"
lblESP.Font = Enum.Font.SourceSansBold
lblESP.TextSize = 16
lblESP.TextColor3 = Color3.new(1,1,1)

local toggleESP = Instance.new("TextButton", pageESP)
toggleESP.Size = UDim2.new(0, 120, 0, 36)
toggleESP.Position = UDim2.new(0, 0, 0, 36)
toggleESP.Text = "Bật ESP"
toggleESP.Font = Enum.Font.SourceSans
toggleESP.TextSize = 16

-- Teleport page
local lblTP = Instance.new("TextLabel", pageTP)
lblTP.Size = UDim2.new(1,0,0,24)
lblTP.Position = UDim2.new(0,0,0,0)
lblTP.BackgroundTransparency = 1
lblTP.Text = "Dịch chuyển tới điểm"
lblTP.Font = Enum.Font.SourceSansBold
lblTP.TextSize = 16
lblTP.TextColor3 = Color3.new(1,1,1)

local tpList = Instance.new("ScrollingFrame", pageTP)
tpList.Size = UDim2.new(1,0,1,-40)
tpList.Position = UDim2.new(0,0,0,36)
tpList.CanvasSize = UDim2.new(0,0)
tpList.BackgroundTransparency = 0.4

-- helper to create small label/button in scroll
local function addListButton(parent, text, callback)
    local btn = Instance.new("TextButton")
    btn.Text = tostring(text)
    btn.Size = UDim2.new(1, -10, 0, 28)
    btn.Position = UDim2.new(0, 5, 0, (#parent:GetChildren()-1) * 32)
    btn.AnchorPoint = Vector2.new(0,0)
    btn.BackgroundTransparency = 0.3
    btn.Font = Enum.Font.SourceSans
    btn.TextSize = 14
    btn.Parent = parent
    btn.MouseButton1Click:Connect(function()
        callback()
    end)
    -- update canvas size
    parent.CanvasSize = UDim2.new(0,0,0, (#parent:GetChildren()) * 32)
    return btn
end

-- show default page
showPage("Auto")

-- Tab button connections
btnAuto.MouseButton1Click:Connect(function() showPage("Auto") end)
btnEvent.MouseButton1Click:Connect(function() showPage("Event") end)
btnESP.MouseButton1Click:Connect(function() showPage("ESP") end)
btnTP.MouseButton1Click:Connect(function() showPage("TP") end)

-- ========== ESP implementation ==========
local createdMarkers = {} -- Instance -> markerPart

local function createESPForEnemy(model)
    if not model or not model.PrimaryPart then return end
    if createdMarkers[model] then return end

    local part = Instance.new("Part")
    part.Name = "Hub_ESP_Marker"
    part.Size = Vector3.new(1,1,1)
    part.Anchored = true
    part.CanCollide = false
    part.Transparency = 1
    part.CFrame = model.PrimaryPart.CFrame
    part.Parent = workspace

    local bb = Instance.new("BillboardGui")
    bb.Name = "HubESP_BB"
    bb.Adornee = part
    bb.Size = UDim2.new(0,160,0,40)
    bb.AlwaysOnTop = true
    bb.Parent = part

    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1,1,1,0)
    lbl.BackgroundTransparency = 1
    lbl.TextScaled = true
    lbl.Text = model.Name
    lbl.Font = Enum.Font.SourceSansBold
    lbl.TextColor3 = Color3.new(1,0.8,0.2)
    lbl.Parent = bb

    createdMarkers[model] = part
end

local function removeESPForEnemy(model)
    local p = createdMarkers[model]
    if p and p.Parent then
        p:Destroy()
    end
    createdMarkers[model] = nil
end

-- monitor enemies (by tag "Enemy")
for _,inst in ipairs(CollectionService:GetTagged("Enemy")) do
    if inst:IsA("Model") then
        enemies[inst] = true
    end
end

CollectionService:GetInstanceAddedSignal("Enemy"):Connect(function(inst)
    if inst:IsA("Model") then
        enemies[inst] = true
    end
end)
CollectionService:GetInstanceRemovedSignal("Enemy"):Connect(function(inst)
    enemies[inst] = nil
    removeESPForEnemy(inst)
end)

-- ESP toggle
toggleESP.MouseButton1Click:Connect(function()
    enabledESP = not enabledESP
    toggleESP.Text = enabledESP and "Tắt ESP" or "Bật ESP"
    if not enabledESP then
        -- clear markers
        for m,_ in pairs(createdMarkers) do removeESPForEnemy(m) end
    end
end)

-- ========== Event tracker (theo dõi tags "EventItem") ==========
-- Khi có instance được tag "EventItem", hiện lên eventList
local function addEventListItem(inst)
    local name = inst.Name or "Event"
    addListButton(eventList, name, function()
        -- khi click, dịch chuyển tới vị trí event (gửi request teleport cho server)
        if inst:IsA("BasePart") then
            RequestTeleport:FireServer(inst)
        elseif inst.PrimaryPart then
            RequestTeleport:FireServer(inst.PrimaryPart)
        end
    end)
end

for _,inst in ipairs(CollectionService:GetTagged("EventItem")) do
    addEventListItem(inst)
end

CollectionService:GetInstanceAddedSignal("EventItem"):Connect(function(inst)
    addEventListItem(inst)
end)
CollectionService:GetInstanceRemovedSignal("EventItem"):Connect(function(inst)
    -- remove by name (simple approach)
    for i,child in ipairs(eventList:GetChildren()) do
        if child:IsA("TextButton") and child.Text == inst.Name then
            child:Destroy()
        end
    end
    eventList.CanvasSize = UDim2.new(0,0,0,#eventList:GetChildren()*32)
end)

-- ========== Teleport list (từ workspace.TeleportPoints) ==========
local function rebuildTeleportList()
    tpList:ClearAllChildren()
    local tpFolder = workspace:FindFirstChild("TeleportPoints")
    if not tpFolder then
        tpList.CanvasSize = UDim2.new(0,0)
        return
    end
    for _,v in ipairs(tpFolder:GetChildren()) do
        if v:IsA("BasePart") then
            addListButton(tpList, v.Name, function()
                RequestTeleport:FireServer(v) -- gửi instance part
            end)
        end
    end
end

-- rebuild on start and when TeleportPoints changes (simple watch)
rebuildTeleportList()
if workspace:FindFirstChild("TeleportPoints") then
    workspace:FindFirstChild("TeleportPoints").ChildAdded:Connect(rebuildTeleportList)
    workspace:FindFirstChild("TeleportPoints").ChildRemoved:Connect(rebuildTeleportList)
end

-- ========== AutoFarm core ==========
local function findNearestEnemyWithin(range)
    if not player.Character or not player.Character:FindFirstChild("HumanoidRootPart") then return nil end
    local hrp = player.Character.HumanoidRootPart
    local best, bestDist = nil, math.huge
    for inst,_ in pairs(enemies) do
        if inst and inst.Parent and inst.PrimaryPart then
            local d = (hrp.Position - inst.PrimaryPart.Position).Magnitude
            if d < bestDist and d <= range then
                bestDist = d
                best = inst
            end
        end
    end
    return best, bestDist
end

-- Toggle auto button
toggleAuto.MouseButton1Click:Connect(function()
    enabledAutoFarm = not enabledAutoFarm
    toggleAuto.Text = enabledAutoFarm and "Tắt AutoFarm" or "Bật AutoFarm"
end)

-- Auto loop: gửi RequestAttack tới server
spawn(function()
    while true do
        if enabledAutoFarm then
            local target, dist = findNearestEnemyWithin(AUTO_TARGET_RANGE)
            if target then
                -- optional: show highlight or marker
                -- gửi request attack (server sẽ validate)
                RequestAttack:FireServer(target)
            end
            wait(AUTO_ATTACK_INTERVAL)
        else
            wait(0.2)
        end
    end
end)

-- ========== Per-frame updates (ESP position updates) ==========
RunService.Heartbeat:Connect(function()
    -- Update ESP markers positions & visibility
    if enabledESP then
        local char = player.Character
        local hrpPos = char and char:FindFirstChild("HumanoidRootPart") and char.HumanoidRootPart.Position or nil
        for inst,_ in pairs(enemies) do
            if inst and inst.Parent and inst.PrimaryPart then
                local markerPart = createdMarkers[inst]
                if markerPart then
                    markerPart.CFrame = inst.PrimaryPart.CFrame
                else
                    -- create only if within ESP_DISTANCE of player (optional)
                    if hrpPos then
                        local d = (hrpPos - inst.PrimaryPart.Position).Magnitude
                        if d <= ESP_DISTANCE then
                            createESPForEnemy(inst)
                        end
                    else
                        createESPForEnemy(inst)
                    end
                end
            else
                -- cleanup
                if createdMarkers[inst] then removeESPForEnemy(inst) end
            end
        end
    end
end)

-- ========== Cleanup when character removed ==========
player.CharacterRemoving:Connect(function()
    -- nothing heavy here; markers are anchored to workspace and will be visible
end)

print("HubClient đã sẵn sàng.")
