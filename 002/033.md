Bright Licorice Crow

medium

# Users may not be able to claim tokens from escrow in the forked DAO.

## Summary
Users can send tokens to the `escrow` and can later claim the tokens with the same `ID` within the `forked DAO`.
Once tokens are sent to the `escrow`, users should be able to claim them at any time.
The `admin` has the ability to withdraw tokens from the `escrow` and sent to other users in order to increase the `total supply`.
And that users can join the current `forked DAO` using these tokens.
As a result, due to the constraint that the same token ID can not be minted twice, users who sent tokens to the `escrow` may not be able to claim tokens in the `forked DAO`.
## Vulnerability Detail
Users have the ability to send tokens to the `escrow` before the `forking period`.
```solidity
function escrowToFork(
    NounsDAOTypes.Storage storage ds,
    uint256[] calldata tokenIds,
    uint256[] calldata proposalIds,
    string calldata reason
) external {
    if (isForkPeriodActive(ds)) revert ForkPeriodActive();
    for (uint256 i = 0; i < tokenIds.length; i++) {
        ds.nouns.safeTransferFrom(msg.sender, address(forkEscrow), tokenIds[i]);
    }
}
```
The `admin` can withdraw tokens from the `escrow`.
```solidity
function withdrawDAONounsFromEscrow(
    NounsDAOTypes.Storage storage ds,
    uint256[] calldata tokenIds,
    address to
) private {
    if (msg.sender != ds.admin) {
        revert AdminOnly();
    }

    ds.forkEscrow.withdrawTokens(tokenIds, to);

    emit DAOWithdrawNounsFromEscrow(tokenIds, to);
}
```
Users can join to the current `forked DAO` using these tokens during the `forking period`.
```solidity
function joinFork(
    NounsDAOTypes.Storage storage ds,
    uint256[] calldata tokenIds,
    uint256[] calldata proposalIds,
    string calldata reason
) external {
    if (!isForkPeriodActive(ds)) revert ForkPeriodNotActive();
    NounsTokenFork(ds.forkDAOToken).claimDuringForkPeriod(msg.sender, tokenIds);
}
```
Tokens with these `IDs` are minted to these users within the `forked DAO`.
```solidity
function claimDuringForkPeriod(address to, uint256[] calldata tokenIds) external {
    for (uint256 i = 0; i < tokenIds.length; i++) {
        uint256 nounId = tokenIds[i]; 
        _mintWithOriginalSeed(to, nounId);   // @audit, here

        if (tokenIds[i] > maxNounId) maxNounId = tokenIds[i];
    }
}
```
The original users should be able to claim tokens with these `IDs`.
However, the transaction will be reverted.
```solidity
function claimFromEscrow(uint256[] calldata tokenIds) external {
    for (uint256 i = 0; i < tokenIds.length; i++) {
        uint256 nounId = tokenIds[i];
        if (escrow.ownerOfEscrowedToken(forkId, nounId) != msg.sender) revert OnlyTokenOwnerCanClaim();

        _mintWithOriginalSeed(msg.sender, nounId);  // @audit, here
    }

    remainingTokensToClaim -= tokenIds.length;
}
```
## Impact

## Code Snippet
https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/8f6879efaf831eb7fc9d4a4ad2b62b5334220d87/nouns-monorepo/packages/nouns-contracts/contracts/governance/fork/NounsDAOFork.sol#L83-L85
https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/8f6879efaf831eb7fc9d4a4ad2b62b5334220d87/nouns-monorepo/packages/nouns-contracts/contracts/governance/fork/NounsDAOFork.sol#L198
https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/8f6879efaf831eb7fc9d4a4ad2b62b5334220d87/nouns-monorepo/packages/nouns-contracts/contracts/governance/fork/NounsDAOFork.sol#L154
https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/8f6879efaf831eb7fc9d4a4ad2b62b5334220d87/nouns-monorepo/packages/nouns-contracts/contracts/governance/fork/newdao/token/NounsTokenFork.sol#L176
https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/8f6879efaf831eb7fc9d4a4ad2b62b5334220d87/nouns-monorepo/packages/nouns-contracts/contracts/governance/fork/newdao/token/NounsTokenFork.sol#L155
## Tool used

Manual Review

## Recommendation
Restrict `admin withdrawals` during the `forking period`.