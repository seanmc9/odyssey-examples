# Delegating an account to a P256 key

[EIP-7702](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-7702.md) and [EIP-7212](https://github.com/ethereum/RIPs/blob/master/RIPS/rip-7212.md) allow to delegate control over an EOA to a P256 key. This has large potential for UX improvement as P256 keys are adopted by commonly used protocols like [Apple Secure Enclave](https://support.apple.com/en-au/guide/security/sec59b0b31ff/web) and [WebAuthn](https://webauthn.io). 

## Why do EIP-7702+EIP-7212 matter?
The traditional flow of crypto onboarding experience can feel cumbersome: Users have to setup a wallet, back up their mnemonic phrase and make sure to keep it safe. What if there was a simpler and more secure way to manage private keys? Passkeys have already solved this problem by allowing users to authenticate using methods like Touch ID while keeping passwords safe. These keys are generated within secure hardware modules, such as Apple's Secure Enclave or a Trusted Platform Module (TPM), which are isolated from the operating system to protect them from being exposed. 

EIP-7212 introduces a precompile for the **secp256r1** elliptic curve, a curve that is widely used in protocols like [Apple Secure Enclave](https://support.apple.com/en-au/guide/security/sec59b0b31ff/web) and [WebAuthn](https://webauthn.io). 
EIP-7702 introduces a new transaction type, allowing an Externally Owned Accounts (EOAs) to function like a smart contract. This unlocks features such as gas sponsorship, transaction bundling or granting limited permissions to a sub-key.

This example demonstrates how the upcoming EIP's EIP-7720 and EIP-7212 (already live in Odyssey's Chapter 1), will enable you to use a passkey to sign an on-chain message, improving the onboarding experience for crypto novices using your Dapp.

## Steps involved

- Run anvil in Odyssey mode to enable support for EIP-7702 and P256 precompile:

```bash
anvil --odyssey
```

Anvil automatically generates pre-funded dev accounts, for the example below we will be using dev account with address `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266` and private key `0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80` 

- Now generate a P256 private and public key pair using the python script in the repo

```bash
python p256.py gen
```

This script will generate a `p256` private and public key pair, save them to `private.pem` and `public.pem` respectively, and print the keys in hex format.

- Deploy [P256Delegation](src/P256Delegation.sol) contract, which we will be delegating to

```bash
forge create P256Delegation --private-key "0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80"
```

- Send an EIP-7702 transaction, which delegates to our newly deployed contract. This transaction will both authorize the delegation and set it to use our P256 public key that we have generated previously:

```bash
export DELEGATE_ADDRESS=<enter-delegate-contract-address>
export PUBKEY_X=<enter-public-key-x>
export PUBKEY_Y=<enter-public-key-y>
cast send 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 'authorize(uint256,uint256)' $PUBKEY_X $PUBKEY_Y --auth $DELEGATE_ADDRESS --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
```

- Verify that new code at our EOA account contains the [delegation designation](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-7702.md#delegation-designation), a special opcode prefix to highlight the code has a special purpose:

```bash
$ cast code 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
0xef0100...
```

- Prepare signature to be able to transact on behalf of the EOA account by using the `transact` function of the delegation contract. Let's generate a signature for sending 1 ether to zero address by using our P256 private key:

```bash
python p256.py sign $(cast abi-encode 'f(uint256,address,bytes,uint256)' $(cast call 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 'nonce()(uint256)') '0x0000000000000000000000000000000000000000' '0x' '1000000000000000000')
```

Let’s look at the command step-by-step 

- `python p256.py sign` function signs the message with our previously generated p256 key
- `cast abi-encode 'f(uint256,address,bytes,uint256)` abi-encodes the payload expected by the `P256Delegation` contract with the following fields
    - nonce: `$(cast call 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 'nonce()(uint256)')` is used to fetch the nonce from our EOA to protect against replay attacks
    - address: `0x0000000000000000000000000000000000000000`
    - bytes: `0x`
    - amount: `1000000000000000000` wei (= 1 ETH)

The command output will respond with the signature r and s values. 

- Send the message including signature via `transact` function of the delegation contract:

```bash
# use dev account
export SENDER_PK=0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
export SIG_R=<enter-signature-r>
export SIG_S=<enter-signature-s>
cast send 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 'transact(address to,bytes data,uint256 value,bytes32 r,bytes32 s)' '0x0000000000000000000000000000000000000000' '0x' '1000000000000000000' $SIG_R $SIG_S --private-key $SENDER_PK
```

Note that, similarly to [simple-7702 example](../simple-7702), there is no restriction on who could submit this transaction.
