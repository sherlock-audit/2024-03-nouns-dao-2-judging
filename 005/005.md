Cheerful Sand Tardigrade

medium

# Temporary DOS of payment delays for proposal rewards

## Summary

To redeem a client reward for an eligible proposal, anyone can call Rewards.updateRewardsForProposalWritingAndVoting(). This function allows users to provide how many proposals should be reviewed for reward distribution. In addition, there are checks to determine how frequently proposal rewards can be rewarded. Because of these two factors, it's possible for clients who should receive eligible proposals to be locked out of their funds longer than a week.

## Vulnerability Detail

The Rewards.updateRewardsForProposalWritingAndVoting() is a complex function that rewards clients with eligible proposal rewards. There are two aspects that are critical to this bug, the `lastProposalId` input argument and the eligibility proposal time check.

The first aspect to this bug is that any user can set how many proposals the function should review for rewards. The `lastProposalId` represents the last proposal to include in the rewards distribution. The updateRewardsForProposalWritingAndVoting() pulls the reviewed proposals via:

```solidity
NounsDAOTypes.ProposalForRewards[] memory proposals = nounsDAO.proposalDataForRewards(
    t.nextProposalIdToReward,
    lastProposalId, // AUDIT: user input. can be set to any accepted value up to the last proposal that ended
    votingClientIds 
);
```

This function will retrieve all proposals up to the lastProposalId. These proposals will then be processed to check if any are eligible for rewards. Once the number of eligible proposals is calculated, the function will then run the following checks:

```solidity
require(t.numEligibleProposals > 0, 'at least one eligible proposal');
if (t.numEligibleProposals < $.params.numProposalsEnoughForReward) {
    require(
        t.lastProposal.creationTimestamp > $.lastProposalRewardsUpdate + $.params.minimumRewardPeriod,
        'not enough time passed'
    );
}
$.lastProposalRewardsUpdate = uint40(t.lastProposal.creationTimestamp);
``` 

These checks are:

- Check there is at least one eligible proposal for rewards
- If the # of eligible proposals is below the numProposalsEnoughForReward, check that enough time has passed since the last reward was distributed.

We are most interested in the second check. This check enforces a waiting period for proposal reward distribution. If the minimumRewardPeriod ends up being longer than a week and no more eligible proposals are created, it's possible for the reward clients to be locked out of their funds till the minimum reward period is reached. 

Let's take the following example to see how this can be exploited. 

**Pre-reqs**:
- Imagine there are 10 proposals that are eligible for rewards.
- The minimumRewardPeriod is set to 14 days.
- Proposals take on average 7 days to be marked as completed.

**Steps**:
1) Client A votes in proposals 9 and 10. Both are eligible for rewards.
1) User A calls Rewards.updateRewardsForProposalWritingAndVoting() and sets the lastProposalId to the proposal 8 id. This leaves proposal 9 and 10 not processed.
2) In the next transaction, Client A calls Rewards.updateRewardsForProposalWritingAndVoting(). Because the reward distribution just occurred, the transaction will revert. Client A will then have to wait a week before enough time has passed to be rewarded.


## Impact

A client who is eligible for a proposal reward may have their proposal reward funds locked for a week or longer.

## Code Snippet

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/client-incentives/Rewards.sol?plain=1#L376-L383

## Tool used

Manual Review

## Recommendation

If a proposal reward is eligible to be redeemed, then it should be redeemed when Rewards.updateRewardsForProposalWritingAndVoting() is called. Do not allow users the ability to specify how many eligible proposals should be redeemed. Alternatively, remove the time restriction to reward eligible proposals.
