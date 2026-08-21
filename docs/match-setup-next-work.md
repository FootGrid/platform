# Match setup: next work

## Purpose

Turn `platform/match-setup.html` from a visual prototype into the first
reliable setup slice described by the core implementation plan:

1. An organizer creates a draft match.
2. Both sides have valid match participants.
3. Initial lineups are saved.
4. The match becomes `READY` only after setup validation succeeds.

This document is intentionally limited to match setup. Live event capture,
projections, WebSockets, and Cognito production wiring remain later work.

## Current state

### Page behavior

- Format selection changes the displayed duration and position labels.
- The roster is an in-memory home-only mock list.
- “Add by mobile number” removes a mock entry from a local pool; it does not
  perform phone lookup.
- The away-team chip is decorative and has no roster state.
- Shirt numbers and participant positions are generated locally.
- “Kick off” has no click handler and never sends a request.

### Backend behavior

- `footgrid/api/openapi.yaml` defines the required setup calls:
  `POST /matches`, `PUT /matches/{matchId}/roster`, and
  `PUT /matches/{matchId}/lineups`.
- `footgrid/internal/match/repository.go` defines the corresponding repository
  operations and validates match duration, format dimensions, and team names.
- `footgrid/internal/match/domain.go` requires every roster participant to have
  an ID, display name, valid shirt number, and a side-specific roster count.
- `footgrid/cmd/match-api/main.go` currently registers only `GET /health`.
- No HTTP handler, repository implementation, or setup integration test exists
  yet for these routes.

## Next smallest slice

Implement a **draft creation flow** first. Do not make the button start a live
session yet.

### Slice 1: make the page's state API-ready

- Replace mock roster entries with objects containing stable participant IDs.
- Maintain separate `homeSquad` and `awaySquad` collections.
- Make the side chips switch the active roster; both sides must be editable.
- Keep shirt numbers unique within each side and validate them before submit.
- Convert the UI values to the API shape:
  - `5v5`, `6v6`, `8v8`, `11v11` -> `5V5`, `6V6`, `8V8`, `11V11`.
  - Match length in minutes -> `total_duration_seconds`.
  - Halves -> `period_count`.
  - Team inputs -> `home.display_name` and `away.display_name`.
  - Add the configured organization UUID through page configuration; do not
    invent one in the browser.
- Disable submission until both sides have at least the selected number of
  players, or show a clear validation state explaining what is missing.

### Slice 2: create the draft match

- Add a small `fetch` client in the page for `POST /v1/matches`.
- Send an `Idempotency-Key` for the mutation and retain it across a retry of
  the same submit attempt.
- Handle `201`, `401/403`, `409`, `422`, network failure, and malformed JSON
  without losing the form state.
- On success, store the returned match ID and status and move to roster save.
- Keep the API base URL and organization ID configurable rather than embedding
  an environment-specific endpoint in the HTML.

### Slice 3: save roster and initial lineups

- Build a `Roster` payload from both collections, including `id`, `side`,
  `shirt_number`, `display_name`, `position_code`, and
  `participation_status`.
- Call `PUT /v1/matches/{matchId}/roster` with an idempotency key.
- Select starters explicitly. For the first slice, the first N players per side
  may be the default, but the UI must show starter versus bench state.
- Call `PUT /v1/matches/{matchId}/lineups` with the participant IDs for both
  sides.
- Do not send the lineups until the roster call succeeds.

### Slice 4: finish setup as `READY`

- Add the missing backend command/route that transitions a valid draft to
  `READY`, or document the exact existing route if it is added first.
- Require authorization and an idempotency key for the transition.
- Make the final button action explicit in the UI, for example “Save and ready”;
  reserve “Kick off” for the later live-session flow.
- On success, show the returned `READY` state and the match ID needed by the
  lineup/logger screens.

## Backend implementation order

1. Add a concrete match repository backed by the existing `match_data` tables.
2. Add request/response DTOs and handlers under the match API boundary.
3. Register the three OpenAPI setup routes in `cmd/match-api/main.go`.
4. Add the `READY` transition with a transaction that verifies both rosters and
   initial lineups.
5. Apply organization authorization at the handler/service boundary; the page
   must not be trusted to choose an organization it cannot manage.
6. Add idempotency handling for each mutation before exposing the flow to the
   browser.

## Cheap discriminating checks

These checks should be completed before adding live-session behavior:

- With 6v6 selected and one side short, submit is blocked and no request is
  made.
- A valid 6v6 form produces exactly two side arrays and seconds, not minutes,
  in the create payload.
- Repeating the same create request with the same `Idempotency-Key` returns the
  original response rather than creating a second match.
- A roster containing a missing ID, duplicate shirt number, or fewer than six
  participants per side is rejected with `422`.
- A successful setup sequence is ordered `POST /matches` -> roster `PUT` ->
  lineups `PUT` -> `READY`; a failed step leaves the match in `DRAFT`.
- A request without organization permission returns `403` and does not write
  match data.

## Definition of done for this page

- The page can create one valid 6v6 draft using real API responses.
- Both home and away rosters are represented by match-scoped participant IDs.
- Starter IDs sent to the API exist in the saved roster.
- Refreshing or retrying after a network failure does not create duplicates.
- The page reaches `READY` only after roster and lineup persistence succeeds.
- The live-session endpoint is not called by this work.

## Deferred work

- Phone lookup and consent-aware identity resolution.
- Team/venue selectors backed by Competition service IDs.
- Anonymous opposition participants and guest creation UX.
- Pitch-slot selection and substitutions.
- `POST /matches/{matchId}/live-session` and the logger handoff.
- Offline queueing, projections, and realtime notifications.

## Source references

- `platform/match-setup.html`
- `platform/docs/engineering-architecture.md`
- `footgrid/docs/implementation-plan.md`
- `footgrid/api/openapi.yaml`
- `footgrid/internal/match/domain.go`
- `footgrid/internal/match/repository.go`
- `footgrid/cmd/match-api/main.go`
- `footgrid/migrations/000001_core.up.sql`