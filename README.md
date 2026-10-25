
-- Demon Hub: salvamento persistente de configurações
-- Coloque em ServerScriptService.
-- Ative API Services nas configurações do jogo no Studio.

local Players = game:GetService("Players")
local DataStoreService = game:GetService("DataStoreService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local store = DataStoreService:GetDataStore("DemonHubConfigs_v1")

local remote = ReplicatedStorage:FindFirstChild("DemonHubConfig")
if not remote then
    remote = Instance.new("RemoteFunction")
    remote.Name = "DemonHubConfig"
    remote.Parent = ReplicatedStorage
end

local allowed = {
    ESP = "boolean",
    NameTag = "boolean",
    HealthBar = "boolean",
    Distance = "boolean",
    Aimbot = "boolean",
    ShowFOV = "boolean",
    NPCESP = "boolean",
    MaxDistance = "number",
    FOV = "number",
    Smoothness = "number",
    FOVTransparency = "number",
}

local defaults = {
    ESP = false,
    NameTag = true,
    HealthBar = true,
    Distance = true,
    Aimbot = false,
    ShowFOV = true,
    NPCESP = false,
    MaxDistance = 1200,
    FOV = 120,
    Smoothness = 5,
    FOVTransparency = 50,
}

local function sanitize(input)
    local output = {}

    if type(input) ~= "table" then
        return nil
    end

    for key, expectedType in pairs(allowed) do
        local value = input[key]

        if type(value) == expectedType then
            if expectedType == "number" then
                if value ~= value or math.abs(value) == math.huge then
                    value = defaults[key]
                end

                if key == "MaxDistance" then
                    value = math.clamp(value, 100, 1200)
                elseif key == "FOV" then
                    value = math.clamp(value, 10, 300)
                elseif key == "Smoothness" then
                    value = math.clamp(value, 1, 20)
                elseif key == "FOVTransparency" then
                    value = math.clamp(value, 0, 100)
                end
            end

            output[key] = value
        else
            output[key] = defaults[key]
        end
    end

    return output
end

remote.OnServerInvoke = function(player, action, name, data)
    if type(name) ~= "string"
        or #name < 1
        or #name > 30
        or not name:match("^[%w _%-]+$") then
        return false, "Nome inválido."
    end

    local key = "user_" .. player.UserId .. "_" .. name

    if action == "Save" then
        local clean = sanitize(data)
        if not clean then
            return false, "Configuração inválida."
        end

        local ok, err = pcall(function()
            store:SetAsync(key, clean)
        end)

        if not ok then
            warn("Demon Hub save error:", err)
            return false, "Falha ao salvar. Verifique o DataStore."
        end

        return true, "Configuração salva!"
    elseif action == "Load" then
        local ok, result = pcall(function()
            return store:GetAsync(key)
        end)

        if not ok then
            warn("Demon Hub load error:", result)
            return false, "Falha ao carregar."
        end

        if type(result) ~= "table" then
            return false, "Configuração não encontrada."
        end

        return true, "Configuração carregada!", sanitize(result)
    end

    return false, "Ação desconhecida."
end

Players.PlayerRemoving:Connect(function()
    -- O salvamento ocorre quando o usuário aperta Save.
end)
