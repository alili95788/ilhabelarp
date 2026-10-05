--========================================================--
-- ILHA BYLA RP
-- FARM
-- AUTO LIXO
--========================================================--

local repo =
    "https://raw.githubusercontent.com/deividcomsono/Obsidian/main/"

local Library =
    loadstring(
        game:HttpGet(
            repo .. "Library.lua"
        )
    )()

local ThemeManager =
    loadstring(
        game:HttpGet(
            repo .. "addons/ThemeManager.lua"
        )
    )()

local SaveManager =
    loadstring(
        game:HttpGet(
            repo .. "addons/SaveManager.lua"
        )
    )()

--========================================================--
-- SERVICES
--========================================================--

local Players =
    game:GetService("Players")

local TweenService =
    game:GetService("TweenService")

local UserInputService =
    game:GetService("UserInputService")

local Player =
    Players.LocalPlayer

--========================================================--
-- WINDOW
--========================================================--

local Window =
    Library:CreateWindow({
        Title = "ILHA BYLA RP",
        Center = true,
        AutoShow = true,
    })

--========================================================--
-- TAB
--========================================================--

local Tabs = {
    Farm = Window:AddTab("Farm"),
}

local FarmGroup =
    Tabs.Farm:AddLeftGroupbox(
        "Coleta de Lixo"
    )

--========================================================--
-- CONFIG
--========================================================--

local FarmSettings = {

    Enabled = false,

    TweenSpeed = 25,

    ArrivalDistance = 2.5,

    PromptDistance = 12,

    TruckSearchRadius = 25,

    PromptSearchTime = 4,

    PromptRetryDelay = 0.1,

}

local Busy = false

local CurrentTween = nil

local CollectedLixos = {}

--========================================================--
-- CHARACTER
--========================================================--

local function GetCharacter()

    local Character =
        Player.Character

    if not Character then

        Character =
            Player.CharacterAdded:Wait()

    end

    local Humanoid =
        Character:FindFirstChildOfClass(
            "Humanoid"
        )

    local Root =
        Character:FindFirstChild(
            "HumanoidRootPart"
        )

    if not Humanoid or not Root then
        return nil, nil, nil
    end

    return Character, Humanoid, Root

end

--========================================================--
-- ANTI-SEAT
--========================================================--

local function SetupAntiSeat(
    Character
)

    local Humanoid =
        Character:WaitForChild(
            "Humanoid"
        )

    Humanoid.Sit = false

    Humanoid:SetStateEnabled(
        Enum.HumanoidStateType.Seated,
        false
    )

    Humanoid.StateChanged:Connect(
        function(
            _,
            NewState
        )

            if NewState ==
                Enum.HumanoidStateType.Seated then

                Humanoid.Sit = false

                Humanoid:ChangeState(
                    Enum.HumanoidStateType.GettingUp
                )

            end

        end
    )

end

if Player.Character then

    SetupAntiSeat(
        Player.Character
    )

end

Player.CharacterAdded:Connect(
    function(
        Character
    )

        SetupAntiSeat(
            Character
        )

    end
)

--========================================================--
-- POSITION
--========================================================--

local function GetPosition(
    Object
)

    if not Object
        or not Object.Parent then

        return nil

    end

    if Object:IsA("BasePart") then

        return Object.Position

    end

    if Object:IsA("Attachment") then

        return Object.WorldPosition

    end

    if Object:IsA("Model") then

        return Object:GetPivot().Position

    end

    local Part =
        Object:FindFirstChildWhichIsA(
            "BasePart",
            true
        )

    if Part then

        return Part.Position

    end

    return nil

end

--========================================================--
-- HORIZONTAL DISTANCE
--========================================================--

local function HorizontalDistance(
    A,
    B
)

    return (
        Vector3.new(
            A.X,
            0,
            A.Z
        )
        -
        Vector3.new(
            B.X,
            0,
            B.Z
        )
    ).Magnitude

end

--========================================================--
-- CANCEL TWEEN
--========================================================--

local function CancelTween()

    if CurrentTween then

        pcall(
            function()

                CurrentTween:Cancel()

            end
        )

        CurrentTween = nil

    end

end

--========================================================--
-- CFRAME TWEEN
--
-- SOMENTE X/Z
-- Y NÃO SOBE
-- Y NÃO DESCE
--========================================================--

local function CFrameTweenHorizontal(
    TargetPosition
)

    local Character,
        Humanoid,
        Root =
        GetCharacter()

    if not Character
        or not Humanoid
        or not Root then

        return false

    end

    if not FarmSettings.Enabled then
        return false
    end

    CancelTween()

    local FixedY =
        Root.Position.Y

    local FinalPosition =
        Vector3.new(
            TargetPosition.X,
            FixedY,
            TargetPosition.Z
        )

    local Distance =
        HorizontalDistance(
            Root.Position,
            FinalPosition
        )

    if Distance <=
        FarmSettings.ArrivalDistance then

        return true

    end

    local Duration =
        math.max(
            Distance /
                FarmSettings.TweenSpeed,
            0.08
        )

    local Rotation =
        Root.CFrame -
        Root.CFrame.Position

    local TargetCFrame =
        CFrame.new(
            FinalPosition
        )
        *
        Rotation

    CurrentTween =
        TweenService:Create(
            Root,

            TweenInfo.new(
                Duration,
                Enum.EasingStyle.Linear,
                Enum.EasingDirection.InOut
            ),

            {
                CFrame =
                    TargetCFrame
            }
        )

    local Finished = false

    local Connection

    Connection =
        CurrentTween.Completed:Connect(
            function()

                Finished = true

                if Connection then
                    Connection:Disconnect()
                end

            end
        )

    CurrentTween:Play()

    while FarmSettings.Enabled
        and not Finished do

        if not Root.Parent then
            break
        end

        -- Anti-seat durante o movimento
        if Humanoid.Sit then

            Humanoid.Sit = false

            Humanoid:ChangeState(
                Enum.HumanoidStateType.GettingUp
            )

        end

        -- Mantém a altura
        local Position =
            Root.Position

        if math.abs(
            Position.Y - FixedY
        ) > 0.03 then

            local CurrentRotation =
                Root.CFrame -
                Root.CFrame.Position

            Root.CFrame =
                CFrame.new(
                    Position.X,
                    FixedY,
                    Position.Z
                )
                *
                CurrentRotation

        end

        task.wait()

    end

    if Root.Parent then

        local Position =
            Root.Position

        local RotationNow =
            Root.CFrame -
            Root.CFrame.Position

        Root.CFrame =
            CFrame.new(
                Position.X,
                FixedY,
                Position.Z
            )
            *
            RotationNow

    end

    CancelTween()

    return FarmSettings.Enabled
        and Finished

end

--========================================================--
-- PROMPT
--========================================================--

local function GetAnyPrompt(
    Object
)

    if not Object then
        return nil
    end

    for _, Child in ipairs(
        Object:GetDescendants()
    ) do

        if Child:IsA(
            "ProximityPrompt"
        ) then

            return Child

        end

    end

    return nil

end

--========================================================--
-- ENCONTRAR LIXO
--========================================================--

local function GetLixos()

    local Result = {}

    for _, Object in ipairs(
        workspace:GetDescendants()
    ) do

        if Object.Name == "Lixo"
            and Object.Parent
            and not CollectedLixos[Object] then

            if GetPosition(
                Object
            ) then

                table.insert(
                    Result,
                    Object
                )

            end

        end

    end

    return Result

end

--========================================================--
-- LIXO MAIS PRÓXIMO
--========================================================--

local function GetBestLixo()

    local _,
        _,
        Root =
        GetCharacter()

    if not Root then
        return nil
    end

    local Best = nil

    local BestDistance =
        math.huge

    for _, Lixo in ipairs(
        GetLixos()
    ) do

        local Position =
            GetPosition(
                Lixo
            )

        if Position then

            local Distance =
                HorizontalDistance(
                    Root.Position,
                    Position
                )

            if Distance <
                BestDistance then

                BestDistance =
                    Distance

                Best =
                    Lixo

            end

        end

    end

    return Best

end

--========================================================--
-- CAMINHÃO
--========================================================--

local function GetTruck()

    local CarrosSpawnados =
        workspace:FindFirstChild(
            "CarrosSpawnados"
        )

    if not CarrosSpawnados then
        return nil
    end

    local Lixeiro =
        CarrosSpawnados:FindFirstChild(
            "Lixeiro"
        )

    if not Lixeiro then
        return nil
    end

    local Body =
        Lixeiro:FindFirstChild(
            "Body"
        )

    if not Body then
        return nil
    end

    local Colisao =
        Body:FindFirstChild(
            "Colisao"
        )

    if not Colisao then
        return nil
    end

    local ColisaoPartNew =
        Colisao:FindFirstChild(
            "ColisaoPartNew"
        )

    if not ColisaoPartNew then
        return nil
    end

    return
        Lixeiro,
        Body,
        Colisao,
        ColisaoPartNew

end

--========================================================--
-- PROMPT DO CAMINHÃO
--========================================================--

local function ScorePrompt(
    Prompt,
    BasePosition
)

    if not Prompt
        or not Prompt.Parent then

        return -math.huge

    end

    local ParentObject =
        Prompt.Parent

    if ParentObject:IsA(
        "Attachment"
    ) then

        ParentObject =
            ParentObject.Parent

    end

    local Position =
        GetPosition(
            ParentObject
        )

    if not Position then
        return -math.huge
    end

    local Distance =
        (
            Position -
            BasePosition
        ).Magnitude

    if Distance >
        FarmSettings.TruckSearchRadius then

        return -math.huge

    end

    local Action =
        string.lower(
            tostring(
                Prompt.ActionText
            )
        )

    local ObjectText =
        string.lower(
            tostring(
                Prompt.ObjectText
            )
        )

    local Score = 0

    if string.find(
        Action,
        "jogar",
        1,
        true
    ) then

        Score += 1000

    end

    if string.find(
        Action,
        "lixo",
        1,
        true
    ) then

        Score += 900

    end

    if string.find(
        ObjectText,
        "jogar",
        1,
        true
    ) then

        Score += 800

    end

    if string.find(
        ObjectText,
        "lixo",
        1,
        true
    ) then

        Score += 700

    end

    Score -= Distance

    return Score

end

local function FindTruckPrompt()

    local Lixeiro,
        _,
        _,
        ColisaoPartNew =
        GetTruck()

    if not Lixeiro then
        return nil
    end

    local BasePosition =
        GetPosition(
            ColisaoPartNew
        )

    if not BasePosition then
        return nil
    end

    local BestPrompt = nil

    local BestScore =
        -math.huge

    for _, Object in ipairs(
        Lixeiro:GetDescendants()
    ) do

        if Object:IsA(
            "ProximityPrompt"
        ) then

            local Score =
                ScorePrompt(
                    Object,
                    BasePosition
                )

            if Score >
                BestScore then

                BestScore =
                    Score

                BestPrompt =
                    Object

            end

        end

    end

    return BestPrompt

end

--========================================================--
-- ESPERAR PROMPT
--========================================================--

local function WaitForTruckPrompt()

    local StartTime =
        tick()

    while FarmSettings.Enabled
        and tick() - StartTime <
            FarmSettings.PromptSearchTime do

        local Prompt =
            FindTruckPrompt()

        if Prompt then
            return Prompt
        end

        task.wait(
            FarmSettings.PromptRetryDelay
        )

    end

    return nil

end

--========================================================--
-- ATIVAR PROMPT
--========================================================--

local function ActivatePrompt(
    Prompt
)

    if not Prompt
        or not Prompt.Parent then

        return false

    end

    if not FarmSettings.Enabled then
        return false
    end

    local OriginalEnabled =
        Prompt.Enabled

    local OriginalLOS =
        Prompt.RequiresLineOfSight

    local OriginalMaxDistance =
        Prompt.MaxActivationDistance

    pcall(
        function()

            Prompt.Enabled = true

            Prompt.RequiresLineOfSight =
                false

            Prompt.MaxActivationDistance =
                math.max(
                    OriginalMaxDistance,
                    FarmSettings.PromptDistance
                )

        end
    )

    task.wait(0.1)

    local Success = false

    for Attempt = 1, 5 do

        if not FarmSettings.Enabled then
            break
        end

        if not Prompt.Parent then

            Success = true
            break

        end

        local OK =
            pcall(
                function()

                    Prompt:InputHoldBegin()

                    local Hold =
                        Prompt.HoldDuration

                    if Hold > 0 then

                        task.wait(
                            Hold + 0.1
                        )

                    else

                        task.wait(
                            0.15
                        )

                    end

                    Prompt:InputHoldEnd()

                end
            )

        if OK then

            Success = true

            task.wait(0.25)

            break

        end

        task.wait(0.15)

    end

    if Prompt.Parent then

        pcall(
            function()

                Prompt.Enabled =
                    OriginalEnabled

                Prompt.RequiresLineOfSight =
                    OriginalLOS

                Prompt.MaxActivationDistance =
                    OriginalMaxDistance

            end
        )

    end

    return Success

end

--========================================================--
-- COLETAR LIXO
--========================================================--

local function CollectLixo(
    Lixo
)

    if not Lixo
        or not Lixo.Parent then

        return false

    end

    local Prompt =
        GetAnyPrompt(
            Lixo
        )

    if Prompt then

        return ActivatePrompt(
            Prompt
        )

    end

    task.wait(0.5)

    return true

end

--========================================================--
-- IR AO CAMINHÃO
--========================================================--

local function GoToTruckAndDiscard()

    if not FarmSettings.Enabled then
        return false
    end

    local Lixeiro,
        _,
        _,
        ColisaoPartNew =
        GetTruck()

    if not ColisaoPartNew then

        warn(
            "[ILHA BYLA RP] " ..
            "ColisaoPartNew não encontrado."
        )

        return false

    end

    local Prompt =
        FindTruckPrompt()

    --====================================================--
    -- IR PARA ColisaoPartNew
    --====================================================--

    if not Prompt then

        local BasePosition =
            GetPosition(
                ColisaoPartNew
            )

        if not BasePosition then
            return false
        end

        local Arrived =
            CFrameTweenHorizontal(
                BasePosition
            )

        if not Arrived then
            return false
        end

        Prompt =
            WaitForTruckPrompt()

    end

    --====================================================--
    -- IR PARA O PROMPT
    --====================================================--

    if Prompt then

        local ParentObject =
            Prompt.Parent

        if ParentObject:IsA(
            "Attachment"
        ) then

            ParentObject =
                ParentObject.Parent

        end

        local PromptPosition =
            GetPosition(
                ParentObject
            )

        if PromptPosition then

            local Arrived =
                CFrameTweenHorizontal(
                    PromptPosition
                )

            if not Arrived then
                return false
            end

        end

        task.wait(0.25)

        local CurrentPrompt =
            FindTruckPrompt()

        if CurrentPrompt then
            Prompt =
                CurrentPrompt
        end

        ActivatePrompt(
            Prompt
        )

        task.wait(0.8)

        return true

    end

    --====================================================--
    -- FALLBACK
    --====================================================--

    local BasePosition =
        GetPosition(
            ColisaoPartNew
        )

    if not BasePosition then
        return false
    end

    local Arrived =
        CFrameTweenHorizontal(
            BasePosition
        )

    if not Arrived then
        return false
    end

    task.wait(0.7)

    Prompt =
        WaitForTruckPrompt()

    if Prompt then

        ActivatePrompt(
            Prompt
        )

    end

    task.wait(0.5)

    return true

end

--========================================================--
-- FARM LOOP
--========================================================--

local function AutomationLoop()

    if Busy then
        return
    end

    Busy = true

    while FarmSettings.Enabled do

        local Character,
            Humanoid,
            Root =
            GetCharacter()

        if not Character
            or not Humanoid
            or not Root
            or Humanoid.Health <= 0 then

            task.wait(1)
            continue

        end

        if Humanoid.Sit then

            Humanoid.Sit = false

            Humanoid:ChangeState(
                Enum.HumanoidStateType.GettingUp
            )

        end

        --================================================--
        -- ENCONTRA LIXO
        --================================================--

        local Lixo =
            GetBestLixo()

        if not Lixo then

            task.wait(0.5)
            continue

        end

        local LixoPosition =
            GetPosition(
                Lixo
            )

        if not LixoPosition then

            CollectedLixos[Lixo] =
                true

            continue

        end

        --================================================--
        -- VAI AO LIXO
        --================================================--

        local Arrived =
            CFrameTweenHorizontal(
                LixoPosition
            )

        if not Arrived then
            continue
        end

        if not FarmSettings.Enabled then
            break
        end

        task.wait(0.2)

        --================================================--
        -- COLETA
        --================================================--

        CollectLixo(
            Lixo
        )

        CollectedLixos[Lixo] =
            true

        task.wait(0.3)

        --================================================--
        -- DESCARTE
        --================================================--

        if FarmSettings.Enabled then

            GoToTruckAndDiscard()

        end

        task.wait(0.5)

    end

    Busy = false

end

--========================================================--
-- UI OBSIDIAN
--========================================================--

FarmGroup:AddToggle(
    "AutoLixo",
    {
        Text = "Auto Lixo",
        Default = false,

        Callback = function(
            Value
        )

            FarmSettings.Enabled =
                Value

            if Value then

                task.spawn(
                    AutomationLoop
                )

                Library:Notify(
                    "Auto Lixo ativado!",
                    3
                )

            else

                CancelTween()

                local _,
                    Humanoid =
                    GetCharacter()

                if Humanoid then
                    Humanoid.Sit = false
                end

                Library:Notify(
                    "Auto Lixo desativado!",
                    3
                )

            end

        end
    }
)

--========================================================--
-- VELOCIDADE
--========================================================--

FarmGroup:AddSlider(
    "TweenSpeed",
    {
        Text = "Velocidade",
        Default = 25,
        Min = 5,
        Max = 50,
        Rounding = 0,

        Callback = function(
            Value
        )

            FarmSettings.TweenSpeed =
                Value

        end
    }
)

--========================================================--
-- INFO
--========================================================--

FarmGroup:AddLabel(
    "Lixo → Caminhão → Repetir"
)

FarmGroup:AddLabel(
    "CFrame Tween • Altura fixa"
)

--========================================================--
-- THEME / CONFIG
--========================================================--

ThemeManager:SetLibrary(
    Library
)

SaveManager:SetLibrary(
    Library
)

ThemeManager:SetDefaultTheme(
    "Dark"
)

--========================================================--
-- NOTIFICAÇÃO
--========================================================--

Library:Notify(
    "ILHA BYLA RP carregado!",
    5
)
