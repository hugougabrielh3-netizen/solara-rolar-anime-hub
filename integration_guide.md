# 🔗 Guia de Integração - Solara Hub

Como integrar o script Luck ao seu servidor Solara.

## 📦 Instalação

### 1. Copiar o Script
```bash
cp luck_script.lua /seu/servidor/solara/scripts/
```

### 2. Registrar Comandos (Arquivo de Configuração)
```lua
-- em config.lua ou commands.lua
local luck = require("scripts.luck_script")

-- Registrar comando !roll
registerCommand("roll", function(player, args)
    local result = luck.executeCommand(player.id, "roll", args)
    if result.success then
        sendMessage(result.message)
    end
end)

-- Registrar comando !stats
registerCommand("stats", function(player, args)
    local result = luck.executeCommand(player.id, "stats", args)
    if result.success then
        sendMessage(result.message)
    end
end)

-- Adicionar outros comandos conforme necessário...
```

## 🎯 Casos de Uso

### Case 1: Sistema de Loteria

```lua
local luck = require("scripts.luck_script")

function startLottery(players, prizeList)
    local winners = {}
    
    for _, playerId in ipairs(players) do
        luck.initializeLuck(playerId)
        local result = luck.rollWithLuck(playerId, "lottery")
        
        table.insert(winners, {
            playerId = playerId,
            roll = result.finalRoll,
            prize = prizeList[math.ceil(result.finalRoll / 10)]
        })
    end
    
    -- Ordena por roll (maior primeiro)
    table.sort(winners, function(a, b) return a.roll > b.roll end)
    
    return winners
end
```

### Case 2: Sistema de Gacha (Raridade)

```lua
function gacha(playerId, rarity)
    local rarityThresholds = {
        comum = 50,
        raro = 75,
        epico = 90,
        lendario = 98
    }
    
    local result = luck.rollWithLuck(playerId, "gacha_" .. rarity)
    local threshold = rarityThresholds[rarity] or 50
    
    return {
        playerId = playerId,
        success = result.finalRoll >= threshold,
        roll = result.finalRoll,
        threshold = threshold,
        critico = result.isCritical
    }
end
```

### Case 3: Batalha com Fator Sorte

```lua
function battleWithLuck(player1Id, player2Id)
    local roll1 = luck.rollWithLuck(player1Id, "battle")
    local roll2 = luck.rollWithLuck(player2Id, "battle")
    
    local winner = roll1.finalRoll > roll2.finalRoll and player1Id or player2Id
    
    return {
        player1 = {roll = roll1.finalRoll, critico = roll1.isCritical},
        player2 = {roll = roll2.finalRoll, critico = roll2.isCritical},
        winner = winner,
        difference = math.abs(roll1.finalRoll - roll2.finalRoll)
    }
end
```

### Case 4: Eventos com Probabilidade

```lua
function triggerEvent(playerId, eventType)
    local events = {
        spawn_boss = {minRoll = 70, reward = "boss_kill"},
        find_treasure = {minRoll = 80, reward = "gold"},
        encounter_rare = {minRoll = 85, reward = "rare_item"},
        super_lucky = {minRoll = 95, reward = "legendary"}
    }
    
    local event = events[eventType]
    if not event then return {success = false} end
    
    local result = luck.rollWithLuck(playerId, eventType)
    
    if result.finalRoll >= event.minRoll then
        return {
            success = true,
            reward = event.reward,
            roll = result.finalRoll,
            bonus = result.isCritical and "CRÍTICO!" or "Normal"
        }
    end
    
    return {success = false, roll = result.finalRoll}
end
```

## 🛠️ Customizações Comuns

### Modificar Multiplicadores de Luck

```lua
-- Editar no luck_script.lua
LUCK_CONFIG.BASE_MULTIPLIER = 1.5  -- Aumenta influência da sorte
LUCK_CONFIG.CRIT_MULTIPLIER = 3.0  -- Críticos mais poderosos
```

### Adicionar Novo Tipo de Roll

```lua
-- Estender o sistema
function rollWithLuckCustom(playerId, customType, customMultiplier)
    local totalLuck = luck.calculateTotalLuck(playerId)
    local luckMultiplier = 1 + (totalLuck / 100) * (customMultiplier or 1.0)
    
    local baseRoll = math.random(1, 100)
    local finalRoll = math.floor(baseRoll * luckMultiplier)
    
    return finalRoll
end
```

### Sistema de Leveling de Sorte

```lua
function levelUpLuck(playerId)
    local stats = luck.getPlayerStats(playerId)
    
    -- A cada 10 críticos, aumenta 1 ponto de sorte
    local levelUp = math.floor(stats.criticals / 10)
    
    if levelUp > 0 then
        local currentLuck = luck.getLuck(playerId)
        luck.setLuck(playerId, currentLuck + levelUp)
        return {success = true, luckGain = levelUp}
    end
    
    return {success = false}
end
```

## 🔐 Segurança

### Validações Recomendadas

```lua
-- Verificar permissão antes de usar boost
function addLuckBoostSafe(playerId, amount, duration)
    if not playerHasPermission(playerId, "use_luck_boost") then
        return {success = false, message = "Sem permissão"}
    end
    
    if amount > 50 then  -- Limite máximo
        return {success = false, message = "Boost muito alto"}
    end
    
    return luck.addLuckBoost(playerId, amount, duration)
end

-- Rate limiting
local lastRoll = {}
function rollWithRateLimit(playerId)
    local now = os.time()
    if lastRoll[playerId] and (now - lastRoll[playerId]) < 1 then
        return {success = false, message = "Aguarde antes de fazer outro roll"}
    end
    
    lastRoll[playerId] = now
    return luck.rollWithLuck(playerId)
end
```

## 📊 Monitoramento

### Logger de Rolls Importantes

```lua
function logImportantRoll(result)
    if result.isCritical or result.finalRoll >= 90 then
        local logEntry = string.format(
            "[%s] Player: %s | Roll: %d | Luck: %d | Crítico: %s",
            os.date("%Y-%m-%d %H:%M:%S"),
            result.playerId,
            result.finalRoll,
            result.luck,
            tostring(result.isCritical)
        )
        
        writeToLog("rolls_importantes.log", logEntry)
    end
end
```

### Dashboard de Estatísticas

```lua
function getServerStats()
    local totalPlayers = #getAllPlayers()
    local avgLuck = 0
    local totalCriticals = 0
    
    for _, playerId in ipairs(getAllPlayers()) do
        local stats = luck.getPlayerStats(playerId)
        avgLuck = avgLuck + stats.luck
        totalCriticals = totalCriticals + stats.criticals
    end
    
    return {
        totalPlayers = totalPlayers,
        avgLuck = math.floor(avgLuck / totalPlayers),
        totalCriticals = totalCriticals,
        timestamp = os.time()
    }
end
```

## 🐛 Troubleshooting

| Problema | Causa | Solução |
|----------|-------|---------|
| Rolls não variam | math.random sem seed | Adicionar `math.randomseed(os.time())` |
| Erro ao inicializar | Jogador já existe | Verificar com `if playerLuck[id] then` |
| Boosts não expiram | Função de update não chamada | Chamar `updateLuckBoosts()` periodicamente |
| Performance lenta | Histórico muito grande | Limitar histórico ou usar limpeza periódica |

## 📞 Suporte

Para dúvidas ou sugestões:
1. Verifique a documentação em README.md
2. Veja exemplos em examples.lua
3. Abra uma issue no repositório

---

**Última atualização:** 2024
