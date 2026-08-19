-- Dan FFA Triggerbot | Key System
-- Premium key: ownertb   |   Free key: tb
-- Free: only triggerbot + notifications, Dan FFA only.
-- Full: all features, all modes + faster trigger priority.

local CORRECT_KEY = "ownertb"
local FREE_KEY = "tb"

local GENV = getgenv()
GENV.DanFFA_KeyTB = GENV.DanFFA_KeyTB or {}
local S = GENV.DanFFA_KeyTB

local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer

if S.running then
    S.enabled = not S.enabled
    if S.WindUI then
        pcall(function()
            S.WindUI:Notify({
                Title = "Triggerbot",
                Content = S.enabled and "ON" or "OFF",
                Duration = 1.5,
                Icon = S.enabled and "check" or "cancel",
            })
        end)
    end
    print("[Triggerbot] Toggled:", S.enabled and "ON" or "OFF")
    return
end

local function fetch(url)
    local ok, src = pcall(function()
        return game:HttpGet(url)
    end)
    if ok and src then
        return src
    end
    return request({ Url = url, Method = "GET" }).Body
end

local WindUI = assert(loadstring(fetch("https://github.com/Footagesus/WindUI/releases/latest/download/main.lua")), "WindUI failed to load")()
pcall(function()
    WindUI:SetParent(gethui())
end)
if not WindUI.Parent then
    pcall(function()
        WindUI:SetParent(game:GetService("CoreGui"))
    end)
end
S.WindUI = WindUI

S.notifications = (S.notifications == nil) and true or S.notifications

local function notify(title, content, icon)
    if not S.notifications then return end
    pcall(function()
        WindUI:Notify({
            Title = title,
            Content = content,
            Duration = 1.5,
            Icon = icon or "info",
        })
    end)
end

-- ===== WINDOW with key system =====
local Window = WindUI:CreateWindow({
    Title = "⚡ jhuzz triggerbot",
    Author = "Key System",
    Folder = "DanFFA",
    Size = UDim2.fromOffset(520, 480),
    Resizable = false,
    HideSearchBar = true,
    KeySystem = {
        KeyValidator = function(enteredKey)
            if enteredKey == CORRECT_KEY then
                S.access = "full"
                return true
            elseif enteredKey == FREE_KEY then
                S.access = "free"
                return true
            else
                pcall(function()
                    LocalPlayer:Kick("Invalid Key")
                end)
                return false
            end
        end,
        Note = "Enter your key to continue.",
        SaveKey = false,
    },
})

while not S.access do
    task.wait(0.1)
end

S.running = true
S.enabled = false
S.toggleKey = S.toggleKey or "C"

-- Set features based on access level
if S.access == "full" then
    S.noRecoil = true
    S.espEnabled = false
    S.botDetect = true
else
    S.noRecoil = false
    S.espEnabled = false
    S.botDetect = false
end

print("[Init] Access:", S.access, "| ToggleKey:", S.toggleKey)

-- ===== Determine game mode (full only, free forced to Dan FFA) =====
local detectedMode = "Dan FFA"
if S.access == "full" then
    local gid = game.GameId
    if gid == 7406797672 then
        detectedMode = "Da Track"
    elseif gid == 8050914790 then
        detectedMode = "Aladia"
    elseif gid == 8795154789 then
        detectedMode = "Flick"
    elseif gid == 4914269443 then
        detectedMode = "Unnamed Shooter"
    elseif gid == 10330158837 then
        detectedMode = "Alkedion PVP"
    end
end
S.mode = detectedMode
print("[Init] Game mode:", S.mode)

local ESPDraws = {}
local function cleanESP()
    for _, d in pairs(ESPDraws) do
        for _, obj in pairs(d) do
            obj:Remove()
        end
    end
    table.clear(ESPDraws)
end

local RunService = game:GetService("RunService")
local UIS = game:GetService("UserInputService")
local WS = game:GetService("Workspace")
local CollectionService = game:GetService("CollectionService")
local Lighting = game:GetService("Lighting")
local Characters = WS:FindFirstChild("Characters")
local Bots = WS:FindFirstChild("Bots")
local Blacklisted = WS:FindFirstChild("Blacklisted")

local TargetFolders = {}
local BotToggle
local triggerHeld = false
local function applyMode(mode)
    if S.mode == "Unnamed Shooter" and triggerHeld then
        triggerHeld = false
        pcall(mouse1release)
    end
    S.mode = mode
    TargetFolders = {}
    if mode == "Da Track" then
        if Bots then
            TargetFolders[#TargetFolders + 1] = Bots
        end
        S.botDetect = true
        if BotToggle then
            BotToggle:Set(true)
        end
    elseif mode == "Flick" then
        S.botDetect = false
        if BotToggle then
            BotToggle:Set(false)
        end
    elseif mode == "Unnamed Shooter" then
        TargetFolders = {}
        S.botDetect = false
        if BotToggle then
            BotToggle:Set(false)
        end
    elseif mode == "Alkedion PVP" then
        if Characters then
            TargetFolders[#TargetFolders + 1] = Characters
        else
            TargetFolders = { WS }
        end
        S.botDetect = false
        if BotToggle then
            BotToggle:Set(false)
        end
    else
        if Characters then
            TargetFolders[#TargetFolders + 1] = Characters
        end
        S.botDetect = false
        if BotToggle then
            BotToggle:Set(false)
        end
    end
end

-- ===== BUILD UI CONDITIONALLY =====
local function buildUI()
    local success, err = pcall(function()
        local Tab = Window:Tab({ Title = "⚙️ Triggerbot", Icon = "crosshair" })

        -- Mode selection: for free, show disabled dropdown with only Dan FFA
        if S.access == "full" then
            local ModeDropdown = Tab:Dropdown({
                Title = "Game Mode",
                Desc = "Headshot-only: Aladia, Alkedion PVP, Flick. Others: body shots.",
                Values = { "Dan FFA", "Da Track", "Aladia", "Alkedion PVP", "Flick", "Unnamed Shooter" },
                Value = S.mode,
                Callback = function(selected)
                    applyMode(selected)
                    notify("Game Mode", selected, "check")
                end,
            })
        else
            -- Free: force Dan FFA, show a read-only dropdown (disabled) or a label
            local ModeDropdown = Tab:Dropdown({
                Title = "Game Mode",
                Desc = "Free version only supports Dan FFA.",
                Values = { "Dan FFA" },
                Value = "Dan FFA",
                Callback = function() end,
            })
            S.mode = "Dan FFA"
            applyMode("Dan FFA")
        end

        -- Common toggles (always present)
        local Toggle = Tab:Toggle({
            Title = "Triggerbot",
            Desc = "Auto-shoots when crosshair is on an enemy. Press " .. S.toggleKey .. " to toggle.",
            Value = false,
            Callback = function(state)
                S.enabled = state
                print("[Triggerbot] UI toggle ->", state and "ON" or "OFF")
                notify("Triggerbot", state and "ON" or "OFF", state and "check" or "cancel")
            end,
        })

        local KeyInput = Tab:Input({
            Title = "Toggle Keybind",
            Desc = "Type any Roblox key name (e.g., C, RightMouseButton, LeftShift, F1).",
            Placeholder = "C",
            Value = S.toggleKey,
            Callback = function(text)
                local keyName = text:gsub("%s+", "")
                if keyName == "" then
                    keyName = "C"
                end
                local ok, enumKey = pcall(function()
                    return Enum.KeyCode[keyName]
                end)
                if ok and enumKey then
                    S.toggleKey = keyName
                    Toggle:SetDesc("Auto-shoots when crosshair is on an enemy. Press " .. keyName .. " to toggle.")
                    print("[Keybind] Set to:", keyName)
                    notify("Keybind", "Toggle key set to " .. keyName, "check")
                else
                    print("[Keybind] Invalid key:", keyName)
                    notify("Keybind", "Invalid key name. Using previous key.", "cancel")
                end
            end,
        })

        local NotifToggle = Tab:Toggle({
            Title = "Notifications",
            Desc = "Show or hide all notification popups.",
            Value = S.notifications,
            Callback = function(state)
                S.notifications = state
                print("[Notifications]", state and "ON" or "OFF")
                if state then
                    notify("Notifications", "Enabled", "check")
                end
            end,
        })

        -- ===== FULL VERSION ONLY =====
        if S.access == "full" then
            -- No Recoil
            local NoRecoil = Tab:Toggle({
                Title = "No Recoil",
                Desc = "Cancels the camera kick on every shot.",
                Value = true,
                Callback = function(state)
                    S.noRecoil = state
                    notify("No Recoil", state and "ON" or "OFF", state and "check" or "cancel")
                end,
            })

            local ESPToggle = Tab:Toggle({
                Title = "ESP",
                Desc = "Boxes, health and distance on all enemies.",
                Value = false,
                Callback = function(state)
                    S.espEnabled = state
                    if not state then
                        cleanESP()
                    end
                    notify("ESP", state and "ON" or "OFF", state and "check" or "cancel")
                end,
            })

            BotToggle = Tab:Toggle({
                Title = "Bot Detected",
                Desc = "Alerts when the trainer bot is in view.",
                Value = true,
                Callback = function(state)
                    S.botDetect = state
                end,
            })

            -- Visual tab (avatar copy)
            local VisualTab = Window:Tab({ Title = "🎨 Visual", Icon = "eye" })
            S.username = S.username or ""
            local UsernameInput = VisualTab:Input({
                Title = "Username",
                Desc = "Type a username, then copy their look onto you.",
                Placeholder = "jhuzz20",
                Value = S.username,
                Callback = function(text)
                    S.username = text
                end,
            })

            local function getDescription(name)
                local target = Players:FindFirstChild(name)
                if target then
                    local char = target.Character
                    if char then
                        local hum = char:FindFirstChildOfClass("Humanoid")
                        if hum then
                            local ok, desc = pcall(function()
                                return hum:GetAppliedDescription() or hum:GetDescription()
                            end)
                            if ok and desc then
                                return desc
                            end
                        end
                    end
                    local ok, userId = pcall(function()
                        return target.UserId
                    end)
                    if ok and userId then
                        local ok2, desc = pcall(function()
                            return Players:GetCharacterAppearanceAsync(userId)
                        end)
                        if ok2 and desc then
                            return desc
                        end
                    end
                end
                local ok, userId = pcall(function()
                    return Players:GetUserIdFromNameAsync(name)
                end)
                if not ok or not userId then
                    return nil, "No player found with that name."
                end
                local ok2, desc = pcall(function()
                    return Players:GetCharacterAppearanceAsync(userId)
                end)
                if not ok2 or not desc then
                    return nil, "Could not load their avatar."
                end
                return desc
            end

            VisualTab:Button({
                Title = "Copy Avatar",
                Icon = "copy",
                Callback = function()
                    local name = (S.username or ""):gsub("^%s+", ""):gsub("%s+$", "")
                    if #name == 0 then
                        notify("Avatar", "Type a username first.", "info")
                        return
                    end
                    task.spawn(function()
                        local desc, err = getDescription(name)
                        if not desc then
                            notify("Avatar", err or "Failed to load.", "cancel")
                            return
                        end
                        local char = LocalPlayer.Character
                        local hum = char and char:FindFirstChildOfClass("Humanoid")
                        if not hum then
                            notify("Avatar", "No character loaded yet.", "cancel")
                            return
                        end
                        local ok = pcall(function()
                            hum:ApplyDescription(desc)
                        end)
                        if ok then
                            notify("Avatar", "You now look like " .. name, "check")
                        else
                            notify("Avatar", "Could not apply - this game may use custom characters.", "cancel")
                        end
                    end)
                end,
            })

            -- Performance tab
            local PerfTab = Window:Tab({ Title = "⚡ Performance", Icon = "zap" })

            local originalSettings = {
                globalShadows = Lighting.GlobalShadows,
                brightness = Lighting.Brightness,
                technology = Lighting.Technology,
                qualityLevel = nil,
            }
            pcall(function()
                local gameSettings = UserSettings().GameSettings
                if gameSettings then
                    originalSettings.qualityLevel = gameSettings.GraphicsQualityLevel
                end
            end)

            local function setGraphicsLevel(level)
                pcall(function()
                    local gameSettings = UserSettings().GameSettings
                    if gameSettings then
                        gameSettings.GraphicsQualityLevel = level
                    end
                end)
            end

            local lowGraphics = false
            local disableShadows = false
            local disableParticles = false
            local disableAtmosphere = false
            local disablePostProcessing = false

            local function applyPerformanceSettings()
                if disableShadows then
                    Lighting.GlobalShadows = false
                else
                    Lighting.GlobalShadows = originalSettings.globalShadows
                end

                if lowGraphics then
                    Lighting.Brightness = 0.5
                else
                    Lighting.Brightness = originalSettings.brightness
                end

                if lowGraphics then
                    Lighting.Technology = Enum.Technology.Voxel
                else
                    Lighting.Technology = originalSettings.technology
                end

                if lowGraphics then
                    setGraphicsLevel(1)
                else
                    if originalSettings.qualityLevel then
                        setGraphicsLevel(originalSettings.qualityLevel)
                    end
                end

                if disableAtmosphere then
                    local atmosphere = Lighting:FindFirstChildOfClass("Atmosphere")
                    if atmosphere then atmosphere.Enabled = false end
                    local sky = Lighting:FindFirstChildOfClass("Sky")
                    if sky then sky.Enabled = false end
                else
                    local atmosphere = Lighting:FindFirstChildOfClass("Atmosphere")
                    if atmosphere then atmosphere.Enabled = true end
                    local sky = Lighting:FindFirstChildOfClass("Sky")
                    if sky then sky.Enabled = true end
                end

                if disablePostProcessing then
                    local bloom = Lighting:FindFirstChildOfClass("BloomEffect")
                    if bloom then bloom.Enabled = false end
                    local cc = Lighting:FindFirstChildOfClass("ColorCorrectionEffect")
                    if cc then cc.Enabled = false end
                    local blur = Lighting:FindFirstChildOfClass("BlurEffect")
                    if blur then blur.Enabled = false end
                    local depth = Lighting:FindFirstChildOfClass("DepthOfFieldEffect")
                    if depth then depth.Enabled = false end
                end

                if disableParticles then
                    for _, obj in ipairs(WS:GetDescendants()) do
                        if obj:IsA("ParticleEmitter") then
                            obj.Enabled = false
                        end
                    end
                else
                    for _, obj in ipairs(WS:GetDescendants()) do
                        if obj:IsA("ParticleEmitter") then
                            obj.Enabled = true
                        end
                    end
                end
            end

            PerfTab:Toggle({
                Title = "Low Graphics",
                Desc = "Sets graphics quality to 1, reduces brightness, uses Voxel lighting.",
                Value = false,
                Callback = function(state)
                    lowGraphics = state
                    applyPerformanceSettings()
                    notify("Performance", "Low Graphics " .. (state and "ON" or "OFF"), state and "check" or "cancel")
                end,
            })

            PerfTab:Toggle({
                Title = "Disable Shadows",
                Desc = "Disables global shadows for a big FPS boost.",
                Value = false,
                Callback = function(state)
                    disableShadows = state
                    applyPerformanceSettings()
                    notify("Performance", "Shadows " .. (state and "OFF" or "ON"), state and "check" or "cancel")
                end,
            })

            PerfTab:Toggle({
                Title = "Disable Particles",
                Desc = "Disables all particle emitters in the workspace.",
                Value = false,
                Callback = function(state)
                    disableParticles = state
                    applyPerformanceSettings()
                    notify("Performance", "Particles " .. (state and "OFF" or "ON"), state and "check" or "cancel")
                end,
            })

            PerfTab:Toggle({
                Title = "Disable Atmosphere & Sky",
                Desc = "Disables Atmosphere and Sky effects.",
                Value = false,
                Callback = function(state)
                    disableAtmosphere = state
                    applyPerformanceSettings()
                    notify("Performance", "Atmosphere " .. (state and "OFF" or "ON"), state and "check" or "cancel")
                end,
            })

            PerfTab:Toggle({
                Title = "Disable Post-Processing",
                Desc = "Disables bloom, color correction, blur, depth of field.",
                Value = false,
                Callback = function(state)
                    disablePostProcessing = state
                    applyPerformanceSettings()
                    notify("Performance", "Post-Processing " .. (state and "OFF" or "ON"), state and "check" or "cancel")
                end,
            })

            PerfTab:Button({
                Title = "Reset All Performance Settings",
                Icon = "refresh",
                Callback = function()
                    lowGraphics = false
                    disableShadows = false
                    disableParticles = false
                    disableAtmosphere = false
                    disablePostProcessing = false
                    Lighting.GlobalShadows = originalSettings.globalShadows
                    Lighting.Brightness = originalSettings.brightness
                    Lighting.Technology = originalSettings.technology
                    if originalSettings.qualityLevel then
                        setGraphicsLevel(originalSettings.qualityLevel)
                    end
                    for _, obj in ipairs(WS:GetDescendants()) do
                        if obj:IsA("ParticleEmitter") then
                            obj.Enabled = true
                        end
                    end
                    local atmosphere = Lighting:FindFirstChildOfClass("Atmosphere")
                    if atmosphere then atmosphere.Enabled = true end
                    local sky = Lighting:FindFirstChildOfClass("Sky")
                    if sky then sky.Enabled = true end
                    local bloom = Lighting:FindFirstChildOfClass("BloomEffect")
                    if bloom then bloom.Enabled = true end
                    local cc = Lighting:FindFirstChildOfClass("ColorCorrectionEffect")
                    if cc then cc.Enabled = true end
                    local blur = Lighting:FindFirstChildOfClass("BlurEffect")
                    if blur then blur.Enabled = true end
                    local depth = Lighting:FindFirstChildOfClass("DepthOfFieldEffect")
                    if depth then depth.Enabled = true end
                    notify("Performance", "All settings reset to default.", "check")
                end,
            })
        end -- end full version

        -- Apply mode again (in case free mode wasn't applied)
        applyMode(S.mode)
        print("[UI] Build complete.")
    end)
    if not success then
        print("[ERROR] UI Build failed:", err)
        notify("UI Error", "Something went wrong. Check console.", "cancel")
    end
end

buildUI()

-- Welcome
local versionText = (S.access == "full" and "Full Version" or "Free Version")
notify("🔓 Key Accepted", versionText .. " – Press " .. S.toggleKey .. " to toggle triggerbot.", "check")

-- ===== CORE ENGINE =====
local RayParams = RaycastParams.new()
RayParams.FilterType = Enum.RaycastFilterType.Exclude
RayParams.IgnoreWater = true

-- No Recoil (only if enabled and full version)
local RECOIL_PITCH = 0.004363323129985824
local recoilState = nil
local lastFired = nil

local function updateNoRecoil()
    if not S.noRecoil then return end
    local char = LocalPlayer.Character
    if not char then return end
    local state = char:FindFirstChild("State")
    if not state then return end
    local fired = state:FindFirstChild("Fired")
    if not fired then return end
    if state ~= recoilState then
        recoilState = state
        lastFired = fired.Value
        return
    end
    local diff = fired.Value - lastFired
    lastFired = fired.Value
    if diff > 0 then
        local cam = WS.CurrentCamera
        if cam then
            cam.CFrame = cam.CFrame * CFrame.Angles(-RECOIL_PITCH * diff, 0, 0)
        end
    end
end

local trackAmmoInst = nil
local trackLastAmmo = nil
local function updateNoRecoilTrack(tool)
    if not S.noRecoil then return end
    if not tool then return end
    local ammo = tool:FindFirstChild("Ammo")
    if not ammo then return end
    if ammo ~= trackAmmoInst then
        trackAmmoInst = ammo
        trackLastAmmo = ammo.Value
        return
    end
    local diff = trackLastAmmo - ammo.Value
    trackLastAmmo = ammo.Value
    if diff > 0 then
        local cam = WS.CurrentCamera
        if cam then
            local mult = 1
            local char = LocalPlayer.Character
            if char and char:GetAttribute("Aiming") ~= true then
                mult = 2
            end
            cam.CFrame = cam.CFrame * CFrame.Angles(-RECOIL_PITCH * diff * mult, 0, 0)
        end
    end
end

local detectedBots = {}
local function updateBotDetect()
    if S.mode ~= "Da Track" or not S.botDetect or not Bots then
        return
    end
    local cam = WS.CurrentCamera
    if not cam then return end
    local char = LocalPlayer.Character
    local filter = { Bots, WS:FindFirstChild("Ignored"), WS:FindFirstChild("ragdolls") }
    if char then filter[#filter + 1] = char end
    for _, bot in ipairs(Bots:GetChildren()) do
        local hum = bot:FindFirstChildOfClass("Humanoid")
        if hum and hum.Health > 0 then
            local root = bot:FindFirstChild("HumanoidRootPart")
            if root then
                local pos = root.Position + Vector3.new(0, 1, 0)
                local dist = (pos - cam.CFrame.Position).Magnitude
                if dist <= 400 then
                    local rp = RaycastParams.new()
                    rp.FilterType = Enum.RaycastFilterType.Exclude
                    rp.FilterDescendantsInstances = filter
                    local hit = WS:Raycast(cam.CFrame.Position, (pos - cam.CFrame.Position).Unit * dist, rp)
                    local detected = not hit or hit.Instance:IsDescendantOf(bot)
                    if detected then
                        if not detectedBots[bot.Name] then
                            detectedBots[bot.Name] = true
                            notify("Bot Detected", bot.Name .. " - " .. tostring(math.floor(dist)) .. "m", "crosshair")
                        end
                    else
                        detectedBots[bot.Name] = nil
                    end
                else
                    detectedBots[bot.Name] = nil
                end
            end
        else
            detectedBots[bot.Name] = nil
        end
    end
end

local function getEnemyModel(instance)
    if not instance then return nil end
    if S.mode == "Flick" then
        local model = instance
        while model and model.Parent ~= WS do
            model = model.Parent
        end
        if not model or model == LocalPlayer.Character or model.Name == LocalPlayer.Name then
            return nil
        end
        local plr = Players:GetPlayerFromCharacter(model)
        if not plr or not plr.Team or plr.Team.Name ~= "Play" then
            return nil
        end
        local hum = model:FindFirstChildOfClass("Humanoid")
        if hum and hum.Health > 0 and hum.Health <= hum.MaxHealth then
            return model
        end
        return nil
    end

    if S.mode == "Unnamed Shooter" then
        local model = instance
        while model do
            if model:IsA("Model") and model:FindFirstChildOfClass("Humanoid") then
                break
            end
            model = model.Parent
        end
        if not model then return nil end
        if model == LocalPlayer.Character then return nil end
        if model.Name == LocalPlayer.Name then return nil end
        local hum = model:FindFirstChildOfClass("Humanoid")
        if hum and hum.Health > 0 and hum.Health <= hum.MaxHealth then
            local plr = Players:GetPlayerFromCharacter(model)
            if plr and plr ~= LocalPlayer then
                return model
            end
        end
        return nil
    end

    -- Generic folder-based (Dan FFA, Aladia, Alkedion PVP)
    local model = instance
    while model do
        local p = model.Parent
        local inTarget = false
        for _, f in ipairs(TargetFolders) do
            if p == f then
                inTarget = true
                break
            end
        end
        if inTarget then
            break
        end
        model = p
    end
    if not model then return nil end
    if model == LocalPlayer.Character then return nil end
    if model.Name == LocalPlayer.Name then return nil end
    local hum = model:FindFirstChildOfClass("Humanoid")
    if hum and hum.Health > 0 and hum.Health <= hum.MaxHealth then
        return model
    end
    return nil
end

local function isHeadHit(instance, model)
    local headPart
    if S.mode == "Flick" then
        headPart = model:FindFirstChild("Crit")
    else
        headPart = model:FindFirstChild("Head")
    end
    if not headPart then return false end
    local p = instance
    while p do
        if p == headPart then
            return true
        end
        p = p.Parent
    end
    return false
end

local function hasAmmo(weapon)
    if S.mode == "Flick" then
        return true
    end
    if S.mode == "Aladia" then
        if weapon:GetAttribute("_reloading") == true then
            return false
        end
        local a = weapon:GetAttribute("_ammo")
        return a == nil or a > 0
    end
    if S.mode == "Alkedion PVP" then
        if weapon:GetAttribute("Reloading") == true then
            return false
        end
        local a = weapon:GetAttribute("Ammo")
        if a ~= nil then return a > 0 end
        local ammoVal = weapon:FindFirstChild("Ammo")
        if ammoVal then return ammoVal.Value > 0 end
        local ammoClient = weapon:FindFirstChild("Ammo_CLIENT")
        if ammoClient then return ammoClient.Value > 0 end
        return true
    end
    if S.mode == "Unnamed Shooter" then
        if not weapon then return false end
        local config = weapon:FindFirstChild("Config")
        if config then
            local clip = config:GetAttribute("Clip") or config:GetAttribute("Ammo")
            if clip ~= nil and clip <= 0 then
                return false
            end
        end
        return true
    end
    local ammo = weapon:FindFirstChild("Ammo_CLIENT") or weapon:FindFirstChild("Ammo")
    return ammo ~= nil and ammo.Value > 0
end

local function canFire(char)
    if char:GetAttribute("Frozen") == true then
        return false
    end
    if char:GetAttribute("Reloading") == true then
        return false
    end
    if S.mode == "Alkedion PVP" then
        local state = char:FindFirstChild("State")
        if state then
            local sc = state:FindFirstChild("Shooting_CLIENT")
            local rc = state:FindFirstChild("Reload_CLIENT")
            local dead = state:FindFirstChild("Dead")
            local down = state:FindFirstChild("Down")
            if sc and sc.Value then return false end
            if rc and rc.Value then return false end
            if dead and dead.Value then return false end
            if down and down.Value then return false end
        end
        return true
    end
    if S.mode == "Flick" then
        if char:GetAttribute("IsFiring") == true then
            return false
        end
        if char:GetAttribute("IsInspecting") == true then
            return false
        end
        return true
    end
    local state = char:FindFirstChild("State")
    if state then
        local sc = state:FindFirstChild("Shooting_CLIENT")
        local rc = state:FindFirstChild("Reload_CLIENT")
        local dead = state:FindFirstChild("Dead")
        local down = state:FindFirstChild("Down")
        if sc and sc.Value then return false end
        if rc and rc.Value then return false end
        if dead and dead.Value then return false end
        if down and down.Value then return false end
    end
    return true
end

-- Aladia recoil (full only)
local aladiaRecoiler = nil
local aladiaRecoilHooked = false
local aladiaOrigRecoil = nil
local function updateAladiaRecoil()
    if S.mode ~= "Aladia" then return end
    if not S.noRecoil then return end
    if not aladiaRecoiler then
        local RS = game:GetService("ReplicatedStorage")
        local mod = RS:FindFirstChild("Blaster") and RS.Blaster:FindFirstChild("Scripts") and RS.Blaster.Scripts:FindFirstChild("CameraRecoiler")
        if not mod then return end
        local ok, rec = pcall(function()
            return require(mod)
        end)
        if not ok or type(rec) ~= "table" then
            return
        end
        aladiaRecoiler = rec
        aladiaOrigRecoil = rec.recoil
    end
    if S.noRecoil and not aladiaRecoilHooked then
        aladiaRecoiler.recoil = function() end
        aladiaRecoilHooked = true
    elseif not S.noRecoil and aladiaRecoilHooked then
        if aladiaOrigRecoil then
            aladiaRecoiler.recoil = aladiaOrigRecoil
        end
        aladiaRecoilHooked = false
    end
end

-- Alkedion recoil (full only)
local alkedionRecoilHooked = false
local function updateAlkedionRecoil()
    if S.mode ~= "Alkedion PVP" then return end
    if not S.noRecoil or alkedionRecoilHooked then return end
    local RS = game:GetService("ReplicatedStorage")
    local guns = RS:FindFirstChild("Modules") and RS.Modules:FindFirstChild("Guns")
    if not guns then return end
    local handler = guns:FindFirstChild("GunHandler")
    if not handler then return end
    local ok, rec = pcall(function()
        return require(handler)
    end)
    if not ok or type(rec) ~= "table" then
        return
    end
    if rec.recoil and type(rec.recoil) == "function" then
        rec.recoil = function() end
        alkedionRecoilHooked = true
        print("[Alkedion] Recoil disabled.")
    end
    if rec.Recoil and type(rec.Recoil) == "function" then
        rec.Recoil = function() end
        alkedionRecoilHooked = true
        print("[Alkedion] Recoil disabled.")
    end
end

local cachedFilter = {}
local function updateFilter()
    local char = LocalPlayer.Character
    local ex = CollectionService:GetTagged("RayExclude")
    local filter = {}
    if char then filter[#filter + 1] = char end
    local vm = WS:FindFirstChild("ViewModel")
    if vm then filter[#filter + 1] = vm end
    for _, v in ipairs(ex) do
        filter[#filter + 1] = v
    end
    if Blacklisted then
        filter[#filter + 1] = Blacklisted
    end
    cachedFilter = filter
    RayParams.FilterDescendantsInstances = filter
end
LocalPlayer.CharacterAdded:Connect(function()
    if triggerHeld then
        triggerHeld = false
        pcall(mouse1release)
    end
    updateFilter()
end)
CollectionService:GetInstanceAddedSignal("RayExclude"):Connect(updateFilter)
CollectionService:GetInstanceRemovedSignal("RayExclude"):Connect(updateFilter)

-- ===== onRender function (shared, but priority differs) =====
local function onRender()
    local char = LocalPlayer.Character
    if char and char.Parent then
        local weapon = nil
        if S.mode == "Unnamed Shooter" then
            local equipped = char:FindFirstChild("Equipped")
            if equipped and equipped:IsA("ObjectValue") and equipped.Value then
                weapon = equipped.Value
            end
        else
            weapon = char:FindFirstChildOfClass("Tool")
        end
        if S.mode == "Da Track" then
            updateNoRecoilTrack(weapon)
        elseif S.mode == "Aladia" then
            updateAladiaRecoil()
        elseif S.mode == "Alkedion PVP" then
            updateAlkedionRecoil()
        elseif S.mode == "Unnamed Shooter" and S.noRecoil then
            local hum = char:FindFirstChildOfClass("Humanoid")
            if hum then
                hum.CameraOffset = Vector3.new(0, 0, 0)
            end
        else
            updateNoRecoil()
        end
        if S.enabled and weapon then
            if hasAmmo(weapon) and canFire(char) then
                local cam = WS.CurrentCamera
                if cam then
                    local hit = WS:Raycast(cam.CFrame.Position, cam.CFrame.LookVector * 500, RayParams)
                    if hit then
                        local enemy = getEnemyModel(hit.Instance)
                        if enemy then
                            local headshotModes = { "Aladia", "Alkedion PVP", "Flick" }
                            local requireHead = false
                            for _, mode in ipairs(headshotModes) do
                                if S.mode == mode then
                                    requireHead = true
                                    break
                                end
                            end
                            if requireHead and not isHeadHit(hit.Instance, enemy) then
                                -- skip
                            else
                                print("[Triggerbot] SHOOTING at", enemy.Name)
                                if S.mode == "Unnamed Shooter" then
                                    if not triggerHeld then
                                        triggerHeld = true
                                        mouse1press()
                                    end
                                else
                                    mouse1click()
                                end
                                return
                            end
                        end
                    end
                end
            end
        end
        if S.mode == "Unnamed Shooter" and triggerHeld then
            triggerHeld = false
            mouse1release()
        end
    end
end

-- ===== Bind the triggerbot with priority based on access =====
local priority = (S.access == "full") and Enum.RenderPriority.Camera.Value + 500 or Enum.RenderPriority.Camera.Value + 100
RunService:BindToRenderStep("DanFFA_Trigger", priority, onRender)
print("[Priority] Triggerbot priority set to:", priority)

-- Bot detection (separate, lower priority)
RunService.Heartbeat:Connect(updateBotDetect)

-- ESP (full only, but we still connect it; it checks internally)
RunService.RenderStepped:Connect(renderESP)

-- ESP function
local function renderESP()
    if not S.espEnabled then return end
    local cam = WS.CurrentCamera
    if not cam then return end
    local alive = {}
    local drawn = {}
    local models = {}

    if S.mode == "Flick" then
        for _, plr in ipairs(Players:GetPlayers()) do
            if plr ~= LocalPlayer and plr.Team and plr.Team.Name == "Play" then
                local model = plr.Character
                if model then
                    models[#models + 1] = model
                end
            end
        end
    elseif S.mode == "Unnamed Shooter" then
        for _, plr in ipairs(Players:GetPlayers()) do
            if plr ~= LocalPlayer then
                local model = plr.Character
                if model then
                    models[#models + 1] = model
                end
            end
        end
    else
        for _, folder in ipairs(TargetFolders) do
            for _, model in ipairs(folder:GetChildren()) do
                models[#models + 1] = model
            end
        end
    end

    for _, model in ipairs(models) do
        if model.Name ~= LocalPlayer.Name and model ~= LocalPlayer.Character then
            local hum = model:FindFirstChildOfClass("Humanoid")
            if hum and hum.Health > 0 and hum.Health <= hum.MaxHealth then
                local root = model:FindFirstChild("HumanoidRootPart")
                if root then
                    local isBot = Bots ~= nil and model.Parent == Bots
                    alive[model.Name] = true
                    local pos = root.Position
                    local p1, on1 = cam:WorldToViewportPoint(pos + Vector3.new(0, 3.2, 0))
                    local p2, on2 = cam:WorldToViewportPoint(pos - Vector3.new(0, 1.6, 0))
                    if on1 and on2 then
                        drawn[model.Name] = true
                        local h = p1.Y - p2.Y
                        local w = h * 0.6
                        local x = p1.X - w / 2
                        local y = p2.Y
                        local d = ESPDraws[model.Name]
                        if not d then
                            d = {
                                box = Drawing.new("Square"),
                                hpBg = Drawing.new("Square"),
                                hpFill = Drawing.new("Square"),
                                text = Drawing.new("Text"),
                            }
                            d.box.Thickness = 1
                            d.box.Filled = false
                            d.hpBg.Filled = true
                            d.hpBg.Color = Color3.new(0, 0, 0)
                            d.text.Size = 13
                            d.text.Center = true
                            d.text.Outline = true
                            d.text.Color = Color3.new(1, 1, 1)
                            ESPDraws[model.Name] = d
                        end
                        local ratio = math.clamp(hum.Health / hum.MaxHealth, 0, 1)
                        local col = isBot and Color3.fromRGB(0, 255, 255) or Color3.fromHSV(ratio * 0.33, 1, 1)
                        d.box.Visible = true
                        d.box.Color = col
                        d.box.Position = Vector2.new(x, y)
                        d.box.Size = Vector2.new(w, h)
                        local bw = math.max(w * 0.07, 1)
                        d.hpBg.Visible = true
                        d.hpBg.Position = Vector2.new(x - bw - 3, y)
                        d.hpBg.Size = Vector2.new(bw, h)
                        d.hpFill.Visible = true
                        d.hpFill.Color = col
                        d.hpFill.Position = Vector2.new(x - bw - 3, y + h * (1 - ratio))
                        d.hpFill.Size = Vector2.new(bw, h * ratio)
                        local dist = (pos - cam.CFrame.Position).Magnitude
                        d.text.Visible = true
                        d.text.Position = Vector2.new(p1.X, y - 16)
                        d.text.Text = (isBot and "BOT " or "") .. model.Name .. "  " .. tostring(math.floor(dist)) .. "m"
                    end
                end
            end
        end
    end
    for name, d in pairs(ESPDraws) do
        if not alive[name] then
            for _, obj in pairs(d) do
                obj:Remove()
            end
            ESPDraws[name] = nil
        elseif not drawn[name] then
            d.box.Visible = false
            d.hpBg.Visible = false
            d.hpFill.Visible = false
            d.text.Visible = false
        end
    end
end

-- Keybind handler
UIS.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    local key = S.toggleKey
    if key and key ~= "" then
        local ok, enumKey = pcall(function()
            return Enum.KeyCode[key]
        end)
        if ok and enumKey and input.KeyCode == enumKey and not UIS:GetFocusedTextBox() then
            S.enabled = not S.enabled
            notify("Triggerbot", S.enabled and "ON" or "OFF", S.enabled and "check" or "cancel")
            print("[Triggerbot] Keybind toggled ->", S.enabled and "ON" or "OFF")
        end
    end
end)

updateFilter()

print("[Init] Script loaded. Triggerbot is OFF. Press", S.toggleKey, "to toggle.")
