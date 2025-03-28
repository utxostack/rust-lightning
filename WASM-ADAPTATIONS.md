# WASM Adaptations Guide

**Branch**: [`wasm-adapt`](https://github.com/utxostack/rust-lightning/tree/wasm-adapt)  

## Core Modifications

```diff
- use std::time::{SystemTime, UNIX_EPOCH, Instant};
+ use lightning_common::{SystemTime, UNIX_EPOCH, Instant};
```

## Scope

- Applies to all source files ​**except** test modules
- Centralized in [`lightning-common`](./lightning-common/src/lib.rs) crate

---

## ​Maintenance Workflow

Upgrade Example (v0.0.125)


1. ​**Rebase to upstream version**  
   ```bash
   git checkout wasm-adapt
   git rebase --onto v0.0.125
   ```

2. ​**Verify adaptation integrity**  

   - Comprehensive code review to ensure that WASM-specific changes have not been missed
   - Perform necessary integration tests

3. **​Tag**

   ```bash
   git tag wasm-v0.0.125
   ```