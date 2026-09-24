[← Back to Home](../README.md)

# Procedural Ring Particle & Lighting Effects

This is the system that drives the cinematic Doctor Strange-style entrance animations. Instead of using standard flat assets, it uses math functions to calculate circle curvatures, force spark paths to follow true physical angles, and inject chaotic noise fluctuations right as space rips open.

```lua
local RunService = game:GetService("RunService")
local Lighting = game:GetService("Lighting")

local portalFolder = workspace:WaitForChild("ActivePortals")

local POINT_COUNT = 96
local RADIUS_X = 3.15
local RADIUS_Y = 4.65

local T_FLICKER = 0.16
local T_FALLING = 0.28
local T_TRACE_START = 0.52
local T_TRACE_END = 1.48
local T_RIP_START = 1.34
local T_EXPAND_END = 2.62
local T_FINAL_BURST = 2.68
local T_STABLE = 3.05

local INITIAL_SCALE = 0.12
local CLOSE_VISUAL_END = 1.38

local WHITE = Color3.fromRGB(255, 250, 220)
local GOLD = Color3.fromRGB(255, 194, 55)
local ORANGE = Color3.fromRGB(255, 105, 14)
local DEEP = Color3.fromRGB(210, 47, 2)

local visuals = {}

local bloom =
	Lighting:FindFirstChild("PortalBloom")
	or Instance.new("BloomEffect")

bloom.Name = "PortalBloom"
bloom.Intensity = 0.52
bloom.Size = 24
bloom.Threshold = 1.08
bloom.Parent = Lighting

local function clamp01(x)
	return math.clamp(x, 0, 1)
end

local function smoothstep(x)
	x = clamp01(x)

	return
		x
		*
		x
		*
		(
			3
			-
			2 * x
		)
end

local function easeOutCubic(x)
	x = clamp01(x)

	return
		1
	-
		(
			1
			-
			x
		)
		^
		3
end

local function ellipsePoint(
	angle,
	rx,
	ry,
	z
)

	return Vector3.new(
		math.cos(angle) * rx,
		math.sin(angle) * ry,
		z or 0
	)
end

local function ellipseTangent(
	angle,
	rx,
	ry
)

	local v =
		Vector3.new(
			-rx * math.sin(angle),
			ry * math.cos(angle),
			0
		)

	return
		v.Magnitude > 0.001
		and v.Unit
		or Vector3.yAxis
end

local function ellipseOutward(
	angle,
	rx,
	ry
)

	local v =
		Vector3.new(
			math.cos(angle)
			/
			math.max(
				rx,
				0.001
			),

			math.sin(angle)
			/
			math.max(
				ry,
				0.001
			),

			0
		)

	return
		v.Magnitude > 0.001
		and v.Unit
		or Vector3.xAxis
end

local function makeAnchor(
	parent,
	name
)

	local part =
		Instance.new("Part")

	part.Name =
		name
		or
		"PortalVFXAnchor"

	part.Size =
		Vector3.new(
			0.03,
			0.03,
			0.03
		)

	part.Transparency = 1
	part.Anchored = true
	part.CanCollide = false
	part.CanTouch = false
	part.CanQuery = false
	part.CastShadow = false
	part.Parent = parent

	local attachment =
		Instance.new(
			"Attachment"
		)

	attachment.Parent =
		part

	return
		part,
		attachment
end

local function makeBeam(
	parent,
	a0,
	a1,
	width,
	colors
)

	local beam =
		Instance.new(
			"Beam"
		)

	beam.Attachment0 = a0
	beam.Attachment1 = a1

	beam.FaceCamera = true
	beam.Segments = 2

	beam.Width0 =
		width

	beam.Width1 =
		width * 0.55

	beam.Color =
		colors

	beam.LightEmission = 1
	beam.LightInfluence = 0

	beam.Transparency =
		NumberSequence.new({
			NumberSequenceKeypoint.new(
				0,
				0.015
			),

			NumberSequenceKeypoint.new(
				0.60,
				0.045
			),

			NumberSequenceKeypoint.new(
				1,
				0.30
			),
		})

	beam.Enabled = false
	beam.Parent = parent

	return beam
end

local function makeEmitter(
	attachment,
	kind
)

	local emitter =
		Instance.new(
			"ParticleEmitter"
		)

	emitter.Enabled = false

	emitter.Texture =
		"rbxasset://textures/particles/sparkles_main.dds"

	emitter.LightEmission = 1
	emitter.LightInfluence = 0

	emitter.Orientation =
		Enum.ParticleOrientation.VelocityParallel

	emitter.EmissionDirection =
		Enum.NormalId.Front

	emitter.ZOffset =
		0.15

	emitter.Color =
		ColorSequence.new({
			ColorSequenceKeypoint.new(
				0,
				WHITE
			),

			ColorSequenceKeypoint.new(
				0.14,
				GOLD
			),

			ColorSequenceKeypoint.new(
				0.62,
				ORANGE
			),

			ColorSequenceKeypoint.new(
				1,
				DEEP
			),
		})

	emitter.Transparency =
		NumberSequence.new({
			NumberSequenceKeypoint.new(
				0,
				0
			),

			NumberSequenceKeypoint.new(
				0.62,
				0.07
			),

			NumberSequenceKeypoint.new(
				1,
				1
			),
		})

	if kind ==
		"Needle" then

		emitter.Lifetime =
			NumberRange.new(
				0.18,
				0.48
			)

		emitter.Speed =
			NumberRange.new(
				24,
				52
			)

		emitter.Drag = 4.3

		emitter.Acceleration =
			Vector3.new(
				0,
				-18,
				0
			)

		emitter.SpreadAngle =
			Vector2.new(
				9,
				9
			)

		emitter.Size =
			NumberSequence.new({
				NumberSequenceKeypoint.new(
					0,
					0.13
				),

				NumberSequenceKeypoint.new(
					0.22,
					0.075
				),

				NumberSequenceKeypoint.new(
					1,
					0
				),
			})

		emitter.Squash =
			NumberSequence.new({
				NumberSequenceKeypoint.new(
					0,
					-0.95
				),

				NumberSequenceKeypoint.new(
					1,
					-0.66
				),
			})

	elseif kind ==
		"Shower" then

		emitter.Lifetime =
			NumberRange.new(
				0.40,
				1.05
			)

		emitter.Speed =
			NumberRange.new(
				11,
				32
			)

		emitter.Drag = 1.8

		emitter.Acceleration =
			Vector3.new(
				0,
				-36,
				0
			)

		emitter.SpreadAngle =
			Vector2.new(
				44,
				44
			)

		emitter.Size =
			NumberSequence.new({
				NumberSequenceKeypoint.new(
					0,
					0.115
				),

				NumberSequenceKeypoint.new(
					0.45,
					0.055
				),

				NumberSequenceKeypoint.new(
					1,
					0
				),
			})

		emitter.Squash =
			NumberSequence.new({
				NumberSequenceKeypoint.new(
					0,
					-0.89
				),

				NumberSequenceKeypoint.new(
					1,
					-0.34
				),
			})

	elseif kind ==
		"Ember" then

		emitter.Lifetime =
			NumberRange.new(
				0.85,
				2.15
			)

		emitter.Speed =
			NumberRange.new(
				2,
				10
			)

		emitter.Drag = 1

		emitter.Acceleration =
			Vector3.new(
				0,
				-26,
				0
			)

		emitter.SpreadAngle =
			Vector2.new(
				68,
				68
			)

		emitter.Size =
			NumberSequence.new({
				NumberSequenceKeypoint.new(
					0,
					0.08
				),

				NumberSequenceKeypoint.new(
					0.65,
					0.035
				),

				NumberSequenceKeypoint.new(
					1,
					0
				),
			})

	else

		emitter.Lifetime =
			NumberRange.new(
				0.10,
				0.30
			)

		emitter.Speed =
			NumberRange.new(
				6,
				20
			)

		emitter.Drag = 4.2

		emitter.Acceleration =
			Vector3.new(
				0,
				-13,
				0
			)

		emitter.SpreadAngle =
			Vector2.new(
				38,
				38
			)

		emitter.Size =
			NumberSequence.new({
				NumberSequenceKeypoint.new(
					0,
					0.06
				),

				NumberSequenceKeypoint.new(
					1,
					0
				),
			})
	end

	emitter.Parent =
		attachment

	return emitter
end

local function emitPack(
	pack,
	needles,
	shower,
	micro,
	ember
)

	if needles > 0 then
		pack.Needle:Emit(
			needles
		)
	end

	if shower > 0 then
		pack.Shower:Emit(
			shower
		)
	end

	if micro > 0 then
		pack.Micro:Emit(
			micro
		)
	end

	if ember > 0 then
		pack.Ember:Emit(
			ember
		)
	end
end

local function getCloseShape(
	closeStarted
)

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
			elapsed * 37
		)
		*
		0.020
		*
		remaining

	local yNoise =
		math.sin(
			elapsed * 43
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
					elapsed * 49
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
			remaining ^ 0.91
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

local function createVFX(
	portal
)

	if visuals[
		portal
		]
			or
			not portal:IsA(
				"BasePart"
			) then

		return
	end

	local model =
		Instance.new(
			"Model"
		)

	model.Name =
		"PortalVFX_"
		..
		portal.Name

	model.Parent =
		workspace

	local layerSpecs = {
		{
			width = 0.060,
			offset = -0.020,
			phase = 0,
			color =
				ColorSequence.new(
					WHITE,
					GOLD
				),
		},

		{
			width = 0.048,
			offset = 0.045,
			phase = 5,
			color =
				ColorSequence.new(
					GOLD,
					ORANGE
				),
		},

		{
			width = 0.034,
			offset = 0.115,
			phase = 11,
			color =
				ColorSequence.new(
					WHITE,
					ORANGE
				),
		},

		{
			width = 0.024,
			offset = 0.205,
			phase = 17,
			color =
				ColorSequence.new(
					ORANGE,
					DEEP
				),
		},

		{
			width = 0.015,
			offset = 0.305,
			phase = 29,
			color =
				ColorSequence.new(
					GOLD,
					DEEP
				),
		},
	}

	local layers = {}

	for _, spec in ipairs(
		layerSpecs
		) do

		local layer = {
			Parts = {},
			Attachments = {},
			Beams = {},
			Spec = spec,
		}

		for i = 1, POINT_COUNT do

			layer.Parts[
			i
			],
				layer.Attachments[
			i
			] =
				makeAnchor(
					model,
					"RimPoint"
				)
		end

		for i = 1, POINT_COUNT do

			layer.Beams[
			i
			] =
				makeBeam(
					model,

					layer.Attachments[
					i
					],

					layer.Attachments[
					(
						i
						%
						POINT_COUNT
					)
					+
					1
					],

					spec.width,
					spec.color
				)
		end

		table.insert(
			layers,
			layer
		)
	end

	local heads = {}

	for i = 1, 3 do

		local part,
			attachment =
			makeAnchor(
				model,
				"TracerHead"
				..
				i
			)

		local pack = {
			Part = part,

			Attachment =
				attachment,

			Needle =
				makeEmitter(
					attachment,
					"Needle"
				),

			Shower =
				makeEmitter(
					attachment,
					"Shower"
				),

			Micro =
				makeEmitter(
					attachment,
					"Micro"
				),

			Ember =
				makeEmitter(
					attachment,
					"Ember"
				),
		}

		local light =
			Instance.new(
				"PointLight"
			)

		light.Color =
			GOLD

		light.Range =
			9

		light.Brightness =
			0

		light.Shadows =
			false

		light.Parent =
			attachment

		pack.Light =
			light

		table.insert(
			heads,
			pack
		)
	end

	local rimEmitters = {}

	for i = 1, POINT_COUNT, 4 do

		local attachment =
			layers[
		2
		].Attachments[
		i
		]

		table.insert(
			rimEmitters,
			{
				Index = i,

				Needle =
					makeEmitter(
						attachment,
						"Needle"
					),

				Shower =
					makeEmitter(
						attachment,
						"Shower"
					),

				Micro =
					makeEmitter(
						attachment,
						"Micro"
					),

				Ember =
					makeEmitter(
						attachment,
						"Ember"
					),
			}
		)
	end

	local rimLights = {}

	local lightIndices = {
		1,
		25,
		49,
		73,
	}

	for _, index in ipairs(
		lightIndices
		) do

		local light =
			Instance.new(
				"PointLight"
			)

		light.Color =
			ORANGE

		light.Range =
			8

		light.Brightness =
			0

		light.Shadows =
			false

		light.Parent =
			layers[
		2
		].Attachments[
		index
		]

		table.insert(
			rimLights,
			light
		)
	end

	visuals[
	portal
	] =
		{
			Portal =
			portal,

			Model =
			model,

			Layers =
			layers,

			Heads =
			heads,

			RimEmitters =
			rimEmitters,

			RimLights =
			rimLights,

			OpenStarted =
			portal:GetAttribute(
				"OpenStarted"
			)
			or
			workspace:GetServerTimeNow(),

			Seed =
			math.random()
			*
			10000,

			LastVisible =
			0,

			DidFlicker =
			false,

			DidFall =
			false,

			DidRip =
			false,

			DidFinal =
			false,

			Closing =
			false,

			CloseLocalStart =
			nil,

			CloseBurst =
			false,

			NextTraceEmit =
			0,

			NextStormEmit =
			0,

			NextStableEmit =
			0,

			NextCloseEmit =
			0,
		}
end

local function setAllBeams(
	data,
	enabled
)

	for _, layer in ipairs(
		data.Layers
		) do

		for _, beam in ipairs(
			layer.Beams
			) do

			beam.Enabled =
				enabled
		end
	end
end

local function updateRimLights(
	data,
	brightness,
	now
)

	for index, light in ipairs(
		data.RimLights
		) do

		local flicker =
			0.72
			+
			0.28
			*
			math.sin(
				now * 17
				+
				index * 2.1
				+
				data.Seed
			)

		light.Brightness =
			math.max(
				0,
				brightness
				*
				flicker
			)
	end
end

local function updateClosing(
	data,
	now,
	portalCF
)

	if not data.Closing then

		data.Closing =
			true

		data.CloseLocalStart =
			data.Portal:GetAttribute(
				"CloseStarted"
			)
			or
			workspace:GetServerTimeNow()

		for _, rim in ipairs(
			data.RimEmitters
			) do

			rim.Needle:Emit(
				math.random(
					5,
					9
				)
			)

			rim.Shower:Emit(
				math.random(
					3,
					6
				)
			)

			rim.Micro:Emit(
				math.random(
					7,
					12
				)
			)
		end
	end

	local closeElapsed,
		closeProgress,
		scaleX,
		scaleY =
		getCloseShape(
			data.CloseLocalStart
		)

	local remaining =
		1
	-
		closeProgress

	for layerIndex, layer in ipairs(
		data.Layers
		) do

		local spec =
			layer.Spec

		for i = 1, POINT_COUNT do

			local fraction =
				(
					i
					-
					1
				)
				/
				POINT_COUNT

			local angle =
				-math.pi / 2
				+
				fraction
				*
				math.pi
				*
				2

			local slowNoise =
				math.noise(
					data.Seed
					+
					i * 0.17,

					now * 8.5,

					layerIndex * 2.3
				)

			local fastNoise =
				math.noise(
					data.Seed
					+
					i * 0.43,

					now * 17,

					layerIndex * 4.1
				)

			local bite =
				math.sin(
					angle * 5
					+
					now * 19
					+
					layerIndex
				)
				*
				0.035
				*
				remaining

			local turbulence =
				(
					slowNoise
					*
					0.13
					+
					fastNoise
					*
					0.045
					+
					bite
				)
				*
				remaining

			local rx =
				math.max(
					0.02,

					(
						RADIUS_X
						+
						spec.offset
						+
						turbulence
					)
					*
					scaleX
				)

			local ry =
				math.max(
					0.02,

					(
						RADIUS_Y
						+
						spec.offset
						+
						turbulence
					)
					*
					scaleY
				)

			local depth =
				fastNoise
				*
				0.07
				*
				remaining

			local worldPosition =
				portalCF:PointToWorldSpace(
					ellipsePoint(
						angle,
						rx,
						ry,
						depth
					)
				)

			local tangent =
				portalCF:VectorToWorldSpace(
					ellipseTangent(
						angle,
						rx,
						ry
					)
				)

			layer.Parts[
			i
			].CFrame =
				CFrame.lookAt(
					worldPosition,
					worldPosition
					+
					tangent
				)

			local beam =
				layer.Beams[
			i
			]

			beam.Enabled =
				closeElapsed
				<
				CLOSE_VISUAL_END

			local flare =
				1
				+
				closeProgress
				*
				0.75

			local pulse =
				0.82
				+
				0.30
				*
				(
					0.5
					+
					0.5
					*
					math.sin(
						now * 24
						+
						i * 0.71
						+
						layerIndex * 1.9
					)
				)

			local finalFade =
				1

			if closeElapsed > 1.30 then

				finalFade =
					math.clamp(
						(
							CLOSE_VISUAL_END
							-
							closeElapsed
						)
						/
						0.08,

						0,
						1
					)
			end

			beam.Width0 =
				spec.width
				*
				pulse
				*
				flare
				*
				finalFade

			beam.Width1 =
				spec.width
				*
				0.55
				*
				pulse
				*
				flare
				*
				finalFade

			beam.CurveSize0 =
				slowNoise
				*
				0.34
				*
				remaining

			beam.CurveSize1 =
				-fastNoise
				*
				0.34
				*
				remaining
		end
	end

	local closeRadiusX =
		RADIUS_X
		*
		scaleX

	local closeRadiusY =
		RADIUS_Y
		*
		scaleY

	for index, head in ipairs(
		data.Heads
		) do

		local angle =
			-math.pi / 2
			+
			(
				(
					index
					-
					1
				)
				/
				#data.Heads
			)
			*
			math.pi
			*
			2
			+
			now
			*
			2
			*
			remaining

		local localPosition

		if closeElapsed >= 1.20 then

			localPosition =
				Vector3.new(
					0,
					0,
					-0.08
					-
					index
					*
					0.01
				)

		else

			localPosition =
				ellipsePoint(
					angle,
					closeRadiusX,
					closeRadiusY,
					-0.07
					-
					index
					*
					0.01
				)
		end

		local worldPosition =
			portalCF:PointToWorldSpace(
				localPosition
			)

		head.Part.CFrame =
			CFrame.lookAt(
				worldPosition,
				worldPosition
				-
				portalCF.LookVector
			)

		head.Light.Brightness =
			math.max(
				0,

				(
					1.4
					+
					closeProgress
					*
					4.2
				)
				*
				remaining
			)
	end

	updateRimLights(
		data,

		closeElapsed
			<
			CLOSE_VISUAL_END
			and
			(
				0.55
				+
				closeProgress
				*
				2
			)
			or
			0,

		now
	)

	if now >=
		data.NextCloseEmit
		and
		closeElapsed
		<
		CLOSE_VISUAL_END then

		data.NextCloseEmit =
			now
			+
			0.035

		local burstCount =
			math.clamp(
				math.floor(
					3
					+
					closeProgress
					*
					7
				),

				3,
				10
			)

		for _ = 1, burstCount do

			local rim =
				data.RimEmitters[
			math.random(
				1,
				#data.RimEmitters
			)
			]

			rim.Needle:Emit(
				math.random(
					1,

					4
						+
						math.floor(
							closeProgress
							*
							4
						)
				)
			)

			rim.Micro:Emit(
				math.random(
					2,

					6
						+
						math.floor(
							closeProgress
							*
							5
						)
				)
			)

			if math.random() <
				0.55
				+
				closeProgress
				*
				0.25 then

				rim.Shower:Emit(
					math.random(
						1,
						3
					)
				)
			end

			if math.random() <
				0.12
				+
				closeProgress
				*
				0.20 then

				rim.Ember:Emit(
					math.random(
						1,
						2
					)
				)
			end
		end
	end

	if closeElapsed >= 1.23
		and
		not data.CloseBurst then

		data.CloseBurst =
			true

		for _, head in ipairs(
			data.Heads
			) do

			emitPack(
				head,
				24,
				18,
				32,
				10
			)
		end

		for _, rim in ipairs(
			data.RimEmitters
			) do

			rim.Needle:Emit(
				math.random(
					5,
					10
				)
			)

			rim.Shower:Emit(
				math.random(
					3,
					7
				)
			)

			rim.Micro:Emit(
				math.random(
					8,
					15
				)
			)

			if math.random() < 0.55 then

				rim.Ember:Emit(
					math.random(
						1,
						3
					)
				)
			end
		end
	end

	if closeElapsed >=
		CLOSE_VISUAL_END then

		setAllBeams(
			data,
			false
		)

		updateRimLights(
			data,
			0,
			now
		)

		for _, head in ipairs(
			data.Heads
			) do

			head.Light.Brightness =
				0
		end
	end
end

local function updateOpening(
	data,
	now,
	portalCF
)

	local openStarted =
		data.Portal:GetAttribute(
			"OpenStarted"
		)
		or
		data.OpenStarted

	local elapsed =
		workspace:GetServerTimeNow()
	-
		openStarted

	local center =
		portalCF.Position
	-
		portalCF.LookVector
		*
		0.055

	if elapsed <
		T_TRACE_START then

		setAllBeams(
			data,
			false
		)
	end

	if elapsed >=
		T_FLICKER
		and
		not data.DidFlicker then

		data.DidFlicker =
			true

		local head =
			data.Heads[
		1
		]

		head.Part.CFrame =
			CFrame.lookAt(
				center,
				center
				-
				portalCF.LookVector
			)

		emitPack(
			head,
			7,
			3,
			22,
			3
		)

		head.Light.Brightness =
			4.2

		task.delay(
			0.07,
			function()

				if head.Light then
					head.Light.Brightness =
						0
				end
			end
		)
	end

	if elapsed >=
		T_FALLING
		and
		not data.DidFall then

		data.DidFall =
			true

		local head =
			data.Heads[
		1
		]

		head.Part.CFrame =
			CFrame.lookAt(
				center,
				center
				-
				portalCF.LookVector
			)

		emitPack(
			head,
			18,
			38,
			34,
			20
		)
	end

	local traceRaw =
		clamp01(
			(
				elapsed
				-
				T_TRACE_START
			)
			/
			(
				T_TRACE_END
				-
				T_TRACE_START
			)
		)

	local trace =
		smoothstep(
			traceRaw
		)

	local visibleCount =
		math.floor(
			trace
			*
			POINT_COUNT
		)

	local traceScale =
		INITIAL_SCALE
		+
		smoothstep(
			traceRaw
		)
		*
		0.10

	local ripRaw =
		clamp01(
			(
				elapsed
				-
				T_RIP_START
			)
			/
			(
				T_EXPAND_END
				-
				T_RIP_START
			)
		)

	local rip =
		easeOutCubic(
			ripRaw
		)

	local finalScale =
		traceScale
		+
		(
			1
			-
			traceScale
		)
		*
		rip

	local unstable =
		1
	-
		ripRaw

	local scaleJitter =
		math.noise(
			data.Seed,
			now * 8.5
		)
		*
		0.032
		*
		unstable

	finalScale =
		math.clamp(
			finalScale
			+
			scaleJitter,

			0.03,
			1.03
		)

	for layerIndex, layer in ipairs(
		data.Layers
		) do

		local spec =
			layer.Spec

		for i = 1, POINT_COUNT do

			local fraction =
				(
					i
					-
					1
				)
				/
				POINT_COUNT

			local angle =
				-math.pi / 2
				+
				fraction
				*
				math.pi
				*
				2

			local slowNoise =
				math.noise(
					data.Seed
					+
					i * 0.19,

					now * 3.6,

					layerIndex * 4.7
				)

			local fastNoise =
				math.noise(
					data.Seed
					+
					i * 0.51,

					now * 11.2,

					layerIndex * 2.4
				)

			local wave =
				math.sin(
					angle * 6
					+
					now * 7.5
					+
					layerIndex * 1.7
				)
				*
				0.050

			local tooth =
				math.max(
					0,

					math.sin(
						angle * 11
						-
						now * 13
						+
						data.Seed
					)
				)
				^
				7
				*
				0.055

			local ragged =
				(
					slowNoise
					*
					0.16

					+
					fastNoise
					*
					0.060

					+
					wave
					*
					unstable

					+
					tooth
					*
					unstable
				)
				*
				math.max(
					finalScale,
					0.15
				)

			local rx =
				(
					RADIUS_X
					+
					spec.offset
					+
					ragged
				)
				*
				finalScale

			local ry =
				(
					RADIUS_Y
					+
					spec.offset
					+
					ragged
				)
				*
				finalScale

			local depth =
				fastNoise
				*
				0.090
				+
				slowNoise
				*
				0.035

			local localPosition =
				ellipsePoint(
					angle,
					rx,
					ry,
					depth
				)

			local worldPosition =
				portalCF:PointToWorldSpace(
					localPosition
				)

			local tangent =
				portalCF:VectorToWorldSpace(
					ellipseTangent(
						angle,
						rx,
						ry
					)
				)

			layer.Parts[
			i
			].CFrame =
				CFrame.lookAt(
					worldPosition,
					worldPosition
					+
					tangent
				)

			local beam =
				layer.Beams[
			i
			]

			local shifted =
				(
					(
						i
						+
						spec.phase
						-
						2
					)
					%
					POINT_COUNT
				)
				+
				1

			local traced =
				shifted
				<=
				visibleCount

			local electricalGap =
				traceRaw
				<
				0.96
				and
				(
					(
						i
						+
						layerIndex * 7
					)
					%
					29
					==
					0

					or

					(
						i
						+
						layerIndex * 5
					)
					%
					43
					==
					0
				)

			beam.Enabled =
				elapsed
				>=
				T_TRACE_START
				and
				traced
				and
				not electricalGap

			local pulse =
				0.78
				+
				0.40
				*
				(
					0.5
					+
					0.5
					*
					math.sin(
						now * 20
						+
						i * 0.81
						+
						layerIndex * 1.9
					)
				)

			local surge =
				1
				+
				math.max(
					0,

					math.sin(
						now * 9
						+
						angle * 3
						+
						layerIndex
					)
				)
				^
				9
				*
				0.45
				*
				unstable

			beam.Width0 =
				spec.width
				*
				pulse
				*
				surge

			beam.Width1 =
				spec.width
				*
				0.55
				*
				pulse
				*
				surge

			beam.CurveSize0 =
				slowNoise
				*
				0.30

			beam.CurveSize1 =
				-fastNoise
				*
				0.30
		end
	end

	if elapsed >=
		T_TRACE_START
		and
		traceRaw < 1 then

		local offsets = {
			0,
			-0.035,
			-0.072,
		}

		for index, head in ipairs(
			data.Heads
			) do

			local localTrace =
				clamp01(
					trace
					+
					offsets[
					index
					]
				)

			local angle =
				-math.pi / 2
				+
				localTrace
				*
				math.pi
				*
				2

			local rx =
				RADIUS_X
				*
				traceScale

			local ry =
				RADIUS_Y
				*
				traceScale

			local localPosition =
				ellipsePoint(
					angle,
					rx,
					ry,
					-0.07
					-
					index
					*
					0.012
				)

			local worldPosition =
				portalCF:PointToWorldSpace(
					localPosition
				)

			local tangent =
				portalCF:VectorToWorldSpace(
					ellipseTangent(
						angle,
						rx,
						ry
					)
				)

			local outward =
				portalCF:VectorToWorldSpace(
					ellipseOutward(
						angle,
						rx,
						ry
					)
				)

			local direction =
				(
					tangent
					+
					outward
					*
					0.32
				).Unit

			head.Part.CFrame =
				CFrame.lookAt(
					worldPosition,
					worldPosition
					+
					direction
				)

			head.Light.Brightness =
				2.5
				+
				math.random()
				*
				1.8
		end

		if now >=
			data.NextTraceEmit then

			data.NextTraceEmit =
				now
				+
				0.035

			for _, head in ipairs(
				data.Heads
				) do

				emitPack(
					head,

					math.random(
						8,
						13
					),

					math.random(
						5,
						9
					),

					math.random(
						9,
						15
					),

					math.random(
						2,
						4
					)
				)
			end
		end

	else

		for _, head in ipairs(
			data.Heads
			) do

			head.Light.Brightness =
				0
		end
	end

	if visibleCount >
		data.LastVisible then

		for _, rim in ipairs(
			data.RimEmitters
			) do

			if rim.Index >
				data.LastVisible
					and
					rim.Index <=
					visibleCount then

				rim.Needle:Emit(
					math.random(
						4,
						8
					)
				)

				rim.Shower:Emit(
					math.random(
						2,
						5
					)
				)

				rim.Micro:Emit(
					math.random(
						5,
						10
					)
				)

				if math.random() <
					0.55 then

					rim.Ember:Emit(
						math.random(
							1,
							2
						)
					)
				end
			end
		end

		data.LastVisible =
			visibleCount
	end

	if elapsed >=
		T_RIP_START
		and
		not data.DidRip then

		data.DidRip =
			true

		for _, head in ipairs(
			data.Heads
			) do

			emitPack(
				head,
				22,
				26,
				30,
				10
			)
		end

		for _, rim in ipairs(
			data.RimEmitters
			) do

			if rim.Index <=
				math.max(
					visibleCount,
					1
				) then

				rim.Needle:Emit(
					math.random(
						3,
						6
					)
				)

				rim.Shower:Emit(
					math.random(
						2,
						4
					)
				)
			end
		end
	end

	if ripRaw > 0
		and
		ripRaw < 1
		and
		now >=
		data.NextStormEmit then

		data.NextStormEmit =
			now
			+
			0.045

		local burstCount =
			5
			+
			math.floor(
				(
					1
					-
					math.abs(
						ripRaw
						-
						0.55
					)
				)
				*
				5
			)

		for _ = 1, burstCount do

			local rim =
				data.RimEmitters[
			math.random(
				1,
				#data.RimEmitters
			)
			]

			rim.Micro:Emit(
				math.random(
					2,
					5
				)
			)

			if math.random() <
				0.68 then

				rim.Needle:Emit(
					math.random(
						1,
						3
					)
				)
			end

			if math.random() <
				0.42 then

				rim.Shower:Emit(
					math.random(
						1,
						2
					)
				)
			end

			if math.random() <
				0.18 then

				rim.Ember:Emit(
					1
				)
			end
		end
	end

	if elapsed >=
		T_FINAL_BURST
		and
		not data.DidFinal then

		data.DidFinal =
			true

		for _, rim in ipairs(
			data.RimEmitters
			) do

			rim.Needle:Emit(
				math.random(
					4,
					8
				)
			)

			rim.Shower:Emit(
				math.random(
					2,
					5
				)
			)

			rim.Micro:Emit(
				math.random(
					6,
					12
				)
			)

			if math.random() <
				0.60 then

				rim.Ember:Emit(
					math.random(
						1,
						2
					)
				)
			end
		end
	end

	local formationLight =
		0

	if elapsed >=
		T_TRACE_START then

		formationLight =
			0.55
			+
			rip
			*
			0.55
	end

	updateRimLights(
		data,
		formationLight,
		now
	)

	if elapsed >=
		T_STABLE
		and
		now >=
		data.NextStableEmit then

		data.NextStableEmit =
			now
			+
			0.075

		local stableBursts =
			math.random(
				3,
				5
			)

		for _ = 1, stableBursts do

			local rim =
				data.RimEmitters[
			math.random(
				1,
				#data.RimEmitters
			)
			]

			rim.Micro:Emit(
				math.random(
					1,
					3
				)
			)

			if math.random() <
				0.38 then

				rim.Needle:Emit(
					math.random(
						1,
						2
					)
				)
			end

			if math.random() <
				0.16 then

				rim.Shower:Emit(
					1
				)
			end

			if math.random() <
				0.10 then

				rim.Ember:Emit(
					1
				)
			end
		end
	end
end

local function updateVFX(
	data,
	now
)

	local portal =
		data.Portal

	if not portal.Parent then
		return
	end

	local state =
		portal:GetAttribute(
			"PortalState"
		)
		or
		"Opening"

	local portalCF =
		portal.CFrame

	if state ==
		"Closing" then

		updateClosing(
			data,
			now,
			portalCF
		)

		return
	end

	if state ==
		"Closed" then

		setAllBeams(
			data,
			false
		)

		updateRimLights(
			data,
			0,
			now
		)

		for _, head in ipairs(
			data.Heads
			) do

			head.Light.Brightness =
				0
		end

		return
	end

	updateOpening(
		data,
		now,
		portalCF
	)
end

local function destroyVFX(
	portal
)

	local data =
		visuals[
	portal
	]

	if not data then
		return
	end

	visuals[
	portal
	] =
		nil

	if data.Model then

		task.delay(
			0.9,
			function()

				if data.Model then
					data.Model:Destroy()
				end
			end
		)
	end
end

for _, portal in ipairs(
	portalFolder:GetChildren()
	) do

	if portal:IsA(
		"BasePart"
		) then

		createVFX(
			portal
		)
	end
end

portalFolder.ChildAdded:Connect(
	function(
		portal
	)

		if portal:IsA(
			"BasePart"
			) then

			task.defer(
				createVFX,
				portal
			)
		end
	end
)

portalFolder.ChildRemoved:Connect(
	destroyVFX
)

RunService.RenderStepped:Connect(
	function()

		local now =
			os.clock()

		for portal, data in pairs(
			visuals
			) do

			if portal.Parent then

				updateVFX(
					data,
					now
				)

			else

				destroyVFX(
					portal
				)
			end
		end
	end
)

print(
	"[PortalVFX] V8.9 CINEMATIC VFX ONLINE"
)
```
