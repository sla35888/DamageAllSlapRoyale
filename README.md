-- Prevenção de execução múltipla no mesmo ambiente de execução
if _G.DamageAllBusExecuted then
    local StarterGui = game:GetService("StarterGui")
    StarterGui:SetCore("SendNotification", {
        Title = "Already Activated",
        Text = "This script has already been executed in this match.",
        Duration = 7
    })
    return
end

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Workspace = game:GetService("Workspace")
local VirtualInputManager = game:GetService("VirtualInputManager")
local StarterGui = game:GetService("StarterGui")
local Player = Players.LocalPlayer

-- Função para localizar itens no Workspace com buscas fallback
local function GetItem(itemName)
    local f1 = Workspace:FindFirstChild("Item")
    local f2 = Workspace:FindFirstChild("Items")
    if f1 and f1:FindFirstChild(itemName) then return f1[itemName] end
    if f2 and f2:FindFirstChild(itemName) then return f2[itemName] end
    return Workspace:FindFirstChild(itemName, true)
end

-- Simulação de pressionamento de tecla
local function PressF()
    VirtualInputManager:SendKeyEvent(true, Enum.KeyCode.F, false, game)
    task.wait(0.05)
    VirtualInputManager:SendKeyEvent(false, Enum.KeyCode.F, false, game)
end

-- Simulação de clique de mouse no centro da tela
local function Click()
    local vp = Workspace.CurrentCamera.ViewportSize
    VirtualInputManager:SendMouseButtonEvent(vp.X / 2, vp.Y / 2, 0, true, game, 0)
    task.wait(0.05)
    VirtualInputManager:SendMouseButtonEvent(vp.X / 2, vp.Y / 2, 0, false, game, 0)
end

-- Equipar ferramentas pelo nome
local function Equip(itemName)
    local char = Player.Character
    if char then
        local tool = Player.Backpack:FindFirstChild(itemName) or char:FindFirstChild(itemName)
        if tool then 
            char:FindFirstChildOfClass("Humanoid"):EquipTool(tool) 
        end
    end
end

-- Verificação inicial do item obrigatório
local tp = GetItem("True Power")
if not tp then
    StarterGui:SetCore("SendNotification", {
        Title = "Execution Cancelled",
        Text = "Required rare items were not found in this match.",
        Duration = 7
    })
    return
end

-- Marca o script como executado após validar a presença do item principal
_G.DamageAllBusExecuted = true

StarterGui:SetCore("SendNotification", {
    Title = "Bus Sequence Started",
    Text = "Gathering required items and preparing bus attack...",
    Duration = 5
})

local hrp = Player.Character and Player.Character:FindFirstChild("HumanoidRootPart")
if not hrp then return end

-- Coleta do True Power
hrp.CFrame = tp:GetPivot()
task.wait(0.25)
PressF()
task.wait(0.25)

-- Coleta do Forcefield Crystal (se existente)
local ff = GetItem("Forcefield Crystal")
if ff then
    hrp.CFrame = ff:GetPivot()
    task.wait(0.25)
    PressF()
    task.wait(0.25)
end

-- Coleta de apenas 1 Bomba
local b1 = GetItem("Bomb")
if b1 then
    hrp.CFrame = b1:GetPivot()
    task.wait(0.25)
    PressF()
    task.wait(0.25)
end

-- Espera pela existência do ônibus no Workspace
local bus = Workspace:WaitForChild("BusModel", 60)
if bus then
    -- Aguarda exatamente 1.35 segundos após a detecção do ônibus
    task.wait(1.35)
    
    -- Sequência de uso dos itens no ônibus
    Equip("Forcefield Crystal")
    task.wait(0.25)
    Click()
    task.wait(0.25)
    
    Equip("Bomb")
    task.wait(0.25)
    Click()
    task.wait(0.25)
    
    -- Disparo do salto do ônibus logo após o uso dos itens
    local busJumpRemote = ReplicatedStorage:FindFirstChild("Remotes") and ReplicatedStorage.Remotes:FindFirstChild("BusJumping")
    if busJumpRemote then 
        busJumpRemote:FireServer() 
    end
    
    -- Espera exatamente 3 segundos após se soltar do ônibus
    task.wait(3)
    
    -- Detecção e teleporte posicionando o jogador 15 studs acima do chão do Terrain
    local currentCharacter = Player.Character
    if currentCharacter and currentCharacter:FindFirstChild("HumanoidRootPart") then
        local currentHrp = currentCharacter.HumanoidRootPart
        
        -- Raycast configurado com Include para detectar exclusivamente o Terrain do Workspace
        local raycastParams = RaycastParams.new()
        raycastParams.FilterType = Enum.RaycastFilterType.Include
        raycastParams.FilterDescendantsInstances = {Workspace.Terrain}
        raycastParams.IgnoreWater = false
        
        -- Projeta o raio vertical de 3000 studs para baixo
        local origin = currentHrp.Position
        local direction = Vector3.new(0, -3000, 0)
        local raycastResult = Workspace:Raycast(origin, direction, raycastParams)
        
        if raycastResult then
            -- Posiciona a raiz do personagem 15 studs acima da superfície para evitar detecção
            local targetPosition = raycastResult.Position + Vector3.new(0, 15, 0)
            currentHrp.CFrame = CFrame.new(targetPosition)
        end
    end
    
    -- Espera 2 segundos no ar
    task.wait(2)
    
    -- Ativa o True Power
    Equip("True Power")
    task.wait(0.25)
    Click()
    
    -- Espera 1 segundo após ativar o True Power
    task.wait(1)
    
    -- Seleção do jogador mais próximo do ônibus
    local targetPlayer = nil
    local shortestDistance = 999
    
    for _, p in pairs(Players:GetPlayers()) do
        if p ~= Player and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
            local dist = (p.Character.HumanoidRootPart.Position - bus:GetPivot().Position).Magnitude
            if dist < shortestDistance then 
                shortestDistance = dist
                targetPlayer = p 
            end
        end
    end
    
    -- Teleporte, equipar o Pow e atacar o alvo
    if targetPlayer and targetPlayer.Character:FindFirstChild("HumanoidRootPart") then
        local targetHrp = targetPlayer.Character.HumanoidRootPart
        
        -- Teleporta para o jogador escolhido
        hrp.CFrame = targetHrp.CFrame
        
        -- Espera 1 segundo após o teleporte
        task.wait(1)
        
        -- Equipa a ferramenta Pow
        Equip("Pow")
        task.wait(0.15)
        
        -- Trava temporária do personagem para garantir o acerto
        hrp.Anchored = true
        task.wait(0.30)
        
        -- Executa o Slap no jogador alvo
        local slapRemote = ReplicatedStorage:FindFirstChild("Remotes") and ReplicatedStorage.Remotes:FindFirstChild("Slap")
        if slapRemote then
            slapRemote:FireServer(targetHrp)
        end
        
        hrp.Anchored = false
    end
    
    StarterGui:SetCore("SendNotification", {
        Title = "Sequence Finished",
        Text = "Bus damage automation completed successfully.",
        Duration = 5
    })
end
