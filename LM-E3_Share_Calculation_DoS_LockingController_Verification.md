# Verification of LM-E3: Share Calculation Precision/DoS (`LockingController.sol`)

## Vulnerability
LM-E3: Share Calculation Precision/DoS in `LockingController.sol`. This vulnerability describes a scenario where, after significant losses are applied to a locking bucket, its internal accounting for `totalReceiptTokens` can become zero while users still hold shares (`LockedPositionToken`) for that bucket. This leads to two primary issues:
1.  **Denial of Service (DoS):** New deposits into the affected bucket via `createPosition()` will revert due to division by zero.
2.  **Value Loss for Existing Depositors:** Users attempting to unwind their shares from the affected bucket via `startUnwinding()` will receive zero `receiptToken`s in return, effectively losing their principal.

## Certainty of Exploitability
**Certain.**

The described scenarios are certain to occur given the precondition that `applyLosses()` reduces a bucket's `data.totalReceiptTokens` to zero while shares for that bucket still exist. The current contract logic directly leads to these outcomes.

## Analysis of Share Calculation Logic

The core of the issue lies in how shares are minted and redeemed relative to the `totalReceiptTokens` held in a bucket and the `totalShares` of its corresponding `LockedPositionToken`.

1.  **`createPosition(uint256 _amount, ..., address _recipient)`:**
    *   `newShares = totalShares == 0 ? _amount : _amount.mulDivDown(totalShares, data.totalReceiptTokens);`
    *   If `data.totalReceiptTokens` is `0` and `totalShares > 0` (i.e., shares exist but are backed by no value), this calculation attempts `_amount.mulDivDown(totalShares, 0)`, causing a **division-by-zero revert.**

2.  **`startUnwinding(uint256 _shares, ..., address _recipient)`:**
    *   `userReceiptToken = _shares.mulDivDown(data.totalReceiptTokens, totalShares);`
    *   If `data.totalReceiptTokens` is `0` and `totalShares > 0` (and `_shares > 0`), this calculation results in `userReceiptToken = _shares.mulDivDown(0, totalShares) = 0`. The user receives no underlying `receiptToken` for their shares.

3.  **`exchangeRate(uint32 _unwindingEpochs)`:**
    *   `return data.totalReceiptTokens.divWadDown(totalShares);` (simplified, actual code checks for `totalShares == 0` first).
    *   If `data.totalReceiptTokens` is `0` and `totalShares > 0`, this correctly returns `0`, indicating valueless shares for that bucket within `LockingController`.

## Step-by-Step Scenario to Reach Vulnerable State (via `applyLosses`)

*   **a. Initial Deposits:**
    1.  A locking bucket for `unwindingEpochs = N` is enabled with `LPT_N` as its `shareToken`.
    2.  User A deposits `1000e18` `receiptToken`s into bucket N by calling `createPosition(1000e18, N, UserA_address)`.
    3.  **State of Bucket N:**
        *   `buckets[N].totalReceiptTokens = 1000e18`.
        *   `IERC20(LPT_N).totalSupply() = 1000e18` (held by User A).
        *   `globalReceiptToken` includes these `1000e18` tokens.

*   **b. Application of Critical Losses:**
    1.  The `FINANCE_MANAGER` calls `LockingController.applyLosses(totalLossAmount)`.
    2.  Assume `totalLossAmount` is significant enough that the portion of losses allocated to Bucket N (based on its share of `globalReceiptToken` compared to `UnwindingModule`'s balance) is `1000e18` or more.
    3.  Inside `applyLosses`, the loop for `enabledBuckets` processes Bucket N:
        *   `BucketData storage data = buckets[N];`
        *   `epochTotalReceiptToken = data.totalReceiptTokens; // 1000e18`
        *   The calculated `allocation` of loss to this bucket becomes `1000e18` (as it's capped by `_min(allocation, epochTotalReceiptToken)`).
        *   `data.totalReceiptTokens = epochTotalReceiptToken - allocation; // 1000e18 - 1000e18 = 0`.
        *   Other global state variables (`globalReceiptToken`, `globalRewardWeight`) are updated accordingly.
    4.  **Resulting State for Bucket N:**
        *   `buckets[N].totalReceiptTokens = 0`.
        *   `IERC20(LPT_N).totalSupply()` remains `1000e18` (User A's shares are not burned by `applyLosses`).
    *   **This is the vulnerable state: The bucket holds no underlying `receiptToken` value, but `LPT_N` shares still exist.**

## Step-by-Step Exploit/DoS Scenarios (Post-Losses)

Following the state achieved above:

*   **Scenario 1: `createPosition` Reverts (DoS on new deposits to Bucket N):**
    1.  User B attempts to deposit `500e18` `receiptToken`s into Bucket N by calling `createPosition(500e18, N, UserB_address)`.
    2.  The function executes:
        *   `data.totalReceiptTokens` is `0`.
        *   `totalShares = IERC20(LPT_N).totalSupply()` is `1000e18`.
        *   The share calculation `newShares = _amount.mulDivDown(totalShares, data.totalReceiptTokens)` becomes `500e18.mulDivDown(1000e18, 0)`.
    3.  **Outcome:** The transaction reverts due to division by zero. No new deposits can be made into Bucket N, effectively bricking it for future use.

*   **Scenario 2: `startUnwinding` Burns User A's Shares for Zero Value:**
    1.  User A, holding `1000e18` shares of `LPT_N`, attempts to unwind them by calling `startUnwinding(1000e18, N, UserA_address)`.
    2.  The function executes:
        *   `data.totalReceiptTokens` is `0`.
        *   `totalShares = IERC20(LPT_N).totalSupply()` is `1000e18`.
        *   `userReceiptToken = _shares.mulDivDown(data.totalReceiptTokens, totalShares)` becomes `1000e18.mulDivDown(0, 1000e18)`, which results in `userReceiptToken = 0`.
        *   User A's `1000e18` `LPT_N` shares are transferred to the controller and then burned via `LockedPositionToken(data.shareToken).burn(1000e18)`.
        *   `UnwindingModule.startUnwinding(UserA_address, 0, N, 0)` is called (0 value, 0 reward weight).
        *   `IERC20(receiptToken).transfer(unwindingModule, 0)` is called.
    3.  **Outcome:** User A's `1000e18` shares are burned, but they receive `0` `receiptToken`s in return. Their initial investment in this bucket is entirely lost. An unwinding position with zero value is created in the `UnwindingModule`.

*   **Scenario 3: `exchangeRate` Reports Zero Value for Shares in Bucket N:**
    1.  Anyone calls `exchangeRate(N)`.
    2.  The function executes:
        *   `data.totalReceiptTokens` is `0`.
        *   `totalShares = IERC20(LPT_N).totalSupply()` is `1000e18` (assuming User A hasn't unwound).
        *   The calculation `data.totalReceiptTokens.divWadDown(totalShares)` becomes `0 .divWadDown(1000e18)`, resulting in `0`.
    3.  **Outcome:** The exchange rate is correctly reported as `0`, indicating the shares for Bucket N have no underlying backing within `LockingController`.

## Detailed Impact Assessment

1.  **Permanent DoS for Specific Buckets:** Once a bucket enters this state (`totalReceiptTokens = 0`, `totalShares > 0`), it cannot accept new deposits due to the division-by-zero error in `createPosition`. This effectively "bricks" the bucket for further capital accumulation.
2.  **Total Loss of Locked Funds for Existing Users in Affected Buckets:** Users who hold shares in a bucket that has been fully depleted by `applyLosses` will lose their entire principal if they attempt to unwind. Their shares are burned, but they receive zero `receiptToken`s in return.
3.  **Useless Unwinding Positions:** Unwinding these valueless shares creates 0-value positions in the `UnwindingModule`. While this might not directly cause errors in `UnwindingModule` (as it should handle 0-value amounts), it adds unnecessary state and could slightly increase gas for global calculations there if many such positions exist.
4.  **Precision Loss (General Case):** The vulnerability is named "Precision/DoS". The DoS occurs when `totalReceiptTokens` is exactly zero. However, if `totalReceiptTokens` becomes a very small non-zero number (e.g., 1 wei) while `totalShares` is large:
    *   `createPosition` for a small `_amount`: `_amount.mulDivDown(large_totalShares, 1 wei)` could mint an excessively large number of new shares. This is an **exploitation risk leading to unfair share dilution for existing holders.**
    *   `startUnwinding` for a typical number of `_shares`: `_shares.mulDivDown(1 wei, large_totalShares)` could result in `0` `userReceiptToken` due to truncation, causing users to lose their value. This makes most shares practically irredeemable for any value.

## Existing Code-Level Mitigation Analysis

*   **No Zero Check for Denominator in `createPosition`:** The `createPosition` function calculates `newShares` using `data.totalReceiptTokens` as the denominator in `mulDivDown` *without* explicitly checking if it's zero when `totalShares > 0`. This directly leads to the division-by-zero revert.
*   **No Protection for Users in `startUnwinding`:** The `startUnwinding` function calculates `userReceiptToken` and proceeds to burn the user's shares and initiate unwinding even if `userReceiptToken` is zero. There's no check to prevent users from burning valuable shares for nothing in this scenario.
*   **`applyLosses` Does Not Handle Shares:** The `applyLosses` function only adjusts `data.totalReceiptTokens`. It does not have a mechanism to deal with the outstanding `LockedPositionToken` shares if a bucket's value is entirely wiped out. These shares become "zombie shares" with no backing.

## Impact
**Critical.**
This vulnerability leads to:
1.  Permanent denial of service for new deposits into affected locking buckets.
2.  Total and permanent loss of users' locked `receiptToken` principal if their bucket's value is reduced to zero by losses.
3.  Potential for extreme unfairness and value extraction due to precision loss if `totalReceiptTokens` becomes very small but non-zero.

## Recommendations

1.  **In `createPosition`:**
    *   Add a check: If `data.totalReceiptTokens == 0` AND `IERC20(data.shareToken).totalSupply() > 0`, the function should revert, perhaps with a specific error like `BucketSlashedToZero()`. This prevents new deposits into a zero-value bucket that still has outstanding shares, thus avoiding the division-by-zero DoS.
    ```solidity
    // Inside createPosition, after reading totalShares and data.totalReceiptTokens
    if (data.totalReceiptTokens == 0 && totalShares > 0) {
        revert BucketSlashedToZero(); // Or similar error
    }
    // The existing newShares calculation:
    // uint256 newShares = totalShares == 0 ? _amount : _amount.mulDivDown(totalShares, data.totalReceiptTokens);
    // This line will still be safe if totalShares == 0 (first depositor)
    // If totalShares > 0, then the check above ensures totalReceiptTokens > 0.
    ```

2.  **In `startUnwinding`:**
    *   Add a check: After calculating `userReceiptToken`, if `userReceiptToken == 0` AND `_shares > 0` (meaning the user is trying to unwind actual shares but will get nothing back because `data.totalReceiptTokens` is zero or too small), the function should revert with an error like `NoValueToUnwind()` or `BucketSlashedToZero()`. This prevents users from unknowingly burning their shares for zero return.
    ```solidity
    // Inside startUnwinding, after calculating userReceiptToken
    if (userReceiptToken == 0 && _shares > 0 && data.totalReceiptTokens == 0) { // Check data.totalReceiptTokens to be sure
        revert NoValueToUnwindSharesSlashed(); // Or similar error
    }
    ```
    Alternatively, the shares could simply not be burned if `userReceiptToken` is zero, and the function could return zero, leaving the shares with the user. However, this might complicate accounting if the intent is to "clear out" valueless positions. Reverting seems safer to inform the user.

3.  **`applyLosses` Strategy for Zeroed Buckets (More Advanced):**
    *   When `applyLosses` causes `data.totalReceiptTokens` for a bucket to become zero (or fall below a dust threshold), consider a mechanism to "close" or "fully slash" the bucket.
    *   This might involve:
        *   Preventing any further calls to `createPosition` or `startUnwinding` for this bucket (other than perhaps a special withdrawal function).
        *   Potentially creating a mechanism for users to burn their now valueless shares, perhaps for a nominal gas refund or just for bookkeeping. This is complex and may not be necessary if the shares are simply understood to be worthless.
4.  **Handle Dust `totalReceiptTokens` (Precision Aspect):**
    *   To address the precision issue where `data.totalReceiptTokens` is extremely small (e.g., 1 wei):
        *   In `createPosition`: If `data.totalReceiptTokens` is below a certain dust threshold (but non-zero) and `totalShares` is large, the share calculation `_amount.mulDivDown(totalShares, data.totalReceiptTokens)` could lead to `newShares` being excessively large. Consider reverting or adjusting the calculation if the resulting share price (`data.totalReceiptTokens / totalShares`) is below a minimum threshold, to prevent minting huge numbers of shares for tiny amounts.
        *   In `startUnwinding`: If `data.totalReceiptTokens` is very small, `userReceiptToken` calculation `_shares.mulDivDown(data.totalReceiptTokens, totalShares)` might truncate to zero for users with small share amounts. This is a known issue with integer arithmetic and pro-rata distribution of very small remaining amounts. Clear communication or allowing users to burn shares for a proportional claim (even if tiny) might be options, though complex. The revert suggested in point 2 is a simpler safety measure.

Priority should be on preventing the DoS in `createPosition` and the direct loss of shares for zero value in `startUnwinding` when a bucket is fully depleted.
