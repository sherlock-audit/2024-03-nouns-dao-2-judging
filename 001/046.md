Breezy Marmalade Mouse

high

# Rewards can be stolen from other proposals and votes by extending auction revenue period with the help of bogus proposals

## Summary

Attacker controlling more than `proposalThresholdBPS` share of votes can systematically create auxiliary proposals to capture a bigger period of auction revenue as rewards for ones they are affiliated with.

It makes sense for a beneficiary of the currently ended `proposal 0` to front-run any next valid proposal creation with a dummy proposal, so it will be an anchor last proposal for `updateRewardsForProposalWritingAndVoting(anchorProposalId, ...)` call so all the auction revenue before the next valid proposal creation moment will go to `proposal 0`. This will systematically increase their rewards at the expense of other client id owners, i.e. stealing from them.

## Vulnerability Detail

The auction reward period can be artificially extended at the expense of other eligible proposals' rewards with the help of automated creation of bogus proposals. That's possible as the auction revenue period is derived from the user supplied last proposal id before proposal eligibility control. I.e. an attacker can front run next valid proposal creation with the creation of the fake proposal in order to maximize the auction revenue period for the proposal they have interest in (i.e. have any affiliation with client id owner of proposal or its votes).

Fake proposal doesn't have to be eligible: it can be heavily voted against, it can be vetoed. The only requirement is that it should have ended (`require(block.number > endBlock, ...`) as of the time of the reward allocation call. Note that `objectionPeriodEndBlock` is activated only for flipped proposals, which is highly unlikely for bogus ones, so `endBlock = max(proposals[i].endBlock, proposals[i].objectionPeriodEndBlock) = proposals[i].endBlock`, which is known as of time of proposal creation. In general it should be as late as possible, for example front running the next rival reward allocation proposal.

Schematic POC:
1) suppose a `proposal 1` that will drive much attention and votes is known to be published
2) attacker is affiliated to a beneficiary of a `clientId` used in creation of some already passed or almost passed not so popular `proposal 0`
3) attacker front runs `proposal 1` creation with creation of its own `proposal 2`. Both have the same creation time
4) attacker runs `updateRewardsForProposalWritingAndVoting(proposal_2_id, ids)` with `ids` covering all the activity of the yet uncovered period
5) Their `clientId` is getting all the auction revenue prior to `creation_time` undiluted by votes of `proposal 1` (which can be expected to be larger than ones of `proposal 0`)

Attacker has effectively stolen rewards from client id owner of `proposal 1` proposal and votes.

## Impact

The attack can be carried deterministically, there are no direct prerequisites, while the total cost is additional proposal creation gas only. The probability of the mentioned setup occurring so the attack will make sense can be estimated as medium as the situation is pretty typical. Auction period manipulation has substantial impact on rewards calculation, so the material impact on the other client id owners' reward revenue is high.

Likelihood: Medium + Impact: High = Severity: High.

## Code Snippet

The revenue allocated for reward distribution is based on the creation time of the `t.lastProposal`:

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/client-incentives/Rewards.sol#L332-L342

```solidity
        t.firstAuctionIdForRevenue = $.nextProposalRewardFirstAuctionId;
        (uint256 auctionRevenue, uint256 lastAuctionIdForRevenue) = getAuctionRevenue({
            firstNounId: t.firstAuctionIdForRevenue,
>>          endTimestamp: t.lastProposal.creationTimestamp
        });
        $.nextProposalRewardFirstAuctionId = uint32(lastAuctionIdForRevenue) + 1;

        require(auctionRevenue > 0, 'auctionRevenue must be > 0');

        t.proposalRewardForPeriod = (auctionRevenue * $.params.proposalRewardBps) / 10_000;
        t.votingRewardForPeriod = (auctionRevenue * $.params.votingRewardBps) / 10_000;
```

Which is user supplied `lastProposalId` as `proposalDataForRewards` doesn't filter the proposals, returning all the results sequentially:

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/client-incentives/Rewards.sol#L319-L330

```solidity
        require(lastProposalId <= nounsDAO.proposalCount(), 'bad lastProposalId');
        require(lastProposalId >= t.nextProposalIdToReward, 'bad lastProposalId');
        require(isSortedAndNoDuplicates(votingClientIds), 'must be sorted & unique');

>>      NounsDAOTypes.ProposalForRewards[] memory proposals = nounsDAO.proposalDataForRewards(
            t.nextProposalIdToReward,
>>          lastProposalId,
            votingClientIds
        );
        $.nextProposalIdToReward = lastProposalId + 1;

>>      t.lastProposal = proposals[proposals.length - 1];
```

I.e. here `t.lastProposal` doesn't have to be eligible, it can be any fake proposal which will be omitted later on in the logic. But since `t.lastProposal.creationTimestamp` is used for `getAuctionRevenue()` it will have an impact of capturing a bigger share of auction profits:

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/client-incentives/Rewards.sol#L531-L542

```solidity
    function getAuctionRevenue(
        uint256 firstNounId,
        uint256 endTimestamp
    ) public view returns (uint256 sumRevenue, uint256 lastAuctionId) {
        INounsAuctionHouseV2.Settlement[] memory s = auctionHouse.getSettlementsFromIdtoTimestamp(
            firstNounId,
>>          endTimestamp,
            true
        );
        sumRevenue = sumAuctions(s);
        lastAuctionId = s[s.length - 1].nounId;
    }
```

Allowing the attacker to maximize the auction revenue period for rewards calculation as the whole period from `startId` to `maxId` will be used, while `endTimestamp` was manipulated to be close to the current moment:

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/NounsAuctionHouseV2.sol#L452-L480

```solidity
    function getSettlementsFromIdtoTimestamp(
        ...
    ) public view returns (Settlement[] memory settlements) {
        uint256 maxId = auctionStorage.nounId;
        require(startId <= maxId, 'startId too large');
        settlements = new Settlement[](maxId - startId + 1);
        uint256 actualCount = 0;
        SettlementState memory settlementState;
>>      for (uint256 id = startId; id <= maxId; ++id) {
            settlementState = settlementHistory[id];

            if (skipEmptyValues && settlementState.blockTimestamp <= 1) continue;

            // don't include the currently auctioned noun if it hasn't settled
            if ((id == maxId) && (settlementState.blockTimestamp <= 1)) continue;

>>          if (settlementState.blockTimestamp > endTimestamp) break;

            settlements[actualCount] = Settlement({
                blockTimestamp: settlementState.blockTimestamp,
                amount: uint64PriceToUint256(settlementState.amount),
                winner: settlementState.winner,
                nounId: id,
                clientId: settlementState.clientId
            });
            ++actualCount;
        }
```

## Tool used

Manual Review

## Recommendation

Consider basing the check on the last eligible proposal by filtering them out in `proposalDataForRewards()`, which isn't used outside reward logic:

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

This way `getAuctionRevenue()` will be called with the `endTimestamp` based on the last proposal after filtration.