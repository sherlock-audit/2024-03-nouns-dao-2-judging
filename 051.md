Breezy Marmalade Mouse

high

# Eligibility of cancelled proposals makes it possible for `proposalEligibilityQuorumBps` controlling actor to create multiple eligible proposals, stealing rewards from all others

## Summary

Actor controlling more than `proposalEligibilityQuorumBps` can spam eligible proposals via creation, voting and canceling them, stealing proposal based rewards from all other participants. Since `proposalEligibilityQuorumBps` is set to `10%`, it can require colluding of the several Nouns holders. Spamming here means that this have much shorter turnaround time compared to usual workflow and has a potential of heavily diluting the rewards for all others.

## Vulnerability Detail

Suppose an actor is affiliated with `clientId` being `approved` and controls (directly or indirectly via collusion) more than `proposalEligibilityQuorumBps` of the current total supply. They can repeat `(create, vote, cancel)` cycle over time obtaining a significant share of proposal rewards since this will produce eligible proposals substantially faster compared to the normal lifecycle as full `votingPeriod` is omitted.

In order to conceal the manipulation this can be intermixed with other activities: i.e. before most valid proposal's, having target `clientId`, voting and execution there might be some deliberately broken versions of this proposal being posted, then voted on and cancelled like if the inconsistency was just being discovered. This can be done gradually in order to avoid raising red flags and keep `clientId` valid.

## Impact

The only prerequisite is some arrangement of Nouns holders in order in exceed `proposalEligibilityQuorumBps` bar. All the rest can be done routinely. The manipulation will not be evident on rewards allocation as `updateRewardsForProposalWritingAndVoting()` will count manipulated proposals automatically.

Increasing the number of eligible proposals has substantial impact on rewards calculation, so the impact on the other client id owners' reward revenue is high.

Likelihood: Medium + Impact: High = Severity: High.

## Code Snippet

Cancelled proposals will be deemed valid and will dilute the proposal rewards base:

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/client-incentives/Rewards.sol#L352-L368

```solidity
        for (uint256 i; i < proposals.length; ++i) {
            // make sure proposal finished voting
            uint endBlock = max(proposals[i].endBlock, proposals[i].objectionPeriodEndBlock);
>>          require(block.number > endBlock, 'all proposals must be done with voting');

            // skip non eligible proposals
>>          if (proposals[i].forVotes < (proposals[i].totalSupply * proposalEligibilityQuorumBps_) / 10_000) {
                delete proposals[i];
                continue;
            }

            // proposal is eligible for reward
            ++t.numEligibleProposals;

            uint256 votesInProposal = proposals[i].forVotes + proposals[i].againstVotes + proposals[i].abstainVotes;
            t.numEligibleVotes += votesInProposal;
        }
```

Proposals can be cancelled by `proposer` early on (whenever in a non-final state), i.e. right after `proposalUpdatablePeriodInBlocks + votingDelay` passed and proposal becomes `Active`, so it can be voted on and cancelled:

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/governance/NounsDAOProposals.sol#L518-L547

```solidity
    function cancel(NounsDAOTypes.Storage storage ds, uint256 proposalId) external {
        NounsDAOTypes.ProposalState proposalState = stateInternal(ds, proposalId);
        if (
            proposalState == NounsDAOTypes.ProposalState.Canceled ||
            proposalState == NounsDAOTypes.ProposalState.Defeated ||
            proposalState == NounsDAOTypes.ProposalState.Expired ||
            proposalState == NounsDAOTypes.ProposalState.Executed ||
            proposalState == NounsDAOTypes.ProposalState.Vetoed
        ) {
>>          revert CantCancelProposalAtFinalState();
        }

        NounsDAOTypes.Proposal storage proposal = ds._proposals[proposalId];
        address proposer = proposal.proposer;
        NounsTokenLike nouns = ds.nouns;

        uint256 votes = nouns.getPriorVotes(proposer, block.number - 1);
        bool msgSenderIsProposer = proposer == msg.sender;
        address[] memory signers = proposal.signers;
        for (uint256 i = 0; i < signers.length; ++i) {
            msgSenderIsProposer = msgSenderIsProposer || msg.sender == signers[i];
            votes += nouns.getPriorVotes(signers[i], block.number - 1);
        }

        require(
>>          msgSenderIsProposer || votes <= proposal.proposalThreshold,
            'NounsDAO::cancel: proposer above threshold'
        );

>>      proposal.canceled = true;
```

After that proposer and other backers can immediately create new proposal since the state of their current one is `Canceled` and the corresponding check is satisfied:

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/governance/NounsDAOProposals.sol#L778-L789

```solidity
>>  function checkNoActiveProp(NounsDAOTypes.Storage storage ds, address proposer) internal view {
        uint256 latestProposalId = ds.latestProposalIds[proposer];
        if (latestProposalId != 0) {
            NounsDAOTypes.ProposalState proposersLatestProposalState = stateInternal(ds, latestProposalId);
            if (
>>              proposersLatestProposalState == NounsDAOTypes.ProposalState.ObjectionPeriod ||
>>              proposersLatestProposalState == NounsDAOTypes.ProposalState.Active ||
>>              proposersLatestProposalState == NounsDAOTypes.ProposalState.Pending ||
>>              proposersLatestProposalState == NounsDAOTypes.ProposalState.Updatable
            ) revert ProposerAlreadyHasALiveProposal();
        }
    }
```

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/governance/NounsDAOProposals.sol#L582-L592

```solidity
    function stateInternal(
        NounsDAOTypes.Storage storage ds,
        uint256 proposalId
    ) internal view returns (NounsDAOTypes.ProposalState) {
        require(ds.proposalCount >= proposalId, 'NounsDAO::state: invalid proposal id');
        NounsDAOTypes.Proposal storage proposal = ds._proposals[proposalId];

        if (proposal.vetoed) {
            return NounsDAOTypes.ProposalState.Vetoed;
        } else if (proposal.canceled) {
>>          return NounsDAOTypes.ProposalState.Canceled;
```

## Tool used

Manual Review

## Recommendation

Consider ignoring cancelled proposals, e.g.:

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/governance/NounsDAOProposals.sol#L719-L732

```diff
    function proposalDataForRewards(
        ...
    ) internal view returns (NounsDAOTypes.ProposalForRewards[] memory) {
        require(lastProposalId >= firstProposalId, 'lastProposalId >= firstProposalId');
        uint256 numProposals = lastProposalId - firstProposalId + 1;
        NounsDAOTypes.ProposalForRewards[] memory data = new NounsDAOTypes.ProposalForRewards[](numProposals);

        NounsDAOTypes.Proposal storage proposal;
        uint256 i;
        for (uint256 pid = firstProposalId; pid <= lastProposalId; ++pid) {
            proposal = ds._proposals[pid];
+           if (proposal.canceled) continue;            
```

Rationale is that cancelled proposal is a kind of discarded draft, even if it was voted on, and is to be replaced by another, corrected, version, so including both in the proposal rewards is essentially double counting.

Vetoed ones can stay as is since, apart from veto power holder being trusted, in order to require a veto the proposal needs to be executable, so there looks to be no possibility to quickly iterate many such proposals and dilute the rewards. I.e. canceling provides a significant speed up for eligible proposals making, while vetoing doesn't.

Also, as a simplification and a mitigation for other issues, the eligibility threshold can be passed to `proposalDataForRewards()` and non-eligible proposals can be excluded early on, e.g.:

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/client-incentives/Rewards.sol#L323-L327

```diff
        NounsDAOTypes.ProposalForRewards[] memory proposals = nounsDAO.proposalDataForRewards(
            t.nextProposalIdToReward,
            lastProposalId,
+           $.params.proposalEligibilityQuorumBps,
            votingClientIds
        );
```

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

Keeping `endBlock > 0` just in case, while the rest is already checked:

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/client-incentives/Rewards.sol#L355-L361

```diff
-           require(block.number > endBlock, 'all proposals must be done with voting');
+           require(endBlock > 0 && block.number > endBlock, 'all voting must be completed');

-           // skip non eligible proposals
-           if (proposals[i].forVotes < (proposals[i].totalSupply * proposalEligibilityQuorumBps_) / 10_000) {
-               delete proposals[i];
-               continue;
-           }
```

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/client-incentives/Rewards.sol#L411-L413

```diff
        for (uint256 i; i < proposals.length; ++i) {
-           // skip non eligible deleted proposals
-           if (proposals[i].endBlock == 0) continue;
```