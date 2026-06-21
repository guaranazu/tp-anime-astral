local UI_NAME = "AdvancedDevTPManager_v3"
local CoreGui = game:GetService("CoreGui")
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local player = Players.LocalPlayer

local targetParent = nil
pcall(function()
    if CoreGui and pcall(function() local a = CoreGui.Name end) then
        targetParent = CoreGui
    end
end)
if not targetParent then
    targetParent = player:WaitForChild("PlayerGui", 10)
end

if targetParent:FindFirstChild(UI_NAME) then
    targetParent[UI_NAME]:Destroy()
end

-- ===================== STATE =====================
local waypoints = {}
local currentIdx = 1
local isLoopRunning = false
local intervalMs = 3000
local distanceThreshold = 500
local minimized = false

-- ===================== PALETTE =====================
local PALETTE = {
    bg          = Color3.fromRGB(18, 18, 23),
    panel       = Color3.fromRGB(26, 26, 33),
    panelLight  = Color3.fromRGB(33, 33, 42),
    stroke      = Color3.fromRGB(45, 45, 56),
    accent      = Color3.fromRGB(110, 89, 255),
    accentDark  = Color3.fromRGB(82, 64, 214),
    green       = Color3.fromRGB(56, 199, 121),
    red         = Color3.fromRGB(235, 87, 87),
    orange      = Color3.fromRGB(255, 159, 67),
    blue        = Color3.fromRGB(64, 156, 255),
    textMain    = Color3.fromRGB(240, 240, 245),
    textDim     = Color3.fromRGB(150, 150, 162),
}

-- ===================== HELPERS =====================
local function corner(parent, radius)
    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0, radius or 8)
    c.Parent = parent
    return c
end

local function stroke(parent, color, thickness)
    local s = Instance.new("UIStroke")
    s.Color = color or PALETTE.stroke
    s.Thickness = thickness or 1
    s.Parent = parent
    return s
end

local function tween(obj, props, time)
    TweenService:Create(obj, TweenInfo.new(time or 0.18, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), props):Play()
end

local function hoverEffect(btn, baseColor, hoverColor)
    btn.MouseEnter:Connect(function()
        tween(btn, { BackgroundColor3 = hoverColor })
    end)
    btn.MouseLeave:Connect(function()
        tween(btn, { BackgroundColor3 = baseColor })
    end)
end

-- ===================== ROOT =====================
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = UI_NAME
ScreenGui.Parent = targetParent
ScreenGui.ResetOnSpawn = false
ScreenGui.IgnoreGuiInset = true

local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Parent = ScreenGui
MainFrame.BackgroundColor3 = PALETTE.bg
MainFrame.Position = UDim2.new(0.5, -160, 0.5, -210)
MainFrame.Size = UDim2.new(0, 320, 0, 420)
MainFrame.Active = true
MainFrame.ClipsDescendants = false
corner(MainFrame, 14)
stroke(MainFrame, PALETTE.stroke, 1)

local MainShadow = Instance.new("ImageLabel")
MainShadow.Name = "Shadow"
MainShadow.Parent = MainFrame
MainShadow.BackgroundTransparency = 1
MainShadow.Image = "rbxassetid://1316045217"
MainShadow.ImageColor3 = Color3.new(0, 0, 0)
MainShadow.ImageTransparency = 0.45
MainShadow.ScaleType = Enum.ScaleType.Slice
MainShadow.SliceCenter = Rect.new(10, 10, 118, 118)
MainShadow.Size = UDim2.new(1, 40, 1, 40)
MainShadow.Position = UDim2.new(0, -20, 0, -20)
MainShadow.ZIndex = -1

-- ===================== TITLE BAR =====================
local TitleBar = Instance.new("Frame")
TitleBar.Name = "TitleBar"
TitleBar.Parent = MainFrame
TitleBar.BackgroundColor3 = PALETTE.panel
TitleBar.Size = UDim2.new(1, 0, 0, 46)
corner(TitleBar, 14)
stroke(TitleBar, PALETTE.stroke, 1)

-- mask the bottom corners of the title bar so it only rounds on top
local TitleBarMask = Instance.new("Frame")
TitleBarMask.Parent = TitleBar
TitleBarMask.BackgroundColor3 = PALETTE.panel
TitleBarMask.BorderSizePixel = 0
TitleBarMask.Size = UDim2.new(1, 0, 0, 14)
TitleBarMask.Position = UDim2.new(0, 0, 1, -14)
TitleBarMask.ZIndex = 0

local TitleIcon = Instance.new("TextLabel")
TitleIcon.Parent = TitleBar
TitleIcon.BackgroundTransparency = 1
TitleIcon.Position = UDim2.new(0, 14, 0, 0)
TitleIcon.Size = UDim2.new(0, 24, 1, 0)
TitleIcon.Text = "🚀"
TitleIcon.TextSize = 18
TitleIcon.Font = Enum.Font.SourceSansBold

local Title = Instance.new("TextLabel")
Title.Name = "Title"
Title.Parent = TitleBar
Title.BackgroundTransparency = 1
Title.Position = UDim2.new(0, 42, 0, 0)
Title.Size = UDim2.new(1, -110, 1, 0)
Title.TextColor3 = PALETTE.textMain
Title.Text = "Loop Teleport"
Title.Font = Enum.Font.GothamBold
Title.TextSize = 15
Title.TextXAlignment = Enum.TextXAlignment.Left

local BtnMinimize = Instance.new("TextButton")
BtnMinimize.Name = "BtnMinimize"
BtnMinimize.Parent = TitleBar
BtnMinimize.BackgroundColor3 = PALETTE.panelLight
BtnMinimize.Position = UDim2.new(1, -76, 0.5, -13)
BtnMinimize.Size = UDim2.new(0, 26, 0, 26)
BtnMinimize.Text = "➖"
BtnMinimize.TextSize = 12
BtnMinimize.TextColor3 = PALETTE.textMain
BtnMinimize.Font = Enum.Font.SourceSansBold
BtnMinimize.AutoButtonColor = false
corner(BtnMinimize, 7)
hoverEffect(BtnMinimize, PALETTE.panelLight, PALETTE.stroke)

local BtnClose = Instance.new("TextButton")
BtnClose.Name = "BtnClose"
BtnClose.Parent = TitleBar
BtnClose.BackgroundColor3 = PALETTE.panelLight
BtnClose.Position = UDim2.new(1, -40, 0.5, -13)
BtnClose.Size = UDim2.new(0, 26, 0, 26)
BtnClose.Text = "✖️"
BtnClose.TextSize = 11
BtnClose.TextColor3 = PALETTE.textMain
BtnClose.Font = Enum.Font.SourceSansBold
BtnClose.AutoButtonColor = false
corner(BtnClose, 7)
hoverEffect(BtnClose, PALETTE.panelLight, PALETTE.red)

-- ===================== DRAG LOGIC =====================
local dragging, dragInput, dragStart, startPos
TitleBar.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = MainFrame.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then dragging = false end
        end)
    end
end)
TitleBar.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
        dragInput = input
    end
end)
UserInputService.InputChanged:Connect(function(input)
    if input == dragInput and dragging then
        local delta = input.Position - dragStart
        MainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
end)

-- ===================== BODY CONTAINER =====================
local Body = Instance.new("Frame")
Body.Name = "Body"
Body.Parent = MainFrame
Body.BackgroundTransparency = 1
Body.Position = UDim2.new(0, 0, 0, 46)
Body.Size = UDim2.new(1, 0, 1, -46)
Body.ClipsDescendants = true

-- ===================== STATUS BAR =====================
local StatusBar = Instance.new("Frame")
StatusBar.Name = "StatusBar"
StatusBar.Parent = Body
StatusBar.BackgroundColor3 = PALETTE.panel
StatusBar.Position = UDim2.new(0, 12, 0, 12)
StatusBar.Size = UDim2.new(1, -24, 0, 40)
corner(StatusBar, 10)
stroke(StatusBar, PALETTE.stroke, 1)

local StatusDot = Instance.new("Frame")
StatusDot.Name = "StatusDot"
StatusDot.Parent = StatusBar
StatusDot.BackgroundColor3 = PALETTE.textDim
StatusDot.Position = UDim2.new(0, 14, 0.5, -5)
StatusDot.Size = UDim2.new(0, 10, 0, 10)
corner(StatusDot, 5)

local StatusLabel = Instance.new("TextLabel")
StatusLabel.Name = "StatusLabel"
StatusLabel.Parent = StatusBar
StatusLabel.BackgroundTransparency = 1
StatusLabel.Position = UDim2.new(0, 34, 0, 0)
StatusLabel.Size = UDim2.new(1, -46, 1, 0)
StatusLabel.TextColor3 = PALETTE.textMain
StatusLabel.Text = "Pronto. Salve um waypoint para começar."
StatusLabel.Font = Enum.Font.Gotham
StatusLabel.TextSize = 12
StatusLabel.TextXAlignment = Enum.TextXAlignment.Left
StatusLabel.TextTruncate = Enum.TextTruncate.AtEnd

-- ===================== SETTINGS SECTION =====================
local SettingsLabel = Instance.new("TextLabel")
SettingsLabel.Parent = Body
SettingsLabel.BackgroundTransparency = 1
SettingsLabel.Position = UDim2.new(0, 14, 0, 62)
SettingsLabel.Size = UDim2.new(1, -28, 0, 16)
SettingsLabel.Text = "⚙️ CONFIGURAÇÕES"
SettingsLabel.TextColor3 = PALETTE.textDim
SettingsLabel.Font = Enum.Font.GothamBold
SettingsLabel.TextSize = 11
SettingsLabel.TextXAlignment = Enum.TextXAlignment.Left

local InputContainer = Instance.new("Frame")
InputContainer.Name = "InputContainer"
InputContainer.Parent = Body
InputContainer.BackgroundTransparency = 1
InputContainer.Position = UDim2.new(0, 12, 0, 82)
InputContainer.Size = UDim2.new(1, -24, 0, 56)

local UIGridLayout = Instance.new("UIGridLayout")
UIGridLayout.Parent = InputContainer
UIGridLayout.CellSize = UDim2.new(0.485, 0, 1, 0)
UIGridLayout.CellPadding = UDim2.new(0.03, 0, 0, 0)

local function makeLabeledInput(name, icon, placeholder, defaultText)
    local holder = Instance.new("Frame")
    holder.Name = name .. "Holder"
    holder.BackgroundColor3 = PALETTE.panel
    holder.Parent = InputContainer
    corner(holder, 8)
    stroke(holder, PALETTE.stroke, 1)

    local icoLabel = Instance.new("TextLabel")
    icoLabel.BackgroundTransparency = 1
    icoLabel.Position = UDim2.new(0, 8, 0, 0)
    icoLabel.Size = UDim2.new(0, 22, 1, 0)
    icoLabel.Text = icon
    icoLabel.TextSize = 14
    icoLabel.Font = Enum.Font.SourceSans
    icoLabel.Parent = holder

    local box = Instance.new("TextBox")
    box.Name = name
    box.Parent = holder
    box.BackgroundTransparency = 1
    box.Position = UDim2.new(0, 30, 0, 0)
    box.Size = UDim2.new(1, -38, 1, 0)
    box.TextColor3 = PALETTE.textMain
    box.PlaceholderColor3 = PALETTE.textDim
    box.Text = defaultText
    box.PlaceholderText = placeholder
    box.Font = Enum.Font.Gotham
    box.TextSize = 13
    box.ClearTextOnFocus = false
    box.TextXAlignment = Enum.TextXAlignment.Left

    box.Focused:Connect(function()
        tween(holder, { BackgroundColor3 = PALETTE.panelLight })
    end)
    box.FocusLost:Connect(function()
        tween(holder, { BackgroundColor3 = PALETTE.panel })
    end)

    return box
end

local InputInterval = makeLabeledInput("InputInterval", "⏱️", "Intervalo (ms)", "3000")
local InputDistance = makeLabeledInput("InputDistance", "📏", "Distância máx.", "500")

-- ===================== ACTION BUTTONS =====================
local ActionContainer = Instance.new("Frame")
ActionContainer.Name = "ActionContainer"
ActionContainer.Parent = Body
ActionContainer.BackgroundTransparency = 1
ActionContainer.Position = UDim2.new(0, 12, 0, 146)
ActionContainer.Size = UDim2.new(1, -24, 0, 38)

local ActionGrid = Instance.new("UIGridLayout")
ActionGrid.Parent = ActionContainer
ActionGrid.CellSize = UDim2.new(0.485, 0, 1, 0)
ActionGrid.CellPadding = UDim2.new(0.03, 0, 0, 0)

local function makeActionButton(name, icon, text, color, hoverColor)
    local btn = Instance.new("TextButton")
    btn.Name = name
    btn.Parent = ActionContainer
    btn.BackgroundColor3 = color
    btn.AutoButtonColor = false
    btn.Text = icon .. "  " .. text
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 13
    corner(btn, 8)
    hoverEffect(btn, color, hoverColor)
    return btn
end

local BtnSave = makeActionButton("BtnSave", "📌", "Salvar", PALETTE.green, Color3.fromRGB(70, 219, 141))
local BtnToggle = makeActionButton("BtnToggle", "▶️", "Iniciar", PALETTE.accent, PALETTE.accentDark)

-- ===================== WAYPOINTS HEADER =====================
local ListHeader = Instance.new("Frame")
ListHeader.Parent = Body
ListHeader.BackgroundTransparency = 1
ListHeader.Position = UDim2.new(0, 14, 0, 196)
ListHeader.Size = UDim2.new(1, -28, 0, 18)

local ListLabel = Instance.new("TextLabel")
ListLabel.Parent = ListHeader
ListLabel.BackgroundTransparency = 1
ListLabel.Size = UDim2.new(0.7, 0, 1, 0)
ListLabel.Text = "📍 WAYPOINTS"
ListLabel.TextColor3 = PALETTE.textDim
ListLabel.Font = Enum.Font.GothamBold
ListLabel.TextSize = 11
ListLabel.TextXAlignment = Enum.TextXAlignment.Left

local CountLabel = Instance.new("TextLabel")
CountLabel.Name = "CountLabel"
CountLabel.Parent = ListHeader
CountLabel.BackgroundTransparency = 1
CountLabel.Size = UDim2.new(0.3, 0, 1, 0)
CountLabel.Text = "0 pontos"
CountLabel.TextColor3 = PALETTE.textDim
CountLabel.Font = Enum.Font.Gotham
CountLabel.TextSize = 11
CountLabel.TextXAlignment = Enum.TextXAlignment.Right

-- ===================== SCROLLING LIST =====================
local ScrollingFrame = Instance.new("ScrollingFrame")
ScrollingFrame.Parent = Body
ScrollingFrame.Position = UDim2.new(0, 12, 0, 218)
ScrollingFrame.Size = UDim2.new(1, -24, 1, -230)
ScrollingFrame.BackgroundColor3 = PALETTE.panel
ScrollingFrame.BorderSizePixel = 0
ScrollingFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
ScrollingFrame.ScrollBarThickness = 3
ScrollingFrame.ScrollBarImageColor3 = PALETTE.accent
corner(ScrollingFrame, 10)
stroke(ScrollingFrame, PALETTE.stroke, 1)

local UIListLayout = Instance.new("UIListLayout")
UIListLayout.Parent = ScrollingFrame
UIListLayout.SortOrder = Enum.SortOrder.LayoutOrder
UIListLayout.Padding = UDim.new(0, 6)

local UIPadding = Instance.new("UIPadding")
UIPadding.Parent = ScrollingFrame
UIPadding.PaddingTop = UDim.new(0, 6)
UIPadding.PaddingLeft = UDim.new(0, 6)
UIPadding.PaddingRight = UDim.new(0, 6)
UIPadding.PaddingBottom = UDim.new(0, 6)

local EmptyState = Instance.new("TextLabel")
EmptyState.Name = "EmptyState"
EmptyState.Parent = ScrollingFrame
EmptyState.BackgroundTransparency = 1
EmptyState.Size = UDim2.new(1, 0, 0, 60)
EmptyState.Text = "🗺️ Nenhum waypoint salvo ainda"
EmptyState.TextColor3 = PALETTE.textDim
EmptyState.Font = Enum.Font.Gotham
EmptyState.TextSize = 12

-- ===================== LOGIC =====================
local function getHRP()
    local character = player.Character or player.CharacterAdded:Wait()
    return character:WaitForChild("HumanoidRootPart", 5)
end

local function setStatus(text, color)
    StatusLabel.Text = text
    StatusLabel.TextColor3 = color or PALETTE.textMain
    tween(StatusDot, { BackgroundColor3 = color or PALETTE.textDim }, 0.15)
end

local function updateVisualList()
    for _, item in pairs(ScrollingFrame:GetChildren()) do
        if item:IsA("Frame") and item.Name == "WaypointItem" then
            item:Destroy()
        end
    end

    CountLabel.Text = #waypoints .. (#waypoints == 1 and " ponto" or " pontos")
    EmptyState.Visible = #waypoints == 0

    for i, position in ipairs(waypoints) do
        local ItemFrame = Instance.new("Frame")
        ItemFrame.Name = "WaypointItem"
        ItemFrame.LayoutOrder = i
        ItemFrame.Size = UDim2.new(1, 0, 0, 36)
        ItemFrame.BackgroundColor3 = PALETTE.panelLight
        ItemFrame.Parent = ScrollingFrame
        corner(ItemFrame, 7)

        local isActive = (i == currentIdx) and isLoopRunning
        local ActiveBar = Instance.new("Frame")
        ActiveBar.Name = "ActiveBar"
        ActiveBar.BackgroundColor3 = isActive and PALETTE.accent or PALETTE.stroke
        ActiveBar.BorderSizePixel = 0
        ActiveBar.Position = UDim2.new(0, 0, 0, 0)
        ActiveBar.Size = UDim2.new(0, 4, 1, 0)
        ActiveBar.Parent = ItemFrame
        corner(ActiveBar, 2)

        local NumberBadge = Instance.new("TextLabel")
        NumberBadge.BackgroundTransparency = 1
        NumberBadge.Position = UDim2.new(0, 14, 0, 0)
        NumberBadge.Size = UDim2.new(0, 24, 1, 0)
        NumberBadge.Text = "#" .. i
        NumberBadge.TextColor3 = PALETTE.accent
        NumberBadge.Font = Enum.Font.GothamBold
        NumberBadge.TextSize = 12
        NumberBadge.TextXAlignment = Enum.TextXAlignment.Left
        NumberBadge.Parent = ItemFrame

        local ItemText = Instance.new("TextLabel")
        ItemText.BackgroundTransparency = 1
        ItemText.Position = UDim2.new(0, 42, 0, 0)
        ItemText.Size = UDim2.new(1, -110, 1, 0)
        ItemText.BackgroundColor3 = Color3.new(0,0,0)
        ItemText.Text = string.format("📍 X:%.0f  Z:%.0f", position.X, position.Z)
        ItemText.TextColor3 = PALETTE.textMain
        ItemText.Font = Enum.Font.Gotham
        ItemText.TextSize = 12
        ItemText.TextXAlignment = Enum.TextXAlignment.Left
        ItemText.Parent = ItemFrame

        local BtnGoto = Instance.new("TextButton")
        BtnGoto.Size = UDim2.new(0, 26, 0, 26)
        BtnGoto.Position = UDim2.new(1, -64, 0.5, -13)
        BtnGoto.BackgroundColor3 = PALETTE.panel
        BtnGoto.Text = "🎯"
        BtnGoto.TextSize = 12
        BtnGoto.AutoButtonColor = false
        BtnGoto.Parent = ItemFrame
        corner(BtnGoto, 6)
        hoverEffect(BtnGoto, PALETTE.panel, PALETTE.blue)

        local BtnDelete = Instance.new("TextButton")
        BtnDelete.Size = UDim2.new(0, 26, 0, 26)
        BtnDelete.Position = UDim2.new(1, -32, 0.5, -13)
        BtnDelete.BackgroundColor3 = PALETTE.panel
        BtnDelete.Text = "🗑️"
        BtnDelete.TextSize = 12
        BtnDelete.AutoButtonColor = false
        BtnDelete.Parent = ItemFrame
        corner(BtnDelete, 6)
        hoverEffect(BtnDelete, PALETTE.panel, PALETTE.red)

        BtnGoto.MouseButton1Click:Connect(function()
            local hrp = getHRP()
            if hrp then
                hrp.CFrame = CFrame.new(position)
                setStatus(string.format("🎯 Teleportado manualmente para #%d", i), PALETTE.blue)
            end
        end)

        BtnDelete.MouseButton1Click:Connect(function()
            table.remove(waypoints, i)
            if currentIdx > #waypoints then currentIdx = 1 end
            updateVisualList()
            setStatus("🗑️ Waypoint removido", PALETTE.orange)
        end)
    end

    ScrollingFrame.CanvasSize = UDim2.new(0, 0, 0, UIListLayout.AbsoluteContentSize.Y + 12)
end

-- ===================== BUTTON EVENTS =====================
BtnSave.MouseButton1Click:Connect(function()
    local hrp = getHRP()
    if hrp then
        table.insert(waypoints, hrp.Position)
        setStatus(string.format("✅ Waypoint #%d salvo!", #waypoints), PALETTE.green)
        updateVisualList()
    else
        setStatus("❌ Erro: personagem não encontrado", PALETTE.red)
    end
end)

InputInterval.FocusLost:Connect(function()
    local val = tonumber(InputInterval.Text)
    if val and val >= 0 then
        intervalMs = val
    else
        InputInterval.Text = tostring(intervalMs)
    end
end)

InputDistance.FocusLost:Connect(function()
    local val = tonumber(InputDistance.Text)
    if val and val > 0 then
        distanceThreshold = val
    else
        InputDistance.Text = tostring(distanceThreshold)
    end
end)

BtnToggle.MouseButton1Click:Connect(function()
    if #waypoints == 0 then
        setStatus("⚠️ Adicione ao menos 1 waypoint", PALETTE.orange)
        return
    end
    isLoopRunning = not isLoopRunning
    if isLoopRunning then
        tween(BtnToggle, { BackgroundColor3 = PALETTE.red })
        BtnToggle.Text = "⏹️  Parar"
        setStatus("🟢 Loop iniciado", PALETTE.green)
    else
        tween(BtnToggle, { BackgroundColor3 = PALETTE.accent })
        BtnToggle.Text = "▶️  Iniciar"
        setStatus("⏸️ Loop pausado", PALETTE.textDim)
    end
    updateVisualList()
end)

BtnMinimize.MouseButton1Click:Connect(function()
    minimized = not minimized
    if minimized then
        tween(MainFrame, { Size = UDim2.new(0, 320, 0, 46) }, 0.22)
        BtnMinimize.Text = "🔼"
    else
        tween(MainFrame, { Size = UDim2.new(0, 320, 0, 420) }, 0.22)
        BtnMinimize.Text = "➖"
    end
end)

BtnClose.MouseButton1Click:Connect(function()
    isLoopRunning = false
    tween(MainFrame, { Size = UDim2.new(0, 0, 0, 0) }, 0.18)
    task.wait(0.2)
    ScreenGui:Destroy()
end)

-- ===================== BACKGROUND LOOP =====================
task.spawn(function()
    while true do
        task.wait(0.05)
        if isLoopRunning and #waypoints > 0 then
            local hrp = getHRP()
            if hrp then
                local targetDestination = waypoints[currentIdx]
                local currentDistance = (hrp.Position - targetDestination).Magnitude

                if currentDistance > distanceThreshold then
                    setStatus(string.format("⚠️ #%d ignorado: muito longe (%.0f studs)", currentIdx, currentDistance), PALETTE.orange)
                    currentIdx = currentIdx + 1
                    if currentIdx > #waypoints then currentIdx = 1 end
                    updateVisualList()
                else
                    hrp.CFrame = CFrame.new(targetDestination)
                    setStatus(string.format("⚡ Teleportado para #%d", currentIdx), PALETTE.blue)
                    updateVisualList()

                    currentIdx = currentIdx + 1
                    if currentIdx > #waypoints then currentIdx = 1 end

                    task.wait(intervalMs / 1000)
                end
            end
        end
    end
end)

updateVisualList()
