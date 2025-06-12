## L1 State and Recovery Options if L2 Fails Post-`sendMessage`

This analysis examines the state of funds on L1 if a `sendMessage` call (for deposits or token mapping) from `RootERC20Bridge.sol` to the `IRootBridgeAdaptor` completes successfully, but the corresponding operation on L2 subsequently fails. It focuses on whether any mechanisms *within the L1 bridge contracts* (`RootERC20Bridge.sol`, `RootERC20BridgeFlowRate.sol`) allow for user recovery or admin intervention to return these L1 funds.

### 1. L1 State Post-Successful `sendMessage`

*   **Funds Custody:**
    *   **Generic ERC20s/IMX Deposits:**
        *   In `RootERC20Bridge._deposit`, the call to `_transferTokensAndEmitEvent` (which executes `IERC20Metadata(rootToken).safeTransferFrom(msg.sender, address(this), amount)`) occurs *after* the `rootBridgeAdaptor.sendMessage` call.
        *   **Therefore, if `sendMessage` has completed successfully, the ERC20/IMX tokens have also been successfully transferred from the user to the bridge contract.**
    *   **WETH Deposits:**
        *   In `RootERC20Bridge._depositWrappedETH`, the WETH transfer from the user to the bridge (`erc20WETH.safeTransferFrom`) and the unwrapping (`IWETH(rootWETHToken).withdraw(amount)`) occur *before* the internal call to `_deposit`, which then calls `sendMessage`.
        *   **Therefore, if `sendMessage` (called from within `_deposit`) completes successfully, the user's WETH has already been converted to native ETH, and this native ETH is held by the bridge contract.**
    *   **Native ETH Deposits:**
        *   In `RootERC20Bridge._depositETH`, the user's ETH (`msg.value`) is transferred to the bridge as part of the function call. The internal call to `_deposit` (which calls `sendMessage`) happens subsequently.
        *   **Therefore, if `sendMessage` completes successfully, the user's native ETH (less any fee amount used for `sendMessage`) is held by the bridge contract.**
    *   **Token Mapping (`_mapToken`):**
        *   No user funds are held by the bridge during `mapToken` other than the `msg.value` provided for the `sendMessage` fee. The `rootTokenToChildToken` mapping is updated. If L2 fails to recognize or act on this mapping, the L1 state change persists but might not be usable. This doesn't directly involve stuck user *principal* assets in the bridge.

    **Conclusion on Funds Custody:** In all deposit scenarios (ERC20, IMX, WETH, native ETH), after a successful `sendMessage` call from the bridge, the L1 assets (or the native ETH derived from WETH) intended for bridging are held by the `RootERC20Bridge` contract.

*   **No L1 "Pending Deposit" State Tracking L2 Outcome:**
    *   A review of the storage variables in `RootERC20Bridge.sol` and `RootERC20BridgeFlowRate.sol` confirms that there are no specific flags, structs, or mappings designed to track the status (pending, success, fail) of an L2 operation corresponding to an L1 deposit for which `sendMessage` has succeeded.
    *   The L1 bridge operates in a "fire-and-forget" manner for deposit messages once `sendMessage` is successful. It assumes the message will be handled correctly by the adaptor and L2 components.

### 2. L1 User-Initiated Recovery Functions

*   **Review of Public/External Functions:**
    *   `RootERC20Bridge.sol`: `mapToken`, `depositETH`, `depositToETH`, `deposit`, `depositTo`, and various `onlyRole` admin functions.
    *   `RootERC20BridgeFlowRate.sol`: Adds `finaliseQueuedWithdrawal`, `finaliseQueuedWithdrawalsAggregated`, and `onlyRole` admin functions.
*   **Analysis:**
    *   None of the user-callable deposit functions (`depositETH`, `deposit`, etc.) have a corresponding "cancel" or "refund" mechanism that a user could trigger if their L2 transaction fails after `sendMessage` succeeded on L1.
    *   The withdrawal-related functions (`onMessageReceive`, `finaliseQueuedWithdrawal`, etc.) are all designed for L2-to-L1 transfers (processing messages originating from L2, typically representing an L2 burn or release). They cannot be used by an L1 user to recall an L1-to-L2 deposit that failed on L2.
*   **Conclusion:** There are **no built-in, user-callable functions** within the provided L1 bridge contracts that allow a user to directly reclaim their L1-deposited funds if the corresponding L2 operation fails after `sendMessage` has succeeded.

### 3. L1 Admin-Initiated Recovery Functions (Specific to L2 Deposit Failures)

*   **Review of Admin Functions:**
    *   Standard admin functions include role management, pausing the contract, updating the bridge adaptor, updating IMX limits, and managing flow rate parameters (delay, thresholds, queue activation).
*   **Analysis:**
    *   None of these administrative functions are designed as a direct, specific mechanism to refund individual users for L1 deposits that failed on L2 post-`sendMessage`. There's no function like `adminRefundFailedDeposit(originalTxHash, recipient, token, amount)`.
*   **Hypothetical Admin Intervention (General Powers):**
    *   The most plausible way for admins to return funds in such a scenario, using existing contract capabilities, would be an indirect and centralized process:
        1.  **Acknowledge Off-Chain:** The user would need to report the issue to the bridge operators with proof of their L1 deposit and evidence of L2 failure.
        2.  **Admin Action via `rootBridgeAdaptor`:** An account with `ADAPTOR_MANAGER_ROLE` could temporarily change the `rootBridgeAdaptor` to a specialized "Rescue Adaptor" contract.
        3.  This Rescue Adaptor would need to be designed to allow an authorized party (e.g., a multi-sig controlled by bridge admins) to instruct it to generate specific withdrawal messages.
        4.  The Rescue Adaptor would then call `onMessageReceive` on the `RootERC20Bridge`, providing a crafted payload that mimics a legitimate L2-to-L1 withdrawal message for the stuck funds, with the original depositor as the `receiver`.
        5.  The `RootERC20Bridge` would process this message via `_withdraw` and `_executeTransfer`, releasing the L1 funds to the user.
        6.  The `rootBridgeAdaptor` would then need to be reset to the original, operational adaptor.
    *   This is **not a built-in recovery feature** for this specific problem but rather an application of the very powerful (and critical) `ADAPTOR_MANAGER_ROLE`. It requires significant trust in the bridge operators, off-chain coordination, and the deployment/use of a special-purpose adaptor.
*   **Conclusion:** There are **no specific, built-in admin functions** in the L1 bridge contracts for targeted recovery of funds stuck due to L2 deposit failures post-`sendMessage`. Recovery would rely on general, high-privilege administrative capabilities to effectively "trick" the bridge into processing a withdrawal, which is a manual, centralized, and complex process.

### 4. Conclusion on L1 State and Recovery (if L2 Fails Post-`sendMessage`)

*   If `sendMessage` on L1 completes successfully for a deposit:
    *   The L1 assets (or ETH derived from them) are custodied by the `RootERC20Bridge` contract.
    *   The L1 bridge contracts consider their part of the L1-to-L2 deposit process complete and do not track the L2 outcome.
*   From the perspective of the **L1 smart contract code alone**:
    *   There are **no user-initiated functions** to recall or refund these assets.
    *   There are **no specific admin-initiated functions** designed for this type of refund.
*   Therefore, if L2 operations fail after `sendMessage` has succeeded on L1, the user's L1 assets are **stuck in the `RootERC20Bridge` contract from the perspective of direct, code-defined recovery mechanisms available to the user.**
*   Recovery would depend on:
    1.  The L2 system eventually resolving its issue and successfully initiating a standard L2-to-L1 withdrawal message for the user.
    2.  Off-chain manual intervention by bridge operators, likely using high-privilege roles (like `ADAPTOR_MANAGER_ROLE`) in a non-standard way to authorize a release of L1 funds. This is a trust-based, centralized recovery path, not a decentralized or user-invokable one defined in the L1 contracts.

This highlights a critical dependency on the reliability of the chosen `IRootBridgeAdaptor` and the L2 bridge components to ensure end-to-end success of bridging operations.
