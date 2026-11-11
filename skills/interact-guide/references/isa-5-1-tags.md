# Reading a repère — a suggestion, never an authority

Industrial documents name their instruments and their equipment with letter
codes. These tables help you **guess what a repère probably is** so you can
propose something concrete and ask. They decide nothing: every site has its own
convention, and the convention wins. Treat a reading from this file exactly as
the inspector treats its Modbus register-role defaults — a starting suggestion,
overridable, confirmed before anything is written.

Two different families turn up in the same document, and they do not come from
the same place. Say which one you are reading when you propose something.

## Family 1 — instrument letters (ISA-5.1)

ISA-5.1 identifies instruments: the first letter is the measured **or
initiating** variable, the following letters indicate function and modifiers.
It is the family behind a transmitter, a switch or a control valve. Not every
first letter is a measurement — `H` is a hand-operated device and `Y` an event
or state, so a repère opening with one of those names an action, not a reading.

| First letter | Measured or initiating variable | Following letters | Function            |
| ------------ | ------------------------------- | ----------------- | ------------------- |
| `L`          | Level                           | `T`               | Transmitter         |
| `F`          | Flow                            | `I`               | Indicator           |
| `P`          | Pressure                        | `S`               | Switch              |
| `T`          | Temperature                     | `V`               | Valve               |
| `A`          | Analysis                        | `C`               | Controller          |
| `S`          | Speed                           | `A`               | Alarm               |
| `H`          | Hand (manual)                   | `H` / `L`         | High / low modifier |
| `Y`          | Event, state or presence        |                   |                     |
| `X`          | Unclassified                    |                   |                     |

The same letter means different things by position: `H` opening a repère is a
hand-operated device, `H` after the function letter is the _high_ modifier.

Combinations you are likely to meet, each a **suggestion to confirm**:

| Repère      | Probably                         | Probably reads as |
| ----------- | -------------------------------- | ----------------- |
| `LIT`       | Level indicating transmitter     | ANA, measurement  |
| `FT`        | Flow transmitter                 | ANA, measurement  |
| `PT` / `TT` | Pressure / temperature transmit. | ANA, measurement  |
| `LSL`       | Level switch, low threshold      | TOR, contact      |
| `LSH`       | Level switch, high threshold     | TOR, contact      |
| `XV`        | On/off valve                     | TOR, command      |
| `FV`        | Control valve on a flow loop     | ANA, command      |

## Family 2 — equipment codes (site convention)

These are **not** ISA-5.1 instrument letters. They abbreviate a machine, and
each plant keeps its own list. `P` as an ISA-5.1 first letter means pressure,
while `PU` in this family means a pump — which is exactly why you must say
which family you are reading before proposing anything.

| Code  | Usually  | Code | Usually        |
| ----- | -------- | ---- | -------------- |
| `PU`  | Pump     | `TK` | Tank           |
| `AG`  | Agitator | `CV` | Conveyor       |
| `MOT` | Motor    | `CP` | Compressor     |
| `FN`  | Fan      | `HX` | Heat exchanger |

A code absent from this table is not an error and not an invitation to guess:
ask what it stands for.

## How to use this while configuring

1. **Propose, with the reading shown.** "`LSL101` looks like a low-level switch
   — a digital contact — is that right for this site?" Never state it flatly.
2. **Let the document outrank the letters.** A label, a unit or a register type
   in the source document is evidence; these tables are a prior. When they
   disagree, follow the document and say so.
3. **Never rename on the strength of a repère.** A tag becomes an IEC
   identifier, and a correction is proposed and confirmed, never silent.
4. **Never qualify a channel from letters alone.** The kind comes from the
   register or node capability and the user's confirmation, one line at a time.
   A repère that suggests a measurement on a writable register is a question to
   ask, not a conflict to resolve on your own.
5. **A numbered repère carries no hierarchy.** `101-PU-131` says nothing
   reliable about the ISA-95 area or work centre; read the topology in Atlas.
