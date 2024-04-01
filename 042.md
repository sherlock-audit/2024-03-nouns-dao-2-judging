Breezy Marmalade Mouse

medium

# Public gas refunds with low bar conditions for both proposal and auction reward allocations allow gas reward funds overspending

## Summary

Refundable rewards calculation can be run for a minimum possible impact: to capture rewards for only one proposal or only `minimumAuctionsBetweenUpdates` auction results. Since these functions do not have access controls (contrary to `castRefundableVoteInternal()` which cost is compensated for voters only) and return gas costs to the caller once low enough minimums are met, this setup incentivizes big number of the smallest possible calls, which can be detrimental for the spending pattern of the Reward contract funds, which are shared between refunds and rewards.

## Vulnerability Detail

Due to `REFUND_BASE_GAS = 36000` additional refund and loose enough minimum requirements of the reward accounting logic, whenever market gas is low (so upper limits won't trigger), it might be profitable to issue as many as possible 1 proposal / `minimumAuctionsBetweenUpdates` Nouns id worth of `updateRewardsForProposalWritingAndVoting()` and `updateRewardsForAuctions()` calls correspondingly to capture this base gas reward multiple times.

Also, given that Rewards contract is upgradable by owner and `setClientApproval(id, false)` can freeze all the rewards due, there is an incentive to call rewards allocation functions frequently for direct reward beneficiaries as well.

## Impact

Both pure refund seeking actors and risk minimizing approved `clientId` owners will tend to call `updateRewardsForProposalWritingAndVoting()` and `updateRewardsForAuctions()` as frequent as possible, bloating gas refund costs. As those are compensated from the same rewards pot, it's equivalent to stealing the part of it over time. I.e. the end result is, having all other factors being equal, that Reward contract insolvency takes less time to occur.

There are no prerequisites, so the probability is high. The total loss is limited by the fixed nature of refund in question and the boundaries on the number of eligible calls both for proposals and auctions, so the monetary impact is low (albeit being continuous and accumulating over time).

Likelihood: High + Impact: Low = Severity: Medium.

## Code Snippet

`REFUND_BASE_GAS` is fixed (note that there is no token transfer in all current refund use cases: voting and rewards calculations) and its spread to the real gas overhead can vary:

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/libs/GasRefund.sol#L27-L28

```solidity
    /// @notice The vote refund gas overhead, including 7K for token transfer and 29K for general transaction overhead
    uint256 public constant REFUND_BASE_GAS = 36000;
```

When gas is cheap enough to not trigger the `MAX_REFUND_BASE_FEE` and `MAX_REFUND_PRIORITY_FEE` limits the uncapped `gasUsed` can go above that was spent on the operations:

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/libs/GasRefund.sol#L34-L46

```solidity
    function refundGas(IERC20 ethToken, uint256 startGas) internal {
        unchecked {
            uint256 balance = ethToken.balanceOf(address(this));
            if (balance == 0) {
                return;
            }
            uint256 basefee = min(block.basefee, MAX_REFUND_BASE_FEE);
            uint256 gasPrice = min(tx.gasprice, basefee + MAX_REFUND_PRIORITY_FEE);
>>          uint256 gasUsed = startGas - gasleft() + REFUND_BASE_GAS;
            uint256 refundAmount = min(gasPrice * gasUsed, balance);
            ethToken.safeTransfer(tx.origin, refundAmount);
        }
    }
```

Callers can end up being rewarded for redundant gas spending as lots of operations have to be repeated when the function is called with minimal increments. I.e from total gas spend perspective it's far from optimal to run many small `updateRewardsForProposalWritingAndVoting()` requests each with a minimal step in `lastProposalId` and `auctionedId`, but exactly this is now incentivized.

`updateRewardsForAuctions()` only requires any `clientId > 0` to be found:

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/client-incentives/Rewards.sol#L255-L261

```solidity
        for (uint256 i; i < settlements.length; ++i) {
            INounsAuctionHouseV2.Settlement memory settlement = settlements[i];
            if (settlement.clientId != 0 && settlement.clientId <= maxClientId) {
>>              sawValidClientId = true;
                m.inc(settlement.clientId, settlement.amount);
            }
        }
```

Each repetitive bidder can create one in case there is no affiliation with an existing id:

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/client-incentives/Rewards.sol#L187-L192

```solidity
    function registerClient(string calldata name, string calldata description) external whenNotPaused returns (uint32) {
        RewardsStorage storage $ = _getRewardsStorage();

        uint32 tokenId = $.nextTokenId;
        $.nextTokenId = tokenId + 1;
        _mint(msg.sender, tokenId);
```

For example, in order to receive extra funds from refund each time calling `updateRewardsForAuctions(justSettledId)`:

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/client-incentives/Rewards.sol#L275-L278

```solidity
        if (sawValidClientId) {
            // refund gas only if we're actually rewarding a client, not just moving the pointer
            GasRefund.refundGas($.ethToken, startGas);
        }
```

## Tool used

Manual Review

## Recommendation

Consider extending the minimal operations required for gas refund by adding a control for the volume of the operations carried out.

For example, this can be done with the help of the `$.minAuctionsPerProposal` parameter introduced in the mitigation of another issue by requiring that at least this number of successful auctions has to be processed in each call (note, that settlements history can carry no bids auctions, while positive client id ensures that some bid was made), e.g.:

https://github.com/sherlock-audit/2024-03-nouns-dao-2/blob/main/nouns-monorepo/packages/nouns-contracts/contracts/client-incentives/Rewards.sol#L234-L278

```diff
-       bool sawValidClientId = false;
+       uint256 auctionsWithClientId;
        uint256 nextAuctionIdToReward_ = $.nextAuctionIdToReward;
        require(
            lastNounId >= nextAuctionIdToReward_ + $.params.minimumAuctionsBetweenUpdates,
            'lastNounId must be higher'
        );
        $.nextAuctionIdToReward = lastNounId + 1;

        INounsAuctionHouseV2.Settlement[] memory settlements = auctionHouse.getSettlements(
            nextAuctionIdToReward_,
            lastNounId + 1,
            true
        );
        INounsAuctionHouseV2.Settlement memory lastSettlement = settlements[settlements.length - 1];
        require(lastSettlement.nounId == lastNounId && lastSettlement.blockTimestamp > 1, 'lastNounId must be settled');

        uint32 maxClientId = nextTokenId() - 1;
        ClientRewardsMemoryMapping.Mapping memory m = ClientRewardsMemoryMapping.createMapping({
            maxClientId: maxClientId
        });

        for (uint256 i; i < settlements.length; ++i) {
            INounsAuctionHouseV2.Settlement memory settlement = settlements[i];
            if (settlement.clientId != 0 && settlement.clientId <= maxClientId) {
-               sawValidClientId = true;
+               auctionsWithClientId++;
                m.inc(settlement.clientId, settlement.amount);
            }
        }

        uint16 auctionRewardBps = $.params.auctionRewardBps;
        uint256 numValues = m.numValues();
        for (uint32 i = 0; i < numValues; ++i) {
            ClientRewardsMemoryMapping.ClientBalance memory cb = m.getValue(i);
            uint256 reward = (cb.balance * auctionRewardBps) / 10_000;
            $._clientMetadata[cb.clientId].rewarded += SafeCast.toUint96(reward);

            emit ClientRewarded(cb.clientId, reward);
        }

        emit AuctionRewardsUpdated(nextAuctionIdToReward_, lastNounId);

-       if (sawValidClientId) {
+       if (auctionsWithClientId >= $.minAuctionsPerProposal) {
-           // refund gas only if we're actually rewarding a client, not just moving the pointer
+           // refund gas only if we're actually rewarding at least a min number of auctions, not just moving the pointer
            GasRefund.refundGas($.ethToken, startGas);
        }
```

The `updateRewardsForProposalWritingAndVoting()` case is covered by the same `$.minAuctionsPerProposal` linked mitigation as described in the another issue (with the introduction of `require(successfulAuctions > $.minAuctionsPerProposal * t.numEligibleProposals, 'min Nouns sold per proposal')` control).