---
layout: post
title: "Motorizing a roller blind with a servo, an ATmega328P, and a bead chain"
date: 2026-08-11
tags: [electronics, avr, atmega328p, servo, 3d-printing, home-automation]
---

I have a west-facing window with a roller blind on a bead chain. Direct afternoon light,
and no power outlet anywhere near the frame. I wanted the blind to go up and down at the
press of a button, and I didn't want to buy a commercial blind motor.

This is what I built, in four parts: the mechanical side, the electrical side, the
firmware, and the two weeks I lost to a fault that wasn't where I thought it was.

## The problem

Commercial motorized blinds solve this, but they solve it by replacing the blind. The
mechanism is integrated into the roller tube, so you're buying a new blind and paying
someone to fit it. I already have a blind that works. The only thing missing is something
to pull the chain.

The constraints that actually shaped the design:

- **No mains power.** Nothing within reach of the window, and I'm not running a cable
  across the wall. Battery, with the option to add a solar panel later.
- **Don't modify the blind.** Whatever I build has to drive the existing bead chain and
  come off without a trace.
- **Instant response.** A press should move it now, not on a schedule.

Everything downstream follows from the battery constraint. It's why the servo is asleep
most of the time, why there's a MOSFET cutting its power at idle, and why current draw
turned out to be the central problem of the whole build.

## Mechanical

The drive is a continuous-rotation servo turning a sprocket that sits in the bead chain
loop. The blind's own weight helps on the way down and fights you on the way up.

I measured that asymmetry before designing anything, by hooking a scale to the chain and
pulling: about **1 kgf to lower, 2.5 kgf to raise**. Raising needs roughly twice the
torque. That single number decides whether a given servo is viable, and it's worth ten
minutes with a luggage scale before you commit to a part.

The servo I used is rated 6 kg-cm stall at 6 V. On 4 AA alkalines it lowers the blind
easily and raises it with visible effort — enough margin to work, not enough to be
comfortable. The fix I'm leaning toward is a 2:1 reduction between servo and sprocket,
trading speed I don't need for torque I do.

Mounting is a printed servo holder screwed to the wall, reusing two holes that were already
there. The bead-chain sprocket is also printed.

I did sketch an alternative: the whole unit floating inside an enclosure, held up by the
chain itself — a driven sprocket sitting in the bottom bight of the loop with two pairs of
idler gears pinching the strands above it. No wall fixing at all, and the chain circulates
through a unit that stays put. It's a nicer idea and I've parked it, because it only works
if the assembly is light, and the battery pack is currently the heaviest thing in the
build.

## Electrical

A standalone ATmega328P on its internal 8 MHz oscillator — no crystal, no board, no USB —
programmed with Arduino-as-ISP. For a device that reads one button and generates one pulse
train, a bare chip in a socket costs a couple of dollars and draws almost nothing.

The circuit:

- Button on **D2** (INT0), switched to ground, using the internal pullup
- Servo signal on **D9**
- **D8** driving the gate of a low-side N-channel MOSFET that switches the servo's ground,
  through a 330 Ω series resistor, with a 10 kΩ pulldown to keep the gate defined while the
  chip sleeps
- A 1N4007 in the Vcc line and a 100 nF ceramic at the chip

The MOSFET exists purely for battery life. A servo left connected idles at a few milliamps
forever, which on a device that actually moves for maybe a minute a day is the dominant
drain. Cutting its ground at rest removes that entirely.

One thing to get right: the MOSFET has a body diode, and if you install it backwards that
diode happily powers the servo whether the gate is driven or not. The tell is measuring the
servo's supply while the gate is off and finding it sitting near rail voltage instead of
near zero. Mine read 4.4 V on a 4.7 V rail before I caught it, and 0.43 V after.

**Open issue:** running from a 5-cell NiMH pack at 6.7 V, I measure 6.39 V at the chip's
Vcc pin. The ATmega328P's absolute maximum is 6.0 V. Only about 0.31 V is dropping across
the 1N4007, less than I'd expect, and the chip is running out of spec. That needs a second
diode in series or a proper regulator before this build becomes permanent.

## Software

The behaviour is deliberately minimal. One button. Press it, the blind travels. Press it
again during travel, it stops where it is. Press it after a completed stroke, it goes the
other way.

Pulse widths are 1500 µs for stop, 1700 for up, 1300 for down, with a deadband around
centre so the servo genuinely holds still rather than creeping.

Two pieces are doing real work:

**Sleep.** The chip spends essentially all its life in AVR power-down sleep, woken by a
level-triggered interrupt on INT0. During a stroke it drops into idle sleep between pulses.
On a battery build this isn't an optimization, it's the difference between months and days.

**Soft start.** Rather than commanding the target pulse width on the first pulse, the
firmware ramps from stop out to target over a few hundred milliseconds. This started as a
fix for the electrical problem below, but it's better mechanically too — the blind eases
into motion instead of jerking.

Travel is currently timed rather than sensed. There's no encoder and no limit switch; a
stroke just runs for a set duration. It's crude, and a stall-detection or current-sense
approach would be better, but timing is what works today.

## The debugging: everything I "found" that wasn't there

The build stopped working, repeatedly, and almost every fault I diagnosed was imaginary.

The real problem was **servo inrush dragging a shared battery rail below the chip's brownout
threshold**. I found that by scoping the rail during a button press: 5.98 V falling to
3.44 V and staying there for about 30 ms. That shape matters. I'd assumed a spike and was
about to fit a bulk capacitor, but a 30 ms sag isn't a spike — it's the pack's internal
resistance losing to sustained draw, and no reasonable capacitor rides that out. The fixes
that actually worked were the firmware soft start, and giving the servo its own pack with
grounds tied.

Getting there took two weeks because of measurement artifacts:

- I read **20 Ω from gate to source** and concluded I'd cooked the MOSFET. Testing the part
  out of circuit later showed OL — it was always fine. Resistance readings on a *powered*
  circuit are fiction; the meter injects its own current and infers resistance, so an
  external supply into that node makes the number meaningless.
- I read an apparent **continuity path from the pulldown to the positive rail**. Gone the
  moment the pack was disconnected.
- I read **~3 V at Vcc** and thought the chip was starved. Measured properly it was 4.51 V
  on a 5.02 V rail — a normal diode drop.
- I checked the **10 kΩ pulldown with a continuity beeper** and got nothing, and briefly
  believed it was disconnected. A 10 kΩ resistor is above the beeper's threshold. It will
  never beep. That reading carries no information at all.

And one that wasn't an artifact: after every component tested good individually, the
perfboard build still failed. I rebuilt the identical circuit on a breadboard and it worked
immediately. The fault is somewhere in my soldering and I never located it precisely — but
that's a useful stopping rule. Once every part tests good in isolation and the assembly
still fails, the assembly is the fault, and rebuilding on a substrate you trust beats
bisecting one you don't.

The pattern across all of it: the measurements that were load-bearing were taken on a scope
with the timebase set correctly, or on the bench with power disconnected. Everything I
measured quickly, live, with a multimeter, because it was convenient, was confidently wrong
and sent me somewhere.

## Where it is now

Working, on a breadboard, with the servo on a separate pack. Seven AA cells hanging off a
window frame is not a finished product, so the next steps are the 2:1 reduction to cut the
torque demand, condensing back to a single supply that can source the current, and fixing
that Vcc overvoltage before any of it goes back onto a soldered board.
