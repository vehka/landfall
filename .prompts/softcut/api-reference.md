# Softcut API Reference

Complete reference for all softcut functions based on official Norns API documentation.

## Enable/Disable

```lua
softcut.enable(voice, state)       -- Enable/disable voice processing (0/1)
softcut.reset()                     -- Reset all softcut state to defaults
softcut.defaults()                  -- Returns table of default values per voice
softcut.params()                    -- Returns controlspec arrays for parameter integration
```

## Buffer Management

```lua
softcut.buffer(voice, buffer_num)        -- Assign voice to buffer (1 or 2)
softcut.buffer_clear()                   -- Clear all buffers completely
softcut.buffer_clear_channel(channel)    -- Clear single channel (1 or 2)

-- File I/O
softcut.buffer_read_mono(path, start_in_buffer, start_in_file, duration)   -- Load mono file
softcut.buffer_read_stereo(path, start_in_buffer, start_in_file, duration) -- Load stereo file
softcut.buffer_write_mono(path, start, duration)                            -- Save mono
softcut.buffer_write_stereo(path, start, duration)                          -- Save stereo

-- Buffer copying
softcut.buffer_copy_mono(src_start, dst_start, duration)    -- Copy mono section
softcut.buffer_copy_stereo(src_start, dst_start, duration)  -- Copy stereo section
```

## Playback Control

```lua
softcut.play(voice, state)          -- Enable/disable playback (0=off, 1=on)
softcut.rate(voice, rate)           -- Set playback speed (1.0 = normal, negative = reverse)
softcut.position(voice, value)      -- Set playback head position (seconds)
softcut.query_position(voice)       -- Query current playback position

-- Smooth transitions
softcut.rate_slew_time(voice, time) -- Exponential slew for rate changes (seconds, -60dB convergence)
```

## Loop Control

```lua
softcut.loop(voice, state)          -- Enable/disable looping (0=one-shot, 1=loop with crossfade)
softcut.loop_start(voice, pos)      -- Set loop start point (seconds, starts at 0)
softcut.loop_end(voice, pos)        -- Set loop end point (seconds)
softcut.fade_time(voice, time)      -- Set crossfade duration at loop boundary (seconds)
```

## Recording Control

```lua
softcut.rec(voice, state)           -- Enable/disable recording (0=off, 1=on)
softcut.rec_level(voice, amp)       -- Set input signal amplitude before buffer write
softcut.pre_level(voice, amp)       -- Set preserve level for overdubbing (existing audio retention)
softcut.rec_offset(voice, value)    -- Adjust record head position offset
```

## Output Control

```lua
softcut.level(voice, amp)           -- Set output amplitude per voice
softcut.pan(voice, pos)             -- Pan voice in stereo field (-1 to +1)

-- Smooth transitions
softcut.level_slew_time(voice, time)  -- Exponential slew for level changes (seconds)
softcut.pan_slew_time(voice, time)    -- Exponential slew for pan changes (seconds)
```

## Input Routing

```lua
softcut.level_input_cut(channel, voice, amp)  -- Route ADC channel to voice input
                                               -- channel: 1 or 2 (ADC input)
                                               -- voice: destination voice
                                               -- amp: routing amplitude
```

## Voice-to-Voice Mixing (Cross-Patching)

```lua
softcut.level_cut_cut(src, dst, amp)  -- Route voice output to another voice input
                                        -- src: source voice (1-6)
                                        -- dst: destination voice (1-6)
                                        -- amp: routing amplitude
```

## Voice Synchronization

```lua
softcut.voice_sync(dst, src, offset)   -- Synchronize playback positions between voices
                                         -- dst: destination voice
                                         -- src: source voice
                                         -- offset: playback position offset
```

## Pre-Record Filtering

Applied to audio **before recording** to the buffer.

```lua
softcut.pre_filter_fc(voice, fc)       -- Cutoff frequency (Hz)
softcut.pre_filter_rq(voice, rq)       -- Reciprocal Q-factor (bandwidth control)
softcut.pre_filter_fc_mod(voice, amount)  -- Rate-dependent filter modulation (0 to 1)
```

## Post-Playback Filtering

Applied to audio **after playback** from buffer. Each filter type has an amplitude control:

```lua
softcut.post_filter_fc(voice, fc)      -- Cutoff frequency (Hz)
softcut.post_filter_rq(voice, rq)      -- Reciprocal Q-factor (bandwidth control)

-- Output tap selection (amplitude levels for each signal path)
softcut.post_filter_dry(voice, amp)    -- Dry (unfiltered) signal level
softcut.post_filter_lp(voice, amp)     -- Low-pass filtered signal level
softcut.post_filter_hp(voice, amp)     -- High-pass filtered signal level
softcut.post_filter_bp(voice, amp)     -- Band-pass filtered signal level
softcut.post_filter_br(voice, amp)     -- Band-reject (notch) filtered signal level
```

## Phase Polling & Callbacks

```lua
softcut.phase_quant(voice, quantum)    -- Set update interval for phase polling
                                        -- quantum: beat divisor (1/4 = quarter note)

softcut.poll_start_phase(voice, quantum, callback)  -- Start phase position polling
                                                     -- callback receives (voice, phase)

softcut.poll_stop_phase(voice)          -- Stop phase position polling

softcut.event_phase(func)               -- Register callback for phase updates
                                        -- func receives (voice, phase)
```

## Buffer Rendering & Callbacks

```lua
softcut.render_buffer(channel, start, duration, num_samples)  -- Request buffer content snapshot
                                                               -- Returns samples array

softcut.event_render(func)              -- Register callback for rendered buffer data
                                        -- func receives rendered samples
```

## Advanced Modulation

```lua
softcut.recpre_slew_time(voice, time)  -- Exponential slew for rec/pre level changes (seconds)
```

## Important Notes

**1-Based Indexing**: All voice indices are 1-based (voice 1-6) following Lua convention.

**Time Quantities**: Start at zero (not 1-based). Positions, durations are in seconds.

**Slew Time Specification**: `-60dB convergence time` - the time for exponential transitions to reach -60dB of target difference.

**Filter Q/RQ**: Reciprocal Q-factor controls bandwidth. Higher values = narrower bandwidth.

**Pre_level**: Controls how much existing buffer content is preserved when recording.
- 1.0 = preserve all (100% overdub)
- 0.9 = 90% preserve + 10% new input
- 0.0 = discard all existing (full replace)

**Rate**: Direction and speed of playback
- 1.0 = normal forward speed
- 0.5 = half speed (one octave down)
- 2.0 = double speed (one octave up)
- -1.0 = normal reverse speed
- Negative values = reverse playback

## Typical Initialization

```lua
function init()
  -- Enable voices
  for i = 1, 6 do
    softcut.enable(i, 1)
  end

  -- Configure voices
  for i = 1, 6 do
    softcut.buffer(i, 1)              -- Assign to buffer 1
    softcut.level(i, 1)               -- Full output level
    softcut.play(i, 0)                -- Start not playing
    softcut.rec(i, 0)                 -- Start not recording

    softcut.loop_start(i, 0)
    softcut.loop_end(i, 30)           -- 30 second loop
    softcut.loop(i, 1)                -- Enable looping
    softcut.fade_time(i, 0.1)         -- 100ms crossfade

    softcut.rec_level(i, 0.95)        -- 95% input level
    softcut.pre_level(i, 0.95)        -- 95% preserve existing
    softcut.level_input_cut(1, i, 0.1)  -- Route ADC to voice at 0.1 level
  end

  softcut.reset()  -- Initialize state
end
```

## Input Sources

Unlike the simplified input() API I documented before, softcut uses `level_input_cut()` and `level_cut_cut()` for explicit routing:

**ADC (Audio Input)**:
```lua
softcut.level_input_cut(1, voice, amplitude)  -- Route ADC channel 1 to voice
softcut.level_input_cut(2, voice, amplitude)  -- Route ADC channel 2 to voice
```

**Engine (Voice-to-Voice)**:
```lua
softcut.level_cut_cut(src_voice, dst_voice, amplitude)  -- Route voice to voice
```

There is no implicit "tape" or "engine" input - you explicitly route what you want.

## Output Filtering Details

Each filter tap has an amplitude control. You can combine multiple taps:

```lua
-- Output only dry signal
softcut.post_filter_dry(voice, 1)
softcut.post_filter_lp(voice, 0)
softcut.post_filter_hp(voice, 0)

-- Output mix of dry + low-pass
softcut.post_filter_dry(voice, 0.5)   -- 50% dry
softcut.post_filter_lp(voice, 0.5)    -- 50% low-pass
```

## Common Mistakes

**Wrong slew_time usage**: Slew times are for smooth transitions
```lua
-- BAD: treating slew_time like fade_time
softcut.level_slew_time(voice, 1.0)
softcut.level(voice, 0)  -- Won't instantly mute

-- GOOD: use for smooth parameter changes
softcut.rate_slew_time(voice, 0.1)
softcut.rate(voice, 2)   -- Smoothly ramp to double speed
```

**Not initializing loop boundaries**: Always set them
```lua
function init()
  softcut.loop_start(1, 0)
  softcut.loop_end(1, 10)
  softcut.loop(1, 1)
  -- Bad things happen without these
end
```

**Confusing pre_level with rec_level**:
- `rec_level`: How loud the input is when recording
- `pre_level`: How much existing audio is preserved (overdub control)

**Extreme routing levels**: Feedback loops need careful level management
```lua
-- DANGEROUS: creates infinite feedback
softcut.level_cut_cut(1, 2, 1.0)
softcut.level_cut_cut(2, 1, 1.0)

-- SAFER: use low levels
softcut.level_cut_cut(1, 2, 0.1)
softcut.level_cut_cut(2, 1, 0.1)
```

## Reference

- Official Softcut API: https://monome.org/docs/norns/api/modules/softcut.html
- Softcut Studies: https://monome.org/docs/norns/softcut/
- Norns Source: ../norns/lua/core/softcut.lua
