[← Back to Home](../README.md)

# Player Teleportation & Camera Handoff Client

This is the client-side system I built, this was used to ensure that the transition between the portals was smooth and manageable. It also checks position changes across the frames, it then sets up a local bounding box to turn off wall collisions as you walk through, and it manages a synchronized camera handoff through a high speed engine binding so the players screen doesn't jitter.

```lua
local Players =
	game:GetService("Players")

local RunService =
	game:GetService("RunService")

local ReplicatedStorage =
	game:GetService("ReplicatedStorage")

local player =
	Players.LocalPlayer

local portalSystem =
	ReplicatedStorage:WaitForChild(
		"PortalSystem"
	)

local PortalMath =
	require(
		portalSystem:WaitForChild(
			"PortalMath"
		)
	)

local portalFolder =
	workspace:WaitForChild(
		"ActivePortals"
	)

local WIDTH_PADDING =
	0.35

local HEIGHT_PADDING =
	1.25

local APERTURE_DISTANCE =
	3.0

local EXIT_EPSILON =
	0.35

local REENTRY_DISTANCE =
	1.5

local previousPosition =
	nil

local blockedPortal =
	nil

local bypassedWalls =
	{}

local function getPortal(
	portalType
)

	return portalFolder:FindFirstChild(
		"Portal_"
			.. player.UserId
			.. "_"
			.. portalType
	)

end

local function getCharacter()

	local character =
		player.Character

	if not character then
		return nil
	end

	local humanoid =
		character:FindFirstChildOfClass(
			"Humanoid"
		)

	local root =
		character:FindFirstChild(
			"HumanoidRootPart"
		)

	if not humanoid
		or not root
		or humanoid.Health <= 0 then

		return nil
	end

	return character,
		humanoid,
		root

end

local function getHostSurface(
	portal
)

	if not portal then
		return nil
	end

	local value =
		portal:FindFirstChild(
			"HostSurface"
		)

	if not value
		or not value:IsA(
			"ObjectValue"
		) then

		return nil
	end

	local host =
		value.Value

	if not host
		or not host:IsA(
			"BasePart"
		) then

		return nil
	end

	if not host:IsDescendantOf(
		workspace
		) then

		return nil
	end

	return host

end

local function nearPortalOpening(
	root,
	portal
)

	local localPosition =
		PortalMath.ToPortalSpace(
			root.Position,
			portal
		)

	if math.abs(
		localPosition.Z
		) > APERTURE_DISTANCE then

		return false
	end

	if localPosition.Z <
		-0.85 then

		return false
	end

	local halfWidth =
		portal.Size.X * 0.5

	local halfHeight =
		portal.Size.Y * 0.5

	local horizontalPadding =
		0.7

	local verticalPadding =
		1.75

	if math.abs(
		localPosition.X
		) > halfWidth
			+ horizontalPadding then

		return false
	end

	if math.abs(
		localPosition.Y
		) > halfHeight
			+ verticalPadding then

		return false
	end

	return true

end

local function restoreWall(
	wall
)

	local original =
		bypassedWalls[
	wall
	]

	if original == nil then
		return
	end

	if wall
		and wall.Parent then

		wall.CanCollide =
			original
	end

	bypassedWalls[
	wall
	] =
		nil

end

local function restoreAllWalls()

	local walls =
		{}

	for wall in pairs(
		bypassedWalls
		) do

		table.insert(
			walls,
			wall
		)

	end

	for _, wall in ipairs(
		walls
		) do

		restoreWall(
			wall
		)

	end

end

local function bypassWall(
	wall
)

	if not wall
		or not wall:IsA(
			"BasePart"
		) then

		return
	end

	if bypassedWalls[
		wall
		] == nil then

		bypassedWalls[
		wall
		] =
			wall.CanCollide

	end

	wall.CanCollide =
		false

end

local function updateApertureCollision(
	root,
	portalA,
	portalB
)

	local wantedWalls =
		{}

	if portalA
		and nearPortalOpening(
			root,
			portalA
		) then

		local host =
			getHostSurface(
				portalA
			)

		if host then

			wantedWalls[
			host
			] =
				true

		end

	end

	if portalB
		and nearPortalOpening(
			root,
			portalB
		) then

		local host =
			getHostSurface(
				portalB
			)

		if host then

			wantedWalls[
			host
			] =
				true

		end

	end

	local restoreList =
		{}

	for wall in pairs(
		bypassedWalls
		) do

		if not wantedWalls[
			wall
			] then

			table.insert(
				restoreList,
				wall
			)

		end

	end

	for _, wall in ipairs(
		restoreList
		) do

		restoreWall(
			wall
		)

	end

	for wall in pairs(
		wantedWalls
		) do

		bypassWall(
			wall
		)

	end

end

local function pushOutsideDestination(
	cframe,
	destination
)

	local localCF =
		destination.CFrame:ToObjectSpace(
			cframe
		)

	local position =
		localCF.Position

	if position.Z <
		EXIT_EPSILON then

		position =
			Vector3.new(
				position.X,
				position.Y,
				EXIT_EPSILON
			)

	end

	local rotation =
		localCF.Rotation

	return destination.CFrame
		*
		CFrame.new(
			position
		)
		*
		rotation

end

local function traverse(
	character,
	root,
	source,
	destination
)

	local oldCFrame =
		root.CFrame

	local currentCamera =
		workspace.CurrentCamera

	local cameraRelativeToRoot =
		nil

	if currentCamera then

		cameraRelativeToRoot =
			oldCFrame:ToObjectSpace(
				currentCamera.CFrame
			)

	end

	local oldLinearVelocity =
		root.AssemblyLinearVelocity

	local oldAngularVelocity =
		root.AssemblyAngularVelocity

	local transformed =
		PortalMath.TransformPhysics(
			oldCFrame,
			oldLinearVelocity,
			oldAngularVelocity,
			source,
			destination
		)

	local targetCFrame =
		pushOutsideDestination(
			transformed.CFrame,
			destination
		)

	character:PivotTo(
		targetCFrame
	)

	if currentCamera
		and cameraRelativeToRoot then

		local cameraHandoffCFrame =
			root.CFrame
			*
			cameraRelativeToRoot

		currentCamera.CFrame =
			cameraHandoffCFrame

		local bindName =
			"PortalCameraHandoff_"
			..
			tostring(
				os.clock()
			)

		RunService:BindToRenderStep(
			bindName,
			Enum.RenderPriority.Camera.Value + 1,
			function()

				if not root.Parent
					or workspace.CurrentCamera
					~= currentCamera then

					RunService:UnbindFromRenderStep(
						bindName
					)

					return
				end

				currentCamera.CFrame =
					root.CFrame
					*
					cameraRelativeToRoot

				RunService:UnbindFromRenderStep(
					bindName
				)

			end
		)

	end

	root.AssemblyLinearVelocity =
		transformed.LinearVelocity

	root.AssemblyAngularVelocity =
		transformed.AngularVelocity

	blockedPortal =
		destination

	previousPosition =
		root.Position

	print(
		"[PortalPhysics] CROSS",
		source.Name,
		"->",
		destination.Name,
		"| speed:",
		math.round(
			transformed.LinearVelocity.Magnitude
				*
				100
		)
			/
			100
	)

end

local function updateBlockedPortal(
	root
)

	if not blockedPortal then
		return
	end

	if not blockedPortal.Parent then

		blockedPortal =
			nil

		return
	end

	local localPosition =
		PortalMath.ToPortalSpace(
			root.Position,
			blockedPortal
		)

	if math.abs(
		localPosition.Z
		) >= REENTRY_DISTANCE then

		blockedPortal =
			nil

	end

end

local function checkCrossing(
	character,
	root,
	source,
	destination,
	oldPosition,
	newPosition
)

	if source ==
		blockedPortal then

		return false
	end

	local crossing =
		PortalMath.GetCrossing(
			oldPosition,
			newPosition,
			source,
			WIDTH_PADDING,
			HEIGHT_PADDING
		)

	if not crossing then
		return false
	end

	if crossing.PreviousLocal.Z <= 0 then
		return false
	end

	if crossing.CurrentLocal.Z >= 0 then
		return false
	end

	traverse(
		character,
		root,
		source,
		destination
	)

	return true

end

RunService.PreSimulation:Connect(function()

	local character,
	humanoid,
	root =
		getCharacter()

	if not character then

		previousPosition =
			nil

		restoreAllWalls()

		return
	end

	local portalA =
		getPortal(
			"A"
		)

	local portalB =
		getPortal(
			"B"
		)

	if not portalA
		or not portalB then

		previousPosition =
			root.Position

		restoreAllWalls()

		return
	end

	if portalA:GetAttribute(
		"PortalState"
		) ~= "Open"
			or portalB:GetAttribute(
				"PortalState"
			) ~= "Open" then

		previousPosition =
			root.Position

		restoreAllWalls()

		return
	end

	updateApertureCollision(
		root,
		portalA,
		portalB
	)

	updateBlockedPortal(
		root
	)

	local currentPosition =
		root.Position

	if not previousPosition then

		previousPosition =
			currentPosition

		return
	end

	if checkCrossing(
		character,
		root,
		portalA,
		portalB,
		previousPosition,
		currentPosition
		) then

		return
	end

	if checkCrossing(
		character,
		root,
		portalB,
		portalA,
		previousPosition,
		currentPosition
		) then

		return
	end

	previousPosition =
		currentPosition

end)

player.CharacterAdded:Connect(function(
	character
)

	restoreAllWalls()

	previousPosition =
		nil

	blockedPortal =
		nil

	local root =
		character:WaitForChild(
			"HumanoidRootPart"
		)

	previousPosition =
		root.Position

end)

player.CharacterRemoving:Connect(function()

	restoreAllWalls()

	previousPosition =
		nil

	blockedPortal =
		nil

end)

script.Destroying:Connect(function()

	restoreAllWalls()

end)
```
