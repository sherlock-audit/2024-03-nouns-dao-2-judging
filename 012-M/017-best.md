Deep Opaque Chicken

medium

# Invalid client ID votes lower the rewards given to valid proposal voters

## Summary

Votes with invalid client IDs lower the rewards given to valid voters, and leave undistributed rewards funds in the rewards contract.

## Vulnerability Detail

When voting on a proposal, a user may submit votes with an invalid client ID. I’ll define an invalid client ID as a client ID that has not yet been registered via the NFT registration process, i.e. a client ID that is greater or equal to `Rewards.sol→nextClientId()`. These votes are counted when tallying the total number of votes and thus used to determine the per-vote reward, but they do not receive rewards as expected since there is no client registered with that ID to receive the rewards.

However, votes with an invalid client ID result in lowered per-vote rewards for the other voters. Therefore, a user can grief the rewards contract by voting with invalid client IDs, resulting in undistributed reward funds that must later by reconciled manually outside the contract.

Example:

This example mirrors the test case `ProposalRewards.t.sol->test_rewardSplitBetweenTwoClients()`

There are 2 valid registered client IDs, `1` and `2`. One proposal has taken place and rewards are updated.

Alice voted with a share of `8` and client ID `1`.

Bob voted with a share of `2` and client ID `2`.

Assuming the total eligible revenue for this vote is `15 ether` with a voting rewards bps of `0.5%`, the expected rewards are `0.06 ether` and `0.015 ether` respectively.

However, if another user Carol voted with a client ID of `3` and a voting share of `100`, the results are as follows:

Carol: `0 ether` 

Alice: `0.005454545454545448  ether`

Bob: `0.001363636363636362 ether`

This leaves approximately `0.07 ether` of rewards out of the `0.075 ether` that was generated through auctions that must be distributed manually. There is no method to distribute these rewards through the contract.

## Impact

Invalid client ID votes result in undistributed rewards that remain in the contract. This revenue reserved for proposal rewards must be distributed to valid voters manually outside of the contract logic, or will remain in the contract until the next rewards distributions leading to needlessly complex off-chain manual accounting and inefficient use of reward funds. 

## Code Snippet

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/client-incentives/Rewards.sol#L352-L387

## Proof of Concept

```solidity
    function test_badClientIdCanSpoilVoteReward() public {
        // cast 8 votes
        assertEq(nounsToken.getCurrentVotes(bidder1), 8);
        vote(bidder1, proposalId, 1, 'i support', clientId1);

        // cast 2 votes
        assertEq(nounsToken.getCurrentVotes(bidder2), 2);
        vote(bidder2, proposalId, 1, 'i support', clientId2);

        // bad client id casts 100 votes, reducing the reward for the other clients
        uint32 badClientId = rewards.nextTokenId();
        assertEq(nounsToken.getCurrentVotes(bidder3), 100);
        vote(bidder3, proposalId, 1, 'i support', badClientId);
        mineBlocks(VOTING_PERIOD);

        settleAuction();
        votingClientIds = [clientId1, clientId2, badClientId]
        rewards.updateRewardsForProposalWritingAndVoting({
            lastProposalId: proposalId,
            votingClientIds: votingClientIds
        });

        assertEq(rewards.clientBalance(clientId1), 0.005454545454545448 ether); // 15 eth * 0.5% * (8/110)
        assertEq(rewards.clientBalance(clientId2), 0.001363636363636362 ether); // 15 eth * 0.5% * (2/110)
        assertEq(rewards.clientBalance(badClientId), 0.0 ether); // bad client ID means no reward
    }
```

## Tool used

Manual Review

## Recommendation

Votes with invalid client IDs should not be considered when tallying the vote count used to determine the per-vote reward.
