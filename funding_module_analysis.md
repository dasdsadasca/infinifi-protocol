# InfiniFi Protocol: Funding Module Analysis

This document provides an analysis of the contracts involved in the funding module of the InfiniFi protocol, specifically focusing on minting and redemption processes.

## Contract Descriptions

### 1. `ReceiptToken.sol`

*   **Purpose:** This contract defines the `ReceiptToken` (e.g., iUSD), which represents a user's share or deposit in the InfiniFi protocol. It is an ERC20 token with additional functionalities for minting, burning, and permit (gas-less approvals).
*   **Functionality:**
    *   Inherits from `CoreControlled`, `ERC20Permit`, and `ERC20Burnable`.
    *   **`CoreControlled`:** Its administrative functions (like `mint` and `burn`) are restricted by roles defined in `InfiniFiCore`.
    *   **`ERC20Permit`:** Allows users to approve token spending via an off-chain signature, rather than an on-chain transaction.
    *   **`ERC20Burnable`:** Provides standard `burn` and `burnFrom` functions.
    *   **`constructor(address _core, string memory _name, string memory _symbol)`:** Initializes the token with a reference to `InfiniFiCore`, its name, and symbol.
    *   **`mint(address _to, uint256 _amount)`:** Mints new receipt tokens to a specified address. This function can only be called by an address holding the `CoreRoles.RECEIPT_TOKEN_MINTER` role (typically the `MintController`).
    *   **`burn(uint256 _value)`:** Burns receipt tokens from the caller's balance. This function can only be called by an address holding the `CoreRoles.RECEIPT_TOKEN_BURNER` role (typically the `RedeemController`).
    *   **`burnFrom(address _account, uint256 _value)`:** Burns receipt tokens from a specified account's balance, provided the caller has sufficient allowance. Also restricted to the `CoreRoles.RECEIPT_TOKEN_BURNER` role.

### 2. `MintController.sol`

*   **Purpose:** This contract manages the process of creating (minting) new `ReceiptToken`s in exchange for an underlying asset (e.g., USDC). It acts as an entry point for new funds into the protocol and is designed to report its assets like a `Farm` for standardized accounting.
*   **Functionality:**
    *   Inherits from `Farm` and implements `IMintController`. `Farm` itself inherits from `CoreControlled`.
    *   **State Variables:**
        *   `receiptToken`: Address of the `ReceiptToken` contract.
        *   `accounting`: Address of the `Accounting` contract (used for price feeds).
        *   `minAssetAmount`: Minimum amount of asset required for minting.
        *   `afterMintHook`: Optional address of a contract to call after a successful mint operation.
    *   **`constructor(address _core, address _assetToken, address _receiptToken, address _accounting)`:** Initializes with core, asset token (e.g., USDC), receipt token, and accounting contract addresses.
    *   **`assetToReceipt(uint256 _assetAmount)`:** Calculates the amount of `ReceiptToken`s a user will receive for a given amount of `assetToken`. It uses prices from the `Accounting` contract.
    *   **`mint(address _to, uint256 _assetAmountIn)`:**
        *   Restricted to callers with the `CoreRoles.ENTRY_POINT` role (typically `InfiniFiGatewayV1`).
        *   Requires `_assetAmountIn` to be above `minAssetAmount`.
        *   Calculates `receiptAmountOut` using `assetToReceipt`.
        *   Transfers `_assetAmountIn` of `assetToken` from the `msg.sender` (the Gateway) to itself.
        *   Calls `mint` on the `ReceiptToken` contract to create `receiptAmountOut` tokens for the user (`_to`).
        *   Optionally calls an `afterMint` function on the `afterMintHook` contract.
    *   **`assets()` & `liquidity()`:** Returns the balance of `assetToken` held by the controller.
    *   **Farm Overrides (`_deposit`, `deposit`, `_withdraw`, `withdraw`):** Implements `Farm`'s deposit/withdraw logic. Deposits are no-ops internally as funds come via `mint`. Withdrawals transfer `assetToken` and are restricted to `CoreRoles.FARM_MANAGER`.

### 3. `RedemptionPool.sol`

*   **Purpose:** This is an abstract contract that provides the core logic for managing a redemption queue. It handles the queuing of redemption requests when immediate fulfillment is not possible and the processing of these requests as funds become available. It does *not* handle token transfers itself; this is left to the inheriting contract.
*   **Functionality:**
    *   Uses `RedemptionQueue` library for queue management.
    *   **State Variables:**
        *   `queue`: The actual queue of `RedemptionRequest` structs (amount, recipient).
        *   `MAX_QUEUE_LENGTH`: Maximum number of requests in the queue.
        *   `userPendingClaims`: Mapping from recipient address to the amount of asset they can claim.
        *   `totalPendingClaims`: Total amount of asset (e.g., USDC) currently available for claiming.
        *   `totalEnqueuedRedemptions`: Total amount of `ReceiptToken` (e.g., iUSD) currently waiting in the queue.
    *   **`_fundRedemptionQueue(uint256 _assetAmount, uint256 _convertReceiptToAssetRatio)`:**
        *   Internal function called when new assets are available to process queued redemptions.
        *   Iterates through the `queue`, funding requests as much as possible with `_assetAmount`.
        *   If a request is fully funded, it's popped from the queue. If partially funded, its amount is updated.
        *   Updates `userPendingClaims` for recipients whose requests are funded.
        *   Updates `totalPendingClaims` and `totalEnqueuedRedemptions`.
        *   Returns the remaining `_assetAmount` (if any) and the total amount of `ReceiptToken` that corresponds to the funded asset amount (this is the amount to be burned by the caller).
    *   **`_claimRedemption(address _recipient)`:**
        *   Internal function to process a user's claim.
        *   Checks `userPendingClaims` for the recipient.
        *   Resets the user's pending claim to zero and decreases `totalPendingClaims`.
        *   Returns the amount claimed.
    *   **`_enqueue(address _recipient, uint256 _amount)`:**
        *   Internal function to add a new redemption request to the `queue`.
        *   Checks against `MAX_QUEUE_LENGTH`.
        *   Increases `totalEnqueuedRedemptions`.

### 4. `RedeemController.sol`

*   **Purpose:** This contract manages the process of redeeming `ReceiptToken`s for the underlying asset. It handles both immediate redemptions (if enough liquidity is available) and queued redemptions (if liquidity is insufficient, using `RedemptionPool` logic). It also acts as a `Farm` for accounting purposes.
*   **Functionality:**
    *   Inherits from `Farm`, `RedemptionPool`, and implements `IRedeemController`.
    *   **State Variables:**
        *   `receiptToken`: Address of the `ReceiptToken` contract.
        *   `accounting`: Address of the `Accounting` contract.
        *   `minRedemptionAmount`: Minimum amount of `ReceiptToken` for redemption.
        *   `beforeRedeemHook`: Optional address of a contract to call before a redeem operation.
    *   **`constructor(...)`:** Similar to `MintController`.
    *   **`receiptToAsset(uint256 _receiptAmount)`:** Calculates the amount of `assetToken` a user will receive for a given amount of `ReceiptToken`, using prices from `Accounting`.
    *   **`redeem(address _to, uint256 _receiptAmountIn)`:**
        *   Restricted to `CoreRoles.ENTRY_POINT` (typically `InfiniFiGatewayV1`).
        *   Calculates `assetAmountOut` using `_getReceiptToAssetConvertRatio`.
        *   **Queue Logic:** If `queueLength() > 0` (from `RedemptionPool`), it transfers `_receiptAmountIn` from `msg.sender` (Gateway) to itself and calls `_enqueue` to add the request to the queue. Returns 0 assets immediately.
        *   **Immediate/Partial Redemption:**
            *   If `assetAmountOut` is less than or equal to `liquidity()` (assets held by the controller minus `totalPendingClaims`):
                *   Burns `_receiptAmountIn` of `ReceiptToken` from `msg.sender` (Gateway).
                *   Transfers `assetAmountOut` of `assetToken` to the user (`_to`).
                *   Returns `assetAmountOut`.
            *   If `assetAmountOut` is greater than `liquidity()`:
                *   Burns an amount of `ReceiptToken` from `msg.sender` (Gateway) corresponding to the `availableAssetAmount`.
                *   Transfers `availableAssetAmount` of `assetToken` to the user (`_to`).
                *   Transfers the remaining `ReceiptToken` (for the unfunded portion) from `msg.sender` (Gateway) to itself.
                *   Calls `_enqueue` to add the remaining amount to the redemption queue.
                *   Returns `availableAssetAmount`.
    *   **`claimRedemption(address _recipient)`:**
        *   Restricted to `CoreRoles.ENTRY_POINT`.
        *   Calls `_claimRedemption` (from `RedemptionPool`) to get the claimable asset amount.
        *   Transfers this amount of `assetToken` to the `_recipient`.
    *   **`_deposit(uint256 assetsToDeposit)` (Farm override):**
        *   When assets are deposited into the `RedeemController` (e.g., by a `FARM_MANAGER`), it calls `_fundRedemptionQueue` to process pending redemptions.
        *   If `_fundRedemptionQueue` indicates that `ReceiptToken`s were effectively redeemed (because their corresponding assets were funded), this function calls `burn` on the `ReceiptToken` contract to remove those tokens from circulation.
    *   **Internal Helpers:** `_getReceiptToAssetConvertRatio`, `_convertReceiptToAsset`, `_convertAssetToReceipt` for price conversions.

## Contract Interactions & Funding Flow

### Minting Process:

1.  **User Interaction (via Gateway):** A user intends to mint `ReceiptToken`s. They interact with a function on `InfiniFiGatewayV1` (e.g., `mint()`, `mintAndStake()`).
2.  **Gateway to MintController:** `InfiniFiGatewayV1` (having `CoreRoles.ENTRY_POINT`) calls `MintController.mint(userAddress, assetAmount)`. Before this, the Gateway would have received `assetAmount` of `assetToken` (e.g., USDC) from the user.
3.  **Asset Transfer:** `MintController` transfers `assetAmount` of `assetToken` from `InfiniFiGatewayV1` to itself.
4.  **Price Calculation:** `MintController` calls `Accounting.price()` for both `assetToken` and `receiptToken` to determine the conversion ratio and calculate the amount of `ReceiptToken`s to be minted (`receiptAmountOut`).
5.  **ReceiptToken Minting:** `MintController` (having `CoreRoles.RECEIPT_TOKEN_MINTER` implicitly through its design or explicit grant) calls `ReceiptToken.mint(userAddress, receiptAmountOut)`.
6.  **Token Issuance:** `ReceiptToken` mints the new tokens and assigns them to the `userAddress`.
7.  **(Optional) Hook:** `MintController` may call an `afterMintHook`.

### Redemption Process:

1.  **User Interaction (via Gateway):** A user intends to redeem `ReceiptToken`s for the underlying asset. They interact with a function on `InfiniFiGatewayV1` (e.g., `redeem()`).
2.  **Gateway to RedeemController:** `InfiniFiGatewayV1` (having `CoreRoles.ENTRY_POINT`) calls `RedeemController.redeem(userAddress, receiptAmountIn)`. Before this, the Gateway would have received/approved `receiptAmountIn` of `ReceiptToken` from the user.
3.  **Price Calculation:** `RedeemController` calls `Accounting.price()` for both `assetToken` and `receiptToken` to determine the `assetAmountOut`.
4.  **Liquidity Check & Action:**
    *   **Queue Active:** If `RedeemController.queueLength() > 0`, `RedeemController` transfers `receiptAmountIn` of `ReceiptToken` from the Gateway to itself and calls `_enqueue` (from `RedemptionPool`) to add the request to the queue. The user receives 0 assets immediately.
    *   **Sufficient Liquidity:** If the queue is empty and `RedeemController` has enough `assetToken` to cover `assetAmountOut`:
        *   `RedeemController` (having `CoreRoles.RECEIPT_TOKEN_BURNER`) calls `ReceiptToken.burnFrom(gatewayAddress, receiptAmountIn)` (or `burn` if tokens were transferred first).
        *   `ReceiptToken` burns the specified tokens.
        *   `RedeemController` transfers `assetAmountOut` of `assetToken` to `userAddress`.
    *   **Insufficient Liquidity:** If the queue is empty but `RedeemController` has some, but not enough, `assetToken`:
        *   The available portion is processed immediately: `ReceiptToken`s corresponding to available assets are burned, and available assets are sent to the user.
        *   The remaining unfunded portion: `RedeemController` transfers the corresponding remaining `ReceiptToken`s from the Gateway to itself and calls `_enqueue` to queue the remainder.
5.  **Funding the Queue (via `RedeemController._deposit`):**
    *   When assets are deposited into `RedeemController` (e.g., by a `FARM_MANAGER` calling `deposit()`), `_deposit` is triggered.
    *   `_deposit` calls `_fundRedemptionQueue` (from `RedemptionPool`) with the deposited asset amount and the current conversion ratio.
    *   `_fundRedemptionQueue` processes queued requests, updating `userPendingClaims` and `totalPendingClaims`. It returns the amount of `ReceiptToken` that now corresponds to newly funded assets (`receiptAmountToBurn`).
    *   `RedeemController` then calls `ReceiptToken.burn(receiptAmountToBurn)` to burn these `ReceiptToken`s which are now effectively backed by the deposited assets and claimable by users.
6.  **Claiming Redemption (User via Gateway):**
    *   User calls `claimRedemption()` on `InfiniFiGatewayV1`.
    *   Gateway calls `RedeemController.claimRedemption(userAddress)`.
    *   `RedeemController` calls `_claimRedemption` (from `RedemptionPool`) to get the amount of `assetToken` owed to the user from `userPendingClaims`.
    *   `RedeemController` transfers this amount of `assetToken` to `userAddress`.

## Mermaid Diagram

```mermaid
graph LR
    subgraph "User Interaction"
        User["User"]
    end

    subgraph "Core Protocol Layer"
        Gateway["InfiniFiGatewayV1"]
    end

    subgraph "Funding Module"
        MintController["MintController (Farm)"]
        RedeemController["RedeemController (Farm, RedemptionPool)"]
        ReceiptToken["ReceiptToken (ERC20, CoreControlled)"]
        RedemptionPool["RedemptionPool (Abstract)"]
    end

    subgraph "Supporting Services"
        Accounting["Accounting"]
        InfiniFiCore["InfiniFiCore"]
    end

    subgraph "Assets"
        AssetToken["Asset Token (e.g., USDC)"]
    end

    %% Inheritance
    MintController --|> Farm_MC[Farm]
    RedeemController --|> Farm_RC[Farm]
    RedeemController --|> RedemptionPool
    ReceiptToken --|> CoreControlled_RT[CoreControlled]
    Farm_MC --|> CoreControlled_FMC[CoreControlled]
    Farm_RC --|> CoreControlled_FRC[CoreControlled]


    %% Minting Flow
    User -- "1. Initiate Mint (e.g., USDC)" --> Gateway
    Gateway -- "2. Calls mint(user, assetAmt)" --> MintController
    MintController -- "3. Transfers AssetToken from Gateway" --> AssetToken
    MintController -- "4. Gets prices" --> Accounting
    MintController -- "5. Calls mint(user, receiptAmt)" --> ReceiptToken
    ReceiptToken -- "6. Mints to User" --> User

    %% Redemption Flow (Immediate / Partial + Queue)
    User -- "1. Initiate Redeem (ReceiptTokens)" --> Gateway
    Gateway -- "2. Calls redeem(user, receiptAmt)" --> RedeemController
    RedeemController -- "3. Gets prices" --> Accounting
    RedeemController -- "4a. IF Queue Active OR Insufficient Liquidity (for part)" -->|Transfers ReceiptToken to self & _enqueue| RedemptionPool
    RedeemController -- "4b. IF Sufficient Liquidity (for all/part)" -->|Calls burnFrom(gateway, receiptAmt) / burn()| ReceiptToken
    ReceiptToken -- "4b. Burns Tokens" -->|Updates Supply| ReceiptToken
    RedeemController -- "4b. Transfers AssetToken to User" --> AssetToken
    AssetToken -- "Funds User" --> User


    %% Queue Funding & Claiming
    subgraph "Farm Manager Interaction (Queue Funding)"
      FarmManager["Farm Manager"]
      FarmManager -- "Deposits Assets" --> RedeemController
    end
    RedeemController -- "_deposit() calls _fundRedemptionQueue()" --> RedemptionPool
    RedemptionPool -- "_fundRedemptionQueue updates claims & returns amount to burn" --> RedeemController
    RedeemController -- "Calls burn() on ReceiptToken" --> ReceiptToken

    User -- "Claim Queued Redemption" --> Gateway
    Gateway -- "Calls claimRedemption(user)" --> RedeemController
    RedeemController -- "_claimRedemption() from RedemptionPool" --> RedemptionPool
    RedemptionPool -- "Returns claimable assets" --> RedeemController
    RedeemController -- "Transfers AssetToken to User" --> AssetToken


    %% Role Control
    MintController -- "onlyCoreRole (ENTRY_POINT for mint)" --> InfiniFiCore
    RedeemController -- "onlyCoreRole (ENTRY_POINT for redeem/claim)" --> InfiniFiCore
    ReceiptToken -- "onlyCoreRole (MINTER/BURNER)" --> InfiniFiCore
    MintController -- "onlyCoreRole (FARM_MANAGER for withdraw)" --> InfiniFiCore
    RedeemController -- "onlyCoreRole (FARM_MANAGER for withdraw/deposit)" --> InfiniFiCore

    classDef abstract fill:#E6E6FA,stroke:#333,stroke-width:2px,color:#000
    class RedemptionPool abstract
```
