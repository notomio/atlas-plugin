---
name: seed-site
description: Use when bootstrapping or seeding an ISA-95 site's topology in Atlas — creating a site and populating its areas, work centers, work units and equipment in bulk. Triggers on "seed a site", "bootstrap the topology", "set up site X's areas / work centers / work units", or importing a plant hierarchy.
---

# Seed an ISA-95 site

Bootstrap a full ISA-95 topology for a site — safely — using the scoped-sync tools.

First follow the [connector and recovery procedure](../configure-plc/references/session-and-recovery.md), retaining the chosen connector for every call. For Connect signals afterwards, read [seed-connect-tags](../seed-connect-tags/SKILL.md).

## When to use

Populating a site's topology tree (Area → Work Center → Work Unit → Equipment) in bulk. For a one-off fix to a single node, use the granular `atlas_isa95_create_*` / `atlas_isa95_update_*` / `atlas_isa95_delete_*` tools instead — not this skill.

## Order matters (ISA-95)

Enterprise → Site → Area → Work Center → Work Unit → Equipment. Always create/sync a parent before its children.

## Recipe

1. **Site**: if it doesn't exist, `atlas_isa95_create_site`. Otherwise confirm with `atlas_isa95_list_sites` / `atlas_isa95_get_site`.
2. **Sync each scope, top-down, one node at a time**, using preview → confirm → apply:
   - Area: `atlas_isa95_preview_area_sync` → review the diff → `atlas_isa95_apply_area_sync`
   - Work center: `atlas_isa95_preview_workcenter_sync` → `atlas_isa95_apply_workcenter_sync`
   - Work unit: `atlas_isa95_preview_workunit_sync` → `atlas_isa95_apply_workunit_sync`
3. Each scoped sync is the **complete truth for that scope**: omitted descendants inside the scope are **deleted** after confirmation. Preserve each entity's `id` to update it; drop `id` to create.

## Guardrails

- **Always preview before apply.** Show the user the diff (creates / updates / deletions) and get an explicit go before applying. Apply tools require `confirmDeletions: true` (or the elicitation form) whenever the sync would delete anything.
- **Bulk scoped sync is the primary path.** Reserve the granular `create_* / update_* / delete_*` tools for surgical corrections.
- **Validate the hierarchy**: a node's parent must already exist in the tree before you sync it.
- **Topology only.** Do not put tags / acquisitions / data sources here — those are Connect (see the `seed-connect-tags` skill), and only after the topology exists.
