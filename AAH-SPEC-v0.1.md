# Agent Artifact Handoff (AAH) Specification

**Version:** 0.1.0-draft  
**Status:** Proposal  
**Authors:** Gracie Redfern, Crouton  
**Date:** 2026-02-11

---

## Abstract

The Agent Artifact Handoff (AAH) specification defines a standard format for artifacts produced by AI agents. It enables interoperability between agent frameworks, consistent rendering in user interfaces, and reliable handoffs between agents in multi-agent systems.

## Motivation

As AI agents become more capable, they produce increasing volumes of artifacts: research documents, code, analysis, specs, and more. Today, these artifacts are stored ad-hoc in local filesystems, Git repositories, or proprietary formats. This fragmentation creates problems:

- **Inaccessibility**: Artifacts trapped on local machines can't be viewed remotely
- **No provenance**: Who created this? When? As part of what task?
- **No interop**: Agent A's output can't easily flow to Agent B
- **No lifecycle**: Everything is permanent or manually deleted
- **No standards**: Every framework invents its own format

AAH addresses these problems with a minimal, extensible specification.

---

## Terminology

- **Artifact**: Any discrete output produced by an agent (document, code, image, structured data)
- **Agent**: An AI system that performs tasks autonomously or semi-autonomously
- **Handoff**: The transfer of an artifact from one agent to another
- **Session**: A bounded execution context for an agent
- **Framework**: Software that orchestrates agent execution (e.g., Clawdbot, CrewAI, AutoGen)

---

## Specification

### 1. Artifact Envelope

Every AAH artifact MUST be wrapped in an envelope with the following structure:

```json
{
  "aah_version": "0.1",
  "artifact": { ... },
  "source": { ... },
  "content": { ... },
  "lifecycle": { ... },
  "handoff": { ... },
  "extensions": { ... }
}
```

### 2. Core Fields

#### 2.1 `aah_version` (REQUIRED)

The version of the AAH specification this artifact conforms to.

```json
"aah_version": "0.1"
```

#### 2.2 `artifact` (REQUIRED)

Core artifact identification.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Unique identifier (recommend UUID or prefixed ID) |
| `type` | string | Yes | Artifact type (see Section 3) |
| `title` | string | No | Human-readable title |
| `summary` | string | No | Brief description (max 500 chars) |
| `created_at` | ISO 8601 | Yes | Creation timestamp |
| `updated_at` | ISO 8601 | No | Last modification timestamp |

```json
"artifact": {
  "id": "aah_7f3b2a1c-9d4e-4f5a-8b6c-1e2d3f4a5b6c",
  "type": "document/markdown",
  "title": "Competitor Analysis: Table Management Software",
  "summary": "Analysis of 5 major competitors in the restaurant table management space",
  "created_at": "2026-02-11T23:10:28Z"
}
```

#### 2.3 `source` (REQUIRED)

Provenance information about who/what created this artifact.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `agent_id` | string | Yes | Identifier for the agent |
| `agent_name` | string | No | Human-readable agent name |
| `agent_role` | string | No | Agent's role (e.g., "researcher", "developer") |
| `framework` | string | No | Agent framework used |
| `framework_version` | string | No | Framework version |
| `session_id` | string | No | Session/run identifier |
| `task_id` | string | No | Task or ticket identifier |
| `task_description` | string | No | What the agent was trying to accomplish |
| `model` | string | No | LLM model used (e.g., "claude-sonnet-4") |
| `organization_id` | string | No | Org identifier for multi-tenant systems |

```json
"source": {
  "agent_id": "research-agent-01",
  "agent_name": "Research Agent",
  "agent_role": "researcher",
  "framework": "clawdbot",
  "framework_version": "1.2.3",
  "session_id": "ses_abc123",
  "task_id": "ART-456",
  "task_description": "Research competitor table management features",
  "model": "claude-sonnet-4"
}
```

#### 2.4 `content` (REQUIRED)

The actual artifact content.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `media_type` | string | Yes | MIME type of content |
| `encoding` | string | No | Character encoding (default: utf-8) |
| `body` | string | Conditional | Inline content (for small artifacts) |
| `body_url` | string | Conditional | URL to fetch content (for large artifacts) |
| `body_hash` | string | No | SHA-256 hash of content for integrity |
| `size_bytes` | integer | No | Content size in bytes |
| `token_count` | integer | No | Estimated token count (for LLM context budgeting) |

Either `body` or `body_url` MUST be present.

```json
"content": {
  "media_type": "text/markdown",
  "encoding": "utf-8",
  "body": "# Competitor Analysis\n\n## Overview\n...",
  "size_bytes": 4523,
  "token_count": 1847
}
```

**Supported media types:**

| Category | Media Types |
|----------|-------------|
| Documents | `text/markdown`, `text/plain`, `text/html` |
| Code | `text/x-python`, `text/javascript`, `text/typescript`, `application/json`, `text/yaml` |
| Data | `application/json`, `text/csv`, `application/x-ndjson` |
| Images | `image/png`, `image/jpeg`, `image/svg+xml` |
| Structured | `application/vnd.aah.structured+json` (see Section 4) |

#### 2.5 `lifecycle` (OPTIONAL)

Artifact lifecycle management.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `retention` | string | No | Retention policy: `ephemeral`, `7d`, `30d`, `90d`, `permanent` |
| `expires_at` | ISO 8601 | No | Explicit expiration timestamp |
| `visibility` | string | No | Access level: `private`, `team`, `organization`, `public` |
| `status` | string | No | Artifact status: `draft`, `review`, `approved`, `superseded`, `archived` |
| `superseded_by` | string | No | ID of artifact that replaces this one |
| `tags` | array | No | Freeform tags for categorization |

```json
"lifecycle": {
  "retention": "30d",
  "visibility": "team",
  "status": "draft",
  "tags": ["competitor-research", "planseats", "q1-2026"]
}
```

#### 2.6 `handoff` (OPTIONAL)

Information for agent-to-agent handoffs.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `target_agent` | string | No | Intended recipient agent ID |
| `target_role` | string | No | Intended recipient role (if agent unknown) |
| `expects_response` | boolean | No | Whether a response artifact is expected |
| `response_deadline` | ISO 8601 | No | Deadline for response |
| `parent_artifact_ids` | array | No | IDs of artifacts this one derives from |
| `context` | object | No | Additional context for recipient agent |
| `priority` | string | No | `low`, `normal`, `high`, `urgent` |

```json
"handoff": {
  "target_role": "pm",
  "expects_response": true,
  "response_deadline": "2026-02-12T12:00:00Z",
  "parent_artifact_ids": ["aah_parent123"],
  "priority": "normal",
  "context": {
    "next_step": "Create feature tickets based on this research",
    "constraints": "Focus on features we can ship in Q1"
  }
}
```

#### 2.7 `extensions` (OPTIONAL)

Framework-specific or custom extensions. Extensions SHOULD be namespaced.

```json
"extensions": {
  "clawdbot": {
    "gateway_id": "gw_abc123",
    "channel": "slack"
  },
  "linear": {
    "issue_id": "ART-456",
    "project_id": "proj_xyz"
  }
}
```

---

### 3. Artifact Types

Artifact types follow a `category/subtype` pattern:

| Type | Description |
|------|-------------|
| `document/markdown` | Markdown document |
| `document/text` | Plain text document |
| `document/html` | HTML document |
| `code/python` | Python source code |
| `code/javascript` | JavaScript source code |
| `code/typescript` | TypeScript source code |
| `code/generic` | Code in other languages |
| `data/json` | JSON data |
| `data/yaml` | YAML data |
| `data/csv` | CSV tabular data |
| `image/screenshot` | Screenshot image |
| `image/diagram` | Diagram or chart |
| `image/generated` | AI-generated image |
| `structured/spec` | Product/technical specification |
| `structured/analysis` | Analysis or research output |
| `structured/plan` | Plan or roadmap |
| `structured/review` | Code review or QA output |

Implementations MAY define additional types using the `x-` prefix (e.g., `x-custom/mytype`).

---

### 4. Structured Artifacts

For artifacts with well-defined schemas, use `application/vnd.aah.structured+json` with a `schema` field:

```json
"content": {
  "media_type": "application/vnd.aah.structured+json",
  "body": {
    "schema": "aah:analysis/competitor",
    "data": {
      "competitors": [
        {
          "name": "OpenTable",
          "strengths": ["Brand recognition", "Large network"],
          "weaknesses": ["High fees", "Complex onboarding"]
        }
      ],
      "recommendation": "Focus on SMB segment underserved by OpenTable"
    }
  }
}
```

---

### 5. Transport

AAH is transport-agnostic. Artifacts MAY be transmitted via:

- **HTTP API**: POST/PUT to artifact storage endpoints
- **Message queues**: Kafka, RabbitMQ, SQS
- **File system**: Written as `.aah.json` files
- **Embedded**: Inline in agent framework messages

Implementations SHOULD support JSON serialization. Implementations MAY additionally support YAML or MessagePack.

---

### 6. Discovery & Querying

Artifact storage systems implementing AAH SHOULD support querying by:

- `artifact.id` — exact match
- `source.agent_id` — artifacts from a specific agent
- `source.session_id` — artifacts from a specific session
- `source.task_id` — artifacts related to a task
- `artifact.type` — by artifact type
- `lifecycle.status` — by status
- `lifecycle.tags` — by tags
- `created_at` — date range

---

### 7. Rendering Hints

To assist UI rendering, artifacts MAY include rendering hints in extensions:

```json
"extensions": {
  "rendering": {
    "preferred_view": "split",
    "syntax_highlight": "python",
    "collapse_sections": ["methodology", "appendix"],
    "table_of_contents": true
  }
}
```

---

## Examples

### Example 1: Research Document

```json
{
  "aah_version": "0.1",
  "artifact": {
    "id": "aah_7f3b2a1c-9d4e-4f5a-8b6c-1e2d3f4a5b6c",
    "type": "document/markdown",
    "title": "Competitor Analysis: Table Management Software",
    "summary": "Analysis of 5 major competitors in the restaurant table management space",
    "created_at": "2026-02-11T23:10:28Z"
  },
  "source": {
    "agent_id": "research-agent",
    "agent_role": "researcher",
    "framework": "clawdbot",
    "session_id": "ses_abc123",
    "task_id": "ART-456",
    "model": "claude-sonnet-4"
  },
  "content": {
    "media_type": "text/markdown",
    "body_url": "https://artifacts.example.com/aah_7f3b2a1c.md",
    "body_hash": "sha256:a1b2c3d4...",
    "size_bytes": 12847,
    "token_count": 4521
  },
  "lifecycle": {
    "retention": "permanent",
    "visibility": "team",
    "status": "approved",
    "tags": ["competitor-research", "planseats"]
  },
  "handoff": {
    "target_role": "pm",
    "expects_response": true,
    "context": {
      "next_step": "Create feature tickets based on findings"
    }
  }
}
```

### Example 2: Code Artifact

```json
{
  "aah_version": "0.1",
  "artifact": {
    "id": "aah_code_123",
    "type": "code/typescript",
    "title": "ArtifactUploader component",
    "created_at": "2026-02-11T23:45:00Z"
  },
  "source": {
    "agent_id": "dev-agent",
    "agent_role": "developer",
    "framework": "clawdbot",
    "task_id": "ART-789"
  },
  "content": {
    "media_type": "text/typescript",
    "body": "import React from 'react';\n\nexport function ArtifactUploader({ onUpload }: Props) {\n  // ...\n}",
    "token_count": 342
  },
  "lifecycle": {
    "retention": "7d",
    "status": "review"
  },
  "handoff": {
    "target_role": "qa",
    "expects_response": true,
    "parent_artifact_ids": ["aah_spec_456"]
  }
}
```

### Example 3: Ephemeral Scratch Artifact

```json
{
  "aah_version": "0.1",
  "artifact": {
    "id": "aah_scratch_999",
    "type": "data/json",
    "created_at": "2026-02-11T23:50:00Z"
  },
  "source": {
    "agent_id": "research-agent",
    "session_id": "ses_xyz"
  },
  "content": {
    "media_type": "application/json",
    "body": "{\"raw_data\": [1, 2, 3], \"intermediate\": true}"
  },
  "lifecycle": {
    "retention": "ephemeral",
    "visibility": "private"
  }
}
```

---

## Security Considerations

- **Authentication**: Artifact storage systems SHOULD require authentication for upload and access
- **Authorization**: Respect `visibility` field; enforce organization boundaries
- **Content validation**: Validate content matches declared `media_type`
- **Size limits**: Implementations SHOULD enforce maximum artifact sizes
- **Secrets**: Artifacts MUST NOT contain credentials, API keys, or other secrets

---

## Future Considerations

The following are under consideration for future versions:

- **Versioning**: Track artifact versions and diffs
- **Signing**: Cryptographic signatures for artifact authenticity
- **Streaming**: Support for streaming large artifacts
- **Relationships**: Richer artifact relationship types (derives-from, supersedes, relates-to)
- **Schemas registry**: Central registry for structured artifact schemas
- **Reactions/Comments**: Human feedback on artifacts

---

## Appendix A: JSON Schema

A formal JSON Schema for AAH v0.1 will be published at:

```
https://aah-spec.org/schemas/v0.1/artifact.json
```

---

## Appendix B: Reference Implementations

- **Clawdbot**: Native AAH support (planned)
- **aah-js**: JavaScript/TypeScript SDK (planned)
- **aah-python**: Python SDK (planned)

---

## Changelog

### v0.1.0-draft (2026-02-11)
- Initial draft specification
