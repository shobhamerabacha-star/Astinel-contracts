# Astinel Contracts — Engineering Issues

> Generated from full-source audit

## Contents

1. [Bugs](#1-contracts-bugs)
2. [Features](#2-contracts-features)
3. [Enhancements](#3-contracts-enhancements)
4. [Security](#4-contracts-security)
5. [Refactoring](#5-contracts-refactoring)
6. [Testing & Quality](#6-contracts-testing--quality)
7. [Build & CI](#7-contracts-build--ci)

---

# 1. Contracts: Bugs

---

## Issue #1 — Allowance unwrap panics in payable::transfer_from

**Labels:** bug, contracts, medium

**Summary:**
`contracts/payable/src/lib.rs:114` calls `.unwrap()` on a persistent storage read for an allowance key that may not exist. If `allowance()` returns a value but the storage entry is absent between the two reads, the contract panics. The identical pattern exists in `contracts/stellar-asset/src/lib.rs:157`.

**Acceptance Criteria:**

- [ ] Neither `transfer_from` panics on missing allowance
- [ ] Both contracts return a clear error message
- [ ] Tests verify behavior with nonexistent, expired, and active allowances
- [ ] Replace `.unwrap()` with `unwrap_or_else` returning a proper error in both contracts
- [ ] Read the allowance value once and reuse it to eliminate the race window

**Difficulty:** Beginner

---

## Issue #2 — extend_ttl unwrap panics on nonexistent key

**Labels:** bug, contracts, medium

**Summary:**
`contracts/ttl-management/src/lib.rs:77` calls `.unwrap()` on a persistent storage get for a key that may not exist. If a caller invokes `extend_ttl` with a key that was never written, the contract panics unconditionally.

**Acceptance Criteria:**

- [ ] `extend_ttl` with nonexistent key does not panic
- [ ] Replace `.unwrap()` with a pattern that returns an error or no-op for missing keys
- [ ] Test verifies graceful handling

**Difficulty:** Beginner

---

## Issue #3 — 15 pre-init unwrap panics across all contracts

**Labels:** bug, contracts, low

**Summary:**
All five contracts call `.unwrap()` on instance storage reads without default fallbacks. If `init()` was never called, every public function panics. While convention requires `init()` first, the panics provide no diagnostic information. Affected lines: access-control (9, 16, 20, 40, 49, 54, 61), payable (29), ttl-management (31, 52, 64, 86), upgradeable (23, 29, 37), stellar-asset (48, 54, 169).

**Acceptance Criteria:**

- [ ] All 21 production unwraps have meaningful error messages
- [ ] Messages follow a consistent format
- [ ] Replace every instance with `.expect("ContractName: not initialized")`

**Difficulty:** Beginner

---

## Issue #4 — burn does not update metadata.total_supply in stellar-asset

**Labels:** bug, contracts, high

**Summary:**
`contracts/stellar-asset/src/lib.rs:101-110`: `burn` updates `TOTAL_SUPPLY` instance storage but does not update `METADATA.total_supply`. After a burn, `metadata().total_supply` returns a stale value that is higher than the actual supply. `mint` correctly updates both.

**Acceptance Criteria:**

- [ ] `metadata().total_supply` equals `total_supply()` after any sequence of mint/burn
- [ ] Add `meta.total_supply = supply - amount` after the supply update in `burn`
- [ ] Test validates the invariant

**Difficulty:** Beginner

---

## Issue #5 — remaining_ttl formula does not match on-ledger TTL

**Labels:** bug, contracts, low

**Summary:**
`contracts/ttl-management/src/lib.rs:89-93`: `remaining_ttl` computes `ttl_ledgers - (current_ledger - created_at)` from stored struct fields. However, `extend_ttl` calls `env.storage().persistent().extend_ttl(&key, ttl_ledgers, ttl_ledgers + LIVE_LEDGERS)`, which sets the on-ledger threshold to `ttl_ledgers + LIVE_LEDGERS`. The computed value does not account for the `LIVE_LEDGERS` buffer, so it reports a lower remaining TTL than the ledger actually provides.

**Acceptance Criteria:**

- [ ] `remaining_ttl` returns an accurate value
- [ ] Audit the formula against Soroban's `extend_ttl` semantics
- [ ] Either fix the formula or document the discrepancy
- [ ] Test verifies `remaining_ttl` matches on-ledger behavior

**Difficulty:** Intermediate

---

# 2. Contracts: Features

---

## Issue #6 — Emit events on all state-changing functions

**Labels:** contracts, feature, enhancement

**Summary:**
No contract emits Soroban events (`env.events().publish()`). Off-chain indexers cannot track state changes. Every mutating function should emit events. The contributing guide (CONTRIBUTING.md:29) recommends this, but zero sites implement it. Affected functions across all 5 contracts: access-control (4 events), payable (4), ttl-management (3), upgradeable (3), stellar-asset (6).

**Acceptance Criteria:**

- [ ] Every mutating function emits at least one event
- [ ] Define event topic symbols for each contract
- [ ] Include relevant data (addresses, amounts, keys) in event payloads
- [ ] Event payloads contain all relevant data for off-chain reconstruction
- [ ] Tests verify event count and content

**Difficulty:** Intermediate

---

## Issue #7 — Add burn function to payable contract

**Labels:** contracts, feature

**Summary:**
`astelin-payable` is an ERC-20-like token but has no `burn` function. `astelin-stellar-asset` has one. For feature parity and to support token destruction use cases, `payable` should implement `burn`.

**Acceptance Criteria:**

- [ ] `payable::burn` works and is tested
- [ ] Add `pub fn burn(env, from, amount)` following the same auth pattern as stellar-asset
- [ ] Deduct from balance and total supply
- [ ] Balance and total supply decrease correctly
- [ ] Add tests for burn: valid amount, zero amount, insufficient balance

**Difficulty:** Beginner

---

## Issue #8 — Implement cross-contract pause check integration

**Labels:** contracts, feature

**Summary:**
`access-control` defines `require_not_paused()` at line 80-82 but no contract calls it. The pause mechanism is non-functional. Other contracts should call `access-control::require_not_paused` before executing mutating operations.

**Acceptance Criteria:**

- [ ] Pausing the access-control contract blocks operations in all other contracts
- [ ] Design the integration pattern (each contract holds an access-control contract ID)
- [ ] Add a `set_access_control` or `init_with_acl` function to each contract
- [ ] Call `require_not_paused` at the start of every mutating function
- [ ] Unpausing restores operation
- [ ] Tests cover the full pause/unpause lifecycle

**Difficulty:** Advanced

---

# 3. Contracts: Enhancements

---

## Issue #9 — Validate identity before require_auth to save gas

**Labels:** contracts, enhancement, medium

**Summary:**
All five contracts use the pattern `caller.require_auth(); assert_eq!(caller, admin)` in their `only_admin` guards. This means non-admin callers pay the auth verification cost (signature check) before being rejected. Checking identity first and only calling `require_auth` if the caller matches would save gas for unauthorized callers.

**Acceptance Criteria:**

- [ ] Unauthorized callers pay less gas
- [ ] Invert the check: `assert_eq!` first, then `require_auth`
- [ ] Security invariant is unchanged (admin must still sign)

**Difficulty:** Beginner

---

## Issue #10 — Refactor unbounded Vec to Map in ttl-management

**Labels:** contracts, enhancement, refactor

**Summary:**
`contracts/ttl-management` stores an `ENTRIES` Vec that grows unboundedly on every `write()`. Each `write` and `remove` reads, modifies, and rewrites the entire Vec — O(n) per operation on an O(n) data structure (O(n²) total). For contracts with many entries, this will hit Soroban storage limits and become prohibitively expensive.

**Acceptance Criteria:**

- [ ] No unbounded data structures remain
- [ ] Replace `Vec<Symbol>` with `Map<Symbol, bool>` or remove the index entirely
- [ ] If an index is needed, use a Map keyed by Symbol with a boolean marker
- [ ] Update `remove` to delete from the Map instead of filtering a Vec
- [ ] Update `entries()` to iterate the Map keys
- [ ] Write and remove are O(1) or O(log n) instead of O(n)
- [ ] All existing tests pass with the new structure

**Difficulty:** Intermediate

---

## Issue #11 — Prevent duplicate operator entries in access-control

**Labels:** contracts, enhancement

**Summary:**
`contracts/access-control/src/lib.rs:52-57`: `add_operator` pushes to the Vec without checking for duplicates. The same address can appear multiple times. `remove_operator` will then remove only the first match, leaving a duplicate behind.

**Acceptance Criteria:**

- [ ] Adding an operator that already exists is rejected
- [ ] Check `operators.contains(&operator)` before pushing
- [ ] No duplicate entries appear in the operators list
- [ ] Return early with a clear message if operator already exists

**Difficulty:** Beginner

---

## Issue #12 — Prevent duplicate keys in ttl-management ENTRIES

**Labels:** contracts, enhancement

**Summary:**
`contracts/ttl-management/src/lib.rs:52-53`: `write` always appends the key to `ENTRIES` without checking if it already exists. Writing the same key twice creates duplicate entries. `remove` filters by value match, removing only the first match.

**Acceptance Criteria:**

- [ ] Writing the same key twice does not create duplicates in ENTRIES
- [ ] Check if key already exists in `ENTRIES` before appending
- [ ] After the Vec→Map refactor, this is naturally handled

**Difficulty:** Beginner

---

## Issue #13 — Validate admin address on set_admin

**Labels:** contracts, enhancement

**Summary:**
No contract validates that `new_admin` is a non-zero or valid address. The admin can be set to the zero address, permanently locking admin functionality across all contracts that implement `set_admin`. Affected: `access-control`, `upgradeable`, `stellar-asset`.

**Acceptance Criteria:**

- [ ] Setting admin to zero address is rejected
- [ ] Add `assert!(new_admin != Address::zero(env), ...)` before setting
- [ ] Tests verify the check

**Difficulty:** Beginner

---

## Issue #14 — Validate expiration_ledger is in the future

**Labels:** contracts, enhancement

**Summary:**
Both `payable::approve` and `stellar-asset::approve` accept any `expiration_ledger` value, including 0 (immediately expired) or `u32::MAX` (effectively infinite). Approvals with past or invalid expiration ledgers should be rejected or warned.

**Acceptance Criteria:**

- [ ] Approvals with past expiration ledgers are rejected
- [ ] Add `assert!(expiration_ledger > env.ledger().sequence(), ...)` to both approve functions
- [ ] Tests verify the check in both contracts

**Difficulty:** Beginner

---

## Issue #15 — Add overflow/underflow protection on balance arithmetic

**Labels:** contracts, enhancement

**Summary:**
Mint, burn, and transfer use unchecked `+` / `-` on `i128` values. While i128 overflow is astronomically unlikely, Soroban best practices recommend using `checked_add`/`checked_sub` with proper error handling. Affected lines: payable (61, 63), stellar-asset (91, 93, 97, 106, 110).

**Acceptance Criteria:**

- [ ] All arithmetic uses checked operations
- [ ] Replace `a + b` with `a.checked_add(b).expect(...)` in all mint functions
- [ ] Replace `a - b` with `a.checked_sub(b).expect(...)` in all burn/transfer functions
- [ ] Overflow produces a clear panic, not silent wrapping
- [ ] Add tests that would trigger overflow conditions

**Difficulty:** Beginner

---

## Issue #16 — Add amount >= 0 check to stellar-asset::approve

**Labels:** contracts, security, bug

**Summary:**
`payable::approve` at line 96 checks `assert!(amount >= 0, ...)` but `stellar-asset::approve` (line 133-147) has no such check. A negative allowance can be set, which would cause underflow in `transfer_from` when computing `existing.amount - amount` (if amount is negative, this increases instead of decreasing).

**Acceptance Criteria:**

- [ ] Negative amounts are rejected in stellar-asset::approve
- [ ] Add `assert!(amount >= 0, "SAFE: amount must be non-negative")` to stellar-asset::approve
- [ ] Test verifies rejection

**Difficulty:** Beginner

---

# 4. Contracts: Security

---

## Issue #17 — Zero unauthorized-caller tests across all contracts

**Labels:** contracts, security, testing, high

**Summary:**
All 14 tests across all 5 contracts use `env.mock_all_auths()`, which automatically approves all authorization requests. No test verifies that an unauthorized caller is rejected by `only_admin` guards. A logic error in the auth pattern would not be caught. Additionally, `contracts/ttl-management/src/lib.rs:133` uses the weaker `mock_all_auths_allowing_non_root_auth()`, suggesting the test is working around auth limitations.

**Acceptance Criteria:**

- [ ] Every contract has at least one test proving unauthorized callers are rejected
- [ ] Add at least one unauthorized-caller test per contract
- [ ] Each test should call a mutating function with a non-admin address and expect a panic
- [ ] Verify the test fails if auth is removed (tests are meaningful)
- [ ] The `mock_all_auths_allowing_non_root_auth` workaround is eliminated or documented

**Difficulty:** Intermediate

---

## Issue #18 — Remove unused _caller parameter from stellar-asset::burn

**Labels:** contracts, security, refactor

**Summary:**
`contracts/stellar-asset/src/lib.rs:101`: `burn` accepts `_caller: Address` which is never used. The function uses `from.require_auth()` instead. This is misleading — callers must provide a useless parameter, and the parameter name `_caller` suggests it should be the authorizer but it is not.

**Acceptance Criteria:**

- [ ] `burn` signature is `fn(env, from: Address, amount: i128)`
- [ ] Remove the `_caller` parameter from `burn`
- [ ] Update all call sites (internal test code)
- [ ] Existing tests still pass

**Difficulty:** Beginner

---

# 5. Contracts: Refactoring

---

## Issue #19 — Unify only_admin placement across contracts

**Labels:** contracts, refactor

**Summary:**
`only_admin` is placed module-level (outside `impl`) in `access-control` and inside `impl` blocks in `payable`, `ttl-management`, `upgradeable`, `stellar-asset`. This inconsistency makes cross-contract patterns harder to reason about.

**Acceptance Criteria:**

- [ ] All `only_admin` functions follow the same placement convention
- [ ] Move all `only_admin` functions inside their respective `impl` blocks (or all outside, decide one convention)
- [ ] The function must remain private in either case

**Difficulty:** Beginner

---

## Issue #20 — Unify ADMIN storage key creation

**Labels:** contracts, refactor

**Summary:**
`payable` creates the `ADMIN` storage key at runtime with `Symbol::new(&env, "ADMIN")` at line 21. All other contracts use compile-time `symbol_short!("ADMIN")`. `symbol_short!` is more efficient (constant-time, no allocation).

**Acceptance Criteria:**

- [ ] All contracts use `symbol_short!` for storage keys
- [ ] Replace `Symbol::new(&env, "ADMIN")` with `symbol_short!("ADMIN")` in `payable`
- [ ] Verify the symbol length fits `symbol_short!` (5 chars, fits the 6-char limit)
- [ ] Tests still pass

**Difficulty:** Beginner

---

## Issue #21 — Extract allowance logic to shared helper

**Labels:** contracts, refactor

**Summary:**
Allowance read, check, and update logic is duplicated between `payable` and `stellar-asset`. The same pattern appears in `allowance()`, `approve()`, and `transfer_from()` in both contracts.

**Acceptance Criteria:**

- [ ] Allowance logic is defined once
- [ ] Create a shared module or macro for allowance operations
- [ ] Replace duplicated logic with the shared implementation
- [ ] Both contracts produce identical behavior
- [ ] Verify all tests still pass

**Difficulty:** Intermediate

---

## Issue #22 — Remove dead only_operator_or_admin function

**Labels:** contracts, refactor, chore

**Summary:**
`contracts/access-control/src/lib.rs:14-22` defines `pub fn only_operator_or_admin` but no function in any contract calls it. It is dead code. This was made `pub` in a previous fix, but nothing calls it externally either.

**Acceptance Criteria:**

- [ ] `only_operator_or_admin` is either removed or documented with its intended usage
- [ ] If the design intent is for external contracts to call it, add documentation and an example

**Difficulty:** Beginner

---

## Issue #23 — Remove dead require_not_paused (if no cross-contract usage)

**Labels:** contracts, refactor

**Summary:**
`contracts/access-control/src/lib.rs:80-82` defines `require_not_paused` but no contract calls it. If cross-contract pause integration is not planned, this is dead code.

**Acceptance Criteria:**

- [ ] `require_not_paused` is either connected or removed
- [ ] Either implement cross-contract pause checks (see feature above) or remove the function
- [ ] Document the decision

**Difficulty:** Beginner

---

# 6. Contracts: Testing & Quality

---

## Issue #24 — Add unauthorized-caller tests to all contracts

**Labels:** contracts, testing

**Summary:**
Each contract needs tests proving that non-admin callers cannot call admin-protected functions. Currently all 14 tests use `mock_all_auths`, leaving auth logic completely untested.

**Acceptance Criteria:**

- [ ] Every contract has at least one test verifying non-admin callers are rejected
- [ ] Tests call a mutating function with a non-admin address and expect a panic
- [ ] Tests are meaningful (fail if auth is removed)

**Difficulty:** Intermediate

---

## Issue #25 — Add edge case tests to payable contract

**Labels:** contracts, testing

**Summary:**
The payable contract needs edge case coverage for the following scenarios: transfer with zero amount, mint with zero amount, approve with amount = 0 (revoke), allowance expiration detection (expired ledger returns 0), and non-admin caller on mint.

**Acceptance Criteria:**

- [ ] Test transfer with zero amount
- [ ] Test mint with zero amount
- [ ] Test approve with amount = 0 (revoke)
- [ ] Test allowance expiration detection (expired ledger returns 0)
- [ ] Test non-admin caller on mint

**Difficulty:** Beginner

---

## Issue #26 — Add edge case tests to stellar-asset contract

**Labels:** contracts, testing

**Summary:**
The stellar-asset contract needs edge case coverage for: burn with zero amount, burn with insufficient balance, negative amount approve (should be rejected after fix), expired allowance returns 0, and set_admin from non-admin.

**Acceptance Criteria:**

- [ ] Test burn with zero amount
- [ ] Test burn with insufficient balance
- [ ] Test negative amount approve (should be rejected after fix)
- [ ] Test expired allowance returns 0
- [ ] Test set_admin from non-admin

**Difficulty:** Beginner

---

## Issue #27 — Add edge case tests to ttl-management contract

**Labels:** contracts, testing

**Summary:**
The ttl-management contract needs edge case coverage for: write with ttl_ledgers = 0, read nonexistent key (returns None), remove nonexistent key, remaining_ttl for missing key, and extend_ttl on existing key.

**Acceptance Criteria:**

- [ ] Test write with ttl_ledgers = 0
- [ ] Test read nonexistent key (returns None)
- [ ] Test remove nonexistent key
- [ ] Test remaining_ttl for missing key
- [ ] Test extend_ttl on existing key

**Difficulty:** Beginner

---

## Issue #28 — Add edge case tests to upgradeable contract

**Labels:** contracts, testing

**Summary:**
The upgradeable contract needs edge case coverage for: upgrade with non-increasing version (should be rejected), read_bytes on nonexistent key, and set_admin from non-admin.

**Acceptance Criteria:**

- [ ] Test upgrade with non-increasing version (should be rejected)
- [ ] Test read_bytes on nonexistent key
- [ ] Test set_admin from non-admin

**Difficulty:** Beginner

---

## Issue #29 — Add edge case tests to access-control contract

**Labels:** contracts, testing

**Summary:**
The access-control contract needs edge case coverage for: remove_operator actually removes from list, set_admin transfers admin role, and require_not_paused panics when paused.

**Acceptance Criteria:**

- [ ] Test remove_operator actually removes from list
- [ ] Test set_admin transfers admin role
- [ ] Test require_not_paused panics when paused

**Difficulty:** Beginner

---

## Issue #30 — Add property-based tests using proptest

**Labels:** contracts, testing

**Summary:**
No contract uses property-based testing. Key invariants that should hold for all inputs: `balance + total_supply` conservation across transfers, `allowance` always reflects unspent approved amount, and `metadata.total_supply == total_supply()` after any sequence.

**Acceptance Criteria:**

- [ ] Add `proptest` dev-dependency
- [ ] Write at least 3 property-based tests covering core invariants
- [ ] Test payable transfer invariant
- [ ] Test stellar-asset supply invariant
- [ ] Test allowance tracking

**Difficulty:** Advanced

---

## Issue #31 — Add doc comments to all public functions

**Labels:** contracts, documentation

**Summary:**
No public function in any contract has `///` doc comments. Users interacting via Soroban CLI or block explorers see no descriptions. Each contract should document every public function with: what it does, parameters (with auth requirements), panics (error conditions), and storage side effects.

**Acceptance Criteria:**

- [ ] Every public function across all 5 contracts has `///` doc comments
- [ ] Each comment describes what the function does
- [ ] Parameters are documented with auth requirements
- [ ] Panic conditions are documented
- [ ] Storage side effects are documented

**Difficulty:** Intermediate

---

## Issue #32 — Add module-level doc comments

**Labels:** contracts, documentation

**Summary:**
No contract has `//!` module documentation. Each should describe: contract purpose, storage schema summary, auth model, and usage example.

**Acceptance Criteria:**

- [ ] Every contract has `//!` module-level documentation
- [ ] Documentation includes contract purpose
- [ ] Documentation includes storage schema summary
- [ ] Documentation includes auth model
- [ ] Documentation includes usage example

**Difficulty:** Intermediate

---

## Issue #33 — Add doc comments to public structs

**Labels:** contracts, documentation

**Summary:**
`AllowanceValue` (payable, stellar-asset), `DataEntry` (ttl-management), and `AssetMetadata` (stellar-asset) lack any documentation.

**Acceptance Criteria:**

- [ ] `AllowanceValue` has doc comments in both payable and stellar-asset
- [ ] `DataEntry` has doc comments in ttl-management
- [ ] `AssetMetadata` has doc comments in stellar-asset

**Difficulty:** Beginner

---

# 7. Contracts: Build & CI

---

## Issue #34 — Add MSRV to workspace Cargo.toml

**Labels:** contracts, build, ci

**Summary:**
No `rust-version` is specified. CI uses `dtolnay/rust-toolchain@stable`, which may break if future toolchain versions introduce incompatible changes.

**Acceptance Criteria:**

- [ ] Cargo.toml has `rust-version` set
- [ ] Determine the minimum supported Rust version (test with `cargo install` on oldest viable)
- [ ] Add `rust-version = "1.81"` (or appropriate) to workspace `[package]`
- [ ] CI includes an MSRV check job

**Difficulty:** Intermediate

---

## Issue #35 — Run clippy with wasm target in CI

**Labels:** contracts, build, ci

**Summary:**
The CI lint job runs `cargo clippy` on the native target only. WASM-specific issues (e.g., `#![no_std]` violations, feature gate issues) are not caught.

**Acceptance Criteria:**

- [ ] WASM-specific compilation issues are caught in CI
- [ ] Add `cargo clippy --target wasm32-unknown-unknown` to the CI lint job
- [ ] Or add a separate clippy-wasm job

**Difficulty:** Beginner

---

## Issue #36 — Remove unused ed25519-dalek dependency

**Labels:** contracts, build, chore

**Summary:**
`ed25519-dalek = "2.1"` is listed as a workspace dependency and in each contract's `Cargo.toml` under `[target.'cfg(not(target_family = "wasm"))'.dependencies]`. No contract imports or uses this crate. It adds unnecessary compile time.

**Acceptance Criteria:**

- [ ] No crate named `ed25519-dalek` in the dependency tree
- [ ] Remove `ed25519-dalek` from workspace `[workspace.dependencies]`
- [ ] Remove it from each contract's `Cargo.toml`
- [ ] Verify tests still compile and pass

**Difficulty:** Beginner

---

## Issue #37 — Add [lints] table to workspace Cargo.toml

**Labels:** contracts, chore

**Summary:**
No workspace-level lint configuration exists. Adding `[lints]` enforces consistent clippy rules across all contracts.

**Acceptance Criteria:**

- [ ] Add `[lints]` table with `workspace = true`
- [ ] Add per-contract `[lints]` entries as needed
- [ ] Verify clippy passes

**Difficulty:** Beginner

---

## Issue #38 — Track edition = "2024" when stable

**Labels:** contracts, chore

**Summary:**
Contracts use `edition = "2021"`. When Rust 2024 stabilizes for `no_std` targets, migrate.

**Acceptance Criteria:**

- [ ] Monitor Rust 2024 stabilization for `no_std` targets
- [ ] Migrate all contracts to `edition = "2024"` when stable
- [ ] Verify all tests and builds pass after migration

**Difficulty:** Beginner

---

## Issue #39 — Add WASM size regression check to CI

**Labels:** contracts, build, ci

**Summary:**
No tracking of WASM binary sizes. A size regression check would alert on bloat introduced by new features or dependency updates.

**Acceptance Criteria:**

- [ ] CI reports WASM sizes
- [ ] Add a CI step that measures `ls -la target/wasm32-unknown-unknown/release/*.wasm`
- [ ] Store baseline sizes and compare on each build
- [ ] CI fails if any WASM binary grows by more than 10%

**Difficulty:** Intermediate
