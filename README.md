# Roblox-Doors-2026-The-Hollow-Esp
An AI script that lets you see candy and salvage through walls and highlight them with tracers.
Toggle with K.
Copy and Paste all below in to your injector or roblox game!
------------------------------------------------------------------------------------------------------------------------
```lua
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")

local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")
local CurrentRooms = workspace:WaitForChild("CurrentRooms")

-- Configuration
local TOGGLE_KEY = Enum.KeyCode.K
local CLUSTER_RADIUS_PIXELS = 60
local SCAN_INTERVAL_SECONDS = 1.5

-- State
local isEspActive = false
local mainRenderConnection = nil
local backgroundScanTask = nil
local screenGui = nil

local trackedTargets = {}
local trackedInstances = {}
local activeLines = {}
local activeLabels = {}

-- ==========================================================
-- NOTIFICATIONS
-- ==========================================================

local function showToggleMessage(text, color)
    local oldGui = PlayerGui:FindFirstChild("EspNotifyGui")
    if oldGui then
        oldGui:Destroy()
    end

    local notifyGui = Instance.new("ScreenGui")
    notifyGui.Name = "EspNotifyGui"
    notifyGui.ResetOnSpawn = false
    notifyGui.IgnoreGuiInset = true
    notifyGui.Parent = PlayerGui

    local label = Instance.new("TextLabel")
    label.BackgroundTransparency = 1
    label.Size = UDim2.fromOffset(360, 50)
    label.Position = UDim2.new(0.5, -180, 0.35, 0)
    label.Font = Enum.Font.Code
    label.TextSize = 18
    label.TextColor3 = color
    label.Text = text
    label.TextStrokeTransparency = 0
    label.TextStrokeColor3 = Color3.new(0, 0, 0)
    label.Parent = notifyGui

    task.delay(1, function()
        if not label.Parent then
            return
        end

        local tween = TweenService:Create(
            label,
            TweenInfo.new(0.5),
            {
                TextTransparency = 1,
                TextStrokeTransparency = 1
            }
        )

        tween:Play()
        tween.Completed:Once(function()
            if notifyGui.Parent then
                notifyGui:Destroy()
            end
        end)
    end)
end

-- ==========================================================
-- CLEANUP
-- ==========================================================

local function clearOldESP()
    if mainRenderConnection then
        mainRenderConnection:Disconnect()
        mainRenderConnection = nil
    end

    if backgroundScanTask then
        task.cancel(backgroundScanTask)
        backgroundScanTask = nil
    end

    if screenGui then
        screenGui:Destroy()
        screenGui = nil
    end

    for _, target in ipairs(trackedTargets) do
        local highlight = target.Highlight
        if highlight and highlight.Parent then
            highlight:Destroy()
        end
    end

    -- Also remove any remaining target-owned highlights.
    for _, line in pairs(activeLines) do
        if line then
            line:Destroy()
        end
    end

    for _, label in pairs(activeLabels) do
        if label then
            label:Destroy()
        end
    end

    activeLines = {}
    activeLabels = {}
    trackedTargets = {}
    trackedInstances = {}
end

-- ==========================================================
-- FIND A PHYSICAL PART
-- ==========================================================

local function getPhysicalPart(object)
    if object:IsA("BasePart") then
        return object
    end

    local root = object:FindFirstChild("CandyRoot", true)
        or object:FindFirstChild("Handle", true)

    if root and root:IsA("BasePart") then
        return root
    end

    for _, descendant in ipairs(object:GetDescendants()) do
        if descendant:IsA("BasePart") then
            return descendant
        end
    end

    return nil
end

-- ==========================================================
-- TRACK TARGETS
-- ==========================================================

local function trackItemInstance(item, defaultName)
    if not isEspActive or not item or not item.Parent then
        return
    end

    -- Use the Instance itself as the key instead of GetDebugId().
    if trackedInstances[item] then
        return
    end

    local part = getPhysicalPart(item)
    if not part then
        return
    end

    local color = Color3.fromHSV(math.random(), 0.9, 0.9)

    local highlight = Instance.new("Highlight")
    highlight.Name = "FakeESPHighlight"
    highlight.FillColor = color
    highlight.FillTransparency = 0.2
    highlight.OutlineColor = Color3.new(1, 1, 1)
    highlight.OutlineTransparency = 0
    highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    highlight.Adornee = item
    highlight.Parent = screenGui

    local target = {
        Instance = item,
        Part = part,
        Highlight = highlight,
        Color = color,
        Name = string.upper(defaultName or item.Name),
    }

    trackedInstances[item] = target
    table.insert(trackedTargets, target)
end

-- ==========================================================
-- DISCOVERY
-- ==========================================================

local function scanAllLocations()
    if not isEspActive then
        return
    end

    local function inspect(container)
        for _, descendant in ipairs(container:GetDescendants()) do
            if descendant.Name == "Candy" then
                trackItemInstance(descendant, "CANDY")
            elseif descendant.Parent
                and descendant.Parent.Name == "SalvageItems" then
                trackItemInstance(descendant)
            end
        end
    end

    inspect(CurrentRooms)

    local dropsFolder = workspace:FindFirstChild("Drops")
    if dropsFolder then
        inspect(dropsFolder)
    end
end

local function startBackgroundLiveScanner()
    if backgroundScanTask then
        task.cancel(backgroundScanTask)
    end

    backgroundScanTask = task.spawn(function()
        while isEspActive do
            scanAllLocations()
            task.wait(SCAN_INTERVAL_SECONDS)
        end

        backgroundScanTask = nil
    end)
end

-- ==========================================================
-- CREATE HUD ELEMENTS
-- ==========================================================

local function createLine()
    local line = Instance.new("Frame")
    line.Name = "TracerLine"
    line.BorderSizePixel = 0
    line.AnchorPoint = Vector2.new(0.5, 0.5)
    line.Size = UDim2.fromOffset(0, 1.5)
    line.Visible = false
    line.Parent = screenGui
    return line
end

local function createLabel()
    local label = Instance.new("TextLabel")
    label.Name = "ClusterLabel"
    label.BackgroundTransparency = 1
    label.Size = UDim2.fromOffset(200, 42)
    label.AnchorPoint = Vector2.new(0.5, 0)
    label.Font = Enum.Font.Code
    label.TextSize = 13
    label.TextStrokeTransparency = 0
    label.TextStrokeColor3 = Color3.new(0, 0, 0)
    label.TextWrapped = true
    label.Visible = false
    label.Parent = screenGui
    return label
end

-- ==========================================================
-- RENDER LOOP
-- ==========================================================

local function startEspLoop()
    if mainRenderConnection then
        mainRenderConnection:Disconnect()
    end

    mainRenderConnection = RunService.RenderStepped:Connect(function()
        if not isEspActive or not screenGui or not screenGui.Parent then
            return
        end

        local camera = workspace.CurrentCamera
        if not camera then
            return
        end

        local viewport = camera.ViewportSize
        local centerX = viewport.X / 2
        local centerY = viewport.Y / 2

        local character = LocalPlayer.Character
        local rootPart = character
            and character:FindFirstChild("HumanoidRootPart")

        local visiblePoints = {}

        for _, line in pairs(activeLines) do
            line.Visible = false
        end

        -- Draw tracers and remove deleted targets.
        for i = #trackedTargets, 1, -1 do
            local target = trackedTargets[i]
            local item = target.Instance
            local part = target.Part

            if not item or not item.Parent or not part or not part.Parent then
                if target.Highlight and target.Highlight.Parent then
                    target.Highlight:Destroy()
                end

                activeLines[target] = nil
                trackedInstances[item] = nil
                table.remove(trackedTargets, i)
            else
                local screenPos, onScreen =
                    camera:WorldToViewportPoint(part.Position)

                if onScreen and screenPos.Z > 0 then
                    local distance = 0

                    if rootPart then
                        distance = math.floor(
                            (rootPart.Position - part.Position).Magnitude / 3.5
                        )
                    end

                    table.insert(visiblePoints, {
                        X = screenPos.X,
                        Y = screenPos.Y,
                        Distance = distance,
                        Color = target.Color,
                        Name = target.Name,
                    })

                    local line = activeLines[target]
                    if not line then
                        line = createLine()
                        activeLines[target] = line
                    end

                    line.BackgroundColor3 = target.Color

                    local dx = screenPos.X - centerX
                    local dy = screenPos.Y - centerY
                    local length = math.sqrt(dx * dx + dy * dy)

                    line.Size = UDim2.fromOffset(length, 1.5)
                    line.Position = UDim2.fromOffset(
                        centerX + dx / 2,
                        centerY + dy / 2
                    )
                    line.Rotation = math.deg(math.atan2(dy, dx))
                    line.Visible = true
                end
            end
        end

        -- Group nearby on-screen items.
        local groups = {}

        for _, point in ipairs(visiblePoints) do
            local nearestGroup = nil
            local nearestDistance = CLUSTER_RADIUS_PIXELS

            for _, group in ipairs(groups) do
                local dx = point.X - group.X
                local dy = point.Y - group.Y
                local distance = math.sqrt(dx * dx + dy * dy)

                if distance <= nearestDistance then
                    nearestDistance = distance
                    nearestGroup = group
                end
            end

            if nearestGroup then
                table.insert(nearestGroup.Items, point)

                -- Keep the group position centered on its members.
                local count = #nearestGroup.Items
                nearestGroup.X =
                    nearestGroup.X + (point.X - nearestGroup.X) / count
                nearestGroup.Y =
                    nearestGroup.Y + (point.Y - nearestGroup.Y) / count
            else
                table.insert(groups, {
                    X = point.X,
                    Y = point.Y,
                    Items = {point},
                })
            end
        end

        for _, label in pairs(activeLabels) do
            label.Visible = false
        end

        -- Display labels.
        for index, group in ipairs(groups) do
            local label = activeLabels[index]

            if not label then
                label = createLabel()
                activeLabels[index] = label
            end

            local count = #group.Items

            if count == 1 then
                local item = group.Items[1]
                label.Text = string.format(
                    "%s\n[%dm]",
                    item.Name,
                    item.Distance
                )
                label.TextColor3 = item.Color
            else
                local totalDistance = 0

                for _, item in ipairs(group.Items) do
                    totalDistance = totalDistance + item.Distance
                end

                local averageDistance = math.floor(totalDistance / count)

                label.Text = string.format(
                    "%dx ITEMS CLUSTERED\n[%dm]",
                    count,
                    averageDistance
                )
                label.TextColor3 = Color3.new(1, 1, 1)
            end

            label.Position = UDim2.fromOffset(group.X, group.Y + 15)
            label.Visible = true
        end
    end)
end

-- ==========================================================
-- TOGGLE
-- ==========================================================

UserInputService.InputBegan:Connect(function(input, gameProcessedEvent)
    if gameProcessedEvent then
        return
    end

    if input.KeyCode ~= TOGGLE_KEY then
        return
    end

    isEspActive = not isEspActive
    clearOldESP()

    if isEspActive then
        screenGui = Instance.new("ScreenGui")
        screenGui.Name = "EspLinesOverlay"
        screenGui.IgnoreGuiInset = true
        screenGui.ResetOnSpawn = false
        screenGui.DisplayOrder = 10
        screenGui.Parent = PlayerGui

        scanAllLocations()
        startBackgroundLiveScanner()
        startEspLoop()

        showToggleMessage(
            "CANDY & SALVAGE ESP: ENABLED",
            Color3.fromRGB(0, 255, 0)
        )
    else
        showToggleMessage(
            "CANDY & SALVAGE ESP: DISABLED",
            Color3.fromRGB(255, 0, 0)
        )
    end
end)

-- Start with ESP disabled.
clearOldESP()
```
