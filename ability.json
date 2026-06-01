--[[
  Fish It tracker probe — minimal extractor with heartbeat loop.

  Sends the equipped ability snapshot to the server every 60s so the
  dashboard's online/offline status stays accurate.

  Payload shape sent to /track:
    {
      "username": "kale0000691",
      "userId": 11035457843,
      "displayName": "kale0000691",
      "ability": {        -- null if no ability is equipped
        "name": "Wind",
        "tier": 2,
        "icon": "rbxassetid://92889423893851"
      }
    }

  The server marks an account offline if it hasn't received a probe in
  > 5 minutes. The 60s cadence here gives 5 missed cycles of headroom
  before the dashboard flips offline.

  To stop the heartbeat loop early, just rejoin the game / restart the
  executor — the script lives only inside the current LocalScript context.
]]

-- ── CONFIG ─────────────────────────────────────────────────────────────
local ENDPOINT = "https://model-favor-variable-departure.trycloudflare.com"
local INTERVAL = 60   -- seconds between heartbeats
-- If your executor runs on a different machine, change ENDPOINT to the
-- public URL (e.g. https://your-tunnel.example/track).

-- ── SERVICES ───────────────────────────────────────────────────────────
local Players           = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local HttpService       = game:GetService("HttpService")

local LocalPlayer = Players.LocalPlayer
if not LocalPlayer then warn("[Probe] not in-game"); return end

-- ── EXTRACT ────────────────────────────────────────────────────────────
local function getEquippedAbility()
    local gui = LocalPlayer:FindFirstChild("PlayerGui")
    if not gui then return nil end
    local label = gui:FindFirstChild("Backpack", true)
    label = label and label:FindFirstChild("Modifiers")
    label = label and label:FindFirstChild("Ability")
    if not label then return nil end

    local name = label:GetAttribute("AbilityName")
    if not name or name == "" then return nil end

    -- Pull tier + icon from the catalog so the payload stays small.
    local tier, icon
    local abMod = ReplicatedStorage:FindFirstChild("Abilities")
    if abMod and abMod:IsA("ModuleScript") then
        local ok, defs = pcall(require, abMod)
        if ok and typeof(defs) == "table" then
            local def = defs[name]
            if typeof(def) == "table" then
                tier = (def.Data and def.Data.Tier) or def.Tier
                icon = (def.Data and def.Data.Icon) or def.Icon
            end
        end
    end

    return {
        name = name,
        tier = tier,
        icon = icon,
    }
end

local function buildPayload()
    return {
        username    = LocalPlayer.Name,
        userId      = LocalPlayer.UserId,
        displayName = LocalPlayer.DisplayName,
        ability     = getEquippedAbility(),
    }
end

-- ── HTTP ───────────────────────────────────────────────────────────────
local httpFn = (request or (syn and syn.request) or http_request)
if not httpFn then
    warn("[Probe] executor has no HTTP request function (request/syn.request/http_request)")
    return
end

local function sendOnce()
    local payload = buildPayload()
    local body    = HttpService:JSONEncode(payload)
    local label   = (payload.ability and payload.ability.name) or "(no ability)"

    local ok, res = pcall(httpFn, {
        Url     = ENDPOINT,
        Method  = "POST",
        Headers = { ["Content-Type"] = "application/json" },
        Body    = body,
    })

    if ok then
        local code = (res and (res.StatusCode or res.status)) or "?"
        print(("[Probe] %s heartbeat → %s (HTTP %s)"):format(label, ENDPOINT, tostring(code)))
    else
        warn(("[Probe] failed to POST: %s"):format(tostring(res)))
    end
end

-- ── LOOP ───────────────────────────────────────────────────────────────
-- We use task.spawn so the loop doesn't block the executor's main thread.
-- Running pcall around sendOnce keeps a transient network/parse error from
-- killing the heartbeat — we just retry on the next interval.
print(("[Probe] starting heartbeat loop · interval=%ds · target=%s"):format(INTERVAL, ENDPOINT))
sendOnce()

task.spawn(function()
    while true do
        task.wait(INTERVAL)
        local ok, err = pcall(sendOnce)
        if not ok then warn("[Probe] sendOnce error: " .. tostring(err)) end
    end
end)
