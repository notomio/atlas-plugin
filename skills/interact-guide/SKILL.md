---
name: interact-guide
description: Use when explaining Interact before or during configuration in Atlas. Triggers on "explain PLC and standard version", "what is a point versus a channel", "DI DO AI AO", "Modbus versus OPC-UA", "FC03", "polarity", "raw or engineering value", "scaling", "explique le câblage", "constante, signal IO ou équipement", "explique la conduite de cette Work Unit", "explique le GRAFCET", "que fait la séquence". An explanation request never authorizes a mutation; route conduct edits to configure-workunit-program and equipment CI/CP to configure-conditions.
---

# Understand Interact

Explain the requested concept in the user's language, with a short example
adapted to their question. Read the relevant bundled reference:

- [PLC, version, object and binding origins](references/plc-and-bindings.md).
- [Data sources, points and protocols](references/points-and-protocols.md).
- [DI/DO/AI/AO, polarity and scaling](references/signals.md).
- [ISA-5.1 letters and equipment codes, as a suggestion](references/isa-5-1-tags.md).

For an explanation alone, use these references without any mutation. Do not
create, associate, pin, qualify, wire, generate or deploy anything. Do not turn
an example into a proposed configuration unless the user requests that work.

If the user asks about an actual equipment's interface, follow the
[connector and read/retry procedure](../configure-plc/references/session-and-recovery.md)
and use `atlas_interact_get_equipment_configuration` to read it. Only present
names, types and fields returned for its pinned version. With no PLC, no pin,
an unavailable reference or a missing read tool, name the missing prerequisite;
continue explaining general concepts without inventing the missing interface.

Keep examples hypothetical: no tenant ids, assumed register map or fixed list
of standard inputs. A DO/AO logical echo is the previous cycle's state, not
physical confirmation that a motor is running or a valve has moved.

## Conduite d'une Work Unit

A Work Unit's conduct is one saved document per Work Unit, with a revision:
its phase state diagram, the links between states, the GRAFCET sequences those
links carry and the phase CI/CP.

- **States**: Ready (`0`), Locked (`-1`) and numbered running states. A
  **link** moves the phase from one state to another; it carries a
  **sequence** or stays immediate. The **abort** sequence has priority and the
  reserved value `-2`.
- **GRAFCET** (IEC 60848): numbered **steps** (the initial one is drawn as a
  double square), **actions** that assign commandable UDT fields while the step
  is active, and **transitions** whose **receptivity** must hold to move on —
  condition clauses, a clause held for a duration, or a step wait. In Atlas the
  GRAFCET is a specification; the emitted code is ST.

To answer about an actual conduct or one of its sequences, resolve the Work
Unit with `atlas_search` and `types: ["workUnit"]`. Verify `type`, `id`,
`siteId`, `identifier` and `target` in `results`, disambiguating with `totals`
when the list is limited. The input `siteId` favors that site without filtering
out others. Then read its verified id with `atlas_interact_get_workunit_program`.
Explain from the returned rows only:
`valueText` and `receptivityText` give each action and transition in one
notation, and a reference written `<id>.FIELD` no longer resolves on the
conduct's PLC — name its diagnostic rather than guessing the equipment.
`hasProgram: false` means no conduct is saved. This read never edits the
conduct; offer its verified Interact page through
[open-page](../open-page/SKILL.md). An explicit request to edit the conduct,
sequences or phase CI/CP goes to
[configure-workunit-program](../configure-workunit-program/SKILL.md), subject
to the required tools being exposed; an explanation alone stays read-only.

A host that supports MCP Apps may also show the result as a read-only view: one
tab for the state diagram, then one per link that carries a sequence, in number
order, then the abort sequence if defined; transitions and actions carry the
same `receptivityText` and `valueText`, diagnostics are outlined on the states
and links and listed in full under the drawing. Links are drawn as right-angle
routes and the user may pan and zoom. When the user selects a state, link, step
or transition there, a short description of it reaches you as context: answer
about that element from the rows. The view never edits the conduct, and its
absence changes nothing to what the rows tell you.

When configuration is requested, use [configure-plc](../configure-plc/SKILL.md)
for the existing PLC and [configure-equipment](../configure-equipment/SKILL.md)
for progressive bindings. These are separate actions with their own effects.
For CI/CP, name the owner before proposing a write: equipment uses
[configure-conditions](../configure-conditions/SKILL.md), phase uses
configure-workunit-program. Ask which owner when the request leaves it unclear.
