# FootGrid core platform architecture

## Scope and product interpretation

This design covers the core path from match setup through live event capture and
post-match statistics. It is deliberately suitable for casual turf games, club
fixtures, and organized leagues. It does not yet include payments, registrations,
streaming, social publishing, or AI-generated reports; those should consume the
same event stream after the core is proven.

The competitor research validates the primitives rather than prescribing an
implementation: Club Duelz emphasizes federation operations, live scoring and
fan-facing updates; GameGrid highlights flexible competition formats, live events,
lineups and standings; NEXTXI makes the long-lived player record a product
asset. The recommended FootGrid core is therefore **a reliable match-recording
system with a durable player history**, not merely a scorekeeping UI.

The three provided prototypes imply these functional requirements:

| Prototype evidence | Backend requirement |
| --- | --- |
| `match-setup.html`: 5v5/6v6/8v8/11v11, duration, halves, venue, teams, shirt numbers and player lookup by phone | Match template, flexible format rules, match rosters, positions, guest/opposition placeholders |
| `match-logger.html`: role-specific actions, goal finish type, opposing shirt number, score controls, undo | Versioned event ledger, action taxonomy, event subjects, score derivation, compensating reversal |
| `pitch-logger.html`: players on the pitch, bench, pitch slot and substitutions | Lineup state machine, position/slot snapshots and substitution events |

### Product invariants

1. A score is never the source of truth. It is a projection of confirmed,
   non-reversed `GOAL` and `SCORE_ADJUSTMENT` events.
2. A logger never mutates a historic event in place. Corrections create a
   reversal or superseding revision linked to the original event.
3. A device retry must never create a duplicate goal. Every event write carries
   a client-generated UUID and an idempotency key.
4. A match has one monotonically increasing event sequence. Writes specify the
   sequence they observed; the API returns `409` when another logger won the
   race.
5. Named and anonymous opposition are both first-class. An opponent may be a
   registered `player_id`, or a match-scoped participant identified by shirt
   number only. Do not create a fake user for an unknown opponent.
6. Statistics are projections. They can be regenerated from the immutable event
   ledger when action definitions or aggregation logic improve.

## Recommended system shape

Use independently deployable Go services, but begin with a **single Aurora
PostgreSQL cluster with schema ownership** rather than prematurely operating
many database clusters. Each service owns its schema and only it may write there;
other services use APIs/events and keep local read models. This keeps the
strongly consistent live-match write path in one transaction while preserving a
clean extraction path.

```text
Mobile app / web logger / public web
             |
CloudFront + WAF
             |
API Gateway HTTP API ------ API Gateway WebSocket API
             |                         |
     Cognito JWT authorizer             +-- Connection registry
             |
  Logger/API BFF (Go Lambda) -----------+-------------------+
             |                                               |
       synchronous commands                            EventBridge
             |                                               |
  +----------v-----------+  +-------------+  +----------------v----------+
  | Match service        |  | Competition |  | Projection / notifications |
  | roster, events,      |  | fixtures,   |  | scores, stats, standings,  |
  | live session         |  | permissions |  | WebSocket fan-out          |
  +----------+-----------+  +------+------+  +---------------------------+
             |                     |                 |
             +------- Aurora PostgreSQL + RDS Proxy --+
                         (logical schemas)
             |
                    transactional outbox
```

### Service boundaries

| Service | Owns | Synchronous responsibility | Publishes |
| --- | --- | --- | --- |
| Identity & tenancy | organizations, memberships, player identities, consent | authorize organization scope and resolve player | `MemberChanged`, `PlayerChanged` |
| Competition | competitions, seasons, stages, teams, venues, fixtures | set up a fixture/match and assign loggers | `MatchScheduled`, `MatchStatusChanged` |
| Match | roster, lineup, live session, event ledger | the sole writer for live match commands | `MatchEventRecorded`, `MatchEventReversed`, `MatchFinalized` |
| Projection | score/timeline/stat/standing read models | serve public and app read APIs | `MatchSnapshotUpdated`, `StandingUpdated` |
| Realtime & notification | WebSocket connections and device subscriptions | fan-out low-latency update notifications | — |
| Media (later) | media asset metadata | direct S3 upload authorization | `MediaAttached` |

The public HTTP API is versioned at `/v1`. A thin BFF aggregates only read
responses needed by mobile screens; it must not duplicate domain rules. The
OpenAPI document defines the externally callable contract. Internal EventBridge
events should be versioned separately (`detail-type: match.event.recorded.v1`).

## AWS serverless deployment decisions

| Concern | Decision | Reason |
| --- | --- | --- |
| Compute | Go 1.x Lambda, ARM64, one small handler per service boundary | fast cold starts, low operational overhead, simple independent deploys |
| Edge/API | API Gateway HTTP API + JWT authorizer; WebSocket API for live subscriptions | cheaper than REST API and suited to token auth/live push |
| Identity | Amazon Cognito user pools; phone OTP optional; app maps `sub` to internal user UUID | avoid storing password/OTP secrets and support future social sign-in |
| Database | Aurora PostgreSQL Serverless v2, private subnets, RDS Proxy, PITR | PostgreSQL constraints/transactions fit a match ledger; Proxy protects connection limits |
| Event delivery | transactional outbox table, Lambda publisher, EventBridge, SQS per consumer, DLQ | no lost publication between database commit and async consumers |
| Cache | DynamoDB only for WebSocket connection registry/rate limiting; Redis only if profiling warrants it | avoids using cache as a source of truth |
| Files | S3 private bucket + pre-signed upload URLs + CloudFront signed access | keep large media outside Postgres |
| Secrets/keys | Secrets Manager + KMS; never Lambda environment plaintext | rotation and least privilege |
| Infrastructure | AWS CDK or Terraform; separate dev/staging/prod accounts | reproducibility and blast-radius control |
| Observability | structured JSON logs, X-Ray/OpenTelemetry trace IDs, CloudWatch metrics/alarms | diagnose a disputed goal across API, DB and fan-out |

Keep all Lambdas and Aurora in a VPC only when they need database access. Use
VPC endpoints for Secrets Manager, EventBridge, SQS and CloudWatch to avoid NAT
costs. Provisioned concurrency is not needed initially; apply it only to the
live command handler during advertised tournaments after measuring p95 latency.

## Match lifecycle and write model

```text
DRAFT -> SCHEDULED -> READY -> LIVE <-> PAUSED -> COMPLETED -> FINALIZED
                                      |              |
                                      +-> ABANDONED -+
```

* `DRAFT`: organizer edits setup and rosters freely.
* `READY`: eligible logger has locked the roster. Last-minute roster changes are
  captured as auditable amendments.
* `LIVE`: only assigned scorer, referee, or organizer can append events.
* `PAUSED`: no events except `PERIOD_ENDED`, `MATCH_RESUMED`, corrections, or
  abandonment.
* `COMPLETED`: clock is stopped; privileged corrections remain possible.
* `FINALIZED`: result and stats are immutable to ordinary users; an organizer
  must explicitly reopen it to correct a material error.

`POST /matches/{matchId}/events` runs in a single database transaction:

1. Authorize the caller against organization membership and match assignment.
2. Lock the `match.live_state` row (`SELECT … FOR UPDATE`).
3. Check `expected_sequence` and deduplicate on `(match_id, client_event_id)`.
4. Validate the event against match status, period, active lineup, action
   definition and related participant.
5. Allocate `sequence = current_sequence + 1`, insert the immutable event and
   its actors/targets, update the match aggregate state, then insert an outbox
   record in the same transaction.
6. Commit. The caller receives the canonical event and updated snapshot; an
   asynchronous publisher pushes the result to all clients.

**Undo** calls the reversal endpoint. It writes `EVENT_REVERSED` (or a new event
whose `reverses_event_id` is set) and causes projections to subtract that event.
The service must never delete the original event or decrement a score counter
directly. Manual score arrows become `SCORE_ADJUSTMENT` events with a mandatory
reason and audit record; the logger UI should prefer a goal event.

### Offline-first logger protocol

The logger app persists each command in a local SQLite/outbox queue before
displaying it optimistically. On connectivity recovery it sends queued commands
in sequence. A rejected command remains visible as `needs_review`, together
with the latest authoritative event stream.

* Event identity is generated on device: `client_event_id` UUIDv7.
* Every HTTP mutation has `Idempotency-Key`; a 24-hour record preserves the
  response for retries.
* Send `expected_sequence` from the last accepted snapshot. On `409`, fetch
  events after the server sequence, rebase if possible, then ask a human to
  resolve ambiguous actions.
* If two people log a goal simultaneously, one succeeds, one receives `409`;
  clients do not silently merge two score-changing actions.
* Device timestamps are retained for diagnostics but match time is supplied by
  the active server-side live session/period clock, so a phone clock cannot
  corrupt the timeline.

## Data and projection design

The SQL schema in `database/core-schema.sql` is the physical reference. The
important split is:

* `match_data.match_events` is the append-only fact table.
* `match_data.match_event_subjects` allows scorer, assister, opponent, player
  off/on and other roles without widening the event table for each action.
* `match_data.match_live_state` is a transactional command-side aggregate, not
  an independently edited score table.
* `read_model.*` stores replaceable snapshots: score, timeline, player match
  stat lines and standings.

The projection worker consumes outbox events in sequence. It uses an inbox table
to make consumption idempotent and stores `last_event_sequence` per match. If a
consumer is delayed, it rebuilds that match's read models from the ledger;
projection lag must never block the scorer. The command response includes the
transactionally updated live score to make the logger responsive even during
projection lag.

### Database ownership and relationship map

| Schema / owner | Tables | Key relationships |
| --- | --- | --- |
| `identity` / Identity service | `organizations`, `users`, `organization_memberships`, `player_profiles` | organization 1:N memberships and player profiles; user 1:N memberships/profiles |
| `competition` / Competition service | `venues`, `teams`, `team_memberships`, `competitions`, `stages`, `competition_entries`, `fixtures` | team N:M player through team membership; competition 1:N stages/entries/fixtures |
| `match_data` / Match service | `matches`, `match_sides`, `match_participants`, `match_live_state`, `live_sessions`, `action_catalog`, `match_events`, `match_event_subjects`, `event_reversals` | match 1:2 sides; side 1:N participants; match 1:1 live state; match 1:N ordered events; event 1:N subjects; reversal 1:1 original event |
| `read_model` / Projection service | `consumer_inbox`, `match_snapshots`, `match_timeline_entries`, `player_match_stat_lines`, `team_match_stat_lines`, `competition_standing_rows` | all rows are materialized/rebuildable views keyed by the source match/competition IDs |
| `platform` / shared infrastructure | `idempotency_records`, `outbox_events`, `audit_log` | every mutation deduplicates, audit logs sensitive actions, and emits an outbox record atomically |

```text
organization --< player_profile        team --< team_membership >-- player_profile
     |                                      
     +--< competition --< fixture --(external match_id)--> match --< match_side
                                                               |          |
                                                           live_state   participant
                                                               |          |
                                                           event ledger -+--< event_subject
                                                               |
                                                      rebuildable score/timeline/stats
```

Cross-service IDs are UUID references without database foreign keys. This is
intentional: the owning service is the authority and can later move to its own
database. Foreign keys are enforced for relations within the same service schema.
The Match service verifies external IDs synchronously at setup time and records
display-name snapshots for historical correctness.

### Action taxonomy

The UI's labels are client presentation strings, not domain enums. Store a
versioned `action_catalog` record such as `GOAL`, `SHOT_ON_TARGET`,
`DUEL_WON`, `GOALKEEPER_SAVE`, `SUBSTITUTION`, `YELLOW_CARD`. Its metadata
specifies valid player roles, required subjects, whether it changes the score,
and a stats mapping. This makes it possible to add futsal-only actions or change
labels without breaking old matches. A persisted event stores both `action_code`
and `action_catalog_version` so historic interpretation never shifts.

For a goal, require `SCORER`; optionally `ASSISTER`, `GOALKEEPER` and
`OPPONENT`. `qualifiers` stores controlled details such as `finish_type`,
`body_part`, `x_pct/y_pct`, and `is_own_goal`. Validate its JSON schema in Go;
do not allow arbitrary user-controlled analytics fields.

### Privacy and authorization

Organization-scoped resources require an organization claim or membership lookup
on every request. Roles: `OWNER`, `ADMIN`, `ORGANIZER`, `TEAM_MANAGER`,
`SCORER`, `REFEREE`, `COACH`, `PLAYER`, `PARENT`, `FAN`.

Phone lookup may be used only by a manager/scorer within their organization and
returns a minimal profile after consent. Phone numbers are normalized to E.164,
encrypted at rest, and are never included in public match responses. Public
match feeds expose approved display names or shirt numbers. Treat medical data,
minor accounts and guardian consent as a later dedicated safeguarding domain;
they must never leak into public projections.

## API policy

* JSON only, UTF-8, ISO-8601 timestamps, UUIDv7 identifiers in new writes.
* Cursor pagination uses opaque `cursor`, never page/offset on growing feeds.
* All mutation requests require `Idempotency-Key`; event append also requires
  `expected_sequence` in its body.
* Errors use RFC 9457 Problem Details. `401` means no/invalid token; `403` means
  valid token without organization/match permission; `409` means a rebase is
  required; `422` means structurally valid but domain-invalid input.
* Public `GET /public/matches/{matchId}` is cacheable for five seconds when
  `LIVE` and longer after finalization. It never exposes private roster fields.
* WebSockets communicate notifications, not authoritative mutations. A client
  that misses a notification reads the HTTP snapshot/events endpoint.

## Reliability, security, and operations

### Targets

| SLO | Target | Measurement |
| --- | --- | --- |
| live event write availability | 99.9% monthly | successful non-4xx event commands / eligible commands |
| event write p95 | < 350 ms regional | API Gateway to transaction commit |
| public score freshness p95 | < 2 s | event commit to WebSocket/snapshot update |
| RPO / RTO | <= 5 min / <= 60 min | Aurora PITR plus documented restore exercise |

### Required controls

* WAF rate limits public endpoints; API Gateway throttles write routes per user
  and match. Apply a stricter phone-lookup limit.
* Store authorization/audit records for roster changes, event reversal, manual
  adjustment, reopening/finalization and role changes.
* Encrypt Aurora and S3 with KMS; TLS everywhere; scoped IAM roles per Lambda.
* SQS retries use exponential backoff and DLQs. Alarms: outbox age, DLQ depth,
  projection lag, Lambda error rate, DB connections, authorization failures,
  and a live match with stale update sequence.
* Back up schema migrations and rehearse restoring a single finalized match and
  its event ledger. Never rely on a projection as the backup.

## Delivery sequence

1. Establish Cognito, organizations/memberships, teams, players, venues and
   match setup. Implement the schema migrations and contract tests first.
2. Ship Match service with a single scorer, event ledger, goal/substitution
   actions, reversal, HTTP snapshot and idempotency/concurrency checks.
3. Add projection worker, public timeline, player match stats and WebSocket
   notifications. Load test concurrent writes and replay a full match ledger.
4. Add competition fixture generation and standings as an asynchronous
   projection. Only then introduce named opponent linking, richer action
   taxonomy and pitch coordinates.
5. Extract physical databases only when a service has materially different
   scaling, release cadence or data-retention needs. The contracts/outbox are
   intentionally designed so this extraction does not change clients.

## Explicit non-decisions

* Do not use DynamoDB as the live event ledger: multi-row relational roster and
  lineup invariants benefit from PostgreSQL transactions and constraints.
* Do not use a synchronous chain of Lambda calls for projections/notifications:
  it couples match recording to optional downstream work.
* Do not calculate career statistics from mutable totals alone: retain the
  event facts so future player-card formulas can be recalculated.
* Do not expose EventBridge directly to mobile clients: API contracts, auth,
  pagination and replay are required.
