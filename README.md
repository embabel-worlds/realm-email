# realm-email

The abstract-inbox realm. Declares the universal `email.thread` DomainType
and the attention-worthiness rules — both deterministic and LLM-judged —
that operate on it. Provider-agnostic: any inbox integration whose signals
satisfy `email.thread` benefits from these rules without per-provider
duplication.

## What's in here

- `types/email.yml` — the `email.thread` DomainType. Properties cover the
  surface predicates need to read (participants, last_sender, age, …) plus
  a `provider` field recording which mail system produced the signal
  (`gmail` today, future: `exchange`, `imap`, …).
- `actions/policy_email_unreplied.yaml` — deterministic fast-path rule
  (`stepType: policy`). Cheap AST-walk predicate; the host's planner
  picks this before any LLM triage.
- `actions/triage_email_attention.yaml` — LLM-judged second-look
  (`stepType: triage`). Runs only when no deterministic policy matched
  the signal; the planner's GOAP composition handles the escalation.

## What's not in here (deliberately)

- **No code.** This is a pure-YAML realm — no `src/`, no `apis/`, no
  authentication wiring. Provider-specific realms (or the in-tree Gmail
  integration) ship the *signal-producing* side; this realm ships the
  *judgment* side.
- **No provider lock-in.** Adding `realm-exchange` later means emitting
  signals with `typeName=email.thread` and `provider=exchange`. The
  policies here apply unchanged.

## How it composes

When a `realm-email`-aware host receives a fresh `email.thread` signal,
it runs one `AgentProcess` per signal containing every loaded
`policy.* + triage.*` Action whose `on:` matches `email.thread`. The
planner's UtilityAI picks the cheap deterministic policy first; if it
matches, it writes an `AttentionCandidate`, the goal is satisfied, the
process terminates — saving the LLM call. If the policy abstains, the
LLM triage fires and judges. See `realm-spec` for the framework details.
