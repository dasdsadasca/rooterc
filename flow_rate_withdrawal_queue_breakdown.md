## Smart Contract Breakdown: `FlowRateWithdrawalQueue.sol` (Abstract Contract)

**Source:** `src/root/flowrate/FlowRateWithdrawalQueue.sol`
**Interface:** `src/interfaces/root/flowrate/IFlowRateWithdrawalQueue.sol`

### Purpose and Function

`FlowRateWithdrawalQueue.sol` is an abstract contract designed to manage a queue of token withdrawals that have been delayed. These delays can occur due to flow rate limits being triggered (as detected by a contract like `FlowRateDetection.sol`) or due to a manual activation of a global withdrawal halt. The contract provides a mechanism for these delayed withdrawals to be processed by users after a configurable waiting period.

It is intended to be inherited by other contracts (e.g., `RootERC20BridgeFlowRate.sol`) that need to implement a delayed withdrawal feature. The contract is marked as upgradeable.

The "queue" is implemented as a per-user array of `PendingWithdrawal` structs. Users can query their own queue and process individual withdrawals once the delay period has passed.

### Key Data Structures

*   **`PendingWithdrawal` Struct:**
    *   `withdrawer (address)`: The account that initiated the cross-chain transfer on the child chain.
    *   `token (address)`: The ERC20 token being withdrawn.
    *   `amount (uint256)`: The quantity of the token to be withdrawn.
    *   `timestamp (uint256)`: The `block.timestamp` when the withdrawal was enqueued. This is used to calculate when the withdrawal becomes available.

*   **`pendingWithdrawals (mapping(address => PendingWithdrawal[])) private`**:
    *   A mapping where the key is the `receiver`'s address (the L1 address entitled to the funds) and the value is an array of `PendingWithdrawal` structs. Each user (`receiver`) has their own independent list of pending withdrawals.

*   **`withdrawalDelay (uint256 public)`**:
    *   The duration (in seconds) that a withdrawal must remain in the queue before it can be processed. This delay is global for all queued withdrawals. It can be updated, and changes will affect withdrawals already in the queue.

### Key Functions

#### Initialization and Configuration

*   **`__FlowRateWithdrawalQueue_init() internal`**:
    *   An internal initializer function (intended to be called by the inheriting contract's initializer).
    *   Calls `_setWithdrawalDelay(DEFAULT_WITHDRAW_DELAY)`, setting the initial withdrawal delay to `1 days` (defined as `DEFAULT_WITHDRAW_DELAY`).
*   **`_setWithdrawalDelay(uint256 delay) internal`**:
    *   Allows the inheriting contract (presumably with appropriate access control) to change the `withdrawalDelay`.
    *   Emits `WithdrawalDelayUpdated(delay, previousDelay)`.
    *   Crucially, changes to `withdrawalDelay` affect pending withdrawals that haven't been processed yet, as the eligibility check in `_processWithdrawal` uses the current `withdrawalDelay`.

#### Queue Management

*   **`_enqueueWithdrawal(address receiver, address withdrawer, address token, uint256 amount) internal`**:
    *   Purpose: Adds a new withdrawal request to the specified `receiver`'s queue.
    *   Input: `receiver` (L1 recipient), `withdrawer` (L2 initiator), `token` address, and `amount`.
    *   Checks:
        *   `token != address(0)` (reverts with `TokenIsZero`).
    *   Logic:
        1.  Creates a `PendingWithdrawal` struct with the provided details and the current `block.timestamp`.
        2.  Appends this struct to the `pendingWithdrawals[receiver]` array.
        3.  `uint256 index = pendingWithdrawals[receiver].length - 1;` (as length is updated before this line, the correct index is `length` if using a temporary variable for it, or `length-1` if referring to the new length directly). The code uses `pendingWithdrawals[receiver].length` *before* the push to get the correct index for the *new* element.
        4.  Emits `EnQueuedWithdrawal(token, withdrawer, receiver, amount, block.timestamp, index)`.

*   **`_processWithdrawal(address receiver, uint256 index) internal returns (address withdrawer, address token, uint256 amount)`**:
    *   Purpose: Allows a `receiver` to process (i.e., validate and retrieve details of) a specific withdrawal from their queue after the `withdrawalDelay` has passed. The actual token transfer is handled by the inheriting contract using the returned details.
    *   Input: `receiver` address and `index` of the withdrawal in their `pendingWithdrawals` array.
    *   Output: `withdrawer`, `token`, `amount` of the processed withdrawal.
    *   Checks:
        1.  `index < pendingWithdrawals[receiver].length` (reverts with `IndexOutsideWithdrawalQueue` if out of bounds).
        2.  Retrieves `PendingWithdrawal storage withdrawal = withdrawals[index]`.
        3.  `withdrawal.token != address(0)` (reverts with `WithdrawalAlreadyProcessed`). This check is important because processed entries are `delete`d, which zeroes out the struct members.
        4.  **Withdrawal Time Calculation:** `uint256 withdrawalTime = withdrawal.timestamp + withdrawalDelay;`
        5.  `block.timestamp >= withdrawalTime` (reverts with `WithdrawalRequestTooEarly` if called before the withdrawal is eligible).
    *   Logic:
        1.  If all checks pass, the function retrieves `withdrawer`, `token`, and `amount` from the `withdrawal` struct.
        2.  `delete withdrawals[index]`: The processed withdrawal entry is deleted (zeroed out) from the array to save gas on future reads of this slot and to mark it as processed. This does not change array length or reorder elements, leaving a "hole."
        3.  Emits `ProcessedWithdrawal(token, withdrawer, receiver, amount, block.timestamp, index)`.
        4.  Returns the withdrawal details for the inheriting contract to act upon.

#### View/Utility Functions

*   **`getPendingWithdrawalsLength(address receiver) external view returns (uint256 length)`**:
    *   Returns the total number of entries (including processed/deleted ones, which are zeroed out but still occupy a slot) in a `receiver`'s `pendingWithdrawals` array.
*   **`getPendingWithdrawals(address receiver, uint256[] calldata indices) external view returns (PendingWithdrawal[] memory pending)`**:
    *   Fetches multiple `PendingWithdrawal` structs for a `receiver` based on an array of `indices`.
    *   If an index is out of bounds, a zero-filled `PendingWithdrawal` struct is returned for that position.
*   **`findPendingWithdrawals(address receiver, address token, uint256 startIndex, uint256 stopIndex, uint256 maxFind) external view returns (FindPendingWithdrawal[] memory found)`**:
    *   Searches a `receiver`'s queue for pending withdrawals matching a specific `token`.
    *   The search is performed within the range `[startIndex, stopIndex)` or until `maxFind` items are found.
    *   Returns an array of `FindPendingWithdrawal` structs, which include the `index`, `amount`, and `timestamp` of matching entries. This is useful for UIs to help users find specific withdrawals.

### Event Emissions (from `IFlowRateWithdrawalQueueEvents`)

*   `EnQueuedWithdrawal(address indexed token, address indexed withdrawer, address indexed receiver, uint256 amount, uint256 timestamp, uint256 index)`: Emitted when a new withdrawal is added to a queue.
*   `ProcessedWithdrawal(address indexed token, address indexed withdrawer, address indexed receiver, uint256 amount, uint256 timestamp, uint256 index)`: Emitted when a withdrawal is successfully processed (i.e., validated and its details retrieved).
*   `WithdrawalDelayUpdated(uint256 delay, uint256 previousDelay)`: Emitted when the global `withdrawalDelay` is changed.

### Custom Errors (from `IFlowRateWithdrawalQueueErrors`)

*   `IndexOutsideWithdrawalQueue(uint256 lengthOfQueue, uint256 requestedIndex)`: The provided `index` for processing is out of bounds of the user's queue.
*   `WithdrawalRequestTooEarly(uint256 timeNow, uint256 currentWithdrawalTime)`: Attempt to process a withdrawal before its `timestamp + withdrawalDelay`.
*   `WithdrawalAlreadyProcessed(address receiver, uint256 index)`: Attempt to process a withdrawal that has already been processed (indicated by its `token` field being `address(0)` due to `delete`).
*   `TokenIsZero(address receiver)`: Attempt to enqueue a withdrawal with `token == address(0)`.

### Calculations Focus

*   **`_processWithdrawal`: `withdrawalTime` calculation**
    *   `uint256 withdrawalTime = withdrawal.timestamp + withdrawalDelay;`
    *   `withdrawal.timestamp`: The time (block timestamp) when the withdrawal was originally enqueued.
    *   `withdrawalDelay`: The current global delay period that all withdrawals must wait.
    *   The sum gives the earliest `block.timestamp` at which the withdrawal can be processed.
    *   The check `block.timestamp < withdrawalTime` ensures this condition is met.
    *   It's important that `withdrawalDelay` is added at the time of processing, not at enqueuing. This means if `withdrawalDelay` is updated by governance, the new delay applies to all withdrawals still in the queue.

### Potential Risks

*   **`withdrawalDelay` Misconfiguration**:
    *   If `withdrawalDelay` is set to an extremely large value (e.g., years), it could effectively lock up queued funds for that duration, severely impacting users. Governance or administrative control over `_setWithdrawalDelay` must be managed carefully.
*   **Gas Limit Issues with Large User Queues**:
    *   The `pendingWithdrawals` array for a user can grow indefinitely if they have many queued withdrawals.
    *   While `_processWithdrawal` only accesses one element by index (which is gas efficient), the `delete` operation on that element provides a gas refund.
    *   The view functions `getPendingWithdrawals` and `findPendingWithdrawals` iterate based on input parameters. If `indices` array is huge for `getPendingWithdrawals`, or `stopIndex - startIndex` is very large for `findPendingWithdrawals`, these calls could consume a lot of gas and potentially hit block gas limits if called on-chain (though they are `view` functions, typically called off-chain).
    *   The primary concern is off-chain systems or UIs needing to parse potentially very large arrays if a user accumulates many queued items. The design expects users to process their withdrawals by providing the specific index.
*   **Timestamp Reliance (`block.timestamp`)**:
    *   Both `_enqueueWithdrawal` (to set `PendingWithdrawal.timestamp`) and `_processWithdrawal` (to check against `withdrawalTime`) use `block.timestamp`.
    *   As with `FlowRateDetection.sol`, minor manipulation by miners/validators is possible but generally has limited impact on the delay period, which is often in hours or days. The `solhint-disable-next-line not-rely-on-time` comments acknowledge this.

### Mermaid Diagram: Withdrawal Queue Mechanism

```mermaid
graph TD
    subgraph EnqueueProcess
        A[User Action on L2 (Withdrawal)] --> B{Bridge Logic: Delay Withdrawal?};
        B -- Yes --> C[_enqueueWithdrawal(receiver, withdrawer, token, amount)];
        C --> D[PendingWithdrawal created (timestamp = block.timestamp)];
        D --> E[Add to pendingWithdrawals[receiver] array];
        E --> F[Emit EnQueuedWithdrawal];
    end

    subgraph ProcessDelay
        G[Later Time...]
    end

    subgraph ProcessWithdrawal
        H[User Calls Inheriting Contract to Process] --> I[_processWithdrawal(receiver, index)];
        I --> J{Index Valid?};
        J -- No --> K[Revert IndexOutsideWithdrawalQueue];
        J -- Yes --> L{Already Processed? (token == 0)};
        L -- Yes --> M[Revert WithdrawalAlreadyProcessed];
        L -- No --> N[entryTimestamp = withdrawal.timestamp];
        N --> O[eligibleTime = entryTimestamp + currentWithdrawalDelay];
        O --> P{block.timestamp >= eligibleTime?};
        P -- No --> Q[Revert WithdrawalRequestTooEarly];
        P -- Yes --> R[Retrieve withdrawal details (token, amount, withdrawer)];
        R --> S[delete pendingWithdrawals[receiver][index]];
        S --> T[Emit ProcessedWithdrawal];
        T --> U[Return details to Inheriting Contract for Token Transfer];
    end

    subgraph Configuration
        V[Admin/Governance] --> W[_setWithdrawalDelay(newDelay)];
        W --> X[Update global withdrawalDelay];
        X --> Y[Emit WithdrawalDelayUpdated];
    end

    F --> G;
    G --> H;


    style A fill:#lightgrey,stroke:#333,stroke-width:1px
    style B fill:# khaki,stroke:#333,stroke-width:1px
    style C fill:#lightblue,stroke:#333,stroke-width:2px
    style I fill:#lightblue,stroke:#333,stroke-width:2px
    style W fill:#lightblue,stroke:#333,stroke-width:2px
    style U fill:#lightgreen,stroke:#333,stroke-width:2px
    style K fill:#lightcoral,stroke:#333,stroke-width:2px
    style M fill:#lightcoral,stroke:#333,stroke-width:2px
    style Q fill:#lightcoral,stroke:#333,stroke-width:2px
```

This diagram outlines the lifecycle of a withdrawal from being enqueued due to a delay, to the point where a user can process it after the requisite `withdrawalDelay` has passed. It also shows the administrative function to update the delay period.
