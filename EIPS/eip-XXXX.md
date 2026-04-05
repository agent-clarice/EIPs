---
eip: XXXX
title: On-Chain Fee Recipient Registry for Validators
description: System contract allowing validators to set and update execution layer fee recipients via on-chain transactions
author: James Ninnes (@james-ninnes)
discussions-to: https://ethereum-magicians.org/t/eip-xxxx-on-chain-fee-recipient-registry-for-validators
status: Draft
type: Standards Track
category: Core
created: 2026-04-05
requires: 4788
---

## Abstract

This EIP introduces a system-level smart contract that allows Ethereum validators to set and update their execution layer fee recipient address via an on-chain transaction. Currently, the fee recipient (`suggested_fee_recipient`) is configured locally in the validator client and communicated to the beacon node via the Beacon API. This proposal moves fee recipient management on-chain with protocol-level enforcement, giving fee recipients the same security guarantees that withdrawal credentials provide for validator principal.

## Motivation

### The Security Gap

Ethereum's proof-of-stake design provides strong, protocol-enforced guarantees for validator funds through withdrawal credentials. Once a validator sets 0x01 withdrawal credentials, no operator, client bug, or misconfiguration can redirect those funds — the protocol enforces the destination at the consensus layer. This is a fundamental security property.

**Fee recipients have no such protection.**

Today, the fee recipient — the address that receives priority fees and MEV revenue from proposed blocks — is a local configuration value in the validator client. It is not enforced by the protocol. It is not recorded on-chain. It is entirely trust-based.

This creates an asymmetry at the heart of Ethereum's staking security model:

| | Validator Principal (Withdrawals) | Validator Revenue (Fee Recipient) |
|---|---|---|
| **Set by** | On-chain (protocol-enforced) | Local config (trust-based) |
| **Can operator redirect?** | No | Yes |
| **Verifiable on-chain?** | Yes | No |
| **Survives client migration?** | Yes | No |
| **Protection against misconfiguration?** | Protocol rejects invalid changes | Silent fallback to zero address |

Withdrawal credentials solved the problem of "an operator can steal your principal." Fee recipients are the unsolved equivalent: **an operator can steal your revenue, and the protocol does nothing to prevent it.**

### Why This Matters Now

As Ethereum's staking ecosystem matures, an increasing share of validators are operated by third parties:

- **Liquid staking protocols** delegate validation to node operators. Stakers trust that operators configure fee recipients honestly. There is no on-chain mechanism to verify or enforce this.
- **DVT clusters** (e.g., SSV, Obol) distribute validator duties across multiple operators. Fee recipient configuration must be agreed upon and correctly set by the active operator — with no protocol enforcement.
- **Institutional custodians** manage validators on behalf of clients. Fee recipient misconfiguration or misappropriation is undetectable on-chain.
- **Solo stakers** migrating between clients, machines, or failover setups must manually reconfigure fee recipients each time, with silent failure on misconfiguration.

The protocol should not rely on operator goodwill for revenue security any more than it relies on operator goodwill for fund security. Both deserve the same guarantee.

### Design Principle

> **If withdrawal credentials make validator principal unstealable, fee recipient credentials should make validator revenue unstealable.**

This EIP closes the gap by introducing an on-chain fee recipient registry with protocol-level enforcement, controlled exclusively by the validator's withdrawal address — the same entity the protocol already trusts with fund security.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Parameters

| Constant | Value |
|---|---|
| `FEE_RECIPIENT_REGISTRY_ADDRESS` | `TBD` |
| `FORK_TIMESTAMP` | `TBD` |

### Fee Recipient Registry Contract

At `FORK_TIMESTAMP`, a system contract is deployed at `FEE_RECIPIENT_REGISTRY_ADDRESS`. The contract exposes the following interface:

```solidity
/// @title IFeeRecipientRegistry
/// @notice On-chain registry for validator fee recipients
interface IFeeRecipientRegistry {
    /// @notice Emitted when a fee recipient is set or updated.
    /// @param validatorIndex The beacon chain validator index.
    /// @param feeRecipient The new fee recipient address.
    /// @param caller The withdrawal address that made the change.
    event FeeRecipientSet(
        uint64 indexed validatorIndex,
        address indexed feeRecipient,
        address indexed caller
    );

    /// @notice Set or update the fee recipient for a validator.
    /// @dev MUST revert if the caller is not the validator's withdrawal address.
    /// @dev MUST revert if the validator does not have 0x01-prefixed withdrawal credentials.
    /// @param validatorIndex The beacon chain validator index.
    /// @param feeRecipient The desired fee recipient address.
    function setFeeRecipient(uint64 validatorIndex, address feeRecipient) external;

    /// @notice Set or update the fee recipient for multiple validators in a single transaction.
    /// @dev MUST revert if the caller is not the withdrawal address for ALL specified validators.
    /// @param validatorIndices Array of beacon chain validator indices.
    /// @param feeRecipient The desired fee recipient address (applied to all).
    function setFeeRecipientBatch(uint64[] calldata validatorIndices, address feeRecipient) external;

    /// @notice Query the on-chain fee recipient for a validator.
    /// @param validatorIndex The beacon chain validator index.
    /// @return recipient The registered fee recipient, or address(0) if unset.
    function getFeeRecipient(uint64 validatorIndex) external view returns (address recipient);
}
```

### Withdrawal Credential Verification

The contract MUST verify that `msg.sender` matches the execution layer address embedded in the validator's 0x01-prefixed withdrawal credentials. To perform this verification on the execution layer, the contract utilises the beacon block root exposed by [EIP-4788](./eip-4788.md).

The caller provides a Merkle proof demonstrating that:

1. The validator at `validatorIndex` exists in the beacon state.
2. The validator's `withdrawal_credentials` field has a `0x01` prefix.
3. The trailing 20 bytes of `withdrawal_credentials` match `msg.sender`.

The proof is verified against a recent beacon block root available via the EIP-4788 oracle.

The `setFeeRecipient` function signature with proof data:

```solidity
function setFeeRecipient(
    uint64 validatorIndex,
    address feeRecipient,
    bytes32 beaconBlockRoot,
    bytes calldata withdrawalCredentialProof
) external;
```

> Note: The simplified interface in the previous section omits proof parameters for readability. Implementations MUST include proof verification.

### Execution Layer Block Construction

After `FORK_TIMESTAMP`, execution layer clients constructing blocks MUST apply the following logic when determining the `feeRecipient` field of the execution payload header:

1. Read the fee recipient registry at `FEE_RECIPIENT_REGISTRY_ADDRESS` for the proposing validator's index.
2. If a non-zero address is registered, use it as the block's `feeRecipient`.
3. If no entry exists (returns `address(0)`), use the `suggestedFeeRecipient` from the `engine_forkchoiceUpdatedV*` call (current behaviour).

This preserves full backward compatibility.

### Consensus Layer

No consensus layer specification changes are required. The `prepare_beacon_proposer` Beacon API endpoint and validator client `ProposerConfig` continue to function as a fallback mechanism.

## Rationale

### Withdrawal address as authoriser

The withdrawal address is the canonical "owner" of a validator — the entity the protocol already trusts with the most sensitive operation (fund withdrawals). Granting it authority over fee recipient configuration is a natural extension of the same security model. Just as no operator can redirect withdrawals, no operator should be able to redirect fee revenue once the owner has registered a recipient on-chain.

### Protocol enforcement, not voluntary adoption

A critical design requirement is that the EL client MUST respect the on-chain registry. An alternative approach — deploying a voluntary contract and asking client teams or operators to read it via sidecar software — does not close the security gap. A malicious or negligent operator simply does not run the sidecar. Protocol-level enforcement is the only mechanism that provides the same unconditional guarantee as withdrawal credentials.

### System contract vs. beacon state field

Two on-chain approaches were considered:

1. **System contract on the execution layer** (this proposal)
2. **New field on the `Validator` container in the beacon state**, changeable via a signed voluntary message (analogous to `BLSToExecutionChange`)

A contract-based approach was chosen because:

- It requires no consensus specification changes beyond reading a contract during block construction.
- It provides a standard ABI and emits events, enabling straightforward integration with tooling, dashboards, and indexers.
- It can be extended in future EIPs (e.g., time-locked changes, multi-sig authorisation) without further protocol modifications.
- It avoids increasing the beacon state size, which has validator-count scaling implications.

The beacon state approach is a valid alternative with different trade-offs (see [Considered Alternatives](#considered-alternatives)).

### Optional with fallback

Mandatory on-chain registration would break every existing validator configuration. The fallback model ensures zero disruption: validators who never interact with the registry experience no change, while those who opt in gain protocol-enforced guarantees equivalent to withdrawal credentials.

### Batch operations

Large operators (staking pools, institutional validators) may manage thousands of validators. The `setFeeRecipientBatch` function avoids the gas cost and UX burden of thousands of individual transactions.

### EIP-4788 dependency

Using the beacon block root oracle (EIP-4788) for withdrawal credential verification is the most trust-minimised approach available on the execution layer. It avoids introducing new precompiles or cross-layer communication mechanisms.

## Considered Alternatives

### 1. Voluntary contract with sidecar software

Deploy a registry contract without protocol changes. Operators run a sidecar that reads the contract and updates their validator client's `proposer_config` or calls `prepare_beacon_proposer`.

**Rejected because:** This does not provide protocol-level enforcement. The operator must voluntarily run the sidecar. A malicious operator simply does not run it, and there is no on-chain mechanism to detect or prevent fee misappropriation. This approach cannot deliver the same security guarantee as withdrawal credentials.

### 2. Beacon state field with voluntary message

Add a `fee_recipient` field to the beacon state `Validator` container, changeable via a new signed voluntary message type (similar to `BLSToExecutionChange`).

**Trade-offs:** This approach provides consensus-layer enforcement and avoids EL contract complexity. However, it increases beacon state size per validator, requires consensus specification changes, and is less extensible than a contract. It remains a viable alternative and could be proposed as a competing or complementary EIP.

### 3. Client-level flag to read a specific contract

Client teams voluntarily add a flag (e.g., `--fee-recipient-registry=0x...`) to read a deployed contract during block construction. No protocol change required.

**Rejected because:** Voluntary client support is fragile. Not all clients may implement it, implementations may differ, and there is no guarantee of consistent behaviour across the network. Protocol-level specification ensures uniform enforcement.

## Backwards Compatibility

This EIP is fully backward compatible:

- Validators who do not interact with the registry are unaffected.
- The `suggestedFeeRecipient` field in `engine_forkchoiceUpdatedV*` remains functional as a fallback.
- Existing validator client configurations continue to work as-is.
- The on-chain registry only takes precedence when a validator (or their withdrawal address holder) explicitly opts in.

## Test Cases

### Case 1: Set fee recipient for a single validator

1. Validator 12345 has 0x01 withdrawal credentials pointing to address `0xAA...AA`.
2. `0xAA...AA` calls `setFeeRecipient(12345, 0xBB...BB, ...)` with a valid proof.
3. `getFeeRecipient(12345)` returns `0xBB...BB`.
4. When validator 12345 next proposes a block, the `feeRecipient` in the execution payload header is `0xBB...BB`.

### Case 2: Unauthorised caller

1. Validator 12345 has 0x01 withdrawal credentials pointing to `0xAA...AA`.
2. `0xCC...CC` calls `setFeeRecipient(12345, 0xCC...CC, ...)`.
3. Transaction reverts — caller does not match withdrawal credentials.

### Case 3: Unregistered validator (fallback)

1. Validator 67890 has never interacted with the registry.
2. `getFeeRecipient(67890)` returns `address(0)`.
3. EL client uses `suggestedFeeRecipient` from `engine_forkchoiceUpdatedV*` (existing behaviour).

### Case 4: Legacy BLS credentials

1. Validator 11111 has 0x00 (BLS) withdrawal credentials.
2. Any call to `setFeeRecipient(11111, ...)` reverts — validator must first convert to 0x01 credentials.

## Reference Implementation

TBD — A reference Solidity implementation and proof generation library will be provided.

## Security Considerations

### Withdrawal credential verification

The contract MUST correctly verify Merkle proofs against the EIP-4788 beacon block root. An incorrect or bypassable verification would allow any address to set the fee recipient for any validator, enabling fee theft.

### Beacon root staleness

The EIP-4788 ring buffer stores the most recent 8,191 beacon block roots (~27 hours). Proofs MUST be generated against a root within this window. Validator credential changes (e.g., via a `BLSToExecutionChange`) that occur after the proof's reference root but before the transaction is included could theoretically allow a stale proof. Implementations SHOULD use a recent root and MAY impose a maximum age on proof references.

### Front-running and MEV

Setting a fee recipient is not time-sensitive — it affects future proposed blocks, not the current transaction. Front-running this transaction provides no economic advantage to an attacker.

### Interaction with MEV-Boost

MEV-Boost and external block builders negotiate fee recipients out-of-band via proposer registrations. This EIP does not alter that flow. However, the on-chain registry could serve as a verifiable source of truth that relay operators and builders reference, improving trust in the MEV supply chain.

### Gas cost

Proof verification involves hashing operations for Merkle proof validation. The gas cost is bounded and predictable. For batch operations, the cost scales linearly with the number of validators.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
