# IEF-Protocol

> IEF Protocol Layer -- shared contracts for AI-worker communication and collaboration.

## One-sentence role inside IEF

Defines the durable shared objects and communication contracts that all IEF layers use to exchange work, status, artifacts, and context.

## What this repository owns

- AgentCard schema and validation
- Capability schema and registry
- TaskEnvelope schema and validation
- Message schema
- Artifact schema and provenance tracking
- ContextRef schema
- StatusEvent schema and type system
- Handoff schema and context preservation
- ApprovalRequest schema
- Protocol versioning rules

## What this repository explicitly does NOT own

- Orchestration policy (that is IEF-Work)
- Task state machine logic (that is IEF-Work)
- Concrete transport runtime (that is IEF-Adapters or host systems)
- Execution implementation (that is IEF-Workers)

## Core objects

| Object | Description |
|---|---|
| `AgentCard` | Machine-readable declaration of an AI worker or service |
| `Capability` | Declared ability of a worker (skills, tools, constraints) |
| `TaskEnvelope` | Wrapped task with metadata, policy profile, and context refs |
| `Message` | Typed communication between workers |
| `Artifact` | Tangible output with provenance, immutability, and approval relevance |
| `StatusEvent` | Typed status update (queued, assigned, started, blocked, completed, etc.) |
| `Handoff` | Work transfer with preserved context references |
| `ApprovalRequest` | Structured approval or exception request |
| `ContextRef` | Stable reference to knowledge, memory, policy, or prior artifacts |

## Protocol rules

1. Object schemas must be explicit (JSON Schema)
2. Artifacts must be first-class objects
3. Status must be typed, not free-form text
4. Handoff must preserve context references
5. Protocol versioning must be explicit in every envelope

## Internal-first strategy

- Start with JSON schemas, internal interfaces, event contracts, validation logic
- Later add external gateways, compatibility bridges, ecosystem interoperability

## Dependencies on other IEF repos

| Dependency | Direction | Purpose |
|---|---|---|
| IEF-Governance | consumed | Policy metadata for task risk classification |
| IEF-Work | produced | Schemas consumed by task and run management |
| IEF-Workers | produced | Schemas consumed by worker implementations |
| IEF-Adapters | produced | Schemas consumed by host-environment bindings |
| IEF-Knowledge | consumed | ContextRef semantics and knowledge addressing |

## Current status

- [x] Repository created
- [ ] AgentCard schema drafted
- [ ] Artifact schema drafted
- [ ] StatusEvent schema drafted
- [ ] Handoff schema drafted
- [ ] Protocol versioning defined

## Near-term roadmap

1. Draft AgentCard JSON schema
2. Draft Artifact JSON schema
3. Draft StatusEvent JSON schema
4. Draft Handoff JSON schema
5. Define protocol versioning strategy
6. Create validation logic for all schemas

## Naming / glossary references

| Term | Meaning |
|---|---|
| AgentCard | Machine-readable worker declaration |
| TaskEnvelope | Wrapped task with metadata and context |
| Artifact | Tangible, versioned output with provenance |
| StatusEvent | Typed work status update |
| Handoff | Work transfer preserving context |

---

**Short term**: Keep protocol objects inside IEF-Work as `internal_protocol/` until the internal contract stabilizes.
**Long term**: Split into IEF-Protocol when contracts are battle-tested.

See [IEF Design Pack v0.2](https://github.com/brantzh6/IEF-Governance/blob/main/docs/IEF_Design_Pack_v0.2.md) section 12 for full protocol model.
