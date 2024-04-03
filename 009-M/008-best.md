Interesting Coal Octopus

medium

# Voter can be rewarded for 0 votes in proposals

## Summary
Voter can be rewarded if `proposals.length` > 1 and if he submitted at least 1 vote in 1 proposal.
## Vulnerability Detail
Attack scenario:
1. There is 3 proposals: for first Amelie submitted 0 votes, for second - 1, for third - 0.
2. `updateRewardsForProposalWritingAndVoting()` is called with `uint32[] calldata votingClientIds` as a parameter.
3. Amelie passed the `require(didClientIdHaveVotes[i], ' ');` because she had 1 vote in second proposal.
4. Amelie is rewarded for all proposals, even in those where she actually did not participate.

Let's look at how this can happen step by step:

1. Voter can cast 0 votes in proposal, as there is no check if `votes == 0`. Then he will be added in array `ds._proposals[proposalId].voteClients[clientId]`, that will be used for rewards distribution:
```solidity
function castVoteInternal(
        NounsDAOTypes.Storage storage ds,
        address voter,
        uint256 proposalId,
        uint8 support,
        uint32 clientId
    ) internal returns (uint96 votes) {
        NounsDAOTypes.ProposalState proposalState = ds.stateInternal(proposalId);

        if (proposalState == NounsDAOTypes.ProposalState.Active) {
            votes = castVoteDuringVotingPeriodInternal(ds, proposalId, voter, support);
        } else if (proposalState == NounsDAOTypes.ProposalState.ObjectionPeriod) {
            if (support != 0) revert CanOnlyVoteAgainstDuringObjectionPeriod();
            votes = castObjectionInternal(ds, proposalId, voter);
        } else {
            revert('NounsDAO::castVoteInternal: voting is closed');
        }

        NounsDAOTypes.ClientVoteData memory voteData = ds._proposals[proposalId].voteClients[clientId];
        ds._proposals[proposalId].voteClients[clientId] = NounsDAOTypes.ClientVoteData({
            votes: uint32(voteData.votes + votes),
            txs: voteData.txs + 1
        });
    }
```
2. Before rewards distribution, function `getVotingClientIds()` is called to make sure that all `clientId` are eligible for rewards. But there is a problem: `sumVotes[i]` doesnt guarantees, that user has votes in ALL proposals. If there was 3 proposals, `sumVotes[i]` must be at least = 3. Check `if (sumVotes[i] > 0)` is wrong, user with zero votes in all proposals except for one will be unfairy added in `nonZeroClientIds[]` array:
```solidity
/**
     * @notice Returns the clientIds that are needed to be passed as a parameter to updateRewardsForProposalWritingAndVoting
     * @dev This is not meant to be called onchain because it may be very gas intensive.
     */
    function getVotingClientIds(uint32 lastProposalId) public view returns (uint32[] memory) {
        RewardsStorage storage $ = _getRewardsStorage();

        uint256 numClientIds = nextTokenId();
        uint32[] memory allClientIds = new uint32[](numClientIds);
        for (uint32 i; i < numClientIds; ++i) {
            allClientIds[i] = i;
        }
        NounsDAOTypes.ProposalForRewards[] memory proposals = nounsDAO.proposalDataForRewards(
            $.nextProposalIdToReward,
            lastProposalId,
            allClientIds
        );

        uint32[] memory sumVotes = new uint32[](numClientIds);
        for (uint256 i; i < proposals.length; ++i) {                //proposals
            for (uint256 j; j < numClientIds; ++j) {                //voters for proposals
                sumVotes[j] += proposals[i].voteData[j].votes;      //sumVotes[i] = 0 + 1 + 0 = 1
            }
        }

        uint256 idx;
        uint32[] memory nonZeroClientIds = new uint32[](numClientIds);
        for (uint32 i; i < numClientIds; ++i) {
            if (sumVotes[i] > 0) nonZeroClientIds[idx++] = i;       //sumVotes[i] = 1
        }

        assembly {
            mstore(nonZeroClientIds, idx)
        }

        return nonZeroClientIds;
    }
```
`sumVotes[AmelieCliendId]` = 
`proposals[0].voteData[AmelieCliendId].votes = 0` +
`proposals[1].voteData[AmelieCliendId].votes = 1` + 
`proposals[2].voteData[AmelieCliendId].votes = 0` = 1

3. There is additional check in `Rewards.sol` to prevent rewarding users with zero votes - if `didClientIdHaveVotes[AmelieCliendId]` is false, then function will revert. But the end of the loop, `didClientIdHaveVotes[AmelieCliendId]` will be true. This is because in the second iteration the condition `votes > 0` is true (because `votes` is 1), and thus the expression `didClientIdHaveVotes[AmelieCliendId] || votes > 0` became `true || true`, which is `true`. After that, `didClientIdHaveVotes[AmelieCliendId]` was overwritten to `true`. In the third iteration, the condition `votes > 0` was `false` (because `votes` is 0), but because `didClientIdHaveVotes[AmelieCliendId]` was already `true`, the expression `true || false` still remains `true`. Thus, `didClientIdHaveVotes[AmelieCliendId]` remains `true` at the end of the loop:
```solidity
bool[] memory didClientIdHaveVotes = new bool[](votingClientIds.length);

        for (uint256 i; i < proposals.length; ++i) {
          //...
            uint256 votesInProposal;
            NounsDAOTypes.ClientVoteData[] memory voteData = proposals[i].voteData;
            for (uint256 j; j < votingClientIds.length; ++j) {
                clientId = votingClientIds[j];
                uint256 votes = voteData[j].votes;
                didClientIdHaveVotes[j] = didClientIdHaveVotes[j] || votes > 0;   // votes = [0, 1, 0]
                if (clientId != 0 && clientId <= t.maxClientId) {
                    m.inc(clientId, votes * t.rewardPerVote);
                }
                votesInProposal += votes;
            }
           //...
        }

        for (uint256 i = 0; i < didClientIdHaveVotes.length; ++i) {
            require(didClientIdHaveVotes[i], 'all clientId must have votes');
        }
```
Malicious Amelie passed all checks and received rewards for 3 proposals, however she actually participate only in 1.
## Impact
Malicious voter can steal funds due to incorrect rewards distibution. This will happen with a high probability because `updateRewardsForProposalWritingAndVoting()` is very gas expensive and not going to be called frequently, after every proposal, thus `proposals.length` can be = 3, 5 or more. 
## Code Snippet
[https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/governance/NounsDAOVotes.sol#L193]()
[https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/client-incentives/Rewards.sol#L518]()
[https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/client-incentives/Rewards.sol#L425-L427]()
## Tool used

Manual Review

## Recommendation
Do not allow voters to submit zero votes in `NounsDAOVotes.castVoteInternal()`:
```diff
+          require(votes != 0, 'NounsDAO::castVoteInternal: cannot cast zero votes');
```
And always check that voter has >= 1 votes in all proposals that going to be rewarded in `Rewards.getVotingClientIds()`:
```diff
for (uint32 i; i < numClientIds; ++i) {
+           if (sumVotes[i] >= proposals.length) nonZeroClientIds[idx++] = i;       
-           if (sumVotes[i] > 0) nonZeroClientIds[idx++] = i;    
        }
```