---
name: open-page
description: Use when the user asks to open, show, or share an Atlas page for a site, area, work center, work unit or its conduct, equipment, PLC, data source, gateway, Connect tag/acquisition, Interact point/channel, or initial/permanent condition (CI/CP). Read-only navigation through the connected Atlas MCP, including "ouvre la CI de la pompe", "montre cette gateway", and "donne-moi le lien du tag".
---

# Open the right Atlas page

Use the [connector and read/retry procedure](../configure-plc/references/session-and-recovery.md).
This skill only reads Atlas and navigates. It does not change configuration.

## Resolve the target

Use ids and site membership from the selected connector, never from a previous
environment or an unverified conversation guess. Reuse a fresh read from this
session when it already establishes both. Otherwise:

- For a site, use `atlas_isa95_list_sites` and match its label or slug.
- For an area, work center, work unit or equipment named by the user, use
  `atlas_search` with `types: ["area", "workCenter", "workUnit", "equipment"]`.
  Read `results`, retaining `type`, `id`, `siteId`, `identifier` and `target`;
  `totals` counts all matches per type, beyond the returned results. `siteId`
  ranks the active site first; it does not restrict the search to that site.
  If there are several plausible matches, show their site and identifier and
  ask the user to choose. Do not use the first result silently.
- For a PLC, use `atlas_interact_plc_list` for the verified site. Select the PLC
  the user named; ask if multiple match.
- For a data source or gateway, use `atlas_search` with
  `types: ["datasource", "gateway"]`. In `results`, verify `type`, `id` and
  `siteId`; use `target` for navigation. `atlas_gateway_list` can disambiguate
  gateways on a known site. Ask the user to choose when several match.
- For a Connect tag, use `atlas_search` with `types: ["tag"]`. Its result's
  `id` is the tag id and `identifier` is its physical ID. Its `target` is the
  equipment parent, or otherwise the work-unit parent, not the tag itself;
  `target: null` means no parent destination. Unattached tags remain searchable.
  For an acquisition, read its verified parent's group through
  `atlas_connect_get_equipment` or `atlas_connect_get_workunit`. Resolve that
  parent's exact `identifier` with an ISA-95 search or a scoped read, matching
  the target's `type` and `id` and the tag's `siteId`, then pass it unchanged as
  `physicalId`; the tag's identifier is not the parent's. Match the acquisition
  in the returned tags. Neither entity is itself a canvas node. If the parent
  cannot be established, ask for it rather than guessing.
- For an Interact point or IO Manager channel, use an already verified source
  and `atlas_interact_list_channels` to resolve the row. Follow every page of
  the source's results. This tool does not search every source in a site: if
  the source cannot be established, ask which one contains the point/channel.
  An unqualified raw point is absent from this read and cannot be resolved by
  this tool.
- For a specific CI/CP, first resolve its equipment and site. Then use
  `atlas_interact_get_equipment_conditions` and find the condition by its
  `kind`, label or position. `init` means CI and `perm` means CP. Rows of one
  condition share a `target`; use that returned value verbatim. If the user's
  wording identifies multiple conditions, present the matches and ask which
  one to open. If they want the equipment's conditions generally, open the
  equipment inspector without a `condition` parameter.

If the target is absent, inaccessible or a required read tool is unavailable,
say what could not be resolved. Do not assemble a link from an unverified id.

## Build the link

Use the app origin corresponding to the selected connector, as documented in
the plugin [README](../../README.md#connecter-atlas-par-oauth). Never use the
MCP/API origin as a web-app origin and never switch environments to make a link.
For an equipment without an indicated product surface, use the current
conversation's Connect or Interact context. If neither is clear, ask which
surface the user wants.

- Site or ISA-95 node in the topology editor:
  `/sites/<siteId>/editor` with `?selectedNode=<type>:<id>` for a node.
  The site's own page needs no `selectedNode`.
- Connect equipment, data source or gateway:
  `/connect?siteId=<siteId>&selectedNode=<type>:<id>`.
- Interact PLC, equipment or data source:
  `/interact?siteId=<siteId>&selectedNode=<type>:<id>`.
- A Work Unit's conduct (state diagram and sequences) in Interact:
  `/interact?siteId=<siteId>&selectedNode=workUnit:<workUnitId>`. Offer it
  after reading a conduct with `atlas_interact_get_workunit_program`, which
  never edits it.
- An Interact equipment's specific view: add
  `&inspectorMode=<equipment|wiring|conditions>` to its URL. Choose the view
  the user requested or the verified change concerns. `equipment` shows its
  properties, PLC and standard object; `wiring` shows bindings; `conditions`
  shows CI/CP. A view unavailable for this equipment falls back to its
  properties in Atlas. Do not promise an unavailable view opened.
- For a search result with a non-null `target`, use `target.type` and
  `target.id` as the selection and the result's `siteId` as the site.
- Connect tag or acquisition: open its verified equipment or work-unit parent
  in Connect. A tag with `target: null` has no parent link; offer the tag table
  instead. The inspector shows the parent's tags and acquisitions but does
  not focus an individual row; name the tag or acquisition to find beside the
  link. If the user wants the tag table, offer
  `/connect/tags?siteId=<siteId>&q=<identifier>` as a
  filtered list. Do not present a filtered list as an exact tag/acquisition deep link.
- Interact point or channel: open its verified data source in Interact. The
  source page contains the channels table, but the URL cannot focus one row;
  name the row to look for beside the link.
- One CI/CP in Interact: use the equipment URL above with
  `&inspectorMode=conditions&condition=<target>`, copying the exact target
  returned by the conditions read. A link carrying `condition` alone also
  opens Conditions for existing bookmarks.

URL-encode each dynamic path segment and query value. A CI/CP link must carry
the site, equipment selection and condition target together. Do not infer the
condition's position from the order of returned clauses.

Offer the full clickable HTTPS link. If the user asked the assistant to open the page
and the client has a browser navigation capability, navigate to that link.
After a verified mutation, also navigate in the already controlled Atlas tab
when it belongs to the selected connector's app origin. This continues the
browser control already in progress; report where it landed. If no matching
Atlas tab is under the assistant's control, offer to open the link and wait for the
user's request before starting browser control. Never replace a tab for a
different app or connector. The matching origin does not prove the browser is
logged into the MCP tenant. Check the landing page before claiming the entity
or condition opened. If navigation fails, lands on login, or cannot access the
target, report that outcome and retain the link for the user. A browser login
may still be needed even when the MCP connector is authenticated.
