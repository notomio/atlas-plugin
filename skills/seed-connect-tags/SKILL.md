---
name: seed-connect-tags
description: Use when syncing or seeding Connect signals — tags, acquisitions, data sources — for a site's equipment or work units in Atlas. Triggers on "sync the tags", "seed Connect signals", "load the tags for equipment X", or importing a tag list into Atlas.
---

# Seed Connect tags & acquisitions

Sync a site's Connect signals safely — after its topology exists, chunked per node.

First follow the [connector and recovery procedure](../configure-plc/references/session-and-recovery.md), retaining the chosen connector for every call. For missing topology, read [seed-site](../seed-site/SKILL.md).

## Prerequisite: topology first

Every Connect tag hangs off exactly one topology node — an **equipment** OR a **work unit**, never both. Before syncing, confirm every referenced `equipmentPhysicalId` / `workUnitPhysicalId` already exists in the ISA-95 topology (use `atlas_search` with `types: ["equipment", "workUnit"]` or `atlas_isa95_get_*`). In search `results`, verify `type`, `id`, `siteId`, `identifier` and `target`; `totals` counts all matches beyond the limited list. The input `siteId` only favors that site. Pass the exact owner `identifier` as the sync's `equipmentPhysicalId` / `workUnitPhysicalId`; never derive it from a tag identifier. Resolve ambiguous or truncated results with a scoped read before applying. If any is missing, seed the topology first (see the `seed-site` skill). Never sync Connect against a node that isn't in the topology.

## Recipe

1. **Partition** the tags by their owner: group them by the equipment or work unit each tag hangs off.
2. **Chunk to ≤ 100 tags per group.** A single equipment/work-unit sync carries **at most 100 tags** (hard cap). Split a larger seeding job into per-equipment / per-work-unit groups — never one oversized payload.
3. **Sync each group** with preview → confirm → apply, using the tool that matches the owner:
   - Tags on an **equipment**: `atlas_connect_preview_equipment_sync` → `atlas_connect_apply_equipment_sync`
   - Tags on a **work unit**: `atlas_connect_preview_workunit_sync` → `atlas_connect_apply_workunit_sync`
4. Each sync is the **complete truth for that scope's signals**: omitted tags/acquisitions inside the scope are **deleted** after confirmation. Data sources referenced by the payload are created/updated but never deleted.

## Guardrails

- **Topology before Connect** — never sync tags for a node absent from the topology.
- **Always preview before apply** — show the diff and confirm deletions.
- **≤ 100 tags per group** — the tool rejects a larger payload. If one equipment/work-unit genuinely exceeds 100 tags, revisit the grouping; large inline payloads also get truncated by the model, so keep each scope small.
- **One tag = one owner** (equipment XOR work unit). The owner fixes which tool to use — it's not a free choice.
- For a single tag/acquisition correction, use the granular `atlas_connect_update_tag` / `atlas_connect_update_acquisition` tools instead of a full scope sync.
