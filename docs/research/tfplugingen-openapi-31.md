# Can tfplugingen-openapi and tfplugingen-framework generate from Discord's OpenAPI 3.1 spec?

Research for [#3](https://github.com/UnstoppableMango/openapi2terraform/issues/3), part of the map [#2](https://github.com/UnstoppableMango/openapi2terraform/issues/2).

## Answer

Partly, and only after new pre-processing.
OpenAPI 3.1 itself is not the problem: the parser (libopenapi) reads the 3.1 document, and every 3.1 type array in Discord's spec is a two-element `[T, "null"]`, which the generator handles.
What breaks is how Discord writes enums and polymorphism.

Run unmodified against the spec with a config for seven must-work resources, `tfplugingen-openapi` exits 0, prints 13 warnings, and drops 5 of the 7 resources and 3 of the 4 data sources completely.
The two resources that survive (role, webhook) lose their whole read response.
The first blocker is Discord's enum style: `type: integer` plus a `oneOf` of `{title, const}` branches. The generator rejects any `oneOf` whose branches are not `[T, null]` or `[string, primitive]`, and a property that fails takes down the whole create request body, and so the whole resource.
pfSense's `patch-openapi` (allOf flattening) changes none of the 13 warnings.

With three small spec rewrites (const-`oneOf` to `enum`, drop the sibling-carrying single-element `allOf`, drop properties typed only `null`), channel, role, webhook, permission overwrite and guild settings come out, plus the guild, role and roles data sources.
Application commands still fail on the 11-way `options` union, and channel and webhook still lose their read response to response unions.
`tfplugingen-framework` then generates schema and model code. Most of it compiles, but the two guild packages do not, because nested type names collide.

Even when generation succeeds, the result is not a usable provider.
The chain never emits CRUD bodies or HTTP calls. The paths and methods in the generator config do not survive into the provider code spec. The update request body is never read. Path parameters are mapped only from the read operation, so a channel's `guild_id` and a webhook's `channel_id`, the parents they are created under, come out `computed` and cannot be set.

## Versions examined

| Input | Version |
| --- | --- |
| `hashicorp/terraform-plugin-codegen-openapi` | `main` at `55b2a49` (2026-08-31, last tag v0.3.0), libopenapi v0.21.8 |
| `hashicorp/terraform-plugin-codegen-framework` | `main` at `8ff3055` (2026-08-31, v0.4.1-39) |
| Discord spec | `discord/discord-api-spec` `specs/openapi.json` on `main`, sha256 `ef338088…7278`, `openapi: 3.1.0`, info.version `10`, 157 paths, 542 schemas |
| pfSense precedent | `terraform-provider-pfsense` `cmd/patch-openapi`, libopenapi v0.41.8 |

Both HashiCorp tools still call themselves "tech preview" (README line 3 in each), and their recent commits are only CI and compliance changes.

## Method

Both tools were built from source and run against the spec.
The generator config used Discord's real paths, with create on the parent path and read, update and delete on the item path:

| Name | Create | Read / Update / Delete |
| --- | --- | --- |
| resource `channel` | `POST /guilds/{guild_id}/channels` | `GET / PATCH / DELETE /channels/{channel_id}` |
| resource `role` | `POST /guilds/{guild_id}/roles` | `/guilds/{guild_id}/roles/{role_id}` |
| resource `permission_overwrite` | `PUT /channels/{channel_id}/permissions/{overwrite_id}` | read `GET /channels/{channel_id}/permissions/role/{role_id}`, update `PUT`, delete `DELETE` on the create path |
| resource `webhook` | `POST /channels/{channel_id}/webhooks` | `/webhooks/{webhook_id}` |
| resource `application_command` | `POST /applications/{application_id}/commands` | `/applications/{application_id}/commands/{command_id}` |
| resource `guild_application_command` | `POST /applications/{application_id}/guilds/{guild_id}/commands` | `…/commands/{command_id}` |
| resource `guild_settings` | `PATCH /guilds/{guild_id}` | read `GET /guilds/{guild_id}` |
| data sources `guild`, `channel`, `role`, `roles` | | the matching `GET` |

Paths with `{…}` have to be quoted in YAML flow mappings, or the config fails to parse.

## Run 1: the unmodified spec

`tfplugingen-openapi generate` exits 0. Its warnings, condensed:

| Target | Effect | Cause (schema) |
| --- | --- | --- |
| `application_command`, `guild_application_command` | resource skipped | `contexts` → `InteractionContextType`: 3-way const `oneOf` |
| `channel` | resource skipped | `default_auto_archive_duration` → `ThreadAutoArchiveDuration`: 4-way const `oneOf` |
| `guild_settings`, data source `guild` | skipped | `afk_timeout` → `AfkTimeouts`: 5-way const `oneOf` |
| `permission_overwrite` | resource skipped | `type` → `ChannelPermissionOverwrites`: a **2-way** const `oneOf`, which goes down the nullable path and fails with "no 'type' array" because a `{const: 0}` branch has no `type` |
| `role`, data sources `role`, `roles` | role's create and read responses dropped; data sources skipped | `RoleTagsResponse.available_for_purchase` and `guild_connections` are `{type: "null"}`: "invalid schema type 'null'" |
| `webhook`, data source `channel` | webhook's create and read responses dropped; data source skipped | response bodies are `oneOf` unions of 3 (webhook) and 4 (channel) response schemas; `WebhookTypes` is also a const `oneOf` |

Survivors: `role` (10 attributes) and `webhook` (`avatar`, `name`, `webhook_id`).

pfSense's `patch-openapi` reads and writes the 3.1 document without error and keeps `openapi: 3.1.0`, but running the generator on its output gives the same 13 warnings.

## How the generator decides (source)

The behaviour above follows from a few functions in `terraform-plugin-codegen-openapi`:

- **Composition.** `buildSchemaProxy` (`internal/mapper/oas/build.go`) accepts only `anyOf`/`oneOf` of exactly two branches. `getMultiTypeSchema` then keeps the non-`null` branch, or keeps `string` when the other branch is integer, number or boolean. Every other `anyOf`/`oneOf` is an error. `allOf` is accepted only with one element. It **replaces** the schema with that element and keeps only the parent's `description`, so siblings like `type` and `enum` are thrown away. Multi-element `allOf` is an error ([upstream #56](https://github.com/hashicorp/terraform-plugin-codegen-openapi/issues/56)). Discriminated `oneOf` is unsupported ([#94](https://github.com/hashicorp/terraform-plugin-codegen-openapi/issues/94)).
- **3.1 type arrays.** `retrieveType` handles `len(type) == 2` the same way: drop `null`, or prefer `string` over another primitive. A type array of length 3 or more is an error. `const` is never read.
- **Failure granularity.** `BuildResourceAttributes` (`internal/mapper/oas/attribute.go`) returns on the first property error. `generateResourceSchema` (`internal/mapper/resource_mapper.go`) treats a create-request error as fatal for the resource, and create-response, read-response or parameter errors as "skip that whole source". `MapToIR` logs a warning and continues, so the command still exits 0.
- **Merging.** Attributes are merged in this order: create request body, then create response, then read response, then read parameters ([DESIGN.md](https://github.com/hashicorp/terraform-plugin-codegen-openapi/blob/main/DESIGN.md)). The result of `Merge` is discarded (`resourceAttributes, _ := …`), so a type conflict between sources keeps the first type and logs nothing.
- **Which operations are read.** Only the create operation (request and response) and the read operation (response and `path`/`query` parameters). `UpdateOp` and `DeleteOp` are stored by the explorer (`internal/explorer/config_explorer.go`) but no mapper ever reads them.
- **Output.** The output is a provider code spec: per resource, a name and a schema. It has no paths, methods or operation ids.

## Each question in the ticket

### 3.1 type arrays and `null` in `type`

All handled, natively.
The spec has 369 `[string, null]`, 178 `[array, null]`, 165 `[integer, null]`, 119 `[boolean, null]`, 80 `[object, null]` and 5 `[number, null]`, and no type array longer than two.
353 two-branch `oneOf`s, mostly `[{type: null}, {$ref}]`, also pass.
What fails is a property whose **only** type is `null`: 4 placeholder properties, among them `RoleTagsResponse.available_for_purchase` and `guild_connections`.
Nullability is not modelled: `[integer, null]` becomes a plain `int64` that is optional and computed, so the attribute does not distinguish "send null" from "omit".

### `oneOf` / `anyOf`

There are two different things here.

1. **Enums written as const `oneOf`** (91 schemas, for example `ChannelTypes`, `AfkTimeouts`, `ThreadAutoArchiveDuration`, `ChannelPermissionOverwrites`, `WebhookTypes`, `InteractionContextType`). These are the dominant blocker. Rewritten to `enum`, they map to `int64validator.OneOf(…)`. The const titles (`GUILD_TEXT` and so on) are lost, which is acceptable because the API takes the integers.
2. **Real unions.** 54 component schemas contain a union of two or more non-null, non-const branches. So do several inline path schemas: the `GET /channels/{channel_id}` response (Guild, Private, PrivateGroup and Thread channel), the `GET /webhooks/{webhook_id}` response (3 webhook kinds), the `PATCH /channels/{channel_id}` body (an `anyOf` tagged `x-discord-union: oneOf`), and `ApplicationCommandCreateRequest.options.items` (11 option types, recursive through the subcommand options). None of them has an OpenAPI `discriminator`: the three `discriminator` hits in the spec are user-tag property names. The discriminant is a `type` property whose `enum` holds one value (for example `{type: integer, enum: [8], allOf: [$ref …Types]}`). The generator cannot map any of these, and the config has no option to pick a branch.

### Snowflakes

These work.
`SnowflakeType` is `{type: string, pattern: "^(0|[1-9][0-9]*)$", format: snowflake}` and maps to `StringAttribute` with a `stringvalidator.RegexMatches` validator. `format: snowflake` is ignored, which is harmless.
`Int53Type` (`int64`, ±2^53-1) maps to `Int64Attribute` with bounds that fit `int64`, so the arm64/amd64 rounding problem that pfSense's `dropUnrepresentableIntBounds` works around does not come up.

### Permission bitfields

These are silently wrong.
The request schemas declare `permissions`, `allow` and `deny` as `[integer, null]`, while the response schemas (`GuildRoleResponse.permissions`, overwrite responses) declare `string`.
Because merge conflicts are discarded, `role.permissions` and `permission_overwrite.allow`/`deny` become `int64`, while the `role` data source, built from the response alone, has `permissions` as a string.
A bitfield string larger than 2^63 cannot be represented in `int64` at all, and the two attributes disagree in type across resource and data source.
This needs a spec patch that makes the request side `string`, so it matches Discord's documented wire format.

### Resources split across paths

The config can express them: each of create, read, update and delete is a free `{path, method}` pair, and nothing requires the paths to match.
What gets lost:

- **The parent id cannot be set.** Only read-operation parameters become attributes. Channel's `guild_id` and webhook's `channel_id` appear only because the create response bodies echo them back, so they are `computed`, not `required`. The user has no way to say which guild a channel is created in. `permission_overwrite` loses `overwrite_id` the same way.
- **There is no identity.** Read path parameters come out `computed_optional` (`channel_id`, `role_id`, `webhook_id`) next to a separate computed `id` from the body. `schema.attributes.aliases` can rename `channel_id` to `id`, but the config cannot mark an attribute as the import or lookup key, or as `RequiresReplace`. Aliasing to a nested attribute is unsupported ([#117](https://github.com/hashicorp/terraform-plugin-codegen-openapi/issues/117)).
- **The read is not always a read of the resource.** Permission overwrites have no GET of their own. `GET /channels/{channel_id}/permissions/role/{role_id}` returns computed effective permissions (`permissions`, `implicit`), not the overwrite. The overwrite is readable only as one element of `GET /channels/{id}`'s `permission_overwrites`, and the config cannot express "read the parent, then select by id". The role-only read path also leaves member overwrites out.
- **The update body is ignored.** Fields that only PATCH accepts never become attributes, and the shape of a PATCH body that differs from create (such as channel's union) is never checked.
- **Singletons and adopt-only resources.** The config can describe them by pointing create at `PATCH /guilds/{guild_id}` and omitting delete, as pfSense's ADR 0002 does for singletons. Only the schema follows from that; behaviour such as "destroy only forgets" must live elsewhere.

What the config can express, in full: `provider.{name, schema_ref, ignores}`, per resource `create/read/update/delete: {path, method}`, per data source `read`, and `schema.{ignores, attributes.aliases, attributes.overrides.<path>.description}` (`internal/config/parse.go`). That is the whole surface: there is no type override, computability override, branch selection, identity or behaviour flag.

### What pfSense's `patch-openapi` already covers

- **Multi-element `allOf` flattening.** This covers the 7 `allOf` lists of two in Discord's spec, but none of the must-work targets fails on those.
- **Single-element `allOf` with sibling keywords.** pfSense copies the outer schema, merges the `allOf` members' `properties` and `required`, and clears `allOf`, so the outer `type`/`enum` survive. Without it, `CreateGuildChannelRequest.type` (`enum: [0,2,4,5,13,14,15]` plus `allOf: [ChannelTypes]`) silently becomes the full `ChannelTypes` set (verified: run 2 below without this step validated all channel types). This is useful, but it does not fix anything on its own, because the referenced const-`oneOf` enum fails first.
- **`dropUnrepresentableIntBounds`.** It does no harm, and it is not needed for Discord.

### Pre-processing that would be new

| Step | Why | Kind |
| --- | --- | --- |
| const-`oneOf` → `enum` | unblocks 5 of 7 resources and 2 data sources | generic, mechanical |
| drop properties whose only type is `null` | unblocks role read and data sources | generic, mechanical |
| keep the outer schema of `allOf: [one]` (pfSense's flatten already does this) | keeps narrowed enums | generic, reuse |
| retype permission bitfields to `string` on the request side | merge silently keeps the wrong type | per-provider spec patch |
| pick a branch for response unions (channel → `GuildChannelResponse`, webhook → `GuildIncomingWebhookResponse`) | restores read-response attributes | per-provider, or a generic "merge union branches" rule |
| represent `options` (recursive 11-way union) | application commands cannot be generated otherwise | needs a modelling decision (merged superset object, or JSON string attribute); recursion also has to be cut |

## Run 2: with the generic steps

The first three rows were prototyped as a 20-line rewrite of the JSON (91 enums, 153 single-element `allOf`s, 4 null-only properties).
The generator then produced:

| Target | Attributes | Remaining warning |
| --- | --- | --- |
| `channel` | 26 | read response dropped (4-way union); attributes come from the create request and response |
| `role` | 15 | none, but `permissions` is `int64` (merge conflict) |
| `webhook` | 11 | read response dropped (3-way union) |
| `permission_overwrite` | 7 | none, but its "read" is effective permissions, see above |
| `guild_settings` | 45 | none |
| `application_command`, `guild_application_command` | skipped | `options`: 11-way `oneOf` |
| data sources `guild`, `role`, `roles` | 45, 15, 2 | none |
| data source `channel` | skipped | 4-way union |

Replacing the channel and webhook GET response unions with one branch removes those warnings. Channel's attribute count does not change, because its create response already contributes the same fields.

`tfplugingen-framework generate all` on that spec writes about 29k lines.
Every package compiles except `resource_guild_settings` and `datasource_guild`, which fail with `UserType redeclared in this block` (also `AvatarDecorationDataType` and others).
The framework names a nested object's Go types after the attribute's own name, so `emojis[].user` and `stickers[].user` collide ([upstream #152, open](https://github.com/hashicorp/terraform-plugin-codegen-framework/issues/152)).
Avoiding it means `ignores` on one of the paths, or a post-processing step.

## What the chain never generates

- **CRUD method bodies and HTTP calls.** `tfplugingen-framework generate` emits `<Name>ResourceSchema`/`<Name>DataSourceSchema`, the `<Name>Model` struct, and custom `Type`/`Value` types for nested objects. `scaffold resource` emits `Create`/`Read`/`Update`/`Delete` stubs whose bodies are comments such as `// Create API call logic` and `data.Id = types.StringValue("example-id")` (`internal/scaffold/templates/resource_scaffold.gotmpl`).
- **Where the operations are.** The provider code spec has no paths or methods, so anything that writes CRUD bodies has to read the generator config, or the OpenAPI document, again by itself.
- **Mapping between model and JSON.** Nothing converts the generated `Value` types to or from request and response bodies.
- **Provider configuration and auth.** Nothing generates the bot token, the base URL or the client (`provider.schema_ref` can only give the provider a schema). pfSense's `patch-provider` hand-writes `Metadata`/`Resources`/`DataSources`/`Schema`/`Configure` and a client stub into the scaffolded provider file, but it writes no CRUD bodies either.
- **Import, plan modifiers, `RequiresReplace`, defaults, sensitive values beyond `format: password`, rate-limit and 429 handling, the `X-Audit-Log-Reason` header.** None of these appears in the spec IR the chain produces.

## Implications for the map

- Building on tfplugingen-openapi is viable only with a pre-processing stage in front of it. The pfSense precedent already has that stage (`patch-openapi`), and Discord needs three more generic rewrites plus a channel for per-provider spec patches (bitfields, union branches). That bears on "what a provider may carry that's specific to it".
- Unions (`options`, polymorphic channels) are a modelling decision that no generator config can make. They feed the "Discord-specific modelling" question.
- Whatever generates CRUD bodies has to own the operation mapping (create on the parent path, read by item, parent id required, identity), because the HashiCorp chain throws all of that away after the schema step. That points toward openapi2terraform owning more than "wrap tfplugingen".
- The generator always exits 0. A pipeline should fail on its `skipping resource schema mapping` warnings, or count the expected resources, or else a re-pinned spec can silently drop resources. This is the same concern pfSense's ADR 0002 raises for `gen-config`.
