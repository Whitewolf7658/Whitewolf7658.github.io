[← Back to Home](../README.md)

# Drone Flight Server Framework (DroneFlightServer)

Selected code from the drone's server script. The full script also contains remote setup, visual controls and the flight controller integration.

## Signal handling

Distance-based signal strength, loss detection and failsafe handling.

```luau
local currentPilot = nil
local MAX_SIGNAL_RANGE = 350
local SIGNAL_CHECK_INTERVAL = 0.10
local RECONNECT_DISTANCE = MAX_SIGNAL_RANGE * 0.90
local signalTimer = 0
local signalLost = false

local function setSignalAttributes(distance, strength, state, connected)
    drone:SetAttribute("SignalMaxRange", MAX_SIGNAL_RANGE)
    drone:SetAttribute("SignalDistance", distance or 0)
    drone:SetAttribute("SignalStrength", math.clamp(strength or 0, 0, 100))
    drone:SetAttribute("SignalState", state or "NO LINK")
    drone:SetAttribute("SignalConnected", connected == true)
end

local function calculateSignalStrength(distance)
    if distance <= 175 then
        return 100 - (distance / 175) * 30
    elseif distance <= 260 then
        local alpha = (distance - 175) / 85
        return 70 - alpha * 30
    elseif distance <= 315 then
        local alpha = (distance - 260) / 55
        return 40 - alpha * 20
    elseif distance < MAX_SIGNAL_RANGE then
        local alpha = (distance - 315) / (MAX_SIGNAL_RANGE - 315)
        return math.max(0.1, 20 - alpha * 20)
    end

    return 0
end

local function getSignalState(strength)
    if strength >= 70 then
        return "STRONG"
    elseif strength >= 40 then
        return "GOOD"
    elseif strength >= 20 then
        return "WEAK"
    elseif strength > 0 then
        return "CRITICAL"
    end

    return "LOST"
end

local function loseSignal()
    if signalLost then
        return
    end

    signalLost = true
    setPrecisionMode(false)
    setSignalAttributes(MAX_SIGNAL_RANGE, 0, "LOST", false)
    controller:SetInput(0, 0, 0, 0)
    setFloodlightEnabled(false)
    setLaserEnabled(false)
    controller:BeginFailsafe()
end

local function updateSignalLink()
    if not currentPilot then
        signalLost = false
        setSignalAttributes(0, 0, "NO LINK", false)
        return
    end

    local character = currentPilot.Character
    local root = character and character:FindFirstChild("HumanoidRootPart")
    if not root then
        loseSignal()
        return
    end

    local distance = (base.Position - root.Position).Magnitude
    local strength = calculateSignalStrength(distance)

    if signalLost then
        if distance <= RECONNECT_DISTANCE then
            signalLost = false
            local state = getSignalState(strength)
            setSignalAttributes(distance, strength, state, true)
        else
            setSignalAttributes(distance, 0, "LOST", false)
        end

        return
    end

    if distance >= MAX_SIGNAL_RANGE then
        loseSignal()
        return
    end

    local state = getSignalState(strength)
    setSignalAttributes(distance, strength, state, true)
end
```

## Pilot ownership

Checks the drone owner before assigning a pilot and transferring network ownership.

```luau
local function isDroneOwner(player)
    if not player then
        return false
    end

    return tonumber(drone:GetAttribute("OwnerUserId")) == player.UserId
end

local function assignPilot(player)
    if not isDroneOwner(player) then
        return false
    end

    if currentPilot == player then
        pcall(function()
            base:SetNetworkOwner(player)
        end)

        return true
    end

    if currentPilot ~= nil then
        return false
    end

    currentPilot = player
    pcall(function()
        base:SetNetworkOwner(player)
    end)

    return true
end
```

## Server commands

Server-side handlers for takeoff, landing, precision mode and equipment.

```luau
droneCommand.OnServerEvent:Connect(function(player, command)
    if typeof(command) ~= "string" then
        return
    end

    if command == "BootTakeoff" then
        if not assignPilot(player) then
            return
        end

        setPrecisionMode(false)
        signalLost = false
        controller.State.FailsafeTriggered = false
        updateSignalLink()

        if drone:GetAttribute("SignalConnected") ~= true then
            return
        end

        controller:RequestTakeoff()
    elseif command == "Land" then
        if player ~= currentPilot then
            return
        end

        setPrecisionMode(false)
        controller:RequestLanding()
    elseif command == "TogglePrecisionMode" then
        if player ~= currentPilot
            or signalLost
            or drone:GetAttribute("SignalConnected") ~= true then
            return
        end

        local mode = controller.State.Mode
        if mode ~= "Armed" and mode ~= "Flying" then
            return
        end

        setPrecisionMode(drone:GetAttribute("PrecisionMode") ~= true)
    elseif command == "ToggleFloodlight" then
        if player ~= currentPilot
            or signalLost
            or drone:GetAttribute("SignalConnected") ~= true then
            return
        end

        toggleFloodlight()
    elseif command == "ToggleLaser" then
        if player ~= currentPilot
            or signalLost
            or drone:GetAttribute("SignalConnected") ~= true then
            return
        end

        toggleLaser()
    end
end)
```

## Flight inputs

Validates pilot identity and flight input values before passing them to the controller.

```luau
local function validNumber(value)
    return typeof(value) == "number"
        and value == value
        and value ~= math.huge
        and value ~= -math.huge
end

flightInput.OnServerEvent:Connect(function(player, forward, strafe, vertical, roll)
    if player ~= currentPilot or signalLost then
        return
    end

    if not validNumber(forward)
        or not validNumber(strafe)
        or not validNumber(vertical) then
        return
    end

    if not validNumber(roll) then
        roll = 0
    end

    controller:SetInput(forward, strafe, vertical, roll)
end)

droneLook.OnServerEvent:Connect(function(player, lookDirection, lookUpDirection)
    if player ~= currentPilot or signalLost then
        return
    end

    if typeof(lookDirection) ~= "Vector3" then
        return
    end

    if lookDirection.Magnitude < 0.01 then
        return
    end

    if lookUpDirection ~= nil and typeof(lookUpDirection) ~= "Vector3" then
        return
    end

    controller:SetLookDirection(lookDirection, lookUpDirection)
end)
```

## Disconnect and signal checks

Resets control when the pilot leaves and updates signal status at a set interval.

```luau
Players.PlayerRemoving:Connect(function(player)
    if player ~= currentPilot then
        return
    end

    currentPilot = nil
    setFloodlightEnabled(false)
    setLaserEnabled(false)
    signalLost = false
    setSignalAttributes(0, 0, "NO LINK", false)
    controller:SetInput(0, 0, 0, 0)

    pcall(function()
        base:SetNetworkOwner(nil)
    end)
end)

RunService.Heartbeat:Connect(function(deltaTime)
    signalTimer += deltaTime
    if signalTimer < SIGNAL_CHECK_INTERVAL then
        return
    end

    signalTimer = 0
    updateSignalLink()
end)
```
