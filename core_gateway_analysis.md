# InfiniFi Protocol: Core and Gateway Contracts Analysis

This document provides an analysis of the core and gateway contracts within the InfiniFi protocol.

## Contract Descriptions

### 1. `InfiniFiCore.sol`

*   **Purpose:** This contract serves as the central access control and role management hub for the InfiniFi protocol. It defines and manages various roles and their administrative hierarchies.
*   **Functionality:**
    *   It inherits from OpenZeppelin's `AccessControlEnumerable` to provide a robust role-based access control (RBAC) system.
    *   Upon deployment, it grants the `GOVERNOR` role to the deployer (`msg.sender`).
    *   It initializes a set of predefined roles (e.g., `PAUSE`, `UNPAUSE`, `PROTOCOL_PARAMETERS`, `ENTRY_POINT`, `RECEIPT_TOKEN_MINTER`, etc.) and sets their admin role to `GOVERNOR`. This means the `GOVERNOR` has the authority to manage these roles.
    *   **`createRole(bytes32 role, bytes32 adminRole)`:** Allows the `GOVERNOR` to create new roles and assign an admin role to them. It prevents the creation of a role that already exists.
    *   **`setRoleAdmin(bytes32 role, bytes32 adminRole)`:** Allows the `GOVERNOR` to change the admin role of an existing role. It requires that the role already exists.
    *   **`grantRoles(bytes32[] calldata roles, address[] calldata accounts)`:** Allows an address with the appropriate admin privileges for a set of roles to grant those roles to multiple accounts in a batch. The caller must have the admin role for *each* role being granted.

### 2. `CoreControlled.sol`

*   **Purpose:** This is an abstract contract designed to be inherited by other contracts within the InfiniFi protocol that need to be controlled or managed by `InfiniFiCore`. It provides common functionalities related to core contract interaction, pausable behavior, and emergency actions.
*   **Functionality:**
    *   It inherits from OpenZeppelin's `Pausable` contract, enabling deriving contracts to implement pause/unpause functionality.
    *   **`_core`:** Stores a reference to the `InfiniFiCore` contract.
    *   **`constructor(address coreAddress)`:** Initializes the contract with the address of the `InfiniFiCore` contract.
    *   **`onlyCoreRole(bytes32 role)`:** A modifier that restricts access to functions to only those callers who have the specified `role` granted in the `InfiniFiCore` contract.
    *   **`core()`:** Returns the address of the `InfiniFiCore` contract.
    *   **`setCore(address newCore)`:** Allows an account with the `GOVERNOR` role (from `InfiniFiCore`) to update the address of the `InfiniFiCore` contract. This is a critical function and can "brick" the contract if set incorrectly.
    *   **`pause()`:** Allows an account with the `PAUSE` role to pause the contract.
    *   **`unpause()`:** Allows an account with the `UNPAUSE` role to unpause the contract.
    *   **`emergencyAction(Call[] calldata calls)`:** A critical function callable only by the `GOVERNOR`. It allows the execution of arbitrary low-level calls (`target.call{value: value}(callData)`) to multiple target contracts. This is intended for emergency situations to perform corrective actions. The `Call` struct defines the target address, ETH value, and calldata for each action.

### 3. `InfiniFiGatewayV1.sol`

*   **Purpose:** This contract acts as the main entry point for users to interact with the InfiniFi protocol's features. It orchestrates various operations like minting, staking, locking, redeeming, and voting by interacting with other specialized controller and token contracts.
*   **Functionality:**
    *   It inherits from `CoreControlled`, making its functions subject to role-based access control managed by `InfiniFiCore` and allowing it to be paused/unpaused.
    *   It also inherits from `ReentrancyGuardTransient` to protect against reentrancy attacks.
    *   **Address Registry:** Maintains a mapping (`addresses`) to store and retrieve addresses of other important protocol contracts (e.g., `USDC`, `mintController`, `stakedToken`, `receiptToken`, `lockingController`, `yieldSharing`, `allocationVoting`). These addresses are configurable by the `GOVERNOR`.
    *   **Zap Functionality:**
        *   Allows users to deposit various tokens (including ETH via `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE`) and convert them into protocol-specific tokens (e.g., receipt tokens, staked tokens, or locked positions) in a single transaction ("zap").
        *   Manages a whitelist of `enabledRouters` (e.g., DEX routers) that can be used for token swaps during zaps.
        *   A `zapFee` can be charged on zap operations, with the collected fees transferred to the `yieldSharing` contract.
        *   Functions: `zapIn`, `zapInAndStake`, `zapInAndLock`.
    *   **Minting:**
        *   `mint(address _to, uint256 _amount)`: Allows users to mint receipt tokens by providing USDC.
        *   `mintAndStake(address _to, uint256 _amount)`: Mints receipt tokens and stakes them in the `StakedToken` contract.
        *   `mintAndLock(address _to, uint256 _amount, uint32 _unwindingEpochs)`: Mints receipt tokens and locks them using the `LockingController`.
    *   **Locking & Unwinding:**
        *   `createPosition(uint256 _amount, uint32 _unwindingEpochs, address _recipient)`: Creates a new locked position.
        *   `startUnwinding(uint256 _shares, uint32 _unwindingEpochs)`: Initiates the unwinding process for a locked position.
        *   `increaseUnwindingEpochs(...)`, `cancelUnwinding(...)`: Manage existing unwinding processes.
        *   `withdraw(uint256 _unwindingTimestamp)`: Allows users to withdraw their assets after the unwinding period, but only if there are no unaccrued losses in the `YieldSharing` contract.
    *   **Redeeming:**
        *   `redeem(address _to, uint256 _amount, uint256 _minAssetsOut)`: Allows users to redeem their receipt tokens for underlying assets (e.g., USDC). Requires no unaccrued losses.
        *   `claimRedemption()`: Allows users to claim assets from a redemption process.
    *   **Voting:**
        *   `vote(...)`, `multiVote(...)`: Allows users to participate in allocation voting using their locked or staked assets, interacting with the `AllocationVoting` contract.
    *   **Configuration:**
        *   `setAddress(string memory _name, address _address)`: `GOVERNOR` can set addresses of dependent contracts.
        *   `setEnabledRouter(address _router, bool _enabled)`: `PROTOCOL_PARAMETERS` role can manage zap routers.
        *   `setZapFee(uint256 _zapFee)`: `PROTOCOL_PARAMETERS` role can set the zap fee.
    *   **Internal Checks:**
        *   `_revertIfThereAreUnaccruedLosses()`: An internal view function that checks the `YieldSharing` contract. If `unaccruedYield()` is negative (indicating pending losses), it reverts the transaction. This is used to protect the protocol during withdrawals and redemptions.

## Contract Interactions

1.  **`InfiniFiCore` as the Central Authority:**
    *   Both `CoreControlled` (and by extension, `InfiniFiGatewayV1`) rely on `InfiniFiCore` for role checking.
    *   `InfiniFiGatewayV1` uses the `onlyCoreRole` modifier (defined in `CoreControlled`) to restrict access to sensitive functions like `setAddress`, `setEnabledRouter`, `setZapFee`, `pause`, and `unpause`. These modifiers query `InfiniFiCore` to verify if `msg.sender` has the required role (e.g., `CoreRoles.GOVERNOR`, `CoreRoles.PROTOCOL_PARAMETERS`).
    *   The `InfiniFiCore` address itself can be updated in `CoreControlled` (and thus in `InfiniFiGatewayV1`) via the `setCore` function, which is restricted to the `GOVERNOR` role.

2.  **`CoreControlled` as a Base for Controlled Contracts:**
    *   `InfiniFiGatewayV1` inherits from `CoreControlled`. This inheritance provides:
        *   A direct reference (`_core`) to the `InfiniFiCore` contract.
        *   The `onlyCoreRole` modifier.
        *   Pausable functionality (`pause()`, `unpause()`), controlled by `PAUSE` and `UNPAUSE` roles defined in `InfiniFiCore`.
        *   The `emergencyAction` function, callable by the `GOVERNOR` role.

3.  **`InfiniFiGatewayV1` Orchestrating Protocol Functions:**
    *   **Address Dependencies:** `InfiniFiGatewayV1` heavily relies on its internal address registry (`addresses`) to interact with other crucial protocol components. These components are not directly detailed in the provided code but are referenced by their roles/names:
        *   `MintController`: For minting receipt tokens.
        *   `RedeemController`: For redeeming receipt tokens.
        *   `LockingController`: For managing locked positions.
        *   `StakedToken` (`siusd`): For staking receipt tokens.
        *   `ReceiptToken` (`iusd`): The primary token representing a user's deposit.
        *   `LockedPositionToken` (`liusd`): Represents shares in a locked position.
        *   `AllocationVoting`: For users to vote on asset allocations.
        *   `YieldSharing`: For managing yield distribution and tracking unaccrued yield/losses.
        *   `ERC20` (USDC): The primary stablecoin used for minting.
    *   **User Flow Example (Mint & Stake):**
        1.  User calls `mintAndStake` on `InfiniFiGatewayV1`.
        2.  `InfiniFiGatewayV1` transfers USDC from the user.
        3.  It approves the `MintController` to spend this USDC.
        4.  It calls `mint` on the `MintController`, which creates `ReceiptToken` (iusd).
        5.  `InfiniFiGatewayV1` then approves the `StakedToken` (siusd) contract to spend these iusd.
        6.  Finally, it calls `deposit` on the `StakedToken` contract to stake the iusd for the user.
    *   **Calls to `InfiniFiCore` (via `CoreControlled`):**
        *   When a user calls a restricted function on `InfiniFiGatewayV1` (e.g., an admin trying to `setAddress`), the `onlyCoreRole` modifier is triggered.
        *   This modifier calls `_core.hasRole(role, msg.sender)` (where `_core` is `InfiniFiCore`) to check permissions.

## Mermaid Diagram

```mermaid
graph TD
    subgraph "Access Control"
        InfiniFiCore["InfiniFiCore (AccessControlEnumerable)"]
    end

    subgraph "Controlled Contracts"
        CoreControlled["CoreControlled (Pausable)"]
        InfiniFiGatewayV1["InfiniFiGatewayV1 (ReentrancyGuardTransient)"]
    end

    subgraph "External Protocol Components (Referenced by InfiniFiGatewayV1)"
        MintController["MintController"]
        RedeemController["RedeemController"]
        LockingController["LockingController"]
        StakedToken["StakedToken (siUSD)"]
        ReceiptToken["ReceiptToken (iUSD)"]
        LockedPositionToken["LockedPositionToken (liUSD)"]
        AllocationVoting["AllocationVoting"]
        YieldSharing["YieldSharing"]
        USDC["USDC (ERC20)"]
        DEXRouter["DEX Router(s)"]
    end

    %% Inheritance
    CoreControlled --|> Pausable_OZ[Pausable @openzeppelin]
    InfiniFiCore --|> AccessControlEnumerable_OZ[AccessControlEnumerable @openzeppelin]
    InfiniFiGatewayV1 --|> CoreControlled
    InfiniFiGatewayV1 --|> ReentrancyGuardTransient_OZ[ReentrancyGuardTransient @openzeppelin]

    %% Interactions: InfiniFiGatewayV1 uses CoreControlled
    InfiniFiGatewayV1 -->|references| CoreControlled_Util["CoreControlled Utilities (modifier, core(), setCore(), pause(), emergencyAction())"]
    CoreControlled_Util --> InfiniFiCore_Instance["InfiniFiCore Instance (_core)"]

    %% Interactions: CoreControlled uses InfiniFiCore
    CoreControlled_Util --"calls for role checks (hasRole)"--> InfiniFiCore
    CoreControlled_Util --"calls to update core (setCore)"--> InfiniFiCore_Instance

    %% Interactions: InfiniFiGatewayV1 calls other protocol components
    InfiniFiGatewayV1 --"calls mint()"--> MintController
    InfiniFiGatewayV1 --"calls redeem()"--> RedeemController
    InfiniFiGatewayV1 --"calls createPosition(), startUnwinding(), etc."--> LockingController
    InfiniFiGatewayV1 --"calls deposit(), redeem()"--> StakedToken
    InfiniFiGatewayV1 --"calls transferFrom(), approve()"--> ReceiptToken
    InfiniFiGatewayV1 --"calls transferFrom(), approve()"--> LockedPositionToken
    InfiniFiGatewayV1 --"calls vote(), multiVote()"--> AllocationVoting
    InfiniFiGatewayV1 --"transfers zap fees, checks unaccruedYield()"--> YieldSharing
    InfiniFiGatewayV1 --"transfers from user, approves"--> USDC
    InfiniFiGatewayV1 --"calls for token swaps (zap)"--> DEXRouter

    %% Role Management by InfiniFiCore
    InfiniFiCore --"manages roles for"--> InfiniFiGatewayV1_AdminFuncs["Admin Functions in InfiniFiGatewayV1 (setAddress, setZapFee, etc.)"]
    InfiniFiCore --"manages GOVERNOR, PAUSE, UNPAUSE roles for"--> CoreControlled

    classDef ozContract fill:#DCDCDC,stroke:#333,stroke-width:2px,color:#000
    class Pausable_OZ,AccessControlEnumerable_OZ,ReentrancyGuardTransient_OZ ozContract
```
