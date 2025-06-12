# Draft Security Audit Report - Immutable zkEVM Bridge Contracts

## 1. High-Level Summary

**Audit Scope:** This report covers the security analysis of the following Solidity smart contracts related to the Immutable zkEVM bridge:
*   `RootERC20Bridge.sol`: The base root chain bridge for ERC20 and ETH transfers.
*   `BridgeRoles.sol`: Abstract contract for access control.
*   `FlowRateDetection.sol`: Abstract contract for detecting large token flows.
*   `FlowRateWithdrawalQueue.sol`: Abstract contract for managing delayed withdrawals.
*   `RootERC20BridgeFlowRate.sol`: An enhanced version of the bridge incorporating flow rate detection and withdrawal queue functionalities.
*   Associated Interface contracts (`IRootBridgeAdaptor.sol`, `IRootERC20Bridge.sol`, `IWETH.sol`, etc.) were also reviewed for context.

**Key Objective:** To identify potential vulnerabilities, design weaknesses, and areas for improvement in the provided smart contracts.

**Most Critical Findings:**
1.  **Reentrancy Vulnerability in Native ETH Withdrawals:** A reentrancy path exists when processing non-queued native ETH withdrawals, potentially allowing a malicious receiver to re-enter the bridge contract before the initial withdrawal transaction fully completes. This vulnerability persists in `RootERC20BridgeFlowRate.sol` as it inherits the vulnerable logic from `RootERC20Bridge.sol` without adding specific mitigation for this path.
2.  **Critical Reliance on Bridge Adaptor Integrity:** The security of funds held by the bridge heavily depends on the trustworthiness and security of the designated `IRootBridgeAdaptor` implementation. A compromised or malicious adaptor can forge messages to drain any and all assets from the bridge.
3.  **Centralized Risks via Administrative Roles:** Several administrative roles (`ADAPTOR_MANAGER_ROLE`, `RATE_CONTROL_ROLE`, `DEFAULT_ADMIN_ROLE`, etc.) have powerful capabilities. If compromised, these roles could lead to theft of funds, permanent or temporary Denial of Service (DoS), or render security mechanisms ineffective.

Overall, while the contracts incorporate several good security practices, the identified critical vulnerabilities, particularly reentrancy and adaptor trust, require immediate attention. Other findings relate to potential DoS vectors through admin role misuse, griefing attacks, and known MEV patterns.

---

## 2. Vulnerability Details

### Critical Severity

#### CRIT-01: Compromised Bridge Adaptor Can Drain All Funds

*   **Severity:** Critical
*   **Contract(s) & Function(s) Involved:** `RootERC20Bridge.onMessageReceive`, `RootERC20Bridge._withdraw`, `RootERC20Bridge._executeTransfer`, `RootERC20Bridge.rootBridgeAdaptor` (storage), `RootERC20Bridge.updateRootBridgeAdaptor`.
*   **Description:** The `onMessageReceive` function, which processes withdrawal messages from L2, is protected by the `onlyBridgeAdaptor` modifier. If the account designated as the `rootBridgeAdaptor` is compromised, or if an attacker gains control of the `ADAPTOR_MANAGER_ROLE` and changes the `rootBridgeAdaptor` to a malicious contract, the attacker can forge arbitrary withdrawal messages.
*   **Exploitation:** The malicious adaptor can call `onMessageReceive` with crafted `data` payload to specify any `rootToken`, `receiver` (attacker's address), and `amount` (up to the bridge's entire balance for that token).
*   **Impact:** Complete theft of all bridgeable assets (ETH and all ERC20 tokens) held by the `RootERC20Bridge` or `RootERC20BridgeFlowRate` contract.
*   **Recommendation:**
    *   The security of the `rootBridgeAdaptor` account/contract is paramount and outside the scope of this code audit but must be ensured through operational security, multi-signature controls, and potentially hardware security.
    *   Strictly limit and secure the `ADAPTOR_MANAGER_ROLE`. Consider making this role a high-security multi-sig or even immutable after initial setup if the adaptor is deemed sufficiently audited and stable.
    *   Implement robust monitoring on L2 to detect anomalous L2 bridge events that might generate malicious messages.
    *   Consider time-locks or governance votes for changing the `rootBridgeAdaptor`.
*   **Reference to Plan Step:** Cross-Contract (Step 3), Other Critical (Step 4).

#### CRIT-02: Administrative Roles Can Cause Fund Freezing or Unintended Behavior

*   **Severity:** Critical (for specific DoS scenarios)
*   **Contract(s) & Function(s) Involved:**
    *   `RootERC20BridgeFlowRate.setWithdrawalDelay` (via `FlowRateWithdrawalQueue._setWithdrawalDelay`)
    *   `RootERC20Bridge.updateRootBridgeAdaptor`
*   **Description:** Compromise of specific administrative roles can lead to scenarios where funds become permanently or for an extremely long time inaccessible, or critical security mechanisms are disabled.
    1.  **`RATE_CONTROL_ROLE`:** Can call `setWithdrawalDelay` to set an extremely large `withdrawalDelay` (e.g., years) in `FlowRateWithdrawalQueue`. This would effectively lock up any currently queued or future queued withdrawals for that duration.
    2.  **`ADAPTOR_MANAGER_ROLE`:** Can call `updateRootBridgeAdaptor` to set a non-functional or malicious adaptor. If set to a blackhole address (though `address(0)` is checked) or a non-operational contract, all bridge functions (deposits and withdrawals) would cease, effectively freezing funds in the bridge.
*   **Impact:**
    *   Effectively permanent freezing of queued funds (if delay is set to decades/centuries).
    *   Complete DoS of all bridge operations if a non-functional adaptor is set.
*   **Recommendation:**
    *   Implement sanity checks for `setWithdrawalDelay`, such as a maximum allowable delay that can be configured (e.g., 30-60 days). Changes beyond this could require a more privileged role or a time-locked governance process.
    *   Strengthen the security and operational procedures around the `RATE_CONTROL_ROLE` and `ADAPTOR_MANAGER_ROLE`. Use multi-signature wallets with trusted, diverse signers.
    *   Consider time-locks for critical changes like `updateRootBridgeAdaptor`.
*   **Reference to Plan Step:** Other Critical (Step 4), Flow Rate Part 1 (Step 2).

### High Severity

#### HIGH-01: Reentrancy in Native ETH Withdrawals (Non-Queued Path)

*   **Severity:** High
*   **Contract(s) & Function(s) Involved:** `RootERC20Bridge._executeTransfer` (specifically the `Address.sendValue` part), `RootERC20Bridge.onMessageReceive`, `RootERC20Bridge._withdraw`. This vulnerability persists in `RootERC20BridgeFlowRate.sol` for non-queued withdrawals.
*   **Description:** When native ETH is withdrawn, `_executeTransfer` uses `Address.sendValue(payable(receiver), amount)`. If the `receiver` is a malicious contract, its `receive()` or `fallback()` payable function can call back into the bridge contract. The call path `onMessageReceive` -> `_withdraw` -> `_executeTransfer` is not protected by a `nonReentrant` modifier in either `RootERC20Bridge` or `RootERC20BridgeFlowRate` (for the non-queued path).
*   **Exploitation:** A malicious contract, acting as the `receiver` of a native ETH withdrawal, could re-enter `onMessageReceive` (if the adaptor allows/relays such re-entrant messages from L2, or if the attacker can somehow bypass the adaptor check during re-entry, though less likely) or other public functions. If `onMessageReceive` could be re-entered with the same message parameters before the first withdrawal fully completes its state updates (though a specific exploit here is complex and depends on adaptor behavior), it might lead to issues. More plausibly, re-entering other unprotected public functions could manipulate state or attempt further actions.
*   **Impact:** Potential for draining more funds than authorized if reentrancy allows triggering additional withdrawals or manipulating state related to the withdrawal process. The exact impact depends on which functions can be profitably re-entered and the behavior of the bridge adaptor in handling re-entrant calls from L2.
*   **Recommendation:** Apply the `nonReentrant` modifier (from OpenZeppelin's `ReentrancyGuardUpgradeable`) to the `onMessageReceive` function in `RootERC20Bridge.sol` (which `RootERC20BridgeFlowRate.sol` inherits and uses). This would protect the entire external message handling flow, including immediate withdrawals.
*   **Reference to Plan Step:** RootERC20Bridge Part 2 (Step 1), RootERC20BridgeFlowRate Part 2 (Step 2).

#### HIGH-02: L1 Funds Potentially Stuck if L2 Execution Fails After `sendMessage`

*   **Severity:** High
*   **Contract(s) & Function(s) Involved:** `RootERC20Bridge._deposit`, `RootERC20Bridge._mapToken`, `IRootBridgeAdaptor.sendMessage`.
*   **Description:** During deposits or token mapping, the L1 `RootERC20Bridge` first locks/holds the L1 assets (or records mapping) and then calls `IRootBridgeAdaptor.sendMessage()`. If `sendMessage` succeeds on L1 (i.e., the adaptor accepts the message and doesn't revert), but the subsequent L2 transaction fails (e.g., L2 bridge runs out of gas, L2 logic error, invalid parameters for L2, malformed message by adaptor), the L1 assets remain locked in the `RootERC20Bridge`. There is no automated mechanism within the L1 bridge contracts to refund or unlock these assets if the L2 side fails post-L1-success.
*   **Impact:** User's funds are stuck on L1, with no corresponding assets or functionality enabled on L2. This can lead to permanent loss of funds for the user unless manual intervention by bridge operators is possible and undertaken.
*   **Recommendation:**
    *   This is a challenging cross-chain problem. Ideal solutions involve robust adaptors and L2 logic that can guarantee execution or provide verifiable proof of failure that L1 can act upon.
    *   Implement comprehensive off-chain monitoring and alerting for L1 deposits to track their corresponding L2 confirmations.
    *   Establish a clear operational procedure for manual refunds or retries in case of L2 failures, though this is centralized and complex.
    *   The `IRootBridgeAdaptor` should be designed to be as robust as possible, potentially with its own retry mechanisms or clear error reporting that could (in future designs) be relayed back.
*   **Reference to Plan Step:** Cross-Contract (Step 3).

#### HIGH-03: Misconfiguration of Flow Rate Parameters by Admin Can Disable Security or Cause DoS

*   **Severity:** High (as an admin-controlled risk)
*   **Contract(s) & Function(s) Involved:** `RootERC20BridgeFlowRate.setRateControlThreshold`, `RootERC20BridgeFlowRate.activateWithdrawalQueue`, `RootERC20BridgeFlowRate.deactivateWithdrawalQueue`.
*   **Description:** A compromised or negligent `RATE_CONTROL_ROLE` holder can set flow rate parameters (`capacity`, `refillRate`, `largeTransferThreshold`) or the global `withdrawalQueueActivated` flag in ways that either disable the flow rate security features or cause a denial of service for legitimate withdrawals.
    *   Setting very high `capacity`/`refillRate` and `largeTransferThreshold` makes flow limits ineffective.
    *   Setting very low `capacity`/`refillRate` or `largeTransferThreshold = 0` can cause most/all withdrawals to be queued.
    *   Permanently setting `withdrawalQueueActivated = true` forces all withdrawals to the queue.
*   **Impact:** Effective disabling of the flow rate based security, or denial of service for immediate withdrawals for specific tokens or all tokens.
*   **Recommendation:**
    *   Strictly control and secure the `RATE_CONTROL_ROLE` using multi-signature wallets and strong operational procedures.
    *   Implement off-chain monitoring for changes to these critical parameters and alert on configurations that are outside expected bounds.
    *   Consider adding sanity checks or limits within the `setRateControlThreshold` function (e.g., minimum sensible `capacity` if `largeTransferThreshold` is also very low, or relationships between parameters), though this can reduce flexibility.
    *   Regularly audit and review flow rate configurations.
*   **Reference to Plan Step:** Flow Rate Part 1 (Step 2), Other Critical (Step 4).

### Medium Severity

#### MED-01: Griefing Attack by Triggering Global Withdrawal Queue

*   **Severity:** Medium
*   **Contract(s) & Function(s) Involved:** `RootERC20BridgeFlowRate._withdraw`, `FlowRateDetection._updateFlowRateBucket`.
*   **Description:** As noted in contract comments, an attacker could make withdrawals on a token with poorly configured (e.g., very low value) flow rate bucket parameters. By emptying this specific token's bucket, they can trigger `withdrawalQueueActivated = true` globally. This forces all subsequent withdrawals for *any* token by *any* user into the withdrawal queue.
*   **Impact:** Disrupts normal bridge operations for all users by forcing their withdrawals to be delayed, even if their specific token and amount would not normally trigger any limits. This is a griefing attack that degrades user experience.
*   **Recommendation:**
    *   The `RATE_CONTROL_ROLE` must carefully configure flow rate parameters (`capacity`, `refillRate`) for *all* mapped tokens to be resilient against such cheap attacks. Thresholds should be set based on expected legitimate flows and require significant capital from an attacker to manipulate.
    *   Implement monitoring for frequent automatic activation of the withdrawal queue, which might indicate misconfigured parameters or an ongoing griefing attack.
*   **Reference to Plan Step:** Flow Rate Part 1 (Step 2), Other Critical (Step 4).

#### MED-02: Front-Running Deposits/Withdrawals for MEV

*   **Severity:** Medium
*   **Contract(s) & Function(s) Involved:** `RootERC20Bridge.deposit` (for IMX), `RootERC20BridgeFlowRate._withdraw`.
*   **Description:**
    1.  **IMX Deposit Limit:** If the `imxCumulativeDepositLimit` is close to being met, an attacker can front-run a legitimate user's IMX deposit to consume the remaining capacity, causing the user's transaction to fail.
    2.  **Flow Rate Limit Griefing:** An attacker can front-run a user's withdrawal that is close to flow rate limits, making a small withdrawal themselves to ensure the user's larger withdrawal gets queued.
*   **Impact:** User's transaction may fail (loss of gas) or be unexpectedly delayed. Attacker may gain an advantage (e.g., securing deposit capacity) or simply grief the user.
*   **Recommendation:** These are common MEV scenarios. While hard to eliminate at the contract level:
    *   For IMX limit, users can be made aware of the current capacity.
    *   For flow rate griefing, robust and well-capitalized thresholds (as mentioned in MED-01) make this attack more expensive and thus less likely.
    *   Consideration of MEV-resistant designs (e.g., batching, commit-reveal) is a broader architectural topic, generally complex for bridges.
*   **Reference to Plan Step:** Other Critical (Step 4).

#### MED-03: Potential DoS for `findPendingWithdrawals` via Array Bloating

*   **Severity:** Medium (bordering Low)
*   **Contract(s) & Function(s) Involved:** `FlowRateWithdrawalQueue._enqueueWithdrawal`, `FlowRateWithdrawalQueue.findPendingWithdrawals`.
*   **Description:** If an attacker can cause a large number of withdrawals to be enqueued for a specific `receiver` address (even if the L2 initiator is the attacker but they target a known L1 `receiver`), the `pendingWithdrawals[receiver]` array can become very large. Calls to `findPendingWithdrawals` (and `getPendingWithdrawals` for many indices) by the victim user or their UI could then become very gas-intensive, potentially hitting block gas limits.
*   **Impact:** Degraded user experience for the victim when trying to locate their specific queued withdrawals via these view functions. It does not prevent withdrawal via `finaliseQueuedWithdrawal` if the index is known.
*   **Recommendation:**
    *   The `TODO` comment about potentially using a mapping instead of an array in `_enqueueWithdrawal` could be explored, but mappings also have complexities for enumeration.
    *   Encourage UIs to allow users to input known withdrawal indices directly.
    *   Consider adding optional paging or limits to `findPendingWithdrawals` if it becomes a practical issue, though this adds complexity.
*   **Reference to Plan Step:** Flow Rate Part 1 (Step 2).

### Low Severity

#### LOW-01: Malicious Token Metadata Impacting Gas/Off-chain Systems

*   **Severity:** Low
*   **Contract(s) & Function(s) Involved:** `RootERC20Bridge._mapToken`, `RootERC20Bridge._getTokenDetails`.
*   **Description:** If a token being mapped via `mapToken` returns excessively long strings for `name()` or `symbol()`, it could lead to increased gas costs for the user calling `mapToken` on L1 due to larger payload encoding. It might also cause issues for off-chain indexers or UIs trying to process or display this metadata, or potentially hit GMP payload size limits or L2 gas limits during message processing.
*   **Impact:** Increased user gas costs for mapping, potential failure to map if limits are exceeded, or display issues in frontends. No direct risk to bridge funds.
*   **Recommendation:** This is a minor issue. The bridge currently reverts with `TokenNotSupported` if metadata functions are missing. Consider if any practical limits on string length are necessary, though this could also limit legitimate tokens. Robust off-chain handling is advisable.
*   **Reference to Plan Step:** RootERC20Bridge Part 1 (Step 1).

---

## 3. Key Security Observations/Concerns

*   **Centralization of Trust in Administrative Roles:** The security of the bridge and its funds heavily relies on the secure management of roles like `DEFAULT_ADMIN_ROLE`, `ADAPTOR_MANAGER_ROLE`, and `RATE_CONTROL_ROLE`. Compromise of these roles can lead to catastrophic failure, including fund theft or permanent DoS. Multi-signature wallets with diverse, trusted holders and strict operational security are essential for these roles.
*   **Bridge Adaptor Dependency:** The `IRootBridgeAdaptor` is a critical component. Its correctness, security, and reliability are paramount:
    *   A compromised adaptor can drain all bridge funds.
    *   A faulty adaptor that succeeds on L1 but fails to ensure L2 execution can lead to user funds being stuck on L1.
*   **L2 System Reliability:** The overall success of bridging operations depends on the reliability of the L2 system to correctly process messages sent from L1 and to send valid messages back. Failures on L2 can impact L1 assets or user experience.
*   **MEV and Griefing:** The system is exposed to standard MEV front-running tactics, and the flow-rate mechanism introduces a specific griefing vector where an attacker can force other users' withdrawals into the queue. While difficult to eliminate entirely, parameter tuning can make these attacks more costly.
*   **Complexity of FlowRate System:** The `RootERC20BridgeFlowRate` contract, by inheriting and combining three distinct functionalities (base bridge, flow detection, withdrawal queue), is complex. While this modularity is good for development, it requires careful analysis of interactions, as demonstrated by the reentrancy issue persisting in one path.

---

## 4. Positive Security Practices Noted

*   **Use of Solidity 0.8.x:** Benefits from built-in checked arithmetic, reducing overflow/underflow risks.
*   **OpenZeppelin Contracts:** Extensive use of audited OpenZeppelin contracts for core functionalities like `AccessControlUpgradeable`, `PausableUpgradeable`, `ReentrancyGuardUpgradeable`, `SafeERC20`, and `Clones` is a strong positive.
*   **Initialization Protection:** Use of `initializer` modifier and `initializerAddress` check in constructor effectively prevents front-running of initialization.
*   **Immutability of Critical Addresses Post-Initialization:** Key addresses like `childERC20Bridge`, `childTokenTemplate`, `rootIMXToken`, `rootWETHToken` are set once and cannot be changed, reducing attack surface.
*   **Specific Input Validations:** Numerous checks for `address(0)`, zero amounts, and correct `msg.value` are present.
*   **`nonReentrant` Modifier Usage:** Applied to the deposit flow (`_deposit`) and to the finalization of queued withdrawals (`finaliseQueuedWithdrawal`, `finaliseQueuedWithdrawalsAggregated`), which is good. (The gap is in the non-queued withdrawal path).
*   **Balance Invariant Checks:** The `expectedBalance` checks in deposit functions (`_depositETH`, `_depositWrappedETH`, `_depositERC20`) provide a strong safeguard against accounting errors or issues with external token interactions (like WETH unwrapping).
*   **Explicit Handling of Non-Standard Tokens:** The `_getTokenDetails` function attempts to fetch metadata and reverts with `TokenNotSupported` if standard functions are missing. Comments also warn about undefined behavior for non-standard ERC20s (e.g., rebasing tokens).
*   **Robust Error Handling:** Custom errors are defined, providing more context than simple reverts with string messages.
*   **Prevention of Zero Capacity/RefillRate:** The `_setFlowRateThreshold` function in `FlowRateDetection.sol` explicitly prevents setting `capacity` or `refillRate` to zero, which is important for the integrity of the flow rate mechanism.

---

This draft report consolidates the findings from the detailed analysis. Further review and discussion are recommended, particularly regarding the critical and high-severity issues.
