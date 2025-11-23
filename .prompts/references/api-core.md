# Norns Core API Reference

## Script Lifecycle Functions

Required functions in a Norns script:

```lua
function init()
  -- Called when script loads
  -- Initialize engines, parameters, UI state, metros
end

function key(n, z)
  -- Called on button press/release
  -- n: key number (1, 2, 3)
  -- z: press (1) or release (0)
  redraw()
end

function enc(n, d)
  -- Called on encoder rotation
  -- n: encoder number (1, 2, 3)
  -- d: delta (-1 or 1, or more for faster rotation)
  redraw()
end

function redraw()
  -- Called to update display
  -- Always call screen.update() last
  screen.clear()
  screen.move(x, y)
  screen.text("text")
  screen.update()
end

function cleanup()
  -- Called on script unload
  -- Stop metros, gates, save state
end
```

## Parameter System (`params`)

### Creating Parameters

```lua
-- In init():
params:add_number("param_id", "Display Name", min, default, max, units)
params:add_option("param_id", "Display Name", {"opt1", "opt2"}, default_index)
params:add_control("param_id", "Display Name", controlspec.UNIPOLAR)
params:add_file("file_id", "File Name", path_default)
params:add_text("text_id", "Display Name", default_text)

-- Create parameter group
params:add_group("Group Name", num_params)
```

### Managing Parameters

```lua
params:get("param_id")              -- Get value
params:set("param_id", value)       -- Set value
params:delta("param_id", amount)    -- Change by delta (for encoders)
params:set_action("param_id", fn)   -- Function called on change

params:read()   -- Load from disk
params:write()  -- Save to disk
```

### Control Specs (for `control` type)

```lua
controlspec.UNIPOLAR      -- 0 to 1
controlspec.BIPOLAR       -- -1 to 1
controlspec.AMP           -- 0 to 6 dB
controlspec.FREQ          -- 20 to 20000 Hz
controlspec.MS            -- time in milliseconds
```

## Screen (`screen`)

Drawing to the 128x64 monochrome display:

```lua
screen.clear()            -- Clear display
screen.move(x, y)         -- Move cursor
screen.text(str)          -- Draw text at cursor
screen.text_center(str)   -- Center text horizontally
screen.line(x1, y1, x2, y2) -- Draw line
screen.circle(x, y, r)    -- Draw circle
screen.rect(x, y, w, h)   -- Draw rectangle
screen.fill()             -- Fill current path
screen.stroke()           -- Stroke current path
screen.level(brightness)  -- Set brightness (0-15)
screen.update()           -- Send to display (always last)

-- Common text positions
screen.move(2, 10)   -- top-left
screen.move(64, 32)  -- center
screen.move(126, 62) -- bottom-right (approximate)
```

## Metro (Timing)

High-resolution scheduling:

```lua
-- Create a metro
local my_metro = metro.init(function()
  -- code runs at specified interval
  redraw()
end, interval_seconds)

my_metro:start()
my_metro:stop()
my_metro:time(new_interval)  -- Change interval

-- Common intervals
1/30      -- 30 Hz (screen updates)
1/60      -- 60 Hz (smooth animation)
0.1       -- 100 ms (UI updates)
```

## Engine Integration

Loading and controlling SuperCollider engines:

```lua
-- In init():
engine.name = "EngineNameHere"

-- Control engine parameters
engine.param_name(value)
engine.gate(1)    -- Note on
engine.gate(0)    -- Note off

-- Check available engines and parameters
-- /System > Engines menu on Norns device
```

## Audio Hardware

System audio controls:

```lua
-- Audio levels
audio.level_adc(level)      -- Input level (0-3)
audio.level_dac(level)      -- Output level (0-3)

-- Other audio functions (check ../norns/lua/core/audio.lua)
```

## Clock (`clock`)

Tempo synchronization with built-in clock:

```lua
clock.sync(divisor)         -- Sync to clock beat
-- Returns immediately, resumes when beat aligns
-- divisor: 1/1, 1/2, 1/4, etc.

clock.get_tempo()           -- Get BPM
clock.get_beats()           -- Get current beat position
```

## Utility Libraries

Norns includes powerful built-in libraries:

### `musicutil`
Music theory utilities:

```lua
-- Note handling
musicutil.note_to_num("C4")           -- Note name to number
musicutil.note_num_to_freq(60)        -- MIDI to Hz
musicutil.freq_to_note_num(440)       -- Hz to MIDI
musicutil.quantize_to_scale(note, scale, root)

-- Common scales
musicutil.MINOR_PENTATONIC
musicutil.MAJOR_PENTATONIC
musicutil.DIATONIC_MINOR
musicutil.DIATONIC_MAJOR
```

### `sequins`
Pattern sequencing:

```lua
local seq = sequins({1, 2, 3, 2})
seq:next()        -- Get next value
seq:reset()       -- Reset to start
seq:settable(new_tbl)  -- Change pattern
```

### `lattice`
Beat-synced timing:

```lua
local lat = lattice.new{divisions={1/4}}
local unit = lat:new_sprocket{
  action = function() end,
  division = 1/4  -- Quarter note
}
unit:start()
lat:start()
```

## File I/O (`norns.data`)

Saving and loading script data:

```lua
-- Check if directory exists
norns.data.dir = "my_script"   -- or
norns.data.path() -- returns ~/dust/data/my_script/

-- File operations
local data = {key = value}
norns.data.write(data, filename)
local loaded = norns.data.read(filename)
```

## System Functions (`norns`)

```lua
norns.version.major
norns.version.minor
norns.version.patch

norns.rerun()           -- Reload script
norns.shutdown()        -- Shutdown Norns
norns.system_cmd(cmd)   -- Execute shell command

-- Version checking (use YYMMDD format)
if norns.version.required("220101") then
  -- code for Norns >= 2022-01-01
end
```

## Reference

Complete API documentation: https://monome.org/docs/norns/api/

For detailed examples and advanced patterns, see:
- `.prompts/references/patterns.md`
- Norns Studies: https://monome.org/docs/norns/studies/
- Example scripts: https://github.com/monome/dust
