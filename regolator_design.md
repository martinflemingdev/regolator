# Regolator Design

## Status

High-level design for **Regolator**, a Rego-based policy authoring and Gatekeeper generation system for Crossplane managed resources.

This document captures the agreed design direction. It intentionally focuses on architecture and data modeling rather than implementation details.

---

# 1. System Boundaries

## Regolator

**Regolator** is the system being built.

It will be a Go/Cobra application with two primary commands:

```text
regolator api
regolator generate
```

### `regolator api`

A long-running Kubernetes workload that:

- authenticates users through the existing Dex / AD integration;
- reads CRDs from the Kubernetes API;
- reads deployed `ResourcePolicy` CRs from the Kubernetes API;
- joins schema + policy into a normalized model for the SPA;
- serves the SPA;
- creates feature branches and pull requests when users submit policy changes.

### `regolator generate`

A build-time command that:

- reads proposed or merged `ResourcePolicy` definitions;
- validates that they can be interpreted;
- validates referenced versions / paths against the available CRD schema;
- generates Gatekeeper resources;
- produces deterministic generated output.

`regolator generate` is the validation boundary as well as the generation step.

There is no separate `regolator validate` architectural concept.

---

## Merge Control

**Merge Control is a separate existing production system.**

Merge Control validates customer infrastructure pull requests.

It will be updated so that for each Kubernetes manifest in a customer PR it:

1. determines Group / Version / Kind;
2. derives the deterministic matching `ResourcePolicy` name;
3. looks up the live `ResourcePolicy` from Kubernetes;
4. selects the matching version rules;
5. combines the manifest and policy into OPA input;
6. evaluates the embedded generic Rego engine;
7. aggregates violations;
8. posts the normal Markdown PR comment / status.

Merge Control does **not** author policy.

Merge Control may also continue to contain a small number of hand-written, resource-specific checks for exceptional cases that do not fit the generic policy model.

---

# 2. ResourcePolicy Is the Central Policy Model

The main custom resource is:

```yaml
apiVersion: policy.mycompany.io/v1alpha1
kind: ResourcePolicy
```

A `ResourcePolicy` represents the organization's policy for one Kubernetes **Group + Kind**, across one or more API versions.

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
      rules:
        - path: spec.forProvider.deleteContentsOnDestroy
          type: MustEqual
          value: false

        - path: spec.forProvider.location
          type: AllowedValues
          values:
            - us-east1
            - us-east4

    - name: v1beta2
      state: Unreviewed
      rules: []

    - name: v1alpha1
      state: Denied
      reason: Alpha API versions are not approved.
      rules: []
```

The `ResourcePolicy` is **policy data**, not Rego code.

---

# 3. Deterministic ResourcePolicy Naming

There should be one `ResourcePolicy` per Group + Kind.

The name should therefore be deterministic.

For example:

```text
bigquery.gcp.m.upbound.io + Dataset
```

could become:

```text
bigquery-gcp-m-upbound-io-dataset
```

This lets Merge Control use a direct Kubernetes `GET` instead of listing all `ResourcePolicy` objects with a label selector.

Useful labels should still be present for browsing and indexing:

```yaml
metadata:
  labels:
    regolator.mycompany.io/group: bigquery.gcp.m.upbound.io
    regolator.mycompany.io/kind: Dataset
```

The exact naming normalization algorithm should be deterministic and shared by Regolator and Merge Control.

---

# 4. Fail-Closed Behavior

Merge Control already behaves fail-closed today.

That behavior should be preserved.

If Merge Control receives a resource and:

- no matching `ResourcePolicy` exists;
- the requested API version is unknown or unreviewed;
- the `ResourcePolicy` cannot be parsed;
- the generic Rego engine cannot evaluate the policy;

the resource should be denied.

The user-facing behavior should remain conceptually equivalent to:

```text
This resource is not supported at this time.
```

A missing policy must never silently mean "allow."

---

# 5. Kubernetes CRDs Are the Schema Source

Regolator does **not** need to pull Crossplane provider package images or mine `package.yaml`.

By the time a resource is ready for security review:

- the provider has already been tested internally;
- the provider is already installed;
- the corresponding CRDs already exist in Kubernetes.

Therefore the Kubernetes API is the authoritative schema source.

For each configured Group + Kind, Regolator reads the installed CRD and discovers:

- served versions;
- OpenAPI schema;
- `spec.forProvider`;
- every nested field;
- field type;
- description;
- array structure;
- map structure;
- nested object structure.

---

# 6. The Regolator API Serves the Full Schema, Not Only Decisions

This is a core design rule.

The SPA must see **every settable field under `spec.forProvider`**, whether or not a policy exists for that field.

Conceptually:

```text
CRD schema
+
ResourcePolicy rules
=
normalized SPA model
```

The CRD answers:

> What fields exist?

The ResourcePolicy answers:

> What decisions has the organization made about those fields?

Regolator joins the two.

A completely untouched resource may have 50 schema fields and zero policy rules.

The API must still return all 50 fields.

---

# 7. Normalized Schema + Policy Overlay Model

A normalized API response could conceptually look like:

```json
{
  "group": "bigquery.gcp.m.upbound.io",
  "kind": "Dataset",
  "version": "v1beta1",
  "state": "Allowed",
  "fields": [
    {
      "name": "location",
      "path": "spec.forProvider.location",
      "type": "string",
      "description": "The geographic location of the dataset.",
      "children": [],
      "rules": [
        {
          "type": "AllowedValues",
          "values": ["us-east1", "us-east4"]
        }
      ]
    },
    {
      "name": "friendlyName",
      "path": "spec.forProvider.friendlyName",
      "type": "string",
      "description": "Friendly display name.",
      "children": [],
      "rules": []
    }
  ]
}
```

The presence of:

```json
"rules": []
```

means the field exists but currently has no restrictions.

---

# 8. Preserve Schema Hierarchy in the SPA

The old Markdown representation flattened everything into rows such as:

```text
spec.forProvider.tableRef
spec.forProvider.tableRef.name
spec.forProvider.tableRef.project
```

The SPA should preserve hierarchy.

Example:

```text
Dataset / v1beta1

spec.forProvider
├── location                      string
├── project                       string
├── deleteContentsOnDestroy       boolean
├── access[]                      array<object>
│   ├── role                      string
│   ├── userByEmail               string
│   └── groupByEmail              string
└── tableRef                      object
    ├── name                      string
    └── project                   string
```

The UI can expand and collapse object and array nodes.

---

# 9. Policy Path Notation

The existing readable path convention should be preserved:

```text
spec.forProvider.access[].foo.bar[].baz
```

Examples:

```text
spec.forProvider.location
spec.forProvider.access[].role
spec.forProvider.foo[].bar[].baz
```

This notation means that the rule applies to every matching value reached through the array traversal.

Example:

```text
spec.forProvider.ipRules[].ipRange
```

means:

> Evaluate `ipRange` for every element in `ipRules`.

The generic Rego engine therefore needs a reusable path resolver that understands:

```text
foo.bar
foo[].bar
foo[].bar[].baz
```

A simple `object.get()` is not sufficient for paths containing `[]`.

This path resolver should be implemented and tested once in the generic Rego engine.

The exact internal representation may later be changed to a structured path AST if useful, but the human-readable `[]` notation should remain supported.

---

# 10. ResourcePolicy Must Not Duplicate the CRD Schema

The `ResourcePolicy` should persist only policy decisions.

It should **not** contain a rule entry for every field just because the field exists.

An initial generated policy should remain small:

```yaml
spec:
  target:
    group: bigquery.gcp.m.upbound.io
    kind: Dataset

  unknownVersionPolicy: Deny

  versions:
    - name: v1beta1
      state: Unreviewed
      rules: []

    - name: v1beta2
      state: Unreviewed
      rules: []
```

All field metadata comes from the CRD dynamically.

This avoids creating a stale duplicate schema inside `ResourcePolicy`.

---

# 11. Version-Specific Policy

Each API version has its own policy state and rules.

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
    reason: Alpha versions are not approved.
    rules: []
```

The version states are:

```text
Unreviewed
Allowed
Denied
```

These states are intentionally distinct from field rules.

For example:

```yaml
- name: v1beta1
  state: Allowed
  rules: []
```

means:

> Security reviewed this version and allows it with no additional field-level restrictions.

While:

```yaml
- name: v1beta1
  state: Unreviewed
  rules: []
```

means:

> Nobody has approved this API version yet.

Unknown versions should default to deny:

```yaml
unknownVersionPolicy: Deny
```

---

# 12. Rule Vocabulary

Initial rule types may include:

```text
Forbidden
Required
MustEqual
AllowedValues
DeniedValues
AllowedRegex
DeniedRegex
```

These are not built-in OPA keywords.

They are Regolator's policy vocabulary.

The generic Rego engine interprets them.

Example:

```yaml
- path: spec.forProvider.location
  type: AllowedValues
  values:
    - us-east1
    - us-east4
```

The policy language should remain intentionally small.

If a requirement does not fit the approved vocabulary, either:

- add a clearly defined new operator; or
- implement a specific Merge Control check outside the generic policy engine.

Do not expose arbitrary Rego through `ResourcePolicy`.

---

# 13. Conditional Rules

Production and non-production policy should be modeled as **conditional rules**, not separate policy files.

Example:

```yaml
- path: spec.forProvider.location
  type: AllowedValues
  values:
    - us-east4

  when:
    path: spec.forProvider.project
    type: StartsWith
    value: prod-
```

This means:

> Enforce this location restriction only when `spec.forProvider.project` starts with `prod-`.

Base policy still runs normally.

So production behavior becomes:

```text
base rules
+
conditional rules whose `when` evaluates true
```

This replaces the old pattern where Merge Control looked up a second policy named something like:

```text
prod-bigquery-dataset
```

The environment condition now lives in policy rather than hidden Merge Control logic.

---

# 14. Condition Vocabulary

Initial condition operators may include:

```text
Equals
NotEquals
StartsWith
EndsWith
MatchesRegex
Exists
NotExists
```

The initial implementation should keep conditions deliberately simple.

A rule should support at most one `when` condition initially.

Do not introduce nested boolean trees (`AND`, `OR`, `NOT`) until there is a concrete use case.

Example:

```yaml
when:
  path: spec.forProvider.project
  type: StartsWith
  value: prod-
```

The same conditional semantics can be used by both:

- Merge Control;
- Gatekeeper.

---

# 15. Enforcement Targets

The old `SkipTemplate` behavior should be preserved semantically, but generalized.

Some policies are safe to enforce in Merge Control but not in Gatekeeper.

Example:

- customers are forbidden from setting `tableRef`;
- customers must use `tableSelector`;
- Crossplane later late-initializes `tableRef`;
- therefore Merge Control should deny `tableRef`;
- Gatekeeper must not deny it after Crossplane writes it.

Model this explicitly:

```yaml
- path: spec.forProvider.tableRef
  type: Forbidden

  enforcement:
    mergeControl: true
    gatekeeper: false

  reason: >-
    Customers must use tableSelector. tableRef is late initialized
    by Crossplane and cannot safely be enforced by Gatekeeper.
```

Default enforcement should be:

```yaml
enforcement:
  mergeControl: true
  gatekeeper: true
```

The UI can expose this as an advanced setting.

This replaces the old `SkipTemplate` implementation detail with an explicit statement of enforcement intent.

---

# 16. SPA Navigation

The SPA should allow hierarchical browsing by platform API.

Conceptually:

```text
GCP
├── BigQuery
│   ├── Dataset
│   │   ├── v1beta1
│   │   └── v1beta2
│   └── Table
│       ├── v1beta1
│       └── v1beta2
├── Storage
│   └── Bucket
└── Pub/Sub
    └── Topic
```

After selecting a version, the user sees the complete normalized `spec.forProvider` tree.

Each field shows:

- name;
- type;
- description;
- nested fields;
- existing rules;
- conditional rules;
- enforcement targets.

---

# 17. Policy Authoring Flow

The SPA is the authoring interface.

Security does not need to write YAML or Rego.

A user:

1. selects a provider / group;
2. selects a Kind;
3. selects a version;
4. expands the `spec.forProvider` schema;
5. adds or edits rules;
6. selects **Raise PR**.

The UI may support bulk actions such as:

```text
Mark version as reviewed
Allow all unrestricted fields
Copy rules from v1beta1 to v1beta2
```

However bulk UI actions should not generate unnecessary rules.

For example, "Allow All" can simply result in:

```yaml
state: Allowed
rules: []
```

if no field restrictions are needed.

---

# 18. Git Is Change Control, Not Runtime Policy Lookup

The runtime systems should not treat Git as the policy database.

Instead:

```text
Git
  ↓
Argo CD
  ↓
ResourcePolicy CRs in Kubernetes
```

The Kubernetes API is the runtime lookup layer.

Git is used when policy changes are authored and reviewed.

---

# 19. Raising ResourcePolicy Pull Requests

When a user clicks **Raise PR**:

```text
Regolator SPA
      ↓
Regolator API
      ↓
fetch latest policy repo main
      ↓
create feature branch
      ↓
update ResourcePolicy YAML
      ↓
commit
      ↓
push
      ↓
open PR
```

Required reviewers may include:

```text
Platform Engineering
Security
```

Normal Git merge conflict behavior is sufficient.

Policy changes are infrequent enough that no special collaborative editing system is required.

---

# 20. PR-Time Generation Check

A `ResourcePolicy` change must prove that it can be generated **before merge**.

Merge Control guards the policy repository.

For a proposed policy PR:

```text
ResourcePolicy PR
      ↓
Merge Control
      ↓
regolator generate
      │
      ├── parse ResourcePolicy
      ├── validate rule structure
      ├── validate version references
      ├── validate field paths against CRD schema
      ├── validate condition syntax
      ├── validate enforcement targets
      ├── render Gatekeeper output
      └── fail if generation is impossible
      ↓
PR check passes / fails
```

If `regolator generate` fails, Merge Control blocks the PR.

This ensures an invalid policy cannot be merged.

---

# 21. Initial ResourcePolicy Generation

For configured Group + Kind pairs, Regolator can create a skeleton `ResourcePolicy`.

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
      state: Unreviewed
      rules: []

    - name: v1beta2
      state: Unreviewed
      rules: []
```

If the CRD later gains a new version:

```text
v1beta3
```

Regolator should surface that version as:

```text
Unreviewed
```

rather than silently treating it as approved.

---

# 22. Generic Embedded Rego Engine

Regolator and Merge Control should use a generic policy engine instead of generated Rego per resource.

Conceptually:

```text
fieldpolicy.rego
```

The engine understands:

```text
Forbidden
Required
MustEqual
AllowedValues
DeniedValues
AllowedRegex
DeniedRegex
```

and conditional operators such as:

```text
StartsWith
Equals
MatchesRegex
```

The engine also understands path traversal including array notation:

```text
foo[].bar[].baz
```

Adding another Dataset rule changes a `ResourcePolicy`.

Adding a brand-new rule type changes `fieldpolicy.rego`.

---

# 23. Merge Control Evaluation Flow

For a customer PR containing:

```yaml
apiVersion: bigquery.gcp.m.upbound.io/v1beta1
kind: Dataset
spec:
  forProvider:
    project: prod-example
    location: us-east1
```

Merge Control performs:

```text
parse manifest
      ↓
determine Group / Version / Kind
      ↓
derive deterministic ResourcePolicy name
      ↓
GET ResourcePolicy from Kubernetes
      ↓
select matching version
      ↓
construct OPA input
      ↓
evaluate embedded Rego
      ↓
aggregate violations
      ↓
post PR status / Markdown comment
```

OPA input may look approximately like:

```json
{
  "review": {
    "object": {
      "...": "customer manifest"
    }
  },
  "resourcePolicy": {
    "...": "matching ResourcePolicy spec"
  }
}
```

No policy bundle download is required.

No Artifactory / ECR policy archive is required.

No per-resource generated `.rego` files are required.

---

# 24. Gatekeeper Generation

`regolator generate` converts `ResourcePolicy` objects into Gatekeeper resources.

The intended model is:

```text
small fixed number of ConstraintTemplates
+
one Constraint per ResourcePolicy
```

The generic ConstraintTemplate contains the reusable Rego policy engine.

The generated Constraint contains the ResourcePolicy rules as parameters.

Rules with:

```yaml
enforcement:
  gatekeeper: false
```

are omitted from Gatekeeper generation but remain available to Merge Control.

---

# 25. Two Argo Applications

Use separate Argo applications:

```text
Argo App: resource-policies
Argo App: gatekeeper-policy
```

This matches the existing operational model.

They do not need to be the same Argo Application.

However, they should be released together from the same logical policy revision.

---

# 26. Common Release Identity

A policy release should identify both:

```text
ResourcePolicy CRs
+
generated Gatekeeper resources
```

For example:

```text
policy-release-v42
```

The important invariant is:

> Merge Control's live `ResourcePolicy` state and Gatekeeper's live constraints should represent the same approved policy revision.

This prevents:

```text
Merge Control enforcing new policy
while
Gatekeeper still enforces old policy
```

---

# 27. Release Generation

After a ResourcePolicy PR merges:

```text
merge
  ↓
Argo Workflow / CI
  ↓
regolator generate
```

`regolator generate` should operate against the exact merged Git commit.

It should not wait for Argo to sync the ResourcePolicy into Kubernetes and then read it back.

This keeps generation deterministic.

The same generation logic should have already run successfully during the PR check.

---

# 28. Lower-Environment Testing

After merge and generation, the resulting policy release should be deployed to lower environments.

Testing should include:

- examples expected to violate;
- examples expected to pass;
- late-initialization behavior;
- conditional production-policy behavior;
- Gatekeeper admission behavior;
- Merge Control behavior.

Some of this can also be automated using KinD.

This is integration / behavioral testing, not first-time policy validation.

---

# 29. Promotion and Rollback

Generated policy artifacts should be versioned in Git.

Argo may pin releases using a tag or immutable commit SHA.

Example:

```text
DEV  → policy-release-v43
TEST → policy-release-v42
PROD → policy-release-v41
```

Promotion means moving an environment to a newer approved release.

Rollback means moving it back to a previously known-good revision.

This retains the release / rollback behavior that previously came from Helm charts in OCI images.

---

# 30. Schema Drift Detection

Even when an API version name does not change, provider upgrades may alter schema details.

Regolator should consider computing a normalized digest of:

```text
spec.forProvider
```

for each Group + Kind + Version.

Example:

```text
schemaHash: sha256:...
```

This allows Regolator to detect:

> Dataset v1beta2 has changed since its last security review.

The exact schema-hash persistence strategy can be decided later, but the normalized API model should allow for it.

---

# 31. Overall Architecture

```text
                         INSTALLED PLATFORM
                    ┌─────────────────────────┐
                    │     Kubernetes API      │
                    │                         │
                    │ CRDs                    │
                    │ live ResourcePolicies   │
                    └────────────┬────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
                    ▼                         ▼
          ┌──────────────────┐       ┌─────────────────┐
          │  Regolator API   │       │  Merge Control  │
          └────────┬─────────┘       └────────┬────────┘
                   │                          │
        CRD schema + live                     │
        ResourcePolicy                        │
                   │                          │
                   ▼                          │
          ┌──────────────────┐                │
          │  Regolator SPA   │                │
          └────────┬─────────┘                │
                   │                          │
              policy edits                    │
                   │                          │
                   ▼                          │
               Raise PR                       │
                   │                          │
                   ▼                          │
        ┌──────────────────────┐              │
        │ Policy Authoring Git │              │
        │                      │              │
        │ ResourcePolicy YAML  │              │
        └──────────┬───────────┘              │
                   │                          │
            review + approval                 │
                   │                          │
             Merge Control                    │
         runs `regolator generate`            │
          as a required PR check              │
                   │                          │
                   ▼                          │
                 merge                        │
                   │                          │
                   ▼                          │
         ┌─────────────────────┐              │
         │ Argo Workflow / CI  │              │
         │                     │              │
         │ regolator generate  │              │
         └──────────┬──────────┘              │
                    │                         │
                    ▼                         │
             generated release                │
                    │                         │
          ┌─────────┴─────────┐               │
          │                   │               │
          ▼                   ▼               │
 ResourcePolicy CRs   Gatekeeper resources    │
          │                   │               │
          ▼                   ▼               │
   Argo App: RP       Argo App: Gatekeeper    │
          │                   │               │
          └─────────┬─────────┘               │
                    │                         │
                    ▼                         │
              Kubernetes API                  │
                    │                         │
                    ├── ResourcePolicies ──────┘
                    │
                    └── Gatekeeper Constraints
```

Customer infrastructure validation is separate:

```text
CUSTOMER INFRASTRUCTURE REPOSITORY

customer raises PR
        │
        ▼
 Merge Control
        │
        ├── parse manifest
        ├── determine Group / Version / Kind
        ├── GET live ResourcePolicy
        ├── select matching version
        ├── evaluate generic embedded Rego
        ├── run any exceptional hand-written checks
        └── aggregate violations
        │
        ▼
PR comment / status
```

---

# 32. Component Ownership Summary

| Component | Responsibility |
|---|---|
| CRD | Defines what fields exist and their schema |
| ResourcePolicy | Stores organizational policy decisions |
| Regolator API | Joins CRD schema + ResourcePolicy and serves the SPA |
| Regolator SPA | Human-friendly authoring interface |
| Git PR | Review and approval mechanism |
| `regolator generate` | Validates generatability and produces Gatekeeper output |
| Merge Control | Validates customer infrastructure PRs and guards policy PRs |
| Generic Rego engine | Interprets the ResourcePolicy rule vocabulary |
| Gatekeeper | Enforces policy at Kubernetes admission/runtime |
| Argo CD | Promotes and rolls back approved policy releases |

---

# 33. Rough ResourcePolicy CRD

The following is intentionally a rough draft.

It captures the current data model and should be tightened as implementation begins.

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
                              type: string
                              minLength: 1
                              description: >-
                                Field path under the Kubernetes resource.
                                Array traversal may be expressed using [].
                                Example:
                                spec.forProvider.access[].role

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

            status:
              type: object
              x-kubernetes-preserve-unknown-fields: true
```

---

# 34. Example ResourcePolicy

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
        - path: spec.forProvider.project
          type: Required

        - path: spec.forProvider.deleteContentsOnDestroy
          type: MustEqual
          value: false

        - path: spec.forProvider.location
          type: AllowedValues
          values:
            - us-east1
            - us-east4

        - path: spec.forProvider.location
          type: AllowedValues
          values:
            - us-east4

          when:
            path: spec.forProvider.project
            type: StartsWith
            value: prod-

        - path: spec.forProvider.tableRef
          type: Forbidden

          enforcement:
            mergeControl: true
            gatekeeper: false

          reason: >-
            Customers must use tableSelector. tableRef is late initialized
            by Crossplane and cannot safely be enforced by Gatekeeper.

    - name: v1beta2
      state: Unreviewed
      rules: []

    - name: v1alpha1
      state: Denied
      reason: Alpha API versions are not approved.
      rules: []
```

---

# 35. Design Principles

The design should remain guided by the following principles:

1. **ResourcePolicy is the policy source model.**
2. **CRDs are the schema source.**
3. **The SPA renders schema + policy overlay, never policy alone.**
4. **Merge Control and Gatekeeper consume the same policy intent.**
5. **Runtime policy lookup comes from Kubernetes, not Git.**
6. **Git is for review, approval, release history, and promotion.**
7. **Policy changes fail closed.**
8. **The generic policy language stays intentionally small.**
9. **Exceptional logic may remain as hand-written Merge Control checks.**
10. **Array traversal semantics must be formalized and heavily tested.**
11. **Late-initialized fields may use enforcement targeting instead of weakening Merge Control checks.**
12. **Production-specific restrictions should be conditional rules, not separate policy bundles.**
13. **Policy PRs must successfully run `regolator generate` before merge.**
14. **Lower-environment testing happens before production promotion.**
15. **ResourcePolicy and Gatekeeper releases must correspond to the same logical policy revision.**
