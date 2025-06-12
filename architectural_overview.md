## Architectural Overview

This document outlines the architecture of the Immutable zkEVM Bridge, focusing on the interaction between the root chain and child chain components, access control mechanisms, and message passing.

### Core Components

The bridge architecture consists of the following key components:

1.  **Root Chain Bridge:** This contract resides on the main Ethereum network (L1). It manages the locking and unlocking of assets on the root chain and facilitates communication with the child chain. While the specific implementation details of `RootERC20Bridge.sol` are not fully explored in this overview, it's understood to be the primary point of interaction for users on L1.
2.  **Child Chain Bridge (Conceptual):** This component resides on the zkEVM chain (L2). It is responsible for minting and burning bridged assets on the child chain, based on messages received from the root chain. The exact code for the child chain bridge is not provided in this context, but its functionality is mirrored to that of the root chain bridge.
3.  **Message Passing Layer:** A generic message passing protocol is used to relay information (e.g., deposit and withdrawal requests) between the root chain and child chain bridges. This layer is crucial for the synchronized operation of the two bridges.

### Access Control: `BridgeRoles.sol`

Access control within the bridge contracts (both root and conceptual child chain) is managed by the `BridgeRoles.sol` contract. This abstract contract leverages OpenZeppelin's `AccessControlUpgradeable` and `PausableUpgradeable` to provide a robust and flexible permissions system.

Key roles defined in `BridgeRoles.sol` include:

*   **`DEFAULT_ADMIN_ROLE`**: This is the highest-level administrative role, typically responsible for granting and revoking other roles.
*   **`PAUSER_ROLE`**: Accounts with this role can pause critical functions of the bridge, halting operations in case of emergencies or upgrades.
*   **`UNPAUSER_ROLE`**: Accounts with this role can unpause the bridge, resuming normal operations.
*   **`ADAPTOR_MANAGER_ROLE`**: This role is responsible for managing the bridge adaptor, which is a key component in the message passing mechanism. This likely includes updating the adaptor address or its configuration.

By inheriting `BridgeRoles.sol`, the bridge contracts ensure that sensitive operations are restricted to authorized entities, enhancing the security and manageability of the system.

### Message Passing: `IRootBridgeAdaptor.sol`

Communication from the root chain to the child chain is facilitated by an abstraction defined in the `IRootBridgeAdaptor.sol` interface. This interface decouples the `RootERC20Bridge` contract from the specific implementation details of the underlying General Purpose Message Passing (GMP) protocol.

The primary function in this interface is:

*   **`sendMessage(bytes calldata payload, address refundRecipient) external payable`**:
    *   This function is called by the `RootERC20Bridge` to send an arbitrary message (payload) to the child chain.
    *   The `payload` contains the encoded information about the action to be replicated on the child chain (e.g., minting tokens after a deposit on the root chain).
    *   The `refundRecipient` address is specified in case the GMP protocol requires fees and needs to refund any excess.
    *   The function is `payable`, indicating that the underlying message passing protocol might require a fee for its services.

This design allows for flexibility, as the specific GMP protocol (e.g., Axelar, LayerZero, or a custom solution) can be swapped or updated without requiring significant changes to the core bridge logic, as long as the new adaptor adheres to the `IRootBridgeAdaptor` interface.

### High-Level Architecture Diagram

```mermaid
graph TD
    subgraph Root Chain (L1)
        UserL1[User] -->|Interacts with| RootBridge[RootERC20Bridge]
        RootBridge -- Inherits --> BridgeRolesL1[BridgeRoles.sol]
        RootBridge -->|Sends message via| RootAdaptor[IRootBridgeAdaptor]
    end

    subgraph Child Chain (L2) - Conceptual
        ChildBridge[Child Chain Bridge]
        ChildBridge -- Inherits --> BridgeRolesL2[BridgeRoles.sol]
        ChildAdaptor[Child Bridge Adaptor] -->|Receives message| ChildBridge
    end

    RootAdaptor -->|Message Passing Protocol (e.g., Axelar, LayerZero)| ChildAdaptor

    style UserL1 fill:#lightgrey,stroke:#333,stroke-width:2px
    style RootBridge fill:#lightblue,stroke:#333,stroke-width:2px
    style BridgeRolesL1 fill:#moccasin,stroke:#333,stroke-width:2px
    style RootAdaptor fill:#thistle,stroke:#333,stroke-width:2px

    style ChildBridge fill:#lightblue,stroke:#333,stroke-width:2px
    style BridgeRolesL2 fill:#moccasin,stroke:#333,stroke-width:2px
    style ChildAdaptor fill:#thistle,stroke:#333,stroke-width:2px
```
This diagram illustrates the flow of operations: A user interacts with the `RootERC20Bridge` on L1. This bridge, governed by `BridgeRoles.sol` for access control, uses an implementation of `IRootBridgeAdaptor` to send a message to the child chain. The message travels through an external message passing protocol and is received by a corresponding adaptor on the child chain, which then interacts with the Child Chain Bridge (also governed by `BridgeRoles.sol`) to execute the corresponding action (e.g., minting tokens).
