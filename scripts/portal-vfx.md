[← Back to Home](../README.md)

# Procedural Ring, Particle & Lighting Effects

Client-side portal opening and closing VFX built using layered beams, procedural ring geometry, noise-based movement and timed particle emission.

Selected portions of `PortalVFX`. Particle presets, object setup and cleanup are omitted.


## Ellipse Geometry

Portion of the geometry helpers. Calculates points, tangents and outward directions along the portal rim.

```lua
local function ellipsePoint(angle, rx, ry, z)
    return Vector3.new(
        math.cos(angle) * rx,
        math.sin(angle) * ry,
        z or 0
    )
end

local function ellipseTangent(angle, rx, ry)
    local v = Vector3.new(
        -rx * math.sin(angle),
        ry * math.cos(angle),
        0
    )

    return v.Magnitude > 0.001 and v.Unit or Vector3.yAxis
end

local function ellipseOutward(angle, rx, ry)
    local v = Vector3.new(
        math.cos(angle) / math.max(rx, 0.001),
        math.sin(angle) / math.max(ry, 0.001),
        0
    )

    return v.Magnitude > 0.001 and v.Unit or Vector3.xAxis
end
```


## Layered Ring Construction

Portion of `createVFX()`. Builds the rim from connected Beam segments across several layers.

```lua
local layers = {}

for _, spec in ipairs(layerSpecs) do
    local layer = {
        Parts = {},
        Attachments = {},
        Beams = {},
        Spec = spec,
    }

    for i = 1, POINT_COUNT do
        layer.Parts[i], layer.Attachments[i] = makeAnchor(model, "RimPoint")
    end

    for i = 1, POINT_COUNT do
        layer.Beams[i] = makeBeam(
            model,
            layer.Attachments[i],
            layer.Attachments[(i % POINT_COUNT) + 1],
            spec.width,
            spec.color
        )
    end

    table.insert(layers, layer)
end
```


## Opening Progression

Portion of `updateOpening()`. Controls the trace, expansion and unstable scale of the opening portal.

```lua
local traceRaw = clamp01((elapsed - T_TRACE_START) / (T_TRACE_END - T_TRACE_START))
local trace = smoothstep(traceRaw)
local visibleCount = math.floor(trace * POINT_COUNT)
local traceScale = INITIAL_SCALE + smoothstep(traceRaw) * 0.10

local ripRaw = clamp01((elapsed - T_RIP_START) / (T_EXPAND_END - T_RIP_START))
local rip = easeOutCubic(ripRaw)
local finalScale = traceScale + (1 - traceScale) * rip
local unstable = 1 - ripRaw

local scaleJitter = math.noise(data.Seed, now * 8.5) * 0.032 * unstable
finalScale = math.clamp(finalScale + scaleJitter, 0.03, 1.03)
```


## Rim Deformation & Beam Animation

Portion of `updateOpening()`. Uses multiple noise samples and oscillations to animate each segment as the ring forms.

```lua
for layerIndex, layer in ipairs(data.Layers) do
    local spec = layer.Spec

    for i = 1, POINT_COUNT do
        local fraction = (i - 1) / POINT_COUNT
        local angle = -math.pi / 2 + fraction * math.pi * 2

        local slowNoise = math.noise(
            data.Seed + i * 0.19,
            now * 3.6,
            layerIndex * 4.7
        )

        local fastNoise = math.noise(
            data.Seed + i * 0.51,
            now * 11.2,
            layerIndex * 2.4
        )

        local wave = math.sin(angle * 6 + now * 7.5 + layerIndex * 1.7) * 0.050
        local tooth = math.max(0, math.sin(angle * 11 - now * 13 + data.Seed)) ^ 7 * 0.055

        local ragged = (
            slowNoise * 0.16
            + fastNoise * 0.060
            + wave * unstable
            + tooth * unstable
        ) * math.max(finalScale, 0.15)

        local rx = (RADIUS_X + spec.offset + ragged) * finalScale
        local ry = (RADIUS_Y + spec.offset + ragged) * finalScale
        local depth = fastNoise * 0.090 + slowNoise * 0.035

        local localPosition = ellipsePoint(angle, rx, ry, depth)
        local worldPosition = portalCF:PointToWorldSpace(localPosition)
        local tangent = portalCF:VectorToWorldSpace(ellipseTangent(angle, rx, ry))

        layer.Parts[i].CFrame = CFrame.lookAt(worldPosition, worldPosition + tangent)

        local beam = layer.Beams[i]
        local shifted = ((i + spec.phase - 2) % POINT_COUNT) + 1
        local traced = shifted <= visibleCount

        local electricalGap = traceRaw < 0.96 and (
            (i + layerIndex * 7) % 29 == 0
            or (i + layerIndex * 5) % 43 == 0
        )

        beam.Enabled = elapsed >= T_TRACE_START and traced and not electricalGap

        local pulse = 0.78 + 0.40 * (
            0.5 + 0.5 * math.sin(now * 20 + i * 0.81 + layerIndex * 1.9)
        )

        local surge = 1 + math.max(
            0,
            math.sin(now * 9 + angle * 3 + layerIndex)
        ) ^ 9 * 0.45 * unstable

        beam.Width0 = spec.width * pulse * surge
        beam.Width1 = spec.width * 0.55 * pulse * surge
        beam.CurveSize0 = slowNoise * 0.30
        beam.CurveSize1 = -fastNoise * 0.30
    end
end
```


## Spark Tracer Movement

Portion of `updateOpening()`. Moves the tracer heads along the elliptical ring and angles their particle emitters along the curve.

```lua
if elapsed >= T_TRACE_START and traceRaw < 1 then
    local offsets = {
        0,
        -0.035,
        -0.072,
    }

    for index, head in ipairs(data.Heads) do
        local localTrace = clamp01(trace + offsets[index])
        local angle = -math.pi / 2 + localTrace * math.pi * 2
        local rx = RADIUS_X * traceScale
        local ry = RADIUS_Y * traceScale

        local localPosition = ellipsePoint(angle, rx, ry, -0.07 - index * 0.012)
        local worldPosition = portalCF:PointToWorldSpace(localPosition)
        local tangent = portalCF:VectorToWorldSpace(ellipseTangent(angle, rx, ry))
        local outward = portalCF:VectorToWorldSpace(ellipseOutward(angle, rx, ry))
        local direction = (tangent + outward * 0.32).Unit

        head.Part.CFrame = CFrame.lookAt(worldPosition, worldPosition + direction)
        head.Light.Brightness = 2.5 + math.random() * 1.8
    end

    if now >= data.NextTraceEmit then
        data.NextTraceEmit = now + 0.035

        for _, head in ipairs(data.Heads) do
            emitPack(
                head,
                math.random(8, 13),
                math.random(5, 9),
                math.random(9, 15),
                math.random(2, 4)
            )
        end
    end
else
    for _, head in ipairs(data.Heads) do
        head.Light.Brightness = 0
    end
end
```


## Closing Shape

Source from `getCloseShape()`. Shrinks the opening with separate horizontal and vertical motion, jitter and a final snap.

```lua
local function getCloseShape(closeStarted)
    local elapsed = math.max(0, workspace:GetServerTimeNow() - closeStarted)

    if elapsed < 0.12 then
        return elapsed, 0, 1, 1
    end

    local progress = math.clamp((elapsed - 0.12) / 1.16, 0, 1)
    local remaining = 1 - progress
    local base = remaining ^ 1.10

    local xNoise = math.sin(elapsed * 37) * 0.020 * remaining
    local yNoise = math.sin(elapsed * 43 + 1.7) * 0.030 * remaining
    local snap = (math.max(0, math.sin(elapsed * 49)) ^ 16) * 0.032 * remaining

    local scaleX = math.clamp(base + xNoise - snap, 0.001, 1)
    local scaleY = math.clamp(remaining ^ 0.91 + yNoise, 0.001, 1)

    if elapsed >= 1.28 then
        local finalProgress = math.clamp((elapsed - 1.28) / 0.10, 0, 1)

        scaleX = math.max(0.001, 0.075 * (1 - finalProgress))
        scaleY = math.max(0.001, 0.31 * (1 - finalProgress))
    end

    return elapsed, progress, scaleX, scaleY
end
```


## Timed Particle Bursts

Portion of `updateOpening()`. Limits particle emissions while the portal is expanding.

```lua
if ripRaw > 0 and ripRaw < 1 and now >= data.NextStormEmit then
    data.NextStormEmit = now + 0.045

    local burstCount = 5 + math.floor((1 - math.abs(ripRaw - 0.55)) * 5)

    for _ = 1, burstCount do
        local rim = data.RimEmitters[math.random(1, #data.RimEmitters)]

        rim.Micro:Emit(math.random(2, 5))

        if math.random() < 0.68 then
            rim.Needle:Emit(math.random(1, 3))
        end

        if math.random() < 0.42 then
            rim.Shower:Emit(math.random(1, 2))
        end

        if math.random() < 0.18 then
            rim.Ember:Emit(1)
        end
    end
end
```
