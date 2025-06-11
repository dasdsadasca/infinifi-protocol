# InfiniFi Protocol: Finance and Integrations Analysis

This document provides an analysis of contracts related to finance (yield sharing, accounting), farm integrations (registry, specific farm implementations), and their interactions within the InfiniFi protocol.

## Contract Descriptions

### 1. `Farm.sol` (Base Contract)

*   **Purpose:** This is an abstract base contract that defines the common interface and core functionalities for all specific "farm" contracts within the InfiniFi protocol. Farms are integrations with external DeFi protocols or strategies designed to generate yield on deposited assets.
*   **Functionality:**
    *   Inherits from `CoreControlled`, making its administrative functions role-based.
    *   Implements `IFarm` interface.
    *   **State Variables:**
        *   `assetToken`: The address of the underlying asset the farm manages (e.g., USDC).
        *   `cap`: Maximum amount of `assetToken` that can be deposited into the farm. Configurable by `PROTOCOL_PARAMETERS`.
        *   `maxSlippage`: Maximum tolerated slippage for deposits/withdrawals, expressed as a percentage (1e18 = 100%). Configurable by `PROTOCOL_PARAMETERS`.
    *   **Core Functions (to be implemented/customized by child farms):**
        *   `assets()`: Abstract view function to return the total value of assets managed by the farm, expressed in `assetToken` units.
        *   `liquidity()`: (As seen in `AaveV3Farm` and `PendleV2Farm`) A view function to report the amount of `assetToken` that can be immediately withdrawn.
        *   `_deposit(uint256 assetsToDeposit)`: Abstract internal function to handle the logic of depositing `assetsToDeposit` into the external protocol.
        *   `_withdraw(uint256 amount, address to)`: Abstract internal function to handle the logic of withdrawing `amount` of `assetToken` from the external protocol to the `to` address.
    *   **Common Functions (implemented in `Farm.sol`):**
        *   `setCap(uint256 _newCap)`: Allows `PROTOCOL_PARAMETERS` to update the deposit cap.
        *   `setMaxSlippage(uint256 _maxSlippage)`: Allows `PROTOCOL_PARAMETERS` to update slippage tolerance.
        *   `maxDeposit()`: Calculates the remaining capacity for deposits.
        *   `deposit()`: Public function callable by `FARM_MANAGER`. It checks the cap, calls the internal `_deposit`, and then verifies slippage based on the change in `assets()`.
        *   `withdraw(uint256 amount, address to)`: Public function callable by `FARM_MANAGER`. It calls internal `_withdraw` and then verifies slippage.

### 2. `AaveV3Farm.sol`

*   **Purpose:** A specific farm implementation that integrates with the Aave V3 lending protocol. It allows the InfiniFi protocol to deposit `assetToken` into Aave to earn lending yield and receive aTokens.
*   **Functionality:**
    *   Inherits from `Farm`.
    *   **State Variables:**
        *   `aToken`: The address of the Aave aToken corresponding to the `assetToken` (e.g., aUSDC).
        *   `lendingPool`: The address of the Aave V3 LendingPool contract.
    *   **`assets()`:** Returns the balance of `aToken` held by this farm contract. Since aTokens are interest-bearing and increase in value relative to the underlying, their balance represents the total principal + yield.
    *   **`liquidity()`:** Determines how much `assetToken` can be withdrawn from Aave. It checks the underlying `assetToken` balance of the `aToken` contract itself and considers if Aave is paused.
    *   **`_deposit(uint256 availableBalance)`:** Approves the Aave `lendingPool` to spend `assetToken` and then calls `supply()` on the Aave pool to deposit the assets.
    *   **`_withdraw(uint256 _amount, address _to)`:** Calls `withdraw()` on the Aave `lendingPool` to redeem aTokens for the underlying `assetToken`.

### 3. `PendleV2Farm.sol`

*   **Purpose:** A specific farm implementation for integrating with Pendle V2, a protocol for tokenizing and trading future yield. This farm likely wraps the `assetToken` into Pendle's Principal Tokens (PT) to earn fixed yield or participate in other Pendle strategies. This is a more complex farm due to maturities and the nature of Pendle's SY (Standardized Yield) and PT tokens.
*   **Functionality:**
    *   Inherits from `Farm` and implements `IMaturityFarm`.
    *   **State Variables:**
        *   `maturity`: Timestamp of when the Pendle market's PT matures.
        *   `pendleMarket`: Address of the specific Pendle market contract.
        *   `pendleOracle`: Address of Pendle's oracle for PT to underlying exchange rates.
        *   `underlyingToken`: The token PTs convert to at maturity (might be different from `assetToken` if there's an intermediate layer, e.g., `assetToken` is USDC, SY is syUSDC, `underlyingToken` is USDC).
        *   `ptToken`: Address of the Principal Token for the market.
        *   `syToken`: Address of the Standardized Yield token for the market.
        *   `accounting`: Reference to the InfiniFi `Accounting` contract (for pricing `underlyingToken` vs `assetToken`).
        *   `pendleRouter`: Address for executing swaps (wrapping/unwrapping PTs).
        *   Internal state to track wrapped/unwrapped assets and PTs for accurate yield interpolation (`totalWrappedAssets`, `_alreadyInterpolatedYield`, `_lastWrappedTimestamp`).
    *   **`assets()`:**
        *   Before `maturity`: Returns `assetToken` balance + `totalWrappedAssets` (principal invested in PTs) + `_interpolatingYield()` (estimated yield accrued on PTs but not yet realized).
        *   After `maturity`: Returns `assetToken` balance + value of any remaining `ptToken`s (priced via `_ptToAssets` which uses `pendleOracle` and `accounting`).
    *   **`liquidity()`:** Returns the balance of `assetToken` held directly by the farm (assets not yet wrapped into PTs or already unwrapped).
    *   **`wrapAssetToPt(uint256 _assetsIn, bytes memory _calldata)`:** `FARM_SWAP_CALLER` role. Approves `pendleRouter` and executes the swap calldata to convert `assetToken` to `ptToken`. Updates internal tracking for yield interpolation.
    *   **`unwrapPtToAsset(uint256 _ptTokensIn, bytes memory _calldata)`:** `FARM_SWAP_CALLER` role. Similar to wrap, but converts `ptToken` back to `assetToken` after maturity.
    *   **`_deposit(uint256)`:** No-op. Actual "deposit" into Pendle happens via `wrapAssetToPt`. The `deposit()` function in `Farm.sol` is overridden to only allow deposits before maturity.
    *   **`_withdraw(uint256 _amount, address _to)`:** Transfers `assetToken` from its own balance. Actual "withdrawal" from Pendle happens via `unwrapPtToAsset`.
    *   **`_interpolatingYield()`:** Calculates the estimated yield accrued on the held PTs since the last wrap, projecting towards the expected value at maturity. This is crucial for providing an up-to-date `assets()` value before maturity.
    *   **`maturity()` (from `IMaturityFarm`):** Returns the `maturity` timestamp.

### 4. `FarmRegistry.sol`

*   **Purpose:** This contract acts as a directory or registry for all farm contracts used by the InfiniFi protocol. It allows the system to discover, categorize, and manage these farms.
*   **Functionality:**
    *   Inherits from `CoreControlled`.
    *   Uses OpenZeppelin's `EnumerableSet.AddressSet` to store lists of addresses for assets and farms, allowing for iteration.
    *   **State Variables (Mappings of Sets):**
        *   `assets`: Set of all enabled underlying asset tokens (e.g., USDC, WETH).
        *   `farms`: Set of all registered farm contract addresses.
        *   `typeFarms[farmType]`: Maps a farm type (e.g., `FarmTypes.LIQUID`, `FarmTypes.MATURITY`) to a set of farm addresses of that type.
        *   `assetFarms[assetAddress]`: Maps an asset address to a set of farm addresses that manage that asset.
        *   `assetTypeFarms[assetAddress][farmType]`: Maps an asset and farm type to a set of specific farm addresses.
    *   **Write Methods (Admin Restricted):**
        *   `enableAsset(address _asset)`: `GOVERNOR` can add a new asset to the system.
        *   `disableAsset(address _asset)`: `GOVERNOR` can remove an asset.
        *   `addFarms(uint256 _type, address[] calldata _list)`: `PROTOCOL_PARAMETERS` can add a list of new farm contracts. It checks if the farm's `assetToken` is enabled before adding.
        *   `removeFarms(uint256 _type, address[] calldata _list)`: `PROTOCOL_PARAMETERS` can remove farm contracts.
    *   **Read Methods (Public View):**
        *   Provides various getters to retrieve arrays of addresses for enabled assets, all farms, farms of a specific type, farms for a specific asset, or farms for a specific asset AND type.
        *   `isAssetEnabled(address _asset)`
        *   `isFarm(address _farm)`
        *   `isFarmOfAsset(address _farm, address _asset)`
        *   `isFarmOfType(address _farm, uint256 _type)`

### 5. `Accounting.sol`

*   **Purpose:** This contract is responsible for providing price information for various assets and for calculating the total value of assets held across all registered farms. It serves as the protocol's primary source of truth for asset valuation.
*   **Functionality:**
    *   Inherits from `CoreControlled`.
    *   **State Variables:**
        *   `farmRegistry`: Address of the `FarmRegistry` contract.
        *   `oracle[assetAddress]`: Maps an asset address to its designated price oracle contract (which must implement `IOracle`).
    *   **Write Methods:**
        *   `setOracle(address _asset, address _oracle)`: `ORACLE_MANAGER` can set or update the price oracle for a given asset.
    *   **Read Methods:**
        *   `price(address _asset)`: Returns the price of `_asset` by querying its registered `IOracle`. Prices are typically in USD with 18 decimals.
        *   `totalAssetsValue()`: Calculates the total USD value of all assets held across all farms. It iterates through enabled assets from `FarmRegistry`, gets all farms for each asset, calls `assets()` on each farm, and sums their values multiplied by their oracle prices.
        *   `totalAssetsValueOf(uint256 _type)`: Similar to `totalAssetsValue()`, but only for farms of a specific `_type`.
        *   `totalAssets(address _asset)`: Calculates the total quantity of a specific `_asset` held across all farms managing that asset.
        *   `totalAssetsOf(address _asset, uint256 _type)`: Similar to `totalAssets()`, but only for farms of a specific `_type` managing that `_asset`.
    *   **Internal Helper:**
        *   `_calculateTotalAssets(address[] memory _farms)`: Internal function that iterates through a list of farm addresses and sums the results of their `assets()` calls.

### 6. `YieldSharing.sol`

*   **Purpose:** This contract is central to the protocol's value accrual and distribution. It determines the overall profit or loss (`unaccruedYield`) of the protocol by comparing the total value of assets in farms (from `Accounting`) against the total supply of `ReceiptToken`s. It then distributes profits or applies losses according to a defined hierarchy and strategy.
*   **Functionality:**
    *   Inherits from `CoreControlled`.
    *   **State Variables:**
        *   `accounting`: Address of the `Accounting` contract.
        *   `receiptToken`: Address of the protocol's main `ReceiptToken` (e.g., iUSD).
        *   `stakedToken`: Address of the `StakedToken` (e.g., siUSD) where users can stake `ReceiptToken`s.
        *   `lockingModule`: Address of the `LockingController`.
        *   `safetyBufferSize`: An amount of `ReceiptToken` held by this contract to absorb small initial losses.
        *   `performanceFee`, `performanceFeeRecipient`: For charging a fee on profits.
        *   `liquidReturnMultiplier`: A multiplier applied to the share of yield going to `stakedToken` holders.
        *   `targetIlliquidRatio`: A target for the proportion of assets that should be in illiquid (locked) positions, potentially influencing yield distribution to incentivize locking.
        *   `stakedReceiptTokenCache`: Caches `receiptToken` balance of `stakedToken` to optimize.
    *   **Core Logic (`accrue` function):**
        *   Calculates `unaccruedYield()`:
            *   Gets `totalAssetsValue()` from `Accounting` (in USD).
            *   Gets `receiptTokenPrice` from `Accounting`.
            *   Converts `totalAssetsValue` to `ReceiptToken` units.
            *   `unaccruedYield = assetsInReceiptTokens - ReceiptToken.totalSupply()`.
        *   If `yield > 0` (`_handlePositiveYield`):
            *   Mints the `yield` amount of `ReceiptToken` to itself.
            *   Replenishes `safetyBufferSize` if it's below target.
            *   Takes `performanceFee` if applicable.
            *   Calculates distribution shares for `stakedToken` holders (potentially adjusted by `liquidReturnMultiplier`) and `lockingModule` users (potentially adjusted by `LockingController.rewardMultiplier()` and `targetIlliquidRatio` logic).
            *   Calls `StakedToken.depositRewards()` and `LockingController.depositRewards()` to distribute the respective profit shares.
        *   If `yield < 0` (`_handleNegativeYield`):
            *   First, attempts to cover losses from its own `safetyBuffer` by burning its `ReceiptToken`s.
            *   If losses exceed buffer, applies remaining losses to `LockingController` by calling `LockingController.applyLosses()`.
            *   If further losses remain, applies them to `StakedToken` by calling `StakedToken.applyLosses()`.
            *   If losses still persist (extreme scenario), it devalues the `ReceiptToken` itself by updating its price oracle (assuming a `FixedPriceOracle`) via `Accounting`.
    *   **Configuration:** Allows `PROTOCOL_PARAMETERS` or `GOVERNOR` to set `safetyBufferSize`, performance fee parameters, `liquidReturnMultiplier`, and `targetIlliquidRatio`.

## Contract Interactions

1.  **Asset Reporting and Valuation:**
    *   Each specific `Farm` (e.g., `AaveV3Farm`, `PendleV2Farm`) implements `assets()` to report its holdings in terms of its `assetToken`.
    *   `FarmRegistry` maintains lists of all active farms, categorized by asset and type.
    *   `Accounting.totalAssetsValue()` queries `FarmRegistry` to get all relevant farms. It then iterates, calling `assets()` on each farm and `price()` on the farm's `assetToken` (via its registered oracle) to get a total USD value.
    *   `Accounting.price(asset)` calls the specific `IOracle` registered for that `asset`.

2.  **Yield Accrual and Distribution:**
    *   `YieldSharing.accrue()` is the trigger.
    *   It calls `YieldSharing.unaccruedYield()`:
        *   This function calls `Accounting.totalAssetsValue()` to get the current total value of assets in all farms.
        *   It calls `Accounting.price(receiptTokenAddress)` to get the current price of the `ReceiptToken`.
        *   It compares the asset value (in `ReceiptToken` terms) to `ReceiptToken.totalSupply()`.
    *   If there's positive yield:
        *   `YieldSharing` mints new `ReceiptToken`s to itself.
        *   After handling safety buffer and performance fees, it calculates splits.
        *   It calls `StakedToken.depositRewards()` with the stakers' share.
        *   It calls `LockingController.depositRewards()` with the lockers' share. The `LockingController` then further distributes this among its various locking buckets and potentially to the `UnwindingModule`.
    *   If there's negative yield:
        *   `YieldSharing` first burns its own `ReceiptToken`s (from safety buffer).
        *   Then calls `LockingController.applyLosses()`. `LockingController` applies losses to its locked balances and may propagate to `UnwindingModule`.
        *   Then calls `StakedToken.applyLosses()`.
        *   Finally, if necessary, calls `Accounting.setOracle()` (indirectly, by getting the oracle address then calling `setPrice` on it if it's a `FixedPriceOracle`) to adjust the `ReceiptToken`'s own price.

3.  **Farm Management:**
    *   `FarmRegistry.addFarms()` allows `PROTOCOL_PARAMETERS` to register new farm contracts. The registry checks if the farm's `assetToken` (obtained by calling `IFarm(farmAddress).assetToken()`) is enabled.
    *   `FarmRegistry.removeFarms()` allows removal.
    *   A `FARM_MANAGER` (role) interacts with individual `Farm` contracts by calling `deposit()` (to push funds from the farm contract itself into the external protocol) or `withdraw()` (to pull funds from the external protocol back to the farm contract).
    *   For farms like `PendleV2Farm` that require swaps, a `FARM_SWAP_CALLER` calls functions like `wrapAssetToPt` or `unwrapPtToAsset` with specific calldata.

4.  **Gateway Interactions:**
    *   While not directly in these contracts, a `Gateway` contract would likely be the `msg.sender` (with `CoreRoles.ENTRY_POINT`) for user-initiated actions that might indirectly lead to farm interactions (e.g., depositing USDC that then gets routed to a farm by a manager).
    *   The `Gateway` might also read farm lists from `FarmRegistry` to display options to users.

## Mermaid Diagram

```mermaid
graph LR
    subgraph "External DeFi Protocols"
        AaveV3_External["Aave V3 Protocol"]
        PendleV2_External["Pendle V2 Protocol"]
    end

    subgraph "Farm Layer (Integrations)"
        FarmBase["Farm (Base Contract)"]
        AaveV3Farm["AaveV3Farm"] --|> FarmBase
        PendleV2Farm["PendleV2Farm"] --|> FarmBase
    end

    subgraph "Registry & Accounting"
        FarmRegistry["FarmRegistry"]
        Accounting["Accounting"]
        Oracle_AssetA["Oracle for Asset A"]
        Oracle_ReceiptT["Oracle for ReceiptToken"]
    end

    subgraph "Yield Processing & Distribution"
        YieldSharing["YieldSharing"]
    end

    subgraph "Staking & Locking Components"
        StakedToken["StakedToken"]
        LockingController["LockingController"]
    end

    subgraph "Token Contracts"
        ReceiptToken["ReceiptToken"]
        AssetA["Asset A (e.g., USDC)"]
    end

    subgraph "Admin Roles"
        FARM_MANAGER["FARM_MANAGER Role"]
        PROTOCOL_PARAMETERS_ROLE["PROTOCOL_PARAMETERS Role"]
        ORACLE_MANAGER_ROLE["ORACLE_MANAGER Role"]
        GOVERNOR_ROLE["GOVERNOR Role"]
        FARM_SWAP_CALLER_ROLE["FARM_SWAP_CALLER Role"]
    end

    %% Farm Interactions with External Protocols
    AaveV3Farm -- "supply/withdraw (Asset A)" --> AaveV3_External
    PendleV2Farm -- "swap via Router (Asset A <-> PT)" --> PendleV2_External

    %% Farm Manager Interactions
    FARM_MANAGER -- "deposit()" --> AaveV3Farm
    FARM_MANAGER -- "withdraw()" --> AaveV3Farm
    %% For Pendle, direct deposit/withdraw is less common; FARM_SWAP_CALLER handles wraps/unwraps
    FARM_SWAP_CALLER_ROLE -- "wrapAssetToPt()" --> PendleV2Farm
    FARM_SWAP_CALLER_ROLE -- "unwrapPtToAsset()" --> PendleV2Farm

    %% Farm Registry Operations
    PROTOCOL_PARAMETERS_ROLE -- "addFarms() / removeFarms()" --> FarmRegistry
    GOVERNOR_ROLE -- "enableAsset() / disableAsset()" --> FarmRegistry
    FarmRegistry -- "Farm.assetToken()" --> AaveV3Farm % to get asset during registration
    FarmRegistry -- "Farm.assetToken()" --> PendleV2Farm % to get asset during registration

    %% Accounting Operations
    Accounting -- "Gets farm lists" --> FarmRegistry
    AaveV3Farm -- "assets()" --> Accounting % Accounting calls farm.assets()
    PendleV2Farm -- "assets()" --> Accounting % Accounting calls farm.assets()
    Accounting -- "price()" --> Oracle_AssetA
    ORACLE_MANAGER_ROLE -- "setOracle()" --> Accounting

    %% YieldSharing Operations
    YieldSharing -- "totalAssetsValue()" --> Accounting
    YieldSharing -- "price(receiptToken)" --> Accounting
    Accounting -- "price()" --> Oracle_ReceiptT
    YieldSharing -- "totalSupply()" --> ReceiptToken
    YieldSharing -- "Mints/Burns" --> ReceiptToken
    YieldSharing -- "depositRewards()" --> StakedToken
    YieldSharing -- "applyLosses()" --> StakedToken
    YieldSharing -- "depositRewards()" --> LockingController
    YieldSharing -- "applyLosses()" --> LockingController
    YieldSharing -- "balanceOf(stakedToken)" --> ReceiptToken % For stakedReceiptTokenCache
    YieldSharing -- "approve(stakedToken)" --> ReceiptToken
    YieldSharing -- "approve(lockingModule)" --> ReceiptToken


    %% Pendle Farm Specifics
    PendleV2Farm -- "Needs asset prices for PT conversion" --> Accounting

    classDef base_contract fill:#e6e6fa,stroke:#333,stroke-width:2px;
    class FarmBase base_contract;
```
