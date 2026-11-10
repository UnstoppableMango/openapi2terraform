# openapi2terraform

A generator that turns an OpenAPI spec into a Terraform provider (plugin framework), shared by every provider generated from a spec.

## Language

**Resource IR**:
openapi2terraform's own description of one resource, singleton or data source between the spec and the generated code: its operations, identity, parent parameters and attributes. Everything generated is generated from it.
_Avoid_: resource model, provider model, provider spec

**Provider code spec**:
HashiCorp's JSON format describing a provider's schemas, consumed by `tfplugingen-framework`. openapi2terraform produces it from the **Resource IR**; it carries schemas only.
_Avoid_: schema.json, provider spec

**Exclusion**:
A resource, data source or attribute that config explicitly keeps out of the generated provider, with a reason. Anything that can't be mapped and isn't an Exclusion fails generation.
_Avoid_: skip, ignore
