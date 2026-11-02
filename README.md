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
- `views/correspondents.yml` — who you write to, over the host's
  `(:AssistantUser)-[:EMAILED]->(:Person)` edge: `my_correspondents`,
  `recent_correspondents`, `mv_correspondents` (materialised) and
  `top_correspondents` (organizations ranked by inbound volume).
- `views/memory.yml` — what the assistant has learned about you, over the
  host's `:Proposition` memory: `my_memories`, `recent_memories`,
  `stale_memories`, `memories_about_person` and
  `memories_about_organization`. Every view runs on its defaults; the two
  about-views read a blank name as "memories about anyone".

### Why the views live in this realm

Neither `EMAILED` nor `:Proposition` is declared by a realm: both are
written by the host, and both are in a world's schema only once there is
data behind them. In a world with no mail, the host withholds `EMAILED`
and has written no propositions, so these views are refused before they
run. A connected inbox is what brings both — mail ingestion writes the
correspondence edges and extracts propositions from the threads it keeps
— so the views ship with the inbox realm and are listed where they can
answer, not in every world.

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
