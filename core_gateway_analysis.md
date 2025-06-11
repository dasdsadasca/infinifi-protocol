# Analysis of InfiniFi Core and Gateway Contracts

This document provides an analysis of the core infrastructure contracts (`InfiniFiCore.sol`, `CoreControlled.sol`) and the primary user interaction point (`InfiniFiGatewayV1.sol`) of the InfiniFi protocol.

## Contract Descriptions

### 1. `InfiniFiCore.sol`

*   **Purpose:** This contract serves as the central hub for Role-Based Access Control (RBAC) within the InfiniFi protocol. It defines and manages all roles and their associated administrative hierarchies.
*   **Functionality:**
    *   Leverages OpenZeppelin's `AccessControlEnumerable` for robust role management.
    *   Initializes a comprehensive set of protocol roles (defined in `CoreRoles.sol`, e.g., `GOVERNOR`, `PAUSE_MANAGER`, `PROTOCOL_PARAMETERS_MANAGER`, `FARM_MANAGER`) upon deployment. The deployer is initially granted the `GOVERNOR` role.
    *   The `GOVERNOR` has the authority to:
        *   Create new roles (`createRole`).
        *   Modify the admin role for existing roles (`setRoleAdmin`).
        *   Grant roles to addresses.
    *   Provides a `grantRoles` function for efficiently assigning multiple roles to multiple accounts in a single transaction. This function ensures the caller possesses the necessary admin privileges for each role being granted.
    *   All roles are initially set to be administered by the `GOVERNOR`, establishing a centralized control structure that can be decentralized later if needed by reassigning admin roles.

### 2. `CoreControlled.sol`

*   **Purpose:** This is an abstract contract intended to be inherited by other protocol contracts that require management or control through `InfiniFiCore`. It standardizes interactions with the core access control system and provides common utilities.
*   **Functionality:**
    *   Maintains a reference to the active `InfiniFiCore` contract instance.
    *   Provides an `onlyCoreRole(bytes32 role)` modifier. This modifier restricts access to functions, ensuring that only `msg.sender`s possessing the specified `role` (as defined in `InfiniFiCore`) can execute them.
    *   Includes a critical function `setCore(address newCore)`, callable exclusively by the `GOVERNOR`, to update the address of the `InfiniFiCore` contract. This allows for upgrading the access control mechanism.
    *   Integrates OpenZeppelin's `Pausable` utility. Contracts inheriting `CoreControlled` can implement pausable functionality, controllable by accounts with `PAUSE` and `UNPAUSE` roles defined in `InfiniFiCore`.
    *   Features an `emergencyAction(Call[] calldata calls)` function. This powerful function, restricted to the `GOVERNOR`, allows for executing arbitrary low-level calls to any contract. It's designed as a fail-safe for critical interventions, enabling the `GOVERNOR` to rectify unforeseen issues or upgrade parts of the system. Each `Call` struct specifies the target address, ETH value, and calldata for the operation.

### 3. `InfiniFiGatewayV1.sol`

*   **Purpose:** This contract is the main user-facing interface and entry point for the InfiniFi protocol. It aggregates functionalities from various underlying modules, simplifying user interactions by providing a single point of contact for most operations.
*   **Functionality:**
    *   **Inheritance & Security:**
        *   Inherits from `CoreControlled`, thereby integrating with `InfiniFiCore` for role-based access control on its administrative functions.
        *   Employs `ReentrancyGuardTransient` to mitigate reentrancy attack vectors on its state-modifying functions.
    *   **Address Registry:**
        *   Maintains a mapping (`addresses`) that serves as an on-chain registry for crucial protocol contract addresses (e.g., `USDC` token, `MintController`, `RedeemController`, `YieldSharing`).
        *   The `GOVERNOR` can update these addresses via `setAddress(string memory _name, address _address)`.
    *   **User Operations:**
        *   **Minting & Staking:**
            *   `mint(address _to, uint256 _amount)`: Allows users to mint iUSD (protocol's receipt token) by depositing USDC.
            *   `mintAndStake(address _to, uint256 _amount)`: Combines minting iUSD and staking it into the `StakedToken` contract in one transaction.
            *   `mintAndLock(address _to, uint256 _amount, uint32 _unwindingEpochs)`: Combines minting iUSD and locking it via the `LockingController`.
        *   **Redemption:**
            *   `redeem(address _to, uint256 _amount, uint256 _minAssetsOut)`: Allows users to redeem their iUSD for the underlying USDC. Interacts with `RedeemController`. It includes a check (`_revertIfThereAreUnaccruedLosses`) to ensure `YieldSharing` has no pending losses before proceeding.
            *   `claimRedemption()`: Allows users to claim previously initiated redemptions that might have been queued.
        *   **Locking & Unwinding (Interacting with `LockingController`):**
            *   `createPosition(uint256 _amount, uint32 _unwindingEpochs, address _recipient)`: Locks iUSD to create a locked position.
            *   `startUnwinding(uint256 _shares, uint32 _unwindingEpochs)`: Initiates the unwinding process for a locked position.
            *   `increaseUnwindingEpochs(...)`, `cancelUnwinding(...)`: Manage existing unwinding processes.
            *   `withdraw(uint256 _unwindingTimestamp)`: Withdraws iUSD from a fully unwound position. Also checks for unaccrued losses.
        *   **Zap Functions (Swap and Interact):**
            *   `zapIn(address _token, uint256 _amount, address _router, bytes calldata _routerData, address _to)`: Allows users to deposit an arbitrary ERC20 token (or ETH), which is then swapped to USDC via a whitelisted external router. The resulting USDC is used to mint iUSD.
            *   `zapInAndStake(...)`: Zaps in and then stakes the minted iUSD.
            *   `zapInAndLock(...)`: Zaps in and then locks the minted iUSD.
            *   A `zapFee` (configurable by `PROTOCOL_PARAMETERS` role) can be charged, with proceeds sent to `YieldSharing`.
            *   `setEnabledRouter(address _router, bool _enabled)`: Manages the whitelist of routers usable for zap functions (controlled by `PROTOCOL_PARAMETERS` role).
        *   **Governance:**
            *   `vote(...)`, `multiVote(...)`: Allows users to participate in allocation voting by calling the `AllocationVoting` contract.
    *   **Safety Checks:**
        *   `_revertIfThereAreUnaccruedLosses()`: Internal view function that checks `YieldSharing.unaccruedYield()`. If there are pending losses (negative yield), relevant withdrawal/redeem functions will revert, prompting a call to `YieldSharing.accrue()` first.

## Contract Interactions

The three contracts interact in the following ways:

1.  **Inheritance:**
    *   `InfiniFiGatewayV1` inherits from `CoreControlled`. This means `InfiniFiGatewayV1` gains all the functionality of `CoreControlled`, including the `onlyCoreRole` modifier, the `core` variable (which stores the `InfiniFiCore` address), and administrative functions like `setCore`, `pause`, `unpause`, and `emergencyAction`.

2.  **Core Access Control (`InfiniFiCore` as the source of truth):**
    *   `CoreControlled` (and by extension, `InfiniFiGatewayV1`) relies on `InfiniFiCore` to perform access control checks.
    *   The `onlyCoreRole` modifier within `CoreControlled` calls `_core.hasRole(role, msg.sender)` on the `InfiniFiCore` instance to verify if the caller has the required role for an action.
    *   Administrative functions within `InfiniFiGatewayV1` (e.g., `setAddress`, `setEnabledRouter`, `setZapFee`) are protected by `onlyCoreRole`, ensuring only authorized accounts (as defined in `InfiniFiCore`) can execute them.

3.  **Gateway as an Orchestrator:**
    *   `InfiniFiGatewayV1` acts as a central dispatcher or facade for user interactions. It does not contain complex business logic itself but rather routes calls to specialized module contracts:
        *   **Minting/Redemption:** `MintController`, `RedeemController`.
        *   **Staking:** `StakedToken`.
        *   **Locking:** `LockingController`, `LockedPositionToken`.
        *   **Governance:** `AllocationVoting`.
        *   **Financial Management:** `YieldSharing` (for fee distribution and loss checks).
        *   **Token Handling:** `USDC` (as the primary stablecoin) and `ReceiptToken` (iUSD).
        *   **Swaps (for Zap):** External DEX aggregators or routers.
    *   The `addresses` mapping within `InfiniFiGatewayV1` is crucial for these interactions, as it provides the locations of these external and internal module contracts.

4.  **Emergency Control:**
    *   If necessary, the `GOVERNOR` can call `emergencyAction` on `InfiniFiGatewayV1` (inherited from `CoreControlled`) to perform arbitrary operations, potentially interacting with any of the modules it controls or other contracts in the ecosystem.

## Mermaid Diagram of Interactions

```mermaid
graph TD
    subgraph CoreInfrastructure [Core Infrastructure]
        style CoreInfrastructure fill:#e0e0e0,stroke:#333,stroke-width:1px
        InfiniFiCore["<strong>InfiniFiCore</strong><br>(Access Control Logic)"]
        CoreControlled["<strong>CoreControlled</strong><br>(Abstract Contract for Core Integration)"]
    end

    subgraph GatewaySystem [Gateway System]
        style GatewaySystem fill:#d0d0ff,stroke:#333,stroke-width:1px
        InfiniFiGatewayV1["<strong>InfiniFiGatewayV1</strong><br>(User Entry Point & Orchestrator)"]
    end

    subgraph ProtocolModules [External & Protocol Modules]
        style ProtocolModules fill:#d0ffd0,stroke:#333,stroke-width:1px
        MintController["MintController"]
        RedeemController["RedeemController"]
        LockingController["LockingController"]
        StakedToken["StakedToken (siUSD)"]
        ReceiptToken["ReceiptToken (iUSD)"]
        AllocationVoting["AllocationVoting"]
        YieldSharing["YieldSharing"]
        ExternalRouters["External Routers (e.g., Uniswap)"]
        USDC["USDC Token"]
        OtherModules["Other Protocol Modules..."]
    end

    %% Inheritance
    InfiniFiGatewayV1 --"Inherits from"--> CoreControlled

    %% Core Interactions
    CoreControlled -. "Uses (for hasRole, getRoleAdmin)" .-> InfiniFiCore
    InfiniFiGatewayV1 -. "Implicitly uses (via CoreControlled for RBAC)" .-> InfiniFiCore

    %% Gateway Orchestration
    InfiniFiGatewayV1 o-- "Manages Addresses of & Calls" --> MintController
    InfiniFiGatewayV1 o-- "Manages Addresses of & Calls" --> RedeemController
    InfiniFiGatewayV1 o-- "Manages Addresses of & Calls" --> LockingController
    InfiniFiGatewayV1 o-- "Manages Addresses of & Calls" --> StakedToken
    InfiniFiGatewayV1 o-- "Manages Addresses of & Calls" --> ReceiptToken
    InfiniFiGatewayV1 o-- "Manages Addresses of & Calls" --> AllocationVoting
    InfiniFiGatewayV1 o-- "Manages Addresses of & Calls" --> YieldSharing
    InfiniFiGatewayV1 o-- "Manages Addresses of & Calls" --> ExternalRouters
    InfiniFiGatewayV1 o-- "Manages Addresses of & Calls" --> USDC
    InfiniFiGatewayV1 o-- "Manages Addresses of & Calls" --> OtherModules

    %% Role Management Flow (Conceptual)
    InfiniFiCore -. "Defines Roles for" .-> InfiniFiGatewayV1
    InfiniFiCore -. "Defines Roles for" .-> CoreControlled %% and its derivatives

    classDef core fill:#ffcccc,stroke:#333,stroke-width:2px;
    classDef gateway fill:#cceeff,stroke:#333,stroke-width:2px;
    classDef module fill:#ccffcc,stroke:#333,stroke-width:2px;

    class InfiniFiCore,CoreControlled core;
    class InfiniFiGatewayV1 gateway;
    class MintController,RedeemController,LockingController,StakedToken,ReceiptToken,AllocationVoting,YieldSharing,ExternalRouters,USDC,OtherModules module;
```

This analysis provides a foundational understanding of how access control is managed and how user interactions are processed at the gateway level in the InfiniFi protocol.
