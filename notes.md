===== Web Audio API =====

[Web Audio API Spec](https://webaudio.github.io/web-audio-api)

// Format

( Audio Context )

To generate and process sounds using the Web Audio API, you create a series of Nodes that connect together to form a signal path.
```
Inputs -> Effects -> Destination
```

Before we can do anything we need to create an AudioContext. This is a global object that is available in most modern browsers on _any_ webpage.

```js
var audioContext = new AudioContext()
```

The OscillatorNode is particularly useful. It can generate an audio pitch at any frequency and all the basic wave shapes (sine, sawtooth, square, or triangle).

```js
var oscillator = audioContext.createOscillator()

oscillator.start(audioContext.currentTime)

oscillator.stop(audioContext.currentTime + 2) // stop after 2 seconds
```
Final code:

```js
var audioContext = new AudioContext()

var oscillator = audioContext.createOscillator()
// WaveForm Type
oscillator.type = 'sawtooth'
// Connect it to the speakers
oscillator.connect(audioContext.destination)
// Play
oscillator.start(audioContext.currentTime)
oscillator.stop(audioContext.currentTime + 2)

```

## Setting Oscillator Frequency (or not)
All oscillator nodes have a property called `frequency`.

Although, if you try and set it directly, it will have no effect.
```js
// this doesn't work
oscillator.frequency = 200
```
That's because oscillator.frequency is an instance of [AudioParam][AP].

### AudioParams

Most properties on Audio Nodes are instances of [AudioParam][AP]. They let you do all kinds of neat things with parameters, such as automation over time, and allow one AudioNode to modulate another's value.

We will get into more depth later but for now all you need to know is param.value = 123.

So to set the frequency of the oscillator to 880 Hz, you would do this:
```js
// woo finally something works!
oscillator.frequency.value = 880
```

## Mapping Frequencies to Musical Notes

```
                         -3  -1   1       4   6       9   11
                       -4  -2   0   2   3   5   7   8   10  12
  .___________________________________________________________________________.
  :  | |  |  | | | |  |  | | | | | |  |  | | | |  |  | | | | | |  |  | | | |  :
  :  | |  |  | | | |  |  | | | | | |  |  | | | |  |  | | | | | |  |  | | | |  :
  :  | |  |  | | | |  |  | | | | | |  |  | | | |  |  | | | | | |  |  | | | |  :
<-:  |_|  |  |_| |_|  |  |_| |_| |_|  |  |_| |_|  |  |_| |_| |_|  |  |_| |_|  :->
  :   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   :
  : A | B | C | D | E | F | G | A | B | C | D | E | F | G | A | B | C | D | E :
  :___|___|___|___|___|___|___|___|___|___|___|___|___|___|___|___|___|___|___:
    ^                           ^           ^               ^           ^
  220 Hz                      440 Hz      523.25 Hz       880 Hz     1174.65 Hz
(-1 Octave)                 (middle A)                 (+1 Octave)

```

Chromatic note frequencies are slightly tricky to calculate because the _frequency **doubles** every octave (12 semitones)_ you go up.

Changing the frequency from 440 to 880 would go up one octave, or +12 semitones.

The formula for this is `baseFrequency * Math.pow(2, noteOffset / 12).`

So if you wanted to transpose from middle A up 7 semitones to E, you would do the following:
```js
oscillator.frequency.value = 440 * Math.pow(2, 7 / 12) // 659.255...
```
This works going down as well. Here we transpose down 14 semitones to G:
```js
oscillator.frequency.value = 440 * Math.pow(2, -14 / 12) // 195.998...
```

### A Slightly Easier Way

Instances of OscillatorNode also have a [detune][DT] [AudioParam][AP], which
allows you to specify transposes in 100ths of semitones.

So using detune, if you wanted to transpose from middle A up 7 semitones to E, you would do the following:

```js
oscillator.detune.value = 700 // noteOffset * 100
```

Using `detune` is quite a bit simpler.


## Scheduling Events (play a short series of notes)

Like we saw in the previous example, we need to specify the start and stop time of each generator node. This is specified relative to `audioContext.currentTime` which is time in seconds since the `AudioContext` was first created.

Play a sound 3 seconds after AudioContext created and stop after 1 second:
```js
play(3, 1)

function play(at, duration) {
  var oscillator = audioContext.createOscillator()
  oscillator.connect(audioContext.destination)
  oscillator.start(at)
  oscillator.stop(at+duration)
}
```
But since we don't know how long the audio context has existed for, it's better to always schedule relative to the current time:
```js
play(3, 1)

function play(delay, duration) {
  var oscillator = audioContext.createOscillator()
  oscillator.connect(audioContext.destination)

  // add audioContext.currentTime
  oscillator.start(audioContext.currentTime + delay)
  oscillator.stop(audioContext.currentTime + delay + duration)
}
```


## Add a high-pass filter

We're going to modify the code so that the audio is filtered to remove all
frequencies lower than 10000 Hz.

### Biquad Filter Node

> The `BiquadFilterNode` interface represents a simple low-order filter, and is created using the `AudioContext.createBiquadFilter()` method. It is an AudioNode that can represent different kinds of filters, tone control devices, and graphic equalizers. A BiquadFilterNode always has exactly one input and one output.

```raw
Number of inputs	1
Number of outputs	1
Channel count mode	"max"
Channel count	2 (not used in the default count mode)
Channel interpretation	"speakers"
```

Use the [BiquadFilterNode][BQ] to shape the audio output frequencies.

```js
var filter = audioContext.createBiquadFilter()
filter.connect(audioContext.destination)
oscillator.connect(filter)
```

The most common [types][types] of filters are `'lowpass'`, `'highpass'` and `'bandpass'`.

#### Lowpass

Lowpass Filters allow signals below a specific [frequency][frequency] to _pass_
and attenuates all signals with a higher frequency.

```js
// filter out all frequencies above 500 Hz
filter.type = 'lowpass'
filter.frequency.value = 500
```

#### Highpass

Works the same as the lowpass, except the other way around. All signals with a frequency higher than frequency are allowed and anything lower is attenuated.

```js
// filter out all frequencies below 3000 Hz
filter.type = 'highpass'
filter.frequency.value = 3000
```

#### Bandpass
Only allows frequencies that are within a certain tolerance of frequency specified by [Q][Q].

The greater the Q value, the smaller the frequency band.
```js
// filter out all frequencies that are not near 1000 Hz
filter.type = 'bandpass'
filter.frequency.value = 1000
filter.Q.value = 1
```

#### Other filter types
```raw
Other filter types
lowshelf
highshelf
peaking
notch
allpass
```


## Modulating Audio Parameters

An [AudioParam][AudioParam] can be set to a specific [value][value] as we did in the previous lesson, but it can also be set to change over time.

Here we will ramp the filter [frequency][frequency] from `200` Hz to `6000` Hz over 2 seconds:
```js
// preset value to avoid popping on start
filter.frequency.value = 200

// schedule the start time
filter.frequency.setValueAtTime(200, audioContext.currentTime)

// ramp the value, and set end time!
filter.frequency.linearRampToValueAtTime(6000, audioContext.currentTime + 2)

```
#### [`setValueAtTime(value, time)`](https://developer.mozilla.org/en-US/docs/Web/API/AudioParam/setValueAtTime)

Schedules an instant change to the value of the AudioParam at a precise time, as measured against AudioContext.currentTime.

#### [`linearRampToValueAtTime(value, endTime)`](https://developer.mozilla.org/en-US/docs/Web/API/AudioParam/linearRampToValueAtTime)

Schedules a gradual linear change in the value of the AudioParam. The change starts at the time specified for the _previous_ event, follows a **linear ramp** to the new value given in the `value` parameter, and reaches the new value at the time given in the `endTime` parameter.

#### [`exponentialRampToValueAtTime(value, endTime)`](https://developer.mozilla.org/en-US/docs/Web/API/AudioParam/exponentialRampToValueAtTime)

Schedules a gradual exponential change in the value of the AudioParam. The change starts at the time specified for the _previous_ event, follows an **exponential ramp** to the new value given in the `value`parameter, and reaches the new value at the time given in the `endTime` parameter.


## Gain Node

You can use the [`GainNode`][GN] to change the output volume of sounds.

It has a single attribute, an [AudioParam][AudioParam].


```js
var amp = audioContext.createGain()
amp.connect(audioContext.destination)
oscillator.connect(amp)

// halve the output volume
amp.gain.value = 0.5
```

### Scheduling attack and release

Just like with `BiquadFilterNode.frequency` in the previous example, we can sweep the value of `amp.gain` over time. Though for the purposes of attack and release envelopes, it sounds a lot nicer (and is easier) to use `setTargetAtTime`.

### [`audioParam.setTargetAtTime(targetValue, startTime, timeConstant)`](https://developer.mozilla.org/en-US/docs/Web/API/AudioParam/setTargetAtTime)

Schedules the start of a change to the value of the AudioParam. The change starts at the time specified in `startTime` and **exponentially moves** towards the value given by the target parameter. The **exponential decay rate** is defined by the `timeConstant` parameter, which is a time measured in seconds.


### Attack

If we want to soften the **"attack"** of the sound (the start), we can sweep the value from `0` to `1`:

```js
amp.gain.value = 0
amp.gain.setTargetAtTime(1, audioContext.currentTime, 0.1)
```

### Release

If we want to soften the **"release"** of the sound (the tail end), we can sweep the value back to 0.

```js
var endTime = audioContext.currentTime + 2
amp.gain.setTargetAtTime(0, endTime, 0.2)
```

Keep in mind that if you are going to add a release envelope to a sound, the sound needs to **keep playing until the release sweep finishes**, otherwise it will just stop.

```js
// provide enough time for the exponential falloff
oscillator.stop(endTime + 2)
```


## Vibrato






[AP]: https://developer.mozilla.org/en-US/docs/Web/API/AudioParam "AudioParam"
[DT]: https://developer.mozilla.org/en-US/docs/Web/API/OscillatorNode/detune "Detune AudioParam"
[BQ]: https://developer.mozilla.org/en-US/docs/Web/API/BiquadFilterNode "Biquad Filter Node"
[types]: https://developer.mozilla.org/en-US/docs/Web/API/BiquadFilterNode/type "Types of filters"
[frequency]: https://developer.mozilla.org/en-US/docs/Web/API/BiquadFilterNode/frequency "Frequency"
[Q]: https://developer.mozilla.org/en-US/docs/Web/API/BiquadFilterNode/Q "Q (AudioParam)"
[value]: https://developer.mozilla.org/en-US/docs/Web/API/AudioParam/value "Value"
[GN]: https://developer.mozilla.org/en-US/docs/Web/API/GainNode "Gain Node"
