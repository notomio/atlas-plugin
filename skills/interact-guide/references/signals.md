# Signal meaning and selection

| Kind | Required raw point capability | Equipment use                                               |
| ---- | ----------------------------- | ----------------------------------------------------------- |
| DI   | Readable boolean              | Read a compatible UDT field                                 |
| DO   | Writable boolean              | Command, or read a compatible field from the previous cycle |
| AI   | Readable numeric              | Read a compatible UDT field                                 |
| AO   | Writable numeric              | Command, or read a compatible field from the previous cycle |

For a new binding, both point and channel must be active and its source must
belong to the equipment PLC's pool. Compare the variable and field's exact
type kind and IEC type name (names are case-insensitive). Numeric types are
not interchangeable: for example, an integer field is not a REAL field.
An unavailable saved binding stays identifiable; do not silently substitute
a similarly named signal or remove it.

## Digital polarity

`direct: true` preserves the boolean sense; `direct: false` inverts it in the
IO layer. An inverted contact remains DI. Polarity does not change a point's
read/write capability or make DI a command output.

DO commands target the pinned UDT's `STATE_LOGIC`, if it exists and has the
required type. The IO layer applies polarity before the physical output;
`STATE_RAW` is not the equipment's command target. Do not add another inversion
as an equipment constant. Reading DO's logical state describes an echo from
the previous cycle, not an independent feedback contact.

## Raw and engineering values

Analog scaling relates `rawMin`/`rawMax` to `engineeringMin`/`engineeringMax`.
AI converts raw measurement to engineering value; AO converts an engineering
command in the opposite direction. Units and bounds belong to the channel.
They do not become configurable constants of the equipment.

In the equipment projection, four null bounds mean raw mode
(`configuration: null` on the channel). Raw mode directly passes the raw value,
without scaling or artificial clipping bounds. Identical raw and engineering
bounds are still configured scaling, not raw mode. A partly missing set of
bounds is not a basis for guessing the mode; refresh and report the inconsistency.

| Equipment command               | Target field, only if present and exactly compatible |
| ------------------------------- | ---------------------------------------------------- |
| DO                              | `STATE_LOGIC`                                        |
| AO with all four scaling bounds | `VAL_SCALED_VALUE`                                   |
| AO with all four bounds null    | `VAL_UNSCALED_VALUE`                                 |

For an analog **input**, first establish whether the user needs raw or
engineering value, then select a returned field with that meaning and type.
Do not assume that every standard version declares the fields in this table.
If an OPC-UA node already supplies the requested engineering value, do not
convert it a second time; an unscaled channel can carry that value unchanged.
If units or conversion upstream are unknown, ask before configuring bounds.

An input may read a compatible field of DO/AO, but observes the previous cycle
before the current commands are written. Only DO/AO accept equipment outputs.
Neither a logical command nor its echo proves physical operation.
