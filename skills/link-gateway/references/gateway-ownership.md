# Gateway ownership

The gateway device is administered exclusively by `platform_admin`: creation,
deletion, renaming and MQTT identity. Tenants can read the site's gateways and
choose which existing data sources they collect. Linking a source is collection
configuration; it does not grant administration of the device.

A site has no default gateway. A platform administrator must provision one
before it can collect. Newly created sources are associated with all the site's
gateways, and a newly created gateway receives the site's existing sources;
the tenant may refine collection links afterwards.

An additive link preserves the gateway's other sources. Its recorded association
does not prove active collection: that requires a subsequent Connect release.
This bundled explanation follows Notom's adopted gateway ownership decision
(ADR-040); reading the repository is not needed to follow the skill.
