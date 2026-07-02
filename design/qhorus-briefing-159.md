# Cross-Repo Briefing: Update qhorus/docs/work-and-workitems.md

**From:** casehub-work session (issue #159)
**Target:** casehub-qhorus repo — `docs/work-and-workitems.md`
**Date:** 2026-07-02

## What needs updating

The mapping table at lines 49-57 of `work-and-workitems.md` has 7 rows and
predates FAULTED, OBSOLETE, MANUALLY_ESCALATED, and PROGRESS_UPDATE. It should
grow to cover all 12 WorkItemStatus values.

### Current table (7 rows)

```
Created → PENDING           Directive issued
PENDING → ASSIGNED          ACKNOWLEDGED
ASSIGNED → IN_PROGRESS      STATUS
IN_PROGRESS → DELEGATED     HANDOFF
ASSIGNED/IN_PROGRESS → REJECTED   FAILURE or DECLINE
IN_PROGRESS → COMPLETED     DONE
→ EXPIRED → ESCALATED       EXPIRED
```

### Missing rows to add

| WorkItem transition | Speech act | Normative meaning |
|---|---|---|
| any → FAULTED | FAILURE | System/infrastructure failure — no actor judgment; always FAILED, never DECLINED |
| any → OBSOLETE | *(no equivalent)* | Context-driven termination — case context changed, making work irrelevant |
| any → CANCELLED | DECLINE | Deliberate withdrawal of the obligation |
| ASSIGNED/IN_PROGRESS → SUSPENDED | STATUS | Human-layer: pause report — "obligation still held, work temporarily halted" |
| SUSPENDED → RESUMED | STATUS | Human-layer: resumption report — "active work resumes" |
| IN_PROGRESS → PROGRESS_UPDATE | STATUS | Actor reports intermediate progress (percentComplete, statusNote) |
| IN_PROGRESS → MANUALLY_ESCALATED | HANDOFF | Actor explicitly escalated to a different candidate group |

### Also consider

The existing table uses transition notation (`PENDING → ASSIGNED`). The new
`docs/NORMATIVE-ALIGNMENT.md` in casehub-work uses a two-table format:
1. WorkItemStatus → CommitmentState (status alignment)
2. WorkEventType → MessageType (event classification)

The Qhorus doc could either expand its existing transition table or adopt the
two-table format. The two-table format is more complete (covers all 26 events)
but may be too detailed for the conceptual document that `work-and-workitems.md`
is. A pragmatic middle ground: expand the transition table to 12 rows (one per
status) and add a note pointing to `docs/NORMATIVE-ALIGNMENT.md` in casehub-work
for the full 26-event classification.

## Reference

See `casehub-work/docs/NORMATIVE-ALIGNMENT.md` for the complete mapping tables
produced by #159.
