# Grid Integration Guide

The Monome Grid is a 16x8 or 8x8 button/LED matrix controller commonly used with Norns for sequencing and performance.

## Basics

### Connection and Event Setup

```lua
local g = nil

function init()
  g = grid.connect()
  g.event = grid_event
end

function grid_event(x, y, z)
  -- x: column (1-16)
  -- y: row (1-8)
  -- z: press (1) or release (0)
end
```

### LED Control

```lua
-- Set individual LED
g:led(x, y, brightness)      -- brightness: 0-15

-- Fill entire grid
g:all(brightness)

-- Sections
g:col(x, brightness)         -- Set entire column
g:row(y, brightness)         -- Set entire row

-- Refresh display
g:refresh()                   -- Send pending updates
```

## Common Patterns

### Piano Keyboard (Rows = Notes, Columns = Octaves)

```lua
function init()
  g = grid.connect()
  g.event = grid_event

  local scale = musicutil.DIATONIC_MAJOR
  local root = 36  -- C2

  -- Build note lookup table
  local note_map = {}
  for y = 1, 8 do
    note_map[y] = root + (8 - y) * 12  -- one octave per row
  end
end

function grid_event(x, y, z)
  if z == 1 then
    local note = note_map[y] + (x - 1)
    engine.hz(musicutil.note_num_to_freq(note))
    engine.gate(1)
  else
    engine.gate(0)
  end
  redraw_grid()
end

function redraw_grid()
  g:all(0)
  -- Highlight active keys or current state
  g:refresh()
end
```

### 16-Step Sequencer

```lua
local pattern = {}
local step = 0
local playing = false
local selected_row = 1

function init()
  g = grid.connect()
  g.event = grid_event

  -- Initialize pattern (8 rows of 16 steps)
  for row = 1, 8 do
    pattern[row] = {}
    for col = 1, 16 do
      pattern[row][col] = 0
    end
  end

  local seq_metro = metro.init(function()
    if playing then
      step = (step % 16) + 1
      trigger_step()
      redraw_grid()
    end
  end, 0.125)  -- 8th notes at ~120 BPM
  seq_metro:start()
end

function grid_event(x, y, z)
  if z == 1 then
    if y == 1 then
      -- Row 1: play/stop and step nav
      if x == 1 then
        playing = not playing
      end
    else
      -- Rows 2-8: step edit
      local row = y - 1
      pattern[row][x] = 1 - pattern[row][x]  -- toggle step
      selected_row = row
    end
  end
  redraw_grid()
end

function trigger_step()
  -- Play notes for current step across all rows
  for row = 1, 8 do
    if pattern[row][step] == 1 then
      local note = 36 + row * 2  -- Scale notes
      engine.hz(musicutil.note_num_to_freq(note))
      engine.gate(1)
      -- Gate off would happen at next step
    end
  end
end

function redraw_grid()
  g:all(0)

  -- Row 1: controls
  g:led(1, 1, playing and 15 or 5)  -- Play/stop toggle

  -- Rows 2-8: step sequencer
  for row = 1, 8 do
    for col = 1, 16 do
      local brightness = pattern[row][col] * 10
      if col == step then
        brightness = brightness + 5  -- Highlight current step
      end
      g:led(col, row + 1, brightness)
    end
  end

  g:refresh()
end

function cleanup()
  if g then
    g:all(0)
    g:refresh()
  end
end
```

### Chord Grid (X-Y as Intervals)

```lua
local root_note = 36
local selected_note = root_note
local held_keys = {}

function init()
  g = grid.connect()
  g.event = grid_event
end

function grid_event(x, y, z)
  local note = root_note + (x - 1) + (y - 1) * 2

  if z == 1 then
    held_keys[note] = true
    engine.hz(musicutil.note_num_to_freq(note))
    engine.gate(1)
  else
    held_keys[note] = nil
    -- Only gate off if no other keys held
    if next(held_keys) == nil then
      engine.gate(0)
    else
      -- Switch to next held note
      for k, v in pairs(held_keys) do
        if v then
          engine.hz(musicutil.note_num_to_freq(k))
          break
        end
      end
    end
  end
  redraw_grid()
end

function redraw_grid()
  g:all(0)
  -- Light up held keys
  for x = 1, 16 do
    for y = 1, 8 do
      local note = root_note + (x - 1) + (y - 1) * 2
      if held_keys[note] then
        g:led(x, y, 15)
      end
    end
  end
  g:refresh()
end
```

## Advanced Patterns

### Multi-Layer Grid (Mode Switching)

```lua
local mode = 1  -- 1: play, 2: record, 3: navigate

function grid_event(x, y, z)
  if z == 1 then
    if y == 1 then
      -- First row: mode switching
      mode = x > 8 and 1 or (x > 4 and 2 or 3)
    elseif mode == 1 then
      handle_play(x, y, z)
    elseif mode == 2 then
      handle_record(x, y, z)
    elseif mode == 3 then
      handle_navigate(x, y, z)
    end
  end
  redraw_grid()
end
```

### Animation on Grid

```lua
local animation_frame = 0

local update_metro = metro.init(function()
  animation_frame = (animation_frame + 1) % 60
  redraw_grid()
end, 1/30)
update_metro:start()

function redraw_grid()
  g:all(0)

  -- Draw animated elements based on animation_frame
  local brightness = math.floor(15 * (animation_frame / 60))
  g:led(8, 4, brightness)

  g:refresh()
end
```

## Grid Device Variants

- **Grid 128** (16x8): Full size
- **Grid 64** (8x8): Smaller variant

Check grid size:
```lua
if g.cols == 8 then
  -- Grid 64 code
else
  -- Grid 128 code
end
```

## Reference

- Grid API: https://monome.org/docs/norns/api/#grid
- Grid Studies: https://monome.org/docs/norns/studies/physical/
- Example scripts with grid: https://github.com/monome/dust

## Common Issues

**Grid not responding**: Ensure device is connected and `g:refresh()` is called
**LEDs not updating**: Call `g:refresh()` after LED changes
**Missing button presses**: Check `z == 1` for press vs `z == 0` for release
