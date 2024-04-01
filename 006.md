Interesting Coal Octopus

medium

# "NounsDAOVotes.sol": gas griefing

## Summary
Voters can burn large amounts of ETH by submitting votes with long reason strings in `castRefundableVoteWithReason()`.
## Vulnerability Detail
Protocol's team [confirmed](https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/client-incentives/Rewards.sol#L403-L404) that gas griefing using `_refundGas` is an unacceptable risk and they do not want it to happen:
```solidity
//// 4. Make sure all voting clientIds were included. This is meant to avoid griefing. Otherwises one could pass 
////    a large array of votingClientIds, spend a lot of gas, and have that gas refunded.
```
Multiple checks in `Rewards.sol` do not allow the user to burn alot of ETH using gas griefing attack. 
But it can happen in `NounsDAOVotes.castRefundableVoteInternal()`, where there is parameter `string memory reason`:

```solidity
function castRefundableVoteInternal(
        NounsDAOTypes.Storage storage ds,
        uint256 proposalId,
        uint8 support,
        string memory reason,
        uint32 clientId
    ) internal {
        uint256 startGas = gasleft();
        uint96 votes = castVoteInternal(ds, msg.sender, proposalId, support, clientId);
        emit VoteCast(msg.sender, proposalId, support, votes, reason);
        if (clientId > 0) emit NounsDAOEventsV3.VoteCastWithClientId(msg.sender, proposalId, clientId);
        if (votes > 0) {
            _refundGas(startGas);
        }
    }
```
The only limits to how long a string argument to a function call can be is the block gas limit of the EVM, currently 30 million. It is therefore possible for a user to cast refundable vote with a very, very long reason string. `castRefundableVoteInternal()` emits an event that includes reason, which is within the region of code covered by gas refunds (gas refunds are measured from `startGas`). Because of this, gas refunds will include the gas price of emitting this event, which could potentially be very large:
```solidity
function _refundGas(uint256 startGas) internal {
        unchecked {
            uint256 balance = address(this).balance;
            if (balance == 0) {
                return;
            }
            uint256 basefee = min(block.basefee, MAX_REFUND_BASE_FEE);                          //200 gwei 
            uint256 gasPrice = min(tx.gasprice, basefee + MAX_REFUND_PRIORITY_FEE);             //200 gwei + 2 gwei
            uint256 gasUsed = min(startGas - gasleft() + REFUND_BASE_GAS, MAX_REFUND_GAS_USED); //200_000 
            uint256 refundAmount = min(gasPrice * gasUsed, balance);                            //200_000 * 202 gwei
            (bool refundSent, ) = tx.origin.call{ value: refundAmount }('');                    //0,04 ETH
            emit RefundableVote(tx.origin, refundAmount, refundSent);
        }
    }
```
Max `refundAmount` can be 0,04 ETH and i believe it's `"a lot of gas"`. Malicious user can calcalate the length of  `reason` string to pay a little less than max refund amount, make call and get that all ETH refunded. This attack is cheap, easy, and can happen many times, leading to huge loss of funds.

## Impact
Malicious user can drain the contract's Ether, damage the protocol's reputation and not allow many other voters to receive their voting refunds.
## Code Snippet
[https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/governance/NounsDAOVotes.sol#L307]()
[https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/governance/NounsDAOVotes.sol#L105]()
## Tool used

Manual Review

## Recommendation
Change the type of reason to `bytes` or add a check to its length in `castRefundableVoteWithReason()`, reverting if it is too long.

