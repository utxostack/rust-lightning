# WASM Adaptations Guide

**Branch**: [`wasm-adapt`](https://github.com/utxostack/rust-lightning/branches?query=wasm-adapt)  

## Core Modifications

The essence of Wasm adaptation, in this context, is to resolve the underlying implementation of the `now()` function for different environments.

```
Get Time (`.now()`)
│
│
├─► Test Environment (`#[cfg(test)]`)
│   │
│   └─ Instant::now() ───── [Mock Clock]
│                          └─ Behavior: Returns a virtual time point controlled manually by test code. (Provided by LDK)
│
│
└─► Production Environment (`#[cfg(not(test))]`)
    │
    │
    ├─► No-Standard-Library (`#[cfg(not(feature = "std"))]`)
    │   │
    │   └─ SystemTime Equivalent ── [Internal Estimation]
    │                             └─ Behavior: Uses a known timestamp as a fallback when no OS clock is available. (Provided by LDK)
    │
    │
    └─► With-Standard-Library (`#[cfg(feature = "std")]`)
        │
        │
        ├─► Native Environment (`#[cfg(not(target_arch = "wasm32"))]`)
        │   │
        │   ├─ SystemTime::now() ── [std::time]
        │   │                      └─ Underlying Mechanism: Direct OS API call (for wall-clock time). (Provided by LDK)
        │   │
        │   └─ Instant::now() ───── [std::time]
        │                          └─ Underlying Mechanism: Direct OS API call (for monotonic time). (Provided by LDK)
        │
        │
        └─► Browser Environment (`#[cfg(target_arch = "wasm32")]`)
            │
            ├─ SystemTime::now() ── [web_time Crate]
            │                      └─ Underlying Mechanism: Calls JavaScript `Date.now()`. (Requires `web_time` dependency)
            │
            └─ Instant::now() ───── [web_time Crate]
                                   └─ Underlying Mechanism: Calls JavaScript `performance.now()`. (Requires `web_time` dependency)
```

```diff
- use std::time::{SystemTime, UNIX_EPOCH, Instant, Duration};
+ use lightning_common::{SystemTime, UNIX_EPOCH, Instant, Duration};
```

## Scope

- Applies to all source files ​**except** test modules
- Centralized in [`lightning-common`](./lightning-common/src/lib.rs) crate

---

## Maintenance Workflow

This workflow ensures that the Wasm adaptations are cleanly applied to new upstream versions of LDK, resulting in a single, consolidated patch. This is superior to git rebase when histories have diverged.

Let's assume you are upgrading to a new upstream version, v0.1.4.

### 1. Create a New Branch from Upstream

   Start from a clean slate by checking out the official upstream version tag. This ensures you're only building on top of the official LDK code.

   ```bash
   # Fetch the latest tags and branches from the official LDK repository
   git fetch upstream --tags

   # Create a new, versioned branch based on the target upstream tag (e.g., v0.1.4)
   git checkout -b wasm-adapt-v0.1.4 upstream/v0.1.4
   ```

### 2. Apply the Core Adaptation Patch

   Use cherry-pick to apply your single, consolidated Wasm adaptation commit from your previous branch onto the new one.

   ```bash
   # First, find the commit hash of your Wasm adaptation patch from your old branch
   git log wasm-adapt

   # Cherry-pick that single commit. This will apply your core changes.
   git cherry-pick <your-single-wasm-adaptation-commit-hash>
   ```

   Note: You may encounter merge conflicts during this step if upstream files have changed significantly. Resolve these conflicts as needed before proceeding.

### 3. Audit and Update for New Code

   The cherry-pick only applies previous changes. You must now manually audit the codebase for any new usages of platform-dependent time APIs introduced in the upstream version.

   The primary targets for replacement are `SystemTime` and `Instant`.

   Rules for Replacement
   Review each search result and apply the following rules to determine if a replacement is necessary:

   ✅ YES, replace these types when found in production code:

   - `std::time::SystemTime` → `lightning_common::SystemTime`
      - Note: Any std::time::UNIX_EPOCH used in the same expression should be updated as a consequence.
   - `std::time::Instant` → `lightning_common::Instant`

   ❌ NO, ignore these cases:

   - `std::time::Duration`: This type is platform-agnostic (it comes from `core::time::Duration`) and should not be replaced.
   - Code within test modules: Any findings inside a `#[cfg(test)]` block or in files under a `tests/` directory should be ignored. The adaptation applies to production code only.

### 4. Consolidate All Changes into One Commit

   This is the key step. After making your manual edits, "amend" the cherry-picked commit to fold your new changes into it. This keeps the history clean with a single, comprehensive adaptation commit.

   ```bash
   # Stage all the manual fixes you just made
   git add .

   # Amend the previous commit, adding your changes to it without changing the commit message
   git commit --amend --no-edit
   ```

### 5. Apply Documentation Changes

   Cherry-pick the commit containing this guide to ensure the documentation on your new branch is also up-to-date. This keeps the explanation of the process alongside the code it describes.

   ```bash
   # Find the commit hash for this documentation file on your previous branch
   git log wasm-adapt-v0.0.125 -- 'WASM-ADAPTATIONS.md'

   # Cherry-pick the documentation commit. It will appear as a new commit.
   git cherry-pick <documentation-commit-hash>
   ```

### 6. Tag the Final Version

   Once all code and documentation changes are finalized and committed, tag the branch's HEAD to mark a stable, distributable release point.

   ```bash
   # Tag the final commit with a corresponding wasm version
   git tag wasm-v0.1.4
   ```