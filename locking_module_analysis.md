# InfiniFi Protocol: Locking Module Analysis

This document provides an analysis of the contracts involved in the locking module of the InfiniFi protocol, specifically focusing on the processes of locking tokens, managing locked positions, and unwinding them.

## Contract Descriptions

### 1. `LockedPositionToken.sol`

*   **Purpose:** This contract defines an ERC20 token that represents a user's locked position for a *specific* unwinding duration (bucket). For each defined unwinding period (e.g., 2 weeks, 4 weeks), a distinct `LockedPositionToken` contract would be deployed. These tokens are also known as "share tokens" within the `LockingController`.
*   **Functionality:**
    *   Inherits from `CoreControlled`, `ERC20Permit`, and `ERC20Burnable`.
    *   **`CoreControlled`:** Its administrative functions (minting, burning) are restricted by `InfiniFiCore` roles.
    *   **`ERC20Permit` & `ERC20Burnable`:** Standard ERC20 extensions.
    *   **`constructor(address _core, string memory _name, string memory _symbol)`:** Initializes the token.
    *   **`mint(address _to, uint256 _amount)`:** Mints new locked position tokens. Only callable by an address with the `CoreRoles.LOCKED_TOKEN_MANAGER` role (which is the `LockingController`).
    *   **`burn(uint256 _value)` & `burnFrom(address _account, uint256 _value)`:** Burns tokens. Also restricted to `CoreRoles.LOCKED_TOKEN_MANAGER`.
    *   **Transfer Restrictions:**
        *   `transferRestrictions`: A mapping to store timestamps until which a user's tokens are restricted from being transferred.
        *   `restrictTransferUntilNextEpoch(address _user)`: Allows an address with `CoreRoles.TRANSFER_RESTRICTOR` to prevent a user from transferring their locked position tokens until the start of the next epoch.
        *   `_update(address _from, address _to, uint256 _value)`: Overrides the internal ERC20 transfer function to enforce these transfer restrictions.

### 2. `UnwindingModule.sol`

*   **Purpose:** This contract manages the lifecycle of locked positions that are currently in the "unwinding" phase. When a user decides to unlock their position, the corresponding receipt tokens are moved from the `LockingController` to this module. It handles the gradual release of these tokens over the specified unwinding period and manages the distribution of rewards or application of losses to these unwinding positions.
*   **Functionality:**
    *   Inherits from `CoreControlled`.
    *   **State Variables:**
        *   `receiptToken`: The address of the underlying token being locked (e.g., iUSD).
        *   `totalShares`, `totalReceiptTokens`: Tracks the total amount of tokens (in shares and receipt token value) within the module.
        *   `slashIndex`: A factor (starts at 1e18) that is reduced when losses are applied, affecting the value of all unwinding positions.
        *   `positions`: Mapping from a unique ID (user address + start timestamp) to an `UnwindingPosition` struct.
            *   `UnwindingPosition`: Contains shares, start/end epochs for unwinding, and reward weight parameters.
        *   `globalPoints`, `lastGlobalPointEpoch`: Manages a series of "global points" in time (epochs) to track changes in total reward weight and distributed rewards, allowing for fair reward calculation for positions that start/end unwinding at different times.
        *   `rewardWeightBiasIncreases`, `rewardWeightIncreases`, `rewardWeightDecreases`: Mappings to track changes to reward weights across epochs.
    *   **Core Logic - Unwinding Positions:**
        *   `startUnwinding(address _user, uint256 _receiptTokens, uint32 _unwindingEpochs, uint256 _rewardWeight)`:
            *   Called by `LockingController` (which has `CoreRoles.LOCKED_TOKEN_MANAGER`).
            *   Creates a new `UnwindingPosition` for the user.
            *   Calculates reward weight decrease per epoch.
            *   Updates total shares/receipt tokens and global reward weight tracking variables.
        *   `cancelUnwinding(address _user, uint256 _startUnwindingTimestamp, uint32 _newUnwindingEpochs)`:
            *   Called by `LockingController` (acting as `LOCKED_TOKEN_MANAGER` for this call).
            *   Allows a user to stop an ongoing unwinding process.
            *   Calculates the current value of the unwinding position (including earned rewards).
            *   Removes the position from this module.
            *   Approves `LockingController` to pull back the receipt tokens, and `LockingController` then calls its `createPosition` function to re-lock the tokens for the `_newUnwindingEpochs`.
        *   `withdraw(uint256 _startUnwindingTimestamp, address _owner)`:
            *   Called by `LockingController`.
            *   Allows a user to withdraw their receipt tokens after the unwinding period (`position.toEpoch`) has passed.
            *   Calculates the final share amount (including rewards).
            *   Deletes the position and updates global state.
            *   Transfers the final amount of `receiptToken` to the owner.
    *   **Rewards and Losses:**
        *   `depositRewards(uint256 _amount)`: Called by `LockingController` to distribute rewards to unwinding positions. Rewards are converted to shares and added to `totalShares` and `point.rewardShares`.
        *   `applyLosses(uint256 _amount)`: Called by `LockingController`. Reduces `totalReceiptTokens` and updates `slashIndex`. This proportionally reduces the value of all unwinding positions.
    *   **Read Functions:**
        *   `balanceOf(address _user, uint256 _startUnwindingTimestamp)`: Calculates the current claimable amount of `receiptToken` for a user's specific unwinding position, including accrued rewards and applied losses.
        *   `rewardWeight(address _user, uint256 _startUnwindingTimestamp)`: Calculates the current reward weight of an unwinding position.
        *   `totalRewardWeight()`: Returns the aggregate reward weight of all positions in the module.

### 3. `LockingController.sol`

*   **Purpose:** This is the main orchestrator for the locking mechanism. It allows users to lock their `receiptToken`s for predefined periods ("buckets" or "unwinding epochs"), manages these locked positions through distinct `LockedPositionToken`s for each bucket, and handles the distribution of rewards or application of losses to these locked (not yet unwinding) positions. It interacts with the `UnwindingModule` when users decide to start or cancel unwinding.
*   **Functionality:**
    *   Inherits from `CoreControlled`.
    *   **State Variables:**
        *   `receiptToken`: The address of the token being locked (e.g., iUSD).
        *   `unwindingModule`: Address of the `UnwindingModule` contract.
        *   `enabledBuckets`: An array of `uint32` representing the allowed unwinding durations in epochs (e.g., [2, 4, 6] weeks).
        *   `buckets`: Mapping from `unwindingEpochs` (duration) to `BucketData`.
            *   `BucketData`: Struct containing the address of the specific `LockedPositionToken` for that bucket, the total `receiptToken`s locked in that bucket, and a `multiplier` for reward weight calculation.
        *   `globalReceiptToken`: Total `receiptToken`s locked across all buckets (not in unwinding).
        *   `globalRewardWeight`: Aggregate reward weight for all locked (not unwinding) positions.
        *   `maxLossPercentage`: A cap on how much loss can be applied in a single `applyLosses` call before the contract pauses.
    *   **Administration:**
        *   `enableBucket(uint32 _unwindingEpochs, address _shareToken, uint256 _multiplier)`: `GOVERNOR` can enable new lock durations, providing the address of a deployed `LockedPositionToken` for it and a reward multiplier.
        *   `setBucketMultiplier(...)`: `PROTOCOL_PARAMETERS` can update a bucket's multiplier.
        *   `setMaxLossPercentage(...)`: `GOVERNOR` can set the loss cap.
    *   **Position Management:**
        *   `createPosition(uint256 _amount, uint32 _unwindingEpochs, address _recipient)`:
            *   Typically called by `InfiniFiGatewayV1` (via `CoreRoles.ENTRY_POINT`). Can also be re-entered by `UnwindingModule` during `cancelUnwinding`.
            *   User transfers `_amount` of `receiptToken` to this contract.
            *   Calculates the number of `LockedPositionToken` shares to mint based on the current exchange rate for that bucket.
            *   Mints these shares (specific `LockedPositionToken` for `_unwindingEpochs`) to the `_recipient`.
            *   Updates `bucket.totalReceiptTokens`, `globalReceiptToken`, and `globalRewardWeight`.
        *   `startUnwinding(uint256 _shares, uint32 _unwindingEpochs, address _recipient)`:
            *   Called by `InfiniFiGatewayV1`.
            *   User transfers their `LockedPositionToken` shares (for `_unwindingEpochs`) to this contract.
            *   These shares are burned.
            *   Calculates the corresponding `receiptToken` amount.
            *   Transfers these `receiptToken`s to the `UnwindingModule`.
            *   Calls `UnwindingModule.startUnwinding(...)` to initiate the unwinding process there.
            *   Updates its own global and bucket-specific token/reward weight counts.
        *   `increaseUnwindingEpochs(...)`: Allows a user to move their locked position from a shorter duration bucket to a longer one. Burns shares from the old bucket's `LockedPositionToken` and mints shares in the new one.
        *   `cancelUnwinding(...)`: Relays the call to `UnwindingModule.cancelUnwinding(...)`.
        *   `withdraw(...)`: Relays the call to `UnwindingModule.withdraw(...)`.
    *   **Rewards and Losses:**
        *   `depositRewards(uint256 _amount)`: `FINANCE_MANAGER` deposits rewards.
            *   Splits rewards between locked positions (managed here) and unwinding positions (transfers a portion to `UnwindingModule.depositRewards`).
            *   For locked positions, increases `bucket.totalReceiptTokens` for each bucket proportionally to its reward weight, effectively increasing the value of each `LockedPositionToken` share in that bucket. Updates global/bucket reward weights.
        *   `applyLosses(uint256 _amount)`: `FINANCE_MANAGER` applies losses.
            *   If losses exceed `maxLossPercentage` of `totalBalance()`, it fully slashes all tokens and pauses the contract.
            *   Splits losses between locked and unwinding positions (calls `UnwindingModule.applyLosses`).
            *   For locked positions, burns `receiptToken` from its own balance and decreases `bucket.totalReceiptTokens` proportionally, reducing share values. Updates global/bucket reward weights.
    *   **Read Functions:** `balanceOf(address _user)`, `rewardWeight(address _user)`, `exchangeRate(uint32 _unwindingEpochs)`, `totalBalance()`, `rewardMultiplier()`.

## Contract Interactions & Locking/Unwinding Flow

### 1. Creating a Locked Position:

1.  **User (via Gateway):** User wants to lock `X` amount of `ReceiptToken` for `N` unwinding epochs. They call a function on `InfiniFiGatewayV1`.
2.  **Gateway to LockingController:** `InfiniFiGatewayV1` calls `LockingController.createPosition(X, N, userAddress)`.
3.  **ReceiptToken Transfer:** `LockingController` pulls `X` `ReceiptToken`s from the Gateway (which would have received them from the user).
4.  **Share Calculation:** `LockingController` determines the target bucket based on `N`. It calculates how many shares of that bucket's specific `LockedPositionToken` to mint for `X` `ReceiptToken`s (based on `buckets[N].totalReceiptTokens` and `LockedPositionToken[N].totalSupply()`).
5.  **LockedPositionToken Minting:** `LockingController` calls `mint(userAddress, shareAmount)` on the specific `LockedPositionToken` for bucket `N`.
6.  **State Update:** `LockingController` updates `buckets[N].totalReceiptTokens`, `globalReceiptToken`, and `globalRewardWeight`.

### 2. Starting the Unwinding Process:

1.  **User (via Gateway):** User wants to start unwinding their position in bucket `N`, for which they hold `S` shares of `LockedPositionToken[N]`. They call a function on `InfiniFiGatewayV1`.
2.  **Gateway to LockingController:** `InfiniFiGatewayV1` calls `LockingController.startUnwinding(S, N, userAddress)`.
3.  **LockedPositionToken Transfer & Burn:** `LockingController` pulls `S` shares of `LockedPositionToken[N]` from the Gateway. It then calls `burn(S)` on `LockedPositionToken[N]`.
4.  **ReceiptToken Calculation:** `LockingController` calculates the amount of `ReceiptToken` corresponding to `S` shares.
5.  **Transfer to UnwindingModule:** `LockingController` transfers this calculated `ReceiptToken` amount to the `UnwindingModule`.
6.  **Initiate Unwinding in Module:** `LockingController` calls `UnwindingModule.startUnwinding(userAddress, receiptTokenAmount, N, rewardWeight)`.
7.  **State Update (LockingController):** `LockingController` reduces its `buckets[N].totalReceiptTokens`, `globalReceiptToken`, and `globalRewardWeight`.
8.  **State Update (UnwindingModule):** `UnwindingModule` creates an `UnwindingPosition` for the user, updates its internal `totalShares`, `totalReceiptTokens`, and global reward point data.

### 3. Cancelling Unwinding:

1.  **User (via Gateway):** User is unwinding (position identified by `startTimestamp`) and wants to cancel and re-lock for `M` new unwinding epochs. They call `InfiniFiGatewayV1`.
2.  **Gateway to LockingController:** `InfiniFiGatewayV1` calls `LockingController.cancelUnwinding(userAddress, startTimestamp, M)`.
3.  **LockingController to UnwindingModule:** `LockingController` calls `UnwindingModule.cancelUnwinding(userAddress, startTimestamp, M)`.
4.  **UnwindingModule Logic:**
    *   Calculates the current value (receipt tokens, including rewards/losses) of the user's unwinding position.
    *   Deletes the `UnwindingPosition`. Updates its internal totals and global reward points.
    *   Approves `LockingController` to spend the calculated `ReceiptToken` amount.
    *   Calls `LockingController.createPosition(calculatedReceiptAmount, M, userAddress)` (re-entrancy).
5.  **LockingController (createPosition re-entry):** The standard `createPosition` logic is followed (as in Flow 1), effectively re-locking the tokens.

### 4. Withdrawing After Unwinding:

1.  **User (via Gateway):** User's unwinding period (identified by `startTimestamp`) has completed. They call `InfiniFiGatewayV1`.
2.  **Gateway to LockingController:** `InfiniFiGatewayV1` calls `LockingController.withdraw(userAddress, startTimestamp)`.
3.  **LockingController to UnwindingModule:** `LockingController` calls `UnwindingModule.withdraw(startTimestamp, userAddress)`.
4.  **UnwindingModule Logic:**
    *   Verifies the unwinding period is complete.
    *   Calculates the final `ReceiptToken` amount for the user (including all rewards/losses during unwinding).
    *   Deletes the `UnwindingPosition`. Updates its internal totals and global reward points.
    *   Transfers the final `ReceiptToken` amount directly to `userAddress`.

### 5. Rewards & Losses Distribution:

*   **Rewards:**
    1.  `FinanceManager` calls `LockingController.depositRewards(totalRewardAmount)`.
    2.  `LockingController` splits `totalRewardAmount`:
        *   A portion is sent to `UnwindingModule.depositRewards(unwindingRewardAmount)`.
            *   `UnwindingModule` converts `unwindingRewardAmount` to shares and adds to its `totalShares` and current `globalPoint.rewardShares`. This benefits all current unwinding positions.
        *   The remaining portion is kept by `LockingController`.
            *   `LockingController` distributes this among its active buckets by increasing each `bucket.totalReceiptTokens` proportionally to its reward weight. This increases the `ReceiptToken` value per share of each `LockedPositionToken`.
*   **Losses:**
    1.  `FinanceManager` calls `LockingController.applyLosses(totalLossAmount)`.
    2.  `LockingController` checks against `maxLossPercentage`. If exceeded, it slashes everything and pauses.
    3.  `LockingController` splits `totalLossAmount`:
        *   A portion is attributed to `UnwindingModule` by calling `UnwindingModule.applyLosses(unwindingLossAmount)`.
            *   `UnwindingModule` burns `unwindingLossAmount` of its `ReceiptToken`s and updates its `slashIndex`. This devalues all unwinding positions.
        *   The remaining portion is applied to `LockingController`.
            *   `LockingController` burns `ReceiptToken`s from its balance and reduces each `bucket.totalReceiptTokens` proportionally. This decreases the `ReceiptToken` value per share of each `LockedPositionToken`.

## Mermaid Diagram

```mermaid
graph TD
    subgraph "User Interaction"
        User["User"]
    end

    subgraph "Core Protocol Layer"
        Gateway["InfiniFiGatewayV1"]
    end

    subgraph "Locking Module"
        LockingController["LockingController"]
        UnwindingModule["UnwindingModule"]
        LockedPositionToken_N["LockedPositionToken (for N epochs)"]
        LockedPositionToken_M["LockedPositionToken (for M epochs)"]
    end

    subgraph "Tokens"
        ReceiptToken["ReceiptToken (e.g., iUSD)"]
    end

    subgraph "Admin/System Roles"
        FinanceManager["Finance Manager"]
        InfiniFiCore["InfiniFiCore"]
    end

    %% Inheritance
    LockingController --|> CoreControlled_LC[CoreControlled]
    UnwindingModule --|> CoreControlled_UM[CoreControlled]
    LockedPositionToken_N --|> CoreControlled_LPTN[CoreControlled]
    LockedPositionToken_N --|> ERC20_LPTN[ERC20]
    LockedPositionToken_M --|> CoreControlled_LPTM[CoreControlled]
    LockedPositionToken_M --|> ERC20_LPTM[ERC20]

    %% Role Control
    LockingController -- "Calls for Roles (ENTRY_POINT, GOVERNOR, etc.)" --> InfiniFiCore
    UnwindingModule -- "Calls for Roles (LOCKED_TOKEN_MANAGER)" --> InfiniFiCore
    LockedPositionToken_N -- "Calls for Roles (LOCKED_TOKEN_MANAGER, TRANSFER_RESTRICTOR)" --> InfiniFiCore

    %% Flow: Create Lock
    User -- "1. Lock X ReceiptToken for N epochs" --> Gateway
    Gateway -- "2. createPosition(X, N, user)" --> LockingController
    LockingController -- "3. Transfers X ReceiptToken from Gateway" --> ReceiptToken
    LockingController -- "4. Mints shares" --> LockedPositionToken_N
    LockedPositionToken_N -- "5. Shares to User" --> User

    %% Flow: Start Unwinding
    User -- "1. Start Unwinding N-epoch lock" --> Gateway
    Gateway -- "2. startUnwinding(shares, N, user)" --> LockingController
    LockingController -- "3. Transfers shares from Gateway & Burns them" --> LockedPositionToken_N
    LockingController -- "4. Calculates ReceiptToken amount" --> LockingController
    LockingController -- "5. Transfers ReceiptToken to UnwindingModule" --> ReceiptToken
    ReceiptToken -- "Funds" --> UnwindingModule
    LockingController -- "6. startUnwinding(user, receiptAmt, N, rewardWt)" --> UnwindingModule
    UnwindingModule -- "7. Creates UnwindingPosition" --> UnwindingModule

    %% Flow: Cancel Unwinding & Re-lock
    User -- "1. Cancel Unwinding (id: startTS), Re-lock M epochs" --> Gateway
    Gateway -- "2. cancelUnwinding(user, startTS, M)" --> LockingController
    LockingController -- "3. cancelUnwinding(user, startTS, M)" --> UnwindingModule
    UnwindingModule -- "4. Calculates value, Deletes Position" --> UnwindingModule
    UnwindingModule -- "5. Approves LC for ReceiptTokens" --> ReceiptToken
    UnwindingModule -- "6. Calls LC.createPosition(newAmt, M, user)" --> LockingController
    %% LockingController then re-uses the Create Lock flow, minting LockedPositionToken_M

    %% Flow: Withdraw after Unwinding
    User -- "1. Withdraw (id: startTS)" --> Gateway
    Gateway -- "2. withdraw(user, startTS)" --> LockingController
    LockingController -- "3. withdraw(startTS, user)" --> UnwindingModule
    UnwindingModule -- "4. Calculates final amt, Deletes Position" --> UnwindingModule
    UnwindingModule -- "5. Transfers ReceiptToken to User" --> ReceiptToken
    ReceiptToken -- "Funds User" --> User

    %% Flow: Rewards
    FinanceManager -- "1. depositRewards(totalAmt)" --> LockingController
    LockingController -- "2a. Transfers portion to UnwindingModule" --> ReceiptToken
    ReceiptToken -- "Funds for Unwinding Rewards" --> UnwindingModule
    UnwindingModule -- "2a. depositRewards(unwindingAmt)" --> UnwindingModule
    LockingController -- "2b. Distributes internally to Buckets" --> LockingController

    %% Flow: Losses
    FinanceManager -- "1. applyLosses(totalAmt)" --> LockingController
    LockingController -- "2a. Calls UnwindingModule.applyLosses" --> UnwindingModule
    UnwindingModule -- "2a. Burns ReceiptToken & updates slashIndex" --> ReceiptToken
    LockingController -- "2b. Burns ReceiptToken & updates Buckets" --> ReceiptToken

    %% Other Interactions
    LockingController -- "Manages (enable, multiplier)" --> LockedPositionToken_N
    LockingController -- "Manages (enable, multiplier)" --> LockedPositionToken_M
    LockingController -- "Reads total{ReceiptTokens,RewardWeight}, slashIndex" --> UnwindingModule
```
