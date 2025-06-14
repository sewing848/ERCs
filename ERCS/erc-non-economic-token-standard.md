---
eip: XXXX
title: Non-Economic Token
description: A non-economic fungible token standard that is intentionally not compatible with ERC-20 and related DeFi protocols.
author: Scott Ewing (@sewing848) sewing848@gmail.com
discussions-to: https://ethereum-magicians.org/t/erc-non-economic-token-standard
status: Draft
type: Standards Track
category: ERC
created: 2025-06-12

---

## Abstract

This ERC defines a standard for non-economic fungible tokens (NETs) that are intentionally not compatible with ERC-20 and related DeFi protocols. NETs are directly transferable between addresses but decay over time when held. They are designed for symbolic representation, ephemeral signaling, data transfer, and facilitating emergent system behavior rather than use cases that are financial or that seek to combine financial and non-financial properties in the same resource.

---

## Motivation

Decentralized applications may require transferable symbolic units, signaling mechanisms, or status indicators that are not intended to function as economic resources. Existing token standards such as ERC-20 are structurally optimized for financial activity. Attempts to use existing financial token standards for non-economic applications risk unintended integration with speculative markets and financial infrastructure. In environments optimized for capital formation and investment behavior, symbolic or expressive uses are routinely distorted or co-opted by economic incentives.

The Non-Economic Token (NET) standard introduces a distinct, structurally incompatible token model designed to avoid economic interpretation. NETs feature:

- Built-in decay properties to simulate impermanence and reduce utility as a store of value
- Naming conventions that explicitly avoid financial semantics
- Interfaces and contracts without `approve()` or `transferFrom()` to prevent decentralized exchange listings
- Open or permissioned creation and distribution mechanisms not tied to payments or supply limits

Standards such as [EIP-4973](./eip-4973.md), [EIP-5114](./eip-5114.md), and [EIP-5484](./eip-5484.md) have introduced non-transferable tokens that reinforce identity and reputation by restricting exchange. These approaches share a similar intent of resisting financialization but are primarily designed for non-fungible tokens. The NET standard extends this anti-financialization ethos into the fungible token space, creating symbolic resources that can decay over time while maintaining strict structural isolation from economic tooling.

---

## Specification

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “NOT RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Interface Requirements

A contract claiming compliance with the NET standard MUST implement the following interface:

```solidity
interface INET {
    function name() external view returns (string memory);
    function symbol() external view returns (string memory);
    function decimals() external view returns (uint8 decimals);
    function decayRate() external view returns (uint256 decayRate);
    function amountAt(address location) external view returns (uint256);
    function move(address to, uint256 amount) external;
    function create(address to, uint256 amount) external;
    event Moved(address indexed from, address indexed to, uint256 amount);
}
```

### General Requirements

- `name()` and `symbol()` MUST return short human-readable identifiers.
- `decimals()` MUST return 18.
- All NET-compliant contracts MUST implement a `create()` function as defined in the interface.
- There MUST NOT be a fixed or hard-coded total supply.
- `create()` MAY be restricted by permissions or rate limits but MUST NOT require payment (aside from gas).
- `move(address,uint256)` MUST transfer the specified amount from `msg.sender` to the target address.
- Implementations MUST apply decay to both the sender and recipient addresses before tokens are moved from one address to another.
- The `Moved` event MUST be emitted by `move()` and `create()` whenever tokens are moved or created.

### Decay Requirements

- All NET-compliant contracts MUST include a baseline linear decay mechanism.
- `decayRate()` MUST return the constant decay rate, expressed in token-wei per second (18 decimals).
- The decay rate MUST be immutable after contract deployment.
- `amountAt(address)` MUST return the current decayed balance at the given address, based on the elapsed time since the last update. It MUST NOT return a raw or unadjusted value.

### Optional and Recommended Additions

Contracts MAY implement more complex decay models layered atop the required linear decay, but MUST NOT allow effective decay rates to fall below the minimum defined by `decayRate()`.

Implementations with additional decay logic SHOULD expose sufficient data or documentation to allow off-chain agents to deterministically reproduce the decay process for any address over time.

It is RECOMMENDED to include ERC-165 support for interface detection:

``` solidity
function supportsInterface(bytes4 interfaceId) public view virtual override returns (bool) {
    return interfaceId == type(INET).interfaceId || super.supportsInterface(interfaceId);
}
```

---

## Rationale

This EIP defines a Non-Economic Token (NET) standard specifically to address the gap left by existing fungible token standards like ERC-20, which inherently facilitate financial and speculative uses. The central objective of NETs is to create a fungible resource deliberately structured to resist economic incentives and avoid integration with financial infrastructure.

The intentional omission of financial mechanisms such as `approve()`, `transferFrom()`, and related events (`Transfer`, `Approval`) ensures NETs cannot be listed or traded via decentralized exchanges, thereby preventing their commodification. This structural incompatibility explicitly enforces non-economic usage patterns and discourages speculative accumulation.

Implementing built-in decay mechanisms reinforces impermanence and reduces utility as a store of value. By mandating that all NET holdings decrease deterministically over time, the standard inherently limits the attractiveness of tokens for economic activities.

---

## Backwards Compatibility

This ERC is not compatible with ERC-20 or any other financial token standard.

---

## Reference Implementation

```solidity
// SPDX-License-Identifier: CC0-1.0
pragma solidity ^0.8.30;

import "@openzeppelin/contracts/utils/introspection/IERC165.sol";
import "@openzeppelin/contracts/utils/introspection/ERC165.sol";

/// @dev This interface MAY be used with ERC-165 for discoverability.
/// @dev Supporting ERC-165 is RECOMMENDED but NOT REQUIRED.
interface INET is IERC165 {
    /// @notice A short name to identify the resource
    /// @return name The name of the resource
    function name() external view returns (string memory name);
    
    /// @notice A short symbol to identify the resource
    /// @return symbol The symbol of the resource
    function symbol() external view returns (string memory symbol);
    
    /// @notice The number of decimals used to represent amounts
    /// @return decimals The number of decimals
    function decimals() external view returns (uint8 decimals);

    /// @notice The rate of decay of tokens at a location address
    /// @return decayRate The rate of linear decay in token-WEI per second
    function decayRate() external view returns (uint256 decayRate);
    
    /// @notice Returns the decayed amount of resource at a given address
    /// @param location The address to query
    /// @return amount The decayed amount currently attributed to the address
    function amountAt(address location) external view returns (uint256 amount);
    
    /// @notice Moves resource from the caller to another address
    /// @param to The address to receive the resource
    /// @param amount The amount to move
    function move(address to, uint256 amount) external;
    
    /// @notice Creates new resource and assigns it to the specified address
    /// @param to The address to receive the newly created resource
    /// @param amount The amount of resource to create
    function create(address to, uint256 amount) external;
    
    /// @notice Emitted when resource is moved between addresses
    /// @param from The address from which resource was moved
    /// @param to The address to which resource was moved
    /// @param amount The amount of resource moved
    event Moved(address indexed from, address indexed to, uint256 amount);
}

contract NETToken is INET, ERC165 {
    /// @inheritdoc INET
    string public constant NAME = "GenericResource";

    /// @inheritdoc INET
    string public constant SYMBOL = "GEN";

    /// @inheritdoc INET
    uint8 public constant DECIMALS = 18;

    /// @inheritdoc INET
    /// Sample setting - the decay rate can be any uint256
    uint256 public constant DECAY_RATE = 1e15;

    /// @notice The total raw (pre-decay) amount of resource across all accounts
    uint256 public totalAmount;

    /// @dev Stores the raw amount of resource assigned to each address
    mapping(address => uint256) internal rawAmounts;

    /// @dev Stores the last block timestamp when an address was updated
    mapping(address => uint256) internal lastUpdated;

    /// @inheritdoc INET
    function amountAt(address location) public view returns (uint256) {
        uint256 timeElapsed = block.timestamp - lastUpdated[location];
        uint256 raw = rawAmounts[location];

        uint256 decay = decayRate * timeElapsed;
        return decay >= raw ? 0 : raw - decay;
    }

    /// @inheritdoc INET
    function move(address to, uint256 amount) external {
        require(to != address(0), "Cannot move to zero address");
        require(to != address(this), "Cannot move to contract address");

        _update(msg.sender);
        _update(to);

        uint256 available = rawAmounts[msg.sender];
        require(amount <= available, "Insufficient amount");

        rawAmounts[msg.sender] -= amount;
        rawAmounts[to] += amount;

        emit Moved(msg.sender, to, amount);
    }

    /// @inheritdoc INET
    function create(address to, uint256 amount) external {
        require(to != address(0), "Cannot create to zero address");
        require(to != address(this), "Cannot create to contract address");

        _update(to);
        require(type(uint256).max - totalAmount >= amount, "Total amount overflow");

        rawAmounts[to] += amount;
        totalAmount += amount;

        emit Moved(address(0), to, amount);
    }

    /// @notice Public helper to trigger an update of caller’s decay-adjusted amount
    function updateSelf() external {
        _update(msg.sender);
    }

    /// @dev Internal function to apply decay and update a specific address
    /// @param location The address whose amount should be updated
    function _update(address location) internal {
        uint256 updatedAmount = amountAt(location);
        totalAmount = totalAmount - rawAmounts[location] + updatedAmount;
        rawAmounts[location] = updatedAmount;
        lastUpdated[location] = block.timestamp;
    }

    /// @notice Indicates support for the INET interface via ERC-165
    /// @param interfaceId The interface identifier, as specified in ERC-165
    /// @return true if the contract implements the specified interface
    function supportsInterface(bytes4 interfaceId) public view virtual override returns (bool) {
        return interfaceId == type(INET).interfaceId || super.supportsInterface(interfaceId);
    }
}
```

---

## Security Considerations

Because NETs are incompatible with economic infrastructure and have no approval or delegation mechanisms, risks such as phishing via approval are avoided. Open creation mechanisms MAY be protected by rate limiting or external governance to prevent denial-of-service via resource flooding. However, highly restrictive creation mechanisms SHOULD NOT be implemented, since doing so may create scarcity that compromises the non-economic character of the resource.

---

## Copyright

Copyright and related rights waived via [CC0](/LICENSE).