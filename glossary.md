## Glossary

This glossary defines key terms specific to the Immutable zkEVM bridge protocol and its associated smart contracts.

| Term                          | Definition                                                                                                                                                              |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **L1 (Layer 1)**              | The primary blockchain, typically referring to Ethereum Mainnet, where the root bridge contracts are deployed and assets are originally held or locked.                    |
| **L2 (Layer 2)**              | The secondary scaling solution, in this context, the Immutable zkEVM chain, where computation is cheaper and faster. Assets are "bridged" to and from L1.                 |
| **Root Chain**                | Synonym for L1, the main Ethereum blockchain where the `RootERC20Bridge` and associated root contracts operate.                                                          |
| **Child Chain**               | Synonym for L2, the Immutable zkEVM chain where corresponding "child" bridge contracts operate to mint/release tokens based on L1 actions.                                |
| **Bridge Contract**           | A smart contract (e.g., `RootERC20Bridge`, `RootERC20BridgeFlowRate`) that facilitates the transfer of assets and information between L1 and L2.                             |
| **Bridge Adaptor**            | A component (e.g., implementing `IRootBridgeAdaptor`) that abstracts the underlying message passing protocol, allowing the bridge to send/receive messages between L1 and L2. |
| **GMP (General Message Passing)** | A generic protocol or system used by the Bridge Adaptor to transmit arbitrary data/messages between different blockchains (L1 and L2).                                  |
| **Token Mapping**             | The process of registering an L1 ERC20 token with the bridge system to enable its transfer to L2, often involving the creation or association of a corresponding L2 token. |
| **Child Token Template**      | An L2 smart contract implementation that is cloned (using `CREATE2`) to create new L2 token contracts when L1 tokens are mapped.                                          |
| **Flow Rate Control**         | A security mechanism to monitor the rate of token withdrawals from the bridge to detect and mitigate rapid outflows.                                                    |
| **Flow Rate Bucket**          | A data structure (`Bucket` struct in `FlowRateDetection.sol`) associated with each token to track its withdrawal volume against configured capacity and refill rates.       |
| **Bucket Capacity**           | The maximum amount of a token that can be withdrawn within a short period before triggering flow rate limits (part of the Flow Rate Bucket).                               |
| **Bucket Depth**              | The current "fill level" of a Flow Rate Bucket. Withdrawals decrease depth; it refills over time at the `refillRate`.                                                     |
| **Refill Rate**               | The rate (tokens per second) at which a Flow Rate Bucket's depth replenishes.                                                                                           |
| **Large Transfer Threshold**  | A configurable limit for a single withdrawal of a specific token. Withdrawals exceeding this threshold are automatically sent to the Withdrawal Queue.                  |
| **Withdrawal Queue**          | A system (`FlowRateWithdrawalQueue.sol`) that holds withdrawal requests that have been delayed due to flow rate limits or manual activation of safety protocols.           |
| **Pending Withdrawal**        | A specific withdrawal request (`PendingWithdrawal` struct) stored in the Withdrawal Queue, awaiting the end of its `withdrawalDelay`.                                     |
| **Withdrawal Delay**          | A configurable period of time that a `PendingWithdrawal` must wait in the queue before it can be finalized and the funds released to the user.                           |
| **IMX**                       | The native utility and gas token of the Immutable X platform and the Immutable zkEVM chain. It has special handling in the bridge (e.g., pre-mapped, deposit limits).    |
| **WETH (Wrapped ETH)**        | An ERC20-compliant representation of Ether. Users often wrap ETH to interact with ERC20-based DeFi protocols. The bridge unwraps WETH deposits on L1.                    |
| **NATIVE_ETH (`0xeee`)**      | A special address used internally by the bridge to represent native Ether in deposit/withdrawal messages and mappings.                                                    |
| **NATIVE_IMX (`0xfff`)**      | A special address used internally by the bridge, likely to represent native IMX on the child chain in messages.                                                            |
| **Access Control Roles**      | Permissions (e.g., `DEFAULT_ADMIN_ROLE`, `PAUSER_ROLE`, `RATE_CONTROL_ROLE`) that restrict who can call sensitive functions on the bridge contracts.                     |
| **`initializerAddress`**      | An address set in the constructor of proxy-upgradeable contracts, authorized to call the `initialize` function to prevent front-running.                                |
| **`nonReentrant` Modifier**   | A common security measure to prevent reentrancy attacks, where an external call back into the contract before the initial function completes could manipulate state.      |
| **`whenNotPaused` Modifier**  | A security measure ensuring that certain functions can only be called when the contract (or parts of it) is not paused.                                                 |

---
