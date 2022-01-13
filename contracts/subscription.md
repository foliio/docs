# Subscription

### Design Considerations&#x20;

In general a subscription encapsulates two types of data: metadata and state.

1. Metadata - Encompasses its 'type' and other static attributes. It defines what the subscription grants access to. For any given subscription type, the metadata is fungible.&#x20;
2. State - Instance specific data such as current owner, expiration date, instance id, and a pointer to the metadata. The state of a subscription is non-fungible and unique for every subscription instance.

In this article we will refer to 'subscription type' a lot. Think of 'subscription type' as a overarching group of permissions. By owning a subscription, an account is granted the set of permissions provided by the subscription.&#x20;

As an example consider the following as two subscriptions types: 'Trial' and 'Standard'

* A **Trial** subscription could have a default expiration state and is non-renewable.&#x20;
* A **Standard** subscription could also have its own default expiration state but be renewable

### Requirements

* Ability to maintain various subscription types from a single contract&#x20;
* A single account should be able to hold any number of unique subscription types&#x20;
* A single account should not be able to hold multiple subscriptions of the same type
* Each subscription must have its own state

### Implementation&#x20;

When designing the subscription contract a few approaches were considered. [EIP-1337](https://eips.ethereum.org/EIPS/eip-1337) was similar  to what we were looking for but wasn't a 'pure' blockchain implementation. It was more of a notification system that would tie into existing infrastructure. Since we needed our tokens to have unique state NFT standards such as ERC721 and ERC1155. Due to our requirements, ERC721 wouldn't be optimal as we want to control multiple tokens from a single contract.

ERC1155 almost completely fit our needs. It allows for mixing fungible / non-fungible tokens from the same contract. To hit our requirements some modifications were made to how we track, store, and locate account balances. Subscription.sol is forked from [ERC1155.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC1155/ERC1155.sol) with the following changes.

#### \_balances

ERC1155 tracks balances by matching a tokenId --> address --> amount

```
// Mapping from token ID to account balances
mapping(uint256 => mapping(address => uint256)) private _balances;
```

Subscription.sol matches a subscriptionType --> address --> subscriptionId. Using the subscriptionId we can lookup the state of the specific instance.

```
// Map a subscription type --> address --> subscriptionId.
mapping(uint256 => mapping(address => uint256)) private _subscriptions;

// Mapping from subscriptionId --> state.
mapping(uint256 => Subscriptions.SubscriptionState) private _subscriptionStates;
```

The subscription state provides everything we need to know about the instance:

```
  struct SubscriptionState {
    // The time stamp since the unix epoch in which this subscription was minted.
    uint256 start;

    // Timestamp when this subscription will expire.
    uint256 end;
    
    // ID of the fungible subscription type.
    uint256 typeId;

    // Unique ID of the subscription
    uint256 id;

    // Total amount of time that has been applied to the subscription over its lifetime.
    uint256 totalAppliedTime;
  }
```

We have removed \_balances completely and as a result, an account can have a balance of 0 or 1. Either a subscription is owned, or it is not. To determine if an account hold a subscription type, we check if there is a non-zero value in the subscriptions mapping. If it is 0 there is no balance. Otherwise it's always a balance of 1 and we know the instanceId of the token.&#x20;

```
    function balanceOf(address account, uint256 id) public view virtual override returns (uint256) {
        require(account != address(0), "ERC1155: balance query for the zero address");
        if(_subscriptions[id][account] == 0) return 0;
        return 1;
    }
```
