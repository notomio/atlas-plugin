# Say it the way the plant reads it

A skill that cites this file renders three things: what it read, what it
changed, and what was refused. This file is the only authority on that rendering. Do not
restate these rules inside a skill body — a copied rule drifts on its first
edit.

## Name the subject on the first line

The first line names what an operator would point at: an equipment repère, a
channel tag, a data-source name, a PLC name, a point name, or a standard
identifier. Never open with a UUID, a `UDT_…` type name or an `IO_…`
declaration name. The IEC identifier may follow **indented**, below, for
whoever opens Beremiz next.

## Qualify it in plant vocabulary

State the nature — **TOR** (digital) or **ANA** (analog) — then what the reader
needs in order to act:

- **Polarity** for a digital signal, said in words an operator reads — a direct
  contact, or an inverted one — never as a raw `direct` flag. What that flag
  means, and what it does not change, belongs to
  [signal meaning and selection](../../interact-guide/references/signals.md):
  read it there, and do not restate it here.
- **Unit** for an analog value, with the engineering range when the read
  returns one.
- **List** (`init` or `perm`) and **position** for a condition, and the clause
  position when a diagnostic targets one clause.

Read a repère's letters with
[ISA-5.1 letters and equipment codes](../../interact-guide/references/isa-5-1-tags.md).
They are a suggestion to confirm against the site convention, never a fact.

## Cite the document, never paraphrase it

When the user supplied a document, quote the source sentence under the
proposal, verbatim. Without a document, do not compose one — say the proposal
comes from the conversation. A sentence naming a repère, channel or field that
no read of the session resolves in the pinned version is a **named gap**, not
an operand to infer.

## State an effect from the response, never from the request

Report what the tool returned, not what was asked for. Say "nothing was
written" only when that operation guarantees it — a refusal describes that call
alone and undoes nothing before it. When a result is best-effort per row,
render it row by row, naming what was created, updated, left unchanged and
refused.

A preview is evidence, not a conclusion. Render the refusals it _does_ report —
they are real, and they spare a failed write. But never announce a clean apply
from a clean preview: each preview tool's own description names what it cannot
predict, so read it there, and say that a refusal may still surface on write.

## An unavailable reference is said, not repaired

When a read returns a `null` repère or tag for an operand or binding that still
carries its identifier, render it as **reference unavailable**, with that
identifier indented below. Never substitute a similarly named signal, and never
drop the line on the user's behalf. Naming the gap is the result.

## Proposing a register line

A line read from a register table is a proposal, and it carries **at most one
kind**. Say it in plant words — the register type, what the label and unit
suggest, and the kind you propose — never as a matrix lookup.

`ioManager` enters the line **only after that line is confirmed**. An
unconfirmed line is not an error and not a retry: it stays a **raw point**,
stored with its address and no channel, and the final report names it as a raw
point rather than omitting it. Say what the document did not give you — word
order, addressing base, a word register's data type — and ask, instead of
assuming a default.

## Rendering a dry-run composition

A composition check answers "is this PLC generable?". Diagnostics are a
successful answer, not an error.

1. **Name each diagnostic by its subject**, chosen from `subjectKind`:
   `equipment` → its repère · `data_source` → its name · `plc` → its name ·
   `io_channel` → its tag · `point` → its name · `standard_identifier` → the
   identifier. Use `subjectName` when it is present.
2. When `subjectName` is `null` because the variant carries only `plcId`, use
   the PLC name from `atlas_interact_plc_list`, **read at the start of the
   workflow**. Do not call it late to patch one line, and never print the id.
3. **Add the list and the position** whenever the diagnostic targets a
   condition, plus its clause position when it names one clause. An identifier
   collision carries its origins in `detail`, JSON text to parse, under
   `origins`. Name each by its `type`: `condition` — repère, list and
   position; `aggregate` — repère and list, no position; `declaration` — the
   clashing declaration's `name` and what it is (`origin`: global
   acknowledgement, IO, equipment data or equipment block).
4. **Say the motive from `code`, `reason` and `detail`**, in plant words.
   `message` is free server text: use it to understand, never display it as-is.
5. `generable: true` comes with no diagnostic: say the PLC is generable, with
   no diagnostic to name, then close as rule 7 says.
6. When `totalDiagnostics` exceeds the rows returned, the list was truncated:
   say how many diagnostics remain unnamed. They are not fine — fix what is
   named, then ask again.
7. Close every dry-run rendering with: **generable means neither deployed nor
   observed on the machine.** It says the program could be generated — not
   compiled, installed or run — and nothing about what the equipment does.

## Three worked examples

A proposed condition — repère first, nature, polarity, list, unit, source
sentence; the IEC names indented, never on the opening line:

> **PU101 — start permissive, list `init`, position 2, 2 clauses**
> Clause 1 — TOR: the tank low-level switch must read clear. Its channel is
> wired direct, so the contact's sense is the field sense.
> Clause 2 — ANA, measured level at or above 0.5 m (range 0–4 m).
> Source (functional analysis, § 3.2): "The pump shall not start when the low
> level switch is active or the measured level is below 0.5 m."
>
> - IEC, for Beremiz: `IO_LSL101`, `IO_LIT101`.

A proposed register line — one kind, plant words, the channel only after
confirmation, and the unconfirmed line named for what it stays:

> **Holding register 40012 — tank level transmitter, proposed as AI**
> ANA, unit m, raw 0–27648 mapped to 0–4 m. Addressing base and word order as
> you confirmed: `addr` 11, 0-based, `abcd`.
> One kind is proposed, never two. Confirm and this line also becomes IO
> Manager channel `LIT101`; decline and it stays a **raw point** — stored, with
> no channel, and reported as a raw point at the end.

Two separate dry-run answers — a blocked composition stops before generation,
so its diagnostics never share an answer with a generation diagnostic. Subject,
motive, and the list and position on the one that targets a condition:

> **Line 1 PLC — no standard version is pinned**
> Nothing composes until a published version is pinned to this PLC
> (`PLC_STANDARD_VERSION_MISSING`). This variant carries only the PLC id, so the
> name above comes from the `atlas_interact_plc_list` read at the start.

Once a version is pinned, a later check on the same PLC:

> **PU101 — a condition cannot be generated**
> List `init`, position 2, clause 1 (`EQUIPMENT_CONDITION_INVALID`,
> `unknown_field`): the clause reads a field its operand does not expose in the
> version this PLC pins.

Close each answer plainly: generable means neither deployed nor observed on the
machine.
