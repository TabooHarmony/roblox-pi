---
name: roblox-networking
description: >
  Remotes, validation, exploit literacy, rate limiting, server-authoritative networking,
  security hardening.
last_reviewed: 2026-05-21
---

<!-- Source: brockmartin/roblox-game-skill (MIT) -->

     1|# Roblox Multiplayer & Networking Reference
     2|
     3|---
     4|
     5|## 1. Overview
     6|
     7|**Load this reference when:**
     8|
     9|- Designing multiplayer game loops (rounds, lobbies, arenas)
    10|- Implementing matchmaking or queue systems
    11|- Building cross-server features (global chat, trading, server browsing)
    12|- Working with TeleportService for multi-place games
    13|- Creating team-based gameplay
    14|- Managing player lifecycles in a multiplayer context
    15|- Setting up private/reserved servers
    16|
    17|This document covers player management, team systems, lobby implementation, round-based game loops, TeleportService, MessagingService, matchmaking, server instance management, and production best practices for multiplayer Roblox games.
    18|
    19|---
    20|
    21|## 2. Player Management
    22|
    23|### Players Service
    24|
    25|The `Players` service is the root of all player-related functionality. Every connected player is represented by a `Player` instance that lives as a child of `Players`.
    26|
    27|```luau
    28|local Players = game:GetService("Players")
    29|
    30|-- Current player count
    31|local count = #Players:GetPlayers()
    32|
    33|-- Server capacity
    34|local maxPlayers = Players.MaxPlayers
    35|
    36|-- Iterate all connected players
    37|for _, player in Players:GetPlayers() do
    38|    print(player.Name, player.UserId)
    39|end
    40|```
    41|
    42|### PlayerAdded / PlayerRemoving
    43|
    44|These are the two most important events for multiplayer games. They fire on the server when a player joins or leaves.
    45|
    46|```luau
    47|-- ServerScriptService/PlayerManager.luau
    48|
    49|local Players = game:GetService("Players")
    50|
    51|local function onPlayerAdded(player: Player)
    52|    -- Load saved data
    53|    -- Initialize player state (score, team assignment, inventory)
    54|    -- Grant starter gear
    55|    -- Teleport to lobby spawn
    56|    print(`{player.Name} joined (UserId: {player.UserId})`)
    57|end
    58|
    59|local function onPlayerRemoving(player: Player)
    60|    -- Save player data (CRITICAL: do this before the player object is destroyed)
    61|    -- Clean up any per-player state tables
    62|    -- Notify other players
    63|    -- Update team balance
    64|    print(`{player.Name} leaving`)
    65|end
    66|
    67|Players.PlayerAdded:Connect(onPlayerAdded)
    68|Players.PlayerRemoving:Connect(onPlayerRemoving)
    69|
    70|-- Handle players who joined before this script ran (studio edge case)
    71|for _, player in Players:GetPlayers() do
    72|    task.spawn(onPlayerAdded, player)
    73|end
    74|```
    75|
    76|**Critical rule:** Always handle `PlayerRemoving` to save data. The player object and its descendants are destroyed shortly after this event fires. If you yield too long (e.g., a slow DataStore call), you risk losing the save. Use `game:BindToClose()` as a fallback for server shutdowns.
    77|
    78|### CharacterAdded / CharacterRemoving
    79|
    80|Each player's character is a `Model` in `Workspace` that contains the `Humanoid`, body parts, and accessories. Characters are created and destroyed on respawn.
    81|
    82|```luau
    83|local function onCharacterAdded(character: Model)
    84|    local humanoid = character:WaitForChild("Humanoid")
    85|    local rootPart = character:WaitForChild("HumanoidRootPart")
    86|
    87|    -- Set custom health
    88|    humanoid.MaxHealth = 150
    89|    humanoid.Health = 150
    90|
    91|    -- Listen for death
    92|    humanoid.Died:Connect(function()
    93|        -- Award kill to attacker, update scoreboard, etc.
    94|    end)
    95|end
    96|
    97|player.CharacterAdded:Connect(onCharacterAdded)
    98|```
    99|
   100|### LoadCharacter (Manual Respawning)
   101|
   102|By default, Roblox auto-spawns characters. For round-based games, disable auto-spawn and control it manually:
   103|
   104|```luau
   105|-- In StarterPlayer properties: set CharacterAutoLoads = false
   106|-- Or set it in script:
   107|Players.CharacterAutoLoads = false
   108|
   109|-- Spawn a specific player
   110|player:LoadCharacter()
   111|
   112|-- Spawn all players
   113|for _, player in Players:GetPlayers() do
   114|    task.spawn(function()
   115|        player:LoadCharacter()
   116|    end)
   117|end
   118|```
   119|
   120|### Player Instance Lifecycle
   121|
   122|Understanding the lifecycle prevents common bugs:
   123|
   124|1. `PlayerAdded` fires -- Player instance exists, no character yet.
   125|2. `CharacterAdded` fires -- Character model is parented to Workspace.
   126|3. `CharacterRemoving` fires -- Character is about to be destroyed (death or manual removal).
   127|4. `CharacterAdded` fires again -- Respawn.
   128|5. `PlayerRemoving` fires -- Player is disconnecting. Character may or may not exist.
   129|
   130|**Gotcha:** `player.Character` can be `nil` at any point. Always nil-check before accessing it.
   131|
   132|---
   133|
   134|## 3. Team Systems
   135|
   136|### Teams Service
   137|
   138|The `Teams` service holds `Team` objects. Teams are automatically replicated to all clients and show up in the default leaderboard.
   139|
   140|```luau
   141|local Teams = game:GetService("Teams")
   142|
   143|-- Create teams programmatically (or place them in Studio under Teams)
   144|local redTeam = Instance.new("Team")
   145|redTeam.Name = "Red"
   146|redTeam.TeamColor = BrickColor.new("Bright red")
   147|redTeam.AutoAssignable = false -- Don't auto-assign players
   148|redTeam.Parent = Teams
   149|
   150|local blueTeam = Instance.new("Team")
   151|blueTeam.Name = "Blue"
   152|blueTeam.TeamColor = BrickColor.new("Bright blue")
   153|blueTeam.AutoAssignable = false
   154|blueTeam.Parent = Teams
   155|
   156|local lobbyTeam = Instance.new("Team")
   157|lobbyTeam.Name = "Lobby"
   158|lobbyTeam.TeamColor = BrickColor.new("Medium stone grey")
   159|lobbyTeam.AutoAssignable = true -- New players go here
   160|lobbyTeam.Parent = Teams
   161|```
   162|
   163|### Assigning Players to Teams
   164|
   165|```luau
   166|-- Direct assignment
   167|player.Team = redTeam
   168|
   169|-- The player's nametag, leaderboard entry, and spawn location
   170|-- all update automatically based on TeamColor.
   171|
   172|-- Get all players on a team
   173|local redPlayers = redTeam:GetPlayers()
   174|print(`Red team has {#redPlayers} players`)
   175|```
   176|
   177|### Team-Based Logic
   178|
   179|Always check teams on the **server** before applying damage or other competitive interactions:
   180|
   181|```luau
   182|local function canDamage(attacker: Player, victim: Player): boolean
   183|    -- No friendly fire
   184|    if attacker.Team == victim.Team then
   185|        return false
   186|    end
   187|
   188|    -- No damaging lobby players
   189|    if victim.Team == lobbyTeam then
   190|        return false
   191|    end
   192|
   193|    return true
   194|end
   195|
   196|-- In a weapon hit handler (server-side)
   197|local function onWeaponHit(attacker: Player, victimCharacter: Model)
   198|    local victim = Players:GetPlayerFromCharacter(victimCharacter)
   199|    if not victim then return end
   200|
   201|    if not canDamage(attacker, victim) then return end
   202|
   203|    local humanoid = victimCharacter:FindFirstChild("Humanoid")
   204|    if humanoid then
   205|        humanoid:TakeDamage(25)
   206|    end
   207|end
   208|```
   209|
   210|### Auto-Balancing Teams
   211|
   212|```luau
   213|local function getSmallestTeam(teamList: {Team}): Team
   214|    local smallest = teamList[1]
   215|    local smallestCount = #smallest:GetPlayers()
   216|
   217|    for i = 2, #teamList do
   218|        local count = #teamList[i]:GetPlayers()
   219|        if count < smallestCount then
   220|            smallest = teamList[i]
   221|            smallestCount = count
   222|        end
   223|    end
   224|
   225|    return smallest
   226|end
   227|
   228|-- Assign player to the team with fewer members
   229|local function assignToBalancedTeam(player: Player)
   230|    player.Team = getSmallestTeam({ redTeam, blueTeam })
   231|end
   232|```
   233|
   234|---
   235|
   236|## 4. Lobby System
   237|
   238|A lobby holds players in a waiting area until enough are ready to start a round. This implementation tracks ready states, shows a ready-up GUI, enforces a minimum player threshold, and auto-starts after a timeout.
   239|
   240|### Server-Side Lobby Manager
   241|
   242|```luau
   243|-- ServerScriptService/LobbyManager.luau
   244|
   245|local Players = game:GetService("Players")
   246|local ReplicatedStorage = game:GetService("ReplicatedStorage")
   247|
   248|local LobbyRemotes = Instance.new("Folder")
   249|LobbyRemotes.Name = "LobbyRemotes"
   250|LobbyRemotes.Parent = ReplicatedStorage
   251|
   252|local ReadyUpEvent = Instance.new("RemoteEvent")
   253|ReadyUpEvent.Name = "ReadyUp"
   254|ReadyUpEvent.Parent = LobbyRemotes
   255|
   256|local LobbyStatusEvent = Instance.new("RemoteEvent")
   257|LobbyStatusEvent.Name = "LobbyStatus"
   258|LobbyStatusEvent.Parent = LobbyRemotes
   259|
   260|-- Configuration
   261|local MIN_PLAYERS = 2
   262|local MAX_WAIT_TIME = 60 -- seconds to auto-start after minimum reached
   263|local COUNTDOWN_DURATION = 10 -- final countdown before round starts
   264|
   265|-- State
   266|local readyPlayers: { [Player]: boolean } = {}
   267|local lobbyActive = true
   268|local countdownRunning = false
   269|
   270|local function getReadyCount(): number
   271|    local count = 0
   272|    for player, isReady in readyPlayers do
   273|        -- Verify the player is still connected
   274|        if isReady and player.Parent == Players then
   275|            count += 1
   276|        end
   277|    end
   278|    return count
   279|end
   280|
   281|local function getTotalPlayers(): number
   282|    return #Players:GetPlayers()
   283|end
   284|
   285|local function broadcastStatus(message: string, countdown: number?)
   286|    for _, player in Players:GetPlayers() do
   287|        LobbyStatusEvent:FireClient(player, message, countdown, getReadyCount(), getTotalPlayers())
   288|    end
   289|end
   290|
   291|local function shouldStart(): boolean
   292|    return getTotalPlayers() >= MIN_PLAYERS and getReadyCount() >= MIN_PLAYERS
   293|end
   294|
   295|local function startCountdown()
   296|    if countdownRunning then return end
   297|    countdownRunning = true
   298|
   299|    for i = COUNTDOWN_DURATION, 1, -1 do
   300|        if not lobbyActive then
   301|            countdownRunning = false
   302|            return
   303|        end
   304|
   305|        -- Recheck player count (someone may have left)
   306|        if getTotalPlayers() < MIN_PLAYERS then
   307|            broadcastStatus("Not enough players. Waiting...", nil)
   308|            countdownRunning = false
   309|            return
   310|        end
   311|
   312|        broadcastStatus(`Round starting in {i}...`, i)
   313|        task.wait(1)
   314|    end
   315|
   316|    countdownRunning = false
   317|    lobbyActive = false
   318|    broadcastStatus("Round starting!", 0)
   319|
   320|    -- Signal to round manager (see Section 5)
   321|    local RoundManager = require(script.Parent:WaitForChild("RoundManager"))
   322|    RoundManager.startRound()
   323|end
   324|
   325|-- Handle ready-up toggle
   326|ReadyUpEvent.OnServerEvent:Connect(function(player: Player)
   327|    if not lobbyActive then return end
   328|
   329|    readyPlayers[player] = not readyPlayers[player]
   330|    broadcastStatus(
   331|        if readyPlayers[player] then `{player.Name} is ready!` else `{player.Name} unreadied.`,
   332|        nil
   333|    )
   334|
   335|    if shouldStart() and not countdownRunning then
   336|        task.spawn(startCountdown)
   337|    end
   338|end)
   339|
   340|-- Clean up when players leave
   341|Players.PlayerRemoving:Connect(function(player: Player)
   342|    readyPlayers[player] = nil
   343|
   344|    if lobbyActive then
   345|        broadcastStatus(`{player.Name} left the lobby.`, nil)
   346|    end
   347|end)
   348|
   349|-- Auto-start timer: once minimum players are present, start a background timer
   350|task.spawn(function()
   351|    local waitElapsed = 0
   352|    while lobbyActive do
   353|        task.wait(1)
   354|        if getTotalPlayers() >= MIN_PLAYERS then
   355|            waitElapsed += 1
   356|            if waitElapsed >= MAX_WAIT_TIME and not countdownRunning then
   357|                -- Force all present players to ready
   358|                for _, player in Players:GetPlayers() do
   359|                    readyPlayers[player] = true
   360|                end
   361|                task.spawn(startCountdown)
   362|            end
   363|        else
   364|            waitElapsed = 0
   365|        end
   366|    end
   367|end)
   368|
   369|-- Public API for reset
   370|local LobbyManager = {}
   371|
   372|function LobbyManager.reset()
   373|    readyPlayers = {}
   374|    lobbyActive = true
   375|    countdownRunning = false
   376|    broadcastStatus("Lobby open. Ready up!", nil)
   377|end
   378|
   379|return LobbyManager
   380|```
   381|
   382|### Client-Side Ready-Up GUI
   383|
   384|```luau
   385|-- StarterPlayerScripts/LobbyGui.client.luau
   386|
   387|local Players = game:GetService("Players")
   388|local ReplicatedStorage = game:GetService("ReplicatedStorage")
   389|
   390|local player = Players.LocalPlayer
   391|local playerGui = player:WaitForChild("PlayerGui")
   392|local lobbyRemotes = ReplicatedStorage:WaitForChild("LobbyRemotes")
   393|local readyUpEvent = lobbyRemotes:WaitForChild("ReadyUp")
   394|local lobbyStatusEvent = lobbyRemotes:WaitForChild("LobbyStatus")
   395|
   396|-- Build GUI
   397|local screenGui = Instance.new("ScreenGui")
   398|screenGui.Name = "LobbyGui"
   399|screenGui.ResetOnSpawn = false
   400|screenGui.Parent = playerGui
   401|
   402|local frame = Instance.new("Frame")
   403|frame.Size = UDim2.fromScale(0.3, 0.15)
   404|frame.Position = UDim2.fromScale(0.35, 0.8)
   405|frame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
   406|frame.BackgroundTransparency = 0.3
   407|frame.Parent = screenGui
   408|
   409|local uiCorner = Instance.new("UICorner")
   410|uiCorner.CornerRadius = UDim.new(0, 12)
   411|uiCorner.Parent = frame
   412|
   413|local statusLabel = Instance.new("TextLabel")
   414|statusLabel.Size = UDim2.fromScale(1, 0.5)
   415|statusLabel.BackgroundTransparency = 1
   416|statusLabel.TextColor3 = Color3.new(1, 1, 1)
   417|statusLabel.TextScaled = true
   418|statusLabel.Text = "Waiting for players..."
   419|statusLabel.Parent = frame
   420|
   421|local readyButton = Instance.new("TextButton")
   422|readyButton.Size = UDim2.fromScale(0.6, 0.4)
   423|readyButton.Position = UDim2.fromScale(0.2, 0.55)
   424|readyButton.BackgroundColor3 = Color3.fromRGB(0, 170, 0)
   425|readyButton.TextColor3 = Color3.new(1, 1, 1)
   426|readyButton.TextScaled = true
   427|readyButton.Text = "Ready Up"
   428|readyButton.Parent = frame
   429|
   430|local readyCorner = Instance.new("UICorner")
   431|readyCorner.CornerRadius = UDim.new(0, 8)
   432|readyCorner.Parent = readyButton
   433|
   434|local isReady = false
   435|
   436|readyButton.Activated:Connect(function()
   437|    isReady = not isReady
   438|    readyButton.Text = if isReady then "Unready" else "Ready Up"
   439|    readyButton.BackgroundColor3 = if isReady
   440|        then Color3.fromRGB(170, 0, 0)
   441|        else Color3.fromRGB(0, 170, 0)
   442|    readyUpEvent:FireServer()
   443|end)
   444|
   445|lobbyStatusEvent.OnClientEvent:Connect(function(message: string, countdown: number?, readyCount: number, totalCount: number)
   446|    statusLabel.Text = `{message}\nReady: {readyCount}/{totalCount}`
   447|
   448|    if countdown and countdown == 0 then
   449|        screenGui.Enabled = false
   450|    end
   451|end)
   452|```
   453|
   454|---
   455|
   456|## 5. Round-Based Games
   457|
   458|### Round Lifecycle State Machine
   459|
   460|A production round system follows a clear state machine:
   461|
   462|```
   463|Intermission --> Countdown --> Playing --> Results --> Intermission
   464|```
   465|
   466|Each state has entry/exit logic, and the system must handle players joining and leaving at any point.
   467|
   468|### Complete Round Manager Implementation
   469|
   470|```luau
   471|-- ServerScriptService/RoundManager.luau
   472|
   473|local Players = game:GetService("Players")
   474|local ReplicatedStorage = game:GetService("ReplicatedStorage")
   475|local ServerStorage = game:GetService("ServerStorage")
   476|local Teams = game:GetService("Teams")
   477|
   478|-- Remotes
   479|local RoundRemotes = Instance.new("Folder")
   480|RoundRemotes.Name = "RoundRemotes"
   481|RoundRemotes.Parent = ReplicatedStorage
   482|
   483|local RoundStateEvent = Instance.new("RemoteEvent")
   484|RoundStateEvent.Name = "RoundState"
   485|RoundStateEvent.Parent = RoundRemotes
   486|
   487|local ScoreUpdateEvent = Instance.new("RemoteEvent")
   488|ScoreUpdateEvent.Name = "ScoreUpdate"
   489|ScoreUpdateEvent.Parent = RoundRemotes
   490|
   491|-- Configuration
   492|local INTERMISSION_TIME = 15
   493|local COUNTDOWN_TIME = 5
   494|local ROUND_TIME = 120 -- 2 minutes per round
   495|local RESULTS_TIME = 10
   496|local SCORE_TO_WIN = 10
   497|
   498|-- Map pool (models stored in ServerStorage/Maps/)
   499|local MAP_NAMES = { "Desert", "Forest", "City" }
   500|
   501|

---

## Security Hardening

     1|# Roblox Security Hardening Reference
     2|
     3|## 1. Overview
     4|
     5|Load this reference when:
     6|
     7|- Building or modifying `RemoteEvent` / `RemoteFunction` handlers
     8|- Handling player input that affects game state
     9|- Designing currency, inventory, or trading systems
    10|- Conducting security reviews of existing game code
    11|- Implementing anti-cheat or anti-exploit measures
    12|- Auditing what data is exposed to clients
    13|
    14|Every system that crosses the client-server boundary is an attack surface. This document provides production-ready patterns to harden those surfaces.
    15|
    16|---
    17|
    18|## 2. The Golden Rule: Never Trust the Client
    19|
    20|Exploiters can:
    21|
    22|- **Modify any LocalScript** -- injecting code, changing variables, hooking functions.
    23|- **Fire any RemoteEvent with arbitrary arguments** -- types, values, and counts are all attacker-controlled.
    24|- **Speed hack, fly, and teleport** -- the character's physics can be overridden entirely on the client.
    25|- **See all client-accessible code** -- anything in `StarterPlayerScripts`, `StarterGui`, `ReplicatedStorage`, or `ReplicatedFirst` is fully readable.
    26|- **Read and modify any client-side state** -- health displays, cooldown timers, UI flags.
    27|- **Intercept and replay network traffic** -- RemoteSpy tools let exploiters see every remote call and replay or modify them.
    28|
    29|**The client is a display layer, not a trusted authority.** It renders the world and collects input. The server decides what actually happens.
    30|
    31|A useful mental model: treat every `RemoteEvent:FireServer()` call as if it were an HTTP request from an anonymous stranger on the internet. Validate everything. Assume nothing.
    32|
    33|---
    34|
    35|## 3. RemoteEvent Validation Patterns
    36|
    37|### The Problem
    38|
    39|A bare remote handler like this is exploitable:
    40|
    41|```luau
    42|-- BAD: No validation at all
    43|DamageRemote.OnServerEvent:Connect(function(player, targetName, damage)
    44|    local target = Players:FindFirstChild(targetName)
    45|    target.Character.Humanoid:TakeDamage(damage)
    46|end)
    47|```
    48|
    49|An exploiter can fire this with any target name and any damage value, instantly killing anyone.
    50|
    51|### Production-Ready Validation Module
    52|
    53|Place this in `ServerScriptService`:
    54|
    55|```luau
    56|-- ServerScriptService/Modules/RemoteValidator.luau
    57|
    58|local RemoteValidator = {}
    59|
    60|--[[ -----------------------------------------------------------------------
    61|    Type Checking
    62|    Validates that arguments match expected types.
    63|----------------------------------------------------------------------- ]]
    64|
    65|type TypeSpec = string | (value: any) -> boolean
    66|
    67|function RemoteValidator.checkType(value: any, expected: TypeSpec): boolean
    68|    if type(expected) == "function" then
    69|        return expected(value)
    70|    end
    71|    return typeof(value) == expected
    72|end
    73|
    74|function RemoteValidator.validateArgs(
    75|    args: { any },
    76|    schema: { { name: string, type: TypeSpec, optional: boolean? } }
    77|): (boolean, string?)
    78|    for i, spec in schema do
    79|        local value = args[i]
    80|
    81|        if value == nil then
    82|            if not spec.optional then
    83|                return false, `Missing required argument: {spec.name}`
    84|            end
    85|            continue
    86|        end
    87|
    88|        if not RemoteValidator.checkType(value, spec.type) then
    89|            return false, `Invalid type for {spec.name}: expected {tostring(spec.type)}, got {typeof(value)}`
    90|        end
    91|    end
    92|
    93|    -- Reject extra arguments that were not declared in the schema
    94|    if #args > #schema then
    95|        return false, `Too many arguments: expected {#schema}, got {#args}`
    96|    end
    97|
    98|    return true, nil
    99|end
   100|
   101|--[[ -----------------------------------------------------------------------
   102|    Range Checking
   103|    Validates that numeric values fall within acceptable bounds.
   104|----------------------------------------------------------------------- ]]
   105|
   106|function RemoteValidator.checkRange(value: number, min: number, max: number): boolean
   107|    return type(value) == "number"
   108|        and value == value -- NaN check
   109|        and value >= min
   110|        and value <= max
   111|end
   112|
   113|function RemoteValidator.checkIntegerRange(value: number, min: number, max: number): boolean
   114|    return RemoteValidator.checkRange(value, min, max)
   115|        and math.floor(value) == value
   116|end
   117|
   118|--[[ -----------------------------------------------------------------------
   119|    Cooldown Tracking
   120|    Per-player, per-action cooldown enforcement.
   121|----------------------------------------------------------------------- ]]
   122|
   123|local cooldowns: { [Player]: { [string]: number } } = {}
   124|
   125|function RemoteValidator.checkCooldown(player: Player, action: string, cooldownSeconds: number): boolean
   126|    local now = os.clock()
   127|    local playerCooldowns = cooldowns[player]
   128|
   129|    if not playerCooldowns then
   130|        playerCooldowns = {}
   131|        cooldowns[player] = playerCooldowns
   132|    end
   133|
   134|    local lastUsed = playerCooldowns[action]
   135|    if lastUsed and (now - lastUsed) < cooldownSeconds then
   136|        return false
   137|    end
   138|
   139|    playerCooldowns[action] = now
   140|    return true
   141|end
   142|
   143|function RemoteValidator.clearPlayerCooldowns(player: Player)
   144|    cooldowns[player] = nil
   145|end
   146|
   147|--[[ -----------------------------------------------------------------------
   148|    Existence Checks
   149|    Validates that targets, objects, and instances actually exist.
   150|----------------------------------------------------------------------- ]]
   151|
   152|function RemoteValidator.playerExists(playerName: string): Player?
   153|    local Players = game:GetService("Players")
   154|    return Players:FindFirstChild(playerName) :: Player?
   155|end
   156|
   157|function RemoteValidator.characterAlive(player: Player): boolean
   158|    local character = player.Character
   159|    if not character then
   160|        return false
   161|    end
   162|
   163|    local humanoid = character:FindFirstChildOfClass("Humanoid")
   164|    if not humanoid then
   165|        return false
   166|    end
   167|
   168|    return humanoid.Health > 0
   169|end
   170|
   171|function RemoteValidator.instanceExists(parent: Instance, name: string, className: string?): Instance?
   172|    local child = parent:FindFirstChild(name)
   173|    if not child then
   174|        return nil
   175|    end
   176|
   177|    if className and not child:IsA(className) then
   178|        return nil
   179|    end
   180|
   181|    return child
   182|end
   183|
   184|--[[ -----------------------------------------------------------------------
   185|    Authorization
   186|    Checks if a player is allowed to perform an action.
   187|----------------------------------------------------------------------- ]]
   188|
   189|function RemoteValidator.playerOwnsItem(player: Player, itemId: string, inventoryFolder: Folder?): boolean
   190|    local folder = inventoryFolder or player:FindFirstChild("Inventory") :: Folder?
   191|    if not folder then
   192|        return false
   193|    end
   194|
   195|    return folder:FindFirstChild(itemId) ~= nil
   196|end
   197|
   198|function RemoteValidator.playerHasAttribute(player: Player, attribute: string, expectedValue: any?): boolean
   199|    local value = player:GetAttribute(attribute)
   200|    if expectedValue ~= nil then
   201|        return value == expectedValue
   202|    end
   203|    return value ~= nil
   204|end
   205|
   206|--[[ -----------------------------------------------------------------------
   207|    Distance Check
   208|    Validates that two positions are within an acceptable range.
   209|----------------------------------------------------------------------- ]]
   210|
   211|function RemoteValidator.withinRange(posA: Vector3, posB: Vector3, maxDistance: number): boolean
   212|    return (posA - posB).Magnitude <= maxDistance
   213|end
   214|
   215|function RemoteValidator.playerWithinRange(player: Player, targetPos: Vector3, maxDistance: number): boolean
   216|    local character = player.Character
   217|    if not character then
   218|        return false
   219|    end
   220|
   221|    local root = character:FindFirstChild("HumanoidRootPart")
   222|    if not root then
   223|        return false
   224|    end
   225|
   226|    return RemoteValidator.withinRange(root.Position, targetPos, maxDistance)
   227|end
   228|
   229|--[[ -----------------------------------------------------------------------
   230|    Cleanup
   231|----------------------------------------------------------------------- ]]
   232|
   233|game:GetService("Players").PlayerRemoving:Connect(function(player)
   234|    RemoteValidator.clearPlayerCooldowns(player)
   235|end)
   236|
   237|return RemoteValidator
   238|```
   239|
   240|### Using the Validation Module
   241|
   242|```luau
   243|-- ServerScriptService/RemoteHandlers/DamageHandler.server.luau
   244|
   245|local ReplicatedStorage = game:GetService("ReplicatedStorage")
   246|local ServerScriptService = game:GetService("ServerScriptService")
   247|
   248|local Validator = require(ServerScriptService.Modules.RemoteValidator)
   249|local DamageRemote = ReplicatedStorage.Remotes.DealDamage
   250|
   251|local MAX_DAMAGE = 50
   252|local DAMAGE_COOLDOWN = 0.5 -- seconds
   253|local ATTACK_RANGE = 15    -- studs
   254|
   255|local ARG_SCHEMA = {
   256|    { name = "targetPlayer", type = "Instance" },
   257|    { name = "damage",       type = "number" },
   258|}
   259|
   260|DamageRemote.OnServerEvent:Connect(function(player: Player, ...: any)
   261|    local args = { ... }
   262|
   263|    -- 1. Validate argument types
   264|    local valid, err = Validator.validateArgs(args, ARG_SCHEMA)
   265|    if not valid then
   266|        warn(`[DamageHandler] {player.Name}: {err}`)
   267|        return
   268|    end
   269|
   270|    local targetPlayer: Player = args[1]
   271|    local damage: number = args[2]
   272|
   273|    -- 2. Validate the target is actually a Player
   274|    if not targetPlayer:IsA("Player") then
   275|        return
   276|    end
   277|
   278|    -- 3. Validate damage range
   279|    if not Validator.checkIntegerRange(damage, 1, MAX_DAMAGE) then
   280|        warn(`[DamageHandler] {player.Name}: damage out of range ({damage})`)
   281|        return
   282|    end
   283|
   284|    -- 4. Cooldown check
   285|    if not Validator.checkCooldown(player, "DealDamage", DAMAGE_COOLDOWN) then
   286|        return
   287|    end
   288|
   289|    -- 5. Verify attacker is alive
   290|    if not Validator.characterAlive(player) then
   291|        return
   292|    end
   293|
   294|    -- 6. Verify target is alive
   295|    if not Validator.characterAlive(targetPlayer) then
   296|        return
   297|    end
   298|
   299|    -- 7. Range check -- attacker must be near the target
   300|    local targetRoot = targetPlayer.Character and targetPlayer.Character:FindFirstChild("HumanoidRootPart")
   301|    if not targetRoot then
   302|        return
   303|    end
   304|
   305|    if not Validator.playerWithinRange(player, targetRoot.Position, ATTACK_RANGE) then
   306|        warn(`[DamageHandler] {player.Name}: target out of range`)
   307|        return
   308|    end
   309|
   310|    -- 8. Authorization -- verify the player has a weapon equipped
   311|    local character = player.Character
   312|    local weapon = character and character:FindFirstChildOfClass("Tool")
   313|    if not weapon or not weapon:GetAttribute("CanDealDamage") then
   314|        warn(`[DamageHandler] {player.Name}: no valid weapon equipped`)
   315|        return
   316|    end
   317|
   318|    -- 9. Server calculates actual damage (never trust client damage value directly)
   319|    local serverDamage = math.min(damage, weapon:GetAttribute("MaxDamage") or MAX_DAMAGE)
   320|
   321|    -- 10. Apply damage
   322|    local targetHumanoid = targetPlayer.Character:FindFirstChildOfClass("Humanoid")
   323|    if targetHumanoid then
   324|        targetHumanoid:TakeDamage(serverDamage)
   325|    end
   326|end)
   327|```
   328|
   329|---
   330|
   331|## 4. Server-Authoritative Design
   332|
   333|The server owns all game state. The client requests actions; the server decides outcomes.
   334|
   335|### Movement Validation
   336|
   337|```luau
   338|-- ServerScriptService/Security/MovementValidator.server.luau
   339|
   340|local Players = game:GetService("Players")
   341|local RunService = game:GetService("RunService")
   342|
   343|local MAX_SPEED = 50             -- studs per second (walk + sprint + tolerance)
   344|local MAX_VERTICAL_SPEED = 100   -- studs per second (jumping/falling tolerance)
   345|local VIOLATION_THRESHOLD = 5    -- strikes before action
   346|local CHECK_INTERVAL = 0.5       -- seconds between checks
   347|
   348|local playerData: { [Player]: {
   349|    lastPosition: Vector3,
   350|    lastCheck: number,
   351|    violations: number,
   352|} } = {}
   353|
   354|Players.PlayerAdded:Connect(function(player)
   355|    player.CharacterAdded:Connect(function(character)
   356|        local root = character:WaitForChild("HumanoidRootPart")
   357|        playerData[player] = {
   358|            lastPosition = root.Position,
   359|            lastCheck = os.clock(),
   360|            violations = 0,
   361|        }
   362|    end)
   363|end)
   364|
   365|Players.PlayerRemoving:Connect(function(player)
   366|    playerData[player] = nil
   367|end)
   368|
   369|RunService.Heartbeat:Connect(function()
   370|    local now = os.clock()
   371|
   372|    for player, data in playerData do
   373|        if (now - data.lastCheck) < CHECK_INTERVAL then
   374|            continue
   375|        end
   376|
   377|        local character = player.Character
   378|        if not character then
   379|            continue
   380|        end
   381|
   382|        local root = character:FindFirstChild("HumanoidRootPart")
   383|        if not root then
   384|            continue
   385|        end
   386|
   387|        local dt = now - data.lastCheck
   388|        local displacement = root.Position - data.lastPosition
   389|        local horizontalSpeed = Vector3.new(displacement.X, 0, displacement.Z).Magnitude / dt
   390|        local verticalSpeed = math.abs(displacement.Y) / dt
   391|
   392|        if horizontalSpeed > MAX_SPEED or verticalSpeed > MAX_VERTICAL_SPEED then
   393|            data.violations += 1
   394|            warn(`[MovementValidator] {player.Name}: speed violation #{data.violations} (h={math.floor(horizontalSpeed)}, v={math.floor(verticalSpeed)})`)
   395|
   396|            if data.violations >= VIOLATION_THRESHOLD then
   397|                -- Teleport player back to last valid position
   398|                root.CFrame = CFrame.new(data.lastPosition)
   399|                -- Or kick for persistent abuse:
   400|                -- player:Kick("Movement anomaly detected.")
   401|            end
   402|        else
   403|            -- Decay violations over time for legitimate edge cases
   404|            data.violations = math.max(0, data.violations - 1)
   405|            data.lastPosition = root.Position
   406|        end
   407|
   408|        data.lastCheck = now
   409|    end
   410|end)
   411|```
   412|
   413|### Damage Validation
   414|
   415|```luau
   416|-- Server decides damage, not the client.
   417|
   418|local function calculateDamage(attacker: Player, weapon: Tool, target: Player): number?
   419|    local weaponConfig = WeaponDatabase[weapon.Name]
   420|    if not weaponConfig then
   421|        return nil
   422|    end
   423|
   424|    -- Server checks weapon cooldown
   425|    local lastFire = weapon:GetAttribute("LastFired") or 0
   426|    if os.clock() - lastFire < weaponConfig.Cooldown then
   427|        return nil
   428|    end
   429|
   430|    -- Server checks range
   431|    local attackerRoot = attacker.Character and attacker.Character:FindFirstChild("HumanoidRootPart")
   432|    local targetRoot = target.Character and target.Character:FindFirstChild("HumanoidRootPart")
   433|    if not attackerRoot or not targetRoot then
   434|        return nil
   435|    end
   436|
   437|    local distance = (attackerRoot.Position - targetRoot.Position).Magnitude
   438|    if distance > weaponConfig.Range then
   439|        return nil
   440|    end
   441|
   442|    -- Server calculates damage
   443|    weapon:SetAttribute("LastFired", os.clock())
   444|    return weaponConfig.BaseDamage
   445|end
   446|```
   447|
   448|### Currency Transactions
   449|
   450|```luau
   451|-- WRONG: Client tells server how much to add
   452|CurrencyRemote.OnServerEvent:Connect(function(player, amount)
   453|    player.leaderstats.Gold.Value += amount -- exploiter sends 999999
   454|end)
   455|
   456|-- RIGHT: Server calculates the reward
   457|QuestCompleteRemote.OnServerEvent:Connect(function(player, questId)
   458|    -- Validate quest ID type
   459|    if typeof(questId) ~= "string" then
   460|        return
   461|    end
   462|
   463|    -- Server checks quest state
   464|    local questData = PlayerQuestData[player]
   465|    if not questData or not questData[questId] then
   466|        return
   467|    end
   468|
   469|    if questData[questId].completed then
   470|        return -- already claimed
   471|    end
   472|
   473|    -- Server looks up the reward from its own data
   474|    local questConfig = QuestDatabase[questId]
   475|    if not questConfig then
   476|        return
   477|    end
   478|
   479|    -- Server awards the reward
   480|    questData[questId].completed = true
   481|    player.leaderstats.Gold.Value += questConfig.Reward
   482|end)
   483|```
   484|
   485|### Inventory Operations
   486|
   487|```luau
   488|-- Server-side trade validation
   489|local function executeTrade(playerA: Player, playerB: Player, itemIdA: string, itemIdB: string): boolean
   490|    -- Both players must be alive and in range
   491|    if not Validator.characterAlive(playerA) or not Validator.characterAlive(playerB) then
   492|        return false
   493|    end
   494|
   495|    -- Verify ownership on the server
   496|    local invA = playerA:FindFirstChild("Inventory")
   497|    local invB = playerB:FindFirstChild("Inventory")
   498|    if not invA or not invB then
   499|        return false
   500|    end
   501|
---

## Rate Limiting

Roblox's built-in throttle (~500 req/sec per client) is NOT a substitute for custom rate limiting. Players can still spam remotes at hundreds of requests per second. You need application-level throttling.

### Pattern 1: Per-Player Cooldown Table

Simple and effective for most games. Each remote has a minimum time between calls per player.

```luau
local cooldowns: {[Player]: {[string]: number}} = {}
local COOLDOWN = 0.2 -- seconds between calls

local function isThrottled(player: Player, remoteName: string): boolean
    local now = os.clock()
    if not cooldowns[player] then
        cooldowns[player] = {}
    end

    local lastCall = cooldowns[player][remoteName]
    if lastCall and (now - lastCall) < COOLDOWN then
        return true -- throttled
    end

    cooldowns[player][remoteName] = now
    return false
end

-- Clean up when player leaves
Players.PlayerRemoving:Connect(function(player)
    cooldowns[player] = nil
end)

-- Usage
BuyItem.OnServerEvent:Connect(function(player, itemId)
    if isThrottled(player, "BuyItem") then return end
    -- process purchase
end)
```

### Pattern 2: Declarative Remote Definitions

Define all remotes in one place with rate limits, validation, and allowed states. Cleaner than scattered OnServerEvent handlers.

```luau
type RemoteDef = {
    RateLimit: number?,
    Validate: (Player, ...any) -> boolean,
    Handler: (Player, ...any) -> (),
}

local Remotes: {[string]: RemoteDef} = {
    BuyItem = {
        RateLimit = 0.5,
        Validate = function(player, itemId)
            return typeof(itemId) == "string" and #itemId < 50
        end,
        Handler = function(player, itemId)
            -- process purchase
        end,
    },
    EquipTool = {
        RateLimit = 0.3,
        Validate = function(player, toolId)
            return typeof(toolId) == "string"
        end,
        Handler = function(player, toolId)
            -- equip tool
        end,
    },
}

-- Wire up automatically
for name, def in Remotes do
    local remote = ReplicatedStorage:WaitForChild(name)
    remote.OnServerEvent:Connect(function(player, ...)
        if def.RateLimit and isThrottled(player, name) then return end
        if not def.Validate(player, ...) then return end
        def.Handler(player, ...)
    end)
end
```

### Pattern 3: Suspicion Scoring

For high-stakes games. Track suspicious behavior over time instead of hard-blocking.

```luau
local suspicion: {[Player]: number} = {}
local SUSPICION_THRESHOLD = 10
local DECAY_RATE = 1 -- points lost per second

local function addSuspicion(player: Player, amount: number, reason: string)
    suspicion[player] = (suspicion[player] or 0) + amount
    if suspicion[player] >= SUSPICION_THRESHOLD then
        warn(`High suspicion for {player.Name}: {reason}`)
    end
end

-- In remote handler
BuyItem.OnServerEvent:Connect(function(player, itemId)
    if isThrottled(player, "BuyItem") then
        addSuspicion(player, 2, "rate limit exceeded")
        return
    end
    -- normal processing
end)

-- Decay suspicion over time
task.spawn(function()
    while true do
        task.wait(1)
        for player, score in suspicion do
            suspicion[player] = math.max(0, score - DECAY_RATE)
        end
    end
end)
```

### What NOT to Do

```luau
-- BAD: no rate limiting at all
BuyItem.OnServerEvent:Connect(function(player, itemId)
    -- exploiter can call this 1000 times/second
    grantItem(player, itemId)
end)

-- BAD: client-side rate limiting (exploiter bypasses)
-- Rate limiting MUST be server-side
```

Source: Roblox Server-Side Detection Guide (Roblox/creator-docs, MIT), DevForum rate limiting patterns
