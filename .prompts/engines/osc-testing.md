# SuperCollider Engine Testing and OSC Debugging

Comprehensive guide to testing SuperCollider engines locally and debugging via OSC.

## Overview

Norns communicates with SuperCollider via **OSC (Open Sound Control)** messages:
- **Norns (Matron)** runs in Lua, sends OSC commands to audio server
- **SuperCollider (Crone)** runs audio server on localhost:57110
- Messages are sent over UDP on loopback interface (127.0.0.1)

By understanding OSC, you can test engines without a Norns device.

## Part 1: Local Testing Setup

### Prerequisites

- SuperCollider installed (free, open source)
- SuperCollider IDE (includes editor and audio server)
- Your engine code (.sc file)

### Step 1: Start SuperCollider Server

1. Open SuperCollider IDE
2. Evaluate the boot code:
   ```supercollider
   s.boot;
   ```
3. Wait for "SuperCollider 3.x.x server booted" message
4. Server now listens on port 57110

### Step 2: Load Your Engine

Paste your engine code into IDE and evaluate:

```supercollider
Engine.register(\MyEngine);

(
SynthDef(\MyEngine_default, {
  | out=0, gate=1, hz=440, amp=0.1 |
  var sig, env;
  env = Env.adsr(0.01, 0.1, 0.7, 0.5);
  env = EnvGen.kr(env, gate, doneAction: 2);
  sig = Saw.ar(hz) * amp * env;
  Out.ar(out, sig!2);
}).add;
);
```

Wait for completion message before continuing.

### Step 3: Create Test Synth

```supercollider
x = Synth(\MyEngine_default, [\hz, 440, \amp, 0.1, \gate, 0]);
```

Check audio output - you should hear nothing (gate is 0).

### Step 4: Send Commands to Synth

Set parameters:
```supercollider
x.set(\gate, 1);           // Gate on - sound starts
x.set(\hz, 880);           // Change pitch
x.set(\hz, 440);           // Back to original
x.set(\gate, 0);           // Gate off - sound stops
```

You should hear the synth respond to each command.

### Step 5: Test Edge Cases

```supercollider
// Test extreme values
x.set(\hz, 20);            // Very low
x.set(\hz, 20000);         // Very high
x.set(\amp, 0);            // Silent
x.set(\amp, 1);            // Loud (be careful!)

// Test rapid changes
Task({
  10.do({
    x.set(\hz, rrand(100, 1000));
    0.1.wait;
  });
}).play;
```

## Part 2: OSC Communication

### How OSC Works

OSC message format:
```
/address arg1 arg2 arg3 ...
```

Example:
```
/n_set 1000 "hz" 440.0 "amp" 0.1
```

Breaks down as:
- `/n_set`: "set node parameters" command
- `1000`: Node ID (synth instance)
- `"hz"`: Parameter name
- `440.0`: Parameter value
- etc.

### Norns OSC Messages

Norns sends engine commands as OSC to SC:

```
/cmd/engine/<name> arg1 arg2 ...
```

Example for setting frequency:
```
/cmd/engine/hz 440.0
```

### Testing with OSC CLI

Install `oscsend` (on macOS via Homebrew):
```bash
brew install liblo
```

Send a test message:
```bash
oscsend 127.0.0.1 57110 /c_set 1 440.0
```

Or use Python:

```python
from pythonosc import udp_client

client = udp_client.SimpleUDPClient("127.0.0.1", 57110)
client.send_message("/c_set", [1, 440.0])
```

## Part 3: Debugging Techniques

### Technique 1: Print Statements

Add print statements to your SynthDef:

```supercollider
SynthDef(\MyEngine_debug, {
  | out=0, gate=1, hz=440, amp=0.1 |

  (hz.postln);      // Print frequency to post window
  (amp.postln);     // Print amplitude

  var sig, env;
  env = Env.adsr(0.01, 0.1, 0.7, 0.5);
  env = EnvGen.kr(env, gate, doneAction: 2);
  sig = Saw.ar(hz) * amp * env;
  Out.ar(out, sig!2);
}).add;
```

Open the Post window to see printed values:
```
Cmd+Shift+P (macOS)
Ctrl+Shift+P (Linux/Windows)
```

### Technique 2: Monitor Parameter Changes

Track when parameters change:

```supercollider
SynthDef(\MyEngine_debug, {
  | out=0, gate=1, hz=440, amp=0.1 |

  var sig, env;

  // Debugging: print when parameters change
  Changed.kr(hz).poll(label: "hz changed");
  Changed.kr(gate).poll(label: "gate changed");

  env = Env.adsr(0.01, 0.1, 0.7, 0.5);
  env = EnvGen.kr(env, gate, doneAction: 2);
  sig = Saw.ar(hz) * amp * env;
  Out.ar(out, sig!2);
}).add;
```

### Technique 3: Audio Scope

Visualize the audio signal:

```supercollider
// Open scope
s.scope;

// Now play synth - watch the waveform
x = Synth(\MyEngine_default, [\hz, 440, \gate, 1]);
```

Watch for:
- Clipping (waveform hitting top/bottom)
- No signal (flat line)
- Unexpected shapes (filter issues, wrong oscillator)

### Technique 4: CPU Monitoring

Check CPU usage:

```supercollider
// Query all nodes
s.queryAllNodes;

// Get average CPU
s.avgCPU;

// Get peak CPU
s.peakCPU;
```

For Norns, keep CPU under 40% to avoid glitches.

### Technique 5: Frequency Analysis

Check which frequencies are present:

```supercollider
// Start FFT analysis
Freq.ugens;

// Or use a simple spectrum analyzer
x = {
  var input = In.ar(0, 2);
  var chain = FFT(LocalBuf(2048), input);
  chain.magnitude.sqrt.log.linlin(-80.dbamp.log, 0, 0, 1);
}.play;
```

### Technique 6: A/B Testing Parameters

Compare two settings:

```supercollider
// Test 1: Original
x = Synth(\MyEngine_default, [\hz, 440, \filter_freq, 5000]);
2.wait;
x.free;

// Test 2: Modified
x = Synth(\MyEngine_default, [\hz, 440, \filter_freq, 1000]);
```

Listen for differences and take notes.

## Part 4: Common Issues and Solutions

### Issue: Synth Doesn't Play

**Symptom**: No sound even though parameters are set.

**Diagnosis**:
```supercollider
// Check if synth is alive
x.alive;  // Should return 'true'

// Check if audio is enabled
s.volume.postln;  // Check volume level

// Scope the output
s.scope;  // Look for signal
```

**Solutions**:
1. Verify `Out.ar(out, ...)` in SynthDef
2. Check `gate=1` is set
3. Verify audio driver is working: `s.boot` succeeded
4. Increase amplitude: try `amp=1` instead of `amp=0.1`

### Issue: Parameter Changes Don't Work

**Symptom**: `set()` doesn't affect sound.

**Diagnosis**:
```supercollider
// Check if parameter name matches
x.set(\wrong_name, 440);  // Should print warning

// Verify parameter is being used
// (add print statements to SynthDef)
```

**Solutions**:
1. Double-check spelling: `hz` not `frequency`
2. Verify parameter is audio-rate or control-rate as needed
3. Ensure parameter name matches in both Lua and SC

### Issue: Audio Clicks and Pops

**Symptom**: Audible glitches when changing parameters.

**Diagnosis**:
```supercollider
// Is amplitude too hot?
s.scope;  // Watch for clipping

// Are envelope changes abrupt?
// Use smoothing
var freq = Lag.kr(hz, 0.1);  // Smooth frequency changes
```

**Solutions**:
1. Reduce amplitude: keep under 0.2
2. Use `Lag.kr()` to smooth parameter changes
3. Use `doneAction: 2` for proper cleanup
4. Reduce filter modulation depth

### Issue: High CPU Usage

**Symptom**: Audio glitches, dropout, or `s.avgCPU` > 40%.

**Diagnosis**:
```supercollider
s.queryAllNodes;  // See how many synths running

s.avgCPU.postln;  // What's the load?

// Disable reverb/effects
// Remove expensive UGens (FFT, granular, etc.)
```

**Solutions**:
1. Use simpler oscillators (Saw cheaper than expensive FM)
2. Avoid `TGrains`, `FFT` on Norns
3. Use control-rate where possible
4. Reduce polyphony
5. Limit effects processing

### Issue: Envelope Not Working

**Symptom**: Note plays continuously, doesn't stop when gate goes 0.

**Diagnosis**:
```supercollider
// Check envelope is gate-responsive
env = Env.adsr(...);
env = EnvGen.kr(env, gate, doneAction: 2);  // Must have gate
```

**Solutions**:
1. Add `gate` parameter to Env.adsr call
2. Add `doneAction: 2` to free synth
3. Verify gate is actually being set to 0
4. Check envelope release time (might be very long)

## Part 5: Integration Testing

### Simulate Norns Behavior

Once local testing works, simulate how Norns will control it:

```supercollider
// Simulate a Norns script playing notes
Task({
  var note_nums = [60, 62, 64, 65, 67];  // MIDI notes

  note_nums.do({ |note|
    var freq = note.midicps;
    x = Synth(\MyEngine_default, [\hz, freq, \amp, 0.1, \gate, 0]);
    0.01.wait;
    x.set(\gate, 1);
    0.5.wait;
    x.set(\gate, 0);
    0.2.wait;
  });
}).play;
```

### Test Parameter Ranges

Verify your engine handles the full parameter range:

```supercollider
x = Synth(\MyEngine_default);

// Sweep frequency range
Task({
  linlin(0, 1, 20, 20000).do({ |hz|
    x.set(\hz, hz);
    0.01.wait;
  });
}).play;
```

## Part 6: Norns Device Testing

Once local testing is complete:

1. Place engine in `~/dust/code/my_script/engine/`
2. Restart Norns
3. Load script that uses engine
4. Check `~/.norns/matron.log` for errors:
   ```bash
   tail -f ~/.norns/matron.log
   ```

5. Use `engine.list_commands()` in script to verify:
   ```lua
   function init()
     engine.name = "MyEngine"
     print(engine.list_commands())
   end
   ```

## Troubleshooting Checklist

- [ ] SC server boots successfully
- [ ] Engine code evaluates without errors
- [ ] Test synth creates without errors
- [ ] Audio output works with s.scope
- [ ] Parameters respond to set() calls
- [ ] All parameter ranges work (min, max, edge cases)
- [ ] No audio clicks or pops
- [ ] CPU usage acceptable (< 40%)
- [ ] Envelope triggers on gate
- [ ] Engine syntax valid Lua when wrapped
- [ ] Engine loads on Norns device
- [ ] Norns matron.log shows no errors

## Reference

- SuperCollider Server: https://doc.sccode.org/Server/
- SuperCollider UGens: https://doc.sccode.org/Browse.html#UGens
- OSC Specification: http://opensoundcontrol.org/
- Norns Engine API: https://monome.org/docs/norns/api/#engine
- SC Debugging: https://doc.sccode.org/Guides/Debugging.html
