#### Game Backend Documentation

## Table of Contents

1. [Session Management](#session-management)  
2. [Persistence & Profile Store](#persistence--profile-store)  
3. [Player Data Service](#player-data-service)  
4. [Currency Service](#currency-service)  
5. [Survey Data Service](#survey-data-service)  
6. [Client-side UI Systems](#client-side-ui-systems)  
   - [Profile UI](#profile-ui)  
   - [Survey UI](#survey-ui)  
   - [Core UI Framework](#core-ui-framework)  
7. [Data Template](#data-template)  
8. [Remotes Reference](#remotes-reference)

---

## Session Management

**File LOcation:** `ServerScriptService/Handlers/SessionHandler.luau`

### Purpose

- Generate a unique 20-digit `SessionID` for each player on join.
- Store it as a Player attribute --> `SessionID` 

### Key Functions

lua
getTimestampMillis() --> string
-- Returns the current Unix time in milliseconds.

fnv1a32(str) --> number
-- 32-bit FNV-1a hash of a string.

generateSessionId(player) --> string
-- Combines timestamp + UserId hashes into a unique 20-digit SessionID.



Player Lifecycle: 
Code 
Players.PlayerAdded:Connect(function(player)
    local sessionID = generateSessionId(player)
    player:SetAttribute("SessionID", sessionID)
    print("SessionID for player", player.Name, "is", sessionID)
end)

## Persistence & Profile Store

### ProfileStore Module

** File Location:** `ServerScriptService/Modules/ProfileStore.luau`

**Purpose:**  
Abstract Roblox DataStore V2 into a session-based persistence layer.  
Handles locking, auto-saving, message queues, versioning, and critical-state detection.

### Key Concepts & API

| Concept         | Description |
|----------------|-------------|
| **Session Locking** | Only one server “owns” a profile at a time. |
| **Auto-save** | Periodically writes `profile.Data` back to the DataStore. |
| **Messaging** | Cross-server updates via `MessagingService`. |
| **Signals** | `OnError(message, store, key)`<br>`OnOverwrite(store, key)`<br>`OnCriticalToggle(isCritical)` |



### Instantiating Profile Store



local PlayerStore = ProfileStore.New(storeName, templateTable)
-- templateTable is a deep-copy default for new profiles



### Methods

| Method | Syntax | Description |
|--------|--------|-------------|
| **StartSessionAsync** | `ProfileStore:StartSessionAsync(profile_key, params?)` | Starts a session and returns a Profile object or nil. <br>**profile_key**: string – DataStore key <br>**params** (optional): table with keys: <br> - `Steal` (bool): Disregard existing session lock <br> - `Cancel` (function): If it returns true, cancels session start |
| **MessageAsync** | `ProfileStore:MessageAsync(profile_key, message)` | Sends a message (as a table) to the profile. <br>Returns `is_success` (boolean) |
| **GetAsync** | `ProfileStore:GetAsync(profile_key, version?)` | Reads a profile without starting a session or auto-saving. <br>**version** is optional (string) |
| **VersionQuery** | `ProfileStore:VersionQuery(profile_key, sort_direction?, min_date?, max_date?)` | Queries version history of a profile. <br>Supports optional sorting and date range |
| **RemoveAsync** | `ProfileStore:RemoveAsync(profile_key)` | Permanently removes the profile from the DataStore. <br>Returns `is_success` (boolean) |



---



## Player Data Service

### PlayerDataService Module

**Location:** `ServerScriptService/Services/PlayerDataService.luau`

---

### Purpose

- Manage per-player `ProfileStore` sessions.
- Provide simple getters/setters for game code.

---

### Initialization

- On server start:
  - Connects `Players.PlayerAdded` → `LoadProfile`
  - Connects `Players.PlayerRemoving` → `RemoveProfile`
- Also loads existing players in case of script reload or failure.

---

### Public API

#### `PlayerDataService.GetData(player)`  
Returns `profile.Data` or `nil`.  
Defined to retrieve data associated with a specific player.


function PlayerDataService.GetData(player)
	local profile = Profiles[player.UserId]
	if profile == nil then
		return nil
	end
	return profile.Data
end



* PlayerDataService.UpdateStat(player, statName, newValue)

function PlayerDataService.UpdateStat(player, stat, value)
	local data = PlayerDataService.GetData(player)
	if not data or data[stat] == nil then
		return
	end

	data[stat] = value
end




# Internals
- `LoadProfile(player)` → starts a ProfileStore session keyed on `player.UserId`; stores the returned Profile in a `Profiles` table.
- `RemoveProfile(player)` → calls `profile:EndSession()` when player leaves.

---

# Currency Service

**CurrencyService Module**

Location: `ServerScriptService/Services/CurrencyService.luau`

## Purpose
- Give in-game tokens to players for each mini-game they play and update their client UI.

## API
- `CurrencyService.GiveTokens(player, amount)`
  - Reads current tokens from `PlayerDataService.GetData(player).tokens`
  - Adds amount, writes back via `PlayerDataService.UpdateStat`
  - Fires the `UpdateStats` RemoteEvent to the client

---

# Survey Data Service

**SurveyDataService Module**

Location: `ServerScriptService/Services/SurveyDataService.luau`

## Purpose
- Send completed survey responses to an external Firebase Realtime Database.

- `SurveyDataService.SendSurvey(player, answersTable)`
  - `answersTable` must include at least `Number` (question ID) and `Response` (value).
  - Uses `HttpService:PostAsync` to `https://healthcare-goals-default-rtdb.firebaseio.com/`.

---

# Client-side UI Systems

## Profile UI

**ProfileController.luau**

Location: `StarterPlayerScripts/UI/Profile/ProfileController.luau`

### Purpose
- Display live token count, level, and stat bars.
- Listen to `UpdateStats.OnClientEvent` for updates.

---

## Survey UI

**SliderController.luau**

Location: `StarterPlayerScripts/UI/Survey/SliderController.luau`

### Purpose
- Render and animate a 0–10 slider with color and emoji feedback.
- Expose current value via `valueLabel.Text`.

**SurveyHandler.luau**

Location: `StarterPlayerScripts/UI/Survey/SurveyHandler.luau`

### Purpose
- Listen for `OpenSurvey.OnClientEvent(questionData)` → populate UI and show only the slider frame.
- On `SubmitButton.MouseButton1Click`, read `valueLabel.Text`, package into `{ Number=…, Response=… }` and `FireServer` via `SubmitSurvey`.

---

# Core UI Framework

**AnimationService.luau**

Location: `StarterPlayerScripts/UI/AnimationService.luau`

### Purpose
- Provide `TweenIn(gui, direction)` and `TweenOut(gui, direction)` for smooth on/off-screen animations.
- Manage an `OriginalPosition` attribute on each GUI for restoration.

**UIService.luau**

Location: `StarterPlayerScripts/UI/UIService.luau`

### Purpose
- Scan a container for any `TextButton` or `ImageButton` tagged "Toggle".
- Read each button’s `.Configuration.TargetGui` (StringValue) and `.Configuration.Direction` (StringValue).
- Move all target GUIs off-screen on load, then wire the button click to call `AnimationService.TweenIn/Out`.
- Maintain a map `UIService.ToggleDirections[guiName]`.

**UIHandler.luau**

Location: `StarterPlayerScripts/UI/UIHandler.luau`

### Purpose
- On client, listen for the server-sent `ToggleUIRequest.OnClientEvent(targetGuiName)`.
- Look up `PlayerGui[targetGuiName]`, read its default direction from `UIService.ToggleDirections`, and call `ToggleGui(gui, direction)`.







## Data Template

**DataTemplate.luau**

Location: `ReplicatedStorage/DataTemplate.luau`

---

## Remotes Reference

| Name           | Type        | Direction        | Parameters / Payload                                                                                                      | Client Handler (if any)                                           | Server Handler (if any)                                                                 |
|----------------|-------------|------------------|--------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------|------------------------------------------------------------------------------------------|
| ToggleUIRequest| RemoteEvent | Server → Client  | `arg1: targetGuiName` (string)                                                                                           | `UIHandler: ToggleUIRequest.OnClientEvent(targetGuiName)`         | `ToggleUIRequest:FireClient("Survey")`                                                  |
| UpdateStats    | RemoteEvent | Server → Client  | `arg1: dataTable` (table) – result of `PlayerDataService.GetData(player)`; keys: `tokens`, `level`, `stat1`–`stat4`        | `ProfileController: UpdateStats.OnClientEvent(data)`              | `UpdateStats:FireClient(dataTable)`                                                     |
| OpenSurvey     | RemoteEvent | Server → Client  | `arg1: questionData` (table) – must include: `Number` (number), `Text` (string), `Title` (string, optional)                | `SurveyHandler: OpenSurvey.OnClientEvent(questionData)`           | `OpenSurvey:FireClient(questionData)`                                                   |
| SubmitSurvey   | RemoteEvent | Client → Server  | `arg1: answerPayload` (table) – must include: `Number` (question ID), `Response` (number)                                  | —                                                                 | `SubmitSurvey.OnServerEvent(function(player, payload))`<br/>Typically calls `SurveyDataService.SendSurvey(player, payload)` |

---









