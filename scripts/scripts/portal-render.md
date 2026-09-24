[← Back to Home](../README.md)

# Viewport World Streaming & Terrain Extraction Client

This client script runs our custom rendering pipeline inside a ViewportFrame container. It handles cloning local sections of the map, purifying instances by stripping active threads, and managing an asynchronous time-sliced batch loop so thousands of parts don't freeze the client's frame rates.

```lua
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Lighting = game:GetService("Lighting")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

local portalFolder = workspace:WaitForChild("ActivePortals")
local terrain = workspace.Terrain

local CANVAS_WIDTH = 600
local CANVAS_HEIGHT = 960

local CAMERA_BACK_DISTANCE = 8
local CAMERA_VERTICAL_OFFSET = -0.65
local CAMERA_LOOK_OFFSET = -0.10
local PORTAL_FOV = 52

local WORLD_RADIUS = 180
local WORLD_BUILD_BATCH = 200
local DYNAMIC_WORLD_SYNC_RATE = 1 / 30

local CHARACTER_HIDE_Z = -0.15
local CHARACTER_SHOW_Z = 0.15

local TERRAIN_ENABLED = true
local TERRAIN_RADIUS = 80
local TERRAIN_VERTICAL_RADIUS = 52
local TERRAIN_VOXEL_RESOLUTION = 4
local TERRAIN_OCCUPANCY_THRESHOLD = 0.10
local TERRAIN_MAX_SURFACE_CELLS = 1800
local TERRAIN_BUILD_BATCH = 100

local PORTAL_REVEAL_DELAY = 1.34
local PORTAL_RIP_END = 2.62

local renderData = {}

local dynamicAccumulator = 0
local pairSceneBuilding = false


local function getPortal(portalType)
	return portalFolder:FindFirstChild(
		"Portal_"
			.. player.UserId
			.. "_"
			.. portalType
	)
end


local function getHostSurface(portal)
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


local function isPortalObject(object)
	if object == portalFolder then
		return true
	end

	if object:IsDescendantOf(
		portalFolder
		) then

		return true
	end

	if object.Name ==
		"PortalPlacementPreview" then

		return true
	end

	return false
end


local function getCharacterOwner(object)
	for _, currentPlayer in ipairs(
		Players:GetPlayers()
		) do

		local character =
			currentPlayer.Character

		if character
			and object:IsDescendantOf(
				character
			) then

			return currentPlayer
		end
	end

	return nil
end


local function cleanWorldClone(clone)
	for _, object in ipairs(
		clone:GetDescendants()
		) do

		if object:IsA("Script")
			or object:IsA("LocalScript")
			or object:IsA("ModuleScript")
			or object:IsA("SurfaceGui")
			or object:IsA("BillboardGui")
			or object:IsA("ProximityPrompt")
			or object:IsA("ClickDetector")
			or object:IsA("TouchTransmitter")
			or object:IsA("Sound")
			or object:IsA("ParticleEmitter")
			or object:IsA("Beam")
			or object:IsA("Trail") then

			object:Destroy()
		end
	end
end


local function syncWorldProperties(
	original,
	clone
)
	clone.CFrame =
		original.CFrame

	clone.Size =
		original.Size

	clone.Color =
		original.Color

	clone.Material =
		original.Material

	clone.Transparency =
		original.Transparency

	clone.Reflectance =
		original.Reflectance

	clone.CastShadow =
		original.CastShadow

	clone.Anchored =
		true

	clone.CanCollide =
		false

	clone.CanTouch =
		false

	clone.CanQuery =
		false

	pcall(function()
		clone.MaterialVariant =
			original.MaterialVariant
	end)

	if original:IsA("MeshPart")
		and clone:IsA("MeshPart") then

		pcall(function()
			clone.TextureID =
				original.TextureID
		end)
	end
end


local function cloneWorldPart(original)
	local oldArchivable =
		original.Archivable

	original.Archivable =
		true

	local success, clone =
		pcall(function()
			return original:Clone()
		end)

	original.Archivable =
		oldArchivable

	if not success
		or not clone then

		return nil
	end

	cleanWorldClone(
		clone
	)

	syncWorldProperties(
		original,
		clone
	)

	return clone
end


local function createRenderer(portal)
	if renderData[
		portal
		] then

		return renderData[
		portal
		]
	end

	local gui =
		Instance.new(
			"SurfaceGui"
		)

	gui.Name =
		"PortalView_"
		.. portal.Name

	gui.Adornee =
		portal

	gui.Face =
		Enum.NormalId.Back

	gui.Enabled =
		true

	gui.AlwaysOnTop =
		false

	gui.LightInfluence =
		0

	gui.MaxDistance =
		1000

	gui.SizingMode =
		Enum.SurfaceGuiSizingMode.FixedSize

	gui.CanvasSize =
		Vector2.new(
			CANVAS_WIDTH,
			CANVAS_HEIGHT
		)

	gui.Parent =
		playerGui


	local container =
		Instance.new(
			"Frame"
		)

	container.Name =
		"PortalRevealContainer"

	container.AnchorPoint =
		Vector2.new(
			0.5,
			0.5
		)

	container.Position =
		UDim2.fromScale(
			0.5,
			0.5
		)

	container.Size =
		UDim2.fromScale(
			1,
			1
		)

	container.BackgroundTransparency =
		1

	container.BorderSizePixel =
		0

	container.ClipsDescendants =
		true

	container.Parent =
		gui


	local containerCorner =
		Instance.new(
			"UICorner"
		)

	containerCorner.CornerRadius =
		UDim.new(
			0.5,
			0
		)

	containerCorner.Parent =
		container


	local viewport =
		Instance.new(
			"ViewportFrame"
		)

	viewport.Name =
		"PortalViewport"

	viewport.AnchorPoint =
		Vector2.new(
			0.5,
			0.5
		)

	viewport.Position =
		UDim2.fromScale(
			0.5,
			0.5
		)

	viewport.Size =
		UDim2.fromScale(
			0.001,
			0.001
		)

	viewport.Visible =
		false

	viewport.BackgroundTransparency =
		0

	viewport.BackgroundColor3 =
		Color3.fromRGB(
			111,
			177,
			216
		)

	viewport.BorderSizePixel =
		0

	viewport.Ambient =
		Color3.fromRGB(
			200,
			200,
			200
		)

	viewport.LightColor =
		Color3.fromRGB(
			255,
			255,
			255
		)

	viewport.LightDirection =
		Vector3.new(
			-1,
			-1,
			-1
		)

	viewport.Parent =
		container


	local viewportCorner =
		Instance.new(
			"UICorner"
		)

	viewportCorner.CornerRadius =
		UDim.new(
			0.5,
			0
		)

	viewportCorner.Parent =
		viewport


	local world =
		Instance.new(
			"WorldModel"
		)

	world.Name =
		"PortalWorld"

	world.Parent =
		viewport


	local camera =
		Instance.new(
			"Camera"
		)

	camera.Name =
		"PortalCamera"

	camera.FieldOfView =
		PORTAL_FOV

	camera.Parent =
		viewport

	viewport.CurrentCamera =
		camera


	local hiddenCharacters =
		Instance.new(
			"Folder"
		)

	hiddenCharacters.Name =
		"HiddenPortalCharacters_"
		.. portal.Name

	hiddenCharacters.Parent =
		playerGui


	local data = {
		Portal = portal,

		Gui = gui,

		Container = container,

		Viewport = viewport,

		World = world,

		Camera = camera,

		HiddenCharacters =
			hiddenCharacters,

		Destination = nil,

		PairRevealStarted = nil,

		PairTarget = nil,

		SceneBuilt = false,

		SceneBuilding = false,

		WorldClones = {},

		DynamicClones = {},

		CharacterClones = {},

		TerrainFolder = nil,

		TerrainBuilding = false,

		TerrainBuiltFor = nil
	}

	renderData[
	portal
	] =
		data

	return data
end


local function hideViewport(data)
	data.Viewport.Visible =
		false

	data.Viewport.AnchorPoint =
		Vector2.new(
			0.5,
			0.5
		)

	data.Viewport.Position =
		UDim2.fromScale(
			0.5,
			0.5
		)

	data.Viewport.Size =
		UDim2.fromScale(
			0.001,
			0.001
		)

	data.Viewport.Rotation =
		0
end


local function fullyOpenViewport(data)
	data.Viewport.Visible =
		true

	data.Viewport.AnchorPoint =
		Vector2.new(
			0.5,
			0.5
		)

	data.Viewport.Position =
		UDim2.fromScale(
			0.5,
			0.5
		)

	data.Viewport.Size =
		UDim2.fromScale(
			1,
			1
		)

	data.Viewport.Rotation =
		0
end


local function getCloseShape(closeStarted)
	local elapsed =
		math.max(
			0,
			workspace:GetServerTimeNow()
			-
			closeStarted
		)

	if elapsed < 0.12 then
		return
			elapsed,
			0,
			1,
			1
	end

	local progress =
		math.clamp(
			(
				elapsed
				-
				0.12
			)
			/
			1.16,
			0,
			1
		)

	local remaining =
		1
	-
		progress

	local base =
		remaining
		^
		1.10

	local xNoise =
		math.sin(
			elapsed
			*
			37
		)
		*
		0.020
		*
		remaining

	local yNoise =
		math.sin(
			elapsed
			*
			43
			+
			1.7
		)
		*
		0.030
		*
		remaining

	local snap =
		(
			math.max(
				0,
				math.sin(
					elapsed
					*
					49
				)
			)
			^
			16
		)
		*
		0.032
		*
		remaining

	local scaleX =
		math.clamp(
			base
			+
			xNoise
			-
			snap,
			0.001,
			1
		)

	local scaleY =
		math.clamp(
			remaining
			^
			0.91
			+
			yNoise,
			0.001,
			1
		)

	if elapsed >= 1.28 then
		local finalProgress =
			math.clamp(
				(
					elapsed
					-
					1.28
				)
				/
				0.10,
				0,
				1
			)

		scaleX =
			math.max(
				0.001,
				0.075
				*
				(
					1
					-
					finalProgress
				)
			)

		scaleY =
			math.max(
				0.001,
				0.31
				*
				(
					1
					-
					finalProgress
				)
			)
	end

	return
		elapsed,
		progress,
		scaleX,
		scaleY
end


local function updateOpening(data)
	local started =
		data.PairRevealStarted

	if not started then
		hideViewport(
			data
		)

		return
	end

	local elapsed =
		workspace:GetServerTimeNow()
	-
		started

	if elapsed <
		PORTAL_REVEAL_DELAY then

		hideViewport(
			data
		)

		return
	end

	data.Viewport.Visible =
		true

	local raw =
		math.clamp(
			(
				elapsed
				-
				PORTAL_REVEAL_DELAY
			)
			/
			math.max(
				PORTAL_RIP_END
				-
				PORTAL_REVEAL_DELAY,
				0.001
			),
			0,
			1
		)

	local eased =
		1
	-
		(
			1
			-
			raw
		)
		^
		3

	local instability =
		1
	-
		raw

	local tearX =
		math.noise(
			started
			*
			0.31,
			elapsed
			*
			11
		)
		*
		0.050
		+
		math.sin(
			elapsed
			*
			23
		)
		*
		0.018

	local tearY =
		math.noise(
			started
			*
			0.47,
			elapsed
			*
			14
		)
		*
		0.070
		+
		math.sin(
			elapsed
			*
			29
			+
			1.4
		)
		*
		0.024

	local width =
		math.clamp(
			eased
			+
			tearX
			*
			instability,
			0.001,
			1
		)

	local height =
		math.clamp(
			eased
			^
			0.90
			+
			tearY
			*
			instability,
			0.001,
			1
		)

	if raw > 0.18
		and raw < 0.82 then

		local snap =
			math.max(
				0,
				math.sin(
					elapsed
					*
					37
				)
			)
			^
			12

		width =
			math.clamp(
				width
				-
				snap
				*
				0.025
				*
				instability,
				0.001,
				1
			)

		height =
			math.clamp(
				height
				-
				snap
				*
				0.040
				*
				instability,
				0.001,
				1
			)
	end

	data.Viewport.AnchorPoint =
		Vector2.new(
			0.5,
			0.5
		)

	data.Viewport.Position =
		UDim2.fromScale(
			0.5
			+
			math.noise(
				started
				*
				0.13,
				elapsed
				*
				19
			)
			*
			0.010
			*
			instability,

			0.5
			+
			math.noise(
				started
				*
				0.21,
				elapsed
				*
				23
			)
			*
			0.014
			*
			instability
		)

	data.Viewport.Size =
		UDim2.fromScale(
			width,
			height
		)

	data.Viewport.Rotation =
		math.sin(
			elapsed
			*
			31
		)
		*
		0.65
		*
		instability

	if raw >= 0.985 then
		fullyOpenViewport(
			data
		)
	end
end


local function updateReveal(data)
	if not data
		or not data.Portal
		or not data.Viewport then

		return
	end

	data.Container.Visible =
		true

	data.Container.AnchorPoint =
		Vector2.new(
			0.5,
			0.5
		)

	data.Container.Position =
		UDim2.fromScale(
			0.5,
			0.5
		)

	data.Container.Size =
		UDim2.fromScale(
			1,
			1
		)

	data.Container.Rotation =
		0

	local state =
		data.Portal:GetAttribute(
			"PortalState"
		)

	if state ==
		"Closing" then

		local closeStarted =
			data.Portal:GetAttribute(
				"CloseStarted"
			)
			or
			workspace:GetServerTimeNow()

		local elapsed,
			progress,
			scaleX,
			scaleY =
			getCloseShape(
				closeStarted
			)

		data.Viewport.Visible =
			elapsed < 1.38

		if elapsed < 0.12 then
			fullyOpenViewport(
				data
			)

			return
		end

		local remaining =
			1
		-
			progress

		data.Viewport.Position =
			UDim2.fromScale(
				0.5
				+
				math.sin(
					elapsed
					*
					31
				)
				*
				0.008
				*
				remaining,

				0.5
				+
				math.sin(
					elapsed
					*
					41
					+
					2.1
				)
				*
				0.007
				*
				remaining
			)

		data.Viewport.Size =
			UDim2.fromScale(
				scaleX,
				scaleY
			)

		data.Viewport.Rotation =
			math.sin(
				elapsed
				*
				53
			)
			*
			0.55
			*
			remaining

		if elapsed >= 1.378 then
			data.Viewport.Visible =
				false
		end

		return
	end

	if state ==
		"Open" then

		if data.PairRevealStarted then
			local pairElapsed =
				workspace:GetServerTimeNow()
			-
				data.PairRevealStarted

			if pairElapsed <
				PORTAL_RIP_END then

				updateOpening(
					data
				)

				return
			end
		end

		fullyOpenViewport(
			data
		)

		return
	end

	updateOpening(
		data
	)
end


local function partWithinWorldRadius(
	part,
	destination
)
	local distance =
		(
			part.Position
			-
			destination.Position
		).Magnitude

	local extent =
		part.Size.Magnitude
		*
		0.5

	return
		distance
	-
		extent
		<=
		WORLD_RADIUS
end


local function shouldRenderWorldPart(
	data,
	part
)
	if not data.Destination then
		return false
	end

	if not part:IsA(
		"BasePart"
		) then

		return false
	end

	if isPortalObject(
		part
		) then

		return false
	end

	if getCharacterOwner(
		part
		) then

		return false
	end

	if part.Transparency >= 1 then
		return false
	end

	local host =
		getHostSurface(
			data.Destination
		)

	if host == part then
		return false
	end

	if not partWithinWorldRadius(
		part,
		data.Destination
		) then

		return false
	end

	return true
end


local function isDynamicWorldPart(part)
	if not part.Anchored then
		return true
	end

	if part:GetAttribute(
		"PortalDynamic"
		) == true then

		return true
	end

	return false
end


local function removeWorldPart(
	data,
	original
)
	local clone =
		data.WorldClones[
	original
	]

	if clone then
		clone:Destroy()
	end

	data.WorldClones[
	original
	] =
		nil

	data.DynamicClones[
	original
	] =
		nil
end


local function addWorldPart(
	data,
	original
)
	if data.WorldClones[
		original
		] then

		return
	end

	if not shouldRenderWorldPart(
		data,
		original
		) then

		return
	end

	local clone =
		cloneWorldPart(
			original
		)

	if not clone then
		return
	end

	clone.Parent =
		data.World

	data.WorldClones[
	original
	] =
		clone

	if isDynamicWorldPart(
		original
		) then

		data.DynamicClones[
		original
		] =
			clone
	end
end


local function clearWorldScene(data)
	for _, clone in pairs(
		data.WorldClones
		) do

		if clone then
			clone:Destroy()
		end
	end

	table.clear(
		data.WorldClones
	)

	table.clear(
		data.DynamicClones
	)

	data.SceneBuilt =
		false

	data.SceneBuilding =
		false
end


local function prepareWorldScene(
	data,
	destination
)
	clearWorldScene(
		data
	)

	data.Destination =
		destination

	data.SceneBuilding =
		true
end


local function ensurePairWorldScenes(
	dataA,
	destinationA,
	dataB,
	destinationB
)
	local needA =
		dataA.Destination
		~=
		destinationA
		or
		(
			not dataA.SceneBuilt
			and
			not dataA.SceneBuilding
		)

	local needB =
		dataB.Destination
		~=
		destinationB
		or
		(
			not dataB.SceneBuilt
			and
			not dataB.SceneBuilding
		)

	if not needA
		and not needB then

		return
	end

	if pairSceneBuilding then
		return
	end

	pairSceneBuilding =
		true

	if needA then
		prepareWorldScene(
			dataA,
			destinationA
		)
	end

	if needB then
		prepareWorldScene(
			dataB,
			destinationB
		)
	end

	task.spawn(function()
		local objects =
			workspace:GetDescendants()

		for index, object in ipairs(
			objects
			) do

			if needA
				and dataA.Portal
				and dataA.Portal.Parent
				and destinationA.Parent then

				addWorldPart(
					dataA,
					object
				)
			end

			if needB
				and dataB.Portal
				and dataB.Portal.Parent
				and destinationB.Parent then

				addWorldPart(
					dataB,
					object
				)
			end

			if index
				%
				WORLD_BUILD_BATCH
				==
				0 then

				task.wait()
			end
		end

		if needA
			and dataA.Portal
			and dataA.Portal.Parent
			and dataA.Destination
			==
			destinationA then

			dataA.SceneBuilt =
				true

			dataA.SceneBuilding =
				false
		end

		if needB
			and dataB.Portal
			and dataB.Portal.Parent
			and dataB.Destination
			==
			destinationB then

			dataB.SceneBuilt =
				true

			dataB.SceneBuilding =
				false
		end

		pairSceneBuilding =
			false
	end)
end


local function syncDynamicWorld(data)
	for original, clone in pairs(
		data.DynamicClones
		) do

		if not original
			or not original.Parent
			or not clone
			or not clone.Parent then

			removeWorldPart(
				data,
				original
			)

			continue
		end

		if not shouldRenderWorldPart(
			data,
			original
			) then

			removeWorldPart(
				data,
				original
			)

			continue
		end

		if clone.CFrame
			~=
			original.CFrame then

			clone.CFrame =
				original.CFrame
		end

		if clone.Size
			~=
			original.Size then

			clone.Size =
				original.Size
		end

		if clone.Transparency
			~=
			original.Transparency then

			clone.Transparency =
				original.Transparency
		end
	end
end


local function terrainMaterialColor(material)
	local success, color =
		pcall(function()
			return terrain:GetMaterialColor(
				material
			)
		end)

	if success
		and color then

		return color
	end

	if material ==
		Enum.Material.Grass then

		return Color3.fromRGB(
			106,
			127,
			63
		)

	elseif material ==
		Enum.Material.Ground then

		return Color3.fromRGB(
			124,
			92,
			70
		)

	elseif material ==
		Enum.Material.Mud then

		return Color3.fromRGB(
			91,
			70,
			55
		)

	elseif material ==
		Enum.Material.Rock then

		return Color3.fromRGB(
			102,
			102,
			102
		)

	elseif material ==
		Enum.Material.Slate then

		return Color3.fromRGB(
			88,
			93,
			96
		)

	elseif material ==
		Enum.Material.Sand then

		return Color3.fromRGB(
			218,
			203,
			158
		)

	elseif material ==
		Enum.Material.Snow then

		return Color3.fromRGB(
			235,
			240,
			242
		)

	elseif material ==
		Enum.Material.Ice then

		return Color3.fromRGB(
			184,
			221,
			235
		)

	elseif material ==
		Enum.Material.Water then

		return Color3.fromRGB(
			57,
			120,
			181
		)
	end

	return Color3.fromRGB(
		120,
		120,
		120
	)
end


local function clearTerrainProxy(data)
	if data.TerrainFolder then
		data.TerrainFolder:Destroy()
	end

	data.TerrainFolder =
		nil

	data.TerrainBuilding =
		false

	data.TerrainBuiltFor =
		nil
end


local function buildTerrainProxy(
	data,
	destination
)
	if not TERRAIN_ENABLED then
		return
	end

	if not data
		or not destination
		or not destination.Parent then

		return
	end

	if data.TerrainBuilding then
		return
	end

	if data.TerrainBuiltFor
		==
		destination then

		return
	end

	data.TerrainBuilding =
		true

	data.TerrainBuiltFor =
		destination

	task.spawn(function()
		if data.TerrainFolder then
			data.TerrainFolder:Destroy()
		end

		local folder =
			Instance.new(
				"Folder"
			)

		folder.Name =
			"PortalTerrainSurface"

		folder.Parent =
			data.World

		data.TerrainFolder =
			folder

		local center =
			destination.Position

		local half =
			Vector3.new(
				TERRAIN_RADIUS,
				TERRAIN_VERTICAL_RADIUS,
				TERRAIN_RADIUS
			)

		local region =
			Region3.new(
				center - half,
				center + half
			):ExpandToGrid(
			TERRAIN_VOXEL_RESOLUTION
		)

		local success,
		materials,
		occupancies =
			pcall(function()

				return terrain:ReadVoxels(
					region,
					TERRAIN_VOXEL_RESOLUTION
				)

			end)

		if not success
			or not materials
			or not occupancies then

			if folder then
				folder:Destroy()
			end

			data.TerrainFolder =
				nil

			data.TerrainBuilding =
				false

			data.TerrainBuiltFor =
				nil

			return
		end

		local nx =
			materials.Size.X

		local ny =
			materials.Size.Y

		local nz =
			materials.Size.Z

		local regionCF =
			region.CFrame

		local regionSize =
			region.Size

		local function inside(
			x,
			y,
			z
		)
			return
				x >= 1
				and x <= nx
				and y >= 1
				and y <= ny
				and z >= 1
				and z <= nz
		end

		local function solidVoxel(
			x,
			y,
			z
		)
			if not inside(
				x,
				y,
				z
				) then

				return false
			end

			local material =
				materials[x][y][z]

			if material ==
				Enum.Material.Air then

				return false
			end

			local occupancy =
				occupancies[x][y][z]

			return
				occupancy
				>
			TERRAIN_OCCUPANCY_THRESHOLD
		end

		local candidates =
			{}

		for x = 1, nx do
			for y = 1, ny do
				for z = 1, nz do
					if solidVoxel(
						x,
						y,
						z
						) then

						local exposedTop =
							not solidVoxel(
								x,
								y + 1,
								z
							)

						local exposedBottom =
							not solidVoxel(
								x,
								y - 1,
								z
							)

						local exposedLeft =
							not solidVoxel(
								x - 1,
								y,
								z
							)

						local exposedRight =
							not solidVoxel(
								x + 1,
								y,
								z
							)

						local exposedFront =
							not solidVoxel(
								x,
								y,
								z - 1
							)

						local exposedBack =
							not solidVoxel(
								x,
								y,
								z + 1
							)

						local surface =
							exposedTop
							or exposedBottom
							or exposedLeft
							or exposedRight
							or exposedFront
							or exposedBack

						if surface then
							local localX =
								-regionSize.X
								*
								0.5
								+
								(
									x
									-
									0.5
								)
								*
								TERRAIN_VOXEL_RESOLUTION

							local localY =
								-regionSize.Y
								*
								0.5
								+
								(
									y
									-
									0.5
								)
								*
								TERRAIN_VOXEL_RESOLUTION

							local localZ =
								-regionSize.Z
								*
								0.5
								+
								(
									z
									-
									0.5
								)
								*
								TERRAIN_VOXEL_RESOLUTION

							local worldPosition =
								regionCF:
								PointToWorldSpace(
									Vector3.new(
										localX,
										localY,
										localZ
									)
								)

							local distance =
								(
									worldPosition
									-
									destination.Position
								).Magnitude

							table.insert(
								candidates,
								{
									LocalX =
										localX,

									LocalY =
										localY,

									LocalZ =
										localZ,

									Material =
										materials[x][y][z],

									Occupancy =
										occupancies[x][y][z],

									TopExposed =
										exposedTop,

									Distance =
										distance
								}
							)
						end
					end
				end
			end
		end

		table.sort(
			candidates,
			function(a, b)
				return
					a.Distance
					<
					b.Distance
			end
		)

		local created =
			0

		for index, cell in ipairs(
			candidates
			) do

			if created >=
				TERRAIN_MAX_SURFACE_CELLS then

				break
			end

			if not data.Portal
				or not data.Portal.Parent
				or data.Destination
				~=
				destination then

				if folder then
					folder:Destroy()
				end

				data.TerrainFolder =
					nil

				data.TerrainBuilding =
					false

				data.TerrainBuiltFor =
					nil

				return
			end

			local occupancy =
				math.clamp(
					cell.Occupancy,
					0,
					1
				)

			local cellHeight =
				TERRAIN_VOXEL_RESOLUTION

			local yOffset =
				0

			if cell.TopExposed then
				cellHeight =
					math.max(
						0.65,

						TERRAIN_VOXEL_RESOLUTION
						*
						math.clamp(
							occupancy,
							0.16,
							1
						)
					)

				yOffset =
					-
					(
						TERRAIN_VOXEL_RESOLUTION
						-
						cellHeight
					)
					*
					0.5
			end

			local block =
				Instance.new(
					"Part"
				)

			block.Name =
				"TerrainSurfaceVoxel"

			block.Anchored =
				true

			block.CanCollide =
				false

			block.CanTouch =
				false

			block.CanQuery =
				false

			block.CastShadow =
				false

			block.Size =
				Vector3.new(
					TERRAIN_VOXEL_RESOLUTION
					+
					0.04,

					cellHeight
					+
					0.04,

					TERRAIN_VOXEL_RESOLUTION
					+
					0.04
				)

			block.CFrame =
				regionCF
				*
				CFrame.new(
					cell.LocalX,

					cell.LocalY
					+
					yOffset,

					cell.LocalZ
				)

			if cell.Material ==
				Enum.Material.Water then

				block.Material =
					Enum.Material.Glass

				block.Transparency =
					0.38
			else
				block.Material =
					cell.Material

				block.Transparency =
					0
			end

			block.Color =
				terrainMaterialColor(
					cell.Material
				)

			block.Parent =
				folder

			created +=
				1

			if index
				%
				TERRAIN_BUILD_BATCH
				==
				0 then

				task.wait()
			end
		end

		data.TerrainBuilding =
			false

		print(
			"[PortalRender] Surface terrain built:",
			created,
			"/",
			#candidates
		)
	end)
end


local function cleanCharacterClone(character)
	for _, object in ipairs(
		character:GetDescendants()
		) do

		if object:IsA("Script")
			or object:IsA("LocalScript")
			or object:IsA("ModuleScript")
			or object:IsA("Tool")
			or object:IsA("ProximityPrompt")
			or object:IsA("ClickDetector")
			or object:IsA("Sound")
			or object:IsA("ParticleEmitter")
			or object:IsA("Beam")
			or object:IsA("Trail") then

			object:Destroy()
		end
	end

	for _, object in ipairs(
		character:GetDescendants()
		) do

		if object:IsA(
			"BasePart"
			) then

			object.Anchored =
				true

			object.CanCollide =
				false

			object.CanTouch =
				false

			object.CanQuery =
				false
		end
	end
end


local function getRelativePath(
	object,
	root
)
	local path =
		{}

	local current =
		object

	while current
		and current ~= root do

		table.insert(
			path,
			1,
			current.Name
		)

		current =
			current.Parent
	end

	return path
end


local function followPath(
	root,
	path
)
	local current =
		root

	for _, name in ipairs(
		path
		) do

		current =
			current:FindFirstChild(
				name
			)

		if not current then
			return nil
		end
	end

	return current
end


local function buildCharacterPartMap(
	originalCharacter,
	cloneCharacter
)
	local partMap =
		{}

	for _, originalPart in ipairs(
		originalCharacter:GetDescendants()
		) do

		if originalPart:IsA(
			"BasePart"
			) then

			local path =
				getRelativePath(
					originalPart,
					originalCharacter
				)

			local clonePart =
				followPath(
					cloneCharacter,
					path
				)

			if clonePart
				and clonePart:IsA(
					"BasePart"
				) then

				partMap[
				originalPart
				] =
					clonePart
			end
		end
	end

	return partMap
end


local function createCharacterClone(
	data,
	currentPlayer
)
	local character =
		currentPlayer.Character

	if not character then
		return
	end

	if data.CharacterClones[
		currentPlayer
		] then

		return
	end

	local previousArchivable =
		character.Archivable

	character.Archivable =
		true

	local success, clone =
		pcall(function()
			return character:Clone()
		end)

	character.Archivable =
		previousArchivable

	if not success
		or not clone then

		return
	end

	cleanCharacterClone(
		clone
	)

	clone.Name =
		"PortalCharacter_"
		..
		currentPlayer.Name

	clone.Parent =
		data.World

	local partMap =
		buildCharacterPartMap(
			character,
			clone
		)

	data.CharacterClones[
	currentPlayer
	] =
		{
			Original =
			character,

			Clone =
			clone,

			PartMap =
			partMap,

			VisibleInPortal =
			nil
		}
end


local function syncCharacterClone(
	data,
	currentPlayer
)
	local character =
		currentPlayer.Character

	if not character then
		local old =
			data.CharacterClones[
		currentPlayer
		]

		if old
			and old.Clone then

			old.Clone:Destroy()
		end

		data.CharacterClones[
		currentPlayer
		] =
			nil

		return
	end

	local entry =
		data.CharacterClones[
	currentPlayer
	]

	if not entry
		or entry.Original
		~=
		character
		or not entry.Clone then

		if entry
			and entry.Clone then

			entry.Clone:Destroy()
		end

		data.CharacterClones[
		currentPlayer
		] =
			nil

		createCharacterClone(
			data,
			currentPlayer
		)

		entry =
			data.CharacterClones[
		currentPlayer
		]
	end

	if not entry then
		return
	end

	for originalPart, clonedPart in pairs(
		entry.PartMap
		) do

		if originalPart
			and originalPart.Parent
			and clonedPart then

			clonedPart.CFrame =
				originalPart.CFrame

			clonedPart.Size =
				originalPart.Size

			clonedPart.Transparency =
				originalPart.Transparency

			clonedPart.Color =
				originalPart.Color
		end
	end
end


local function syncCharacters(data)
	for _, currentPlayer in ipairs(
		Players:GetPlayers()
		) do

		syncCharacterClone(
			data,
			currentPlayer
		)
	end

	for currentPlayer, entry in pairs(
		data.CharacterClones
		) do

		if not currentPlayer.Parent
			or not currentPlayer.Character then

			if entry.Clone then
				entry.Clone:Destroy()
			end

			data.CharacterClones[
			currentPlayer
			] =
				nil
		end
	end
end


local function updateCharacterCulling(
	data,
	destination
)
	if not destination then
		return
	end

	for currentPlayer, entry in pairs(
		data.CharacterClones
		) do

		local character =
			currentPlayer.Character

		local clone =
			entry.Clone

		if character
			and clone then

			local root =
				character:FindFirstChild(
					"HumanoidRootPart"
				)

			if root then
				local localPosition =
					destination.CFrame:
					PointToObjectSpace(
						root.Position
					)

				local localZ =
					localPosition.Z

				local visible =
					entry.VisibleInPortal

				if visible == nil then
					visible =
						localZ >= 0
				end

				if visible then
					if localZ <
						CHARACTER_HIDE_Z then

						visible =
							false
					end
				else
					if localZ >
						CHARACTER_SHOW_Z then

						visible =
							true
					end
				end

				entry.VisibleInPortal =
					visible

				local targetParent

				if visible then
					targetParent =
						data.World
				else
					targetParent =
						data.HiddenCharacters
				end

				if clone.Parent
					~=
					targetParent then

					clone.Parent =
						targetParent
				end
			end
		end
	end
end


local function updateViewportBackground(data)
	local clock =
		Lighting.ClockTime

	local daylight =
		math.clamp(
			1
			-
			math.abs(
				clock
				-
				12
			)
			/
			7.5,
			0,
			1
		)

	local night =
		Color3.fromRGB(
			16,
			22,
			38
		)

	local day =
		Color3.fromRGB(
			111,
			177,
			216
		)

	local base =
		night:Lerp(
			day,
			daylight
		)

	data.Viewport.BackgroundColor3 =
		base:Lerp(
			Lighting.OutdoorAmbient,
			0.16
		)

	data.Viewport.BackgroundTransparency =
		0
end


local function updateDestinationCamera(
	data,
	destination
)
	if not destination
		or not destination.Parent then

		return
	end

	local outward =
		-destination.CFrame.LookVector

	local up =
		destination.CFrame.UpVector

	local cameraPosition =
		destination.Position
	-
		outward
		*
		CAMERA_BACK_DISTANCE
		+
		up
		*
		CAMERA_VERTICAL_OFFSET

	local targetPosition =
		destination.Position
		+
		outward
		*
		20
		+
		up
		*
		CAMERA_LOOK_OFFSET

	data.Camera.CFrame =
		CFrame.lookAt(
			cameraPosition,
			targetPosition,
			up
		)

	data.Camera.FieldOfView =
		PORTAL_FOV
end


local function destroyRenderer(portal)
	local data =
		renderData[
	portal
	]

	if not data then
		return
	end

	if data.Gui then
		data.Gui:Destroy()
	end

	if data.HiddenCharacters then
		data.HiddenCharacters:Destroy()
	end

	renderData[
	portal
	] =
		nil
end


workspace.DescendantAdded:Connect(function(
	object
)
	if not object:IsA(
		"BasePart"
		) then

		return
	end

	task.defer(function()
		if not object.Parent then
			return
		end

		for _, data in pairs(
			renderData
			) do

			if data.SceneBuilt
				and data.Destination then

				addWorldPart(
					data,
					object
				)
			end
		end
	end)
end)


workspace.DescendantRemoving:Connect(function(
	object
)
	for _, data in pairs(
		renderData
		) do

		if data.WorldClones[
			object
			] then

			removeWorldPart(
				data,
				object
			)
		end
	end
end)


RunService.RenderStepped:Connect(function(
	deltaTime
)
	local portalA =
		getPortal(
			"A"
		)

	local portalB =
		getPortal(
			"B"
		)

	local dataA =
		portalA
		and
		createRenderer(
			portalA
		)
		or
		nil

	local dataB =
		portalB
		and
		createRenderer(
			portalB
		)
		or
		nil

	for portal in pairs(
		renderData
		) do

		if not portal.Parent then
			destroyRenderer(
				portal
			)
		end
	end

	if dataA then
		local stateA =
			portalA:GetAttribute(
				"PortalState"
			)

		local stateB =
			portalB
			and
			portalB:GetAttribute(
				"PortalState"
			)
			or
			nil

		dataA.Gui.Enabled =
			portalB ~= nil
			and
			stateA ~= "Closed"
			and
			stateB ~= "Closed"

		updateReveal(
			dataA
		)
	end

	if dataB then
		local stateA =
			portalA
			and
			portalA:GetAttribute(
				"PortalState"
			)
			or
			nil

		local stateB =
			portalB:GetAttribute(
				"PortalState"
			)

		dataB.Gui.Enabled =
			portalA ~= nil
			and
			stateA ~= "Closed"
			and
			stateB ~= "Closed"

		updateReveal(
			dataB
		)
	end

	if not portalA
		or not portalB
		or not dataA
		or not dataB then

		return
	end

	local pairChanged =
		dataA.PairTarget
		~=
		portalB
		or
		dataB.PairTarget
		~=
		portalA

	if pairChanged then
		local pairStart =
			workspace:GetServerTimeNow()

		dataA.PairTarget =
			portalB

		dataB.PairTarget =
			portalA

		dataA.PairRevealStarted =
			pairStart

		dataB.PairRevealStarted =
			pairStart
	end

	ensurePairWorldScenes(
		dataA,
		portalB,

		dataB,
		portalA
	)

	buildTerrainProxy(
		dataA,
		portalB
	)

	buildTerrainProxy(
		dataB,
		portalA
	)

	updateViewportBackground(
		dataA
	)

	updateViewportBackground(
		dataB
	)

	updateDestinationCamera(
		dataA,
		portalB
	)

	updateDestinationCamera(
		dataB,
		portalA
	)

	syncCharacters(
		dataA
	)

	syncCharacters(
		dataB
	)

	updateCharacterCulling(
		dataA,
		portalB
	)

	updateCharacterCulling(
		dataB,
		portalA
	)

	dynamicAccumulator +=
		deltaTime

	if dynamicAccumulator >=
		DYNAMIC_WORLD_SYNC_RATE then

		dynamicAccumulator =
			0

		syncDynamicWorld(
			dataA
		)

		syncDynamicWorld(
			dataB
		)
	end
end)


portalFolder.ChildRemoved:Connect(function(
	child
)
	destroyRenderer(
		child
	)
end)


Players.PlayerRemoving:Connect(function(
	leavingPlayer
)
	for _, data in pairs(
		renderData
		) do

		local entry =
			data.CharacterClones[
		leavingPlayer
		]

		if entry then
			if entry.Clone then
				entry.Clone:Destroy()
			end

			data.CharacterClones[
			leavingPlayer
			] =
				nil
		end
	end
end)


print(
	"[Portal] V8.21 OPTIMIZED SURFACE TERRAIN ONLINE"
)
```
