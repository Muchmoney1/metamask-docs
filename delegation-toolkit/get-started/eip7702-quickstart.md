---
description: Upgrade an externally owned account (EOA) to a smart account
sidebar_position: 4
sidebar_label: EIP-7702 quickstart
---

# EIP-7702 quickstart

This page demonstrates how to upgrade your externally owned account (EOA) to support MetaMask Smart Accounts 
functionality using an [EIP-7702](https://eips.ethereum.org/EIPS/eip-7702) transaction. This enables your EOA to leverage the benefits of account 
abstraction, such as batch transactions, gas sponsorship, and [ERC-7710 delegation capabilities](./../concepts/delegation.md).

## Prerequisites

- [Install and set up the Delegation Toolkit.](install.md)
- [Install Viem](https://viem.sh/)

## Steps

### 1. Set up a Public Client

Set up a [Viem Public Client](https://viem.sh/docs/clients/public) using Viem's `createPublicClient` function. 
This client will let the EOA query the account state and interact with blockchain network.

```typescript
import { createPublicClient, http } from "viem";
import { sepolia as chain } from "viem/chains";
 
const publicClient = createPublicClient({
  chain,
  transport: http(),
});
```

### 2. Set up a Bundler Client

Set up a [Viem Bundler Client](https://viem.sh/account-abstraction/clients/bundler) using Viem's `createBundlerClient` function.
This lets you use the bundler service to estimate gas for user operations and submit transactions to the network.

```typescript
import { createBundlerClient } from "viem/account-abstraction";

const bundlerClient = createBundlerClient({
  client: publicClient,
  transport: http("https://your-bundler-rpc.com"),
});
```

### 3. Set up a Wallet Client

Set up [Viem Wallet Client](https://viem.sh/docs/clients/wallet) using Viem's `createWalletClient` function. 
This lets you sign and submit EIP-7702 authorization.

```typescript
import { createWalletClient, http } from "viem";
import { sepolia as chain } from "viem/chains";
import { privateKeyToAccount } from "viem/accounts";
 
export const account = privateKeyToAccount("0x...");
 
export const walletClient = createWalletClient({
  account,
  chain,
  transport: http(),
});
```

### 4. Authorize a 7702 delegation

Create an authorization to map the contract code to an EOA, and sign it 
using Viem's [`signAuthorization`](https://viem.sh/docs/eip7702/signAuthorization) action. The `signAuthorization` action 
does not support JSON-RPC accounts.

This example uses [`EIP7702StatelessDeleGator`](https://github.com/MetaMask/delegation-framework/blob/main/src/EIP7702/EIP7702StatelessDeleGator.sol) as the EIP-7702 delegator contract. 
It follows a stateless design, as it does not store signer data in the contract's state. This approach
provides a lightweight and secure way to upgrade an EOA to a smart account.

```typescript
import {
  Implementation,
  toMetaMaskSmartAccount,
  getDeleGatorEnvironment,
} from "@metamask/delegation-toolkit";
import { privateKeyToAccount } from "viem/accounts";

const environment = getDeleGatorEnvironment(sepolia.id);
const contractAddress = environment.implementations.EIP7702StatelessDeleGatorImpl;

const authorization = await walletClient.signAuthorization({
  account, 
  contractAddress,
  executor: "self", 
});
```

### 5. Submit the authorization

Once you have signed an authorization, you can send an EIP-7702 transaction to set the EOA code.
Since the authorization cannot be sent by itself, you can include it alongside a dummy transaction.

```ts
import { zeroAddress } from "viem";

const hash = await walletClient.sendTransaction({ 
  authorizationList: [authorization], 
  data: "0x", 
  to: zeroAddress, 
});
```

### 6. Create a MetaMask smart account

Create a smart account instance for the EOA and start 
leveraging the benefits of account abstraction.

```ts
import { 
  Implementation, 
  toMetaMaskSmartAccount,
} from "@metamask/delegation-toolkit";

const addresses = await walletClient.getAddresses();
const address = addresses[0];

const smartAccount = await toMetaMaskSmartAccount({
  client: publicClient,
  implementation: Implementation.Stateless7702,
  address, 
  signatory: { walletClient },
});
```

### 7. Send a user operation

Send a user operation through the upgraded EOA, using Viem's [`sendUserOperation`](https://viem.sh/account-abstraction/actions/bundler/sendUserOperation) method.

```ts
import { parseEther } from "viem";

// Appropriate fee per gas must be determined for the specific bundler being used.
const maxFeePerGas = 1n;
const maxPriorityFeePerGas = 1n;

const userOperationHash = await bundlerClient.sendUserOperation({
  account: smartAccount,
  calls: [
    {
      to: "0x1234567890123456789012345678901234567890",
      value: parseEther("1")
    }
  ],
  maxFeePerGas,
  maxPriorityFeePerGas
});
```
