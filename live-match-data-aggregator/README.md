# Live Sports Match Data Aggregator

## Overview

FanCode displays live match scores, match status, and ball-by-ball commentary. Match data comes from third-party sports providers. Those providers retry deliveries, delay them, repeat them, reorder them, and sometimes correct a fact they already sent.

Build a backend that ingests one live cricket match from two kinds of provider and serves one set of read APIs. You choose the language, framework, database, and architecture. A single local process with one database is a complete solution for this assignment.

**Time budget.** Submit within 2–3 calendar days. The implementation is sized at about 8–10 hours. A smaller solution that stores the right state and explains its limits is a better submission than a larger one that mishandles ordering, duplicates, or restarts.

You may use libraries and coding tools you are comfortable with. A reviewer will ask you to explain and modify what you submit, so you should understand the code you hand in.

## Cricket, only as far as this assignment needs

You are not building a cricket scoring engine. Do not compute runs, wickets, or overs by interpreting commentary.

- A score of `98/2` means 98 runs and 2 wickets, as stated by the provider snapshot.
- `overs` is an opaque string from the provider, such as `"12.4"`. Store and return it unchanged.
- `over` and `ball` are descriptive integers. Do not validate them against `overs`.
- Each ball has a stable id. A provider may later correct the commentary for that ball.

## What you will build

No starter repository, fixtures, or mock services are provided. You will build all of the following.

1. A mock push provider that sends webhook events to your aggregator.
2. A mock polling provider with HTTP endpoints and an evolving match.
3. An aggregation service that normalizes and stores what both providers send.
4. Persistent storage.
5. Client-facing read APIs.
6. Automated tests for the cases listed below.
7. A `README` and a `DESIGN.md`.

No authentication, user accounts, frontend, or cloud deployment is required. A reviewer must be able to run the project on one machine from the README.

## The match

Ingest at least one cricket match, available from both providers:

| Role | Identifier |
|---|---|
| Provider A match | `a-match-1001` |
| Provider B match | `b-match-1001` |

You choose the match id that clients see. Record how each provider match id maps to it. Configuration or seed data is enough.

Status values used below: `scheduled`, `live`, `innings_break`, `completed`, `abandoned`.

### Score snapshots

A snapshot is the provider's current scoreboard, not a delta.

| Field | Meaning |
|---|---|
| `sequence` | That provider's publication order for this match. A greater value is newer. Provider A and Provider B number their sequences independently. Those numbers are not comparable across providers. |
| `revision` | Correction counter for this snapshot `sequence`. Starts at `1`. |
| `status` | Match status. |
| `innings` | Innings number, starting at `1`. |
| `runs` | Runs in the snapshot. |
| `wickets` | Wickets in the snapshot. |
| `overs` | Provider string such as `"12.4"`. |

From a single provider, snapshot `(sequence, revision)` is **strictly newer** when it is lexicographically greater, comparing `sequence` first.

- `(5, 1)` is newer than `(4, 9)`.
- `(4, 2)` is newer than `(4, 1)`.
- `(4, 1)` is not newer than `(4, 1)`.

Apply a snapshot only when it is strictly newer than the snapshot you have stored for that same provider. This includes a delayed message, a retried message, and a poll response.

### Commentary items

| Field | Meaning |
|---|---|
| `ballId` | Stable id of one logical ball from that provider. A correction keeps the same `ballId`. |
| `sequence` | Display order for that provider's balls. |
| `revision` | Starts at `1`. A greater revision replaces the stored text for that `ballId`. An equal or lower revision leaves the stored text as it is. |
| `innings`, `over`, `ball` | Descriptive fields. Store them with the item. |
| `text` | Commentary string. Store it as given. |

Client-facing commentary is ordered by commentary `sequence`, not by arrival time. A missing sequence is allowed. Serve a newer score immediately, and insert the late ball when it arrives. Do not delay the score until the gap is filled.

## Provider A — push

Build a generator that POSTs webhook events to your aggregator. You choose the webhook path. The generator must accept that URL as configuration.

Representative event:

```json
{
  "provider": "provider-a",
  "deliveryId": "d-1042",
  "occurredAt": "2026-10-22T14:03:11.000Z",
  "providerMatchId": "a-match-1001",
  "snapshot": {
    "sequence": 42,
    "revision": 1,
    "status": "live",
    "innings": 1,
    "runs": 98,
    "wickets": 2,
    "overs": "12.4"
  },
  "commentary": {
    "ballId": "inn1-12.4",
    "sequence": 42,
    "revision": 1,
    "innings": 1,
    "over": 12,
    "ball": 4,
    "text": "Full and straight, driven to long-on for four."
  }
}
```

`snapshot` or `commentary` may be absent. A status change may send only a snapshot. A commentary correction may send only commentary.

| Field | Contract |
|---|---|
| `deliveryId` | Identifies the message. A retry uses the same `deliveryId`. Processing the same `provider` and `deliveryId` again must leave stored state unchanged. |
| `occurredAt` | Provider event time. It may be shown to clients or used to reason about lag. It does not decide which snapshot or revision wins. |
| Correction | A new `deliveryId`, the same `ballId`, and a higher commentary `revision`. |

The generator must deterministically produce all of the following:

- ordered delivery
- a repeated `deliveryId`
- out-of-order delivery, where a higher commentary sequence arrives before a lower one
- delayed delivery of an older snapshot
- a correction of an earlier ball

Named scenarios, a fixture, or a scripted timeline are all acceptable. The graded scenarios must not depend on randomness.

### Worked push example

Score notation below is `runs/wickets`.

| Step | Arrival | Score afterwards | Commentary afterwards |
|---|---|---|---|
| 1 | `d1`, snapshot `(1, 1)` = 4/0 in `0.4`, ball `b1` rev 1, text "four" | 4/0 | b1 |
| 2 | `d2`, snapshot `(2, 1)` = 4/1 in `0.5`, ball `b2` rev 1, text "out" | 4/1 | b1, b2 |
| 3 | `d2` again | 4/1 | unchanged |
| 4 | `d4`, snapshot `(4, 1)` = 10/1 in `1.1`, ball `b4` | 10/1 | b1, b2, b4 |
| 5 | `d3`, snapshot `(3, 1)` = 8/1 in `1.0`, ball `b3` | 10/1 | b1, b2, b3, b4 |
| 6 | `d5`, snapshot `(4, 1)` repeated, ball `b2` rev 2, text "out, review stays out" | 10/1 | one b2, text replaced |

Step 5 keeps the newer score and still stores the late ball. Step 6 replaces commentary because revision `2` is newer, and leaves the score because snapshot `(4, 1)` is not strictly newer. A genuine score correction is a higher snapshot revision, such as `(4, 2)`, and that snapshot must be applied.

Document what your webhook response means, including what a provider should do if your process crashes around the acknowledgement.

## Provider B — poll

Expose these endpoints from the mock provider:

- `GET /provider/matches/:id/score`
- `GET /provider/matches/:id/commentary`

Score response:

```json
{
  "provider": "provider-b",
  "providerMatchId": "b-match-1001",
  "snapshot": {
    "sequence": 15,
    "revision": 1,
    "status": "live",
    "innings": 1,
    "runs": 98,
    "wickets": 2,
    "overs": "12.4",
    "asOf": "2026-10-22T14:03:11.000Z"
  }
}
```

Commentary response:

```json
{
  "provider": "provider-b",
  "providerMatchId": "b-match-1001",
  "items": [
    {
      "ballId": "inn1-12.4",
      "sequence": 76,
      "revision": 1,
      "innings": 1,
      "over": 12,
      "ball": 4,
      "text": "Full and straight, driven to long-on for four.",
      "occurredAt": "2026-10-22T14:03:11.000Z"
    }
  ]
}
```

Returning the full commentary list each time is enough. A cursor is optional.

Use the same snapshot and revision rules as push. Polling an unchanged body must not create another copy of a ball. `asOf` and `occurredAt` do not make an older tuple newer. A stale response can carry a recent timestamp and must still lose to a newer stored tuple.

The mock must deterministically support:

- an unchanged response
- a newer score
- a new commentary item
- a stale response whose snapshot tuple is older than one already served
- a temporary error (HTTP `503` is a suitable choice)

You choose the poll interval. A provider error must leave the last good stored match available to clients. Automatically switching the client API to the other provider is optional.

## Client API

These routes are a suggestion. Ship an equivalent design if you prefer, and document the real paths.

| Endpoint | Response |
|---|---|
| `GET /matches` | Matches the service knows, with client id, status, and a summary score once one exists. |
| `GET /matches/:id` | One match, including current status and score. |
| `GET /matches/:id/score` | Current status, innings, runs, wickets, and overs. |
| `GET /matches/:id/commentary` | One entry per logical ball, in sequence order, using the newest applied revision and its text. |

An unknown match id is a client error. Commentary may be empty before any ball is stored. Additional endpoints are fine when `DESIGN.md` explains why they exist. Include curl examples for the API you actually implement.

## One client view from two providers

Persist what both providers send, in a common internal model. Your field names may differ from the payloads above.

The read API returns one score and one commentary list. Choose a deterministic precedence rule for disagreements and implement that rule. A configured primary provider is a valid rule. Sequence numbers must not be compared across providers.

During a primary-provider error, serving the last good score from that provider is acceptable. Automatic failover may be described in `DESIGN.md` and left unimplemented.

If two updates for the same match are processed concurrently, the stored result must still match the snapshot, revision, and duplicate rules.

Restarting the process against the same database must leave previously stored score, status, and commentary available through the read API. Surviving a wiped database is unnecessary. More than one running instance is unnecessary. If your correctness assumes a single process, say that in `DESIGN.md`.

## Deliverables

- Runnable source for the aggregator, both mocks, and tests.
- `README` with commands to start each component, run the tests, and exercise a default match locally.
- `DESIGN.md` of about one to three pages, covering:
  - database choice
  - data model
  - concurrency
  - idempotency
  - differences between your push path and your poll path
  - provider precedence and conflict handling
  - failure recovery, including crash and provider errors
  - caching
  - the first bottleneck you expect
  - what you would change for hundreds of concurrent live matches and heavy read traffic
- Tests that cover:
  - ordered ingestion
  - duplicate push delivery
  - an older snapshot arriving after a newer one, with the late ball still returned in sequence order
  - a commentary correction
  - repeated identical polls
  - a stale poll
  - a provider error leaving the last good state in place
  - a restart against the same database

## Scope

**Build this.** Both ingestion paths, durable storage, the read API, deterministic mocks, duplicate handling, the snapshot and revision rules, one documented precedence rule, and the tests above.

**Discuss this in `DESIGN.md` unless you have time left.** Automatic failover, long chains of historical corrections, multiple instances, cache behavior under load, fan-out to large numbers of subscribers, production observability, and any deployment beyond local processes.

Reviewers will use `DESIGN.md` to understand your choices and the limits of the system you built. Extra infrastructure is optional and is not an advantage on its own.

Write any further assumptions in `DESIGN.md`. Reviewers will use your README only, on a local machine, with no shared environment and no provider credentials.
