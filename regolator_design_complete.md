# Regolator — Complete Architecture and Implementation Design

**Status:** High-level design / implementation specification  
**Project:** Regolator  
**Primary language:** Go  
**CLI framework:** Cobra  
**Policy language:** OPA Rego v1  
**Primary consumers:** Regolator SPA/API, Merge Control, Gatekeeper  
**Primary workload location:** CI/CD EKS cluster

---

# Table of Contents

1. Executive Summary
2. System Boundaries
3. Core Design Principles
4. Runtime and Cluster Topology
5. Repository Layout
6. `config.yaml`
7. Provider Package Images as the Upstream Schema Source
8. Persisted CRDs as Auditable Schema Snapshots
9. CRD Provenance Metadata
10. CRD Cache / Provider Pull Algorithm
11. Registering Schema-Only CRDs in the CI/CD Cluster
12. Custom / Handwritten CRDs
13. Regolator API
14. SPA Navigation and User Model
15. Schema + Policy Overlay Contract
16. ResourcePolicy Overview
17. Deterministic ResourcePolicy Naming
18. ResourcePolicy Version Model
19. ResourcePolicy Rule Vocabulary
20. Conditional Rules / Production Policy
21. Enforcement Targets / Late Initialization
22. Policy Path Representation
23. Array Path Semantics
24. Generic Monolithic Rego Policy Engine
25. Array Policy Proof of Concept
26. Rego Version Selection
27. Rego Conditional Evaluation
28. Rego Enforcement Target Filtering
29. Merge Control Integration
30. Merge Control Fail-Closed Behavior
31. OPA Input Contract
32. OPA Go Integration
33. Gatekeeper Architecture
34. Gatekeeper Generation
35. Regolator CLI
36. PR-Time Generation and Validation
37. Policy Authoring PR Flow
38. Config / Provider Update Flow
39. Post-Merge Generation
40. Argo Applications
41. Release Identity, Promotion, and Rollback
42. Lower-Environment Testing
43. Schema Drift
44. Rough ResourcePolicy CRD
45. Complete ResourcePolicy Example
46. Example `config.yaml`
47. End-to-End Architecture
48. Implementation Order
49. Explicit Non-Goals
50. Open Implementation Details
51. Official Technical References

---

# 1. Executive Summary

Regolator is a policy-authoring and policy-generation system designed primarily for Crossplane managed resources.

The core architectural change from the previous system is:

> **Policy is represented as structured data in `ResourcePolicy` custom resources rather than generated Rego files.**

A small, generic Rego v1 engine interprets those `ResourcePolicy` rules at runtime.

This removes the old relationship:

```text
CRD
  ↓
Markdown
  ↓
generated .rego files
  ↓
Konstraint
  ↓
ConstraintTemplate + Constraint per Rego file
```

and replaces it with:

```text
CRD schema
    +
ResourcePolicy data
    ↓
generic Rego engine
```

The same policy intent is then consumed by:

1. **Merge Control**, when validating customer infrastructure PRs.
2. **Gatekeeper**, when enforcing policy in Kubernetes clusters.

Regolator itself provides:

- provider schema acquisition;
- persisted CRD snapshots;
- the API behind the policy-authoring SPA;
- creation of `ResourcePolicy` pull requests;
- deterministic policy generation;
- Gatekeeper generation;
- release artifacts.

The CI/CD cluster does **not** run Crossplane providers.

Instead, selected provider CRDs are extracted from Crossplane provider OCI/xpkg images, persisted in Git, and registered on the CI/CD cluster solely as schema objects.

---

# 2. System Boundaries

## 2.1 Regolator

Regolator is the system being built.

It will be a Go binary using Cobra.

The two primary commands are:

```text
regolator api
regolator generate
```

There is intentionally no separate architectural `regolator validate` command.

Validation is part of generation.

---

## 2.2 `regolator api`

`regolator api` is a long-running workload on the CI/CD Kubernetes cluster.

Responsibilities:

- authenticate users using the organization's existing Dex / AD integration;
- discover Regolator-managed CRDs from the CI/CD Kubernetes API;
- discover live `ResourcePolicy` CRs from the CI/CD Kubernetes API;
- normalize CRD schemas;
- overlay current policy rules onto the schema;
- serve a normalized API to the SPA;
- serve the SPA itself if desired;
- accept policy edits from the SPA;
- create feature branches;
- update `ResourcePolicy` YAML;
- raise pull requests.

Important:

> `regolator api` does **not** pull provider images.

It assumes the configured schemas have already been materialized as CRDs in the CI/CD cluster.

---

## 2.3 `regolator generate`

`regolator generate` is a finite build/generation command.

It is expected to run in CI / Argo Workflows.

Responsibilities include:

- parse `config.yaml`;
- determine which provider package images and resources are configured;
- determine whether provider images need to be pulled;
- extract selected CRDs from provider package images when required;
- annotate and persist CRD snapshots;
- create initial skeleton `ResourcePolicy` objects for newly supported resources;
- validate `ResourcePolicy` structure;
- validate configured API versions;
- validate policy paths against the corresponding CRD schema;
- validate array path syntax;
- validate conditions;
- validate enforcement-target configuration;
- render Gatekeeper Constraints;
- ensure generated output is deterministic.

The same command should be safe to run:

- during a PR check;
- after merge;
- during release generation.

---

## 2.4 Merge Control

Merge Control is a **separate existing production system**.

It is not part of Regolator.

Merge Control already validates customer infrastructure pull requests.

It will be updated to consume Regolator's output.

For every Kubernetes manifest being reviewed, Merge Control will:

1. determine Group / Version / Kind;
2. derive the deterministic matching `ResourcePolicy` name;
3. query Kubernetes for that `ResourcePolicy`;
4. fail closed if it does not exist;
5. combine the incoming manifest and policy into the OPA input document;
6. evaluate the generic embedded Rego policy engine;
7. run any exceptional hand-written checks that remain outside the generic engine;
8. aggregate violations;
9. post the existing Markdown PR comment / status.

---

# 3. Core Design Principles

The following are architectural invariants.

1. **CRDs define schema.**
2. **ResourcePolicy defines organizational policy decisions.**
3. **ResourcePolicy does not duplicate the entire CRD schema.**
4. **The SPA displays the complete schema, not merely fields with policy.**
5. **The SPA is a schema + policy overlay UI.**
6. **The generic Rego engine interprets policy data.**
7. **Regolator does not generate one Rego file per resource or field.**
8. **Merge Control and Gatekeeper enforce the same policy vocabulary.**
9. **A missing ResourcePolicy is a deny.**
10. **An unknown or unreviewed API version is a deny.**
11. **The policy language remains deliberately small.**
12. **Exceptional checks may remain explicit Merge Control code.**
13. **Git is the change-control and release mechanism.**
14. **Kubernetes is the runtime lookup mechanism.**
15. **Provider images are upstream schema sources.**
16. **Persisted CRDs are auditable snapshots of those schemas.**
17. **The CI/CD cluster runs schema-only CRDs, not Crossplane provider controllers.**
18. **Generated provider CRDs are read-only artifacts.**
19. **Production-specific policy is represented using conditional rules.**
20. **Late-initialized exceptions are represented using enforcement targets.**
21. **Array semantics are first-class and must be deterministic.**
22. **Generation must succeed before a policy PR can merge.**
23. **The ResourcePolicy and Gatekeeper release must identify the same policy revision.**

---

# 4. Runtime and Cluster Topology

Regolator and Merge Control run on the CI/CD cluster.

Crossplane providers do not.

```text
CI/CD EKS CLUSTER
│
├── Regolator API
├── Merge Control
├── ResourcePolicy CRD
├── ResourcePolicy CRs
└── selected provider/custom CRDs
    └── schema only — no provider controllers
```

The actual Crossplane provider controllers and managed resources live on other target clusters.

```text
TARGET / PLATFORM CLUSTERS
│
├── Crossplane
├── provider controllers
├── managed resources
└── Gatekeeper
    └── Regolator-generated Constraints
```

This means the CI/CD cluster cannot depend on installed providers for schema discovery.

Instead:

```text
provider OCI/xpkg image
        ↓
regolator generate
        ↓
persist CRD snapshot in Git
        ↓
Argo CRD app
        ↓
CI/CD Kubernetes API
        ↓
regolator api
```

---

# 5. Repository Layout

A single policy/output repository may contain:

```text
regolator-policy/
│
├── config.yaml
│
├── crds/
│   ├── providers/
│   │   └── gcp/
│   │       └── bigquery/
│   │           ├── datasets.bigquery.gcp.m.upbound.io.yaml
│   │           └── tables.bigquery.gcp.m.upbound.io.yaml
│   │
│   └── custom/
│       └── applications.platform.mycompany.io.yaml
│
├── resource-policies/
│   ├── bigquery-gcp-m-upbound-io-dataset.yaml
│   └── platform-mycompany-io-application.yaml
│
└── gatekeeper/
    ├── templates/
    │   └── k8s-resource-policy.yaml
    └── constraints/
        ├── bigquery-gcp-m-upbound-io-dataset.yaml
        └── platform-mycompany-io-application.yaml
```

Logical ownership:

```text
config.yaml
    = supported provider/resource inventory

crds/
    = auditable schema snapshots

resource-policies/
    = human-reviewed policy intent

gatekeeper/
    = generated deployment representation
```

Provider-backed files under `crds/` are generated artifacts and must not be manually edited.

---

# 6. `config.yaml`

The main provider configuration should be explicit but small.

Recommended shape:

```yaml
clouds:
  gcp:
    providers:
      - name: BigQuery

        image: >-
          123456789012.dkr.ecr.us-east-1.amazonaws.com/
          provider-gcp-bigquery@sha256:0123456789abcdef...

        resources:
          - group: bigquery.gcp.m.upbound.io
            kind: Dataset

          - group: bigquery.gcp.m.upbound.io
            kind: Table

          - group: bigquery.gcp.m.upbound.io
            kind: Routine
```

## 6.1 `name`

`name` is human-facing.

Example:

```yaml
name: BigQuery
```

It may be used for:

- logging;
- SPA grouping;
- status output;
- directory organization;
- error messages.

It must **not** be part of resource identity.

Resource identity remains:

```text
Group + Kind
```

---

## 6.2 `image`

The field is intentionally named:

```yaml
image:
```

rather than:

```yaml
ecr:
```

because the backing registry is an implementation detail and may change.

The image should preferably be pinned by OCI digest:

```text
repository@sha256:...
```

rather than a mutable tag.

If tags are accepted, Regolator should resolve the actual digest and persist that resolved digest into CRD provenance.

---

## 6.3 `resources`

Resources are identified using:

```yaml
group:
kind:
```

not:

- plural name;
- CRD metadata name;
- filename;
- API version.

Example:

```yaml
resources:
  - group: bigquery.gcp.m.upbound.io
    kind: Dataset
```

Versions are discovered from the selected CRD.

This intentionally matches the identity used throughout Regolator:

```text
ResourcePolicy.spec.target
Merge Control lookup
Gatekeeper matching
CRD extraction
```

---

# 7. Provider Package Images as the Upstream Schema Source

Crossplane provider packages are OCI images / xpkgs.

The Crossplane xpkg specification defines the provider package base layer as containing a root-level `package.yaml` YAML stream.

For provider packages, that YAML stream may contain:

- exactly one Provider package metadata object;
- zero or more Kubernetes `CustomResourceDefinition` objects;
- supported webhook configuration objects.

Regolator only cares about the CRD documents for schema extraction.

Conceptual extraction:

```text
provider image
    ↓
OCI/xpkg base layer
    ↓
package.yaml YAML stream
    ↓
decode documents
    ↓
keep kind: CustomResourceDefinition
    ↓
index by:
    spec.group
    spec.names.kind
```

For each configured resource:

```text
(group, kind)
```

Regolator requires exactly one matching CRD.

If no CRD exists, generation fails.

If more than one ambiguous match exists, generation fails.

Official xpkg specification:

https://github.com/crossplane/crossplane/blob/main/contributing/specifications/xpkg.md

---

# 8. Persisted CRDs as Auditable Schema Snapshots

Provider images remain the **upstream source**.

The CRDs committed under `crds/` become the **pinned auditable schema snapshot** used by Regolator.

This solves several problems:

- prevents pulling every provider image on every generation;
- provides Git history for schema changes;
- makes schema diffs visible;
- allows the CI/CD cluster to register the CRDs through Argo;
- provides a common representation for provider and custom CRDs;
- makes Regolator independent from GitHub provider source repositories;
- works with internally forked providers;
- preserves exact schema associated with the configured OCI image.

The relationship is:

```text
provider package image
        ↓
upstream schema source

persisted CRD
        ↓
auditable snapshot/cache

CI/CD registered CRD
        ↓
runtime schema view used by Regolator API
```

---

# 9. CRD Provenance Metadata

Regolator should annotate every provider-derived CRD it persists.

Example:

```yaml
metadata:
  labels:
    regolator.mycompany.io/managed: "true"
    regolator.mycompany.io/cloud: gcp

  annotations:
    regolator.mycompany.io/provider-name: BigQuery

    regolator.mycompany.io/source-type: xpkg

    regolator.mycompany.io/source-image: >-
      123456789012.dkr.ecr.us-east-1.amazonaws.com/provider-gcp-bigquery

    regolator.mycompany.io/source-digest: >-
      sha256:0123456789abcdef...

    regolator.mycompany.io/schema-root: spec.forProvider

    regolator.mycompany.io/schema-hash: >-
      sha256:abcdef0123456789...
```

Recommended meanings:

| Metadata | Meaning |
|---|---|
| `managed=true` | Regolator API should consider this CRD part of the supported schema inventory |
| `cloud` | SPA grouping |
| `provider-name` | Human-readable provider grouping |
| `source-type` | `xpkg`, `custom`, etc. |
| `source-image` | OCI repository |
| `source-digest` | Exact provider package digest |
| `schema-root` | UI/policy root, normally `spec.forProvider` |
| `schema-hash` | Canonical hash of the relevant schema |

Provider-derived files should also contain a generated-file comment when practical:

```yaml
# GENERATED BY REGOLATOR FROM A PROVIDER PACKAGE.
# DO NOT EDIT THIS FILE MANUALLY.
```

---

# 10. CRD Cache / Provider Pull Algorithm

The provider image should be pulled **once per changed provider**, not once per configured resource.

For each configured provider:

```text
config provider
      ↓
desired image digest
      ↓
inspect persisted CRD provenance
```

## 10.1 Cache hit

If:

- every configured Group + Kind has a persisted CRD;
- every selected CRD records the same desired `source-digest`;
- provenance is complete;

then:

```text
SKIP OCI PULL
```

---

## 10.2 Cache miss

Pull the provider package once when any of these are true:

- provider digest changed;
- configured CRD snapshot is missing;
- provenance is missing or invalid;
- a new resource was added under the provider;
- regeneration was explicitly requested.

Then:

```text
pull provider image once
    ↓
extract package.yaml
    ↓
parse all CRDs once
    ↓
index CRDs by Group + Kind
    ↓
select all configured resources
    ↓
annotate selected CRDs
    ↓
write canonical snapshots under crds/
```

With 30 providers and one provider changing:

```text
29 providers → cache hit → zero pull
1 provider   → cache miss → one pull
```

---

## 10.3 Removed resources

If a resource is removed from `config.yaml`, generated provider-backed CRD snapshots that are no longer configured should be considered stale.

`regolator generate` should either:

- remove them automatically; or
- fail generation and explicitly report the stale generated file.

Automatic deterministic cleanup is preferable.

This prevents removed resources from remaining visible in the SPA simply because an old CRD file still exists.

---

# 11. Registering Schema-Only CRDs in the CI/CD Cluster

An Argo Application should sync the `crds/` directory to the CI/CD cluster.

Example:

```text
Git crds/
   ↓
Argo App: regolator-crds
   ↓
CI/CD API server
```

No provider controllers are installed.

No ProviderConfig is installed.

No reconciliation occurs.

The CRDs are present solely so the Kubernetes API contains the schema.

This allows `regolator api` to remain Kubernetes-native:

```text
Regolator API
    ↓
Kubernetes API
    ├── managed CRDs
    └── ResourcePolicies
```

The API does not need to parse `config.yaml`.

The configured inventory is indirectly materialized as the set of CRDs labeled:

```text
regolator.mycompany.io/managed=true
```

---

## 11.1 Schema-only CRD safety check

Before registering provider CRDs on a cluster without provider controllers, generation should inspect CRD features that may rely on external runtime components.

In particular, Regolator should detect conversion configuration such as:

```yaml
spec:
  conversion:
    strategy: Webhook
```

If a selected CRD requires an unavailable conversion webhook, Regolator should:

- reject it by default; or
- explicitly normalize it for schema-only use through a separately designed mechanism.

Do not silently install a schema that depends on a nonexistent webhook.

This may never occur for the provider CRDs in use, but it is worth enforcing.

---

# 12. Custom / Handwritten CRDs

Persisting CRDs creates a clean extension point for non-Crossplane resources.

Example:

```text
crds/custom/applications.platform.mycompany.io.yaml
```

A custom CRD should carry the same Regolator metadata:

```yaml
metadata:
  labels:
    regolator.mycompany.io/managed: "true"
    regolator.mycompany.io/cloud: platform

  annotations:
    regolator.mycompany.io/provider-name: Internal Platform
    regolator.mycompany.io/source-type: custom
    regolator.mycompany.io/schema-root: spec
```

Provider-backed CRDs:

```text
upstream = provider OCI image
snapshot = generated/read-only CRD
```

Custom CRDs:

```text
upstream = checked-in CRD itself
```

The downstream Regolator model does not care where the CRD came from.

---

## 12.1 Schema root

Crossplane defaults to:

```text
spec.forProvider
```

A custom resource may use:

```text
spec
```

or another subtree.

The normalized CRD should therefore expose a schema-root annotation:

```yaml
regolator.mycompany.io/schema-root: spec.forProvider
```

or:

```yaml
regolator.mycompany.io/schema-root: spec
```

This allows the Regolator API to remain independent from `config.yaml`.

---

# 13. Regolator API

`regolator api` runs continuously on the CI/CD cluster.

It should discover supported CRDs using a metadata selector where possible:

```text
regolator.mycompany.io/managed=true
```

For each supported CRD, it obtains:

- Group;
- Kind;
- human provider grouping;
- schema root;
- versions;
- OpenAPI schema;
- descriptions;
- types;
- child properties;
- array item schemas;
- map schemas.

It separately obtains the matching live `ResourcePolicy`.

It then builds the UI model:

```text
CRD schema
   +
ResourcePolicy
   ↓
normalized schema + policy overlay
```

---

# 14. SPA Navigation and User Model

The SPA should be schema-oriented, not policy-oriented.

Conceptually:

```text
GCP
├── BigQuery
│   ├── Dataset
│   │   ├── v1beta1
│   │   └── v1beta2
│   ├── Table
│   └── Routine
│
├── Storage
│   └── Bucket
│
└── Pub/Sub
    └── Topic
```

Selecting:

```text
GCP → BigQuery → Dataset → v1beta1
```

displays the entire configured schema root.

Example:

```text
spec.forProvider
├── project                         string
├── location                        string
├── deleteContentsOnDestroy         boolean
├── access[]                        array<object>
│   ├── role                        string
│   ├── userByEmail                 string
│   └── groupByEmail                string
└── ...
```

Fields with policy display their policy.

Fields without policy remain visible.

---

# 15. Schema + Policy Overlay Contract

A normalized API node could look like:

```json
{
  "name": "location",
  "path": [
    "spec",
    "forProvider",
    "location"
  ],
  "displayPath": "spec.forProvider.location",
  "type": "string",
  "description": "The geographic location of the dataset.",
  "children": [],
  "rules": [
    {
      "type": "AllowedValues",
      "values": [
        "us-east1",
        "us-east4"
      ]
    }
  ]
}
```

An unrestricted field:

```json
{
  "name": "friendlyName",
  "path": [
    "spec",
    "forProvider",
    "friendlyName"
  ],
  "displayPath": "spec.forProvider.friendlyName",
  "type": "string",
  "description": "...",
  "children": [],
  "rules": []
}
```

The UI is therefore capable of displaying an untouched resource with:

```text
50 schema fields
0 policy rules
```

and allowing the first policy decisions to be made.

---

# 16. ResourcePolicy Overview

The policy CRD is:

```yaml
apiVersion: policy.mycompany.io/v1alpha1
kind: ResourcePolicy
```

There is logically one `ResourcePolicy` per:

```text
Group + Kind
```

Example:

```yaml
apiVersion: policy.mycompany.io/v1alpha1
kind: ResourcePolicy
metadata:
  name: bigquery-gcp-m-upbound-io-dataset

spec:
  target:
    group: bigquery.gcp.m.upbound.io
    kind: Dataset

  unknownVersionPolicy: Deny

  versions:
    - name: v1beta1
      state: Allowed
      rules: []

    - name: v1beta2
      state: Unreviewed
      rules: []
```

The ResourcePolicy contains **only policy state**.

It does not copy every field from the CRD.

---

# 17. Deterministic ResourcePolicy Naming

Merge Control should avoid listing all ResourcePolicies.

Given:

```text
group = bigquery.gcp.m.upbound.io
kind  = Dataset
```

both Regolator and Merge Control should call the same deterministic naming function.

Example output:

```text
bigquery-gcp-m-upbound-io-dataset
```

Then Merge Control performs a direct cluster-scoped GET:

```text
GET ResourcePolicy/bigquery-gcp-m-upbound-io-dataset
```

Useful labels should still exist for browsing:

```yaml
metadata:
  labels:
    regolator.mycompany.io/group: bigquery.gcp.m.upbound.io
    regolator.mycompany.io/kind: Dataset
```

The exact normalization algorithm must be:

- deterministic;
- DNS-name safe;
- collision-resistant;
- shared from a common Go package if possible.

If truncation is required, append a stable hash.

---

# 18. ResourcePolicy Version Model

Each Group + Kind contains policy for N API versions.

Example:

```yaml
versions:
  - name: v1beta1
    state: Allowed
    rules: [...]

  - name: v1beta2
    state: Allowed
    rules: [...]

  - name: v1alpha1
    state: Denied
    reason: Alpha API versions are not approved.
    rules: []
```

Supported states:

```text
Unreviewed
Allowed
Denied
```

Important distinction:

```yaml
state: Allowed
rules: []
```

means:

> Security reviewed this version and allows it with no field restrictions.

While:

```yaml
state: Unreviewed
rules: []
```

means:

> This version has not been approved.

Unknown versions default to:

```yaml
unknownVersionPolicy: Deny
```

---

# 19. ResourcePolicy Rule Vocabulary

Initial field rule operators:

```text
Forbidden
Required
MustEqual
AllowedValues
DeniedValues
AllowedRegex
DeniedRegex
```

Possible later additions:

```text
MinItems
MaxItems
MinValue
MaxValue
```

Rule names are **Regolator vocabulary**.

They are not OPA built-ins.

For example:

```yaml
type: MustEqual
```

works because the generic Rego engine contains logic that says:

```rego
rule.type == "MustEqual"
```

and then implements that operator.

The policy language should remain intentionally limited.

Do not expose arbitrary Rego in `ResourcePolicy`.

---

# 20. Conditional Rules / Production Policy

The current platform distinguishes production and non-production projects using the explicitly required project field.

Examples:

```text
non-...
prod-...
```

Production policy should no longer require a second policy object such as:

```text
prod-bigquery-dataset
```

Instead, production restrictions are ordinary rules with a condition.

Example:

```yaml
- path:
    - spec
    - forProvider
    - location

  type: AllowedValues

  values:
    - us-east4

  when:
    path:
      - spec
      - forProvider
      - project

    type: StartsWith
    value: prod-
```

Semantics:

```text
base rules
+
rules whose when condition matches
```

A non-production project receives only base policy.

A production project receives base policy plus production-specific policy.

---

## 20.1 Initial condition vocabulary

Initial condition types:

```text
Equals
NotEquals
StartsWith
EndsWith
MatchesRegex
Exists
NotExists
```

Keep the initial model simple:

> One optional `when` condition per rule.

Do not add arbitrary nested AND/OR expression trees without a real requirement.

---

# 21. Enforcement Targets / Late Initialization

Some rules are valid in customer-authored configuration but cannot safely be applied to the reconciled live object.

Example:

- customers must use `TableSelector`;
- customers are forbidden from setting `TableRef`;
- Crossplane late-initializes `TableRef`;
- Merge Control must reject customer-authored `TableRef`;
- Gatekeeper must not reject Crossplane's late-initialized `TableRef`.

The old model represented this as:

```text
SkipTemplate
```

The new model should describe the actual enforcement intent:

```yaml
- path:
    - spec
    - forProvider
    - tableRef

  type: Forbidden

  enforcement:
    mergeControl: true
    gatekeeper: false

  reason: >-
    Customers must use tableSelector.
    tableRef is late initialized by Crossplane.
```

Default:

```yaml
enforcement:
  mergeControl: true
  gatekeeper: true
```

This is more general than `SkipTemplate` and avoids encoding the old Konstraint implementation into the new policy model.

---

# 22. Policy Path Representation

Policy paths should be stored as structured tokens.

Normal path:

```yaml
path:
  - spec
  - forProvider
  - location
```

Array traversal:

```yaml
path:
  - spec
  - forProvider
  - access
  - "[]"
  - role
```

Nested arrays:

```yaml
path:
  - spec
  - forProvider
  - foo
  - "[]"
  - bar
  - "[]"
  - baz
```

Human display form:

```text
spec.forProvider.foo[].bar[].baz
```

The `[]` token means:

> Match every concrete array index at this position.

This avoids trying to overload a field-name string such as:

```text
"access[]"
```

and gives the Rego engine a clean token stream.

---

# 23. Array Path Semantics

Suppose a manifest contains:

```yaml
spec:
  forProvider:
    lists:
      - name: first
        encryption: true

      - name: second
        encryption: false

      - name: third
```

Policy:

```yaml
- path:
    - spec
    - forProvider
    - lists
    - "[]"
    - encryption

  type: MustEqual
  value: true
```

Semantics:

```text
lists[0].encryption = true     → pass
lists[1].encryption = false    → violation
lists[2].encryption = missing  → violation
```

Definition:

> `MustEqual` on an array-wildcard path means every existing parent element addressed by the wildcard path must contain the final field and that field must equal the configured value.

If:

```yaml
lists: []
```

there are no elements to violate the rule, so the rule passes.

If the policy requires the list itself to be non-empty, that should be a separate rule such as a future:

```text
MinItems
```

Do not overload `MustEqual` to mean list cardinality.

---

# 24. Generic Monolithic Rego Policy Engine

The old system generated resource-specific Rego.

The new system contains one generic engine, conceptually:

```text
fieldpolicy.rego
```

That engine is embedded into Merge Control.

It contains implementations for:

- version selection;
- unknown-version handling;
- version state handling;
- path resolution;
- wildcard array traversal;
- conditions;
- enforcement target filtering;
- `Forbidden`;
- `Required`;
- `MustEqual`;
- `AllowedValues`;
- `DeniedValues`;
- regex rules;
- violation construction.

New resources do **not** change Rego.

New field policies do **not** change Rego.

Only a brand-new policy operator requires changing the engine.

---

# 25. Array Policy Proof of Concept

OPA provides the `walk()` built-in.

Official documentation:

https://www.openpolicyagent.org/docs/policy-reference/builtins/graph

`walk(x, output)` recursively produces:

```text
[path, value]
```

pairs for nested documents.

For the example:

```yaml
spec:
  forProvider:
    lists:
      - encryption: true
      - encryption: false
```

`walk()` can expose concrete paths such as:

```text
["spec", "forProvider", "lists", 0]
["spec", "forProvider", "lists", 1]
```

The ResourcePolicy pattern is:

```text
["spec", "forProvider", "lists", "[]"]
```

The matching rule is:

```text
"[]" matches any numeric array index
all other segments match exactly
```

---

## 25.1 Rego prototype

The following is the intended shape of the generic resolver.

It should be compile-tested and unit-tested against the exact OPA version selected for implementation, but it uses documented Rego v1 concepts.

```rego
package fieldpolicy


# Unique sentinel for a missing field.
missing := {"__regolator_missing__": true}


# ------------------------------------------------------------
# PATH MATCHING
# ------------------------------------------------------------

# [] matches a concrete numeric array index.
segment_matches(expected, actual) if {
    expected == "[]"
    is_number(actual)
}

# Ordinary segments match literally.
segment_matches(expected, actual) if {
    expected != "[]"
    expected == actual
}


# A concrete path matches a policy pattern when:
#   1. lengths match
#   2. every corresponding segment matches
path_matches(pattern, concrete) if {
    count(pattern) == count(concrete)

    every i, expected in pattern {
        segment_matches(expected, concrete[i])
    }
}


contains_wildcard(path) if {
    "[]" in path
}


# ------------------------------------------------------------
# TARGET RESOLUTION
# ------------------------------------------------------------

# Direct paths can use object.get() efficiently.
direct_targets(rule) contains target if {
    not contains_wildcard(rule.path)

    actual := object.get(
        input.review.object,
        rule.path,
        missing,
    )

    target := {
        "path": rule.path,
        "actual": actual,
    }
}


# Wildcard paths use walk() to locate every concrete parent.
#
# Policy:
#   spec.forProvider.lists[].encryption
#
# Parent pattern:
#   spec.forProvider.lists[]
#
# Leaf:
#   encryption
wildcard_targets(rule) contains target if {
    leaf_index := count(rule.path) - 1

    parent_pattern := array.slice(
        rule.path,
        0,
        leaf_index,
    )

    leaf := rule.path[leaf_index]

    walk(
        input.review.object,
        [parent_path, parent],
    )

    path_matches(
        parent_pattern,
        parent_path,
    )

    is_object(parent)

    actual := object.get(
        parent,
        leaf,
        missing,
    )

    target := {
        "path": array.concat(parent_path, [leaf]),
        "actual": actual,
    }
}


targets(rule) := direct_targets(rule) if {
    not contains_wildcard(rule.path)
}

targets(rule) := wildcard_targets(rule) if {
    contains_wildcard(rule.path)
}


# ------------------------------------------------------------
# MUST EQUAL
# ------------------------------------------------------------

violation contains {
    "msg": msg,
    "path": target.path,
} if {
    some rule in selected_rules

    rule.type == "MustEqual"

    rule_applies(rule)

    enforcement_enabled(rule)

    some target in targets(rule)

    target.actual != rule.value

    msg := sprintf(
        "%v must equal %v; got %v",
        [
            target.path,
            rule.value,
            target.actual,
        ],
    )
}
```

---

## 25.2 Why nested arrays work

Policy:

```text
spec.forProvider.foo[].bar[].baz
```

Stored path:

```yaml
path:
  - spec
  - forProvider
  - foo
  - "[]"
  - bar
  - "[]"
  - baz
```

A concrete object found by `walk()` may have a parent path:

```text
[
  "spec",
  "forProvider",
  "foo",
  2,
  "bar",
  7
]
```

Pattern:

```text
[
  "spec",
  "forProvider",
  "foo",
  "[]",
  "bar",
  "[]"
]
```

Matching:

```text
spec          == spec
forProvider   == forProvider
foo           == foo
[]            matches 2
bar           == bar
[]            matches 7
```

No generated Rego is required.

The nesting complexity is data.

---

## 25.3 Direct-path optimization

OPA `object.get()` supports nested path arrays.

Official documentation:

https://www.openpolicyagent.org/docs/policy-reference/builtins/object

For example:

```rego
object.get(
    input.review.object,
    ["spec", "forProvider", "location"],
    missing,
)
```

Therefore:

```text
path without []
    → object.get()

path containing []
    → walk() + wildcard matcher
```

This avoids recursively walking the entire manifest for ordinary scalar paths.

---

## 25.4 `every` and `some`

OPA documents:

```rego
some x in collection
```

for selecting elements and:

```rego
every x in collection {
    ...
}
```

for universal quantification.

References:

https://www.openpolicyagent.org/docs/policy-language

https://www.openpolicyagent.org/docs/policy-reference/keywords/some

https://www.openpolicyagent.org/docs/policy-reference/keywords/every

The path matcher uses `every` because every segment in the path must match.

The violation engine uses `some rule in ...` and `some target in ...` because each matching rule/target may independently produce a violation.

---

# 26. Rego Version Selection

The incoming object contains:

```yaml
apiVersion: bigquery.gcp.m.upbound.io/v1beta2
```

The engine derives:

```text
v1beta2
```

and selects that entry from:

```yaml
resourcePolicy:
  versions:
    - name: v1beta1
      ...
    - name: v1beta2
      ...
```

Conceptually:

```rego
resource_version := version if {
    parts := split(
        input.review.object.apiVersion,
        "/",
    )

    count(parts) == 2
    version := parts[1]
}


selected_version := version_policy if {
    some version_policy in input.resourcePolicy.versions

    version_policy.name == resource_version
}


selected_rules := object.get(
    selected_version,
    "rules",
    [],
)
```

The implementation must also handle:

- missing version;
- `state: Unreviewed`;
- `state: Denied`;
- `unknownVersionPolicy: Deny`.

All should fail closed.

---

# 27. Rego Conditional Evaluation

A rule with no `when` applies normally.

A rule with `when` applies only when the condition succeeds.

Example policy:

```yaml
when:
  path:
    - spec
    - forProvider
    - project

  type: StartsWith
  value: prod-
```

Conceptual helper:

```rego
rule_applies(rule) if {
    object.get(rule, "when", null) == null
}


rule_applies(rule) if {
    condition := rule.when

    condition.type == "StartsWith"

    actual := object.get(
        input.review.object,
        condition.path,
        missing,
    )

    actual != missing

    startswith(actual, condition.value)
}
```

Other condition operators should follow the same model.

The initial `when` implementation does not need nested boolean expressions.

---

# 28. Rego Enforcement Target Filtering

The core semantics should understand which enforcement environment is evaluating a rule.

Merge Control evaluates:

```text
mergeControl
```

Gatekeeper evaluates:

```text
gatekeeper
```

Conceptually:

```rego
enforcement_enabled(rule) if {
    enforcement := object.get(
        rule,
        "enforcement",
        {},
    )

    object.get(
        enforcement,
        input.enforcementTarget,
        true,
    ) == true
}
```

Merge Control can include:

```json
"enforcementTarget": "mergeControl"
```

in its input.

Gatekeeper has a different fixed input model, so the Gatekeeper adapter/template can use the same core semantics with a fixed target of:

```text
gatekeeper
```

Alternatively, Gatekeeper generation may remove rules where:

```yaml
enforcement:
  gatekeeper: false
```

before placing rules in Constraint parameters.

Filtering at generation time is simple and reduces Gatekeeper input size.

---

# 29. Merge Control Integration

Merge Control's existing customer infrastructure flow becomes:

```text
customer PR
    ↓
parse YAML documents
    ↓
for each manifest:
    ↓
determine Group / Version / Kind
    ↓
derive deterministic ResourcePolicy name
    ↓
GET ResourcePolicy from CI/CD cluster
    ↓
construct OPA input
    ↓
evaluate prepared generic Rego query
    ↓
aggregate violations
    ↓
run exceptional handwritten checks
    ↓
post Markdown result
```

There is no:

- policy bundle download;
- archive expansion;
- directory search;
- generated-Rego naming convention;
- one-file-per-field policy lookup.

---

# 30. Merge Control Fail-Closed Behavior

This preserves current production behavior.

If a customer submits a resource and no corresponding policy exists:

```text
DENY
```

Conceptually:

```text
This resource is not supported at this time.
```

Fail closed when:

- ResourcePolicy not found;
- Group/Kind mismatch;
- API version not represented and unknown-version policy is Deny;
- version state is Unreviewed;
- version state is Denied;
- ResourcePolicy cannot be decoded;
- Rego preparation failed;
- evaluation fails unexpectedly.

Do not treat missing policy as unrestricted policy.

---

# 31. OPA Input Contract

For Merge Control, a normalized input may be:

```json
{
  "review": {
    "object": {
      "apiVersion": "example.mycompany.io/v1beta1",
      "kind": "Thing",
      "metadata": {
        "name": "example"
      },
      "spec": {
        "forProvider": {
          "lists": [
            {
              "name": "first",
              "encryption": true
            },
            {
              "name": "second",
              "encryption": false
            },
            {
              "name": "third"
            }
          ]
        }
      }
    }
  },

  "resourcePolicy": {
    "target": {
      "group": "example.mycompany.io",
      "kind": "Thing"
    },

    "unknownVersionPolicy": "Deny",

    "versions": [
      {
        "name": "v1beta1",
        "state": "Allowed",

        "rules": [
          {
            "path": [
              "spec",
              "forProvider",
              "lists",
              "[]",
              "encryption"
            ],

            "type": "MustEqual",
            "value": true
          }
        ]
      }
    ]
  },

  "enforcementTarget": "mergeControl"
}
```

Important:

- `input` is simply the JSON object passed to OPA.
- `review`, `resourcePolicy`, and `enforcementTarget` are Regolator/Merge Control contract names.
- They are not magic OPA keywords.
- Rego accesses them because the caller created those keys.

---

# 32. OPA Go Integration

OPA's official Go integration recommends:

1. construct a prepared query;
2. reuse it for evaluations;
3. pass a new input document for each evaluation.

Official documentation:

https://www.openpolicyagent.org/docs/integration

The Rego module should be embedded into Merge Control.

Conceptual Go:

```go
//go:embed fieldpolicy.rego
var fieldPolicyRego string
```

Prepare once:

```go
prepared, err := rego.New(
    rego.Query("data.fieldpolicy.violation"),
    rego.Module(
        "fieldpolicy.rego",
        fieldPolicyRego,
    ),
).PrepareForEval(ctx)

if err != nil {
    return err
}
```

Evaluate for each manifest:

```go
results, err := prepared.Eval(
    ctx,
    rego.EvalInput(input),
)
```

Prepared queries should be reused rather than recompiling Rego for every manifest.

OPA documents prepared queries as reusable and safe to share across goroutines.

---

# 33. Gatekeeper Architecture

Gatekeeper should not receive one ConstraintTemplate per policy rule.

The intended design is:

```text
small fixed number of generic ConstraintTemplates
+
one Constraint per ResourcePolicy
```

Likely:

```text
1 generic K8sResourcePolicy ConstraintTemplate
N ResourcePolicy-backed Constraints
```

The template contains generic Rego policy code.

The Constraint contains policy data.

Gatekeeper exposes Constraint parameters to Rego as:

```rego
input.parameters
```

Official documentation:

https://open-policy-agent.github.io/gatekeeper/website/docs/constrainttemplates/

https://open-policy-agent.github.io/gatekeeper/website/docs/howto/

Gatekeeper supports Rego v1 in modern ConstraintTemplates using:

```yaml
code:
  - engine: Rego
    source:
      version: "v1"
      rego: |
        ...
```

---

# 34. Gatekeeper Generation

A ResourcePolicy:

```yaml
kind: ResourcePolicy
metadata:
  name: bigquery-gcp-m-upbound-io-dataset
```

becomes a Constraint such as:

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sResourcePolicy

metadata:
  name: bigquery-gcp-m-upbound-io-dataset

spec:
  match:
    kinds:
      - apiGroups:
          - bigquery.gcp.m.upbound.io
        kinds:
          - Dataset

  parameters:
    unknownVersionPolicy: Deny
    versions:
      - name: v1beta1
        state: Allowed
        rules:
          ...
```

Rules with:

```yaml
enforcement:
  gatekeeper: false
```

should be omitted from Gatekeeper parameters during generation.

This handles late initialization without weakening Merge Control.

---

# 35. Regolator CLI

Primary commands:

```text
regolator api
regolator generate
```

## `regolator api`

Long-running server.

## `regolator generate`

Finite deterministic build.

Possible flags may later include:

```text
--config
--repo-root
--output
--check
```

A `--check` flag is CLI ergonomics, not a separate validation architecture.

There is no need for:

```text
regolator validate
```

as a separate system concept.

---

# 36. PR-Time Generation and Validation

Validation must happen before merge.

For any ResourcePolicy/config PR, Merge Control should require successful generation.

Conceptually:

```text
policy/config PR
      ↓
Merge Control required check
      ↓
regolator generate
      │
      ├── parse config
      ├── resolve provider schema if needed
      ├── validate persisted CRDs
      ├── validate ResourcePolicies
      ├── validate field paths
      ├── validate [] traversal
      ├── validate conditions
      ├── validate enforcement targets
      ├── render Gatekeeper output
      └── exit nonzero on error
      ↓
pass / fail
```

An invalid policy cannot be merged.

---

# 37. Policy Authoring PR Flow

For an existing resource:

```text
Regolator API
    ↓
GET CRD from Kubernetes
    ↓
GET ResourcePolicy
    ↓
serve schema + policy overlay
    ↓
user edits through SPA
    ↓
Raise PR
    ↓
Regolator API fetches current Git main
    ↓
create feature branch
    ↓
update resource-policies/<resource>.yaml
    ↓
optionally run regulator generate
    ↓
push branch
    ↓
open PR
    ↓
Merge Control runs regulator generate as required check
    ↓
Platform + Security approval
    ↓
merge
```

Normal Git merge conflict behavior is sufficient.

Policy changes are infrequent and do not require collaborative document editing.

---

# 38. Config / Provider Update Flow

A platform engineer changes:

```text
config.yaml
```

Examples:

- add Dataset;
- add Table;
- add another provider;
- bump provider digest;
- remove a supported resource.

Generation performs:

```text
config change
    ↓
desired provider digest/resource set
    ↓
inspect crds/ provenance
    ↓
cache hit?
   /       \
 yes        no
  │          │
  │          └── pull provider xpkg once
  │              parse package.yaml
  │              select configured CRDs
  │              annotate snapshots
  │              write crds/
  │
  └──────────────► reconcile skeleton ResourcePolicies
                   validate
                   render Gatekeeper
```

For a newly configured resource, generation should create an initial skeleton:

```yaml
spec:
  target:
    group: ...
    kind: ...

  unknownVersionPolicy: Deny

  versions:
    - name: v1beta1
      state: Unreviewed
      rules: []
```

After merge:

```text
crds/
   ↓
Argo CRD app
   ↓
CI/CD API server
   ↓
Regolator API automatically discovers new resource
```

No API restart or config parsing is required.

---

# 39. Post-Merge Generation

After a PR merges:

```text
merge
  ↓
Argo Workflow / CI
  ↓
regolator generate
```

The post-merge run uses the exact merged commit.

It should reproduce the same output that succeeded pre-merge.

Purposes:

- persist deterministic generated artifacts;
- prepare release content;
- create/update Gatekeeper output;
- update CRD snapshots if config changed;
- provide release metadata.

It is not the first point at which invalid policy is detected.

---

# 40. Argo Applications

At minimum there are three logical sync targets.

## 40.1 CRD schema app

```text
Argo App: regolator-crds
```

Source:

```text
crds/
```

Destination:

```text
CI/CD cluster
```

---

## 40.2 ResourcePolicy app

```text
Argo App: resource-policies
```

Source:

```text
resource-policies/
```

Destination:

```text
CI/CD cluster
```

This is the live runtime policy store consumed by Merge Control.

---

## 40.3 Gatekeeper policy app

```text
Argo App: gatekeeper-policy
```

Source:

```text
gatekeeper/
```

Destination:

```text
Crossplane target clusters
```

There may be multiple Argo applications per environment/cluster using the same generated policy release.

---

# 41. Release Identity, Promotion, and Rollback

ResourcePolicy and Gatekeeper output must correspond to the same logical policy revision.

Example release:

```text
policy-release-v42
```

It represents:

```text
ResourcePolicy CRs
+
Gatekeeper Constraints
+
associated CRD/config revision
```

Recommended release metadata can include:

```yaml
regolator.mycompany.io/release: policy-release-v42
```

Argo can pin Git tags or immutable commit SHAs.

The desired model:

```text
DEV  → release 43
TEST → release 42
PROD → release 41
```

Promotion moves the exact same generated output forward.

Rollback moves the Argo target revision back to a previously known-good release.

No OCI Helm package is required for rollback semantics.

Argo tracking reference:

https://argo-cd.readthedocs.io/en/stable/user-guide/tracking_strategies/

---

# 42. Lower-Environment Testing

After generation:

```text
generated policy release
    ↓
lower environment
```

Tests should include:

- expected violations;
- expected passes;
- production conditional rules;
- non-production behavior;
- array wildcard rules;
- missing array child fields;
- late-initialized fields;
- Gatekeeper admission;
- Merge Control evaluation.

Automated KinD testing is also appropriate.

Examples:

```text
should violate
should not violate
```

should be part of the rule-engine test suite.

---

# 43. Schema Drift

Persisted CRDs make schema drift visible.

Regolator should compute a canonical hash of the relevant schema root, for example:

```text
spec.forProvider
```

and persist:

```yaml
regolator.mycompany.io/schema-hash: sha256:...
```

Useful distinctions:

```text
source digest changed
schema hash changed
```

versus:

```text
source digest changed
schema hash unchanged
```

This can eventually drive UI warnings such as:

> Dataset v1beta2 schema changed since policy review.

The exact policy for forced re-review can be introduced later.

---

# 44. Rough ResourcePolicy CRD

This is a **rough implementation draft**, not the final production CRD.

In particular, the arbitrary JSON `value` / `values` fields should be generated/tested carefully against Kubernetes structural-schema requirements.

Kubernetes documents `x-kubernetes-preserve-unknown-fields: true` as a way for a CRD field to hold arbitrary JSON.

Reference:

https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition

metadata:
  name: resourcepolicies.policy.mycompany.io

spec:
  group: policy.mycompany.io

  scope: Cluster

  names:
    plural: resourcepolicies
    singular: resourcepolicy
    kind: ResourcePolicy
    shortNames:
      - rp

  versions:
    - name: v1alpha1
      served: true
      storage: true

      schema:
        openAPIV3Schema:
          type: object

          properties:
            apiVersion:
              type: string

            kind:
              type: string

            metadata:
              type: object

            spec:
              type: object

              required:
                - target
                - unknownVersionPolicy
                - versions

              properties:
                target:
                  type: object

                  required:
                    - group
                    - kind

                  properties:
                    group:
                      type: string
                      minLength: 1

                    kind:
                      type: string
                      minLength: 1

                unknownVersionPolicy:
                  type: string

                  enum:
                    - Allow
                    - Deny

                  default: Deny

                versions:
                  type: array

                  items:
                    type: object

                    required:
                      - name
                      - state
                      - rules

                    properties:
                      name:
                        type: string
                        minLength: 1

                      state:
                        type: string

                        enum:
                          - Unreviewed
                          - Allowed
                          - Denied

                      reason:
                        type: string

                      rules:
                        type: array

                        items:
                          type: object

                          required:
                            - path
                            - type

                          properties:
                            path:
                              type: array
                              minItems: 1

                              items:
                                type: string
                                minLength: 1

                              description: >-
                                Structured field path.
                                The token [] represents any array index.
                                Example:
                                [spec, forProvider, access, "[]", role]

                            type:
                              type: string

                              enum:
                                - Forbidden
                                - Required
                                - MustEqual
                                - AllowedValues
                                - DeniedValues
                                - AllowedRegex
                                - DeniedRegex

                            value:
                              x-kubernetes-preserve-unknown-fields: true

                            values:
                              type: array

                              items:
                                x-kubernetes-preserve-unknown-fields: true

                            pattern:
                              type: string

                            reason:
                              type: string

                            enforcement:
                              type: object

                              properties:
                                mergeControl:
                                  type: boolean
                                  default: true

                                gatekeeper:
                                  type: boolean
                                  default: true

                            when:
                              type: object

                              required:
                                - path
                                - type

                              properties:
                                path:
                                  type: array
                                  minItems: 1

                                  items:
                                    type: string
                                    minLength: 1

                                type:
                                  type: string

                                  enum:
                                    - Equals
                                    - NotEquals
                                    - StartsWith
                                    - EndsWith
                                    - MatchesRegex
                                    - Exists
                                    - NotExists

                                value:
                                  x-kubernetes-preserve-unknown-fields: true
```

Future CRD validation should enforce rule-specific contracts, for example:

```text
MustEqual       requires value
AllowedValues   requires values
DeniedValues    requires values
AllowedRegex    requires pattern
DeniedRegex     requires pattern
```

This can be implemented in Go validation and potentially CRD CEL validation where practical.

---

# 45. Complete ResourcePolicy Example

```yaml
apiVersion: policy.mycompany.io/v1alpha1
kind: ResourcePolicy

metadata:
  name: bigquery-gcp-m-upbound-io-dataset

  labels:
    regolator.mycompany.io/group: bigquery.gcp.m.upbound.io
    regolator.mycompany.io/kind: Dataset

spec:
  target:
    group: bigquery.gcp.m.upbound.io
    kind: Dataset

  unknownVersionPolicy: Deny

  versions:
    - name: v1beta1
      state: Allowed

      rules:

        # Customers must explicitly specify project.
        - path:
            - spec
            - forProvider
            - project

          type: Required


        # Destructive dataset deletion must remain disabled.
        - path:
            - spec
            - forProvider
            - deleteContentsOnDestroy

          type: MustEqual
          value: false


        # Base location policy.
        - path:
            - spec
            - forProvider
            - location

          type: AllowedValues

          values:
            - us-east1
            - us-east4


        # Additional production-only restriction.
        - path:
            - spec
            - forProvider
            - location

          type: AllowedValues

          values:
            - us-east4

          when:
            path:
              - spec
              - forProvider
              - project

            type: StartsWith
            value: prod-


        # Example array policy:
        # every access element must use one of the approved roles.
        - path:
            - spec
            - forProvider
            - access
            - "[]"
            - role

          type: AllowedValues

          values:
            - READER
            - WRITER


        # Merge Control restriction only.
        # Gatekeeper cannot safely enforce this because Crossplane
        # late-initializes the field.
        - path:
            - spec
            - forProvider
            - tableRef

          type: Forbidden

          enforcement:
            mergeControl: true
            gatekeeper: false

          reason: >-
            Customers must use tableSelector.
            tableRef is late initialized by Crossplane.


    - name: v1beta2
      state: Unreviewed
      rules: []


    - name: v1alpha1
      state: Denied
      reason: Alpha API versions are not approved.
      rules: []
```

---

# 46. Example `config.yaml`

```yaml
clouds:

  gcp:

    providers:

      - name: BigQuery

        image: >-
          123456789012.dkr.ecr.us-east-1.amazonaws.com/
          provider-gcp-bigquery@sha256:0123456789abcdef0123456789abcdef

        resources:

          - group: bigquery.gcp.m.upbound.io
            kind: Dataset

          - group: bigquery.gcp.m.upbound.io
            kind: Table

          - group: bigquery.gcp.m.upbound.io
            kind: Routine


      - name: Storage

        image: >-
          123456789012.dkr.ecr.us-east-1.amazonaws.com/
          provider-gcp-storage@sha256:abcdef0123456789abcdef0123456789

        resources:

          - group: storage.gcp.m.upbound.io
            kind: Bucket
```

Provider version numbers do not need to be duplicated here.

The provider CRD defines its versions.

---

# 47. End-to-End Architecture

```text
                            POLICY / OUTPUT GIT REPOSITORY
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  config.yaml                                                           │
│                                                                        │
│  crds/                  resource-policies/             gatekeeper/      │
│  ├── providers/...      ├── Dataset RP                ├── template     │
│  └── custom/...         └── Bucket RP                 └── constraints  │
│                                                                        │
└──────────────┬────────────────────┬──────────────────────────┬──────────┘
               │                    │                          │
               │                    │                          │
               ▼                    ▼                          ▼
       Argo: regolator-crds   Argo: ResourcePolicy      Argo: Gatekeeper
               │                    │                          │
               │                    │                          │
               ▼                    ▼                          ▼
       ┌─────────────────────────────────┐          ┌─────────────────────┐
       │        CI/CD EKS CLUSTER        │          │ PLATFORM CLUSTERS   │
       │                                 │          │                     │
       │ selected CRDs                   │          │ Crossplane          │
       │ ResourcePolicy CRs              │          │ providers           │
       │ Regolator API                   │          │ Gatekeeper          │
       │ Merge Control                   │          │ generated policy    │
       └──────────────┬──────────────────┘          └─────────────────────┘
                      │
                      │
          ┌───────────┴───────────┐
          │                       │
          ▼                       ▼
   ┌───────────────┐      ┌────────────────┐
   │ Regolator API │      │ Merge Control  │
   └───────┬───────┘      └───────┬────────┘
           │                      │
           │                      │
  GET CRD + ResourcePolicy        │ customer PR
           │                      │
           ▼                      │
   ┌───────────────┐              │
   │ Regolator SPA │              │
   └───────┬───────┘              │
           │                      │
      policy changes              │
           │                      │
           ▼                      │
       Raise PR                   │
           │                      │
           ▼                      │
     Git feature branch           │
           │                      │
           │              deterministic RP GET
           │                      │
           │                      ▼
           │                build OPA input
           │                      │
           │                      ▼
           │                embedded generic
           │                 fieldpolicy.rego
           │                      │
           │                      ▼
           │                  violations
           │                      │
           │                      ▼
           │                  PR comment
           │
           ▼
 Merge Control required check
 calls `regolator generate`
           │
           ▼
     human approval
           │
           ▼
          merge
           │
           ▼
    Argo Workflow / CI
           │
           ▼
   `regolator generate`
           │
           ├── provider cache check
           ├── xpkg pull if needed
           ├── CRD snapshot generation
           ├── ResourcePolicy generation/validation
           └── Gatekeeper generation
```

---

# 48. Implementation Order

A practical implementation sequence:

## Phase 1 — ResourcePolicy API type

Implement:

- Go types;
- CRD generation;
- version/state model;
- rule model;
- path token model;
- conditions;
- enforcement targets.

Add table-driven unit tests.

---

## Phase 2 — Generic Rego engine

Implement:

- direct-path resolver;
- `[]` wildcard resolver using `walk`;
- `MustEqual`;
- `Forbidden`;
- `Required`;
- `AllowedValues`;
- `DeniedValues`;
- regex rules;
- version selection;
- fail-closed version behavior;
- condition evaluation;
- enforcement filtering.

Build a comprehensive input/output fixture suite before building the UI.

---

## Phase 3 — Merge Control integration

Implement:

- deterministic ResourcePolicy names;
- Kubernetes GET;
- OPA prepared query;
- input construction;
- violation conversion to current Markdown model;
- fail-closed behavior.

This proves the core architecture without a SPA.

---

## Phase 4 — `config.yaml` and xpkg CRD extraction

Implement:

- config parser;
- OCI image retrieval with IRSA/ECR;
- xpkg base-layer/package.yaml parsing;
- Group + Kind selection;
- provenance annotations;
- schema hash;
- canonical CRD persistence;
- digest cache behavior.

---

## Phase 5 — CI/CD CRD registration

Add:

```text
Argo App: regolator-crds
```

and confirm:

- generated CRDs register cleanly;
- no provider controllers are necessary;
- Regolator metadata is queryable;
- conversion-webhook checks work.

---

## Phase 6 — Regolator API normalized schema service

Implement:

- list Regolator-managed CRDs;
- parse version schemas;
- recursive schema normalization;
- tree model;
- type handling;
- description handling;
- arrays;
- maps;
- ResourcePolicy overlay.

---

## Phase 7 — SPA

Implement navigation:

```text
cloud
→ provider
→ kind
→ version
→ schema tree
```

Then policy-edit controls.

---

## Phase 8 — Gatekeeper generator

Implement:

- generic ConstraintTemplate;
- Constraint rendering;
- enforcement filtering;
- conditional rules;
- version handling;
- release output.

---

## Phase 9 — Release / promotion workflow

Implement:

- pre-merge generation gate;
- post-merge deterministic generation;
- release identity;
- lower-environment testing;
- Argo promotion;
- rollback.

---

# 49. Explicit Non-Goals

Initial Regolator should **not** attempt to:

- expose arbitrary Rego through ResourcePolicy;
- support arbitrary boolean condition expression trees;
- replace exceptional Merge Control checks;
- run Crossplane provider controllers on the CI/CD cluster;
- dynamically infer policy from provider behavior;
- automatically approve newly discovered API versions;
- generate per-resource Rego files;
- generate a ConstraintTemplate per rule;
- use provider GitHub repositories as the schema source;
- treat mutable provider source code as more authoritative than the configured package image.

---

# 50. Open Implementation Details

These do not block the architecture but should be decided during implementation.

## 50.1 OCI client

Choose whether provider xpkg retrieval uses:

- `go-containerregistry`;
- ORAS libraries;
- Crossplane package helper libraries;
- another OCI-native Go implementation.

Requirement:

> Read the exact package image referenced by config and extract `package.yaml` deterministically.

---

## 50.2 Generated artifacts in PRs

Possible approaches:

1. Regolator generates CRD/Gatekeeper files before opening the PR.
2. Merge Control generation check computes the diff and a bot commits generated output.
3. Source changes merge, and post-merge generation commits output into a dedicated generated revision.

For maximum auditability, having generated CRD diffs visible before approval is preferable.

The architecture does not depend on which mechanism is selected.

---

## 50.3 Arbitrary JSON rule values

The rough CRD uses arbitrary JSON for:

```text
value
values[]
```

The implementation should verify the final structural OpenAPI schema generated for these fields.

Potential Go representation:

```text
apiextensionsv1.JSON
```

or equivalent.

---

## 50.4 Map wildcard semantics

Array semantics use:

```text
[]
```

Map wildcard semantics are not yet required.

If eventually needed, define a distinct token rather than overloading `[]`.

Example future syntax might be:

```text
{}
```

but should not be introduced until required.

---

## 50.5 Violation formatting

Rego should preferably return structured data:

```json
{
  "path": [
    "spec",
    "forProvider",
    "lists",
    1,
    "encryption"
  ],
  "message": "...",
  "ruleType": "MustEqual"
}
```

Merge Control can format the final path:

```text
spec.forProvider.lists[1].encryption
```

This keeps presentation logic out of the policy engine.

Gatekeeper may require Rego to generate a human-readable message directly.

---

# 51. Official Technical References

These are the primary official references implementation agents should use.

## OPA / Rego

### Policy language

https://www.openpolicyagent.org/docs/policy-language

Relevant concepts:

- `package`;
- `input`;
- `if`;
- `contains`;
- `some ... in`;
- `every`;
- sets and comprehensions;
- Rego v1 semantics.

---

### Rego style guide

https://www.openpolicyagent.org/docs/style-guide

Relevant guidance:

- prefer `some ... in` for iteration;
- use `every` for "for all" semantics;
- modern Rego v1 idioms.

---

### `walk()` built-in

https://www.openpolicyagent.org/docs/policy-reference/builtins/graph

Key property:

> `walk(x, output)` recursively emits `[path, value]` tuples for nested data.

This is the basis of Regolator's `[]` wildcard path resolution.

---

### `object.get()` built-in

https://www.openpolicyagent.org/docs/policy-reference/builtins/object

Key property:

> The key argument can be an array representing a nested object/array path.

Example supported conceptually by OPA:

```rego
object.get(
    {"a": [{"b": true}]},
    ["a", 0, "b"],
    false,
)
```

This is the fast path for rules without wildcard array traversal.

---

### OPA Go integration

https://www.openpolicyagent.org/docs/integration

Relevant implementation pattern:

```text
rego.New(...)
    ↓
PrepareForEval(...)
    ↓
PreparedEvalQuery
    ↓
Eval(..., rego.EvalInput(input))
```

Prepared queries should be reused rather than recompiling policy for every resource.

---

## Crossplane

### xpkg package specification

https://github.com/crossplane/crossplane/blob/main/contributing/specifications/xpkg.md

Relevant facts:

- Crossplane packages are OCI images;
- the xpkg base layer contains root-level `package.yaml`;
- `package.yaml` is a YAML stream;
- provider package streams may contain Kubernetes CRDs.

This is the basis for Regolator provider-schema extraction.

---

## Kubernetes CRDs

### CRD documentation

https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/

Relevant topics:

- OpenAPI schemas;
- structural schemas;
- CRD versions;
- `x-kubernetes-preserve-unknown-fields`;
- conversion;
- pruning.

---

### CRD API reference

https://kubernetes.io/docs/reference/kubernetes-api/apiextensions/custom-resource-definition-v1/

Relevant topics:

- `spec.versions`;
- OpenAPI schema fields;
- conversion configuration;
- Kubernetes schema extensions.

---

## Gatekeeper

### ConstraintTemplates

https://open-policy-agent.github.io/gatekeeper/website/docs/constrainttemplates/

Relevant concepts:

- generic policy code in ConstraintTemplates;
- Constraints instantiate templates;
- Rego v1 support;
- `input.review`;
- `input.parameters`.

---

### Gatekeeper parameters

https://open-policy-agent.github.io/gatekeeper/website/docs/howto/

Relevant concept:

> Constraint `spec.parameters` is exposed to Rego as `input.parameters`.

This is why ResourcePolicy rule data can be transformed directly into generic Gatekeeper Constraint parameters.

---

## Argo CD

### Revision tracking

https://argo-cd.readthedocs.io/en/stable/user-guide/tracking_strategies/

Relevant concepts:

- branch tracking;
- tag tracking;
- immutable commit SHA pinning;
- controlled promotion and rollback.

---

# Final Architectural Statement

Regolator's fundamental abstraction is:

```text
SCHEMA
  +
POLICY DATA
  +
GENERIC POLICY ENGINE
```

Specifically:

```text
CRD
  +
ResourcePolicy
  +
fieldpolicy.rego
```

The CRD explains what users **can technically configure**.

The ResourcePolicy explains what the organization **allows them to configure**.

The generic Rego engine explains **how those policy decisions are evaluated**.

Regolator turns that model into:

- a human-friendly policy-authoring SPA;
- runtime Merge Control enforcement;
- Gatekeeper admission enforcement;
- auditable schema snapshots;
- versioned policy releases.

No generated Rego-per-resource layer is required.
