# Verification of FM-E1: Exploitable Oracle Price Manipulation in Funding Module

## Vulnerability
FM-E1: Exploitable Oracle Price Manipulation in Funding Module. This vulnerability posits that an attacker can manipulate the price oracles used by the `MintController` and `RedeemController` to mint `ReceiptToken`s at an unfairly low price or redeem them for an unfairly high amount of `assetToken`.

## Certainty of Exploitability
**Conditionally Certain.**

The exploit is **certain** if the following condition holds:
*   Either the `assetToken` or the `receiptToken` (or both) is configured in `Accounting.sol` to use an oracle that is vulnerable to atomic manipulation within a single transaction. This typically includes oracles that:
    *   Read spot prices directly from on-chain Decentralized Exchanges (DEXes) like Uniswap V2/V3 without using robust Time-Weighted Average Prices (TWAPs).
    *   Rely on external oracle systems whose prices can be momentarily skewed by large on-chain actions (e.g., if the oracles used by `EthenaOracle`, `LevelOracle`, or `ResolvOracle` are themselves susceptible to such manipulation).

The provided codebase for `Accounting.sol`, `MintController.sol`, and `RedeemController.sol` does **not** contain inherent mitigations (like internal TWAP calculations on received prices, or price deviation circuit breakers) that would prevent exploitation if a vulnerable underlying oracle is used.

## Detailed Price Flow Analysis

Both `MintController.sol` and `RedeemController.sol` rely on `Accounting.sol` to fetch prices for `assetToken` and `receiptToken`.

**1. `MintController.assetToReceipt(uint256 _assetAmount)`:**
   *   `assetTokenPrice = Accounting(accounting).price(assetToken);`
   *   `receiptTokenPrice = Accounting(accounting).price(receiptToken);`
   *   The `Accounting.price(address _asset)` function calls `IOracle(oracle[_asset]).price()`, where `oracle[_asset]` is the specific oracle contract registered for that asset.
   *   `convertRatio = receiptTokenPrice.divWadUp(assetTokenPrice);`
   *   `receiptAmountOut = _assetAmount.divWadDown(convertRatio);`
   *   **Impact:** If `receiptTokenPrice` is manipulated downwards relative to `assetTokenPrice`, `convertRatio` decreases, leading to a larger `receiptAmountOut` for the same `_assetAmount`.

**2. `RedeemController.receiptToAsset(uint256 _receiptAmount)` (via internal helpers):**
   *   `_assetTokenPrice = Accounting(accounting).price(assetToken);`
   *   `_receiptTokenPrice = Accounting(accounting).price(receiptToken);`
   *   `convertRatio = _receiptTokenPrice.divWadDown(_assetTokenPrice);` (This is `_getReceiptToAssetConvertRatio`)
   *   `assetAmountOut = _receiptAmount.mulWadDown(convertRatio);` (This is `_convertReceiptToAsset`)
   *   **Impact:** If `receiptTokenPrice` is manipulated upwards relative to `assetTokenPrice`, `convertRatio` increases, leading to a larger `assetAmountOut` for the same `_receiptAmount`.

## Analysis of Each Oracle's Manipulability

The `Accounting.sol` contract allows any contract implementing the `IOracle` interface to be set as a price feed for an asset via `setOracle(address _asset, address _oracle)` by the `ORACLE_MANAGER`.

*   **`FixedPriceOracle.sol`:**
    *   **Source:** Admin-set fixed price.
    *   **Manipulability:** Not vulnerable to on-chain manipulation by external actors (assuming the `ORACLE_MANAGER` role is secure). This is a safe oracle if the price represents fair market value and is managed correctly.

*   **`EthenaOracle.sol`:**
    *   **Source:** `ERC4626(sUSDe).convertToAssets(1e18)`. Relies on the sUSDe vault's internal calculation of its share price.
    *   **Manipulability:** Depends on the robustness of the external Ethena sUSDe contract (at `0x9D39A5DE30e57443BfF2A8307A4256c8797A3497`). If large, atomic deposits/withdrawals into the sUSDe contract itself can temporarily skew its reported share price, this oracle becomes a vector. It does not use an independent DEX spot price.

*   **`LevelOracle.sol`:**
    *   **Source:** `ILevelReserveLens(levelReserveLens).getReservePrice()`. Relies on Level Finance's `ReserveLens` contract.
    *   **Manipulability:** Depends on the robustness of the external `LevelReserveLens` contract (at `0x29759944834e08acE755dcEA71491413f7e2CBAD`). If the values it uses (USD reserves, lvlUSD supply) can be influenced by atomic on-chain actions to skew the reported price, this oracle is a vector.

*   **`ResolvOracle.sol`:**
    *   **Source:** `IResolvFundamentalPriceOracle(fundamentalPriceOracle).lastPrice()`. Relies on Resolv Finance's oracle.
    *   **Manipulability:** Depends on the robustness of the external `ResolvFundamentalPriceOracle` contract (at `0x7f45180d6fFd0435D8dD695fd01320E6999c261c`). If this oracle's `lastPrice()` can be influenced by atomic on-chain actions (e.g., if it internally uses a manipulatable spot price), it's a vector.

**Conclusion on Oracles:** None of the custom oracles (`EthenaOracle`, `LevelOracle`, `ResolvOracle`) show internal TWAP mechanisms or other anti-manipulation features; they trust external systems. `FixedPriceOracle` is safe from direct manipulation. The core issue arises if *any* asset (especially the `receiptToken` or a common `assetToken`) is assigned an oracle that *is* manipulatable, such as one directly reading from a DEX spot market.

## Step-by-Step Exploit Scenario (Minting)

This scenario assumes `assetToken` is USDC (priced reliably at $1.00) and `receiptToken` (iUSD) is priced by an oracle that reflects its spot price against USDC on a Uniswap V2-like pool, making it manipulatable.

*   **a. Initial State:**
    *   `assetToken` (USDC) oracle price: $1.00 (1e18 normalized).
    *   `receiptToken` (iUSD) oracle price (from iUSD/USDC DEX pool): Initially $1.00 (1e18 normalized).
    *   DEX Pool (iUSD/USDC): 1,000,000 iUSD / 1,000,000 USDC.
    *   Attacker's USDC: 10,000 USDC.
    *   Expected mint: 10,000 USDC for 10,000 iUSD.

*   **b. Manipulation Phase (Flash Loan):**
    1.  Attacker flash loans 2,000,000 iUSD.
    2.  Attacker swaps these 2,000,000 iUSD for USDC on the iUSD/USDC DEX pool.
        *   Pool becomes: 3,000,000 iUSD / ~333,333 USDC.
        *   Attacker receives ~666,667 USDC.
        *   **New spot price of iUSD via oracle: ~$0.11.**

*   **c. Exploitation Phase (Atomic Transaction with Manipulation):**
    1.  Attacker calls `InfiniFiGatewayV1.mint(attacker_address, 10000 USDC)`.
    2.  `MintController.assetToReceipt(10000e18)` is called:
        *   `assetTokenPrice` (USDC) = 1e18.
        *   `receiptTokenPrice` (iUSD) = 0.11e18 (manipulated).
        *   `convertRatio = 0.11e18.divWadUp(1e18) = 0.11e18`.
        *   `receiptAmountOut = 10000e18.divWadDown(0.11e18) = ~90,909 iUSD`.
    3.  Attacker receives ~90,909 iUSD for 10,000 USDC.

*   **d. Post-Exploitation & Flash Loan Repayment:**
    1.  Attacker uses the ~666,667 USDC (from manipulation swap) to buy back iUSD from the DEX pool, restoring its price and retrieving ~2,000,000 iUSD.
    2.  Attacker repays the 2,000,000 iUSD flash loan (plus fees).

*   **e. Profit Calculation:**
    *   Attacker spent 10,000 USDC.
    *   Attacker gained ~90,909 iUSD instead of 10,000 iUSD.
    *   Profit: ~80,909 iUSD (worth ~$80,909 if sold at the restored price of $1), minus transaction and flash loan fees.

## Step-by-Step Exploit Scenario (Redeeming)

This scenario makes similar assumptions. Attacker aims to inflate the perceived value of iUSD.

*   **a. Initial State:**
    *   `assetToken` (USDC) oracle price: $1.00 (1e18 normalized).
    *   `receiptToken` (iUSD) oracle price (from iUSD/USDC DEX pool): Initially $1.00 (1e18 normalized).
    *   DEX Pool (iUSD/USDC): 1,000,000 iUSD / 1,000,000 USDC.
    *   Attacker's iUSD: 10,000 iUSD.
    *   Expected redeem: 10,000 iUSD for 10,000 USDC.

*   **b. Manipulation Phase (Flash Loan):**
    1.  Attacker flash loans 1,000,000 USDC.
    2.  Attacker swaps these 1,000,000 USDC for iUSD on the iUSD/USDC DEX pool.
        *   Pool becomes: ~500,000 iUSD / 2,000,000 USDC.
        *   Attacker receives ~500,000 iUSD.
        *   **New spot price of iUSD via oracle: ~$4.00.**

*   **c. Exploitation Phase (Atomic Transaction with Manipulation):**
    1.  Attacker calls `InfiniFiGatewayV1.redeem(attacker_address, 10000 iUSD, min_assets_out)`.
    2.  `RedeemController` calculates `assetAmountOut`:
        *   `_assetTokenPrice` (USDC) = 1e18.
        *   `_receiptTokenPrice` (iUSD) = 4e18 (manipulated).
        *   `convertRatio = 4e18.divWadDown(1e18) = 4e18`.
        *   `assetAmountOut = 10000e18.mulWadDown(4e18) = 40,000 USDC`.
    3.  Attacker receives ~40,000 USDC for 10,000 iUSD (assuming `RedeemController` has this liquidity).

*   **d. Post-Exploitation & Flash Loan Repayment:**
    1.  Attacker uses the ~500,000 iUSD (from manipulation swap) to buy back USDC from the DEX pool, restoring its price and retrieving ~1,000,000 USDC.
    2.  Attacker repays the 1,000,000 USDC flash loan (plus fees).

*   **e. Profit Calculation:**
    *   Attacker spent 10,000 iUSD.
    *   Attacker gained ~40,000 USDC instead of 10,000 USDC.
    *   Profit: ~30,000 USDC, minus transaction and flash loan fees.

## Existing Code-Level Mitigation Analysis

*   **`Accounting.sol`:** Contains no specific mitigations. It directly calls `oracle[_asset].price()`.
*   **Oracle Contracts:**
    *   `FixedPriceOracle.sol`: Is inherently a mitigation against on-chain price manipulation for assets it's assigned to, relying on secure admin control.
    *   `EthenaOracle.sol`, `LevelOracle.sol`, `ResolvOracle.sol`: These contracts do not implement their own TWAP or anti-manipulation logic. They pass through the price from the external contracts they query. Any mitigation would have to be present in those external systems.
*   **`MintController.sol` & `RedeemController.sol`:** These contracts do not implement any price validation logic (e.g., comparing against a recent price, TWAP, maximum allowable deviation). They use the prices from `Accounting.sol` as is. The `RedemptionPool` in `RedeemController` is for liquidity management, not oracle security.

**Conclusion on Mitigations:** The protocol currently lacks in-built, code-level defenses within `Accounting.sol` or the funding controllers against manipulated prices coming from a compromised or inherently vulnerable (e.g., spot-price based) oracle. The security heavily relies on the `ORACLE_MANAGER` assigning only robust, non-manipulatable oracles to all assets, especially the `receiptToken`.

## Impact
**Critical.** If a manipulatable oracle is configured for either the `assetToken` or `receiptToken`, an attacker can drain significant value from the protocol by minting `ReceiptTokens` at an artificially low cost or redeeming them for an artificially high amount of `assetToken`. This directly leads to a loss of funds for other users and the protocol.

## Recommendations

1.  **Mandate Robust Oracles:**
    *   For all actively traded tokens, especially the protocol's own `receiptToken` if it's traded externally, **mandate the use of robust, manipulation-resistant oracles.** Industry best practices point towards:
        *   **Chainlink Price Feeds:** These are widely used, decentralized, and aggregate prices from multiple sources, with built-in checks and balances.
        *   **Time-Weighted Average Price (TWAP) Oracles:** If using on-chain DEX prices, always use TWAP oracles (e.g., Uniswap V3's built-in TWAP capabilities, or custom-built TWAPs for Uniswap V2) over a sufficiently long period to make manipulation prohibitively expensive. Ensure the TWAP window cannot be easily manipulated within a few blocks.
2.  **Secure Oracle Administration:**
    *   The `ORACLE_MANAGER` role, responsible for `Accounting.setOracle()`, is critical. Access to this role must be strictly controlled, ideally via a multi-signature wallet with a Timelock for changes.
3.  **Implement Price Deviation Checks (Defense in Depth):**
    *   Consider adding a layer of defense within `Accounting.sol` or directly in the `MintController`/`RedeemController`:
        *   **Staleness Checks:** Ensure prices are not too old.
        *   **Price Deviation Checks:** When `Accounting.price()` is called, or before a mint/redeem operation, compare the returned price against a recently cached, known-good price. If the deviation is too large (e.g., >5-10% within a short timeframe), the transaction could be reverted or flagged. This requires careful design to avoid being overly restrictive during legitimate volatility.
4.  **For Protocol's Own ReceiptToken:**
    *   If `receiptToken` (e.g., iUSD) is intended to be pegged (e.g., to $1), a `FixedPriceOracle` (set to 1e18) is a simple and secure option, provided the peg is maintained by other protocol mechanisms (arbitrage, reserve backing).
    *   If `receiptToken`'s value can float or is determined by market forces, it *must* use a manipulation-resistant oracle if its price is used in calculations that transfer value.
5.  **External Oracle Audits:** Before integrating oracles like `EthenaOracle`, `LevelOracle`, `ResolvOracle`, ensure that the *external contracts they point to* are themselves audited and proven resistant to atomic manipulation of their reported prices. The current wrapper oracles do nothing to enhance their security.
6.  **Emergency Procedures:** Have a plan for rapidly pausing minting/redeeming or updating faulty oracles if a manipulation event is detected or an underlying oracle system is compromised. The existing `pause()` function in `CoreControlled` (and thus `InfiniFiGatewayV1`) can be used for this.

By implementing these recommendations, the protocol can significantly reduce the risk of oracle price manipulation attacks.
