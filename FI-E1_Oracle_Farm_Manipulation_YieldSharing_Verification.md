# Verification of FI-E1: Exploitable Oracle/Farm Asset Reporting Manipulation in `YieldSharing.sol`

## Vulnerability
FI-E1: Exploitable Oracle/Farm Asset Reporting Manipulation affecting `YieldSharing.sol`. This vulnerability posits that an attacker can manipulate the inputs to `YieldSharing.unaccruedYield()`—either `Accounting.totalAssetsValue()` (via oracle price manipulation or direct manipulation of a farm's reported `assets()`) or `Accounting.price(receiptToken)`—to cause `YieldSharing.accrue()` to mint unbacked `ReceiptToken`s or unfairly slash/devalue existing `ReceiptToken`s.

## Certainty of Exploitability
**Certain, assuming underlying inputs to `YieldSharing.accrue()` can be manipulated.**

The `YieldSharing.accrue()` function directly uses values from `Accounting.sol` to determine yield. If `Accounting.totalAssetsValue()` or `Accounting.price(receiptToken)` can be significantly altered by an attacker within a short timeframe before `accrue()` is called, then `YieldSharing` will act on this manipulated data. Such manipulation is:
*   **Certain for oracle prices:** If oracles providing prices for farm assets or the `receiptToken` itself are vulnerable (e.g., spot DEX prices), they can be manipulated (as detailed in FM-E1).
*   **Plausible for farm `assets()` values:** Depending on the specific farm implementation, its reported `assets()` might be temporarily inflatable/deflatable (e.g., through flash loans if `assets()` naively returns `balanceOf(this)` and deposits are not carefully managed relative to `accrue()` calls, or if the farm depends on other manipulatable on-chain values).

`YieldSharing.sol` and `Accounting.sol` currently lack mechanisms to detect or mitigate such sudden, large, malicious fluctuations in reported values.

## Analysis of `unaccruedYield()` and `accrue()` Logic

1.  **`YieldSharing.unaccruedYield()`:**
    *   Calculates: `(Accounting.totalAssetsValue() / Accounting.price(receiptToken)) - ReceiptToken.totalSupply()`.
    *   This determines the protocol's overall profit or loss in terms of `receiptToken` value.

2.  **`YieldSharing.accrue()`:**
    *   Calls `unaccruedYield()` to get the `yield`.
    *   If `yield > 0` (profit): Calls `_handlePositiveYield(yield)`.
        *   `_handlePositiveYield` mints this `yield` amount of `ReceiptToken`s to the `YieldSharing` contract.
        *   These newly minted tokens are then distributed: first to replenish a `safetyBufferSize`, then a `performanceFee` is taken, and the remainder is split between `StakedToken` holders (via `StakedToken.depositRewards()`) and `LockingController` users (via `LockingController.depositRewards()`).
    *   If `yield < 0` (loss): Calls `_handleNegativeYield(abs(yield))`.
        *   `_handleNegativeYield` attempts to cover the loss by:
            1.  Burning `ReceiptToken`s from its own `safetyBuffer`.
            2.  If loss persists, calling `LockingController.applyLosses()`.
            3.  If loss still persists, calling `StakedToken.applyLosses()`.
            4.  If loss *still* persists, it devalues the `receiptToken` by calling `setPrice()` on its `FixedPriceOracle` (assuming `receiptToken` uses one), effectively socializing the loss among all `ReceiptToken` holders.

## Identify Manipulation Vectors for Inputs to `unaccruedYield()`

*   **`Accounting.totalAssetsValue()`:** This is the sum of values of all assets in all registered farms.
    `value_farm_i = IFarm_i.assets() * Accounting.price(asset_farm_i)`.
    *   **Vector 1: Manipulating `Accounting.price(asset_farm_i)`:** An attacker can manipulate the oracle for a significant farm's underlying asset (e.g., `asset_X`) to report an artificially high or low price. This was covered in FM-E1.
    *   **Vector 2: Manipulating `IFarm_i.assets()`:**
        *   If a farm's `assets()` method simply returns `underlyingToken.balanceOf(address(this))`, an attacker could flash-loan a large amount of `underlyingToken`, deposit it into the farm (if `deposit()` is permissionless or compromised), trigger `accrue()`, and then withdraw. This would temporarily inflate `assets()`.
        *   For `PendleV2Farm.assets()`: `assetTokenBalance` (e.g., USDC held by the farm) is part of the calculation. If `accrue()` is called after a large deposit to the farm but before the farm converts these assets to Pendle PTs (via the admin-only `wrapAssetToPt`), `assets()` would be inflated. This is a timing/front-running vector, potentially abusable if admin actions are predictable or if there's a delay. The `_interpolatingYield` component also depends on Pendle oracles, which could be another vector if they are manipulatable.

*   **`Accounting.price(receiptToken)`:**
    *   If the `receiptToken` (e.g., iUSD) is priced by a `FixedPriceOracle` (as implied by `_handleNegativeYield`), manipulation requires `ORACLE_MANAGER` compromise or error.
    *   If it were priced by a market-based oracle (e.g., an iUSD/USDC DEX pool), this price could be directly manipulated. Inflating this price would reduce `assetsInReceiptTokens`, potentially causing artificial losses. Deflating it would increase `assetsInReceiptTokens`, potentially causing artificial profits.

## Step-by-Step Exploit Scenario (Inflating Yield for Unfair Gain)

**Assumptions:**
*   Attacker can manipulate the price of `assetMajor`, a key asset in `FarmMajor`.
*   `receiptToken` price is stable at $1.00.
*   Attacker holds `ReceiptToken`s and is staked in `StakedToken`.

*   **a. Initial State:**
    *   `ReceiptToken.totalSupply() = 2,000,000 * 1e18`.
    *   `Accounting.price(receiptToken) = 1e18` ($1.00).
    *   `FarmMajor` holds `10,000 assetMajor`. Real price of `assetMajor` = $100. Oracle for `assetMajor` correctly reports $100. Value from `FarmMajor` = `10,000 * $100 = $1,000,000`.
    *   Other farms contribute $1,000,000.
    *   `Accounting.totalAssetsValue() = $1,000,000 (FarmMajor) + $1,000,000 (Others) = $2,000,000`.
    *   Expected `unaccruedYield = ($2,000,000 / $1) - 2,000,000 = 0`.

*   **b. Manipulation Phase (just before `accrue()` call):**
    1.  Attacker flash-loans funds and manipulates the oracle for `assetMajor` to temporarily report its price as $200 (instead of $100).
    2.  `Accounting.totalAssetsValue()` is now calculated as:
        *   Value from `FarmMajor` = `10,000 assetMajor * $200 = $2,000,000` (artificially inflated by $1,000,000).
        *   Value from other farms = $1,000,000.
        *   Manipulated `Accounting.totalAssetsValue() = $2,000,000 + $1,000,000 = $3,000,000`.

*   **c. Trigger `accrue()`:**
    *   A keeper (or the attacker, if `accrue` is permissionless) calls `YieldSharing.accrue()`.

*   **d. Exploitation within `accrue()`:**
    1.  `unaccruedYield()` calculates:
        *   `assetsInReceiptTokens = $3,000,000 (manipulated) / $1.00 = 3,000,000 * 1e18`.
        *   `yield = (3,000,000 * 1e18) - (2,000,000 * 1e18) = 1,000,000 * 1e18` (a false profit of 1M iUSD).
    2.  `_handlePositiveYield(1000000e18)` is called.
    3.  `ReceiptToken.mint(address(YieldSharing), 1000000e18)` occurs. These are largely unbacked `ReceiptToken`s.
    4.  These 1,000,000 new `ReceiptToken`s are distributed (e.g., to safety buffer, fees, `StakedToken`, `LockingController`). Attacker receives a share via their stake in `StakedToken`.

*   **e. Profit:**
    1.  Attacker restores the oracle for `assetMajor` to $100 (e.g., by repaying flash loan and reversing DEX manipulation).
    2.  The actual `totalAssetsValue` of the protocol is still ~$2,000,000. However, `ReceiptToken.totalSupply()` is now `3,000,000e18`.
    3.  The attacker's share of the 1,000,000 fraudulently minted `ReceiptToken`s can be sold on the market or redeemed (if redemption does not use the same manipulated oracles or if done quickly). This extracts real value from the protocol (diluting other holders or draining reserves). The next `accrue()` call would likely show a massive loss.

## Step-by-Step Exploit Scenario (Inducing Unfair Slashing/Devaluation)

**Assumptions:** Similar initial state. Attacker will deflate an asset's price.

*   **a. Initial State:** As above. `totalAssetsValue = $2,000,000`, `totalSupply = 2,000,000 iUSD`.

*   **b. Manipulation Phase:**
    1.  Attacker manipulates the oracle for `assetMajor` to report $0 (instead of $100).
    2.  `Accounting.totalAssetsValue()` is now:
        *   Value from `FarmMajor` = `10,000 assetMajor * $0 = $0` (artificially deflated by $1,000,000).
        *   Value from other farms = $1,000,000.
        *   Manipulated `Accounting.totalAssetsValue() = $0 + $1,000,000 = $1,000,000`.

*   **c. Trigger `accrue()`:** Keeper/Attacker calls `YieldSharing.accrue()`.

*   **d. Exploitation within `accrue()`:**
    1.  `unaccruedYield()` calculates:
        *   `assetsInReceiptTokens = $1,000,000 (manipulated) / $1.00 = 1,000,000 * 1e18`.
        *   `yield = (1,000,000 * 1e18) - (2,000,000 * 1e18) = -1,000,000 * 1e18` (a false loss of 1M iUSD).
    2.  `_handleNegativeYield(1000000e18)` is called.
    3.  The safety buffer is depleted.
    4.  `LockingController.applyLosses()` is called, unfairly slashing lockers.
    5.  `StakedToken.applyLosses()` is called, unfairly slashing stakers.
    6.  If the loss is larger than what lockers and stakers can absorb, `FixedPriceOracle(receiptTokenOracle).setPrice()` is called, devaluing `receiptToken` for all holders based on false data.

*   **e. Profit/Impact:**
    *   Legitimate users are unfairly slashed or their `ReceiptToken`s devalued.
    *   Attacker could profit by:
        *   Shorting `ReceiptToken` before the manipulation.
        *   Buying `ReceiptToken` at the artificially lowered price after the manipulation, then waiting for recovery.
        *   Causing cascading liquidations if the `ReceiptToken` or its LP positions are used as collateral elsewhere.

## Existing Code-Level Mitigation Analysis within `YieldSharing`/`Accounting`

*   **`YieldSharing.sol`:**
    *   Does not have any mechanisms to validate the `yield` calculated from `unaccruedYield()`. It trusts the input.
    *   No checks for excessive yield (positive or negative) compared to previous periods or a reasonable threshold.
*   **`Accounting.sol`:**
    *   `totalAssetsValue()` directly sums values based on current oracle prices and farm-reported `assets()`.
    *   No use of Time-Weighted Average Prices (TWAPs) for oracle inputs.
    *   No smoothing or sanity checks for the `assets()` values reported by farms (e.g., rate of change limits).
    *   The security relies entirely on:
        1.  The robustness of every registered oracle.
        2.  The robustness of every registered farm's `assets()` reporting.

If any of these external dependencies can be manipulated, `YieldSharing` is vulnerable.

## Impact
**Critical.**
Exploiting this vulnerability allows an attacker to:
1.  **Mint unbacked `ReceiptToken`s:** By inflating `totalAssetsValue`, the attacker causes `YieldSharing` to mint tokens that are not backed by a corresponding increase in real asset value. This dilutes legitimate `ReceiptToken` holders and allows the attacker (if they are a recipient of these new tokens via staking/locking) to extract value.
2.  **Trigger unfair slashing or devaluation:** By deflating `totalAssetsValue` or manipulating `receiptTokenPrice`, the attacker can cause `YieldSharing` to incorrectly register losses, leading to the burning of tokens from the safety buffer, slashing of user positions in `LockingController` and `StakedToken`, or an unwarranted devaluation of `ReceiptToken` itself. This harms legitimate users directly.

This vulnerability can lead to significant financial loss for users and the protocol, and can break the core accounting and yield/loss distribution mechanisms.

## Recommendations

1.  **Mandate Robust Oracles:** (Reiteration of FM-E1 advice)
    *   For all farm assets and critically for the `receiptToken` itself (if not using `FixedPriceOracle`), **only use manipulation-resistant oracles** (e.g., Chainlink, robust TWAPs over sufficiently long periods). This is the most important mitigation.
2.  **Strengthen Farm `assets()` Reporting:**
    *   Each farm's `assets()` implementation must be resilient to same-block manipulation.
    *   Avoid direct `balanceOf(this)` if deposits/withdrawals can temporarily inflate/deflate this without reflecting true deployed value. Use internal accounting within farms for "deployed assets."
    *   If farms rely on external DEXes or oracles for their own internal calculations that feed into `assets()`, these must also be robust.
3.  **Consider Delayed Accrual / Oracle Readings (Defense-in-Depth for `Accounting.sol`):**
    *   This is complex, but `Accounting.sol` could potentially incorporate a mechanism to read oracle prices from N blocks in the past or use a short TWAP for prices it consumes. This makes flash loan based oracle manipulation harder but introduces latency.
4.  **Rate Limiting or Circuit Breakers in `YieldSharing.sol` (Defense-in-Depth):**
    *   `YieldSharing.accrue()` could implement checks to ensure the calculated `yield` (positive or negative) is within some reasonable bounds compared to, for example, `ReceiptToken.totalSupply()` or the change since the last accrual.
    *   If `abs(yield) / totalSupply > X%` (where X is a configurable threshold), the accrual could revert or require privileged approval. This can prevent extreme manipulations but needs careful tuning to avoid blocking legitimate large yields/losses.
5.  **Staleness Checks for Oracles in `Accounting.sol`:**
    *   Ensure that oracles provide recent prices. `Accounting.price()` could revert if an oracle's price is too stale, although this is often best handled by the oracle implementation itself.
6.  **Monitoring and Alerting:** Off-chain monitoring for drastic changes in `totalAssetsValue` or individual farm values reported to `Accounting` immediately before `accrue()` calls can help detect manipulation attempts.

The primary solution lies in ensuring the integrity of the inputs to `Accounting.totalAssetsValue()` and `Accounting.price(receiptToken)`. Without robust inputs, `YieldSharing` will make incorrect decisions.
