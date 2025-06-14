---
eip: XXXX
title: Stateless Encrypted Communication Standard
description: A log-based protocol for encrypted communication between two addresses on EVM chains.
author: Scott Ewing (@sewing848) <sewing848@gmail.com>
discussions-to: https://ethereum-magicians.org/t/erc-stateless-encrypted-communication-standard
status: Draft
type: Standards Track
category: ERC
created: 2025-06-10
---

## Abstract

This ERC defines the Stateless Encrypted Communication Standard, a minimal, non-financial, gas-efficient communication protocol. It enables encrypted peer-to-peer communication through a single emitted event. The protocol maintains no on-chain state and avoids economic constructs. It assumes the use of an ECDH key exchange to establish a shared symmetric key for secure encrypted data transfer. All message semantics are interpreted and enforced off-chain by client applications.

---

## Motivation

This ERC proposes a shared, minimal primitive for encrypted messaging over EVM-compatible blockchains. The intent is not merely to introduce a new protocol, but to establish an interoperable, gas-efficient interface that can serve as the default on-chain transport layer for peer-to-peer encrypted communication.

Rather than prescribing a complete protocol with architectural decisions about key management, encryption algorithms, or session handling, this ERC defines only the essential primitives needed to transmit encrypted messages. This minimal approach enables maximum flexibility: applications can implement their own key exchange strategies, choose their preferred encryption methods, and add economic or spam-prevention mechanisms as needed, while maintaining interoperability through the standardized event structure.

Existing token-based and messaging systems typically rely on state mutation, require financial interactions, or assume specific application logic. There is currently no minimal, stateless messaging standard that can serve as a common substrate for decentralized communication.

This standard defines a simple, composable interface that:

- Supports ECDH-based encrypted communication and connection setup
- Emits only one event for all message types
- Leaves interpretation and enforcement to off-chain clients
- Enables privacy-preserving and censorship-resistant messaging applications

By converging on a stateless, event-based design, this ERC aims to provide a stable and composable core that enables interoperability among messaging clients, libraries, and extensions. Applications requiring additional features, such as on-chain key registries, structured session management, or protocol-enforced encryption algorithms, can layer these capabilities on top of this foundational standard while maintaining compatibility with the basic message emission interface.

---

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Contract Interface

Implementations of this protocol MUST define an event named by concatenating the contract name with "Transfer".
Implementations SHOULD expose the contract and event names as public constant strings to support introspection and off-chain tooling.

```solidity

string public constant CONTRACT_NAME = "ContractName";  // Implementation-specific
string public constant EVENT_NAME = "ContractNameTransfer"; // Implementation-specific

// Implementers MUST define an event using the specified structure, with an implementation-specific name exposed via `eventName`.
event ContractNameTransfer(
    address indexed from,
    address indexed to,
    uint256 indexed messageType,
    bytes data
);

function sendMessage(address to, uint256 messageType, bytes calldata data) external;
```

- **from**: the sender of the message (automatically `msg.sender`)
- **to**: the intended recipient
- **messageType**: a message classifier indicating how the data payload should be interpreted
- **data**: an opaque binary payload, formatted off-chain by convention

### Message Type Semantics

The protocol reserves the following `messageType` values for standardized encrypted communication primitives:

| messageType | Meaning                        | Payload                                 |
|-------------|--------------------------------|-----------------------------------------|
| 0           | Connection Request             | Public key + encrypted request body     |
| 1           | Connection Request Response    | Public key + encrypted response body    |
| 2           | Encrypted Text Message         | AES-GCM encrypted message data          |


Client applications MUST interpret message types 0, 1, and 2 according to the above semantics to ensure basic interoperability.

These types are intended to follow this general RECOMMENDED cryptographic model:

- A shared symmetric key is derived using Elliptic Curve Diffie-Hellman (ECDH) from the sender and recipient key pairs.
- Payloads are encrypted using an authenticated encryption scheme, typically AES-GCM.

Implementers MAY vary the details of symmetric key derivation (e.g., direct use of shared secret, use of HKDF, or key rotation strategies) and encryption parameters (e.g., nonce generation), so long as the overall structure and intent are preserved. However, if substantial deviations from ECDH-based key exchange or AES-GCM semantics are used, it is RECOMMENDED to define a new messageType (e.g., >= 3) to avoid misinterpretation by other clients.

All encryption, decryption, and key exchange logic MUST be handled off-chain.

---

## Rationale

This core protocol is intended for encrypted communication between two addresses. Multiple message types can be filtered using `eth_getLogs` or `eth_subscribe` without needing contract storage. The core protocol omits economic interactions and access controls. By emitting only a single event, the protocol reduces on-chain complexity and minimizes gas fees.

### Related EIPs

This proposal is intentionally minimal and stateless, in the spirit of [ERC-3722: Poster](https://eips.ethereum.org/EIPS/eip-3722), which emits plaintext posts for decentralized social media. While Poster focuses on public broadcast-style messaging, this proposed standard defines a private, encrypted peer-to-peer communication model.

The two differ not just in use case but in security assumptions: posting plaintext messages to an immutable, public blockchain poses long-term risks, especially if illegal or harmful content becomes permanently associated with a project or platform. This standard is specifically intended for encrypted communication between two addresses - not plaintext public posts.

This proposal also contrasts with [ERC-7627: Secure Messaging Protocol](https://eips.ethereum.org/EIPS/eip-7627), which defines a comprehensive messaging interface, including on-chain key management, encryption algorithm enumeration, and session identifiers.

By contrast, this standard is intended to facilitate basic encrypted communication between two addresses within a wide range of app environments, while offering the flexibility to incorporate other features through app-specific use of the messageType parameter.

---

## Backwards Compatibility

No backward compatibility issues found.

---

## Reference Implementation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.30;

/// @title Stateless Encrypted Communication Standard
/// @notice Stateless messaging protocol for encrypted peer-to-peer communication via event emission
/// @dev This contract does not store any state and emits all communication as logs.
/// @dev All semantics are interpreted off-chain.

contract Ataraxia {

    /// @notice The name of the protocol
    string public constant CONTRACT_NAME = "Ataraxia";

    /// @notice The name of the data-carrying event
    string public constant EVENT_NAME = "AtaraxiaTransfer";

    /// @notice Emitted when a message is sent using the protocol
    /// @param from The sender of the message (automatically msg.sender)
    /// @param to The recipient of the message
    /// @param messageType The classifier of the message
    /// @param data Encrypted or structured message content, interpreted off-chain
    event AtaraxiaTransfer(
        address indexed from,
        address indexed to,
        uint256 indexed messageType,
        bytes data
    );

    /// @notice Send a message of the given type to the specified recipient
    /// @param to The recipient address
    /// @param messageType The classifier of the message
    /// @param data The encrypted or structured payload
    function sendMessage(address to, uint256 messageType, bytes calldata data) external {
        emit AtaraxiaTransfer(msg.sender, to, messageType, data);
    }

}
```

---

## Security Considerations

- The protocol emits all messages publicly as logs.
- Public keys are broadcast to establish a secure connection using ECDH key exchange and AES-GCM encryption.
- All encryption and identity management must be handled off-chain.
- Messages are publicly visible; therefore, encryption is essential to preserve confidentiality.
- The core protocol does not include any on-chain spam prevention mechanisms. Implementers can extend the protocol with anti-spam measures by requiring token transfers, or native currency payments alongside message emission.

---

## Copyright

Copyright and related rights waived via [CC0](/LICENSE).
