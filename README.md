-- Mobile Detection
local UserInputService = game:GetService("UserInputService")
local isMobile = UserInputService.TouchEnabled and not (UserInputService.KeyboardEnabled or UserInputService.MouseEnabled)

-- Load UI Library based on device
local Fluent, DrRay, SaveManager, InterfaceManager
if not isMobile then
    -- Load Fluent for non-mobile devices
    Fluent = loadstring(game:HttpGet("https://github.com/dawid-scripts/Fluent/releases/latest/download/main.lua"))()
    SaveManager = loadstring(game:HttpGet("https://raw.githubusercontent.com/dawid-scripts/Fluent/master/Addons/SaveManager.lua"))()
    InterfaceManager = loadstring(game:HttpGet("https://raw.githubusercontent.com/dawid-scripts/Fluent/master/Addons/InterfaceManager.lua"))()
else
    -- Load DrRay for mobile devices
    DrRay = loadstring(game:HttpGet("https://raw.githubusercontent.com/AZYsGithub/DrRay-UI-Library/main/DrRay.lua"))()
end

-- Window Setup
local Window, Tabs
if not isMobile then
    Window = Fluent:CreateWindow({
        Title = "NFL UNIVERSE " .. Fluent.Version,
        SubTitle = "by @f3a2 | Beta | discord.gg/Bqye595N72",
        TabWidth = 160,
        Size = UDim2.fromOffset(580, 460),
        Theme = "Dark",
        Acrylic = true,
        MinimizeKey = Enum.KeyCode.LeftControl
    })
    Tabs = {
        Main = Window:AddTab({ Title = "Main", Icon = "" }),
        Settings = Window:AddTab({ Title = "Settings", Icon = "settings" })
    }
else
    Window = DrRay -- DrRay uses a simpler window setup
    Tabs = {
        Main = Window.newTab("Main", "rbxassetid://3926305904"), -- Example icon ID
        Settings = Window.newTab("Settings", "rbxassetid://3926307971")
    }
    Window:SetTheme(Color3.fromRGB(10, 30, 10), Color3.fromRGB(50, 50, 10)) -- Custom theme for mobile
end

-- Services
local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local PathfindingService = game:GetService("PathfindingService")

-- Player and character setup
local player = Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()
local humanoid = character:WaitForChild("Humanoid")
local humanoidRootPart = character:WaitForChild("HumanoidRootPart")
local defaultWalkSpeed = humanoid.WalkSpeed
local defaultJumpPower = humanoid.JumpPower

-- Unlock zoom distance
player.CameraMaxZoomDistance = math.huge
spawn(function()
    while true do
        if player.CameraMaxZoomDistance ~= math.huge then
            player.CameraMaxZoomDistance = math.huge
        end
        wait(0.1)
    end
end)

-- Variables (unchanged)
local pullVectorEnabled = false
local smoothPullEnabled = false
local isPullingBall = false
local isSmoothPulling = false
local flyEnabled = false
local isFlying = false
local walkSpeedEnabled = false
local teleportForwardEnabled = false
local kickingAimbotEnabled = false
local landingIndicatorEnabled = false
local jumpPowerEnabled = false

local offsetDistance = 4
local magnetSmoothness = 0.01
local updateInterval = 0.01
local customWalkSpeed = 50
local flySpeed = 50
local customJumpPower = 50

local catchboxSize = Vector3.new(5.197499752044678, 6.299999713897705, 2.309999942779541)
local upperTorsoOffset = Vector3.new(0, 1.5, 0)
local flyBodyVelocity = nil
local flyBodyGyro = nil
local throwingArcPath = nil
local landingMarker = nil
local markerConnection = nil
local rainbowConnection = nil
local jumpConnection = nil
local isParkMatch = Workspace:FindFirstChild("ParkMatchMap") ~= nil

-- Catching Section
local CatchingSection
if not isMobile then
    CatchingSection = Tabs.Main:AddSection("Catching", { Box = true, Collapsible = true })
else
    CatchingSection = Tabs.Main -- DrRay doesn't use sections, so we add directly to tab
end

local function addToggle(section, name, options)
    if not isMobile then
        section:AddToggle(name, options)
    else
        section.newToggle(options.Title, options.Description or "No description", options.Default, options.Callback)
    end
end

local function addSlider(section, name, options)
    if not isMobile then
        section:AddSlider(name, options)
    else
        section.newSlider(options.Title, options.Description or "No description", options.Max, options.Default, options.Callback)
    end
end

addToggle(CatchingSection, "PullVector", {
    Title = "Pull Vector [M1]",
    Default = false,
    Callback = function(value)
        pullVectorEnabled = value
    end
})

addSlider(CatchingSection, "OffsetDistance", {
    Title = "Offset Distance",
    Description = "Distance in front of the ball",
    Default = 4,
    Min = 5,
    Max = 30,
    Rounding = 0,
    Callback = function(value)
        offsetDistance = value
    end
})

addToggle(CatchingSection, "LegitMagnets", {
    Title = "Legit Pull Vector [M1]",
    Default = false,
    Callback = function(value)
        smoothPullEnabled = value
    end
})

addSlider(CatchingSection, "Smoothing", {
    Title = "Magnet Smoothness",
    Description = "Lower = smoother, higher = faster",
    Default = 0.01,
    Min = 0.1,
    Max = 1.0,
    Rounding = 2,
    Callback = function(value)
        magnetSmoothness = value
    end
})

-- Physics Section
local PhysicsSection
if not isMobile then
    PhysicsSection = Tabs.Main:AddSection("Physics", { Box = true, Collapsible = true })
else
    PhysicsSection = Tabs.Main
end

local function enforceWalkSpeed()
    while walkSpeedEnabled and humanoid do
        humanoid.WalkSpeed = customWalkSpeed
        wait(0.1)
    end
    if humanoid and not walkSpeedEnabled then
        humanoid.WalkSpeed = defaultWalkSpeed
    end
end

addToggle(PhysicsSection, "WalkSpeed", {
    Title = "WalkSpeed",
    Description = "Increases your walking speed",
    Default = false,
    Callback = function(value)
        walkSpeedEnabled = value
        if value then
            spawn(enforceWalkSpeed)
        elseif humanoid then
            humanoid.WalkSpeed = defaultWalkSpeed
        end
    end
})

addSlider(PhysicsSection, "WalkSpeedValue", {
    Title = "Custom WalkSpeed",
    Default = 50,
    Min = 10,
    Max = 100,
    Rounding = 0,
    Callback = function(value)
        customWalkSpeed = value
    end
})

local function applyCustomJump()
    if jumpConnection then jumpConnection:Disconnect() end
    jumpConnection = humanoid.Jumping:Connect(function()
        if jumpPowerEnabled and humanoidRootPart then
            local jumpVelocity = Vector3.new(0, customJumpPower, 0)
            humanoidRootPart.Velocity = Vector3.new(humanoidRootPart.Velocity.X, 0, humanoidRootPart.Velocity.Z) + jumpVelocity
        end
    end)
end

addToggle(PhysicsSection, "JumpPower", {
    Title = "JumpPower",
    Description = "Increases your jump height",
    Default = false,
    Callback = function(value)
        jumpPowerEnabled = value
        if value then
            applyCustomJump()
        else
            if jumpConnection then jumpConnection:Disconnect() end
            jumpConnection = nil
        end
    end
})

addSlider(PhysicsSection, "JumpPowerValue", {
    Title = "Custom JumpPower",
    Default = 50,
    Min = 10,
    Max = 200,
    Rounding = 0,
    Callback = function(value)
        customJumpPower = value
        if jumpPowerEnabled then
            applyCustomJump()
        end
    end
})

local function startFlying()
    if flyBodyVelocity or flyBodyGyro then return end
    flyBodyVelocity = Instance.new("BodyVelocity")
    flyBodyVelocity.MaxForce = Vector3.new(100000, 100000, 100000)
    flyBodyVelocity.Velocity = Vector3.new(0, 0, 0)
    flyBodyVelocity.Parent = humanoidRootPart
    flyBodyGyro = Instance.new("BodyGyro")
    flyBodyGyro.MaxTorque = Vector3.new(100000, 100000, 100000)
    flyBodyGyro.P = 1000
    flyBodyGyro.D = 100
    flyBodyGyro.Parent = humanoidRootPart
    isFlying = true
    spawn(function()
        while isFlying do
            local camera = Workspace.CurrentCamera
            local moveDirection = Vector3.new(0, 0, 0)
            if UserInputService:IsKeyDown(Enum.KeyCode.W) then
                moveDirection = moveDirection + camera.CFrame.LookVector
            end
            if UserInputService:IsKeyDown(Enum.KeyCode.S) then
                moveDirection = moveDirection - camera.CFrame.LookVector
            end
            if UserInputService:IsKeyDown(Enum.KeyCode.A) then
                moveDirection = moveDirection - camera.CFrame.RightVector
            end
            if UserInputService:IsKeyDown(Enum.KeyCode.D) then
                moveDirection = moveDirection + camera.CFrame.RightVector
            end
            if UserInputService:IsKeyDown(Enum.KeyCode.Space) then
                moveDirection = moveDirection + Vector3.new(0, 1, 0)
            end
            if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then
                moveDirection = moveDirection - Vector3.new(0, 1, 0)
            end
            if moveDirection.Magnitude > 0 then
                flyBodyVelocity.Velocity = moveDirection.Unit * flySpeed
            else
                flyBodyVelocity.Velocity = Vector3.new(0, 0, 0)
            end
            wait()
        end
    end)
end

local function stopFlying()
    if flyBodyVelocity then flyBodyVelocity:Destroy() flyBodyVelocity = nil end
    if flyBodyGyro then flyBodyGyro:Destroy() flyBodyGyro = nil end
    isFlying = false
end

addToggle(PhysicsSection, "FlyToggle", {
    Title = "Fly",
    Description = "Allows your character to fly",
    Default = false,
    Callback = function(value)
        flyEnabled = value
        if value then
            startFlying()
        else
            stopFlying()
        end
    end
})

addSlider(PhysicsSection, "FlySpeed", {
    Title = "Fly Speed",
    Default = 50,
    Min = 10,
    Max = 200,
    Rounding = 0,
    Callback = function(value)
        flySpeed = value
    end
})

addToggle(PhysicsSection, "TeleportForward", {
    Title = "Teleport Forward (Press Z)",
    Description = "Teleports you forward 3 studs",
    Default = false,
    Callback = function(value)
        teleportForwardEnabled = value
    end
})

-- Miscellaneous Section
local MiscSection
if not isMobile then
    MiscSection = Tabs.Main:AddSection("Miscellaneous", { Box = true, Collapsible = true })
else
    MiscSection = Tabs.Main
end

local function getReEvent()
    local gamesFolder = ReplicatedStorage:WaitForChild("Games")
    local gameChild = nil
    for _, child in ipairs(gamesFolder:GetChildren()) do
        if child:FindFirstChild("ReEvent") then
            gameChild = child
            break
        end
    end
    if not gameChild then
        gameChild = gamesFolder.ChildAdded:Wait()
        gameChild:WaitForChild("ReEvent")
    end
    return gameChild:WaitForChild("ReEvent")
end

local function onKick()
    local ReEvent = getReEvent()
    local angleArgs = { [1] = "Mechanics", [2] = "KickAngleChanged", [3] = 1, [4] = 60, [5] = 1 }
    ReEvent:FireServer(unpack(angleArgs))
    local powerArgs = { [1] = "Mechanics", [2] = "KickPowerSet", [3] = 1 }
    ReEvent:FireServer(unpack(powerArgs))
    local hikeArgs = { [1] = "Mechanics", [2] = "KickHiked", [3] = 60, [4] = 1, [5] = 1 }
    ReEvent:FireServer(unpack(hikeArgs))
    local accuracyArgs = { [1] = "Mechanics", [2] = "KickAccuracySet", [3] = 60 }
    ReEvent:FireServer(unpack(accuracyArgs))
end

addToggle(MiscSection, "KickAimbot", {
    Title = "Kick Aimbot (Press K)",
    Description = "Can make up to 60 yard field goals",
    Default = false,
    Callback = function(value)
        kickingAimbotEnabled = value
    end
})

local function getThrowingArcPath()
    local miniGames = Workspace:FindFirstChild("MiniGames")
    local games = Workspace:FindFirstChild("Games")
    if miniGames then
        for _, gameInstance in ipairs(miniGames:GetChildren()) do
            local localFolder = gameInstance:FindFirstChild("Local")
            if localFolder and localFolder:FindFirstChild("Center") then
                local throwingArc = localFolder.Center:FindFirstChild("ThrowingArc")
                if throwingArc and throwingArc:IsA("Beam") then return localFolder.Center end
            end
        end
    end
    if games then
        for _, gameInstance in ipairs(games:GetChildren()) do
            local localFolder = gameInstance:FindFirstChild("Local")
            if localFolder and localFolder:FindFirstChild("Center") then
                local throwingArc = localFolder.Center:FindFirstChild("ThrowingArc")
                if throwingArc and throwingArc:IsA("Beam") then return localFolder.Center end
            end
        end
    end
    return nil
end

local function createMarker()
    if not landingMarker then
        landingMarker = Instance.new("Part")
        landingMarker.Anchored = true
        landingMarker.CanCollide = false
        landingMarker.Size = Vector3.new(5, 10, 5)
        landingMarker.Transparency = 0.3
        landingMarker.Parent = Workspace
        local hue = 0
        rainbowConnection = RunService.RenderStepped:Connect(function(deltaTime)
            if landingMarker and landingMarker.Parent then
                hue = (hue + deltaTime * 0.5) % 1
                landingMarker.Color = Color3.fromHSV(hue, 1, 1)
            else
                rainbowConnection:Disconnect()
                rainbowConnection = nil
            end
        end)
    end
end

local function removeMarker()
    if landingMarker then landingMarker:Destroy() landingMarker = nil end
    if rainbowConnection then rainbowConnection:Disconnect() rainbowConnection = nil end
end

local function updateLandingMarker()
    throwingArcPath = getThrowingArcPath()
    if not throwingArcPath then removeMarker() return end
    local throwingArc = throwingArcPath:FindFirstChild("ThrowingArc")
    if throwingArc and throwingArc:IsA("Beam") and throwingArc.Enabled and throwingArc.Attachment0 and throwingArc.Attachment1 then
        if not landingMarker or not landingMarker.Parent then createMarker() end
        local startPosition = throwingArc.Attachment0.WorldPosition
        local endPosition = throwingArc.Attachment1.WorldPosition
        local direction = (endPosition - startPosition).Unit
        local offsetPosition = endPosition + (direction * 0)
        landingMarker.Position = offsetPosition - Vector3.new(0, landingMarker.Size.Y / 3, 0)
    else
        removeMarker()
    end
end

local function startTracking()
    if markerConnection then return end
    local games = Workspace:FindFirstChild("Games")
    if games then
        games.ChildAdded:Connect(function() if landingIndicatorEnabled then throwingArcPath = getThrowingArcPath() end end)
        games.ChildRemoved:Connect(function() if landingIndicatorEnabled then throwingArcPath = getThrowingArcPath() end end)
    end
    markerConnection = RunService.RenderStepped:Connect(updateLandingMarker)
end

local function stopTracking()
    if markerConnection then markerConnection:Disconnect() markerConnection = nil end
    removeMarker()
end

addToggle(MiscSection, "LandingIndicator", {
    Title = "Football Landing Indicator",
    Description = "Shows precise landing location when throwing",
    Default = false,
    Callback = function(value)
        landingIndicatorEnabled = value
        if value then startTracking() else stopTracking() end
    end
})

-- Game Logic Functions (unchanged)
local function getFootball()
    if isParkMatch then
        local parkMatchFootball = Workspace:FindFirstChild("ParkMatchMap")
        if parkMatchFootball and parkMatchFootball:FindFirstChild("Replicated") then
            parkMatchFootball = parkMatchFootball.Replicated:FindFirstChild("Fields")
            if parkMatchFootball and parkMatchFootball:FindFirstChild("MatchField") then
                parkMatchFootball = parkMatchFootball.MatchField:FindFirstChild("Replicated")
                if parkMatchFootball then
                    local football = parkMatchFootball:FindFirstChild("Football")
                    if football and football:IsA("BasePart") then return football end
                end
            end
        end
    end

    for _, otherPlayer in ipairs(Players:GetPlayers()) do
        if otherPlayer.Character then
            local ball = otherPlayer.Character:FindFirstChild("Football")
            if ball and ball:IsA("BasePart") then return ball end
        end
    end

    local gamesFolder = Workspace:FindFirstChild("Games")
    if gamesFolder then
        for _, gameInstance in ipairs(gamesFolder:GetChildren()) do
            local replicatedFolder = gameInstance:FindFirstChild("Replicated")
            if replicatedFolder then
                local kickoffFootball = replicatedFolder:FindFirstChild("918f5408-d86a-4fb8-a88c-5cab57410acf")
                if kickoffFootball and kickoffFootball:IsA("BasePart") then return kickoffFootball end
                for _, item in ipairs(replicatedFolder:GetChildren()) do
                    if item:IsA("BasePart") and item.Name == "Football" then return item end
                end
            end
        end
    end
    return nil
end

local function teleportToBall()
    local ball = getFootball()
    if ball and humanoidRootPart then
        local ballVelocity = ball.Velocity
        local ballPosition = ball.Position
        local direction = ballVelocity.Unit
        local targetPosition = ballPosition + (direction * 12) - upperTorsoOffset + Vector3.new(0, catchboxSize.Y / 6, 0)
        local lookDirection = (ballPosition - humanoidRootPart.Position).Unit
        humanoidRootPart.CFrame = CFrame.new(targetPosition, targetPosition + lookDirection)
    end
end

local function smoothTeleportToBall()
    local ball = getFootball()
    if ball and humanoidRootPart then
        local ballVelocity = ball.Velocity
        local ballSpeed = ballVelocity.Magnitude
        local offset = (ballSpeed > 0) and (ballVelocity.Unit * offsetDistance) or Vector3.new(0, 0, 0)
        local targetPosition = ball.Position + offset + Vector3.new(0, 3, 0)
        local lookDirection = (ball.Position - humanoidRootPart.Position).Unit
        humanoidRootPart.CFrame = humanoidRootPart.CFrame:Lerp(CFrame.new(targetPosition, targetPosition + lookDirection), magnetSmoothness)
    end
end

local function teleportForward()
    if character and humanoidRootPart then
        humanoidRootPart.CFrame = humanoidRootPart.CFrame + (humanoidRootPart.CFrame.LookVector * 3)
    end
end

-- Input Handling
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.UserInputType == Enum.UserInputType.MouseButton1 or 
       (input.UserInputType == Enum.UserInputType.Gamepad1 and input.KeyCode == Enum.KeyCode.ButtonR2) or
       (isMobile and input.UserInputType == Enum.UserInputType.Touch) then
        if pullVectorEnabled then
            isPullingBall = true
            spawn(function()
                while isPullingBall do
                    teleportToBall()
                    wait(0.05)
                end
            end)
        end
        if smoothPullEnabled then
            isSmoothPulling = true
            spawn(function()
                while isSmoothPulling do
                    smoothTeleportToBall()
                    wait(updateInterval)
                end
            end)
        end
    elseif input.UserInputType == Enum.UserInputType.Keyboard then
        if teleportForwardEnabled and input.KeyCode == Enum.KeyCode.Z then
            teleportForward()
        end
        if input.KeyCode == Enum.KeyCode.K and kickingAimbotEnabled then
            onKick()
        end
    end
end)

UserInputService.InputEnded:Connect(function(input, gameProcessed)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or 
       (input.UserInputType == Enum.UserInputType.Gamepad1 and input.KeyCode == Enum.KeyCode.ButtonR2) or
       (isMobile and input.UserInputType == Enum.UserInputType.Touch) then
        isPullingBall = false
        isSmoothPulling = false
    end
end)

-- Settings Tab and Notifications
if not isMobile then
    SaveManager:SetLibrary(Fluent)
    InterfaceManager:SetLibrary(Fluent)
    SaveManager:IgnoreThemeSettings()
    SaveManager:SetIgnoreIndexes({})
    InterfaceManager:SetFolder("NFLUniverse")
    SaveManager:SetFolder("NFLUniverse/specific-game")

    InterfaceManager:BuildInterfaceSection(Tabs.Settings)
    SaveManager:BuildConfigSection(Tabs.Settings)

    Window:SelectTab(1)

    Fluent:Notify({
        Title = "NFL UNIVERSE",
        Content = "Script loaded successfully! (Desktop)",
        Duration = 5
    })

    SaveManager:LoadAutoloadConfig()
else
    -- Mobile notification (DrRay doesn't have a built-in notify function, so we improvise)
    Tabs.Settings.newButton("NFL UNIVERSE Loaded", "Script loaded successfully! (Mobile)", function()
        print("Script loaded on mobile!")
    end)
end
