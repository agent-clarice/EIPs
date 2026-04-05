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

This EIP introduces a system-level smart contract that allows Ethereum validators to set and update their execution layer fee recipient address via an on-chain transaction. Currently, the fee recipient (`suggested_fee_recipient`) is configured locally in the validator client and communicated to the beacon node via the Beacon API. This proposal moves fee recipient management on-chain, making it auditable, portable across clients, and independent of local configuration.

## Motivation

The current mechanism for setting execution layer fee recipients has several shortcomings:

1. **Configuration fragility** — The fee recipient is set per-validator in client configuration files. Misconfiguration or omission silently defaults to the zero address or a client-specific fallback, resulting in lost revenue.

2. **No portability** — Migrating between validator clients, machines, or staking services requires manually re-specifying fee recipients. There is no canonical source of truth shared across the network.

3. **No auditability** — There is no on-chain record of a validator's intended fee recipient. Delegators in staking pools and DVT clusters cannot independently verify where priority fees and MEV rewards are directed.

4. **Operator trust** — Delegated staking arrangements (liquid staking, DVT clusters, institutional custodians) require trusting the operator to correctly configure the fee recipient. An on-chain registry provides a trust-minimised alternative with a verifiable commitment.

5. **Multi-client redundancy** — Operators running redundant or failover validator clients must keep fee recipient configuration synchronised across all instances manually.

An on-chain registry, updatable only by the validator's withdrawal address, solves these problems with a single transaction.

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

The withdrawal address is the canonical "owner" of a validator. It already controls the most security-critical operation — fund withdrawals. Extending its authority to fee recipient configuration is a natural and minimal trust escalation.

### System contract vs. protocol-level field

A contract-based approach was chosen over adding a new field to the `Validator` container in the beacon state because:

- It requires no consensus specification changes beyond reading a contract during block construction.
- It provides a standard ABI and emits events, enabling straightforward integration with tooling, dashboards, and indexers.
- It can be extended in future EIPs (e.g., time-locked changes, multi-sig authorisation) without further protocol modifications.

### Optional with fallback

Mandatory on-chain registration would break every existing validator configuration. The fallback model ensures zero disruption: validators who never interact with the registry experience no change, while those who opt in gain stronger guarantees.

### Batch operations

Large operators (staking pools, institutional validators) may manage thousands of validators. The `setFeeRecipientBatch` function avoids the gas cost and UX burden of thousands of individual transactions.

### EIP-4788 dependency

Using the beacon block root oracle (EIP-4788) for withdrawal credential verification is the most trust-minimised approach available on the execution layer. It avoids introducing new precompiles or cross-layer communication mechanisms.

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
