# SuperCollider Engine Patterns and Examples

Working code examples for common engine development patterns.

## Pattern 1: Simple Monophonic Synth

A basic synth with gate, frequency, and amplitude control.

```supercollider
Engine.register(\Simple);

(
SynthDef(\Simple_default, {
  | out=0, gate=1, hz=440, amp=0.1 |

  var sig, env;

  env = Env.adsr(0.01, 0.1, 0.7, 0.5);
  env = EnvGen.kr(env, gate, doneAction: 2);

  sig = Saw.ar(hz) * amp * env;

  Out.ar(out, sig!2);
}).add;
);
```

**Usage in Lua**:
```lua
function init()
  engine.name = "Simple"
  params:add_number("hz", "Pitch", 20, 440, 20000)
  params:set_action("hz", function(x) engine.hz(x) end)
end

function key(n, z)
  if z == 1 then
    engine.gate(1)
  else
    engine.gate(0)
  end
end
```

## Pattern 2: Polyphonic Synthesizer

Multiple voices for simultaneous notes.

```supercollider
Engine.register(\PolySynth, trickle: false);

(
SynthDef(\PolySynth_default, {
  | out=0, gate=1, hz=440, amp=0.1, attack=0.01, decay=0.1, sustain=0.7, release=0.5 |

  var sig, env, filter_env;

  // Amplitude envelope
  env = Env.adsr(attack, decay, sustain, release);
  env = EnvGen.kr(env, gate, doneAction: 2);

  // Filter envelope
  filter_env = Env.adsr(attack, decay, sustain, release);
  filter_env = EnvGen.kr(filter_env, gate);

  // Oscillator with filter sweep
  sig = Saw.ar(hz) * amp * env;
  sig = LPF.ar(sig, 1000 + (filter_env * 4000));

  Out.ar(out, sig!2);
}).add;
);
```

**Key differences**:
- `trickle: false` enables multiple synths (polyphony)
- Each note creates a new synth instance
- Each instance has independent envelope
- Use for instruments where overlapping notes matter

## Pattern 3: FM Synthesis

Frequency modulation for rich, evolving tones.

```supercollider
Engine.register(\FM);

(
SynthDef(\FM_default, {
  | out=0, gate=1, hz=440, amp=0.1, ratio=2, index=1, attack=0.01, release=0.5 |

  var sig, env, mod;

  env = Env.adsr(attack, 0.1, 0.7, release);
  env = EnvGen.kr(env, gate, doneAction: 2);

  // Modulator (changes at control rate, cheap)
  mod = SinOsc.ar(hz * ratio, mul: hz * index);

  // Carrier (modulated by modulator)
  sig = SinOsc.ar(hz + mod) * amp * env;

  Out.ar(out, sig!2);
}).add;
);
```

**Parameters**:
- `ratio`: Modulation ratio (2 = octave up)
- `index`: Modulation depth (higher = more harmonic content)

## Pattern 4: Granular Synthesis

Triggering short grain envelopes for textural sounds.

```supercollider
Engine.register(\Granular);

(
SynthDef(\Granular_default, {
  | out=0, gate=1, hz=10, amp=0.1, grain_size=0.1, density=1 |

  var sig, env, grain_env;

  env = Env.new([0, amp, amp, 0], [0.05, 1, 0.5], 'sine');
  env = EnvGen.kr(env, gate, doneAction: 2);

  // Trigger grains at hz rate
  grain_env = Env.perc(0.001, grain_size);

  sig = TGrains.ar(
    2,                          // num channels
    Impulse.ar(hz),             // trigger rate
    Buffer.read(0),             // buffer (not used here)
    1,                          // rate
    0 + LFNoise0.ar(hz)         // position randomization
  );

  Out.ar(out, sig * env * 0.5);
}).add;
);
```

**Note**: Granular synthesis is CPU-intensive on norns. Use simpler patterns when possible.

## Pattern 5: Additive Synthesis

Combining multiple harmonics for complex tones.

```supercollider
Engine.register(\Additive);

(
SynthDef(\Additive_default, {
  | out=0, gate=1, hz=440, amp=0.1 |

  var sig, env;
  var harm1, harm2, harm3, harm4;

  env = Env.adsr(0.05, 0.2, 0.6, 0.5);
  env = EnvGen.kr(env, gate, doneAction: 2);

  // Create harmonics at different amplitudes
  harm1 = SinOsc.ar(hz) * 0.5;          // fundamental
  harm2 = SinOsc.ar(hz * 2) * 0.3;      // 2nd harmonic
  harm3 = SinOsc.ar(hz * 3) * 0.2;      // 3rd harmonic
  harm4 = SinOsc.ar(hz * 4) * 0.1;      // 4th harmonic

  sig = (harm1 + harm2 + harm3 + harm4) * amp * env;

  Out.ar(out, sig!2);
}).add;
);
```

**Advanced**: Use `Array.series()` to generate many harmonics programmatically.

## Pattern 6: Audio Effect Processor

Process incoming audio (effects engine).

```supercollider
Engine.register(\Reverb);

(
SynthDef(\Reverb_default, {
  | out=0, mix=0.5, size=0.5, damp=0.5 |

  var sig, input;

  // Read from audio input
  input = In.ar(0, 2);

  // Apply reverb
  sig = FreeVerb.ar(input, mix, size, damp);

  // Write to output
  Out.ar(out, sig);
}).add;
);
```

**Key**:
- Use `In.ar(0, 2)` to read audio input
- No gate/envelope needed (processes continuously)
- No `doneAction` needed (synth runs continuously)

## Pattern 7: Sample Playback

Load and play back samples (use sparingly on norns).

```supercollider
Engine.register(\Sampler);

(
SynthDef(\Sampler_default, {
  | out=0, gate=1, bufnum=0, hz=1, amp=0.1 |

  var sig, env;

  env = Env.adsr(0.01, 0.1, 0.7, 0.5);
  env = EnvGen.kr(env, gate, doneAction: 2);

  // Play back buffer at variable rate
  sig = PlayBuf.ar(
    2,              // num channels
    bufnum,         // buffer number
    BufRateScale.ir(bufnum) * hz,  // playback rate
    gate,           // trigger
    0,              // start position
    Done.freeSelf   // cleanup when done
  );

  Out.ar(out, sig * amp * env);
}).add;
);
```

**Warning**: Sample loading uses disk I/O. Avoid on norns if possible.

## Pattern 8: LFO Modulation

Modulate parameters with low-frequency oscillators.

```supercollider
Engine.register(\LFOSynth);

(
SynthDef(\LFOSynth_default, {
  | out=0, gate=1, hz=440, amp=0.1, lfo_rate=1, lfo_amount=100 |

  var sig, env, lfo;

  env = Env.adsr(0.01, 0.1, 0.7, 0.5);
  env = EnvGen.kr(env, gate, doneAction: 2);

  // Low-frequency oscillator for modulation
  lfo = SinOsc.ar(lfo_rate);

  // Modulate frequency with LFO
  sig = Saw.ar(hz + (lfo * lfo_amount)) * amp * env;

  Out.ar(out, sig!2);
}).add;
);
```

**Usage**:
- `lfo_rate`: How fast the LFO oscillates (Hz)
- `lfo_amount`: How much the LFO modulates frequency

## Pattern 9: Resonant Filter

Add resonance for classic subtractive synthesis.

```supercollider
Engine.register(\ResonantFilter);

(
SynthDef(\ResonantFilter_default, {
  | out=0, gate=1, hz=440, amp=0.1, filter_freq=5000, filter_res=0.5 |

  var sig, env;

  env = Env.adsr(0.01, 0.1, 0.7, 0.5);
  env = EnvGen.kr(env, gate, doneAction: 2);

  sig = Saw.ar(hz) * amp * env;

  // Resonant low-pass filter (0-1 resonance range)
  sig = RLPF.ar(sig, filter_freq, 1 - filter_res);

  Out.ar(out, sig!2);
}).add;
);
```

**Parameters**:
- `filter_freq`: Cutoff frequency (Hz)
- `filter_res`: Resonance (0-1, 1 = max resonance)

## Parameter Best Practices

**Control-Rate Parameters** (efficient):
- `Env.kr()`: Envelopes
- `SinOsc.kr()`: LFOs
- `LFNoise0.kr()`: Random values

**Audio-Rate Parameters** (more expensive):
- `Saw.ar()`: Oscillators
- `SinOsc.ar()`: Oscillators
- Use sparingly

**Parameter Ranges**:
```supercollider
| hz=440,           // frequency in Hz
  amp=0.1,          // amplitude 0-1
  ratio=2,          // ratio multiplier
  index=1,          // modulation index
  lfo_rate=1,       // LFO frequency Hz
  lfo_amount=100,   // LFO modulation amount
  filter_freq=5000, // filter cutoff Hz
  filter_res=0.5 |  // resonance 0-1
```

## Debugging Snippets

**Print parameter values**:
```supercollider
| hz=440 |
(hz.postln);  // Prints to SC post window
```

**Monitor CPU usage**:
```supercollider
s.queryAllNodes;  // Shows all running synths
s.avgCPU;         // CPU percentage
```

**Simple test synth**:
```supercollider
x = Synth(\Engine_default, [\hz, 440, \amp, 0.1, \gate, 1]);
x.set(\hz, 880);           // Change parameter
x.set(\gate, 0);           // Release
```

## Common Gotchas

**Synth won't play**: Missing `doneAction: 2`
```supercollider
// BAD
env = EnvGen.kr(env, gate);

// GOOD
env = EnvGen.kr(env, gate, doneAction: 2);
```

**Parameters don't change**: Using wrong rate
```supercollider
// BAD (audio-rate, very expensive)
sig = Saw.ar(SinOsc.ar(lfo_rate) * 100);

// GOOD (control-rate, efficient)
sig = Saw.ar(hz + (SinOsc.kr(lfo_rate) * 100));
```

**Audio glitches**: Output too hot
```supercollider
// BAD
sig = Saw.ar(hz) * 0.5;     // Up to 0.5

// GOOD
sig = Saw.ar(hz) * 0.1;     // Keep below 0.2
```

## Reference

- SuperCollider UGen docs: https://doc.sccode.org/
- Engine API: https://monome.org/docs/norns/api/#engine
- Example engines: ../norns/lua/engine/
