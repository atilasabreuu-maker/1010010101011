--[[
================================================================================
PRYPOT-0X1 — VERSAO CORRIGIDA / RECONSTRUIDA
Roblox LocalScript | colocar em StarterPlayer > StarterPlayerScripts

OBJETIVO DESTA VERSAO
- Corrigir a causa mais provavel de a armadura desaparecer: as pecas antigas eram
  criadas sem ancoragem e sem juntas enquanto o modelo ficava estacionado.
- Deixar todas as pecas ancoradas e visiveis enquanto a PRYPOT esta estacionada.
- Ao entrar, reposicionar a estrutura junto ao avatar e soldar cada peca a uma
  parte correspondente do personagem. Ao sair, remover as soldas e ancorar tudo
  novamente no local em que ficou.
- Recriar o modelo para nao reutilizar um modelo antigo incompleto.
- Dar suporte a controles touch e teclado, movimento arcade e escavacao visual
  local do Terrain, com tentativa de seguranca contra o vazio e reparo temporizado.

NOTAS IMPORTANTES
- Este LocalScript altera apenas a visao local. Outros jogadores nao recebem
  replicacao confiavel das mudancas do Terrain feitas aqui.
- O soco contra pecas e visual; nao destrui objetos reais do mapa.
- A seguranca contra o vazio e uma protecao de melhor esforco, nao uma garantia
  contra todo mapa personalizado, streaming ou geometria complexa.
- Nao foi possivel executar/testar o script dentro do Roblox Studio neste ambiente.
  Use o Output para verificar erros de API ou particularidades do seu mapa.
================================================================================
]]

-- [01] SERVICOS E CONFIGURACAO -------------------------------------------------
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local Debris = game:GetService("Debris")
local Workspace = game:GetService("Workspace")
local Terrain = Workspace.Terrain

local player = Players.LocalPlayer
if not player then
    Players.PlayerAdded:Wait()
    player = Players.LocalPlayer
end
local playerGui = player:WaitForChild("PlayerGui")

local CONFIG = {
    ModelName = "PryPotArmor_0X1",
    DisplayName = "PRYPOT-0X1",
    SpawnSideOffset = 7,
    PromptDistance = 12,
    WalkSpeed = 22,
    SprintMultiplier = 1.8,
    JumpPower = 72,
    DoubleJumpVelocity = 72,
    DashVelocity = 86,
    DashCooldown = 0.75,
    DrillRadius = 4.1,
    DrillInterval = 0.24,
    DrillPush = 24,
    DrillRayDistance = 9,
    VoidRayDistance = 120,
    SafetyPlatformLifetime = 11,
    TerrainRepairDelay = 7,
    TerrainRepairInterval = 0.5,
    MaxSavedTerrainCells = 100,
    WallCheckDistance = 3.8,
    ClimbVelocity = 19,
    MaxEnergy = 100,
    SprintDrain = 13,
    DrillDrain = 21,
    ClimbDrain = 17,
    EnergyRegen = 17,
    EnergyRegenDelay = 0.8,
    Debug = true,
}

local C = {
    Dark = Color3.fromRGB(20, 25, 25),
    Metal = Color3.fromRGB(45, 52, 51),
    Olive = Color3.fromRGB(59, 67, 47),
    Bone = Color3.fromRGB(198, 196, 173),
    Gear = Color3.fromRGB(91, 99, 95),
    Cyan = Color3.fromRGB(60, 245, 232),
    Red = Color3.fromRGB(235, 45, 50),
    Yellow = Color3.fromRGB(235, 194, 45),
}

local function log(...)
    if CONFIG.Debug then print("[PRYPOT-0X1 FIX]", ...) end
end
local function warnLog(...)
    warn("[PRYPOT-0X1 FIX]", ...)
end

-- Remove telas antigas de outra tentativa do sistema.
for _, child in ipairs(playerGui:GetChildren()) do
    if child.Name == "PryPotArmorGui" then child:Destroy() end
end

-- [02] ESTADO ---------------------------------------------------------------
local State = {
    Model = nil,
    Parts = {},
    HomeOffsets = {},
    Prompt = nil,
    Worn = false,
    Stealth = false,
    Sprinting = false,
    Drilling = false,
    Climbing = false,
    CeilingGrabbing = false,
    WallRunning = false,
    WallRunUntil = 0,
    LastWallRun = 0,
    Sliding = false,
    InCutscene = false,
    MaxFallSpeed = 0,
    Energy = CONFIG.MaxEnergy,
    LastEnergyUse = 0,
    LastDrill = 0,
    LastDash = 0,
    AirJumps = 0,
    LastGrounded = true,
    Character = nil,
    Humanoid = nil,
    Root = nil,
    OriginalWalkSpeed = nil,
    OriginalJumpPower = nil,
    OriginalUseJumpPower = nil,
    OriginalTransparency = {},
    Welds = {},
    RepairQueue = {},
    SafetyPlatforms = {},
    InitialPlacementDone = false,
    Connections = {},
    Gui = {},
    LastPunch = 0,
    ArmorIntegrity = 100,
}

local function track(connection)
    table.insert(State.Connections, connection)
    return connection
end
local function disconnectAll()
    for _, connection in ipairs(State.Connections) do
        pcall(function() connection:Disconnect() end)
    end
    table.clear(State.Connections)
end

-- [03] CONSTRUTOR VISUAL ----------------------------------------------------
local function makePart(model, name, size, offset, color, material, slot, shape)
    local part = Instance.new("Part")
    part.Name = name
    part.Size = size
    part.Color = color or C.Metal
    part.Material = material or Enum.Material.Metal
    part.TopSurface = Enum.SurfaceType.Smooth
    part.BottomSurface = Enum.SurfaceType.Smooth
    part.CanCollide = false
    part.CanTouch = false
    part.CanQuery = true
    part.CastShadow = true
    part.Massless = true
    part.Anchored = true -- FIX PRINCIPAL: estacionada, nenhuma peca cai/espalha.
    part.CFrame = offset
    if shape then part.Shape = shape end
    part:SetAttribute("PryPotSlot", slot or "UpperTorso")
    part.Parent = model
    State.Parts[name] = part
    return part
end

local function makeMesh(model, name, size, offset, color, slot, meshType, meshScale)
    local part = makePart(model, name, size, offset, color, Enum.Material.Metal, slot)
    local mesh = Instance.new("SpecialMesh")
    mesh.MeshType = meshType or Enum.MeshType.Sphere
    mesh.Scale = meshScale or Vector3.new(1, 1, 1)
    mesh.Parent = part
    return part
end

local function addLight(part, color, brightness, range)
    local light = Instance.new("PointLight")
    light.Color = color
    light.Brightness = brightness or 1.5
    light.Range = range or 9
    light.Shadows = false
    light.Parent = part
    return light
end

local function makeGear(model, name, position, radius, slot)
    local disc = makePart(model, name, Vector3.new(0.18, radius * 2, radius * 2),
        CFrame.new(position) * CFrame.Angles(0, 0, math.rad(90)), C.Gear, Enum.Material.Metal, slot)
    disc.Shape = Enum.PartType.Cylinder
    makePart(model, name .. "Hub", Vector3.new(0.22, radius * 0.45, radius * 0.45),
        CFrame.new(position) * CFrame.Angles(0, 0, math.rad(90)), C.Cyan, Enum.Material.Neon, slot,
        Enum.PartType.Cylinder)
    for i = 1, 6 do
        local a = math.pi * 2 * i / 6
        local toothPos = Vector3.new(position.X, position.Y + math.sin(a) * radius * 0.82,
            position.Z + math.cos(a) * radius * 0.82)
        makePart(model, name .. "Tooth" .. i, Vector3.new(0.24, 0.13, 0.22), CFrame.new(toothPos), C.Olive,
            Enum.Material.Metal, slot)
    end
    return disc
end

local function buildArmor()
    State.Parts = {}
    local old = Workspace:FindFirstChild(CONFIG.ModelName)
    if old then old:Destroy() end -- nao herda modelo quebrado de versao anterior.

    local model = Instance.new("Model")
    model.Name = CONFIG.ModelName

    -- O pivô representa o centro do tronco. Tudo e criado em offsets relativos a ele.
    local core = makePart(model, "CoreTorso", Vector3.new(1.6, 1.8, 0.75), CFrame.new(0, 0, 0), C.Dark,
        Enum.Material.Metal, "UpperTorso")
    model.PrimaryPart = core

    -- Placas e costelas do peito.
    makePart(model, "ChestPlateL", Vector3.new(0.76, 1.16, 0.34), CFrame.new(-0.49, 0.14, -0.48) * CFrame.Angles(0, 0, -0.16), C.Olive, Enum.Material.Metal, "UpperTorso")
    makePart(model, "ChestPlateR", Vector3.new(0.76, 1.16, 0.34), CFrame.new(0.49, 0.14, -0.48) * CFrame.Angles(0, 0, 0.16), C.Olive, Enum.Material.Metal, "UpperTorso")
    makePart(model, "Sternum", Vector3.new(0.23, 1.25, 0.22), CFrame.new(0, 0.05, -0.68), C.Bone, Enum.Material.Metal, "UpperTorso")
    local coreGlow = makePart(model, "CoreGlow", Vector3.new(0.48, 0.56, 0.18), CFrame.new(0, 0.22, -0.72), C.Cyan, Enum.Material.Neon, "UpperTorso")
    coreGlow.Shape = Enum.PartType.Ball
    addLight(coreGlow, C.Cyan, 2.2, 12)
    for i = 1, 5 do
        local y = 0.53 - i * 0.23
        makePart(model, "RibL" .. i, Vector3.new(0.78, 0.13, 0.18), CFrame.new(-0.4, y, -0.4) * CFrame.Angles(0, 0, -0.28), C.Bone, Enum.Material.Metal, "UpperTorso")
        makePart(model, "RibR" .. i, Vector3.new(0.78, 0.13, 0.18), CFrame.new(0.4, y, -0.4) * CFrame.Angles(0, 0, 0.28), C.Bone, Enum.Material.Metal, "UpperTorso")
    end
    makePart(model, "BeltFrame", Vector3.new(1.65, 0.35, 0.92), CFrame.new(0, -1.03, 0), C.Gear, Enum.Material.Metal, "LowerTorso")
    makePart(model, "PelvisShell", Vector3.new(1.54, 0.58, 0.86), CFrame.new(0, -1.28, 0), C.Olive, Enum.Material.Metal, "LowerTorso")
    makePart(model, "SpineRail", Vector3.new(0.26, 2.55, 0.25), CFrame.new(0, -0.35, 0.5), C.Gear, Enum.Material.Metal, "UpperTorso")
    for i = 1, 7 do
        local y = 0.68 - i * 0.34
        makePart(model, "Vertebra" .. i, Vector3.new(0.58, 0.2, 0.36), CFrame.new(0, y, 0.57), i % 2 == 0 and C.Cyan or C.Bone, Enum.Material.Metal, "UpperTorso")
    end

    -- Cabeca animatronica, sensores e mandibula.
    makePart(model, "HelmetSkull", Vector3.new(1.18, 1.12, 1.05), CFrame.new(0, 1.58, 0), C.Dark, Enum.Material.Metal, "Head")
    makePart(model, "HelmetBrow", Vector3.new(1.25, 0.2, 0.28), CFrame.new(0, 1.91, -0.48), C.Olive, Enum.Material.Metal, "Head")
    makePart(model, "JawFrame", Vector3.new(0.86, 0.27, 0.48), CFrame.new(0, 1.1, -0.38), C.Gear, Enum.Material.Metal, "Head")
    makePart(model, "JawL", Vector3.new(0.13, 0.34, 0.16), CFrame.new(-0.34, 1.18, -0.56), C.Bone, Enum.Material.Metal, "Head")
    makePart(model, "JawR", Vector3.new(0.13, 0.34, 0.16), CFrame.new(0.34, 1.18, -0.56), C.Bone, Enum.Material.Metal, "Head")
    makePart(model, "EyeBar", Vector3.new(0.92, 0.16, 0.13), CFrame.new(0, 1.61, -0.55), C.Red, Enum.Material.Neon, "Head")
    local eyeLight = addLight(State.Parts.EyeBar, C.Red, 2, 10)
    makePart(model, "HeadSensorStem", Vector3.new(0.18, 0.37, 0.18), CFrame.new(0, 2.17, 0), C.Gear, Enum.Material.Metal, "Head")
    makeMesh(model, "HeadSensor", Vector3.new(0.32, 0.32, 0.32), CFrame.new(0, 2.38, -0.02), C.Cyan, "Head", Enum.MeshType.Sphere, Vector3.new(1, 0.75, 1))
    makePart(model, "AntennaL", Vector3.new(0.09, 0.52, 0.09), CFrame.new(-0.43, 2.13, 0.12) * CFrame.Angles(0, 0, -0.25), C.Olive, Enum.Material.Metal, "Head")
    makePart(model, "AntennaR", Vector3.new(0.09, 0.52, 0.09), CFrame.new(0.43, 2.13, 0.12) * CFrame.Angles(0, 0, 0.25), C.Olive, Enum.Material.Metal, "Head")

    -- Ombros, bracos, juntas, maos e garras.
    for _, sideData in ipairs({
        {name = "Left", sign = -1}, {name = "Right", sign = 1},
    }) do
        local n, s = sideData.name, sideData.sign
        local shoulder = Vector3.new(s * 1.13, 0.55, 0)
        local elbow = Vector3.new(s * 1.35, -0.55, 0)
        local wrist = Vector3.new(s * 1.46, -1.42, -0.02)
        makePart(model, n .. "ShoulderSocket", Vector3.new(0.65, 0.66, 0.66), CFrame.new(shoulder), C.Gear, Enum.Material.Metal, n .. "UpperArm", Enum.PartType.Ball)
        makePart(model, n .. "UpperArm", Vector3.new(0.55, 1.0, 0.58), CFrame.new(s * 1.28, 0.02, 0), C.Olive, Enum.Material.Metal, n .. "UpperArm")
        makePart(model, n .. "BicepRail", Vector3.new(0.16, 0.84, 0.16), CFrame.new(s * 1.31, 0.03, -0.32), C.Bone, Enum.Material.Metal, n .. "UpperArm")
        makeGear(model, n .. "ElbowGear", elbow, 0.28, n .. "LowerArm")
        makePart(model, n .. "LowerArm", Vector3.new(0.49, 0.98, 0.54), CFrame.new(s * 1.4, -1.0, 0), C.Dark, Enum.Material.Metal, n .. "LowerArm")
        makePart(model, n .. "ForearmShell", Vector3.new(0.58, 0.52, 0.65), CFrame.new(s * 1.4, -1.13, -0.1), C.Olive, Enum.Material.Metal, n .. "LowerArm")
        makePart(model, n .. "DrillBlade", Vector3.new(0.13, 0.72, 0.16), CFrame.new(s * 1.43, -1.55, -0.4), C.Cyan, Enum.Material.Neon, n .. "LowerArm")
        makePart(model, n .. "WristJoint", Vector3.new(0.38, 0.28, 0.38), CFrame.new(wrist), C.Gear, Enum.Material.Metal, n .. "Hand", Enum.PartType.Ball)
        makePart(model, n .. "Palm", Vector3.new(0.55, 0.48, 0.31), CFrame.new(s * 1.46, -1.8, -0.1), C.Dark, Enum.Material.Metal, n .. "Hand")
        for finger = 1, 3 do
            local fx = s * 1.46 + (finger - 2) * 0.17
            makePart(model, n .. "Claw" .. finger, Vector3.new(0.11, 0.38, 0.13), CFrame.new(fx, -2.14, -0.25) * CFrame.Angles(math.rad(-18), 0, (finger - 2) * 0.1), C.Bone, Enum.Material.Metal, n .. "Hand")
        end
        makePart(model, n .. "ThumbClaw", Vector3.new(0.13, 0.34, 0.13), CFrame.new(s * 1.76, -1.94, -0.22) * CFrame.Angles(0, 0, s * -0.6), C.Cyan, Enum.Material.Metal, n .. "Hand")
    end

    -- Pernas e molas de absorcao.
    for _, sideData in ipairs({ {name="Left", sign=-1}, {name="Right", sign=1} }) do
        local n, s = sideData.name, sideData.sign
        makePart(model, n .. "HipJoint", Vector3.new(0.47, 0.48, 0.48), CFrame.new(s * 0.48, -1.5, 0), C.Gear, Enum.Material.Metal, n .. "UpperLeg", Enum.PartType.Ball)
        makePart(model, n .. "UpperLeg", Vector3.new(0.63, 1.18, 0.68), CFrame.new(s * 0.54, -2.02, 0), C.Olive, Enum.Material.Metal, n .. "UpperLeg")
        makeGear(model, n .. "KneeGear", Vector3.new(s * 0.56, -2.62, -0.06), 0.25, n .. "LowerLeg")
        makePart(model, n .. "LowerLeg", Vector3.new(0.54, 1.05, 0.57), CFrame.new(s * 0.58, -3.13, 0), C.Dark, Enum.Material.Metal, n .. "LowerLeg")
        makePart(model, n .. "ShinRail", Vector3.new(0.16, 0.92, 0.15), CFrame.new(s * 0.58, -3.14, -0.31), C.Bone, Enum.Material.Metal, n .. "LowerLeg")
        for ring = 1, 5 do
            makePart(model, n .. "SpringRing" .. ring, Vector3.new(0.58, 0.09, 0.58), CFrame.new(s * 0.58, -3.78 - ring * 0.11, 0.02) * CFrame.Angles(0, ring * 0.25, 0), ring % 2 == 0 and C.Cyan or C.Gear, Enum.Material.Metal, n .. "Foot", Enum.PartType.Cylinder)
        end
        makePart(model, n .. "ShockPad", Vector3.new(0.62, 0.18, 0.76), CFrame.new(s * 0.58, -4.42, 0.02), C.Gear, Enum.Material.Metal, n .. "Foot")
        makePart(model, n .. "Foot", Vector3.new(0.75, 0.28, 1.05), CFrame.new(s * 0.58, -4.62, -0.1), C.Dark, Enum.Material.Metal, n .. "Foot")
        makePart(model, n .. "ToePlate", Vector3.new(0.72, 0.15, 0.42), CFrame.new(s * 0.58, -4.55, -0.49), C.Olive, Enum.Material.Metal, n .. "Foot")
    end

    -- Armadura lateral/dorsal, con marcas de advertencia.
    makePart(model, "BackpackFrame", Vector3.new(1.18, 1.65, 0.48), CFrame.new(0, -0.05, 0.77), C.Dark, Enum.Material.Metal, "UpperTorso")
    makePart(model, "NeuralSpineBase", Vector3.new(0.48, 0.48, 0.35), CFrame.new(0, 0.65, 0.92), C.Red, Enum.Material.Neon, "UpperTorso")
    addLight(State.Parts.NeuralSpineBase, C.Red, 1.4, 7)
    for i = 1, 4 do
        makePart(model, "BackFin" .. i, Vector3.new(0.16, 0.72, 0.34), CFrame.new((i - 2.5) * 0.24, 0.55 - i * 0.28, 0.98), C.Olive, Enum.Material.Metal, "UpperTorso")
    end
    makePart(model, "WarningMarkL", Vector3.new(0.12, 0.42, 0.08), CFrame.new(-0.69, 0.56, -0.67), C.Yellow, Enum.Material.Neon, "UpperTorso")
    makePart(model, "WarningMarkR", Vector3.new(0.12, 0.42, 0.08), CFrame.new(0.69, 0.56, -0.67), C.Yellow, Enum.Material.Neon, "UpperTorso")

    model.Parent = Workspace
    State.Model = model
    State.Parts = State.Parts
    State.HomeOffsets = {}
    for _, part in ipairs(model:GetDescendants()) do
        if part:IsA("BasePart") then
            State.HomeOffsets[part] = core.CFrame:ToObjectSpace(part.CFrame)
            part.Anchored = true
            part.CanCollide = false
            part.Massless = true
        end
    end

    local prompt = Instance.new("ProximityPrompt")
    prompt.Name = "WearPrompt"
    prompt.ActionText = "Entrar"
    prompt.ObjectText = CONFIG.DisplayName
    prompt.HoldDuration = 0.35
    prompt.MaxActivationDistance = CONFIG.PromptDistance
    prompt.RequiresLineOfSight = false
    prompt.Parent = core
    State.Prompt = prompt
    log("Modelo montado; pecas ancoradas enquanto estacionado:", #model:GetDescendants())
end

-- [04] POSICIONAMENTO E SOLDAGEM --------------------------------------------
local RIG_SLOTS = {
    Head = {"Head"}, UpperTorso = {"UpperTorso", "Torso"}, LowerTorso = {"LowerTorso", "Torso"},
    LeftUpperArm = {"LeftUpperArm", "Left Arm"}, LeftLowerArm = {"LeftLowerArm", "Left Arm"}, LeftHand = {"LeftHand", "Left Arm"},
    RightUpperArm = {"RightUpperArm", "Right Arm"}, RightLowerArm = {"RightLowerArm", "Right Arm"}, RightHand = {"RightHand", "Right Arm"},
    LeftUpperLeg = {"LeftUpperLeg", "Left Leg"}, LeftLowerLeg = {"LeftLowerLeg", "Left Leg"}, LeftFoot = {"LeftFoot", "Left Leg"},
    RightUpperLeg = {"RightUpperLeg", "Right Leg"}, RightLowerLeg = {"RightLowerLeg", "Right Leg"}, RightFoot = {"RightFoot", "Right Leg"},
}

local function getCharacterPart(slot)
    if not State.Character then return nil end
    local names = RIG_SLOTS[slot] or {"UpperTorso", "Torso", "HumanoidRootPart"}
    for _, name in ipairs(names) do
        local part = State.Character:FindFirstChild(name)
        if part and part:IsA("BasePart") then return part end
    end
    return State.Root
end

local function raycast(origin, direction, ignore)
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
    params.FilterDescendantsInstances = ignore or {}
    params.IgnoreWater = true
    return Workspace:Raycast(origin, direction, params)
end

local function setCharacterHidden(hidden)
    if not State.Character then return end
    if hidden then
        table.clear(State.OriginalTransparency)
        for _, desc in ipairs(State.Character:GetDescendants()) do
            if desc:IsA("BasePart") then
                State.OriginalTransparency[desc] = desc.LocalTransparencyModifier
                desc.LocalTransparencyModifier = 1
            elseif desc:IsA("Decal") or desc:IsA("Texture") then
                State.OriginalTransparency[desc] = desc.Transparency
                desc.Transparency = 1
            end
        end
    else
        for inst, value in pairs(State.OriginalTransparency) do
            if inst and inst.Parent then
                pcall(function()
                    if inst:IsA("BasePart") then inst.LocalTransparencyModifier = value else inst.Transparency = value end
                end)
            end
        end
        table.clear(State.OriginalTransparency)
    end
end

local function setArmorAnchored(anchored)
    if not State.Model then return end
    for _, part in ipairs(State.Model:GetDescendants()) do
        if part:IsA("BasePart") then
            part.Anchored = anchored
            part.CanCollide = false
            part.Massless = true
        end
    end
end

local function clearWelds()
    for _, weld in ipairs(State.Welds) do
        if weld and weld.Parent then weld:Destroy() end
    end
    table.clear(State.Welds)
end

local function findGroundPosition(origin, ignore)
    local hit = raycast(origin + Vector3.new(0, 8, 0), Vector3.new(0, -160, 0), ignore)
    return hit and hit.Position or origin
end

local function placeParkedArmorNearCharacter()
    if not State.Model then return end
    local character = player.Character
    local root = character and character:FindFirstChild("HumanoidRootPart")
    local target
    local facing = CFrame.new()
    if root then
        local right = root.CFrame.RightVector
        local wanted = root.Position + right * CONFIG.SpawnSideOffset
        local ground = findGroundPosition(wanted, {character, State.Model})
        target = Vector3.new(wanted.X, ground.Y + 4.75, wanted.Z)
        local look = Vector3.new(root.CFrame.LookVector.X, 0, root.CFrame.LookVector.Z)
        if look.Magnitude > 0.01 then facing = CFrame.lookAt(target, target + look.Unit) else facing = CFrame.new(target) end
        State.InitialPlacementDone = true
    else
        target = Vector3.new(0, 7, 0)
        facing = CFrame.new(target)
    end
    State.Model:PivotTo(facing)
    setArmorAnchored(true)
end

local function applyMovementStats(worn)
    if not State.Humanoid then return end
    if worn then
        State.OriginalWalkSpeed = State.Humanoid.WalkSpeed
        State.OriginalJumpPower = State.Humanoid.JumpPower
        State.OriginalUseJumpPower = State.Humanoid.UseJumpPower
        State.Humanoid.UseJumpPower = true
        State.Humanoid.WalkSpeed = CONFIG.WalkSpeed
        State.Humanoid.JumpPower = CONFIG.JumpPower
    else
        if State.OriginalWalkSpeed then State.Humanoid.WalkSpeed = State.OriginalWalkSpeed end
        if State.OriginalJumpPower then State.Humanoid.JumpPower = State.OriginalJumpPower end
        if State.OriginalUseJumpPower ~= nil then State.Humanoid.UseJumpPower = State.OriginalUseJumpPower end
    end
end

local function playEntryCutscene()
    -- Cutscene curta e local: nao requer assets nem altera o rig do personagem.
    local camera = Workspace.CurrentCamera
    local character, root, humanoid = State.Character, State.Root, State.Humanoid
    if not camera or not root or not humanoid then return end
    State.InCutscene = true
    local oldType, oldSubject = camera.CameraType, camera.CameraSubject
    local startPosition = root.Position
    camera.CameraType = Enum.CameraType.Scriptable
    camera.CFrame = CFrame.lookAt(startPosition + root.CFrame.LookVector * 5 + Vector3.new(0, 1.8, 0), startPosition + Vector3.new(0, 0.7, 0))
    local target = CFrame.lookAt(startPosition + root.CFrame.RightVector * 3.8 + Vector3.new(0, 1.2, 4.6), startPosition + Vector3.new(0, 0.7, 0))
    local tween = TweenService:Create(camera, TweenInfo.new(0.85, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {CFrame = target})
    tween:Play()
    task.wait(1.25)
    if Workspace.CurrentCamera == camera then
        camera.CameraType = oldType
        camera.CameraSubject = (humanoid and humanoid.Parent) and humanoid or oldSubject
    end
    State.InCutscene = false
end

local function updateGui()
    local gui = State.Gui
    if gui.Mode then
        gui.Mode.Text = State.Worn and (State.Stealth and "PRYPOT // STEALTH" or (State.Drilling and "PRYPOT // PERFURANDO" or "PRYPOT // ONLINE")) or "PRYPOT // ESTACIONADA"
    end
    if gui.EnergyFill then gui.EnergyFill.Size = UDim2.fromScale(math.clamp(State.Energy / CONFIG.MaxEnergy, 0, 1), 1) end
    if gui.Enter then gui.Enter.Text = State.Worn and "SAIR DA PRYPOT" or "ENTRAR NA PRYPOT" end
    for _, name in ipairs({"Jump", "Sprint", "Dash", "Drill", "Climb", "Slide", "Punch", "Stealth"}) do
        local button = gui[name]
        if button then button.Visible = State.Worn end
    end
end

local function enterArmor()
    if State.Worn or not State.Model then return end
    local character = player.Character
    local humanoid = character and character:FindFirstChildOfClass("Humanoid")
    local root = character and character:FindFirstChild("HumanoidRootPart")
    if not character or not humanoid or not root or humanoid.Health <= 0 then
        warnLog("Nao foi possivel entrar: personagem ainda nao esta pronto.")
        return
    end

    State.Character, State.Humanoid, State.Root = character, humanoid, root
    -- Move o veiculo montado para a posicao atual do avatar antes de conectar as pecas.
    State.Model:PivotTo(root.CFrame * CFrame.new(0, 0.25, 0))
    setArmorAnchored(false)
    clearWelds()

    for _, part in ipairs(State.Model:GetDescendants()) do
        if part:IsA("BasePart") then
            local slot = part:GetAttribute("PryPotSlot") or "UpperTorso"
            local bodyPart = getCharacterPart(slot) or root
            local weld = Instance.new("WeldConstraint")
            weld.Name = "PryPotAttach_" .. part.Name
            weld.Part0 = bodyPart
            weld.Part1 = part
            weld.Parent = part
            table.insert(State.Welds, weld)
        end
    end

    State.Worn = true
    State.InCutscene = false
    State.Stealth = false
    State.Drilling = false
    State.Climbing = false
    State.Sprinting = false
    State.AirJumps = 0
    State.Energy = CONFIG.MaxEnergy
    if State.Prompt then State.Prompt.Enabled = false end
    setCharacterHidden(true)
    applyMovementStats(true)
    updateGui()
    task.spawn(playEntryCutscene)
    log("Entrou na armadura; pecas soldadas ao personagem.")
end

local function exitArmor(reason)
    if not State.Worn then return end
    local model = State.Model
    local root = State.Root
    local humanoid = State.Humanoid
    local exitCFrame
    if model and model.PrimaryPart then
        exitCFrame = model.PrimaryPart.CFrame * CFrame.new(3.5, 0, 0)
    elseif root then
        exitCFrame = root.CFrame * CFrame.new(3.5, 0, 0)
    end

    State.Worn = false
    State.Drilling = false
    State.Climbing = false
    State.Sprinting = false
    State.Stealth = false
    State.InCutscene = false
    State.CeilingGrabbing = false
    State.WallRunning = false
    State.Sliding = false
    clearWelds()
    setArmorAnchored(true) -- FIX: a armadura estacionada continua inteira no Workspace.
    setCharacterHidden(false)
    applyMovementStats(false)
    if root and exitCFrame then
        root.CFrame = exitCFrame
        root.AssemblyLinearVelocity = Vector3.zero
    end
    if State.Prompt and State.Prompt.Parent then State.Prompt.Enabled = true end
    if humanoid then
        humanoid:ChangeState(Enum.HumanoidStateType.GettingUp)
    end
    updateGui()
    log("Saiu da armadura; modelo permaneceu estacionado.", reason or "manual")
end

-- [05] INTERFACE TOUCH / HUD -------------------------------------------------
local function roundify(instance, radius)
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, radius or 10)
    corner.Parent = instance
end
local function addStroke(instance, color, thickness)
    local stroke = Instance.new("UIStroke")
    stroke.Color = color or C.Cyan
    stroke.Thickness = thickness or 1.5
    stroke.Transparency = 0.15
    stroke.Parent = instance
end
local function makeButton(parent, name, text, position, size, color)
    local button = Instance.new("TextButton")
    button.Name = name
    button.Text = text
    button.Font = Enum.Font.GothamBold
    button.TextScaled = true
    button.TextColor3 = Color3.fromRGB(242, 245, 236)
    button.BackgroundColor3 = color or Color3.fromRGB(21, 30, 30)
    button.BackgroundTransparency = 0.12
    button.Position = position
    button.Size = size
    button.AnchorPoint = Vector2.new(0.5, 0.5)
    button.AutoButtonColor = true
    button.ZIndex = 10
    button.Parent = parent
    roundify(button, 12)
    addStroke(button, C.Cyan, 1.3)
    return button
end

local function makeGui()
    local gui = Instance.new("ScreenGui")
    gui.Name = "PryPotArmorGui"
    gui.ResetOnSpawn = false
    gui.IgnoreGuiInset = true
    gui.DisplayOrder = 40
    gui.Parent = playerGui

    local panel = Instance.new("Frame")
    panel.Name = "StatusPanel"
    panel.Position = UDim2.new(0.015, 0, 0.025, 0)
    panel.Size = UDim2.new(0.29, 0, 0.115, 0)
    panel.BackgroundColor3 = Color3.fromRGB(11, 16, 17)
    panel.BackgroundTransparency = 0.2
    panel.Parent = gui
    roundify(panel, 8)
    addStroke(panel, C.Cyan, 1.2)

    local mode = Instance.new("TextLabel")
    mode.Name = "Mode"
    mode.BackgroundTransparency = 1
    mode.Position = UDim2.new(0.04, 0, 0.05, 0)
    mode.Size = UDim2.new(0.92, 0, 0.35, 0)
    mode.Font = Enum.Font.GothamBold
    mode.Text = "PRYPOT // ESTACIONADA"
    mode.TextColor3 = C.Cyan
    mode.TextScaled = true
    mode.TextXAlignment = Enum.TextXAlignment.Left
    mode.Parent = panel

    local energyBack = Instance.new("Frame")
    energyBack.Name = "EnergyBack"
    energyBack.Position = UDim2.new(0.04, 0, 0.53, 0)
    energyBack.Size = UDim2.new(0.92, 0, 0.24, 0)
    energyBack.BackgroundColor3 = Color3.fromRGB(48, 53, 51)
    energyBack.BorderSizePixel = 0
    energyBack.Parent = panel
    roundify(energyBack, 5)
    local energyFill = Instance.new("Frame")
    energyFill.Name = "EnergyFill"
    energyFill.Size = UDim2.fromScale(1, 1)
    energyFill.BackgroundColor3 = C.Cyan
    energyFill.BorderSizePixel = 0
    energyFill.Parent = energyBack
    roundify(energyFill, 5)

    local enter = makeButton(gui, "EnterExit", "ENTRAR NA PRYPOT", UDim2.new(0.5, 0, 0.88, 0), UDim2.new(0.25, 0, 0.065, 0), Color3.fromRGB(24, 42, 40))
    local controls = {
        Jump = makeButton(gui, "Jump", "PULO", UDim2.new(0.91, 0, 0.42, 0), UDim2.new(0.105, 0, 0.075, 0), Color3.fromRGB(28, 45, 44)),
        Sprint = makeButton(gui, "Sprint", "CORRER", UDim2.new(0.91, 0, 0.51, 0), UDim2.new(0.105, 0, 0.075, 0), Color3.fromRGB(28, 45, 44)),
        Dash = makeButton(gui, "Dash", "DASH", UDim2.new(0.91, 0, 0.60, 0), UDim2.new(0.105, 0, 0.075, 0), Color3.fromRGB(28, 45, 44)),
        Drill = makeButton(gui, "Drill", "CAVAR", UDim2.new(0.79, 0, 0.69, 0), UDim2.new(0.105, 0, 0.075, 0), Color3.fromRGB(49, 36, 20)),
        Climb = makeButton(gui, "Climb", "AGARRAR", UDim2.new(0.91, 0, 0.69, 0), UDim2.new(0.105, 0, 0.075, 0), Color3.fromRGB(28, 45, 44)),
        Slide = makeButton(gui, "Slide", "DESLIZAR", UDim2.new(0.79, 0, 0.51, 0), UDim2.new(0.105, 0, 0.075, 0), Color3.fromRGB(28, 45, 44)),
        Punch = makeButton(gui, "Punch", "SOCO", UDim2.new(0.79, 0, 0.78, 0), UDim2.new(0.105, 0, 0.075, 0), Color3.fromRGB(49, 25, 28)),
        Stealth = makeButton(gui, "Stealth", "STEALTH", UDim2.new(0.91, 0, 0.78, 0), UDim2.new(0.105, 0, 0.075, 0), Color3.fromRGB(25, 27, 43)),
    }
    State.Gui = {Screen = gui, Panel = panel, Mode = mode, EnergyFill = energyFill, Enter = enter}
    for name, button in pairs(controls) do State.Gui[name] = button end
    updateGui()
    return controls
end

-- [06] ENERGIA, SALTOS, DASH E WALL-CLIMB -----------------------------------
local function useEnergy(amount)
    if State.Energy < amount then return false end
    State.Energy = math.max(0, State.Energy - amount)
    State.LastEnergyUse = os.clock()
    return true
end

local function jump()
    if not State.Worn or not State.Humanoid or not State.Root then return end
    local humanoidState = State.Humanoid:GetState()
    if humanoidState == Enum.HumanoidStateType.Running or humanoidState == Enum.HumanoidStateType.Landed or humanoidState == Enum.HumanoidStateType.RunningNoPhysics then
        State.Humanoid.Jump = true
        State.AirJumps = 0
    elseif State.AirJumps < 1 and useEnergy(8) then
        State.AirJumps += 1
        local v = State.Root.AssemblyLinearVelocity
        State.Root.AssemblyLinearVelocity = Vector3.new(v.X, CONFIG.DoubleJumpVelocity, v.Z)
    else
        return
    end
    updateGui()
end

local function dash()
    if not State.Worn or not State.Root or os.clock() - State.LastDash < CONFIG.DashCooldown or not useEnergy(12) then return end
    State.LastDash = os.clock()
    local direction = State.Humanoid and State.Humanoid.MoveDirection or Vector3.zero
    direction = Vector3.new(direction.X, 0, direction.Z)
    if direction.Magnitude < 0.1 then
        local look = State.Root.CFrame.LookVector
        direction = Vector3.new(look.X, 0, look.Z)
    end
    if direction.Magnitude > 0.1 then
        local v = State.Root.AssemblyLinearVelocity
        State.Root.AssemblyLinearVelocity = Vector3.new(direction.Unit.X * CONFIG.DashVelocity, math.max(v.Y, 8), direction.Unit.Z * CONFIG.DashVelocity)
    end
    updateGui()
end

local function startSprint()
    if not State.Worn or State.Sprinting or not useEnergy(1) then return end
    State.Sprinting = true
    if State.Humanoid then State.Humanoid.WalkSpeed = CONFIG.WalkSpeed * CONFIG.SprintMultiplier end
    updateGui()
end
local function stopSprint()
    State.Sprinting = false
    if State.Worn and State.Humanoid then State.Humanoid.WalkSpeed = CONFIG.WalkSpeed end
    updateGui()
end

local function isWallAhead()
    if not State.Root then return nil end
    local origin = State.Root.Position
    return raycast(origin, State.Root.CFrame.LookVector * CONFIG.WallCheckDistance, {State.Character, State.Model})
end
local function startClimb()
    if not State.Worn or not State.Root or not useEnergy(3) then return end
    local ceiling = raycast(State.Root.Position, Vector3.new(0, CONFIG.WallCheckDistance, 0), {State.Character, State.Model})
    if ceiling then
        State.CeilingGrabbing = true
        State.Climbing = false
    elseif isWallAhead() then
        State.Climbing = true
        State.CeilingGrabbing = false
    else
        return
    end
    updateGui()
end
local function stopClimb()
    local wasCeiling = State.CeilingGrabbing
    State.Climbing = false
    State.CeilingGrabbing = false
    if wasCeiling and State.Root then
        local velocity = State.Root.AssemblyLinearVelocity
        State.Root.AssemblyLinearVelocity = Vector3.new(velocity.X, 42, velocity.Z)
    end
    updateGui()
end

-- [07] EFEITOS VISUAIS DE IMPACTO / SOCO ------------------------------------
local function makeSparks(position, color, amount)
    local anchor = Instance.new("Part")
    anchor.Name = "PryPotLocalFX"
    anchor.Anchored = true
    anchor.CanCollide = false
    anchor.CanTouch = false
    anchor.CanQuery = false
    anchor.Transparency = 1
    anchor.Size = Vector3.new(0.2, 0.2, 0.2)
    anchor.CFrame = CFrame.new(position)
    anchor.Parent = Workspace
    local emitter = Instance.new("ParticleEmitter")
    emitter.Texture = "rbxasset://textures/particles/sparkles_main.dds"
    emitter.Color = ColorSequence.new(color or C.Cyan)
    emitter.Lifetime = NumberRange.new(0.18, 0.55)
    emitter.Speed = NumberRange.new(4, 12)
    emitter.SpreadAngle = Vector2.new(160, 160)
    emitter.Rate = 0
    emitter.LightEmission = 0.8
    emitter.Parent = anchor
    emitter:Emit(amount or 16)
    Debris:AddItem(anchor, 1.5)
end

local function punch()
    if not State.Worn or not State.Root or os.clock() - State.LastPunch < 0.55 then return end
    State.LastPunch = os.clock()
    local origin = State.Root.Position + Vector3.new(0, 0.25, 0)
    local direction = State.Root.CFrame.LookVector * 5
    local hit = raycast(origin, direction, {State.Character, State.Model})
    if hit then makeSparks(hit.Position, C.Yellow, 24) end
    -- A acao e deliberadamente cosmetica: nao danifica pecas reais de outros sistemas.
end

local function toggleStealth()
    if not State.Worn then return end
    State.Stealth = not State.Stealth
    for _, part in ipairs(State.Model:GetDescendants()) do
        if part:IsA("BasePart") then
            part.LocalTransparencyModifier = State.Stealth and 0.45 or 0
            local light = part:FindFirstChildOfClass("PointLight")
            if light then light.Enabled = not State.Stealth end
        end
    end
    updateGui()
end

-- [08] ESCAVACAO DE TERRAIN LOCAL + RESTAURO --------------------------------
local function getIgnoreList()
    local list = {}
    if State.Character then table.insert(list, State.Character) end
    if State.Model then table.insert(list, State.Model) end
    return list
end

local function snapRegion(point, radius)
    local grid = 4
    local minV = Vector3.new(
        math.floor((point.X - radius) / grid) * grid,
        math.floor((point.Y - radius) / grid) * grid,
        math.floor((point.Z - radius) / grid) * grid)
    local maxV = Vector3.new(
        math.ceil((point.X + radius) / grid) * grid,
        math.ceil((point.Y + radius) / grid) * grid,
        math.ceil((point.Z + radius) / grid) * grid)
    return Region3.new(minV, maxV):ExpandToGrid(grid)
end

local function saveTerrainRegion(center)
    local region = snapRegion(center, CONFIG.DrillRadius + 2)
    local ok, materials, occupancies = pcall(function()
        return Terrain:ReadVoxels(region, 4)
    end)
    if not ok then
        warnLog("Nao foi possivel guardar voxels para restauracao:", materials)
        return nil
    end
    local entry = {Region = region, Materials = materials, Occupancies = occupancies, Timestamp = os.clock(), Center = center}
    table.insert(State.RepairQueue, entry)
    while #State.RepairQueue > CONFIG.MaxSavedTerrainCells do table.remove(State.RepairQueue, 1) end
    return entry
end

local function createSafetyPlatform(point)
    local plate = Instance.new("Part")
    plate.Name = "PryPotLocalSafetyPlatform"
    plate.Size = Vector3.new(CONFIG.DrillRadius * 2.5, 1, CONFIG.DrillRadius * 2.5)
    plate.CFrame = CFrame.new(point - Vector3.new(0, 6.5, 0))
    plate.Anchored = true
    plate.CanCollide = true
    plate.CanTouch = false
    plate.Material = Enum.Material.ForceField
    plate.Color = C.Cyan
    plate.Transparency = 0.42
    plate.Parent = Workspace
    table.insert(State.SafetyPlatforms, plate)
    Debris:AddItem(plate, CONFIG.SafetyPlatformLifetime)
end

local function groundSafe(point)
    local hit = raycast(point + Vector3.new(0, 2, 0), Vector3.new(0, -CONFIG.VoidRayDistance, 0), getIgnoreList())
    return hit ~= nil
end

local function carveTerrain(center)
    -- So remove Terrain se os voxels originais puderem ser guardados para restauracao.
    local saved = saveTerrainRegion(center)
    if not saved then
        warnLog("Escavacao cancelada para nao deixar dano sem reparo.")
        return false
    end
    local ok, err = pcall(function()
        Terrain:FillBall(center, CONFIG.DrillRadius, Enum.Material.Air)
    end)
    if not ok then
        warnLog("FillBall falhou:", err)
        return false
    end
    if not groundSafe(center) then createSafetyPlatform(center) end
    makeSparks(center, C.Cyan, 10)
    return true
end

local function processRepairQueue()
    local now = os.clock()
    local index = 1
    while index <= #State.RepairQueue do
        local entry = State.RepairQueue[index]
        local stillInside = State.Root and (State.Root.Position - entry.Center).Magnitude < CONFIG.DrillRadius + 3
        if now - entry.Timestamp >= CONFIG.TerrainRepairDelay and not stillInside then
            local ok, err = pcall(function()
                Terrain:WriteVoxels(entry.Region, 4, entry.Materials, entry.Occupancies)
            end)
            if not ok then warnLog("WriteVoxels falhou ao restaurar tunel:", err) end
            table.remove(State.RepairQueue, index)
        else
            index += 1
        end
    end
end

local function toggleDrill()
    if not State.Worn then return end
    State.Drilling = not State.Drilling
    if State.Drilling then State.Climbing = false end
    updateGui()
end

local function updateDrill()
    if not State.Worn or not State.Drilling or not State.Root or not State.Humanoid then return end
    if os.clock() - State.LastDrill < CONFIG.DrillInterval then return end
    State.LastDrill = os.clock()
    if not useEnergy(CONFIG.DrillDrain * CONFIG.DrillInterval) then
        State.Drilling = false
        updateGui()
        return
    end
    local moveDir = State.Humanoid.MoveDirection
    local forward = Vector3.new(moveDir.X, 0, moveDir.Z)
    if forward.Magnitude < 0.1 then
        local look = State.Root.CFrame.LookVector
        forward = Vector3.new(look.X, 0, look.Z)
    end
    if forward.Magnitude < 0.1 then return end
    forward = forward.Unit
    local origin = State.Root.Position + Vector3.new(0, 0.2, 0)
    local hit = raycast(origin, forward * CONFIG.DrillRayDistance, getIgnoreList())
    if hit and hit.Instance == Terrain then
        local center = hit.Position + forward * 1.8
        carveTerrain(center)
        local vel = State.Root.AssemblyLinearVelocity
        State.Root.AssemblyLinearVelocity = Vector3.new(forward.X * CONFIG.DrillPush, vel.Y, forward.Z * CONFIG.DrillPush)
    elseif hit and hit.Instance:IsA("BasePart") then
        -- No Terrain? Mantem efeito visual e nao tenta alterar paredes de pecas.
        makeSparks(hit.Position, C.Yellow, 8)
    else
        -- Sem obstaculo: deixa o impulso arcade continuar de forma limitada.
        local vel = State.Root.AssemblyLinearVelocity
        State.Root.AssemblyLinearVelocity = Vector3.new(forward.X * (CONFIG.DrillPush * 0.65), vel.Y, forward.Z * (CONFIG.DrillPush * 0.65))
    end
end

-- [09] BOTAO ENTRAR, CONTROLES E EVENTOS ------------------------------------
local function nearArmor()
    if State.Worn then return true end
    local root = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
    local pivot = State.Model and State.Model.PrimaryPart
    return root and pivot and (root.Position - pivot.Position).Magnitude <= CONFIG.PromptDistance + 2 or false
end

local function handleEnterButton()
    if State.Worn then exitArmor("botao") elseif nearArmor() then enterArmor() end
end

local function slide()
    if not State.Worn or not State.Root or State.Sliding then return end
    local move = State.Humanoid and State.Humanoid.MoveDirection or Vector3.zero
    local flat = Vector3.new(move.X, 0, move.Z)
    if flat.Magnitude < 0.1 then
        local look = State.Root.CFrame.LookVector
        flat = Vector3.new(look.X, 0, look.Z)
    end
    if flat.Magnitude > 0.1 and useEnergy(7) then
        State.Sliding = true
        local v = State.Root.AssemblyLinearVelocity
        State.Root.AssemblyLinearVelocity = Vector3.new(flat.Unit.X * 42, math.max(v.Y, 0), flat.Unit.Z * 42)
        task.delay(0.65, function() State.Sliding = false end)
    end
end

local controls = makeGui()
controls.Enter.Activated:Connect(handleEnterButton)
controls.Jump.Activated:Connect(jump)
controls.Dash.Activated:Connect(dash)
controls.Drill.Activated:Connect(toggleDrill)
controls.Slide.Activated:Connect(slide)
controls.Punch.Activated:Connect(punch)
controls.Stealth.Activated:Connect(toggleStealth)

-- Botoes de segurar continuam funcionando por toque e mouse.
controls.Sprint.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then startSprint() end
end)
controls.Sprint.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then stopSprint() end
end)
controls.Climb.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then startClimb() end
end)
controls.Climb.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then stopClimb() end
end)

track(UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.KeyCode == Enum.KeyCode.E then
        if State.Worn then exitArmor("tecla E") elseif nearArmor() then enterArmor() end
    elseif State.Worn then
        if input.KeyCode == Enum.KeyCode.Space then jump()
        elseif input.KeyCode == Enum.KeyCode.LeftShift then startSprint()
        elseif input.KeyCode == Enum.KeyCode.Q then punch()
        elseif input.KeyCode == Enum.KeyCode.F then toggleDrill()
        elseif input.KeyCode == Enum.KeyCode.G then startClimb()
        elseif input.KeyCode == Enum.KeyCode.C then toggleStealth()
        elseif input.KeyCode == Enum.KeyCode.X then dash()
        elseif input.KeyCode == Enum.KeyCode.LeftControl then slide() end
    end
end))
track(UserInputService.InputEnded:Connect(function(input, processed)
    if input.KeyCode == Enum.KeyCode.LeftShift then stopSprint() end
    if input.KeyCode == Enum.KeyCode.G then stopClimb() end
end))

local function bindPrompt()
    if State.Prompt then
        track(State.Prompt.Triggered:Connect(function(triggeringPlayer)
            if triggeringPlayer == player and not State.Worn then enterArmor() end
        end))
    end
end

-- [10] RESPAWN E ESTADO DO HUMANOID -----------------------------------------
local function unmountForRespawn()
    if State.Worn then
        -- O personagem pode estar sendo removido; apenas estaciona a armadura e limpa referencias.
        State.Worn = false
        State.Drilling = false
        State.Climbing = false
        State.Sprinting = false
        clearWelds()
        setArmorAnchored(true)
        setCharacterHidden(false)
        if State.Prompt and State.Prompt.Parent then State.Prompt.Enabled = true end
        updateGui()
    end
    State.Character, State.Humanoid, State.Root = nil, nil, nil
end

local function onCharacterAdded(character)
    State.Character = character
    State.Humanoid = character:WaitForChild("Humanoid", 10)
    State.Root = character:WaitForChild("HumanoidRootPart", 10)
    if State.Worn then exitArmor("respawn") end
    task.wait(0.2)
    if State.Model and not State.Worn and not State.InitialPlacementDone then placeParkedArmorNearCharacter() end
    updateGui()
    log("Personagem detectado.")
end

track(player.CharacterRemoving:Connect(unmountForRespawn))
track(player.CharacterAdded:Connect(onCharacterAdded))
if player.Character then task.spawn(onCharacterAdded, player.Character) end

-- [11] ANIMACAO IDLE E CICLO PRINCIPAL --------------------------------------
local idleClock = 0
local promptPulse = 0
track(RunService.Heartbeat:Connect(function(dt)
    if State.Worn then
        if State.Sprinting then
            if not useEnergy(CONFIG.SprintDrain * dt) then stopSprint() end
        end
        if State.CeilingGrabbing and State.Root then
            local ceiling = raycast(State.Root.Position, Vector3.new(0, CONFIG.WallCheckDistance, 0), {State.Character, State.Model})
            if not ceiling or not useEnergy(CONFIG.ClimbDrain * dt) then
                State.CeilingGrabbing = false
            else
                local move = State.Humanoid and State.Humanoid.MoveDirection or Vector3.zero
                State.Root.AssemblyLinearVelocity = Vector3.new(move.X * 8, 0, move.Z * 8)
            end
        end
        if State.Climbing and State.Root then
            if not useEnergy(CONFIG.ClimbDrain * dt) then
                stopClimb()
            else
                local wall = isWallAhead()
                if not wall then
                    stopClimb()
                else
                    local v = State.Root.AssemblyLinearVelocity
                    State.Root.AssemblyLinearVelocity = Vector3.new(v.X * 0.5, CONFIG.ClimbVelocity, v.Z * 0.5)
                end
            end
        end
        updateDrill()
        -- Wall-run automatico: so inicia no ar, com deslocamento e uma parede lateral proxima.
        local hState = State.Humanoid and State.Humanoid:GetState()
        local airborne = hState == Enum.HumanoidStateType.Freefall or hState == Enum.HumanoidStateType.Jumping
        if airborne and not State.WallRunning and not State.Climbing and not State.CeilingGrabbing and State.Humanoid and State.Humanoid.MoveDirection.Magnitude > 0.5 and os.clock() - State.LastWallRun > 0.35 then
            local right = State.Root.CFrame.RightVector
            local wall = raycast(State.Root.Position, right * CONFIG.WallCheckDistance, {State.Character, State.Model})
                or raycast(State.Root.Position, -right * CONFIG.WallCheckDistance, {State.Character, State.Model})
            if wall and useEnergy(2) then
                State.WallRunning = true
                State.WallRunUntil = os.clock() + 1.45
                State.LastWallRun = os.clock()
            end
        end
        if State.WallRunning and State.Root then
            local right = State.Root.CFrame.RightVector
            local wall = raycast(State.Root.Position, right * CONFIG.WallCheckDistance, {State.Character, State.Model})
                or raycast(State.Root.Position, -right * CONFIG.WallCheckDistance, {State.Character, State.Model})
            if not airborne or not wall or os.clock() >= State.WallRunUntil or not useEnergy(7 * dt) then
                State.WallRunning = false
            else
                local tangent = Vector3.new(-wall.Normal.Z, 0, wall.Normal.X)
                local move = State.Humanoid.MoveDirection
                if tangent.Magnitude > 0.1 then
                    if tangent:Dot(move) < 0 then tangent = -tangent end
                    State.Root.AssemblyLinearVelocity = Vector3.new(tangent.Unit.X * 25, 3, tangent.Unit.Z * 25)
                end
            end
        end
        if State.Root and State.Root.AssemblyLinearVelocity.Y < 0 then
            State.MaxFallSpeed = math.max(State.MaxFallSpeed, -State.Root.AssemblyLinearVelocity.Y)
        elseif hState == Enum.HumanoidStateType.Landed then
            if State.MaxFallSpeed > 28 then makeSparks(State.Root.Position - Vector3.new(0, 2.2, 0), C.Cyan, 20) end
            State.MaxFallSpeed = 0
        end
        if hState == Enum.HumanoidStateType.Landed or hState == Enum.HumanoidStateType.Running then
            State.AirJumps = 0
        end
    else
        -- Animacao muito leve de sensor e pulso enquanto o traje esta estacionado.
        idleClock += dt
        local sensor = State.Parts.HeadSensor
        local sensorStem = State.Parts.HeadSensorStem
        if sensor and sensor.Parent and State.Model and State.Model.PrimaryPart then
            local pivot = State.Model.PrimaryPart.CFrame
            local base = State.HomeOffsets[sensor]
            if base then
                sensor.CFrame = pivot * base * CFrame.Angles(0, math.sin(idleClock * 0.7) * 0.18, 0)
            end
        end
        local core = State.Parts.CoreGlow
        if core then
            local light = core:FindFirstChildOfClass("PointLight")
            if light then light.Brightness = 1.8 + math.sin(idleClock * 3) * 0.5 end
        end
        if sensorStem and sensorStem.Parent then
            -- O haste permanece firme; o sensor superior faz a varredura.
        end
    end

    if State.Energy < CONFIG.MaxEnergy and os.clock() - State.LastEnergyUse >= CONFIG.EnergyRegenDelay and not State.Sprinting and not State.Drilling and not State.Climbing then
        State.Energy = math.min(CONFIG.MaxEnergy, State.Energy + CONFIG.EnergyRegen * dt)
    end
    promptPulse += dt
    if promptPulse >= 0.18 then
        promptPulse = 0
        if State.Gui.Enter then
            State.Gui.Enter.Visible = State.Worn or nearArmor()
        end
        updateGui()
    end
end))

track(RunService.Heartbeat:Connect(function()
    if os.clock() - (State.LastRepairCheck or 0) >= CONFIG.TerrainRepairInterval then
        State.LastRepairCheck = os.clock()
        processRepairQueue()
    end
end))

-- [12] INICIALIZACAO COM VALIDACAO REAL -------------------------------------
local function initialize()
    log("Iniciando construcao do modelo...")
    buildArmor()
    if not State.Model or not State.Model.Parent or not State.Model.PrimaryPart then
        error("O construtor nao produziu um Model valido com PrimaryPart.")
    end
    local partCount = 0
    for _, descendant in ipairs(State.Model:GetDescendants()) do
        if descendant:IsA("BasePart") then
            partCount += 1
            if not descendant.Anchored then
                error("Peca estacionada sem ancoragem: " .. descendant:GetFullName())
            end
        end
    end
    if partCount < 35 then error("Modelo incompleto; pecas encontradas: " .. tostring(partCount)) end
    bindPrompt()
    placeParkedArmorNearCharacter()
    updateGui()
    log("Inicializacao concluida. Pecas:", partCount, " | modelo:", State.Model:GetFullName())
    log("A armadura deve estar visivel ao lado do personagem. Aproxima-te e toca em ENTRAR.")
end

local ok, err = xpcall(initialize, debug.traceback)
if not ok then
    warnLog("FALHA DE INICIALIZACAO:\n", err)
    if State.Gui.Mode then State.Gui.Mode.Text = "PRYPOT // ERRO - VER OUTPUT" end
else
    print("[PRYPOT-0X1 FIX] SISTEMA PRONTO. Se nao vires o traje, procura outra mensagem [PRYPOT-0X1 FIX] no Output.")
end

-- Fim do LocalScript.
