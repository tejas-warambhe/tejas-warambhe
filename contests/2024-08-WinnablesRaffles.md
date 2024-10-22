# Winnables Raffles
Winnables Raffles contest || 16 Aug 2024 to 20 Aug 2024 on [Sherlock](https://audits.sherlock.xyz/contests/516?filter=questions)

## My Findings Summary

|ID|Title|Severity|
|--|-----|:------:|
|[H&#8209;01](#h-01-permanent-l1---l2-dos-because-of-whitelisting-linked-list-logic)|Allows player at the start of each round to generate a risk free position by setting [accumulatedPointsPerFighter](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/RankedBattle.sol#L479) greater than 0 by winning a few battles.|HIGH|
|[M&#8209;01](#h-01-permanent-l1---l2-dos-because-of-whitelisting-linked-list-logic)|`setCCIPCounterpart()` allows admin to deny raffle winner from claiming prize.|MEDIUM|

---

## [H-01]: Raffle termination due to insufficient checks in `WinnablesTicketManager::_checkShouldCancel`
### Summary
Malicious actor can cancel the raffle as soon as [`RafflePrizeLocked`](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesTicketManager.sol#L383) event is triggered on [`WinnablesTicketManager::_ccipReceive`](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesTicketManager.sol#L365)using [`cancelRaffle`](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesTicketManager.sol#L278C14-L278C26)

### Root Cause
In [WinnablesTicketManager.sol:436](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesTicketManager.sol#L436) the control is returned if the raffle status is set to `PRIZE_LOCKED`

```solidity
    function _checkShouldCancel(uint256 raffleId) internal view {
        Raffle storage raffle = _raffles[raffleId];
        if (raffle.status == RaffleStatus.PRIZE_LOCKED) return;        <@ audit
        if (raffle.status != RaffleStatus.IDLE) revert InvalidRaffle();
        if (raffle.endsAt block.timestamp) revert RaffleIsStillOpen();
        uint256 supply = IWinnablesTicket(TICKETS_CONTRACT).supplyOf(raffleId);
        if (supply raffle.minTicketsThreshold) revert TargetTicketsReached();
    }
```

Upon any prize lock by admin using [lockETH](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesPrizeManager.sol#L172), [lockNFT](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesPrizeManager.sol#L148) or [lockTokens](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesPrizeManager.sol#L196) will send a message to `WinnablesTicketManager` contract via `WinnablesPrizeManager::_sendCCIPMessage`, which is received using `WinnablesTicketManager::_ccipReceive`. This triggers an [`RafflePrizeLocked`](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesTicketManager.sol#L383) event which can be actively listened by the attacker who will immediately call the [`WinnablesTicketManager::cancelRaffle`](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesTicketManager.sol#L278) as it's an external function.

```solidity
function cancelRaffle(address prizeManager, uint64 chainSelector, uint256 raffleId) external {
        _checkShouldCancel(raffleId);              <@ audit - does not revert

        _raffles[raffleId].status = RaffleStatus.CANCELED;
        _sendCCIPMessage(
            prizeManager,
            chainSelector,
            abi.encodePacked(uint8(CCIPMessageType.RAFFLE_CANCELED), raffleId)
        );
        IWinnablesTicket(TICKETS_CONTRACT).refreshMetadata(raffleId);
    }
```

This leads to a DoS (Denial of Service) of the protocol's main feature which is organising raffles.

### Internal pre-conditions
1. Admin needs to lock prizes using [`lockETH`](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesPrizeManager.sol#L172), [`lockNFT`](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesPrizeManager.sol#L148) or [`lockTokens`](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesPrizeManager.sol#L196) functions present in `WinnablesPrizeManager` .

### External pre-conditions
1. Chainlink should be sending the message that the prize has been locked on the source chain.

### Attack Path
1. Malicious user needs to actively listen to the `WinnablesTicketManager` contract events.
2. As soon as the `RafflePrizeLocked` is emitted, attacker will call the [`WinnablesTicketManager::cancelRaffle`](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesTicketManager.sol#L278) external function which will immediately cancel the raffle.

### Impact
1. Protocol is no longer usable as intended as this disallows admin to create raffles.
2. Attempts made to lock prizes would result in loss of LINK on chainlink's subscribed messaging service.

### PoC
_No response_

### Mitigation
It is recommended to only allow the admin to cancel prize when the `RaffleStatus` is equivalent to `PRIZE_LOCKED`.


---


## [M-01]: `setCCIPCounterpart()` allows admin to deny raffle winner from claiming prize.
### Summary
Using the function [`setCCIPCounterpart()`](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesTicketManager.sol#L238) to change the CCIPCounterpart address can deny winner from claiming their prize.

### Root Cause
The function [`setCCIPCounterpart()`](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesTicketManager.sol#L238) inherently is unaware about the current state of the contract.

```solidity
    function setCCIPCounterpart(
        address contractAddress,
        uint64 chainSelector,
        bool enabled
    ) external onlyRole(0) {
        _setCCIPCounterpart(contractAddress, chainSelector, enabled);
    }
```

In an Ideal scenario, when a raffle's duration is completed, the functions [`drawWinner()`](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesTicketManager.sol#L310) and [`propagateRaffleWinner()`](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesTicketManager.sol#L334) are to be called simultaneously by anyone in order to disburse prize to the winner.

In a case where a winner is drawn using [`drawWinner()`](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesTicketManager.sol#L310) and the admin decides to call [`setCCIPCounterpart()`](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesTicketManager.sol#L238) knowingly or unknowingly before the [`propagateRaffleWinner()`](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesTicketManager.sol#L334) call, will lead to denial of prize to that winner as it would never get propagated.

This violates the one of the principles mentioned by the protocol in `README.md`. `Winnables admins cannot do anything to prevent a winner from withdrawing their prize`

### Internal pre-conditions
1. A raffle should have just passed it's designated duration

### External pre-conditions
_No response_

### Attack Path
1. Anyone calls the [`drawWinner()`](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesTicketManager.sol#L310) after a particular raffle has completed it's duration.
2. Admin will call the [`setCCIPCounterpart()`](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesTicketManager.sol#L238) function before anyone calls the [`propagateRaffleWinner()`](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/81b28633d0f450e33a8b32976e17122418f5d47e/public-contracts/contracts/WinnablesTicketManager.sol#L334)

### Impact
1. Raffle winner will be denied from claiming his prize.

### PoC
_No response_

### Mitigation
_No response_

