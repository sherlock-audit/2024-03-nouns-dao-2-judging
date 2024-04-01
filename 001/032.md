Breezy Marmalade Mouse

medium

# Rewards can be allocated for less than minimal reward period with the help of bogus proposal

## Summary

Anyone controlling more than `proposalThresholdBPS` share of votes can ignore the minimum reward period to allocate the rewards for the proposal they benefit from.

## Vulnerability Detail

The `minimumRewardPeriod` check can be surpassed with the help of any bogus proposal since the check comes before proposal eligibility control. I.e. once current `block.timestamp` is big enough an attacker can just create a proposal only to get past the check by later using that proposal as a anchor for reward distribution call.

This proposal doesn't have to be eligible (can be heavily voted against, can even be vetoed), the only requirement is that it should have ended (`require(block.number > endBlock, ...` check), i.e. the attacker needs to create it beforehand, but the timing is exact and is known in advance, so this can be done all the times. Note that `objectionPeriodEndBlock` is activated only for flipped proposals, which is higly unlikely for bogus ones, so `endBlock = max(proposals[i].endBlock, proposals[i].objectionPeriodEndBlock) = proposals[i].endBlock`, which is known as of time of proposal creation.

## Impact

The attack can be carried deterministically, there are no direct prerequisites, while the total cost is additional proposal creation gas cost only (it's not compensated). Surpassing the `minimumRewardPeriod` check can be used for gaming reward allocation, i.e. placing good auctions to the proposal attacker benefit ahead of other proposals, who have to wait for the period to expire.

I.e. in a situation when there are only few active proposals that are about to end, while attacker's one is ended, `$.params.numProposalsEnoughForReward` isn't met, while current auctions have good revenue, the attacker can steal the rewards from other proposals by not waiting for them with the help of the anchor bogus one. Overall, this setup has medium probability of occurring, while the reward loss can be material, so the impact can be estimated as medium as well.

Likelihood: Medium + Impact: Medium = Severity: Medium.

## Code Snippet

`t.lastProposal.creationTimestamp` comes from user supplied `lastProposalId`:

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/client-incentives/Rewards.sol#L370-L383

```solidity
        //// Check that distribution is allowed:
        //// 1. At least one eligible proposal.
        //// 2. One of the two conditions must be true:
        //// 2.a. Number of eligible proposals is at least `numProposalsEnoughForReward`.
        //// 2.b. At least `minimumRewardPeriod` seconds have passed since the last update.

        require(t.numEligibleProposals > 0, 'at least one eligible proposal');
        if (t.numEligibleProposals < $.params.numProposalsEnoughForReward) {
            require(
>>              t.lastProposal.creationTimestamp > $.lastProposalRewardsUpdate + $.params.minimumRewardPeriod,
                'not enough time passed'
            );
        }
        $.lastProposalRewardsUpdate = uint40(t.lastProposal.creationTimestamp);
```

With the only checks being `nounsDAO.proposalCount() >= lastProposalId >= t.nextProposalIdToReward`, since `proposalDataForRewards` doesn't filter the proposals, returning all the results sequentially:

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/client-incentives/Rewards.sol#L319-L330

```solidity
>>      require(lastProposalId <= nounsDAO.proposalCount(), 'bad lastProposalId');
>>      require(lastProposalId >= t.nextProposalIdToReward, 'bad lastProposalId');
        require(isSortedAndNoDuplicates(votingClientIds), 'must be sorted & unique');

>>      NounsDAOTypes.ProposalForRewards[] memory proposals = nounsDAO.proposalDataForRewards(
            t.nextProposalIdToReward,
            lastProposalId,
            votingClientIds
        );
        $.nextProposalIdToReward = lastProposalId + 1;

>>      t.lastProposal = proposals[proposals.length - 1];
```

User that can create proposals can also anyhow time the bogus one:

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/governance/NounsDAOProposals.sol#L871-L881

```solidity
    function createNewProposal(
        ...
    ) internal returns (NounsDAOTypes.Proposal storage newProposal) {
        uint64 updatePeriodEndBlock = SafeCast.toUint64(block.number + ds.proposalUpdatablePeriodInBlocks);
        uint256 startBlock = updatePeriodEndBlock + ds.votingDelay;
>>      uint256 endBlock = startBlock + ds.votingPeriod;
```

## Tool used

Manual Review

## Recommendation

Consider basing the check on the last eligible proposal, not just the user supplied one, e.g. by filtering them out in `proposalDataForRewards()`, which isn't used outside reward logic:

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/governance/NounsDAOProposals.sol#L719-L737

```diff
    function proposalDataForRewards(
        NounsDAOTypes.Storage storage ds,
        uint256 firstProposalId,
        uint256 lastProposalId,
+       uint16 proposalEligibilityQuorumBps,
        uint32[] calldata votingClientIds
    ) internal view returns (NounsDAOTypes.ProposalForRewards[] memory) {
        require(lastProposalId >= firstProposalId, 'lastProposalId >= firstProposalId');
        uint256 numProposals = lastProposalId - firstProposalId + 1;
        NounsDAOTypes.ProposalForRewards[] memory data = new NounsDAOTypes.ProposalForRewards[](numProposals);

        NounsDAOTypes.Proposal storage proposal;
        uint256 i;
        for (uint256 pid = firstProposalId; pid <= lastProposalId; ++pid) {
            proposal = ds._proposals[pid];
+           if (proposal.canceled || proposals.forVotes < (proposals.totalSupply * proposalEligibilityQuorumBps) / 10_000) continue;

            NounsDAOTypes.ClientVoteData[] memory c = new NounsDAOTypes.ClientVoteData[](votingClientIds.length);
            for (uint256 j; j < votingClientIds.length; ++j) {
                c[j] = proposal.voteClients[votingClientIds[j]];
            }
```

In order to minimize the changes this suggestion is shared with another issues.