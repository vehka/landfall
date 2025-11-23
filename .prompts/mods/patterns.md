# norns Mod Patterns and Examples

Common patterns for mod development with working code examples.

## Basic Mod Structure

Every mod needs this minimal setup:

```lua
-- lib/mod.lua
local mod = {}

function mod:init()
  -- Called when mod loads
end

_norns.hook.register("system_post_startup", "MyMod", function()
  -- Mod is now active
end)

return mod
```

## Pattern 1: Utility Library Mod

Provides helper functions for scripts to use.

**File: lib/mod.lua**
```lua
local mod = {}

function mod:init()
  print("MyMod utility library loaded")
end

_norns.hook.register("system_post_startup", "MyMod", function()
  print("MyMod ready for use by scripts")
end)

return mod
```

**File: lib/my_mod.lua** (provides the actual library)
```lua
local MyMod = {}

-- Array utilities
function MyMod.array_rotate(arr, n)
  local len = #arr
  if len == 0 then return arr end
  n = ((n % len) + len) % len
  return {table.unpack(arr, n+1), table.unpack(arr, 1, n)}
end

function MyMod.array_sum(arr)
  local sum = 0
  for _, v in ipairs(arr) do sum = sum + v end
  return sum
end

function MyMod.array_reverse(arr)
  local rev = {}
  for i = #arr, 1, -1 do table.insert(rev, arr[i]) end
  return rev
end

-- String utilities
function MyMod.split(str, sep)
  local parts = {}
  for part in string.gmatch(str, "[^" .. sep .. "]+") do
    table.insert(parts, part)
  end
  return parts
end

function MyMod.trim(str)
  return str:match("^%s*(.-)%s*$")
end

-- Math utilities
function MyMod.clamp(val, min, max)
  return math.max(min, math.min(max, val))
end

function MyMod.scale(val, in_min, in_max, out_min, out_max)
  local normalized = (val - in_min) / (in_max - in_min)
  return out_min + normalized * (out_max - out_min)
end

return MyMod
```

**Usage in a script**:
```lua
function init()
  local my_mod = require "my_mod"

  local arr = {1, 2, 3, 4}
  local sum = my_mod.array_sum(arr)  -- 10
  local rotated = my_mod.array_rotate(arr, 1)  -- {2, 3, 4, 1}
end
```

## Pattern 2: Parameter Extension Mod

Adds custom parameter types or behaviors to the param system.

**File: lib/mod.lua**
```lua
local mod = {}

function mod:init()
  print("ParamExtend mod initializing")
end

-- Extend params with custom getter/setter
_norns.hook.register("script_post_init", "ParamExtend", function()
  if params == nil then return end

  -- Add a custom parameter type
  params.add_color = function(self, id, name, default_rgb)
    self:add{
      type = "control",
      id = id,
      name = name,
      controlspec = ControlSpec.new(0, 15, "lin", 1, default_rgb or 0),
      action = function(val)
        -- Custom handler for color
      end
    }
  end

  print("Param extension registered")
end)

return mod
```

**Usage in a script**:
```lua
function init()
  params:add_color("color1", "Primary Color", 8)
  params:add_color("color2", "Secondary Color", 4)
end
```

## Pattern 3: Global Mod with State

Maintains state across script reloads.

**File: lib/mod.lua**
```lua
local mod = {}

-- Persistent state
mod.state = {
  current_mode = 1,
  history = {},
  settings = {}
}

function mod:init()
  self:load_state()
end

function mod:load_state()
  local data = norns.data.read("mod_state")
  if data then
    self.state = data
    print("Mod state loaded")
  end
end

function mod:save_state()
  norns.data.write(self.state, "mod_state")
end

function mod:record_action(action)
  table.insert(self.state.history, action)
  if #self.state.history > 100 then
    table.remove(self.state.history, 1)
  end
  self:save_state()
end

-- System hooks
_norns.hook.register("system_post_startup", "GlobalMod", function()
  print("Global mod active with " .. #mod.state.history .. " items in history")
end)

_norns.hook.register("system_pre_shutdown", "GlobalMod", function()
  mod:save_state()
  print("Global mod state saved")
end)

return mod
```

## Pattern 4: Script Environment Modifier

Modifies what's available to ALL scripts at load time.

**File: lib/mod.lua**
```lua
local mod = {}

_norns.hook.register("script_pre_init", "EnvMod", function()
  -- Add global helpers before script loads

  -- Enhanced printing with timestamps
  local old_print = print
  function print(...)
    local msg = table.concat({...}, " ")
    old_print("[" .. os.date("%H:%M:%S") .. "] " .. msg)
  end

  -- Global utility function
  function script_log(level, msg)
    print(level .. ": " .. msg)
  end
end)

_norns.hook.register("script_post_cleanup", "EnvMod", function()
  -- No cleanup needed for this pattern
end)

return mod
```

**Usage in a script** (automatically available):
```lua
function init()
  script_log("INFO", "Script starting")  -- Prints with level and timestamp
end
```

## Pattern 5: MIDI Processing Mod

Intercepts and modifies MIDI messages system-wide.

**File: lib/mod.lua**
```lua
local mod = {}

mod.midi_state = {
  velocity_curve = "linear",
  transpose = 0,
  enabled_channels = {true, true, true, true}  -- channels 1-4 enabled
}

_norns.hook.register("script_post_init", "MIDIMod", function()
  if midi == nil then return end

  -- Wrap existing MIDI event handler
  -- Note: This is simplified; actual implementation depends on MIDI architecture

  print("MIDI mod initialized")
end)

_norns.hook.register("system_pre_shutdown", "MIDIMod", function()
  -- Save MIDI settings
  norns.data.write(mod.midi_state, "midi_mod_settings")
end)

return mod
```

## Pattern 6: Metro Management Mod

Provides global metronome or timing services.

**File: lib/mod.lua**
```lua
local mod = {}

mod.metros = {}
mod.master_tempo = 120

function mod:create_metro(id, interval, action)
  local m = metro.init(action, interval)
  self.metros[id] = m
  return m
end

function mod:set_tempo(bpm)
  self.master_tempo = bpm
  -- Update all dependent metros
  for id, m in pairs(self.metros) do
    if m.running then
      m:time(60 / bpm / 4)  -- For quarter notes at given BPM
    end
  end
end

_norns.hook.register("script_post_init", "MetroMod", function()
  print("Metro mod ready at " .. mod.master_tempo .. " BPM")
end)

_norns.hook.register("script_post_cleanup", "MetroMod", function()
  -- Stop all metros when script unloads
  for id, m in pairs(mod.metros) do
    if m.running then m:stop() end
  end
end)

return mod
```

## Pattern 7: Screen Drawing Helper Mod

Provides utilities for scripts to draw to the screen.

**File: lib/screen_helpers.lua**
```lua
local Helpers = {}

function Helpers.draw_box(x, y, w, h, border)
  border = border or 1
  screen.rect(x, y, w, h)
  if border > 0 then
    for i = 1, border do
      screen.rect(x + i, y + i, w - 2*i, h - 2*i)
      screen.stroke()
    end
  else
    screen.fill()
  end
end

function Helpers.draw_text_centered(x, y, text)
  local len = #text * 4  -- Approximate character width
  screen.move(x - len/2, y)
  screen.text(text)
end

function Helpers.draw_progress_bar(x, y, w, h, percent)
  screen.rect(x, y, w, h)
  screen.stroke()
  local fill_w = w * percent
  screen.rect(x, y, fill_w, h)
  screen.fill()
end

function Helpers.draw_vertical_menu(x, y, items, selected)
  for i, item in ipairs(items) do
    local level = (i == selected) and 15 or 5
    screen.level(level)
    screen.move(x, y + (i-1) * 10)
    screen.text(item)
  end
  screen.level(15)
end

return Helpers
```

**File: lib/mod.lua**
```lua
local mod = {}

_norns.hook.register("system_post_startup", "ScreenHelpers", function()
  print("Screen helpers mod loaded")
end)

return mod
```

**Usage in a script**:
```lua
function init()
  screen_helpers = require "screen_helpers"
end

function redraw()
  screen.clear()

  screen_helpers.draw_box(10, 10, 50, 30)
  screen_helpers.draw_text_centered(64, 32, "Hello")
  screen_helpers.draw_progress_bar(10, 50, 100, 5, 0.6)

  screen.update()
end
```

## Pattern 8: Documentation Template

Create a README.md file to document your mod:

```markdown
# My Awesome Mod

Brief description of what this mod does and how it extends norns.

## Installation

1. Clone this repository into `~/dust/mods/my_awesome_mod/`
2. Restart norns
3. Enable the mod in `SYSTEM > MODS > My Awesome Mod`

## Usage

Describe how scripts can use this mod:

### As a Library

```lua
local my_mod = require "my_awesome_mod"
my_mod.some_function()
```

### What It Provides

- Feature 1: Description
- Feature 2: Description
- Feature 3: Description

## Requirements

- norns version: [minimum version if any]
- Dependencies: [other mods if required]

## API Reference

Document public functions and hooks...

## Contributing

[How to contribute or report issues]
```

## Common Patterns Summary

| Pattern | Use Case | Hooks Used |
|---------|----------|-----------|
| Utility Library | Provide helper functions | `system_post_startup` |
| Parameter Extension | Add param types or behaviors | `script_post_init` |
| Global State | Track system-wide data | `system_pre_shutdown` |
| Environment Modifier | Inject globals into scripts | `script_pre_init` |
| MIDI Processing | Intercept MIDI messages | `script_post_init` |
| Metro Management | Control timing globally | `script_post_cleanup` |
| Screen Helpers | Provide drawing utilities | `system_post_startup` |

## Performance Considerations

- `script_post_init` runs for EVERY script load - keep it fast
- `system_post_startup` runs once - OK for heavy initialization
- Avoid large allocations in hooks
- Use local variables; minimize global state
- Clean up resources in corresponding hooks

## Debugging Tips

1. **Check hook execution**:
   ```lua
   _norns.hook.register("system_post_startup", "MyMod", function()
     print("DEBUG: system_post_startup called")
   end)
   ```

2. **Verify mod loads**:
   - Check `~/.norns/matron.log` for mod initialization messages
   - Verify `lib/mod.lua` exists and has valid Lua syntax

3. **Test mod isolation**:
   - Disable all other mods
   - Test your mod alone
   - Then enable mods one by one to find conflicts

4. **Monitor memory**:
   - Print state size periodically in hooks
   - Clean up old data in `script_post_cleanup`

## Reference

- norns Mods Documentation: https://monome.org/docs/norns/mods/
- norns Hook System: ../norns/lua/core/mods.lua
- Example Mods: https://github.com/monome (search "mod-")
