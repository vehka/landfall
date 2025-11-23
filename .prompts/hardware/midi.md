# MIDI Integration Guide

MIDI (Musical Instrument Digital Interface) allows norns scripts to communicate with external synthesizers, controllers, and software.

## Basics

### Connecting MIDI Devices

```lua
local midi_device = nil

function init()
  -- Connect to first available MIDI device
  midi_device = midi.connect(1)

  -- Attach event handler
  midi_device.event = midi_event
end

function midi_event(data)
  local msg = midi.to_msg(data)
  -- Handle message
end
```

### MIDI Message Types

```lua
function midi_event(data)
  local msg = midi.to_msg(data)

  if msg.type == "note_on" then
    print("Note On: " .. msg.note .. " Velocity: " .. msg.vel)
    engine.hz(musicutil.note_num_to_freq(msg.note))
    engine.gate(1)

  elseif msg.type == "note_off" then
    print("Note Off: " .. msg.note)
    engine.gate(0)

  elseif msg.type == "cc" then
    print("Control Change: CC " .. msg.cc .. " = " .. msg.val)
    -- msg.cc: controller number (0-127)
    -- msg.val: value (0-127)
    if msg.cc == 1 then
      params:set("param1", msg.val / 127)
    end

  elseif msg.type == "pitch_bend" then
    print("Pitch Bend: " .. msg.val)
    -- msg.val: -8192 to 8191

  elseif msg.type == "channel_pressure" then
    print("Channel Pressure: " .. msg.val)

  elseif msg.type == "program_change" then
    print("Program: " .. msg.program)
  end
end
```

## Common Patterns

### Simple Keyboard Controller

```lua
function init()
  midi_device = midi.connect(1)
  midi_device.event = midi_event
end

function midi_event(data)
  local msg = midi.to_msg(data)

  if msg.type == "note_on" then
    -- Play note with velocity
    engine.hz(musicutil.note_num_to_freq(msg.note))
    engine.gate(1)
    redraw()

  elseif msg.type == "note_off" then
    engine.gate(0)
    redraw()
  end
end
```

### CC-based Parameter Control

```lua
local cc_map = {
  [1] = "param1",      -- Modulation wheel
  [7] = "volume",      -- Volume
  [10] = "pan",        -- Pan
  [64] = "sustain",    -- Sustain pedal
}

function midi_event(data)
  local msg = midi.to_msg(data)

  if msg.type == "cc" then
    local param_id = cc_map[msg.cc]
    if param_id then
      -- Map 0-127 to parameter range
      local normalized = msg.val / 127
      params:set(param_id, normalized)
      redraw()
    end
  end
end
```

### Sending MIDI Out (to External Synth)

```lua
local midi_out = midi.connect(2)  -- Second device is usually output

function trigger_note(note_num, velocity)
  -- Send note on
  midi_out:note_on(note_num, velocity, channel)

  -- Schedule note off
  metro.init(function()
    midi_out:note_off(note_num, channel)
  end, note_duration):start()
end

-- Or send raw MIDI
function send_cc(cc_num, value)
  midi_out:cc(cc_num, value)
end
```

### Velocity Sensitivity

```lua
function midi_event(data)
  local msg = midi.to_msg(data)

  if msg.type == "note_on" then
    local vel = msg.vel / 127  -- Normalize to 0-1
    local volume = 0.3 + (vel * 0.7)  -- Range 0.3 to 1.0
    engine.volume(volume)
    engine.hz(musicutil.note_num_to_freq(msg.note))
    engine.gate(1)
    redraw()

  elseif msg.type == "note_off" then
    engine.gate(0)
    redraw()
  end
end
```

### Arpeggiator

```lua
local notes = {}
local arp_step = 0
local arp_metro = nil

function init()
  midi_device = midi.connect(1)
  midi_device.event = midi_event

  arp_metro = metro.init(function()
    if #notes > 0 then
      arp_step = (arp_step % #notes) + 1
      local note = notes[arp_step]
      engine.hz(musicutil.note_num_to_freq(note))
      engine.gate(1)
      redraw()
    end
  end, 0.1)
  arp_metro:start()
end

function midi_event(data)
  local msg = midi.to_msg(data)

  if msg.type == "note_on" then
    table.insert(notes, msg.note)

  elseif msg.type == "note_off" then
    for i, note in ipairs(notes) do
      if note == msg.note then
        table.remove(notes, i)
        break
      end
    end
    if #notes == 0 then
      engine.gate(0)
    end
  end
end
```

### Sustain Pedal Handling

```lua
local sustain_enabled = false
local sustain_notes = {}

function midi_event(data)
  local msg = midi.to_msg(data)

  if msg.type == "note_on" then
    engine.hz(musicutil.note_num_to_freq(msg.note))
    engine.gate(1)
    if sustain_enabled then
      sustain_notes[msg.note] = true
    end

  elseif msg.type == "note_off" then
    if not sustain_enabled then
      engine.gate(0)
    elseif sustain_notes[msg.note] then
      sustain_notes[msg.note] = nil
    end

  elseif msg.type == "cc" and msg.cc == 64 then
    -- Sustain pedal
    sustain_enabled = msg.val > 63
    if not sustain_enabled then
      -- Release all sustained notes
      sustain_notes = {}
      engine.gate(0)
    end
  end
end
```

### Multi-Channel MIDI

```lua
function midi_event(data)
  local msg = midi.to_msg(data)
  local channel = msg.ch  -- Channel 1-16

  if msg.type == "note_on" then
    -- Route to different engines based on channel
    if channel == 1 then
      engine1.hz(musicutil.note_num_to_freq(msg.note))
      engine1.gate(1)
    elseif channel == 2 then
      engine2.hz(musicutil.note_num_to_freq(msg.note))
      engine2.gate(1)
    end
  end
end
```

## MIDI Implementation Reference

norns MIDI functions (see https://monome.org/docs/norns/api/#midi):

```lua
midi.connect(device_num)        -- Connect to device
midi.disconnect(device_num)     -- Disconnect
midi.to_msg(raw_data)           -- Parse MIDI bytes

-- MIDI device methods
device:note_on(note, velocity, channel)
device:note_off(note, channel)
device:cc(cc_number, value, channel)
device:program_change(program, channel)
device:pitch_bend(value, channel)  -- -8192 to 8191
device:channel_pressure(value, channel)
```

## Debugging MIDI

```lua
function midi_event(data)
  -- Print raw MIDI data
  print("Raw MIDI: " .. table.concat(data, ", "))

  local msg = midi.to_msg(data)
  print("Parsed: " .. msg.type .. " ch=" .. msg.ch)
end
```

## Common CC Numbers

```
1  - Modulation Wheel
7  - Channel Volume
10 - Pan
11 - Expression
64 - Sustain Pedal
65 - Portamento
91 - Reverb
93 - Chorus
```

## Reference

- MIDI API: https://monome.org/docs/norns/api/#midi
- MIDI Specification: https://www.midi.org/specifications
- Example scripts: https://github.com/monome/dust

## Common Issues

**No MIDI received**: Check device is connected and `midi.connect()` uses correct device number
**Wrong note values**: Verify channel number matches controller
**Lagging response**: Reduce redraw frequency if MIDI handling is in redraw
