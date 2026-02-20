# MVP

## User & Issue

**User:** The casual arcade player (Lucas, 20) who wants an immediate, frictionless experience and is looking for a tense, evolving challenge.
**Issue:** Traditional arcade cabinets are solitary and passive experiences — you play, you lose, you leave. There is no physical pinball machine that integrates a true real-time combat system, which deprives players of a replayable progression dimension.

## Spec

1. A first board layout with its flipper components that are controllable.
2. The 3D physics work well enough to be able to play the game.
3. The score / life system works.
4. The back screen and the DMD display a light UI that is synced with the game state.
5. One playable character with one spell to cast and one boss to fight.

## Success Criteria

1. A complete PvE session playable from start to finish without any crash or blocking bug.
2. Perceived latency < 100ms on flipper inputs in local play.
3. Onboarding time < 3 minutes for a new player (tested on random users).

## Constraints

We have limited access to the real machine. We do not yet know on which device the game will be run, so we cannot properly test the real environment. We need low latency to make the game feel good and to have a solid foundation before starting the PvP feature. All screens must be in sync. A large number of 3D assets need to be produced.

## Risks & Fallbacks

**Risk 1 — Ball physics**
Miscalibrated Rapier.js results in a game that feels bad, not one that crashes. Wrong bounces, laggy flippers — hard to fix late.
→ Plan B: a `physics.config.ts` file from S3, tunable without touching the code.

**Risk 2 — Latency**
The S3–S4 POC is local. It does not validate the GCP + 2 remote cabinets scenario.
→ Plan B: remote WS test from S5. If p95 > 80ms, switch to client-side prediction.

**Risk 3 — Ability balancing**
5 characters land in S9–S12, too late to iterate. A broken ability during the demo = bad impression.
→ Plan B: freeze risky abilities (invisible, black hole) out of demo scope — only ship what is testable in under 1 hour of playtesting.
