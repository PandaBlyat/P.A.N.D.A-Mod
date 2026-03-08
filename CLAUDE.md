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
