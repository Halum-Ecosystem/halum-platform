# halum-platform design (DRAFT)

Status: **DRAFT, revisit at M4.** These are temporary decisions made so that engineering can start. Expect them to change once the offline processing (M3) and live pipeline (M4) work has taught us more.

## 1. Problem statement

Biosignal sources differ in format, sample rate, units and hidden processing, so code written for one doesn't work with another. Version 0.1 lets any source feed one processing pipeline through a common format, demonstrated by simulated EMG driving a virtual hand.

## 2. Scope for v0.1

### Goals

- **G1:** Any source (simulator, replayed recording, later the Arduino) delivers data to processing in one common format, through a swappable adapter.
- **G2:** Every stream describes itself before data flows: channels, units, nominal sample rate, signal type, and processing already applied upstream.
- **G3:** Simulated EMG drives the virtual hand end to end within a latency budget of `TBD` ms, taken from the literature and measured.
- **G4:** Any recording can be replayed and gives the same output, because session metadata and config are saved with it.
- **G5:** It runs on a clean machine with no hardware.
- **G6:** If a source is lost, the output enters a defined safe state within `TBD` ms.

### Non-goals (for now)

- Hospital integration (HL7, FHIR, EHR export).
- Diagnosis or clinical decision support of any kind.
- Real patients or identifiable data.
- Streaming between machines, or cloud.
- Proprietary clinical device protocols.
- Writing our own time-sync protocol before evaluating Lab Streaming Layer.

### Stress cases

The design should survive these without being built specifically for them.

- **S1:** Add an ECG stream at 250 Hz beside EMG at 1000 Hz without changing the core.
- **S2:** Swap in a source that already filters its output (for example MyoWare's envelope output). The system should make the difference visible instead of silently producing wrong results.

## 3. Block diagram

```
            config file
                 |
                 v
 Source  -->  Processor(s)  -->  Output
 (adapter)    (in order)         (e.g. virtual hand)
     |
     +-->  Recorder (saves stream description, chunks and session)
```

Data flows left to right as chunks. Each stream's description is sent once, before its first chunk.

## 4. Data model

Three things move through or around the system.

| Thing | What it holds |
|---|---|
| **Stream description** (sent once, before data) | stream ID, signal type (EMG, ECG...), channel names, unit per channel, nominal sample rate, source device and firmware, adapter version, upstream processing already applied |
| **Chunk** (the data itself) | stream ID, a block of values (samples x channels), one timestamp per sample, a sequence number so dropped chunks are detectable |
| **Session** | session ID, start time, a saved copy of the config, free-text notes, no identifiable personal data |

Data moves in chunks rather than single samples. Sending 1000 separate samples per second costs far more overhead than sending 20 blocks of 50.

The upstream-processing field and the per-sample timestamps with sequence numbers are required from day one. They are cheap to add now and expensive to retrofit, and without them recordings can't be trusted and latency can't be measured.

## 5. Time

The host computer stamps data on arrival, using a monotonic clock (one that never jumps backwards when the system clock is adjusted). This is simple and good enough for one laptop.

Revisit at M4: evaluate Lab Streaming Layer and write this up as the first decision record.

## 6. Module interfaces

Three kinds of module, each with a small contract.

- **Source:** describe itself, start, give the next chunk (or nothing yet), stop.
- **Processor:** given an input stream description, say what its output looks like, then transform chunks.
- **Output:** start, accept chunks, enter a safe state on command or when data stops arriving, stop.

## 7. Wiring

A config file lists one source, an ordered list of processors and one output, each with its parameters. Swapping a module means editing the config, not the core.

## 8. Failure behaviour

- **Source lost or no data for `TBD` ms:** the output enters its safe state.
- **Dropped chunks:** detected from gaps in sequence numbers and counted.
- **Crashed module:** `TBD`.

## Open questions

- What latency budget does the literature support for myoelectric control?
- What is the virtual hand's safe state: hold its position, or open?
- What sample rate should the simulator use, and why?
- How are multiple streams with different rates combined (stress case S1)?

## Decision records to write

- Timestamps and sync: host clock vs Lab Streaming Layer (at M4).
- Why we don't use FHIR or IEEE 11073 internally: they are for exchange at the system's edges, so the inside stays simple and translates to them later.
