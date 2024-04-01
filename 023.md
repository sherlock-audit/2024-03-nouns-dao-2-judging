Scruffy Fleece Sardine

high

# `OwnableUpgradeable` is not initialized in `NounsAuctionHousePreV2Migrations.sol`

## Summary
`OwnableUpgradeable` is not initialized in `NounsAuctionHousePreV2Migrations.sol`

## Vulnerability Detail

`NounsAuctionHousePreV2Migration.sol` is a contract for storage migration between V1 and V2 versions. It has `migrate()` function to acheive the purpose of storage migration and it can only be accessed by owner of contract.

```solidity
    function migrate() public onlyOwner {
        OldLayout storage oldLayout = _oldLayout();
        NewLayout storage newLayout = _newLayout();
        OldLayout memory oldLayoutCache = oldLayout;

        // Clear the old storage layout
        oldLayout.nouns = address(0);
        oldLayout.weth = address(0);
        oldLayout.timeBuffer = 0;
        oldLayout.reservePrice = 0;
        oldLayout.minBidIncrementPercentage = 0;
        oldLayout.duration = 0;
        oldLayout.auction = INounsAuctionHouse.Auction(0, 0, 0, 0, payable(0), false);

        // Populate the new layout from the cache
        newLayout.reservePrice = uint192(oldLayoutCache.reservePrice);
        newLayout.timeBuffer = uint56(oldLayoutCache.timeBuffer);
        newLayout.minBidIncrementPercentage = oldLayoutCache.minBidIncrementPercentage;
        newLayout.auction = INounsAuctionHouseV2.AuctionV2({
            nounId: uint96(oldLayoutCache.auction.nounId),
            clientId: 0,
            amount: uint128(oldLayoutCache.auction.amount),
            startTime: uint40(oldLayoutCache.auction.startTime),
            endTime: uint40(oldLayoutCache.auction.endTime),
            bidder: oldLayoutCache.auction.bidder,
            settled: oldLayoutCache.auction.settled
        });
    }
```

To be noted, `NounsAuctionHousePreV2Migrations.sol` has used openzeppelin's `OwnableUpgradeable` for owner access functionality in contract.

```solidity
contract NounsAuctionHousePreV2Migration is PausableUpgradeable, ReentrancyGuardUpgradeable, OwnableUpgradeable {
```

The issue here is that, the contract has not initialized the owner which must be done as inherited from `OwnableUpgradeable`.

Openzeppeline's `OwnableUpgradeable` specifically states to initialize below function in order to get the `onlyOwner` modifier functionality in contracts inheriting it.

```solidity
    /**
     * @dev Initializes the contract setting the deployer as the initial owner.
     */
    function __Ownable_init() internal initializer {
        __Context_init_unchained();
        __Ownable_init_unchained();
    }

    function __Ownable_init_unchained() internal initializer {
        address msgSender = _msgSender();
        _owner = msgSender;
        emit OwnershipTransferred(address(0), msgSender);
    }
```

Therefore, in current implementation it can be seen `__Ownable_init()` is not initlialized therefore the contract is ownerless since `OwnableUpgradeable` sets the deployer of contract as owner if it initialize the `__Ownable_init()`.

`migrate()` has made use of `onlyOwner()` modifier and it can not be called by contract deployer due to the owner address by default would be zero address. The function will create permanent deniel of service and will always revert if some other account calls it.

This breaks the core functionality of `NounsAuctionHousePreV2Migrations.sol` which is supposed to call `migrate()` by contract owner and contract deployer can't access it.

## Impact
The uninitlized `__Ownable_init()` of `OwnableUpgradeable`  in `NounsAuctionHousePreV2Migrations.sol` will make `_owner` address by default to zero address, Therefore `migrate()` function can not be called and it would break the core functionlity of contract. This will make `migrate()` unusable and the contract can not acheive its true purpose and in worst case it will result in redeployment of contract.

## Code Snippet
https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/NounsAuctionHousePreV2Migration.sol#L55

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/NounsAuctionHousePreV2Migration.sol#L26

## Tool used
Manual Review

## Recommendation
Initialize the `__Ownable_init()` in contract to make use of `onlyOwner()` functionality in contract so that `migrate()` can be accessed by contract deployer i.e owner without any issues.