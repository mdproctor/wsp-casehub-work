# Design Journal — epic-exclusion-audit

### 2026-05-19 · §Build Roadmap

#186 — `ExclusionPolicy.isExcluded() : boolean` replaced by `check() : PolicyDecision`.
`PolicyDecision` is a pure-Java record (Tier 1 api module) carrying `denied`, `reason`,
and a shared `ALLOW` constant. The reason for a denial now travels from the policy that
knows it (the implementation) to the audit entry and exception message, rather than being
hardcoded in `WorkItemService`. This makes custom policy implementations (LDAP, role-based,
time-window) first-class — they control the audit narrative without touching the service layer.

`BlockedAttemptAuditService` — a new `@ApplicationScoped` CDI bean with
`@Transactional(REQUIRES_NEW)` — writes `CLAIM_DENIED` and `DELEGATE_DENIED` audit entries
at the `claim()` and `delegate()` enforcement points, committing independently of the
outer transaction that then rolls back. Guard ordering corrected: status check fires before
exclusion check so that rejected-status operations do not generate phantom audit entries.
