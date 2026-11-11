# Choose the connector, then preserve it

1. Establish the connector selected by the user for this workflow. If ambiguous,
   ask which environment before the first tool call. For staging acceptance,
   use `atlas-staging` only. Keep its actual client tool prefix with every call,
   including calls made while following another bundled skill.
2. Inspect the tools exposed in that connected session and their input schemas.
   No separate tool-search capability is required. A name in these references
   is not evidence that the connected server offers it. Before a mutation,
   ensure both its tool and the reads needed to verify it are available.
3. A missing required tool blocks that step and its dependent mutations. Name
   the tool, connector, blocked step and completed effects. Ask for the current
   Atlas server/connection to be restored. Do not call an arbitrary endpoint,
   substitute the removed whole-equipment replacement, or try production.
4. Use ids obtained from that connector. The tenant comes from the OAuth
   identity, never a payload value. Reauthentication must preserve the chosen
   environment and intended tenant; discard prior discovery if the identity
   changes. Do not mix ids or prior reads from different tenants/environments.

## Before a write

Read the current target and show the requested change and its known effects.
Resolve missing choices. Existing authorization covers routine steps within
that request; obtain a decision for a newly discovered destructive effect or
scope change before calling. An explanation request authorizes no writes.
Use `atlas_datasource_create` / `atlas_datasource_update` for data-source
operations when required; do not invent names under the Connect namespace.

## Recover without guessing

Track each successful call and its returned ids and effects. Several calls are
not a transaction. A confirmed domain refusal only describes that operation;
it does not undo preceding calls. Read per-row results for point/channel sync:
an overall response can contain both successful and refused rows.

For a timeout, lost connection or other uncertain transport result:

1. Stop dependent mutations. Report which earlier steps are confirmed and which
   call has an **unknown outcome**. Do not say that nothing was written.
2. Restore the same connector, then reread the target before any retry. For a
   PLC pin/link, use `atlas_interact_plc_list` and the affected equipment reads;
   for equipment, `atlas_interact_get_equipment_configuration`; for a created
   entity, search/read the intended owner and stable identity first.
3. If the requested state is already present, report the observed state. If it
   is absent, retry only the remaining authorized action against fresh context.
   If reads are unavailable or inconclusive, leave the step blocked. Never
   recreate an entity blindly after an uncertain create response.
4. Report any differences that could reflect concurrent writes. A reread is
   current evidence, not proof that one particular call caused every difference.

For Interact binding patches, a confirmed rejected patch changes none of its
commands, including removals; earlier patches stay saved. An unchanged valid
repeat returns `changed: false`. For a point/channel sync, inspect
`rows[].pointErrorCode`, `rows[].channelErrorCode`, actions and applied counts
instead of assuming batch atomicity.

`atlas_interact_update_point` and `atlas_interact_update_channel` both accept
`{ rows: [...] }` (1–100 distinct ids), including for a single correction.
Inspect every returned row's `applied`, `errorCode` and `errorMessage`: a
confirmed domain refusal wrote nothing for that row only; other rows may have
succeeded. Do not retry the whole batch to fix one refused row.

`atlas_interact_list_channels` returns at most 100 channels per call (default
`limit: 100`). For each source, start without `cursor`, then pass each
`nextCursor` as `cursor` until it is `null`. Collect all pages before planning
corrections or reconciling a partially applied batch. Results are ordered by
stable channel id, so renaming a point or tag does not move it between pages.
Pages are live reads, not a snapshot: restart from the first page if channels
are added or removed during discovery.

After an unexpected error or uncertain transport result, an earlier part of the
batch may be committed. Call `atlas_interact_list_channels` for every affected
`dataSourceId`, find each original `point.id` or `channel.id` (not the old name
or tag), and compare the requested fields with their current values. If they
already match, no retry is needed.
Otherwise report concurrent differences and retry only the remaining authorized
patch against fresh context. A missing entry is inconclusive: this read excludes
raw points without a channel. Do not recreate it or retry blindly; leave the
dependent step blocked until its state can be established.

Finish with: confirmed steps; state reread (or why unavailable); outstanding
choices/inputs; exact blocked step and next read/action. Never perform a global
rollback, remove bindings to force an adoption, or generate, deploy or publish
implicitly. A staging outage never authorizes a production fallback.

After a confirmed change and its verification read, propose the relevant Atlas
page using [open-page](../../open-page/SKILL.md): the changed node or its
surviving parent after deletion, and the equipment view that matches the work
(configuration, wiring or conditions). Include a clickable link when the site
and identity are verified in this connector. Follow the navigation rule in
`open-page`: if the assistant already controls an Atlas tab for this connector,
navigate it to the verified URL and report the result. Otherwise offer to open
the link; start browser control only when the user asks.
An uncertain write or missing read is not a confirmed change to link as saved.
