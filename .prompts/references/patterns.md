# Norns Script Patterns and Best Practices

## Common Code Patterns

### Simple Parameter with Encoder Control

```lua
function init()
  params:add_number("volume", "Volume", 0, 0.5, 1.0)
  params:set_action("volume", function(x)
    engine.volume(x)
  end)
end

function enc(n, d)
  if n == 1 then
    params:delta("volume", d * 0.1)
  end
  redraw()
end

function redraw()
  screen.clear()
  screen.move(2, 10)
  screen.text("Volume: " .. string.format("%.2f", params:get("volume")))
  screen.update()
end
```

### State Machine (Mode Switching)

```lua
local mode = 1  -- 1: play, 2: record, 3: edit

function key(n, z)
  if z == 1 then  -- on press
    if n == 1 then
      mode = (mode % 3) + 1  -- cycle modes
    elseif n == 2 then
      handle_mode_key_2()
    elseif n == 3 then
      handle_mode_key_3()
    end
  end
  redraw()
end

function handle_mode_key_2()
  if mode == 1 then
    -- play mode logic
  elseif mode == 2 then
    -- record mode logic
  elseif mode == 3 then
    -- edit mode logic
  end
end

function redraw()
  screen.clear()
  screen.move(2, 10)
  screen.text("Mode: " .. ({"Play", "Record", "Edit"})[mode])
  screen.update()
end
```

### Metro-based Sequencer

```lua
local step = 0
local max_steps = 16
local steps = {0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0}
local is_playing = false

function init()
  local seq_metro = metro.init(function()
    if is_playing then
      step = (step % max_steps) + 1
      trigger_step(step)
      redraw()
    end
  end, 0.1)  -- 100ms = ~600 BPM at 16th notes
  seq_metro:start()
end

function trigger_step(step_num)
  local note = steps[step_num]
  if note > 0 then
    engine.gate(1)
    engine.hz(musicutil.note_num_to_freq(note))
  end
end

function key(n, z)
  if z == 1 then
    if n == 1 then
      is_playing = not is_playing
    elseif n == 2 then
      step = 0
    end
  end
  redraw()
end

function enc(n, d)
  if n == 1 and step > 0 then
    steps[step] = math.max(0, steps[step] + d)
  end
  redraw()
end

function redraw()
  screen.clear()
  screen.move(2, 10)
  screen.text((is_playing and "▶" or "■") .. " Step: " .. step)
  screen.update()
end
```

### Data Persistence

```lua
local data_file = "script_data"
local state = {volume = 0.5, pattern = {}}

function init()
  load_state()
  params:set("volume", state.volume)
end

function load_state()
  local loaded = norns.data.read(data_file)
  if loaded then
    state = loaded
  end
end

function save_state()
  state.volume = params:get("volume")
  norns.data.write(state, data_file)
end

function cleanup()
  save_state()
end
```

### Grid Integration

```lua
function init()
  g = grid.connect()
  g.event = grid_event
end

function grid_event(x, y, z)
  if z == 1 then  -- button press
    -- Handle grid button at (x, y)
    engine.hz(musicutil.note_num_to_freq(36 + x + (8 - y) * 8))
    engine.gate(1)
  else  -- button release
    engine.gate(0)
  end
  redraw_grid()
end

function redraw_grid()
  g:all(0)  -- Clear grid
  -- Draw patterns or indicators on grid
  g:led(1, 1, 15)  -- LED at (1,1) brightness 15
  g:refresh()
end
```

### Arc Integration

```lua
function init()
  a = arc.connect()
  a.event = arc_event
end

function arc_event(n, d)
  -- n: ring number (1-4)
  -- d: delta (-1 or 1)
  if n == 1 then
    params:delta("param1", d * 0.1)
  elseif n == 2 then
    params:delta("param2", d * 0.1)
  end
  redraw_arc()
  redraw()
end

function redraw_arc()
  a:all(0)  -- Clear all rings
  -- Draw indicators for parameters
  local val = params:get("param1")
  a:segment(1, 0, val * 64, 15)  -- Segment from 0 to (val*64) on ring 1
  a:refresh()
end
```

### MIDI Integration

```lua
local midi_device = midi.connect(1)

function init()
  midi_device.event = midi_event
end

function midi_event(data)
  local msg = midi.to_msg(data)
  if msg.type == "note_on" then
    engine.hz(musicutil.note_num_to_freq(msg.note))
    engine.gate(1)
  elseif msg.type == "note_off" then
    engine.gate(0)
  elseif msg.type == "cc" then
    -- Handle CC messages
    if msg.cc == 1 then
      params:set("param1", msg.val / 127)
    end
  end
end
```

### Screen Layout with Multiple Elements

```lua
function redraw()
  screen.clear()
  screen.level(15)  -- Max brightness

  -- Header
  screen.move(2, 10)
  screen.text("My Script")

  -- Dividing line
  screen.line(0, 14, 128, 14)
  screen.stroke()

  -- Main content area
  screen.move(2, 24)
  screen.text("Param 1: " .. params:get("param1"))

  screen.move(2, 34)
  screen.text("Param 2: " .. params:get("param2"))

  -- Status bar (bottom)
  screen.level(5)  -- Lower brightness
  screen.move(2, 62)
  screen.text("Ready")

  screen.update()
end
```

### Quantization Pattern

```lua
local scale = musicutil.DIATONIC_MAJOR
local root = 36  -- C2

function note_in(midi_note)
  local quantized = musicutil.quantize_to_scale(
    midi_note, scale, root
  )
  return quantized
end
```

### Parameter Group Organization

```lua
function init()
  -- Oscillator group
  params:add_group("Oscillator", 3)
  params:add_number("osc_wave", "Waveform", 1, 1, 3)
  params:add_number("osc_freq", "Frequency", -24, 0, 24)
  params:add_number("osc_level", "Level", 0, 0.5, 1)

  -- Envelope group
  params:add_group("Envelope", 4)
  params:add_number("env_attack", "Attack", 0.01, 0.1, 1)
  params:add_number("env_decay", "Decay", 0.01, 0.2, 2)
  params:add_number("env_sustain", "Sustain", 0, 0.5, 1)
  params:add_number("env_release", "Release", 0.01, 0.5, 5)

  params:bang()
end
```

## Performance Tips

1. **Screen Rendering**: Call `redraw()` at ~30 Hz max, not on every frame
   ```lua
   local redraw_metro = metro.init(redraw, 1/30)
   redraw_metro:start()
   ```

2. **Avoid Allocations in Hot Paths**: Create tables in `init()`, reuse in `key()`/`enc()`
   ```lua
   -- BAD: allocates every call
   function enc(n, d)
     local t = {}  -- Don't do this
   end

   -- GOOD: allocate once
   local reusable = {}
   function enc(n, d)
     -- reuse reusable table
   end
   ```

3. **Use Local Variables**: Faster than global lookups
   ```lua
   -- BAD
   function enc(n, d)
     myvar = myvar + 1
   end

   -- GOOD
   local myvar = 0
   function enc(n, d)
     myvar = myvar + 1
   end
   ```

4. **Batch Parameter Updates**: Set multiple params before redraw
   ```lua
   params:set("p1", v1)
   params:set("p2", v2)
   params:set("p3", v3)
   redraw()  -- once at the end
   ```

## Common Gotchas

1. **Screen Coordinates**: (0,0) is top-left. Text baseline is approximate - test your layout
2. **Encoder Speed**: `d` can be > 1 for fast turns; scale accordingly
3. **Gate Timing**: Call `engine.gate(1)` then `engine.gate(0)` with some delay
4. **Cleanup Required**: Always stop metros and gates in `cleanup()`
5. **Parameter Persistence**: Changes persist automatically; use `params:write()` to force save

## Reference Resources

- Complete Norns API: https://monome.org/docs/norns/api/
- Example scripts: https://github.com/monome/dust
- Norns Studies: https://monome.org/docs/norns/studies/
