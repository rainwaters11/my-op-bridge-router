# Uniswap v4 Swap & Bridge Router (Optimism)

## Project Summary
This repository contains a custom Uniswap v4 Swap Router integrated directly with Optimism's `L1StandardBridge`. Designed as a comprehensive cross-chain execution environment, this contract allows users to seamlessly swap assets via the v4 `PoolManager` on Layer 1 and optionally bridge the resulting output tokens to Layer 2 (Optimism) in a single transaction lifecycle. 

This serves as a simplified proof-of-concept for cross-chain liquidity execution (similar to Squid Router), demonstrating how to take assets on Chain 1, execute a swap, and immediately bridge the output to Chain 2.

## The Job of the Router
End users (EOAs) cannot call the Uniswap v4 `PoolManager` directly. While standard operations typically route through the `UniversalRouter` (for swaps) or the `NFT Position Manager` (for liquidity modification), this custom router specifically handles:

1. **Routing:** Properly routing the user's action and parameters over the `PoolManager`.
2. **Settlement:** Settling delta balances with the `PoolManager` post-swap.
3. **Execution/Bridging:** Delivering output tokens directly to the user on L1, or bridging them to Optimism if the user signaled that intent via the `SwapSettings` struct.

## Mechanism Design & Constraints
Not all tokens are eligible to be bridged from L1 to L2 natively. 

To manage this, the router maintains an `ownerOnly` mapping called `l1ToL2TokenAddresses` (e.g., mapping L1 USDC `0xabc...` to L2 USDC `0xxyz...`). If a user requests a swap where the output token is *not* registered in this mapping, the transaction reverts to prevent stranded assets.

> **Note:** On Sepolia, the native bridge currently only officially supports native `ETH` and `OUTb` (Optimism Useless Token Bridgeable).

## Testing Architecture
Cross-chain testing presents unique challenges. When a user initiates a bridge transaction on the ETH L1, it emits an event. Off-chain processors (Optimism nodes) listen for this event, validate it, and subsequently trigger a transaction on the L2 (OP Sepolia) to deliver the funds. 

Because we cannot easily replicate this off-chain node infrastructure locally, our testing strategy is split into two parts:

### 1. Local Network Forking
Using Foundry's network forking capabilities, we fork the Sepolia testnet locally. This allows us to deploy our v4-related contracts and conduct swaps through our router against real network state without needing to deploy our own `L1StandardBridge` mocks.

### 2. Event Emission Validation
Instead of waiting for an L2 execution locally, our test suite rigorously verifies that the exact expected events are emitted from the L1 contracts. 

In the EVM, events are structured as follows:
* **Topic 0:** `keccak256(event signature)` – Identifies *which* event is being emitted.
* **Topic 1-3:** Optional topics utilized if the event contains `indexed` parameters (enabling easy filtering for the off-chain nodes).
* **Data:** The ABI-encoded version of all remaining non-indexed information.

Our Foundry tests utilize `vm.expectEmit(checkTopic1, checkTopic2, checkTopic3, checkData)`. (Note: Foundry always checks Topic 0 implicitly). By ensuring the `ERC20DepositInitiated` and `SentMessage` events are perfectly constructed, we can trust the off-chain Optimism nodes will process the transaction.

### Live Verification
Alongside the local tests, a deployment script is included to interact directly with the Sepolia testnet. This allows you to broadcast a public transaction and manually verify the corresponding Optimism execution on a block explorer after the standard 10-20 minute bridging delay.

## Installation & Usage

### Prerequisites
You will need [Foundry](https://getfoundry.sh/) installed to compile and test this project. 

```bash
curl -L [https://foundry.paradigm.xyz](https://foundry.paradigm.xyz) | bash
foundryup

Build & Test
Clone the repository and install the required submodules (Uniswap v4-periphery and Optimism contracts):

Bash
git clone [https://github.com/rainwaters11/my-op-bridge-router.git](https://github.com/rainwaters11/my-op-bridge-router.git)
cd my-op-bridge-router
forge install
Compile the smart contracts:

Bash
forge build
Run the local test suite against the Sepolia fork to verify the routing logic and bridge event emissions:

Bash
forge test -vvv
