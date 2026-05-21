---
name: arch-key-logic
description: Critical code paths for lead/cost logic, RLVisionKick x-guard, Adjust node, and threatLevel — verified locations
metadata:
  type: feedback
---

## Lead/Cost Logic (brain.cpp lines 530-622)
- `BALL_CONTROL_COST_THRESHOLD` gates whether a teammate's cost qualifies them as a candidate leader, NOT whether the cost differential is large enough.
- Condition: if any teammate has `cost < threshold` AND that cost is less than mine (`tmMyCost > tmMinCost`), I lose lead.
- Raising the threshold WIDENS transfers (more teammates qualify), does NOT add hysteresis. True oscillation fix requires differential hysteresis logic.
- `myCostRank >= 2` also forces non-lead regardless of threshold.

## RLVisionKick x-guard bug (brain_tree.cpp line 1038)
- Condition: `ball.posToField.x > brain->config->fieldDimensions.length / 2 - 14.3`
- For adult_size field (length=14.16m): `7.08 - 14.3 = -7.22` — always true since field boundary is -7.08.
- This x-guard was meant to prevent RLVisionKick from own half but is completely ineffective.
- Proposed fix: `ball.posToField.x > -1.0` but this still allows RL kick at kickoff (x=0). Should also gate on `!brain->data->isFreekickKickingOff` if kickoff RL kick is undesired.

## Adjust node near-ball speed (brain_tree.cpp lines 804-865)
- Adjust node does NOT read `limitNearBallSpeed` or `nearBallSpeedLimit`. The near-ball speed taper in Chase only applies to the Chase node.
- With P1's chase_threshold=1.2 and P2's near_ball_range=1.5, taper window during Chase is only 1.2-1.5m (0.3m band).
- Most approach overshoot happens inside Adjust (orbiting), not Chase.

## threatLevel() (brain.cpp lines 3181-3196)
- Returns 0.0 when nearest opponent > 2.5m, 1.0 when 1.0-2.5m, 2.0 when < 1.0m.
- enableStableKick gate: `threatLevel() < 0.5` — means stable kick only fires when nearest opponent is > 2.5m away.

## Kick abort on range (brain_tree.cpp lines 1273-1280)
- Kick::onRunning aborts if `ballRange > KICK_RANGE`. With kick_range=0.7 (P1), any ball drift beyond 0.7m mid-kick aborts the action.
- K1 crabWalk may push ball forward before foot contact — abort-restart cycles possible.

## Field dimensions (types.h lines 32-34)
- adult_size: length=14.16, width=9.22
- kid_size: length=9, width=6

## Striker subtree XML (subtree_striker_play.xml line 31, 42)
- chase_threshold attribute currently hardcoded "1.0" on StrikerDecide node
- turn_first_threshold currently "0.5" on Adjust node
