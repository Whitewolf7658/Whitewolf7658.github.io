[← Back to Home](../README.md)

# Drone Flight Controller (FlightController)

Selected code from the drone's flight controller. The full module also handles inputs, flight states, failsafes, and shutdowns.

## Flight physics

Portion of `FlightController:ApplyVelocity()`. Calculates aerodynamic drag, limits acceleration and jerk, and applies thrust using `VectorForce`.

```luau
local measuredVelocity = self.Base.AssemblyLinearVelocity
if not finiteVector(measuredVelocity) then
    measuredVelocity = self.State.Velocity
end
if not finiteVector(measuredVelocity) then
    measuredVelocity = Vector3.zero
end
self.State.Velocity = measuredVelocity

local horizontalCurrent = Vector3.new(measuredVelocity.X, 0, measuredVelocity.Z)
local horizontalTarget = Vector3.new(targetVelocity.X, 0, targetVelocity.Z)
local horizontalError = horizontalTarget - horizontalCurrent
local currentSpeed = horizontalCurrent.Magnitude
local targetSpeed = horizontalTarget.Magnitude

if targetSpeed < 0.01 and currentSpeed < self.Config.HoverSpeedDeadband then
    horizontalError = Vector3.zero
end

local dragAcceleration = Vector3.zero
if currentSpeed > 0.01 then
    dragAcceleration = -horizontalCurrent * self.Config.LinearDrag
        - horizontalCurrent.Unit * currentSpeed * currentSpeed * self.Config.QuadraticDrag
end
self.AerodynamicAcceleration = dragAcceleration

local accelerationLimit = tuning.Acceleration
if currentSpeed > 0.5 and targetSpeed > 0.5
    and horizontalCurrent:Dot(horizontalTarget) < 0 then
    accelerationLimit = tuning.ReverseAcceleration
elseif targetSpeed + 0.5 < currentSpeed then
    accelerationLimit = tuning.Deceleration
end

local gravity = math.max(workspace.Gravity, 0.001)
local maxTilt = math.max(tuning.AutoPitchAngle, tuning.AutoBankAngle)
local tiltAccelerationLimit = gravity * math.tan(maxTilt)
accelerationLimit = math.min(accelerationLimit, tiltAccelerationLimit)

local desiredNetHorizontalAcceleration = clampMagnitude(
    horizontalError * tuning.VelocityResponse,
    accelerationLimit
)

if self.State.Mode == "Landing"
    and self.LandingHorizontalTarget
    and not self.LandingContactLatched then
    local positionError = self.LandingHorizontalTarget - Vector3.new(
        self.Base.Position.X,
        0,
        self.Base.Position.Z
    )
    local wn = self.Config.LandingHorizontalNaturalFrequency
    local zeta = self.Config.LandingHorizontalDampingRatio
    local landingHorizontalAcceleration = positionError * (wn * wn)
        - horizontalCurrent * (2 * zeta * wn)

    desiredNetHorizontalAcceleration = clampMagnitude(
        landingHorizontalAcceleration,
        self.Config.LandingHorizontalMaxAcceleration
    )
end

local verticalError = targetVelocity.Y - measuredVelocity.Y
local verticalLimit = tuning.VerticalAcceleration
if math.abs(targetVelocity.Y) + 0.25 < math.abs(measuredVelocity.Y)
    or targetVelocity.Y * measuredVelocity.Y < 0 then
    verticalLimit = tuning.VerticalDeceleration
end

local desiredNetVerticalAcceleration = math.clamp(
    verticalError * tuning.VelocityResponse,
    -verticalLimit,
    verticalLimit
)

if self.State.Mode == "Landing"
    and self.LandingHeightAboveRest ~= nil
    and self.LandingHeightAboveRest <= self.Config.LandingFinalControllerHeight then
    local height = math.max(self.LandingHeightAboveRest, 0)
    local wn = self.Config.LandingFinalNaturalFrequency
    local zeta = self.Config.LandingFinalDampingRatio
    local positionTerm = -(wn * wn) * height
    local dampingTerm = -(2 * zeta * wn) * measuredVelocity.Y
    local finalAcceleration = positionTerm + dampingTerm

    desiredNetVerticalAcceleration = math.clamp(
        finalAcceleration,
        -self.Config.LandingFinalMaxDownAcceleration,
        self.Config.LandingFinalMaxBrakeAcceleration
    )
end

local desiredNetAcceleration = Vector3.new(
    desiredNetHorizontalAcceleration.X,
    desiredNetVerticalAcceleration,
    desiredNetHorizontalAcceleration.Z
)
local requiredControlAcceleration = desiredNetAcceleration - dragAcceleration
local currentAcceleration = self.CommandAcceleration or Vector3.zero
local currentHorizontalAcceleration = Vector3.new(
    currentAcceleration.X,
    0,
    currentAcceleration.Z
)
local targetHorizontalControl = Vector3.new(
    requiredControlAcceleration.X,
    0,
    requiredControlAcceleration.Z
)

local newHorizontalAcceleration = moveVectorTowards(
    currentHorizontalAcceleration,
    targetHorizontalControl,
    tuning.HorizontalJerk * deltaTime
)

local verticalJerk = tuning.VerticalJerk
if self.State.Mode == "Landing"
    and self.LandingHeightAboveRest ~= nil
    and self.LandingHeightAboveRest <= self.Config.LandingFinalControllerHeight then
    verticalJerk = math.max(verticalJerk, 900)
end

local newVerticalAcceleration = moveNumberTowards(
    currentAcceleration.Y,
    requiredControlAcceleration.Y,
    verticalJerk * deltaTime
)
local controlAcceleration = Vector3.new(
    newHorizontalAcceleration.X,
    newVerticalAcceleration,
    newHorizontalAcceleration.Z
)
controlAcceleration = clampMagnitude(controlAcceleration, tuning.MaxControlAcceleration)
self.CommandAcceleration = controlAcceleration
self.AppliedControlAcceleration = controlAcceleration

local mass = math.max(self.Base.AssemblyMass, 0.001)
local requestedSpecificThrust = controlAcceleration + Vector3.new(0, gravity, 0)
local maximumSpecificThrust = gravity * self.Config.MaxThrustToWeight
requestedSpecificThrust = clampMagnitude(requestedSpecificThrust, maximumSpecificThrust)

if self.FlightForce then
    self.FlightForce.Enabled = true
    self.FlightForce.Force = requestedSpecificThrust * mass
end
if self.AeroForce then
    self.AeroForce.Enabled = true
    self.AeroForce.Force = dragAcceleration * mass
end
```

## Steering and banking

`FlightController:UpdateLookOrientation()` handles smooth steering, manual roll, and automatic banking and pitch based on acceleration.

```luau
function FlightController:UpdateLookOrientation(deltaTime)
    local tuning = self:GetManualFlightTuning()
    if not self.AlignOrientation or not self.AlignOrientation.Enabled then
        return
    end
    if not self.TargetLookDirection then
        return
    end

    if self.State.Mode == "Takeoff" or self.State.Mode == "Landing" then
        self:LevelForAutomaticFlight(deltaTime)
    end

    local targetForward = self.TargetLookDirection
    if targetForward.Magnitude < 0.01 then
        return
    end
    targetForward = targetForward.Unit

    local targetUp = self.TargetLookUpDirection or Vector3.yAxis
    if targetUp.Magnitude < 0.01 then
        targetUp = Vector3.yAxis
    end

    local targetRight = targetForward:Cross(targetUp.Unit)
    if targetRight.Magnitude < 0.001 then
        targetRight = self.Base.CFrame.RightVector
    else
        targetRight = targetRight.Unit
    end
    targetUp = targetRight:Cross(targetForward).Unit

    local currentForward = self.LookDirection
    if currentForward.Magnitude < 0.01 then
        currentForward = targetForward
    end

    local currentUp = self.LookUpDirection or targetUp
    local currentRight = currentForward.Unit:Cross(currentUp.Unit)
    if currentRight.Magnitude < 0.001 then
        currentRight = targetRight
    else
        currentRight = currentRight.Unit
    end
    currentUp = currentRight:Cross(currentForward.Unit).Unit

    local currentSteering = CFrame.lookAt(
        Vector3.zero,
        currentForward.Unit,
        currentUp
    )
    local targetSteering = CFrame.lookAt(
        Vector3.zero,
        targetForward,
        targetUp
    )

    self.AlignOrientation.MaxAngularVelocity = tuning.MaxAngularVelocity
    local lookAlpha = 1 - math.exp(-tuning.LookResponsiveness * deltaTime)
    local steering = currentSteering
    local automaticSteering = self.State.Mode == "ReturnToHome"

    if self.ControlsEnabled or automaticSteering then
        steering = currentSteering:Lerp(targetSteering, lookAlpha)
    end

    self.LookDirection = steering.LookVector
    self.LookUpDirection = steering.UpVector

    if self.ControlsEnabled then
        self.RollAngle += (self.Input.Roll or 0) * tuning.RollSpeed * deltaTime
    end

    if math.abs(self.RollAngle) > math.pi * 2 then
        self.RollAngle = self.RollAngle % (math.pi * 2)
    end

    local targetBank = 0
    local targetPitch = 0

    if self.ControlsEnabled or self.State.Mode == "ReturnToHome" then
        local forwardAxis = self.LookDirection.Magnitude > 0.01
            and self.LookDirection.Unit
            or self.Base.CFrame.LookVector
        local upAxis = self.LookUpDirection.Magnitude > 0.01
            and self.LookUpDirection.Unit
            or self.Base.CFrame.UpVector

        local rightAxis = forwardAxis:Cross(upAxis)
        if rightAxis.Magnitude < 0.01 then
            rightAxis = self.Base.CFrame.RightVector
        else
            rightAxis = rightAxis.Unit
        end

        local acceleration = self.FilteredAcceleration or Vector3.zero
        local forwardAcceleration = acceleration:Dot(forwardAxis)
        local rightAcceleration = acceleration:Dot(rightAxis)

        targetPitch = math.clamp(
            -math.atan2(forwardAcceleration, workspace.Gravity),
            -tuning.AutoPitchAngle,
            tuning.AutoPitchAngle
        )
        targetBank = math.clamp(
            -math.atan2(rightAcceleration, workspace.Gravity),
            -tuning.AutoBankAngle,
            tuning.AutoBankAngle
        )
    end

    local leanAlpha = 1 - math.exp(-tuning.AutoLeanResponsiveness * deltaTime)
    self.AutoBankAngle += (targetBank - self.AutoBankAngle) * leanAlpha
    self.AutoPitchAngle += (targetPitch - self.AutoPitchAngle) * leanAlpha

    local lookCFrame = CFrame.lookAt(
        self.Base.Position,
        self.Base.Position + self.LookDirection,
        self.LookUpDirection
    )
    self.AlignOrientation.CFrame = lookCFrame * CFrame.Angles(
        self.AutoPitchAngle,
        0,
        self.RollAngle + self.AutoBankAngle
    )
end
```

## Automatic takeoff

`FlightController:CalculateTakeoffVelocity()` uses a minimum-jerk trajectory and position correction to control lift-off.

```luau
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

    local desiredPosition = self.TakeoffStartY + travel * minimumJerk01(alpha)
    local feedForwardVelocity = travel * minimumJerkDerivative01(alpha) / duration
    local positionError = desiredPosition - self.Base.Position.Y
    local commandedY = feedForwardVelocity + positionError * self.Config.TakeoffTrackingGain

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
```

## Landing surface detection

`FlightController:GetLandingGroundInfo()` checks five points beneath the drone to detect ground height and clearance.

```luau
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
```

## Landing and descent

`FlightController:CalculateLandingVelocity()` adjusts descent speed based on ground clearance and pauses descent until horizontal alignment is within tolerance.

```luau
function FlightController:CalculateLandingVelocity()
    if not self.LandingTargetY then
        return Vector3.zero
    end
    if self.TouchdownHolding then
        return Vector3.zero
    end

    local groundY, clearance = self:GetLandingGroundInfo()
    if not groundY or not clearance then
        groundY = self.LandingGroundY
        if groundY then
            clearance = self.Base.Position.Y - groundY
        else
            clearance = math.max(
                self.Base.Position.Y - self.LandingTargetY,
                self.RestingClearance
            )
        end
    else
        self.LandingGroundY = groundY
        self.LandingTargetY = groundY + self.RestingClearance
    end

    local heightAboveRest = math.max(clearance - self.RestingClearance, 0)
    self.LandingMeasuredClearance = clearance
    self.LandingHeightAboveRest = heightAboveRest

    local usableDistance = math.max(
        heightAboveRest - self.Config.LandingFlareMargin - self.Config.LandingTouchdownTolerance,
        0
    )
    local safeDescentSpeed = math.sqrt(
        2 * self.Config.LandingFlareDeceleration * usableDistance
    )
    local descentSpeed = math.min(self.Config.LandingSpeed, safeDescentSpeed)

    if heightAboveRest <= self.Config.LandingFinalApproachHeight then
        local finalSpeed = self.Config.LandingFinalDescentSpeed
            + math.max(heightAboveRest - self.Config.LandingTouchdownTolerance, 0)
            * self.Config.LandingFinalApproachGain

        descentSpeed = math.min(
            descentSpeed,
            self.Config.LandingMaxFinalDescentSpeed,
            finalSpeed
        )
    end

    if heightAboveRest > self.Config.LandingTouchdownTolerance then
        descentSpeed = math.max(descentSpeed, self.Config.LandingFinalDescentSpeed)
    else
        descentSpeed = 0
    end

    local commandedY = -descentSpeed
    local horizontalVelocity = Vector3.zero

    if self.LandingHorizontalTarget then
        local horizontalErrorVector = self.LandingHorizontalTarget - Vector3.new(
            self.Base.Position.X,
            0,
            self.Base.Position.Z
        )
        local horizontalSpeed = Vector3.new(
            self.Base.AssemblyLinearVelocity.X,
            0,
            self.Base.AssemblyLinearVelocity.Z
        ).Magnitude

        if not self.LandingDescentCommitted then
            if horizontalErrorVector.Magnitude <= self.Config.LandingHorizontalAcquireTolerance
                and horizontalSpeed <= self.Config.LandingHorizontalAcquireMaxSpeed then
                self.LandingDescentCommitted = true
            else
                commandedY = 0
            end
        end

        if heightAboveRest <= self.Config.LandingHorizontalFinalHoldHeight
            and (
                horizontalErrorVector.Magnitude > self.Config.LandingHorizontalCaptureTolerance
                or horizontalSpeed > self.Config.LandingHorizontalCaptureMaxSpeed
            ) then
            commandedY = 0
        end
    end

    return Vector3.new(horizontalVelocity.X, commandedY, horizontalVelocity.Z)
end
```

## Return to home

`FlightController:CalculateReturnHomeVelocity()` calculates a route back to the home position with distance-based braking and altitude correction.

```luau
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
        distance > 8 and "RETURN"
            or (distance > self.Config.ReturnHomeStopDistance and "APPROACH" or "HOME HOLD")
    )

    return Vector3.new(horizontalVelocity.X, verticalVelocity, horizontalVelocity.Z)
end
```

## Movement utilities

Selected helper functions used for acceleration limits and takeoff interpolation.

```luau
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
```
