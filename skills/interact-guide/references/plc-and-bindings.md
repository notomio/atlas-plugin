# PLC, standard version and equipment

A PLC executes the site's generated project. It selects one published version
of the integral standard. All its equipment resolves interfaces from that same
pin; an equipment does not select its own version. A pin is configuration, not
an installation on the physical PLC.

An equipment selects an object key from that pinned version. The key is the
POU's actual name. Its configurable inputs come from the main block's
`VAR_INPUT`, excluding the reserved global acknowledgement `I_ACK`. There is
no separate parameter category and no standard-published default to fill in.

The exchange UDT is the structured data type referenced by the block's typed
`VAR_IN_OUT`. Use the returned interface: the parameter's spelling does not
determine its type. Do not construct a UDT name from an object key.

The returned `wiringSections` groups IO wiring targets in declared order:
`BLOCK_INPUTS` and `FROM_SCADA` carry direction `input`; `TO_SCADA` and
`BLOCK_OUTPUTS` carry direction `output`. Empty sections remain present.
Exchange-field names include the actual `VAR_IN_OUT` parameter. Use the
returned name and direction, never a guessed prefix. This grouping does not
create or deploy a SCADA connection.

`outputVariables` is broader than the inspector sections: it also exposes
accepted `NOT_SENT` UDT commands still used by the standard. Keep these output
targets available alongside declared `VAR_OUTPUT` variables.

## What supplies a configurable block input

For a `configurableInputs` entry, first distinguish a **literal constant** from a **UDT field value**. For a field,
choose its source: **IO Manager signal** or **another equipment**.

| Origin          | Meaning                                                                | Hypothetical example                                                   |
| --------------- | ---------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| Constant        | A fixed value with the input's exact declared scalar type              | Enable an option with a boolean, if the actual interface offers it     |
| IO signal       | Read a compatible field from a qualified channel's exchange UDT        | Read a sensor's logical state from an active digital signal            |
| Equipment field | Read a compatible exchange field of a source equipment on the same PLC | Read a peer's process value, if its actual UDT exposes a matching type |

The input receives the field's value; it does not become a UDT-typed input
merely because the value comes from a UDT. A constant never goes through IO
Manager. One input has one origin: changing it replaces that input's previous
binding, while unrelated inputs and outputs remain configured.

A `FROM_SCADA` field is a separate IO wiring target: configure it with `setIo`
and its returned `input` direction. It is not a configurable block input, so
do not offer constants or peer-equipment bindings through `setConstant` or
`setEquipmentField` merely because the field appears in `wiringSections`.

An equipment may use channels from several sources attached to its PLC. There
is no one-source-per-equipment limit. The PLC's pool is the source boundary.

## What a saved configuration proves

An unbound input remains to be configured; the skill must report it instead of
inventing a value. Even complete saved coverage does not prove generation,
compilation, deployment or physical operation. Those stages have their own
validation. An unpinned PLC is not publishable.

Adopting a version can remove standard bindings whose inputs disappear while
preserving retained inputs, equipment and IO wiring. Moving an equipment or
changing its object type has other effects, including bindings on peers that
read it. The configuration procedures explain those effects before writing.
