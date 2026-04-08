# PSV-IM Agent — Pipeline Flow

## Full Pipeline Diagram

```
                    ┌─────────────────────┐
                    │    JIRA (PSV)        │
                    │  P0/P1 tickets       │
                    └─────────┬───────────┘
                              │ poll every 3 min
                              ▼
                ┌─────────────────────────────┐
                │  Stage 0: Jira Poller       │
                │  Agent: jira-poller          │
                │  Model: haiku                │
                │  ─────────────────────────   │
                │  JQL query → filter new →    │
                │  add label → spawn workflow  │
                └─────────────┬───────────────┘
                              │ for each new ticket
                              ▼
                ┌─────────────────────────────┐
                │  Stage 1: Ticket Parser      │
                │  Agent: ticket-parser         │
                │  Model: haiku                 │
                │  ─────────────────────────    │
                │  Extract: brand, component,   │
                │  org_id, dataflow_id, etc.    │
                │  Component = ROUTING KEY      │
                └─────────────┬───────────────┘
                              │
                    ┌─────────┴─────────┐
                    │                   │
                    ▼                   ▼
    ┌───────────────────────┐ ┌───────────────────────┐
    │ Stage 2a: Context     │ │ Stage 2b: 5-Whys RCA  │
    │ Agent: context-agent  │ │ Agent: five-whys-agent │
    │ Model: sonnet         │ │ Model: opus            │
    │ ───────────────────── │ │ ───────────────────── │
    │ Fetch BRD/SDD from    │ │ Iterative root cause   │
    │ brand DB + Confluence │ │ 3-5 levels deep        │
    │                       │ │ Generate data requests │
    └───────────┬───────────┘ └───────────┬───────────┘
                │                         │
                └────────────┬────────────┘
                             │ parallel complete
                             ▼
                ┌─────────────────────────────┐
                │  Stage 3: KB Correlation     │
                │  Agent: kb-agent             │
                │  Model: sonnet               │
                │  ─────────────────────────   │
                │  Search KB + past tickets    │
                │  >80% match → FAST PATH      │
                │  <80% → full investigation   │
                └─────────────┬───────────────┘
                              │
                    ┌─────────┴─────────┐
                    │                   │
               FAST PATH           FULL PATH
                    │                   │
                    │                   ▼
                    │   ┌─────────────────────────────┐
                    │   │  Stage 4: Component Router   │
                    │   │  Agent: component-router      │
                    │   │  Model: haiku                 │
                    │   │  ─────────────────────────    │
                    │   │  Component → Team Skill       │
                    │   │  ─────────────────────────    │
                    │   │  Integrations → integration   │
                    │   │  Webapp-CRM → webapp          │
                    │   │  MobileApp → mobile           │
                    │   │  Config → config              │
                    │   │  Unknown → ESCALATE           │
                    │   └─────────────┬───────────────┘
                    │                 │
                    │                 ▼
                    │   ┌─────────────────────────────┐
                    │   │  Stage 5: Team Skill         │
                    │   │  (External — team-owned)     │
                    │   │  ─────────────────────────   │
                    │   │  Investigate → Fix → Test    │
                    │   │  Uses team's own APIs/repos  │
                    │   └─────────────┬───────────────┘
                    │                 │
                    │                 ▼
                    │   ┌─────────────────────────────┐
                    │   │  Stage 6: QA Agent           │
                    │   │  Agent: qa-agent             │
                    │   │  Model: sonnet               │
                    │   │  ─────────────────────────   │
                    │   │  Score: /13                   │
                    │   │  >=10 PASS                    │
                    │   │  7-9 CONDITIONAL              │
                    │   │  <7 FAIL → retry (max 3x)    │
                    │   └───────┬──────────┬──────────┘
                    │           │          │
                    │        PASS       FAIL
                    │           │          │
                    │           │     ┌────┘
                    │           │     │ retry < 3?
                    │           │     ├─ YES → back to Stage 5
                    │           │     └─ NO  → ESCALATE
                    │           │
                    └─────┬─────┘
                          │
                          ▼
                ┌─────────────────────────────┐
                │  Stage 7: Approval Gate      │
                │  Agent: approval-gate         │
                │  Model: haiku                 │
                │  ─────────────────────────    │
                │  Post RCA + fix + tests       │
                │  Wait for: APPROVE/REJECT     │
                │  ─────────────────────────    │
                │  Timeout: 24h P0, 72h P1      │
                └───────┬──────────┬──────────┘
                        │          │
                     APPROVE    REJECT
                        │          │
                        │          └─→ Learn + escalate or retry
                        ▼
                ┌─────────────────────────────┐
                │  Stage 8: Reporter           │
                │  Agent: reporter             │
                │  Model: sonnet               │
                │  ─────────────────────────   │
                │  Post resolution comment     │
                │  Save to KB                  │
                │  Record metrics              │
                │  Clean up labels             │
                │  ─────────────────────────   │
                │  ✅ DONE                     │
                └─────────────────────────────┘
```

## Decision Points

### Decision 1: KB Fast Path (Stage 3)
```
IF kb_match.similarity > 0.80:
  → Skip Stages 4-6
  → Go directly to Stage 7 (Approval) with known fix
ELSE:
  → Continue to Stage 4 (Component Router)
```

### Decision 2: Component Routing (Stage 4)
```
IF component has registered skill:
  → Invoke team skill
ELIF component in ESCALATE list:
  → Post partial RCA to Jira, ESCALATE
ELSE:
  → Attempt inference from keywords
  → If no match → ESCALATE
```

### Decision 3: QA Retry (Stage 6)
```
IF qa_score >= 10:
  → PASS → Stage 7
ELIF qa_score >= 7:
  → CONDITIONAL PASS → Stage 7 (with warnings)
ELIF retry_count < 3:
  → FAIL → Retry Stage 5 with QA feedback
ELSE:
  → FAIL → ESCALATE to human
```

### Decision 4: Approval (Stage 7)
```
IF response == "APPROVE":
  → Apply fix → Stage 8
ELIF response == "REJECT":
  → If feedback provided → retry with feedback
  → If final rejection → ESCALATE
ELIF response == "MORE INFO":
  → Fetch requested data → re-post → continue waiting
ELIF timeout exceeded:
  → Re-notify → if still no response → ESCALATE
```

## Concurrency Model

```
┌─────────────────────────────────────────────┐
│            Polling Loop (continuous)          │
│                                              │
│  Slot 1: PSV-12345 [Stage 5 — integration]  │
│  Slot 2: PSV-12346 [Stage 2 — context]      │
│  Slot 3: PSV-12347 [Stage 7 — approval]     │
│  Slot 4: (empty)                             │
│  Slot 5: (empty)                             │
│                                              │
│  Queue: PSV-12348 (P1, waiting for slot)     │
│                                              │
│  P0 PREEMPTION: P0 ticket always gets a slot │
│  by pausing lowest-priority P1 workflow      │
└─────────────────────────────────────────────┘
```

## Labels Used

| Label | Applied At | Removed At |
|---|---|---|
| `psv-im-processing` | Stage 0 (poller detects) | Stage 8 (resolved) |
| `psv-im-awaiting-approval` | Stage 7 (approval posted) | Stage 7 (response received) |
| `psv-im-resolved` | Stage 8 (resolution complete) | Never |
| `psv-im-escalated` | Any stage (escalation) | Human takes over |
| `psv-im-needs-human` | Any stage (needs human) | Human takes over |
| `psv-im-needs-info` | Stage 1 (missing fields) | Info provided |
