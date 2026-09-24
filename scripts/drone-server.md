[← Back to Home](../README.md)

# Drone Flight Server Framework (DroneFlightServer)

This server side script handles the networking for my drone system. It manages the remote events between the client and server, furthermore it validates which player owns and controls each drone, it also keeps track of active drone sessions, and then calculates the drone's signal range. It also validates requests on the server rather than relying entirely on information sent by the client.


```lua
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")

local drone =
	workspace:WaitForChild("Drone")

local base =
	drone:WaitForChild("Base")

local remoteFolder =
	ReplicatedStorage:FindFirstChild(
		"DroneRemotes"
	)

if not remoteFolder then
	remoteFolder =
		Instance.new("Folder")

	remoteFolder.Name =
		"DroneRemotes"

	remoteFolder.Parent =
		ReplicatedStorage
end

local function getOrCreateRemote(name)
	local remote =
		remoteFolder:FindFirstChild(
			name
		)

	if not remote then
		remote =
			Instance.new("RemoteEvent")

		remote.Name =
			name

		remote.Parent =
			remoteFolder
	end

	return remote
end

local flightInput =
	getOrCreateRemote(
		"FlightInput"
	)

local droneLook =
	getOrCreateRemote(
		"DroneLook"
	)

local droneCommand =
	getOrCreateRemote(
		"DroneCommand"
	)

local function ensureVisualPropellerMotors()
	local folder =
		base:FindFirstChild(
			"PropellerVisuals"
		)

	if not folder then
		return
	end

	for _, propeller in ipairs(
		folder:GetChildren()
	) do
		if propeller:IsA("BasePart")
			and string.find(
				propeller.Name,
				"VisualPropeller"
			) then

			local relativeCFrame =
				base.CFrame:ToObjectSpace(
					propeller.CFrame
				)

			propeller.Anchored =
				false

			propeller.CanCollide =
				false

			propeller.CanTouch =
				false

			propeller.CanQuery =
				false

			propeller.Massless =
				true

			local motor =
				propeller:FindFirstChild(
					"VisualMotor"
				)

			if not motor then
				motor =
					Instance.new(
						"Motor6D"
					)

				motor.Name =
					"VisualMotor"

				motor.Parent =
					propeller
			end

			motor.Part0 =
				base

			motor.Part1 =
				propeller

			motor.C0 =
				relativeCFrame

			motor.C1 =
				CFrame.new()
		end
	end
end

ensureVisualPropellerMotors()

local moduleScript =
	script:WaitForChild(
		"FlightController"
	)

local requireSuccess, FlightController =
	pcall(
		require,
		moduleScript
	)

if not requireSuccess then
	error(
		tostring(
			FlightController
		)
	)
end

local controller =
	FlightController.new(
		drone
	)

controller:Start()

local floodlightAttachment =
	base:WaitForChild(
		"FloodlightAttachment"
	)

local floodlight =
	floodlightAttachment:WaitForChild(
		"SpotLight"
	)

local FLOODLIGHT_BRIGHTNESS =
	math.max(
		floodlight.Brightness,
		4
	)

local FLOODLIGHT_FADE_OUT =
	0.05

local floodlightTween =
	nil

local floodlightGeneration =
	0

local function cancelFloodlightTween()
	if floodlightTween then
		floodlightTween:Cancel()
		floodlightTween = nil
	end
end

local function setFloodlightEnabled(enabled)
	enabled =
		enabled == true

	floodlightGeneration += 1

	local generation =
		floodlightGeneration

	cancelFloodlightTween()

	drone:SetAttribute(
		"FloodlightEnabled",
		enabled
	)

	if enabled then
		floodlight.Brightness =
			FLOODLIGHT_BRIGHTNESS

		floodlight.Enabled =
			true

		return
	end

	if not floodlight.Enabled then
		floodlight.Brightness =
			FLOODLIGHT_BRIGHTNESS

		return
	end

	local tween =
		TweenService:Create(
			floodlight,
			TweenInfo.new(
				FLOODLIGHT_FADE_OUT,
				Enum.EasingStyle.Quad,
				Enum.EasingDirection.In
			),
			{
				Brightness = 0
			}
		)

	floodlightTween =
		tween

	tween:Play()

	tween.Completed:Connect(function()
		if generation ~= floodlightGeneration then
			return
		end

		if drone:GetAttribute(
			"FloodlightEnabled"
		) ~= true then
			floodlight.Enabled =
				false

			floodlight.Brightness =
				FLOODLIGHT_BRIGHTNESS
		end

		if floodlightTween == tween then
			floodlightTween = nil
		end
	end)
end

local function toggleFloodlight()
	setFloodlightEnabled(
		drone:GetAttribute(
			"FloodlightEnabled"
		) ~= true
	)
end

floodlight.Enabled =
	false

floodlight.Brightness =
	FLOODLIGHT_BRIGHTNESS

drone:SetAttribute(
	"FloodlightEnabled",
	false
)

local function setLaserEnabled(enabled)
	drone:SetAttribute(
		"LaserEnabled",
		enabled == true
	)
end

local function toggleLaser()
	setLaserEnabled(
		drone:GetAttribute(
			"LaserEnabled"
		) ~= true
	)
end

setLaserEnabled(false)

local function setPrecisionMode(enabled)
	enabled =
		enabled == true

	drone:SetAttribute(
		"PrecisionMode",
		enabled
	)

	drone:SetAttribute(
		"FlightProfile",
		enabled
			and "PRECISION"
			or "NORMAL"
	)
end

setPrecisionMode(false)

local currentPilot =
	nil

local MAX_SIGNAL_RANGE =
	350

local SIGNAL_CHECK_INTERVAL =
	0.10

local RECONNECT_DISTANCE =
	MAX_SIGNAL_RANGE * 0.90

local signalTimer =
	0

local signalLost =
	false

local function setSignalAttributes(
	distance,
	strength,
	state,
	connected
)
	drone:SetAttribute(
		"SignalMaxRange",
		MAX_SIGNAL_RANGE
	)

	drone:SetAttribute(
		"SignalDistance",
		distance or 0
	)

	drone:SetAttribute(
		"SignalStrength",
		math.clamp(
			strength or 0,
			0,
			100
		)
	)

	drone:SetAttribute(
		"SignalState",
		state or "NO LINK"
	)

	drone:SetAttribute(
		"SignalConnected",
		connected == true
	)
end

local function calculateSignalStrength(distance)
	if distance <= 175 then
		return 100
			- (distance / 175)
			* 30
	elseif distance <= 260 then
		local alpha =
			(distance - 175)
			/ 85

		return 70
			- alpha
			* 30
	elseif distance <= 315 then
		local alpha =
			(distance - 260)
			/ 55

		return 40
			- alpha
			* 20
	elseif distance < MAX_SIGNAL_RANGE then
		local alpha =
			(distance - 315)
			/
			(MAX_SIGNAL_RANGE - 315)

		return math.max(
			0.1,
			20 - alpha * 20
		)
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

	signalLost =
		true

	setPrecisionMode(false)

	setSignalAttributes(
		MAX_SIGNAL_RANGE,
		0,
		"LOST",
		false
	)

	controller:SetInput(
		0,
		0,
		0,
		0
	)

	setFloodlightEnabled(false)
	setLaserEnabled(false)

	controller:BeginFailsafe()
end

local function updateSignalLink()
	if not currentPilot then
		signalLost =
			false

		setSignalAttributes(
			0,
			0,
			"NO LINK",
			false
		)

		return
	end

	local character =
		currentPilot.Character

	local root =
		character
		and character:FindFirstChild(
			"HumanoidRootPart"
		)

	if not root then
		loseSignal()
		return
	end

	local distance =
		(
			base.Position
			- root.Position
		).Magnitude

	local strength =
		calculateSignalStrength(
			distance
		)

	if signalLost then
		if distance <= RECONNECT_DISTANCE then
			signalLost =
				false

			local state =
				getSignalState(
					strength
				)

			setSignalAttributes(
				distance,
				strength,
				state,
				true
			)
		else
			setSignalAttributes(
				distance,
				0,
				"LOST",
				false
			)
		end

		return
	end

	if distance >= MAX_SIGNAL_RANGE then
		loseSignal()
		return
	end

	local state =
		getSignalState(
			strength
		)

	setSignalAttributes(
		distance,
		strength,
		state,
		true
	)
end

setSignalAttributes(
	0,
	0,
	"NO LINK",
	false
)

drone:GetAttributeChangedSignal(
	"FlightMode"
):Connect(function()
	if drone:GetAttribute(
		"FlightMode"
	) == "PoweredOff" then
		setFloodlightEnabled(false)
		setLaserEnabled(false)
	end
end)

local function isDroneOwner(player)
	if not player then
		return false
	end

	return tonumber(
		drone:GetAttribute(
			"OwnerUserId"
		)
	) == player.UserId
end

local function assignPilot(player)
	if not isDroneOwner(player) then
		return false
	end

	if currentPilot == player then
		pcall(function()
			base:SetNetworkOwner(
				player
			)
		end)

		return true
	end

	if currentPilot ~= nil then
		return false
	end

	currentPilot =
		player

	pcall(function()
		base:SetNetworkOwner(
			player
		)
	end)

	return true
end

local function validNumber(value)
	return typeof(value) == "number"
		and value == value
		and value ~= math.huge
		and value ~= -math.huge
end

droneCommand.OnServerEvent:Connect(
	function(player, command)
		if typeof(command) ~= "string" then
			return
		end

		if command == "BootTakeoff" then
			if not assignPilot(player) then
				return
			end

			setPrecisionMode(false)

			signalLost =
				false

			controller.State.FailsafeTriggered =
				false

			updateSignalLink()

			if drone:GetAttribute(
				"SignalConnected"
			) ~= true then
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
				or drone:GetAttribute(
					"SignalConnected"
				) ~= true then

				return
			end

			local mode =
				controller.State.Mode

			if mode ~= "Armed"
				and mode ~= "Flying" then

				return
			end

			setPrecisionMode(
				drone:GetAttribute(
					"PrecisionMode"
				) ~= true
			)

		elseif command == "ToggleFloodlight" then
			if player ~= currentPilot
				or signalLost
				or drone:GetAttribute(
					"SignalConnected"
				) ~= true then

				return
			end

			toggleFloodlight()

		elseif command == "ToggleLaser" then
			if player ~= currentPilot
				or signalLost
				or drone:GetAttribute(
					"SignalConnected"
				) ~= true then

				return
			end

			toggleLaser()
		end
	end
)

flightInput.OnServerEvent:Connect(
	function(
		player,
		forward,
		strafe,
		vertical,
		roll
	)
		if player ~= currentPilot
			or signalLost then

			return
		end

		if not validNumber(forward)
			or not validNumber(strafe)
			or not validNumber(vertical) then

			return
		end

		if not validNumber(roll) then
			roll =
				0
		end

		controller:SetInput(
			forward,
			strafe,
			vertical,
			roll
		)
	end
)

droneLook.OnServerEvent:Connect(
	function(
		player,
		lookDirection,
		lookUpDirection
	)
		if player ~= currentPilot
			or signalLost then

			return
		end

		if typeof(
			lookDirection
		) ~= "Vector3" then
			return
		end

		if lookDirection.Magnitude < 0.01 then
			return
		end

		if lookUpDirection ~= nil
			and typeof(
				lookUpDirection
			) ~= "Vector3" then

			return
		end

		controller:SetLookDirection(
			lookDirection,
			lookUpDirection
		)
	end
)

Players.PlayerRemoving:Connect(
	function(player)
		if player ~= currentPilot then
			return
		end

		currentPilot =
			nil

		setFloodlightEnabled(false)
		setLaserEnabled(false)

		signalLost =
			false

		setSignalAttributes(
			0,
			0,
			"NO LINK",
			false
		)

		controller:SetInput(
			0,
			0,
			0,
			0
		)

		pcall(function()
			base:SetNetworkOwner(nil)
		end)
	end
)

RunService.Heartbeat:Connect(
	function(deltaTime)
		signalTimer +=
			deltaTime

		if signalTimer
			< SIGNAL_CHECK_INTERVAL then

			return
		end

		signalTimer =
			0

		updateSignalLink()
	end
)
```
