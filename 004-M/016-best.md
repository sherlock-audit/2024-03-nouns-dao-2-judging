Breezy Marmalade Mouse

medium

# Setting `nextProposalIdToReward` to the current proposal artificially lowers first proposal and voting rewards

## Summary

`nextProposalIdToReward` is now initialized on Rewards deployment to be the last proposal id known to the system. Since the client id is being introducing with the current upgrade, all the actions of this proposal, i.e. proposal itself and its votes prior to upgrade, will guaranteed to have zero client id. This artificially dilutes the rewards for all proposals and votes of the first successful `updateRewardsForProposalWritingAndVoting()` run.

## Vulnerability Detail

On the one hand votes after upgrade can have `clientId`. But it can be a small enough part of total votes for the proposal. In the same time rewards to be twisted rather substantially: the proposal is guaranteed not to have `clientId` and also lots of corresponding Nouns auctions will be missed as `nextProposalRewardFirstAuctionId == currentNounId`, i.e. all in-between that proposal creation time and current nouns id. This differs from a typical situation after the upgrade when all the auctioned nouns during lifecycle of a proposal contribute to its reward.

## Impact

Having simultaneously the proposal and its votes adding to the denominator of the corresponding rewards calculations, while the numerator being reduced because the corresponding auctions will be missed, will lower the first distribution rewards. This is a loss for the corresponding client id owners with regard to the stated reward rules.

The only prerequisite is having the last proposal live for some time before upgrade, which is highly likely as tuning upgrade to minimize the distance from the last proposal creation time is burdensome. The loss of rewards is proportional to this time and can be big enough if the last proposal is already ended when upgrade happens (in this case all the actions linked to it will have zero client id).

Likelihood: High + Impact: Low = Severity: Medium.

## Code Snippet

`nextProposalIdToReward` is set to the last currently active proposal:

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/script/Rewards/DeployRewardsBase.s.sol#L28-L38

```solidity
        rewards = RewardsDeployer.deployRewards({
            dao: dao,
            admin: admin,
            auctionHouse: address(auctionHouse),
            erc20: ethToken,
>>          nextProposalIdToReward: uint32(dao.proposalCount()),
            nextAuctionIdToReward: currentNounId,
            nextProposalRewardFirstAuctionId: currentNounId,
            rewardParams: params,
            descriptor: address(new NounsClientTokenDescriptor())
        });
```

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/governance/NounsDAOProposals.sol#L81-L97

```solidity
    function propose(
        ...
    ) internal returns (uint256) {
        uint256 adjustedTotalSupply = ds.adjustedTotalSupply();
        uint256 proposalThreshold_ = checkPropThreshold(
            ds,
            ds.nouns.getPriorVotes(msg.sender, block.number - 1),
            adjustedTotalSupply
        );
        checkProposalTxs(txs);
        checkNoActiveProp(ds, msg.sender);

        ds.proposalCount = ds.proposalCount + 1;
>>      uint32 proposalId = SafeCast.toUint32(ds.proposalCount);
```

This proposal doesn't have client id, have actual creation time in the past, but its creation time slot will be empty as it's introduced only [during the upgrade](https://github.com/nounsDAO/nouns-monorepo/pull/826/files?file-filters%5B%5D=.sol&show-deleted-files=true&show-viewed-files=true#diff-79479440adb89c0b2d6068d98f4c483b0e39c33f00395596d04f32d2707841d8R896):

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/governance/NounsDAOProposals.sol#L871-L898

```solidity
    function createNewProposal(
        ...
    ) internal returns (NounsDAOTypes.Proposal storage newProposal) {
        ...
        newProposal.totalSupply = adjustedTotalSupply;
        newProposal.creationBlock = SafeCast.toUint32(block.number);
>>      newProposal.creationTimestamp = uint32(block.timestamp);  // @audit this line is introduced with the upgrade
        newProposal.updatePeriodEndBlock = updatePeriodEndBlock;
    }
```

The main effect the inclusion of the current proposal brings in is rewards dilution for the subsequent proposals.

Having `t.lastProposal.creationTimestamp == 0` makes it not possible to immediately run `updateRewardsForProposalWritingAndVoting(dao.proposalCount(), ...)` to get rid of this old proposal as it will be `auctionRevenue == 0` since `getSettlementsFromIdtoTimestamp()` will return empty set:

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/client-incentives/Rewards.sol#L333-L339

```solidity
        (uint256 auctionRevenue, uint256 lastAuctionIdForRevenue) = getAuctionRevenue({
            firstNounId: t.firstAuctionIdForRevenue,
            endTimestamp: t.lastProposal.creationTimestamp
        });
        $.nextProposalRewardFirstAuctionId = uint32(lastAuctionIdForRevenue) + 1;

>>      require(auctionRevenue > 0, 'auctionRevenue must be > 0');
```

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/NounsAuctionHouseV2.sol#L452-L470

```solidity
    function getSettlementsFromIdtoTimestamp(
        uint256 startId,
>>      uint256 endTimestamp,
        bool skipEmptyValues
    ) public view returns (Settlement[] memory settlements) {
        uint256 maxId = auctionStorage.nounId;
        require(startId <= maxId, 'startId too large');
        settlements = new Settlement[](maxId - startId + 1);
        uint256 actualCount = 0;
        SettlementState memory settlementState;
        for (uint256 id = startId; id <= maxId; ++id) {
            settlementState = settlementHistory[id];

            if (skipEmptyValues && settlementState.blockTimestamp <= 1) continue;

            // don't include the currently auctioned noun if it hasn't settled
            if ((id == maxId) && (settlementState.blockTimestamp <= 1)) continue;

>>          if (settlementState.blockTimestamp > endTimestamp) break;
```

## Tool used

Manual Review

## Recommendation

Consider skipping the current proposal, e.g.:

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/script/Rewards/DeployRewardsBase.s.sol#L28-L38

```diff
        rewards = RewardsDeployer.deployRewards({
            dao: dao,
            admin: admin,
            auctionHouse: address(auctionHouse),
            erc20: ethToken,
-           nextProposalIdToReward: uint32(dao.proposalCount()),
+           nextProposalIdToReward: uint32(dao.proposalCount() + 1),
            nextAuctionIdToReward: currentNounId,
            nextProposalRewardFirstAuctionId: currentNounId,
            rewardParams: params,
            descriptor: address(new NounsClientTokenDescriptor())
        });
```