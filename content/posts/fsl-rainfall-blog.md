---
title: "Federated Split Learning for IoT Rainfall Prediction: Lessons from Building One End-to-End"
date: 2026-05-18
draft: false
---

This project started as a coursework assignment, but the more I worked on it, the more it began to resemble a small distributed system in its own right — with all the usual problems that come with one.

## The Problem

Weather stations are everywhere now. The cheap ones live on rooftops, balconies, and farms, and they collectively produce more environmental data than any single observatory ever did. The conventional way to turn that data into a useful rainfall predictor is to ship everything to a central server and train a model there.

That approach has three issues that get harder to ignore the closer you look:

- **Privacy** — raw sensor streams expose location and behavioural patterns.
- **Bandwidth** — residential uplinks are slow, and weather data is continuous.
- **Reliability** — a single training server is a single point of failure.

Federated Split Learning (FSL) offers a more reasonable middle ground. The model is split: clients run the early layers locally and transmit only the intermediate activations. Raw data never leaves the device, and the server only ever sees compact 64-dimensional vectors.

But FSL trades one bottleneck for another. Instead of moving raw data once, you now move activations and gradients *every step*, plus periodic full-model synchronisation. On a real IoT network, that round-trip cost can easily dominate training time.

This is the problem the project tackles.

## What We Built

The system is a fully working FSL platform with three pieces that matter:

**A split LSTM.** A 2-layer LSTM encoder runs on each client, reading a 48-hour sliding window across five meteorological variables (temperature, humidity, pressure, wind speed, rain). It outputs a 64-dim activation. The server hosts an MLP with two heads — one classifies rain vs. no-rain, the other regresses the rainfall amount. Focal loss handles the obvious class imbalance, since most hours in Newcastle are, predictably, not raining.

**A gRPC protocol with four RPCs.** `Register`, `Forward`, `Synchronize`, `NotifyCompletion`. Nothing exotic. The interesting design choice is that the `Forward` RPC carries scheduler directives in its response — so the server can change a client's compression mode or sync interval mid-training without a separate control channel.

**An adaptive scheduler.** This is the part the project is actually about. The server measures each client's latency with an EMA, and uses it to pick compression mode (`float32` → `float16` → `int8`) and synchronisation interval ρ together. Fast clients keep full precision and sync every step. Slow clients drop to int8 and sync every three steps. The two knobs are controlled jointly rather than independently, which turns out to matter.

## What Surprised Me

A few things didn't go the way I expected.

**INT8 compression is essentially free, accuracy-wise.** Going from 256 B/step (float32) to 68 B/step (int8) doesn't move AUPRC outside seed-level variance. I went in assuming there would be a visible trade-off; there isn't one for this task.

**Sparser synchronisation can actually help.** Setting ρ=3 instead of ρ=1 cut synchronisation traffic by about 53% and often *improved* AUPRC slightly. Less frequent averaging seems to give each client more room to specialise on its local distribution before getting pulled back to the mean.

**The Pi deployment was where the design earned its keep.** In simulation, all four strategies look fine. On 11 Raspberry Pi 4B clients over a real wide-area link (~21–24 ms RTT), the adaptive scheme reduced activation payload by 87% and runtime jitter from ±688 s to ±10 s. That's not a percentage point on a metric — that's the difference between a system that finishes when it's supposed to and one that doesn't.

## Reflections

The thing FSL papers tend to under-explain is how much of the engineering effort goes into the *infrastructure* around the model: barrier coordination with timeouts so a missing client doesn't stall everything, paired checkpointing so offline evaluation actually matches the training-time state, a reproducible experiment matrix so 17 scenarios × 3 seeds doesn't turn into a manual graveyard of mismatched configs.

In the end, the most useful thing I took away wasn't the result that joint adaptive control beats independent tuning — that's a satisfying conclusion but a fairly intuitive one. It was the reminder that, for a system like this, the model is the easy part. Everything else is where the project lives or dies.

---

Source code: [github.com/Neltharion1127/csc8114](https://github.com/Neltharion1127/csc8114)