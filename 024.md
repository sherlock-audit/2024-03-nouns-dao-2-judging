Bright Licorice Crow

medium

# A proposer can use the same signature for multiple proposals.

## Summary
There are several ways to create a `proposal`.
One approach involves using `signatures` from several users.
The `proposer` should then pass the `quorum threshold check`.
`Votes` from `signers` are also taken into account.
This approach allows `proposers` to create `proposals` without delegating `votes`.
However, a `signature` can be used multiple times before it's `expiration date`.
In other words, some `signers` agree on the proposal at a specific time and set an enough `expiration date`.
After executing the `proposal`, the `proposer` needs to make the same `proposal` again.
At this point, he can reuse previous `signature` if they have not yet expired.
## Vulnerability Detail
When creating a `proposal` using `signatures`, the `votes` from `signers` are also used to pass the `threshold check`.
```solidity
 function proposeBySigs(
    NounsDAOTypes.Storage storage ds,
    NounsDAOTypes.ProposerSignature[] memory proposerSignatures,
    ProposalTxs memory txs,
    string memory description,
    uint32 clientId
) external returns (uint256) {
    (uint256 votes, address[] memory signers) = verifySignersCanBackThisProposalAndCountTheirVotes(
        ds,
        proposerSignatures,
        txs,
        description,
        temp.proposalId
    );
    if (signers.length == 0) revert MustProvideSignatures();
    if (votes <= temp.propThreshold) revert VotesBelowProposalThreshold();

}
```
The sign data includes information about the `proposer`, `proposal contents`, and `description`.
```solidity
function calcProposalEncodeData(
    address proposer,
    ProposalTxs memory txs,
    string memory description
) internal pure returns (bytes memory) {
    return
        abi.encode(
            proposer,
            keccak256(abi.encodePacked(txs.targets)),
            keccak256(abi.encodePacked(txs.values)),
            keccak256(abi.encodePacked(signatureHashes)),
            keccak256(abi.encodePacked(calldatasHashes)),
            keccak256(bytes(description))
        );
}
```
Consequently, a `signature` can be used for the same `proposals` multiple times before it's `expiration date`.
I believe this should not be permitted.
Even for the same `proposal`, a new `proposal` requires collecting new `signatures` again.
## Impact

## Code Snippet
https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/8f6879efaf831eb7fc9d4a4ad2b62b5334220d87/nouns-monorepo/packages/nouns-contracts/contracts/governance/NounsDAOProposals.sol#L187-L195
https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/8f6879efaf831eb7fc9d4a4ad2b62b5334220d87/nouns-monorepo/packages/nouns-contracts/contracts/governance/NounsDAOProposals.sol#L849-L857
## Tool used

Manual Review

## Recommendation
`Signatures` should be cancelled once they have been used.