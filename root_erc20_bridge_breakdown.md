## Smart Contract Breakdown: `RootERC20Bridge.sol`

**Source:** `src/root/RootERC20Bridge.sol`

### Purpose and Function

The `RootERC20Bridge.sol` contract is a cornerstone of the Immutable zkEVM bridging mechanism, residing on the Layer 1 (root chain, e.g., Ethereum). Its primary purpose is to facilitate the movement of assets between the root chain and the child chain (Immutable zkEVM). It handles:

1.  **Token Mapping:** Registering ERC20 tokens from the root chain to enable their bridging to corresponding tokens on the child chain.
2.  **Deposits:** Allowing users to deposit native ETH, Wrapped ETH (WETH), IMX tokens, and other standard ERC20 tokens into the bridge. These assets are then locked on the root chain, and a corresponding representation is (or will be) minted/unlocked on the child chain.
3.  **Withdrawals:** Processing messages from the child chain to release previously locked assets back to users on the root chain.

The contract is designed to be upgradeable (using OpenZeppelin's TransparentUpgradeableProxy pattern) and interacts with a bridge adaptor (`IRootBridgeAdaptor`) to decouple its core logic from the specifics of the underlying cross-chain messaging protocol. It also incorporates role-based access control inherited from `BridgeRoles.sol`.

### Key Functions

#### Initialization

*   **`constructor(address _initializerAddress)`**:
    *   Sets the `initializerAddress`, which is the only address authorized to call the `initialize` function. This is a security measure against front-running.
    *   Reverts if `_initializerAddress` is the zero address.
*   **`initialize(InitializationRoles memory newRoles, ...)`**:
    *   A standard OpenZeppelin `initializer` function, callable only once.
    *   Restricted to `initializerAddress` set in the constructor.
    *   Sets up initial roles (admin, pauser, unpauser, variable manager, adaptor manager) by calling `__AccessControl_init()`, `__Pausable_init()`, `__ReentrancyGuard_init()`, and then granting roles.
    *   Stores crucial addresses: `rootBridgeAdaptor`, `childERC20Bridge`, `childTokenTemplate` (for predicting child token addresses), `rootIMXToken`, and `rootWETHToken`.
    *   Predicts and stores `childETHToken` address using `Clones.predictDeterministicAddress`.
    *   Sets the initial `imxCumulativeDepositLimit`.
    *   Pre-maps IMX, ETH, and WETH:
        *   `rootTokenToChildToken[rootIMXToken] = rootIMXToken` (IMX is native on child)
        *   `rootTokenToChildToken[NATIVE_ETH] = NATIVE_ETH` (ETH represented as native on child, or wrapped if childETHToken is ERC20)
        *   `rootTokenToChildToken[rootWETHToken] = NATIVE_ETH` (WETH on L1 becomes native/wrapped ETH on L2)
    *   Reverts if any critical address parameters are zero.

#### Token Mapping

*   **`mapToken(IERC20Metadata rootToken) external payable returns (address)`**:
    *   Allows registration of a standard ERC20 token on the root chain for bridging.
    *   Requires `msg.value > 0` to pay for the cross-chain message.
    *   Prevents mapping of zero address, `rootIMXToken`, `NATIVE_ETH`, or `rootWETHToken` as these are handled specially or pre-mapped.
    *   Reverts if the token is `AlreadyMapped`.
    *   **Child Token Address Prediction:** Calculates the corresponding child token address deterministically using `Clones.predictDeterministicAddress(childTokenTemplate, keccak256(abi.encodePacked(rootToken)), childERC20Bridge)`. This allows knowing the child token address before it's deployed.
    *   Stores the mapping: `rootTokenToChildToken[address(rootToken)] = childToken`.
    *   Retrieves token `name`, `symbol`, and `decimals` from the `rootToken`. Reverts with `TokenNotSupported` if these functions are not present (as they are optional in ERC20).
    *   Sends a message to the child chain via `rootBridgeAdaptor.sendMessage()` with a payload containing `MAP_TOKEN_SIG`, root token details, and child token details.
    *   Emits `L1TokenMapped(address(rootToken), childToken)`.

#### Deposit Mechanisms

The contract supports depositing generic ERC20s, ETH, and WETH. IMX is handled as a generic ERC20 deposit but with special limits.

*   **`depositETH(uint256 amount) external payable`**:
    *   Public function for users to deposit native ETH. Calls `_depositETH(msg.sender, amount)`.
*   **`depositToETH(address receiver, uint256 amount) external payable`**:
    *   Public function for users to deposit native ETH to a specific `receiver` on the child chain. Calls `_depositETH(receiver, amount)`.
*   **`_depositETH(address receiver, uint256 amount)` (private)**:
    *   Ensures `msg.value >= amount`.
    *   **`expectedBalance` Calculation:** `uint256 expectedBalance = address(this).balance - (msg.value - amount);` This accounts for the actual ETH received (`msg.value`) potentially being more than `amount` (e.g., to cover gas for the message), ensuring the contract's ETH balance increases by exactly `amount`.
    *   Calls the internal `_deposit(IERC20Metadata(NATIVE_ETH), receiver, amount)`.
    *   Performs a balance invariant check: `if (address(this).balance != expectedBalance) revert BalanceInvariantCheckFailed(...)`.
*   **`deposit(IERC20Metadata rootToken, uint256 amount) external payable`**:
    *   Public function for users to deposit ERC20 tokens. Calls `_depositToken(rootToken, msg.sender, amount)`.
*   **`depositTo(IERC20Metadata rootToken, address receiver, uint256 amount) external payable`**:
    *   Public function for users to deposit ERC20 tokens to a specific `receiver`. Calls `_depositToken(rootToken, receiver, amount)`.
*   **`_depositToken(IERC20Metadata rootToken, address receiver, uint256 amount)` (private)**:
    *   If `rootToken` is `rootWETHToken`, calls `_depositWrappedETH(receiver, amount)`.
    *   Otherwise, calls `_depositERC20(rootToken, receiver, amount)`.
*   **`_depositWrappedETH(address receiver, uint256 amount)` (private)**:
    *   **`expectedBalance` Calculation:** `uint256 expectedBalance = address(this).balance + amount;` (Native ETH balance should increase by `amount` after unwrapping WETH).
    *   Transfers `amount` of WETH from `msg.sender` to the bridge contract using `safeTransferFrom`.
    *   Unwraps the received WETH into native ETH by calling `IWETH(rootWETHToken).withdraw(amount)`. The bridge's `receive()` function handles the incoming ETH.
    *   Performs a balance invariant check for native ETH: `if (address(this).balance != expectedBalance) revert BalanceInvariantCheckFailed(...)`.
    *   Calls `_deposit(IERC20Metadata(rootWETHToken), receiver, amount)`. Note: `rootWETHToken` is passed, but it's understood to represent native ETH on the child chain due to pre-mapping.
*   **`_depositERC20(IERC20Metadata rootToken, address receiver, uint256 amount)` (private)**:
    *   **`expectedBalance` Calculation:** `uint256 expectedBalance = rootToken.balanceOf(address(this)) + amount;`
    *   Calls `_deposit(rootToken, receiver, amount)`.
    *   Performs a balance invariant check for the specific `rootToken`: `if (rootToken.balanceOf(address(this)) != expectedBalance) revert BalanceInvariantCheckFailed(...)`.
*   **`_deposit(IERC20Metadata rootToken, address receiver, uint256 amount)` (private, core logic)**:
    *   `nonReentrant` and `whenNotPaused` modifiers.
    *   `wontIMXOverflow(address(rootToken), amount)` modifier (see below).
    *   Validates `receiver` and `rootToken` are not zero addresses, `amount > 0`, and `msg.value > 0` (for gas).
    *   Reverts with `NotMapped` if `rootTokenToChildToken[address(rootToken)]` is zero (unless it's ETH, WETH, or IMX which are pre-mapped).
    *   Determines `payloadToken`: if `rootToken` is `rootWETHToken`, `payloadToken` becomes `NATIVE_ETH`; otherwise, it's `address(rootToken)`.
    *   Encodes `payload` with `DEPOSIT_SIG`, `payloadToken`, `msg.sender` (depositor), `receiver`, and `amount`.
    *   Calculates `feeAmount` for `sendMessage`: if depositing native ETH, it's `msg.value - amount`; otherwise, it's `msg.value`.
    *   Sends the message via `rootBridgeAdaptor.sendMessage{value: feeAmount}(payload, msg.sender)`.
    *   Calls `_transferTokensAndEmitEvent` to handle token transfer (if not ETH/WETH) and event emission.
*   **`_transferTokensAndEmitEvent(address rootToken, address receiver, uint256 amount)` (private)**:
    *   If `rootToken == NATIVE_ETH`, emits `NativeEthDeposit`.
    *   Else if `rootToken == rootWETHToken`, emits `WETHDeposit`.
    *   Else if `rootToken == rootIMXToken`, emits `IMXDeposit` and transfers IMX from `msg.sender` to the bridge.
    *   Else (generic ERC20), emits `ChildChainERC20Deposit` and transfers the ERC20 from `msg.sender` to the bridge.

#### Withdrawal Handling (Message from Child Chain)

*   **`onMessageReceive(bytes calldata data) external override whenNotPaused onlyBridgeAdaptor`**:
    *   Callable only by the registered `rootBridgeAdaptor`.
    *   Validates `data.length > 32`.
    *   Checks if the first 32 bytes of `data` (the signature) match `WITHDRAW_SIG`.
    *   If it's a withdraw message, calls `_withdraw(data[32:])` with the rest of the payload.
    *   Reverts with `InvalidData` if the signature is unsupported or data is too short.
*   **`_withdraw(bytes memory data) internal virtual`**:
    *   Decodes and validates the withdrawal data using `_decodeAndValidateWithdrawal`.
    *   Calls `_executeTransfer` to perform the actual asset transfer.
*   **`_decodeAndValidateWithdrawal(bytes memory data)` (internal view)**:
    *   Decodes `data` into `rootToken`, `withdrawer`, `receiver`, and `amount`.
    *   Reverts if `rootToken` is zero.
    *   Determines `childToken` for event emission:
        *   If `rootToken == rootIMXToken`, `childToken = NATIVE_IMX`.
        *   If `rootToken == NATIVE_ETH`, `childToken = childETHToken`.
        *   Otherwise, looks up `childToken = rootTokenToChildToken[rootToken]`. Reverts with `NotMapped` if not found.
    *   Returns all decoded and derived values.
*   **`_executeTransfer(address rootToken, address childToken, address withdrawer, address receiver, uint256 amount)` (internal `whenNotPaused`)**:
    *   If `rootToken == NATIVE_ETH`, transfers native ETH to `receiver` using `Address.sendValue(payable(receiver), amount)` and emits `RootChainETHWithdraw`.
    *   Else (ERC20, including IMX), transfers the `rootToken` to `receiver` using `safeTransfer` and emits `RootChainERC20Withdraw`.

#### Administrative Functions

*   **`updateRootBridgeAdaptor(address newRootBridgeAdaptor) external onlyRole(ADAPTOR_MANAGER_ROLE)`**:
    *   Allows an account with `ADAPTOR_MANAGER_ROLE` to change the `rootBridgeAdaptor` address.
    *   Reverts if `newRootBridgeAdaptor` is the zero address.
    *   Emits `RootBridgeAdaptorUpdated`.
*   **`updateImxCumulativeDepositLimit(uint256 newImxCumulativeDepositLimit) external onlyRole(VARIABLE_MANAGER_ROLE)`**:
    *   Allows an account with `VARIABLE_MANAGER_ROLE` to change `imxCumulativeDepositLimit`.
    *   **Risk Mitigation:** If `newImxCumulativeDepositLimit` is not `UNLIMITED_DEPOSIT` (0), it checks that `newImxCumulativeDepositLimit >= IERC20Metadata(rootIMXToken).balanceOf(address(this))`. This prevents setting the limit lower than the amount of IMX already held by the bridge, which could trap funds. Reverts with `ImxDepositLimitTooLow` if this check fails.
    *   Emits `NewImxDepositLimit`.
*   **`grantVariableManagerRole(address account)` / `revokeVariableManagerRole(address account)`**:
    *   Standard role management functions restricted to `DEFAULT_ADMIN_ROLE`.

#### Other

*   **`receive() external payable`**:
    *   Fallback function to receive native ETH.
    *   Crucially used during WETH deposits: after the bridge calls `IWETH(rootWETHToken).withdraw(amount)`, the WETH contract sends native ETH back to the bridge, which is caught by this function.
    *   Reverts with `NonWrappedNativeTransfer` if `msg.sender != rootWETHToken` to prevent accidental ETH transfers directly to the bridge outside the WETH unwrapping flow.
*   **`_getTokenDetails(IERC20Metadata token)` (private view)**:
    *   Helper to fetch `name`, `symbol`, and `decimals` from an ERC20 token.
    *   Uses `try...catch` for each call. If any of these fail (e.g., token doesn't implement them), it reverts with `TokenNotSupported`.

### Token Handling Summary

*   **Generic ERC20s:**
    *   Must be `mapToken`'d first. Child token address is deterministically predicted.
    *   Deposits: User's ERC20s are transferred to the bridge. A message is sent to mint corresponding tokens on the child chain.
    *   Withdrawals: Bridge releases locked ERC20s based on message from child chain.
*   **Native ETH (NATIVE_ETH = `address(0xeee)`):**
    *   Pre-mapped during `initialize` to `childETHToken` (predicted address for wrapped ETH on child chain) or `NATIVE_ETH` conceptually if child chain uses native ETH.
    *   Deposits: User sends ETH (via `msg.value`). Bridge holds ETH. Message sent for child chain.
    *   Withdrawals: Bridge sends ETH to user.
*   **Wrapped ETH (WETH - `rootWETHToken`):**
    *   Pre-mapped during `initialize` conceptually to native ETH on the child chain (`childETHToken` or `NATIVE_ETH`).
    *   Deposits: User's WETH is transferred to the bridge, then immediately unwrapped to native ETH (which the bridge holds). Message sent for child chain (representing native ETH deposit). The `receive()` external payable function is vital here.
    *   Withdrawals: Processed as native ETH withdrawal on L1 because the bridge holds ETH, not WETH.
*   **IMX (`rootIMXToken`):**
    *   Pre-mapped during `initialize` to `rootIMXToken` itself (or `NATIVE_IMX = address(0xfff)` conceptually, as IMX is the native gas token on the child chain).
    *   Deposits: Handled like a generic ERC20 (IMX transferred to bridge), but subject to `imxCumulativeDepositLimit` via the `wontIMXOverflow` modifier.
    *   Withdrawals: Handled like a generic ERC20 (IMX transferred from bridge).

### Modifiers

*   **`onlyBridgeAdaptor()`**: Ensures `msg.sender == address(rootBridgeAdaptor)`. Used for `onMessageReceive`.
*   **`wontIMXOverflow(address rootToken, uint256 amount)`**:
    *   Checks if the current deposit involves `rootIMXToken`.
    *   If it is IMX and `imxCumulativeDepositLimit` is not `UNLIMITED_DEPOSIT` (0):
        *   It verifies that the current IMX balance of the bridge plus the `amount` being deposited does not exceed `imxCumulativeDepositLimit`.
        *   `if (IERC20Metadata(imxToken).balanceOf(address(this)) + amount > depositLimit) revert ImxDepositLimitExceeded();`
    *   This modifier is crucial for managing the total exposure to IMX locked in the bridge.

### Event Emissions

The contract emits events for all significant actions, crucial for off-chain monitoring and user interface updates:

*   `RootBridgeAdaptorUpdated(address oldRootBridgeAdaptor, address newRootBridgeAdaptor)`
*   `NewImxDepositLimit(uint256 oldImxDepositLimit, uint256 newImxDepositLimit)`
*   `L1TokenMapped(address indexed rootToken, address indexed childToken)`
*   `ChildChainERC20Deposit(address indexed rootToken, address indexed childToken, address depositor, address indexed receiver, uint256 amount)`
*   `IMXDeposit(address indexed rootToken, address depositor, address indexed receiver, uint256 amount)`
*   `WETHDeposit(address indexed rootToken, address indexed childToken, address depositor, address indexed receiver, uint256 amount)`
*   `NativeEthDeposit(address indexed rootToken, address indexed childToken, address depositor, address indexed receiver, uint256 amount)`
*   `RootChainERC20Withdraw(address indexed rootToken, address indexed childToken, address withdrawer, address indexed receiver, uint256 amount)`
*   `RootChainETHWithdraw(address indexed rootToken, address indexed childToken, address withdrawer, address indexed receiver, uint256 amount)`

### Custom Errors

The contract defines several custom errors for clarity and gas efficiency:

*   `InsufficientValue()`: `msg.value` less than required amount for ETH deposit.
*   `ZeroAmount()`: Attempting to bridge zero tokens/ETH.
*   `ZeroAddress()`: Critical address parameter is zero where it shouldn't be.
*   `NoGas()`: `msg.value` is zero when a fee is required for messaging.
*   `InvalidChildChain()`: (Not used in this specific contract but defined in interface)
*   `AlreadyMapped()`: Token is already mapped.
*   `NotMapped()`: Token is not mapped when it's expected to be.
*   `CantMapIMX()`, `CantMapETH()`, `CantMapWETH()`: Attempting to map tokens that are handled specially or pre-mapped.
*   `BalanceInvariantCheckFailed(uint256 actualBalance, uint256 expectedBalance)`: Sanity check failed after a deposit, indicating unexpected balance change.
*   `InvalidData(string reason)`: Payload in `onMessageReceive` is malformed.
*   `NotBridgeAdaptor()`: Caller of `onMessageReceive` is not the registered adaptor.
*   `ImxDepositLimitExceeded()`: IMX deposit would exceed the global limit.
*   `ImxDepositLimitTooLow()`: Attempt to set IMX limit below current bridge balance.
*   `NonWrappedNativeTransfer()`: Direct ETH transfer to bridge not from WETH contract.
*   `UnauthorizedInitializer()`: `initialize()` called by an unauthorized address.
*   `TokenNotSupported()`: ERC20 token lacks `name()`, `symbol()`, or `decimals()` functions, required for mapping.

### Calculations Focus

*   **`_depositETH` `expectedBalance`:** `address(this).balance - (msg.value - amount)`
    *   This correctly calculates the expected native ETH balance *after* the deposit operation, accounting for the fact that `msg.value` (total ETH sent with the call) might be greater than `amount` (the ETH to be bridged). The difference (`msg.value - amount`) is assumed to be for the message fee, which will be consumed by the `rootBridgeAdaptor.sendMessage` call. So, the bridge's balance should increase by exactly `amount`.
*   **`_depositWrappedETH` `expectedBalance`:** `address(this).balance + amount`
    *   After WETH is received and unwrapped by calling `IWETH.withdraw(amount)`, the bridge's native ETH balance should increase by `amount`.
*   **`_depositERC20` `expectedBalance`:** `rootToken.balanceOf(address(this)) + amount`
    *   The bridge's balance of the specific `rootToken` should increase by `amount` after the `safeTransferFrom` in `_deposit` (via `_transferTokensAndEmitEvent`).
*   **`_mapToken` `childToken` address prediction:** `Clones.predictDeterministicAddress(childTokenTemplate, keccak256(abi.encodePacked(rootToken)), childERC20Bridge)`
    *   This uses OpenZeppelin's `Clones` library to predict the address where a new contract (the child token) will be deployed using `CREATE2`. The salt combines the `rootToken` address (ensuring uniqueness per L1 token) and the `childERC20Bridge` address (the deployer on L2). `childTokenTemplate` is the implementation contract that will be cloned.
*   **`wontIMXOverflow` modifier:** `IERC20Metadata(imxToken).balanceOf(address(this)) + amount > depositLimit`
    *   This check ensures that the current total IMX held by the bridge plus the new `amount` being deposited does not surpass the `imxCumulativeDepositLimit`.

### Potential Risks and Mitigations

*   **Reentrancy:**
    *   Mitigated by the `nonReentrant` modifier (from OpenZeppelin's `ReentrancyGuardUpgradeable`) on the core `_deposit` function.
    *   The `whenNotPaused` modifier also prevents state changes during a paused state.
*   **Front-running of `initialize`:**
    *   The contract uses a two-step initialization pattern. The `constructor` sets an `initializerAddress`. Only this address can call the actual `initialize` function. This makes it harder for an attacker to front-run the initialization with their own parameters.
*   **Bridging Non-Standard ERC20s:**
    *   Comments explicitly warn: "There is undefined behaviour for bridging non-standard ERC20 tokens (e.g. rebasing tokens)." Such tokens might behave unpredictably when transferred (e.g., balance changes on transfer, fees on transfer), potentially breaking the bridge's balance invariants or leading to loss of funds. Users should exercise caution.
*   **IMX Deposit to Non-IMX-Receptive Contracts on Child Chain:**
    *   Comments warn: "When depositing IMX (L1 -> L2) it's crucial to make sure that the receiving address on the child chain, if it's a contract, has a receive or fallback function that allows it to accept native IMX on the child chain." If the L2 recipient contract cannot receive native IMX (the gas token on Immutable zkEVM), the L2 transaction might revert, potentially locking funds. This is more of a user/integrator caution.
*   **`updateImxCumulativeDepositLimit` Risk:**
    *   The function itself includes a check: `if (newImxCumulativeDepositLimit != UNLIMITED_DEPOSIT && newImxCumulativeDepositLimit < IERC20Metadata(rootIMXToken).balanceOf(address(this))) { revert ImxDepositLimitTooLow(); }`. This prevents the `VARIABLE_MANAGER_ROLE` from setting a new limit that is lower than the IMX already secured by the bridge, which would otherwise make those existing funds impossible to count towards the limit correctly or potentially create issues with accounting.

### Mermaid Diagram: Functional Interactions

```mermaid
graph TD
    subgraph UserInteractions
        User[User]
    end

    subgraph RootChainBridgeSystem [RootERC20Bridge.sol]
        RB_Init["initialize()"]
        RB_Map["mapToken()"]
        RB_Deposit["deposit() / depositTo()"]
        RB_DepositETH["depositETH() / depositToETH()"]
        RB_OnMessage["onMessageReceive() (Withdrawal)"]
        RB_UpdateAdaptor["updateRootBridgeAdaptor()"]
        RB_UpdateLimit["updateImxCumulativeDepositLimit()"]

        subgraph CoreLogic
            Core_Deposit["_deposit()"]
            Core_Withdraw["_withdraw() / _executeTransfer()"]
            Core_Map["_mapToken() (predicts ChildToken)"]
        end
    end

    subgraph ExternalContracts
        ERC20[ERC20 Token Contract]
        WETH[IWETH Contract]
        Adaptor[IRootBridgeAdaptor]
        ChildTokenTemplate[Child Token Template (for CREATE2)]
        BridgeRolesC[BridgeRoles.sol]
    end

    User -- "Call deposit()" --> RB_Deposit
    User -- "Call depositETH()" --> RB_DepositETH
    User -- "Call mapToken()" --> RB_Map

    RB_Deposit -- Calls --> Core_Deposit
    RB_DepositETH -- Calls --> Core_Deposit

    Core_Deposit -- "safeTransferFrom()" --> ERC20
    Core_Deposit -- "IWETH.withdraw()" --> WETH
    WETH -- "Sends ETH back" --> RootChainBridgeSystem % Representing receive()
    Core_Deposit -- "sendMessage()" --> Adaptor

    RB_Map -- Calls --> Core_Map
    Core_Map -- "predictDeterministicAddress()" --> ChildTokenTemplate
    Core_Map -- "sendMessage()" --> Adaptor

    Adaptor -- "Calls onMessageReceive()" --> RB_OnMessage
    RB_OnMessage -- Calls --> Core_Withdraw
    Core_Withdraw -- "safeTransfer() / sendValue()" --> ERC20
    Core_Withdraw -- "safeTransfer() / sendValue()" --> User % Direct ETH transfer to user

    RootChainBridgeSystem -- Inherits --> BridgeRolesC
    RB_UpdateAdaptor -- Modifies --> Adaptor
    RB_UpdateLimit -- Modifies state --> RootChainBridgeSystem

    style User fill:#lightgrey,stroke:#333,stroke-width:2px
    style ERC20 fill:#lightgreen,stroke:#333,stroke-width:2px
    style WETH fill:#lightyellow,stroke:#333,stroke-width:2px
    style Adaptor fill:#thistle,stroke:#333,stroke-width:2px
    style ChildTokenTemplate fill:#add8e6,stroke:#333,stroke-width:2px
    style RootChainBridgeSystem fill:#lightblue,stroke:#333,stroke-width:2px
```

This breakdown provides a detailed look into the `RootERC20Bridge.sol` contract, covering its main functionalities, interactions, and important considerations.
