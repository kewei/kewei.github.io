---
title: Debugging GNSS Code-Phase Drift in Acquisition
description: When sampling frequency is not integer multiples of CA code frequency, non-coherent acquisition over long signal needs to take consideration of it. This article explains why that matters, how a fractional sample error accumulates, and how to distinguish this problem from other possible causes of code-phase drift.
layout: post
author: kewei
date: 2026-09-08 21:23:00 +0200
categories:
  - GNSS
  - Rust
tags:
  - Acquisition
  - C/A
pin: true
math: true
mermaid: true
comments: true
image:
  path: /assets/img/2026-09-08/gnss_code_phase.jpeg
  alt: Code phase in GNSS receiver acquisition
---


When implementing GPS C/A-code acquisition, a surprisingly problem can appear when processing consecutive 1-ms blocks.

You may observe something like:

```text
Block 0:  Code Phase = 1617 samples
Block 1:  Code Phase = 1616 samples
Block 2:  Code Phase = 1615 samples
Block 3:  Code Phase = 1615 samples
Block 4:  Code Phase = 1614 samples
Block 5:  Code Phase = 1615 samples
Block 6:  Code Phase = 1615 samples
Block 7:  Code Phase = 1614 samples
Block 8:  Code Phase = 1612 samples
Block 9:  Code Phase = 1613 samples
```
After 10 blocks (10ms), the code phase drifts $4$ samples.

At first this can look like a GPS signal problem, a navigation-bit problem, or an error in the correlation algorithm.

However, one of the first things to check is much simpler:

> **Does one GPS C/A-code period correspond to an integer number of ADC samples?**


This article explains why that matters, how a fractional sample error accumulates, and how to distinguish this problem from other possible causes of code-phase drift.

---

## 1. GPS C/A code repeats every 1 ms

The GPS L1 C/A code has:

- 1023 chips
    
- chip rate = 1.023 MHz
    
- code period = 1 ms
    

Therefore:

$T_\text{C/A} = \frac{1023}{1.023\times10^6} = 1\text{ ms}.$

So the code repeats every 1 ms:

```text
C/A code:

|---------------- 1 ms ----------------|
| 1023 chips                            |
|                                      |
| 0 1 2 3 ... 1022                     |
|--------------------------------------|

|---------------- 1 ms ----------------|
| 1023 chips                            |
|--------------------------------------|
```

But receiver measures time in **samples**, not in chips.

---

## 2. The sampling frequency may not give an integer number of samples per millisecond

Suppose the receiver sampling frequency is:

`fs=16,367,600 Hz`.

The number of samples occurring during exactly 1 ms is:

`Nsamples=16,367,600×0.001= 16,367.6`.

But a digital signal cannot contain 0.6 of a sample.

Therefore, if we create a fixed-size 1-ms processing block, we might round this to:

`round⁡(16,367.6)=16,368`.

So we might be tempted to write:

```rust
let samples_per_ms = 16_368;
```

and process the signal as:

```text
block 0: 16,368 samples
block 1: 16,368 samples
block 2: 16,368 samples
block 3: 16,368 samples
...
```

This is where the problem starts.

Because the actual sampling clock produces:

16,367.6 samples per millisecond.

But our processing assumes:

16,368 samples per millisecond.

The difference is:

$16,368−16,367.6=0.4$

So every 1-ms block introduces an apparent timing error of:

$0.4 sample/ms$

The error is tiny after one millisecond.

But it accumulates.

After 10 ms: 4.0 samples.

After 100 ms: 40 samples.

This is why a small error in the number of samples per code period can become very visible during longer processing.

---

## 3. Visualizing the problem

The actual GPS code epochs occur every 1 ms.

Conceptually:

```text
Actual time:

0 ms             1 ms             2 ms             3 ms
 |----------------|----------------|----------------|
```

In samples, the boundaries are:

```text
0
16367.6
32735.2
49102.8
```

Of course, the ADC cannot give us a sample at index `16367.6`.

Our integer sample indices look like:

```text
0
16368
32736
49104
```

The difference grows:

```text
Actual boundary       Processing boundary       Error

0                      0                         0
16367.6                16368                     +0.4
32735.2                32736                     +0.8
49102.8                49104                     +1.2
65470.4                65472                     +1.6
81838.0                81840                     +2.0
...
163676.0               163680                    +4.0
```

After 10 ms, the processing boundary is four samples away from the ideal continuous-time boundary.

This is exactly the scale of drift that can appear in a code-phase measurement.


---

## 4. Do not confuse block size with the actual sampling clock

A common mistake is to think:

> "I take `fs / 1000` samples, so every block is exactly 1 ms."

That is only true if `fs / 1000` is an integer.

For:

```text
fs = 16_367_600 Hz
```

we have:

```text
fs / 1000 = 16_367.6
```

not:

```text
16_368
```

The ADC produces samples according to the real sampling clock.

The fact that our software has to use integer sample indices does not change the physical sampling rate.

This distinction is fundamental:

```text
Physical sampling:

16,367.6 samples / ms


Software block size:

16,368 samples / block
```

They are not exactly the same thing.

---

## 5. The correct way to determine block boundaries

In long time non-coherent acquisition, instead of repeatedly doing:

```rust
let samples_per_ms = 16_368;

for block in 0..num_blocks {
    let start = block * samples_per_ms;
    let end = start + samples_per_ms;
}
```

use the continuous-time relationship to calculate each boundary.

For example:

```rust
let start =
    ((block_index as f64) * f_sampling / 1000.0).round() as usize;

let end =
    (((block_index + 1) as f64) * f_sampling / 1000.0).round() as usize;
```

For the example sampling frequency, this produces approximately:

```text
Block 0: 16368 samples
Block 1: 16367 samples
Block 2: 16368 samples
Block 3: 16367 samples
...
```

The exact sequence depends on the rounding operation, but the important property is:

> **The fractional 0.6 sample is not discarded at every block boundary.**

The long-term average remains consistent with the actual sampling frequency.

---

## 6. The same principle applies to the local C/A code

There is another side to the problem.

The local C/A code must have a consistent relationship with the receiver samples.

For sample $n$, the ideal code phase is:

$\phi[n] = \phi_0+ n\frac{f_\text{code}}{f_s}$.

The C/A chip index is:

$k[n] = \left\lfloor \phi[n] \right\rfloor \bmod 1023$.

For example:

```rust
let chip = (
    initial_code_phase
        + n as f32 * code_rate / f_sampling
).floor() as usize % 1023;
```

---

## 7. Do not independently reset an inaccurate timing model every millisecond

A dangerous implementation pattern is:

```rust
for block in blocks {
    let local_code = generate_ca_code_samples(...);

    // Correlate block
}
```

if `generate_ca_code_samples()` implicitly assumes that every block starts at exactly the same sample/code timing relationship.

The GPS C/A code does repeat every 1 ms, but your **sample clock and the code clock are not necessarily represented by an integer number of samples per code period**.

The safer approach is to maintain the relationship between absolute sample time and code phase.

For example:

```rust
let mut code_phase = initial_code_phase;

for block in signal_blocks {
    let local_code = generate_code(
        prn,
        code_phase,
        block.len(),
        code_rate,
        f_sampling,
    );

    // Correlate...

    code_phase +=
        block.len() as f64 * code_rate / f_sampling;

    code_phase %= 1023.0;
}
```

Alternatively, calculate code phase directly from the absolute sample index.

The idea is:

```text
absolute sample index
        │
        ▼
continuous sample time
        │
        ▼
continuous code phase
        │
        ▼
C/A chip index
```

rather than independently restarting all timing calculations for every block.

---

## 8. A very useful debugging experiment

Before changing the algorithm, measure the code-phase peak independently for every 1-ms block.

For example:

```text
Block 0 → correlation → peak
Block 1 → correlation → peak
Block 2 → correlation → peak
...
Block 9 → correlation → peak
```

Record:

```text
block     peak_index
--------------------
0         1617
1         1616
2         1615
3         1615
4         1614
5         1615
6         1615
7         1614
8         1612
9         1613
```

Then compare the measured drift with the theoretical drift:

$\Delta N = N_\text{used}-N_\text{actual}$.

For the example:

$\Delta N = 16,368-16,367.6 = 0.4$.

Therefore:

$\Delta N_{10ms}=10 * 0.4=4$.

If the measured drift is approximately four samples, you have a very strong indication that your block timing is the cause.

---

## 9. But don't assume every code-phase variation is a sample-rate problem

The 0.4 sample/ms effect is only one possible cause.

A good GNSS acquisition implementation should be debugged systematically.

There are several other possible causes.

---

### 1) The actual sampling frequency differs from the configured frequency

This is especially important with SDRs.

Suppose your software says:

```text
16,367,600 Hz
```

but the actual hardware clock is slightly different.

The difference might be caused by:

- oscillator tolerance
    
- SDR clock configuration
    
- external reference
    
- clock calibration
    
- resampling
    
- hardware driver behavior
    

For example, if the actual rate is:

16,367,500 Hz

but the software assumes:

16,367,600 Hz

the local code and received signal gradually become misaligned.

This is fundamentally a **sample-clock mismatch**.

---

### 2) FFT/IFFT indexing

GPS acquisition is frequently implemented using FFT-based circular correlation.

A typical operation is:

$R= IFFT \left( FFT(x) \cdot FFT(c)^* \right)$.

The result is circular correlation.

That means the correlation index does not automatically equal the intuitive code phase unless the indexing convention is handled correctly.

A one-time indexing mistake can produce a constant offset.

A more subtle mistake involving block shifts or circular shifts can make the apparent peak behave incorrectly across blocks.

Therefore, check:

- FFT length
    
- zero padding
    
- circular indexing
    
- conjugation
    
- code reversal
    
- relationship between IFFT index and code phase
    

If every block has approximately the same error, suspect indexing.

If the error changes with time, suspect timing or alignment.

---

### 3) Doppler mismatch

A GPS signal has Doppler shift.

During acquisition, the receiver searches over candidate Doppler frequencies.

If the local Doppler is wrong, coherent correlation loses energy.

For a long coherent integration, even a relatively small Doppler error can cause significant phase rotation.

If:

$Delta f=f_\text{actual}-f_\text{local}$,

then after integration time $T$, the phase error is approximately:

$\Delta\phi=2\pi\Delta fT$.

Therefore, longer coherent integrations require more accurate Doppler estimates.

However, if you are performing **non-coherent power accumulation**, Doppler error normally reduces the accumulated peak rather than directly creating a systematic code-phase drift.

So if the code phase moves by several samples over 10 ms, Doppler mismatch would not be the first suspect.