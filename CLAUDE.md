# CLAUDE.md — P.A.N.D.A Mod

## Project Overview

**P.A.N.D.A** (PDA And News Dynamic Expansion) is a game modification for **S.T.A.L.K.E.R.: Anomaly** (an Anomaly-engine mod built on the Call of Pripyat platform). It adds:

- **Interactive Conversations** — player-initiated PDA-based dialogue with nearby NPCs, generating dynamic tasks and outcomes
- **Background NPC Chatter** — automatic NPC-to-NPC conversations that play in the world
- **Dynamic News** — PDA messages about in-world events (deaths, artifacts, faction activity, weather, etc.)
- **Private Inbox** — a full PDA tab with chat-bubble UI, typing indicators, and contact management for NPC message threads

The mod is authored by **PandaBlyat** and builds on earlier Dynamic News work by ARS Team / Alundaio / Tronex.

---

## Repository Structure

```
P.A.N.D.A-Mod/
└── P.A.N.D.A DEV/
    └── gamedata/
        ├── scripts/                  # All Lua game scripts
        │   ├── pda_interactive_conv.script       # Core interactive conversations engine
        │   ├── pda_interactive_conv_mcm.script   # MCM settings menu definition
        │   ├── pda_private_tab.script            # Private Inbox PDA UI tab class
        │   └── dynamic_news_manager.script       # Dynamic news/PDA message system
        ├── configs/
        │   ├── ui/
        │   │   ├── pda_private.xml               # Private Inbox UI layout (positions, sizes)
        │   │   └── textures_descr/
        │   │       └── ui_pda_private.xml        # Texture atlas descriptors for private tab UI
        │   └── text/eng/                         # Localization string tables (XML)
        │       ├── st_PANDA_friendly_conversations.xml
        │       ├── st_PANDA_hostile_conversations.xml
        │       ├── st_PANDA_task_conversations.xml
        │       ├── st_dynamic_news_conversations.xml
        │       ├── st_dynamic_news_faction_conversations.xml
        │       ├── st_dynamic_news_player_conversations.xml
        │       ├── st_dynamic_news.xml
        │       ├── st_dialog_manager.xml
        │       ├── st_dialogs.xml
        │       ├── ui_mcm_pda_interactive_conv.xml   # MCM UI label strings
        │       └── ui_st_loadscreen.xml
        └── textures/
            ├── banner_panda_n1_icon.dds
            ├── banner_panda_interactive_conv_icon.dds
            ├── banner_panda_dyn_news_icon.dds
            ├── banner_panda_npc_conv_icon.dds
            └── ui/
                └── ui_pda_private_app.dds        # Private tab UI texture atlas
```

---

## Language & Engine Details

- **Scripts:** Lua 5.1 (with Anomaly/X-Ray engine extensions), using `.script` file extension
- **Configs:** XML (UTF-8, but game may use windows-1251 for Russian localization)
- **Textures:** DDS (DirectDraw Surface) format
- **UI layouts:** XML-driven via `CScriptXmlInit` / `CUIScriptWnd` engine classes
- **MCM integration:** Uses `ui_mcm` global (Mod Configuration Menu framework)
- **No build step** — files are deployed directly into the game's `gamedata/` folder

---

## Script Architecture

### `pda_interactive_conv.script` — Core Engine

The main module (currently refactored to V7). Responsibilities:

- **Global state:** `pda_private_log` (message log table), `pda_typing_state`, `fake_talking_npc_id`, `pda_private_tab_open`
- **Message API:** `pda_private_add_message(contact_name, sender_name, text, icon, is_player)` — inserts messages newest-first
- **Typing state:** `set_pda_typing_state(name, active)` / `get_pda_typing_state(name)`
- **Delivery state:** `set_last_player_message_state(contact_name, state)`
- **Conversation trigger:** timed events using `time_global()`, configurable min/max intervals
- **NPC selection:** scans nearby NPCs, applies cooldowns, filters by faction, distance, etc.
- **Dialogue flow:** keyboard input (1–4 for choices, Space/0 to skip), turn-based NPC reply with configurable delays
- **Outcomes:** fetch, rescue, bounty, assault, eliminate, stash, delivery, measure, fate — mapped via `task_map`
- **Faction goodwill:** modified via `relation_registry.change_community_goodwill()`
- **Safe time events:** always use `safe_remove_time_event()` wrapper to avoid crashes when function is unavailable

Key constants:
```lua
local DIK_SPACE, DIK_0, DIK_1, DIK_2, DIK_3, DIK_4 = 57, 11, 2, 3, 4, 5
local FALLBACK_NPC_ICON = "ui_inGame2_Neutral_1"
```

### `pda_private_tab.script` — Private Inbox UI

OOP-style class using Anomaly's `CUIScriptWnd` OOP system:

```lua
class "pda_private_tab" (CUIScriptWnd)
local SINGLETON = nil
function get_ui() ... end  -- Returns singleton, calling Reset() each time
```

Caches invalidation pattern — always compare current state to cached values before rebuilding UI:
```lua
self.cached_log_size    -- invalidate when pda_private_log size changes
self.cached_conv_turn   -- invalidate when conversation turn advances
self.cached_has_conv    -- invalidate when active conversation starts/ends
self.cached_npc_data    -- NPC icon/name per contact
```

UI controls are initialized from `pda_private.xml` using XML paths like `"private_pda:info:description:chat_header_name"`.

### `pda_interactive_conv_mcm.script` — Settings Menu

Defines the full MCM settings tree returned from `on_mcm_load()`. The root ID is `"pda_interactive_conv"`.

MCM option paths follow the pattern: `"pda_interactive_conv/<section>/<option_id>"`

Precondition helpers gate visibility of sub-options:
```lua
local function isInteractiveEnabled() return ui_mcm.get(ROOT_ID.."/interactive/enabled") end
local function isFlexibleMode()       return ui_mcm.get(ROOT_ID.."/interactive/cooldown_mode") == 2 end
```

Three main sections:
| Section ID       | Color              | Description                          |
|------------------|--------------------|--------------------------------------|
| `mod_info`       | white              | Credits/info banner                  |
| `interactive`    | light purple       | Player-initiated NPC conversations   |
| `background`     | (defined in file)  | Auto NPC-to-NPC chatter              |
| `dynamic_news`   | (defined in file)  | PDA news messages                    |

### `dynamic_news_manager.script` — Dynamic News

Built on top of earlier community code (ARS Team → Alundaio → Tronex → P.A.N.D.A). All settings are read through `update_settings()` which calls `ui_mcm.get()` with fallback defaults.

Settings key path format: `"pda_interactive_conv/dynamic_news/<key>"`

Cycle tick intervals (seconds): `TickSpecial=240`, `TickRandom=240`, `TickCompanion=240`, `TickTask=300`

---

## Configuration System (MCM)

All runtime configuration flows through the **MCM (Mod Configuration Menu)** framework via the `ui_mcm` global. Never hardcode configurable values — always read them from MCM with a sensible default.

Pattern:
```lua
local function get_opt(key, default)
    if ui_mcm then
        local val = ui_mcm.get(key)
        if val ~= nil then return val end
    end
    return default
end
```

Key interactive conversation defaults:
| MCM Key                    | Default | Description                          |
|----------------------------|---------|--------------------------------------|
| `trigger_min`              | 8       | Min minutes between triggers         |
| `trigger_max`              | 15      | Max minutes between triggers         |
| `default_cooldown`         | 60      | Default NPC cooldown (minutes)       |
| `cooldown_mode`            | 2       | 1=strict, 2=flexible                 |
| `cooldown_hard_min`        | 5       | Hard minimum cooldown                |
| `spawn_distance_min`       | 30      | Min NPC scan distance (meters)       |
| `spawn_distance_max`       | 60      | Max NPC scan distance                |
| `companion_limit`          | 4       | Max companions per trigger           |
| `npc_reply_delay_min`      | 2       | Min NPC reply delay (seconds)        |
| `npc_reply_delay_max`      | 4       | Max NPC reply delay                  |
| `choice_tip_duration`      | 40      | Duration of choice tip (seconds)     |
| `pda_retry_delay`          | 2       | Retry delay                          |

---

## String / Localization Tables

All player-visible text lives in XML string tables under `gamedata/configs/text/eng/`. MCM label strings are in `ui_mcm_pda_interactive_conv.xml`.

String ID naming convention:
- `pda_interactive_conv_*` — MCM UI strings
- `st_panda_*` — conversation dialogue strings
- `st_dn_*` — dynamic news strings

When adding new dialogue or UI text, always add the string ID to the appropriate XML file rather than hardcoding text in Lua.

---

## Faction System

Factions are normalized internally. Known faction name mappings:

| Normalized   | Raw aliases              |
|--------------|--------------------------|
| stalker      | loner, stalker           |
| Duty     | duty, dolg               |
| freedom      | freedom                  |
| clear sky         | csky (Clear Sky)         |
| ecologists      | ecolog                   |
| mercenary    | killer                   |
| military        | army                     |
| bandit       | bandit                   |
| monolith     | monolith                 |
| zombified      | zombied                  |
| isg          | isg                      |
| renegade     | renegade                 |
| greh         | greh                     |

---

## Task Type System

The `task_map` in `pda_interactive_conv.script` maps simulation task IDs to conversation types:

| Type         | Simulation IDs                                      |
|--------------|-----------------------------------------------------|
| fetch        | 1–25, 53–55, 57–58, 60, 62                         |
| rescue       | 26–29                                               |
| bounty       | 30–35, 56, 63                                       |
| assault      | 36–40, 59, 64                                       |
| eliminate    | 41–43, 65–67                                        |
| stash        | 44–46                                               |
| delivery     | 47–49                                               |
| measure      | 50–51, 61                                           |
| fate         | 52                                                  |

---

## Coding Conventions

### Lua Style

- Use `local` for all helper functions and variables — minimize globals
- Globals are only declared at module top-level when needed by other scripts (e.g., `pda_private_log`, `fake_talking_npc_id`)
- Use descriptive variable names: `contact_name`, `sender_icon`, `delivery_state`
- Tables that are shared across scripts are initialized at the top of the primary module:
  ```lua
  pda_private_log = {}
  pda_typing_state = { active=false, contact=nil, dots=0 }
  ```
- Version history and fix notes go in the top-of-file block comment `--[[ ... --]]`

### Safety Patterns

Always wrap optional engine APIs with safety checks:

```lua
-- Safe time event removal (engine may not have this function in all versions)
local function safe_remove_time_event(id, fn)
    if level and level.remove_time_event then
        level.remove_time_event(id, fn)
    end
end
```

Use `pcall()` for any operation that might fail on optional APIs.

### UI / XML

- UI class inherits from `CUIScriptWnd` using Anomaly's OOP extension: `class "name" (CUIScriptWnd)`
- Use singleton pattern for UI windows: one global instance, reset on reopen
- Initialize all controls from XML in `InitControls()`, register callbacks in `InitCallBacks()`
- Always reference controls by XML hierarchy path string: `"parent:child:grandchild"`
- Invalidation caching: store last-known state in `self.cached_*` fields and only rebuild UI when values change

### MCM Definitions

- All MCM options must have a `def` (default) value
- Use `precondition` to conditionally show options based on parent toggles
- Banner slides use `.dds` files from `gamedata/textures/`
- Section color scheme: use `clr = {255, R, G, B}` (alpha first)

---

## Development Workflow

### Making Changes

1. Edit `.script` files in `P.A.N.D.A DEV/gamedata/scripts/`
2. Edit `.xml` files in `P.A.N.D.A DEV/gamedata/configs/`
3. Test by deploying `P.A.N.D.A DEV/gamedata/` into the Anomaly game directory
4. There is no automated build, test runner, or linter — testing is done in-game

### Commit Message Style

The project uses version-tagged commit messages:

```
V0.9.XX - Brief description of change
```

For AI-assisted changes, use descriptive messages without version tags unless bumping the version manually.

### Branching

- `master` — stable releases
- Feature/fix branches: `codex/<short-description>` (used for AI-assisted PRs)
- Current AI development branch: `claude/claude-md-mmi1i63ezepbp906-ApSlc`

### File Deployment

The mod has no build system. To test:
- Copy the contents of `P.A.N.D.A DEV/gamedata/` into `<Anomaly Install>/gamedata/`
- Scripts reload on game load; some changes require a new game or save reload

---

## Key Integration Points

When modifying this mod, be aware of these cross-script dependencies:

| Consumer                     | Dependency                                              |
|------------------------------|---------------------------------------------------------|
| `pda_private_tab.script`     | Reads `pda_private_log`, `pda_typing_state` globals from `pda_interactive_conv.script` |
| `pda_interactive_conv.script`| Calls `pda_private_tab.set_pda_private_tab_open()` to track UI state |
| `dynamic_news_manager.script`| Reads all settings via `ui_mcm.get("pda_interactive_conv/dynamic_news/...")` |
| `pda_interactive_conv_mcm.script` | Defines MCM tree consumed by the MCM framework; no direct script calls |
| All scripts                  | `ui_mcm` global must be available (loaded by MCM framework) |

---

## Known Constraints & Gotchas

- **XML nodes must be optional** — creating new XML nodes in `pda_private.xml` must handle missing nodes gracefully to prevent PDA crashes (lesson from commit `8a1dd0b`)
- **Time events must use safe removal** — `level.remove_time_event` may not exist in all engine versions; always guard it
- **Conversation restart prevention** — duplicate `open_ui` time events must be cancelled before scheduling new ones
- **Faction normalization is required** — raw faction names from the engine don't always match expected strings; normalize before comparisons
- **`fake_talking_npc_id`** is a global used by engine override hooks to simulate NPC talking state — do not repurpose this variable
- **Message log is newest-first** — `pda_private_log` inserts at index 1; UI must account for this ordering



## Stalker Anomaly Vanilla resources:

S.T.A.L.K.E.R. Anomaly Modding Reference: Core Systems & Functions
This document summarizes important global functions, tables, and systems exposed in _G.script (and related modules) that are essential for modding NPCs, squads, mutants, player mechanics, and the underlying simulation. Use this as a quick reference when developing scripts for Anomaly.

1. General Scripting Tips (from header)
Never keep userdata (engine objects) in global scope – always store object IDs and retrieve them via level.object_by_id(id) or db.storage[id].object. Otherwise objects cannot be destroyed properly, leading to crashes or silent errors.

Avoid using reserved names like task, level, weather as variables – they are engine namespaces.

Be careful with variable names that collide with global functions; check _G to avoid accidental overrides.

2. Global Variables & Constants
Name	Purpose
GAME_VERSION	Current game version string
DEV_DEBUG, DEV_DEBUG_DEV	Set when -dbg or -dbgdev command line args are present
USE_INI_MEMOIZE	Enables caching of system_ini() reads for performance
DEACTIVATE_SIM_ON_NON_LINKED_LEVELS	If true, simulation updates only on actor's level and linked levels
USE_MARSHAL	Whether the marshal library is available (for advanced table serialization)
ini_sys	Cached system_ini() handle
AC_ID	Actor’s object ID (set during simulation)
mus_vol, amb_vol	Temporary volume storage for sound muting
KEYS_UNLOCK	When false, action keybinds are disabled (used by scripted GUIs)
_GUIs, _GUIsInstances, _GUIs_keyfree	Tables tracking active UI dialogs
_EVENT	Global key-value storage for passing data between scripts
_ALIFE_CNT, _ALIFE_WARNING	Count of alive server objects and warning threshold
_ALIFE_CACHE, _ALIFE_CACHE_RECORD	Debug logging of object creation/deletion
_ITM	Lookup table for item types (filled by Parse_ITM())
schemes, schemes_by_stype	Scheme registration tables used by modules.script
VEC_ZERO, VEC_X, VEC_Y, VEC_Z	Common vector constants
BoneID, HitTypeID, BoosterID, SCANNED_SLOTS	Enumerations for bones, hit types, boosters, inventory slots
3. Core Systems
3.1 Callback System
Allows scripts to register callbacks for engine or scripted events.

lua
RegisterScriptCallback(name, func_or_userdata)
UnregisterScriptCallback(name, func_or_userdata)
SendScriptCallback(name, ...)   -- triggers all registered callbacks and axr_main[name]
AddScriptCallback(name)          -- adds a callback to the engine's list
Common callbacks are defined in axr_main.script and can be extended.

3.2 Delayed Event Queue
Run functions after a delay or repeatedly until they return true. Does not persist through saves.

lua
CreateTimeEvent(ev_id, act_id, timer_sec, func, ...)
RemoveTimeEvent(ev_id, act_id)
ResetTimeEvent(ev_id, act_id, new_timer_sec)
ProcessEventQueue([force])  -- called in update loops
3.3 Persistent Data Storage
db.storage – holds online object data (set on net_spawn, cleared on net_destroy). Contains:

object – the game object (userdata)

pstor – table for arbitrary Lua data (saved via marshal)

pstor_ctime – for game.CTime values

disable_input_time, disable_input_idle – actor input blocking

Save/Load helpers (for online objects):

lua
save_var(obj, varname, value)
load_var(obj, varname, default)
save_ctime(obj, varname, ctime_value)
load_ctime(obj, varname)
Server‑side storage (offline objects, via alife_storage_manager):

lua
se_save_var(id, name, varname, value)
se_load_var(id, name, varname)
3.4 Server Object Management (ALife)
Functions to create, release, and manipulate server objects.

lua
alife_object(id)                     → server object
alife_create(section, pos, lvid, gvid, [id], [state])
alife_create(section, obj, [with_id], [state])   -- using another object's properties
alife_create_item(section, owner_obj_or_table, [properties_table])
alife_process_item(section, item_id, properties_table)   -- apply properties to online item
alife_release(se_obj, [msg])          -- release server object (handles squads/NPCs specially)
alife_release_id(id, [msg])
alife_clone_weapon(se_obj, [new_section], [parent_id])
alife_character_community(se_obj)     → community string
alife_on_limit() → bool               -- true if near object limit (64000)
alife_record(se_obj, created)         -- internal counter update
alife_first_update()                  -- count existing objects after game start
create_ammo(section, pos, lvi, gvi, parent_id, count) → table of ammo objects
3.5 Object Identification Helpers
Check the type of a game or server object.

lua
IsStalker([obj], [clsid])
IsMonster([obj], [clsid])
IsAnomaly([obj], [clsid])
IsTrader([obj], [clsid])
IsCar([obj], [clsid])
IsHelicopter([obj], [clsid])
IsInvbox([obj], [clsid])
IsWounded(obj)                 -- checks if NPC is critically wounded and in special state
IsOutfit([obj], [clsid])
IsHeadgear([obj], [clsid])
IsWeapon([obj], [clsid])
IsPistol, IsSniper, IsShotgun, IsRifle, IsLauncher, IsMelee, IsGrenade, IsAmmo, IsBolt, IsArtefact
Many of these use a cached lookup table (_ITM) for items; Parse_ITM() initializes it from system_ini.

3.6 Item Type Lookup (_ITM)
After Parse_ITM() is called, _ITM[type][section] contains true or a specific value for that item. Useful types:

"ammo", "sil", "scope", "gl" (grenade launcher)

"helmet", "outfit", "backpack", "artefact"

"device" (detectors, etc.), "craft", "repair", "workshop", "disassemble", "cook", "map", "money", "recipe", "letter", "meal", "eatable"

"tool", "part", "upgrade", "quest", "consumable", "multiuse", "multiuse_r"

"fake_ammo", "fake_ammo_wpn"

lua
IsItem(typ, sec)               → bool or value
GetItemList(typ)                → table of sections for that type
4. Player & NPC Specifics
4.1 Actor
lua
give_object_to_actor(section, [count])     -- spawn item into actor's inventory
get_actor_true_community()                  → string (e.g., "stalker", ignoring disguise)
set_actor_true_community(new_comm, now)     – changes player’s faction (used by story mode)
get_player_level_id()                       → level ID where actor currently is
IsMoveState(state_name, [compare_state])    – check actor movement state (see actor_move_states table)
set_inactivate_input_time(delta_sec)        – disable input for delta seconds
Actor movement states: "mcFwd", "mcBack", "mcLStrafe", "mcRStrafe", "mcCrouch", "mcAccel", "mcSprint", etc.

4.2 NPCs
lua
character_community(obj)                    → community string
get_object_community(obj)                    – works for both game and server objects
get_object_squad(obj)                        → server squad object (or nil)
npc_in_actor_frustrum(npc)                   → bool (if NPC is within ~35° of camera direction)
change_team_squad_group(se_obj, team, squad, group) – changes both online and offline representation
get_speaker([safe], [all])                    – returns NPC currently talking to actor
4.3 Wounded State
IsWounded(obj) checks if an NPC is in the special wounded state (e.g., after being downed). Additional variables like load_var(obj, "wounded_state") and load_var(obj, "wounded_fight") control the behavior.

4.4 Squad & Group Handling
When releasing an NPC, the squad object automatically updates its member list:

lua
alife_release(se_obj)  – if NPC, it calls squad:remove_npc(id, true)
Squads themselves can be released with alife_release(squad_obj), which calls se_obj:remove_squad().

5. Level Changing & Teleportation
5.1 Changing Level
lua
ChangeLevel(pos, lvid, gvid, angle, [anim])   – triggers level change (with optional fade animation)
change_level_now(pos, lvid, gvid, angle)       – immediate change (sends network packet)
JumpToLevel(new_level_name)                     – teleports actor to first suitable game vertex on that level
ChangeLevel with anim uses a delayed event to hide PDA and apply a fade effect before actual travel.

5.2 Teleporting Objects (requires OpenXRay)
lua
TeleportObject(id, pos, lvid, gvid)            – teleports a single object (updates offline flags)
TeleportSquad(squad, pos, lvid, gvid)           – teleports squad leader and all members
These functions clear cached level vertices to force engine re‑pathfinding.

5.3 Level Handlers
lua
AddUniqueCall(functor)       – adds a level call that runs until functor returns true (removes duplicates)
RemoveUniqueCall(functor)
level_changing()              → true if a level transition is in progress
in_time_interval(val1, val2)  – checks current hour against a day‑night interval (supports wrap‑around)
6. Story IDs & Global Events
6.1 Story Objects
Functions to retrieve objects by their story ID (set in all.spawn or via scripts).

lua
get_story_se_object(story_id)       → server object
get_story_se_item(story_id)         → server item (special handling)
get_story_object(story_id)           → game object (if online)
get_story_squad(story_id)            → alias for get_story_se_object
get_object_story_id(obj_id)           → story ID of given object ID
get_story_object_id(story_id)         → object ID
unregister_story_object_by_id(obj_id) – removes story association
level_object_by_sid(story_id)         → game object on current level
id_by_sid(story_id)                   → object ID
6.2 Event Storage (_EVENT)
A global table for ad‑hoc data sharing between scripts.

lua
SetEvent(key, value)                 – store value under key
SetEvent(key, subkey, value)          – store in a nested table
GetEvent(key)                          – retrieve value
GetEvent(key, subkey)                  – retrieve nested value
7. GUI System
7.1 Registering/Unregistering UI
Scripts can register custom dialogs to block input and prevent conflicts.

lua
Register_UI(name, path, instance)      – marks UI as active; blocks keybinds unless in _GUIs_keyfree
Unregister_UI(name)                     – removes UI, restores keybinds if last
Check_UI([name])                         – true if any (or specific) UI is open
Overlapped_UI(name)                       – true if more than one UI is open (excluding dialog)
CloseAll_UI()                              – closes all registered UIs via their Close() method
DestroyAll_UI()                             – completely removes UI references from _GUIsInstances
Global keybinds are automatically disabled when any non‑keyfree UI is open (KEYS_UNLOCK).

7.2 HUD Control
lua
show_indicators(state)                    – enable/disable HUD indicators
main_hud_shown()                           → true if indicators on, inventory/PDA closed, no zoom, etc.
hide_hud_all()                              – closes inventory, PDA, and all registered UIs
hide_hud_inventory() / hide_hud_pda()       – specific closers
show_all_ui(show)                           – hides/shows all UI elements (including indicators)
7.3 Actor Menu Helper
lua
GetActorMenu() → actor_menu object (if available)
8. Utility Functions
8.1 Math
lua
round(x), round_idp(num, idp), round_100(x)
clamp(val, min, max)
normalize(val, min, max), normalize_100(val, min, max)
random_choice(...)                 – picks one argument at random
random_number([min, max])           – integer random
random_float(min, max)               – float random
yaw(v1, v2), yaw_degree(v1, v2)      – angle between 2D vectors (XZ)
yaw_degree3d(v1, v2)                  – 3D angle
vector_cross(v1, v2)                   – cross product
vector_rotate_y(v, angle_deg)          – rotate around Y axis
distance_2d(a, b), distance_2d_sqr(a, b)
8.2 String
lua
trim(s)
strformat(fmt, ...)                     – custom formatter (replaces %s with tostring)
str_explode(str, sep, [plain])           → table of strings
parse_list(ini, key, val, [convert])     – parses comma‑separated list from ini
parse_names(s), parse_nums(s)             – extract words/numbers
parse_key_value(s)                         – “key=value” pairs
starts_with(str, start)
has_translation(string_id)                  – true if game.translate_string returns different string
get_param_string(src_string, obj)           – replaces $script_id$ with object’s script ID
8.3 Table
lua
is_empty(t), is_not_empty(t)
empty_table(t), iempty_table(t)          – clear a table
size_table(t)
random_key_table(t)                        → random key from table
copy_table(dest, src), dup_table(src)       – shallow/deep copy
shuffle_table(t)
invert_table(t)                              – swaps keys and values
t2k_table(t) / k2t_table(t)                  – convert between list and set
print_table(table, [subs])                    – debug print
store_table(table, [subs])                     – print in a loadable format
spairs(t, [order])                              – sorted iterator
8.4 Vector
lua
vec_sub(a,b), vec_add(a,b), vec_set(vec)     – returns new vector
VEC_ZERO, VEC_X, VEC_Y, VEC_Z
9. Engine Hooks / Callbacks
These functions are called by the engine and can be intercepted via SendScriptCallback or overridden. They are defined in the script and often trigger script callbacks.

Hook	Description
CInventory__eat(item)	Called when an item is used; returning false prevents usage. Triggers actor_on_item_before_use.
CActor__HitArtefactsOnBelt(hit_table, hit_power, hit_type)	Allows overriding artefact hit protection. Returns modified hit_table.
CHudItem__OnMotionMark(state, mark)	Called on weapon animation marks. Triggers actor_on_hud_animation_mark.
CHudItem__PlayHUDMotion(anm_table, obj)	Can modify HUD animation before playing. Triggers actor_on_hud_animation_play.
player_hud__OnMovementChanged(cmd)	Triggers actor_on_movement_changed.
CMissile__PutNextToSlot(itm)	Called when selecting a throwable; allows replacing item. Triggers actor_on_before_throwable_select.
CBulletManager__ObjectHit(section, obj, pos, dir, mtl, speed, wpn_id)	Bullet impact callback. Triggers bullet_on_hit.
CAI_Stalker__GetWeaponAccuracy(obj, wpn, dispersion, body_state, move_type)	Can modify NPC accuracy. Triggers npc_shot_dispersion.
CActor__BeforeHitCallback(actor, hit, bone_id)	Return false to ignore hit. Triggers actor_on_before_hit.
CAI_Stalker__BeforeHitCallback(npc, hit, bone_id)	Triggers npc_on_before_hit.
CBaseMonster__BeforeHitCallback(monster, hit, bone_id)	Triggers monster_on_before_hit.
CActor__FootstepCallback(material, power, hud_view)	Return false to mute footstep. Triggers actor_on_footstep.
CActor_Fire()	Return false to prevent weapon firing. Triggers actor_on_weapon_before_fire.
CCustomZone_BeforeActivateCallback(zone, obj)	Return false to prevent zone activation on object. Triggers anomaly_on_before_activate.
CBurer_BeforeWeaponDropCallback(monster, wpn)	Return false to prevent burer from disarming. Triggers burer_on_before_weapon_drop.
CBolt__State(id)	If returns true and limited bolts option is on, bolt is not released after throwing.
CZone_Touch(obj)	Called when actor touches anomaly. Return false to prevent feeling. Triggers actor_on_feeling_anomaly.
CHUDManager_OnScreenResolutionChanged()	Triggers on_screen_resolution_changed and destroys all UI.
CActor_on_jump(), CActor_on_land(landing_speed)	Triggers actor jump/land callbacks.
CALifeUpdateManager__on_before_change_level(packet)	Called just before level change save; can modify packet. Also handles attached vehicle teleport.
update_best_weapon(npc, cur_wpn)	Called when engine chooses best weapon; can override by setting flags.gun_id via callback npc_on_choose_weapon.
10. Console & Debugging
10.1 Logging
lua
printf(fmt, ...)         – formatted print to console/log
printe(fmt, ...)          – error logging (adds custom static if DEV_DEBUG and option enabled)
printdbg(fmt, ...)         – only prints if DEV_DEBUG
abort(msg, ...)            – prints error and stack trace, then stops
callstack([c1], [to_str])  – prints Lua stack trace; c1 = only first caller, to_str = return string
10.2 Console Commands
lua
exec_console_cmd(cmd)
get_console_cmd(typ, name)  – 0=string, 1=bool, 2=float, else token
add_console_command(name, func) – registers a new console command (debug only)
11. INI File Extensions
11.1 Cached INI Reads
lua
ini_file.r_string_ex(ini, s, k, [def])
ini_file.r_bool_ex(ini, s, k, [def])
ini_file.r_float_ex(ini, s, k, [def])
ini_file.r_sec_ex(ini, s, k, [def])   – validates section existence
ini_file.r_line_ex(ini, s, k)           – returns key, value, comment
ini_file.r_string_to_condlist(ini, s, k, [def])
ini_file.r_list(ini, s, k, [def])
ini_file.r_mult(ini, s, k, ...)          – returns multiple parsed values
When USE_INI_MEMOIZE is true, results are cached for performance.

11.2 ini_file_ex Class
A wrapper that allows writing and caching:

lua
ini = ini_file_ex("path\\file.ltx", [advanced_mode])
ini:r_value(s, k, typ, def)    – typ: 0=string, 1=bool, 2=float
ini:w_value(s, k, val, [comment])
ini:save()
ini:collect_section(section)     – returns table of all key-value pairs in section
ini:get_sections([keytable])      – list of section names (or table with keys if true)
ini:remove_line(section, key)
11.3 Global Param Cache (SYS_GetParam)
lua
SYS_GetParam(typ, sec, param, [def]) – cached read from system_ini
typ: 0=string, 1=bool, 2=float
12. Important Constants
12.1 Bone IDs
lua
BoneID = {
    ["bip01_pelvis"]=2, ["bip01_l_thigh"]=3, ..., ["bip01_head"]=15,
    ["eye_left"]=16, ["eye_right"]=17, ...
}
12.2 Hit Types
lua
HitTypeID = {
    Burn=0, Shock=1, ChemicalBurn=2, Radiation=3, Telepatic=4,
    Wound=5, FireWound=6, Strike=7, Explosion=8, Wound_2=9, LightBurn=10
}
12.3 Booster Effect IDs
lua
BoosterID = {
    HpRestore=0, PowerRestore=1, RadiationRestore=2, BleedingRestore=3,
    MaxWeight=4, RadiationProtection=5, TelepaticProtection=6, ...
}
12.4 Inventory Slots (for scanning)
lua
SCANNED_SLOTS = { [1]=true, [2]=true, ..., [13]=true } – excludes script animation slot
12.5 Actor Movement Flags
See actor_move_states table: bitmask values for crouch, sprint, lookout, etc.

13. Mode Testing Functions
lua
IsAzazelMode()
IsHardcoreMode()
IsStoryMode()
IsSurvivalMode()
IsAgonyMode()
IsTimerMode()
IsCampfireMode()
IsWarfare()
IsTestMode()        – true on fake_start level
IsStoryPlayer()     – true if player faction is one of the main story factions
These read configuration and save data to determine which game modes are active.


1. Core Global Functions & Namespaces
These are your entry points to the engine.

Function / Namespace	Purpose
game_object()	Get any in-game object by ID (e.g., level.object_by_id(id)).
alife()	Access the ALife simulator – spawn/teleport/kill objects, manage online/offline.
system_ini(), game_ini()	Read/write config files.
level namespace	Manipulate current level: time, weather, camera effects, actor input, etc.
relation_registry	Manage faction goodwill and inter‑faction relations.
weather namespace	Control weather parameters (rain, wind, etc.) and pause weather cycles.
time_global(), time_continual()	Get global game time (paused/unpaused).
log()	Print debug messages to the console/log.
get_console():execute()	Run console commands from script.
2. The game_object Class (Most Important)
Every object in the game (actor, NPC, item, anomaly) inherits from game_object. These are the methods you'll use constantly.

General Object Info
Method	Description
id()	Unique object ID.
name(), section()	Object's script name and config section.
clsid()	Class ID (see clsid constants).
position(), level_vertex_id(), game_vertex_id()	Location data.
parent()	The object that holds this one (e.g., an inventory owner).
Type Checking
Method	Use
is_actor(), is_stalker(), is_monster(), is_weapon(), is_artefact(), etc.	Determine what kind of object you're dealing with.
Actor & NPC Specific
Method	Description
health, power, radiation, psy_health, bleeding, morale	Properties (read/write).
kill(actor), hit(hit)	Inflict damage or death.
give_info_portion(), has_info(), disable_info_portion()	Manage script info portions (used for quests, dialogue conditions).
rank(), character_rank(), character_reputation(), character_community()	NPC stats.
set_relation(), change_goodwill(), force_set_goodwill()	Modify interpersonal relations.
see(obj)	Check if NPC sees another object.
best_enemy(), set_enemy(), get_enemy()	Combat targeting.
memory_visible_objects(), memory_hit_objects(), memory_sound_objects()	Access NPC's memory (what they know).
wounded(), critically_wounded()	State checks.
set_mental_state(), set_body_state(), set_movement_type()	Control NPC behaviour (see MonsterSpace enums).
set_sight()	Make NPC look at a point or object.
set_dest_level_vertex_id(), set_dest_game_vertex_id()	Pathfinding.
set_patrol_path(), patrol()	Assign patrol routes.
set_home()	Define NPC's home area (for respawn, retreat).
Inventory & Items
Method	Description
active_item(), active_slot(), activate_slot()	Current weapon/item.
item_in_slot(), item_on_belt()	Access inventory slots.
move_to_ruck(), move_to_slot(), move_to_belt()	Rearrange inventory.
drop_item(), transfer_item(), give_money(), money()	Trade and loot.
iterate_inventory(), iterate_ruck(), iterate_belt()	Loop through items.
condition(), set_condition()	Item durability.
weight(), cost()	Basic item properties.
get_ammo_total(), get_ammo_in_magazine(), unload_magazine()	Weapon ammo.
install_upgrade(), has_upgrade()	Weapon/outfit upgrades.
weapon_addon_attach(), weapon_addon_detach()	Attach/detach scopes, silencers, grenade launchers.
Casting to Specific Classes
Use cast_ClassName() to access class-specific methods:

obj:cast_Actor() → returns CActor (player-specific functions like conditions()).

obj:cast_CustomOutfit() → get protection values, additional weight.

obj:cast_WeaponMagazined() → reload, fire mode, etc.

obj:cast_Artefact() → artefact properties.

3. Actor (Player) Specific – CActor
Obtain via db.actor or level.get_actor(). Extends game_object.

Method	Description
conditions()	Returns CActorCondition – access health, power, radiation, satiety, bleeding, boosts.
inventory_disabled(), set_inventory_disabled()	Block inventory opening.
get_actor_max_weight(), set_actor_max_weight()	Carry capacity.
get_actor_jump_speed(), set_actor_jump_speed()	Movement parameters.
enable_night_vision(), torch_enabled(), enable_torch()	Devices.
give_game_news()	Show PDA‑style message.
actor_look_at_point()	Force player's view direction.
set_actor_position()	Teleport actor (use with care).
4. ALife (Online/Offline Simulation) – alife_simulator
Accessed via alife(). Manages the persistent world.

Method	Description
object(id)	Get an object by ID, even if offline.
actor()	Get the actor object.
create(section, position, lvid, gvid)	Spawn a new object (item, NPC, anomaly) into the world.
release(obj, boolean)	Remove an object permanently.
kill_entity(obj, killer)	Kill an NPC or destroy an object.
teleport_object(id, gvid, lvid, position)	Move an object to another level/location.
switch_online(id, bool), switch_offline(id, bool)	Force object online/offline (advanced).
story_object(story_id)	Find an object by story ID (quest markers).
5. Inventory, Items, and Trade
Key classes: CInventoryItem, CWeapon, CCustomOutfit, CArtefact, CAmmo.

Important Functions	Description
game_object:transfer_item(item, target_owner)	Move an item between inventories.
game_object:buy_condition(), sell_condition()	Set trade filters (money range, faction).
game_object:buy_supplies(ini, section)	Use a trade profile from a config.
game_object:enable_trade(), disable_trade()	Toggle trading ability.
CWeapon:GetAmmoElapsed(), SetAmmoElapsed()	Magazine ammo.
CWeapon:GetSuitableAmmoTotal()	Total ammo for this weapon in inventory.
CWeapon:SwitchAmmoType()	Cycle ammo types.
CWeapon:IsZoomed(), GetZoomFactor()	Scope usage.
CCustomOutfit:GetHitTypeProtection()	Armor values.
CArtefact:GetAfRank()	Artefact level.
CArtefact:ActivateArtefact()	Use artefact's active ability.
CInventoryOwner:get_money(), give_money()	Money management.
6. Anomalies and Zones
Classes: CCustomZone, CTorridZone, CMosquitoBald, etc.

Method	Description
game_object:get_anomaly_radius(), set_anomaly_radius()	Change zone size.
game_object:set_anomaly_position(x,y,z)	Move zone.
game_object:enable_anomaly(), disable_anomaly()	Turn zone on/off.
game_object:get_anomaly_power(), set_anomaly_power()	Modify damage/effect strength.
zone:cast_CustomZone():set_restrictor_type()	Change zone behaviour (e.g., teleport, no weapons).
7. UI and Dialogs
Classes: CUIScriptWnd, CUIDialogWnd, CUIStatic, CUIWindow, CScriptXmlInit.

Important	Description
CScriptXmlInit:ParseFile() + Init...	Create UI elements from XML.
CUIDialogWnd:ShowDialog(), HideDialog()	Show/hide a window.
CUIGameCustom:AddCustomStatic(), RemoveCustomStatic()	Show on‑screen messages/icons.
game:translate_string()	Get localized text.
CUIStatic:SetText(), SetTexture()	Change displayed content.
CUIWindow:AddCallback()	Handle UI events (click, etc.).
8. Callbacks and Events
Use game_object:set_callback() to hook into engine events.

Callback ID (from callback class)	Triggered when...
callback.death	Object dies.
callback.hit	Object receives a hit.
callback.use_object	Actor uses an object.
callback.trade_start, trade_stop, trade_sell_buy_item	Trading events.
callback.actor_sleep	Actor goes to sleep.
callback.zone_enter, zone_exit	Actor enters/exits a zone.
callback.level_border_enter, level_border_exit	Cross level boundaries.
callback.on_item_drop, on_item_take	Item moved in/out of inventory.
callback.weapon_no_ammo	Weapon runs out of ammo.
Example:

lua
obj:set_callback(callback.hit, function(obj, hit)
    printf("Object %s was hit for %f damage", obj:name(), hit.power)
end)
9. Level Environment – level namespace
Function	Use
level.add_pp_effector(), remove_pp_effector()	Apply post‑process effects (e.g., blur, radiation vignette).
level.set_weather(), level.get_weather()	Change weather section.
level.set_time_factor()	Speed up/slow down time.
level.get_time_hours(), level.get_time_minutes(), level.change_game_time()	Read/modify in‑game time.
level.map_add_object_spot(), map_remove_object_spot()	Add/remove map markers.
level.rain_factor(), level.get_env_rads()	Environmental values (for detectors).
level.get_target_obj(), level.get_target_dist()	Get object currently under crosshair.
level.disable_input(), level.enable_input()	Block player controls (cutscenes).
level.iterate_nearest()	Find objects in radius.
10. AI & Combat
Low‑level AI manipulation via action_planner, action_base, etc. But for most mods, high‑level game_object methods are enough.

Method	Description
game_object:set_enemy()	Force NPC to attack a specific target.
game_object:set_mental_state(MonsterSpace.mental_state)	danger, panic, free.
game_object:set_body_state(MonsterSpace.body_state)	stand, crouch, lie.
game_object:set_movement_type(MonsterSpace.movement_type)	walk, run, steal.
game_object:set_sight(SightManager.look)	Make NPC look at a point/object.
game_object:set_patrol_path()	Assign patrol.
game_object:set_home()	Define home point and radius.
game_object:set_script()	Force NPC into scripted control (see bind_object).
11. Utilities
vector class
Essential for positions and directions.

lua
local pos = vector():set(10, 0, 10)
local dir = pos2:sub(pos1):normalize()
ini_file class
Read/write config files.

lua
local ini = ini_file("mydata.ltx")
local value = ini:r_string("section", "field")
time_global(), time_continual()
Get current game time in milliseconds. Use for timers.

game:get_game_time() → CTime
Returns structured time object (date, hour, minute, second). Can compare CTime objects.

script_light() / script_glow()
Create dynamic lights and glow effects.

12. Spawning and Object Creation
From alife():

lua
-- Spawn an item at actor's feet
local pos = db.actor:position()
local lvid = db.actor:level_vertex_id()
local gvid = db.actor:game_vertex_id()
local ammo = alife():create("ammo_9x19_fmj", pos, lvid, gvid)

-- Spawn a stalker
local npc = alife():create("sim_default_stalker", pos, lvid, gvid)

-- Teleport a stalker to another level
alife():teleport_object(npc:id(), new_gvid, new_lvid, new_pos)

------------------
-- server_entity_on_register
------------------
function server_entity_on_register(se_obj,type_name)
	local id = se_obj.id
	if (id == AC_ID) then
		story_objects.register(id,"actor")
	else
		story_objects.check_spawn_ini_for_story_id(se_obj)
	end
end

------------------
-- server_entity_on_unregister
------------------
-- good place to remove ids from persistent tables
function server_entity_on_unregister(se_obj,type_name)
	local id = se_obj.id
	local m_data = alife_storage_manager.get_state()
	if (m_data) then 
		if (m_data.se_object) then
			m_data.se_object[id] = nil
			--printf("$ alife_storage | cleaning server object [%s](%s) mdata", se_obj:name(), id)
		end
		if (m_data.game_object) then
			m_data.game_object[id] = nil
			--printf("$ alife_storage | cleaning game object [%s](%s) mdata", se_obj:name(), id)
		end
	end
	story_objects.unregister(id)
end

1. Global Instance
lua
SIMBOARD = get_sim_board()   -- returns the singleton simulation_board object
Use get_sim_board() to access the simulation board anywhere.

2. simulation_board Class
2.1 Internal Tables
Field	Description
self.smarts	Table of registered smart terrains: key = smart ID, value = {smrt = object, squads = {}, population = 0}
self.smarts_by_names	Lookup smart terrain by name (string) → server object
self.squads	(Seems unused)
self.tmp_assigned_squad	Temporary storage for squads assigned to smart terrains that are not yet initialized (waiting for init_smart).
self.start_position_filled	Flag to prevent duplicate initial squad spawning.
2.2 Core Methods
register_smart(obj)
Registers a smart terrain server object with the simulation board. Called when a smart terrain is created or loaded.

unregister_smart(obj)
Removes a smart terrain from the board.

init_smart(obj)
Called after a smart terrain is fully initialized. Moves any squads from tmp_assigned_squad into the smart's squads list via assign_squad_to_smart.

create_squad(spawn_smart, sq_id)
Creates a new squad server object using the given squad section (sq_id) at the location of spawn_smart.

Calls squad:create_npc(spawn_smart) to generate its members.

Assigns the squad to the smart via assign_squad_to_smart.

For each NPC, calls setup_squad_and_group and triggers callback squad_on_npc_creation.

Returns the squad server object.

create_squad_at_named_location(loc_name, squad_id)
Creates a squad at a predefined named location (from named_locations.ltx). Parses position and vertex IDs from the location entry and spawns the squad.

Similar to create_squad but without an initial smart assignment.

remove_squad(squad)
Removes a squad from the simulation:

Unassigns it from its current smart via assign_squad_to_smart(squad, nil).

Calls squad:remove_squad() to release all its NPCs.

assign_squad_to_smart(squad, smart_id)
Moves a squad to a different smart terrain.

If smart_id is nil, the squad is removed from its current smart.

If the target smart is not yet registered, the squad is stored in tmp_assigned_squad.

Updates the smart's squads table and population count (excluding squads with a target_smart).

Triggers callbacks squad_on_leave_smart and squad_on_enter_smart.

setup_squad_and_group(se_obj)
Sets the team/squad/group of an NPC based on its squad and the squad's assigned smart.

If the NPC has a squad and that squad is assigned to a smart, it sets the group to 1 (otherwise 0).

Calls change_team_squad_group (global function from _G.script).

fill_start_position()
Spawns initial squads according to simulation.ltx (or simulation_survival_mode.ltx if Survival Mode is active).

Reads sections (smart terrain names) and spawns the specified squads with count adjustments based on population factors (alife_stalker_pop, alife_mutant_pop).

Skips if already filled.

Triggers callback fill_start_position before spawning, allowing modders to override or add custom spawns.

get_smart_by_name(name)
Returns the smart terrain server object by its string name.

get_smart_population(smart)
Returns the current population count of a smart terrain (number of squads stationed there, excluding those with a target).

get_squad_target(squad)
Selects a target for a squad from the list of registered simulation objects (simulation_objects.object_registry).

Evaluates each potential target using se_target:evaluate_prior(squad) and checks se_target:target_precondition(squad).

Selects a target randomly from the top 5 priorities, with a 50% chance to pick the highest-priority target.

Returns the best target server object.

3. Global Precondition Functions (for target selection)
These functions are used by target objects (e.g., bases, territories, resources) to determine if a squad can target them. They are called from the target's target_precondition method.

lua
general_squad_precondition(squad, target)   -- Currently always returns false (override in your target)
general_base_precondition(squad, target)    -- Checks faction occupation; allows attack during early morning (3–6) if enemy
general_territory_precondition(squad, target) -- Handles mutant activity times (day/night predators) and faction checks
general_resource_precondition(squad, target) -- Returns true only between 10 PM and 6 AM (night time for resource gathering)
general_lair_precondition(squad, target)    -- Always true (monsters fill lairs first)
These are used by the built‑in target types (simulation_objects.script). Modders can write their own precondition functions and assign them to custom targets.

4. Important Constants & Lookups
is_squad_monster
lua
is_squad_monster = {
    ["monster_predatory_day"] = true,
    ["monster_predatory_night"] = true,
    ["monster_vegetarian"] = true,
    ["monster_zombied_day"] = true,
    ["monster_zombied_night"] = true,
    ["monster_special"] = true,
    ["monster"] = true,
    ["zoo_monster"] = true
}
Used in fill_start_position to apply the mutant population factor.

squad_community_by_behaviour
Maps squad behaviour strings (e.g., "stalker", "monster_predatory_day") to the actual community name (e.g., "stalker", "monster").
Used elsewhere (not directly in this file) to determine faction relations.

5. Callbacks Triggered
The simulation board fires the following script callbacks (via SendScriptCallback):

"squad_on_npc_creation" – after each NPC is created for a squad (arguments: squad server object, NPC server object, [spawn_smart]).

"squad_on_leave_smart" – when a squad is removed from a smart (arguments: squad, smart object).

"squad_on_enter_smart" – when a squad is assigned to a smart (arguments: squad, smart object).

"fill_start_position" – just before the initial squad spawn, allowing custom spawns.

Modders can register functions to these callbacks using RegisterScriptCallback.

6. Integration with Other Systems
simulation_objects.script – provides the target registry (object_registry) and available_by_id table used by get_squad_target. Each target object must implement evaluate_prior(squad) and target_precondition(squad).

alife() – used to create squads and access server objects.

smart_terrain.script – the smart_terrain class provides methods like smart_terrain_squad_count (used to calculate population).

game_relations.script – for faction checks (is_factions_enemies, is_valid).

utils_data.script – for reading INI values (e.g., read_from_ini in create_squad_at_named_location).

Named locations – defined in named_locations.ltx; used by create_squad_at_named_location.

7. Usage Examples for Modders
Spawn a custom squad at a smart terrain
lua
local smart = SIMBOARD:get_smart_by_name("smart_terrain_100")
if smart then
    local squad = SIMBOARD:create_squad(smart, "squad_section_name")
end
Move a squad to a different smart
lua
SIMBOARD:assign_squad_to_smart(squad, new_smart_id)
Get current population of a smart
lua
local pop = SIMBOARD:get_smart_population(smart_object)
Hook into squad creation to add custom properties
lua
RegisterScriptCallback("squad_on_npc_creation", function(squad, npc, spawn_smart)
    -- e.g., set custom inventory or story ID
    se_save_var(npc.id, npc:name(), "my_custom_flag", true)
end)
Define a new target type
In simulation_objects.script, create a new target class that implements:

evaluate_prior(squad)

target_precondition(squad) – can call one of the general precondition functions or a custom one.



1.1 General Tips
Never store userdata (engine objects) in global scope – keep object IDs and retrieve via level.object_by_id(id) or db.storage[id].object. Otherwise objects cannot be destroyed, leading to crashes.

Avoid reserved names like task, level, weather as variables.

Check _G to avoid overriding global functions.

1.2 Logging & Debugging
lua
printf(fmt, ...)          -- formatted print to console/log
printe(fmt, ...)           -- error logging (adds debug static if DEV_DEBUG enabled)
printdbg(fmt, ...)          -- only prints if DEV_DEBUG
abort(msg, ...)              -- prints error, stack trace, then stops
callstack([c1], [to_str])    -- prints Lua stack trace; c1 = only first caller, to_str = return string
exec_console_cmd(cmd)        -- execute console command
get_console_cmd(typ, name)   -- read console variable (0=string,1=bool,2=float)
add_console_command(name, func) -- register new console command (debug only)
1.3 Math Functions
lua
round(x), round_idp(num, idp), round_100(x)
clamp(val, min, max)
normalize(val, min, max), normalize_100(val, min, max)
random_choice(...)              -- picks one argument randomly
random_number([min, max])        -- integer random
random_float(min, max)            -- float random
yaw(v1, v2), yaw_degree(v1, v2)   -- angle between 2D vectors (XZ)
yaw_degree3d(v1, v2)               -- 3D angle
vector_cross(v1, v2)                -- cross product
vector_rotate_y(v, angle_deg)       -- rotate around Y axis
distance_2d(a, b), distance_2d_sqr(a, b)
1.4 String Functions
lua
trim(s)
strformat(fmt, ...)                   -- custom formatter (replaces %s with tostring)
str_explode(str, sep, [plain])          → table of strings
parse_list(ini, key, val, [convert])    – parses comma-separated list from ini
parse_names(s), parse_nums(s)            – extract words/numbers
parse_key_value(s)                        – “key=value” pairs
starts_with(str, start)
has_translation(string_id)                 – true if game.translate_string returns different string
get_param_string(src_string, obj)          – replaces $script_id$ with object’s script ID
1.5 Table Functions
lua
is_empty(t), is_not_empty(t)
empty_table(t), iempty_table(t)       – clear a table
size_table(t)
random_key_table(t)                     → random key from table
copy_table(dest, src), dup_table(src)    – shallow/deep copy
shuffle_table(t)
invert_table(t)                           – swaps keys and values
t2k_table(t) / k2t_table(t)                – convert between list and set
print_table(table, [subs])                  – debug print
store_table(table, [subs])                   – print in a loadable format
spairs(t, [order])                            – sorted iterator
1.6 Vector Helpers
lua
VEC_ZERO, VEC_X, VEC_Y, VEC_Z        – common vectors
vec_sub(a,b), vec_add(a,b), vec_set(vec) – returns new vector
vec_to_str(vector)                      – converts vector to "[x:y:z]" string
2. Core Systems
2.1 Callback System
lua
RegisterScriptCallback(name, func_or_userdata)
UnregisterScriptCallback(name, func_or_userdata)
SendScriptCallback(name, ...)       -- triggers all registered callbacks and axr_main[name]
AddScriptCallback(name)               -- adds a callback to the engine's list
Many engine events (hits, item use, etc.) trigger these callbacks – see Section 10.

2.2 Delayed Event Queue
Run functions after a delay or repeatedly until they return true. Does not persist through saves.

lua
CreateTimeEvent(ev_id, act_id, timer_sec, func, ...)
RemoveTimeEvent(ev_id, act_id)
ResetTimeEvent(ev_id, act_id, new_timer_sec)
ProcessEventQueue([force])          -- called in update loops
2.3 Persistent Data Storage (pstor)
db.storage – holds online object data. Contains:

object – game object (userdata)

pstor – table for arbitrary Lua data (saved via marshal)

pstor_ctime – for game.CTime values

disable_input_time, disable_input_idle – actor input blocking

Helpers (for online objects):

lua
save_var(obj, varname, value)
load_var(obj, varname, default)
save_ctime(obj, varname, ctime_value)
load_ctime(obj, varname)
Server-side storage (offline objects, via alife_storage_manager):

lua
se_save_var(id, name, varname, value)
se_load_var(id, name, varname)
2.4 Server Object Management (ALife)
lua
alife_object(id)                     → server object
alife_create(sec, pos, lvid, gvid, [id], [state])
alife_create(sec, obj, [with_id], [state])   -- using another object's properties
alife_create_item(section, owner_obj_or_table, [properties_table])
alife_process_item(section, item_id, properties_table)   -- apply properties to online item
alife_release(se_obj, [msg])          -- release server object (handles squads/NPCs specially)
alife_release_id(id, [msg])
alife_clone_weapon(se_obj, [new_section], [parent_id])
alife_character_community(se_obj)     → community string
alife_on_limit() → bool               -- true if near object limit (64000)
create_ammo(section, pos, lvi, gvi, parent_id, count) → table of ammo objects
2.5 Story ID Handlers
lua
get_story_se_object(story_id)       → server object
get_story_se_item(story_id)         → server item (special handling)
get_story_object(story_id)           → game object (if online)
get_story_squad(story_id)            → alias for get_story_se_object
get_object_story_id(obj_id)           → story ID of given object ID
get_story_object_id(story_id)         → object ID
unregister_story_object_by_id(obj_id) – removes story association
level_object_by_sid(story_id)         → game object on current level
id_by_sid(story_id)                   → object ID
2.6 Global Event Storage (_EVENT)
A global table for ad‑hoc data sharing between scripts.

lua
SetEvent(key, value)                 – store value under key
SetEvent(key, subkey, value)          – store in a nested table
GetEvent(key)                          – retrieve value
GetEvent(key, subkey)                  – retrieve nested value
2.7 Game Mode Testing
lua
IsAzazelMode(), IsHardcoreMode(), IsStoryMode(), IsSurvivalMode()
IsAgonyMode(), IsTimerMode(), IsCampfireMode(), IsWarfare()
IsTestMode()        – true on fake_start level
IsStoryPlayer()     – true if player faction is one of the main story factions
3. Player & NPC Specifics
3.1 Actor Functions
lua
give_object_to_actor(section, [count])     – spawn item into actor's inventory
get_actor_true_community()                  → string (e.g., "stalker", ignoring disguise)
set_actor_true_community(new_comm, now)     – changes player’s faction
get_player_level_id()                       → level ID where actor currently is
IsMoveState(state_name, [compare_state])    – check actor movement state (see 7.5)
set_inactivate_input_time(delta_sec)        – disable input for delta seconds
3.2 NPC Identification & Properties
lua
IsStalker([obj], [clsid]), IsMonster(...), IsAnomaly(...), IsTrader(...)
IsCar(...), IsHelicopter(...), IsInvbox(...)
IsWounded(obj)                 – checks if NPC is critically wounded and in special state
character_community(obj)        → community string
get_object_community(obj)        – works for both game and server objects
get_object_squad(obj)             → server squad object (or nil)
npc_in_actor_frustrum(npc)         → bool (if NPC is within ~35° of camera direction)
change_team_squad_group(se_obj, team, squad, group) – changes both online and offline
get_speaker([safe], [all])          – returns NPC currently talking to actor
3.3 Squad Management
Creating a squad: Use alife_create() with a squad section, then call squad:create_npc(spawn_smart).

Releasing a squad: alife_release(squad) calls squad:remove_squad().

Assigning to smart: Use SIMBOARD:assign_squad_to_smart(squad, smart_id) (see Section 8).

Squad members iteration: for k in squad:squad_members() do ... end (k is a table with id, object if online).

3.4 Wounded State
IsWounded(obj) checks if an NPC is in the special wounded state. Additional variables:

load_var(obj, "wounded_state") – state string

load_var(obj, "wounded_fight") – if "true", NPC will fight while wounded

4. Level & Teleportation
4.1 Level Changing
lua
ChangeLevel(pos, lvid, gvid, angle, [anim])   – triggers level change (with optional fade)
change_level_now(pos, lvid, gvid, angle)       – immediate change (sends network packet)
JumpToLevel(new_level_name)                     – teleports actor to first suitable game vertex
4.2 Teleportation (requires OpenXRay)
lua
TeleportObject(id, pos, lvid, gvid)            – teleports a single object
TeleportSquad(squad, pos, lvid, gvid)           – teleports squad leader and all members
4.3 Level Handlers
lua
AddUniqueCall(functor)       – adds a level call that runs until functor returns true
RemoveUniqueCall(functor)
level_changing()              → true if a level transition is in progress
in_time_interval(val1, val2)  – checks current hour against a day‑night interval
5. GUI System
5.1 UI Registration
lua
Register_UI(name, path, instance)      – marks UI as active; blocks keybinds unless in _GUIs_keyfree
Unregister_UI(name)                     – removes UI, restores keybinds if last
Check_UI([name])                         – true if any (or specific) UI is open
Overlapped_UI(name)                       – true if more than one UI is open (excluding dialog)
CloseAll_UI()                              – closes all registered UIs via their Close() method
DestroyAll_UI()                             – completely removes UI references
5.2 HUD Control
lua
show_indicators(state)                    – enable/disable HUD indicators
main_hud_shown()                           → true if indicators on, inventory/PDA closed, no zoom
hide_hud_all()                              – closes inventory, PDA, and all registered UIs
hide_hud_inventory() / hide_hud_pda()       – specific closers
show_all_ui(show)                           – hides/shows all UI elements (including indicators)
GetActorMenu() → actor_menu object          – access inventory GUI
6. INI File Extensions
6.1 Cached INI Reads
When USE_INI_MEMOIZE = true, results are cached for performance.

lua
ini_file.r_string_ex(ini, s, k, [def])
ini_file.r_bool_ex(ini, s, k, [def])
ini_file.r_float_ex(ini, s, k, [def])
ini_file.r_sec_ex(ini, s, k, [def])   – validates section existence
ini_file.r_line_ex(ini, s, k)           – returns key, value, comment
ini_file.r_string_to_condlist(ini, s, k, [def])
ini_file.r_list(ini, s, k, [def])
ini_file.r_mult(ini, s, k, ...)          – returns multiple parsed values
6.2 ini_file_ex Class
A writable INI wrapper with caching.

lua
ini = ini_file_ex("path\\file.ltx", [advanced_mode])
ini:r_value(s, k, typ, def)    – typ: 0=string, 1=bool, 2=float
ini:w_value(s, k, val, [comment])
ini:save()
ini:collect_section(section)     – returns table of all key-value pairs
ini:get_sections([keytable])      – list of section names
ini:remove_line(section, key)
6.3 Global Param Cache (SYS_GetParam)
lua
SYS_GetParam(typ, sec, param, [def]) – cached read from system_ini
typ: 0=string, 1=bool, 2=float
7. Constants & Enumerations
7.1 Bone IDs
lua
BoneID = {
    ["bip01_pelvis"]=2, ["bip01_l_thigh"]=3, ..., ["bip01_head"]=15,
    ["eye_left"]=16, ["eye_right"]=17, ...
}
7.2 Hit Types
lua
HitTypeID = {
    Burn=0, Shock=1, ChemicalBurn=2, Radiation=3, Telepatic=4,
    Wound=5, FireWound=6, Strike=7, Explosion=8, Wound_2=9, LightBurn=10
}
7.3 Booster IDs
lua
BoosterID = {
    HpRestore=0, PowerRestore=1, RadiationRestore=2, BleedingRestore=3,
    MaxWeight=4, RadiationProtection=5, TelepaticProtection=6, ...
}
7.4 Inventory Slots (for scanning)
lua
SCANNED_SLOTS = { [1]=true, [2]=true, ..., [13]=true } – excludes script animation slot
7.5 Actor Movement States
lua
actor_move_states = {
    mcFwd=1, mcBack=2, mcLStrafe=4, mcRStrafe=8,
    mcCrouch=16, mcAccel=32, mcTurn=64, mcJump=128,
    mcFall=256, mcLanding=512, mcLanding2=1024, mcClimb=2048,
    mcSprint=4096, mcLLookout=8192, mcRLookout=16384,
    mcAnyMove=15, mcAnyAction=1935, mcAnyState=6192, mcLookout=24576
}
8. Simulation Board (sim_board.script)
8.1 Overview
The simulation board (SIMBOARD) manages smart terrains and squads. It handles squad creation, assignment to smarts, and target selection for A-Life.

Singleton access:

lua
SIMBOARD = get_sim_board()
8.2 Core Methods
lua
:register_smart(obj)               – register a smart terrain server object
:unregister_smart(obj)              – remove from board
:init_smart(obj)                     – called after smart is initialized; assigns pending squads
:create_squad(spawn_smart, sq_id)    – create a squad at smart using squad section
:create_squad_at_named_location(loc_name, squad_id) – spawn squad at named location
:remove_squad(squad)                  – remove squad from simulation
:assign_squad_to_smart(squad, smart_id) – move squad to a smart (nil to remove)
:setup_squad_and_group(se_obj)        – set team/squad/group for an NPC based on its squad
:fill_start_position()                 – spawn initial squads from simulation.ltx
:get_smart_by_name(name)               → smart server object
:get_smart_population(smart)           → number of squads at smart (excluding targets)
:get_squad_target(squad)                → best target object for squad
8.3 Precondition Functions
Used by target objects to determine if a squad can target them.

lua
general_squad_precondition(squad, target)       – always false (override)
general_base_precondition(squad, target)        – faction checks; allows attack at 3-6 if enemy
general_territory_precondition(squad, target)   – handles mutant day/night; faction checks at 9-15
general_resource_precondition(squad, target)    – true only between 22:00 and 6:00
general_lair_precondition(squad, target)        – always true (monsters fill lairs)
8.4 Callbacks Triggered
lua
SendScriptCallback("squad_on_npc_creation", squad, npc, [spawn_smart])
SendScriptCallback("squad_on_leave_smart", squad, smart)
SendScriptCallback("squad_on_enter_smart", squad, smart)
SendScriptCallback("fill_start_position")
9. Module Manager (modules.script)
9.1 Scheme Types
lua
stype_stalker    = 0   – NPC schemes
stype_mobile     = 1   – monster schemes
stype_item       = 2   – physical object schemes
stype_heli       = 3   – helicopter schemes
stype_restrictor = 4   – restrictor schemes
stype_trader     = 5   – trader schemes
9.2 Scheme Registration
Schemes are registered with LoadScheme(filename, scheme_name, ...stypes).
Example: LoadScheme("xr_walker", "walker", stype_stalker)

_G.schemes[scheme_name] = filename

_G.schemes_by_stype[stype][scheme_name] = true

9.3 Generic Scheme Management
Generic schemes are automatically enabled/disabled for NPCs via:

lua
enable_generic_schemes(npc, ini, section, stype)
reset_generic_schemes(npc, scheme, section)
disable_generic_schemes(npc, stype)
Generic schemes must implement:

setup_generic_scheme(npc, ini, scheme, section, stype, temp) – create actions/evaluators.

configure_actions(npc, ini, scheme, section, stype, temp) – add preconditions/effects (optional).

reset_generic_scheme(npc, scheme, section, stype, storage) – cleanup.

disable_generic_scheme(npc, scheme, stype) – disable.

9.4 Lifecycle Functions
lua
on_game_start() – calls each scheme's `on_game_start()` if defined.
add_common_precondition(scheme, action) – adds common preconditions (meet, wounded, etc.) to an action.
10. Engine Hooks / Callbacks
These functions are called by the engine and often trigger script callbacks. Modders can intercept them by registering to the corresponding callback name.

10.1 Actor & NPC Hit Callbacks
Hook	Callback Triggered
CActor__BeforeHitCallback(actor, hit, bone_id)	actor_on_before_hit
CAI_Stalker__BeforeHitCallback(npc, hit, bone_id)	npc_on_before_hit
CBaseMonster__BeforeHitCallback(monster, hit, bone_id)	monster_on_before_hit
CActor__HitArtefactsOnBelt(hit_table, hit_power, hit_type)	actor_on_before_hit_belt (modifies hit table)
CActor__FootstepCallback(material, power, hud_view)	actor_on_footstep (return false to mute)
10.2 Weapon & Inventory Callbacks
Hook	Callback
CInventory__eat(item)	actor_on_item_before_use (return false to prevent use)
CMissile__PutNextToSlot(itm)	actor_on_before_throwable_select (can replace item)
CActor_Fire()	actor_on_weapon_before_fire (return false to prevent firing)
CBulletManager__ObjectHit(section, obj, pos, dir, mtl, speed, wpn_id)	bullet_on_hit
CAI_Stalker__GetWeaponAccuracy(obj, wpn, dispersion, body_state, move_type)	npc_shot_dispersion (modifies dispersion)
CBurer_BeforeWeaponDropCallback(monster, wpn)	burer_on_before_weapon_drop (return false to prevent drop)
10.3 Movement & Animation Callbacks
Hook	Callback
player_hud__OnMovementChanged(cmd)	actor_on_movement_changed
CHudItem__OnMotionMark(state, mark)	actor_on_hud_animation_mark
CHudItem__PlayHUDMotion(anm_table, obj)	actor_on_hud_animation_play (modifies animation table)
CActor_on_jump()	actor_on_jump
CActor_on_land(landing_speed)	actor_on_land
10.4 Other Callbacks
Hook	Callback
CCustomZone_BeforeActivateCallback(zone, obj)	anomaly_on_before_activate (return false to ignore object)
CZone_Touch(obj)	actor_on_feeling_anomaly (return false to prevent feeling)
CBolt__State(id)	– (if returns true and limited bolts on, bolt not released)
CHUDManager_OnScreenResolutionChanged()	on_screen_resolution_changed (destroys all UI)
CALifeUpdateManager__on_before_change_level(packet)	– (can modify level change packet; also handles attached vehicle)
update_best_weapon(npc, cur_wpn)	npc_on_choose_weapon (can override by setting flags.gun_id)



########
utils
########

UICellItem – Represents a single item in the UI, handling its icon, attachments, condition bar, stack counter, upgrade indicator, and coloring.

UICellContainer – Manages a grid of UICellItems, supporting scrolling, drag‑and‑drop, hover effects, selection, and callbacks.

UIInfoItem / UIInfoUpgr – Displays detailed information about an item or upgrade, including stats, comparison with equipped items, and ammo types.

UICellProperties – A context menu (right‑click) that can show custom actions for an item.

UIHint – A simple tooltip window.

All UI elements are built on the engine’s XML‑based UI system and use utility functions from utils_xml and utils_item.

Constants and Utility Tables
Color Definitions (clr_list, clr_list_hl)
Colors are stored as ARGB values for easy reuse. Example:

lua
local clr_list = {
    ["def"]   = GetARGB(255, 255, 255, 255),   -- normal
    ["red"]   = GetARGB(255, 255, 50, 50),     -- negative
    ["green"] = GetARGB(255, 100, 255, 150),   -- positive
    -- ... etc.
}
Use them to colorize icons, text, or highlights.

Progress Bar Profiles (bar_list)
Defines the look of condition/power/usage bars:

lua
local bar_list = {
    ["condition_progess_bar"] = {
        min  = {255,196,18,18,0},
        mid  = {255,255,255,118,0.5},
        max  = {255,107,207,119,1},
        background = true
    },
    ["power_progess_bar"] = { def = GetARGB(255,86,196,209), background = true },
    -- ...
}
Each entry can define a solid color (def) or a gradient via min/mid/max. background toggles the bar’s background visibility.

Sorting Functions
Several sorters are provided to arrange items in a container:

sort_by_size – sort by inventory grid dimensions.

sort_by_kind – sort by item kind (defined in item_kind_order in the system ini).

sort_by_sizekind – combine size and kind (default).

sort_by_props – for items of the same section: ammo count > upgraded > condition.

These are used in UICellContainer:Reinit() and can be overridden.

Stats Tables (stats_table)
stats_table defines which stats are shown for each item type (weapon, outfit, artefact, booster, backpack). Each stat entry includes:

index – display order.

typ – value type (float, etc.).

name – translation string.

icon_p / icon_n – texture for positive/negative values.

track – if true, render as a progress bar; else as text.

magnitude – multiplier for the raw value.

unit – translation string for the unit (e.g., "st_kg").

compare – whether comparison with equipped items is allowed.

sign – show + for positive values.

show_always – show even if zero.

value_functor – optional function to compute the value (e.g., {"utils_ui","prop_accuracy"}).

Example for a weapon stat:

lua
stats_table["weapon"]["accuracy"] = {
    index   = 1,
    typ     = "float",
    name    = "ui_inv_accuracy",
    icon_p  = "ui_wp_prop_tochnost",
    track   = true,
    magnitude = 1,
    value_functor = {"utils_ui","prop_accuracy"}
}
Modders can extend these tables or add new types by calling add_stats_table(k1, k2, v).

Class UICellItem
This class encapsulates a single item cell. It handles:

Icon, shadow, and layered attachments (scope, silencer, etc.).

Condition bar, stack counter, upgrade indicator.

Custom text overlay.

Highlighting and coloring.

Key Methods
__init(container, st, indx, manual) – Creates a cell. manual means the cell is placed at a fixed position, not auto‑grid.

Set(obj, area) – Places an item (obj or section) into the cell. Calculates its position based on inventory grid coordinates.

Update(obj) – Refreshes the cell (condition bar, attachments, etc.). Called automatically after changes.

AddChild(obj) – Stacks another item onto this cell (if allowed).

PopChild(obj) – Removes one stacked item.

Reset() – Clears the cell and frees its grid area.

Colorize(clr_id) – Applies a predefined color to all icon layers.

Highlight(state, clr_id, main_clr) – Shows/highlights the cell background.

Note: When stacking is enabled (container.stack_all), the cell shows a counter and hides the condition bar/upgrade indicator for stacked items.

Class UICellContainer
Manages a grid of cells, handles user interaction, and provides callbacks to the owner.

Key Properties
grid_size, grid_line – dimensions of each grid cell (default 41 px, 2 px line).

sort_method – sorting function to use.

showcase – if true, cells are filled by section name (no real objects).

stack_all – allow stacking items of the same section.

disable_drag, disable_info, disable_stack – toggles features.

trade_profile – if set, enables trade‑mode coloring and cost display.

Important Methods
Reinit(t, tf) – Clears and rebuilds the container from a table t (either objects or section names). Optional tf provides flags for each entry.

AddItem(obj, sec, info) – Adds an item, finds a free grid cell, and creates a UICellItem. Returns the cell index.

AddItemManual(obj, sec, indx) – Adds an item at a pre‑defined index (used for fixed slots).

RemoveItem(obj, sec) – Removes an item; if stacked, removes one child.

TransferItem(cont_to, obj, sec) – Moves an item to another container (used in drag‑drop).

GetCell_Focused() – Returns the cell under the mouse.

On_Drag(idx, tg, set) – Handles drag‑start and drag‑end. Creates a floating icon.

Callbacks: On_CC_Add, On_CC_Remove, On_CC_DragDrop, On_CC_Hover, On_CC_Mouse1, On_CC_Mouse1_DB, On_CC_Mouse2 – these are invoked on the owner object if defined.

Grid Management
The container maintains a 2D grid of booleans (grid[row][col]) indicating occupied cells. It grows automatically when needed. FindFreeCell searches for a contiguous block of the required size.

Scrolling
Scrollable containers are built on CUIScrollView. The container calculates the required height based on the number of rows and adjusts the scroll range. It also supports a small scroll pad for mouse dragging.

Info Boxes
UIInfoItem
Displays item details: name, weight, cost (if in trade mode), description, and a list of stats. Stats are rendered either as progress bars or text, with optional comparison against an item already equipped in the same slot.

Update(obj, sec, flags) – Called each frame when hovering over an item. flags can contain value_str (cost) and note_str (trade restriction message).

Comparison works by checking the slot of the hovered item and retrieving the item currently in that slot.

UIInfoUpgr
Shows upgrade information: name, cost, description, prerequisites, and a list of properties (icons + text). The prerequisite line is colored green/red based on whether it’s already installed.

Both classes respect a delay before showing, to avoid flickering.

Context Menu: UICellProperties
A subclass of CUIScriptWnd that displays a list of actions. The owner must implement the functions that correspond to each menu item.

Reset(pos_override, action_list, name_list, params_list) – Builds the menu.
action_list is a table of function names (strings), name_list contains translation strings for display, and params_list (optional) holds parameters to pass.

Double‑clicking an item calls the corresponding function on the owner.

The menu auto‑closes when clicking outside.

Utility Functions
The script includes several small helpers:

prop_accuracy, prop_handling, prop_damage, prop_rpm, prop_condition – used in value_functor to compute weapon stats.

sync_cursor, sync_element, lerp_color – from utils_xml.

has_upgrades, get_item_trade_status, get_item_cost – from utils_item.

They are already used internally and can be reused by modders.

News is sorted by faction channels (e.g., "stalker", "dolg", "monolith"). Each channel has its own queue, and messages are popped one per tick to avoid spam.

lua
self.queue = {
    ["general"] = {},
    ["stalker"] = {},
    ["monolith"] = {},
    ...
}

-- Pushing a message
function DynamicNewsManager:PushToChannel(name, t)
    if self.counter > self.max_cnt then return end
    table.insert(self.queue[name], 1, t)   -- insert at front
    self.counter = self.counter + 1
end
Takeaway: Use a queue with a max cap to control message frequency.

2.2. Timed Events for Periodic News
The manager creates several repeating timers with different intervals, each triggering a specific news category (special, task, random, companion). Intervals are configurable via UI options.

lua
CreateTimeEvent("DynamicNewsManager","TickSpecial",10,self.TickSpecial,self)
-- Inside TickSpecial, the timer is reset with a random delay based on user setting
ResetTimeEvent("DynamicNewsManager","TickSpecial",math.random(cycle_TickSpecial + 1, cycle_TickSpecial*2 + 1))
Takeaway: Use CreateTimeEvent and ResetTimeEvent to schedule recurring checks. Randomize intervals to avoid predictability.

2.3. Callback Registration and Handling
The manager registers itself as a callback receiver for game events (death, looting) and handles them via methods.

lua
RegisterScriptCallback("npc_on_death_callback",self)
function DynamicNewsManager:npc_on_death_callback(victim,who)
    -- process death
end
Takeaway: You can pass a table (self) as the callback target, and the engine will call the corresponding method if it exists. This is cleaner than global functions.

2.4. Speaker Selection Logic
Finding a suitable NPC to "speak" a news message is crucial. The script provides multiple methods:

FindSpeaker: finds an online stalker not involved in the event, optionally from the same faction, not in combat, etc.

FindSpeakerWithEnemy: finds someone currently in combat (for SOS messages).

FindSpeakerRandom: picks any random online stalker.

FindSpeakerAnywhere: finds offline stalkers via the alife() iterator (for offline squad reports).

These functions filter by community, combat state, distance, and story status.

Takeaway: Build flexible speaker-finding functions that accept filters (faction, combat, visibility) to reuse across different news types.

2.5. String Building with Translation and Formatting
The script extensively uses translation keys and utils_data.parse_string_keys to replace placeholders like $name, $where, $clr_1. It also collects arrays of translated sentences (e.g., st_dyn_news_builder_hear_1, _2, ...) for random variation.

lua
local tbl = utils_data.collect_translations("st_dyn_news_weather_clear_", true)
if tbl then msg = tbl[math.random(#tbl)] end
Takeaway: Store multiple variations of a sentence with numbered keys (e.g., st_mynews_clear_1, _2) and use a helper to load them into a table for random selection.

2.6. Context-Aware Messages
Many news functions check the situation:

Weapon class (e.g., ignore knife kills)

Whether killer and victim are from same faction (to avoid reporting friendly fire as news)

If the community is "mysterious" (ISG, greh) – hide their identity

If the surge is active – suppress news and clear queue

Takeaway: Always add sanity checks to avoid inappropriate or repetitive messages.

2.7. Integration with Other Systems
Surge/Emission: The NewsToggle() function disables news during a blowout and sends a "communication lost/restored" message.

Story Mode: Special news for storylines (Living Legend, Mortal Sin) are triggered via info portions.

Bounties: Uses sim_squad_bounty to check for nearby bounty hunters.

Companions: The TickCompanion news only triggers if companions are present and not in combat.

Takeaway: Hook into existing game systems (surge_manager, axr_task_manager, faction_expansions) to make news feel integrated.

3. Important Functions & Snippets
3.1. PushToChannel – Queue a message
lua
function DynamicNewsManager:PushToChannel(name, t)
    if not enable_news or self:NewsToggle() or not item_device.is_pda_charged(true) or self.counter > self.max_cnt then
        return false
    end
    -- Replace any stray translation keys in the message
    if t.Mg then
        for s in string.gmatch(t.Mg, "(st_dyn_news_ch_[%w%d_]*)") do
            t.Mg = string.gsub(t.Mg, s, game.translate_string(s))
        end
    end
    table.insert(self.queue[name], 1, t)
    self.counter = self.counter + 1
end
3.2. FindSpeaker – Complex filter example
lua
function DynamicNewsManager:FindSpeaker(victim, who, same_as_victim, same_as_who, not_in_combat, can_see)
    local comm = character_community(victim)
    local who_id = who:id()
    local t = {}
    for i=1, #db.OnlineStalkers do
        local st = db.storage[db.OnlineStalkers[i]]
        local npc = st and st.object
        if npc and IsStalker(npc) and npc:alive() and not get_object_story_id(db.OnlineStalkers[i]) then
            if (not_in_combat == nil) or (not_in_combat and not npc:best_enemy()) then
                local comm_sender = npc:character_community()
                if (same_as_victim == nil or (same_as_victim and comm_sender == comm) or (not same_as_victim and comm_sender ~= comm)) and
                   (same_as_who == nil or (same_as_who and comm_sender == character_community(who)) or (not same_as_who and comm_sender ~= character_community(who))) then
                    if self.channel_status[comm_sender] then
                        if (can_see == nil) or (can_see and npc:see(victim)) then
                            t[#t+1] = npc
                        end
                    end
                end
            end
        end
    end
    return (#t > 0) and t[math.random(#t)] or nil
end
3.3. NewsToggle – Dynamic disabling during events
lua
function DynamicNewsManager:NewsToggle()
    if device():is_paused() then return true end
    if dynamic_news_helper.IsInvalidMap(level.name()) then return true end

    -- Welcome message once
    if not has_alife_info("trx_dynamic_news_welcome_to_network") then
        self:WelcomeToNetwork()
        db.actor:give_info_portion("trx_dynamic_news_welcome_to_network")
        return true
    end

    local news_state = true
    if xr_conditions.surge_started() then
        news_state = false
        -- Clear queues
        for _,messages in pairs(self.queue) do
            for i=#messages,1,-1 do messages[i]=nil end
        end
    end

    -- Detect surge start/end and send notifications
    if self.surge_shift and xr_conditions.surge_started() then
        self.surge_shift = false
    elseif not self.surge_shift and xr_conditions.surge_complete() then
        self.surge_shift = true
        self:GossipEmissionEnd(self.surge_type)
    end

    -- Notify when communication is lost/restored
    if news_state ~= self.news_toggle then
        self.news_toggle = news_state
        local msg = game.translate_string(news_state and "st_dyn_news_spc_commu_on" or "st_dyn_news_spc_commu_off")
        dynamic_news_helper.send_tip(msg, game.translate_string("st_dyn_news_sender_com_centre"), 5, msg_duration, "communication", news_state and "welcome" or "communication_lost", "gr")
    end
    return not news_state
end
3.4. Death Callback – Example of conditional news generation
lua
function DynamicNewsManager:npc_on_death_callback(victim, who)
    if not db.actor or not victim then return end
    local comm = character_community(victim)
    if not self.channel_status[comm] then return end
    if self:IsUnknownCommunity(who) then return end

    if not who or not who.clsid then
        if shw_death_generic and surge_manager.is_killing_all() then
            self:DeathBySurge(victim, who, comm)
        end
        return
    end

    -- Death report (obituary style)
    if shw_death_report and self.spammer.show_about_death_response == 1 then
        if IsStalker(who) then self:ReportDeathByStalker(victim, who)
        elseif IsMonster(who) then self:ReportDeathByMutant(victim, who) end
    end

    -- Regular death news with anti-spam
    local say = false
    if shw_death_stalker and self.spammer.show_about_death == 0 then
        if IsStalker(who) then
            say = self:SOSDeathByStalker(victim, who, comm)
            if not say then
                -- choose between direct death report or gossip from a witness
                if math.random(1,3) == 1 then
                    self:DeathByStalker(victim, who, comm)
                else
                    local sender = self:FindSpeaker(victim, who, false, nil, true)
                    if sender and sender:see(victim) then
                        self:SeenDeathOfStalker(sender, victim, who, comm)
                    else
                        self:GossipDeathByStalker(sender, victim, who)
                    end
                end
            end
        elseif IsMonster(who) then
            -- similar handling for monster kills
        end
    end

    -- Update spam counters
    if say then
        self.spammer.show_about_death = (self.spammer.show_about_death + 1) % 4
    end
    self.spammer.show_about_death_response = (self.spammer.show_about_death_response + 1) % 3
end
3.5. Utility: Collect Translations
The script uses a helper utils_data.collect_translations to gather all strings with a given prefix (e.g., st_dyn_news_weather_clear_) into a table. This is a powerful pattern for random messages.

lua
-- Example implementation (not in script but assumed)
function utils_data.collect_translations(prefix, numbered_only)
    local tbl = {}
    local i = 1
    while true do
        local key = prefix .. i
        local str = game.translate_string(key)
        if str == key then break end  -- translation missing
        tbl[#tbl+1] = str
        i = i + 1
    end
    return tbl
end
4. Tips for Modders
Use a manager class to encapsulate state and callbacks – it avoids global variables and makes cleanup easier.

Leverage existing game data structures: db.storage, alife(), game_graph(), simulation_objects to access NPCs and their properties.

Respect performance: In FindSpeaker loops, early exit with conditions; limit searches when possible.

Anti-spam: Use counters (self.spammer) to prevent repeated messages of the same type within a short time.

Configurability: Tie every news category to a UI option (ui_options.get) so players can toggle them.

Translation: Always use game.translate_string for any text that might appear to the player. Store variations as numbered keys for easy expansion.

Test edge cases: What if no NPC is online? What if the killer is the actor? The script handles these gracefully.