[← Back to Home](../README.md)

# UAV Flight Controller Physics Module (FlightController)

This module was created to handle the drones manual thrust vectors, it's angular velocities, the torque offsets, the translational flight updates, and it's ground aware landing flare calculation that is manually used to fly the drone's physics.

```lua
local RunService = game:GetService("RunService")

local FlightController = {}
FlightController.__index = FlightController

------------------------------------------------
-- CONFIG
------------------------------------------------

local DEFAULT_CONFIG = {
	-- Commanded airspeed envelope.
	MaxSpeed = 72,
	VerticalSpeed = 30,

	-- Camera/airframe steering. This remains responsive, but the actuator now
	-- has an angular-speed cap so the body does not teleport between attitudes.
	LookResponsiveness = 5.6,
	RollSpeed = math.rad(150),
	MaxAngularVelocity = math.rad(300),
	OrientationResponsiveness = 20,

	-- Automatic body lean is no longer based on a cosmetic input percentage.
	-- It is derived from the acceleration vector using atan2(a, g), then capped
	-- to believable surveillance-UAV attitudes.
	AutoBankAngle = math.rad(18),
	AutoPitchAngle = math.rad(18),
	AutoLeanResponsiveness = 6.5,

	-- Translational flight model. The controller turns velocity error into a
	-- bounded acceleration request, then rate-limits that request with jerk.
	-- This creates real spool-up, momentum, braking and direction-change weight.
	VelocityResponse = 3.8,
	Acceleration = 66,
	Deceleration = 76,
	ReverseAcceleration = 86,
	VerticalAcceleration = 46,
	VerticalDeceleration = 58,
	HorizontalJerk = 620,
	VerticalJerk = 420,

	-- Indoor precision / cine profile. NORMAL remains unchanged.
	-- These values only apply while PrecisionMode=true and the aircraft
	-- is under manual control in Armed/Flying.
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

	-- Mild aerodynamic damping. Linear drag handles low-speed settling while
	-- quadratic drag becomes more noticeable near maximum speed.
	LinearDrag = 0.020,
	QuadraticDrag = 0.00048,

	-- Force-based authority. Horizontal/vertical acceleration is bounded and
	-- the physical rotor-thrust vector is capped by a thrust-to-weight ratio.
	MaxControlAcceleration = 96,
	ForceReserve = 1.25, -- retained for config compatibility
	MaxThrustToWeight = 1.72,

	-- Rotor visual load now follows calculated acceleration/thrust demand rather
	-- than simply mirroring key presses.
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

	-- Small deterministic hover movement prevents a perfectly rail-locked look.
	HoverDriftSpeed = 0,
	HoverDriftFrequency = 0,

	-- Boot / takeoff / landing.
	-- Staged motor arming: ESC pause -> ground idle -> lift power -> hold -> liftoff.
	BootDuration = 3.4, -- retained for compatibility; staged timings below drive the sequence.
	MotorStartDelay = 0.45,
	GroundIdlePropellerThrottle = 0.46,
	GroundSpoolDuration = 1.45,
	GroundIdleHoldDuration = 0.45,
	LiftSpoolDuration = 0.90,
	PreLiftHoldDuration = 0.55,

	-- Takeoff follows a fifth-order minimum-jerk trajectory. It starts and ends
	-- at zero vertical speed instead of snapping into/out of climb velocity.
	TakeoffHeight = 4.5,
	TakeoffSpeed = 6.2,
	TakeoffProfileDuration = 1.55,
	TakeoffTrackingGain = 2.8,
	TakeoffCorrectionSpeed = 1.10,
	TakeoffThreshold = 0.28,
	TakeoffSettleVerticalSpeed = 1.00,

	-- Landing is rangefinder-driven. A downward "radar altimeter" measures the
	-- real surface every physics frame and the descent speed is derived from the
	-- remaining stopping distance instead of blindly following a timed path.
	LandingSpeed = 6.5,
	LandingTrackingGain = 2.6, -- retained for compatibility with older profiles
	LandingMinDuration = 2.0, -- retained for telemetry/debug compatibility
	LandingMaxDuration = 40.0,
	LandingThreshold = 0.08,
	LandingTouchdownTolerance = 0.11,
	LandingTouchdownMaxSpeed = 0.42,
	LandingTouchdownConfirmTime = 0.20,
	LandingContactCreepSpeed = 0.05,
	-- Landing X/Z is a true position-hold loop, not a proportional velocity
	-- command.  An over-damped PD controller removes sideways oscillation and
	-- keeps the aircraft over the exact point captured when T was pressed.
	LandingHorizontalGain = 2.2, -- retained for backwards config compatibility
	LandingHorizontalMaxSpeed = 5.5, -- retained for backwards config compatibility
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

	-- Ground-aware flare controller.
	LandingSensorRange = 120,
	LandingProbeSpread = 0.34,
	LandingFlareDeceleration = 3.2,
	LandingFlareMargin = 0.42,
	LandingFinalApproachHeight = 3.00,
	LandingFinalApproachGain = 0.42,
	LandingFinalDescentSpeed = 0.05,
	LandingMaxFinalDescentSpeed = 0.80,

	-- Final 3 m uses a critically-damped height/vertical-speed controller rather
	-- than relying only on a velocity target. This actively removes sink rate
	-- before contact instead of waiting for the floor collision to stop the UAV.
	LandingFinalControllerHeight = 3.0,
	LandingFinalNaturalFrequency = 2.6,
	LandingFinalDampingRatio = 1.20,
	LandingFinalMaxDownAcceleration = 4.5,
	LandingFinalMaxBrakeAcceleration = 22.0,

	-- Final touchdown capture. Once the rangefinder is inside this tiny cushion
	-- above the recorded resting height, the landing-gear damper becomes the
	-- authority for vertical motion. The latch is one-way: a collision impulse
	-- cannot kick the controller back into descent and create repeated bouncing.
	LandingCaptureHeight = 0.18,
	LandingSettleSpeed = 0.08,
	LandingSettleLiftRatio = 0.985,
	-- 0.012 studs was too strict for a deployed model whose landing feet/collision
	-- geometry can settle a few hundredths of a stud above the exact sampled plane.
	-- 0.075 still represents a very small touchdown cushion, but reliably lets the
	-- shutdown state begin once the landing gear has physically captured the floor.
	LandingGroundContactTolerance = 0.075,
	LandingCapturedSettleTimeout = 0.90,
	LandingGroundConfirmTime = 0.12,
	LandingShutdownBounceSpeed = 0.06,

	-- Landing gear/contact restitution. Real multirotor feet do not behave like
	-- rubber balls; collidable drone parts are biased toward zero restitution.
	LandingContactElasticity = 0.0,
	LandingContactElasticityWeight = 100,
	ShutdownDuration = 2.3,

	-- Thrust-to-RPM visual mapping. Ideal prop thrust is approximately
	-- proportional to RPM^2, hence the square-root conversion below.
	RotorThrustVisualGain = 1.45,

	GroundRayDistance = 50,
	AutoLevelResponsiveness = 5.5,

	MaxAltitude = 300,
	MinAltitude = 5,

	InputTimeout = 999,
	StartupGracePeriod = 3,

	ReturnHomeSpeed = 32,
	ReturnHomeStopDistance = 0.45,
	ReturnHomeAltitude = 0, -- retained for config compatibility; V5 holds the current safe altitude
	ReturnHomeBrakeAcceleration = 24,
	ReturnHomePositionGain = 2.4,
	ReturnHomeVerticalGain = 1.35,
	ReturnHomeVerticalSpeed = 8,
	ReturnHomeArrivalSpeed = 1.0,
	TakeoffMaxSettleTime = 1.8,
	HoverSpeedDeadband = 0.10,
}

------------------------------------------------
-- FLIGHT MATH HELPERS
------------------------------------------------

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

-- Fifth-order minimum-jerk trajectory. Position, velocity and acceleration are
-- all smooth at the endpoints, which makes it ideal for takeoff/landing motion.
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

------------------------------------------------
-- CONSTRUCTOR
------------------------------------------------

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

	-- One-way touchdown latch used by the final landing-gear damper.
	self.LandingContactLatched =
		false

	self.LandingContactLatchTime =
		nil

	-- One-way landing-position acquisition latch.  The descent does not begin
	-- until the aircraft has arrested lateral motion over the point captured
	-- when the landing command was issued.
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

	-- Two explicit forces are used instead of a velocity servo:
	-- FlightForce produces rotor/control thrust and AeroForce applies drag.
	-- This makes acceleration come from F = m*a instead of repeatedly nudging
	-- a LinearVelocity target one tiny timestep ahead.
	self.FlightForce =
		nil

	self.AeroForce =
		nil

	self.AlignOrientation =
		nil

	-- Temporary final-contact velocity constraint. It is disabled for all normal
	-- flight and only engages inside the last few centimetres of landing.
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

	-- Physics state used by the acceleration/jerk model.
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

------------------------------------------------
-- ATTRIBUTES / STATE
------------------------------------------------

function FlightController:SetMode(mode)

	if not self.ValidStates[mode] then

		warn(
			"[Drone] Invalid flight mode:",
			mode
		)

		return false

	end

	if self.State.Mode == mode then
		return true
	end

	local previous =
		self.State.Mode

	self.State.Mode =
		mode

	self.Drone:SetAttribute(
		"FlightMode",
		mode
	)

	print(
		"[Drone] Flight mode:",
		previous,
		"->",
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

------------------------------------------------
-- INPUT
------------------------------------------------

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

		-- Re-orthogonalise the supplied camera up vector so a full
		-- flip can pass through vertical and inverted orientations
		-- without world-up snapping the craft upright.
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

------------------------------------------------
-- PHYSICS
------------------------------------------------

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

	------------------------------------------------
	-- PHYSICAL THRUST FORCE
	------------------------------------------------

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

	------------------------------------------------
	-- AERODYNAMIC DRAG FORCE
	------------------------------------------------

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

	------------------------------------------------
	-- ATTITUDE ACTUATOR
	------------------------------------------------

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

	------------------------------------------------
	-- FINAL-CONTACT VELOCITY DAMPER
	------------------------------------------------

	self.LandingVelocityDamper =
		Instance.new("LinearVelocity")

	self.LandingVelocityDamper.Name =
		"LandingVelocityDamper"

	self.LandingVelocityDamper.Attachment0 =
		self.FlightAttachment

	self.LandingVelocityDamper.RelativeTo =
		Enum.ActuatorRelativeTo.World

	-- Vertical-only constraint.  V8 used Vector mode, which also forced X/Z to
	-- zero with infinite force during final contact.  That fought the horizontal
	-- landing controller and could kick the aircraft sideways.
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

	print(
		"[Drone] Force-based UAV physics created - VectorForce thrust + aerodynamic drag"
	)

end

-- Configure the aircraft's collidable surfaces as low-restitution landing
-- contacts. This preserves each part's density/friction while removing the
-- material bounce that can otherwise launch the assembly back upward at
-- touchdown before the controller gets its next simulation step.
function FlightController:ConfigureLandingContactPhysics()
	for _, descendant in ipairs(self.Drone:GetDescendants()) do
		if descendant:IsA("BasePart") and descendant.CanCollide then
			pcall(function()
				local current = descendant.CurrentPhysicalProperties
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

------------------------------------------------
-- GROUND DETECTION
------------------------------------------------

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



-- Five-point landing rangefinder. The centre probe plus four footprint probes
-- make touchdown detection much less sensitive to a single ray missing an edge,
-- stair, slope or small change in terrain. The highest valid surface is treated
-- as the limiting ground plane so a corner/leg cannot strike before the centre.
function FlightController:GetLandingGroundInfo()
	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	params.FilterDescendantsInstances = { self.Drone }
	params.IgnoreWater = false

	local base = self.Base
	local spreadX = math.max(base.Size.X * self.Config.LandingProbeSpread, 0.18)
	local spreadZ = math.max(base.Size.Z * self.Config.LandingProbeSpread, 0.18)

	local right = base.CFrame.RightVector
	local forward = base.CFrame.LookVector
	local origins = {
		base.Position,
		base.Position + right * spreadX + forward * spreadZ,
		base.Position - right * spreadX + forward * spreadZ,
		base.Position + right * spreadX - forward * spreadZ,
		base.Position - right * spreadX - forward * spreadZ,
	}

	local bestGroundY = nil
	local bestResult = nil
	local closestClearance = math.huge
	local rayDistance = math.max(
		self.Config.LandingSensorRange or self.Config.GroundRayDistance,
		self.Config.GroundRayDistance
	)

	for _, origin in ipairs(origins) do
		local result = workspace:Raycast(
			origin,
			Vector3.new(0, -rayDistance, 0),
			params
		)

		if result then
			-- Clearance is measured from the aircraft centre to the hit plane,
			-- matching RestingClearance and the rest of the landing controller.
			local clearance = base.Position.Y - result.Position.Y
			if clearance >= 0 and clearance < closestClearance then
				closestClearance = clearance
				bestGroundY = result.Position.Y
				bestResult = result
			end
		end
	end

	if not bestGroundY then
		return nil, nil, nil
	end

	return bestGroundY, closestClearance, bestResult
end


function FlightController:GetGroundInfoAt(position, rayHeight)
	if typeof(position) ~= "Vector3" then
		return nil, nil
	end

	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	params.FilterDescendantsInstances = { self.Drone }
	params.IgnoreWater = false

	-- Probe the stored launch X/Z from well above the expected surface. This is
	-- independent of the aircraft's current altitude, so RTH cannot get stuck just
	-- because the drone is more than GroundRayDistance above the floor.
	local probeHeight = math.max(rayHeight or 0, self.Config.GroundRayDistance + 20)
	local origin = Vector3.new(position.X, position.Y + probeHeight, position.Z)
	local result = workspace:Raycast(
		origin,
		Vector3.new(0, -(probeHeight * 2), 0),
		params
	)

	if not result then
		return nil, nil
	end

	return result.Position.Y, result
end

------------------------------------------------
-- TAKEOFF / LAND COMMANDS
------------------------------------------------

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

	-- Record a provisional launch point now, then refresh it again at the exact
	-- instant liftoff starts after the ground spool/hold sequence.
	self.HomeGroundY = groundY or (self.Base.Position.Y - self.RestingClearance)
	self.HomePosition = Vector3.new(
		self.Base.Position.X,
		self.HomeGroundY + self.RestingClearance,
		self.Base.Position.Z
	)
	self.Drone:SetAttribute("DroneHomePosition", self.HomePosition)
	self.Drone:SetAttribute("DroneHomeGroundY", self.HomeGroundY)
	self.Drone:SetAttribute("RTHPhase", "STANDBY")

	-- Do not choose the flight trajectory until the spool sequence has finished.
	-- This lets the aircraft remain fully supported by the floor while the motors
	-- come up to speed and avoids launch-height errors caused by ground settling.
	self.TakeoffTargetY = nil

	self.BootStartTime =
		os.clock()

	self.ShutdownStartThrottle =
		nil

	self.TakeoffStartY = nil
	self.TakeoffStartTime = nil
	self.LandingGroundY = nil
	self.LandingStartY = nil
	self.LandingStartTime = nil
	self.LandingProfileDuration = nil
	self.TouchdownCandidateTime = nil
	self.TouchdownHolding = false

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

	self:SetControlsEnabled(false)

	self:SetPropellerThrottle(0)

	self:SetPowerState(
		"Booting"
	)

	self:SetMode(
		"Booting"
	)

	print(
		"[Drone] Boot sequence started"
	)

	return true

end


function FlightController:StartLandingProfile(groundY, horizontalTarget)
	if typeof(groundY) ~= "number" then
		return false
	end

	self.LandingGroundY = groundY
	self.LandingTargetY = groundY + self.RestingClearance

	local target = horizontalTarget
	if typeof(target) ~= "Vector3" then
		target = self.Base.Position
	end

	self.LandingHorizontalTarget = Vector3.new(target.X, 0, target.Z)
	self.LandingStartY = self.Base.Position.Y
	self.LandingStartTime = os.clock()
	self.TouchdownCandidateTime = nil
	self.TouchdownHolding = false
	self.LandingContactLatched = false
	self.LandingContactLatchTime = nil
	self.LandingDescentCommitted = false
	self.LandingMeasuredClearance = nil
	self.LandingHeightAboveRest = nil

	-- Publish the requested touchdown point for diagnostics / portfolio telemetry.
	self.Drone:SetAttribute(
		"LandingTargetPosition",
		Vector3.new(
			self.LandingHorizontalTarget.X,
			self.LandingTargetY,
			self.LandingHorizontalTarget.Z
		)
	)

	local distance =
		math.max(self.LandingStartY - self.LandingTargetY, 0)

	-- A minimum-jerk curve has a normalized peak derivative of 1.875. Choose
	-- duration from that fact so its peak descent speed respects LandingSpeed.
	local durationFromSpeed =
		1.875 * distance / math.max(self.Config.LandingSpeed, 0.1)

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
		Vector3.new(self.Base.Position.X, 0, self.Base.Position.Z)
	)

	self:SetControlsEnabled(false)

	self:SetPowerState(
		"Landing"
	)

	self:SetMode(
		"Landing"
	)

	print(
		"[Drone] Landing requested"
	)

	return true

end

------------------------------------------------
-- BOOT
------------------------------------------------

function FlightController:UpdateBooting()
	if self.State.Mode ~= "Booting" then
		return
	end

	local elapsed = os.clock() - (self.BootStartTime or os.clock())
	local config = self.Config

	local tArm = math.max(config.MotorStartDelay, 0)
	local tGroundSpoolEnd = tArm + math.max(config.GroundSpoolDuration, 0.05)
	local tGroundHoldEnd = tGroundSpoolEnd + math.max(config.GroundIdleHoldDuration, 0)
	local tLiftSpoolEnd = tGroundHoldEnd + math.max(config.LiftSpoolDuration, 0.05)
	local tLiftHoldEnd = tLiftSpoolEnd + math.max(config.PreLiftHoldDuration, 0)

	-- Stage 1: ESC arming pause. Blades remain stopped.
	if elapsed < tArm then
		self:SetPropellerThrottle(0)
		return
	end

	-- Stage 2: smoothly spool to a stable ground-idle RPM.
	if elapsed < tGroundSpoolEnd then
		local alpha = (elapsed - tArm) / math.max(config.GroundSpoolDuration, 0.05)
		self:SetPropellerThrottle(
			minimumJerk01(alpha) * config.GroundIdlePropellerThrottle
		)
		return
	end

	-- Stage 3: hold ground idle so the startup is visibly deliberate.
	if elapsed < tGroundHoldEnd then
		self:SetPropellerThrottle(config.GroundIdlePropellerThrottle)
		return
	end

	-- Stage 4: raise rotor power from ground idle to the liftoff setting.
	if elapsed < tLiftSpoolEnd then
		local alpha =
			(elapsed - tGroundHoldEnd) / math.max(config.LiftSpoolDuration, 0.05)
		local blend = minimumJerk01(alpha)
		local throttle =
			config.GroundIdlePropellerThrottle
			+ (config.TakeoffPropellerThrottle - config.GroundIdlePropellerThrottle) * blend
		self:SetPropellerThrottle(throttle)
		return
	end

	-- Stage 5: actually sit at liftoff power before releasing the aircraft.
	self:SetPropellerThrottle(config.TakeoffPropellerThrottle)
	if elapsed < tLiftHoldEnd then
		return
	end

	if self.FlightForce then
		local mass = math.max(self.Base.AssemblyMass, 0.001)
		self.FlightForce.Force =
			Vector3.new(0, mass * workspace.Gravity, 0)
		self.FlightForce.Enabled = true
	end

	if self.AeroForce then
		self.AeroForce.Force = Vector3.zero
		self.AeroForce.Enabled = true
	end

	if self.AlignOrientation then
		self.AlignOrientation.CFrame = self.Base.CFrame
		self.AlignOrientation.Enabled = true
	end

	-- THIS is the authoritative home point: the exact X/Z and floor height at
	-- the instant the aircraft is released from the ground into its takeoff path.
	local liftoffGroundY, liftoffClearance = self:GetGroundInfo()
	if liftoffGroundY and liftoffClearance then
		self.RestingClearance = math.max(liftoffClearance, 0.1)
		self.HomeGroundY = liftoffGroundY
	else
		self.HomeGroundY = self.HomeGroundY or (self.Base.Position.Y - self.RestingClearance)
	end

	self.HomePosition = Vector3.new(
		self.Base.Position.X,
		self.HomeGroundY + self.RestingClearance,
		self.Base.Position.Z
	)
	self.Drone:SetAttribute("DroneHomePosition", self.HomePosition)
	self.Drone:SetAttribute("DroneHomeGroundY", self.HomeGroundY)

	self.State.Velocity = Vector3.zero
	self.State.TargetVelocity = Vector3.zero
	self.CommandAcceleration = Vector3.zero
	self.LastMeasuredVelocity = self.Base.AssemblyLinearVelocity

	self.TakeoffStartY = self.Base.Position.Y
	self.TakeoffTargetY = self.TakeoffStartY + self.Config.TakeoffHeight
	self.TakeoffStartTime = os.clock()

	-- Clear tiny ground-contact velocities before the trajectory begins.
	self.CommandAcceleration = Vector3.zero
	self.FilteredAcceleration = Vector3.zero

	self:SetPowerState("TakingOff")
	self:SetMode("Takeoff")

	print("[Drone] Lift power stabilised - minimum-jerk takeoff beginning")
end

------------------------------------------------
-- TAKEOFF
------------------------------------------------

function FlightController:CalculateTakeoffVelocity()
	if not self.TakeoffTargetY
		or not self.TakeoffStartY
		or not self.TakeoffStartTime then

		return Vector3.zero
	end

	local duration = math.max(self.Config.TakeoffProfileDuration, 0.1)
	local elapsed = os.clock() - self.TakeoffStartTime
	local alpha = math.clamp(elapsed / duration, 0, 1)

	local travel = self.TakeoffTargetY - self.TakeoffStartY
	local desiredPosition =
		self.TakeoffStartY + travel * minimumJerk01(alpha)

	local feedForwardVelocity =
		travel * minimumJerkDerivative01(alpha) / duration

	local positionError = desiredPosition - self.Base.Position.Y
	local commandedY =
		feedForwardVelocity
		+ positionError * self.Config.TakeoffTrackingGain

	-- After the trajectory completes, retain a very small bidirectional
	-- correction authority so any physical overshoot can settle precisely.
	if alpha >= 1 then
		local finalError = self.TakeoffTargetY - self.Base.Position.Y
		commandedY = math.clamp(
			finalError * self.Config.TakeoffTrackingGain,
			-self.Config.TakeoffCorrectionSpeed,
			self.Config.TakeoffCorrectionSpeed
		)
	end

	commandedY = math.clamp(
		commandedY,
		-self.Config.TakeoffCorrectionSpeed,
		self.Config.TakeoffSpeed
	)

	return Vector3.new(0, commandedY, 0)
end

function FlightController:UpdateTakeoff()
	if self.State.Mode ~= "Takeoff" then
		return
	end

	if not self.TakeoffTargetY or not self.TakeoffStartTime then
		return
	end

	local elapsed = os.clock() - self.TakeoffStartTime
	local profileFinished = elapsed >= self.Config.TakeoffProfileDuration
	local positionError = self.TakeoffTargetY - self.Base.Position.Y
	local verticalSpeed = math.abs(self.Base.AssemblyLinearVelocity.Y)

	-- A real autopilot does not wait for a mathematically impossible exact sample.
	-- Accept a small capture envelope once the trajectory has finished. A wider
	-- fallback envelope prevents the state machine getting trapped forever due to
	-- tiny solver/contact oscillations.
	local captured =
		profileFinished
		and math.abs(positionError) <= self.Config.TakeoffThreshold
		and verticalSpeed <= self.Config.TakeoffSettleVerticalSpeed

	local settleTimedOut =
		elapsed >= self.Config.TakeoffProfileDuration + self.Config.TakeoffMaxSettleTime
		and math.abs(positionError) <= 0.65
		and verticalSpeed <= 1.6

	if not captured and not settleTimedOut then
		return
	end

	self.State.Velocity = self.Base.AssemblyLinearVelocity
	self.State.TargetVelocity = Vector3.zero
	self.CommandAcceleration = Vector3.zero
	self.FilteredAcceleration = Vector3.zero
	self.LastMeasuredVelocity = self.Base.AssemblyLinearVelocity

	self:SetPowerState("Ready")
	self:SetPropellerThrottle(self.Config.HoverPropellerThrottle)
	self:SetControlsEnabled(true)
	self:SetMode("Armed")

	print("[Drone] Takeoff complete - stable hover acquired")
end

------------------------------------------------
-- LANDING
------------------------------------------------

function FlightController:CalculateLandingVelocity()
	if not self.LandingTargetY then
		return Vector3.zero
	end

	if self.TouchdownHolding then
		return Vector3.zero
	end

	------------------------------------------------
	-- LIVE GROUND-CLEARANCE SENSOR
	------------------------------------------------

	local groundY, clearance =
		self:GetLandingGroundInfo()

	-- If every landing probe temporarily loses the surface, fall back to the
	-- known landing plane instead of suddenly changing state or dropping.
	if not groundY or not clearance then
		groundY = self.LandingGroundY

		if groundY then
			clearance =
				self.Base.Position.Y
			-
				groundY
		else
			clearance =
				math.max(
					self.Base.Position.Y - self.LandingTargetY,
					self.RestingClearance
				)
		end
	else
		-- Continuously follow the actual local surface. This is what allows the
		-- aircraft to land correctly on small slopes/steps rather than relying on
		-- the Y coordinate measured when the landing sequence first started.
		self.LandingGroundY = groundY
		self.LandingTargetY =
			groundY
			+
			self.RestingClearance
	end

	local heightAboveRest =
		math.max(
			clearance - self.RestingClearance,
			0
		)

	-- Cache the live rangefinder measurement for the force loop. ApplyVelocity
	-- runs in the same PreSimulation step and can therefore use both height and
	-- measured sink rate to brake before the collision solver ever sees contact.
	self.LandingMeasuredClearance = clearance
	self.LandingHeightAboveRest = heightAboveRest

	------------------------------------------------
	-- BRAKING-DISTANCE DESCENT LAW
	------------------------------------------------

	-- For a required braking acceleration a, the maximum safe downward speed at
	-- remaining distance d is v = sqrt(2*a*d). As the ground gets closer, the
	-- requested descent speed therefore falls automatically BEFORE contact.
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

	------------------------------------------------
	-- FINAL APPROACH / FLARE
	------------------------------------------------

	if heightAboveRest <= self.Config.LandingFinalApproachHeight then
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

	-- Never let the mathematical stopping-speed reach exactly zero while still
	-- above the surface; that would leave the UAV hovering a few centimetres up.
	if heightAboveRest > self.Config.LandingTouchdownTolerance then
		descentSpeed =
			math.max(
				descentSpeed,
				self.Config.LandingFinalDescentSpeed
			)
	else
		descentSpeed = 0
	end

	local commandedY =
		-descentSpeed

	------------------------------------------------
	-- PRECISION LANDING POINT ACQUISITION
	------------------------------------------------

	-- X/Z is controlled directly by the damped position controller in
	-- ApplyVelocity.  Do not create another proportional velocity loop here:
	-- stacking the two loops was what produced the left/right hunting.
	local horizontalVelocity = Vector3.zero

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

		-- Before descending, arrest any sideways motion and get back over the
		-- exact point that existed when T was pressed.  Once acquired this is a
		-- one-way latch, so tiny sensor noise cannot keep pausing the descent.
		if not self.LandingDescentCommitted then
			if horizontalErrorVector.Magnitude
				<= self.Config.LandingHorizontalAcquireTolerance
				and horizontalSpeed
				<= self.Config.LandingHorizontalAcquireMaxSpeed then

				self.LandingDescentCommitted = true
			else
				commandedY = 0
			end
		end

		-- In the final half-stud, never continue sinking while the airframe is
		-- measurably off the requested touchdown point.  Hold height, centre,
		-- then finish the last few centimetres vertically.
		if heightAboveRest <= self.Config.LandingHorizontalFinalHoldHeight
			and (
				horizontalErrorVector.Magnitude
					> self.Config.LandingHorizontalCaptureTolerance
					or horizontalSpeed
					> self.Config.LandingHorizontalCaptureMaxSpeed
			) then

			commandedY = 0
		end
	end

	return Vector3.new(
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

	local groundY, clearance = self:GetLandingGroundInfo()
	if not groundY or not clearance then
		-- Once final-contact capture has begun, a one-frame raycast miss must NOT
		-- release the latch. Use the last known landing plane instead.
		if self.LandingContactLatched and self.LandingGroundY then
			groundY = self.LandingGroundY
			clearance = self.Base.Position.Y - groundY
		else
			self.TouchdownCandidateTime = nil
			return
		end
	end

	self.LandingGroundY = groundY
	self.LandingTargetY = groundY + self.RestingClearance

	local heightAboveRest = clearance - self.RestingClearance
	self.LandingMeasuredClearance = clearance
	self.LandingHeightAboveRest = math.max(heightAboveRest, 0)

	local velocity = self.Base.AssemblyLinearVelocity
	local horizontalSpeed = Vector3.new(velocity.X, 0, velocity.Z).Magnitude

	local horizontalError = 0
	if self.LandingHorizontalTarget then
		horizontalError = (
			self.LandingHorizontalTarget
			- Vector3.new(self.Base.Position.X, 0, self.Base.Position.Z)
		).Magnitude
	end

	------------------------------------------------
	-- ONE-WAY FINAL-CONTACT CAPTURE
	------------------------------------------------

	if not self.LandingContactLatched
		and heightAboveRest <= self.Config.LandingCaptureHeight
		and horizontalError <= self.Config.LandingHorizontalCaptureTolerance
		and horizontalSpeed <= self.Config.LandingHorizontalCaptureMaxSpeed then

		self.LandingContactLatched = true
		self.LandingContactLatchTime = os.clock()
		self.TouchdownHolding = true
		self.TouchdownCandidateTime = nil
		self.CommandAcceleration = Vector3.zero

		-- Dissipate the remaining tiny amount of vertical kinetic energy before
		-- Roblox's collision response can return it as an upward rebound. This is
		-- the software equivalent of a damped landing leg / shock absorber.
		local current = self.Base.AssemblyLinearVelocity
		self.Base.AssemblyLinearVelocity = Vector3.new(
			current.X,
			math.clamp(current.Y, -self.Config.LandingSettleSpeed, 0),
			current.Z
		)

		if self.LandingVelocityDamper then
			self.LandingVelocityDamper.LineVelocity =
				-self.Config.LandingSettleSpeed
			self.LandingVelocityDamper.Enabled = true
		end

		print("[Drone] Landing gear capture - final settle damping engaged")
	end

	if not self.LandingContactLatched then
		return
	end

	-- IMPORTANT: this latch is intentionally NEVER cleared by a rebound velocity.
	-- V7 cleared TouchdownHolding whenever its instantaneous contactCandidate was
	-- false; a collision impulse therefore restarted the descent and could bounce
	-- over and over. V8 treats touchdown capture as a one-way state transition.
	self.TouchdownHolding = true

	local current = self.Base.AssemblyLinearVelocity
	local dampedY = math.clamp(
		current.Y,
		-self.Config.LandingSettleSpeed,
		0
	)

	self.Base.AssemblyLinearVelocity = Vector3.new(
		current.X,
		dampedY,
		current.Z
	)

	local onGroundPlane =
		heightAboveRest <= self.Config.LandingGroundContactTolerance

	if self.LandingVelocityDamper then
		self.LandingVelocityDamper.Enabled = true
		self.LandingVelocityDamper.LineVelocity =
			onGroundPlane
			and 0
			or -self.Config.LandingSettleSpeed
	end

	local horizontallySettled =
		horizontalError <= self.Config.LandingHorizontalTouchdownTolerance
		and horizontalSpeed <= self.Config.LandingHorizontalTouchdownMaxSpeed

	-- Roblox collision geometry does not always allow the aircraft centre to reach
	-- the mathematically exact pre-takeoff resting clearance again. Once the
	-- one-way landing-gear latch has been engaged for a short period, the craft is
	-- horizontally settled, and it is still inside the original capture cushion,
	-- treat that as genuine physical touchdown instead of holding the motors at
	-- landing RPM forever.
	local capturedLongEnough =
		self.LandingContactLatchTime ~= nil
		and (
			os.clock()
			-
			self.LandingContactLatchTime
		)
		>= (self.Config.LandingCapturedSettleTimeout or 0.90)

	local settledInsideCapture =
		heightAboveRest <= self.Config.LandingCaptureHeight
		and math.abs(current.Y) <= (self.Config.LandingSettleSpeed + 0.02)

	local touchdownConfirmedByGeometry =
		onGroundPlane
		or (
			capturedLongEnough
			and settledInsideCapture
		)

	if not touchdownConfirmedByGeometry
		or not horizontallySettled then

		self.TouchdownCandidateTime = nil
		return
	end

	if not self.TouchdownCandidateTime then
		self.TouchdownCandidateTime = os.clock()

		if not onGroundPlane and capturedLongEnough then
			print(
				"[Drone] Touchdown confirmed by landing-gear settle tolerance - beginning shutdown confirmation"
			)
		end

		return
	end

	if os.clock() - self.TouchdownCandidateTime
		< self.Config.LandingGroundConfirmTime then
		return
	end

	self.ShutdownStartTime = os.clock()
	self.ShutdownStartThrottle =
		tonumber(self.Drone:GetAttribute("PropellerThrottle"))
		or self.Config.LandingPropellerThrottle

	self:SetPowerState("ShuttingDown")
	self:SetMode("ShuttingDown")

	print("[Drone] Touchdown locked - motors spooling down without rebound")
end

------------------------------------------------
-- SHUTDOWN
------------------------------------------------

function FlightController:UpdateShuttingDown()
	if self.State.Mode ~= "ShuttingDown" then
		return
	end

	local elapsed = os.clock() - (self.ShutdownStartTime or os.clock())
	local alpha = math.clamp(
		elapsed / math.max(self.Config.ShutdownDuration, 0.1),
		0,
		1
	)

	local shutdownThrottle =
		self.ShutdownStartThrottle or self.Config.HoverPropellerThrottle

	-- Keep the landing-gear damper active during spool-down so the collision
	-- solver cannot relaunch the aircraft after touchdown.
	if self.LandingContactLatched then
		if self.LandingVelocityDamper then
			self.LandingVelocityDamper.Enabled = true
			self.LandingVelocityDamper.LineVelocity = 0
		end

		local current = self.Base.AssemblyLinearVelocity
		self.Base.AssemblyLinearVelocity = Vector3.new(
			current.X,
			math.clamp(
				current.Y,
				-self.Config.LandingShutdownBounceSpeed,
				0
			),
			current.Z
		)
	end

	-- Rotor RPM follows a minimum-jerk spool-down. At the same time, physical
	-- lift is reduced from weight-support to zero. Because shutdown only starts
	-- after confirmed touchdown, the floor progressively takes the vehicle's
	-- weight instead of the controller simply dropping it from a threshold.
	local spoolDown = 1 - minimumJerk01(alpha)
	self:SetPropellerThrottle(shutdownThrottle * spoolDown)

	local mass = math.max(self.Base.AssemblyMass, 0.001)

	if self.FlightForce then
		self.FlightForce.Enabled = true
		self.FlightForce.Force =
			Vector3.new(
				0,
				mass * workspace.Gravity * spoolDown,
				0
			)
	end

	if self.AeroForce then
		self.AeroForce.Enabled = true
		self.AeroForce.Force = Vector3.zero
	end

	if alpha < 1 then
		return
	end

	self:SetPropellerThrottle(0)

	if self.FlightForce then
		self.FlightForce.Force = Vector3.zero
		self.FlightForce.Enabled = false
	end

	if self.AeroForce then
		self.AeroForce.Force = Vector3.zero
		self.AeroForce.Enabled = false
	end

	if self.AlignOrientation then
		self.AlignOrientation.Enabled = false
	end

	if self.LandingVelocityDamper then
		self.LandingVelocityDamper.LineVelocity = 0
		self.LandingVelocityDamper.Enabled = false
	end

	self.State.Velocity = Vector3.zero
	self.State.TargetVelocity = Vector3.zero
	self.CommandAcceleration = Vector3.zero
	self.AppliedControlAcceleration = Vector3.zero
	self.AerodynamicAcceleration = Vector3.zero
	self:SetControlsEnabled(false)
	self:SetPowerState("Off")
	self:SetMode("PoweredOff")

	self.BootStartTime = nil
	self.ShutdownStartTime = nil
	self.ShutdownStartThrottle = nil
	self.TakeoffTargetY = nil
	self.TakeoffStartY = nil
	self.TakeoffStartTime = nil
	self.LandingTargetY = nil
	self.LandingGroundY = nil
	self.LandingStartY = nil
	self.LandingStartTime = nil
	self.LandingProfileDuration = nil
	self.LandingHorizontalTarget = nil
	self.TouchdownCandidateTime = nil
	self.TouchdownHolding = false
	self.LandingContactLatched = false
	self.LandingContactLatchTime = nil
	self.LandingDescentCommitted = false
	self.Drone:SetAttribute("LandingTargetPosition", nil)
	self.Drone:SetAttribute("RTHPhase", "COMPLETE")
	self.Drone:SetAttribute("RTHDistance", 0)

	print("[Drone] Shutdown complete - rotor thrust fully unloaded on ground")
end

------------------------------------------------
-- AUTO LEVEL
------------------------------------------------

function FlightController:LevelForAutomaticFlight(deltaTime)

	local alpha =
		1 -
		math.exp(
			-self.Config.AutoLevelResponsiveness
			*
			deltaTime
		)

	local normalizedRoll =
		math.atan2(
			math.sin(self.RollAngle),
			math.cos(self.RollAngle)
		)

	self.RollAngle =
		normalizedRoll
		+
		(0 - normalizedRoll)
		*
		alpha

	self.AutoBankAngle +=
		(0 - self.AutoBankAngle)
		*
		alpha

	self.AutoPitchAngle +=
		(0 - self.AutoPitchAngle)
		*
		alpha

	-- During automated takeoff/landing, keep the
	-- commanded look direction horizontal.
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

------------------------------------------------
-- RETURN HOME SUPPORT
------------------------------------------------

function FlightController:GetHomeDistance()

	if not self.HomePosition then
		return math.huge
	end

	return (
		self.Base.Position
		-
			self.HomePosition
	).Magnitude

end


function FlightController:CalculateReturnHomeVelocity()

	if not self.HomePosition then
		return Vector3.zero
	end

	local position = self.Base.Position
	local horizontalOffset = Vector3.new(
		self.HomePosition.X - position.X,
		0,
		self.HomePosition.Z - position.Z
	)

	local distance = horizontalOffset.Magnitude
	local horizontalVelocity = Vector3.zero

	if distance > 0.001 then
		local direction = horizontalOffset.Unit

		-- Two independent limits shape the approach:
		--   1) sqrt(2*a*d) guarantees enough stopping distance.
		--   2) k*d behaves like a position hold near home and smoothly tends to 0.
		-- Unlike V4, the target does NOT become zero several studs away.
		local brakingSpeed = math.sqrt(
			2 * self.Config.ReturnHomeBrakeAcceleration * distance
		)
		local positionSpeed = distance * self.Config.ReturnHomePositionGain
		local speed = math.min(
			self.Config.ReturnHomeSpeed,
			brakingSpeed,
			positionSpeed
		)

		horizontalVelocity = direction * speed

		if distance > self.Config.ReturnHomeStopDistance then
			self.TargetLookDirection = direction
			self.TargetLookUpDirection = Vector3.yAxis
		end
	end

	local holdAltitude = self.RTHAltitude or position.Y
	local altitudeError = holdAltitude - position.Y
	local verticalVelocity = math.clamp(
		altitudeError * self.Config.ReturnHomeVerticalGain,
		-self.Config.ReturnHomeVerticalSpeed,
		self.Config.ReturnHomeVerticalSpeed
	)

	self.Drone:SetAttribute("RTHDistance", distance)
	self.Drone:SetAttribute(
		"RTHPhase",
		distance > 8 and "RETURN" or (distance > self.Config.ReturnHomeStopDistance and "APPROACH" or "HOME HOLD")
	)

	return Vector3.new(horizontalVelocity.X, verticalVelocity, horizontalVelocity.Z)
end

------------------------------------------------
-- TARGET VELOCITY
------------------------------------------------

function FlightController:IsPrecisionMode()

	if self.Drone:GetAttribute("PrecisionMode") ~= true then
		return false
	end

	-- Precision is manual-pilot tuning only. Automatic takeoff, landing,
	-- failsafe and RTH retain their existing dedicated controllers.
	return self.ControlsEnabled
		and (
			self.State.Mode == "Armed"
			or self.State.Mode == "Flying"
		)

end


function FlightController:GetManualFlightTuning()

	local config = self.Config

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

	local config = self.Config
	local input = self.Input
	local tuning = self:GetManualFlightTuning()

	local forward = self.LookDirection
	if not forward or forward.Magnitude < 0.01 then
		forward = self.Base.CFrame.LookVector
	end
	forward = forward.Unit

	local up = self.LookUpDirection
	if not up or up.Magnitude < 0.01 then
		up = self.Base.CFrame.UpVector
	end

	local right = forward:Cross(up.Unit)
	if right.Magnitude > 0.01 then
		right = right.Unit
	else
		right = self.Base.CFrame.RightVector
	end

	local direction = forward * input.Forward + right * input.Strafe
	if direction.Magnitude > 1 then
		direction = direction.Unit
	end

	local targetVelocity = direction * tuning.MaxSpeed
	targetVelocity += Vector3.new(0, input.Vertical * tuning.VerticalSpeed, 0)

	-- No artificial hover wandering. A surveillance quadcopter in position/velocity
	-- hold should look planted, not sway back and forth to appear "alive".
	local altitude = self.Base.Position.Y
	if altitude >= config.MaxAltitude and targetVelocity.Y > 0 then
		targetVelocity = Vector3.new(targetVelocity.X, 0, targetVelocity.Z)
	end

	if altitude <= config.MinAltitude and targetVelocity.Y < 0 then
		targetVelocity = Vector3.new(targetVelocity.X, 0, targetVelocity.Z)
	end

	return targetVelocity
end


function FlightController:CalculateTargetVelocity()

	if self.State.Mode == "PoweredOff"
		or self.State.Mode == "Booting"
		or self.State.Mode == "ShuttingDown" then

		return Vector3.zero

	end

	if self.State.Mode == "Takeoff" then

		return self:CalculateTakeoffVelocity()

	end

	if self.State.Mode == "Landing" then

		return self:CalculateLandingVelocity()

	end

	if self.State.Mode == "ReturnToHome" then

		return self:CalculateReturnHomeVelocity()

	end

	if self.State.Mode == "Failsafe" then

		return Vector3.zero

	end

	return self:CalculateNormalVelocity()

end

------------------------------------------------
-- ORIENTATION
------------------------------------------------

function FlightController:UpdateLookOrientation(deltaTime)

	local tuning = self:GetManualFlightTuning()

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
		targetUp = Vector3.yAxis
	end

	------------------------------------------------
	-- FULL ACRO STEERING ORIENTATION
	------------------------------------------------

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
		currentForward = targetForward
	end

	local currentUp =
		self.LookUpDirection
		or targetUp

	local currentRight =
		currentForward.Unit:Cross(
			currentUp.Unit
		)

	if currentRight.Magnitude < 0.001 then
		currentRight = targetRight
	else
		currentRight = currentRight.Unit
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

	-- Precision lowers angular authority rather than adding camera lag.
	self.AlignOrientation.MaxAngularVelocity =
		tuning.MaxAngularVelocity

	local lookAlpha =
		1 -
		math.exp(
			-tuning.LookResponsiveness
			*
			deltaTime
		)

	local steering =
		currentSteering

	local automaticSteering = self.State.Mode == "ReturnToHome"

	if self.ControlsEnabled or automaticSteering then

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

	------------------------------------------------
	-- MANUAL Q / E ROLL
	------------------------------------------------

	if self.ControlsEnabled then

		self.RollAngle +=
			(self.Input.Roll or 0)
			*
			tuning.RollSpeed
			*
			deltaTime

	end

	if math.abs(self.RollAngle) >
		math.pi * 2 then

		self.RollAngle =
			self.RollAngle
			%
			(math.pi * 2)

	end

	------------------------------------------------
	-- ACCELERATION-DERIVED BODY LEAN
	------------------------------------------------

	local targetBank = 0
	local targetPitch = 0

	if self.ControlsEnabled or self.State.Mode == "ReturnToHome" then
		local forwardAxis =
			self.LookDirection.Magnitude > 0.01
			and self.LookDirection.Unit
			or self.Base.CFrame.LookVector

		local upAxis =
			self.LookUpDirection.Magnitude > 0.01
			and self.LookUpDirection.Unit
			or self.Base.CFrame.UpVector

		local rightAxis =
			forwardAxis:Cross(upAxis)

		if rightAxis.Magnitude < 0.01 then
			rightAxis = self.Base.CFrame.RightVector
		else
			rightAxis = rightAxis.Unit
		end

		-- Use measured, low-pass acceleration for body attitude. Command acceleration
		-- can change sign quickly around a setpoint and was the source of the rapid
		-- forward/back visual rocking.
		local acceleration =
			self.FilteredAcceleration or Vector3.zero

		local forwardAcceleration =
			acceleration:Dot(forwardAxis)

		local rightAcceleration =
			acceleration:Dot(rightAxis)

		-- tan(theta) = a_horizontal / g. This produces stronger lean while
		-- accelerating, an opposite lean while braking, and naturally settles
		-- toward level flight once acceleration approaches zero.
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
		1 -
		math.exp(
			-tuning.AutoLeanResponsiveness
			*
			deltaTime
		)

	self.AutoBankAngle +=
		(targetBank - self.AutoBankAngle)
		*
		leanAlpha

	self.AutoPitchAngle +=
		(targetPitch - self.AutoPitchAngle)
		*
		leanAlpha

	------------------------------------------------
	-- APPLY BODY ORIENTATION
	------------------------------------------------

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

------------------------------------------------
-- FAILSAFE SUPPORT
------------------------------------------------

function FlightController:BeginFailsafe()

	if self.State.FailsafeTriggered then
		return
	end

	if not self.ControlsEnabled then
		return
	end

	self.State.FailsafeTriggered =
		true

	-- Hold the altitude that was actually safe at the moment the link failed.
	-- Using an absolute world-Y RTH altitude caused unnecessary vertical jumps.
	self.RTHAltitude = self.Base.Position.Y
	self.CommandAcceleration = Vector3.zero
	self.Drone:SetAttribute("RTHPhase", "FAILSAFE")

	self:SetControlsEnabled(
		false
	)

	self:SetPowerState(
		"Failsafe"
	)

	self:SetMode(
		"Failsafe"
	)

	print(
		"[Drone] Pilot signal lost"
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

	local horizontalOffset = Vector3.new(
		self.HomePosition.X - self.Base.Position.X,
		0,
		self.HomePosition.Z - self.Base.Position.Z
	)

	local horizontalDistance = horizontalOffset.Magnitude
	local velocity = self.Base.AssemblyLinearVelocity
	local horizontalSpeed = Vector3.new(velocity.X, 0, velocity.Z).Magnitude

	-- Capture only when the aircraft is genuinely centred over the launch point
	-- and nearly stationary. V4 allowed landing to start several studs away.
	if horizontalDistance > self.Config.ReturnHomeStopDistance
		or horizontalSpeed > self.Config.ReturnHomeArrivalSpeed then
		return
	end

	-- Use the floor height stored at TAKEOFF, not a short ray from the current
	-- RTH altitude. That old ray could miss the floor entirely above 50 studs and
	-- leave the aircraft hovering forever.
	local groundY = self.HomeGroundY
	if typeof(groundY) ~= "number" then
		local probedY = self:GetGroundInfoAt(
			self.HomePosition,
			math.max(self.Base.Position.Y - self.HomePosition.Y + 30, 80)
		)
		groundY = probedY
	end

	if typeof(groundY) ~= "number" then
		warn("[Drone] RTH reached home X/Z but home ground height is unavailable")
		return
	end

	self:StartLandingProfile(groundY, self.HomePosition)
	self.Drone:SetAttribute("RTHPhase", "LANDING")
	self:SetPowerState("Landing")
	self:SetMode("Landing")
	print("[Drone] Exact home position acquired - commencing precision landing")
end

------------------------------------------------
-- VISUAL PROPELLER LOAD
------------------------------------------------

function FlightController:UpdatePropellerThrottle()
	local mode = self.State.Mode

	-- Boot and shutdown own their explicit spool curves.
	if mode == "PoweredOff"
		or mode == "Booting"
		or mode == "ShuttingDown" then

		return
	end

	if mode == "Failsafe" then
		self:SetPropellerThrottle(
			math.min(1, self.Config.HoverPropellerThrottle + 0.08)
		)
		return
	end

	local acceleration = self.CommandAcceleration or Vector3.zero
	local gravity = math.max(workspace.Gravity, 0.001)

	-- Specific thrust required from the rotors is the commanded acceleration
	-- plus the acceleration required to support the craft against gravity.
	local requiredSpecificForce =
		acceleration + Vector3.new(0, gravity, 0)

	local thrustRatio =
		requiredSpecificForce.Magnitude / gravity

	-- For a propeller, thrust is approximately proportional to RPM^2. Therefore
	-- the RPM fraction follows sqrt(thrust ratio), not thrust ratio directly.
	local idealRotorThrottle =
		self.Config.HoverPropellerThrottle
		* math.sqrt(math.clamp(thrustRatio, 0.20, 2.40))

	local throttle =
		self.Config.HoverPropellerThrottle
		+ (idealRotorThrottle - self.Config.HoverPropellerThrottle)
		* self.Config.RotorThrustVisualGain

	local forward =
		self.LookDirection.Magnitude > 0.01
		and self.LookDirection.Unit
		or self.Base.CFrame.LookVector

	local up =
		self.LookUpDirection.Magnitude > 0.01
		and self.LookUpDirection.Unit
		or self.Base.CFrame.UpVector

	local right = forward:Cross(up)
	if right.Magnitude < 0.01 then
		right = self.Base.CFrame.RightVector
	else
		right = right.Unit
	end

	local forwardAcceleration = acceleration:Dot(forward)
	local rightAcceleration = acceleration:Dot(right)

	self.Drone:SetAttribute(
		"PropellerForwardDemand",
		math.clamp(
			forwardAcceleration / math.max(self.Config.Acceleration, 0.001),
			-1,
			1
		)
	)

	self.Drone:SetAttribute(
		"PropellerStrafeDemand",
		math.clamp(
			rightAcceleration / math.max(self.Config.Acceleration, 0.001),
			-1,
			1
		)
	)

	self.Drone:SetAttribute(
		"PropellerVerticalDemand",
		math.clamp(
			acceleration.Y / math.max(self.Config.VerticalAcceleration, 0.001),
			-1,
			1
		)
	)

	self.Drone:SetAttribute(
		"PropellerRollDemand",
		self.ControlsEnabled and self.Input.Roll or 0
	)

	-- Near the floor the same rotor speed produces slightly more effective lift;
	-- visually reduce the required RPM a touch to suggest ground effect.
	local groundY = self:GetGroundInfo()
	if groundY then
		local height = math.max(self.Base.Position.Y - groundY, 0)
		local groundEffect = math.clamp(
			1 - height / self.Config.GroundEffectHeight,
			0,
			1
		)
		throttle -= groundEffect * self.Config.GroundEffectThrottleReduction
	end

	-- At the instant of liftoff retain the already-achieved ground power, then
	-- blend naturally toward the calculated rotor requirement during the climb.
	if mode == "Takeoff" and self.TakeoffStartTime then
		local takeoffAlpha = math.clamp(
			(os.clock() - self.TakeoffStartTime)
				/ math.max(self.Config.TakeoffProfileDuration, 0.1),
			0,
			1
		)
		local retainedLiftPower =
			self.Config.TakeoffPropellerThrottle
			+ (self.Config.HoverPropellerThrottle - self.Config.TakeoffPropellerThrottle)
			* minimumJerk01(takeoffAlpha)
		throttle = math.max(throttle, retainedLiftPower)
	elseif mode == "ReturnToHome" then
		throttle = math.max(throttle, self.Config.ReturnHomePropellerThrottle)
	end

	-- Roll requires differential motor authority even if translational load is low.
	throttle += math.abs(self.Input.Roll or 0) * self.Config.RollPropellerBoost

	self:SetPropellerThrottle(math.clamp(throttle, 0.56, 1))
end

------------------------------------------------
-- MOVEMENT SMOOTHING
------------------------------------------------

function FlightController:ApplyVelocity(
	targetVelocity,
	deltaTime
)

	self.State.TargetVelocity =
		targetVelocity

	if deltaTime <= 0 then
		return
	end

	local tuning = self:GetManualFlightTuning()

	-- Booting and powered-off states intentionally have no flight force. The
	-- ground carries the drone while the rotors visually spool.
	if self.State.Mode == "PoweredOff"
		or self.State.Mode == "Booting" then

		if self.FlightForce then
			self.FlightForce.Force = Vector3.zero
			self.FlightForce.Enabled = false
		end

		if self.AeroForce then
			self.AeroForce.Force = Vector3.zero
			self.AeroForce.Enabled = false
		end

		return
	end

	-- Shutdown owns its own continuously decreasing lift curve so the normal
	-- controller must not overwrite it with hover thrust.
	if self.State.Mode == "ShuttingDown" then
		return
	end

	------------------------------------------------
	-- FINAL LANDING-GEAR DAMPER
	------------------------------------------------

	if self.State.Mode == "Landing" and self.LandingContactLatched then
		local mass = math.max(self.Base.AssemblyMass, 0.001)
		local gravity = math.max(workspace.Gravity, 0.001)
		local current = self.Base.AssemblyLinearVelocity

		-- Remove upward rebound completely and cap the final downward sink.
		local settleY = math.clamp(
			current.Y,
			-self.Config.LandingSettleSpeed,
			0
		)

		local settleX = math.abs(current.X) < 0.05 and 0 or current.X
		local settleZ = math.abs(current.Z) < 0.05 and 0 or current.Z

		self.Base.AssemblyLinearVelocity = Vector3.new(
			settleX,
			settleY,
			settleZ
		)

		if self.LandingVelocityDamper then
			local capturedOnPlane =
				(self.LandingHeightAboveRest or math.huge)
				<= self.Config.LandingGroundContactTolerance

			self.LandingVelocityDamper.Enabled = true
			self.LandingVelocityDamper.LineVelocity =
				capturedOnPlane
				and 0
				or -self.Config.LandingSettleSpeed
		end

		-- Slightly under-support weight so the craft settles onto the feet instead
		-- of hovering just above the stored rest plane.
		if self.FlightForce then
			self.FlightForce.Enabled = true
			self.FlightForce.Force = Vector3.new(
				0,
				mass * gravity * self.Config.LandingSettleLiftRatio,
				0
			)
		end

		if self.AeroForce then
			self.AeroForce.Enabled = true
			self.AeroForce.Force = Vector3.zero
		end

		self.CommandAcceleration = Vector3.zero
		self.AppliedControlAcceleration = Vector3.zero
		self.State.Velocity = self.Base.AssemblyLinearVelocity
		return
	end

	local measuredVelocity =
		self.Base.AssemblyLinearVelocity

	if not finiteVector(measuredVelocity) then
		measuredVelocity = self.State.Velocity
	end

	if not finiteVector(measuredVelocity) then
		measuredVelocity = Vector3.zero
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
		horizontalTarget - horizontalCurrent

	local currentSpeed =
		horizontalCurrent.Magnitude

	local targetSpeed =
		horizontalTarget.Magnitude

	-- Suppress solver noise around a stationary hover. This is not a fake drift
	-- correction; it simply prevents tiny replicated velocity errors from making
	-- the controller alternate thrust direction every frame.
	if targetSpeed < 0.01
		and currentSpeed < self.Config.HoverSpeedDeadband then
		horizontalError = Vector3.zero
	end

	------------------------------------------------
	-- AERODYNAMIC MODEL
	------------------------------------------------

	-- Drag is applied as a real second force, not just hidden inside the
	-- velocity target. Quadratic drag dominates at higher speed.
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

	------------------------------------------------
	-- VELOCITY FEEDBACK -> DESIRED NET ACCELERATION
	------------------------------------------------

	local accelerationLimit =
		tuning.Acceleration

	if currentSpeed > 0.5
		and targetSpeed > 0.5
		and horizontalCurrent:Dot(horizontalTarget) < 0 then

		accelerationLimit =
			tuning.ReverseAcceleration

	elseif targetSpeed + 0.5 < currentSpeed then

		accelerationLimit =
			tuning.Deceleration

	end

	-- Respect the amount of horizontal acceleration a tilted multirotor can
	-- create while still supporting its weight.
	local gravity =
		math.max(workspace.Gravity, 0.001)

	local maxTilt =
		math.max(
			tuning.AutoPitchAngle,
			tuning.AutoBankAngle
		)

	local tiltAccelerationLimit =
		gravity * math.tan(maxTilt)

	accelerationLimit =
		math.min(
			accelerationLimit,
			tiltAccelerationLimit
		)

	local desiredNetHorizontalAcceleration =
		clampMagnitude(
			horizontalError * tuning.VelocityResponse,
			accelerationLimit
		)

	------------------------------------------------
	-- LANDING X/Z POSITION CONTROLLER
	------------------------------------------------

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

		-- Over-damped PD position hold:
		--
		-- a = wn^2 * positionError - 2*zeta*wn * velocity
		--
		-- The derivative term removes lateral momentum instead of overshooting the
		-- landing point and correcting back the other way.
		local landingHorizontalAcceleration =
			positionError * (wn * wn)
		-
			horizontalCurrent * (2 * zeta * wn)

		desiredNetHorizontalAcceleration =
			clampMagnitude(
				landingHorizontalAcceleration,
				self.Config.LandingHorizontalMaxAcceleration
			)
	end

	local verticalError =
		targetVelocity.Y - measuredVelocity.Y

	local verticalLimit =
		tuning.VerticalAcceleration

	if math.abs(targetVelocity.Y) + 0.25 < math.abs(measuredVelocity.Y)
		or targetVelocity.Y * measuredVelocity.Y < 0 then

		verticalLimit =
			tuning.VerticalDeceleration
	end

	local desiredNetVerticalAcceleration =
		math.clamp(
			verticalError * tuning.VelocityResponse,
			-verticalLimit,
			verticalLimit
		)

	------------------------------------------------
	-- RANGEFINDER FINAL-LANDING CONTROLLER
	------------------------------------------------

	if self.State.Mode == "Landing"
		and self.LandingHeightAboveRest ~= nil
		and self.LandingHeightAboveRest <= self.Config.LandingFinalControllerHeight then

		local height =
			math.max(
				self.LandingHeightAboveRest,
				0
			)

		-- Critically-damped second-order controller:
		--
		--     a = -wn^2 * h - 2*zeta*wn * v
		--
		-- h is the remaining height above the known resting clearance and v is
		-- vertical velocity. The damping term becomes upward braking while the
		-- aircraft is descending, killing sink rate BEFORE physical contact.
		local wn =
			self.Config.LandingFinalNaturalFrequency

		local zeta =
			self.Config.LandingFinalDampingRatio

		local positionTerm =
			-(wn * wn) * height

		local dampingTerm =
			-(2 * zeta * wn) * measuredVelocity.Y

		local finalAcceleration =
			positionTerm + dampingTerm

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

	------------------------------------------------
	-- DRAG FEED-FORWARD COMPENSATION
	------------------------------------------------

	-- The rotors must overcome aerodynamic drag to maintain constant speed.
	-- Adding -drag here means the aircraft still reaches the requested airspeed
	-- without a permanent controller error, while AeroForce applies the actual
	-- opposing drag separately.
	local requiredControlAcceleration =
		desiredNetAcceleration
	-
		dragAcceleration

	------------------------------------------------
	-- MOTOR / THRUST JERK LIMITING
	------------------------------------------------

	local currentAcceleration =
		self.CommandAcceleration or Vector3.zero

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
			tuning.HorizontalJerk * deltaTime
		)

	local verticalJerk =
		tuning.VerticalJerk

	if self.State.Mode == "Landing"
		and self.LandingHeightAboveRest ~= nil
		and self.LandingHeightAboveRest <= self.Config.LandingFinalControllerHeight then
		-- Landing flare must be able to remove sink rate promptly; this is still
		-- rate-limited, just with more actuator authority near the surface.
		verticalJerk =
			math.max(verticalJerk, 900)
	end

	local newVerticalAcceleration =
		moveNumberTowards(
			currentAcceleration.Y,
			requiredControlAcceleration.Y,
			verticalJerk * deltaTime
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

	------------------------------------------------
	-- F = m a MULTIROTOR THRUST
	------------------------------------------------

	local mass =
		math.max(
			self.Base.AssemblyMass,
			0.001
		)

	-- Rotor thrust must first cancel gravity, then supply the commanded control
	-- acceleration. Cap total specific thrust using a thrust-to-weight ratio.
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
		self.FlightForce.Enabled = true
		self.FlightForce.Force =
			requestedSpecificThrust * mass
	end

	if self.AeroForce then
		self.AeroForce.Enabled = true
		self.AeroForce.Force =
			dragAcceleration * mass
	end

	------------------------------------------------
	-- MEASURED PHYSICS TELEMETRY
	------------------------------------------------

	local rawAcceleration =
		(
			measuredVelocity
			-
			(self.LastMeasuredVelocity or measuredVelocity)
		)
		/
		math.max(deltaTime, 1 / 240)

	rawAcceleration =
		clampMagnitude(
			rawAcceleration,
			tuning.MaxControlAcceleration * 3
		)

	local accelerationAlpha =
		1 - math.exp(-7 * deltaTime)

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
		self.PhysicsTelemetryTimer = 0

		local supportAcceleration =
			self.FilteredAcceleration
			+
			Vector3.new(0, gravity, 0)

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
			supportAcceleration.Magnitude / gravity
		)

		self.Drone:SetAttribute(
			"FlightDrag",
			dragAcceleration.Magnitude
		)

		self.Drone:SetAttribute(
			"FlightThrustRatio",
			requestedSpecificThrust.Magnitude / gravity
		)
	end

end

------------------------------------------------
-- UPDATE LOOP
------------------------------------------------

function FlightController:Update(deltaTime)

	if not self.State.Active then
		return
	end

	self.State.FlightTime +=
		deltaTime

	------------------------------------------------
	-- POWER STATE MACHINE
	------------------------------------------------

	self:UpdateBooting()
	self:UpdateTakeoff()
	self:UpdateLanding()
	self:UpdateShuttingDown()

	------------------------------------------------
	-- FAILSAFE STATE MACHINE
	------------------------------------------------

	self:UpdateFailsafe()
	self:UpdateReturnHome()

	------------------------------------------------
	-- ORIENTATION
	------------------------------------------------

	self:UpdateLookOrientation(
		deltaTime
	)

	------------------------------------------------
	-- NORMAL FLYING STATE
	------------------------------------------------

	if self.ControlsEnabled then

		local hasInput =
			math.abs(self.Input.Forward) > 0.01
			or math.abs(self.Input.Strafe) > 0.01
			or math.abs(self.Input.Vertical) > 0.01
			or math.abs(self.Input.Roll) > 0.01

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

	------------------------------------------------
	-- TEMPORARY SIGNAL TIMEOUT
	------------------------------------------------

	local elapsed =
		os.clock()
	-
		self.StartTime

	if self.ControlsEnabled
		and elapsed >
		self.Config.StartupGracePeriod then

		local signalAge =
			os.clock()
		-
			self.LastInputTime

		if signalAge >
			self.Config.InputTimeout then

			self:BeginFailsafe()

		end

	end

	------------------------------------------------
	-- VELOCITY
	------------------------------------------------

	local targetVelocity =
		self:CalculateTargetVelocity()

	self:ApplyVelocity(
		targetVelocity,
		deltaTime
	)

	self:UpdatePropellerThrottle()

end

------------------------------------------------
-- START / STOP
------------------------------------------------

function FlightController:Start()

	if self.State.Active then
		return
	end

	self:CreatePhysics()
	self:ConfigureLandingContactPhysics()

	pcall(function()

		self.Base:SetNetworkOwner(nil)

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
	self.HomeGroundY = nil
	self.LandingHorizontalTarget = nil
	self.Drone:SetAttribute("DroneHomePosition", self.HomePosition)
	self.Drone:SetAttribute("DroneHomeGroundY", nil)
	self.Drone:SetAttribute("RTHPhase", "STANDBY")
	self.Drone:SetAttribute("RTHDistance", 0)

	self:SetControlsEnabled(false)

	self:SetPropellerThrottle(0)

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

	-- Force control belongs in PreSimulation so thrust, landing range data and
	-- braking commands are applied BEFORE Roblox integrates the next physics
	-- step. Heartbeat runs after physics and was allowing the collision solver
	-- to see the ground before the landing controller could react.
	self.Connection =
		RunService.PreSimulation:Connect(
			function(deltaTime)

				self:Update(
					deltaTime
				)

			end
		)

	print(
		"[Drone] Stable quad V8 online - one-way touchdown latch + landing-gear damper"
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

		self.Base:SetNetworkOwner(nil)

	end)

	print(
		"[Drone] Flight controller stopped"
	)

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

print("[Drone] Precision flight profile support online")

return FlightController

```
