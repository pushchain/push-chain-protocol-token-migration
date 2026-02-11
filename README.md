# PUSH Token Migration System

This project implements a secure, two-phase token migration system for the PUSH token ecosystem, enabling users to migrate from the existing PUSH token to a new implementation. The project is built using Solidity and Foundry, with transparent upgradeable proxies for future extensibility.

## Details on Migration
The migration of tokens is from:
- **Old Token:** ERC20 Push Token on Ethereum [@0xf418588522d5dd018b425e472991e52ebbeeeeee](https://etherscan.io/token/0xf418588522d5dd018b425e472991e52ebbeeeeee)
- **New Token:** Native $PC Token on Push Chain. ( *Push Chain is a universal blockchain with a native token called PC*)

**Migration Amounts**
- Token migration is planned to be at a 1:15 ratio (1 Push Protocol token = 15 Push Chain tokens).
- The token holders on Ethereum will be required to lock their token in the `MigrationLocker` contract.
- The token holders will be able to release/claim their migrated tokens using the `MigrationRelease` contract on Push Chain.
- The release of tokens will be a two phase release:
a. **Instant Release:** Allows users to claim 50% of their migrated tokens ( 7.5 out of 15 ratio ) instantly.
b. **Vested Release:** Allows users to claim the remaininig 50% of their migrated token 90 days after the instant release.

**Migration Proofs**
- The verification of token locked is done using Merkle Proofs.
- When users lock their tokens, an event emission occurs which is recorded offchain.
- These emissions are then used to generate proofs for each deposit by a given address.
- Users can use these proofs later on Push Chain's `MigrationRelease` contract to claim their migration tokens.

---

## System Architecture

The migration system consists of two main components:

1. **MigrationLocker**: A contract that allows users to lock their PUSH tokens for migration. *To Deployed on Ethereum Mainnet*
2. **MigrationRelease**: A contract that enables whitelisted users to claim their migrated tokens in two phases. *To Deployed on Push Mainnet*

### Technical Stack

- **Solidity**: ^0.8.20
- **Framework**: Foundry, with Hardhat for deployments
- **Proxy Pattern**: OpenZeppelin Transparent Upgradeable Proxy
- **Verification Mechanism**: Merkle Tree for secure, gas-efficient verification

## Contract Details

### MigrationLocker.sol

The MigrationLocker contract is responsible for allowing users to lock their PUSH tokens as part of the migration process.

**Key Features:**
- Token locking mechanism with unique identifier generation
- Openzeppelin's Pausable library to prevent/allow locking - Owner Controlled
- Proper access control with Ownable2Step pattern
- Token burning capability for migrated tokens
- Emergency fund recovery functionality

**Main Functions:**
- `lock(uint _amount, address _recipient)`: Allows users to lock tokens for migration
- `burn(uint _amount)`: Burns tokens that have been successfully migrated
- `pause()`: Pauses the contract to prevent new locks
- `unpause()`: Unpauses the contract to allow new locks
- `initiateNewEpoch()`: Starts a new epoch for organizing locks
- `recoverFunds(address _token, address _to, uint _amount)`: Emergency function to recover funds

**Events:**
- `Locked(address caller, address recipient, uint amount, uint epoch)`: Emitted when tokens are locked

### Epoch System in MigrationLocker

The MigrationLocker contract uses an epoch-based system to organize token locks into time periods for efficient Merkle Tree generation.

It should be noted, however, that epochs are owner-controlled.

**How it Works:**
- Each epoch represents a specific time period for token locking
- The current epoch is recorded when users lock tokens via the `Locked` event
- Owners can start new epochs using `initiateNewEpoch()`
- Each epoch tracks its start block for off-chain event processing
- Epochs help organize locks into batches for systematic Merkle Tree generation and verification

This system ensures organized processing of token locks across different time periods.

**Expected Workflow**
1. Owner initiates LOCKING with `initiateNewEpoch()`, i.e., Epoch-1
2. Duration for how long EPOCH-1 will run is flexible and decided by owner, hence owner-controlled.
3. Users starts locking token in EPOCH-1. All such locks emit out `Locked(msg.sender, _recipient, _amount, 1)`. These event and params will be used for merkle proof creation.
4. After 30 days, for example, owner initiates pause() and triggers `initiateNewEpoch(). i.e., EPOCH-2.
5. Same locking cycle starts again, but now locks emits out `Locked(msg.sender, _recipient, _amount, 2)`.

---
### MigrationRelease.sol

The MigrationRelease contract manages the release of migrated tokens to eligible users based on Merkle proofs.

**Key Features:**
- Two-phase token release (instant + vested)
- Merkle Tree-based verification for gas efficiency
- Fixed allocation ratios for instant and vested portions
- Fair and transparent distribution mechanism
- Fund recovery safety mechanism

## Important Constants and Params

- `VESTING_PERIOD`: 90 days
- `INSTANT_RATIO`: 75 (interpreted as 7.5x)
- `VESTING_RATIO`: 75 (interpreted as 7.5x)

**Release Model:**
- **Instant Release**: 
a. As users provide proof of their fund-lock, 50% of the locked amount is immediately released.
b. Once released, we record the timestamp of instant release.
c. Only after 90 days of this timestamp, users can unlock their vested release.

- **Vested Release**: 
a. Vested release is the release that takes place 90 days after instant release. 
b. Merkle proof is still required but the 90 days check is additionally imposed.

**Main Functions:**
- `releaseInstant(address _recipient, uint _amount, uint _epoch, bytes32[] calldata _merkleProof)`: Claims instant portion
- `releaseVested(address _recipient, uint _amount, uint _epoch)`: Claims vested portion after vesting period
- `setMerkleRoot(bytes32 _merkleRoot)`: Updates the Merkle root for verification
- `addFunds()`: Adds funds to the contract for distribution
- `pause()`: Pauses the contract to prevent claims
- `unpause()`: Unpauses the contract to allow claims
- `recoverFunds(address _token, address _to, uint _amount)`: Emergency function to recover funds

**Events:**
- `ReleasedInstant(address indexed recipient, uint indexed amount, uint indexed epochId)`
- `ReleasedVested(address indexed recipient, uint indexed amount, uint indexed epochId)`
- `FundsAdded(uint indexed amount, uint indexed timestamp)`
- `MerkleRootUpdated(bytes32 indexed oldMerkleRoot, bytes32 indexed newMerkleRoot)`

## Merkle Tree Implementation

The system uses a Merkle Tree for efficient and secure verification of eligible claims. This approach significantly reduces gas costs compared to on-chain storage of all claims.

### Merkle Tree Generation Process

1. Events are collected from the MigrationLocker contract using `fetchAndStoreEvents.js`
2. Each lock event produces a leaf in the Merkle Tree with `(address, amount, epochId)` as parameters
3. The Merkle root is calculated and set in the MigrationRelease contract
4. Users can provide proofs to verify their eligibility when claiming tokens

### Technical Implementation

The utility scripts in the `script/utils` folder handle the Merkle tree generation process:

- **merkle.js**: Core Merkle tree implementation with functions to hash leaves, generate roots, create proofs, and verify claims using the (address, amount, epoch) format.
- **fetchAndStoreEvents.js**: Fetches all "Locked" events from the MigrationLocker contract, groups them by address and epoch, combines amounts for duplicate addresses within the same epoch, and saves the processed claims.
- **getRoot.js**: Simple utility that loads processed claims and generates the Merkle root for deployment to the MigrationRelease contract.
- **config.js**: Contains configuration settings for contract addresses, ABIs, and file paths used by the utility scripts.

Key functions in `merkle.js`:

- `hashLeaf(address, amount, epochId)`: Creates hashed leaves for the Merkle Tree
- `getRoot(claims)`: Generates the Merkle root from an array of claims
- `getProof(address, amount, epochId, claims)`: Generates a Merkle proof for a specific claim
- `verify(address, amount, epochId, claims)`: Verifies a claim against the Merkle Tree

## Security Considerations

### Claims Verification

The system uses the following security measures for claims verification:

1. **Double-claim prevention**: Both instant and vested claims track their status in mappings
2. **Tamper-proof verification**: Merkle Tree verification ensures users can only claim their allocated amounts
3. **Parameter binding**: The address, amount, and epochId must all match the Merkle proof
4. **Contract locking**: MigrationLocker can be locked to prevent new tokens from being locked

### Access Control

- Both contracts use OpenZeppelin's `Ownable2StepUpgradeable` for secure ownership management
- Critical functions are protected with `onlyOwner` modifier
- Both contracts can be paused/unpaused by owners to control operations

### Deployment Scripts

- `script/deploy/DeployLocker.s.sol`: Deploys the MigrationLocker contract
- `script/deploy/DeployRelease.s.sol`: Deploys the MigrationRelease contract and sets the Merkle root

## Usage Instructions

### Building the Project

```shell
forge build
```

### Testing the Project

```shell
forge test
```

## License

This project is licensed under the MIT License with Attribution - see the [LICENSE](LICENSE) file for details.

Any use of this code must include visible attribution to Push Protocol (https://push.org).
