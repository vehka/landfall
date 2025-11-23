# Softcut: Multi-Voice Buffer Looping and Recording

## What is Softcut?

Softcut is a **built-in multi-voice sample playback and recording system** on norns. It provides:
- **6 independent voices** (play/record heads) with individual parameters
- **2 mono buffers** (~5 minutes 49 seconds each at 48kHz)
- **Real-time recording and overdubbing** with level control
- **Loop control** with smooth crossfading at loop points
- **Advanced filtering** (pre-record and post-playback)
- **Matrix mixing** with cross-patching between voices
- **File I/O** for loading/saving audio

**Key Advantage**: Built-in, optimized for norns hardware - no CPU overhead compared to custom scripting.

## When to Use Softcut

Consider softcut for any script involving:
- **Loopers**: Record and loop audio in real-time
- **Tape simulators**: Emulate multitrack tape behavior
- **Time-stretch/pitch-shift**: Variable-speed playback
- **Live sampling**: Record external audio and manipulate
- **Buffer-based effects**: Use buffers as effect processors
- **Granular synthesis**: Sample playback with loop points
- **Audiovisual loops**: Sync video to audio looping

**Don't use softcut if**:
- You just need to play pre-recorded samples (use built-in engines)
- No audio manipulation needed (keep it simple)
- Script doesn't interact with audio buffers

## Basic Workflow

### 1. Enable Voices

```lua
function init()
  -- Enable 6 voices (or however many you need)
  for i = 1, 6 do
    softcut.enable(i, 1)
  end

  -- Assign buffer to each voice (1 or 2)
  softcut.buffer(1, 1)  -- voice 1 uses buffer 1
  softcut.buffer(2, 1)  -- voice 2 uses buffer 1
  softcut.buffer(3, 2)  -- voice 3 uses buffer 2
  -- etc.

  -- Set output levels
  softcut.level(1, 1)   -- voice 1 at full volume
  softcut.level(2, 0.5) -- voice 2 at half volume

  softcut.reset()       -- Initialize state
end
```

### 2. Set Loop Boundaries

```lua
-- Define loop region
softcut.loop_start(voice, 0)           -- Start at beginning
softcut.loop_end(voice, 10)            -- End at 10 seconds
softcut.loop(voice, 1)                 -- Enable looping

-- Fade time when crossing loop boundary
softcut.fade_time(voice, 0.1)          -- 100ms crossfade
```

### 3. Control Playback

```lua
-- Basic playback
softcut.play(voice, 1)                 -- Play
softcut.play(voice, 0)                 -- Stop

-- Position control
softcut.position(voice, pos)           -- Jump to position
local current_pos = softcut.position(voice)  -- Get current position

-- Playback rate (1 = normal speed)
softcut.rate(voice, 1.0)               -- Normal speed
softcut.rate(voice, 0.5)               -- Half speed
softcut.rate(voice, 2.0)               -- Double speed
softcut.rate(voice, -1.0)              -- Reverse

-- Smooth parameter transitions
softcut.rate_slew_time(voice, 0.1)     -- 100ms ramp to new rate
```

### 4. Control Recording

```lua
-- Enable recording
softcut.rec(voice, 1)                  -- Record from input

-- Recording levels
softcut.rec_level(voice, 1.0)          -- Record new audio at full level
softcut.pre_level(voice, 0.9)          -- Keep 90% of existing buffer (overdub)

-- Input source
softcut.input(1, 1)                    -- Voice 1: input from ADC (audio in)
softcut.input(1, 2)                    -- Voice 1: input from engine
softcut.input(1, 3)                    -- Voice 1: input from tape (feedback)
```

### 5. Apply Filters

```lua
-- Pre-record filter (filters input before recording)
softcut.pre_filter_fc(voice, 5000)     -- Cutoff at 5 kHz
softcut.pre_filter_rq(voice, 2)        -- Q/resonance

-- Post-playback filter (filters output after playback)
softcut.post_filter_fc(voice, 8000)    -- Cutoff at 8 kHz
softcut.post_filter_rq(voice, 1)

-- Filter types (set on output)
softcut.post_filter_on(voice, 1)       -- Enable filter
softcut.post_filter_dry(voice, 1)      -- Tap point: dry
softcut.post_filter_lp(voice, 1)       -- Tap point: low-pass
softcut.post_filter_hp(voice, 0)       -- Tap point: high-pass (off)
```

### 6. Matrix Mixing (Cross-Patching)

```lua
-- Route voice output to another voice's input
softcut.patch_in(voice_to, voice_from, 1.0)  -- 1.0 = full level
softcut.patch_in(2, 1, 0.5)                   -- Send voice 1 to voice 2 at half level
softcut.patch_in(3, 2, -0.5)                  -- Inverted polarity
```

## Common Patterns

### Pattern 1: Simple Looper

```lua
local recording = false
local rec_voice = 1
local play_voice = 2

function init()
  softcut.enable(rec_voice, 1)
  softcut.enable(play_voice, 1)

  softcut.buffer(rec_voice, 1)
  softcut.buffer(play_voice, 1)

  -- Set loop to first 10 seconds
  softcut.loop_start(rec_voice, 0)
  softcut.loop_end(rec_voice, 10)
  softcut.loop(rec_voice, 1)
  softcut.fade_time(rec_voice, 0.05)

  softcut.loop_start(play_voice, 0)
  softcut.loop_end(play_voice, 10)
  softcut.loop(play_voice, 1)
  softcut.fade_time(play_voice, 0.05)

  softcut.rec_level(rec_voice, 0.95)   -- Slight feedback
  softcut.pre_level(rec_voice, 0.95)   -- Keep existing audio
  softcut.level(play_voice, 1)
end

function key(n, z)
  if z == 1 then
    if n == 1 then
      -- Toggle recording
      recording = not recording
      softcut.rec(rec_voice, recording and 1 or 0)
      softcut.play(rec_voice, 1)
    elseif n == 2 then
      -- Play recorded loop
      softcut.play(play_voice, 1)
    elseif n == 3 then
      -- Clear buffer
      softcut.buffer_clear()
    end
  end
  redraw()
end

function redraw()
  screen.clear()
  screen.move(2, 10)
  screen.text(recording and "REC" or "LOOP")
  screen.update()
end
```

### Pattern 2: Multitrack Tape Simulator

```lua
local num_tracks = 4

function init()
  for i = 1, num_tracks do
    softcut.enable(i, 1)
    softcut.buffer(i, (i <= 2) and 1 or 2)  -- Distribute across 2 buffers
    softcut.level(i, 1)
    softcut.rec_level(i, 0.95)
    softcut.pre_level(i, 0.95)
  end

  -- All voices loop the same region
  softcut.loop_start(1, 0)
  softcut.loop_end(1, 30)
  for i = 1, num_tracks do
    softcut.loop(i, 1)
    softcut.fade_time(i, 0.1)
  end

  softcut.reset()
end

-- Each encoder controls one track's level
function enc(n, d)
  local voice = n
  local level = softcut.level(voice)
  softcut.level(voice, math.max(0, math.min(2, level + d * 0.1)))
  redraw()
end

-- Keys toggle recording on each track
function key(n, z)
  if z == 1 then
    local voice = n
    local is_recording = softcut.rec(voice) == 1
    softcut.rec(voice, is_recording and 0 or 1)
    redraw()
  end
end
```

### Pattern 3: Time-Stretch/Pitch-Shift

```lua
local voice = 1
local playback_rate = 1

function init()
  softcut.enable(voice, 1)
  softcut.buffer(voice, 1)
  softcut.loop_start(voice, 0)
  softcut.loop_end(voice, 5)  -- 5 second loop
  softcut.loop(voice, 1)
  softcut.fade_time(voice, 0.05)
  softcut.rate_slew_time(voice, 0.2)  -- Smooth transitions
  softcut.play(voice, 1)
end

-- Encoder 1: Pitch control (±24 semitones)
function enc(n, d)
  if n == 1 then
    -- Pitch shift via rate change
    local semitones = math.max(-24, math.min(24, playback_rate * 12 + d))
    playback_rate = semitones / 12
    softcut.rate(voice, 2 ^ playback_rate)  -- Exponential for musical intervals
    redraw()
  end
end

function redraw()
  screen.clear()
  screen.move(2, 10)
  local cents = math.floor(playback_rate * 1200)
  screen.text("Pitch: " .. cents .. " cents")
  screen.update()
end
```

### Pattern 4: Granular Looping

```lua
local voice = 1
local grain_size = 0.2  -- 200ms grains
local grain_rate = 1.0

function init()
  softcut.enable(voice, 1)
  softcut.buffer(voice, 1)

  -- Set grain boundaries as loop points
  softcut.loop_start(voice, 0)
  softcut.loop_end(voice, grain_size)
  softcut.loop(voice, 1)
  softcut.fade_time(voice, 0.01)  -- Very tight crossfade for grain effect

  softcut.play(voice, 1)
  softcut.rate(voice, grain_rate)
end

function key(n, z)
  if z == 1 then
    if n == 1 then
      -- Tighter grains
      grain_size = math.max(0.05, grain_size - 0.05)
      softcut.loop_end(voice, grain_size)
    elseif n == 2 then
      -- Larger grains
      grain_size = math.min(2, grain_size + 0.05)
      softcut.loop_end(voice, grain_size)
    end
  end
  redraw()
end
```

## Important Notes

**Sample Rate**: All audio must be **48 kHz**. Mismatched sample rates cause pitch issues.

**Buffer Size**: Two mono buffers, ~5:49 each. Plan loop regions accordingly.

**Input Sources**:
- `1` = ADC (audio input)
- `2` = Engine output
- `3` = Tape (feedback from other voices)

**Output Filtering**: Configure filter **taps** (which signal to output):
- Dry: unfiltered
- Low-pass: filter output
- High-pass: high-pass filter output
- Band-pass: band-pass filter output
- Band-reject: notch filter output

**Feedback Loops**: Be careful with voice cross-patching - can create feedback loops!

## Reference

- Official Softcut docs: https://monome.org/docs/norns/softcut/
- Softcut API: https://monome.org/docs/norns/api/#softcut
- Softcut Studies: https://monome.org/docs/norns/studies/ (search "softcut")
- norns source: ../norns/lua/core/softcut.lua
