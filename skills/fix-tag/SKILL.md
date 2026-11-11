---
name: fix-tag
description: Use when correcting ONE existing Connect signal in Atlas — a single tag or its acquisition — NOT re-seeding a scope. Triggers on "fix this tag", "correct the tag", "rename/retype a signal", "update an acquisition", "fix the polling frequency / address", or "move this tag to another equipment/work unit".
---

# Fix a single Connect tag or acquisition

Correct ONE existing Connect signal with granular Connect tools — the surgical path, never a scoped sync.

First follow the [connector and recovery procedure](../configure-plc/references/session-and-recovery.md), retaining the chosen connector for every call. For an authorized bulk operation, read [seed-connect-tags](../seed-connect-tags/SKILL.md). Render every read, effect and refusal as [restitution](../configure-plc/references/restitution.md) prescribes.

## When to use

A one-off correction to an existing signal: fix its acquisition (the tag↔data-source link — `address`, `frequencyMs`, `enabled`), fix the tag's own metadata (label, physicalId, unit, dataType), or re-associate the tag to another equipment/work unit. To bulk-seed or re-sync every signal on a node, use the `seed-connect-tags` skill instead.

**Never run a scoped sync for a one-off fix.** `atlas_connect_apply_equipment_sync` / `atlas_connect_apply_workunit_sync` treat their payload as the _complete truth_ for that scope — every tag you omit is DELETED. Fixing one signal that way wipes its siblings. Use the granular tools below.

## Recipe

1. **Locate** the signal with `atlas_search` and `types: ["tag"]` (by label / identifier). Read `results` and `totals`; verify `type: "tag"`, `id` (the `tagId`), `siteId`, `identifier` and `target`. The input `siteId` only favors that site, so verify the returned site. Tags without a parent remain included by default: `target: null` still permits a granular correction by `tagId`, but supplies no parent or acquisition id. A non-null `target` identifies the equipment parent, or otherwise the work-unit parent; never use its id as the tag id. To see acquisitions and the referenced source, use `atlas_connect_get_equipment` / `atlas_connect_get_workunit` on a verified parent. Resolve the parent's exact `identifier` through an ISA-95 search with `types: ["equipment", "workUnit"]` or a scoped read, matching `target.type`, `target.id` and `siteId`, then pass it unchanged as `physicalId`. Capture the real `acquisitionId` from that read; if no read establishes it, stop the acquisition correction and ask for the missing scope. Never guess an id or derive it from an OPC-UA node name. If several tags match or results are truncated, narrow or ask the user to pick the exact one.
2. **Preview** the proposed patch before mutating: entity, owner (including an unattached tag), stable id, and each field as `old -> new`. Read the existing metadata from the parent's state when needed; search does not return acquisitions, data type or unit. If the necessary baseline cannot be read, stop that patch rather than inventing old values. Confirm it is the right signal and the intended owner (`equipment` XOR `workUnit` when attached).
3. **Confirm before any deletion** — `atlas_connect_delete_tag` and `atlas_connect_delete_acquisition` are destructive. Say exactly what will disappear and get an explicit go.
4. **Apply the smallest granular tool:**
   - Fix acquisition fields (`address`, `frequencyMs`, `enabled`) → `atlas_connect_update_acquisition` with `acquisitionId` and only the changed fields. It can also patch the tag's `label`, `dataType`, and `unit` atomically.
   - Fix tag metadata or ownership (`physicalId`, `label`, `description`, `unit`, `dataType`, `equipmentId`, `workUnitId`) → `atlas_connect_update_tag` with `tagId` and only the changed fields.
   - Move the acquisition to another data source → `atlas_connect_create_acquisition` on the existing `tagId`, then `atlas_connect_delete_acquisition` for the old acquisition after confirmation. `dataSourceId` is not patchable.
   - Remove one acquisition → `atlas_connect_delete_acquisition`; if it is the tag's last acquisition and the tag should disappear too, pass `removeOrphanTag: true`.
   - Remove the tag → `atlas_connect_delete_tag` (cascades its acquisitions).
5. **Verify** the tag identity/site/identifier with `atlas_search` and `types: ["tag"]`, and other changed metadata/acquisitions through the matching `atlas_connect_get_equipment` / `atlas_connect_get_workunit` when a parent exists. For an unattached tag, use the mutation result and the fields search can reread; report any fields that cannot be independently verified. Report the final tag/acquisition state.

## Guardrails

- **Granular only — never a scoped sync.** These tools touch exactly one entity; a `*_sync` apply deletes every omitted sibling in the scope. A one-off fix is never a sync.
- **Never touch the data source.** Fix the tag↔data-source _link_ (the acquisition) and the tag — not the source itself. To move a tag to another source, re-bind (create the new acquisition, then delete the old); `dataSourceId` is not patchable.
- **Locate before you mutate.** Resolve the real `tagId` / `acquisitionId` via `search` / `get_*` — never invent one.
- **Patch only what changed.** On an `update_*`, omitted fields are left untouched — nothing is deleted for being absent (unlike a sync).
- **At most one owner per tag** (equipment XOR work unit when attached); an acquisition's `address` must stay coherent with its data source's protocol (MQTT => `frequencyMs 0`).
