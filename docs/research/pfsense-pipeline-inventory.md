# Which parts of the pfSense pipeline are pfSense-specific?

Research for [#5](https://github.com/UnstoppableMango/openapi2terraform/issues/5), part of the map [#2](https://github.com/UnstoppableMango/openapi2terraform/issues/2).

Sources, all read at a pinned commit:

- terraform-provider-pfsense `main` at [`3b19c5e`](https://github.com/UnstoppableMango/terraform-provider-pfsense/tree/3b19c5e) (abbreviated **pf** below).
- a2b at [`01c27c2`](https://github.com/UnstoppableMango/a2b/tree/01c27c2985ed83828952b5f9106b275bbb5f9213), the rev pinned in pf's `flake.lock` (abbreviated **a2b**). The public clone worked; `main` is two lock-file commits ahead with no change under `nix/lib/terraform/`.
- HashiCorp's `terraform-plugin-codegen-openapi` v0.3.0, which pf pins in `nix/tools.nix` to copy its config structs.

## Summary

- **What exists today is almost entirely generic.** Only one pf tool holds pfSense logic: `cmd/patch-provider`, which hardcodes the `pfsenseProvider` receiver, `host`/`username`/`password` attributes, the `/api/v2/auth/jwt` call and the `{"data":{"token":…}}` envelope. `cmd/gen-config` is a stub that writes a single hardcoded `firewall_rule` entry. Everything else is either generic as written or holds pfSense only as literals (names, URLs, hashes).
- **What ADRs 0001-0003 commit to mostly does not exist yet.** At `3b19c5e` the provider makes no API calls at all. Resources are `tfplugingen-framework scaffold` stubs: the integration test checks that `Create` hardcodes `id = "example-id"` ([`test/firewall_rule_test.go`](https://github.com/UnstoppableMango/terraform-provider-pfsense/blob/3b19c5e/test/firewall_rule_test.go)). The ADRs ask for the derivation rule, Keys, Lookup, Markers, a write lock, apply flags and failure-to-state handling. All of that is future generator work, and none of it fits through the tfplugingen toolchain, which has no config for any of it and generates no CRUD bodies.
- **Classification of the ADR mechanisms:** Lookup, the write lock, the Singleton shape, and "extra request parameters that are not attributes" are generic mechanisms worth building into openapi2terraform. Markers are generic in shape, but whether a provider needs them at all is a pfSense quirk (positional ids). The `**Details:**` description parsing in ADR 0002, `apply`/`async`, and the one-second spacing between firewall rule creates are pfSense logic. They need either a plug-in point or a way to express them as data.
- **The only a2b Terraform consumer is pf**, plus a2b's own petstore check. A GitHub code search for `genProviderSpec` turns up no other user in any repo.

## Inventory

Classes used below:

- **G**: generic as written.
- **D**: pfSense-specific in data only; the values could come from config.
- **L**: pfSense-specific in logic.

### a2b `terraform` library

| Piece | Class | What it does / why |
| --- | --- | --- |
| `genProviderSpec` ([gen-provider-spec.nix](https://github.com/UnstoppableMango/a2b/blob/01c27c2985ed83828952b5f9106b275bbb5f9213/nix/lib/terraform/gen-provider-spec.nix)) | G | `runCommand` around `tfplugingen-openapi generate [--config] --output $out <spec>`. Takes `name`, `openapi-spec`, optional `config`, `flags`, `env`. Has no knowledge of pfSense. |
| `genProvider` ([gen-provider.nix](https://github.com/UnstoppableMango/a2b/blob/01c27c2985ed83828952b5f9106b275bbb5f9213/nix/lib/terraform/gen-provider.nix)) | G | `tfplugingen-framework generate <command=all> --input <schema> --output $out`. Generates schema and model code only, with no CRUD bodies. |
| `scaffold` ([scaffold.nix](https://github.com/UnstoppableMango/a2b/blob/01c27c2985ed83828952b5f9106b275bbb5f9213/nix/lib/terraform/scaffold.nix)) | G | `tfplugingen-framework scaffold <command> --name <snake> [--package]`. Produces stub resources, data sources and providers. |
| Tool versions | G | Both binaries come from `mangopkgs` ([flake-module.nix](https://github.com/UnstoppableMango/a2b/blob/01c27c2985ed83828952b5f9106b275bbb5f9213/nix/lib/flake-module.nix)). |
| Check ([checks/default.nix](https://github.com/UnstoppableMango/a2b/blob/01c27c2985ed83828952b5f9106b275bbb5f9213/nix/lib/terraform/checks/default.nix)) | G | Runs petstore through all three functions. |

a2b already follows the "a2b hosts the builder, the generator repo packages itself" pattern for openapi2go: `inherit (inputs'.openapi2go.packages) openapi2go;` with the comment "openapi2go packages itself rather than going through mangopkgs" ([flake-module.nix](https://github.com/UnstoppableMango/a2b/blob/01c27c2985ed83828952b5f9106b275bbb5f9213/nix/lib/flake-module.nix)). [`openapi2go/generate.nix`](https://github.com/UnstoppableMango/a2b/blob/01c27c2985ed83828952b5f9106b275bbb5f9213/nix/lib/openapi2go/generate.nix) is a thin `runCommand` over the CLI. That is the ready-made shape for "a2b calls openapi2terraform".

### pf `cmd/`

| Tool | Class | Detail |
| --- | --- | --- |
| `patch-openapi` ([openapi.go](https://github.com/UnstoppableMango/terraform-provider-pfsense/blob/3b19c5e/cmd/patch-openapi/openapi.go)) | G | Two passes. (1) `dropUnrepresentableIntBounds` removes integer `minimum`/`maximum` outside int64, a workaround for tfplugingen's `int64(float64)` conversion, which differs by architecture. It handles 3.1 type arrays. pfrest's `2^63-1` is what triggered it, but the code is spec-agnostic. (2) `flattenAllOfs` merges `allOf` properties and `required` into one schema across components, parameters, bodies, responses and headers, then bundles. Both passes are tfplugingen-compatibility fixes that any spec could need. Only the cobra `Short` text mentions pfrest. One caveat: the flatten keeps only `properties` and `required` from members and drops the members' other keywords. |
| `gen-config` ([config.go](https://github.com/UnstoppableMango/terraform-provider-pfsense/blob/3b19c5e/cmd/gen-config/config.go)) | D today, L per ADR 0002 | Parses the spec, then *ignores it*. `ConfigFor` returns `Provider.Name: "pfsense"` and one resource, `firewall_rule`, with only Create (POST) and Read (GET) at `/api/v2/firewall/rule`, then validates and writes YAML. As written, it is pure data. ADR 0002 turns it into the place where pfSense logic lives: parsing `**Details:**`, pairing by model, the Singleton rule, the override table, the Key table and the async list. |
| `gen-main` ([gen.go](https://github.com/UnstoppableMango/terraform-provider-pfsense/blob/3b19c5e/cmd/gen-main/gen.go), [main.go.tmpl](https://github.com/UnstoppableMango/terraform-provider-pfsense/blob/3b19c5e/cmd/gen-main/main.go.tmpl)) | G | Template over `RegistryAddress`, `ModulePath` and `ProviderPackage`, all passed as CLI args from `provider-src.nix`. |
| `slurp-source` ([hack.go](https://github.com/UnstoppableMango/terraform-provider-pfsense/blob/3b19c5e/cmd/slurp-source/hack.go)) | G | Copies `internal/config/parse.go` out of `terraform-plugin-codegen-openapi` (pinned v0.3.0) and adds a codegen header, so `gen-config` can build that tool's config with typed structs. `nix/tools.nix` has a TODO noting that this pin is separate from the `tfplugingen-openapi` binary that mangopkgs supplies, so the two can drift. |
| `patch-provider` ([patch.go](https://github.com/UnstoppableMango/terraform-provider-pfsense/blob/3b19c5e/cmd/patch-provider/patch.go)) | mixed G / D / L | **G:** it fills in the scaffolded provider's `Resources()`/`DataSources()` from `schema.json`, mapping `resource_<name>_resource.New<Pascal>Resource`, and appends provider-schema attributes from the spec. The splice-by-AST approach is reusable. **D:** the receiver `pfsenseProvider`, the `TypeName "pfsense"` and the module path `github.com/unstoppablemango/terraform-provider-pfsense/...` are hardcoded. **L:** the embedded `clientGo` is a pfSense client. It holds username/password, does `POST {host}/api/v2/auth/jwt` and decodes `data.token`. `Schema()` always injects `host`, `username`, `password (Sensitive)`, and `Configure()` builds a `client.Config` with a 30 s timeout (ADR 0003 asks for 5 min). The JWT is fetched by `GetJWT` but nothing calls it yet. It also writes `internal/client/client.go` relative to the working directory as a side effect. |

### pf `nix/` and `gen/`

| Piece | Class | Detail |
| --- | --- | --- |
| `openapi.nix` | D | `fetchurl` of `pfrest/pfSense-pkg-RESTAPI` release `v${version}/openapi.json` (2.8.1), plus a hash, piped through `patch-openapi`. The URL, version and hash are the only pfSense parts. Discord has no tagged releases (map #2 notes), so "fetch a release asset" is itself a per-provider choice. |
| `provider-spec.nix` | G | `gen-config` produces `config.yaml`, then `genProviderSpec` produces `schema.json`. |
| `provider-src.nix` | D, with generic structure | Hardcoded: `goPackage`, `scaffold provider name "pfSense" package "provider_pfsense"`, the registry address `registry.terraform.io/unstoppablemango/pfsense` and the binary name. The structure is generic: per-resource `scaffold`, then `genProvider`, then a `go mod init` symlink-join, then `go mod tidy -diff` into `gen/go.mod.patch`, then `applyPatches` and `gomod2nix` into `gen/gomod2nix.toml`. The comment `# TODO: Factor out the patching of the provider` acknowledges the coupling. |
| `default.nix` | D | `pname`, version, `subPackages`, and an ldflags `-X …/provider_pfsense.Version`. |
| `tests.nix` | D | Wraps OpenTofu as `terraform`. Sets `PFSENSE_PROVIDER_BINARY` and runs `test/` with `TF_ACC=1`. |
| `tools.nix` | G | Builds the cmd tools; the slurp pin is described above. |
| `gen/schema.json` | D (output) | A tfplugingen provider spec: `provider.name = pfsense`, one resource with 33 attributes. A flake check diffs it against a fresh build ([flake.nix](https://github.com/UnstoppableMango/terraform-provider-pfsense/blob/3b19c5e/flake.nix)). |
| `gen/go.mod.patch`, `gen/gomod2nix.toml` | G mechanism, D content | Exist to avoid import-from-derivation: `provider-src.nix` reads `schema.json` at evaluation time to scaffold one package per resource, and `default.nix` reads `gomod2nix.toml`. Any Nix consumer that scaffolds per resource hits the same import-from-derivation constraint, so this "check generated outputs in" pattern belongs in a2b or openapi2terraform, not per provider. |
| `mock/`, `test/` | L | A hand-written mock of the pfrest envelope `{code,status,response_id,message,data}` with firewall-rule handlers. The test asserts only that the scaffold loads. |

## What ADRs 0001-0003 commit the generator to

None of this is implemented at `3b19c5e`. It is all the generator's job, and `cmd/patch-provider` is named in ADR 0003 as the place where request code will be generated.

| Commitment | Source | Generic mechanism | pfSense quirk |
| --- | --- | --- | --- |
| **Key** stored as the Terraform id; Key fields force replacement; import by Key, with child Keys prefixed by the parent id and multi-field Keys joined by `/` | [ADR 0001](https://github.com/UnstoppableMango/terraform-provider-pfsense/blob/3b19c5e/docs/adr/0001-resources-are-identified-by-key-not-api-id.md) | Yes. "Which fields identify an object" is a generic per-resource declaration, and the import-id format is generic. | Needing it at all comes from pfSense's positional `id`, which shifts when siblings change. Discord's snowflake ids are stable, so Key = API id there, the same as pf's "stable string API id" case. |
| Per-resource Key table declared in `gen-config` | ADR 0001 | Data. The spec carries no uniqueness metadata, so any provider needs this somewhere. Config YAML is the natural home. | The entries (`tracker`, `name`, `parent_id`+`mac`). |
| **Lookup**: query the plural endpoint with an exact-match filter, then re-check the Key fields client-side; 0 matches on Read removes from state, more than 1 is an error; Update and Delete fail unless exactly 1 | ADR 0001 | Yes, as an optional "resolve id via list" step. The client-side recheck is good defensive practice anywhere. | Filter syntax (query params with exact match), and the reason for the recheck: pfSense ignores unknown filter keys. Discord lists usually have no filter, so a generic Lookup needs pluggable filtering. |
| **Marker**: ` [tf:<12 hex>]` appended to `descr`, stripped from the attribute, exposed as computed `marker`; Marker-less import matches on every configured field | ADR 0001 | The shape is generic: a provider-owned tag in a free-text field, stripped on read. Parameters would be the field name, the format and the attribute name. | Choosing `descr`, and the fact that most pfSense models need it. Unlikely to be needed for Discord. |
| One-second spacing between `firewall_rule` creates | ADR 0001 | Could be generalised as a per-resource minimum create interval, but that is speculative. | `tracker` is a whole-second timestamp with no uniqueness check. Pure pfSense. |
| Models with neither a Key nor `descr` are skipped and reported | ADR 0001, [ADR 0002](https://github.com/UnstoppableMango/terraform-provider-pfsense/blob/3b19c5e/docs/adr/0002-resources-are-derived-from-endpoint-metadata.md) | Yes: a "not generated, why" report is useful for any spec. | 44 such models at this pin. |
| **Write lock**: every write from one provider instance goes through one lock, which also serialises Lookup-then-write | ADR 0001, [ADR 0003](https://github.com/UnstoppableMango/terraform-provider-pfsense/blob/3b19c5e/docs/adr/0003-every-write-applies-in-the-same-request.md) | Yes. A provider-wide mutex is a cheap generic option. It is probably opt-in, since it serialises everything. | The motivation: there is no compare-and-swap, and applies reload the filter. Discord's per-route rate limits suggest a different concurrency control (a 429-aware limiter), not a global lock. |
| **Derive resources from endpoint metadata**: pair Singular and Plural endpoints by model; full CRUD on Singular plus a Plural endpoint makes a resource; GET+PATCH only makes a Singleton; Actions and `/apply` are excluded; the override table | ADR 0002 | The *output* shape is generic: per resource, a Create/Read/Update/Delete location plus a Lookup location. So are Singletons (Create=PATCH, Destroy forgets, fixed id) and an override and exclude list. | The *input* is pfSense logic. It reads pfrest's `**Details:**` block (Endpoint type, Associated model, Parent model, Required packages) from each operation's description and fails the build if the block is missing. Discord's spec has no such block. Its resources split across paths (`POST /guilds/{id}/channels`, then `/channels/{id}`), so derivation must be pluggable or explicit per resource. |
| Naming: the singular path minus `/api/v2/`, with `/` replaced by `_` | ADR 0002 | Generic as "path minus a configured prefix". | The prefix value. Path-based naming fits Discord poorly, because the same object has two paths. |
| **Apply flags**: every Create/Update sends `"apply": true, "async": false` in the body, every Delete sends `?apply=true&async=false`; they are added in request code, not the spec, so they never become attributes; an async-exception list (`WebGUISettings`) | ADR 0003 | The mechanism is generic: a per-provider or per-resource set of fixed extra body fields and query parameters on writes, invisible to the schema. Discord's `X-Audit-Log-Reason` *header* is the same idea, so headers belong in that mechanism too. | The values `apply` and `async`, the exception list, and the reason: config-only writes are staged in pfSense. |
| Only non-2xx is failure; on a failed write, state follows the server (Create: Lookup, and save if found so Terraform taints it; Update: keep prior state; Delete: keep state, and a later empty Lookup counts as deleted) | ADR 0003 | Yes. These are sound generic Terraform semantics. "Save on failed create" needs Lookup, or an id in the error body. | None beyond the reliance on Lookup. |
| Per-request timeout of 5 min | ADR 0003 | Data (a timeout value). | The 300 s cap of the pfrest dispatcher. |

## Implications for openapi2terraform

These are observations, not decisions.

1. The tfplugingen pair covers only *schema* generation. `tfplugingen-openapi`'s config ([v0.3.0 parse.go](https://github.com/hashicorp/terraform-plugin-codegen-openapi/blob/v0.3.0/internal/config/parse.go), copied as pf's `internal/config/config.go`) has fields only for per-operation path/method, attribute ignores, aliases and description overrides. It has no field for a Key, Lookup, Marker, lock, extra request fields or Singleton semantics. `tfplugingen-framework` emits schemas and models but no CRUD. So every ADR commitment above needs code that openapi2terraform generates itself, whether or not it keeps tfplugingen for schemas.
2. The generic tools (`patch-openapi`, `gen-main`, `slurp-source`, the generic half of `patch-provider`, the `gen/` check-in pattern) can move into openapi2terraform almost unchanged. `patch-openapi`'s `allOf` flattening is lossy (it keeps only `properties` and `required`), which may matter for Discord's 3.1 schemas.
3. The pfSense logic falls into three groups: authentication and the client (JWT exchange, response envelope), resource derivation (`**Details:**`), and quirks (`apply`/`async`, the tracker spacing). Auth and the envelope look like candidates for a small "client profile" config: an auth scheme, a token endpoint, a response-unwrap path. Derivation from description text is hard to express as data and argues for either explicit per-resource config (which Discord needs anyway) or a plug-in point.
4. a2b's Terraform functions have exactly one external consumer, pf. Changing or replacing them has no other blast radius.

## Open questions

- Whether `tfplugingen-openapi` handles Discord's OpenAPI 3.1 spec. This was not checked. `patch-openapi`'s comment anticipates 3.1 `type` arrays, but pfrest's spec version was not checked here.
- Whether pf's ADRs could make "Key = stable API id, no Lookup" the default, with Lookup and Markers opt-in. That would make Discord the simple case and pfSense the configured one.
