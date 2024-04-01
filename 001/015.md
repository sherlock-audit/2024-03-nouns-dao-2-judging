Breezy Marmalade Mouse

medium

# Reward recording can be manipulated to steal from other proposers and voters

## Summary

On the one hand there is a volatility in realized auction prices, and even no bid auction settlements are recorded (i.e. there can be any number of zero amount records in the settlement history). On the other revenue structure isn't a part of rewards distribution criteria (any positive revenue is deemed correct for reward distribution). This way a malicious caller can run `updateRewardsForProposalWritingAndVoting()` to allocate the minimized reward to proposals of other participants, up to only one auction revenue allocated to dozens proposals and hundreds of votes.

I.e. the ability to pinpoint the proposal reward to a particular set of auctions by optimizing the `updateRewardsForProposalWritingAndVoting()` call time allows to depress the rewards for all other actors.

## Vulnerability Detail

For example, when after some days with no bids for the auctioned Nouns the first sale came and current proposal beneficiary runs `updateRewardsForProposalWritingAndVoting(lastProposalId, ...)` with `lastProposalId` being the previous currently ended proposal just before their own. This way this one Noun sale revenue will be allocated to the proposals up to `lastProposalId` and the caller will get rid of these proposals so they won't participate in the further profit sharing, so caller's further proposal will receive undiluted revenue share.

Same example can be extended to attacker calling reward distribution on observing that Nouns price goes up to allocate low price period to proposals and votes of others. On the other hand, when price drops the attacker might want to allocate higher prices to the proposal they benefit from, cutting off the subsequent ones. In other words any minimal revenue is now enough for reward recording, which is immutable, can be run by anyone, so it is a race condition setup.

In the same time value of proposals and votes has no link to the current Nouns price, i.e. shouldn't depend on how successful are auctions right now. Ideally the reward to be based on some long term moving average of the auction revenue, not on the late set of auctions.

## Impact

Currently there is a possibility for reward revenue manipulation proportionally to the volatility of Nouns price.

The prerequisite is price volatility, but that is quite typical. Revenue share impact is proportional to it, so there is a medium probability of medium impact (typical volatility is frequently available, but the resulting reward misallocation is limited) or low probability of high impact (but sometimes there are substantial price movements, including price impacting Nouns forks, and executing the attack yields more in that periods), i.e. medium severity.

Likelihood: Medium/Low + Impact: Medium/High = Severity: Medium.

## Code Snippet

There is no control on how many prices are contributing to the current reward allocation:

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/client-incentives/Rewards.sol#L333-L342

```solidity
        (uint256 auctionRevenue, uint256 lastAuctionIdForRevenue) = getAuctionRevenue({
            firstNounId: t.firstAuctionIdForRevenue,
            endTimestamp: t.lastProposal.creationTimestamp
        });
        $.nextProposalRewardFirstAuctionId = uint32(lastAuctionIdForRevenue) + 1;

>>      require(auctionRevenue > 0, 'auctionRevenue must be > 0');

>>      t.proposalRewardForPeriod = (auctionRevenue * $.params.proposalRewardBps) / 10_000;
>>      t.votingRewardForPeriod = (auctionRevenue * $.params.votingRewardBps) / 10_000;
```

Even one successful auction is enough:

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/client-incentives/Rewards.sol#L694-L698

```solidity
    function sumAuctions(INounsAuctionHouseV2.Settlement[] memory s) internal pure returns (uint256 sum) {
        for (uint256 i = 0; i < s.length; ++i) {
            sum += s[i].amount;
        }
    }
```

I.e. arbitrary low amount of price data points can form rewards for the arbitrary high number of proposal and votes.

## Tool used

Manual Review

## Recommendation

Consider adding a control parameter, for example `minAuctionsPerProposal`, and requiring that the number of non-zero auctions to be greater than `$.minAuctionsPerProposal * t.numEligibleProposals`, e.g.:

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/client-incentives/Rewards.sol#L333-L336

```diff
-       (uint256 auctionRevenue, uint256 lastAuctionIdForRevenue) = getAuctionRevenue({
+       (uint256 auctionRevenue, uint256 lastAuctionIdForRevenue, uint256 successfulAuctions) = getAuctionRevenue({
            firstNounId: t.firstAuctionIdForRevenue,
            endTimestamp: t.lastProposal.creationTimestamp
        });
```

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/client-incentives/Rewards.sol#L531-L542

```diff
    function getAuctionRevenue(
        uint256 firstNounId,
        uint256 endTimestamp
-   ) public view returns (uint256 sumRevenue, uint256 lastAuctionId) {
+   ) public view returns (uint256 sumRevenue, uint256 lastAuctionId, uint256 successfulAuctions) {
        INounsAuctionHouseV2.Settlement[] memory s = auctionHouse.getSettlementsFromIdtoTimestamp(
            firstNounId,
            endTimestamp,
            true
        );
-       sumRevenue = sumAuctions(s);
+       (sumRevenue, successfulAuctions) = sumAuctions(s);
        lastAuctionId = s[s.length - 1].nounId;
    }
```

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/client-incentives/Rewards.sol#L694-L698

```diff
    function sumAuctions(INounsAuctionHouseV2.Settlement[] memory s) internal pure returns (uint256 sum, uint256 numAuctions) {
        for (uint256 i = 0; i < s.length; ++i) {
+           if (s[i].amount > 0) {
                sum += s[i].amount;
+               numAuctions++;
+           }
        }
    }
```

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/client-incentives/Rewards.sol#L370-L382

```diff
        //// Check that distribution is allowed:
        //// 1. At least one eligible proposal.
        //// 2. One of the two conditions must be true:
        //// 2.a. Number of eligible proposals is at least `numProposalsEnoughForReward`.
        //// 2.b. At least `minimumRewardPeriod` seconds have passed since the last update.

        require(t.numEligibleProposals > 0, 'at least one eligible proposal');
+       require(successfulAuctions >= $.minAuctionsPerProposal * t.numEligibleProposals, 'min Nouns sold per proposal');
        if (t.numEligibleProposals < $.params.numProposalsEnoughForReward) {
            require(
                t.lastProposal.creationTimestamp > $.lastProposalRewardsUpdate + $.params.minimumRewardPeriod,
                'not enough time passed'
            );
        }
```