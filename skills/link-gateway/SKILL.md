---
name: link-gateway
description: Use when an Atlas employee wants to link, attach, associate, reconnect, or "rattacher" an existing Connect data source to an existing site gateway — attaching what the gateway collects. Triggers on "link the gateway", "attach this data source to a gateway", "connect the data source to the gateway", "add data source X to gateway Y", "rattache cette data source à la gateway", or any request to make one gateway collect one existing data source.
---

# Link an existing data source to a gateway

Attach an existing Connect data source to an existing gateway — additively, over the read-only gateway surface.

First follow the [connector and recovery procedure](../configure-plc/references/session-and-recovery.md), retaining the chosen connector for every call.

## When to use

Linking a data source the site's gateways don't already collect, or adding a data source to a specific gateway after refining its links. New data sources auto-associate to **all** of a site's gateways, so reach for this skill when that default was narrowed and you need to (re)link a specific pair.

This skill only ever **links** a data source to a gateway. The gateway **device** — create, delete, rename, MQTT identity — is administered exclusively by `platform_admin`; tenants see gateways read-only. Never create, delete, or rename a gateway here (see [gateway ownership](references/gateway-ownership.md)). For a full topology or Connect-signal seeding job, use the [seed-site](../seed-site/SKILL.md) / [seed-connect-tags](../seed-connect-tags/SKILL.md) skills first.

## Stop before linking

- **The data source must already exist.** Linking never creates a data source. If the data source id is unknown or the tool returns `NOT_FOUND`, stop with "no link was modified" and send the user back to the Connect seeding/update workflow (`seed-connect-tags`) before retrying.
- **The site must already have a gateway.** A site is born without one; `platform_admin` sets up the device out-of-band. If `atlas_gateway_list` returns an empty `gateways` list, stop and explain that gateway provisioning is a `platform_admin` operation.

## Recipe

1. **Resolve the site**. Use the provided `siteId`, or find it with `atlas_isa95_list_sites` when the user only gives a site name.
2. **List the site's gateways** with `atlas_gateway_list`. It returns each gateway's `id`, `name`, `mqttClientId` and linked `dataSourceIds`. If there are no gateways, stop. If there is more than one plausible gateway and the user did not name one, ask which `id` to use.
3. **Check existing links** on the selected gateway. If `dataSourceId` is already present, still call `atlas_gateway_link_data_source` to get the structured confirmation; the tool no-op returns `alreadyLinked: true`.
4. **Link** with `atlas_gateway_link_data_source`, passing `siteId`, `gatewayId` and `dataSourceId`. The link is **additive** — the server adds this one source and leaves every existing link in place — and **idempotent**.
5. **Confirm** from the tool result: report `alreadyLinked` and the returned `dataSourceIds`, the full list now on the gateway.

## Guardrails

- **Read-only on the device.** Link data sources only. Never create, delete, or rename a gateway, and never touch its `mqttClientId` — those are `platform_admin`-only. If the user asks for one of those, say it's a platform-admin operation, not available here.
- **Data source first.** It must already exist in Connect (`seed-connect-tags`); otherwise the link fails `NOT_FOUND` and nothing is written.
- **No gateway, no link.** If the site has no gateway, there's nothing to link to — that needs `platform_admin` to set up the device. Don't work around it.
- **Additive & idempotent** — safe to re-run. Existing links are always preserved; a re-link returns `alreadyLinked: true`.
- **Atomic.** The server adds the one source in a single statement, so linking the same gateway from several clients at once (another MCP session, the Atlas UI) cannot drop a link. Editing the gateway's whole list through the Atlas UI is still last-write-wins — that surface replaces the set.
- **Linking ≠ collecting.** A link only records the association; collection starts at the next **release** (publish), which regenerates and pushes the Telegraf config.
