# Discord API paths and Terraform resources

Research for [#4](https://github.com/UnstoppableMango/openapi2terraform/issues/4), part of the [#2](https://github.com/UnstoppableMango/openapi2terraform/issues/2) map.
It inventories Discord's spec the way terraform-provider-pfsense did in its ADR 0002 and `docs/resource-paths.md`: every path, which operations pair into a resource, and by what signal.

## Sources

- **Spec**: [`discord/discord-api-spec` `specs/openapi.json`](https://github.com/discord/discord-api-spec/blob/main/specs/openapi.json), OpenAPI 3.1.0, `info.version` `10`, title "Discord HTTP API (Preview)". 157 paths, 250 operations, 542 schemas. sha256 `ef338088f4c6a73fdb9840acd1ce2bf8f16c1870f4437bfc563418c7e0de7278`, identical to `main` on 2026-11-10. Every count below is computed from this file.
- **Docs**: [`discord/discord-api-docs`](https://github.com/discord/discord-api-docs) at `da52492` (2026-11-02), the source of docs.discord.com. Cited as `developers/<file>.mdx:<line>` for behaviour the spec doesn't state.
- **Precedent**: terraform-provider-pfsense `docs/adr/0002-resources-are-derived-from-endpoint-metadata.md` and `docs/resource-paths.md`.

## Answer in brief

- Of 250 operations, **115 are in scope** (bot-token guild configuration, application commands, data sources) on 63 of the 157 paths; 135 are out of scope under #2's exclusions.
- The in-scope operations yield **17 resources by a general rule**, plus 3 at the edge of scope (application settings), and 1 that no rule can build:
  - 8 full CRUD on one collection/item path pair (commands global and per guild, auto-moderation rule, emoji, role, scheduled event, soundboard sound, sticker);
  - 3 with CRUD split across paths (channel, webhook, guild template);
  - 4 singletons (guild, onboarding, welcome screen, widget settings);
  - 1 PUT-only upsert (channel permission overwrite);
  - 1 that has Create, Update and Delete but no Read anywhere (scheduled-event exception).
- **Path shape is the best primary signal, confirmed by response schema.** Tags are absent from the whole spec, descriptions exist on 25 of 250 operations and carry no structured metadata, and `operationId` nouns disagree with each other on the most important resource (channel). A two-step rule (item path is the collection plus `/{param}`; otherwise a path ending `/<same segment>/{param}` whose GET response contains the POST response) pairs all 11 one-pair and split resources (12 with application emoji) and nothing wrong in scope.
- Emoji, listed in #4 as a split resource, is not split in this spec: guild emojis are ordinary CRUD under `/guilds/{guild_id}/emojis`.

## The signals

pfSense pairs singular and plural endpoints by a structured `**Details:**` block in every operation's description (ADR 0002). Discord has no equivalent. Each candidate signal, tested against the spec:

| Signal | What the spec has | Verdict |
| --- | --- | --- |
| Tags | No operation has `tags`, and there is no top-level `tags` list. | Unusable. |
| Descriptions | 25 of 250 operations have one, all free text (for example "Modifies the incident actions of the guild"). | Unusable. |
| Extensions | One vendor extension, `x-discord-union: oneOf`, on 16 request bodies that are written as `anyOf`. It marks polymorphic bodies, not pairing. | Useful for schema, not pairing. |
| `operationId` | Present on all 250, snake case `verb_noun`. Pairs the 8 one-pair resources plus webhook (`create_webhook` / `get_webhook`) and template (`create_guild_template` / `get_guild_template`). | Fails channel: `create_guild_channel` against `get_channel`, `update_channel`, `delete_channel`. Verbs are inconsistent too: `list_` and `get_` both name lists (`get_guild_webhooks`, `get_entitlements`), and upserts use `set_` and `put_` (`set_channel_permission_overwrite`, `put_guilds_onboarding`). |
| Path shape | An item path that is its collection plus one `/{param}`. | Pairs the 8 one-pair resources (and out-of-scope message, lobby, entitlement, stage instance) with no false pairing. Misses the 3 split resources by construction. |
| Response schema | The POST's 2xx response is one of the item GET's response variants. | Finds all 11 one-pair and split resources, but is ambiguous alone: `EmojiResponse` is shared by guild and application emojis, `ApplicationCommandResponse` by global and guild commands, the webhook POST matches both `/webhooks/{webhook_id}` and `/webhooks/{webhook_id}/{webhook_token}`, and `GuildStickerResponse` also matches `/stickers/{sticker_id}`. |

The combined rule, which reproduces every pairing in the classification below:

1. For a collection path with a POST, the item path is the collection plus `/{param}`. GET, PATCH and DELETE there are Read, Update and Delete.
2. For any of those missing, look for a path of the form `…/<collection's last segment>/{param}` whose GET response contains the POST's response. This finds `/channels/{channel_id}` for `/guilds/{guild_id}/channels`, `/webhooks/{webhook_id}` for `/channels/{channel_id}/webhooks` (the token path ends in `{webhook_token}`, not a `webhooks` segment, so it is not a candidate), and `/guilds/templates/{code}` for the template's Read.
3. The collection's GET is the Lookup (import, and the list data source).

Out of scope, step 2 would also pair `POST /users/@me/channels` (create a DM) with `/channels/{channel_id}` and `POST /channels/{channel_id}/invites` with `/invites/{code}`; scope filtering, not the rule, keeps those out. The rule needs a scope list either way, since #2's scope is about subject matter (messages, members, voice) that no structural signal encodes.

## Classification

Counts are per operation (a path such as `/guilds/{guild_id}/roles` holds a Lookup, a Create and a bulk operation). "Edge" marks operations that fit a rule but whose subject, application settings rather than guild configuration, #2 doesn't settle.

### In scope: 115 operations

| Class | Ops | Resources or uses |
| --- | --- | --- |
| Full CRUD on one path pair | 32 | `application_command`, `guild_application_command`, `auto_moderation_rule`, `guild_emoji`, `guild_role`, `guild_scheduled_event`, `guild_soundboard_sound`, `guild_sticker` (8 × POST, GET, PATCH, DELETE) |
| CRUD split across paths | 12 | `channel`, `webhook`, `guild_template` |
| Singleton (GET + PATCH or PUT) | 8 | `guild`, `guild_onboarding`, `guild_welcome_screen`, `guild_widget_settings` |
| PUT-only upsert | 2 | `channel_permission_overwrite` (PUT, DELETE) |
| Create/Update/Delete with no Read | 3 | `guild_scheduled_event_exception` |
| Bulk ordering or overwrite | 4 | role positions, channel positions, global and guild command bulk overwrite |
| Lookup (collection GET of a resource; also a list data source) | 11 | one per resource above with a collection |
| Read-only GET, data source candidate | 21 | see below |
| Excluded actions and oddities | 11 | see below |
| Edge: full CRUD on one path pair | 4 + 1 Lookup | `application_emoji` |
| Edge: singleton | 6 | `application` (twice: `/applications/{application_id}` and `/applications/@me`), `application_role_connection_metadata` |

Data source candidates (21): command permissions (guild-wide and per command), computed role permissions (channel and guild level), guild preview, guild voice regions, role member counts, guild integrations, `new-member-welcome`, vanity URL, guild-wide webhooks, public widget JSON, `/oauth2/applications/@me` (gives `application_id` for the command resources), `/users/@me`, `/users/@me/guilds`, `/users/{user_id}`, `/voice/regions`, `/soundboard-default-sounds`, `/sticker-packs`, `/sticker-packs/{pack_id}`, `/stickers/{sticker_id}`.

Excluded (11): command permissions PUT (needs a Bearer token, below), follow announcement channel, audit log, incident actions PUT, integration DELETE, template sync PUT, widget PNG, leave guild, and the three token-authenticated webhook operations (GET, PATCH, DELETE on `/webhooks/{webhook_id}/{webhook_token}`) that duplicate `/webhooks/{webhook_id}`.

### Out of scope: 135 operations

| Area (#2 exclusion) | Ops |
| --- | --- |
| Messages, threads, pins, reactions, polls, webhook execution | 45 |
| Social SDK (lobbies, partner-sdk) | 20 |
| Members, member roles, prune, join requests, member permissions, event subscribers | 18 |
| Invites | 12 |
| OAuth2 user flows and the bot's own account | 10 |
| Voice (voice states, voice status, soundboard playback, stage instances) | 10 |
| Monetization (entitlements, SKU subscriptions) | 7 |
| Bans | 5 |
| DMs | 3 |
| Activities | 2 |
| Gateway | 2 |
| Interactions callback | 1 |

Monetization, Activities, the Social SDK and interactions are not named in #2's out-of-scope list, but none is guild configuration or application commands. 94 paths are wholly out of scope; `/users/@me` and `/webhooks/{webhook_id}/{webhook_token}` mix in- and out-of-scope operations.

## Cases no general rule covers

1. **Guild template.** Create and List are under `/guilds/{guild_id}/templates`, Update and Delete under `/guilds/{guild_id}/templates/{code}`, and the only Read is `GET /guilds/templates/{code}`, a different parent (`developers/resources/guild-template.mdx:107`). The item path's PUT is not an upsert but "Sync Guild Template", which snapshots the guild's current state into the template (`guild-template.mdx:130`). Its id is a string `code`, not a snowflake. Step 2 of the rule finds the Read; the PUT has to be excluded by name.
2. **Channel permission overwrite.** PUT and DELETE at `/channels/{channel_id}/permissions/{overwrite_id}` and no GET; the state is the `permission_overwrites` array on the channel, so Read is "GET the parent, pick the element by id". The spec types the PUT body's `allow` and `deny` as integers, while the response (`ChannelPermissionOverwriteResponse`) and the docs type them as strings (`developers/resources/channel.mdx:558`). Overwrites can also be set inline when the channel is created (`CreateGuildChannelRequest.permission_overwrites`), so the two resources overlap. Member roles (`/guilds/{guild_id}/members/{user_id}/roles/{role_id}`) have the same PUT/DELETE shape but are out of scope.
3. **Scheduled-event exception.** POST, PATCH and DELETE, no GET; exceptions are readable only as `guild_scheduled_event_exceptions` on the parent event. The endpoints appear nowhere in the public docs.
4. **Application command permissions PUT.** The spec lists `BotToken` and `OAuth2` security, but the docs say it "requires authentication with a Bearer token that has permission to manage the guild and its roles" (`developers/interactions/application-commands.mdx:1427`). A bot-token provider can only read them, so the path is data-source only. The spec's `security` can't be trusted to decide what a bot token can write.
5. **Bulk operations.** `PATCH /guilds/{guild_id}/roles` is the only way to set a role's position (`UpdateRoleRequestPartial` has no `position`); `PATCH /guilds/{guild_id}/channels` sets channel position, parent and `lock_permissions`, though single-channel PATCH also accepts `position`. `PUT .../commands` replaces every command at once (`application-commands.mdx:1297`), so a resource built on it would fight the per-command resources. None maps one-to-one to a resource; how ordering is modelled is #2's "Discord-specific modelling" fog.
6. **Guild singleton.** Adopt-only (create and delete are out of scope), so Destroy forgets. GET returns `GuildWithCountsResponse`, PATCH returns `GuildResponse`, which differ by two computed counts.
7. **Onboarding.** A singleton written by PUT, not PATCH, and GET and PUT return differently named schemas (`UserGuildOnboardingResponse`, `GuildOnboardingResponse`) with identical properties. Enabling it enforces at least 7 default channels (`developers/resources/guild.mdx:1528`).
8. **Webhook.** The create response includes the secret `token`; the webhook object carries it for incoming webhooks (`developers/resources/webhook.mdx:30`), so it is a sensitive computed attribute. The same object is readable and writable at `/webhooks/{webhook_id}/{webhook_token}` without bot auth. Lookup is per channel (`/channels/{channel_id}/webhooks`) though a guild-wide list exists, and a PATCH can move the webhook to another channel.
9. **Polymorphic responses.** Channel GET returns `oneOf` four variants (guild, DM, group DM, thread) of which only `GuildChannelResponse` is in scope; auto-moderation rules have five trigger types, scheduled events three entity types, webhooks three kinds. A resource built from these needs the variant chosen, or a discriminator, before attributes can be generated.
10. **List envelopes.** Application emojis and guild soundboard sounds list as `{ "items": [...] }`; every other list is a bare array, so the Lookup can't assume an array.
11. **Delete with a body.** Channel and template DELETE return 200 with the deleted object; every other in-scope DELETE returns 204.
12. **Incident actions.** PUT with no GET; the state is the guild's `incidents_data`, and both fields can be set at most 24 hours ahead (`guild.mdx:1549`), so it is a temporary switch, not configuration.
13. **Integrations.** List and DELETE only; nothing creates one through this API.
14. **`@me` aliases.** `/applications/@me` duplicates `/applications/{application_id}`; a generator that pairs by path sees two singletons for one object.
15. **Follow announcement channel.** `POST /channels/{channel_id}/followers` creates a webhook in the target channel and is the only way to create a channel-follower webhook, which is then readable at `/webhooks/{webhook_id}`. Create-only by its own path.
16. **Undocumented endpoints.** The spec carries operations the public docs don't describe: `new-member-welcome`, scheduled-event exceptions, guild join requests. Spec preview status and these gaps bear on #2's spec-pinning fog.
17. **Stage instance** (out of scope, voice) pairs by shape but is keyed by `channel_id`, so its id is another resource's id.

## Comparison with pfSense

pfSense's rule reads a declared model per operation, so pairing across differently spelled paths is free and naming follows the singular path. Discord declares nothing, so the pairing has to be inferred from path shape and response schemas, and the resource name can't come from the path either: `/channels/{channel_id}` and `/webhooks/{webhook_id}` are fine, but `/guilds/{guild_id}/auto-moderation/rules/{rule_id}` and `/applications/{application_id}/guilds/{guild_id}/commands/{command_id}` would give long names that `operationId` nouns (`auto_moderation_rule`, `guild_application_command`) state better. pfSense's Singleton (GET + PATCH, Destroy forgets) carries over unchanged to the guild, welcome screen and widget settings, with onboarding needing PUT accepted as the write. pfSense has no PUT-upsert or Read-from-parent class; Discord needs both.

## Every path

| Path | Methods | Class | Maps to | Notes |
| --- | --- | --- | --- | --- |
| `/applications/@me` | GET,PATCH | edge: singleton resource | `application (via @me)` | the bot's own application; duplicate of /applications/{application_id} |
| `/applications/{application_id}` | GET,PATCH | edge: singleton resource | `application` | application settings, not guild configuration; scope call for the map |
| `/applications/{application_id}/activity-instances/{instance_id}` | GET | out of scope: activities |  | Activities runtime state |
| `/applications/{application_id}/attachment` | POST | out of scope: activities |  | upload for Activities |
| `/applications/{application_id}/commands` | GET | Lookup / data source | `application_command` | list of global commands; also data source |
| `/applications/{application_id}/commands` | PUT | bulk |  | bulk overwrite of every global command |
| `/applications/{application_id}/commands` | POST | resource (one path pair) | `application_command` | Create |
| `/applications/{application_id}/commands/{command_id}` | GET,PATCH,DELETE | resource (one path pair) | `application_command` | Read, Update, Delete |
| `/applications/{application_id}/emojis` | GET | edge: Lookup / data source | `application_emoji` | list wrapped in ListApplicationEmojisResponse {items}, not a bare array |
| `/applications/{application_id}/emojis` | POST | edge: resource (one path pair) | `application_emoji` | application-owned, not guild configuration |
| `/applications/{application_id}/emojis/{emoji_id}` | GET,PATCH,DELETE | edge: resource (one path pair) | `application_emoji` |  |
| `/applications/{application_id}/entitlements` | GET,POST | out of scope: monetization |  |  |
| `/applications/{application_id}/entitlements/{entitlement_id}` | GET,DELETE | out of scope: monetization |  | test entitlements |
| `/applications/{application_id}/entitlements/{entitlement_id}/consume` | POST | out of scope: monetization |  |  |
| `/applications/{application_id}/guilds/{guild_id}/commands` | GET | Lookup / data source | `guild_application_command` | also data source |
| `/applications/{application_id}/guilds/{guild_id}/commands` | PUT | bulk |  | bulk overwrite of every command in the guild |
| `/applications/{application_id}/guilds/{guild_id}/commands` | POST | resource (one path pair) | `guild_application_command` | Create |
| `/applications/{application_id}/guilds/{guild_id}/commands/{command_id}` | GET,PATCH,DELETE | resource (one path pair) | `guild_application_command` |  |
| `/applications/{application_id}/guilds/{guild_id}/commands/permissions` | GET | data source candidate |  | permissions of every command in the guild |
| `/applications/{application_id}/guilds/{guild_id}/commands/{command_id}/permissions` | GET | data source candidate |  |  |
| `/applications/{application_id}/guilds/{guild_id}/commands/{command_id}/permissions` | PUT | excluded |  | needs an OAuth2 Bearer token, though the spec lists BotToken |
| `/applications/{application_id}/role-connections/metadata` | GET,PUT | edge: singleton resource | `application_role_connection_metadata` | whole list replaced by PUT; application-level |
| `/channels/{channel_id}` | GET,PATCH,DELETE | resource (split paths) | `channel` | Read, Update, Delete; created at POST /guilds/{guild_id}/channels |
| `/channels/{channel_id}/followers` | POST | excluded |  | follow an announcement channel: creates a webhook in the target channel, create-only |
| `/channels/{channel_id}/invites` | GET,POST | out of scope: invites |  |  |
| `/channels/{channel_id}/messages` | GET,POST | out of scope: messages |  |  |
| `/channels/{channel_id}/messages/bulk-delete` | POST | out of scope: messages |  |  |
| `/channels/{channel_id}/messages/pins` | GET | out of scope: messages |  |  |
| `/channels/{channel_id}/messages/pins/{message_id}` | PUT,DELETE | out of scope: messages |  |  |
| `/channels/{channel_id}/messages/{message_id}` | GET,PATCH,DELETE | out of scope: messages |  |  |
| `/channels/{channel_id}/messages/{message_id}/crosspost` | POST | out of scope: messages |  |  |
| `/channels/{channel_id}/messages/{message_id}/reactions` | DELETE | out of scope: messages |  |  |
| `/channels/{channel_id}/messages/{message_id}/reactions/{emoji_name}` | GET,DELETE | out of scope: messages |  |  |
| `/channels/{channel_id}/messages/{message_id}/reactions/{emoji_name}/@me` | PUT,DELETE | out of scope: messages |  |  |
| `/channels/{channel_id}/messages/{message_id}/reactions/{emoji_name}/{user_id}` | DELETE | out of scope: messages |  |  |
| `/channels/{channel_id}/messages/{message_id}/threads` | POST | out of scope: messages |  | thread from a message |
| `/channels/{channel_id}/permissions/member/{user_id}` | GET | out of scope: members |  | computed permissions of a member |
| `/channels/{channel_id}/permissions/role/{role_id}` | GET | data source candidate |  | computed permissions of a role in a channel |
| `/channels/{channel_id}/permissions/{overwrite_id}` | PUT,DELETE | PUT-upsert resource | `channel_permission_overwrite` | no GET; read from the channel's permission_overwrites |
| `/channels/{channel_id}/pins` | GET | out of scope: messages |  | deprecated |
| `/channels/{channel_id}/pins/{message_id}` | PUT,DELETE | out of scope: messages |  | deprecated |
| `/channels/{channel_id}/polls/{message_id}/answers/{answer_id}` | GET | out of scope: messages |  |  |
| `/channels/{channel_id}/polls/{message_id}/expire` | POST | out of scope: messages |  |  |
| `/channels/{channel_id}/recipients/{user_id}` | PUT,DELETE | out of scope: dms |  |  |
| `/channels/{channel_id}/send-soundboard-sound` | POST | out of scope: voice |  |  |
| `/channels/{channel_id}/thread-members` | GET | out of scope: messages |  | threads |
| `/channels/{channel_id}/thread-members/@me` | PUT,DELETE | out of scope: messages |  | threads |
| `/channels/{channel_id}/thread-members/{user_id}` | GET,PUT,DELETE | out of scope: messages |  | threads |
| `/channels/{channel_id}/threads` | POST | out of scope: messages |  | threads |
| `/channels/{channel_id}/threads/archived/private` | GET | out of scope: messages |  | threads |
| `/channels/{channel_id}/threads/archived/public` | GET | out of scope: messages |  | threads |
| `/channels/{channel_id}/threads/search` | GET | out of scope: messages |  | threads |
| `/channels/{channel_id}/typing` | POST | out of scope: messages |  |  |
| `/channels/{channel_id}/users/@me/threads/archived/private` | GET | out of scope: messages |  | threads |
| `/channels/{channel_id}/voice-status` | PUT | out of scope: voice |  |  |
| `/channels/{channel_id}/webhooks` | GET | Lookup / data source | `webhook` | webhooks of one channel; also data source |
| `/channels/{channel_id}/webhooks` | POST | resource (split paths) | `webhook` | Create; read, updated, deleted at /webhooks/{webhook_id} |
| `/gateway` | GET | out of scope: gateway |  |  |
| `/gateway/bot` | GET | out of scope: gateway |  |  |
| `/guilds/templates/{code}` | GET | resource (split paths) | `guild_template` | the only Read for a template; not under /guilds/{guild_id} |
| `/guilds/{guild_id}` | GET,PATCH | singleton resource | `guild` | adopt-only: guild create/delete out of scope; GET returns GuildWithCountsResponse, PATCH GuildResponse |
| `/guilds/{guild_id}/audit-logs` | GET | excluded |  | event log, not configuration (pfSense excludes logs the same way) |
| `/guilds/{guild_id}/auto-moderation/rules` | GET | Lookup / data source | `auto_moderation_rule` | also data source |
| `/guilds/{guild_id}/auto-moderation/rules` | POST | resource (one path pair) | `auto_moderation_rule` | polymorphic: oneOf five trigger types |
| `/guilds/{guild_id}/auto-moderation/rules/{rule_id}` | GET,PATCH,DELETE | resource (one path pair) | `auto_moderation_rule` |  |
| `/guilds/{guild_id}/bans` | GET | out of scope: bans |  |  |
| `/guilds/{guild_id}/bans/{user_id}` | GET,PUT,DELETE | out of scope: bans |  |  |
| `/guilds/{guild_id}/bulk-ban` | POST | out of scope: bans |  |  |
| `/guilds/{guild_id}/channels` | GET | Lookup / data source | `channel` | also data source |
| `/guilds/{guild_id}/channels` | POST | resource (split paths) | `channel` | Create |
| `/guilds/{guild_id}/channels` | PATCH | bulk |  | channel positions, parent and lock_permissions |
| `/guilds/{guild_id}/emojis` | GET | Lookup / data source | `guild_emoji` | also data source |
| `/guilds/{guild_id}/emojis` | POST | resource (one path pair) | `guild_emoji` |  |
| `/guilds/{guild_id}/emojis/{emoji_id}` | GET,PATCH,DELETE | resource (one path pair) | `guild_emoji` |  |
| `/guilds/{guild_id}/incident-actions` | PUT | excluded |  | time-boxed (24h at most) pause of invites and DMs; no GET, state is the guild's incidents_data |
| `/guilds/{guild_id}/integrations` | GET | data source candidate |  | integrations are created by installing an app or linking an account, never by this API |
| `/guilds/{guild_id}/integrations/{integration_id}` | DELETE | excluded |  | delete-only |
| `/guilds/{guild_id}/invites` | GET | out of scope: invites |  |  |
| `/guilds/{guild_id}/members` | GET | out of scope: members |  |  |
| `/guilds/{guild_id}/members/@me` | PATCH | out of scope: members |  | the bot's own nickname |
| `/guilds/{guild_id}/members/search` | GET | out of scope: members |  |  |
| `/guilds/{guild_id}/members/{user_id}` | GET,PUT,PATCH,DELETE | out of scope: members |  |  |
| `/guilds/{guild_id}/members/{user_id}/roles/{role_id}` | PUT,DELETE | out of scope: members |  | PUT-only upsert shape, like an overwrite |
| `/guilds/{guild_id}/messages/search` | GET | out of scope: messages |  |  |
| `/guilds/{guild_id}/new-member-welcome` | GET | data source candidate |  | undocumented; 200 or 204 |
| `/guilds/{guild_id}/onboarding` | GET,PUT | singleton resource | `guild_onboarding` | Create and Update PUT the whole document |
| `/guilds/{guild_id}/permissions/member/{user_id}` | GET | out of scope: members |  |  |
| `/guilds/{guild_id}/permissions/role/{role_id}` | GET | data source candidate |  | computed guild-level permissions of a role |
| `/guilds/{guild_id}/preview` | GET | data source candidate |  |  |
| `/guilds/{guild_id}/prune` | GET,POST | out of scope: members |  |  |
| `/guilds/{guild_id}/regions` | GET | data source candidate |  | voice regions valid for the guild (channel rtc_region) |
| `/guilds/{guild_id}/requests` | GET | out of scope: members |  | join requests, undocumented |
| `/guilds/{guild_id}/requests/{request_id}` | PATCH | out of scope: members |  | approve or reject, undocumented |
| `/guilds/{guild_id}/roles` | GET | Lookup / data source | `guild_role` | also data source |
| `/guilds/{guild_id}/roles` | POST | resource (one path pair) | `guild_role` |  |
| `/guilds/{guild_id}/roles` | PATCH | bulk |  | role positions |
| `/guilds/{guild_id}/roles/member-counts` | GET | data source candidate |  | map of role id to count |
| `/guilds/{guild_id}/roles/{role_id}` | GET,PATCH,DELETE | resource (one path pair) | `guild_role` |  |
| `/guilds/{guild_id}/scheduled-events` | GET | Lookup / data source | `guild_scheduled_event` | also data source |
| `/guilds/{guild_id}/scheduled-events` | POST | resource (one path pair) | `guild_scheduled_event` | polymorphic: external, stage, voice |
| `/guilds/{guild_id}/scheduled-events/{guild_scheduled_event_id}` | GET,PATCH,DELETE | resource (one path pair) | `guild_scheduled_event` |  |
| `/guilds/{guild_id}/scheduled-events/{guild_scheduled_event_id}/exceptions` | POST | no Read (uncovered) | `guild_scheduled_event_exception` | undocumented; no GET, read from the event's guild_scheduled_event_exceptions |
| `/guilds/{guild_id}/scheduled-events/{guild_scheduled_event_id}/exceptions/{exception_id}` | PATCH,DELETE | no Read (uncovered) | `guild_scheduled_event_exception` |  |
| `/guilds/{guild_id}/scheduled-events/{guild_scheduled_event_id}/users` | GET | out of scope: members |  | subscribers |
| `/guilds/{guild_id}/scheduled-events/{guild_scheduled_event_id}/users/counts` | GET | out of scope: members |  |  |
| `/guilds/{guild_id}/scheduled-events/{guild_scheduled_event_id}/{guild_scheduled_event_exception_id}/users` | GET | out of scope: members |  |  |
| `/guilds/{guild_id}/soundboard-sounds` | GET | Lookup / data source | `guild_soundboard_sound` | list wrapped in {items} |
| `/guilds/{guild_id}/soundboard-sounds` | POST | resource (one path pair) | `guild_soundboard_sound` |  |
| `/guilds/{guild_id}/soundboard-sounds/{sound_id}` | GET,PATCH,DELETE | resource (one path pair) | `guild_soundboard_sound` |  |
| `/guilds/{guild_id}/stickers` | GET | Lookup / data source | `guild_sticker` | also data source |
| `/guilds/{guild_id}/stickers` | POST | resource (one path pair) | `guild_sticker` | multipart/form-data upload |
| `/guilds/{guild_id}/stickers/{sticker_id}` | GET,PATCH,DELETE | resource (one path pair) | `guild_sticker` |  |
| `/guilds/{guild_id}/templates` | GET | Lookup / data source | `guild_template` | also data source |
| `/guilds/{guild_id}/templates` | POST | resource (split paths) | `guild_template` | Create |
| `/guilds/{guild_id}/templates/{code}` | PATCH,DELETE | resource (split paths) | `guild_template` | Update, Delete; no GET on this path |
| `/guilds/{guild_id}/templates/{code}` | PUT | excluded |  | sync: snapshot the guild's current state into the template (action) |
| `/guilds/{guild_id}/threads/active` | GET | out of scope: messages |  | threads |
| `/guilds/{guild_id}/vanity-url` | GET | data source candidate |  | code is settable only through guild features, not here |
| `/guilds/{guild_id}/voice-states/@me` | GET,PATCH | out of scope: voice |  |  |
| `/guilds/{guild_id}/voice-states/{user_id}` | GET,PATCH | out of scope: voice |  |  |
| `/guilds/{guild_id}/webhooks` | GET | data source candidate |  | every webhook in the guild; webhook's Lookup is the per-channel list |
| `/guilds/{guild_id}/welcome-screen` | GET,PATCH | singleton resource | `guild_welcome_screen` |  |
| `/guilds/{guild_id}/widget` | GET,PATCH | singleton resource | `guild_widget_settings` |  |
| `/guilds/{guild_id}/widget.json` | GET | data source candidate |  | public widget |
| `/guilds/{guild_id}/widget.png` | GET | excluded |  | image |
| `/interactions/{interaction_id}/{interaction_token}/callback` | POST | out of scope: interactions |  |  |
| `/invites/{code}` | GET,DELETE | out of scope: invites |  |  |
| `/invites/{code}/target-users` | GET,PUT | out of scope: invites |  |  |
| `/invites/{code}/target-users/bulk-add` | POST | out of scope: invites |  |  |
| `/invites/{code}/target-users/bulk-delete` | POST | out of scope: invites |  |  |
| `/invites/{code}/target-users/job-status` | GET | out of scope: invites |  |  |
| `/invites/{code}/target-users/{user_id}` | PUT,DELETE | out of scope: invites |  |  |
| `/lobbies` | PUT,POST | out of scope: social-sdk |  |  |
| `/lobbies/{lobby_id}` | GET,PATCH,DELETE | out of scope: social-sdk |  |  |
| `/lobbies/{lobby_id}/channel-linking` | PATCH | out of scope: social-sdk |  |  |
| `/lobbies/{lobby_id}/members/@me` | DELETE | out of scope: social-sdk |  |  |
| `/lobbies/{lobby_id}/members/@me/invites` | POST | out of scope: social-sdk |  |  |
| `/lobbies/{lobby_id}/members/bulk` | POST | out of scope: social-sdk |  |  |
| `/lobbies/{lobby_id}/members/{user_id}` | PUT,DELETE | out of scope: social-sdk |  |  |
| `/lobbies/{lobby_id}/members/{user_id}/invites` | POST | out of scope: social-sdk |  |  |
| `/lobbies/{lobby_id}/messages` | GET,POST | out of scope: social-sdk |  |  |
| `/lobbies/{lobby_id}/messages/{message_id}/moderation-metadata` | PUT | out of scope: social-sdk |  |  |
| `/oauth2/@me` | GET | out of scope: oauth2 |  |  |
| `/oauth2/applications/@me` | GET | data source candidate |  | the bot's application; yields application_id for command resources |
| `/oauth2/keys` | GET | out of scope: oauth2 |  |  |
| `/oauth2/userinfo` | GET | out of scope: oauth2 |  |  |
| `/partner-sdk/dms/{user_id_1}/{user_id_2}/messages/{message_id}/moderation-metadata` | PUT | out of scope: social-sdk |  |  |
| `/partner-sdk/provisional-accounts/unmerge` | POST | out of scope: social-sdk |  |  |
| `/partner-sdk/provisional-accounts/unmerge/bot` | POST | out of scope: social-sdk |  |  |
| `/partner-sdk/token` | POST | out of scope: social-sdk |  |  |
| `/partner-sdk/token/bot` | POST | out of scope: social-sdk |  |  |
| `/skus/{sku_id}/subscriptions` | GET | out of scope: monetization |  |  |
| `/skus/{sku_id}/subscriptions/{subscription_id}` | GET | out of scope: monetization |  |  |
| `/soundboard-default-sounds` | GET | data source candidate |  |  |
| `/stage-instances` | POST | out of scope: voice |  | full CRUD by shape, keyed by channel_id |
| `/stage-instances/{channel_id}` | GET,PATCH,DELETE | out of scope: voice |  |  |
| `/sticker-packs` | GET | data source candidate |  |  |
| `/sticker-packs/{pack_id}` | GET | data source candidate |  |  |
| `/stickers/{sticker_id}` | GET | data source candidate |  | guild or standard sticker |
| `/users/@me` | GET | data source candidate |  | the bot user |
| `/users/@me` | PATCH | out of scope: oauth2 |  | the bot's username and avatar; account, not guild configuration |
| `/users/@me/applications/{application_id}/entitlements` | GET | out of scope: oauth2 |  |  |
| `/users/@me/applications/{application_id}/role-connection` | GET,PUT,DELETE | out of scope: oauth2 |  | OAuth2 only |
| `/users/@me/channels` | POST | out of scope: dms |  |  |
| `/users/@me/connections` | GET | out of scope: oauth2 |  |  |
| `/users/@me/guilds` | GET | data source candidate |  | guilds the bot is in |
| `/users/@me/guilds/{guild_id}` | DELETE | excluded |  | leave a guild (action) |
| `/users/@me/guilds/{guild_id}/member` | GET | out of scope: oauth2 |  | OAuth2 only |
| `/users/{user_id}` | GET | data source candidate |  | user lookup (e.g. a member overwrite's target) |
| `/voice/regions` | GET | data source candidate |  |  |
| `/webhooks/{webhook_id}` | GET,PATCH,DELETE | resource (split paths) | `webhook` | Read, Update, Delete |
| `/webhooks/{webhook_id}/{webhook_token}` | GET,PATCH,DELETE | excluded |  | same webhook, authenticated by its token instead of the bot; duplicates /webhooks/{webhook_id} |
| `/webhooks/{webhook_id}/{webhook_token}` | POST | out of scope: messages |  | execute |
| `/webhooks/{webhook_id}/{webhook_token}/github` | POST | out of scope: messages |  |  |
| `/webhooks/{webhook_id}/{webhook_token}/messages/@original` | GET,PATCH,DELETE | out of scope: messages |  |  |
| `/webhooks/{webhook_id}/{webhook_token}/messages/{message_id}` | GET,PATCH,DELETE | out of scope: messages |  |  |
| `/webhooks/{webhook_id}/{webhook_token}/slack` | POST | out of scope: messages |  |  |
