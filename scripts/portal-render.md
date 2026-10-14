[← Back to Home](../README.md)

# Viewport World Streaming & Terrain Extraction Client

Renders the world beyond each portal using `ViewportFrame` and `WorldModel`. Nearby geometry is cloned in batches, terrain is reconstructed from voxel data, and moving objects and characters are kept in sync.

Selected portions of `PortalRender`. The full script also handles viewport setup, opening and closing effects, and cleanup.

## World streaming

Portion of `ensurePairWorldScenes()`. Builds both portal scenes in batches instead of cloning the entire map in one frame.

```lua
local function ensurePairWorldScenes(dataA, destinationA, dataB, destinationB)
    local needA = dataA.Destination ~= destinationA
        or (not dataA.SceneBuilt and not dataA.SceneBuilding)

    local needB = dataB.Destination ~= destinationB
        or (not dataB.SceneBuilt and not dataB.SceneBuilding)

    if not needA and not needB then
        return
    end

    if pairSceneBuilding then
        return
    end

    pairSceneBuilding = true

    if needA then
        prepareWorldScene(dataA, destinationA)
    end

    if needB then
        prepareWorldScene(dataB, destinationB)
    end

    task.spawn(function()
        local objects = workspace:GetDescendants()

        for index, object in ipairs(objects) do
            if needA and dataA.Portal and dataA.Portal.Parent and destinationA.Parent then
                addWorldPart(dataA, object)
            end

            if needB and dataB.Portal and dataB.Portal.Parent and destinationB.Parent then
                addWorldPart(dataB, object)
            end

            if index % WORLD_BUILD_BATCH == 0 then
                task.wait()
            end
        end

        if needA and dataA.Portal and dataA.Portal.Parent
            and dataA.Destination == destinationA then
            dataA.SceneBuilt = true
            dataA.SceneBuilding = false
        end

        if needB and dataB.Portal and dataB.Portal.Parent
            and dataB.Destination == destinationB then
            dataB.SceneBuilt = true
            dataB.SceneBuilding = false
        end

        pairSceneBuilding = false
    end)
end
```

## World clone tracking

Portion of `addWorldPart()`. Filters objects before cloning, then tracks moving parts separately from static geometry.

```lua
local function addWorldPart(data, original)
    if data.WorldClones[original] then
        return
    end

    if not shouldRenderWorldPart(data, original) then
        return
    end

    local clone = cloneWorldPart(original)

    if not clone then
        return
    end

    clone.Parent = data.World
    data.WorldClones[original] = clone

    if isDynamicWorldPart(original) then
        data.DynamicClones[original] = clone
    end
end
```

## Terrain surface extraction

Portion of `buildTerrainProxy()`. Checks voxel occupancy and stores exposed surface cells rather than rendering every solid voxel.

```lua
local nx = materials.Size.X
local ny = materials.Size.Y
local nz = materials.Size.Z

local regionCF = region.CFrame
local regionSize = region.Size

local function inside(x, y, z)
    return x >= 1 and x <= nx
        and y >= 1 and y <= ny
        and z >= 1 and z <= nz
end

local function solidVoxel(x, y, z)
    if not inside(x, y, z) then
        return false
    end

    local material = materials[x][y][z]

    if material == Enum.Material.Air then
        return false
    end

    local occupancy = occupancies[x][y][z]
    return occupancy > TERRAIN_OCCUPANCY_THRESHOLD
end

local candidates = {}

for x = 1, nx do
    for y = 1, ny do
        for z = 1, nz do
            if solidVoxel(x, y, z) then
                local exposedTop = not solidVoxel(x, y + 1, z)
                local exposedBottom = not solidVoxel(x, y - 1, z)
                local exposedLeft = not solidVoxel(x - 1, y, z)
                local exposedRight = not solidVoxel(x + 1, y, z)
                local exposedFront = not solidVoxel(x, y, z - 1)
                local exposedBack = not solidVoxel(x, y, z + 1)

                local surface = exposedTop or exposedBottom or exposedLeft
                    or exposedRight or exposedFront or exposedBack

                if surface then
                    local localX = -regionSize.X * 0.5
                        + (x - 0.5) * TERRAIN_VOXEL_RESOLUTION
                    local localY = -regionSize.Y * 0.5
                        + (y - 0.5) * TERRAIN_VOXEL_RESOLUTION
                    local localZ = -regionSize.Z * 0.5
                        + (z - 0.5) * TERRAIN_VOXEL_RESOLUTION

                    local worldPosition = regionCF:PointToWorldSpace(
                        Vector3.new(localX, localY, localZ)
                    )
                    local distance = (worldPosition - destination.Position).Magnitude

                    table.insert(candidates, {
                        LocalX = localX,
                        LocalY = localY,
                        LocalZ = localZ,
                        Material = materials[x][y][z],
                        Occupancy = occupancies[x][y][z],
                        TopExposed = exposedTop,
                        Distance = distance
                    })
                end
            end
        end
    end
end

table.sort(candidates, function(a, b)
    return a.Distance < b.Distance
end)
```

## Terrain proxy geometry

Another portion of `buildTerrainProxy()`. Converts selected surface cells into lightweight parts, accounting for voxel occupancy and material.

```lua
local occupancy = math.clamp(cell.Occupancy, 0, 1)
local cellHeight = TERRAIN_VOXEL_RESOLUTION
local yOffset = 0

if cell.TopExposed then
    cellHeight = math.max(
        0.65,
        TERRAIN_VOXEL_RESOLUTION * math.clamp(occupancy, 0.16, 1)
    )

    yOffset = -(TERRAIN_VOXEL_RESOLUTION - cellHeight) * 0.5
end

local block = Instance.new("Part")
block.Name = "TerrainSurfaceVoxel"
block.Anchored = true
block.CanCollide = false
block.CanTouch = false
block.CanQuery = false
block.CastShadow = false
block.Size = Vector3.new(
    TERRAIN_VOXEL_RESOLUTION + 0.04,
    cellHeight + 0.04,
    TERRAIN_VOXEL_RESOLUTION + 0.04
)
block.CFrame = regionCF * CFrame.new(
    cell.LocalX,
    cell.LocalY + yOffset,
    cell.LocalZ
)

if cell.Material == Enum.Material.Water then
    block.Material = Enum.Material.Glass
    block.Transparency = 0.38
else
    block.Material = cell.Material
    block.Transparency = 0
end

block.Color = terrainMaterialColor(cell.Material)
block.Parent = folder
```

## Dynamic world synchronization

`syncDynamicWorld()` updates the position and properties of moving objects without rebuilding the entire portal scene.

```lua
local function syncDynamicWorld(data)
    for original, clone in pairs(data.DynamicClones) do
        if not original or not original.Parent or not clone or not clone.Parent then
            removeWorldPart(data, original)
            continue
        end

        if not shouldRenderWorldPart(data, original) then
            removeWorldPart(data, original)
            continue
        end

        if clone.CFrame ~= original.CFrame then
            clone.CFrame = original.CFrame
        end

        if clone.Size ~= original.Size then
            clone.Size = original.Size
        end

        if clone.Transparency ~= original.Transparency then
            clone.Transparency = original.Transparency
        end
    end
end
```

## Character visibility

`updateCharacterCulling()` uses portal-local coordinates and separate visibility thresholds to prevent characters flickering across the portal plane.

```lua
local function updateCharacterCulling(data, destination)
    if not destination then
        return
    end

    for currentPlayer, entry in pairs(data.CharacterClones) do
        local character = currentPlayer.Character
        local clone = entry.Clone

        if character and clone then
            local root = character:FindFirstChild("HumanoidRootPart")

            if root then
                local localPosition = destination.CFrame:PointToObjectSpace(root.Position)
                local localZ = localPosition.Z
                local visible = entry.VisibleInPortal

                if visible == nil then
                    visible = localZ >= 0
                end

                if visible then
                    if localZ < CHARACTER_HIDE_Z then
                        visible = false
                    end
                else
                    if localZ > CHARACTER_SHOW_Z then
                        visible = true
                    end
                end

                entry.VisibleInPortal = visible

                local targetParent

                if visible then
                    targetParent = data.World
                else
                    targetParent = data.HiddenCharacters
                end

                if clone.Parent ~= targetParent then
                    clone.Parent = targetParent
                end
            end
        end
    end
end
```
