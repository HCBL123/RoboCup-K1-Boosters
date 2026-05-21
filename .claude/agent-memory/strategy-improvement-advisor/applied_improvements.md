---
name: applied-improvements-p1-p7
description: Improvements P1-P7 already applied to K1 Football Demo codebase; baseline for future proposals
metadata:
  type: project
---

The following improvements are already active in the codebase (visible via commented-out old values in config.yaml and XML files). Do NOT re-propose these.

| ID | Change | File | Value |
|----|--------|------|-------|
| P1 | `kick_range` 1.0→0.7, `chase_threshold` 1.0→1.2 | config.yaml + subtree_striker_play.xml | Creates clean 0.5m adjust band (0.7–1.2m), eliminates kick/adjust oscillation |
| P2 | `near_ball_speed_limit` 0.6→0.45, `near_ball_range` 2.0→1.2, `limit_near_ball_speed: true` | config.yaml | Slower final approach avoids ball overshoot |
| P3 | `auto_visual_kick_enable_dist_max` 4.0→2.5, `auto_visual_kick_enable_angle` 0.8→0.5 | config.yaml | RL kick fires only from mid-field forward with better alignment |
| P4 | `enable_stable_kick: true`, `kick_theta_range` 0.2→0.15 | config.yaml | Stabilizes 1s before kick when no opponent pressure; tighter angle gate |
| P5 | `ball_control_cost_threshold` 5.0→3.5 | config.yaml | Narrower role-switch window, prevents oscillation when both robots near threshold |
| P6 | `min_marker_count` 5→4, `max_residual` 0.35→0.25 | config.yaml | More localization attempts in corners; tighter quality gate |
| P7 | `turn_first_threshold` 0.5→0.35 in Adjust | subtree_striker_play.xml | Earlier lateral arc during adjust, faster repositioning under pressure |

**Convention**: All changes preserve old value commented out with annotation (e.g., `// old value — reason` then new value).

## FindBall Subsystem Analysis (Session: 2026-05-21)

Key structural facts confirmed from code:
- `ball.posToField` persists past `ball_memory_timeout`; only `ball_location_known` flag is cleared (brain.cpp:684).
- `updateRelativePos(data->ball)` called unconditionally every tick at brain.cpp:689 — `yawToRobot` stays fresh from last known field pos even when ball is not detected.
- `TurnOnSpot` is used ONLY in `subtree_find_ball.xml` — safe to add new port without breaking other trees.
- `GoToBallLastPos stale_max_msec=5000ms` but preceding steps (CamFastScan×2 + TurnOnSpot) consume ~5-7s — node nearly always fails due to timing.
- `GoToFindBallFallback` uses `myStrikerIDRank` for zone split — adaptive fallback zone proposals must preserve this split.
- Teammate ball (`tm_ball_pos_reliable`) is NOT usable in 2v2 mixed-team competition — teammate runs different codebase.

**Why:** Track these to avoid duplicate proposals. **How to apply:** Any analysis session should check these are still in place before proposing P1-P7 equivalent changes. Next improvement proposals should be numbered P8+.
