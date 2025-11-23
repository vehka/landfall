# Arc Integration Guide

The Monome Arc is a rotary encoder interface (1, 2, or 4 rings) that can be used for parameter control and real-time interaction with norns scripts.

## Basics

### Connection and Event Setup

```lua
local a = nil

function init()
  a = arc.connect()
  a.event = arc_event
end

function arc_event(n, d)
  -- n: ring number (1-4)
  -- d: delta/change (-1, 0, or 1 typically, faster turns can be larger)
end
```

### LED Display

```lua
-- Set single LED on a ring
a:led(ring, position, brightness)   -- position 0-63, brightness 0-15

-- Set range of LEDs
a:segment(ring, start, end, brightness)   -- smooth range from start to end

-- Set all LEDs on a ring
a:all(ring, brightness)

-- Clear all rings
a:all(0)

-- Update display
a:refresh()
```

## Common Patterns

### Simple Parameter Control

```lua
function init()
  params:add_number("param1", "Param 1", 0, 0.5, 1.0)
  params:add_number("param2", "Param 2", 0, 0.5, 1.0)
  params:add_number("param3", "Param 3", 0, 0.5, 1.0)
  params:add_number("param4", "Param 4", 0, 0.5, 1.0)

  a = arc.connect()
  a.event = arc_event
end

function arc_event(n, d)
  if n == 1 then
    params:delta("param1", d * 0.05)
  elseif n == 2 then
    params:delta("param2", d * 0.05)
  elseif n == 3 then
    params:delta("param3", d * 0.05)
  elseif n == 4 then
    params:delta("param4", d * 0.05)
  end
  redraw()
end

function redraw()
  screen.clear()
  screen.move(2, 10)
  screen.text("P1: " .. string.format("%.2f", params:get("param1")))
  screen.move(2, 20)
  screen.text("P2: " .. string.format("%.2f", params:get("param2")))
  -- etc...
  screen.update()

  redraw_arc()
end

function redraw_arc()
  a:all(0)

  -- Draw segments for each parameter
  a:segment(1, 0, params:get("param1") * 64, 15)
  a:segment(2, 0, params:get("param2") * 64, 15)
  a:segment(3, 0, params:get("param3") * 64, 15)
  a:segment(4, 0, params:get("param4") * 64, 15)

  a:refresh()
end
```

### Tempo/Speed Control

```lua
local tempo_bpm = 120
local tempo_fine = 0  -- fine-tuning offset

function init()
  a = arc.connect()
  a.event = arc_event

  -- Create metro that responds to tempo
  local step_metro = metro.init(function()
    -- trigger step at current tempo
  end, 60 / tempo_bpm / 4)
  step_metro:start()
end

function arc_event(n, d)
  if n == 1 then
    -- Coarse tempo control (ring 1)
    tempo_bpm = math.max(30, math.min(300, tempo_bpm + d * 5))
  elseif n == 2 then
    -- Fine tempo control (ring 2)
    tempo_fine = (tempo_fine + d) % 12
  end
  redraw_arc()
end

function redraw_arc()
  a:all(0)

  -- Show tempo as position around ring 1
  local tempo_pos = ((tempo_bpm - 30) / (300 - 30)) * 64
  a:segment(1, 0, tempo_pos, 15)

  -- Show fine tuning on ring 2
  local fine_pos = (tempo_fine / 12) * 64
  a:led(2, fine_pos, 15)

  a:refresh()
end
```

### Volume and Filter Control

```lua
function init()
  params:add_number("volume", "Volume", 0, 0.5, 1.0)
  params:add_number("filter_freq", "Filter Freq", 100, 1000, 20000)
  params:add_number("filter_res", "Resonance", 0, 0.5, 1.0)

  a = arc.connect()
  a.event = arc_event
end

function arc_event(n, d)
  if n == 1 then
    -- Volume on ring 1
    params:delta("volume", d * 0.05)
  elseif n == 2 then
    -- Filter frequency on ring 2 (exponential)
    local freq = params:get("filter_freq")
    freq = freq * (1 + d * 0.1)
    params:set("filter_freq", math.max(100, math.min(20000, freq)))
  elseif n == 3 then
    -- Resonance on ring 3
    params:delta("filter_res", d * 0.05)
  end
  redraw()
end

function redraw_arc()
  a:all(0)

  -- Volume (ring 1)
  local vol = params:get("volume") * 64
  a:segment(1, 0, vol, 15)

  -- Filter freq (ring 2, log scale)
  local freq = params:get("filter_freq")
  local freq_norm = (math.log(freq) - math.log(100)) /
                    (math.log(20000) - math.log(100))
  a:segment(2, 0, freq_norm * 64, 12)

  -- Resonance (ring 3)
  local res = params:get("filter_res") * 64
  a:segment(3, 0, res, 15)

  a:refresh()
end
```

### XY Pad (Two Rings)

```lua
local x_val = 0.5
local y_val = 0.5

function init()
  a = arc.connect()
  a.event = arc_event
end

function arc_event(n, d)
  if n == 1 then
    x_val = math.max(0, math.min(1, x_val + d * 0.05))
  elseif n == 2 then
    y_val = math.max(0, math.min(1, y_val + d * 0.05))
  end

  -- Use x_val and y_val to control parameters
  engine.param_x(x_val)
  engine.param_y(y_val)

  redraw()
end

function redraw_arc()
  a:all(0)

  -- X position on ring 1
  a:led(1, math.floor(x_val * 64), 15)

  -- Y position on ring 2
  a:led(2, math.floor(y_val * 64), 15)

  a:refresh()
end
```

### Mode Switching with Visual Feedback

```lua
local mode = 1
local modes = {"Synth", "Filter", "Reverb"}

function init()
  a = arc.connect()
  a.event = arc_event
end

function arc_event(n, d)
  if n == 1 then
    -- Ring 1: mode selection
    mode = ((mode - 1 + d) % #modes) + 1
  elseif mode == 1 then
    -- Synth mode
    if n == 2 then params:delta("osc_level", d * 0.05) end
    if n == 3 then params:delta("osc_freq", d * 0.1) end
  elseif mode == 2 then
    -- Filter mode
    if n == 2 then params:delta("filter_freq", d * 0.1) end
    if n == 3 then params:delta("filter_res", d * 0.05) end
  elseif mode == 3 then
    -- Reverb mode
    if n == 2 then params:delta("reverb_size", d * 0.05) end
    if n == 3 then params:delta("reverb_mix", d * 0.05) end
  end
  redraw()
end

function redraw_arc()
  a:all(0)

  -- Ring 1: mode indicator
  local mode_pos = ((mode - 1) / #modes) * 64
  a:led(1, mode_pos, 15)

  -- Highlight current mode with brightness
  for i = 1, 4 do
    if i == 1 then
      a:segment(i, 0, 64, 5)  -- dim non-active rings
    else
      a:segment(i, 0, 64, 15)
    end
  end

  a:refresh()
end
```

### Velocity/Gate Envelope

```lua
local gate_velocity = 1.0

function init()
  a = arc.connect()
  a.event = arc_event

  params:add_number("sustain_level", "Sustain", 0, 0.7, 1)
  params:add_number("release_time", "Release", 0.01, 0.5, 5)
end

function arc_event(n, d)
  if n == 1 then
    -- Velocity control
    gate_velocity = math.max(0, math.min(1, gate_velocity + d * 0.05))
  elseif n == 2 then
    params:delta("sustain_level", d * 0.05)
  elseif n == 3 then
    params:delta("release_time", d * 0.1)
  end
  redraw()
end

function key(n, z)
  if z == 1 then
    engine.level(gate_velocity)
    engine.gate(1)
  else
    engine.gate(0)
  end
  redraw()
end

function redraw_arc()
  a:all(0)
  a:segment(1, 0, gate_velocity * 64, 15)
  a:segment(2, 0, params:get("sustain_level") * 64, 12)
  a:segment(3, 0, (params:get("release_time") / 5) * 64, 12)
  a:refresh()
end
```

## Arc Device Variants

- **Arc 4**: 4 rings (1, 2, and 4-ring variants exist)
- **Arc 2**: 2 rings
- **Arc 1**: Single ring (rare)

Check arc ring count:
```lua
if a.rings == 4 then
  -- 4-ring code
elseif a.rings == 2 then
  -- 2-ring code
end
```

## LED Positioning

Arc uses 0-63 positions per ring (64 positions total). Brightness 0-15 (0 is off).

```lua
-- Simple LED at position
a:led(ring, position, brightness)

-- Segment from start to end
a:segment(ring, start, end, brightness)

-- All 64 positions on a ring
a:all(ring, brightness)
```

## Reference

- Arc API: https://monome.org/docs/norns/api/#arc
- Arc Studies: https://monome.org/docs/norns/studies/physical-arc/
- Example scripts: https://github.com/monome/dust

## Common Issues

**Arc not responding**: Ensure device connected and `a:refresh()` called
**LEDs dim/off**: Check brightness values (0-15)
**Lagging response**: Reduce screen redraw frequency, arc should be responsive
**Unexpected delta values**: Large `d` values on fast turns; normalize with multiplier
