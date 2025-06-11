# Verification of LM-E2: Gas DoS in Epoch Iteration (`UnwindingModule.sol`)

## Vulnerability
LM-E2: Gas DoS in Epoch Iteration (`UnwindingModule.sol`). This vulnerability suggests that certain functions in `UnwindingModule.sol` contain loops that iterate over epochs. If a significant number of epochs pass without these states being updated, calls to these functions can consume excessive gas, potentially exceeding the block gas limit and leading to a Denial of Service (DoS).

## Certainty of Exploitability
**Certain and Inevitable under described conditions.**

The Gas DoS vulnerability is certain. The functions `_getLastGlobalPoint()` and `_userShares()` (and by extension, public functions calling them) contain loops that iterate a number of times proportional to `currentEpoch - historicalEpoch`. If enough epochs (weeks) pass without specific transactions that update these historical epoch markers, the number of iterations will grow indefinitely, leading to gas costs that will eventually exceed the block gas limit.

## Analysis of Loop Structures and Gas Consumption

**1. `_getLastGlobalPoint()` internal function:**
   *   **Loop:** `for (uint32 epoch = point.epoch; epoch < currentEpoch; epoch++)`
   *   **Iterations:** `currentEpoch - point.epoch` (where `point.epoch` is derived from `lastGlobalPointEpoch`).
   *   **Purpose:** Extrapolates the `GlobalPoint` state from `lastGlobalPointEpoch` to `currentEpoch`.
   *   **Gas per Iteration:** Each iteration reads up to three storage slots (`rewardWeightIncreases[epoch]`, `rewardWeightDecreases[epoch]`, `rewardWeightBiasIncreases[epoch]`) which can be cold (approx. 2100 gas each if cold, 100 if warm). Plus arithmetic operations. Conservatively, **200-6300+ gas per iteration.**

**2. `_userShares(address _user, uint256 _startUnwindingTimestamp)` internal function:**
   *   **Loop:** `for (uint32 epoch = position.fromEpoch - 1; epoch <= currentEpoch; epoch++)`
   *   **Iterations:** `currentEpoch - (position.fromEpoch - 1)`.
   *   **Purpose:** Calculates a user's current shares by iterating through epochs from their unwinding start, applying rewards and updating a local copy of the global point iteratively.
   *   **Gas per Iteration:** Reads `globalPoints[epoch]` (1 SLOAD). Then, similar to `_getLastGlobalPoint`, it reads up to three more SLOADs for `rewardWeight*` arrays, plus arithmetic (including `mulDivDown`). Conservatively, **200-8400+ gas per iteration** (potentially 4 SLOADs).

**Epoch Duration:** As per `EpochLib.sol`, 1 epoch = 1 week.

**Public functions affected (callers of the above):**
*   `_getLastGlobalPoint()` is called by: `totalRewardWeight()`, `startUnwinding()`, `cancelUnwinding()`, `withdraw()`, `depositRewards()`.
*   `_userShares()` is called by: `balanceOf()`, `cancelUnwinding()`, `withdraw()`.
*   `rewardWeight()` also contains a loop: `for (uint32 epoch = position.fromEpoch + 1; epoch <= currentEpoch && epoch <= position.toEpoch; epoch++)`. This loop is cheaper per iteration (only subtractions) but still iterates proportionally to `currentEpoch - position.fromEpoch`.

## Step-by-Step Scenario to Trigger DoS

*   **a. Initial State (Time T0):**
    *   `UnwindingModule` is deployed. `lastGlobalPointEpoch` is set to `epoch(T0)`.
    *   User Alice starts an unwinding position at T0. `Alice_position.fromEpoch` is set to `epoch(T0) + 1`. `_updateGlobalPoint` is called, so `lastGlobalPointEpoch` becomes `epoch(T0)`.

*   **b. Passage of Time (Many Epochs):**
    *   Assume 10 years pass (520 epochs: `10 years * 52 weeks/year`).
    *   During these 10 years, no transactions call functions like `startUnwinding`, `cancelUnwinding`, `withdraw`, or `depositRewards` that would update `lastGlobalPointEpoch` or process Alice's position.
    *   `currentEpoch` is now `epoch(T0) + 520`.
    *   `lastGlobalPointEpoch` is still `epoch(T0)`.
    *   Alice's `position.fromEpoch` is still `epoch(T0) + 1`.

*   **c. Triggering Transaction (Time T0 + 10 years):**
    *   Alice's unwinding is complete, and she attempts to call `withdraw(alice_start_timestamp, alice_address)`.

*   **d. Gas Exhaustion:**
    1.  `withdraw()` calls `_userShares(alice_address, alice_start_timestamp)`.
        *   The loop in `_userShares` runs from `epoch = Alice_position.fromEpoch - 1` (i.e., `epoch(T0)`) up to `currentEpoch` (i.e., `epoch(T0) + 520`).
        *   Number of iterations = `(epoch(T0) + 520) - epoch(T0) + 1 = 521 iterations`.
        *   Estimated gas for this loop: `521 iterations * (e.g., 4 * 2100 gas/SLOAD + overhead) approx 521 * 8500 gas = ~4,428,500 gas`.
    2.  `withdraw()` then calls `_getLastGlobalPoint()`.
        *   The loop in `_getLastGlobalPoint` runs from `point.epoch = lastGlobalPointEpoch` (i.e., `epoch(T0)`) up to `currentEpoch` (i.e., `epoch(T0) + 520`).
        *   Number of iterations = `(epoch(T0) + 520) - epoch(T0) = 520 iterations`.
        *   Estimated gas for this loop: `520 iterations * (e.g., 3 * 2100 gas/SLOAD + overhead) approx 520 * 6500 gas = ~3,380,000 gas`.
    3.  **Total Estimated Gas for Loops:** `4,428,500 + 3,380,000 = 7,808,500 gas`.
    *   This is for 10 years. For longer periods:
        *   **20 years (1040 epochs):** Approx. `1040 * 8500 + 1040 * 6500 = 8,840,000 + 6,760,000 = 15,600,000 gas`.
        *   **30 years (1560 epochs):** Approx. `1560 * 8500 + 1560 * 6500 = 13,260,000 + 10,140,000 = 23,400,000 gas`.
        *   **40 years (2080 epochs):** Approx. `2080 * 8500 + 2080 * 6500 = 17,680,000 + 13,520,000 = 31,200,000 gas`.
    *   A 40-year scenario without any relevant interactions on the `UnwindingModule` would almost certainly cause the `withdraw` transaction to fail due to exceeding the block gas limit (typically around 30 million gas), when accounting for other non-loop opcodes in the transaction. Even 20-30 years can make transactions prohibitively expensive.

## Detailed Impact Assessment

*   **User Fund Access:**
    *   `withdraw()`: Users whose positions have been unwinding for many unprocessed epochs will be unable to withdraw their funds. The transaction will revert due to out-of-gas errors. This is a **permanent freeze of user funds** unless the DoS condition is somehow mitigated (e.g., by someone else forcing intermediate global point updates, if possible and economical).
    *   `cancelUnwinding()`: Users will be unable to cancel their unwinding positions if they have been active for too long, preventing them from re-locking or adjusting their strategy.
    *   `balanceOf()` / `rewardWeight()`: Users and other contracts will be unable to accurately check balances or reward weights, hindering off-chain and on-chain accounting or decision-making.
*   **Protocol Operations:**
    *   `depositRewards()`: If `lastGlobalPointEpoch` is very stale, the `LOCKED_TOKEN_MANAGER` will be unable to deposit new rewards into the `UnwindingModule` as this function calls `_getLastGlobalPoint()`. This halts reward distribution for all users in the unwinding module.
    *   Other functions in `LockingController` that call `UnwindingModule` (e.g., `startUnwinding`, `cancelUnwinding`, `withdraw`) will also fail if the `UnwindingModule` itself is DoS'd by these loops.

The Gas DoS renders critical functionalities of the `UnwindingModule` unusable for affected users or for the entire module over time.

## Existing Code-Level Mitigation Analysis
*   **No Bounded Iteration:** The loops in `_getLastGlobalPoint` and `_userShares` iterate based on the difference between the current epoch and a stored historical epoch. There are no mechanisms to limit the number of iterations within a single call (e.g., processing only a fixed number of epochs per call).
*   **No Piece-meal Update Functions:** The contract does not expose public functions that would allow for processing pending epoch updates in smaller, manageable batches (e.g., a keeper-callable function to advance `lastGlobalPointEpoch` by N epochs at a time). While functions like `startUnwinding`, `cancelUnwinding`, `withdraw`, and `depositRewards` do call `_updateGlobalPoint`, they always attempt to update to the *current* epoch, meaning they will try to process the entire accumulated gap in one go if `_getLastGlobalPoint` is called first.

The vulnerability exists because the design assumes that these state-updating functions will be called frequently enough to prevent a large backlog of unprocessed epochs. If this assumption fails, the DoS occurs.

## Impact
**Critical.**
This Gas DoS vulnerability can lead to:
1.  **Permanent loss of access to user funds:** Users may be unable to withdraw their assets from completed unwinding positions.
2.  **Inability to manage positions:** Users may be unable to cancel ongoing unwinding.
3.  **Failure of reward distribution:** Admins may be unable to deposit new rewards into the module.
4.  **Disruption of dependent contract interactions:** `LockingController` relies on `UnwindingModule`; a DoS here impacts `LockingController`'s functionality.

The financial impact is high as it can trap user assets and halt critical protocol functions.

## Recommendations

1.  **Implement Bounded Loops for Epoch Processing:**
    *   Modify `_getLastGlobalPoint` and `_userShares` (and any similar loops) to process only a limited number of epochs per transaction (e.g., a fixed batch size like 10-20 epochs).
    *   These functions would need to return an indicator if more epochs remain to be processed.
2.  **Introduce Public Keeper Functions for State Advancement:**
    *   Create new public, permissionless (or role-restricted if necessary, but preferably open) functions that can be called to advance the global state or a user's position state by a limited number of epochs.
    *   For example, `processGlobalPoints(uint256 numEpochsToProcess)` and `processUserPosition(address user, uint256 startTimestamp, uint256 numEpochsToProcess)`.
    *   These functions would allow anyone (or keepers) to help keep the system state up-to-date in smaller, gas-friendly transactions, preventing the buildup of a large processing backlog.
3.  **Store User Position Updates:**
    *   When `_userShares` calculates a user's shares up to `currentEpoch`, consider storing the `userShares` and the `currentEpoch` (as `lastUpdatedEpochForPosition`) within the `UnwindingPosition` struct.
    *   Subsequent calls would then only need to iterate from `lastUpdatedEpochForPosition` to the new `currentEpoch`, significantly reducing iterations for frequently accessed active positions. This is a form of checkpointing user state.
4.  **Global Point Checkpointing:**
    *   The `_updateGlobalPoint` function already saves points when write operations occur. However, if these are infrequent, a dedicated keeper function as mentioned in R2 could create intermediate `GlobalPoint` entries even without other user/admin activity.
5.  **Gas Cost Analysis and Limits:** Thoroughly analyze the gas costs of these loops with realistic worst-case scenarios (many SLOADs being cold) and set conservative bounds for iterations per transaction.
6.  **Off-Chain Monitoring:** Encourage off-chain monitoring to detect when `lastGlobalPointEpoch` or user positions become significantly stale, so keepers can be triggered.

These changes would make the `UnwindingModule` more resilient to DoS attacks caused by long periods of inactivity.
