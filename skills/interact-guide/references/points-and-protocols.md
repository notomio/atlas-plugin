# Data source, point and channel

A **data source** identifies a field connection. A **point** describes one raw
protocol address and its capabilities. An optional **IO Manager channel**
qualifies that point with a signal tag, DI/DO/AI/AO kind, activity, unit and
digital polarity or analog bounds. A raw point is not automatically a usable
equipment signal.

The equipment binds a channel id and a returned UDT field. It never binds a
register, OPC-UA NodeId or generated IO instance name. The equipment projection
deliberately omits those addresses. Discover connection details through the
data-source/point workflow only when needed and authorized.

## Modbus TCP

Modbus addresses identify tables and offsets. In this model the read functions
identify the address family:

| Function | Family            | Basic value/capability                   |
| -------- | ----------------- | ---------------------------------------- |
| FC01     | Coils             | Boolean, readable and writable           |
| FC02     | Discrete inputs   | Boolean, readable                        |
| FC03     | Holding registers | Numeric registers, readable and writable |
| FC04     | Input registers   | Numeric registers, readable              |

Those capabilities permit qualification; they do not pick the intended channel
kind for the user. A readable/writable point can serve a reading or a command
when its value type fits. A register's meaning, offset, encoding and byte order
come from the actual device documentation, never a universal register map.
FC03 describes a Modbus read; writing uses the protocol's appropriate operation.

## OPC-UA

OPC-UA points use a node identity, data type and access direction. They have no
FC01–FC04 function. Do not translate an OPC-UA point into a Modbus register
because both happen to expose numbers. A server may already expose a value in
engineering units; confirm what the node actually represents before proposing
scaling. OPC-UA alone does not prove a value is already converted.

For a hypothetical pressure measurement, establish whether the value is raw
counts or pressure in the desired unit. Its bounds and unit are site/device
facts to read or ask for, not defaults to infer from the protocol.

## Source preparation

Use [link-plc-data-source](../../link-plc-data-source/SKILL.md) to create an
authorized missing Modbus TCP or OPC-UA source and/or link an existing one to
the existing PLC. A source may belong to at most one PLC. Linking records a
pool membership; it does not establish field communication.
