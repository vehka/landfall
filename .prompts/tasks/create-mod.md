# Task: Create a Norns Mod

## Objective
Build a system-level modification that extends core Norns functionality across all scripts.

## What is a Mod?

**Mods** are different from scripts:
- **Scripts**: Standalone applications in `~/dust/code/`
- **Mods**: System modifications in `~/dust/mods/` that load during Matron startup
- Mods affect ALL scripts via hooks and global state
- Mods can provide library APIs for scripts to use
- Mods require explicit user enablement (disabled by default)

## Workflow

### 1. Define Mod Purpose

Answer these questions:
- **What system-level modification does this mod provide?**
  - Extend OSC capabilities?
  - Add MIDI processing?
  - Provide a custom parameter type?
  - Add screen drawing helpers?
  - Modify audio behavior?
  - Provide a utility library for scripts?

- **What lifecycle does it need?**
  - System startup/shutdown? (system hooks)
  - Per-script initialization? (script hooks)
  - Menu integration?
  - Persistent state?

- **What is the mod's scope?**
  - Beginner: Simple parameter extension, utility library
  - Intermediate: System behavior modification, menu UI
  - Advanced: Audio routing changes, global event interception

### 2. Plan Hook Strategy

Mods interact with the system via **5 hooks** (called in order during script lifecycle):

```
SYSTEM STARTUP
    ↓
system_post_startup        (mod initializes)
    ↓
USER LOADS SCRIPT
    ↓
script_pre_init            (before script's init())
    ↓
[Script engine loads]
    ↓
script_post_init           (after script's init())
    ↓
USER RUNS SCRIPT
[Normal operation]
    ↓
USER UNLOADS SCRIPT
    ↓
script_post_cleanup        (after script cleanup())
    ↓
USER SLEEPS (optional)
    ↓
system_pre_shutdown        (before Matron shutdown)
```

**Which hooks does your mod need?**
- `system_post_startup`: One-time initialization
- `system_pre_shutdown`: Cleanup or save state
- `script_pre_init`: Modify script environment before it loads
- `script_post_init`: Inject features after script initializes
- `script_post_cleanup`: Handle per-script teardown

### 3. File Structure

```
~/dust/mods/my_mod/
├── lib/
│   ├── mod.lua              (required - main mod entry point)
│   └── my_mod.lua           (optional - library code for scripts)
└── docs/
    └── README.md            (optional - user documentation)
```

**Critical**: The `lib/mod.lua` file is required. It registers hooks and defines mod behavior.

A mod is defined as: any project in `~/dust/mods/` that contains a `lib/mod.lua` file.

### 4. Register Hooks

In `lib/mod.lua`, register callbacks for needed hooks:

```lua
-- lib/mod.lua

local mod = {}

function mod:init()
  -- Called once when mod is loaded
  print("MyMod initializing")
end

-- Register hook callbacks
_norns.hook.register("system_post_startup", "MyMod", function()
  print("System started - MyMod is active")
end)

_norns.hook.register("script_pre_init", "MyMod", function()
  print("Script loading - MyMod preparing environment")
end)

_norns.hook.register("script_post_init", "MyMod", function()
  print("Script loaded - MyMod features ready")
end)

_norns.hook.register("script_post_cleanup", "MyMod", function()
  print("Script unloaded - MyMod cleaning up")
end)

_norns.hook.register("system_pre_shutdown", "MyMod", function()
  print("System shutting down - MyMod saving state")
end)

return mod
```

### 5. Expose Mod API (Optional)

If your mod provides utilities for scripts, create a library:

```lua
-- lib/my_mod.lua

local MyMod = {}

function MyMod.custom_feature(param)
  return param * 2
end

return MyMod
```

Scripts can then `require` your mod:

```lua
-- In a script
local my_mod = require "my_mod"
local result = my_mod.custom_feature(5)  -- returns 10
```

### 6. Optional: Add Documentation

Create a README.md file to document your mod:

```markdown
# My Mod

Brief description of what this mod does.

## Installation

1. Clone into `~/dust/mods/my_mod/`
2. Restart Norns
3. Enable in SYSTEM > MODS > My Mod

## Usage

How users interact with this mod...

## Requirements

Any dependencies or Norns version requirements...
```

### 7. Develop Modularly

**Keep mods focused**:
- One primary function per mod
- Don't duplicate core Norns functionality
- Document what global state your mod creates
- Minimize side effects on other mods

**Safety practices**:
- Test with other mods enabled to check for conflicts
- Use unique naming for global functions (e.g., `MyMod_helper()`)
- Handle errors gracefully; mods shouldn't crash the system
- Save state in user-accessible locations

**Performance considerations**:
- Hooks should run quickly; defer heavy computation
- `script_post_init` runs for every script load (be efficient)
- System hooks run once at startup/shutdown (more flexibility)

### 8. Menu Integration (Advanced)

Mods can provide custom menu pages:

```lua
_norns.menu.register("MyMod", function()
  -- Return a menu definition
  return {
    "Option 1",
    "Option 2",
    "Option 3"
  }
end)
```

See `.prompts/mods/menu-integration.md` for detailed patterns.

### 9. Testing Strategy

**Local Testing**:
1. Place mod in `~/dust/mods/my_mod/`
2. Restart Norns
3. Enable mod via `SYSTEM > MODS`
4. Load various scripts and verify behavior
5. Check for errors in logs

**Test Scenarios**:
- Does mod load without errors?
- Does mod affect all scripts consistently?
- Does mod conflict with other enabled mods?
- Does mod clean up properly when disabled?
- Does state persist across script reloads?
- Does mod handle edge cases (missing files, invalid data)?

**Debugging**:
- Check `~/.norns/matron.log` for hook execution
- Use `print()` statements in hooks for debugging
- Test individually with other mods disabled first

### 10. Distribution

When ready to share:

1. **Version Control**: Push to GitHub
2. **Documentation**: Write comprehensive README.md
3. **Testing**: Get feedback from other Norns users
4. **Maiden Integration**: Submit to Maiden project manager (optional)

## Common Mod Patterns

### Utility Library Mod
Provides helper functions for scripts:
- Hooks: `system_post_startup` only
- Exposes library via `require "mod_name"`
- Examples: Extended musicutil functions, UI helpers

### System Enhancement Mod
Modifies core system behavior:
- Hooks: `script_pre_init`, `script_post_init`
- Modifies global environment or script params
- Examples: Custom parameter types, MIDI routing enhancements

### Audio Processing Mod
Extends audio capabilities:
- Hooks: `script_post_init`, `script_post_cleanup`
- Manages audio resources globally
- Examples: Effects chain, audio input monitoring

### Menu/UI Mod
Adds menu pages or UI features:
- Hooks: `system_post_startup`
- Integrates with menu system
- Examples: Mod settings, system monitoring dashboard

## Mod Compatibility

**Assume multiple mods enabled**:
- Don't pollute global namespace (use unique prefixes)
- Don't assume you're the only mod modifying scripts
- Test with at least 2-3 other mods enabled
- Document any conflicts you discover

**Mod ordering**:
- Hooks execute alphabetically by mod name
- If ordering matters, document it
- Use hook names like `AA_MyMod` to force early loading

## Reference Resources

- Norns Mods Documentation: https://monome.org/docs/norns/mods/
- Norns Source (mods): ../norns/lua/core/mods.lua
- Existing mods: https://github.com/monome (search for mod-* repos)
- Hook system: ../norns/lua/core/mods.lua (grep for "hook.register")

## Common Issues

**Mod not loading**:
- Check `lib/mod.lua` exists in correct location
- Verify syntax in Lua files
- Check Matron logs for errors

**Mod not executing hooks**:
- Verify hook name is correct
- Check mod is enabled in SYSTEM > MODS
- Ensure callback function is valid

**Conflicts with other mods**:
- Check for global variable collisions
- Test with other mods disabled
- Verify hook execution order doesn't matter

**State not persisting**:
- Use `norns.data` for file storage
- Verify paths are correct
- Check file permissions
