## Analysis of Transaction Rollback on `sendMessage` Revert

This document analyzes the behavior of `RootERC20Bridge.sol`'s deposit and token mapping functions when the external call to `rootBridgeAdaptor.sendMessage` reverts. The core principle in Solidity is that if an external call reverts without being caught by a `try/catch` block, the entire calling transaction reverts, rolling back all state changes made within that transaction.

The `RootERC20Bridge.sol` contract does **not** use `try/catch` around its `rootBridgeAdaptor.sendMessage` calls. Therefore, any revert from `sendMessage` will propagate up and cause the entire calling function (and thus the user's transaction) to revert.

### 1. Review of `_deposit` (Internal Common Function)

*   **Code Snippet (from `_deposit`):**
    ```solidity
    // ... previous checks ...
    rootBridgeAdaptor.sendMessage{value: feeAmount}(payload, msg.sender);
    _transferTokensAndEmitEvent(address(rootToken), receiver, amount);
    ```
*   **Analysis:** `_deposit` is the central internal function that calls `sendMessage`. If `sendMessage` reverts, `_deposit` will revert.

### 2. ETH Deposit (`depositETH` -> `_depositETH` -> `_deposit`)

*   **Flow:**
    1.  User calls `depositETH(amount)` or `depositToETH(receiver, amount)`, sending ETH (`msg.value`).
    2.  `_depositETH` is called. It checks `msg.value >= amount`.
    3.  `_depositETH` calls `_deposit(IERC20Metadata(NATIVE_ETH), receiver, amount)`.
*   **If `sendMessage` (within `_deposit`) reverts:**
    *   The `_deposit` call reverts.
    *   This causes `_depositETH` to revert.
    *   This causes the public `depositETH` (or `depositToETH`) function to revert.
*   **Conclusion:** The entire transaction, including the initial transfer of `msg.value` ETH to the bridge contract, is rolled back. **User's ETH is not stuck in the bridge.**

### 3. Generic ERC20 Token Deposit (`deposit` -> `_depositToken` -> `_depositERC20` -> `_deposit`)

*   **Flow:**
    1.  User calls `deposit(rootToken, amount)` or `depositTo(rootToken, receiver, amount)`.
    2.  `_depositToken` calls `_depositERC20(rootToken, receiver, amount)`.
    3.  `_depositERC20` calls `_deposit(rootToken, receiver, amount)`.
    4.  Inside `_deposit`, the call to `rootBridgeAdaptor.sendMessage` occurs *before* the call to `_transferTokensAndEmitEvent`.
    5.  `_transferTokensAndEmitEvent` is where `IERC20Metadata(rootToken).safeTransferFrom(msg.sender, address(this), amount)` happens for generic ERC20s.
*   **If `sendMessage` (within `_deposit`) reverts:**
    *   The `_transferTokensAndEmitEvent` function (and thus the `safeTransferFrom`) is **never called**.
    *   The `_deposit` call reverts.
    *   This causes `_depositERC20`, then `_depositToken`, then the public `deposit` (or `depositTo`) function to revert.
*   **Conclusion:** Since the actual token transfer (`safeTransferFrom`) to the bridge occurs *after* the `sendMessage` call, if `sendMessage` reverts, the tokens are never transferred from the user to the bridge. **User's ERC20 tokens are not stuck in the bridge.**

### 4. WETH Deposit (`deposit` -> `_depositToken` -> `_depositWrappedETH` -> `_deposit`)

*   **Flow:**
    1.  User calls `deposit(wethTokenAddress, amount)` or `depositTo(wethTokenAddress, receiver, amount)`.
    2.  `_depositToken` calls `_depositWrappedETH(receiver, amount)`.
    3.  `_depositWrappedETH` executes the following sequence *before* calling `_deposit`:
        a.  `erc20WETH.safeTransferFrom(msg.sender, address(this), amount);` (WETH is transferred to the bridge).
        b.  `IWETH(rootWETHToken).withdraw(amount);` (WETH is unwrapped; bridge receives native ETH).
    4.  `_depositWrappedETH` then calls `_deposit(IERC20Metadata(rootWETHToken), receiver, amount)`.
*   **If `sendMessage` (within the nested `_deposit` call) reverts:**
    *   The nested `_deposit` call reverts.
    *   This causes `_depositWrappedETH` to revert.
    *   This causes `_depositToken`, then the public `deposit` (or `depositTo`) function to revert.
*   **Conclusion:** Because the `safeTransferFrom` of WETH and the `IWETH.withdraw` (unwrapping) occur *before* the `sendMessage` call (which is nested inside `_deposit`), these state changes (bridge receiving WETH, then bridge's ETH balance increasing and WETH balance decreasing due to unwrap) are part of the overall transaction. If `sendMessage` causes a revert that propagates all the way up, the entire transaction is rolled back. This means the WETH transfer from the user to the bridge is reverted, and the unwrapping effects are also reverted. **User's WETH is not stuck in the bridge, nor is the unwrapped ETH from their WETH stuck.**

### 5. IMX Deposit (Handled like Generic ERC20)

*   **Flow:** IMX deposits are handled by the `_depositERC20` path.
*   **Conclusion:** Similar to Generic ERC20 Token Deposits, the `safeTransferFrom` of IMX tokens from the user to the bridge occurs in `_transferTokensAndEmitEvent`, which is *after* the `sendMessage` call in `_deposit`. If `sendMessage` reverts, the IMX transfer does not happen. **User's IMX tokens are not stuck in the bridge.**

### 6. Token Mapping (`mapToken` -> `_mapToken`)

*   **Flow:**
    1.  User calls `mapToken(rootToken)`.
    2.  `_mapToken` is called.
    3.  The state change `rootTokenToChildToken[address(rootToken)] = childToken;` occurs *before* the call to `rootBridgeAdaptor.sendMessage(...)`.
*   **If `sendMessage` (within `_mapToken`) reverts:**
    *   The `_mapToken` function will revert.
    *   This causes the public `mapToken` function to revert.
*   **Conclusion:** The entire transaction, including the state change to `rootTokenToChildToken`, is rolled back. The mapping is not persisted if `sendMessage` fails. **No incorrect state is committed.**

---

**Overall Summary:**

The `RootERC20Bridge.sol` contract correctly handles reverts from the `rootBridgeAdaptor.sendMessage` call. Due to standard Solidity transaction atomicity and the lack of `try/catch` blocks around `sendMessage` calls:

*   For **ETH deposits**, the ETH transfer to the bridge is part of the transaction that reverts, so the user's ETH is returned.
*   For **generic ERC20 and IMX deposits**, the actual transfer of tokens from the user to the bridge (`safeTransferFrom`) is strategically placed *after* the `sendMessage` call within the `_deposit` function's logic. If `sendMessage` fails, the token transfer does not occur.
*   For **WETH deposits**, the WETH transfer to the bridge and its subsequent unwrapping into ETH occur *before* the `sendMessage` call (which is nested in a subsequent `_deposit` call). If `sendMessage` fails, the entire transaction, including these initial WETH operations, is reverted.
*   For **token mapping**, state changes to mappings occur before `sendMessage`. If `sendMessage` fails, these state changes are rolled back.

In all analyzed scenarios, if `sendMessage` reverts, the transaction is rolled back appropriately, preventing user funds or state from being incorrectly committed or stuck in the bridge due to this specific L1 external call failure. The primary risk of stuck funds remains if `sendMessage` *succeeds* on L1 but the corresponding L2 action fails, as discussed in previous analyses.
