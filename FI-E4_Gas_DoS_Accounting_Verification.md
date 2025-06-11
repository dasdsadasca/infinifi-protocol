# Verification of FI-E4: Gas DoS in `Accounting.totalAssetsValue()`

## Vulnerability
FI-E4: Gas DoS in `Accounting.totalAssetsValue()`. The `totalAssetsValue()` function (and similarly `totalAssetsValueOf()`) in `Accounting.sol` iterates through all registered assets and, for each asset, iterates through all its registered farms to sum their reported asset values. If the number of registered assets and/or farms becomes sufficiently large, the gas required for these nested loops and associated external calls can exceed the block gas limit, causing transactions that call this function (notably `YieldSharing.accrue()`) to revert.

## Certainty of Exploitability
**Certain, given a large enough number of registered assets and/or farms.**

The Gas DoS is a result of unbounded iteration. The gas cost of `totalAssetsValue()` scales linearly with the number of enabled assets and, more significantly, with the total number of farm entries across all assets. As the protocol grows and registers more assets and farms, the gas cost for this function will inevitably increase. It is certain that a point can be reached where this cost exceeds the block gas limit, making the function (and dependent functions like `YieldSharing.accrue()`) unusable.

## Analysis of Loop Structures, External Calls, and Gas Consumption

**1. `Accounting.totalAssetsValue()` Function:**
   *   Calls `FarmRegistry(farmRegistry).getEnabledAssets()` to get an array of all enabled asset addresses. The `FarmRegistry` uses OpenZeppelin's `EnumerableSet.AddressSet` for `assets`. The `.values()` function iterates the set to build an array; gas cost is proportional to `N_assets` (number of enabled assets).
   *   It then enters a `for` loop iterating `N_assets` times. In each iteration:
        a.  `IOracle(oracle[assets[i]]).price()`: Makes an external call to fetch the asset's price. (1 SLOAD + 1 external call).
        b.  `FarmRegistry(farmRegistry).getAssetFarms(assets[i])`: Makes an external call to get an array of farms for the current asset. This again uses `EnumerableSet.AddressSet.values()`; gas cost is proportional to `N_farms_for_asset_i`.
        c.  `_calculateTotalAssets(...)`: This internal function is called with the array of farms for `assets[i]`.

**2. `Accounting._calculateTotalAssets(address[] memory _farms)` Function:**
   *   This function iterates through the passed `_farms` array (length `N_farms_for_asset_i`).
   *   In each iteration: `_totalAssets += IFarm(_farms[index]).assets();`: Makes an external call to the `assets()` function of each farm.

**Summary of Gas-Intensive Operations:**
*   Fetching enabled assets array: Proportional to `N_assets`.
*   Outer loop (per asset):
    *   Oracle price call: Constant overhead per asset.
    *   Fetching farms array for that asset: Proportional to `N_farms_for_asset_i`.
    *   Inner loop (per farm of that asset):
        *   Farm `assets()` call: Constant overhead per farm (but farm's implementation can vary).

The total number of `farm.assets()` calls is the sum of all farms across all registered assets. The total number of `oracle.price()` calls is `N_assets`. The array creation for farms is called `N_assets` times (once per asset type).

**Estimated Gas Costs:**
*   `EnumerableSet.values()` for `K` items: Approx. `K * (500 to 1000+ gas)` depending on slot warmth.
*   External call (simple, e.g., `oracle.price`, simple `farm.assets`): Approx. `1000-5000 gas` (call overhead + SLOADs + execution). Complex farms can be more.

If `N_assets = 50` and average `N_farms_per_asset = 200`:
*   `getEnabledAssets()`: `50 * 1000 = 50,000 gas`.
*   Outer loop (50 iterations):
    *   Per asset:
        *   `oracle.price()`: `~2,500 gas`.
        *   `getAssetFarms(asset_i)` (200 farms): `200 * 1000 = 200,000 gas`.
        *   `_calculateTotalAssets(farms_for_asset_i)` (200 farms): `200 * ~4,000 gas/farm = 800,000 gas`.
        *   Subtotal per asset: `2,500 + 200,000 + 800,000 = ~1,002,500 gas`.
*   Total for outer loop: `50 * 1,002,500 = 50,125,000 gas`.
*   Grand Total: `50,000 + 50,125,000 = 50,175,000 gas`.
This significantly exceeds typical block gas limits (e.g., 30 million gas).

## Step-by-Step Scenario to Trigger DoS

*   **a. Setup State (Gradual Growth):**
    1.  Over time, the protocol governance enables a large number of asset types via `FarmRegistry.enableAsset()`. For instance, 50 different underlying assets (USDC, DAI, ETH, WBTC, various LSTs, etc.).
    2.  For each asset type, or for a few popular ones, many different yield-generating farms are registered via `FarmRegistry.addFarms()`. For instance, `USDC` might have 200 farms (Aave, Compound, Yearn, Convex, Pendle pools for USDC-backed PTs, etc.). Other assets also accumulate a substantial number of farms.
    3.  The total number of (asset, farm) pairs grows such that the scenario calculated above (e.g., 50 assets, averaging 200 farms each, or any combination leading to similar total iterations and calls) is reached.

*   **b. Triggering Transaction:**
    *   A keeper or any authorized entity calls `YieldSharing.accrue()`.
    *   Internally, `accrue()` calls `YieldSharing.unaccruedYield()`.
    *   `unaccruedYield()` calls `Accounting.totalAssetsValue()`.

*   **c. Gas Exhaustion:**
    *   `Accounting.totalAssetsValue()` begins execution.
    *   `FarmRegistry.getEnabledAssets()` constructs a large array (e.g., 50 assets).
    *   The main loop starts. For each asset:
        *   `FarmRegistry.getAssetFarms()` constructs another potentially large array (e.g., 200 farms for that asset).
        *   `_calculateTotalAssets()` then loops through this second array, making an external call to each farm's `assets()` function.
    *   Due to the sheer number of assets and farms, the cumulative gas cost from array constructions, external calls (each with its own overhead and execution cost), and loop overheads surpasses the block gas limit.
    *   The transaction calling `YieldSharing.accrue()` reverts with an out-of-gas error.

## Detailed Impact Assessment

1.  **`YieldSharing.accrue()` Blocked:** The most direct impact is that `YieldSharing.accrue()` becomes uncallable. This function is critical for:
    *   **Profit Distribution:** Realized profits from farms cannot be minted as `ReceiptToken`s and distributed to stakers (`StakedToken`) and lockers (`LockingController`). Users do not receive their yield.
    *   **Loss Socialization:** Realized losses from farms cannot be processed. The `safetyBuffer` cannot be used, `LockingController` and `StakedToken` cannot have `applyLosses` called, and if necessary, the `ReceiptToken` price cannot be devalued via its `FixedPriceOracle`.
2.  **Stale `unaccruedYield`:** The value returned by `unaccruedYield()` becomes fixed, as `accrue()` is the function that resets it (implicitly, by minting/burning tokens to match assets to supply).
3.  **Impact on Other Protocol Functions:**
    *   Functions that rely on `YieldSharing.unaccruedYield()` for safety checks, such as `maxRedeem`/`maxWithdraw` in `StakedToken.sol` or the `_revertIfThereAreUnaccruedLosses()` check in `InfiniFiGatewayV1.sol`, will operate on increasingly stale and potentially misleading data. If `unaccruedYield` was negative before the DoS, withdrawals could be permanently blocked. If it was positive, and then farms incur actual losses, the check might permit withdrawals that should have been subject to loss.
4.  **Drift in `ReceiptToken` Value:** If `accrue()` cannot run, the `ReceiptToken.totalSupply()` will not adjust to reflect the true value of underlying assets. The peg or backing of `ReceiptToken` can drift significantly, but the system has no mechanism to reconcile it.
5.  **System Financial Operations Halted:** Essentially, the core financial reconciliation and value distribution mechanism of the protocol is halted.

The impact is **Critical** because it breaks a core financial process, preventing yield distribution and loss socialization, and can lead to users being unable to withdraw or interact correctly due to stale safety checks.

## Existing Code-Level Mitigation Analysis

*   **`Accounting.sol`:** Contains no pagination, batching, or per-farm/per-asset incremental calculation mechanisms for `totalAssetsValue()`. It attempts a full summation in one call.
*   **`FarmRegistry.sol`:** Uses `EnumerableSet` from OpenZeppelin. While `EnumerableSet` allows fetching elements by index (`at(index)`), the `values()` function (which `getEnabledAssets`, `getAssetFarms`, etc., use to return arrays) iterates the entire set to construct the array in memory. There's no built-in support in these getter functions for returning a subset or paginated results directly from `FarmRegistry` that `Accounting` could then use.

There are no existing mitigations in the codebase to prevent this DoS scenario as the number of registered entities grows.

## Recommendations

1.  **Implement Paginated/Batched Processing for `totalAssetsValue()`:**
    *   Refactor `totalAssetsValue()` and its callers to process assets/farms in batches across multiple transactions if necessary.
    *   This would involve `Accounting.sol` needing to store intermediate sums and track progress. `FarmRegistry.sol` might need new functions to return subsets of assets/farms (e.g., `getAssets(startIndex, count)`).
    *   `YieldSharing.accrue()` would then need to be callable multiple times to complete a full accrual cycle if data is processed in batches. This is a significant architectural change.
2.  **Shift to Incremental Per-Farm or Per-Asset Accrual:**
    *   Instead of a global `totalAssetsValue()` for `accrue()`, consider if `YieldSharing` can operate by accruing yield/losses from one farm or one asset type at a time.
    *   A keeper could call a function like `accrueForFarm(address farmAddress)` or `accrueForAsset(address assetAddress)` repeatedly.
    *   This would require careful management of overall `unaccruedYield` or how `ReceiptToken` supply is adjusted.
3.  **Limit Number of Registered Entities (Pragmatic Short-Term Fix):**
    *   While not a true fix, impose administrative limits on the number of distinct assets and farms per asset that can be registered in `FarmRegistry`. This could prevent the DoS threshold from being reached but limits protocol growth. Clear warnings about gas implications should be provided if limits are high.
4.  **Off-Chain Computation for `totalAssetsValue` (Centralization Trade-off):**
    *   For view functions or less frequent calculations, consider if parts of `totalAssetsValue` could be computed off-chain and submitted by a trusted source if the on-chain cost is too high for frequent, essential operations. This is generally not suitable for core, trustless operations like `accrue()`.
5.  **Gas Cost Monitoring:** Implement off-chain monitoring to track the gas cost of `totalAssetsValue()` as new assets/farms are added, to anticipate when it might approach dangerous levels.
6.  **Optimize `FarmRegistry` Array Creation:**
    *   If `EnumerableSet.values()` is too costly for large sets, `FarmRegistry` could maintain its own explicit arrays that are updated during add/remove, though this adds complexity and gas to write operations. Alternatively, allow `Accounting` to read farm lists entry by entry using `EnumerableSet.at(index)` if batching is implemented.

The most robust solutions involve architectural changes to allow batched or incremental processing of asset values, rather than an unbounded monolithic summation.
