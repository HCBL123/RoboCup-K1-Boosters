---
name: architecture-notes
description: Non-obvious architectural constraints and findings that affect strategy changes
metadata:
  type: project
---

## Blackboard init — goalie_mode
`goalie_mode` is initialized to `"attack"` in `BrainTree::initEntry()` (brain_tree.cpp:123). No transition to `"guard"` exists. The guard branch in `subtree_goal_keeper_play.xml:49` is dead code. See [[known-bugs]].

## safe_shoot decision is latent dead-end
`StrikerDecide` can emit `"safe_shoot"` but no XML handler exists. Currently safe because `enable_shoot: false` in config. See [[known-bugs]].

## SelfLocate cascade order
`subtree_locate.xml` runs: SelfLocate(trust_direction) → SelfLocate1M → SelfLocate2T → SelfLocatePT → SelfLocateLT → SelfLocate2X → SelfLocateBorder. Each runs only if previous failed. `SelfLocate2X` is near the end — it fires only when earlier methods fail. The dy bug in SelfLocate2X affects this rare-but-important fallback path.

## Robot memory vs obstacle memory timeout mismatch
`updateRobotMemory()` (brain.cpp:718): hardcoded 1000ms cutoff.
Obstacle memory: configurable `obstacle_memory_msecs: 500ms`.
Robots linger 2x longer than obstacles in memory — creates ghost opponent artifacts.

## Cost EMA speed
`data->tmMyCost = lastCost * 0.8 + cost * 0.2` (brain.cpp:887).
Alpha = 0.2 → ~4-5 ticks (~130-167ms at 30Hz) to react to a step change.
Fall event adds +15 to cost. With EMA, full impact takes ~10+ ticks to propagate through, so leadership transfer after a fall is delayed ~300-500ms. Acceptable but worth noting for multi-robot tuning.

## KICKOFF_DURATION
`handleSpecialStates()` uses `KICKOFF_DURATION = 10.0` seconds for kickoff window. This is a hardcoded constant in brain.cpp (not configurable via config.yaml).

## COM_TIMEOUT vs TEAMMATE_TIMEOUT_MS gap
- `COM_TIMEOUT = 5000ms` in brain.cpp — robot declared "dead" for cooperation after 5s no message
- `TEAMMATE_TIMEOUT_MS = 20000ms` in brain_communication.h — address table entry survives 20s
- 15s window where unicast sends to brain-declared-dead robot. Wasted UDP traffic but not a correctness issue.

## d-robotics camera config
Currently using d-robotics camera (544x448, FOV 69.4°x42.5°). RealSense config is commented out. Do not mix camera configs.

## enable_auto_visual_kick: true
RL+vision kick is enabled and fires when decision == 'auto_visual_kick'. The Kick node for 'cross' uses `speed_limit="0.6"` (softer than regular kick's 0.9). RLVisionKick node uses `min_msec_kick="1500" max_msec_kick="7000" range="6.0"`.
