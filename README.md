-- =============================================================================
-- AUTOMATED POSITION-SAVING & INSTANT TELEPORTATION (ULTRA-COMPATIBLE VERSION)
-- =============================================================================

local UI_NAME = "AdvancedDevTPManager_v3"
local CoreGui = game:GetService("CoreGui")
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local player = Players.LocalPlayer

-- 1. UNIVERSAL DETACH & DUPLICATION CHECK
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

-- 2. CORE LOGIC VARIABLES
local waypoints = {}
local currentIdx = 1
local isLoopRunning = false
local intervalMs = 3000       
local distanceThreshold = 500 

-- 3. INTERFACE INITIALIZATION
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = UI_NAME
ScreenGui.Parent = targetParent
ScreenGui.ResetOnSpawn = false

local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Parent = ScreenGui
MainFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
MainFrame.Position = UDim2.new(0.1, 0, 0.25, 0)
MainFrame.Size = UDim2.new(0, 280, 0, 380)
MainFrame.Active = true

local MainCorner = Instance.new("UICorner")
MainCorner.CornerRadius = UDim.new(0, 10)
MainCorner.Parent = MainFrame

local Title = Instance.new("TextLabel")
Title.Name = "Title"
Title.Parent = MainFrame
Title.Size = UDim2.new(1, 0, 0, 40)
Title.BackgroundColor3 = Color3.fromRGB(30, 30, 38)
Title.TextColor3 = Color3.fromRGB(255, 255, 255)
Title.Text = "⚡ INSTANT LOOP TELEPORT"
Title.Font = Enum.Font.SourceSansBold
Title.TextSize = 15

local TitleCorner = Instance.new("UICorner")
TitleCorner.CornerRadius = UDim.new(0, 10)
TitleCorner.Parent = Title

-- MANUAL DRAG LOGIC (Fixes broken executor .Draggable property)
local dragging, dragInput, dragStart, startPos
Title.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = MainFrame.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then dragging = false end
        end)
    end
end)
Title.InputChanged:Connect(function(input)
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

-- CONTROLS CONTAINER
local InputContainer = Instance.new("Frame")
InputContainer.Name = "InputContainer"
InputContainer.Parent = MainFrame
InputContainer.BackgroundTransparency = 1
InputContainer.Position = UDim2.new(0.05, 0, 0.13, 0)
InputContainer.Size = UDim2.new(0.9, 0, 0, 70)

local UIGridLayout = Instance.new("UIGridLayout")
UIGridLayout.Parent = InputContainer
UIGridLayout.CellSize = UDim2.new(0.48, 0, 0, 30)
UIGridLayout.CellPadding = UDim2.new(0.04, 0, 0, 10)

local InputInterval = Instance.new("TextBox")
InputInterval.Name = "InputInterval"
InputInterval.Parent = InputContainer
InputInterval.BackgroundColor3 = Color3.fromRGB(35, 35, 42)
InputInterval.TextColor3 = Color3.fromRGB(255, 255, 255)
InputInterval.Text = "3000"
InputInterval.PlaceholderText = "Interval (ms)"
InputInterval.Font = Enum.Font.SourceSans
InputInterval.TextSize = 13
Instance.new("UICorner", InputInterval).CornerRadius = UDim.new(0, 6)

local InputDistance = Instance.new("TextBox")
InputDistance.Name = "InputDistance"
InputDistance.Parent = InputContainer
InputDistance.BackgroundColor3 = Color3.fromRGB(35, 35, 42)
InputDistance.TextColor3 = Color3.fromRGB(255, 255, 255)
InputDistance.Text = "500"
InputDistance.PlaceholderText = "Max Dist (Studs)"
InputDistance.Font = Enum.Font.SourceSans
InputDistance.TextSize = 13
Instance.new("UICorner", InputDistance).CornerRadius = UDim.new(0, 6)

local BtnSave = Instance.new("TextButton")
BtnSave.Name = "BtnSave"
BtnSave.Parent = InputContainer
BtnSave.BackgroundColor3 = Color3.fromRGB(46, 125, 50)
BtnSave.TextColor3 = Color3.fromRGB(255, 255, 255)
BtnSave.Text = "📌 Save Location"
BtnSave.Font = Enum.Font.SourceSansBold
BtnSave.TextSize = 13
Instance.new("UICorner", BtnSave).CornerRadius = UDim.new(0, 6)

local BtnToggle = Instance.new("TextButton")
BtnToggle.Name = "BtnToggle"
BtnToggle.Parent = InputContainer
BtnToggle.BackgroundColor3 = Color3.fromRGB(21, 101, 192)
BtnToggle.TextColor3 = Color3.fromRGB(255, 255, 255)
BtnToggle.Text = "▶️ Start Loop"
BtnToggle.Font = Enum.Font.SourceSansBold
BtnToggle.TextSize = 13
Instance.new("UICorner", BtnToggle).CornerRadius = UDim.new(0, 6)

local StatusLabel = Instance.new("TextLabel")
StatusLabel.Name = "StatusLabel"
StatusLabel.Parent = MainFrame
StatusLabel.Position = UDim2.new(0.05, 0, 0.35, 0)
StatusLabel.Size = UDim2.new(0.9, 0, 0, 20)
StatusLabel.BackgroundTransparency = 1
StatusLabel.TextColor3 = Color3.fromRGB(160, 160, 165)
StatusLabel.Text = "Status: Ready. Awaiting points..."
StatusLabel.Font = Enum.Font.SourceSansItalic
StatusLabel.TextSize = 12
StatusLabel.TextXAlignment = Enum.TextXAlignment.Left

-- SCROLLABLE LIST
local ScrollingFrame = Instance.new("ScrollingFrame")
ScrollingFrame.Parent = MainFrame
ScrollingFrame.Position = UDim2.new(0.05, 0, 0.42, 0)
ScrollingFrame.Size = UDim2.new(0.9, 0, 0, 205)
ScrollingFrame.BackgroundColor3 = Color3.fromRGB(14, 14, 18)
ScrollingFrame.BorderSizePixel = 0
ScrollingFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
ScrollingFrame.ScrollBarThickness = 4
Instance.new("UICorner", ScrollingFrame).CornerRadius = UDim.new(0, 6)

local UIListLayout = Instance.new("UIListLayout")
UIListLayout.Parent = ScrollingFrame
UIListLayout.SortOrder = Enum.SortOrder.LayoutOrder
UIListLayout.Padding = UDim.new(0, 6)
local UIPadding = Instance.new("UIPadding")
UIPadding.Parent = ScrollingFrame
UIPadding.PaddingTop = UDim.new(0, 4)
UIPadding.PaddingLeft = UDim.new(0, 4)

-- 4. LOGIC ENGINE
local function getHRP()
    local character = player.Character or player.CharacterAdded:Wait()
    return character:WaitForChild("HumanoidRootPart", 5)
end

local function updateVisualList()
    for _, item in pairs(ScrollingFrame:GetChildren()) do
        if item:IsA("Frame") then item:Destroy() end
    end

    for i, position in ipairs(waypoints) do
        local ItemFrame = Instance.new("Frame")
        ItemFrame.Size = UDim2.new(1, -8, 0, 32)
        ItemFrame.BackgroundColor3 = Color3.fromRGB(28, 28, 35)
        ItemFrame.Parent = ScrollingFrame
        Instance.new("UICorner", ItemFrame).CornerRadius = UDim.new(0, 5)

        local ItemText = Instance.new("TextLabel")
        ItemText.Size = UDim2.new(0.75, 0, 1, 0)
        ItemText.Position = UDim2.new(0.04, 0, 0, 0)
        ItemText.BackgroundTransparency = 1
        ItemText.TextColor3 = Color3.fromRGB(220, 220, 225)
        ItemText.Text = string.format("Waypoint #%d [X:%.0f, Z:%.0f]", i, position.X, position.Z)
        ItemText.Font = Enum.Font.SourceSans
        ItemText.TextSize = 13
        ItemText.TextXAlignment = Enum.TextXAlignment.Left
        ItemText.Parent = ItemFrame

        local BtnDelete = Instance.new("TextButton")
        BtnDelete.Size = UDim2.new(0.16, 0, 0.75, 0)
        BtnDelete.Position = UDim2.new(0.81, 0, 0.125, 0)
        BtnDelete.BackgroundColor3 = Color3.fromRGB(176, 0, 32)
        BtnDelete.TextColor3 = Color3.fromRGB(255, 255, 255)
        BtnDelete.Text = "❌"
        BtnDelete.Font = Enum.Font.SourceSansBold
        BtnDelete.TextSize = 10
        BtnDelete.Parent = ItemFrame
        Instance.new("UICorner", BtnDelete).CornerRadius = UDim.new(0, 4)

        BtnDelete.MouseButton1Click:Connect(function()
            table.remove(waypoints, i)
            if currentIdx > #waypoints then currentIdx = 1 end
            updateVisualList()
            StatusLabel.Text = "Status: Waypoint removed."
        end)
    end
    ScrollingFrame.CanvasSize = UDim2.new(0, 0, 0, UIListLayout.AbsoluteContentSize.Y + 10)
end

-- 5. BUTTONS AND INPUT SIGNALS
BtnSave.MouseButton1Click:Connect(function()
    local hrp = getHRP()
    if hrp then
        table.insert(waypoints, hrp.Position)
        StatusLabel.TextColor3 = Color3.fromRGB(76, 175, 80)
        StatusLabel.Text = string.format("Status: Saved Waypoint #%d!", #waypoints)
        updateVisualList()
    else
        StatusLabel.TextColor3 = Color3.fromRGB(229, 57, 53)
        StatusLabel.Text = "Status: Character Error!"
    end
end)

InputInterval.FocusLost:Connect(function()
    local val = tonumber(InputInterval.Text)
    if val and val >= 0 then intervalMs = val else InputInterval.Text = tostring(intervalMs) end
end)

InputDistance.FocusLost:Connect(function()
    local val = tonumber(InputDistance.Text)
    if val and val > 0 then distanceThreshold = val else InputDistance.Text = tostring(distanceThreshold) end
end)

-- 6. BACKGROUND RUNNING THREAD (SMART FILTER)
task.spawn(function()
    while true do
        task.wait(0.05)
        if isLoopRunning and #waypoints > 0 then
            local hrp = getHRP()
            if hrp then
                local targetDestination = waypoints[currentIdx]
                local currentDistance = (hrp.Position - targetDestination).Magnitude

                if currentDistance > distanceThreshold then
                    StatusLabel.TextColor3 = Color3.fromRGB(255, 152, 0)
                    StatusLabel.Text = string.format("⚠️ Skipped #%d: Too far (%.0f Studs)", currentIdx, currentDistance)
                    currentIdx = currentIdx + 1
                    if currentIdx > #waypoints then currentIdx = 1 end
                else
                    hrp.CFrame = CFrame.new(targetDestination)
                    StatusLabel.TextColor3 = Color3.fromRGB(33, 150, 243)
                    StatusLabel.Text = string.format("⚡ Teleported to Waypoint #%d!", currentIdx)
                    
                    currentIdx = currentIdx + 1
                    if currentIdx > #waypoints then currentIdx = 1 end
                    
                    task.wait(intervalMs / 1000)
                end
            end
        end
    end
end)

BtnToggle.MouseButton1Click:Connect(function()
    if #waypoints == 0 then
        StatusLabel.TextColor3 = Color3.fromRGB(251, 192, 45)
        StatusLabel.Text = "Status: Loop is empty!"
        return
    end
    isLoopRunning = not isLoopRunning
    if isLoopRunning then
        BtnToggle.BackgroundColor3 = Color3.fromRGB(198, 40, 40)
        BtnToggle.Text = "⏹️ Stop Loop"
    else
        BtnToggle.BackgroundColor3 = Color3.fromRGB(21, 101, 192)
        BtnToggle.Text = "▶️ Start Loop"
        StatusLabel.TextColor3 = Color3.fromRGB(160, 160, 165)
        StatusLabel.Text = "Status: Loop paused."
    end
end)
