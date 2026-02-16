# AAH Extension: Human-in-the-Loop Decisions

**Proposal:** AAH-HITL-001  
**Status:** Draft  
**Authors:** Crouton (AI), Gracie Redfern  
**Date:** 2026-02-16  

---

## Abstract

This proposal extends AAH to support structured human-in-the-loop (HITL) decision points in multi-agent workflows. It defines a standard format for agents to request human decisions, for humans to respond with granular approvals/rejections, and for those decisions to flow back into the agent pipeline.

## Motivation

Multi-agent systems frequently need human oversight at critical decision points:

- **Approval gates**: "Should we proceed with this strategy?"
- **Multi-choice decisions**: "Which of these options should we pursue?"
- **Parameter validation**: "Are these budget allocations correct?"
- **Quality gates**: "Does this output meet standards?"

Current approaches are ad-hoc:
- Agents dump markdown with ✅/❌ emojis (not machine-readable)
- Humans approve/reject entire checkpoints (no granularity)
- Decision history is lost (no audit trail)
- No standard for decision responses to flow back to agents

This proposal standardizes HITL decision artifacts to enable:
- **Granular decisions**: Approve/reject individual items within a checkpoint
- **Machine-readable responses**: Structured data agents can act on
- **Audit trails**: Who decided what, when, with what rationale
- **Interoperability**: Any framework can emit/consume decision requests

---

## Specification

### 1. New Artifact Type: `decision/request`

A decision request artifact represents a set of decisions an agent is requesting from a human (or another agent with approval authority).

```json
{
  "aah_version": "0.1",
  "artifact": {
    "id": "aah_decision_abc123",
    "type": "decision/request",
    "title": "Approve Marketing Strategy",
    "created_at": "2026-02-16T10:00:00Z"
  },
  "source": {
    "agent_id": "product-manager",
    "task_id": "TASK-456"
  },
  "content": {
    "media_type": "application/vnd.aah.decision-request+json",
    "body": {
      "schema": "aah:decision/request@1.0",
      "data": {
        "context": "Review the proposed marketing strategy before launch.",
        "decisions": [
          {
            "id": "d1",
            "type": "approval",
            "prompt": "Approve target audience segments?",
            "description": "Segments: DIY brides (60%), Event planners (30%), Venues (10%)",
            "required": true,
            "default": null
          },
          {
            "id": "d2", 
            "type": "approval",
            "prompt": "Approve budget allocation?",
            "description": "$5k/month: Google (50%), LinkedIn (25%), Meta (15%), Content (10%)",
            "required": true,
            "default": null
          },
          {
            "id": "d3",
            "type": "choice",
            "prompt": "Select launch timing",
            "options": [
              { "value": "immediate", "label": "Launch immediately" },
              { "value": "next_week", "label": "Launch next Monday" },
              { "value": "next_month", "label": "Wait for Q2" }
            ],
            "required": true,
            "default": "next_week"
          },
          {
            "id": "d4",
            "type": "approval",
            "prompt": "Offer 14-day free trial?",
            "description": "No credit card required for trial signup",
            "required": false,
            "default": true
          }
        ],
        "blocking": [
          { "task_id": "TASK-457", "description": "Launch campaign" },
          { "task_id": "TASK-458", "description": "Create landing pages" }
        ],
        "deadline": "2026-02-17T17:00:00Z",
        "escalation": {
          "after": "24h",
          "to": "manager@example.com"
        }
      }
    }
  },
  "handoff": {
    "target_role": "human",
    "expects_response": true,
    "response_deadline": "2026-02-17T17:00:00Z",
    "priority": "high"
  },
  "lifecycle": {
    "status": "pending",
    "visibility": "team"
  }
}
```

### 2. Decision Types

| Type | Description | Response Format |
|------|-------------|-----------------|
| `approval` | Yes/No decision | `{ "approved": boolean }` |
| `choice` | Select from options | `{ "selected": string }` |
| `multi_choice` | Select multiple options | `{ "selected": string[] }` |
| `text` | Free-form text input | `{ "value": string }` |
| `number` | Numeric input | `{ "value": number }` |
| `date` | Date/time selection | `{ "value": ISO8601 }` |

### 3. Decision Request Schema

```typescript
interface DecisionRequest {
  schema: "aah:decision/request@1.0";
  data: {
    // Human-readable context for the decision set
    context?: string;
    
    // Array of decisions to be made
    decisions: Decision[];
    
    // Tasks/artifacts blocked by these decisions
    blocking?: BlockingItem[];
    
    // When decisions are needed by
    deadline?: string; // ISO 8601
    
    // Escalation if not resolved
    escalation?: {
      after: string;  // Duration (e.g., "24h", "7d")
      to: string;     // Email, agent_id, or role
    };
  };
}

interface Decision {
  id: string;                    // Unique within this request
  type: DecisionType;            // approval, choice, etc.
  prompt: string;                // The question being asked
  description?: string;          // Additional context
  required: boolean;             // Must be answered to proceed
  default?: any;                 // Default value if not answered
  options?: Option[];            // For choice/multi_choice types
  constraints?: {                // For text/number types
    min?: number;
    max?: number;
    pattern?: string;
  };
}

interface Option {
  value: string;
  label: string;
  description?: string;
}

interface BlockingItem {
  task_id?: string;
  artifact_id?: string;
  description: string;
}
```

### 4. New Artifact Type: `decision/response`

When a human (or authorized agent) responds to a decision request:

```json
{
  "aah_version": "0.1",
  "artifact": {
    "id": "aah_decision_resp_xyz789",
    "type": "decision/response",
    "title": "Response: Approve Marketing Strategy",
    "created_at": "2026-02-16T14:30:00Z"
  },
  "source": {
    "agent_id": "human:gracie",
    "agent_name": "Gracie Redfern",
    "agent_role": "product_owner"
  },
  "content": {
    "media_type": "application/vnd.aah.decision-response+json",
    "body": {
      "schema": "aah:decision/response@1.0",
      "data": {
        "request_id": "aah_decision_abc123",
        "responses": [
          {
            "decision_id": "d1",
            "approved": true,
            "comment": "Good segmentation"
          },
          {
            "decision_id": "d2",
            "approved": true,
            "comment": null
          },
          {
            "decision_id": "d3",
            "selected": "next_week",
            "comment": "Align with newsletter schedule"
          },
          {
            "decision_id": "d4",
            "approved": false,
            "comment": "Require CC to reduce spam signups"
          }
        ],
        "overall_status": "partial",  // all_approved, partial, all_rejected
        "summary": "Approved strategy with modification: require CC for trial"
      }
    }
  },
  "handoff": {
    "parent_artifact_ids": ["aah_decision_abc123"],
    "target_agent": "product-manager"
  },
  "lifecycle": {
    "status": "final"
  }
}
```

### 5. Decision Response Schema

```typescript
interface DecisionResponse {
  schema: "aah:decision/response@1.0";
  data: {
    // ID of the decision request being responded to
    request_id: string;
    
    // Individual decision responses
    responses: DecisionAnswer[];
    
    // Aggregated status
    overall_status: "all_approved" | "partial" | "all_rejected" | "pending";
    
    // Human summary of the decision
    summary?: string;
  };
}

interface DecisionAnswer {
  decision_id: string;           // Matches decision.id in request
  
  // For approval type
  approved?: boolean;
  
  // For choice/multi_choice type
  selected?: string | string[];
  
  // For text type
  value?: string;
  
  // For number type
  value?: number;
  
  // Optional comment/rationale
  comment?: string;
  
  // When this specific decision was made
  decided_at?: string;  // ISO 8601
}
```

### 6. Lifecycle States for Decision Artifacts

Decision requests move through these states:

| Status | Description |
|--------|-------------|
| `pending` | Awaiting human response |
| `partial` | Some decisions made, others pending |
| `resolved` | All decisions made |
| `expired` | Deadline passed without resolution |
| `escalated` | Escalated to another party |
| `withdrawn` | Cancelled by requesting agent |

### 7. Integration with Existing AAH

Decision artifacts integrate with existing AAH concepts:

- **Lineage**: Response artifacts link to requests via `parent_artifact_ids`
- **Handoff**: Requests specify `target_role: "human"` and `expects_response: true`
- **Lifecycle**: Standard status field tracks decision state
- **Extensions**: Frameworks can add UI hints, notification preferences, etc.

### 8. Example: Full Workflow

```
1. Agent creates decision/request artifact
   → Status: pending
   → Handoff: target_role=human, expects_response=true

2. Storage system notifies human (email, Slack, dashboard)

3. Human reviews decisions in UI
   → Clicks approve/reject on each item
   → Adds comments where needed

4. Human submits responses
   → decision/response artifact created
   → Links to request via parent_artifact_ids
   → Status: resolved (or partial)

5. Orchestrator receives response artifact
   → Parses structured decisions
   → Routes approved items forward
   → Handles rejections (retry, escalate, cancel)

6. Blocked tasks unblock based on decision outcomes
```

---

## Rendering Hints

UIs implementing decision request rendering SHOULD:

1. Display each decision as an interactive element (checkbox, radio, input)
2. Show decision descriptions and context
3. Indicate required vs optional decisions
4. Show default values where specified
5. Allow batch approve/reject for approval-type decisions
6. Capture comments for audit trail
7. Show blocking tasks/artifacts
8. Display deadline and escalation info

---

## Backwards Compatibility

This proposal adds new artifact types and schemas. It does not modify existing AAH structures. Implementations not supporting HITL can ignore `decision/*` artifact types.

---

## Security Considerations

- **Authorization**: Implementations MUST verify responder has authority to make decisions
- **Audit**: All decision responses SHOULD be immutable once submitted
- **Tampering**: Response artifacts SHOULD be signed or verified
- **Escalation**: Escalation targets SHOULD be validated

---

## References

- [AAH Specification v0.1](./AAH-SPEC-v0.1.md)
- [Human-in-the-Loop ML Patterns](https://arxiv.org/abs/...)
- [Approval Workflows in BPM](https://...)

---

## Changelog

- 2026-02-16: Initial draft
