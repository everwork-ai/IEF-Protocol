# IEF Protocol v0

> IEF Protocol Layer — shared contracts for AI-worker communication and collaboration.

## 1. Protocol Positioning

IEF-Protocol is the **shared contract layer** of the IEF ecosystem. It defines the durable, versioned object schemas and communication contracts that all IEF layers use to exchange work, status, artifacts, and context.

Protocol is the **lingua franca** of IEF: every layer speaks it, no layer owns it alone.

### 1.1 What Protocol Owns

| Domain | Description |
|---|---|
| Object schemas | JSON Schema definitions for all shared objects |
| Field contracts | Required fields, types, formats, and validation rules |
| Object relationships | How objects reference each other (e.g., TaskEnvelope → ContextRef) |
| Extension strategies | How fields and types can be extended without breaking compatibility |
| Versioning rules | How protocol versions are declared and evolve |

### 1.2 What Protocol Does NOT Own

| Domain | Owner |
|---|---|
| Task lifecycle / state machine | IEF-Operations |
| Run lifecycle / run management | IEF-Operations |
| Governance rules and profiles | IEF-Governance |
| Runner implementations | IEF-Runners |
| Adapter implementations | IEF-Adapters |
| Knowledge storage | IEF-Knowledge |

### 1.3 Boundary Rules

1. Protocol defines **objects and communication contracts only**.
2. Protocol does **not** define task lifecycle — Operations owns state machines.
3. Protocol does **not** define run lifecycle — Operations owns run management.
4. Protocol does **not** define governance rules — Governance owns policy.
5. Protocol does **not** implement runners, adapters, or knowledge storage.
6. Protocol may define a `status` field, but **status values are constrained by Operations** specification, not defined here.
7. Protocol may define a `governance_profile` field, but **profile definitions belong to Governance**.

---

## 2. The Five Core Objects

### 2.1 AgentCard

**Purpose:** Machine-readable declaration of an AI agent, runner, or worker's capabilities.

**Schema:** [`schemas/agent-card.schema.json`](../schemas/agent-card.schema.json)

| Field | Type | Required | Description |
|---|---|---|---|
| `agent_id` | string | yes | Globally unique agent identifier |
| `name` | string | yes | Human-readable name |
| `type` | string | yes | Agent type: `worker`, `runner`, `service`, `adapter`, `coordinator` (extensible) |
| `capabilities` | array | yes | List of capability objects (min 1 item) |
| `runner_type` | string | yes | Runner type: `claude-code`, `codex`, `openclaw`, `hermes`, `generic`, or `custom:*` |
| `version` | string | yes | Agent declaration version (semver) |
| `description` | string | yes | Human-readable role and purpose |
| `supported_protocol_version` | string (const) | yes | Exact IEF Protocol version this AgentCard conforms to — pinned to `"0.1.0"` for the v0.1.0 schema |

**Capabilities structure:**

Each capability is an object with:
- `name` (required): e.g., `code-generation`, `code-review`, `testing`
- `description` (optional): what the capability provides
- `parameters` (optional): key-value parameters (languages, frameworks, tools)

**Runner type extension:**

Core values: `claude-code`, `codex`, `openclaw`, `hermes`, `generic`.
Custom runners use the `custom:` prefix (e.g., `custom:my-runner`).

**How used by Runners / Adapters:**
- Runners use `runner_type` and `capabilities` to determine if they can handle a task
- Adapters use `AgentCard` to map host-surface capabilities to IEF worker selection
- Operations uses `agent_id` and `type` for task assignment

### 2.2 TaskEnvelope

**Purpose:** A governed, dispatchable, traceable task exchange format.

**Schema:** [`schemas/task-envelope.schema.json`](../schemas/task-envelope.schema.json)

| Field | Type | Required | Description |
|---|---|---|---|
| `task_id` | string | yes | Globally unique task identifier |
| `title` | string | yes | Human-readable task title |
| `description` | string | yes | Detailed task description |
| `assignee` | string | yes | Agent or runner assigned to this task |
| `governance_profile` | string | yes | Reference to a Governance profile (defined in IEF-Governance#2) |
| `status` | string | yes | Current task status (values constrained by IEF-Operations#2) |
| `protocol_version` | string (const) | yes | Exact IEF Protocol version this TaskEnvelope conforms to — pinned to `"0.1.0"` for the v0.1.0 schema |
| `created_at` | string (date-time) | yes | When the task envelope was created |
| `source` | string | yes | Origin of the task |
| `priority` | string | yes | Task priority: `critical`, `high`, `medium`, `low` |
| `context_refs` | array of ContextRef | no | Knowledge and context references |
| `artifact_refs` | array of ArtifactRef | no | References to task artifacts |

**Key design decisions:**
- TaskEnvelope is the **exchange format** for task objects, not the Operations state machine itself
- `status` field can reference Operations status values, but Protocol does not define the complete state machine
- `governance_profile` must reference a profile defined by IEF-Governance
- `context_refs` uses the ContextRef schema
- `artifact_refs` uses the ArtifactRef schema

### 2.3 RunEvent

**Purpose:** A runtime event emitted during task execution, used in the Operations run ledger.

**Schema:** [`schemas/run-event.schema.json`](../schemas/run-event.schema.json)

| Field | Type | Required | Description |
|---|---|---|---|
| `event_id` | string | yes | Globally unique event identifier |
| `run_id` | string | yes | Run this event belongs to |
| `task_id` | string | yes | Task this event relates to |
| `event_type` | string | yes | Event type (extensible) |
| `timestamp` | string (date-time) | yes | When the event occurred |
| `agent_id` | string | yes | Agent or runner that emitted the event |
| `payload` | object | yes | Structured event payload (content depends on event_type) |
| `severity` | string | yes | Severity: `info`, `warning`, `error`, `critical` |
| `sequence` | integer | yes | Monotonically increasing sequence for append-only ledger ordering |

**Core event types:** `started`, `progress`, `blocked`, `error`, `completed`, `artifact_produced`, `context_retrieved`, `approval_requested`, `handoff`

**Key design decisions:**
- `event_type` is extensible — consumers must ignore unknown event types
- `payload` is a structured object whose content depends on `event_type`
- `sequence` enables append-only ledger ordering within a run

### 2.4 ArtifactRef

**Purpose:** A reference to a task artifact (tangible output), without carrying the artifact content itself.

**Schema:** [`schemas/artifact-ref.schema.json`](../schemas/artifact-ref.schema.json)

| Field | Type | Required | Description |
|---|---|---|---|
| `artifact_id` | string | yes | Globally unique artifact identifier |
| `task_id` | string | yes | Task that produced this artifact |
| `run_id` | string | yes | Run that produced this artifact |
| `type` | string | yes | Artifact type (extensible) |
| `uri` | string (uri + pattern) | yes | Absolute URI pointing to the artifact |
| `hash` | string | no | Content hash for verification (recommended) |
| `created_at` | string (date-time) | yes | When the artifact was created |
| `produced_by` | string | yes | Agent or runner that produced this artifact |
| `description` | string | no | Human-readable description |

**Core artifact types:** `pr`, `commit`, `document`, `log`, `report`, `schema`, `test_result`, `review`

**URI policy:** The `uri` field must be a valid absolute URI. Both `format: uri` and `pattern` are enforced:
- `format: uri` — declarative URI syntax annotation
- `pattern: ^[a-zA-Z][a-zA-Z0-9+.-]*:` — explicit absolute-URI assertion that guarantees a scheme component

Accepted URI schemes:
- `https://` — GitHub URLs, API endpoints
- `file:///` — Local or repository-rooted file references
- `urn:ief:artifact:...` — IEF-specific URN identifiers

Relative paths (e.g., `docs/spec.md`) are **invalid** and will be rejected by the `pattern` constraint regardless of whether the validator enforces `format` assertions. Convert to `file:///` or `https://` or `urn:ief:*` scheme.

**Key design decisions:**
- URI must be a resolvable absolute URI — this ensures artifacts are unambiguously addressable across environments
- Dual validation (`format` + `pattern`) ensures absolute-URI enforcement even when validators treat `format` as annotation-only
- `hash` is optional but recommended for verifiable artifacts
- `type` is extensible — consumers must handle unknown types gracefully

### 2.5 ContextRef

**Purpose:** A reference to knowledge, context, historical decisions, run summaries, or external materials.

**Schema:** [`schemas/context-ref.schema.json`](../schemas/context-ref.schema.json)

| Field | Type | Required | Description |
|---|---|---|---|
| `context_id` | string | yes | Globally unique context reference identifier |
| `source_type` | string | yes | Type of source (extensible) |
| `source_id` | string | yes | Identifier within the source domain |
| `uri` | string (uri + pattern) | yes | Absolute URI pointing to the context source |
| `relevance_score` | number | yes | Relevance score (0.0–1.0) |
| `retrieved_at` | string (date-time) | yes | When this context was retrieved |
| `summary` | string | no | Brief summary for quick scanning |

**Core source types:** `knowledge_entry`, `decision_record`, `run_summary`, `external_document`, `playbook`, `pattern`

**URI policy:** The `uri` field must be a valid absolute URI. Both `format: uri` and `pattern` are enforced:
- `format: uri` — declarative URI syntax annotation
- `pattern: ^[a-zA-Z][a-zA-Z0-9+.-]*:` — explicit absolute-URI assertion that guarantees a scheme component

Accepted URI schemes:
- `https://` — GitHub URLs, API endpoints, documentation sites
- `file:///` — Local or repository-rooted file references
- `urn:ief:context:...` — IEF-specific URN identifiers

Relative paths (e.g., `docs/adr/001.md`) are **invalid** and will be rejected by the `pattern` constraint regardless of whether the validator enforces `format` assertions. Convert to `file:///` or `https://` or `urn:ief:*` scheme.

**Key design decisions:**
- ContextRef is NOT the Knowledge storage itself
- Knowledge (IEF-Knowledge) is responsible for producing and managing the content ContextRef points to
- Protocol only defines the reference format
- URI must be a resolvable absolute URI — this ensures context is unambiguously addressable
- Dual validation (`format` + `pattern`) ensures absolute-URI enforcement even when validators treat `format` as annotation-only

---

## 3. Object Relationships

```text
AgentCard ────────────────────────────────────────────────────
  │ declared by agents, used for selection and routing       │
  │                                                          │
TaskEnvelope ◄───────────────────────────────────────────── │
  │ contains: context_refs[] ──► ContextRef                  │
  │ contains: artifact_refs[] ──► ArtifactRef                │
  │                                                          │
RunEvent ◄───────────────────────────────────────────────── │
  │ references: task_id (→ TaskEnvelope)                     │
  │ references: agent_id (→ AgentCard)                       │
  │                                                          │
ArtifactRef ◄────────────────────────────────────────────── │
  │ references: task_id (→ TaskEnvelope)                     │
  │ references: run_id (→ RunEvent.run_id)                   │
  │                                                          │
ContextRef ◄─────────────────────────────────────────────── │
  │ referenced by: TaskEnvelope.context_refs[]               │
  │ managed by: IEF-Knowledge                                │
```

**Key relationships:**
- `TaskEnvelope.context_refs` → array of `ContextRef`
- `TaskEnvelope.artifact_refs` → array of `ArtifactRef`
- `RunEvent.task_id` → `TaskEnvelope.task_id`
- `RunEvent.agent_id` → `AgentCard.agent_id`
- `ArtifactRef.task_id` → `TaskEnvelope.task_id`
- `ArtifactRef.run_id` → `RunEvent.run_id`

---

## 4. Minimal Examples

### 4.1 AgentCard

```json
{
  "agent_id": "codex-worker-001",
  "name": "Codex Code Worker",
  "type": "worker",
  "capabilities": [
    {
      "name": "code-generation",
      "description": "Generates code from specifications",
      "parameters": {
        "languages": ["typescript", "python"],
        "frameworks": ["react", "fastapi"]
      }
    },
    {
      "name": "testing",
      "description": "Writes and runs tests"
    }
  ],
  "runner_type": "codex",
  "version": "0.1.0",
  "description": "Primary code generation and testing worker",
  "supported_protocol_version": "0.1.0"
}
```

### 4.2 TaskEnvelope

```json
{
  "task_id": "task-2026-001",
  "title": "Define authentication schema",
  "description": "Create JSON schema for user authentication protocol objects",
  "assignee": "codex-worker-001",
  "governance_profile": "contract-critical",
  "status": "assigned",
  "protocol_version": "0.1.0",
  "created_at": "2026-04-28T00:00:00Z",
  "source": "program",
  "priority": "high",
  "context_refs": [],
  "artifact_refs": []
}
```

### 4.3 RunEvent

```json
{
  "event_id": "evt-2026-001-001",
  "run_id": "run-2026-001",
  "task_id": "task-2026-001",
  "event_type": "started",
  "timestamp": "2026-04-28T01:00:00Z",
  "agent_id": "codex-worker-001",
  "payload": {
    "message": "Execution started for task-2026-001"
  },
  "severity": "info",
  "sequence": 0
}
```

### 4.4 ArtifactRef

```json
{
  "artifact_id": "art-2026-001",
  "task_id": "task-2026-001",
  "run_id": "run-2026-001",
  "type": "pr",
  "uri": "https://github.com/everwork-ai/IEF-Protocol/pull/3",
  "hash": "sha256:a1b2c3d4e5f6...",
  "created_at": "2026-04-28T02:00:00Z",
  "produced_by": "codex-worker-001",
  "description": "PR defining authentication schema"
}
```

### 4.5 ContextRef

```json
{
  "context_id": "ctx-2026-001",
  "source_type": "decision_record",
  "source_id": "adr-001",
  "uri": "https://github.com/everwork-ai/IEF-Program/blob/main/docs/adr/001-json-schema-contracts.md",
  "relevance_score": 0.92,
  "retrieved_at": "2026-04-28T00:30:00Z",
  "summary": "Decision to use JSON Schema draft 2020-12 for all IEF protocol objects"
}
```

---

## 5. How Other Layers Use Protocol Objects

### 5.1 Operations (IEF-Operations#2)

Operations **must** reference Protocol objects:
- **TaskEnvelope**: Operations defines the task lifecycle and state machine, but uses `TaskEnvelope` as the exchange format
- **RunEvent**: Operations manages the run ledger using `RunEvent` objects
- **ArtifactRef**: Operations tracks artifacts produced during runs
- **ContextRef**: Operations passes context references during task assignment

Operations **must not** redefine these objects. Operations extends behavior (lifecycle, transitions, queues) but uses Protocol schemas as the object contract.

### 5.2 Governance (IEF-Governance#2)

Governance constrains Protocol:
- **Contract-Critical** profile (defined in IEF-Governance#2) governs this PR's merge criteria
- `TaskEnvelope.governance_profile` references a profile name defined by Governance
- Governance does not redefine object schemas — it applies rules to them

### 5.3 Runners (IEF-Runners)

Runners consume Protocol objects:
- Use `AgentCard` to declare their capabilities
- Receive `TaskEnvelope` as work assignment
- Emit `RunEvent` during execution
- Produce `ArtifactRef` for outputs
- Can extend `AgentCard.capabilities` and `runner_type` using the defined extension strategies

### 5.4 Knowledge (IEF-Knowledge)

Knowledge produces and manages `ContextRef` targets:
- Knowledge is the source of truth for content that `ContextRef` points to
- Knowledge does not redefine `ContextRef` schema — it creates instances
- Knowledge may produce `RunEvent` summaries that become `ContextRef` sources

### 5.5 Adapters (IEF-Adapters)

Adapters map between external systems and Protocol objects:
- Use `AgentCard` for host-surface capability mapping
- Translate external requests into `TaskEnvelope` objects
- Map external events to `RunEvent` objects

---

## 6. Version Strategy

### 6.1 Version Format

Protocol version follows [Semantic Versioning](https://semver.org/): `MAJOR.MINOR.PATCH`

- **MAJOR**: Breaking changes to existing schemas (field removal, type change)
- **MINOR**: Additive changes (new optional fields, new event types, new enum values)
- **PATCH**: Clarifications, description fixes, no schema changes

### 6.2 Version Declaration

Every `TaskEnvelope` must include `protocol_version`. `AgentCard` must declare `supported_protocol_version`.

Both fields are pinned using `const` to the exact protocol version the schema implements. For the v0.1.0 schemas:
- `TaskEnvelope.protocol_version` must be `"0.1.0"`
- `AgentCard.supported_protocol_version` must be `"0.1.0"`

A payload declaring `protocol_version: "2.0.0"` will **not** validate against the v0.1.0 schema.

### 6.3 Version–Schema Agreement

Versioned schema `$id` and declared protocol version **must agree**:

1. A v0.1.0 `TaskEnvelope` must declare `protocol_version: "0.1.0"` — the `const` constraint enforces this.
2. A v0.1.0 `AgentCard` must declare `supported_protocol_version: "0.1.0"` — the `const` constraint enforces this.
3. A payload declaring a different protocol version **must not** validate against a v0.1.0 schema.
4. Future protocol versions **must** publish new versioned schema `$id` paths and update `const` values accordingly.
5. If multi-version agent support is needed in the future, it should be modeled explicitly (e.g., a `supported_protocol_versions` array), **not** by loosening the `const` constraint in a versioned schema.

### 6.4 Compatibility Rules

1. Consumers must ignore unknown fields (`additionalProperties: true` enables forward compatibility)
2. New optional fields are minor-version changes
3. Removing or renaming a required field is a major-version change
4. New `event_type`, `type`, or `source_type` values are minor-version changes
5. Consumers must handle unknown enum values gracefully (extensibility principle)

### 6.5 Current Version

`0.1.0` — Initial draft. All v0.x versions may have breaking changes between minor versions.

---

## 7. Extension Strategy

### 7.1 Field Extension

All schemas use `additionalProperties: true`, allowing layers to add custom fields. Custom fields should use a namespace prefix to avoid collisions:

```json
{
  "task_id": "task-001",
  "x_operations_retry_count": 3
}
```

### 7.2 Enum Extension

Extensible enums (`event_type`, `type`, `source_type`, `runner_type`) follow these rules:
- Core values are defined in this document
- Unknown values must be handled gracefully by consumers
- Custom `runner_type` values use the `custom:` prefix

---

## 8. Schema Resolution Strategy

### 8.1 Schema `$id` — Immutable and Versioned

Every schema file defines an **immutable, versioned** `$id` under the `https://everwork-ai.github.io/ief-protocol/schemas/` namespace. The version is embedded in the URI path:

| Schema | `$id` |
|---|---|
| AgentCard | `https://everwork-ai.github.io/ief-protocol/schemas/v0.1.0/agent-card.schema.json` |
| TaskEnvelope | `https://everwork-ai.github.io/ief-protocol/schemas/v0.1.0/task-envelope.schema.json` |
| RunEvent | `https://everwork-ai.github.io/ief-protocol/schemas/v0.1.0/run-event.schema.json` |
| ArtifactRef | `https://everwork-ai.github.io/ief-protocol/schemas/v0.1.0/artifact-ref.schema.json` |
| ContextRef | `https://everwork-ai.github.io/ief-protocol/schemas/v0.1.0/context-ref.schema.json` |

**Immutability guarantee:** Once a versioned `$id` is published, its schema content **must not change**. A v0.1.0 schema `$id` will never be overwritten by a v0.2.0 release.

**Breaking change rule:** When a schema introduces breaking changes (field removal, type change, new required fields), it **must** use a new `$id` path with an incremented version (e.g., `v0.2.0`). This prevents validators from applying a newer contract to older payloads.

**Consumer guidance:** Consumers **should pin to exact `$id` / `protocol_version` values** to ensure deterministic validation. When upgrading protocol versions, both producer and consumer must align on the new `$id`.

### 8.2 Cross-File `$ref` Resolution

`TaskEnvelope` uses `$ref` to reference `ContextRef` and `ArtifactRef` schemas. These references use the versioned `$id` URIs:

```json
{
  "$ref": "https://everwork-ai.github.io/ief-protocol/schemas/v0.1.0/context-ref.schema.json"
}
```

This ensures deterministic resolution regardless of how schemas are loaded (file system, HTTP, or in-memory compilation). Validators that compile schemas from in-memory objects must register each schema by its `$id` so that `$ref` values resolve correctly.

When a new protocol version is released (e.g., v0.2.0), TaskEnvelope's `$ref` values are updated to point to the new versioned URIs, while the v0.1.0 schemas remain available at their original `$id` paths.

### 8.3 URI Policy for Data Fields

The `uri` fields in `ArtifactRef` and `ContextRef` use **dual validation** to enforce absolute URIs:

1. `format: uri` — declarative URI syntax annotation (may be annotation-only in some validators per JSON Schema draft 2020-12)
2. `pattern: ^[a-zA-Z][a-zA-Z0-9+.-]*:` — explicit absolute-URI assertion that guarantees a scheme component

This dual approach ensures absolute-URI enforcement even when validators treat `format` as annotation-only and do not enforce format assertions.

**Valid URI schemes:**

| Scheme | Example | Use Case |
|---|---|---|
| `https://` | `https://github.com/everwork-ai/IEF-Protocol/pull/3` | GitHub PRs, commits, web resources |
| `file:///` | `file:///repo/schemas/task-envelope.schema.json` | Local or repository-rooted file references |
| `urn:ief:*` | `urn:ief:artifact:task-2026-001:pr-3` | IEF-specific identifiers |

**Invalid examples** (rejected by `pattern`):
- `docs/spec.md` — relative path, no scheme
- `../schemas/agent-card.schema.json` — relative path, no scheme
- `README.md` — relative path, no scheme

Relative paths **must** be converted to `file:///` or `https://` or `urn:ief:*` scheme to pass validation.

---

## 9. Cross-References

| Reference | Relationship |
|---|---|
| [IEF-Program#6](https://github.com/everwork-ai/IEF-Program/issues/6) | P1-Contracts execution plan — coordinates this work with Governance#2 and Operations#2 |
| [IEF-Governance#2](https://github.com/everwork-ai/IEF-Governance/issues/2) | Defines Contract-Critical profile that constrains this PR's merge criteria |
| [IEF-Operations#2](https://github.com/everwork-ai/IEF-Operations/issues/2) | Must reference TaskEnvelope / RunEvent / ArtifactRef / ContextRef — must NOT redefine them |

---

## 10. Schema File Index

| File | Object | `$id` |
|---|---|---|
| `schemas/agent-card.schema.json` | AgentCard | `https://everwork-ai.github.io/ief-protocol/schemas/v0.1.0/agent-card.schema.json` |
| `schemas/task-envelope.schema.json` | TaskEnvelope | `https://everwork-ai.github.io/ief-protocol/schemas/v0.1.0/task-envelope.schema.json` |
| `schemas/run-event.schema.json` | RunEvent | `https://everwork-ai.github.io/ief-protocol/schemas/v0.1.0/run-event.schema.json` |
| `schemas/artifact-ref.schema.json` | ArtifactRef | `https://everwork-ai.github.io/ief-protocol/schemas/v0.1.0/artifact-ref.schema.json` |
| `schemas/context-ref.schema.json` | ContextRef | `https://everwork-ai.github.io/ief-protocol/schemas/v0.1.0/context-ref.schema.json` |

All schemas use JSON Schema draft 2020-12.
