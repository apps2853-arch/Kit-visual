--[[
    KIT VISUALS v3 | Kit PvP [Season 1]
    Локальные визуальные эффекты (только на твоём клиенте).
    RightShift = меню. На телефоне: тап по острову сверху -> МЕНЮ.

    v3:
    * Снег: лучи по всей земле (блоки, меши, террейн) - без дыр
    * Таргет: без таблички с ником/HP
    * Новый GUI (вкладки) + новый остров с подсветкой кнопок
    * Небо: планета / чёрная дыра / луна, 3D-сияние, метеоры, молнии
    * Аватар: крылья, нимб, кольца, волны от шагов, трейл, аура
    * НЕРЕАЛ-режим: одна кнопка включает всё красивое
]]

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local Lighting = game:GetService("Lighting")
local Stats = game:GetService("Stats")
local Debris = game:GetService("Debris")

local player = Players.LocalPlayer
local env = (getgenv and getgenv()) or _G
local C = Color3.fromRGB
local WHITE = Color3.new(1, 1, 1)

if env.KitVisualsUnload then
    pcall(env.KitVisualsUnload)
end

local alive = true
local connections = {}
local function track(connection)
    table.insert(connections, connection)
    return connection
end

----------------------------------------------------------------
-- HELPERS / THEME
----------------------------------------------------------------

local function make(class, props, children)
    local object = Instance.new(class)
    for key, value in pairs(props or {}) do
        object[key] = value
    end
    for _, child in ipairs(children or {}) do
        child.Parent = object
    end
    return object
end

local function round(object, radius)
    return make("UICorner", {CornerRadius = UDim.new(0, radius or 9), Parent = object})
end

local T = {
    bg = C(9, 11, 18),
    panel = C(17, 20, 32),
    panel2 = C(28, 32, 50),
    border = C(46, 52, 82),
    text = C(236, 240, 252),
    muted = C(138, 146, 172),
    a1 = C(70, 215, 255),
    a2 = C(160, 105, 255),
    pink = C(255, 95, 190),
    green = C(95, 235, 170),
    red = C(255, 90, 100),
}

local themeFns = {}
local function themed(fn)
    table.insert(themeFns, fn)
    pcall(fn)
end
local function applyTheme()
    for _, fn in ipairs(themeFns) do pcall(fn) end
end

local spinGrads = {}
local function accentGradient(object, rotation, spin)
    local g = make("UIGradient", {Rotation = rotation or 0, Parent = object})
    themed(function() g.Color = ColorSequence.new(T.a1, T.a2) end)
    if spin then table.insert(spinGrads, g) end
    return g
end

local function stroke(object, color, transparency, thickness)
    return make("UIStroke", {
        Color = color or T.border,
        Transparency = transparency or 0,
        Thickness = thickness or 1,
        ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
        Parent = object,
    })
end

local function tween(object, duration, properties, style)
    local animation = TweenService:Create(
        object,
        TweenInfo.new(duration, style or Enum.EasingStyle.Quint, Enum.EasingDirection.Out),
        properties
    )
    animation:Play()
    return animation
end

local function randomRange(a, b) return a + math.random() * (b - a) end

local function getRoot(plr)
    local character = plr and plr.Character
    return character and character:FindFirstChild("HumanoidRootPart")
end

----------------------------------------------------------------
-- FOLDERS
----------------------------------------------------------------

local function folder(name)
    local f = Instance.new("Folder")
    f.Name = name
    f.Parent = workspace
    return f
end
local particleFolder = folder("KV_ClientParticles")
local snowCaps = folder("KV_SnowCaps")
local targetFolder = folder("KV_TargetFX")
local killFolder = folder("KV_KillFX")
local skyFolder = folder("KV_Sky")
local selfFolder = folder("KV_SelfFX")

----------------------------------------------------------------
-- GUI ROOTS
----------------------------------------------------------------

local uiParent
pcall(function()
    if gethui then uiParent = gethui() end
end)
uiParent = uiParent or player:WaitForChild("PlayerGui")

local gui = make("ScreenGui", {
    Name = "KV3_Menu", ResetOnSpawn = false, IgnoreGuiInset = true,
    DisplayOrder = 999, ZIndexBehavior = Enum.ZIndexBehavior.Sibling, Parent = uiParent,
})
local fxGui = make("ScreenGui", {
    Name = "KV3_ScreenFX", ResetOnSpawn = false, IgnoreGuiInset = true,
    DisplayOrder = 990, ZIndexBehavior = Enum.ZIndexBehavior.Sibling, Parent = uiParent,
})

----------------------------------------------------------------
-- WINDOW
----------------------------------------------------------------

local window = make("Frame", {
    Name = "Main",
    AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.fromScale(0.5, 0.58),
    Size = UDim2.fromOffset(600, 350),
    BackgroundColor3 = WHITE,
    BackgroundTransparency = 0.06,
    BorderSizePixel = 0,
    Parent = gui,
})
round(window, 20)
make("UIGradient", {Color = ColorSequence.new(C(20, 24, 42), C(7, 8, 14)), Rotation = 90, Parent = window})
accentGradient(stroke(window, WHITE, 0.1, 1.6), 35, true)
local uiScale = make("UIScale", {Scale = 1, Parent = window})

local function baseScale()
    local camera = workspace.CurrentCamera
    if not camera then return 1 end
    local v = camera.ViewportSize
    return math.clamp(math.min(v.X / 660, v.Y / 390), 0.5, 1.1)
end

local opened = true
local function toggleWindow()
    opened = not opened
    if opened then
        window.Visible = true
        uiScale.Scale = baseScale() * 0.92
        tween(uiScale, 0.25, {Scale = baseScale()}, Enum.EasingStyle.Back)
    else
        tween(uiScale, 0.15, {Scale = baseScale() * 0.92})
        task.delay(0.16, function()
            if alive and not opened then window.Visible = false end
        end)
    end
end

local topbar = make("Frame", {Size = UDim2.new(1, 0, 0, 50), BackgroundTransparency = 1, Parent = window})
local title = make("TextLabel", {
    Position = UDim2.fromOffset(18, 7), Size = UDim2.fromOffset(220, 24),
    BackgroundTransparency = 1, Text = "KIT VISUALS", Font = Enum.Font.GothamBlack,
    TextSize = 21, TextColor3 = WHITE, TextXAlignment = Enum.TextXAlignment.Left, Parent = topbar,
})
accentGradient(title, 0)
make("TextLabel", {
    Position = UDim2.fromOffset(19, 31), Size = UDim2.fromOffset(300, 14),
    BackgroundTransparency = 1, Text = "v3  •  Kit PvP Season 1  •  RightShift / остров",
    Font = Enum.Font.Gotham, TextSize = 10, TextColor3 = T.muted,
    TextXAlignment = Enum.TextXAlignment.Left, Parent = topbar,
})
local closeButton = make("TextButton", {
    AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -16, 0, 25),
    Size = UDim2.fromOffset(30, 30), BackgroundColor3 = T.panel2, Text = "—",
    Font = Enum.Font.GothamBold, TextSize = 14, TextColor3 = T.muted, AutoButtonColor = false, Parent = topbar,
})
round(closeButton, 15)

local tabBar = make("ScrollingFrame", {
    Position = UDim2.fromOffset(14, 52), Size = UDim2.new(1, -28, 0, 34),
    BackgroundTransparency = 1, BorderSizePixel = 0, ScrollBarThickness = 0,
    CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.X,
    ScrollingDirection = Enum.ScrollingDirection.X, Parent = window,
}, {
    make("UIListLayout", {
        FillDirection = Enum.FillDirection.Horizontal, Padding = UDim.new(0, 6),
        SortOrder = Enum.SortOrder.LayoutOrder, VerticalAlignment = Enum.VerticalAlignment.Center,
    }),
})

local content = make("Frame", {
    Position = UDim2.fromOffset(14, 92), Size = UDim2.new(1, -28, 1, -102),
    BackgroundTransparency = 1, Parent = window,
})

do -- drag by top bar
    local dragging, startInput, startPosition = false, nil, nil
    track(topbar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            startInput = input.Position
            startPosition = window.Position
        end
    end))
    track(UserInputService.InputChanged:Connect(function(input)
        if dragging and startInput and (
            input.UserInputType == Enum.UserInputType.MouseMovement
            or input.UserInputType == Enum.UserInputType.Touch
        ) then
            local d = input.Position - startInput
            window.Position = UDim2.new(
                startPosition.X.Scale, startPosition.X.Offset + d.X,
                startPosition.Y.Scale, startPosition.Y.Offset + d.Y
            )
        end
    end))
    track(UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end))
end

----------------------------------------------------------------
-- TABS / PAGES / CARDS
----------------------------------------------------------------

local pages, tabButtons = {}, {}
local tabOrder = 0

local function selectTab(name)
    for key, page in pairs(pages) do page.Visible = (key == name) end
    for key, b in pairs(tabButtons) do
        local on = key == name
        b.button.BackgroundColor3 = on and WHITE or T.panel2
        b.grad.Enabled = on
        b.button.TextColor3 = on and WHITE or T.muted
    end
end

local function newPage(key, label)
    tabOrder += 1
    local page = make("ScrollingFrame", {
        Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, BorderSizePixel = 0,
        ScrollBarThickness = 3, ScrollBarImageColor3 = T.a1,
        CanvasSize = UDim2.new(), AutomaticCanvasSize = Enum.AutomaticSize.Y,
        ScrollingDirection = Enum.ScrollingDirection.Y, Visible = false, Parent = content,
    }, {
        make("UIListLayout", {Padding = UDim.new(0, 8), SortOrder = Enum.SortOrder.LayoutOrder}),
        make("UIPadding", {PaddingRight = UDim.new(0, 6), PaddingBottom = UDim.new(0, 8)}),
    })
    themed(function() page.ScrollBarImageColor3 = T.a1 end)
    pages[key] = page

    local button = make("TextButton", {
        LayoutOrder = tabOrder, Size = UDim2.fromOffset(0, 28), AutomaticSize = Enum.AutomaticSize.X,
        BackgroundColor3 = T.panel2, Text = label, Font = Enum.Font.GothamBold, TextSize = 11,
        TextColor3 = T.muted, AutoButtonColor = false, Parent = tabBar,
    }, {
        make("UIPadding", {PaddingLeft = UDim.new(0, 14), PaddingRight = UDim.new(0, 14)}),
    })
    round(button, 14)
    local grad = accentGradient(button, 0)
    grad.Enabled = false
    tabButtons[key] = {button = button, grad = grad}
    button.Activated:Connect(function() selectTab(key) end)
    return page
end

local function card(page, name)
    local frame = make("Frame", {
        Size = UDim2.new(1, 0, 0, 0), AutomaticSize = Enum.AutomaticSize.Y,
        BackgroundColor3 = T.panel, BackgroundTransparency = 0.15, BorderSizePixel = 0, Parent = page,
    }, {
        make("UIListLayout", {Padding = UDim.new(0, 6), SortOrder = Enum.SortOrder.LayoutOrder}),
        make("UIPadding", {
            PaddingTop = UDim.new(0, 10), PaddingBottom = UDim.new(0, 11),
            PaddingLeft = UDim.new(0, 12), PaddingRight = UDim.new(0, 12),
        }),
    })
    round(frame, 14)
    stroke(frame, T.border, 0.35)
    local label = make("TextLabel", {
        Size = UDim2.new(1, 0, 0, 18), BackgroundTransparency = 1, Text = string.upper(name),
        Font = Enum.Font.GothamBlack, TextSize = 11, TextColor3 = WHITE,
        TextXAlignment = Enum.TextXAlignment.Left, Parent = frame,
    })
    accentGradient(label, 0)
    return frame
end

----------------------------------------------------------------
-- CONTROLS
----------------------------------------------------------------

local function toggle(parent, name, default, callback)
    local value = default
    local row = make("TextButton", {
        Size = UDim2.new(1, 0, 0, 30), BackgroundTransparency = 1, Text = "",
        AutoButtonColor = false, Parent = parent,
    })
    make("TextLabel", {
        Size = UDim2.new(1, -52, 1, 0), BackgroundTransparency = 1, Text = name,
        Font = Enum.Font.GothamMedium, TextSize = 12, TextColor3 = T.text,
        TextXAlignment = Enum.TextXAlignment.Left, Parent = row,
    })
    local switch = make("Frame", {
        AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, 0, 0.5, 0),
        Size = UDim2.fromOffset(40, 22), BackgroundColor3 = T.panel2, BorderSizePixel = 0, Parent = row,
    })
    round(switch, 11)
    local knob = make("Frame", {
        Position = UDim2.fromOffset(3, 3), Size = UDim2.fromOffset(16, 16),
        BackgroundColor3 = T.muted, BorderSizePixel = 0, Parent = switch,
    })
    round(knob, 8)

    local function set(v, fire)
        value = v
        tween(switch, 0.15, {BackgroundColor3 = v and T.a1 or T.panel2})
        tween(knob, 0.15, {
            Position = v and UDim2.new(1, -19, 0, 3) or UDim2.fromOffset(3, 3),
            BackgroundColor3 = v and WHITE or T.muted,
        })
        if fire and callback then callback(v) end
    end
    row.Activated:Connect(function() set(not value, true) end)
    set(value, false)
    themed(function() if value then switch.BackgroundColor3 = T.a1 end end)
    return {Set = set, Get = function() return value end}
end

local function button(parent, text, callback)
    local b = make("TextButton", {
        Size = UDim2.new(1, 0, 0, 32), BackgroundColor3 = T.panel2, Text = text,
        Font = Enum.Font.GothamBold, TextSize = 11, TextColor3 = T.text, AutoButtonColor = false, Parent = parent,
    })
    round(b, 10)
    stroke(b, T.border, 0.4)
    b.Activated:Connect(function()
        tween(b, 0.08, {BackgroundColor3 = T.a2})
        task.delay(0.12, function()
            if b.Parent then tween(b, 0.2, {BackgroundColor3 = T.panel2}) end
        end)
        callback()
    end)
    return b
end

local function slider(parent, name, min, max, default, step, callback)
    local row = make("Frame", {Size = UDim2.new(1, 0, 0, 44), BackgroundTransparency = 1, Parent = parent})
    make("TextLabel", {
        Size = UDim2.new(1, -60, 0, 16), BackgroundTransparency = 1, Text = name,
        Font = Enum.Font.GothamMedium, TextSize = 12, TextColor3 = T.text,
        TextXAlignment = Enum.TextXAlignment.Left, Parent = row,
    })
    local valueLabel = make("TextLabel", {
        AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, 0, 0, 0), Size = UDim2.new(0, 60, 0, 16),
        BackgroundTransparency = 1, Font = Enum.Font.GothamBold, TextSize = 12,
        TextXAlignment = Enum.TextXAlignment.Right, Parent = row,
    })
    themed(function() valueLabel.TextColor3 = T.a1 end)
    local hitbox = make("TextButton", {
        Position = UDim2.fromOffset(0, 18), Size = UDim2.new(1, 0, 0, 24),
        BackgroundTransparency = 1, Text = "", AutoButtonColor = false, Parent = row,
    })
    local rail = make("Frame", {
        AnchorPoint = Vector2.new(0, 0.5), Position = UDim2.fromScale(0, 0.5),
        Size = UDim2.new(1, 0, 0, 5), BackgroundColor3 = T.panel2, BorderSizePixel = 0, Parent = hitbox,
    })
    round(rail, 3)
    local fill = make("Frame", {Size = UDim2.fromScale(0, 1), BackgroundColor3 = WHITE, BorderSizePixel = 0, Parent = rail})
    round(fill, 3)
    accentGradient(fill, 0)
    local knob = make("Frame", {
        AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0, 0.5),
        Size = UDim2.fromOffset(14, 14), BackgroundColor3 = WHITE, BorderSizePixel = 0, ZIndex = 2, Parent = rail,
    })
    round(knob, 7)
    local knobStroke = stroke(knob, T.a1, 0.2, 2)
    themed(function() knobStroke.Color = T.a1 end)

    local value = default
    local decimals = step >= 1 and 0 or (step >= 0.1 and 1 or 2)
    local function set(v, fire)
        v = math.clamp(min + math.floor((v - min) / step + 0.5) * step, min, max)
        value = v
        local rel = (v - min) / (max - min)
        fill.Size = UDim2.fromScale(rel, 1)
        knob.Position = UDim2.fromScale(rel, 0.5)
        valueLabel.Text = string.format("%." .. decimals .. "f", v)
        if fire and callback then callback(v) end
    end
    local dragging = false
    local scroller = parent:FindFirstAncestorOfClass("ScrollingFrame")
    local function update(x)
        if rail.AbsoluteSize.X <= 0 then return end
        local rel = math.clamp((x - rail.AbsolutePosition.X) / rail.AbsoluteSize.X, 0, 1)
        set(min + (max - min) * rel, true)
    end
    hitbox.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            if scroller then scroller.ScrollingEnabled = false end
            update(input.Position.X)
        end
    end)
    track(UserInputService.InputChanged:Connect(function(input)
        if dragging and (
            input.UserInputType == Enum.UserInputType.MouseMovement
            or input.UserInputType == Enum.UserInputType.Touch
        ) then update(input.Position.X) end
    end))
    track(UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
            if scroller then scroller.ScrollingEnabled = true end
        end
    end))
    set(default, false)
    return {Set = set, Get = function() return value end}
end

local function chips(parent, names, selected, callback)
    local rows = math.ceil(#names / 2)
    local holder = make("Frame", {Size = UDim2.new(1, 0, 0, rows * 33), BackgroundTransparency = 1, Parent = parent})
    make("UIGridLayout", {
        CellSize = UDim2.new(0.5, -4, 0, 28), CellPadding = UDim2.fromOffset(8, 5),
        SortOrder = Enum.SortOrder.LayoutOrder, Parent = holder,
    })
    local items = {}
    local function pick(name, fire)
        selected = name
        for itemName, b in pairs(items) do
            local active = itemName == name
            b.BackgroundColor3 = active and T.a2 or T.panel2
            b.TextColor3 = active and WHITE or T.muted
        end
        if fire and callback then callback(name) end
    end
    for i, name in ipairs(names) do
        local b = make("TextButton", {
            LayoutOrder = i, BackgroundColor3 = T.panel2, Text = name, Font = Enum.Font.GothamMedium,
            TextSize = 11, TextColor3 = T.muted, AutoButtonColor = false, Parent = holder,
        })
        round(b, 9)
        items[name] = b
        b.Activated:Connect(function() pick(name, true) end)
    end
    pick(selected, false)
    themed(function() pick(selected, false) end)
    return {Pick = pick}
end

local PALETTE = {
    C(70, 215, 255), C(160, 105, 255), C(255, 95, 190), C(95, 235, 170),
    C(255, 205, 90), C(245, 245, 255), C(255, 105, 90), C(90, 120, 255),
}

local function colorPicker(parent, label, initial, callback)
    local row = make("Frame", {Size = UDim2.new(1, 0, 0, 46), BackgroundTransparency = 1, Parent = parent})
    make("TextLabel", {
        Size = UDim2.new(1, 0, 0, 18), BackgroundTransparency = 1, Text = label,
        Font = Enum.Font.GothamMedium, TextSize = 12, TextColor3 = T.text,
        TextXAlignment = Enum.TextXAlignment.Left, Parent = row,
    })
    local swatches = make("Frame", {
        Position = UDim2.fromOffset(0, 22), Size = UDim2.new(1, 0, 0, 22), BackgroundTransparency = 1, Parent = row,
    }, {
        make("UIListLayout", {
            FillDirection = Enum.FillDirection.Horizontal, Padding = UDim.new(0, 7), SortOrder = Enum.SortOrder.LayoutOrder,
        }),
    })
    local strokes = {}
    local function mark(color)
        for col, s in pairs(strokes) do s.Transparency = (col == color) and 0 or 1 end
    end
    for i, color in ipairs(PALETTE) do
        local swatch = make("TextButton", {
            LayoutOrder = i, Size = UDim2.fromOffset(26, 22), BackgroundColor3 = color,
            Text = "", AutoButtonColor = false, Parent = swatches,
        })
        round(swatch, 7)
        strokes[color] = stroke(swatch, WHITE, 1, 2)
        swatch.Activated:Connect(function()
            mark(color)
            callback(color)
        end)
    end
    mark(initial)
end

----------------------------------------------------------------
-- ISLAND + TOAST
----------------------------------------------------------------

local H = {}
local currentTarget
local quickItems = {}

local function refreshQuick()
    for _, it in ipairs(quickItems) do
        local on = it.on and it.on()
        it.b.BackgroundColor3 = on and T.a2 or T.panel2
        it.b.TextColor3 = on and WHITE or T.muted
    end
end

local island = make("Frame", {
    AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.new(0.5, 0, 0, 8),
    Size = UDim2.fromOffset(190, 30), BackgroundColor3 = C(8, 9, 14), BackgroundTransparency = 0.1,
    BorderSizePixel = 0, ClipsDescendants = true, Parent = gui,
})
local islandCorner = round(island, 15)
accentGradient(stroke(island, WHITE, 0.15, 1.3), 0, true)

local islandDot = make("Frame", {
    Position = UDim2.fromOffset(13, 11), Size = UDim2.fromOffset(8, 8),
    BackgroundColor3 = T.green, BorderSizePixel = 0, Parent = island,
})
round(islandDot, 4)
local islandText = make("TextLabel", {
    Position = UDim2.fromOffset(28, 0), Size = UDim2.new(1, -36, 0, 30), BackgroundTransparency = 1,
    Text = "KIT VISUALS", Font = Enum.Font.GothamBold, TextSize = 11, TextColor3 = T.text,
    TextXAlignment = Enum.TextXAlignment.Left, Parent = island,
})
local islandHit = make("TextButton", {
    Size = UDim2.new(1, 0, 0, 30), BackgroundTransparency = 1, Text = "",
    AutoButtonColor = false, ZIndex = 5, Parent = island,
})
local quickRow = make("Frame", {
    Position = UDim2.fromOffset(10, 38), Size = UDim2.new(1, -20, 0, 28), BackgroundTransparency = 1, Parent = island,
}, {
    make("UIListLayout", {
        FillDirection = Enum.FillDirection.Horizontal, Padding = UDim.new(0, 5), SortOrder = Enum.SortOrder.LayoutOrder,
    }),
})

local islandOpen = false
local islandToken = 0
local function setIsland(open)
    islandOpen = open
    islandToken += 1
    local id = islandToken
    tween(island, 0.35, {Size = open and UDim2.fromOffset(330, 76) or UDim2.fromOffset(190, 30)}, Enum.EasingStyle.Back)
    tween(islandCorner, 0.35, {CornerRadius = UDim.new(0, open and 20 or 15)})
    if open then
        refreshQuick()
        task.delay(5, function()
            if alive and islandOpen and id == islandToken then setIsland(false) end
        end)
    end
end
islandHit.Activated:Connect(function() setIsland(not islandOpen) end)

local function quick(order, text, fn, isOn)
    local b = make("TextButton", {
        LayoutOrder = order, Size = UDim2.new(0.2, -4, 1, 0), BackgroundColor3 = T.panel2,
        Text = text, Font = Enum.Font.GothamBold, TextSize = 9, TextColor3 = T.muted,
        AutoButtonColor = false, Parent = quickRow,
    })
    round(b, 9)
    table.insert(quickItems, {b = b, on = isOn})
    b.Activated:Connect(function()
        fn()
        refreshQuick()
    end)
end

local toast = make("TextLabel", {
    AnchorPoint = Vector2.new(0.5, 0), Position = UDim2.new(0.5, 0, 0, -60), Size = UDim2.fromOffset(240, 28),
    BackgroundColor3 = C(8, 9, 14), BackgroundTransparency = 0.1, Text = "", Font = Enum.Font.GothamMedium,
    TextSize = 11, TextColor3 = T.text, BorderSizePixel = 0, Parent = gui,
})
round(toast, 14)
local toastStroke = stroke(toast, T.a2, 0.3)
themed(function() toastStroke.Color = T.a2 end)
local toastId = 0
local function notify(message)
    toastId += 1
    local id = toastId
    toast.Text = message
    toast.Position = UDim2.new(0.5, 0, 0, -60)
    tween(toast, 0.25, {Position = UDim2.new(0.5, 0, 0, islandOpen and 86 or 44)})
    task.delay(1.8, function()
        if alive and id == toastId then
            tween(toast, 0.25, {Position = UDim2.new(0.5, 0, 0, -60)})
        end
    end)
end

----------------------------------------------------------------
-- LIGHTING BACKUP / RESTORE
----------------------------------------------------------------

local function snapshot(object, properties)
    local result = {}
    for _, property in ipairs(properties) do
        pcall(function() result[property] = object[property] end)
    end
    return result
end
local function restore(object, properties)
    for property, value in pairs(properties or {}) do
        pcall(function() object[property] = value end)
    end
end

local lightingBackup = snapshot(Lighting, {
    "ClockTime", "Ambient", "OutdoorAmbient", "Brightness",
    "ExposureCompensation", "GlobalShadows", "FogColor", "FogStart", "FogEnd",
})
local oldAtmosphere = Lighting:FindFirstChildOfClass("Atmosphere")
local oldAtmosphereBackup = oldAtmosphere and snapshot(oldAtmosphere, {"Color", "Decay", "Density", "Offset", "Glare", "Haze"})
local oldCC = Lighting:FindFirstChildOfClass("ColorCorrectionEffect")
local oldCCBackup = oldCC and snapshot(oldCC, {"Enabled", "TintColor", "Saturation", "Contrast", "Brightness"})
local oldBloom = Lighting:FindFirstChildOfClass("BloomEffect")
local oldBloomBackup = oldBloom and snapshot(oldBloom, {"Enabled", "Intensity", "Size", "Threshold"})

local atmosphere, cc, bloom, rays, dof
local madeAtmosphere, madeCC, madeBloom, madeRays, madeDOF = false, false, false, false, false

local function getAtmosphere()
    if atmosphere and atmosphere.Parent then return atmosphere end
    atmosphere = oldAtmosphere
    if not atmosphere then
        atmosphere = Instance.new("Atmosphere")
        atmosphere.Name = "KV_Atmosphere"
        atmosphere.Parent = Lighting
        madeAtmosphere = true
    end
    return atmosphere
end
local function getCC()
    if cc and cc.Parent then return cc end
    cc = Instance.new("ColorCorrectionEffect")
    cc.Name = "KV_Color"
    cc.Enabled = false
    cc.Parent = Lighting
    madeCC = true
    return cc
end
local function getBloom()
    if bloom and bloom.Parent then return bloom end
    bloom = Instance.new("BloomEffect")
    bloom.Name = "KV_Bloom"
    bloom.Enabled = false
    bloom.Size = 24
    bloom.Threshold = 1.1
    bloom.Parent = Lighting
    madeBloom = true
    return bloom
end
local function getRays()
    if rays and rays.Parent then return rays end
    rays = Instance.new("SunRaysEffect")
    rays.Name = "KV_Rays"
    rays.Enabled = false
    rays.Intensity = 0.12
    rays.Spread = 0.7
    rays.Parent = Lighting
    madeRays = true
    return rays
end
local function getDOF()
    if dof and dof.Parent then return dof end
    dof = Instance.new("DepthOfFieldEffect")
    dof.Name = "KV_DOF"
    dof.Enabled = false
    dof.FarIntensity = 0.18
    dof.NearIntensity = 0
    dof.FocusDistance = 28
    dof.InFocusRadius = 55
    dof.Parent = Lighting
    madeDOF = true
    return dof
end

local S = {
    World = true, Time = 17.7, Density = 0.3, Haze = 1.5,
    Fog = C(255, 165, 130), Decay = C(255, 100, 100),
    Ambient = C(95, 75, 100), Outdoor = C(180, 130, 130),
    Tint = C(255, 225, 215), Sat = 0.1, Contrast = 0.1,
    Brightness = 0, Exposure = 0, Bloom = true, BloomI = 0.5,
    Rays = false, Shadows = true,
    Rainbow = false, Cycle = false, CycleSpeed = 0.1, DOF = false,
}

local PRESETS = {
    ["Закат"] = {Time = 17.7, Density = 0.3, Haze = 1.5, Fog = C(255, 165, 130), Decay = C(255, 100, 100),
        Ambient = C(95, 75, 100), Outdoor = C(180, 130, 130), Tint = C(255, 225, 215), Sat = 0.1, Contrast = 0.1},
    ["Рассвет"] = {Time = 6.2, Density = 0.18, Haze = 1.1, Fog = C(255, 205, 160), Decay = C(255, 150, 125),
        Ambient = C(110, 90, 100), Outdoor = C(190, 160, 145), Tint = C(255, 240, 220), Sat = 0.12, Contrast = 0.05},
    ["Космос"] = {Time = 0.3, Density = 0.2, Haze = 1.2, Fog = C(55, 65, 145), Decay = C(20, 20, 65),
        Ambient = C(35, 40, 90), Outdoor = C(60, 65, 140), Tint = C(190, 205, 255), Sat = 0.15, Contrast = 0.15},
    ["Полночь"] = {Time = 0, Density = 0.28, Haze = 1.6, Fog = C(35, 45, 85), Decay = C(15, 20, 45),
        Ambient = C(25, 30, 55), Outdoor = C(45, 55, 90), Tint = C(190, 205, 255), Sat = -0.15, Contrast = 0.12},
    ["Мороз"] = {Time = 9, Density = 0.22, Haze = 1.8, Fog = C(180, 220, 255), Decay = C(130, 180, 255),
        Ambient = C(105, 130, 165), Outdoor = C(180, 205, 235), Tint = C(220, 240, 255), Sat = -0.35, Contrast = 0.05},
    ["Синтвейв"] = {Time = 19.2, Density = 0.3, Haze = 2, Fog = C(255, 75, 195), Decay = C(105, 50, 255),
        Ambient = C(75, 35, 110), Outdoor = C(150, 65, 180), Tint = C(255, 200, 255), Sat = 0.3, Contrast = 0.15},
    ["Изумруд"] = {Time = 12, Density = 0.28, Haze = 1.7, Fog = C(90, 255, 170), Decay = C(25, 110, 65),
        Ambient = C(35, 90, 60), Outdoor = C(90, 160, 110), Tint = C(210, 255, 225), Sat = 0.15, Contrast = 0.1},
    ["Кровавая луна"] = {Time = 23.5, Density = 0.32, Haze = 2.2, Fog = C(190, 30, 45), Decay = C(90, 10, 20),
        Ambient = C(70, 25, 35), Outdoor = C(120, 40, 55), Tint = C(255, 190, 190), Sat = 0.1, Contrast = 0.2},
}
local PRESET_NAMES = {"Закат", "Рассвет", "Космос", "Полночь", "Мороз", "Синтвейв", "Изумруд", "Кровавая луна"}

local function applyWorld()
    if not alive or not S.World then return end
    pcall(function()
        Lighting.ClockTime = S.Time
        Lighting.Ambient = S.Ambient
        Lighting.OutdoorAmbient = S.Outdoor
        Lighting.ExposureCompensation = S.Exposure
        Lighting.GlobalShadows = S.Shadows

        local a = getAtmosphere()
        a.Color = S.Fog
        a.Decay = S.Decay
        a.Density = S.Density
        a.Haze = S.Haze

        local color = getCC()
        color.Enabled = true
        color.TintColor = S.Tint
        color.Saturation = S.Sat
        color.Contrast = S.Contrast
        color.Brightness = S.Brightness

        local b = getBloom()
        b.Enabled = S.Bloom
        b.Intensity = S.BloomI

        getRays().Enabled = S.Rays
        getDOF().Enabled = S.DOF
    end)
end

local function restoreWorld()
    restore(Lighting, lightingBackup)
    if madeAtmosphere and atmosphere then atmosphere:Destroy(); atmosphere = nil; madeAtmosphere = false
    elseif oldAtmosphere then restore(oldAtmosphere, oldAtmosphereBackup) end
    if madeCC and cc then cc:Destroy(); cc = nil; madeCC = false
    elseif oldCC then restore(oldCC, oldCCBackup) end
    if madeBloom and bloom then bloom:Destroy(); bloom = nil; madeBloom = false
    elseif oldBloom then restore(oldBloom, oldBloomBackup) end
    if madeRays and rays then rays:Destroy(); rays = nil; madeRays = false end
    if madeDOF and dof then dof:Destroy(); dof = nil; madeDOF = false end
end

----------------------------------------------------------------
-- FALLING PARTICLES
----------------------------------------------------------------

local P = {Enabled = true, Style = "Снег", Count = 90, Speed = 1, Size = 1, Radius = 45, Wind = 0}
local STYLES = {
    ["Снег"] = {colors = {C(245,250,255), C(190,225,255)}, size = {0.12,0.25}, speed = {5,9}, sway = 1},
    ["Сакура"] = {colors = {C(255,165,205), C(255,210,230)}, size = {0.18,0.32}, speed = {2,4}, sway = 3},
    ["Звёзды"] = {colors = {C(255,245,160), C(180,220,255), C(255,255,255)}, size = {0.12,0.22}, speed = {7,12}, sway = 0.5},
    ["Пепел"] = {colors = {C(125,125,135), C(220,120,75)}, size = {0.12,0.22}, speed = {1.5,3}, sway = 2},
    ["Неон"] = {colors = {T.pink, T.a1, T.a2, T.green}, size = {0.13,0.25}, speed = {3,7}, sway = 2},
    ["Светлячки"] = {colors = {C(180,255,110), C(255,245,130), C(100,255,200)}, size = {0.12,0.22}, speed = {0.5,1.8}, sway = 2.5},
}
local pool = {}
local particleClock = 0

local function spawnParticle(p)
    local camera = workspace.CurrentCamera
    if not camera then return end
    local pos = camera.CFrame.Position
    local angle = math.random() * math.pi * 2
    local radius = math.sqrt(math.random()) * P.Radius
    p.pos = Vector3.new(pos.X + math.cos(angle) * radius, pos.Y + randomRange(8, 36), pos.Z + math.sin(angle) * radius)
    p.part.Position = p.pos
    p.part.Transparency = 0.12
end

local function styleParticle(p)
    local style = STYLES[P.Style]
    p.baseSize = randomRange(style.size[1], style.size[2])
    p.fall = randomRange(style.speed[1], style.speed[2])
    p.sway = style.sway
    p.phase = math.random() * 6.28
    p.part.Color = style.colors[math.random(1, #style.colors)]
    p.part.Size = Vector3.one * p.baseSize * P.Size
    spawnParticle(p)
end

local function newParticle()
    local part = Instance.new("Part")
    part.Name = "KV_Particle"
    part.Shape = Enum.PartType.Ball
    part.Anchored = true
    part.CanCollide = false
    part.CanTouch = false
    part.CanQuery = false
    part.CastShadow = false
    part.Size = Vector3.one * 0.15
    part.Material = Enum.Material.Neon
    part.Parent = particleFolder
    local p = {part = part}
    styleParticle(p)
    return p
end

local function setParticleCount(n)
    n = math.clamp(math.floor(n), 0, 180)
    while #pool < n do table.insert(pool, newParticle()) end
    while #pool > n do
        local p = table.remove(pool)
        p.part:Destroy()
    end
end
local function clearParticles()
    for _, p in ipairs(pool) do
        if p.part then p.part:Destroy() end
    end
    table.clear(pool)
end
local function restyleParticles()
    for _, p in ipairs(pool) do styleParticle(p) end
end

local function stepParticles(dt)
    if not P.Enabled or #pool == 0 then return end
    local camera = workspace.CurrentCamera
    if not camera then return end
    local cam = camera.CFrame.Position
    for _, p in ipairs(pool) do
        local swayX = math.sin(particleClock * 1.4 + p.phase) * p.sway
        local swayZ = math.cos(particleClock + p.phase) * p.sway * 0.5
        if P.Style == "Светлячки" then
            p.pos += Vector3.new(swayX, math.sin(particleClock * 1.7 + p.phase) * 0.5, swayZ) * dt
            p.part.Transparency = 0.15 + (math.sin(particleClock * 3 + p.phase) + 1) * 0.3
        else
            p.pos += Vector3.new(swayX + P.Wind, -p.fall * P.Speed, swayZ) * dt
            if p.pos.Y < cam.Y - 12 then spawnParticle(p) end
        end
        local dx, dz = p.pos.X - cam.X, p.pos.Z - cam.Z
        if dx * dx + dz * dz > (P.Radius * 1.5) ^ 2 then spawnParticle(p) end
        p.part.Position = p.pos
        p.part.Size = Vector3.one * p.baseSize * P.Size
    end
end

----------------------------------------------------------------
-- SNOW ON GROUND (raycast tiles: works on any geometry)
----------------------------------------------------------------

local SnowB = {Enabled = false, Radius = 90, Step = 4, Thick = 0.25, Color = C(240, 247, 255)}
local SNOW_CASTS, SNOW_REFRESH = 60, 10
local snowCells, snowQueue = {}, {}
local snowQi, snowCenter, snowBuilt = 1, nil, 0

local snowRay = RaycastParams.new()
snowRay.FilterType = Enum.RaycastFilterType.Exclude
snowRay.IgnoreWater = true

local function snowExclude()
    local list = {snowCaps, particleFolder, targetFolder, killFolder, skyFolder, selfFolder}
    for _, p in ipairs(Players:GetPlayers()) do
        if p.Character then table.insert(list, p.Character) end
    end
    snowRay.FilterDescendantsInstances = list
end

local function snowCast(x, z, y0)
    local origin = Vector3.new(x, y0, z)
    local dir = Vector3.new(0, -450, 0)
    for _ = 1, 4 do
        local r = workspace:Raycast(origin, dir, snowRay)
        if not r then return nil end
        local inst = r.Instance
        local bad = inst.Transparency > 0.5 or inst.Material == Enum.Material.ForceField
        local mdl = inst:FindFirstAncestorOfClass("Model")
        if mdl and mdl:FindFirstChildOfClass("Humanoid") then bad = true end
        if not bad then return r end
        origin = r.Position - Vector3.new(0, 0.1, 0)
    end
    return nil
end

local function placeCap(cell, r)
    local st = SnowB.Step
    local part = cell.part
    if not part then
        part = Instance.new("Part")
        part.Name = "KV_SnowCap"
        part.Anchored = true
        part.CanCollide = false
        part.CanTouch = false
        part.CanQuery = false
        part.CastShadow = false
        part.Material = Enum.Material.Snow
        part.Parent = snowCaps
        cell.part = part
    end
    local n = r.Normal
    part.Color = SnowB.Color:Lerp(C(205, 225, 248), math.random() * 0.3)
    part.Size = Vector3.new(st + 0.15, SnowB.Thick, st + 0.15)
    local pos = r.Position + n * (SnowB.Thick / 2 - 0.02)
    if n.Y > 0.99 then
        part.CFrame = CFrame.new(pos)
    else
        part.CFrame = CFrame.lookAt(pos, pos + n) * CFrame.Angles(-math.pi / 2, 0, 0)
    end
end

local function clearSnow()
    for _, cell in pairs(snowCells) do
        if cell.part then cell.part:Destroy() end
    end
    table.clear(snowCells)
    table.clear(snowQueue)
    snowQi, snowCenter = 1, nil
    snowCaps:ClearAllChildren()
end

local function buildSnowQueue(cx, cz)
    local st = SnowB.Step
    local n = math.ceil(SnowB.Radius / st)
    table.clear(snowQueue)
    for dx = -n, n do
        for dz = -n, n do
            local d = dx * dx + dz * dz
            if d <= n * n then table.insert(snowQueue, {cx + dx, cz + dz, d}) end
        end
    end
    table.sort(snowQueue, function(a, b) return a[3] < b[3] end)
    snowQi = 1
    snowCenter = {cx, cz}
    snowBuilt = os.clock()
    snowExclude()
    for key, cell in pairs(snowCells) do
        local ddx, ddz = cell.gx - cx, cell.gz - cz
        if ddx * ddx + ddz * ddz > (n + 3) ^ 2 then
            if cell.part then cell.part:Destroy() end
            snowCells[key] = nil
        end
    end
end

local function stepSnow()
    if not SnowB.Enabled then return end
    local cam = workspace.CurrentCamera
    if not cam then return end
    local pos = cam.CFrame.Position
    local st = SnowB.Step
    local cx, cz = math.floor(pos.X / st + 0.5), math.floor(pos.Z / st + 0.5)
    local now = os.clock()
    if not snowCenter
    or math.abs(snowCenter[1] - cx) + math.abs(snowCenter[2] - cz) >= 3
    or (snowQi > #snowQueue and now - snowBuilt > 3) then
        buildSnowQueue(cx, cz)
    end
    local y0 = pos.Y + 60
    local casts, iter = 0, 0
    while casts < SNOW_CASTS and iter < 500 do
        iter += 1
        local q = snowQueue[snowQi]
        if not q then break end
        snowQi += 1
        local key = q[1] .. ":" .. q[2]
        local cell = snowCells[key]
        if not cell or now - cell.t > SNOW_REFRESH then
            casts += 1
            if not cell then
                cell = {gx = q[1], gz = q[2], t = now}
                snowCells[key] = cell
            end
            cell.t = now
            local r = snowCast(q[1] * st, q[2] * st, y0)
            if r and r.Normal.Y > 0.55 then
                placeCap(cell, r)
            elseif cell.part then
                cell.part:Destroy()
                cell.part = nil
            end
        end
    end
end

----------------------------------------------------------------
-- SCREEN FX
----------------------------------------------------------------

local FX = {Aurora = false, Vignette = false, Cinematic = false}

local aurora = make("Frame", {
    Position = UDim2.new(-0.1, 0, 0, 0), Size = UDim2.new(1.2, 0, 0.38, 0),
    BackgroundColor3 = C(100, 255, 200), BackgroundTransparency = 0.78,
    BorderSizePixel = 0, Visible = false, Parent = fxGui,
})
local auroraGradient = make("UIGradient", {
    Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, C(70, 215, 255)),
        ColorSequenceKeypoint.new(0.5, T.green),
        ColorSequenceKeypoint.new(1, C(160, 105, 255)),
    }),
    Transparency = NumberSequence.new({
        NumberSequenceKeypoint.new(0, 1), NumberSequenceKeypoint.new(0.3, 0.25),
        NumberSequenceKeypoint.new(0.7, 0.25), NumberSequenceKeypoint.new(1, 1),
    }),
    Parent = aurora,
})

local vignette = make("Frame", {Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Visible = false, Parent = fxGui})
local function addVignetteEdge(position, size, rotation)
    local edge = make("Frame", {
        Position = position, Size = size, BackgroundColor3 = Color3.new(0, 0, 0), BorderSizePixel = 0, Parent = vignette,
    })
    make("UIGradient", {
        Rotation = rotation,
        Transparency = NumberSequence.new({NumberSequenceKeypoint.new(0, 0.05), NumberSequenceKeypoint.new(1, 1)}),
        Parent = edge,
    })
end
addVignetteEdge(UDim2.new(0, 0, 0, 0), UDim2.new(1, 0, 0.22, 0), 90)
addVignetteEdge(UDim2.new(0, 0, 0.78, 0), UDim2.new(1, 0, 0.22, 0), 270)
addVignetteEdge(UDim2.new(0, 0, 0, 0), UDim2.new(0.13, 0, 1, 0), 0)
addVignetteEdge(UDim2.new(0.87, 0, 0, 0), UDim2.new(0.13, 0, 1, 0), 180)

local cinematicEffect
local function setCinematic(enabled)
    FX.Cinematic = enabled
    if enabled then
        if not cinematicEffect or not cinematicEffect.Parent then
            cinematicEffect = Instance.new("ColorCorrectionEffect")
            cinematicEffect.Name = "KV_Cinematic"
            cinematicEffect.TintColor = C(210, 225, 255)
            cinematicEffect.Saturation = -0.12
            cinematicEffect.Contrast = 0.12
            cinematicEffect.Parent = Lighting
        end
        cinematicEffect.Enabled = true
    elseif cinematicEffect then
        cinematicEffect.Enabled = false
    end
end

local killFlash = make("Frame", {
    Size = UDim2.fromScale(1, 1), BackgroundColor3 = WHITE, BackgroundTransparency = 1, BorderSizePixel = 0, Parent = fxGui,
})
local lightFlash = make("Frame", {
    Size = UDim2.fromScale(1, 1), BackgroundColor3 = C(205, 225, 255), BackgroundTransparency = 1, BorderSizePixel = 0, Parent = fxGui,
})

----------------------------------------------------------------
-- TARGET FX (контур + кольцо + луч + орбиты, БЕЗ таблички)
----------------------------------------------------------------

local TargetFX = {
    Enabled = false, Style = "Кристаллы", Color = C(160, 105, 255),
    Count = 6, Radius = 3.5, Height = 1.5, Speed = 1.5, Range = 70,
    Highlight = true, Ring = true, Tracer = false,
}
local tObjs, orbit, ringOuter, ringInner = {}, {}, {}, {}
local builtFor, builtMyRoot, builtChar
local targetClock, targetDirty = 0, true

local function clearTarget()
    for _, o in ipairs(tObjs) do pcall(function() o:Destroy() end) end
    table.clear(tObjs)
    table.clear(orbit)
    table.clear(ringOuter)
    table.clear(ringInner)
end

local function tPart(shape, size, material, color)
    local part = Instance.new("Part")
    part.Name = "KV_T"
    part.Shape = shape or Enum.PartType.Block
    part.Anchored = true
    part.CanCollide = false
    part.CanTouch = false
    part.CanQuery = false
    part.CastShadow = false
    part.Size = size
    part.Color = color or TargetFX.Color
    part.Material = material or Enum.Material.Neon
    part.Transparency = 0.1
    part.Parent = targetFolder
    table.insert(tObjs, part)
    return part
end

local function buildTarget(target)
    clearTarget()
    if not target or not TargetFX.Enabled then return end
    local char = target.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if not root then return end
    local col = TargetFX.Color
    local n = math.clamp(math.floor(TargetFX.Count), 2, 12)

    if TargetFX.Highlight then
        local hl = make("Highlight", {
            Adornee = char, FillColor = col, FillTransparency = 0.72,
            OutlineColor = WHITE, OutlineTransparency = 0.05,
            DepthMode = Enum.HighlightDepthMode.Occluded, Parent = targetFolder,
        })
        table.insert(tObjs, hl)
    end

    if TargetFX.Ring then
        for _ = 1, 24 do
            table.insert(ringOuter, tPart(Enum.PartType.Block, Vector3.new(0.25, 0.1, 0.8), Enum.Material.Neon, col))
        end
        for _ = 1, 14 do
            table.insert(ringInner, tPart(Enum.PartType.Block, Vector3.new(0.18, 0.08, 0.5), Enum.Material.Neon, col:Lerp(WHITE, 0.5)))
        end
    end

    if TargetFX.Tracer then
        local myRoot = getRoot(player)
        if myRoot then
            local a0 = Instance.new("Attachment")
            a0.Parent = myRoot
            local a1 = Instance.new("Attachment")
            a1.Parent = root
            local beam = make("Beam", {
                Attachment0 = a0, Attachment1 = a1, Width0 = 0.14, Width1 = 0.05,
                Color = ColorSequence.new(col), LightEmission = 1,
                Transparency = NumberSequence.new(0.25), FaceCamera = true, Parent = targetFolder,
            })
            table.insert(tObjs, a0)
            table.insert(tObjs, a1)
            table.insert(tObjs, beam)
        end
    end

    local style = TargetFX.Style
    local function add(count, shape, size, lerp)
        for i = 1, count do
            local p = tPart(shape, size, Enum.Material.Neon, col:Lerp(WHITE, lerp(i)))
            table.insert(orbit, {part = p, i = i})
        end
    end
    if style == "Кристаллы" then
        add(n, Enum.PartType.Block, Vector3.new(0.3, 0.95, 0.3), function(i) return (i % 3) * 0.2 end)
    elseif style == "Цепи" then
        add(n * 3, Enum.PartType.Block, Vector3.new(0.14, 0.14, 0.55), function() return 0.2 end)
    elseif style == "Руны" then
        add(n * 2, Enum.PartType.Ball, Vector3.new(0.28, 0.28, 0.28), function(i) return (i % 2) * 0.35 end)
    elseif style == "Нимб" then
        add(n * 3, Enum.PartType.Block, Vector3.new(0.12, 0.12, 0.42), function() return 0.4 end)
    elseif style == "Смерч" then
        add(n * 4, Enum.PartType.Ball, Vector3.new(0.25, 0.25, 0.25), function(i) return (i % 3) * 0.2 end)
    end
end

local function nearestTarget()
    local myRoot = getRoot(player)
    if not myRoot then return nil end
    local closest, closestDistance = nil, TargetFX.Range
    local currentDistance
    for _, other in ipairs(Players:GetPlayers()) do
        if other ~= player then
            local character = other.Character
            local root = character and character:FindFirstChild("HumanoidRootPart")
            local humanoid = character and character:FindFirstChildOfClass("Humanoid")
            if root and humanoid and humanoid.Health > 0 then
                local distance = (root.Position - myRoot.Position).Magnitude
                if other == currentTarget then currentDistance = distance end
                if distance <= closestDistance then
                    closest, closestDistance = other, distance
                end
            end
        end
    end
    if currentTarget and currentDistance and currentDistance <= TargetFX.Range
    and closest ~= currentTarget and currentDistance - closestDistance < 4 then
        return currentTarget
    end
    return closest
end

local function updateTarget()
    local t = builtFor
    if not t then return end
    local char = t.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    if not root or not hum or hum.Health <= 0 then
        clearTarget()
        builtFor, builtChar = nil, nil
        currentTarget = nil
        return
    end

    local center = root.Position
    local tc = targetClock
    local R, Hh = TargetFX.Radius, TargetFX.Height

    if #ringOuter > 0 then
        local drop = hum.HipHeight > 0 and (hum.HipHeight + root.Size.Y / 2) or 3
        local floorY = center.Y - drop + 0.12
        local pulse = 1 + math.sin(tc * 3) * 0.05
        local count = #ringOuter
        for i, p in ipairs(ringOuter) do
            local a = tc * 0.8 + 2 * math.pi * i / count
            local r = R * pulse
            local pos = Vector3.new(center.X + math.cos(a) * r, floorY, center.Z + math.sin(a) * r)
            local tangent = Vector3.new(-math.sin(a), 0, math.cos(a))
            p.CFrame = CFrame.lookAt(pos, pos + tangent)
            p.Size = Vector3.new(0.25, 0.1, 2 * math.pi * r / count * 0.6)
            p.Transparency = 0.05 + (math.sin(tc * 4 + i) + 1) * 0.1
        end
        local icount = #ringInner
        for i, p in ipairs(ringInner) do
            local a = -tc * 1.3 + 2 * math.pi * i / icount
            local r = R * 0.6
            local pos = Vector3.new(center.X + math.cos(a) * r, floorY, center.Z + math.sin(a) * r)
            local tangent = Vector3.new(-math.sin(a), 0, math.cos(a))
            p.CFrame = CFrame.lookAt(pos, pos + tangent)
            p.Size = Vector3.new(0.18, 0.08, 2 * math.pi * r / icount * 0.55)
        end
    end

    local count = #orbit
    local style = TargetFX.Style
    if count > 0 then
        if style == "Кристаллы" then
            for _, o in ipairs(orbit) do
                local a = tc + 2 * math.pi * o.i / count
                local y = Hh + math.sin(tc * 2 + o.i) * 0.3
                local pos = center + Vector3.new(math.cos(a) * R, y, math.sin(a) * R)
                o.part.CFrame = CFrame.new(pos) * CFrame.Angles(tc * 1.3 + o.i, a, tc)
            end
        elseif style == "Цепи" then
            local half = math.ceil(count / 2)
            for _, o in ipairs(orbit) do
                local ring = o.i <= half and 1 or 2
                local k = ring == 1 and o.i or (o.i - half)
                local cnt = ring == 1 and half or (count - half)
                local dir = ring == 1 and 1 or -1
                local a = dir * tc * 0.8 + 2 * math.pi * k / cnt
                local y = ring == 1 and (Hh * 0.35) or (Hh * 1.45)
                y += math.sin(tc * 2 + k) * 0.12
                local pos = center + Vector3.new(math.cos(a) * R, y, math.sin(a) * R)
                local tangent = Vector3.new(-math.sin(a), 0, math.cos(a)) * dir
                o.part.CFrame = CFrame.lookAt(pos, pos + tangent) * CFrame.Angles(0, 0, (k % 2) * math.pi / 2)
            end
        elseif style == "Руны" then
            for _, o in ipairs(orbit) do
                local k = o.i % 3
                local tilt = k * math.pi / 3
                local a = tc * (1 + k * 0.25) + 2 * math.pi * o.i / count * 1.5
                local offset = CFrame.Angles(tilt, tc * 0.25, 0) * Vector3.new(math.cos(a) * R, 0, math.sin(a) * R)
                o.part.CFrame = CFrame.new(center + Vector3.new(0, Hh, 0) + offset)
            end
        elseif style == "Нимб" then
            for _, o in ipairs(orbit) do
                local a = tc * 1.2 + 2 * math.pi * o.i / count
                local r = 1.25
                local pos = center + Vector3.new(math.cos(a) * r, 3.2 + math.sin(tc * 2) * 0.1, math.sin(a) * r)
                local tangent = Vector3.new(-math.sin(a), 0, math.cos(a))
                o.part.CFrame = CFrame.lookAt(pos, pos + tangent)
            end
        elseif style == "Смерч" then
            for _, o in ipairs(orbit) do
                local f = o.i / count
                local a = tc * 2.2 + f * math.pi * 7
                local r = 0.6 + f * R * 0.65
                local y = -2.8 + f * (Hh * 2 + 2.5)
                o.part.Position = center + Vector3.new(math.cos(a) * r, y, math.sin(a) * r)
                o.part.Size = Vector3.one * (0.18 + f * 0.17)
            end
        end
    end
end

local function setTargetEnabled(value)
    TargetFX.Enabled = value
    targetDirty = true
    if value then
        notify("Таргет FX включён")
    else
        currentTarget = nil
        builtFor, builtChar = nil, nil
        clearTarget()
        notify("Таргет FX выключен")
    end
end

----------------------------------------------------------------
-- KILL FX
----------------------------------------------------------------

local KillFX = {
    Enabled = true, Style = "Кристаллы", Color = C(160, 105, 255),
    Duration = 1.3, Range = 150, Text = true, Flash = true, RemoveDetect = true,
}
local ICE = C(190, 235, 255)

local function fxPart(shape, size, color, cf, material)
    local p = Instance.new("Part")
    p.Name = "KV_KillPart"
    p.Shape = shape
    p.Size = size
    p.Anchored = true
    p.CanCollide = false
    p.CanTouch = false
    p.CanQuery = false
    p.CastShadow = false
    p.Material = material or Enum.Material.Neon
    p.Color = color
    p.CFrame = cf
    p.Parent = killFolder
    return p
end

local function fade(p, dur, props)
    local tw = TweenService:Create(p, TweenInfo.new(dur, Enum.EasingStyle.Quint, Enum.EasingDirection.Out), props)
    tw:Play()
    Debris:AddItem(p, dur + 0.15)
end

local function randDir(minY)
    local v = Vector3.new(math.random() - 0.5, math.random() * 0.8 + (minY or 0.2), math.random() - 0.5)
    if v.Magnitude < 0.05 then v = Vector3.new(0, 1, 0) end
    return v.Unit
end

local function centralFlash(pos, color, dur)
    local p = fxPart(Enum.PartType.Ball, Vector3.one * 1.5, color:Lerp(WHITE, 0.4), CFrame.new(pos))
    p.Transparency = 0.15
    local light = Instance.new("PointLight")
    light.Color = color
    light.Brightness = 6
    light.Range = 22
    light.Parent = p
    TweenService:Create(light, TweenInfo.new(math.min(dur, 0.6)), {Brightness = 0}):Play()
    fade(p, math.min(dur, 0.45), {Size = Vector3.one * 8, Transparency = 1})
end

local KILL_STYLES = {}

KILL_STYLES["Кристаллы"] = function(pos, color, dur)
    for i = 1, 24 do
        local p = fxPart(Enum.PartType.Block, Vector3.new(0.22, 0.65, 0.22), color:Lerp(WHITE, (i % 3) * 0.25),
            CFrame.new(pos) * CFrame.Angles(math.random() * 6, math.random() * 6, math.random() * 6))
        local dest = pos + randDir(0.15) * math.random(4, 9)
        fade(p, dur, {
            CFrame = CFrame.new(dest) * CFrame.Angles(math.random() * 8, math.random() * 8, math.random() * 8),
            Transparency = 1, Size = Vector3.new(0.05, 0.15, 0.05),
        })
    end
    for i = 1, 22 do
        local a = 2 * math.pi * i / 22
        local p = fxPart(Enum.PartType.Ball, Vector3.one * 0.14, color, CFrame.new(pos + Vector3.new(math.cos(a) * 0.5, -2.7, math.sin(a) * 0.5)))
        fade(p, math.min(dur, 0.7), {Position = pos + Vector3.new(math.cos(a) * 5, -2.7, math.sin(a) * 5), Transparency = 1})
    end
end

KILL_STYLES["Мороз"] = function(pos, color, dur)
    for _ = 1, 28 do
        local p = fxPart(Enum.PartType.Ball, Vector3.one * randomRange(0.15, 0.3), ICE:Lerp(WHITE, math.random() * 0.5), CFrame.new(pos))
        fade(p, dur, {Position = pos + randDir(0.1) * math.random(3, 8), Transparency = 1, Size = Vector3.one * 0.03})
    end
    for i = 1, 9 do
        local a = 2 * math.pi * i / 9 + math.random() * 0.3
        local base = pos + Vector3.new(math.cos(a) * 2.4, -2.8, math.sin(a) * 2.4)
        local h = randomRange(3, 5.5)
        local spike = fxPart(Enum.PartType.Block, Vector3.new(0.4, 0.2, 0.4), ICE, CFrame.new(base) * CFrame.Angles(0, a, 0), Enum.Material.Ice)
        spike.Transparency = 0.1
        fade(spike, dur, {
            Size = Vector3.new(0.4, h, 0.4),
            CFrame = CFrame.new(base + Vector3.new(0, h / 2, 0)) * CFrame.Angles(0, a, 0),
            Transparency = 1,
        })
    end
end

KILL_STYLES["Души"] = function(pos, color, dur)
    for i = 1, 14 do
        local a = 2 * math.pi * i / 14
        local start = pos + Vector3.new(math.cos(a) * 1.2, -1.5, math.sin(a) * 1.2)
        local p = fxPart(Enum.PartType.Ball, Vector3.one * randomRange(0.25, 0.45), color:Lerp(WHITE, math.random() * 0.4), CFrame.new(start))
        p.Transparency = 0.1
        fade(p, dur * 1.4, {
            Position = start + Vector3.new(math.cos(a + 1) * 2, randomRange(9, 15), math.sin(a + 1) * 2),
            Transparency = 1, Size = Vector3.one * 0.05,
        })
    end
    local pillar = fxPart(Enum.PartType.Cylinder, Vector3.new(16, 1.4, 1.4), color,
        CFrame.new(pos + Vector3.new(0, 5, 0)) * CFrame.Angles(0, 0, math.pi / 2))
    pillar.Transparency = 0.35
    fade(pillar, dur, {Size = Vector3.new(16, 0.1, 0.1), Transparency = 1})
end

KILL_STYLES["Ударная волна"] = function(pos, color, dur)
    for k = 0, 2 do
        task.delay(k * 0.12, function()
            if not alive then return end
            local p = fxPart(Enum.PartType.Cylinder, Vector3.new(0.25, 1, 1), color:Lerp(WHITE, k * 0.2),
                CFrame.new(pos + Vector3.new(0, -2.8, 0)) * CFrame.Angles(0, 0, math.pi / 2))
            p.Transparency = 0.2
            fade(p, dur * 0.8, {Size = Vector3.new(0.25, 24, 24), Transparency = 1})
        end)
    end
    local pillar = fxPart(Enum.PartType.Cylinder, Vector3.new(10, 1.6, 1.6), color,
        CFrame.new(pos + Vector3.new(0, 2, 0)) * CFrame.Angles(0, 0, math.pi / 2))
    pillar.Transparency = 0.3
    fade(pillar, dur * 0.7, {Size = Vector3.new(10, 0.1, 0.1), Transparency = 1})
    for _ = 1, 16 do
        local p = fxPart(Enum.PartType.Ball, Vector3.one * 0.2, color, CFrame.new(pos))
        fade(p, dur, {Position = pos + randDir(0.3) * math.random(4, 8), Transparency = 1})
    end
end

local function killText(pos, name, color, dur)
    local anchor = fxPart(Enum.PartType.Block, Vector3.one * 0.2, color, CFrame.new(pos + Vector3.new(0, 3, 0)))
    anchor.Transparency = 1
    local bb = make("BillboardGui", {
        Adornee = anchor, Size = UDim2.fromOffset(240, 44), AlwaysOnTop = true, LightInfluence = 0, Parent = anchor,
    })
    local lbl = make("TextLabel", {
        Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Text = "ELIMINATED  •  " .. name,
        Font = Enum.Font.GothamBlack, TextSize = 17, TextColor3 = color:Lerp(WHITE, 0.3),
        TextStrokeTransparency = 0.35, TextStrokeColor3 = Color3.new(0, 0, 0), Parent = bb,
    })
    TweenService:Create(lbl, TweenInfo.new(dur + 0.5), {TextTransparency = 1, TextStrokeTransparency = 1}):Play()
    fade(anchor, dur + 0.5, {Position = anchor.Position + Vector3.new(0, 3.5, 0)})
end

local function killEffect(pos, name)
    local style = KillFX.Style
    local color = style == "Мороз" and ICE or KillFX.Color
    local dur = KillFX.Duration
    centralFlash(pos, color, dur)
    local fn = KILL_STYLES[style] or KILL_STYLES["Кристаллы"]
    pcall(fn, pos, color, dur)
    if KillFX.Text then pcall(killText, pos, name or "", color, dur) end
    local myRoot = getRoot(player)
    if KillFX.Flash and myRoot and (myRoot.Position - pos).Magnitude < 60 then
        killFlash.BackgroundColor3 = color
        killFlash.BackgroundTransparency = 0.8
        tween(killFlash, 0.5, {BackgroundTransparency = 1})
    end
    notify("KILL  •  " .. (name or ""))
end

local tracked = {}
local lastKill = {}

local function fireKill(other, pos)
    if not alive or not KillFX.Enabled or not pos then return end
    local now = os.clock()
    if lastKill[other] and now - lastKill[other] < 1.5 then return end
    lastKill[other] = now
    local myRoot = getRoot(player)
    if myRoot and (myRoot.Position - pos).Magnitude > KillFX.Range then return end
    killEffect(pos, other.DisplayName)
end

local function dropTracked(other)
    local info = tracked[other]
    if info then
        for _, c in ipairs(info.conns) do pcall(function() c:Disconnect() end) end
        tracked[other] = nil
    end
end

local function watchCharacter(other, character)
    if other == player or not character then return end
    local hum = character:FindFirstChildOfClass("Humanoid") or character:WaitForChild("Humanoid", 5)
    local root = character:FindFirstChild("HumanoidRootPart") or character:WaitForChild("HumanoidRootPart", 5)
    if not alive or not hum or not root then return end

    dropTracked(other)
    local info = {conns = {}, root = root, pos = root.Position, hp = hum.Health}
    tracked[other] = info

    local function died() fireKill(other, info.pos) end
    table.insert(info.conns, hum.Died:Connect(died))
    table.insert(info.conns, hum.HealthChanged:Connect(function(h)
        info.hp = h
        if h <= 0 then died() end
    end))
    table.insert(info.conns, character.AncestryChanged:Connect(function(_, parent)
        if parent == nil and KillFX.RemoveDetect
        and (info.hp <= 0 or info.hp <= hum.MaxHealth * 0.4) then
            died()
        end
    end))
end

local function hookPlayer(other)
    if other == player then return end
    if other.Character then task.spawn(watchCharacter, other, other.Character) end
    track(other.CharacterAdded:Connect(function(character)
        task.spawn(watchCharacter, other, character)
    end))
end

----------------------------------------------------------------
-- CROSSHAIR
----------------------------------------------------------------

local XH = {Enabled = false, Size = 9, Gap = 5, Thick = 2, Color = C(70, 215, 255), Spin = false, Dot = true}
local xhFrame = make("Frame", {
    AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
    Size = UDim2.fromOffset(100, 100), BackgroundTransparency = 1, Visible = false, Parent = fxGui,
})
local xhBars = {}
for i = 1, 4 do
    xhBars[i] = make("Frame", {AnchorPoint = Vector2.new(0.5, 0.5), BorderSizePixel = 0, Parent = xhFrame})
end
local xhDot = make("Frame", {
    AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5), BorderSizePixel = 0, Parent = xhFrame,
})
round(xhDot, 4)

local function layoutCrosshair()
    local g, s, t = XH.Gap, XH.Size, XH.Thick
    xhFrame.Visible = XH.Enabled
    xhBars[1].Size = UDim2.fromOffset(t, s)
    xhBars[1].Position = UDim2.new(0.5, 0, 0.5, -(g + s / 2))
    xhBars[2].Size = UDim2.fromOffset(t, s)
    xhBars[2].Position = UDim2.new(0.5, 0, 0.5, g + s / 2)
    xhBars[3].Size = UDim2.fromOffset(s, t)
    xhBars[3].Position = UDim2.new(0.5, -(g + s / 2), 0.5, 0)
    xhBars[4].Size = UDim2.fromOffset(s, t)
    xhBars[4].Position = UDim2.new(0.5, g + s / 2, 0.5, 0)
    for _, b in ipairs(xhBars) do b.BackgroundColor3 = XH.Color end
    xhDot.Visible = XH.Dot
    xhDot.Size = UDim2.fromOffset(t + 1, t + 1)
    xhDot.BackgroundColor3 = XH.Color
end

----------------------------------------------------------------
-- SELF: trail / aura
----------------------------------------------------------------

local Self = {Trail = false, Aura = false, Color = C(160, 105, 255)}
local selfObjs = {}
local function clearSelf()
    for _, o in ipairs(selfObjs) do pcall(function() o:Destroy() end) end
    table.clear(selfObjs)
end
local function applySelf()
    clearSelf()
    local char = player.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if not root then return end
    if Self.Trail then
        local a0 = Instance.new("Attachment")
        a0.Position = Vector3.new(0, 0.9, 0)
        a0.Parent = root
        local a1 = Instance.new("Attachment")
        a1.Position = Vector3.new(0, -1.8, 0)
        a1.Parent = root
        local trail = make("Trail", {
            Attachment0 = a0, Attachment1 = a1, Lifetime = 0.55, LightEmission = 1,
            Color = ColorSequence.new(Self.Color, Self.Color:Lerp(WHITE, 0.6)),
            Transparency = NumberSequence.new(0.1, 1), WidthScale = NumberSequence.new(1, 0), Parent = root,
        })
        table.insert(selfObjs, a0)
        table.insert(selfObjs, a1)
        table.insert(selfObjs, trail)
    end
    if Self.Aura then
        local hl = make("Highlight", {
            Adornee = char, FillColor = Self.Color, FillTransparency = 0.82,
            OutlineColor = Self.Color, OutlineTransparency = 0.15,
            DepthMode = Enum.HighlightDepthMode.Occluded, Parent = selfFolder,
        })
        table.insert(selfObjs, hl)
    end
end

local FOVS = {Enabled = false, Value = 80, Original = nil}
local function setFOV(on)
    FOVS.Enabled = on
    local camera = workspace.CurrentCamera
    if not camera then return end
    if on then
        FOVS.Original = FOVS.Original or camera.FieldOfView
    elseif FOVS.Original then
        camera.FieldOfView = FOVS.Original
        FOVS.Original = nil
    end
end

----------------------------------------------------------------
-- AVATAR: крылья / нимб / кольца / волны от шагов
----------------------------------------------------------------

local Av = {Wings = false, Halo = false, Rings = false, Ripples = false, Color = C(160, 105, 255), Flap = 1}
local avObjs, avWings, avHalo, avRings = {}, {}, {}, {}
local clock = 0

local function avPart(shape, size, color)
    local p = Instance.new("Part")
    p.Name = "KV_Av"
    p.Shape = shape
    p.Size = size
    p.Color = color
    p.Material = Enum.Material.Neon
    p.Anchored = true
    p.CanCollide = false
    p.CanTouch = false
    p.CanQuery = false
    p.CastShadow = false
    p.Transparency = 0.1
    p.Parent = selfFolder
    table.insert(avObjs, p)
    return p
end

local function clearAvatar()
    for _, o in ipairs(avObjs) do pcall(function() o:Destroy() end) end
    table.clear(avObjs)
    table.clear(avWings)
    table.clear(avHalo)
    table.clear(avRings)
end

local function buildAvatar()
    clearAvatar()
    local char = player.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if not root then return end
    if Av.Wings then
        for side = -1, 1, 2 do
            for f = 1, 7 do
                local p = avPart(Enum.PartType.Block, Vector3.new(0.18, 0.1, 3), Av.Color:Lerp(WHITE, (f - 1) / 9))
                table.insert(avWings, {part = p, side = side, f = f})
            end
        end
    end
    if Av.Halo then
        for i = 1, 18 do
            table.insert(avHalo, {part = avPart(Enum.PartType.Block, Vector3.new(0.12, 0.12, 0.45), Av.Color:Lerp(WHITE, 0.4)), i = i})
        end
    end
    if Av.Rings then
        for r = 1, 2 do
            for i = 1, 26 do
                table.insert(avRings, {part = avPart(Enum.PartType.Ball, Vector3.one * 0.28, Av.Color:Lerp(WHITE, (i % 2) * 0.4)), r = r, i = i})
            end
        end
    end
end

local function updateAvatar()
    if #avWings + #avHalo + #avRings == 0 then return end
    local char = player.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if not root then return end
    local hum = char:FindFirstChildOfClass("Humanoid")
    local cf = root.CFrame
    local t = clock
    local moving = hum and hum.MoveDirection.Magnitude > 0.1
    local flap = math.sin(t * (moving and 5 or 2.2) * Av.Flap) * 0.22

    for _, w in ipairs(avWings) do
        local f = w.f
        local a = 0.25 + (f - 1) * 0.24 + flap * (0.5 + f * 0.12)
        local d = Vector3.new(w.side * math.sin(a), math.cos(a), 0.35).Unit
        local L = 4.4 - (f - 1) * 0.2
        local rootOffset = Vector3.new(w.side * 0.5, 0.7, 0.7)
        local center = cf:PointToWorldSpace(rootOffset + d * (L / 2))
        local dir = cf:VectorToWorldSpace(d)
        w.part.Size = Vector3.new(0.18, 0.1, L)
        w.part.CFrame = CFrame.lookAt(center, center + dir)
    end

    local n = #avHalo
    for _, h in ipairs(avHalo) do
        local a = t * 1.3 + 2 * math.pi * h.i / n
        local pos = cf.Position + Vector3.new(math.cos(a) * 1.1, 3.3 + math.sin(t * 2) * 0.08, math.sin(a) * 1.1)
        local tangent = Vector3.new(-math.sin(a), 0, math.cos(a))
        h.part.CFrame = CFrame.lookAt(pos, pos + tangent)
    end

    for _, g in ipairs(avRings) do
        local dirSign = g.r == 1 and 1 or -1
        local rot = CFrame.Angles(g.r == 1 and 0.9 or -0.9, t * 0.6 * dirSign, g.r == 1 and 0 or 0.6)
        local a = 2 * math.pi * g.i / 26
        g.part.Position = cf.Position + Vector3.new(0, 0.3, 0) + rot * Vector3.new(math.cos(a) * 3.2, 0, math.sin(a) * 3.2)
    end
end

local rippleTimer = 0
local function stepRipples(dt)
    if not Av.Ripples then return end
    rippleTimer += dt
    if rippleTimer < 0.3 then return end
    local char = player.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    if not root or not hum or hum.MoveDirection.Magnitude < 0.1 then return end
    rippleTimer = 0
    local drop = hum.HipHeight > 0 and (hum.HipHeight + root.Size.Y / 2) or 3
    local p = Instance.new("Part")
    p.Name = "KV_Ripple"
    p.Shape = Enum.PartType.Cylinder
    p.Size = Vector3.new(0.12, 1.5, 1.5)
    p.Color = Av.Color
    p.Material = Enum.Material.Neon
    p.Anchored = true
    p.CanCollide = false
    p.CanTouch = false
    p.CanQuery = false
    p.CastShadow = false
    p.Transparency = 0.2
    p.CFrame = CFrame.new(root.Position.X, root.Position.Y - drop + 0.1, root.Position.Z) * CFrame.Angles(0, 0, math.pi / 2)
    p.Parent = selfFolder
    TweenService:Create(p, TweenInfo.new(0.8, Enum.EasingStyle.Quint, Enum.EasingDirection.Out),
        {Size = Vector3.new(0.12, 10, 10), Transparency = 1}):Play()
    Debris:AddItem(p, 0.9)
end

----------------------------------------------------------------
-- SKY OBJECT (планета / чёрная дыра / луна)
----------------------------------------------------------------

local Sky = {Enabled = false, Style = "Планета", Scale = 1, Color = C(160, 105, 255), Dir = Vector3.new(0.45, 0.38, -0.8).Unit}
local SKY_DIST = 520
local SKY_TILT = CFrame.Angles(0.5, 0.3, 0.35)
local skyModel, skyCore, skyAnim

local function clearSky()
    if skyModel then pcall(function() skyModel:Destroy() end) end
    skyModel, skyCore, skyAnim = nil, nil, nil
end

local function skyPart(shape, size, color, material, parent)
    local p = Instance.new("Part")
    p.Shape = shape
    p.Size = size
    p.Color = color
    p.Material = material or Enum.Material.Neon
    p.Anchored = true
    p.CanCollide = false
    p.CanTouch = false
    p.CanQuery = false
    p.CastShadow = false
    p.Parent = parent
    return p
end

local function dustRing(model, radius, count, size, colorFn, yJitter)
    for i = 1, count do
        local a = 2 * math.pi * i / count + math.random() * 0.05
        local r = radius + randomRange(-3, 3)
        local p = skyPart(Enum.PartType.Ball, Vector3.one * size * randomRange(0.7, 1.3), colorFn(i), Enum.Material.Neon, model)
        p.Position = Vector3.new(math.cos(a) * r, randomRange(-yJitter, yJitter), math.sin(a) * r)
        p.Transparency = randomRange(0, 0.25)
    end
end

local function buildSky()
    clearSky()
    if not Sky.Enabled then return end
    local sc = Sky.Scale
    local col = Sky.Color
    local m = Instance.new("Model")
    m.Name = "KV_SkyModel"
    m.Parent = skyFolder
    local style = Sky.Style

    if style == "Планета" then
        local core = skyPart(Enum.PartType.Ball, Vector3.one * 150 * sc, col, Enum.Material.Neon, m)
        core.Position = Vector3.zero
        local glow = skyPart(Enum.PartType.Ball, Vector3.one * 168 * sc, col:Lerp(WHITE, 0.3), Enum.Material.Neon, m)
        glow.Transparency = 0.85
        glow.Position = Vector3.zero
        dustRing(m, 110 * sc, 60, 2.2 * sc, function(i) return col:Lerp(WHITE, (i % 3) * 0.25) end, 1.2)
        dustRing(m, 134 * sc, 80, 2.0 * sc, function(i) return col:Lerp(WHITE, 0.5 + (i % 2) * 0.2) end, 1.2)
        dustRing(m, 162 * sc, 100, 1.8 * sc, function(i) return col:Lerp(C(255, 200, 150), (i % 4) * 0.2) end, 1.5)
        local moons = {}
        for k = 1, 2 do
            moons[k] = skyPart(Enum.PartType.Ball, Vector3.one * (14 + k * 6) * sc, WHITE:Lerp(col, 0.2 * k), Enum.Material.Neon, m)
        end
        m.PrimaryPart = core
        skyCore = core
        skyAnim = function(pivot)
            for k, moon in ipairs(moons) do
                local a = clock * (0.35 / k) + k * 2
                moon.CFrame = pivot * CFrame.new(math.cos(a) * (210 + k * 30) * sc, math.sin(a * 1.3) * 25 * sc, math.sin(a) * (210 + k * 30) * sc)
            end
        end

    elseif style == "Чёрная дыра" then
        local core = skyPart(Enum.PartType.Ball, Vector3.one * 70 * sc, Color3.new(0, 0, 0), Enum.Material.Neon, m)
        core.Position = Vector3.zero
        local shell = skyPart(Enum.PartType.Ball, Vector3.one * 78 * sc, C(255, 170, 90):Lerp(col, 0.4), Enum.Material.Neon, m)
        shell.Transparency = 0.72
        shell.Position = Vector3.zero
        local radii = {52, 70, 92, 118}
        local counts = {50, 70, 90, 110}
        for k, r in ipairs(radii) do
            dustRing(m, r * sc, counts[k], (3.2 - k * 0.3) * sc, function(i)
                local f = (k - 1) / 3
                local hot = C(255, 245, 220):Lerp(C(255, 140, 50), f)
                return hot:Lerp(col, f * 0.55 + (i % 3) * 0.05)
            end, 1.5 + k)
        end
        m.PrimaryPart = core
        skyCore = core

    else -- Гигантская луна
        local core = skyPart(Enum.PartType.Ball, Vector3.one * 170 * sc, C(235, 235, 225), Enum.Material.Neon, m)
        core.Position = Vector3.zero
        local glow = skyPart(Enum.PartType.Ball, Vector3.one * 190 * sc, col:Lerp(WHITE, 0.6), Enum.Material.Neon, m)
        glow.Transparency = 0.88
        glow.Position = Vector3.zero
        local toCam = -Sky.Dir
        for _ = 1, 16 do
            local v
            repeat
                v = Vector3.new(math.random() * 2 - 1, math.random() * 2 - 1, math.random() * 2 - 1)
            until v.Magnitude > 0.2 and v.Magnitude <= 1 and v.Unit:Dot(toCam) > 0.25
            local crater = skyPart(Enum.PartType.Ball, Vector3.one * randomRange(10, 28) * sc, C(185, 185, 175), Enum.Material.Neon, m)
            crater.Position = v.Unit * 85 * sc
        end
        m.PrimaryPart = core
        skyCore = core
    end
    skyModel = m
end

local skyAcc, skyFrame = 0, 0
local function updateSky(dt)
    skyAcc += dt
    skyFrame += 1
    if skyFrame % 3 ~= 0 then return end
    local acc = skyAcc
    skyAcc = 0
    if not skyModel or not skyCore or not skyModel.Parent then return end
    local cam = workspace.CurrentCamera
    if not cam then return end
    local p = cam.CFrame.Position + Sky.Dir * SKY_DIST
    local pivot
    if Sky.Style == "Гигантская луна" then
        pivot = CFrame.new(p)
    else
        local spin = clock * (Sky.Style == "Чёрная дыра" and 0.3 or 0.07)
        pivot = CFrame.new(p) * SKY_TILT * CFrame.Angles(0, spin, 0)
    end
    skyModel:PivotTo(pivot)
    if skyAnim then skyAnim(pivot) end
end

----------------------------------------------------------------
-- 3D AURORA (ленты в небе)
----------------------------------------------------------------

local Aur3 = {Enabled = false, Bands = 4, Height = 95}
local aurObjs, aurBands = {}, {}

local function clearAurora3D()
    for _, o in ipairs(aurObjs) do pcall(function() o:Destroy() end) end
    table.clear(aurObjs)
    table.clear(aurBands)
end

local function buildAurora3D()
    clearAurora3D()
    if not Aur3.Enabled then return end
    for b = 1, Aur3.Bands do
        local nodes, beams = {}, {}
        for k = 1, 9 do
            local part = Instance.new("Part")
            part.Name = "KV_AurNode"
            part.Size = Vector3.one
            part.Transparency = 1
            part.Anchored = true
            part.CanCollide = false
            part.CanTouch = false
            part.CanQuery = false
            part.CastShadow = false
            part.Parent = skyFolder
            local att = Instance.new("Attachment")
            att.Parent = part
            table.insert(aurObjs, part)
            nodes[k] = {part = part, att = att}
        end
        for k = 1, 8 do
            local edge = 1 - math.abs((k - 4.5) / 3.5)
            local tr = 0.92 - 0.45 * math.clamp(edge, 0, 1)
            local beam = make("Beam", {
                Attachment0 = nodes[k].att, Attachment1 = nodes[k + 1].att,
                Width0 = 46, Width1 = 46, LightEmission = 1, LightInfluence = 0,
                FaceCamera = true, Segments = 1,
                Transparency = NumberSequence.new(tr), Parent = skyFolder,
            })
            table.insert(aurObjs, beam)
            beams[k] = beam
        end
        aurBands[b] = {nodes = nodes, beams = beams}
    end
end

local function updateAurora3D()
    if #aurBands == 0 then return end
    local cam = workspace.CurrentCamera
    if not cam then return end
    local pos = cam.CFrame.Position
    local count = #aurBands
    for b, band in ipairs(aurBands) do
        local ang = (b - 1) * (2 * math.pi / count) + clock * 0.01
        local radial = Vector3.new(math.cos(ang), 0, math.sin(ang))
        local tangent = Vector3.new(-math.sin(ang), 0, math.cos(ang))
        local center = pos + radial * 260
        for k, node in ipairs(band.nodes) do
            local u = (k - 5) / 4
            node.part.Position = center + tangent * u * 170
                + radial * (math.sin(clock * 0.4 + u * 2 + b) * 20)
                + Vector3.new(0, Aur3.Height + b * 14 + math.sin(clock * 0.7 + u * 3 + b * 2) * 16, 0)
        end
        for k, beam in ipairs(band.beams) do
            local h = 0.38 + 0.2 * math.sin(clock * 0.2 + b + k * 0.3)
            beam.Color = ColorSequence.new(Color3.fromHSV(h, 0.7, 1))
        end
    end
end

----------------------------------------------------------------
-- METEORS
----------------------------------------------------------------

local Met = {Enabled = false, Rate = 1.5, Color = C(255, 190, 120)}
local meteors = {}
local metAcc = 0

local function spawnMeteor()
    local cam = workspace.CurrentCamera
    if not cam then return end
    local cpos = cam.CFrame.Position
    local ang = math.random() * math.pi * 2
    local r = randomRange(150, 330)
    local start = cpos + Vector3.new(math.cos(ang) * r, randomRange(130, 210), math.sin(ang) * r)
    local tang = Vector3.new(-math.sin(ang), 0, math.cos(ang)) * (math.random() < 0.5 and 1 or -1) * randomRange(0.5, 1)
    local dir = (tang + Vector3.new(0, -0.55, 0)).Unit
    local part = Instance.new("Part")
    part.Name = "KV_Meteor"
    part.Shape = Enum.PartType.Ball
    part.Size = Vector3.one * 1.8
    part.Color = WHITE
    part.Material = Enum.Material.Neon
    part.Anchored = true
    part.CanCollide = false
    part.CanTouch = false
    part.CanQuery = false
    part.CastShadow = false
    part.Position = start
    part.Parent = skyFolder
    local a0 = Instance.new("Attachment")
    a0.Position = Vector3.new(0, 1.4, 0)
    a0.Parent = part
    local a1 = Instance.new("Attachment")
    a1.Position = Vector3.new(0, -1.4, 0)
    a1.Parent = part
    make("Trail", {
        Attachment0 = a0, Attachment1 = a1, Lifetime = 1.2, LightEmission = 1,
        Color = ColorSequence.new(WHITE, Met.Color),
        Transparency = NumberSequence.new(0, 1), WidthScale = NumberSequence.new(1, 0.1), Parent = part,
    })
    table.insert(meteors, {part = part, pos = start, vel = dir * randomRange(200, 320), life = randomRange(1.4, 2.2)})
end

local function updateMeteors(dt)
    if Met.Enabled then
        metAcc += dt * Met.Rate
        while metAcc >= 1 do
            metAcc -= 1
            spawnMeteor()
        end
    end
    for i = #meteors, 1, -1 do
        local m = meteors[i]
        m.pos += m.vel * dt
        m.life -= dt
        if m.part.Parent then m.part.Position = m.pos end
        if m.life <= 0 then
            m.part.Transparency = 1
            Debris:AddItem(m.part, 1.3)
            table.remove(meteors, i)
        end
    end
end

----------------------------------------------------------------
-- LIGHTNING
----------------------------------------------------------------

local Lightning = {Enabled = false, Interval = 5, Color = C(190, 215, 255)}
local lightTimer, nextStrike = 0, 3

local function boltSegments(a, b, segs, width)
    local prev = a
    for i = 1, segs do
        local f = i / segs
        local nxt = a:Lerp(b, f)
        if i < segs then
            nxt += Vector3.new(randomRange(-6, 6), randomRange(-3, 3), randomRange(-6, 6)) * (1 - f * 0.3)
        end
        local len = (nxt - prev).Magnitude
        if len > 0.05 then
            local p = Instance.new("Part")
            p.Name = "KV_Bolt"
            p.Anchored = true
            p.CanCollide = false
            p.CanTouch = false
            p.CanQuery = false
            p.CastShadow = false
            p.Material = Enum.Material.Neon
            p.Color = Lightning.Color:Lerp(WHITE, 0.5)
            p.Size = Vector3.new(width, width, len)
            p.CFrame = CFrame.lookAt((prev + nxt) / 2, nxt)
            p.Parent = skyFolder
            TweenService:Create(p, TweenInfo.new(0.45, Enum.EasingStyle.Quint, Enum.EasingDirection.Out),
                {Transparency = 1, Size = Vector3.new(width * 0.2, width * 0.2, len)}):Play()
            Debris:AddItem(p, 0.55)
        end
        prev = nxt
    end
end

local function strikeLightning()
    local cam = workspace.CurrentCamera
    if not cam then return end
    local cpos = cam.CFrame.Position
    local ang = math.random() * math.pi * 2
    local dist = randomRange(70, 200)
    local gx, gz = cpos.X + math.cos(ang) * dist, cpos.Z + math.sin(ang) * dist
    local rr = workspace:Raycast(Vector3.new(gx, cpos.Y + 80, gz), Vector3.new(0, -300, 0), snowRay)
    local gy = rr and rr.Position.Y or (cpos.Y - 4)
    local top = Vector3.new(gx + randomRange(-15, 15), gy + randomRange(130, 170), gz + randomRange(-15, 15))
    local ground = Vector3.new(gx, gy, gz)
    boltSegments(top, ground, 14, 0.7)
    for _ = 1, 2 do
        local s = top:Lerp(ground, randomRange(0.3, 0.7))
        local e = s + Vector3.new(randomRange(-40, 40), -randomRange(20, 50), randomRange(-40, 40))
        boltSegments(s, e, 6, 0.35)
    end
    local anchor = Instance.new("Part")
    anchor.Anchored = true
    anchor.CanCollide = false
    anchor.CanQuery = false
    anchor.Transparency = 1
    anchor.Size = Vector3.one
    anchor.Position = ground + Vector3.new(0, 4, 0)
    anchor.Parent = skyFolder
    local light = Instance.new("PointLight")
    light.Color = Lightning.Color
    light.Brightness = 8
    light.Range = 70
    light.Parent = anchor
    TweenService:Create(light, TweenInfo.new(0.5), {Brightness = 0}):Play()
    Debris:AddItem(anchor, 0.6)

    lightFlash.BackgroundTransparency = 0.75
    tween(lightFlash, 0.4, {BackgroundTransparency = 1})
end

local function stepLightning(dt)
    if not Lightning.Enabled then return end
    lightTimer += dt
    if lightTimer >= nextStrike then
        lightTimer = 0
        nextStrike = Lightning.Interval * randomRange(0.5, 1.5)
        pcall(strikeLightning)
        if math.random() < 0.3 then
            task.delay(0.18, function()
                if alive and Lightning.Enabled then pcall(strikeLightning) end
            end)
        end
    end
end

----------------------------------------------------------------
-- BUILD PAGES
----------------------------------------------------------------

-- МИР
local pgWorld = newPage("world", "Мир")
local c = card(pgWorld, "Атмосфера")
H.World = toggle(c, "Своя атмосфера", true, function(v)
    S.World = v
    if v then applyWorld() else restoreWorld() end
end)
chips(c, PRESET_NAMES, "Закат", function(name)
    for key, value in pairs(PRESETS[name]) do S[key] = value end
    applyWorld()
    notify("Пресет: " .. name)
    if name == "Мороз" then
        if H.Snow then H.Snow.Set(true, true) end
        if H.PartChips then
            P.Enabled = true
            if H.Particles then H.Particles.Set(true, false) end
            H.PartChips.Pick("Снег", true)
        end
    end
end)
c = card(pgWorld, "Параметры")
slider(c, "Время суток", 0, 24, S.Time, 0.1, function(v) S.Time = v; applyWorld() end)
slider(c, "Плотность тумана", 0, 1, S.Density, 0.05, function(v) S.Density = v; applyWorld() end)
slider(c, "Дымка", 0, 6, S.Haze, 0.1, function(v) S.Haze = v; applyWorld() end)
colorPicker(c, "Цвет атмосферы", S.Fog, function(col) S.Fog = col; applyWorld() end)
colorPicker(c, "Цвет затухания", S.Decay, function(col) S.Decay = col; applyWorld() end)
colorPicker(c, "Цвет окружения", S.Ambient, function(col) S.Ambient = col; applyWorld() end)
c = card(pgWorld, "Динамика")
toggle(c, "Радужная атмосфера", false, function(v) S.Rainbow = v end)
toggle(c, "Смена дня и ночи", false, function(v) S.Cycle = v end)
slider(c, "Скорость цикла", 0.02, 1, S.CycleSpeed, 0.02, function(v) S.CycleSpeed = v end)
toggle(c, "Размытие дали (DoF)", false, function(v) S.DOF = v; applyWorld() end)

-- ЦВЕТ
local pgColor = newPage("color", "Цвет")
c = card(pgColor, "Цветокоррекция")
toggle(c, "Свечение (Bloom)", S.Bloom, function(v) S.Bloom = v; applyWorld() end)
slider(c, "Сила свечения", 0, 3, S.BloomI, 0.1, function(v) S.BloomI = v; applyWorld() end)
slider(c, "Насыщенность", -1, 1, S.Sat, 0.1, function(v) S.Sat = v; applyWorld() end)
slider(c, "Контраст", -0.5, 1, S.Contrast, 0.1, function(v) S.Contrast = v; applyWorld() end)
slider(c, "Экспозиция", -2, 2, S.Exposure, 0.1, function(v) S.Exposure = v; applyWorld() end)
toggle(c, "Лучи солнца", S.Rays, function(v) S.Rays = v; applyWorld() end)
toggle(c, "Тени мира", S.Shadows, function(v) S.Shadows = v; applyWorld() end)

-- СНЕГ
local pgSnow = newPage("snow", "Снег")
c = card(pgSnow, "Снег на земле")
H.Snow = toggle(c, "Снег на всех поверхностях", false, function(v)
    SnowB.Enabled = v
    if v then
        notify("Снег включён")
    else
        clearSnow()
        notify("Снег убран")
    end
end)
slider(c, "Радиус покрытия", 40, 200, SnowB.Radius, 10, function(v)
    SnowB.Radius = v
    snowCenter = nil
end)
slider(c, "Шаг сетки (меньше = плотнее)", 2, 8, SnowB.Step, 1, function(v)
    SnowB.Step = v
    if SnowB.Enabled then clearSnow() end
end)
slider(c, "Толщина слоя", 0.1, 1, SnowB.Thick, 0.05, function(v)
    SnowB.Thick = v
    for _, cell in pairs(snowCells) do cell.t = 0 end
end)
button(c, "Пересыпать снег заново", function()
    if SnowB.Enabled then
        clearSnow()
        notify("Пересыпаю снег…")
    end
end)

-- ЧАСТИЦЫ
local pgPart = newPage("parts", "Частицы")
c = card(pgPart, "Падающие частицы")
H.Particles = toggle(c, "Включить частицы", P.Enabled, function(v)
    P.Enabled = v
    if v then setParticleCount(P.Count) else clearParticles() end
end)
H.PartChips = chips(c, {"Снег", "Сакура", "Звёзды", "Пепел", "Неон", "Светлячки"}, P.Style, function(name)
    P.Style = name
    if P.Enabled then
        setParticleCount(P.Count)
        restyleParticles()
    end
end)
slider(c, "Количество", 0, 180, P.Count, 10, function(v)
    P.Count = v
    if P.Enabled then setParticleCount(v) end
end)
slider(c, "Скорость падения", 0.2, 3, P.Speed, 0.1, function(v) P.Speed = v end)
slider(c, "Размер", 0.4, 3, P.Size, 0.1, function(v) P.Size = v end)
slider(c, "Радиус появления", 15, 90, P.Radius, 5, function(v) P.Radius = v end)
slider(c, "Ветер", -10, 10, P.Wind, 1, function(v) P.Wind = v end)

-- НЕБО
local pgSky = newPage("sky", "Небо")
c = card(pgSky, "Небесный объект")
H.Sky = toggle(c, "Показать в небе", false, function(v)
    Sky.Enabled = v
    buildSky()
end)
chips(c, {"Планета", "Чёрная дыра", "Гигантская луна"}, Sky.Style, function(name)
    Sky.Style = name
    buildSky()
end)
colorPicker(c, "Цвет объекта", Sky.Color, function(col) Sky.Color = col; buildSky() end)
slider(c, "Размер", 0.5, 2, Sky.Scale, 0.1, function(v) Sky.Scale = v; buildSky() end)
c = card(pgSky, "Сияние 3D")
H.Aur3 = toggle(c, "Ленты сияния в небе", false, function(v)
    Aur3.Enabled = v
    buildAurora3D()
end)
slider(c, "Количество лент", 2, 6, Aur3.Bands, 1, function(v) Aur3.Bands = v; buildAurora3D() end)
slider(c, "Высота", 50, 160, Aur3.Height, 5, function(v) Aur3.Height = v end)
c = card(pgSky, "Метеоры")
H.Met = toggle(c, "Метеоритный дождь", false, function(v) Met.Enabled = v end)
slider(c, "Частота (в секунду)", 0.3, 6, Met.Rate, 0.1, function(v) Met.Rate = v end)
colorPicker(c, "Цвет следа", Met.Color, function(col) Met.Color = col end)
c = card(pgSky, "Молнии")
H.Light = toggle(c, "Грозовые молнии", false, function(v) Lightning.Enabled = v end)
slider(c, "Средний интервал (сек)", 1, 12, Lightning.Interval, 0.5, function(v) Lightning.Interval = v end)
button(c, "Ударить молнией сейчас", function() pcall(strikeLightning) end)

-- ЭКРАН
local pgScreen = newPage("screen", "Экран")
c = card(pgScreen, "Экранные эффекты")
toggle(c, "Северное сияние (2D)", false, function(v) FX.Aurora = v; aurora.Visible = v end)
toggle(c, "Тёмная виньетка", false, function(v) FX.Vignette = v; vignette.Visible = v end)
toggle(c, "Кинематографичные цвета", false, setCinematic)
c = card(pgScreen, "Прицел")
H.Cross = toggle(c, "Свой прицел", false, function(v) XH.Enabled = v; layoutCrosshair() end)
toggle(c, "Точка в центре", true, function(v) XH.Dot = v; layoutCrosshair() end)
toggle(c, "Вращение", false, function(v) XH.Spin = v end)
slider(c, "Длина", 3, 24, XH.Size, 1, function(v) XH.Size = v; layoutCrosshair() end)
slider(c, "Отступ", 0, 20, XH.Gap, 1, function(v) XH.Gap = v; layoutCrosshair() end)
slider(c, "Толщина", 1, 5, XH.Thick, 1, function(v) XH.Thick = v; layoutCrosshair() end)
colorPicker(c, "Цвет прицела", XH.Color, function(col) XH.Color = col; layoutCrosshair() end)

-- ТАРГЕТ
local pgTarget = newPage("target", "Таргет")
c = card(pgTarget, "Ближайший игрок")
H.Target = toggle(c, "Включить Таргет FX", false, setTargetEnabled)
chips(c, {"Кристаллы", "Цепи", "Руны", "Нимб", "Смерч"}, TargetFX.Style, function(style)
    TargetFX.Style = style
    targetDirty = true
    notify("Стиль: " .. style)
end)
colorPicker(c, "Цвет эффекта", TargetFX.Color, function(col) TargetFX.Color = col; targetDirty = true end)
c = card(pgTarget, "Элементы")
toggle(c, "Контур (Highlight)", TargetFX.Highlight, function(v) TargetFX.Highlight = v; targetDirty = true end)
toggle(c, "Кольцо на земле", TargetFX.Ring, function(v) TargetFX.Ring = v; targetDirty = true end)
toggle(c, "Луч к цели", TargetFX.Tracer, function(v) TargetFX.Tracer = v; targetDirty = true end)
c = card(pgTarget, "Настройки")
slider(c, "Количество", 2, 12, TargetFX.Count, 1, function(v) TargetFX.Count = v; targetDirty = true end)
slider(c, "Радиус орбиты", 1, 7, TargetFX.Radius, 0.5, function(v) TargetFX.Radius = v end)
slider(c, "Высота орбиты", 0, 5, TargetFX.Height, 0.5, function(v) TargetFX.Height = v end)
slider(c, "Скорость", 0.2, 4, TargetFX.Speed, 0.1, function(v) TargetFX.Speed = v end)
slider(c, "Дальность цели", 10, 200, TargetFX.Range, 5, function(v) TargetFX.Range = v end)

-- СМЕРТЬ
local pgKill = newPage("kill", "Смерть")
c = card(pgKill, "Kill FX")
toggle(c, "Включить Kill FX", true, function(v) KillFX.Enabled = v end)
chips(c, {"Кристаллы", "Мороз", "Души", "Ударная волна"}, KillFX.Style, function(style) KillFX.Style = style end)
colorPicker(c, "Цвет эффекта", KillFX.Color, function(col) KillFX.Color = col end)
toggle(c, "Надпись ELIMINATED", true, function(v) KillFX.Text = v end)
toggle(c, "Вспышка экрана", true, function(v) KillFX.Flash = v end)
toggle(c, "Ловить удаление персонажа", true, function(v) KillFX.RemoveDetect = v end)
slider(c, "Длительность", 0.3, 3, KillFX.Duration, 0.1, function(v) KillFX.Duration = v end)
slider(c, "Дальность", 20, 400, KillFX.Range, 10, function(v) KillFX.Range = v end)
button(c, "Тест эффекта перед тобой", function()
    local root = getRoot(player)
    if root then killEffect(root.Position + root.CFrame.LookVector * 9, "Тест") end
end)

-- АВАТАР
local pgAv = newPage("avatar", "Аватар")
c = card(pgAv, "Энергия")
H.Wings = toggle(c, "Энергетические крылья", false, function(v) Av.Wings = v; buildAvatar() end)
H.Halo = toggle(c, "Нимб над головой", false, function(v) Av.Halo = v; buildAvatar() end)
H.Rings = toggle(c, "Орбитальные кольца", false, function(v) Av.Rings = v; buildAvatar() end)
toggle(c, "Волны от шагов", false, function(v) Av.Ripples = v end)
slider(c, "Скорость взмаха", 0.3, 3, Av.Flap, 0.1, function(v) Av.Flap = v end)
colorPicker(c, "Цвет эффектов", Av.Color, function(col)
    Av.Color = col
    Self.Color = col
    buildAvatar()
    applySelf()
end)
c = card(pgAv, "Следы")
toggle(c, "Трейл за персонажем", false, function(v) Self.Trail = v; applySelf() end)
toggle(c, "Аура вокруг тебя", false, function(v) Self.Aura = v; applySelf() end)

-- ПРОЧЕЕ
local pgMisc = newPage("misc", "Прочее")
c = card(pgMisc, "Интерфейс")
local THEMES = {
    ["Неон"] = {C(70, 215, 255), C(160, 105, 255)},
    ["Сакура"] = {C(255, 120, 190), C(180, 110, 255)},
    ["Лайм"] = {C(120, 255, 170), C(60, 200, 255)},
    ["Огонь"] = {C(255, 170, 70), C(255, 70, 90)},
}
chips(c, {"Неон", "Сакура", "Лайм", "Огонь"}, "Неон", function(name)
    T.a1, T.a2 = THEMES[name][1], THEMES[name][2]
    applyTheme()
    refreshQuick()
end)
c = card(pgMisc, "Камера")
toggle(c, "Свой FOV", false, setFOV)
slider(c, "Значение FOV", 40, 120, FOVS.Value, 1, function(v) FOVS.Value = v end)
c = card(pgMisc, "Сброс")
button(c, "Сбросить атмосферу на закат", function()
    for key, value in pairs(PRESETS["Закат"]) do S[key] = value end
    S.Bloom, S.BloomI, S.Exposure, S.Rays, S.Shadows = true, 0.5, 0, false, true
    S.Rainbow, S.Cycle, S.DOF = false, false, false
    applyWorld()
    notify("Атмосфера сброшена")
end)
button(c, "Убрать частицы", function()
    clearParticles()
    notify("Частицы убраны")
end)
button(c, "Выгрузить скрипт", function()
    if env.KitVisualsUnload then env.KitVisualsUnload() end
end)

selectTab("world")

----------------------------------------------------------------
-- НЕРЕАЛ-РЕЖИМ
----------------------------------------------------------------

local unreal = false
local function setUnreal(on)
    unreal = on
    if on then
        for key, value in pairs(PRESETS["Космос"]) do S[key] = value end
        S.Bloom, S.BloomI = true, 0.9
        applyWorld()
        H.Sky.Set(true, true)
        H.Aur3.Set(true, true)
        H.Met.Set(true, true)
        H.Wings.Set(true, true)
        H.Halo.Set(true, true)
        if not P.Enabled then H.Particles.Set(true, true) end
        H.PartChips.Pick("Звёзды", true)
        notify("НЕРЕАЛ-режим включён")
    else
        H.Sky.Set(false, true)
        H.Aur3.Set(false, true)
        H.Met.Set(false, true)
        H.Light.Set(false, true)
        H.Wings.Set(false, true)
        H.Halo.Set(false, true)
        H.Rings.Set(false, true)
        notify("НЕРЕАЛ-режим выключен")
    end
end

c = card(pgSky, "Всё сразу")
button(c, "✦ НЕРЕАЛ-режим вкл / выкл", function() setUnreal(not unreal) end)

----------------------------------------------------------------
-- ISLAND QUICK BUTTONS
----------------------------------------------------------------

quick(1, "МЕНЮ", function() toggleWindow() end, function() return opened end)
quick(2, "ТАРГЕТ", function() H.Target.Set(not H.Target.Get(), true) end, function() return TargetFX.Enabled end)
quick(3, "СНЕГ", function() H.Snow.Set(not H.Snow.Get(), true) end, function() return SnowB.Enabled end)
quick(4, "ПРИЦЕЛ", function() H.Cross.Set(not H.Cross.Get(), true) end, function() return XH.Enabled end)
quick(5, "НЕРЕАЛ", function() setUnreal(not unreal) end, function() return unreal end)

closeButton.Activated:Connect(toggleWindow)
track(UserInputService.InputBegan:Connect(function(input, processed)
    if not processed and input.KeyCode == Enum.KeyCode.RightShift then
        toggleWindow()
    end
end))
uiScale.Scale = baseScale()

----------------------------------------------------------------
-- PLAYER HOOKS
----------------------------------------------------------------

for _, other in ipairs(Players:GetPlayers()) do hookPlayer(other) end
track(Players.PlayerAdded:Connect(hookPlayer))
track(Players.PlayerRemoving:Connect(function(other)
    dropTracked(other)
    lastKill[other] = nil
end))
track(player.CharacterAdded:Connect(function()
    task.delay(0.6, function()
        if alive then
            applySelf()
            buildAvatar()
            targetDirty = true
        end
    end)
end))

----------------------------------------------------------------
-- MAIN LOOPS
----------------------------------------------------------------

local reapply, worldTick = 0, 0
local fpsFrames, fpsTime, targetScan = 0, 0, 0
local rainbowHue = 0

track(RunService.RenderStepped:Connect(function(dt)
    if not alive then return end
    clock += dt
    particleClock += dt
    targetClock += dt * TargetFX.Speed
    fpsFrames += 1
    fpsTime += dt

    stepParticles(dt)

    local spin = (clock * 45) % 360
    for _, g in ipairs(spinGrads) do g.Rotation = spin end

    if FX.Aurora then
        aurora.Position = UDim2.new(-0.1 + math.sin(clock * 0.12) * 0.035, 0, math.sin(clock * 0.2) * 0.02, 0)
        auroraGradient.Rotation = math.sin(clock * 0.15) * 12
    end

    if XH.Enabled and XH.Spin then
        xhFrame.Rotation = (clock * 90) % 360
    elseif xhFrame.Rotation ~= 0 then
        xhFrame.Rotation = 0
    end

    if FOVS.Enabled then
        local camera = workspace.CurrentCamera
        if camera then camera.FieldOfView = FOVS.Value end
    end

    if S.World and (S.Rainbow or S.Cycle) then
        if S.Rainbow then
            rainbowHue = (rainbowHue + dt * 0.05) % 1
            S.Fog = Color3.fromHSV(rainbowHue, 0.55, 1)
            S.Decay = Color3.fromHSV((rainbowHue + 0.12) % 1, 0.7, 0.8)
            S.Tint = Color3.fromHSV(rainbowHue, 0.12, 1)
        end
        if S.Cycle then
            S.Time = (S.Time + dt * S.CycleSpeed) % 24
        end
        worldTick += dt
        if worldTick >= 0.05 then
            worldTick = 0
            applyWorld()
        end
    end

    if TargetFX.Enabled then
        targetScan += dt
        if targetScan >= 0.15 or targetDirty then
            targetScan = 0
            local t = nearestTarget()
            local myRoot = getRoot(player)
            local tChar = t and t.Character
            if t ~= builtFor or targetDirty or myRoot ~= builtMyRoot or tChar ~= builtChar then
                targetDirty = false
                builtFor, builtMyRoot, builtChar = t, myRoot, tChar
                buildTarget(t)
            end
            currentTarget = t
        end
        updateTarget()
    end

    pcall(updateAvatar)
    pcall(updateSky, dt)
    pcall(updateAurora3D)
    pcall(updateMeteors, dt)
    stepLightning(dt)

    if fpsTime >= 1 then
        local fps = math.floor(fpsFrames / fpsTime + 0.5)
        fpsFrames, fpsTime = 0, 0
        local ping = 0
        pcall(function()
            ping = math.floor(Stats.Network.ServerStatsItem["Data Ping"]:GetValue())
        end)
        islandText.Text = string.format("%d FPS  •  %d MS", fps, ping)
        islandDot.BackgroundColor3 = fps >= 45 and T.green or (fps >= 30 and C(255, 205, 90) or T.red)
        if opened then uiScale.Scale = baseScale() end
    end
end))

track(RunService.Heartbeat:Connect(function(dt)
    for _, info in pairs(tracked) do
        local r = info.root
        if r and r.Parent then info.pos = r.Position end
    end
    reapply += dt
    if reapply >= 0.6 then
        reapply = 0
        if S.World and not (S.Rainbow or S.Cycle) then applyWorld() end
    end
    stepRipples(dt)
    pcall(stepSnow)
end))

----------------------------------------------------------------
-- UNLOAD
----------------------------------------------------------------

env.KitVisualsUnload = function()
    if not alive then return end
    alive = false

    for _, connection in ipairs(connections) do
        pcall(function() connection:Disconnect() end)
    end
    for other in pairs(tracked) do dropTracked(other) end

    pcall(clearSnow)
    pcall(clearTarget)
    pcall(clearSelf)
    pcall(clearAvatar)
    pcall(clearSky)
    pcall(clearAurora3D)
    pcall(restoreWorld)
    pcall(function()
        local camera = workspace.CurrentCamera
        if camera and FOVS.Original then camera.FieldOfView = FOVS.Original end
    end)
    if cinematicEffect then pcall(function() cinematicEffect:Destroy() end) end
    pcall(function() targetFolder:Destroy() end)
    pcall(function() killFolder:Destroy() end)
    pcall(function() particleFolder:Destroy() end)
    pcall(function() snowCaps:Destroy() end)
    pcall(function() skyFolder:Destroy() end)
    pcall(function() selfFolder:Destroy() end)
    pcall(function() gui:Destroy() end)
    pcall(function() fxGui:Destroy() end)

    env.KitVisualsUnload = nil
end

----------------------------------------------------------------
-- START
----------------------------------------------------------------

layoutCrosshair()
setParticleCount(P.Count)
applyWorld()
refreshQuick()
for _, other in ipairs(Players:GetPlayers()) do
    if other ~= player and other.Character then
        task.spawn(watchCharacter, other, other.Character)
    end
end

task.delay(0.6, function()
    if alive then notify("KIT VISUALS v3 загружен") end
end)
