---
name: known-bugs
description: Confirmed bugs in K1 Football Demo brain package identified during code analysis — not yet fixed
metadata:
  type: project
---

## Bug 1: SelfLocate2X dy copy-paste error
**File**: `src/brain/src/brain_tree.cpp:2432`
**Severity**: High — corrupts localization correction in "double X marker" mode

```cpp
// BUG — p1 used twice; should be p0 + p1
double dy = - (p1.posToField.y + p1.posToField.y) / 2.0;
```

Fix: change to `(p0.posToField.y + p1.posToField.y) / 2.0`

The same bug was previously identified and fixed in `SelfLocateLocal::_doubleX()` at line 2107 (comment there confirms it). `SelfLocate2X` got a copy of the broken version.

**Why critical**: Robot using 2X localization mode gets wrong y-correction — always uses p1's y twice instead of averaging p0+p1. Causes positional drift in y-axis during localization updates from symmetric field markers. No one noticed because `SelfLocate2X` fires late in the localization cascade.

## Bug 2: goalie_mode 'guard' branch is dead code
**File**: `src/brain/behavior_trees/subtrees/subtree_goal_keeper_play.xml:49`
**Severity**: High — entire guard mode logic never executes

`goalie_mode` blackboard key initialized to `"attack"` in `BrainTree::initEntry()` (brain_tree.cpp:123). `Intercept::onRunning()` resets it back to `"attack"` at lines 1552 and 1593. **No code anywhere sets it to `"guard"`.**

The `goalie_mode == 'guard'` ReactiveSequence at XML line 49 is unreachable.

Fix: Either (a) remove the dead branch and keep only "attack" mode, or (b) add logic in `GoalieDecide` to set goalie_mode to "guard" when ball is distant/safe and keeper should hold position.

## Bug 3: safe_shoot decision has no XML handler
**File**: `src/brain/src/brain_tree.cpp:1077` + `subtree_striker_play.xml`
**Severity**: Medium (currently latent — `enable_shoot: false`)

`StrikerDecide` emits `newDecision = "safe_shoot"` when `enableShoot == true && threatLevel < threatThreshold(-2.0)`. No `_while="decision=='safe_shoot'"` handler exists in any behavior tree XML. If `enable_shoot: true` is ever set, the robot will silently do nothing when a safe-shoot opportunity arises.

Fix: Either add a `<Kick _while="decision=='safe_shoot'" .../>` entry in `subtree_striker_play.xml` (treat as regular kick), or rename the decision to `"kick"` in `StrikerDecide` for the safe-shoot path.

**Why:** **How to apply:** Verify both these bugs are still open before proposing fixes — check git log for any changes to brain_tree.cpp and subtree_goal_keeper_play.xml. See [[applied-improvements-p1-p7]] for the convention for annotating applied fixes.
