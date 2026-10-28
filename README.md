local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")

local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

-- ====================== VARIÁVEIS ======================
local espEnabled, espNicksEnabled, espDistanceEnabled, espLifeEnabled, espGlowEnabled = false, false, false, false, false
local teamCheck, wallCheck, aimbotEnabled, showFov, aimColorEnabled = false, false, false, false, false
local showAimColorFov, targetPart, fovRadius, aimColorFovRadius = true, "Head", 150, 100
local aimColorPrediction, aimColorStrength, recoilCompensation = 0.08, 0.75, 2
local aimColorDisfarceEnabled, smoothingFactor, minimized, showFps, tpEnabled = false, 0.7, false, false, false
local toggleKey, switchTargetKey = Enum.KeyCode.Insert, Enum.KeyCode.T
local currentTitleColor = Color3.fromRGB(255, 140, 0)
local savedPosition, panelLoaded, anonimatoEnabled = nil, false, false
local clickCount, lastClickTime, nextFireTime = 0, 0, 0
local mouse1Down, ignoreMouse1Release = false, false
local infiniteAmmoEnabled, noRecoilEnabled = false, false
local nextRecoilTime, nextRecoilReturnTime = 0, 0
local DISFARCE_PARTS = {"Head", "Torso", "HumanoidRootPart", "Left Arm", "Right Arm", "Left Leg", "Right Leg"}
local frameCount, lastTime, currentFps, fpsLabel, espUpdateTimer = 0, tick(), 0, nil, 0
local lastESPUpdate = 0

local visibilityCache = {}
local VISIBILITY_CACHE_TIME = 0.2

local espBillboardObjects = {}
local glowObjects = {}
local lifeObjects = {}
local selectorButtons = {}
local sliderFills = {}

local ScreenGui, MainFrame, MinimizedFrame, ColorMenu, KeyMenu, AimbotKeyMenu = nil, nil, nil, nil, nil, nil
local ChangeColorButton, ChangeKeyButton, ChangeAimbotKeyButton = nil, nil, nil
local CurrentKeyButton, CurrentAimbotKeyButton, CurrentColorButton = nil, nil, nil
local AnonimatoButton, AnonimatoIndicator = nil, nil
local targetPartsList = {"Head", "HumanoidRootPart"}
local currentTargetIndex = 1

local aimbotTargetFrame = nil
local aimbotTargetLabel = nil

-- ===== DETECÇÃO DO BOTÃO ESQUERDO (TENTA TODAS AS OPÇÕES - XENO) =====
local function isLeftMouseDown()
    if ismouse1pressed then
        local ok, r = pcall(ismouse1pressed)
        if ok and r then return true end
    end
    if isMouse1Down then
        local ok, r = pcall(isMouse1Down)
        if ok and r then return true end
    end
    if isMouseButtonPressed then
        local ok, r = pcall(isMouseButtonPressed, 1)
        if ok and r then return true end
    end
    if mouse1Down then return true end
    local ok, r = pcall(function()
        return UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton1)
    end)
    if ok and r then return true end
    return false
end

local mouseMoveFunc = nil
if mousemoverel then
    mouseMoveFunc = function(dx, dy) mousemoverel(dx, dy) end
elseif setmousepos then
    mouseMoveFunc = function(dx, dy)
        local cur = UserInputService:GetMouseLocation()
        setmousepos(cur.X + dx, cur.Y + dy)
    end
elseif mouse_move then
    mouseMoveFunc = function(dx, dy) mouse_move(dx, dy) end
elseif move_mouse then
    mouseMoveFunc = function(dx, dy) move_mouse(dx, dy) end
else
    warn("[AimColor] Nenhuma função de mover mouse encontrada.")
end

local mouseClickFunc = nil
if mouse1click then
    mouseClickFunc = function() mouse1click() end
elseif mouse1press and mouse1release then
    mouseClickFunc = function()
        mouse1press()
        task.wait(0.05)
        mouse1release()
    end
elseif game:GetService("VirtualInputManager") then
    local vim = game:GetService("VirtualInputManager")
    mouseClickFunc = function()
        local mousePos = UserInputService:GetMouseLocation()
        pcall(function()
            vim:SendMouseButtonEvent(0, 0, true, mousePos, 0)
            task.wait(0.05)
            vim:SendMouseButtonEvent(0, 0, false, mousePos, 0)
        end)
    end
end

-- ====================== FUNÇÕES AUXILIARES ======================
local function createFPSDisplay()
    if fpsLabel then pcall(function() fpsLabel:Remove() end); fpsLabel = nil end
    if not showFps then return end
    fpsLabel = Drawing.new("Text")
    fpsLabel.Visible = true
    fpsLabel.Center = false
    fpsLabel.Outline = false
    fpsLabel.Font = 2
    fpsLabel.Size = 16
    fpsLabel.Color = Color3.fromRGB(255, 140, 0)
    fpsLabel.Position = Vector2.new(Camera.ViewportSize.X - 60, 15)
end

local function updateFPS()
    frameCount = frameCount + 1
    local currentTime = tick()
    if currentTime - lastTime >= 1 then
        currentFps = frameCount
        frameCount = 0
        lastTime = currentTime
        if fpsLabel and showFps then
            fpsLabel.Color = Color3.fromRGB(255, 140, 0)
            fpsLabel.Text = ("FPS: %d"):format(currentFps)
            fpsLabel.Position = Vector2.new(Camera.ViewportSize.X - 60, 15)
        end
    end
end

local function teleportToMouse()
    local mouse = LocalPlayer:GetMouse()
    if mouse and LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
        local position = mouse.Hit.p
        LocalPlayer.Character.HumanoidRootPart.CFrame = CFrame.new(position.X, position.Y + 3, position.Z)
    end
end

local function toggleMinimize()
    if not MainFrame or not MinimizedFrame then return end
    minimized = not minimized
    if minimized then
        savedPosition = MainFrame.Position
        MainFrame.Visible = false
        MinimizedFrame.Visible = not anonimatoEnabled
        MinimizedFrame.Position = savedPosition
        if ColorMenu then ColorMenu.Visible = false end
        if KeyMenu then KeyMenu.Visible = false end
        if AimbotKeyMenu then AimbotKeyMenu.Visible = false end
    else
        MainFrame.Visible = true
        MinimizedFrame.Visible = false
        if savedPosition then MainFrame.Position = savedPosition end
    end
end

local function getTeamColor(player)
    if player.Team then
        if player.Team.TeamColor then
            return player.Team.TeamColor.Color
        elseif player.Team.BrickColor then
            return player.Team.BrickColor.Color
        end
    end
    return Color3.fromRGB(180, 180, 180)
end

local function getLifeFontSizeByDistance(distance)
    local clampedDistance = math.clamp(distance, 5, 200)
    return math.max(8, math.min(18, 18 - (clampedDistance - 5) * (10 / 195)))
end

local function getFontSizeByDistance(distance)
    local clampedDistance = math.clamp(distance, 5, 150)
    return math.max(8, math.min(16, 16 - (clampedDistance - 5) * (8 / 145)))
end

local function isPlayerAlive(player)
    local humanoid = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
    return humanoid and humanoid.Health > 0 or false
end

local function passesFilters(player)
    return true
end

local function isMouseOverPanel()
    if not MainFrame or not MainFrame.Visible then return false end
    local mousePos = UserInputService:GetMouseLocation()
    local absPos = MainFrame.AbsolutePosition
    local absSize = MainFrame.AbsoluteSize
    return mousePos.X >= absPos.X and mousePos.X <= absPos.X + absSize.X and
           mousePos.Y >= absPos.Y and mousePos.Y <= absPos.Y + absSize.Y
end

-- ====================== CORES ======================
local DEFAULT_ACCENT = Color3.fromRGB(255, 140, 0)
local COLORS = {
    PRIMARY = Color3.fromRGB(0, 0, 0),
    SECONDARY = Color3.fromRGB(35, 35, 35),
    ACCENT = DEFAULT_ACCENT,
    ACCENT_DARK = Color3.fromRGB(0, 0, 0),
    ENABLED = Color3.fromRGB(0, 200, 0),
    DISABLED = Color3.fromRGB(200, 0, 0),
    TEXT = Color3.fromRGB(255, 255, 255),
    HEART = Color3.fromRGB(255, 80, 80)
}

local TITLE_COLORS = {
    {Nome = "Vermelho", Cor = Color3.fromRGB(220, 20, 60)},
    {Nome = "Azul", Cor = Color3.fromRGB(0, 100, 200)},
    {Nome = "Preto", Cor = Color3.fromRGB(0, 0, 0)},
    {Nome = "Roxo", Cor = Color3.fromRGB(128, 0, 128)},
    {Nome = "Verde", Cor = Color3.fromRGB(34, 139, 34)},
    {Nome = "Amarelo", Cor = Color3.fromRGB(218, 165, 32)},
    {Nome = "Laranja", Cor = Color3.fromRGB(255, 140, 0)},
    {Nome = "Rosa", Cor = Color3.fromRGB(255, 105, 180)},
    {Nome = "Ciano", Cor = Color3.fromRGB(0, 206, 209)},
    {Nome = "Branco", Cor = Color3.fromRGB(255, 255, 255)},
    {Nome = "Cinza", Cor = Color3.fromRGB(128, 128, 128)},
    {Nome = "RGB", Cor = Color3.fromRGB(255, 0, 0)}
}

-- ====================== CRIAÇÃO DA GUI ======================
ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "ESP_UI"
ScreenGui.ResetOnSpawn = false
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

pcall(function()
    if syn and syn.protect_gui then
        syn.protect_gui(ScreenGui)
    end
    ScreenGui.Parent = game.CoreGui
end)
if not ScreenGui.Parent then
    pcall(function()
        ScreenGui.Parent = game.Players.LocalPlayer.PlayerGui
    end)
end

MainFrame = Instance.new("Frame", ScreenGui)
MainFrame.Size = UDim2.new(0, 480, 0, 450)
MainFrame.Position = UDim2.new(0.5, -240, 0.5, -225)
MainFrame.BackgroundColor3 = COLORS.PRIMARY
MainFrame.BackgroundTransparency = 0.6
MainFrame.BorderSizePixel = 0
MainFrame.Active = true
MainFrame.Visible = true

local MainFrameCorner = Instance.new("UICorner", MainFrame)
MainFrameCorner.CornerRadius = UDim.new(0, 12)

local Shadow = Instance.new("ImageLabel", MainFrame)
Shadow.Size = UDim2.new(1, 30, 1, 30)
Shadow.Position = UDim2.new(0, -15, 0, -15)
Shadow.BackgroundTransparency = 1
Shadow.Image = "rbxassetid://5028857084"
Shadow.ImageColor3 = Color3.new(0, 0, 0)
Shadow.ScaleType = Enum.ScaleType.Slice
Shadow.SliceCenter = Rect.new(24, 24, 276, 276)
Shadow.ImageTransparency = 0.5
Shadow.ZIndex = 0

local TitleBar = Instance.new("Frame", MainFrame)
TitleBar.Size = UDim2.new(1, 0, 0, 45)
TitleBar.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
TitleBar.BackgroundTransparency = 0.5
TitleBar.BorderSizePixel = 0
TitleBar.Active = true

local TitleBarCorner = Instance.new("UICorner", TitleBar)
TitleBarCorner.CornerRadius = UDim.new(0, 12)

local Title = Instance.new("TextLabel", TitleBar)
Title.Text = "Blackhole Project"
Title.Size = UDim2.new(0.5, 0, 1, 0)
Title.Position = UDim2.new(0, 15, 0, 0)
Title.BackgroundTransparency = 1
Title.TextColor3 = Color3.fromRGB(255, 140, 0)
Title.Font = Enum.Font.GothamBlack
Title.TextSize = 16
Title.TextXAlignment = Enum.TextXAlignment.Left

local CreditText = Instance.new("TextLabel", TitleBar)
CreditText.Text = "by: machadinho900, zks_frozen"
CreditText.Size = UDim2.new(0.25, 0, 1, 0)
CreditText.Position = UDim2.new(0.35, 0, 0, 0)
CreditText.BackgroundTransparency = 1
CreditText.TextColor3 = Color3.fromRGB(255, 140, 0)
CreditText.Font = Enum.Font.GothamBlack
CreditText.TextSize = 12
CreditText.TextXAlignment = Enum.TextXAlignment.Left

local MinimizeButton = Instance.new("TextButton", TitleBar)
MinimizeButton.Size = UDim2.new(0, 30, 0, 30)
MinimizeButton.Position = UDim2.new(1, -40, 0.5, -15)
MinimizeButton.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
MinimizeButton.BorderSizePixel = 0
MinimizeButton.Text = ""

local MinimizeCorner = Instance.new("UICorner", MinimizeButton)
MinimizeCorner.CornerRadius = UDim.new(0, 6)

local MinimizeIcon = Instance.new("TextLabel", MinimizeButton)
MinimizeIcon.Text = "+"
MinimizeIcon.Size = UDim2.new(1, 0, 1, 0)
MinimizeIcon.Position = UDim2.new(0, 0, 0, 0)
MinimizeIcon.BackgroundTransparency = 1
MinimizeIcon.TextColor3 = Color3.fromRGB(255, 140, 0)
MinimizeIcon.Font = Enum.Font.GothamBlack
MinimizeIcon.TextSize = 24
MinimizeIcon.TextScaled = true

AnonimatoButton = Instance.new("TextButton", TitleBar)
AnonimatoButton.Size = UDim2.new(0, 100, 0, 30)
AnonimatoButton.Position = UDim2.new(1, -140, 0.5, -15)
AnonimatoButton.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
AnonimatoButton.BackgroundTransparency = 0.5
AnonimatoButton.BorderSizePixel = 0
AnonimatoButton.Text = "Anonimato"
AnonimatoButton.TextColor3 = Color3.fromRGB(255, 140, 0)
AnonimatoButton.Font = Enum.Font.GothamBlack
AnonimatoButton.TextSize = 12
AnonimatoButton.AutoButtonColor = false

local AnonimatoCorner = Instance.new("UICorner", AnonimatoButton)
AnonimatoCorner.CornerRadius = UDim.new(0, 6)

AnonimatoIndicator = Instance.new("Frame", AnonimatoButton)
AnonimatoIndicator.Size = UDim2.new(0, 14, 0, 14)
AnonimatoIndicator.Position = UDim2.new(1, -16, 0.5, -7)
AnonimatoIndicator.BorderSizePixel = 0
AnonimatoIndicator.BackgroundColor3 = Color3.fromRGB(200, 0, 0)
local AnonimatoIndicatorCorner = Instance.new("UICorner", AnonimatoIndicator)
AnonimatoIndicatorCorner.CornerRadius = UDim.new(0, 7)

local function updateAnonimatoVisual()
    AnonimatoIndicator.BackgroundColor3 = anonimatoEnabled and Color3.fromRGB(0, 200, 0) or Color3.fromRGB(200, 0, 0)
    MinimizeButton.Visible = not anonimatoEnabled
end

AnonimatoButton.MouseButton1Click:Connect(function()
    anonimatoEnabled = not anonimatoEnabled
    clickCount = 0
    updateAnonimatoVisual()
end)

-- Drag do painel
local draggingPanel = false
local dragStartPos
local panelStartPos

TitleBar.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        draggingPanel = true
        dragStartPos = input.Position
        panelStartPos = MainFrame.Position
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if draggingPanel and input.UserInputType == Enum.UserInputType.MouseMovement then
        local delta = input.Position - dragStartPos
        MainFrame.Position = UDim2.new(panelStartPos.X.Scale, panelStartPos.X.Offset + delta.X,
                                        panelStartPos.Y.Scale, panelStartPos.Y.Offset + delta.Y)
        savedPosition = MainFrame.Position
        if ColorMenu and ColorMenu.Visible then
            local buttonPos = ChangeColorButton and ChangeColorButton.AbsolutePosition
            if buttonPos then
                ColorMenu.Position = UDim2.new(0, buttonPos.X - 150, 0, buttonPos.Y + 35)
            end
        end
        if KeyMenu and KeyMenu.Visible then
            local buttonPos = ChangeKeyButton and ChangeKeyButton.AbsolutePosition
            if buttonPos then
                KeyMenu.Position = UDim2.new(0, buttonPos.X - 150, 0, buttonPos.Y + 35)
            end
        end
        if AimbotKeyMenu and AimbotKeyMenu.Visible then
            local buttonPos = ChangeAimbotKeyButton and ChangeAimbotKeyButton.AbsolutePosition
            if buttonPos then
                AimbotKeyMenu.Position = UDim2.new(0, buttonPos.X - 150, 0, buttonPos.Y + 35)
            end
        end
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        draggingPanel = false
    end
end)

MinimizedFrame = Instance.new("Frame", ScreenGui)
MinimizedFrame.Size = UDim2.new(0, 50, 0, 50)
MinimizedFrame.Position = UDim2.new(0.5, -25, 0.5, -25)
MinimizedFrame.BackgroundColor3 = currentTitleColor
MinimizedFrame.BorderSizePixel = 0
MinimizedFrame.Visible = false
MinimizedFrame.Active = true

local MinimizedCorner = Instance.new("UICorner", MinimizedFrame)
MinimizedCorner.CornerRadius = UDim.new(0, 25)

local MinimizedGhost = Instance.new("TextLabel", MinimizedFrame)
MinimizedGhost.Text = "🪐"
MinimizedGhost.Size = UDim2.new(1, 0, 1, 0)
MinimizedGhost.BackgroundTransparency = 1
MinimizedGhost.TextColor3 = Color3.fromRGB(255, 140, 0)
MinimizedGhost.Font = Enum.Font.GothamBlack
MinimizedGhost.TextSize = 30
MinimizedGhost.TextScaled = true
MinimizedGhost.Active = false

MinimizeButton.MouseButton1Click:Connect(toggleMinimize)

local draggingMinimized = false
local dragStartPosMin
local panelStartPosMin

MinimizedFrame.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        draggingMinimized = true
        dragStartPosMin = input.Position
        panelStartPosMin = MinimizedFrame.Position
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if draggingMinimized and input.UserInputType == Enum.UserInputType.MouseMovement then
        local delta = input.Position - dragStartPosMin
        MinimizedFrame.Position = UDim2.new(panelStartPosMin.X.Scale, panelStartPosMin.X.Offset + delta.X,
                                            panelStartPosMin.Y.Scale, panelStartPosMin.Y.Offset + delta.Y)
        savedPosition = MinimizedFrame.Position
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        if draggingMinimized then
            local delta = input.Position - dragStartPosMin
            if math.abs(delta.X) < 5 and math.abs(delta.Y) < 5 then
                toggleMinimize()
            end
        end
        draggingMinimized = false
    end
end)

-- ====================== CONTEÚDO DAS ABAS ======================
local TabContent = Instance.new("Frame", MainFrame)
TabContent.Size = UDim2.new(1, -20, 0, 250)
TabContent.Position = UDim2.new(0, 10, 0, 80)
TabContent.BackgroundTransparency = 1
TabContent.BorderSizePixel = 0

local ESPTabFrame = Instance.new("Frame", TabContent)
ESPTabFrame.Size = UDim2.new(1, 0, 1, 0)
ESPTabFrame.Position = UDim2.new(0, 0, 0, 0)
ESPTabFrame.BackgroundTransparency = 1
ESPTabFrame.Visible = true

local ExtraTabFrame = Instance.new("Frame", TabContent)
ExtraTabFrame.Size = UDim2.new(1, 0, 1, 0)
ExtraTabFrame.Position = UDim2.new(0, 0, 0, 0)
ExtraTabFrame.BackgroundTransparency = 1
ExtraTabFrame.Visible = false

local AimbotTabFrame = Instance.new("Frame", TabContent)
AimbotTabFrame.Size = UDim2.new(1, 0, 1, 0)
AimbotTabFrame.Position = UDim2.new(0, 0, 0, 0)
AimbotTabFrame.BackgroundTransparency = 1
AimbotTabFrame.Visible = false

local AimColorTabFrame = Instance.new("Frame", TabContent)
AimColorTabFrame.Size = UDim2.new(1, 0, 1, 0)
AimColorTabFrame.Position = UDim2.new(0, 0, 0, 0)
AimColorTabFrame.BackgroundTransparency = 1
AimColorTabFrame.Visible = false

local ESPListLayout = Instance.new("UIListLayout", ESPTabFrame)
ESPListLayout.Padding = UDim.new(0, 6)
ESPListLayout.FillDirection = Enum.FillDirection.Vertical
ESPListLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
ESPListLayout.VerticalAlignment = Enum.VerticalAlignment.Top

local ExtraListLayout = Instance.new("UIListLayout", ExtraTabFrame)
ExtraListLayout.Padding = UDim.new(0, 6)
ExtraListLayout.FillDirection = Enum.FillDirection.Vertical
ExtraListLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
ExtraListLayout.VerticalAlignment = Enum.VerticalAlignment.Top

local AimbotLeftFrame = Instance.new("Frame", AimbotTabFrame)
AimbotLeftFrame.Size = UDim2.new(0.5, -5, 0, 220)
AimbotLeftFrame.Position = UDim2.new(0, 0, 0, 0)
AimbotLeftFrame.BackgroundTransparency = 1
AimbotLeftFrame.BorderSizePixel = 0
local AimbotLeftLayout = Instance.new("UIListLayout", AimbotLeftFrame)
AimbotLeftLayout.Padding = UDim.new(0, 6)
AimbotLeftLayout.FillDirection = Enum.FillDirection.Vertical
AimbotLeftLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
AimbotLeftLayout.VerticalAlignment = Enum.VerticalAlignment.Top

local AimbotRightFrame = Instance.new("Frame", AimbotTabFrame)
AimbotRightFrame.Size = UDim2.new(0.5, -5, 0, 220)
AimbotRightFrame.Position = UDim2.new(0.5, 5, 0, 0)
AimbotRightFrame.BackgroundTransparency = 1
AimbotRightFrame.BorderSizePixel = 0
local AimbotRightLayout = Instance.new("UIListLayout", AimbotRightFrame)
AimbotRightLayout.Padding = UDim.new(0, 6)
AimbotRightLayout.FillDirection = Enum.FillDirection.Vertical
AimbotRightLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
AimbotRightLayout.VerticalAlignment = Enum.VerticalAlignment.Top

local AimColorLeftFrame = Instance.new("Frame", AimColorTabFrame)
AimColorLeftFrame.Size = UDim2.new(0.5, -5, 0, 220)
AimColorLeftFrame.Position = UDim2.new(0, 0, 0, 0)
AimColorLeftFrame.BackgroundTransparency = 1
AimColorLeftFrame.BorderSizePixel = 0
local AimColorLeftLayout = Instance.new("UIListLayout", AimColorLeftFrame)
AimColorLeftLayout.Padding = UDim.new(0, 6)
AimColorLeftLayout.FillDirection = Enum.FillDirection.Vertical
AimColorLeftLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
AimColorLeftLayout.VerticalAlignment = Enum.VerticalAlignment.Top

local AimColorRightFrame = Instance.new("Frame", AimColorTabFrame)
AimColorRightFrame.Size = UDim2.new(0.5, -5, 0, 220)
AimColorRightFrame.Position = UDim2.new(0.5, 5, 0, 0)
AimColorRightFrame.BackgroundTransparency = 1
AimColorRightFrame.BorderSizePixel = 0
local AimColorRightLayout = Instance.new("UIListLayout", AimColorRightFrame)
AimColorRightLayout.Padding = UDim.new(0, 6)
AimColorRightLayout.FillDirection = Enum.FillDirection.Vertical
AimColorRightLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
AimColorRightLayout.VerticalAlignment = Enum.VerticalAlignment.Top

-- ====================== PAINÉIS INFERIORES ======================
local KeyPanel = Instance.new("Frame", MainFrame)
KeyPanel.Size = UDim2.new(1, -20, 0, 45)
KeyPanel.Position = UDim2.new(0, 10, 1, -50)
KeyPanel.BackgroundColor3 = COLORS.SECONDARY
KeyPanel.BackgroundTransparency = 0.5
KeyPanel.BorderSizePixel = 0
KeyPanel.ZIndex = 1

local KeyPanelCorner = Instance.new("UICorner", KeyPanel)
KeyPanelCorner.CornerRadius = UDim.new(0, 10)

local KeyLabel = Instance.new("TextLabel", KeyPanel)
KeyLabel.Text = "Tecla para minimizar:"
KeyLabel.Size = UDim2.new(0, 150, 1, 0)
KeyLabel.Position = UDim2.new(0, 10, 0, 0)
KeyLabel.BackgroundTransparency = 1
KeyLabel.TextColor3 = COLORS.TEXT
KeyLabel.Font = Enum.Font.GothamBlack
KeyLabel.TextSize = 11
KeyLabel.TextXAlignment = Enum.TextXAlignment.Left

CurrentKeyButton = Instance.new("TextButton", KeyPanel)
CurrentKeyButton.Size = UDim2.new(0, 80, 0, 28)
CurrentKeyButton.Position = UDim2.new(0, 165, 0.5, -14)
CurrentKeyButton.BackgroundColor3 = currentTitleColor
CurrentKeyButton.BorderSizePixel = 0
CurrentKeyButton.Text = "Insert"
CurrentKeyButton.TextColor3 = Color3.new(1, 1, 1)
CurrentKeyButton.Font = Enum.Font.GothamBlack
CurrentKeyButton.TextSize = 11

local CurrentKeyCorner = Instance.new("UICorner", CurrentKeyButton)
CurrentKeyCorner.CornerRadius = UDim.new(0, 6)

ChangeKeyButton = Instance.new("TextButton", KeyPanel)
ChangeKeyButton.Size = UDim2.new(0, 60, 0, 28)
ChangeKeyButton.Position = UDim2.new(0, 255, 0.5, -14)
ChangeKeyButton.BackgroundColor3 = currentTitleColor
ChangeKeyButton.BorderSizePixel = 0
ChangeKeyButton.Text = "Mudar"
ChangeKeyButton.TextColor3 = Color3.new(1, 1, 1)
ChangeKeyButton.Font = Enum.Font.GothamBlack
ChangeKeyButton.TextSize = 11

local ChangeKeyCorner = Instance.new("UICorner", ChangeKeyButton)
ChangeKeyCorner.CornerRadius = UDim.new(0, 6)

local AimbotKeyPanel = Instance.new("Frame", MainFrame)
AimbotKeyPanel.Size = UDim2.new(1, -20, 0, 45)
AimbotKeyPanel.Position = UDim2.new(0, 10, 1, -50)
AimbotKeyPanel.BackgroundColor3 = COLORS.SECONDARY
AimbotKeyPanel.BackgroundTransparency = 0.5
AimbotKeyPanel.BorderSizePixel = 0

local AimbotKeyPanelCorner = Instance.new("UICorner", AimbotKeyPanel)
AimbotKeyPanelCorner.CornerRadius = UDim.new(0, 10)

local AimbotKeyLabel = Instance.new("TextLabel", AimbotKeyPanel)
AimbotKeyLabel.Text = "Tecla trocar alvo:"
AimbotKeyLabel.Size = UDim2.new(0, 150, 1, 0)
AimbotKeyLabel.Position = UDim2.new(0, 10, 0, 0)
AimbotKeyLabel.BackgroundTransparency = 1
AimbotKeyLabel.TextColor3 = COLORS.TEXT
AimbotKeyLabel.Font = Enum.Font.GothamBlack
AimbotKeyLabel.TextSize = 11
AimbotKeyLabel.TextXAlignment = Enum.TextXAlignment.Left

CurrentAimbotKeyButton = Instance.new("TextButton", AimbotKeyPanel)
CurrentAimbotKeyButton.Size = UDim2.new(0, 80, 0, 28)
CurrentAimbotKeyButton.Position = UDim2.new(0, 165, 0.5, -14)
CurrentAimbotKeyButton.BackgroundColor3 = currentTitleColor
CurrentAimbotKeyButton.BorderSizePixel = 0
CurrentAimbotKeyButton.Text = "T"
CurrentAimbotKeyButton.TextColor3 = Color3.new(1, 1, 1)
CurrentAimbotKeyButton.Font = Enum.Font.GothamBlack
CurrentAimbotKeyButton.TextSize = 11

local CurrentAimbotKeyCorner = Instance.new("UICorner", CurrentAimbotKeyButton)
CurrentAimbotKeyCorner.CornerRadius = UDim.new(0, 6)

ChangeAimbotKeyButton = Instance.new("TextButton", AimbotKeyPanel)
ChangeAimbotKeyButton.Size = UDim2.new(0, 60, 0, 28)
ChangeAimbotKeyButton.Position = UDim2.new(0, 255, 0.5, -14)
ChangeAimbotKeyButton.BackgroundColor3 = currentTitleColor
ChangeAimbotKeyButton.BorderSizePixel = 0
ChangeAimbotKeyButton.Text = "Mudar"
ChangeAimbotKeyButton.TextColor3 = Color3.new(1, 1, 1)
ChangeAimbotKeyButton.Font = Enum.Font.GothamBlack
ChangeAimbotKeyButton.TextSize = 11

local ChangeAimbotKeyCorner = Instance.new("UICorner", ChangeAimbotKeyButton)
ChangeAimbotKeyCorner.CornerRadius = UDim.new(0, 6)

local ColorPanel = Instance.new("Frame", MainFrame)
ColorPanel.Size = UDim2.new(1, -20, 0, 45)
ColorPanel.Position = UDim2.new(0, 10, 1, -100)
ColorPanel.BackgroundColor3 = COLORS.SECONDARY
ColorPanel.BackgroundTransparency = 0.5
ColorPanel.BorderSizePixel = 0

local ColorPanelCorner = Instance.new("UICorner", ColorPanel)
ColorPanelCorner.CornerRadius = UDim.new(0, 10)

local ColorLabel = Instance.new("TextLabel", ColorPanel)
ColorLabel.Text = "Cor do Topo:"
ColorLabel.Size = UDim2.new(0, 80, 1, 0)
ColorLabel.Position = UDim2.new(0, 10, 0, 0)
ColorLabel.BackgroundTransparency = 1
ColorLabel.TextColor3 = COLORS.TEXT
ColorLabel.Font = Enum.Font.GothamBlack
ColorLabel.TextSize = 11
ColorLabel.TextXAlignment = Enum.TextXAlignment.Left

CurrentColorButton = Instance.new("TextButton", ColorPanel)
CurrentColorButton.Size = UDim2.new(0, 50, 0, 28)
CurrentColorButton.Position = UDim2.new(0, 95, 0.5, -14)
CurrentColorButton.BackgroundColor3 = currentTitleColor
CurrentColorButton.BorderSizePixel = 0
CurrentColorButton.Text = ""
CurrentColorButton.AutoButtonColor = false

local CurrentColorCorner = Instance.new("UICorner", CurrentColorButton)
CurrentColorCorner.CornerRadius = UDim.new(0, 6)

ChangeColorButton = Instance.new("TextButton", ColorPanel)
ChangeColorButton.Size = UDim2.new(0, 60, 0, 28)
ChangeColorButton.Position = UDim2.new(0, 155, 0.5, -14)
ChangeColorButton.BackgroundColor3 = currentTitleColor
ChangeColorButton.BorderSizePixel = 0
ChangeColorButton.Text = "Mudar"
ChangeColorButton.TextColor3 = Color3.new(1, 1, 1)
ChangeColorButton.Font = Enum.Font.GothamBlack
ChangeColorButton.TextSize = 11

local ChangeColorCorner = Instance.new("UICorner", ChangeColorButton)
ChangeColorCorner.CornerRadius = UDim.new(0, 6)

local function updatePanelsVisibility(tabName)
    if tabName == "Aimbot" or tabName == "AimColor" then
        KeyPanel.Visible = false
        ColorPanel.Visible = false
        AimbotKeyPanel.Visible = true
    else
        KeyPanel.Visible = true
        ColorPanel.Visible = true
        AimbotKeyPanel.Visible = false
    end
end

local function showTab(tabName)
    ESPTabFrame.Visible = (tabName == "ESP")
    ExtraTabFrame.Visible = (tabName == "Extra")
    AimbotTabFrame.Visible = (tabName == "Aimbot")
    AimColorTabFrame.Visible = (tabName == "AimColor")
    ESPTabButton.BackgroundColor3 = (tabName == "ESP") and currentTitleColor or Color3.fromRGB(20, 20, 20)
    ExtraTabButton.BackgroundColor3 = (tabName == "Extra") and currentTitleColor or Color3.fromRGB(20, 20, 20)
    AimbotTabButton.BackgroundColor3 = (tabName == "Aimbot") and currentTitleColor or Color3.fromRGB(20, 20, 20)
    AimColorTabButton.BackgroundColor3 = (tabName == "AimColor") and currentTitleColor or Color3.fromRGB(20, 20, 20)
    updatePanelsVisibility(tabName)
end

local TabBar = Instance.new("Frame", MainFrame)
TabBar.Size = UDim2.new(1, 0, 0, 30)
TabBar.Position = UDim2.new(0, 0, 0, 45)
TabBar.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
TabBar.BackgroundTransparency = 0.5
TabBar.BorderSizePixel = 0
TabBar.ZIndex = 100
TabBar.Active = true

local function createTabButton(parent, text, position, onClick)
    local button = Instance.new("TextButton", parent)
    button.Size = UDim2.new(0, 100, 1, 0)
    button.Position = position
    button.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
    button.BackgroundTransparency = 0.3
    button.BorderSizePixel = 0
    button.Text = text
    button.TextColor3 = COLORS.TEXT
    button.Font = Enum.Font.GothamBlack
    button.TextSize = 12
    button.AutoButtonColor = false
    button.ZIndex = 101
    button.Active = true
    local corner = Instance.new("UICorner", button)
    corner.CornerRadius = UDim.new(0, 8)
    button.MouseButton1Click:Connect(onClick)
    button.MouseButton1Down:Connect(onClick)
    return button
end

ESPTabButton = createTabButton(TabBar, "ESP", UDim2.new(0, 10, 0, 0), function() showTab("ESP") end)
ExtraTabButton = createTabButton(TabBar, "Extra", UDim2.new(0, 120, 0, 0), function() showTab("Extra") end)
AimColorTabButton = createTabButton(TabBar, "AimColor", UDim2.new(0, 230, 0, 0), function() showTab("AimColor") end)
AimbotTabButton = createTabButton(TabBar, "Aimbot", UDim2.new(0, 340, 0, 0), function() showTab("Aimbot") end)

showTab("ESP")

-- ====================== MENUS DE TECLA E COR ======================
KeyMenu = Instance.new("Frame", ScreenGui)
KeyMenu.Size = UDim2.new(0, 250, 0, 120)
KeyMenu.BackgroundColor3 = COLORS.PRIMARY
KeyMenu.BorderSizePixel = 0
KeyMenu.BackgroundTransparency = 0.5
KeyMenu.Visible = false
KeyMenu.ZIndex = 10

local KeyMenuCorner = Instance.new("UICorner", KeyMenu)
KeyMenuCorner.CornerRadius = UDim.new(0, 8)

local KeyMenuTitle = Instance.new("TextLabel", KeyMenu)
KeyMenuTitle.Text = "Pressione uma tecla..."
KeyMenuTitle.Size = UDim2.new(1, 0, 0, 40)
KeyMenuTitle.Position = UDim2.new(0, 0, 0, 0)
KeyMenuTitle.BackgroundColor3 = currentTitleColor
KeyMenuTitle.TextColor3 = Color3.new(1, 1, 1)
KeyMenuTitle.Font = Enum.Font.GothamBlack
KeyMenuTitle.TextSize = 14
KeyMenuTitle.TextXAlignment = Enum.TextXAlignment.Center

local KeyMenuCorner2 = Instance.new("UICorner", KeyMenuTitle)
KeyMenuCorner2.CornerRadius = UDim.new(0, 8)

local KeyMenuSub = Instance.new("TextLabel", KeyMenu)
KeyMenuSub.Text = "Aguardando tecla..."
KeyMenuSub.Size = UDim2.new(1, 0, 0, 30)
KeyMenuSub.Position = UDim2.new(0, 0, 0, 50)
KeyMenuSub.BackgroundTransparency = 1
KeyMenuSub.TextColor3 = COLORS.TEXT
KeyMenuSub.Font = Enum.Font.GothamBlack
KeyMenuSub.TextSize = 12
KeyMenuSub.TextXAlignment = Enum.TextXAlignment.Center

AimbotKeyMenu = Instance.new("Frame", ScreenGui)
AimbotKeyMenu.Size = UDim2.new(0, 250, 0, 120)
AimbotKeyMenu.BackgroundColor3 = COLORS.PRIMARY
AimbotKeyMenu.BorderSizePixel = 0
AimbotKeyMenu.BackgroundTransparency = 0.5
AimbotKeyMenu.Visible = false
AimbotKeyMenu.ZIndex = 10

local AimbotKeyMenuCorner = Instance.new("UICorner", AimbotKeyMenu)
AimbotKeyMenuCorner.CornerRadius = UDim.new(0, 8)

local AimbotKeyMenuTitle = Instance.new("TextLabel", AimbotKeyMenu)
AimbotKeyMenuTitle.Text = "Pressione uma tecla..."
AimbotKeyMenuTitle.Size = UDim2.new(1, 0, 0, 40)
AimbotKeyMenuTitle.Position = UDim2.new(0, 0, 0, 0)
AimbotKeyMenuTitle.BackgroundColor3 = currentTitleColor
AimbotKeyMenuTitle.TextColor3 = Color3.new(1, 1, 1)
AimbotKeyMenuTitle.Font = Enum.Font.GothamBlack
AimbotKeyMenuTitle.TextSize = 14
AimbotKeyMenuTitle.TextXAlignment = Enum.TextXAlignment.Center

local AimbotKeyMenuCorner2 = Instance.new("UICorner", AimbotKeyMenuTitle)
AimbotKeyMenuCorner2.CornerRadius = UDim.new(0, 8)

local AimbotKeyMenuSub = Instance.new("TextLabel", AimbotKeyMenu)
AimbotKeyMenuSub.Text = "Aguardando tecla..."
AimbotKeyMenuSub.Size = UDim2.new(1, 0, 0, 30)
AimbotKeyMenuSub.Position = UDim2.new(0, 0, 0, 50)
AimbotKeyMenuSub.BackgroundTransparency = 1
AimbotKeyMenuSub.TextColor3 = COLORS.TEXT
AimbotKeyMenuSub.Font = Enum.Font.GothamBlack
AimbotKeyMenuSub.TextSize = 12
AimbotKeyMenuSub.TextXAlignment = Enum.TextXAlignment.Center

local waitingForKey = false
local keyConnection = nil

local function closeKeyMenu()
    KeyMenu.Visible = false
    if keyConnection then keyConnection:Disconnect(); keyConnection = nil end
    waitingForKey = false
end

ChangeKeyButton.MouseButton1Click:Connect(function()
    if KeyMenu.Visible then
        closeKeyMenu()
    else
        KeyMenu.Visible = true
        if keyConnection then keyConnection:Disconnect() end
        waitingForKey = true
        KeyMenuTitle.Text = "Pressione uma tecla..."
        KeyMenuTitle.BackgroundColor3 = Color3.fromRGB(255, 140, 0)
        keyConnection = UserInputService.InputBegan:Connect(function(input, gameProcessed)
            if gameProcessed then return end
            if waitingForKey and input.KeyCode ~= Enum.KeyCode.Unknown then
                toggleKey = input.KeyCode
                CurrentKeyButton.Text = tostring(input.KeyCode):gsub("Enum.KeyCode.", "")
                KeyMenuTitle.Text = "Tecla selecionada!"
                KeyMenuTitle.BackgroundColor3 = currentTitleColor
                waitingForKey = false
                task.wait(0.5)
                closeKeyMenu()
            end
        end)
    end
end)

local waitingForAimbotKey = false
local aimbotKeyConnection = nil

local function closeAimbotKeyMenu()
    AimbotKeyMenu.Visible = false
    if aimbotKeyConnection then aimbotKeyConnection:Disconnect(); aimbotKeyConnection = nil end
    waitingForAimbotKey = false
end

ChangeAimbotKeyButton.MouseButton1Click:Connect(function()
    if AimbotKeyMenu.Visible then
        closeAimbotKeyMenu()
    else
        AimbotKeyMenu.Visible = true
        if aimbotKeyConnection then aimbotKeyConnection:Disconnect() end
        waitingForAimbotKey = true
        AimbotKeyMenuTitle.Text = "Pressione uma tecla..."
        AimbotKeyMenuTitle.BackgroundColor3 = Color3.fromRGB(255, 140, 0)
        aimbotKeyConnection = UserInputService.InputBegan:Connect(function(input, gameProcessed)
            if gameProcessed then return end
            if waitingForAimbotKey and input.KeyCode ~= Enum.KeyCode.Unknown then
                switchTargetKey = input.KeyCode
                CurrentAimbotKeyButton.Text = tostring(input.KeyCode):gsub("Enum.KeyCode.", "")
                AimbotKeyMenuTitle.Text = "Tecla selecionada!"
                AimbotKeyMenuTitle.BackgroundColor3 = currentTitleColor
                waitingForAimbotKey = false
                task.wait(0.5)
                closeAimbotKeyMenu()
            end
        end)
    end
end)

local function handleToggleKey()
    if anonimatoEnabled then
        local currentTime = tick()
        if currentTime - lastClickTime > 1 then clickCount = 0 end
        clickCount = clickCount + 1
        lastClickTime = currentTime
        if clickCount >= 3 then toggleMinimize(); clickCount = 0 end
    else
        toggleMinimize()
    end
end

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then mouse1Down = true end
    if gameProcessed then return end
    if input.KeyCode == toggleKey then handleToggleKey() end
    if input.KeyCode == switchTargetKey then
        currentTargetIndex = currentTargetIndex % #targetPartsList + 1
        targetPart = targetPartsList[currentTargetIndex]
    end
    if tpEnabled and input.UserInputType == Enum.UserInputType.MouseButton1 then
        local ctrlPressed = UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) or UserInputService:IsKeyDown(Enum.KeyCode.RightControl)
        if ctrlPressed then teleportToMouse() end
    end
end)

UserInputService.InputEnded:Connect(function(input, gameProcessed)
    if input.UserInputType == Enum.UserInputType.MouseButton1 and not ignoreMouse1Release then
        mouse1Down = false
    end
end)

-- ====================== SEÇÕES E ELEMENTOS ======================
local colorMenuTitle = nil
local fovCircle = nil
local aimColorFovCircle = nil
local usingRgb = false
local rgbHue = 0
local rgbConnection = nil

local function updateAllDynamicColors(color)
    TitleBar.BackgroundColor3 = color
    MinimizedFrame.BackgroundColor3 = color
    CurrentColorButton.BackgroundColor3 = color
    ChangeColorButton.BackgroundColor3 = color
    CurrentKeyButton.BackgroundColor3 = color
    ChangeKeyButton.BackgroundColor3 = color
    CurrentAimbotKeyButton.BackgroundColor3 = color
    ChangeAimbotKeyButton.BackgroundColor3 = color
    MinimizeButton.BackgroundColor3 = Color3.new(color.R * 0.7, color.G * 0.7, color.B * 0.7)
    if colorMenuTitle then colorMenuTitle.BackgroundColor3 = color end
    if KeyMenuTitle then KeyMenuTitle.BackgroundColor3 = color end
    if AimbotKeyMenuTitle then AimbotKeyMenuTitle.BackgroundColor3 = color end
    for _, button in pairs(selectorButtons) do
        pcall(function() button.BackgroundColor3 = color end)
    end
    for _, sliderFill in pairs(sliderFills) do
        pcall(function() sliderFill.BackgroundColor3 = color end)
    end
    if fovCircle then pcall(function() fovCircle.Color = color end) end
    if aimColorFovCircle then pcall(function() aimColorFovCircle.Color = color end) end
end

local function startRgbMode()
    usingRgb = true
    if rgbConnection then rgbConnection:Disconnect() end
    rgbConnection = RunService.RenderStepped:Connect(function()
        rgbHue = (rgbHue + 0.002) % 1
        local rgbColor = Color3.fromHSV(rgbHue, 1, 1)
        updateAllDynamicColors(rgbColor)
        if fovCircle then fovCircle.Color = rgbColor end
        if aimColorFovCircle then aimColorFovCircle.Color = rgbColor end
    end)
end

local function stopRgbMode()
    if rgbConnection then rgbConnection:Disconnect(); rgbConnection = nil end
    usingRgb = false
end

ColorMenu = Instance.new("Frame", ScreenGui)
ColorMenu.Size = UDim2.new(0, 220, 0, 400)
ColorMenu.BackgroundColor3 = COLORS.PRIMARY
ColorMenu.BorderSizePixel = 0
ColorMenu.BackgroundTransparency = 0.5
ColorMenu.Visible = false
ColorMenu.ZIndex = 10

local ColorMenuCorner = Instance.new("UICorner", ColorMenu)
ColorMenuCorner.CornerRadius = UDim.new(0, 8)

colorMenuTitle = Instance.new("TextLabel", ColorMenu)
colorMenuTitle.Text = "Selecione uma cor"
colorMenuTitle.Size = UDim2.new(1, 0, 0, 30)
colorMenuTitle.BackgroundColor3 = currentTitleColor
colorMenuTitle.TextColor3 = Color3.new(1, 1, 1)
colorMenuTitle.Font = Enum.Font.GothamBlack
colorMenuTitle.TextSize = 14
colorMenuTitle.TextXAlignment = Enum.TextXAlignment.Center

local ColorMenuCorner2 = Instance.new("UICorner", colorMenuTitle)
ColorMenuCorner2.CornerRadius = UDim.new(0, 8)

local ColorList = Instance.new("ScrollingFrame", ColorMenu)
ColorList.Size = UDim2.new(1, -10, 1, -40)
ColorList.Position = UDim2.new(0, 5, 0, 35)
ColorList.BackgroundTransparency = 1
ColorList.BorderSizePixel = 0
ColorList.ScrollBarThickness = 4
ColorList.ScrollingEnabled = true

local ColorListLayout = Instance.new("UIListLayout", ColorList)
ColorListLayout.Padding = UDim.new(0, 5)
ColorListLayout.SortOrder = Enum.SortOrder.LayoutOrder

for i, colorData in ipairs(TITLE_COLORS) do
    local colorButton = Instance.new("TextButton", ColorList)
    colorButton.Size = UDim2.new(1, -10, 0, 35)
    colorButton.BackgroundColor3 = colorData.Cor or DEFAULT_ACCENT
    colorButton.BorderSizePixel = 0
    colorButton.Text = colorData.Nome
    colorButton.TextColor3 = (colorData.Nome == "Preto" or colorData.Nome == "Roxo") and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(0, 0, 0)
    colorButton.Font = Enum.Font.GothamBlack
    colorButton.TextSize = 12
    local colorCorner = Instance.new("UICorner", colorButton)
    colorCorner.CornerRadius = UDim.new(0, 4)
    colorButton.MouseButton1Click:Connect(function()
        if colorData.Nome == "RGB" then startRgbMode() else stopRgbMode(); updateAllDynamicColors(colorData.Cor) end
        ColorMenu.Visible = false
    end)
end

task.wait()
ColorList.CanvasSize = UDim2.new(0, 0, 0, ColorListLayout.AbsoluteContentSize.Y + 10)

ChangeColorButton.MouseButton1Click:Connect(function()
    ColorMenu.Visible = not ColorMenu.Visible
    if ColorMenu.Visible then
        local buttonPos = ChangeColorButton.AbsolutePosition
        ColorMenu.Position = UDim2.new(0, buttonPos.X - 150, 0, buttonPos.Y - 200)
    end
end)

local function createToggleButton(parent, name, getState, onToggle)
    local button = Instance.new("Frame", parent)
    button.Size = UDim2.new(1, 0, 0, 32)
    button.BackgroundColor3 = COLORS.SECONDARY
    button.BackgroundTransparency = 0.5
    button.BorderSizePixel = 0
    local UICorner = Instance.new("UICorner", button)
    UICorner.CornerRadius = UDim.new(0, 6)
    local statusIndicator = Instance.new("Frame", button)
    statusIndicator.Size = UDim2.new(0, 14, 0, 14)
    statusIndicator.Position = UDim2.new(0, 8, 0.5, -7)
    statusIndicator.BorderSizePixel = 0
    local statusCorner = Instance.new("UICorner", statusIndicator)
    statusCorner.CornerRadius = UDim.new(0, 7)
    local label = Instance.new("TextLabel", button)
    label.Text = name
    label.Size = UDim2.new(1, -40, 1, 0)
    label.Position = UDim2.new(0, 30, 0, 0)
    label.BackgroundTransparency = 1
    label.TextColor3 = Color3.fromRGB(255, 255, 255)
    label.Font = Enum.Font.GothamBlack
    label.TextSize = 13
    label.TextXAlignment = Enum.TextXAlignment.Left
    local hitBox = Instance.new("TextButton", button)
    hitBox.Size = UDim2.new(1, 0, 1, 0)
    hitBox.BackgroundTransparency = 1
    hitBox.Text = ""
    local function updateColor()
        statusIndicator.BackgroundColor3 = getState() and COLORS.ENABLED or COLORS.DISABLED
    end
    hitBox.MouseButton1Click:Connect(function()
        onToggle()
        updateColor()
    end)
    updateColor()
    return button
end

local function createSlider(parent, name, min, max, default, onChanged)
    local sliderContainer = Instance.new("Frame", parent)
    sliderContainer.Size = UDim2.new(1, 0, 0, 48)
    sliderContainer.BackgroundColor3 = COLORS.SECONDARY
    sliderContainer.BackgroundTransparency = 0.5
    sliderContainer.BorderSizePixel = 0
    local UICorner = Instance.new("UICorner", sliderContainer)
    UICorner.CornerRadius = UDim.new(0, 6)
    local label = Instance.new("TextLabel", sliderContainer)
    label.Text = name .. ": " .. default
    label.Size = UDim2.new(1, -20, 0, 18)
    label.Position = UDim2.new(0, 10, 0, 5)
    label.BackgroundTransparency = 1
    label.TextColor3 = Color3.fromRGB(255, 255, 255)
    label.Font = Enum.Font.GothamBlack
    label.TextSize = 12
    label.TextXAlignment = Enum.TextXAlignment.Left
    local sliderBg = Instance.new("Frame", sliderContainer)
    sliderBg.Size = UDim2.new(1, -20, 0, 5)
    sliderBg.Position = UDim2.new(0, 10, 0, 28)
    sliderBg.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
    sliderBg.BorderSizePixel = 0
    local sliderBgCorner = Instance.new("UICorner", sliderBg)
    sliderBgCorner.CornerRadius = UDim.new(0, 3)
    local sliderFill = Instance.new("Frame", sliderBg)
    sliderFill.Size = UDim2.new((default - min) / (max - min), 0, 1, 0)
    sliderFill.BackgroundColor3 = currentTitleColor
    sliderFill.BorderSizePixel = 0
    table.insert(sliderFills, sliderFill)
    local sliderFillCorner = Instance.new("UICorner", sliderFill)
    sliderFillCorner.CornerRadius = UDim.new(0, 3)
    local currentValue = default
    local mouseDown = false
    local inputConn = nil
    local endConn = nil

    local function updateSlider(input)
        local pos = math.clamp((input.Position.X - sliderBg.AbsolutePosition.X) / sliderBg.AbsoluteSize.X, 0, 1)
        local value = math.floor(min + (max - min) * pos)
        currentValue = value
        sliderFill.Size = UDim2.new(pos, 0, 1, 0)
        label.Text = name .. ": " .. value
        onChanged(value)
    end

    sliderBg.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            mouseDown = true
            updateSlider(input)
            if inputConn then inputConn:Disconnect() end
            if endConn then endConn:Disconnect() end
            inputConn = UserInputService.InputChanged:Connect(function(input)
                if mouseDown and input.UserInputType == Enum.UserInputType.MouseMovement then
                    updateSlider(input)
                end
            end)
            endConn = UserInputService.InputEnded:Connect(function(input)
                if input.UserInputType == Enum.UserInputType.MouseButton1 then
                    mouseDown = false
                    if inputConn then inputConn:Disconnect(); inputConn = nil end
                    if endConn then endConn:Disconnect(); endConn = nil end
                end
            end)
        end
    end)

    return sliderContainer
end

-- ====================== ESP / GLOW / LIFE ======================
local function cleanupLifeESP(player)
    local data = lifeObjects[player]
    if data then
        if data.billboard then pcall(function() data.billboard:Destroy() end) end
        if data.healthConnection then pcall(function() data.healthConnection:Disconnect() end) end
        if data.deathConnection then pcall(function() data.deathConnection:Disconnect() end) end
        if data.waitingConn then pcall(function() data.waitingConn:Disconnect() end) end
        if data.characterConn then pcall(function() data.characterConn:Disconnect() end) end
        if data.cameraConnection then pcall(function() data.cameraConnection:Disconnect() end) end
        lifeObjects[player] = nil
    end
end

local function updateLifeVisual(player)
    local data = lifeObjects[player]
    if not data or not data.billboard or not data.humanoid or not data.rootPart then return end
    local humanoid = data.humanoid
    local rootPart = data.rootPart
    if not humanoid.Parent or not rootPart.Parent then return end
    local health = humanoid.Health
    local maxHealth = humanoid.MaxHealth
    local percent = math.max(0, math.min(1, health / maxHealth))
    local distance = (Camera.CFrame.Position - rootPart.Position).Magnitude
    local fontSize = getLifeFontSizeByDistance(distance)
    local scale = fontSize / 18
    data.billboard.Size = UDim2.new(0, 60 * scale, 0, 22 * scale)
    data.healthText.TextSize = fontSize
    data.healthText.Text = string.format("%.0f", health)
    data.healthText.TextColor3 = Color3.fromRGB(0, 255, 0)
    if percent > 0.6 then
        data.healthFill.BackgroundColor3 = Color3.fromRGB(0, 255, 0)
    elseif percent > 0.3 then
        data.healthFill.BackgroundColor3 = Color3.fromRGB(255, 200, 0)
    else
        data.healthFill.BackgroundColor3 = Color3.fromRGB(255, 0, 0)
    end
    data.healthFill.Size = UDim2.new(percent, 0, 1, 0)
end

local function updateLifeESP(player)
    cleanupLifeESP(player)
    if not espEnabled or not espLifeEnabled or player == LocalPlayer then return end
    local character = player.Character
    if not character or not character.Parent then
        local charConn = player.CharacterAdded:Connect(function()
            charConn:Disconnect()
            task.wait(0.3)
            updateLifeESP(player)
        end)
        lifeObjects[player] = {waitingConn = charConn}
        return
    end
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    local rootPart = character:FindFirstChild("HumanoidRootPart")
    if not humanoid or not rootPart then return end

    local billboard = Instance.new("BillboardGui")
    billboard.Size = UDim2.new(0, 60, 0, 22)
    billboard.StudsOffset = Vector3.new(0, -3.5, 0)
    billboard.AlwaysOnTop = true
    billboard.Parent = rootPart

    local mainFrame = Instance.new("Frame", billboard)
    mainFrame.Size = UDim2.new(1, 0, 1, 0)
    mainFrame.BackgroundTransparency = 0.85
    mainFrame.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    local mainCorner = Instance.new("UICorner", mainFrame)
    mainCorner.CornerRadius = UDim.new(1, 0)

    local healthText = Instance.new("TextLabel", mainFrame)
    healthText.Size = UDim2.new(1, 0, 0.8, 0)
    healthText.Position = UDim2.new(0, 0, 0.1, 0)
    healthText.BackgroundTransparency = 1
    healthText.TextColor3 = Color3.fromRGB(0, 255, 0)
    healthText.TextSize = 16
    healthText.Font = Enum.Font.GothamBlack
    healthText.TextXAlignment = Enum.TextXAlignment.Center
    healthText.TextYAlignment = Enum.TextYAlignment.Center

    local healthBar = Instance.new("Frame", mainFrame)
    healthBar.Size = UDim2.new(1, -8, 0, 2)
    healthBar.Position = UDim2.new(0, 4, 1, -4)
    healthBar.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
    healthBar.BorderSizePixel = 0
    local healthBarCorner = Instance.new("UICorner", healthBar)
    healthBarCorner.CornerRadius = UDim.new(1, 0)

    local healthFill = Instance.new("Frame", healthBar)
    healthFill.Size = UDim2.new(1, 0, 1, 0)
    healthFill.BackgroundColor3 = COLORS.HEART
    healthFill.BorderSizePixel = 0
    local healthFillCorner = Instance.new("UICorner", healthFill)
    healthFillCorner.CornerRadius = UDim.new(1, 0)

    local healthConnection = humanoid.HealthChanged:Connect(function()
        updateLifeVisual(player)
    end)
    local deathConn = humanoid.Died:Connect(function()
        cleanupLifeESP(player)
        local respawnConn = player.CharacterAdded:Connect(function()
            respawnConn:Disconnect()
            task.wait(0.3)
            updateLifeESP(player)
        end)
    end)
    local characterConn = player.CharacterAdded:Connect(function()
        task.wait(0.3)
        updateLifeESP(player)
    end)

    lifeObjects[player] = {
        billboard = billboard,
        healthText = healthText,
        healthFill = healthFill,
        humanoid = humanoid,
        rootPart = rootPart,
        healthConnection = healthConnection,
        deathConnection = deathConn,
        characterConn = characterConn
    }

    local shouldShow = passesFilters(player) and (not teamCheck or player.Team ~= LocalPlayer.Team)
    billboard.Enabled = shouldShow
    updateLifeVisual(player)
end

local function updateAllLifeESP()
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            updateLifeESP(player)
        end
    end
end

local function clearAllLifeESP()
    for player, _ in pairs(lifeObjects) do cleanupLifeESP(player) end
    lifeObjects = {}
end

local function updateGlowForPlayer(player)
    if not espGlowEnabled or not espEnabled then
        if glowObjects[player] then
            if glowObjects[player].highlight then pcall(function() glowObjects[player].highlight:Destroy() end) end
            glowObjects[player] = nil
        end
        return
    end
    if player == LocalPlayer then return end
    local character = player.Character
    if not character then
        if glowObjects[player] then
            if glowObjects[player].highlight then pcall(function() glowObjects[player].highlight:Destroy() end) end
            glowObjects[player] = nil
        end
        return
    end
    if glowObjects[player] and glowObjects[player].character ~= character then
        if glowObjects[player].highlight then pcall(function() glowObjects[player].highlight:Destroy() end) end
        glowObjects[player] = nil
    end
    if not glowObjects[player] then
        local teamColor = getTeamColor(player)
        local highlight = Instance.new("Highlight")
        highlight.FillColor = teamColor
        highlight.OutlineColor = teamColor
        highlight.FillTransparency = 0.5
        highlight.OutlineTransparency = 0.3
        highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
        highlight.Adornee = character
        highlight.Parent = game.CoreGui
        glowObjects[player] = {highlight = highlight, character = character}
    else
        local currentTeamColor = getTeamColor(player)
        if glowObjects[player].highlight.FillColor ~= currentTeamColor then
            glowObjects[player].highlight.FillColor = currentTeamColor
            glowObjects[player].highlight.OutlineColor = currentTeamColor
        end
    end
    local shouldShow = passesFilters(player) and (not teamCheck or player.Team ~= LocalPlayer.Team) and isPlayerAlive(player)
    if glowObjects[player] and glowObjects[player].highlight then
        glowObjects[player].highlight.Enabled = shouldShow
    end
end

local function clearAllGlow()
    for player, data in pairs(glowObjects) do
        if data.highlight then pcall(function() data.highlight:Destroy() end) end
    end
    glowObjects = {}
end

fovCircle = Drawing.new("Circle")
fovCircle.Visible = showFov
fovCircle.Thickness = 2
fovCircle.Radius = fovRadius
fovCircle.Transparency = 1
fovCircle.Color = currentTitleColor
fovCircle.Position = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)

aimColorFovCircle = Drawing.new("Circle")
aimColorFovCircle.Visible = false
aimColorFovCircle.Thickness = 2
aimColorFovCircle.Radius = aimColorFovRadius
aimColorFovCircle.Transparency = 1
aimColorFovCircle.Color = currentTitleColor
aimColorFovCircle.Position = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)

aimbotTargetFrame = Instance.new("Frame", ScreenGui)
aimbotTargetFrame.Size = UDim2.new(0, 200, 0, 40)
aimbotTargetFrame.Position = UDim2.new(0.5, 0, 0, 40)
aimbotTargetFrame.AnchorPoint = Vector2.new(0.5, 0.5)
aimbotTargetFrame.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
aimbotTargetFrame.BackgroundTransparency = 0.5
aimbotTargetFrame.BorderSizePixel = 0
aimbotTargetFrame.Visible = false
aimbotTargetFrame.ZIndex = 50

local aimbotTargetCorner = Instance.new("UICorner", aimbotTargetFrame)
aimbotTargetCorner.CornerRadius = UDim.new(0, 12)

aimbotTargetLabel = Instance.new("TextLabel", aimbotTargetFrame)
aimbotTargetLabel.Size = UDim2.new(1, 0, 1, 0)
aimbotTargetLabel.Position = UDim2.new(0, 0, 0, 0)
aimbotTargetLabel.BackgroundTransparency = 1
aimbotTargetLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
aimbotTargetLabel.Font = Enum.Font.GothamBlack
aimbotTargetLabel.TextSize = 18
aimbotTargetLabel.TextXAlignment = Enum.TextXAlignment.Center
aimbotTargetLabel.TextYAlignment = Enum.TextYAlignment.Center
aimbotTargetLabel.Text = ""

-- ====================== NOVO ESP COM BILLBOARDGUI ======================
local function createESP(player)
    if player == LocalPlayer or espBillboardObjects[player] then return end
    local head = player.Character and player.Character:FindFirstChild("Head")
    if not head then return end

    local billboard = Instance.new("BillboardGui")
    billboard.Name = "ESP"
    billboard.Size = UDim2.new(0, 200, 0, 50)
    billboard.StudsOffset = Vector3.new(0, 1.5, 0)
    billboard.AlwaysOnTop = true
    billboard.MaxDistance = 1000
    billboard.Parent = head

    local nickLabel = Instance.new("TextLabel", billboard)
    nickLabel.Size = UDim2.new(1, 0, 0.5, 0)
    nickLabel.Position = UDim2.new(0, 0, 0, 0)
    nickLabel.BackgroundTransparency = 1
    nickLabel.Text = player.Name
    nickLabel.TextColor3 = getTeamColor(player)
    nickLabel.Font = Enum.Font.GothamBlack
    nickLabel.TextSize = 14
    nickLabel.TextStrokeTransparency = 0.5
    nickLabel.TextStrokeColor3 = Color3.new(0, 0, 0)
    nickLabel.Visible = espNicksEnabled

    local distanceLabel = Instance.new("TextLabel", billboard)
    distanceLabel.Size = UDim2.new(1, 0, 0.5, 0)
    distanceLabel.Position = UDim2.new(0, 0, 0.5, 0)
    distanceLabel.BackgroundTransparency = 1
    distanceLabel.Text = ""
    distanceLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    distanceLabel.Font = Enum.Font.GothamBlack
    distanceLabel.TextSize = 12
    distanceLabel.TextStrokeTransparency = 0.5
    distanceLabel.TextStrokeColor3 = Color3.new(0, 0, 0)

    espBillboardObjects[player] = {
        gui = billboard,
        nickLabel = nickLabel,
        distanceLabel = distanceLabel
    }
end

local function removeESP(player)
    local data = espBillboardObjects[player]
    if data then
        pcall(function() data.gui:Destroy() end)
        espBillboardObjects[player] = nil
    end
end

local function clearAllESP()
    for player, data in pairs(espBillboardObjects) do
        pcall(function() data.gui:Destroy() end)
    end
    espBillboardObjects = {}
end

local function updateESPTexts()
    if not espEnabled then return end
    for player, data in pairs(espBillboardObjects) do
        local passes = passesFilters(player)
        local teamOk = not teamCheck or player.Team ~= LocalPlayer.Team
        local show = passes and teamOk and isPlayerAlive(player)
        data.gui.Enabled = show
        if show then
            local head = player.Character and player.Character:FindFirstChild("Head")
            if head then
                local distance = (Camera.CFrame.Position - head.Position).Magnitude
                data.distanceLabel.Text = espDistanceEnabled and string.format("%.0fm", distance) or ""
                data.nickLabel.Visible = espNicksEnabled
                data.distanceLabel.Visible = espDistanceEnabled
                local fontSize = getFontSizeByDistance(distance)
                data.nickLabel.TextSize = fontSize
                data.distanceLabel.TextSize = math.max(8, fontSize - 2)
                data.nickLabel.TextColor3 = getTeamColor(player)
            end
        end
    end
end

local function updateAllESP()
    if not espEnabled then return end
    for _, player in ipairs(Players:GetPlayers()) do
        if player == LocalPlayer then continue end
        if not espBillboardObjects[player] then
            createESP(player)
        end
        if espGlowEnabled then updateGlowForPlayer(player) end
        if espLifeEnabled then
            if not lifeObjects[player] and player.Character then
                updateLifeESP(player)
            end
            if lifeObjects[player] and lifeObjects[player].billboard then
                lifeObjects[player].billboard.Enabled = passesFilters(player) and (not teamCheck or player.Team ~= LocalPlayer.Team) and isPlayerAlive(player)
                updateLifeVisual(player)
            end
        end
    end
    for player, _ in pairs(espBillboardObjects) do
        if not Players:GetPlayerFromCharacter(player.Character) and not Players:GetPlayers()[player] then
            removeESP(player)
        end
    end
end

local function updateAllGlows()
    if not espGlowEnabled then return end
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then updateGlowForPlayer(player) end
    end
end

local function recreateAllESP()
    clearAllESP()
    clearAllGlow()
    clearAllLifeESP()
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            createESP(player)
            updateGlowForPlayer(player)
            updateLifeESP(player)
        end
    end
end

-- ====================== FUNÇÕES DE VISIBILIDADE E ALVO ======================
local function isVisible(part)
    if not wallCheck then return true end
    local now = tick()
    local cached = visibilityCache[part]
    if cached and (now - cached.time) < VISIBILITY_CACHE_TIME then
        return cached.result
    end
    local origin = Camera.CFrame.Position
    local direction = (part.Position - origin)
    local raycastParams = RaycastParams.new()
    raycastParams.FilterType = Enum.RaycastFilterType.Blacklist
    raycastParams.FilterDescendantsInstances = {LocalPlayer.Character}
    local raycastResult = Workspace:Raycast(origin, direction.Unit * math.min(direction.Magnitude, 1000), raycastParams)
    local result = false
    if raycastResult then
        result = raycastResult.Instance:IsDescendantOf(part.Parent)
    end
    visibilityCache[part] = {time = now, result = result}
    return result
end

local function getClosestPart(radius)
    local closestPart, closestPlayer = nil, nil
    local shortestDist = radius or fovRadius
    local center = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild(targetPart) then
            if not isPlayerAlive(player) or not passesFilters(player) then continue end
            if teamCheck and player.Team == LocalPlayer.Team then continue end
            local part = player.Character[targetPart]
            local pos, onScreen = Camera:WorldToViewportPoint(part.Position)
            local dist = (Vector2.new(pos.X, pos.Y) - center).Magnitude
            if onScreen and dist < shortestDist and isVisible(part) then
                shortestDist = dist
                closestPart = part
                closestPlayer = player
            end
        end
    end
    return closestPart, closestPlayer
end

local function getAimColorTargetPart(radius, partsList)
    local mousePos = UserInputService:GetMouseLocation()
    local center = Vector2.new(mousePos.X, mousePos.Y)
    local closestPart = nil
    local shortestDist = radius
    for _, player in ipairs(Players:GetPlayers()) do
        if player == LocalPlayer or not player.Character then continue end
        if not isPlayerAlive(player) or not passesFilters(player) then continue end
        if teamCheck and player.Team == LocalPlayer.Team then continue end
        for _, partName in ipairs(partsList) do
            local part = player.Character:FindFirstChild(partName)
            if part and part:IsA("BasePart") then
                if wallCheck and not isVisible(part) then continue end
                local pos, onScreen = Camera:WorldToViewportPoint(part.Position)
                if onScreen then
                    local dist = (Vector2.new(pos.X, pos.Y) - center).Magnitude
                    if dist < shortestDist then
                        shortestDist = dist
                        closestPart = part
                    end
                end
            end
        end
    end
    return closestPart
end

local function smoothLookAt(target)
    if not target then return end
    local targetPos = target.Position
    local currentCFrame = Camera.CFrame
    local direction = (targetPos - currentCFrame.Position).Unit
    local targetCFrame = CFrame.new(currentCFrame.Position, currentCFrame.Position + direction)
    local lerpCFrame = currentCFrame:Lerp(targetCFrame, 1 - smoothingFactor)
    Camera.CFrame = lerpCFrame
end

local function moveMouseTowardsPart(part, strength, prediction)
    if not mouseMoveFunc or not part then return end
    local predictedPos = part.Position
    local velocity = part.AssemblyLinearVelocity
    if not velocity or velocity.Magnitude == 0 then
        local rootPart = part.Parent and part.Parent:FindFirstChild("HumanoidRootPart")
        velocity = (rootPart and rootPart.AssemblyLinearVelocity) or Vector3.zero
    end
    predictedPos = predictedPos + velocity * prediction
    local targetScreenPos, onScreen = Camera:WorldToViewportPoint(predictedPos)
    if not onScreen then return end
    local currentMousePos = UserInputService:GetMouseLocation()
    local deltaX = targetScreenPos.X - currentMousePos.X
    local deltaY = targetScreenPos.Y - currentMousePos.Y
    if math.abs(deltaX) < 2 and math.abs(deltaY) < 2 then return end
    mouseMoveFunc(deltaX * strength, deltaY * strength)
end

-- ====================== FUNÇÕES DE ITENS ======================
local function localPlayerHasItem()
    local char = LocalPlayer.Character
    if not char then return false end
    for _, v in ipairs(char:GetChildren()) do
        if v:IsA("Tool") then return true end
        if v:IsA("Model") and v:FindFirstChild("Handle") then return true end
    end
    return false
end

local function getEquippedTool()
    local char = LocalPlayer.Character
    if not char then return nil end
    for _, v in ipairs(char:GetChildren()) do
        if v:IsA("Tool") then return v end
    end
    return nil
end

-- ====================== CRIAÇÃO DOS TOGGLES E SLIDERS ======================
createToggleButton(ESPTabFrame, "ESP", function() return espEnabled end, function()
    espEnabled = not espEnabled
    if espEnabled then recreateAllESP() else clearAllESP(); clearAllGlow(); clearAllLifeESP() end
end)
createToggleButton(ESPTabFrame, "ESP Nicks", function() return espNicksEnabled end, function() espNicksEnabled = not espNicksEnabled end)
createToggleButton(ESPTabFrame, "ESP Distance", function() return espDistanceEnabled end, function() espDistanceEnabled = not espDistanceEnabled end)
createToggleButton(ESPTabFrame, "ESP Life", function() return espLifeEnabled end, function()
    espLifeEnabled = not espLifeEnabled
    if espLifeEnabled and espEnabled then updateAllLifeESP() else clearAllLifeESP() end
end)
createToggleButton(ESPTabFrame, "ESP Glow", function() return espGlowEnabled end, function()
    espGlowEnabled = not espGlowEnabled
    if espGlowEnabled and espEnabled then updateAllGlows() else clearAllGlow() end
end)
createToggleButton(ESPTabFrame, "Verificar Time", function() return teamCheck end, function()
    teamCheck = not teamCheck
    if espEnabled then updateAllESP() end
end)
createToggleButton(ESPTabFrame, "Verificar Parede", function() return wallCheck end, function() wallCheck = not wallCheck end)

createToggleButton(ExtraTabFrame, "Mostrar FPS", function() return showFps end, function()
    showFps = not showFps
    if showFps then createFPSDisplay() else if fpsLabel then pcall(function() fpsLabel:Remove() end) end end
end)
createToggleButton(ExtraTabFrame, "TP (Ctrl + Click)", function() return tpEnabled end, function() tpEnabled = not tpEnabled end)
createToggleButton(ExtraTabFrame, "Infinite Ammo", function() return infiniteAmmoEnabled end, function() infiniteAmmoEnabled = not infiniteAmmoEnabled end)
createToggleButton(ExtraTabFrame, "No Recoil", function() return noRecoilEnabled end, function() noRecoilEnabled = not noRecoilEnabled end)

createToggleButton(AimbotLeftFrame, "Aimbot", function() return aimbotEnabled end, function() aimbotEnabled = not aimbotEnabled end)
createToggleButton(AimbotLeftFrame, "Exibir FOV", function() return showFov end, function() showFov = not showFov end)
createSlider(AimbotRightFrame, "FOV Aimbot", 50, 500, fovRadius, function(value)
    fovRadius = value
    if fovCircle then fovCircle.Radius = value end
end)
createSlider(AimbotRightFrame, "Suavização Aimbot", 0, 10, smoothingFactor * 10, function(value)
    smoothingFactor = value / 10
end)

createToggleButton(AimColorLeftFrame, "AimColor", function() return aimColorEnabled end, function() aimColorEnabled = not aimColorEnabled end)
createToggleButton(AimColorLeftFrame, "Exibir FOV AimColor", function() return showAimColorFov end, function() showAimColorFov = not showAimColorFov end)
createToggleButton(AimColorLeftFrame, "AimColor Disfarce", function() return aimColorDisfarceEnabled end, function() aimColorDisfarceEnabled = not aimColorDisfarceEnabled end)
createSlider(AimColorRightFrame, "FOV AimColor", 20, 500, aimColorFovRadius, function(value)
    aimColorFovRadius = value
    if aimColorFovCircle then aimColorFovCircle.Radius = value end
end)
createSlider(AimColorRightFrame, "Força AimColor", 5, 100, aimColorStrength * 100, function(value)
    aimColorStrength = value / 100
end)
createSlider(AimColorRightFrame, "Predição AimColor", 0, 500, aimColorPrediction * 1000, function(value)
    aimColorPrediction = value / 1000
end)

-- ====================== CONEXÕES DE JOGADORES ======================
for _, player in ipairs(Players:GetPlayers()) do
    if player ~= LocalPlayer then
        createESP(player)
        updateGlowForPlayer(player)
        updateLifeESP(player)
    end
end

Players.PlayerAdded:Connect(function(player)
    task.wait(0.3)
    if espEnabled then
        createESP(player)
        updateGlowForPlayer(player)
        if espLifeEnabled then updateLifeESP(player) end
    end
    player.CharacterAdded:Connect(function()
        task.wait(0.3)
        if espEnabled and espLifeEnabled then updateLifeESP(player) end
        if espEnabled and espGlowEnabled then updateGlowForPlayer(player) end
        if espEnabled then
            removeESP(player)
            createESP(player)
        end
    end)
end)

for _, player in ipairs(Players:GetPlayers()) do
    if player ~= LocalPlayer then
        player.CharacterAdded:Connect(function()
            task.wait(0.3)
            if espEnabled and espLifeEnabled then updateLifeESP(player) end
            if espEnabled and espGlowEnabled then updateGlowForPlayer(player) end
            if espEnabled then
                removeESP(player)
                createESP(player)
            end
        end)
    end
end

Players.PlayerRemoving:Connect(function(player)
    removeESP(player)
    if glowObjects[player] then
        if glowObjects[player].highlight then pcall(function() glowObjects[player].highlight:Destroy() end) end
        glowObjects[player] = nil
    end
    cleanupLifeESP(player)
end)

-- ====================== LOOP DE MUNIÇÃO INFINITA ======================
task.spawn(function()
    while true do
        if infiniteAmmoEnabled then
            pcall(function()
                for _, v in ipairs(LocalPlayer.Backpack:GetChildren()) do
                    if v:IsA("Model") then
                        local ammo = v:FindFirstChild("Ammo")
                        if ammo and ammo:IsA("DoubleConstrainedValue") then
                            ammo.MinValue = 29
                        end
                    end
                end
            end)
        end
        task.wait(1)
    end
end)

-- ====================== LOOP PRINCIPAL ======================
local lastAimColorTargetUpdate = 0
local cachedAimColorPart = nil
local noRecoilNextFire = 0

RunService.RenderStepped:Connect(function()
    if not panelLoaded then return end
    local mousePos = UserInputService:GetMouseLocation()
    local mouseOverPanel = isMouseOverPanel()

    fovCircle.Visible = showFov and aimbotEnabled
    fovCircle.Radius = fovRadius
    fovCircle.Position = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)

    aimColorFovCircle.Visible = aimColorEnabled and showAimColorFov
    aimColorFovCircle.Radius = aimColorFovRadius
    aimColorFovCircle.Position = Vector2.new(mousePos.X, mousePos.Y)

    if usingRgb then
        fovCircle.Color = Color3.fromHSV(rgbHue, 1, 1)
        aimColorFovCircle.Color = Color3.fromHSV(rgbHue, 1, 1)
    else
        fovCircle.Color = currentTitleColor
        aimColorFovCircle.Color = currentTitleColor
    end

    updateFPS()

    if tick() - espUpdateTimer >= 0.5 then
        espUpdateTimer = tick()
        updateAllESP()
        updateESPTexts()
    end

    if aimbotEnabled and UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton2) and not mouseOverPanel then
        local target, targetPlayer = getClosestPart(fovRadius)
        if target then
            smoothLookAt(target)
            if targetPlayer then
                aimbotTargetFrame.Visible = true
                aimbotTargetLabel.Text = "Alvo: " .. targetPlayer.Name
            end
        else
            aimbotTargetFrame.Visible = false
        end
    else
        aimbotTargetFrame.Visible = false
    end

    -- ===== AIMCOLOR: DETECTA O BOTÃO ESQUERDO POR TODAS AS FONTES =====
    local lmbNow = isLeftMouseDown()

    if aimColorEnabled and lmbNow and not mouseOverPanel then
        local partsList = aimColorDisfarceEnabled and DISFARCE_PARTS or {"Head", "HumanoidRootPart"}
        local searchRadius = aimColorFovRadius

        if tick() - lastAimColorTargetUpdate >= 0.1 or cachedAimColorPart == nil then
            cachedAimColorPart = getAimColorTargetPart(searchRadius, partsList)
            lastAimColorTargetUpdate = tick()
        end

        local part = cachedAimColorPart
        if part then
            local effStrength = aimColorDisfarceEnabled and 0.2 or aimColorStrength
            local effPrediction = aimColorDisfarceEnabled and 0.008 or aimColorPrediction
            moveMouseTowardsPart(part, effStrength, effPrediction)
            if localPlayerHasItem() and tick() >= nextFireTime then
                task.spawn(function()
                    local tool = getEquippedTool()
                    if tool then
                        pcall(function() tool:Activate() end)
                    else
                        if mouseClickFunc then
                            mouseClickFunc()
                        end
                    end
                end)
                nextFireTime = tick() + 0.1
                if not noRecoilEnabled and mouseMoveFunc then mouseMoveFunc(0, recoilCompensation) end
            end
        else
            cachedAimColorPart = nil
        end
    else
        cachedAimColorPart = nil
        lastAimColorTargetUpdate = 0
    end

    -- ====== ANTI-RECOIL (puxada forte) + AUTO-CLICK ======
    if noRecoilEnabled and lmbNow and not mouseOverPanel and getEquippedTool() ~= nil then
        local now = tick()

        if localPlayerHasItem() and tick() >= noRecoilNextFire then
            task.spawn(function()
                local tool = getEquippedTool()
                if tool then
                    pcall(function() tool:Activate() end)
                else
                    if mouseClickFunc then
                        mouseClickFunc()
                    end
                end
            end)
            noRecoilNextFire = tick() + 0.1
        end

        if now >= nextRecoilTime then
            if mouseMoveFunc then
                local phase = math.floor(now * 20) % 2
                if phase == 0 then
                    mouseMoveFunc(1, 0.6)
                else
                    mouseMoveFunc(-1, 0.6)
                end
            end
            nextRecoilTime = now + 0.05
        end

        if now >= nextRecoilReturnTime then
            if mouseMoveFunc then
                local phase = math.floor(now * 20) % 2
                if phase == 0 then
                    mouseMoveFunc(-1, -0.2)
                else
                    mouseMoveFunc(1, -0.2)
                end
            end
            nextRecoilReturnTime = now + 0.15
        end

        pcall(function()
            if Camera then
                Camera.CFrame = Camera.CFrame * CFrame.Angles(
                    math.rad(-0.06),
                    math.rad(0.004),
                    0
                )
            end
        end)
    end
end)

updateAnonimatoVisual()
panelLoaded = true
