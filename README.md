# Atlas for Claude and Codex

Atlas brings twelve skills for site topology, Connect signals and Interact
configuration to your assistant. The skills use the remote Atlas service to read
your existing configuration, prepare changes and check the saved result. They
also find the relevant pages in Atlas.

The public package connects to **Atlas production** through `atlas-prod`. You
need an Atlas account with access to the relevant tenant and permission for each
operation. Installing the plugin does not create an account or grant additional
roles.

## Install in Claude Desktop Chat

The Claude directory listing requires a separate review and publication. A
release in this repository does not mean that the listing is live.

When Atlas is available in Claude's plugin directory, open the plugin browser,
find **Atlas** by **Notom**, and install it. Start a new conversation in **Chat**
and check that the skills below are available. Complete the `atlas-prod`
connection when prompted. If your organization manages plugins, its
administrator may need to make the plugin and connector available first.

Use the [Claude plugin guide](https://support.claude.com/en/articles/13837440-use-plugins-in-claude)
for your plan's installation controls. An installation in Cowork or Claude Code
does not establish compatibility with Desktop Chat.

## Install in Codex

This repository supplies the `atlas-public` marketplace with one plugin, `atlas`.
Installing from this marketplace does not depend on an OpenAI directory listing.

### Codex application

For Codex in the ChatGPT desktop app, register the marketplace on the same
computer using Codex CLI:

```sh
codex plugin marketplace add notomio/atlas-plugin --ref main
```

Restart the desktop app. Open the Plugins Directory, choose **atlas-public**,
and install **atlas**. Complete the Atlas connection and start a new chat.
Check the skills and perform the read-only connection check below in the
application itself.

The [OpenAI packaging guide](https://developers.openai.com/plugins/build/plugins)
describes marketplace registration and discovery. App names and controls can
vary by version; record the exact application and version when reporting a
problem.

### Codex CLI

Register the source, install the plugin, then start a new session:

```sh
codex plugin marketplace add notomio/atlas-plugin --ref main
codex plugin add atlas@atlas-public
codex
```

In the session, open `/plugins`, confirm Atlas is installed, and complete the
`atlas-prod` OAuth connection. Check skill discovery and the read-only operation
below separately from the application. The command syntax was checked with
Codex CLI `0.166.0`; that check alone is not an Atlas client acceptance test.

### Receive an update

Refresh the configured marketplace:

```sh
codex plugin marketplace upgrade atlas-public
```

Restart the application and open a new CLI session. Check the effective Atlas
version, skill files and `atlas-prod` connection in each client. If the client
still shows the older version, use its plugin controls to refresh or reinstall
Atlas from `atlas-public`, then repeat those checks. A refreshed marketplace
snapshot alone does not prove that a running session has loaded the update.

## Connecter Atlas par OAuth

| Connector    | MCP endpoint                   | Atlas application      |
| ------------ | ------------------------------ | ---------------------- |
| `atlas-prod` | `https://app-api.notom.io/mcp` | `https://app.notom.io` |

Connect `atlas-prod` through the client's connector controls. Follow the Atlas
login and consent flow, then return to the assistant. OAuth settings are
discovered from the service; no personal token or client secret belongs in this
package or in a conversation. The portable [MCP configuration](mcp.json) and
the [Claude configuration](.mcp.json) target the same service.

The tenant comes from the signed-in identity. To change it, disconnect and
authenticate again with the appropriate Atlas account, then rediscover the
objects. Do not reuse object identifiers from the previous tenant.

Start with: **“Using atlas-prod, list the sites I can access without changing
anything.”** Then try the `open-page` skill to find an existing site's page.
A browser may need its own Atlas login even when the MCP connection is
authenticated.

If Claude requires the connector to be added separately, use
`https://app-api.notom.io/mcp` and follow the
[remote connector guide](https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp).
Keep the selected connector when reconnecting. The public package does not
provide a staging connection.

## Choose a skill

| Skill                                                                    | Use it to                                                                             |
| ------------------------------------------------------------------------ | ------------------------------------------------------------------------------------- |
| [seed-site](skills/seed-site/SKILL.md)                                   | Prepare a site's ISA-95 topology                                                      |
| [seed-connect-tags](skills/seed-connect-tags/SKILL.md)                   | Prepare equipment tags and Connect acquisitions                                       |
| [fix-tag](skills/fix-tag/SKILL.md)                                       | Correct a specific tag or acquisition                                                 |
| [link-gateway](skills/link-gateway/SKILL.md)                             | Link an existing data source to an existing gateway                                   |
| [link-plc-data-source](skills/link-plc-data-source/SKILL.md)             | Link a Modbus TCP or OPC-UA data source to a PLC                                      |
| [interact-guide](skills/interact-guide/SKILL.md)                         | Explain points, channels, polarity and scaling without changing Atlas                 |
| [configure-plc](skills/configure-plc/SKILL.md)                           | Prepare an existing PLC and explicitly select its version                             |
| [configure-equipment](skills/configure-equipment/SKILL.md)               | Configure equipment constants, fields and IO signals                                  |
| [import-register-map](skills/import-register-map/SKILL.md)               | Turn a register table into confirmed points and channels                              |
| [configure-conditions](skills/configure-conditions/SKILL.md)             | Prepare equipment interlocks and permissives (CI/CP)                                  |
| [configure-workunit-program](skills/configure-workunit-program/SKILL.md) | Edit Work Unit conduct, Grafcet and phase CI/CP when the required tools are available |
| [open-page](skills/open-page/SKILL.md)                                   | Find the Atlas page for a site, source, gateway, tag, PLC or condition                |

Skills discover the tools actually exposed by your connector. A missing tool
blocks the step that requires it; installing a skill does not add capabilities
to the server. An explanation request does not authorize a change. For changes,
the skills read the current state, show the proposal and its effects, apply the
authorized operations, and read back the result.

For Interact, the PLC must already exist. Choose a published standard version
when none is pinned; an existing pin is preserved unless a change is requested.
Saving configuration is not evidence of compilation, deployment or physical
machine operation.

Navigation always provides a clickable HTTPS link. Browser control and embedded
MCP views are optional client capabilities; text and returned data remain
usable without them. The plugin's normal workflows do not require a local
shell, scripts, hooks or repository checkout.

## Recover from an interrupted step

An operation can succeed even if its response is lost. Reconnect the same
`atlas-prod` identity and read the saved state before repeating a write. Keep the
completed steps visible in the conversation. The shared
[session and recovery guide](skills/configure-plc/references/session-and-recovery.md)
describes this process for the skills.

## Compatibility and support

The required clients are Claude Desktop Chat, the Codex application and Codex
CLI. Initial installation and update acceptance results have not yet been
recorded for this public package. Valid manifests and a published Git repository
are not a substitute for those checks.

For a reproducible plugin issue, use this repository's
[issue tracker](https://github.com/notomio/atlas-plugin/issues). Include the
client name and version, plugin version, installation source, failing step and
sanitized error. Keep access tokens, customer data and private configuration out
of public reports. Contact your Atlas administrator about account access or
tenant permissions.

## License

The files distributed in this plugin package are licensed under [MIT](LICENSE),
copyright 2026 Notom. This license covers the packaged skills, references,
manifests, connector configuration and documentation. It does not license the
hosted Atlas application or MCP server, or grant access to those services.
