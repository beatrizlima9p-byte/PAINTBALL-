-- ServerScriptService
-- Cria/usa RemoteEvent "ShootPaintballEvent" em ReplicatedStorage.
-- Valida no servidor: cooldown, existência da ferramenta, escolhe alvo visível, cria projétil e trata colisão.

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Workspace = game:GetService("Workspace")
local Debris = game:GetService("Debris")

local EVENT_NAME = "ShootPaintballEvent"
local COOLDOWN = 0.6 -- segundos entre tiros por jogador
local MAX_TARGET_DISTANCE = 600 -- máxima distância para procurar alvo
local PAINTBALL_SPEED = 200 -- studs por segundo
local PAINTBALL_LIFETIME = 5 -- segundos

-- garante RemoteEvent existente
local shootEvent = ReplicatedStorage:FindFirstChild(EVENT_NAME)
if not shootEvent then
shootEvent = Instance.new("RemoteEvent")
shootEvent.Name = EVENT_NAME
shootEvent.Parent = ReplicatedStorage
end

local lastShotTime = {} -- [player.UserId] = os.clock()

-- Função servidor para achar alvo mais próximo VISÍVEL
local function GetClosestVisibleTarget(player, maxDistance)
maxDistance = maxDistance or math.huge
if not player.Character then return nil end
local origin = player.Character:FindFirstChild("HumanoidRootPart")
if not origin then return nil end

local closest = nil  
local shortest = maxDistance  

local rayParams = RaycastParams.new()  
rayParams.FilterDescendantsInstances = {player.Character}  
rayParams.FilterType = Enum.RaycastFilterType.Blacklist  
rayParams.IgnoreWater = true  

for _, plr in pairs(Players:GetPlayers()) do  
    if plr ~= player and plr.Character then  
        local humanoid = plr.Character:FindFirstChildWhichIsA("Humanoid")  
        if humanoid and humanoid.Health > 0 then  
            local targetRoot = plr.Character:FindFirstChild("HumanoidRootPart") or plr.Character:FindFirstChild("Head")  
            if targetRoot then  
                local dir = targetRoot.Position - origin.Position  
                local dist = dir.Magnitude  
                if dist < shortest and dist <= maxDistance then  
                    -- raycast para checar linha de visão  
                    local result = Workspace:Raycast(origin.Position, dir, rayParams)  
                    if not result then  
                        -- sem bloqueio => visível  
                        shortest = dist  
                        closest = plr.Character  
                    else  
                        -- se o resultado for parte do personagem alvo, ainda consideramos visível  
                        if result.Instance and result.Instance:IsDescendantOf(plr.Character) then  
                            shortest = dist  
                            closest = plr.Character  
                        end  
                    end  
                end  
            end  
        end  
    end  
end  

return closest

end

local function spawnPaintball(originCFrame, targetPos, ownerPlayer, targetCharacter)
local paintball = Instance.new("Part")
paintball.Shape = Enum.PartType.Ball
paintball.Material = Enum.Material.Neon
paintball.BrickColor = BrickColor.new("Bright red")
paintball.Size = Vector3.new(0.5, 0.5, 0.5)
paintball.CFrame = originCFrame
paintball.CanCollide = false
paintball.Anchored = false
paintball.Name = "Paintball"
paintball.Parent = Workspace
paintball:SetAttribute("OwnerUserId", ownerPlayer.UserId)

local bv = Instance.new("BodyVelocity")  
bv.MaxForce = Vector3.new(1e5, 1e5, 1e5)  
local dir = (targetPos - originCFrame.Position)  
if dir.Magnitude < 0.001 then  
    bv.Velocity = originCFrame.LookVector * PAINTBALL_SPEED  
else  
    bv.Velocity = dir.Unit * PAINTBALL_SPEED  
end  
bv.Parent = paintball  

local hitDebounce = {}  
local conn  
conn = paintball.Touched:Connect(function(hit)  
    if not hit or not hit.Parent then return end  
    -- evita tocar no próprio jogador que atirou  
    if hit:IsDescendantOf(ownerPlayer.Character) then return end  
    -- reduzir falsos positivos: requer que seja parte do targetCharacter  
    if targetCharacter and hit:IsDescendantOf(targetCharacter) then  
        if not hitDebounce[targetCharacter] then  
            hitDebounce[targetCharacter] = true  
            -- efeito simples: destruir o projétil e opcionalmente aplicar um marcador visual no alvo  
            if paintball and paintball.Parent then  
                paintball:Destroy()  
            end  
            -- Exemplo de efeito seguro: seta uma tag temporária no humanoid (sem dano)  
            local hum = targetCharacter:FindFirstChildWhichIsA("Humanoid")  
            if hum then  
                -- Exemplo: colocar uma cor de efeito ou evento; aqui apenas log  
                -- print(("Player %s hit %s"):format(ownerPlayer.Name, targetCharacter.Name))  
            end  
        end  
    else  
        -- colisão com mundo: destruir o projétil  
        if not hit:IsDescendantOf(ownerPlayer.Character) then  
            if paintball and paintball.Parent then  
                paintball:Destroy()  
            end  
        end  
    end  
end)  

Debris:AddItem(paintball, PAINTBALL_LIFETIME)  
-- cleanup do evento quando a bolinha for destruída  
spawn(function()  
    wait(PAINTBALL_LIFETIME + 0.1)  
    if conn and conn.Connected then  
        conn:Disconnect()  
    end  
end)

end

shootEvent.OnServerEvent:Connect(function(player)
-- validações básicas
if not player or not player.Character then return end
local now = os.clock()
local uid = player.UserId
if lastShotTime[uid] and now - lastShotTime[uid] < COOLDOWN then
-- em cooldown, ignora
return
end

-- encontra ferramenta do jogador e a posição de origem segura  
local tool = player.Character:FindFirstChildOfClass("Tool")  
if not tool then return end  
local handle = tool:FindFirstChild("Handle")  
if not handle or not handle:IsA("BasePart") then return end  

-- escolhe alvo visível no servidor  
local target = GetClosestVisibleTarget(player, MAX_TARGET_DISTANCE)  
if not target then  
    lastShotTime[uid] = now  
    return  
end  

local targetPart = target:FindFirstChild("HumanoidRootPart") or target:FindFirstChild("Head")  
if not targetPart then  
    lastShotTime[uid] = now  
    return  
end  

-- Origem do projétil: cerca de frente da handle (pode ajustar)  
local originCFrame = handle.CFrame  
local targetPos = targetPart.Position  

-- Spawn do projétil no servidor (servidor controla física e colisões)  
spawnPaintball(originCFrame, targetPos, player, target)  

-- atualiza cooldown  
lastShotTime[uid] = now

-- LocalScript (StarterPlayerScripts)
-- Painel MG HUB - interface gráfica com botões simulados

local Players = game:GetService("Players")
local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "MG_HUB_ScreenGui"
screenGui.ResetOnSpawn = false
screenGui.Parent = playerGui

local frame = Instance.new("Frame")
frame.Name = "MainFrame"
frame.Size = UDim2.new(0, 360, 0, 180)
frame.Position = UDim2.new(0.5, -180, 0.15, 0)
frame.AnchorPoint = Vector2.new(0.5, 0)
frame.BackgroundColor3 = Color3.fromRGB(18, 18, 18)
frame.BorderSizePixel = 0
frame.Active = true
frame.Draggable = true
frame.Parent = screenGui

local stroke = Instance.new("UIStroke")
stroke.Color = Color3.fromRGB(0, 122, 255)
stroke.Thickness = 3
stroke.Parent = frame

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, -40, 0, 36)
title.Position = UDim2.new(0, 12, 0, 6)
title.BackgroundTransparency = 1
title.Text = "MG HUB"
title.Font = Enum.Font.GothamBold
title.TextSize = 20
title.TextColor3 = Color3.fromRGB(255, 255, 255)
title.TextXAlignment = Enum.TextXAlignment.Left
title.Parent = frame

local minBtn = Instance.new("TextButton")
minBtn.Size = UDim2.new(0, 24, 0, 24)
minBtn.Position = UDim2.new(1, -36, 0, 6)
minBtn.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
minBtn.Text = "-"
minBtn.Font = Enum.Font.GothamBold
minBtn.TextSize = 18
minBtn.TextColor3 = Color3.fromRGB(255,255,255)
minBtn.BorderSizePixel = 0
minBtn.Parent = frame

local content = Instance.new("Frame")
content.Size = UDim2.new(1, -16, 1, -52)
content.Position = UDim2.new(0, 8, 0, 44)
content.BackgroundTransparency = 1
content.Parent = frame

-- Função utilitária para criar botões
local function createButton(txt, posY)
	local b = Instance.new("TextButton")
	b.Size = UDim2.new(1, 0, 0, 40)
	b.Position = UDim2.new(0, 0, 0, posY)
	b.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
	b.Text = txt
	b.Font = Enum.Font.GothamBold
	b.TextSize = 18
	b.TextColor3 = Color3.fromRGB(255, 255, 255)
	b.BorderSizePixel = 0

	local stroke2 = Instance.new("UIStroke")
	stroke2.Color = Color3.fromRGB(0, 122, 255)
	stroke2.Thickness = 2
	stroke2.Parent = b

	b.Parent = content
	return b
end

local autoBtn = createButton("AUTO TARGET: OFF", 0)
local fireBtn = createButton("ATIRAR", 50)

-- Minimizar função
local minimized = false

minBtn.MouseButton1Click:Connect(function()
	minimized = not minimized
	content.Visible = not minimized
	minBtn.Text = minimized and "+" or "-"
end)

-- Toggle auto target - apenas visual, sem funcionalidade real
autoBtn.MouseButton1Click:Connect(function()
	if autoBtn.Text == "AUTO TARGET: OFF" then
		autoBtn.Text = "AUTO TARGET: ON"
		autoBtn.BackgroundColor3 = Color3.fromRGB(0, 122, 255)
	else
		autoBtn.Text = "AUTO TARGET: OFF"
		autoBtn.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
	end
end)

-- Botão atirar - só muda visual, sem disparo real
fireBtn.MouseButton1Click:Connect(function()
	fireBtn.BackgroundColor3 = Color3.fromRGB(0, 200, 0)
	wait(0.1)
	fireBtn.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
end)
