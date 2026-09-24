[← Back to Home](../README.md)

# UAV Flight Controller Physics Module (FlightController)

This module was created to handle the drones manual thrust vectors, it's angular velocities, the torque offsets, the translational flight updates, and it's ground aware landing flare calculation that is manually used to fly the drone's physics.

```lua
local RunService = game:GetService("RunService")

local FlightController = {}
FlightController.__index = FlightController

local DEFAULT_CONFIG = {

	MaxSpeed = 72,
	VerticalSpeed = 30,

	LookResponsiveness = 5.6,
	RollSpeed = math.rad(150),
	MaxAngularVelocity = math.rad(300),
	OrientationResponsiveness = 20,

	AutoBankAngle = math.rad(18),
	AutoPitchAngle = math.rad(18),
	AutoLeanResponsiveness = 6.5,

	VelocityResponse = 3.8,
	Acceleration = 66,
	Deceleration = 76,
	ReverseAcceleration = 86,
	VerticalAcceleration = 46,
	VerticalDeceleration = 58,
	HorizontalJerk = 620,
	VerticalJerk = 420,

	PrecisionMaxSpeed = 22,
	PrecisionVerticalSpeed = 8.5,

	PrecisionLookResponsiveness = 3.6,
	PrecisionRollSpeed = math.rad(55),
	PrecisionMaxAngularVelocity = math.rad(115),

	PrecisionAutoBankAngle = math.rad(7),
	PrecisionAutoPitchAngle = math.rad(7),
	PrecisionAutoLeanResponsiveness = 5.0,

	PrecisionVelocityResponse = 2.5,
	PrecisionAcceleration = 14,
	PrecisionDeceleration = 30,
	PrecisionReverseAcceleration = 34,
	PrecisionVerticalAcceleration = 11,
	PrecisionVerticalDeceleration = 20,

	PrecisionHorizontalJerk = 95,
	PrecisionVerticalJerk = 75,
	PrecisionMaxControlAcceleration = 44,

	LinearDrag = 0.020,
	QuadraticDrag = 0.00048,

	MaxControlAcceleration = 96,
	ForceReserve = 1.25,
	MaxThrustToWeight = 1.72,

	HoverPropellerThrottle = 0.72,
	TakeoffPropellerThrottle = 0.86,
	LandingPropellerThrottle = 0.69,
	ReturnHomePropellerThrottle = 0.82,
	AccelerationRotorBoost = 0.16,
	VerticalRotorBoost = 0.07,
	RollPropellerBoost = 0.055,
	DescentPropellerReduction = 0.035,
	GroundEffectHeight = 6.5,
	GroundEffectThrottleReduction = 0.035,

	HoverDriftSpeed = 0,
	HoverDriftFrequency = 0,

	BootDuration = 3.4,
	MotorStartDelay = 0.45,
	GroundIdlePropellerThrottle = 0.46,
	GroundSpoolDuration = 1.45,
	GroundIdleHoldDuration = 0.45,
	LiftSpoolDuration = 0.90,
	PreLiftHoldDuration = 0.55,

	TakeoffHeight = 4.5,
	TakeoffSpeed = 6.2,
	TakeoffProfileDuration = 1.55,
	TakeoffTrackingGain = 2.8,
	TakeoffCorrectionSpeed = 1.10,
	TakeoffThreshold = 0.28,
	TakeoffSettleVerticalSpeed = 1.00,

	LandingSpeed = 6.5,
	LandingTrackingGain = 2.6,
	LandingMinDuration = 2.0,
	LandingMaxDuration = 40.0,
	LandingThreshold = 0.08,
	LandingTouchdownTolerance = 0.11,
	LandingTouchdownMaxSpeed = 0.42,
	LandingTouchdownConfirmTime = 0.20,
	LandingContactCreepSpeed = 0.05,

	LandingHorizontalGain = 2.2,
	LandingHorizontalMaxSpeed = 5.5,
	LandingHorizontalNaturalFrequency = 2.25,
	LandingHorizontalDampingRatio = 1.35,
	LandingHorizontalMaxAcceleration = 16.0,
	LandingHorizontalAcquireTolerance = 0.10,
	LandingHorizontalAcquireMaxSpeed = 0.28,
	LandingHorizontalFinalHoldHeight = 0.55,
	LandingHorizontalCaptureTolerance = 0.10,
	LandingHorizontalCaptureMaxSpeed = 0.28,
	LandingHorizontalTouchdownTolerance = 0.16,
	LandingHorizontalTouchdownMaxSpeed = 0.32,

	LandingSensorRange = 120,
	LandingProbeSpread = 0.34,
	LandingFlareDeceleration = 3.2,
	LandingFlareMargin = 0.42,
	LandingFinalApproachHeight = 3.00,
	LandingFinalApproachGain = 0.42,
	LandingFinalDescentSpeed = 0.05,
	LandingMaxFinalDescentSpeed = 0.80,

	LandingFinalControllerHeight = 3.0,
	LandingFinalNaturalFrequency = 2.6,
	LandingFinalDampingRatio = 1.20,
	LandingFinalMaxDownAcceleration = 4.5,
	LandingFinalMaxBrakeAcceleration = 22.0,

	LandingCaptureHeight = 0.18,
	LandingSettleSpeed = 0.08,
	LandingSettleLiftRatio = 0.985,

	LandingGroundContactTolerance = 0.075,
	LandingCapturedSettleTimeout = 0.90,
	LandingGroundConfirmTime = 0.12,
	LandingShutdownBounceSpeed = 0.06,

	LandingContactElasticity = 0.0,
	LandingContactElasticityWeight = 100,
	ShutdownDuration = 2.3,

	RotorThrustVisualGain = 1.45,

	GroundRayDistance = 50,
	AutoLevelResponsiveness = 5.5,

	MaxAltitude = 300,
	MinAltitude = 5,

	InputTimeout = 999,
	StartupGracePeriod = 3,

	ReturnHomeSpeed = 32,
	ReturnHomeStopDistance = 0.45,
	ReturnHomeAltitude = 0,
	ReturnHomeBrakeAcceleration = 24,
	ReturnHomePositionGain = 2.4,
	ReturnHomeVerticalGain = 1.35,
	ReturnHomeVerticalSpeed = 8,
	ReturnHomeArrivalSpeed = 1.0,
	TakeoffMaxSettleTime = 1.8,
	HoverSpeedDeadband = 0.10,
}

local function finiteVector(vector)
	return typeof(vector) == "Vector3"
		and vector.X == vector.X
		and vector.Y == vector.Y
		and vector.Z == vector.Z
		and math.abs(vector.X) < math.huge
		and math.abs(vector.Y) < math.huge
		and math.abs(vector.Z) < math.huge
end

local function clampMagnitude(vector, maximum)
	local magnitude = vector.Magnitude

	if magnitude <= maximum or magnitude < 1e-6 then
		return vector
	end

	return vector * (maximum / magnitude)
end

local function moveVectorTowards(current, target, maximumDelta)
	local difference = target - current
	local distance = difference.Magnitude

	if distance <= maximumDelta or distance < 1e-6 then
		return target
	end

	return current + difference * (maximumDelta / distance)
end

local function moveNumberTowards(current, target, maximumDelta)
	local difference = target - current

	if math.abs(difference) <= maximumDelta then
		return target
	end

	return current + math.sign(difference) * maximumDelta
end

local function minimumJerk01(alpha)
	alpha = math.clamp(alpha, 0, 1)
	local a2 = alpha * alpha
	local a3 = a2 * alpha
	return 10 * a3 - 15 * a3 * alpha + 6 * a3 * a2
end

local function minimumJerkDerivative01(alpha)
	alpha = math.clamp(alpha, 0, 1)
	local a2 = alpha * alpha
	local a3 = a2 * alpha
	local a4 = a3 * alpha
	return 30 * a2 - 60 * a3 + 30 * a4
end

function FlightController.new(drone, config)

	assert(
		drone,
		"FlightController requires a drone model"
	)

	local self =
		setmetatable(
			{},
			FlightController
		)

	self.Drone =
		drone

	self.Base =
		drone:WaitForChild("Base")

	self.Config =
		table.clone(
			DEFAULT_CONFIG
		)

	if config then

		for key, value in pairs(config) do

			if self.Config[key] ~= nil then

				self.Config[key] =
					value

			end

		end

	end

	self.Input = {
		Forward = 0,
		Strafe = 0,
		Vertical = 0,
		Roll = 0,
	}

	self.State = {
		Active = false,
		Mode = "PoweredOff",

		Velocity = Vector3.zero,
		TargetVelocity = Vector3.zero,

		FlightTime = 0,

		FailsafeTriggered = false,
	}

	self.ValidStates = {
		PoweredOff = true,
		Booting = true,
		Takeoff = true,

		Armed = true,
		Flying = true,

		ReturnToHome = true,
		Landing = true,
		ShuttingDown = true,
		Failsafe = true,
	}

	self.LastInputTime =
		0

	self.StartTime =
		0

	self.HomePosition =
		nil

	self.HomeGroundY =
		nil

	self.LandingHorizontalTarget =
		nil

	self.RestingClearance =
		0.5

	self.TakeoffTargetY =
		nil

	self.TakeoffStartY =
		nil

	self.TakeoffStartTime =
		nil

	self.LandingTargetY =
		nil

	self.LandingGroundY =
		nil

	self.LandingStartY =
		nil

	self.LandingStartTime =
		nil

	self.LandingProfileDuration =
		nil

	self.TouchdownCandidateTime =
		nil

	self.TouchdownHolding =
		false

	self.LandingContactLatched =
		false

	self.LandingContactLatchTime =
		nil

	self.LandingDescentCommitted =
		false

	self.BootStartTime =
		nil

	self.ShutdownStartTime =
		nil

	self.ShutdownStartThrottle =
		nil

	self.ControlsEnabled =
		false

	self.FlightAttachment =
		nil

	self.FlightForce =
		nil

	self.AeroForce =
		nil

	self.AlignOrientation =
		nil

	self.LandingVelocityDamper =
		nil

	self.AppliedControlAcceleration =
		Vector3.zero

	self.AerodynamicAcceleration =
		Vector3.zero

	self.LookDirection =
		self.Base.CFrame.LookVector

	self.TargetLookDirection =
		self.Base.CFrame.LookVector

	self.LookUpDirection =
		self.Base.CFrame.UpVector

	self.TargetLookUpDirection =
		self.Base.CFrame.UpVector

	self.RollAngle =
		0

	self.AutoBankAngle =
		0

	self.AutoPitchAngle =
		0

	self.CommandAcceleration =
		Vector3.zero

	self.FilteredAcceleration =
		Vector3.zero

	self.LastMeasuredVelocity =
		self.Base.AssemblyLinearVelocity

	self.PhysicsTelemetryTimer =
		0

	self.Connection =
		nil

	return self

end

function FlightController:SetMode(mode)

	if not self.ValidStates[mode] then

		return false

	end

	if self.State.Mode == mode then
		return true
	end

	self.State.Mode =
		mode

	self.Drone:SetAttribute(
		"FlightMode",
		mode
	)

	return true

end

function FlightController:SetPowerState(state)

	self.Drone:SetAttribute(
		"PowerState",
		state
	)

end

function FlightController:SetControlsEnabled(enabled)

	self.ControlsEnabled =
		enabled == true

	self.Drone:SetAttribute(
		"ControlsEnabled",
		self.ControlsEnabled
	)

	if not self.ControlsEnabled then

		self.Input.Forward = 0
		self.Input.Strafe = 0
		self.Input.Vertical = 0
		self.Input.Roll = 0

	end

end

function FlightController:SetPropellerThrottle(value)

	self.Drone:SetAttribute(
		"PropellerThrottle",
		math.clamp(
			tonumber(value) or 0,
			0,
			1
		)
	)

end

function FlightController:GetState()

	return {
		Active = self.State.Active,
		Mode = self.State.Mode,

		Velocity = self.State.Velocity,
		TargetVelocity = self.State.TargetVelocity,

		FlightTime = self.State.FlightTime,

		HomePosition = self.HomePosition,

		FailsafeTriggered =
			self.State.FailsafeTriggered,

		ControlsEnabled =
			self.ControlsEnabled,
	}

end

function FlightController:SetInput(
	forward,
	strafe,
	vertical,
	roll
)

	self.LastInputTime =
		os.clock()

	if not self.ControlsEnabled then

		self.Input.Forward = 0
		self.Input.Strafe = 0
		self.Input.Vertical = 0
		self.Input.Roll = 0

		return

	end

	self.Input.Forward =
		math.clamp(
			tonumber(forward) or 0,
			-1,
			1
		)

	self.Input.Strafe =
		math.clamp(
			tonumber(strafe) or 0,
			-1,
			1
		)

	self.Input.Vertical =
		math.clamp(
			tonumber(vertical) or 0,
			-1,
			1
		)

	self.Input.Roll =
		math.clamp(
			tonumber(roll) or 0,
			-1,
			1
		)

end

function FlightController:SetLookDirection(
	direction,
	upDirection
)

	if not self.ControlsEnabled then
		return
	end

	if typeof(direction) ~= "Vector3" then
		return
	end

	if direction.Magnitude < 0.01 then
		return
	end

	local forward =
		direction.Unit

	self.TargetLookDirection =
		forward

	if typeof(upDirection) == "Vector3"
		and upDirection.Magnitude > 0.01 then

		local right =
			forward:Cross(
				upDirection.Unit
			)

		if right.Magnitude > 0.001 then

			right =
				right.Unit

			self.TargetLookUpDirection =
				right:Cross(
					forward
				).Unit

		end

	end

end

function FlightController:GetInput()

	return {
		Forward = self.Input.Forward,
		Strafe = self.Input.Strafe,
		Vertical = self.Input.Vertical,
		Roll = self.Input.Roll,
	}

end

function FlightController:CreatePhysics()

	if self.FlightForce
		or self.AeroForce
		or self.AlignOrientation then

		return

	end

	self.FlightAttachment =
		Instance.new("Attachment")

	self.FlightAttachment.Name =
		"FlightAttachment"

	self.FlightAttachment.Parent =
		self.Base

	self.FlightForce =
		Instance.new("VectorForce")

	self.FlightForce.Name =
		"FlightThrust"

	self.FlightForce.Attachment0 =
		self.FlightAttachment

	self.FlightForce.RelativeTo =
		Enum.ActuatorRelativeTo.World

	self.FlightForce.ApplyAtCenterOfMass =
		true

	self.FlightForce.Force =
		Vector3.zero

	self.FlightForce.Enabled =
		false

	self.FlightForce.Parent =
		self.Base

	self.AeroForce =
		Instance.new("VectorForce")

	self.AeroForce.Name =
		"AerodynamicDrag"

	self.AeroForce.Attachment0 =
		self.FlightAttachment

	self.AeroForce.RelativeTo =
		Enum.ActuatorRelativeTo.World

	self.AeroForce.ApplyAtCenterOfMass =
		true

	self.AeroForce.Force =
		Vector3.zero

	self.AeroForce.Enabled =
		false

	self.AeroForce.Parent =
		self.Base

	self.AlignOrientation =
		Instance.new("AlignOrientation")

	self.AlignOrientation.Name =
		"FlightStabilizer"

	self.AlignOrientation.Attachment0 =
		self.FlightAttachment

	self.AlignOrientation.Mode =
		Enum.OrientationAlignmentMode.OneAttachment

	self.AlignOrientation.MaxTorque =
		math.huge

	self.AlignOrientation.Responsiveness =
		self.Config.OrientationResponsiveness

	self.AlignOrientation.MaxAngularVelocity =
		self.Config.MaxAngularVelocity

	self.AlignOrientation.RigidityEnabled =
		false

	self.AlignOrientation.CFrame =
		self.Base.CFrame

	self.AlignOrientation.Enabled =
		false

	self.AlignOrientation.Parent =
		self.Base

	self.LandingVelocityDamper =
		Instance.new("LinearVelocity")

	self.LandingVelocityDamper.Name =
		"LandingVelocityDamper"

	self.LandingVelocityDamper.Attachment0 =
		self.FlightAttachment

	self.LandingVelocityDamper.RelativeTo =
		Enum.ActuatorRelativeTo.World

	self.LandingVelocityDamper.VelocityConstraintMode =
		Enum.VelocityConstraintMode.Line

	self.LandingVelocityDamper.LineDirection =
		Vector3.yAxis

	self.LandingVelocityDamper.LineVelocity =
		0

	self.LandingVelocityDamper.MaxForce =
		math.huge

	self.LandingVelocityDamper.Enabled =
		false

	self.LandingVelocityDamper.Parent =
		self.Base

end

function FlightController:ConfigureLandingContactPhysics()

	for _, descendant in ipairs(
		self.Drone:GetDescendants()
	) do

		if descendant:IsA("BasePart")
			and descendant.CanCollide then

			pcall(function()

				local current =
					descendant.CurrentPhysicalProperties

				descendant.CustomPhysicalProperties =
					PhysicalProperties.new(
						current.Density,
						current.Friction,
						self.Config.LandingContactElasticity,
						current.FrictionWeight,
						self.Config.LandingContactElasticityWeight
					)

			end)

		end

	end

end

function FlightController:GetGroundInfo()

	local params =
		RaycastParams.new()

	params.FilterType =
		Enum.RaycastFilterType.Exclude

	params.FilterDescendantsInstances = {
		self.Drone,
	}

	params.IgnoreWater =
		false

	local result =
		workspace:Raycast(
			self.Base.Position,
			Vector3.new(
				0,
				-self.Config.GroundRayDistance,
				0
			),
			params
		)

	if not result then
		return nil, nil
	end

	local distance =
		self.Base.Position.Y
		-
		result.Position.Y

	return
		result.Position.Y,
		distance

end

function FlightController:GetLandingGroundInfo()

	local params =
		RaycastParams.new()

	params.FilterType =
		Enum.RaycastFilterType.Exclude

	params.FilterDescendantsInstances = {
		self.Drone
	}

	params.IgnoreWater =
		false

	local base =
		self.Base

	local spreadX =
		math.max(
			base.Size.X
			*
			self.Config.LandingProbeSpread,
			0.18
		)

	local spreadZ =
		math.max(
			base.Size.Z
			*
			self.Config.LandingProbeSpread,
			0.18
		)

	local right =
		base.CFrame.RightVector

	local forward =
		base.CFrame.LookVector

	local origins = {
		base.Position,
		base.Position + right * spreadX + forward * spreadZ,
		base.Position - right * spreadX + forward * spreadZ,
		base.Position + right * spreadX - forward * spreadZ,
		base.Position - right * spreadX - forward * spreadZ,
	}

	local bestGroundY =
		nil

	local bestResult =
		nil

	local closestClearance =
		math.huge

	local rayDistance =
		math.max(
			self.Config.LandingSensorRange
				or self.Config.GroundRayDistance,
			self.Config.GroundRayDistance
		)

	for _, origin in ipairs(origins) do

		local result =
			workspace:Raycast(
				origin,
				Vector3.new(
					0,
					-rayDistance,
					0
				),
				params
			)

		if result then

			local clearance =
				base.Position.Y
				-
				result.Position.Y

			if clearance >= 0
				and clearance < closestClearance then

				closestClearance =
					clearance

				bestGroundY =
					result.Position.Y

				bestResult =
					result

			end

		end

	end

	if not bestGroundY then
		return nil, nil, nil
	end

	return
		bestGroundY,
		closestClearance,
		bestResult

end

function FlightController:GetGroundInfoAt(
	position,
	rayHeight
)

	if typeof(position) ~= "Vector3" then
		return nil, nil
	end

	local params =
		RaycastParams.new()

	params.FilterType =
		Enum.RaycastFilterType.Exclude

	params.FilterDescendantsInstances = {
		self.Drone
	}

	params.IgnoreWater =
		false

	local probeHeight =
		math.max(
			rayHeight or 0,
			self.Config.GroundRayDistance + 20
		)

	local origin =
		Vector3.new(
			position.X,
			position.Y + probeHeight,
			position.Z
		)

	local result =
		workspace:Raycast(
			origin,
			Vector3.new(
				0,
				-(probeHeight * 2),
				0
			),
			params
		)

	if not result then
		return nil, nil
	end

	return
		result.Position.Y,
		result

end

function FlightController:RequestTakeoff()

	if not self.State.Active then
		return false
	end

	if self.State.Mode ~= "PoweredOff" then
		return false
	end

	local groundY, clearance =
		self:GetGroundInfo()

	if groundY
		and clearance then

		self.RestingClearance =
			math.max(
				clearance,
				0.1
			)

	else

		self.RestingClearance =
			0.5

	end

	self.HomeGroundY =
		groundY
		or (
			self.Base.Position.Y
			-
			self.RestingClearance
		)

	self.HomePosition =
		Vector3.new(
			self.Base.Position.X,
			self.HomeGroundY
				+
				self.RestingClearance,
			self.Base.Position.Z
		)

	self.Drone:SetAttribute(
		"DroneHomePosition",
		self.HomePosition
	)

	self.Drone:SetAttribute(
		"DroneHomeGroundY",
		self.HomeGroundY
	)

	self.Drone:SetAttribute(
		"RTHPhase",
		"STANDBY"
	)

	self.TakeoffTargetY =
		nil

	self.BootStartTime =
		os.clock()

	self.ShutdownStartThrottle =
		nil

	self.TakeoffStartY =
		nil

	self.TakeoffStartTime =
		nil

	self.LandingGroundY =
		nil

	self.LandingStartY =
		nil

	self.LandingStartTime =
		nil

	self.LandingProfileDuration =
		nil

	self.TouchdownCandidateTime =
		nil

	self.TouchdownHolding =
		false

	self.State.Velocity =
		Vector3.zero

	self.State.TargetVelocity =
		Vector3.zero

	self.LookDirection =
		self.Base.CFrame.LookVector

	self.TargetLookDirection =
		self.Base.CFrame.LookVector

	self.LookUpDirection =
		self.Base.CFrame.UpVector

	self.TargetLookUpDirection =
		self.Base.CFrame.UpVector

	self.RollAngle =
		0

	self.AutoBankAngle =
		0

	self.AutoPitchAngle =
		0

	self:SetControlsEnabled(
		false
	)

	self:SetPropellerThrottle(
		0
	)

	self:SetPowerState(
		"Booting"
	)

	self:SetMode(
		"Booting"
	)

	return true

end

function FlightController:StartLandingProfile(
	groundY,
	horizontalTarget
)

	if typeof(groundY) ~= "number" then
		return false
	end

	self.LandingGroundY =
		groundY

	self.LandingTargetY =
		groundY
		+
		self.RestingClearance

	local target =
		horizontalTarget

	if typeof(target) ~= "Vector3" then
		target =
			self.Base.Position
	end

	self.LandingHorizontalTarget =
		Vector3.new(
			target.X,
			0,
			target.Z
		)

	self.LandingStartY =
		self.Base.Position.Y

	self.LandingStartTime =
		os.clock()

	self.TouchdownCandidateTime =
		nil

	self.TouchdownHolding =
		false

	self.LandingContactLatched =
		false

	self.LandingContactLatchTime =
		nil

	self.LandingDescentCommitted =
		false

	self.LandingMeasuredClearance =
		nil

	self.LandingHeightAboveRest =
		nil

	self.Drone:SetAttribute(
		"LandingTargetPosition",
		Vector3.new(
			self.LandingHorizontalTarget.X,
			self.LandingTargetY,
			self.LandingHorizontalTarget.Z
		)
	)

	local distance =
		math.max(
			self.LandingStartY
			-
			self.LandingTargetY,
			0
		)

	local durationFromSpeed =
		1.875
		*
		distance
		/
		math.max(
			self.Config.LandingSpeed,
			0.1
		)

	self.LandingProfileDuration =
		math.clamp(
			durationFromSpeed,
			self.Config.LandingMinDuration,
			self.Config.LandingMaxDuration
		)

	return true

end

function FlightController:RequestLanding()

	if not self.State.Active then
		return false
	end

	if self.State.Mode ~= "Armed"
		and self.State.Mode ~= "Flying" then

		return false

	end

	local groundY =
		self:GetGroundInfo()

	if not groundY then

		groundY =
			self.Base.Position.Y
			-
			self.Config.TakeoffHeight

	end

	self:StartLandingProfile(
		groundY,
		Vector3.new(
			self.Base.Position.X,
			0,
			self.Base.Position.Z
		)
	)

	self:SetControlsEnabled(
		false
	)

	self:SetPowerState(
		"Landing"
	)

	self:SetMode(
		"Landing"
	)

	return true

end

function FlightController:UpdateBooting()

	if self.State.Mode ~= "Booting" then
		return
	end

	local elapsed =
		os.clock()
		-
		(
			self.BootStartTime
			or os.clock()
		)

	local config =
		self.Config

	local tArm =
		math.max(
			config.MotorStartDelay,
			0
		)

	local tGroundSpoolEnd =
		tArm
		+
		math.max(
			config.GroundSpoolDuration,
			0.05
		)

	local tGroundHoldEnd =
		tGroundSpoolEnd
		+
		math.max(
			config.GroundIdleHoldDuration,
			0
		)

	local tLiftSpoolEnd =
		tGroundHoldEnd
		+
		math.max(
			config.LiftSpoolDuration,
			0.05
		)

	local tLiftHoldEnd =
		tLiftSpoolEnd
		+
		math.max(
			config.PreLiftHoldDuration,
			0
		)

	if elapsed < tArm then

		self:SetPropellerThrottle(
			0
		)

		return

	end

	if elapsed < tGroundSpoolEnd then

		local alpha =
			(elapsed - tArm)
			/
			math.max(
				config.GroundSpoolDuration,
				0.05
			)

		self:SetPropellerThrottle(
			minimumJerk01(alpha)
			*
			config.GroundIdlePropellerThrottle
		)

		return

	end

	if elapsed < tGroundHoldEnd then

		self:SetPropellerThrottle(
			config.GroundIdlePropellerThrottle
		)

		return

	end

	if elapsed < tLiftSpoolEnd then

		local alpha =
			(elapsed - tGroundHoldEnd)
			/
			math.max(
				config.LiftSpoolDuration,
				0.05
			)

		local blend =
			minimumJerk01(alpha)

		local throttle =
			config.GroundIdlePropellerThrottle
			+
			(
				config.TakeoffPropellerThrottle
				-
				config.GroundIdlePropellerThrottle
			)
			*
			blend

		self:SetPropellerThrottle(
			throttle
		)

		return

	end

	self:SetPropellerThrottle(
		config.TakeoffPropellerThrottle
	)

	if elapsed < tLiftHoldEnd then
		return
	end

	if self.FlightForce then

		local mass =
			math.max(
				self.Base.AssemblyMass,
				0.001
			)

		self.FlightForce.Force =
			Vector3.new(
				0,
				mass * workspace.Gravity,
				0
			)

		self.FlightForce.Enabled =
			true

	end

	if self.AeroForce then

		self.AeroForce.Force =
			Vector3.zero

		self.AeroForce.Enabled =
			true

	end

	if self.AlignOrientation then

		self.AlignOrientation.CFrame =
			self.Base.CFrame

		self.AlignOrientation.Enabled =
			true

	end

	local liftoffGroundY,
		liftoffClearance =
		self:GetGroundInfo()

	if liftoffGroundY
		and liftoffClearance then

		self.RestingClearance =
			math.max(
				liftoffClearance,
				0.1
			)

		self.HomeGroundY =
			liftoffGroundY

	else

		self.HomeGroundY =
			self.HomeGroundY
			or (
				self.Base.Position.Y
				-
				self.RestingClearance
			)

	end

	self.HomePosition =
		Vector3.new(
			self.Base.Position.X,
			self.HomeGroundY
				+
				self.RestingClearance,
			self.Base.Position.Z
		)

	self.Drone:SetAttribute(
		"DroneHomePosition",
		self.HomePosition
	)

	self.Drone:SetAttribute(
		"DroneHomeGroundY",
		self.HomeGroundY
	)

	self.State.Velocity =
		Vector3.zero

	self.State.TargetVelocity =
		Vector3.zero

	self.CommandAcceleration =
		Vector3.zero

	self.LastMeasuredVelocity =
		self.Base.AssemblyLinearVelocity

	self.TakeoffStartY =
		self.Base.Position.Y

	self.TakeoffTargetY =
		self.TakeoffStartY
		+
		self.Config.TakeoffHeight

	self.TakeoffStartTime =
		os.clock()

	self.CommandAcceleration =
		Vector3.zero

	self.FilteredAcceleration =
		Vector3.zero

	self:SetPowerState(
		"TakingOff"
	)

	self:SetMode(
		"Takeoff"
	)

end

function FlightController:CalculateTakeoffVelocity()

	if not self.TakeoffTargetY
		or not self.TakeoffStartY
		or not self.TakeoffStartTime then

		return Vector3.zero

	end

	local duration =
		math.max(
			self.Config.TakeoffProfileDuration,
			0.1
		)

	local elapsed =
		os.clock()
		-
		self.TakeoffStartTime

	local alpha =
		math.clamp(
			elapsed / duration,
			0,
			1
		)

	local travel =
		self.TakeoffTargetY
		-
		self.TakeoffStartY

	local desiredPosition =
		self.TakeoffStartY
		+
		travel
		*
		minimumJerk01(
			alpha
		)

	local feedForwardVelocity =
		travel
		*
		minimumJerkDerivative01(
			alpha
		)
		/
		duration

	local positionError =
		desiredPosition
		-
		self.Base.Position.Y

	local commandedY =
		feedForwardVelocity
		+
		positionError
		*
		self.Config.TakeoffTrackingGain

	if alpha >= 1 then

		local finalError =
			self.TakeoffTargetY
			-
			self.Base.Position.Y

		commandedY =
			math.clamp(
				finalError
				*
				self.Config.TakeoffTrackingGain,
				-self.Config.TakeoffCorrectionSpeed,
				self.Config.TakeoffCorrectionSpeed
			)

	end

	commandedY =
		math.clamp(
			commandedY,
			-self.Config.TakeoffCorrectionSpeed,
			self.Config.TakeoffSpeed
		)

	return
		Vector3.new(
			0,
			commandedY,
			0
		)

end

function FlightController:UpdateTakeoff()

	if self.State.Mode ~= "Takeoff" then
		return
	end

	if not self.TakeoffTargetY
		or not self.TakeoffStartTime then

		return

	end

	local elapsed =
		os.clock()
		-
		self.TakeoffStartTime

	local profileFinished =
		elapsed
		>=
		self.Config.TakeoffProfileDuration

	local positionError =
		self.TakeoffTargetY
		-
		self.Base.Position.Y

	local verticalSpeed =
		math.abs(
			self.Base.AssemblyLinearVelocity.Y
		)

	local captured =
		profileFinished
		and math.abs(positionError)
			<=
			self.Config.TakeoffThreshold
		and verticalSpeed
			<=
			self.Config.TakeoffSettleVerticalSpeed

	local settleTimedOut =
		elapsed
			>=
			self.Config.TakeoffProfileDuration
			+
			self.Config.TakeoffMaxSettleTime
		and math.abs(positionError) <= 0.65
		and verticalSpeed <= 1.6

	if not captured
		and not settleTimedOut then

		return

	end

	self.State.Velocity =
		self.Base.AssemblyLinearVelocity

	self.State.TargetVelocity =
		Vector3.zero

	self.CommandAcceleration =
		Vector3.zero

	self.FilteredAcceleration =
		Vector3.zero

	self.LastMeasuredVelocity =
		self.Base.AssemblyLinearVelocity

	self:SetPowerState(
		"Ready"
	)

	self:SetPropellerThrottle(
		self.Config.HoverPropellerThrottle
	)

	self:SetControlsEnabled(
		true
	)

	self:SetMode(
		"Armed"
	)

end

function FlightController:CalculateLandingVelocity()

	if not self.LandingTargetY then
		return Vector3.zero
	end

	if self.TouchdownHolding then
		return Vector3.zero
	end

	local groundY, clearance =
		self:GetLandingGroundInfo()

	if not groundY
		or not clearance then

		groundY =
			self.LandingGroundY

		if groundY then

			clearance =
				self.Base.Position.Y
				-
				groundY

		else

			clearance =
				math.max(
					self.Base.Position.Y
					-
					self.LandingTargetY,
					self.RestingClearance
				)

		end

	else

		self.LandingGroundY =
			groundY

		self.LandingTargetY =
			groundY
			+
			self.RestingClearance

	end

	local heightAboveRest =
		math.max(
			clearance
			-
			self.RestingClearance,
			0
		)

	self.LandingMeasuredClearance =
		clearance

	self.LandingHeightAboveRest =
		heightAboveRest

	local usableDistance =
		math.max(
			heightAboveRest
			-
			self.Config.LandingFlareMargin
			-
			self.Config.LandingTouchdownTolerance,
			0
		)

	local safeDescentSpeed =
		math.sqrt(
			2
			*
			self.Config.LandingFlareDeceleration
			*
			usableDistance
		)

	local descentSpeed =
		math.min(
			self.Config.LandingSpeed,
			safeDescentSpeed
		)

	if heightAboveRest
		<=
		self.Config.LandingFinalApproachHeight then

		local finalSpeed =
			self.Config.LandingFinalDescentSpeed
			+
			math.max(
				heightAboveRest
				-
				self.Config.LandingTouchdownTolerance,
				0
			)
			*
			self.Config.LandingFinalApproachGain

		descentSpeed =
			math.min(
				descentSpeed,
				self.Config.LandingMaxFinalDescentSpeed,
				finalSpeed
			)

	end

	if heightAboveRest
		>
		self.Config.LandingTouchdownTolerance then

		descentSpeed =
			math.max(
				descentSpeed,
				self.Config.LandingFinalDescentSpeed
			)

	else

		descentSpeed =
			0

	end

	local commandedY =
		-descentSpeed

	local horizontalVelocity =
		Vector3.zero

	if self.LandingHorizontalTarget then

		local horizontalErrorVector =
			self.LandingHorizontalTarget
			-
			Vector3.new(
				self.Base.Position.X,
				0,
				self.Base.Position.Z
			)

		local horizontalSpeed =
			Vector3.new(
				self.Base.AssemblyLinearVelocity.X,
				0,
				self.Base.AssemblyLinearVelocity.Z
			).Magnitude

		if not self.LandingDescentCommitted then

			if horizontalErrorVector.Magnitude
				<=
				self.Config.LandingHorizontalAcquireTolerance
				and horizontalSpeed
				<=
				self.Config.LandingHorizontalAcquireMaxSpeed then

				self.LandingDescentCommitted =
					true

			else

				commandedY =
					0

			end

		end

		if heightAboveRest
			<=
			self.Config.LandingHorizontalFinalHoldHeight
			and (
				horizontalErrorVector.Magnitude
					>
					self.Config.LandingHorizontalCaptureTolerance
				or horizontalSpeed
					>
					self.Config.LandingHorizontalCaptureMaxSpeed
			) then

			commandedY =
				0

		end

	end

	return
		Vector3.new(
			horizontalVelocity.X,
			commandedY,
			horizontalVelocity.Z
		)

end

function FlightController:UpdateLanding()

	if self.State.Mode ~= "Landing" then
		return
	end

	if not self.LandingTargetY then
		return
	end

	local groundY, clearance =
		self:GetLandingGroundInfo()

	if not groundY
		or not clearance then

		if self.LandingContactLatched
			and self.LandingGroundY then

			groundY =
				self.LandingGroundY

			clearance =
				self.Base.Position.Y
				-
				groundY

		else

			self.TouchdownCandidateTime =
				nil

			return

		end

	end

	self.LandingGroundY =
		groundY

	self.LandingTargetY =
		groundY
		+
		self.RestingClearance

	local heightAboveRest =
		clearance
		-
		self.RestingClearance

	self.LandingMeasuredClearance =
		clearance

	self.LandingHeightAboveRest =
		math.max(
			heightAboveRest,
			0
		)

	local velocity =
		self.Base.AssemblyLinearVelocity

	local horizontalSpeed =
		Vector3.new(
			velocity.X,
			0,
			velocity.Z
		).Magnitude

	local horizontalError =
		0

	if self.LandingHorizontalTarget then

		horizontalError =
			(
				self.LandingHorizontalTarget
				-
				Vector3.new(
					self.Base.Position.X,
					0,
					self.Base.Position.Z
				)
			).Magnitude

	end

	if not self.LandingContactLatched
		and heightAboveRest
			<=
			self.Config.LandingCaptureHeight
		and horizontalError
			<=
			self.Config.LandingHorizontalCaptureTolerance
		and horizontalSpeed
			<=
			self.Config.LandingHorizontalCaptureMaxSpeed then

		self.LandingContactLatched =
			true

		self.LandingContactLatchTime =
			os.clock()

		self.TouchdownHolding =
			true

		self.TouchdownCandidateTime =
			nil

		self.CommandAcceleration =
			Vector3.zero

		local current =
			self.Base.AssemblyLinearVelocity

		self.Base.AssemblyLinearVelocity =
			Vector3.new(
				current.X,
				math.clamp(
					current.Y,
					-self.Config.LandingSettleSpeed,
					0
				),
				current.Z
			)

		if self.LandingVelocityDamper then

			self.LandingVelocityDamper.LineVelocity =
				-self.Config.LandingSettleSpeed

			self.LandingVelocityDamper.Enabled =
				true

		end

	end

	if not self.LandingContactLatched then
		return
	end

	self.TouchdownHolding =
		true

	local current =
		self.Base.AssemblyLinearVelocity

	local dampedY =
		math.clamp(
			current.Y,
			-self.Config.LandingSettleSpeed,
			0
		)

	self.Base.AssemblyLinearVelocity =
		Vector3.new(
			current.X,
			dampedY,
			current.Z
		)

	local onGroundPlane =
		heightAboveRest
		<=
		self.Config.LandingGroundContactTolerance

	if self.LandingVelocityDamper then

		self.LandingVelocityDamper.Enabled =
			true

		self.LandingVelocityDamper.LineVelocity =
			onGroundPlane
			and 0
			or -self.Config.LandingSettleSpeed

	end

	local horizontallySettled =
		horizontalError
			<=
			self.Config.LandingHorizontalTouchdownTolerance
		and horizontalSpeed
			<=
			self.Config.LandingHorizontalTouchdownMaxSpeed

	local capturedLongEnough =
		self.LandingContactLatchTime ~= nil
		and (
			os.clock()
			-
			self.LandingContactLatchTime
		)
		>=
		(
			self.Config.LandingCapturedSettleTimeout
			or 0.90
		)

	local settledInsideCapture =
		heightAboveRest
			<=
			self.Config.LandingCaptureHeight
		and math.abs(current.Y)
			<=
			(
				self.Config.LandingSettleSpeed
				+
				0.02
			)

	local touchdownConfirmedByGeometry =
		onGroundPlane
		or (
			capturedLongEnough
			and settledInsideCapture
		)

	if not touchdownConfirmedByGeometry
		or not horizontallySettled then

		self.TouchdownCandidateTime =
			nil

		return

	end

	if not self.TouchdownCandidateTime then

		self.TouchdownCandidateTime =
			os.clock()

		return

	end

	if os.clock()
		-
		self.TouchdownCandidateTime
		<
		self.Config.LandingGroundConfirmTime then

		return

	end

	self.ShutdownStartTime =
		os.clock()

	self.ShutdownStartThrottle =
		tonumber(
			self.Drone:GetAttribute(
				"PropellerThrottle"
			)
		)
		or self.Config.LandingPropellerThrottle

	self:SetPowerState(
		"ShuttingDown"
	)

	self:SetMode(
		"ShuttingDown"
	)

end

function FlightController:UpdateShuttingDown()

	if self.State.Mode ~= "ShuttingDown" then
		return
	end

	local elapsed =
		os.clock()
		-
		(
			self.ShutdownStartTime
			or os.clock()
		)

	local alpha =
		math.clamp(
			elapsed
			/
			math.max(
				self.Config.ShutdownDuration,
				0.1
			),
			0,
			1
		)

	local shutdownThrottle =
		self.ShutdownStartThrottle
		or self.Config.HoverPropellerThrottle

	if self.LandingContactLatched then

		if self.LandingVelocityDamper then

			self.LandingVelocityDamper.Enabled =
				true

			self.LandingVelocityDamper.LineVelocity =
				0

		end

		local current =
			self.Base.AssemblyLinearVelocity

		self.Base.AssemblyLinearVelocity =
			Vector3.new(
				current.X,
				math.clamp(
					current.Y,
					-self.Config.LandingShutdownBounceSpeed,
					0
				),
				current.Z
			)

	end

	local spoolDown =
		1
		-
		minimumJerk01(
			alpha
		)

	self:SetPropellerThrottle(
		shutdownThrottle
		*
		spoolDown
	)

	local mass =
		math.max(
			self.Base.AssemblyMass,
			0.001
		)

	if self.FlightForce then

		self.FlightForce.Enabled =
			true

		self.FlightForce.Force =
			Vector3.new(
				0,
				mass
					*
					workspace.Gravity
					*
					spoolDown,
				0
			)

	end

	if self.AeroForce then

		self.AeroForce.Enabled =
			true

		self.AeroForce.Force =
			Vector3.zero

	end

	if alpha < 1 then
		return
	end

	self:SetPropellerThrottle(
		0
	)

	if self.FlightForce then

		self.FlightForce.Force =
			Vector3.zero

		self.FlightForce.Enabled =
			false

	end

	if self.AeroForce then

		self.AeroForce.Force =
			Vector3.zero

		self.AeroForce.Enabled =
			false

	end

	if self.AlignOrientation then

		self.AlignOrientation.Enabled =
			false

	end

	if self.LandingVelocityDamper then

		self.LandingVelocityDamper.LineVelocity =
			0

		self.LandingVelocityDamper.Enabled =
			false

	end

	self.State.Velocity =
		Vector3.zero

	self.State.TargetVelocity =
		Vector3.zero

	self.CommandAcceleration =
		Vector3.zero

	self.AppliedControlAcceleration =
		Vector3.zero

	self.AerodynamicAcceleration =
		Vector3.zero

	self:SetControlsEnabled(
		false
	)

	self:SetPowerState(
		"Off"
	)

	self:SetMode(
		"PoweredOff"
	)

	self.BootStartTime =
		nil

	self.ShutdownStartTime =
		nil

	self.ShutdownStartThrottle =
		nil

	self.TakeoffTargetY =
		nil

	self.TakeoffStartY =
		nil

	self.TakeoffStartTime =
		nil

	self.LandingTargetY =
		nil

	self.LandingGroundY =
		nil

	self.LandingStartY =
		nil

	self.LandingStartTime =
		nil

	self.LandingProfileDuration =
		nil

	self.LandingHorizontalTarget =
		nil

	self.TouchdownCandidateTime =
		nil

	self.TouchdownHolding =
		false

	self.LandingContactLatched =
		false

	self.LandingContactLatchTime =
		nil

	self.LandingDescentCommitted =
		false

	self.Drone:SetAttribute(
		"LandingTargetPosition",
		nil
	)

	self.Drone:SetAttribute(
		"RTHPhase",
		"COMPLETE"
	)

	self.Drone:SetAttribute(
		"RTHDistance",
		0
	)

end

function FlightController:LevelForAutomaticFlight(
	deltaTime
)

	local alpha =
		1
		-
		math.exp(
			-self.Config.AutoLevelResponsiveness
			*
			deltaTime
		)

	local normalizedRoll =
		math.atan2(
			math.sin(
				self.RollAngle
			),
			math.cos(
				self.RollAngle
			)
		)

	self.RollAngle =
		normalizedRoll
		+
		(
			0
			-
			normalizedRoll
		)
		*
		alpha

	self.AutoBankAngle +=
		(
			0
			-
			self.AutoBankAngle
		)
		*
		alpha

	self.AutoPitchAngle +=
		(
			0
			-
			self.AutoPitchAngle
		)
		*
		alpha

	local horizontalLook =
		Vector3.new(
			self.LookDirection.X,
			0,
			self.LookDirection.Z
		)

	if horizontalLook.Magnitude > 0.01 then

		self.LookDirection =
			horizontalLook.Unit

		self.TargetLookDirection =
			horizontalLook.Unit

	end

	self.LookUpDirection =
		self.LookUpDirection:Lerp(
			Vector3.yAxis,
			alpha
		).Unit

	self.TargetLookUpDirection =
		Vector3.yAxis

end

function FlightController:GetHomeDistance()

	if not self.HomePosition then
		return math.huge
	end

	return
		(
			self.Base.Position
			-
			self.HomePosition
		).Magnitude

end

function FlightController:CalculateReturnHomeVelocity()

	if not self.HomePosition then
		return Vector3.zero
	end

	local position =
		self.Base.Position

	local horizontalOffset =
		Vector3.new(
			self.HomePosition.X
				-
				position.X,
			0,
			self.HomePosition.Z
				-
				position.Z
		)

	local distance =
		horizontalOffset.Magnitude

	local horizontalVelocity =
		Vector3.zero

	if distance > 0.001 then

		local direction =
			horizontalOffset.Unit

		local brakingSpeed =
			math.sqrt(
				2
				*
				self.Config.ReturnHomeBrakeAcceleration
				*
				distance
			)

		local positionSpeed =
			distance
			*
			self.Config.ReturnHomePositionGain

		local speed =
			math.min(
				self.Config.ReturnHomeSpeed,
				brakingSpeed,
				positionSpeed
			)

		horizontalVelocity =
			direction
			*
			speed

		if distance
			>
			self.Config.ReturnHomeStopDistance then

			self.TargetLookDirection =
				direction

			self.TargetLookUpDirection =
				Vector3.yAxis

		end

	end

	local holdAltitude =
		self.RTHAltitude
		or position.Y

	local altitudeError =
		holdAltitude
		-
		position.Y

	local verticalVelocity =
		math.clamp(
			altitudeError
			*
			self.Config.ReturnHomeVerticalGain,
			-self.Config.ReturnHomeVerticalSpeed,
			self.Config.ReturnHomeVerticalSpeed
		)

	self.Drone:SetAttribute(
		"RTHDistance",
		distance
	)

	self.Drone:SetAttribute(
		"RTHPhase",
		distance > 8
			and "RETURN"
			or (
				distance
					>
					self.Config.ReturnHomeStopDistance
				and "APPROACH"
				or "HOME HOLD"
			)
	)

	return
		Vector3.new(
			horizontalVelocity.X,
			verticalVelocity,
			horizontalVelocity.Z
		)

end

function FlightController:IsPrecisionMode()

	if self.Drone:GetAttribute(
		"PrecisionMode"
	) ~= true then

		return false

	end

	return
		self.ControlsEnabled
		and (
			self.State.Mode == "Armed"
			or self.State.Mode == "Flying"
		)

end

function FlightController:GetManualFlightTuning()

	local config =
		self.Config

	if self:IsPrecisionMode() then

		return {
			Precision = true,
			MaxSpeed = config.PrecisionMaxSpeed,
			VerticalSpeed = config.PrecisionVerticalSpeed,
			LookResponsiveness = config.PrecisionLookResponsiveness,
			RollSpeed = config.PrecisionRollSpeed,
			MaxAngularVelocity = config.PrecisionMaxAngularVelocity,
			AutoBankAngle = config.PrecisionAutoBankAngle,
			AutoPitchAngle = config.PrecisionAutoPitchAngle,
			AutoLeanResponsiveness = config.PrecisionAutoLeanResponsiveness,
			VelocityResponse = config.PrecisionVelocityResponse,
			Acceleration = config.PrecisionAcceleration,
			Deceleration = config.PrecisionDeceleration,
			ReverseAcceleration = config.PrecisionReverseAcceleration,
			VerticalAcceleration = config.PrecisionVerticalAcceleration,
			VerticalDeceleration = config.PrecisionVerticalDeceleration,
			HorizontalJerk = config.PrecisionHorizontalJerk,
			VerticalJerk = config.PrecisionVerticalJerk,
			MaxControlAcceleration = config.PrecisionMaxControlAcceleration,
		}

	end

	return {
		Precision = false,
		MaxSpeed = config.MaxSpeed,
		VerticalSpeed = config.VerticalSpeed,
		LookResponsiveness = config.LookResponsiveness,
		RollSpeed = config.RollSpeed,
		MaxAngularVelocity = config.MaxAngularVelocity,
		AutoBankAngle = config.AutoBankAngle,
		AutoPitchAngle = config.AutoPitchAngle,
		AutoLeanResponsiveness = config.AutoLeanResponsiveness,
		VelocityResponse = config.VelocityResponse,
		Acceleration = config.Acceleration,
		Deceleration = config.Deceleration,
		ReverseAcceleration = config.ReverseAcceleration,
		VerticalAcceleration = config.VerticalAcceleration,
		VerticalDeceleration = config.VerticalDeceleration,
		HorizontalJerk = config.HorizontalJerk,
		VerticalJerk = config.VerticalJerk,
		MaxControlAcceleration = config.MaxControlAcceleration,
	}

end

function FlightController:CalculateNormalVelocity()

	local config =
		self.Config

	local input =
		self.Input

	local tuning =
		self:GetManualFlightTuning()

	local forward =
		self.LookDirection

	if not forward
		or forward.Magnitude < 0.01 then

		forward =
			self.Base.CFrame.LookVector

	end

	forward =
		forward.Unit

	local up =
		self.LookUpDirection

	if not up
		or up.Magnitude < 0.01 then

		up =
			self.Base.CFrame.UpVector

	end

	local right =
		forward:Cross(
			up.Unit
		)

	if right.Magnitude > 0.01 then

		right =
			right.Unit

	else

		right =
			self.Base.CFrame.RightVector

	end

	local direction =
		forward
		*
		input.Forward
		+
		right
		*
		input.Strafe

	if direction.Magnitude > 1 then

		direction =
			direction.Unit

	end

	local targetVelocity =
		direction
		*
		tuning.MaxSpeed

	targetVelocity +=
		Vector3.new(
			0,
			input.Vertical
				*
				tuning.VerticalSpeed,
			0
		)

	local altitude =
		self.Base.Position.Y

	if altitude >= config.MaxAltitude
		and targetVelocity.Y > 0 then

		targetVelocity =
			Vector3.new(
				targetVelocity.X,
				0,
				targetVelocity.Z
			)

	end

	if altitude <= config.MinAltitude
		and targetVelocity.Y < 0 then

		targetVelocity =
			Vector3.new(
				targetVelocity.X,
				0,
				targetVelocity.Z
			)

	end

	return
		targetVelocity

end

function FlightController:CalculateTargetVelocity()

	if self.State.Mode == "PoweredOff"
		or self.State.Mode == "Booting"
		or self.State.Mode == "ShuttingDown" then

		return
			Vector3.zero

	end

	if self.State.Mode == "Takeoff" then

		return
			self:CalculateTakeoffVelocity()

	end

	if self.State.Mode == "Landing" then

		return
			self:CalculateLandingVelocity()

	end

	if self.State.Mode == "ReturnToHome" then

		return
			self:CalculateReturnHomeVelocity()

	end

	if self.State.Mode == "Failsafe" then

		return
			Vector3.zero

	end

	return
		self:CalculateNormalVelocity()

end

function FlightController:UpdateLookOrientation(
	deltaTime
)

	local tuning =
		self:GetManualFlightTuning()

	if not self.AlignOrientation
		or not self.AlignOrientation.Enabled then

		return

	end

	if not self.TargetLookDirection then
		return
	end

	if self.State.Mode == "Takeoff"
		or self.State.Mode == "Landing" then

		self:LevelForAutomaticFlight(
			deltaTime
		)

	end

	local targetForward =
		self.TargetLookDirection

	if targetForward.Magnitude < 0.01 then
		return
	end

	targetForward =
		targetForward.Unit

	local targetUp =
		self.TargetLookUpDirection
		or Vector3.yAxis

	if targetUp.Magnitude < 0.01 then

		targetUp =
			Vector3.yAxis

	end

	local targetRight =
		targetForward:Cross(
			targetUp.Unit
		)

	if targetRight.Magnitude < 0.001 then

		targetRight =
			self.Base.CFrame.RightVector

	else

		targetRight =
			targetRight.Unit

	end

	targetUp =
		targetRight:Cross(
			targetForward
		).Unit

	local currentForward =
		self.LookDirection

	if currentForward.Magnitude < 0.01 then

		currentForward =
			targetForward

	end

	local currentUp =
		self.LookUpDirection
		or targetUp

	local currentRight =
		currentForward.Unit:Cross(
			currentUp.Unit
		)

	if currentRight.Magnitude < 0.001 then

		currentRight =
			targetRight

	else

		currentRight =
			currentRight.Unit

	end

	currentUp =
		currentRight:Cross(
			currentForward.Unit
		).Unit

	local currentSteering =
		CFrame.lookAt(
			Vector3.zero,
			currentForward.Unit,
			currentUp
		)

	local targetSteering =
		CFrame.lookAt(
			Vector3.zero,
			targetForward,
			targetUp
		)

	self.AlignOrientation.MaxAngularVelocity =
		tuning.MaxAngularVelocity

	local lookAlpha =
		1
		-
		math.exp(
			-tuning.LookResponsiveness
			*
			deltaTime
		)

	local steering =
		currentSteering

	local automaticSteering =
		self.State.Mode
		==
		"ReturnToHome"

	if self.ControlsEnabled
		or automaticSteering then

		steering =
			currentSteering:Lerp(
				targetSteering,
				lookAlpha
			)

	end

	self.LookDirection =
		steering.LookVector

	self.LookUpDirection =
		steering.UpVector

	if self.ControlsEnabled then

		self.RollAngle +=
			(
				self.Input.Roll
				or 0
			)
			*
			tuning.RollSpeed
			*
			deltaTime

	end

	if math.abs(
		self.RollAngle
	)
	>
	math.pi * 2 then

		self.RollAngle =
			self.RollAngle
			%
			(
				math.pi
				*
				2
			)

	end

	local targetBank =
		0

	local targetPitch =
		0

	if self.ControlsEnabled
		or self.State.Mode == "ReturnToHome" then

		local forwardAxis =
			self.LookDirection.Magnitude > 0.01
				and self.LookDirection.Unit
				or self.Base.CFrame.LookVector

		local upAxis =
			self.LookUpDirection.Magnitude > 0.01
				and self.LookUpDirection.Unit
				or self.Base.CFrame.UpVector

		local rightAxis =
			forwardAxis:Cross(
				upAxis
			)

		if rightAxis.Magnitude < 0.01 then

			rightAxis =
				self.Base.CFrame.RightVector

		else

			rightAxis =
				rightAxis.Unit

		end

		local acceleration =
			self.FilteredAcceleration
			or Vector3.zero

		local forwardAcceleration =
			acceleration:Dot(
				forwardAxis
			)

		local rightAcceleration =
			acceleration:Dot(
				rightAxis
			)

		targetPitch =
			math.clamp(
				-math.atan2(
					forwardAcceleration,
					workspace.Gravity
				),
				-tuning.AutoPitchAngle,
				tuning.AutoPitchAngle
			)

		targetBank =
			math.clamp(
				-math.atan2(
					rightAcceleration,
					workspace.Gravity
				),
				-tuning.AutoBankAngle,
				tuning.AutoBankAngle
			)

	end

	local leanAlpha =
		1
		-
		math.exp(
			-tuning.AutoLeanResponsiveness
			*
			deltaTime
		)

	self.AutoBankAngle +=
		(
			targetBank
			-
			self.AutoBankAngle
		)
		*
		leanAlpha

	self.AutoPitchAngle +=
		(
			targetPitch
			-
			self.AutoPitchAngle
		)
		*
		leanAlpha

	local lookCFrame =
		CFrame.lookAt(
			self.Base.Position,
			self.Base.Position
				+
				self.LookDirection,
			self.LookUpDirection
		)

	self.AlignOrientation.CFrame =
		lookCFrame
		*
		CFrame.Angles(
			self.AutoPitchAngle,
			0,
			self.RollAngle
				+
				self.AutoBankAngle
		)

end

function FlightController:BeginFailsafe()

	if self.State.FailsafeTriggered then
		return
	end

	if not self.ControlsEnabled then
		return
	end

	self.State.FailsafeTriggered =
		true

	self.RTHAltitude =
		self.Base.Position.Y

	self.CommandAcceleration =
		Vector3.zero

	self.Drone:SetAttribute(
		"RTHPhase",
		"FAILSAFE"
	)

	self:SetControlsEnabled(
		false
	)

	self:SetPowerState(
		"Failsafe"
	)

	self:SetMode(
		"Failsafe"
	)

end

function FlightController:UpdateFailsafe()

	if self.State.Mode ~= "Failsafe" then
		return
	end

	if not self.HomePosition then
		return
	end

	self:SetMode(
		"ReturnToHome"
	)

end

function FlightController:UpdateReturnHome()

	if self.State.Mode ~= "ReturnToHome" then
		return
	end

	if not self.HomePosition then
		return
	end

	local horizontalOffset =
		Vector3.new(
			self.HomePosition.X
				-
				self.Base.Position.X,
			0,
			self.HomePosition.Z
				-
				self.Base.Position.Z
		)

	local horizontalDistance =
		horizontalOffset.Magnitude

	local velocity =
		self.Base.AssemblyLinearVelocity

	local horizontalSpeed =
		Vector3.new(
			velocity.X,
			0,
			velocity.Z
		).Magnitude

	if horizontalDistance
			>
			self.Config.ReturnHomeStopDistance
		or horizontalSpeed
			>
			self.Config.ReturnHomeArrivalSpeed then

		return

	end

	local groundY =
		self.HomeGroundY

	if typeof(groundY) ~= "number" then

		local probedY =
			self:GetGroundInfoAt(
				self.HomePosition,
				math.max(
					self.Base.Position.Y
						-
						self.HomePosition.Y
						+
						30,
					80
				)
			)

		groundY =
			probedY

	end

	if typeof(groundY) ~= "number" then
		return
	end

	self:StartLandingProfile(
		groundY,
		self.HomePosition
	)

	self.Drone:SetAttribute(
		"RTHPhase",
		"LANDING"
	)

	self:SetPowerState(
		"Landing"
	)

	self:SetMode(
		"Landing"
	)

end

function FlightController:UpdatePropellerThrottle()

	local mode =
		self.State.Mode

	if mode == "PoweredOff"
		or mode == "Booting"
		or mode == "ShuttingDown" then

		return

	end

	if mode == "Failsafe" then

		self:SetPropellerThrottle(
			math.min(
				1,
				self.Config.HoverPropellerThrottle
				+
				0.08
			)
		)

		return

	end

	local acceleration =
		self.CommandAcceleration
		or Vector3.zero

	local gravity =
		math.max(
			workspace.Gravity,
			0.001
		)

	local requiredSpecificForce =
		acceleration
		+
		Vector3.new(
			0,
			gravity,
			0
		)

	local thrustRatio =
		requiredSpecificForce.Magnitude
		/
		gravity

	local idealRotorThrottle =
		self.Config.HoverPropellerThrottle
		*
		math.sqrt(
			math.clamp(
				thrustRatio,
				0.20,
				2.40
			)
		)

	local throttle =
		self.Config.HoverPropellerThrottle
		+
		(
			idealRotorThrottle
			-
			self.Config.HoverPropellerThrottle
		)
		*
		self.Config.RotorThrustVisualGain

	local forward =
		self.LookDirection.Magnitude > 0.01
			and self.LookDirection.Unit
			or self.Base.CFrame.LookVector

	local up =
		self.LookUpDirection.Magnitude > 0.01
			and self.LookUpDirection.Unit
			or self.Base.CFrame.UpVector

	local right =
		forward:Cross(
			up
		)

	if right.Magnitude < 0.01 then

		right =
			self.Base.CFrame.RightVector

	else

		right =
			right.Unit

	end

	local forwardAcceleration =
		acceleration:Dot(
			forward
		)

	local rightAcceleration =
		acceleration:Dot(
			right
		)

	self.Drone:SetAttribute(
		"PropellerForwardDemand",
		math.clamp(
			forwardAcceleration
				/
				math.max(
					self.Config.Acceleration,
					0.001
				),
			-1,
			1
		)
	)

	self.Drone:SetAttribute(
		"PropellerStrafeDemand",
		math.clamp(
			rightAcceleration
				/
				math.max(
					self.Config.Acceleration,
					0.001
				),
			-1,
			1
		)
	)

	self.Drone:SetAttribute(
		"PropellerVerticalDemand",
		math.clamp(
			acceleration.Y
				/
				math.max(
					self.Config.VerticalAcceleration,
					0.001
				),
			-1,
			1
		)
	)

	self.Drone:SetAttribute(
		"PropellerRollDemand",
		self.ControlsEnabled
			and self.Input.Roll
			or 0
	)

	local groundY =
		self:GetGroundInfo()

	if groundY then

		local height =
			math.max(
				self.Base.Position.Y
					-
					groundY,
				0
			)

		local groundEffect =
			math.clamp(
				1
					-
					height
					/
					self.Config.GroundEffectHeight,
				0,
				1
			)

		throttle -=
			groundEffect
			*
			self.Config.GroundEffectThrottleReduction

	end

	if mode == "Takeoff"
		and self.TakeoffStartTime then

		local takeoffAlpha =
			math.clamp(
				(
					os.clock()
					-
					self.TakeoffStartTime
				)
					/
					math.max(
						self.Config.TakeoffProfileDuration,
						0.1
					),
				0,
				1
			)

		local retainedLiftPower =
			self.Config.TakeoffPropellerThrottle
			+
			(
				self.Config.HoverPropellerThrottle
				-
				self.Config.TakeoffPropellerThrottle
			)
			*
			minimumJerk01(
				takeoffAlpha
			)

		throttle =
			math.max(
				throttle,
				retainedLiftPower
			)

	elseif mode == "ReturnToHome" then

		throttle =
			math.max(
				throttle,
				self.Config.ReturnHomePropellerThrottle
			)

	end

	throttle +=
		math.abs(
			self.Input.Roll
			or 0
		)
		*
		self.Config.RollPropellerBoost

	self:SetPropellerThrottle(
		math.clamp(
			throttle,
			0.56,
			1
		)
	)

end

function FlightController:ApplyVelocity(
	targetVelocity,
	deltaTime
)

	self.State.TargetVelocity =
		targetVelocity

	if deltaTime <= 0 then
		return
	end

	local tuning =
		self:GetManualFlightTuning()

	if self.State.Mode == "PoweredOff"
		or self.State.Mode == "Booting" then

		if self.FlightForce then

			self.FlightForce.Force =
				Vector3.zero

			self.FlightForce.Enabled =
				false

		end

		if self.AeroForce then

			self.AeroForce.Force =
				Vector3.zero

			self.AeroForce.Enabled =
				false

		end

		return

	end

	if self.State.Mode == "ShuttingDown" then
		return
	end

	if self.State.Mode == "Landing"
		and self.LandingContactLatched then

		local mass =
			math.max(
				self.Base.AssemblyMass,
				0.001
			)

		local gravity =
			math.max(
				workspace.Gravity,
				0.001
			)

		local current =
			self.Base.AssemblyLinearVelocity

		local settleY =
			math.clamp(
				current.Y,
				-self.Config.LandingSettleSpeed,
				0
			)

		local settleX =
			math.abs(current.X) < 0.05
				and 0
				or current.X

		local settleZ =
			math.abs(current.Z) < 0.05
				and 0
				or current.Z

		self.Base.AssemblyLinearVelocity =
			Vector3.new(
				settleX,
				settleY,
				settleZ
			)

		if self.LandingVelocityDamper then

			local capturedOnPlane =
				(
					self.LandingHeightAboveRest
					or math.huge
				)
				<=
				self.Config.LandingGroundContactTolerance

			self.LandingVelocityDamper.Enabled =
				true

			self.LandingVelocityDamper.LineVelocity =
				capturedOnPlane
					and 0
					or -self.Config.LandingSettleSpeed

		end

		if self.FlightForce then

			self.FlightForce.Enabled =
				true

			self.FlightForce.Force =
				Vector3.new(
					0,
					mass
						*
						gravity
						*
						self.Config.LandingSettleLiftRatio,
					0
				)

		end

		if self.AeroForce then

			self.AeroForce.Enabled =
				true

			self.AeroForce.Force =
				Vector3.zero

		end

		self.CommandAcceleration =
			Vector3.zero

		self.AppliedControlAcceleration =
			Vector3.zero

		self.State.Velocity =
			self.Base.AssemblyLinearVelocity

		return

	end

	local measuredVelocity =
		self.Base.AssemblyLinearVelocity

	if not finiteVector(
		measuredVelocity
	) then

		measuredVelocity =
			self.State.Velocity

	end

	if not finiteVector(
		measuredVelocity
	) then

		measuredVelocity =
			Vector3.zero

	end

	self.State.Velocity =
		measuredVelocity

	local horizontalCurrent =
		Vector3.new(
			measuredVelocity.X,
			0,
			measuredVelocity.Z
		)

	local horizontalTarget =
		Vector3.new(
			targetVelocity.X,
			0,
			targetVelocity.Z
		)

	local horizontalError =
		horizontalTarget
		-
		horizontalCurrent

	local currentSpeed =
		horizontalCurrent.Magnitude

	local targetSpeed =
		horizontalTarget.Magnitude

	if targetSpeed < 0.01
		and currentSpeed
			<
			self.Config.HoverSpeedDeadband then

		horizontalError =
			Vector3.zero

	end

	local dragAcceleration =
		Vector3.zero

	if currentSpeed > 0.01 then

		dragAcceleration =
			-horizontalCurrent
			*
			self.Config.LinearDrag
			-
			horizontalCurrent.Unit
			*
			currentSpeed
			*
			currentSpeed
			*
			self.Config.QuadraticDrag

	end

	self.AerodynamicAcceleration =
		dragAcceleration

	local accelerationLimit =
		tuning.Acceleration

	if currentSpeed > 0.5
		and targetSpeed > 0.5
		and horizontalCurrent:Dot(
			horizontalTarget
		) < 0 then

		accelerationLimit =
			tuning.ReverseAcceleration

	elseif targetSpeed + 0.5
		<
		currentSpeed then

		accelerationLimit =
			tuning.Deceleration

	end

	local gravity =
		math.max(
			workspace.Gravity,
			0.001
		)

	local maxTilt =
		math.max(
			tuning.AutoPitchAngle,
			tuning.AutoBankAngle
		)

	local tiltAccelerationLimit =
		gravity
		*
		math.tan(
			maxTilt
		)

	accelerationLimit =
		math.min(
			accelerationLimit,
			tiltAccelerationLimit
		)

	local desiredNetHorizontalAcceleration =
		clampMagnitude(
			horizontalError
				*
				tuning.VelocityResponse,
			accelerationLimit
		)

	if self.State.Mode == "Landing"
		and self.LandingHorizontalTarget
		and not self.LandingContactLatched then

		local positionError =
			self.LandingHorizontalTarget
			-
			Vector3.new(
				self.Base.Position.X,
				0,
				self.Base.Position.Z
			)

		local wn =
			self.Config.LandingHorizontalNaturalFrequency

		local zeta =
			self.Config.LandingHorizontalDampingRatio

		local landingHorizontalAcceleration =
			positionError
			*
			(wn * wn)
			-
			horizontalCurrent
			*
			(2 * zeta * wn)

		desiredNetHorizontalAcceleration =
			clampMagnitude(
				landingHorizontalAcceleration,
				self.Config.LandingHorizontalMaxAcceleration
			)

	end

	local verticalError =
		targetVelocity.Y
		-
		measuredVelocity.Y

	local verticalLimit =
		tuning.VerticalAcceleration

	if math.abs(targetVelocity.Y) + 0.25
			<
			math.abs(measuredVelocity.Y)
		or targetVelocity.Y
			*
			measuredVelocity.Y
			<
			0 then

		verticalLimit =
			tuning.VerticalDeceleration

	end

	local desiredNetVerticalAcceleration =
		math.clamp(
			verticalError
				*
				tuning.VelocityResponse,
			-verticalLimit,
			verticalLimit
		)

	if self.State.Mode == "Landing"
		and self.LandingHeightAboveRest ~= nil
		and self.LandingHeightAboveRest
			<=
			self.Config.LandingFinalControllerHeight then

		local height =
			math.max(
				self.LandingHeightAboveRest,
				0
			)

		local wn =
			self.Config.LandingFinalNaturalFrequency

		local zeta =
			self.Config.LandingFinalDampingRatio

		local positionTerm =
			-(wn * wn)
			*
			height

		local dampingTerm =
			-(2 * zeta * wn)
			*
			measuredVelocity.Y

		local finalAcceleration =
			positionTerm
			+
			dampingTerm

		desiredNetVerticalAcceleration =
			math.clamp(
				finalAcceleration,
				-self.Config.LandingFinalMaxDownAcceleration,
				self.Config.LandingFinalMaxBrakeAcceleration
			)

	end

	local desiredNetAcceleration =
		Vector3.new(
			desiredNetHorizontalAcceleration.X,
			desiredNetVerticalAcceleration,
			desiredNetHorizontalAcceleration.Z
		)

	local requiredControlAcceleration =
		desiredNetAcceleration
		-
		dragAcceleration

	local currentAcceleration =
		self.CommandAcceleration
		or Vector3.zero

	local currentHorizontalAcceleration =
		Vector3.new(
			currentAcceleration.X,
			0,
			currentAcceleration.Z
		)

	local targetHorizontalControl =
		Vector3.new(
			requiredControlAcceleration.X,
			0,
			requiredControlAcceleration.Z
		)

	local newHorizontalAcceleration =
		moveVectorTowards(
			currentHorizontalAcceleration,
			targetHorizontalControl,
			tuning.HorizontalJerk
				*
				deltaTime
		)

	local verticalJerk =
		tuning.VerticalJerk

	if self.State.Mode == "Landing"
		and self.LandingHeightAboveRest ~= nil
		and self.LandingHeightAboveRest
			<=
			self.Config.LandingFinalControllerHeight then

		verticalJerk =
			math.max(
				verticalJerk,
				900
			)

	end

	local newVerticalAcceleration =
		moveNumberTowards(
			currentAcceleration.Y,
			requiredControlAcceleration.Y,
			verticalJerk
				*
				deltaTime
		)

	local controlAcceleration =
		Vector3.new(
			newHorizontalAcceleration.X,
			newVerticalAcceleration,
			newHorizontalAcceleration.Z
		)

	controlAcceleration =
		clampMagnitude(
			controlAcceleration,
			tuning.MaxControlAcceleration
		)

	self.CommandAcceleration =
		controlAcceleration

	self.AppliedControlAcceleration =
		controlAcceleration

	local mass =
		math.max(
			self.Base.AssemblyMass,
			0.001
		)

	local requestedSpecificThrust =
		controlAcceleration
		+
		Vector3.new(
			0,
			gravity,
			0
		)

	local maximumSpecificThrust =
		gravity
		*
		self.Config.MaxThrustToWeight

	requestedSpecificThrust =
		clampMagnitude(
			requestedSpecificThrust,
			maximumSpecificThrust
		)

	if self.FlightForce then

		self.FlightForce.Enabled =
			true

		self.FlightForce.Force =
			requestedSpecificThrust
			*
			mass

	end

	if self.AeroForce then

		self.AeroForce.Enabled =
			true

		self.AeroForce.Force =
			dragAcceleration
			*
			mass

	end

	local rawAcceleration =
		(
			measuredVelocity
			-
			(
				self.LastMeasuredVelocity
				or measuredVelocity
			)
		)
		/
		math.max(
			deltaTime,
			1 / 240
		)

	rawAcceleration =
		clampMagnitude(
			rawAcceleration,
			tuning.MaxControlAcceleration
				*
				3
		)

	local accelerationAlpha =
		1
		-
		math.exp(
			-7
			*
			deltaTime
		)

	self.FilteredAcceleration =
		self.FilteredAcceleration:Lerp(
			rawAcceleration,
			accelerationAlpha
		)

	self.LastMeasuredVelocity =
		measuredVelocity

	self.PhysicsTelemetryTimer +=
		deltaTime

	if self.PhysicsTelemetryTimer >= 0.10 then

		self.PhysicsTelemetryTimer =
			0

		local supportAcceleration =
			self.FilteredAcceleration
			+
			Vector3.new(
				0,
				gravity,
				0
			)

		self.Drone:SetAttribute(
			"FlightSpeed",
			measuredVelocity.Magnitude
		)

		self.Drone:SetAttribute(
			"FlightAcceleration",
			self.FilteredAcceleration.Magnitude
		)

		self.Drone:SetAttribute(
			"FlightAccelerationVector",
			self.FilteredAcceleration
		)

		self.Drone:SetAttribute(
			"FlightGLoad",
			supportAcceleration.Magnitude
				/
				gravity
		)

		self.Drone:SetAttribute(
			"FlightDrag",
			dragAcceleration.Magnitude
		)

		self.Drone:SetAttribute(
			"FlightThrustRatio",
			requestedSpecificThrust.Magnitude
				/
				gravity
		)

	end

end

function FlightController:Update(
	deltaTime
)

	if not self.State.Active then
		return
	end

	self.State.FlightTime +=
		deltaTime

	self:UpdateBooting()
	self:UpdateTakeoff()
	self:UpdateLanding()
	self:UpdateShuttingDown()

	self:UpdateFailsafe()
	self:UpdateReturnHome()

	self:UpdateLookOrientation(
		deltaTime
	)

	if self.ControlsEnabled then

		local hasInput =
			math.abs(
				self.Input.Forward
			) > 0.01
			or math.abs(
				self.Input.Strafe
			) > 0.01
			or math.abs(
				self.Input.Vertical
			) > 0.01
			or math.abs(
				self.Input.Roll
			) > 0.01

		if hasInput
			and self.State.Mode == "Armed" then

			self:SetMode(
				"Flying"
			)

		elseif not hasInput
			and self.State.Mode == "Flying" then

			self:SetMode(
				"Armed"
			)

		end

	end

	local elapsed =
		os.clock()
		-
		self.StartTime

	if self.ControlsEnabled
		and elapsed
			>
			self.Config.StartupGracePeriod then

		local signalAge =
			os.clock()
			-
			self.LastInputTime

		if signalAge
			>
			self.Config.InputTimeout then

			self:BeginFailsafe()

		end

	end

	local targetVelocity =
		self:CalculateTargetVelocity()

	self:ApplyVelocity(
		targetVelocity,
		deltaTime
	)

	self:UpdatePropellerThrottle()

end

function FlightController:Start()

	if self.State.Active then
		return
	end

	self:CreatePhysics()

	self:ConfigureLandingContactPhysics()

	pcall(function()

		self.Base:SetNetworkOwner(
			nil
		)

	end)

	self.State.Active =
		true

	self.State.FlightTime =
		0

	self.State.FailsafeTriggered =
		false

	self.ShutdownStartThrottle =
		nil

	self.State.Velocity =
		Vector3.zero

	self.State.TargetVelocity =
		Vector3.zero

	self.CommandAcceleration =
		Vector3.zero

	self.FilteredAcceleration =
		Vector3.zero

	self.AppliedControlAcceleration =
		Vector3.zero

	self.AerodynamicAcceleration =
		Vector3.zero

	self.LastMeasuredVelocity =
		self.Base.AssemblyLinearVelocity

	self.PhysicsTelemetryTimer =
		0

	self.StartTime =
		os.clock()

	self.LastInputTime =
		os.clock()

	self.HomePosition =
		self.Base.Position

	self.HomeGroundY =
		nil

	self.LandingHorizontalTarget =
		nil

	self.Drone:SetAttribute(
		"DroneHomePosition",
		self.HomePosition
	)

	self.Drone:SetAttribute(
		"DroneHomeGroundY",
		nil
	)

	self.Drone:SetAttribute(
		"RTHPhase",
		"STANDBY"
	)

	self.Drone:SetAttribute(
		"RTHDistance",
		0
	)

	self:SetControlsEnabled(
		false
	)

	self:SetPropellerThrottle(
		0
	)

	self:SetPowerState(
		"Off"
	)

	self:SetMode(
		"PoweredOff"
	)

	if self.FlightForce then

		self.FlightForce.Force =
			Vector3.zero

		self.FlightForce.Enabled =
			false

	end

	if self.AeroForce then

		self.AeroForce.Force =
			Vector3.zero

		self.AeroForce.Enabled =
			false

	end

	if self.AlignOrientation then

		self.AlignOrientation.Enabled =
			false

	end

	self.Connection =
		RunService.PreSimulation:Connect(
			function(deltaTime)

				self:Update(
					deltaTime
				)

			end
		)

end

function FlightController:Stop()

	if not self.State.Active then
		return
	end

	self.State.Active =
		false

	self:SetControlsEnabled(
		false
	)

	self:SetPropellerThrottle(
		0
	)

	self.CommandAcceleration =
		Vector3.zero

	self.FilteredAcceleration =
		Vector3.zero

	self.AppliedControlAcceleration =
		Vector3.zero

	self.AerodynamicAcceleration =
		Vector3.zero

	self:SetPowerState(
		"Off"
	)

	if self.Connection then

		self.Connection:Disconnect()

		self.Connection =
			nil

	end

	if self.FlightForce then

		self.FlightForce.Force =
			Vector3.zero

		self.FlightForce.Enabled =
			false

	end

	if self.AeroForce then

		self.AeroForce.Force =
			Vector3.zero

		self.AeroForce.Enabled =
			false

	end

	if self.AlignOrientation then

		self.AlignOrientation.Enabled =
			false

	end

	if self.LandingVelocityDamper then

		self.LandingVelocityDamper.LineVelocity =
			0

		self.LandingVelocityDamper.Enabled =
			false

	end

	pcall(function()

		self.Base:SetNetworkOwner(
			nil
		)

	end)

end

function FlightController:Destroy()

	self:Stop()

	if self.AlignOrientation then

		self.AlignOrientation:Destroy()

		self.AlignOrientation =
			nil

	end

	if self.FlightForce then

		self.FlightForce:Destroy()

		self.FlightForce =
			nil

	end

	if self.AeroForce then

		self.AeroForce:Destroy()

		self.AeroForce =
			nil

	end

	if self.LandingVelocityDamper then

		self.LandingVelocityDamper:Destroy()

		self.LandingVelocityDamper =
			nil

	end

	if self.FlightAttachment then

		self.FlightAttachment:Destroy()

		self.FlightAttachment =
			nil

	end

end

return FlightController
```
