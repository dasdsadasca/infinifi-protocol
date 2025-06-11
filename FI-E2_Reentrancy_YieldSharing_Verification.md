# Verification of FI-E2: Reentrancy in `YieldSharing.accrue()`

## Vulnerability
FI-E2: Reentrancy in `YieldSharing.accrue()`. This vulnerability suggests that the `accrue()` function, along with its internal helpers `_handlePositiveYield()` and `_handleNegativeYield()`, can be re-entered. This could lead to the same calculated yield (especially negative yield) being processed multiple times, causing excessive minting, burning, slashing of user funds, or `ReceiptToken` price devaluation.

## Certainty of Exploitability
**Certain, given a reentrant external target.**

A reentrancy path exists and is exploitable if any of the external contracts called during the `accrue()` process (particularly within `_handleNegativeYield`, such as `LockingController.applyLosses()`, `StakedToken.applyLosses()`, or a malicious `FixedPriceOracle.setPrice()`) can make a call back into `YieldSharing.accrue()` before the initial call has completed its state changes. The `accrue()` function lacks a `nonReentrant` modifier.

## Analysis of `accrue()`, `_handlePositiveYield()`, and `_handleNegativeYield()` Flow

1.  **`accrue()` Function:**
    *   Calculates `int256 yield = unaccruedYield();`. This involves external calls to `Accounting.price()`, `Accounting.totalAssetsValue()`, and `ReceiptToken.totalSupply()`.
    *   If `yield > 0`, it calls `_handlePositiveYield(uint256(yield))`.
    *   If `yield < 0`, it calls `_handleNegativeYield(uint256(-yield))`.

2.  **`_handlePositiveYield(uint256 _positiveYield)`:**
    *   Reads states from `StakedToken`, `ReceiptToken`, `LockingController` (external calls).
    *   **Effect:** `ReceiptToken(receiptToken).mint(address(this), _positiveYield);`. This crucial state change (increasing `totalSupply`) happens relatively early.
    *   **Interactions (Potential Reentrancy Points after mint):**
        *   `ReceiptToken(receiptToken).transfer(performanceFeeRecipient, fee);`
        *   `StakedToken(stakedToken).depositRewards(stakingProfit);`
        *   `LockingController(lockingModule).depositRewards(lockingProfit);`
    *   **Reentrancy Impact on Positive Yield:** If `accrue()` is re-entered via these later interactions, the `unaccruedYield()` in the re-entrant call will use the *updated* `totalSupply` (which now includes `_positiveYield` from the outer call). This makes the re-calculated yield smaller or zero, effectively preventing the same positive yield from being minted multiple times. This part appears relatively robust against reentrancy causing repeated minting of the *same* yield amount.

3.  **`_handleNegativeYield(uint256 _negativeYield)`:**
    *   Local variable `_negativeYield` tracks the remaining loss to be applied.
    *   **Path 1 (Safety Buffer):**
        *   Reads `ReceiptToken.balanceOf(address(this))`.
        *   **Interaction/Effect:** `ReceiptToken(receiptToken).burn(_negativeYield_portion);`. If re-entered from here, `totalSupply` is updated, and re-calculated `unaccruedYield` becomes less negative. Robust.
    *   **Path 2 (Locking Controller):**
        *   If loss remains: `_negativeYield` is updated.
        *   Calls `LockingController(lockingModule).totalBalance()` (External Read).
        *   **Interaction:** `LockingController(lockingModule).applyLosses(loss_for_locking);` **(Potential Reentrancy Vector)**. `loss_for_locking` is the smaller of current `_negativeYield` or `locking_balance`.
        *   `_negativeYield -= loss_for_locking_applied_by_controller;` (This subtraction happens *after* the external call returns).
    *   **Path 3 (Staked Token):**
        *   If loss remains: Similar logic, calls `StakedToken(stakedToken).applyLosses(loss_for_staking);` **(Potential Reentrancy Vector)**.
        *   `_negativeYield -= loss_for_staking_applied_by_staked_token;`
    *   **Path 4 (Devalue ReceiptToken):**
        *   If loss remains:
        *   Calls `Accounting(accounting).oracle(receiptToken)`, `FixedPriceOracle(oracle).price()` (External Reads).
        *   **Interaction:** `FixedPriceOracle(oracle).setPrice(newPrice);` **(Potential Reentrancy Vector)**.
    *   **Reentrancy Impact on Negative Yield:** The critical issue is that the local `_negativeYield` variable in the outer call is not updated until *after* an external interaction (e.g., `LockingController.applyLosses`) returns. If this interaction re-enters `accrue()`, the inner `accrue()` will calculate `unaccruedYield()`. While `totalSupply` might have been reduced by prior steps (e.g., safety buffer burn), it won't reflect the burn that the *outer call's current step* (e.g., `LockingController.applyLosses`) is *supposed* to perform. The inner call then proceeds through its own `_handleNegativeYield` logic, potentially applying losses to the same tranches (e.g., `StakedToken`, or devaluing `ReceiptToken` price) based on a similar remaining loss amount. When the outer call resumes, it continues with its stale `_negativeYield` value, applying losses again.

## Step-by-Step Exploit Scenario (Compounded Slashing via `_handleNegativeYield`)

**Assumptions:**
*   An attacker can make `LockingController.applyLosses()` call back into `YieldSharing.accrue()`. This might be via a malicious `LockingController` contract set by a compromised governor, or if the `receiptToken` used by `LockingController.applyLosses` for its internal burn was an ERC777 token controlled by the attacker (though `ReceiptToken.sol` is standard). We assume the former for simplicity.
*   `YieldSharing.accrue()` is callable by anyone (or by the attacker).

*   **a. Initial State:**
    *   `Accounting.totalAssetsValue() = 1,800,000 * 1e18`.
    *   `ReceiptToken.totalSupply() = 2,000,000 * 1e18`.
    *   `Accounting.price(receiptToken) = 1e18` ($1.00).
    *   Calculated `unaccruedYield()` = `(1.8M / 1) - 2M = -200,000 * 1e18`.
    *   `YieldSharing.safetyBufferSize = 0` (or already depleted).
    *   `MaliciousLockingController.totalBalance() = 100,000 * 1e18`.
    *   `StakedToken` holds `100,000 * 1e18` `ReceiptToken`s.
    *   `MaliciousLockingController` is configured as the `lockingModule`.

*   **b. Setup:**
    *   `MaliciousLockingController.applyLosses(uint256 amount)` is crafted to call `YieldSharing.accrue()` reentrantly before performing its own token burns. It will claim to have burned/processed the `amount`.

*   **c. Trigger `accrue()` (Outer Call):**
    1.  `YieldSharing.accrue()` is called.
    2.  `yield = unaccruedYield()` calculates `-200,000e18`.
    3.  `_handleNegativeYield(200000e18)` is invoked.
    4.  Safety buffer is 0. `_negativeYield` remains `200,000e18`.
    5.  `lockingReceiptTokens = MaliciousLockingController.totalBalance() = 100,000e18`.
    6.  Since `_negativeYield (200k) > lockingReceiptTokens (100k)`, the code will attempt to apply `100,000e18` loss to `MaliciousLockingController`.
    7.  Call `MaliciousLockingController.applyLosses(100000e18)`. **(Interaction - Reentrancy Point)**.
    8.  The local `_negativeYield` in this outer `_handleNegativeYield` is *not yet reduced*. It's still `200,000e18` before this line, and will be reduced to `200k - 100k = 100k` *after* this call returns.

*   **d. Reentrant Call via `MaliciousLockingController`:**
    1.  `MaliciousLockingController.applyLosses()` immediately calls `YieldSharing.accrue()` (Inner Call).
    2.  `yield_inner = unaccruedYield()`:
        *   `totalAssetsValue` and `receiptTokenPrice` are unchanged.
        *   `ReceiptToken.totalSupply()` is still `2,000,000e18` (MaliciousLockingController hasn't burned tokens yet for the outer call).
        *   So, `yield_inner` is still `-200,000e18`.
    3.  `_handleNegativeYield(200000e18)` is invoked (Inner Call).
    4.  Safety buffer is 0. `_negativeYield_inner` remains `200,000e18`.
    5.  `lockingReceiptTokens_inner = MaliciousLockingController.totalBalance() = 100,000e18`.
    6.  `MaliciousLockingController.applyLosses(100000e18)` is called again (Inner context). (The malicious contract allows this, doesn't re-re-enter `accrue`, but also doesn't burn tokens or reduce its reported `totalBalance` for this example).
    7.  `_negativeYield_inner` is reduced: `200,000e18 - 100,000e18 = 100,000e18`.
    8.  Now, `StakedToken.applyLosses(100000e18)` is called (Inner context). `StakedToken` is honest, burns `100,000e18` of its `ReceiptToken`s. `ReceiptToken.totalSupply()` becomes `1,900,000e18`.
    9.  `_negativeYield_inner` (now `100,000e18` from `StakedToken`'s capacity) is reduced by `100,000e18` (amount `StakedToken` burned). `_negativeYield_inner` becomes `0`.
    10. Inner `_handleNegativeYield` finishes. Inner `accrue` finishes.

*   **e. `MaliciousLockingController.applyLosses()` (Outer call context) returns.** It effectively did nothing to `totalSupply`.

*   **f. `YieldSharing.accrue()` (Outer Call resumes):**
    1.  The call to `MaliciousLockingController.applyLosses(100000e18)` returns.
    2.  The outer `_handleNegativeYield` reduces its local `_negativeYield`: `200,000e18 - 100,000e18 (amount attributed to LockingController) = 100,000e18`.
    3.  Now, it proceeds to `StakedToken`. `stakedReceiptTokens = ReceiptToken.balanceOf(stakedToken)`. This balance is now `0` because the inner call already caused `StakedToken` to burn all its `100,000e18` `ReceiptToken`s.
    4.  `_negativeYield (100k)` is NOT `<= stakedReceiptTokens (0)`.
    5.  `StakedToken.applyLosses(0)` is called (as `stakedReceiptTokens` is 0). No more tokens burned here.
    6.  `_negativeYield` remains `100,000e18`.
    7.  This remaining `100,000e18` loss is now applied to `ReceiptToken` holders via price devaluation:
        *   `totalSupply = ReceiptToken.totalSupply()` is `1,900,000e18`.
        *   `oracle = Accounting.oracle(receiptToken)`. `price = FixedPriceOracle(oracle).price()` (still $1.00).
        *   `newPrice = 1e18 * (1.9M - 100k) / 1.9M = 1e18 * 1.8M / 1.9M = ~0.947e18`.
        *   `FixedPriceOracle(oracle).setPrice(~0.947e18)`. The `ReceiptToken` is devalued.

*   **Outcome:**
    *   `LockingController` was *supposed* to absorb 100k loss, but the malicious contract didn't actually burn tokens that would affect `totalSupply` before reentrancy.
    *   `StakedToken` absorbed 100k loss (correctly, but due to the inner call). In the outer call, it appears to absorb 0 more as its balance is already depleted.
    *   The remaining 100k loss (which should have been partially or fully covered by `LockingController` if it had behaved honestly and its burns were reflected before the reentrant `unaccruedYield` calculation) is unfairly passed on to cause a general devaluation of `ReceiptToken`.
    *   Effectively, `StakedToken` took a 100k hit, and then all `ReceiptToken` holders took an additional 100k hit via devaluation. The `LockingController`'s portion of the loss was effectively magnified onto other parties due to the reentrancy.

## Existing Code-Level Mitigation Analysis

*   **`nonReentrant` Modifier:** The `accrue()` function **lacks a `nonReentrant` modifier.** This is the primary oversight.
*   **Checks-Effects-Interactions (CEI) Pattern:**
    *   **`_handlePositiveYield`:** The `ReceiptToken.mint()` (Effect) occurs before the main distribution Interactions (`transfer` to fee recipient, `depositRewards` to `StakedToken` & `LockingController`). This is good and largely prevents reentrancy from duplicating positive yield minting.
    *   **`_handleNegativeYield`:** The local variable `_negativeYield` is passed through a chain of loss-absorbing steps. Each step involves an external call (Interaction, e.g., `LockingController.applyLosses`) *before* this local `_negativeYield` is definitively reduced by the *actual amount of tokens burned or value lost* within that external component *as reflected in global state like `totalSupply`*. If the external call re-enters `accrue`, `unaccruedYield` will be calculated using a `totalSupply` that doesn't account for the burn that the *outer instance* of `applyLosses` (or similar) was supposed to have just caused. This stale `totalSupply` (relative to the outer call's logic path) allows the reentrant call to miscalculate the *remaining* system-wide loss, leading to compounded effects.

The critical CEI violation in `_handleNegativeYield` is that the Effects on `totalSupply` (via external contract calls like `LockingController.applyLosses` which *should* burn tokens) are not completed and reflected before a reentrant call can recalculate `unaccruedYield` based on an insufficiently updated `totalSupply`.

## Impact
**Critical.**
A successful reentrancy attack on `accrue()` can lead to:
1.  **Compounded Application of Losses:** As shown in the scenario, negative yield can be applied multiple times to later tranches of loss absorbers (like `StakedToken` or all `ReceiptToken` holders via devaluation) because the reentrant call doesn't see the full effect of the loss absorption that should have occurred in the outer call's current step. This leads to excessive and unfair slashing/devaluation.
2.  **Integrity of Yield/Loss Accounting:** The system's accounting of where losses were applied or how profits were distributed can become corrupted.
3.  **Significant Financial Loss:** Users can lose more funds than warranted by actual protocol performance.
4.  **Denial of Service:** While not the primary goal, complex reentrancy could also lead to unexpected reverts or gas exhaustion.

## Recommendations

1.  **URGENT: Add `nonReentrant` Modifier:**
    *   Apply a `nonReentrant` modifier (e.g., from OpenZeppelin's `ReentrancyGuard`) to the `accrue()` function immediately. This is the most critical and direct mitigation.
    ```solidity
    import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

    contract YieldSharing is CoreControlled, ReentrancyGuard { // Inherit ReentrancyGuard
        // ...
        function accrue() external whenNotPaused nonReentrant { // Add modifier
            // ...
        }
    }
    ```

2.  **Refine State Handling in `_handleNegativeYield` (Checks-Effects-Interactions):**
    *   While `nonReentrant` is primary, ideally, ensure that before an external call like `LockingController.applyLosses(_amountToApply)`, the `_amountToApply` is first deducted from `_negativeYield` or the relevant `totalSupply` is updated, if possible, before the external call.
    *   However, this is complex because the actual amount burned/slashed is determined *by* the external contract. The `nonReentrant` guard is essential because `YieldSharing` has to trust the external contract to perform the action and then rely on its effects on `totalSupply` for the *next* `unaccruedYield` calculation, not for a reentrant one.
    *   The current pattern of:
        `lossAmountForX = calculate_loss_for_X_based_on_current_negativeYield_and_X_balance`
        `X.applyLosses(lossAmountForX)`
        `_negativeYield -= lossAmountForX` (or some reported actual loss from X)
      is inherently problematic if `X.applyLosses` re-enters, because the reentrant call won't see the effect of `_negativeYield` being reduced by `lossAmountForX` that the outer call *intends* to apply via X.

The `nonReentrant` modifier effectively ensures that the entire `accrue()` operation, including all its internal calls to `_handlePositiveYield` or `_handleNegativeYield` and subsequent external contract interactions, completes atomically without interference from reentrant calls trying to restart or interleave with the process based on stale intermediate states.
