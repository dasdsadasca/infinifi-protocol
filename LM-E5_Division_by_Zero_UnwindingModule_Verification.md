# Verification of LM-E5: Division by Zero in Reward Calculation (`UnwindingModule.sol`)

## Vulnerability
LM-E5: Division by Zero in Reward Calculation in `UnwindingModule.sol`. This vulnerability occurs in the `_userShares` internal function when calculating reward distribution. If a `GlobalPoint` has `rewardShares > 0` but `totalRewardWeight == 0`, the calculation `globalPoint.rewardShares.mulDivDown(userRewardWeight, globalPoint.totalRewardWeight)` will cause a division by zero, leading to a revert.

## Certainty of Exploitability
**Certain.**

The scenario leading to division by zero is plausible and can occur through a specific sequence of legitimate operations:
1.  All existing unwinding positions are withdrawn or cancelled, resulting in the `totalRewardWeight` for the current epoch's `GlobalPoint` becoming zero.
2.  Rewards are then deposited (`depositRewards()`) into the module *while* this `totalRewardWeight` is zero for the current global point being updated. This sets `rewardShares > 0` for that point.
3.  An existing user whose unwinding period spans this specific malformed `GlobalPoint` attempts an operation (e.g., `balanceOf`, `withdraw`, `cancelUnwinding`) that calls `_userShares`.
If these conditions are met, the transaction will revert due to division by zero.

## Analysis of Reward Calculation Logic and State Transitions

The critical line is within `_userShares(...)`:
`userShares += globalPoint.rewardShares.mulDivDown(userRewardWeight, globalPoint.totalRewardWeight);`

*   **`globalPoint.totalRewardWeight` can become zero:**
    *   The `GlobalPoint` struct (with `epoch`, `totalRewardWeight`, `totalRewardWeightDecrease`, `rewardShares`) is managed via the `globalPoints` mapping and the `_getLastGlobalPoint()` and `_updateGlobalPoint()` internal functions.
    *   When users `withdraw()` or `cancelUnwinding()`, their `userRewardWeight` is subtracted from the current `GlobalPoint.totalRewardWeight` before `_updateGlobalPoint()` saves it. If all users exit, `totalRewardWeight` for that epoch's `GlobalPoint` can become `0`.

*   **`globalPoint.rewardShares` can be non-zero when `totalRewardWeight` is zero:**
    *   The `depositRewards(uint256 _amount)` function first calls `_getLastGlobalPoint()` to get the current `GlobalPoint` (let's call it `P_current`).
    *   It then calculates `rewardShares_to_add` based on `_amount`.
    *   It then updates `P_current.rewardShares += rewardShares_to_add`.
    *   Finally, it calls `_updateGlobalPoint(P_current)`, saving this point.
    *   If `P_current` (as returned by `_getLastGlobalPoint()`) already had `totalRewardWeight = 0`, this sequence results in a stored `GlobalPoint` with `totalRewardWeight = 0` and `rewardShares > 0`.

*   **Division by Zero Condition:** The division occurs if, for a `GlobalPoint` being processed in the `_userShares` loop:
    *   `globalPoint.rewardShares > 0`
    *   `userRewardWeight > 0` (the user's position still has claimable weight for that epoch)
    *   `globalPoint.totalRewardWeight == 0` (denominator is zero)

## Step-by-Step Scenario to Trigger Division by Zero

*   **a. Setup Initial State (Epoch E0):**
    1.  User A starts an unwinding position. `_updateGlobalPoint` is called. `globalPoints[E0]` now has `totalRewardWeight = W_A > 0` and `rewardShares = 0`. `lastGlobalPointEpoch = E0`.
    2.  User B starts an unwinding position later in Epoch E0. `_updateGlobalPoint` is called again. `globalPoints[E0]` now has `totalRewardWeight = W_A + W_B > 0` and `rewardShares = 0`.

*   **b. Emptying `totalRewardWeight` (Epoch E1, where E1 is the epoch User A and B's positions end or are cancelled):**
    1.  Assume User A's unwinding period ends. User A calls `withdraw()`.
        *   Inside `withdraw()`, `_getLastGlobalPoint()` provides `P_E1_current`.
        *   `P_E1_current.totalRewardWeight` is reduced by User A's `userRewardWeight`.
        *   `_updateGlobalPoint(P_E1_current)` stores this updated point for epoch `E1`.
    2.  In the same Epoch E1, User B also calls `withdraw()`.
        *   `_getLastGlobalPoint()` gets the point updated by User A's withdrawal.
        *   `P_E1_current.totalRewardWeight` is further reduced by User B's `userRewardWeight`. Since A and B were the only contributors, `P_E1_current.totalRewardWeight` now becomes `0`.
        *   `_updateGlobalPoint(P_E1_current)` stores `globalPoints[E1]` with `epoch = E1`, `totalRewardWeight = 0`, and `rewardShares = 0` (assuming no rewards deposited yet). `lastGlobalPointEpoch = E1`.

*   **c. Depositing Rewards (Still in Epoch E1, after A & B withdrew):**
    1.  The `LOCKED_TOKEN_MANAGER` calls `UnwindingModule.depositRewards(rewardsAmount > 0)`.
    2.  `_getLastGlobalPoint()` is called. It returns the `globalPoints[E1]` which has `totalRewardWeight = 0` and `rewardShares = 0`.
    3.  `rewardShares_from_deposit = _amountToShares(rewardsAmount)` is calculated. This will be `> 0` if `rewardsAmount > 0`.
    4.  The loaded point's `rewardShares` becomes `rewardShares_from_deposit > 0`.
    5.  `_updateGlobalPoint()` saves this point. Now, `globalPoints[E1]` is `{epoch: E1, totalRewardWeight: 0, rewardShares: X > 0}`.

*   **d. Triggering Division by Zero (Still in Epoch E1, or a later epoch if no new points are made):**
    1.  User D is an existing user whose unwinding position (`Position_D`) started at `D_fromEpoch < E1` and is scheduled to end at `D_toEpoch > E1`. So, User D's position is active during Epoch E1.
    2.  User D (or any contract interacting with their position) calls `balanceOf(UserD_address, UserD_startTimestamp)`. This will invoke `_userShares()`.
    3.  Inside `_userShares()` for User D:
        *   The loop `for (uint32 epoch = Position_D.fromEpoch - 1; epoch <= currentEpoch; epoch++)` runs.
        *   When the loop variable `epoch` reaches `E1`:
            *   `GlobalPoint memory epochGlobalPoint = globalPoints[E1];` This loads the point `{epoch: E1, totalRewardWeight: 0, rewardShares: X > 0}`.
            *   The local `globalPoint` variable is set to this `epochGlobalPoint`.
            *   The condition `if (epoch >= Position_D.fromEpoch)` is true.
            *   User D's `userRewardWeight` for epoch E1 is calculated and is `> 0`.
            *   The critical line executes: `userShares += globalPoint.rewardShares.mulDivDown(userRewardWeight, globalPoint.totalRewardWeight);`
            *   This becomes: `userShares += X.mulDivDown(userRewardWeight_D_at_E1, 0);`
    4.  **Outcome:** The `mulDivDown` attempts division by zero, causing the entire transaction to revert. User D cannot get their balance, withdraw, or cancel their unwinding.

## Detailed Impact Assessment

*   **Denial of Service for Users:**
    *   Any user whose active unwinding position's share calculation (`_userShares`) needs to process a `GlobalPoint` that has `rewardShares > 0` and `totalRewardWeight = 0` will be affected.
    *   This prevents them from calling `balanceOf()`, `withdraw()`, and `cancelUnwinding()`.
    *   If a user's unwinding period has completed but their withdrawal calculation path hits such a point, their **funds are effectively frozen** as they cannot complete the withdrawal.
*   **Scope of Impact:**
    *   The DoS affects users whose unwinding periods span the specific epoch(s) where these malformed `GlobalPoint`s exist.
    *   It does not necessarily affect users whose unwinding periods are entirely before or entirely after such points (if new, valid global points are created).
    *   New users starting to unwind *after* the problematic point is created might also be affected if their `_userShares` loop needs to iterate past that point for any reason (e.g., if `lastGlobalPointEpoch` is still the epoch of the bad point).

This is a critical vulnerability as it can prevent users from accessing their funds or managing their positions.

## Existing Code-Level Mitigation Analysis

*   **No Zero Check for Denominator:** The line `userShares += globalPoint.rewardShares.mulDivDown(userRewardWeight, globalPoint.totalRewardWeight);` in `_userShares` does **not** check if `globalPoint.totalRewardWeight` is zero before performing the division.
*   **No Prevention in `depositRewards`:** The `depositRewards` function does not prevent `rewardShares` from being added to a `GlobalPoint` that currently has `totalRewardWeight = 0`.

There are no existing mitigations for this specific division by zero.

## Impact
**Critical.**
This vulnerability leads to a Denial of Service (DoS) for users trying to interact with their unwinding positions (`balanceOf`, `withdraw`, `cancelUnwinding`). If triggered, it can make it impossible for affected users to retrieve their funds or manage their positions, effectively freezing their assets within the `UnwindingModule`.

## Recommendations

1.  **Conditional Reward Distribution in `_userShares`:**
    *   Modify the reward distribution line in `_userShares` to only attempt the division if `globalPoint.totalRewardWeight > 0`.
    ```solidity
    // Inside _userShares, in the loop:
    if (epoch >= position.fromEpoch) {
        if (globalPoint.totalRewardWeight > 0) { // Add this check
            userShares += globalPoint.rewardShares.mulDivDown(userRewardWeight, globalPoint.totalRewardWeight);
        }
        // If totalRewardWeight is 0, no rewards from this point can be proportionally distributed,
        // so effectively 0 rewards are added for this point.
    }
    ```
    This is the most direct fix to prevent the division by zero.

2.  **Review Logic in `depositRewards` (Optional Consideration):**
    *   Consider the implications if `depositRewards` is called when `point.totalRewardWeight` (from `_getLastGlobalPoint`) is zero.
    *   Currently, it adds `rewardShares` to this point. These shares might become inaccessible if `totalRewardWeight` remains zero for that point's epoch indefinitely.
    *   One option could be to prevent adding `rewardShares` if `totalRewardWeight` is zero, perhaps by reverting the reward deposit or by allocating those rewards to a general reserve or future epochs. However, this changes the current reward distribution logic. The primary fix (recommendation 1) is safer and less intrusive. Allowing rewards to be deposited and then not distributed if `totalRewardWeight` is zero for that point is a reasonable outcome as there's no basis for proportional distribution.

The primary and most crucial recommendation is to add the conditional check before the division in `_userShares`.
