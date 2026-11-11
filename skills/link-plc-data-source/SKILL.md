---
name: link-plc-data-source
description: Use when an Atlas employee wants to give an Interact PLC a field data source to collect — creating a Modbus TCP or OPC-UA source and/or linking it to a PLC. Triggers on "link this data source to the PLC", "add a Modbus source to the PLC", "rattache cette source au PLC", "create an OPC-UA source for the PLC", "make the PLC read this controller", or any request to set up what a PLC collects.
---

# Give a PLC a data source to collect

Create a generic field data source when needed, then link it to an Interact PLC — additively, over the existing data-sources and PLC surfaces.

First follow the [connector and recovery procedure](../configure-plc/references/session-and-recovery.md), retaining the chosen connector for every call. Creating a missing source requires authorization in the user's request; after uncertain transport, read its identity before retrying creation. Render every read, effect and refusal as [restitution](../configure-plc/references/restitution.md) prescribes.

## When to use

Setting up what an Interact PLC collects: a Modbus TCP controller or an OPC-UA server whose points will later be addressed and wired to equipment. Two jobs, and you may need only the second: the source may already exist in Connect (site-level data sources are one shared table), in which case skip straight to linking.

This skill never creates a PLC. A PLC is created on the tenant surface in Atlas, and there is no MCP tool for it — if the site has none, stop and say so.

## Stop before linking

- **Only `modbus_tcp` and `opcua` can be linked.** A `siemens_s7` or `mqtt` source is a valid Connect source but Interact cannot emit it: the link is refused with `UNSUPPORTED_PROTOCOL` and nothing is written. Never create a source with one of those protocols for a PLC.
- **The site must already have a PLC.** If `atlas_interact_plc_list` returns an empty `plcs` list, stop and explain the PLC has to be created in Atlas first. Do not try to work around it.
- **A data source belongs to at most one PLC.** If the source is already linked to another PLC, the link is refused with a conflict. Unlink it there first, or pick another source — do not reassign silently.

## Recipe

1. **Resolve the site**. Use the provided `siteId`, or find it with `atlas_isa95_list_sites` when the user only gives a site name.
2. **List the site's PLCs** with `atlas_interact_plc_list`. It returns each PLC's `id`, `name`, `standardVersionId` (`null` means it is not publishable yet) and the `dataSourceIds` it already collects. If there are none, stop. If there is more than one plausible PLC and the user did not name one, ask which `id` to use.
3. **Resolve one exact source.** The `dataSourceIds` from step 2 identify what this PLC already collects; they do not prove which source the user means. Use `atlas_search` with `types: ["datasource"]`. It searches the entire tenant and returns ranked, limited `results` plus `totals` for all seven types. Retain only `type: "datasource"` with the requested `siteId`; the input `siteId` only favors that site in ranking. Record the source's `id` and `target`, keeping its nullable `identifier` as returned. Never select the first result by rank or name alone.
   - If several candidates remain, present their ids/labels and wait for an explicit choice. If results are absent or potentially truncated, narrow discovery or ask the user to confirm the source in Atlas. Do not link or create from an inconclusive search.
   - Before reusing an id, confirm its site, protocol and relevant connection configuration against the request (for example, Modbus controller/slave or OPC-UA endpoint). Search results expose `protocol`, but not connection configuration. Use an available read that actually returns the source, such as a known equipment/work-unit's Connect state when it references that source; otherwise ask the user to confirm those details in Atlas. Never invent a datasource-get tool, infer configuration from the label or request secrets just to identify a source.
   - Proceed to step 5 only with one confirmed `dataSourceId`. If the user confirms in Atlas that the source is missing and authorizes creation, proceed to step 4 instead. An absent search result alone never establishes that prerequisite.
4. **Create the confirmed missing source** with `atlas_datasource_create`, passing the site id, `protocol` (`modbus_tcp` or `opcua` only), a name, and the protocol `config`:
   - `modbus_tcp` → `controller` (host:port), `slaveId`, optionally `byteOrder` and `timeout`;
   - `opcua` → `endpoint`, `securityMode`, `securityPolicy`, `authMethod`, plus credentials when the auth method needs them.
     Check the returned source's site, protocol and configuration against the requested creation, then record `dataSource.id` before going further.
5. **Link** with `atlas_interact_link_plc_data_source`, passing `plcId` and `dataSourceId`. The link is **additive** — the server adds this one source and leaves every existing link in place — and **idempotent**: re-linking returns `alreadyLinked: true` without mutating.
6. **Confirm** from the tool result: report `alreadyLinked` and the returned `dataSourceIds`, the full list the PLC now collects.

## Guardrails

- **Create and link are two calls, and they are not atomic.** If step 5 fails after step 4 succeeded, the data source exists and is unlinked. Report its id explicitly so the user can retry the link or delete the source — never pretend the whole operation rolled back.
- **Emittable protocols only.** `modbus_tcp` and `opcua`. If the user asks for a Siemens S7 or MQTT source on a PLC, say Interact cannot emit it rather than creating a source that can never be linked.
- **Never create a PLC**, and never rename or delete one here. This skill only creates data sources and links them.
- **Additive & idempotent** — safe to re-run. Existing links are always preserved.
- **The link itself is atomic.** The server reads the pool, allocates the channel index and writes inside one transaction serialized per PLC, so linking the same PLC from several clients at once cannot drop a link. Editing the PLC's whole source list through the Atlas UI is still last-write-wins — that surface replaces the set.
- **Linking ≠ collecting.** A link records which sources the PLC project draws from; the addresses only reach the machine once the points are declared and the PLC project is generated and deployed. Do not tell the user data is flowing.
- **The points come next.** A linked source collects nothing until its addressing points exist. Say so, so the user knows the job is not finished.
