# openapi2terraform maps specs itself and uses HashiCorp's tooling only for schema code

openapi2terraform parses specs with libopenapi (Go) and maps them into its own **Resource IR**, replacing `tfplugingen-openapi`. From the IR it emits a HashiCorp **provider code spec** for `tfplugingen-framework` to turn into schema and model code, and it generates CRUD, client calls and import from the same IR itself. Anything it can't map fails generation unless config declares an **Exclusion**, and every Exclusion is written to a report with its reason.

We chose this because `tfplugingen-openapi` throws away what CRUD needs. It only reads the create and read operations, takes path parameters from read alone (so a channel's `guild_id` can't be set), ignores update bodies, drops type conflicts without a word, and has no way to express identity, Lookup or unions. Run against Discord's spec unmodified, it dropped 5 of 7 resources and still exited 0 ([research](https://github.com/UnstoppableMango/openapi2terraform/blob/research/tfplugingen-openapi-31/docs/research/tfplugingen-openapi-31.md)). Its config surface can't carry what terraform-provider-pfsense's ADRs 0001-0003 commit to either ([inventory](https://github.com/UnstoppableMango/openapi2terraform/blob/research/pfsense-pipeline-inventory/docs/research/pfsense-pipeline-inventory.md)).

## Considered options

- **Wrap both HashiCorp tools** and generate CRUD on top. Rejected: CRUD generation would have to re-read the spec for everything the first tool discarded, and fixes would mean rewriting the spec to suit someone else's mapper.
- **Generate everything ourselves**, schema code included. Deferred, not rejected. `tfplugingen-framework` is a dormant tech preview with a nested-name collision bug that breaks Discord's guild packages, but it sits behind the documented provider code spec, so it can be replaced without touching the Resource IR.

## Consequences

- terraform-provider-pfsense's `gen-config` and `slurp-source` (which build `tfplugingen-openapi` config) go away when it migrates. Its `patch-openapi` passes become generic normalisation in openapi2terraform, and the logic `patch-provider` holds moves into CRUD generation or per-provider config.
- A new spec pin that breaks a resource breaks the build rather than quietly shrinking the provider.
