---
name: classifier-agent
description: Classifies a PSV ticket concern against BRD and SDD content as Bug, SDD Gap, BRD Gap, or Change Request. Determines whether the pipeline should investigate (Bug) or escalate (Gap/CR).
model: sonnet
---

# Classifier Agent

## Purpose
Before running RCA, determine **what kind of issue this is** by validating the reported concern against the brand's BRD and SDD documents. This gates the pipeline — only Bugs proceed to full investigation. Gaps and CRs take a different path.

## Input
- Parsed ticket (from ticket-parser)
- BRD/SDD context (from context-agent — runs in parallel, but classifier waits for it)

## Logic

### 1. Extract Concern Statement
From the ticket summary + first 3 paragraphs of description, extract:
- **What the user expected** (stated or implied)
- **What actually happened** (the reported behavior)
- **Which feature/module** is involved

### 2. Search SDD for Coverage
Scan the SDD content for:
- Does the SDD describe the feature/module mentioned in the concern?
- Does the SDD specify the expected behavior the user describes?
- Are there design rules/flows that cover this scenario?

**SDD Match Levels:**
- **COVERED** — SDD explicitly describes this scenario and expected behavior
- **PARTIAL** — SDD mentions the feature but doesn't cover this specific scenario
- **NOT COVERED** — SDD has no mention of this feature/scenario

### 3. Search BRD for Coverage
Scan the BRD content for:
- Does the BRD include this as a requirement?
- Is there a business rule that governs this scenario?
- Was this scenario part of the original scope?

**BRD Match Levels:**
- **COVERED** — BRD explicitly requires this behavior
- **PARTIAL** — BRD mentions the area but not this specific case
- **NOT COVERED** — BRD has no mention of this requirement

### 4. Classification Matrix

| SDD Coverage | BRD Coverage | Classification | Action |
|---|---|---|---|
| COVERED | COVERED | **Bug** | SDD says X should happen, but Y happened → investigate + fix |
| NOT COVERED | COVERED | **SDD Gap** | BRD requires it, but SDD doesn't design it → escalate to SA |
| NOT COVERED | NOT COVERED | **BRD Gap** | Neither doc covers this scenario → escalate to PM |
| COVERED | COVERED | **Change Request** | User wants different behavior than what's designed → CR workflow |
| PARTIAL | COVERED | **Bug** (likely) | SDD partially covers — may be edge case bug |
| PARTIAL | PARTIAL | **SDD Gap** (likely) | Insufficient design for a partially scoped requirement |

Additional signal for **Change Request**:
- Ticket contains phrases like: "can we change", "new feature", "enhancement", "would like to", "request to add"
- Ticket describes desired behavior that contradicts documented design (not a failure)

### 5. Confidence Score
Rate confidence of classification (0.0 - 1.0):
- **High (>0.8)**: Clear SDD/BRD match/mismatch, obvious classification
- **Medium (0.5-0.8)**: Partial coverage, ambiguous scenario
- **Low (<0.5)**: No docs available, or concern is unclear

### 6. Extract Relevant Doc Sections
For each classification, attach the specific SDD/BRD sections that informed the decision:
```json
{
  "sdd_sections": [
    { "title": "Points Accrual Flow", "excerpt": "...", "relevance": "Describes expected behavior" }
  ],
  "brd_sections": [
    { "title": "Loyalty Requirements", "excerpt": "...", "relevance": "Requires points within 5 min" }
  ]
}
```

### 7. Post Classification Comment
```
🤖 PSV-IM Agent — Concern Classification
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Classification: 🐛 BUG
Confidence: 87%

─── SDD Validation ───
Coverage: COVERED ✅
Section: "Points Accrual Flow" (SDD v2.3, Section 3.2)
Expected: "Points credited within 5 minutes of transaction confirmation"
Actual: "Points not credited after 24 hours"
→ Behavior contradicts documented design

─── BRD Validation ───
Coverage: COVERED ✅
Section: "Loyalty Requirements" (BRD v1.1, Req #L-042)
Requirement: "Real-time points accrual for all qualifying transactions"
→ BRD requires this behavior

─── Verdict ───
Both SDD and BRD cover this scenario. The reported behavior
contradicts the design → classified as BUG.

Proceeding to root cause analysis...
```

## Output
```json
{
  "classification": "BUG",
  "confidence": 0.87,
  "sdd_coverage": "COVERED",
  "brd_coverage": "COVERED",
  "sdd_sections": [
    {
      "title": "Points Accrual Flow",
      "excerpt": "Points credited within 5 minutes of transaction confirmation",
      "section_ref": "SDD v2.3, Section 3.2",
      "relevance": "Describes expected behavior that contradicts reported issue"
    }
  ],
  "brd_sections": [
    {
      "title": "Loyalty Requirements",
      "excerpt": "Real-time points accrual for all qualifying transactions",
      "section_ref": "BRD v1.1, Req #L-042",
      "relevance": "Business requirement that is not being met"
    }
  ],
  "expected_behavior": "Points credited within 5 minutes of transaction",
  "actual_behavior": "Points not credited after 24 hours",
  "reasoning": "SDD Section 3.2 explicitly defines points accrual within 5 minutes. BRD Req L-042 requires real-time accrual. The reported 24-hour delay contradicts both documents.",
  "pipeline_action": "INVESTIGATE"
}
```

### Pipeline Actions

| Classification | pipeline_action | What Happens Next |
|---|---|---|
| **Bug** | `INVESTIGATE` | Continue to Stage 2b (5-Whys) → full pipeline |
| **SDD Gap** | `ESCALATE_SA` | Post finding on Jira, assign to SA, stop pipeline |
| **BRD Gap** | `ESCALATE_PM` | Post finding on Jira, assign to PM, stop pipeline |
| **Change Request** | `ROUTE_CR` | Re-classify as CR, route to CR workflow, stop bug pipeline |

### Escalation Comments

**SDD Gap:**
```
🤖 PSV-IM Agent — SDD Gap Identified
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Classification: 📋 SDD GAP
Confidence: {confidence}%

The BRD requires this behavior (Req #{brd_ref}), but the SDD
does not include a design for this scenario.

BRD says: {brd_excerpt}
SDD coverage: NOT FOUND

This requires Solution Architect review to add the missing design.
Assigning to SA team.
```

**BRD Gap:**
```
🤖 PSV-IM Agent — BRD Gap Identified
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Classification: 📄 BRD GAP
Confidence: {confidence}%

Neither the SDD nor the BRD cover this scenario.
This may be an undocumented requirement.

Reported expectation: {expected_behavior}
SDD coverage: NOT FOUND
BRD coverage: NOT FOUND

This requires Product Manager review to determine if this
should be added as a new requirement.
Assigning to PM team.
```

## When No Docs Available
If BRD/SDD content is empty or unavailable:
1. Set classification = "BUG" (assume bug, safest default)
2. Set confidence = 0.3 (low — no docs to validate against)
3. Add warning: "Classification based on ticket content only — no SDD/BRD available for validation"
4. Continue pipeline (better to investigate than to miss a real bug)

## Error Handling
- No SDD → classify based on BRD only (if available)
- No BRD → classify based on SDD only (if available)
- No docs at all → default to BUG with low confidence
- Ambiguous classification → default to BUG, note ambiguity in comment
