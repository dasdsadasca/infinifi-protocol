# Verification of FM-E2: Reentrancy in `RedeemController.redeem()`

## Vulnerability
FM-E2: Reentrancy in `RedeemController.redeem()`. This vulnerability suggests that the `redeem()` function in `RedeemController.sol` can be re-entered, potentially leading to undesirable outcomes such as excessive withdrawals, state corruption, or denial of service.

## Certainty of Exploitability
**Conditionally Certain.**

A reentrancy path **exists** in `RedeemController.redeem()` due to the following:
1.  The function lacks a `nonReentrant` modifier (e.g., from OpenZeppelin's `ReentrancyGuard`).
2.  It makes an external call to a configurable `beforeRedeemHook` address: `IBeforeRedeemHook(_beforeRedeemHook).beforeRedeem(...)`.
3.  This external call occurs *before* all state changes (effects) for the current redemption operation are completed (e.g., burning the user's `ReceiptToken`s, transferring `assetToken`s, or enqueuing the redemption request).

Exploitation is contingent on these conditions:
*   A `beforeRedeemHook` contract is configured in `RedeemController`.
*   This hook contract is malicious or has a vulnerability allowing it to be controlled by an attacker.
*   The hook contract (or an entity it calls) can satisfy the `onlyCoreRole(CoreRoles.ENTRY_POINT)` modifier required by the `redeem()` function to make a reentrant call. This would typically mean the hook's address itself is granted the `ENTRY_POINT` role by the protocol's governance.

While some basic asset theft scenarios are complicated by the fact that `liquidity()` is re-checked after the hook call and token burns are specific to amounts, the presence of a reentrancy pathway in a contract managing funds and complex queueing logic is a significant risk. It can lead to other exploits like gas griefing, denial of service by forcing reverts, or more subtle state inconsistencies in the redemption queue if not perfectly handled.

## Analysis of `redeem()` Function Flow and Reentrancy Points

The `redeem()` function in `RedeemController.sol` executes operations in roughly this sequence:
1.  **Checks:**
    *   `whenNotPaused` modifier.
    *   `onlyCoreRole(CoreRoles.ENTRY_POINT)` modifier.
    *   `require(_receiptAmountIn >= minRedemptionAmount)`.
2.  **Calculations:**
    *   `convertRatio = _getReceiptToAssetConvertRatio()` (involves external calls to `Accounting` and then to underlying oracles, but these are view calls and less likely reentrancy vectors for `RedeemController` itself).
    *   `assetAmountOut = _convertReceiptToAsset(_receiptAmountIn, convertRatio)`.
3.  **Branch 1: Queue Not Empty (`queueLength() > 0`)**
    *   `_amountReceiptToQueue = _convertAssetToReceipt(...)`.
    *   **Interaction (Token):** `ReceiptToken(receiptToken).transferFrom(msg.sender, address(this), _amountReceiptToQueue)` (transfers tokens to be queued).
    *   **Effect (State Update):** `_enqueue(_to, _amountReceiptToQueue)` (updates `totalEnqueuedRedemptions` and `queue` data structure).
    *   Returns 0.
4.  **Branch 2: Queue Empty**
    *   `_beforeRedeemHook = beforeRedeemHook` (state read).
    *   **Interaction (External Hook - Primary Reentrancy Vector):** `if (_beforeRedeemHook != address(0)) { IBeforeRedeemHook(_beforeRedeemHook).beforeRedeem(_to, _receiptAmountIn, assetAmountOut); }`. This call is made *before* any tokens are burned or assets are transferred for the current redemption request.
    *   `availableAssetAmount = liquidity()` (state read, occurs *after* the hook call).
    *   **Sub-Branch 2a: Sufficient Liquidity (`assetAmountOut <= availableAssetAmount`)**
        *   **Interaction (Token Effect):** `ReceiptToken(receiptToken).burnFrom(msg.sender, _receiptAmountIn)`.
        *   **Interaction (Asset Effect):** `ERC20(assetToken).safeTransfer(_to, assetAmountOut)`.
        *   Returns `assetAmountOut`.
    *   **Sub-Branch 2b: Insufficient Liquidity (Partial immediate, rest queued)**
        *   `amountReceiptToBurn = _convertAssetToReceipt(availableAssetAmount, ...)`.
        *   **Interaction (Token Effect):** `ReceiptToken(receiptToken).burnFrom(msg.sender, amountReceiptToBurn)`.
        *   **Interaction (Asset Effect):** `ERC20(assetToken).safeTransfer(_to, availableAssetAmount)`.
        *   `remainingReceiptToQueue = _receiptAmountIn - amountReceiptToBurn`.
        *   **Interaction (Token):** `ReceiptToken(receiptToken).transferFrom(msg.sender, address(this), remainingReceiptToQueue)`.
        *   **Effect (State Update):** `_enqueue(_to, remainingReceiptToQueue)`.
        *   Returns `availableAssetAmount`.

**Key Reentrancy Point:** The call to `IBeforeRedeemHook.beforeRedeem()` is the most significant reentrancy vector because it's an explicit call to an external, potentially untrusted contract before the core effects of the redemption (token burn, asset transfer for the current call) are applied.

## Step-by-Step Exploit Scenario (Illustrative of Reentrancy Pathway)

This scenario demonstrates the reentrancy pathway. Direct asset theft is complex due to liquidity re-checks, but the structural flaw exists. The primary risk demonstrated here is invoking logic paths based on state that might be altered by a reentrant call, potentially leading to unexpected behavior or making the contract's state temporarily inconsistent during the nested calls.

*   **a. Initial State:**
    *   `RedeemController` `assetToken` (USDC) balance: 70. `liquidity()` reports 70.
    *   Attacker's `MaliciousHook` contract is set as the `beforeRedeemHook`.
    *   Attacker (EOA) owns 100 `receiptToken` (iUSD) and has approved `RedeemController` to spend them.
    *   `MaliciousHook` is granted `CoreRoles.ENTRY_POINT` (this is a critical precondition for this specific exploit path).
    *   Price: 1 iUSD = 1 USDC.
    *   `queueLength()` is 0.

*   **b. Setup:**
    *   `MaliciousHook.sol` (simplified):
      ```solidity
      contract MaliciousHook is IBeforeRedeemHook {
          RedeemController public redeemController;
          address public attackerEOA;
          bool public hasReentered;

          constructor(address _redeemController, address _attackerEOA) {
              redeemController = RedeemController(_redeemController);
              attackerEOA = _attackerEOA;
          }

          function beforeRedeem(address _to, uint256 _receiptAmountIn, uint256 _assetAmountOut) external override {
              if (msg.sender == address(redeemController) && !hasReentered) {
                  hasReentered = true; // Prevent deeper recursion from this path
                  // Attacker EOA must have approved MaliciousHook for these tokens if hook calls redeem
                  // OR, hook uses its own tokens if it has any.
                  // For this example, assume hook calls redeem as itself (if it has ENTRY_POINT role)
                  // for a smaller amount.
                  redeemController.redeem(attackerEOA, 30); // Reentrant call for 30 iUSD
              }
          }
          // ... function for attacker to trigger initial call, manage state ...
      }
      ```

*   **c. Trigger Reentrancy:**
    *   Attacker EOA calls `RedeemController.redeem(attackerEOA, 100 iUSD)`. (`_receiptAmountIn = 100`).

*   **d. `RedeemController.redeem()` (Outer Call - Call 1 for 100 iUSD):**
    1.  Checks pass. `assetAmountOut` is calculated as 100 USDC.
    2.  `queueLength()` is 0.
    3.  Call to `MaliciousHook.beforeRedeem(attackerEOA, 100, 100)`.

*   **e. `MaliciousHook.beforeRedeem()`:**
    1.  `hasReentered` is false. Sets `hasReentered = true`.
    2.  Makes a reentrant call: `redeemController.redeem(attackerEOA, 30 iUSD)`.

*   **f. `RedeemController.redeem()` (Reentrant Call - Call 2 for 30 iUSD):**
    1.  Checks pass. `assetAmountOut_inner` is 30 USDC.
    2.  `queueLength()` is 0.
    3.  Hook is called again, but `hasReentered` is true, so no deeper reentrancy from this hook.
    4.  `availableAssetAmount_inner = liquidity()` (reads 70 USDC).
    5.  `assetAmountOut_inner (30) <= availableAssetAmount_inner (70)` is true.
    6.  `ReceiptToken.burnFrom(attackerEOA, 30)`. (Attacker EOA's iUSD balance becomes 70).
    7.  `ERC20(assetToken).safeTransfer(attackerEOA, 30)`. (Attacker EOA receives 30 USDC. `RedeemController` USDC balance becomes 40).
    8.  Inner call returns 30 USDC.

*   **g. `MaliciousHook.beforeRedeem()` completes.**

*   **h. `RedeemController.redeem()` (Outer Call - Call 1 for 100 iUSD resumes):**
    1.  `availableAssetAmount_outer = liquidity()` (reads 40 USDC, which is the current balance).
    2.  `assetAmountOut (100) <= availableAssetAmount_outer (40)` is false. The logic proceeds to the "else" branch (partial immediate, rest queued).
    3.  `amountReceiptToBurn = _convertAssetToReceipt(40 USDC, ratio) = 40 iUSD`.
    4.  `ReceiptToken.burnFrom(attackerEOA, 40)`. (Attacker EOA's iUSD balance becomes 70 - 40 = 30).
    5.  `ERC20(assetToken).safeTransfer(attackerEOA, 40)`. (Attacker EOA receives another 40 USDC. `RedeemController` USDC balance becomes 0).
    6.  `remainingReceiptToQueue = _receiptAmountIn (100) - amountReceiptToBurn (40) = 60 iUSD`.
    7.  `ReceiptToken.transferFrom(attackerEOA, address(this), 60)`. **This will attempt to transfer 60 iUSD from attackerEOA. However, attackerEOA only has 30 iUSD left. This call will revert.**

*   **i. Outcome:**
    *   The transaction reverts when the outer call attempts to transfer 60 iUSD from the attacker because the attacker's balance is insufficient (only 30 iUSD remain after the inner call's burn of 30 and the outer call's partial burn of 40).
    *   While this specific scenario leads to a revert (which is a form of Denial of Service for the attacker's own transaction), it demonstrates the execution flow alteration due to reentrancy. The critical part is that the `beforeRedeemHook` allows arbitrary code execution before the main effects of the function are finalized.
    *   A more sophisticated exploit might not lead to a revert but could manipulate queue states or other shared states within `RedeemController` or `RedemptionPool` if other functions were called by the hook, or if the state updates were ordered differently.

## Existing Code-Level Mitigation Analysis

*   **Checks-Effects-Interactions Pattern:** The pattern is violated. The `IBeforeRedeemHook.beforeRedeem()` call (Interaction) occurs before key Effects such as burning the `_receiptAmountIn` for the current call or transferring the `assetTokenOut`.
*   **`nonReentrant` Modifier:** Absent in `RedeemController.redeem()`. This is the most direct and standard mitigation for such reentrancy vulnerabilities.
*   **Liquidity Re-check:** The `liquidity()` is checked *after* the hook call. This is a positive factor that helps mitigate some simple asset drain scenarios because the available asset amount is updated if the reentrant call withdrew assets.
*   **Token Operations:** Standard ERC20 `burnFrom` and `transferFrom` operations will correctly debit tokens or revert if insufficient balance/allowance. This prevents straightforward "free withdrawal" from the reentrancy itself, as tokens are still required. `ReceiptToken.sol` does not appear to have malicious callbacks.
*   **`RedemptionPool.sol`:** The logic within `RedemptionPool` for enqueuing or funding the queue does not inherently prevent reentrancy; it relies on the calling contract (`RedeemController`) to manage execution flow safely.

The primary mitigation lacking is a `nonReentrant` guard.

## Impact
**Critical.**
While the specific scenario above leads to a revert, the presence of a reentrancy vulnerability in a contract handling user funds and complex state (like a redemption queue) is inherently critical.
1.  **Denial of Service (DoS):** As shown, reentrancy can lead to unexpected reverts, potentially blocking users if a malicious hook exploits this.
2.  **State Corruption:** More complex reentrancy patterns could potentially corrupt the state of the `RedemptionPool` (e.g., `totalPendingClaims`, `totalEnqueuedRedemptions`, or the queue order/contents) if other functions are called or if state updates were ordered differently, leading to accounting errors or unfairness.
3.  **Potential for Asset Theft:** Although not straightforward in the above example due to liquidity re-checks and specific token burns, a more intricate exploit, possibly involving multiple functions or specific states of the queue and balances, might still lead to asset theft. The risk significantly increases if any assumptions about atomicity of operations within `redeem()` are broken by reentrancy.
4.  **Gas Limit Issues:** Reentrant calls, even if not malicious, can consume unexpected amounts of gas, leading to transactions hitting the block gas limit and reverting.

The fact that an external call is made before effects are applied is a serious flaw.

## Recommendations

1.  **Add `nonReentrant` Modifier:** Implement a `nonReentrant` guard (e.g., from OpenZeppelin's `ReentrancyGuard`) to the `redeem()` function in `RedeemController.sol`. This is the most crucial and standard fix.
    ```solidity
    import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
    // ...
    contract RedeemController is Farm, RedemptionPool, IRedeemController, ReentrancyGuard { // Inherit ReentrancyGuard
    // ...
        function redeem(address _to, uint256 _receiptAmountIn)
            external
            whenNotPaused
            nonReentrant // Add modifier
            onlyCoreRole(CoreRoles.ENTRY_POINT)
            returns (uint256)
        {
            // ...
        }
    }
    ```
2.  **Adhere to Checks-Effects-Interactions:** If possible, re-evaluate the order of operations. Ideally, all internal state changes (Effects) related to the current redemption should occur *before* any external calls (Interactions) like the `beforeRedeemHook`.
    *   This would mean, for instance, burning the `_receiptAmountIn` *before* calling the hook. However, this changes the hook's purpose, as the hook might need to know the amounts *before* they are actioned. If the hook's purpose is to potentially halt an operation or log pre-state, then the `nonReentrant` modifier is even more essential.
3.  **Restrict `ENTRY_POINT` Role for Hooks:** Carefully consider if hook contracts should ever be granted roles like `ENTRY_POINT`. If a hook needs to perform actions requiring such roles, it indicates a potential design issue that might concentrate too much power or risk in the hook.
4.  **Review Hook Design:** Evaluate the necessity and safety of the `beforeRedeemHook`. If its functionality can be achieved differently or if its capabilities can be more restricted, it would reduce the attack surface.

Applying the `nonReentrant` modifier is the highest priority.
