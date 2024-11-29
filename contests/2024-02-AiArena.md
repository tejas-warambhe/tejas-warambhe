# Ai Arena
Ai Arena contest || 10 Feb 2024 to 22 Feb 2024 on [Code4Arena](https://code4rena.com/audits/2024-02-ai-arena)

## My Findings Summary

|ID|Title|Severity|
|--|-----|:------:|
|[H&#8209;01](#h-01-allows-player-at-the-start-of-each-round-to-generate-a-risk-free-position-by-setting-accumulatedpointsperfighter-greater-than-0-by-winning-a-few-battles)|Allows player at the start of each round to generate a risk free position by setting [accumulatedPointsPerFighter](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/RankedBattle.sol#L479) greater than 0 by winning a few battles.|HIGH|

---

## [H-01] Allows player at the start of each round to generate a risk free position by setting [accumulatedPointsPerFighter](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/RankedBattle.sol#L479) greater than 0 by winning a few battles.

## Impact
Allows player at the start of each round to generate a risk free position by setting [`accumulatedPointsPerFighter`](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/RankedBattle.sol#L479) greater than 0 by winning a few battles.


## Description
The [`accumulatedPointsPerFighter`](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/RankedBattle.sol#L527) stores points accumulated by the fighter when the fighter wins a battle.


If we have a closer look at the the [`_addResultPoints`](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/RankedBattle.sol#L479) function, inside the condition where the fighter loses the battle, there's a [`if block`](https://github.com/code-423n4/2024-02-ai-arena/blob/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/RankedBattle.sol#L479) which checks if the `accumulatedPointsPerFighter` is greater than 0.


```solidity
File: src/RankedBattle.sol


479:          if (accumulatedPointsPerFighter[tokenId][roundId] > 0) {
480:                /// If the fighter has a positive point balance for this round, deduct points
481:                points = stakingFactor[tokenId] * eloFactor;
482:                if (points > accumulatedPointsPerFighter[tokenId][roundId]) {
483:                    points = accumulatedPointsPerFighter[tokenId][roundId];
484:                }
485:                accumulatedPointsPerFighter[tokenId][roundId] -= points;
486:                accumulatedPointsPerAddress[fighterOwner][roundId] -= points;
487:               totalAccumulatedPoints[roundId] -= points;
488:                if (points > 0) {
489:                    emit PointsChanged(tokenId, points, false);
490:                }
491:            } else {


```


This can be used as a way to gamify the points system, where a player would stake a very minimal amount of NRN and all he needs to do is gain a positive `accumulatedPointsPerFighter` which can be achieved by playing a few battles.


Once the `accumulatedPointsPerFighter` is greater than 0, the player would stake a very high amount of NRNs by using the `RankedBattle::stakeNRN` function.
The next round for the player is essentially a risk free game, in case the player loses the next battle, that would result in a very minimal point loss as the points earned were on the basis of previous staked amount.


But, if he wins, his [`stakingFactor`](https://github.com/code-423n4/2024-02-ai-arena/blame/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/RankedBattle.sol#L258C13-L258C26) would be too large and as soon as the battle was finished, the player can choose to immediately unstake his NRNs and leave, also unlocks the potential mis-use of this by using several accounts to play the game where the NRNs are transferred from one account to another as long as he can keep replicating the above position.
This also opens a way to have a significantly higher chance of getting an select for the NFT mint via merging pool as during winning condition the player can decide to use 100% of his [`mergingPortion`](https://github.com/code-423n4/2024-02-ai-arena/blame/cd1a0e6d1b40168657d1aaee8223dc050e15f8cc/src/RankedBattle.sol#L324).


An example for this scenario:


1. Player A stakes 1 NRN to become eligible for participating in fights.
2. Now, player A will try to win just enough battles to increase his `accumulatedPointsPerFighter` to be greater than 0.
3. As soon as it's greater than 0, the Player A will stake an abysmally large amount of NRNs for now let's consider the player A staked 3999 NRNs.
4. In the next battle there are three possibilities:-
   - Player A lost the battle
       - The player A here would lose only a small amount of points that he had earned using the earlier staked 1 NRN, let's consider it to be only 1500 points (staking factor would have been 1 and eloFactor would have been 1500).
       - Now, the player A will decide to not engage in battle further and immediately withdraw his staked NRNs.
   - Player A drew the battle
       - The player A will wait and play one more battle.
   - Player A wont he battle
       - Here, the amount of points player A would've earned would be 96000.
       - Now, the player A will decide to not engage in battle further and immediately withdraw his staked NRNs.
5. The player A can again play such a favorable battle in future rounds or create multiple accounts of his own and gamify the system.




## Proof of Concept
The test below is written inside the file `src/test/RankedBattle.t.sol`.
```solidity
   File: src/test/RankedBattle.t.sol


   function testUpdateBattleRecordPlayerFavourableBet() public {
       address playerA = vm.addr(3);
       address playerB = vm.addr(4);

       _mintFromMergingPool(playerA);
       _fundUserWith4kNeuronByTreasury(playerA);
  
       _mintFromMergingPool(playerB);
       _fundUserWith4kNeuronByTreasury(playerB);

       vm.prank(playerA);
       _rankedBattleContract.stakeNRN(1 * 10 ** 18, 0);

       vm.prank(playerB);
       _rankedBattleContract.stakeNRN(1 * 10 ** 18, 1);

       assertEq(_rankedBattleContract.amountStaked(0), 1 * 10 ** 18); // Player A staked 1 NRN
       assertEq(_rankedBattleContract.amountStaked(1), 1 * 10 ** 18); // Player B staked 1 NRN

       vm.prank(address(_GAME_SERVER_ADDRESS));
       _rankedBattleContract.updateBattleRecord(0, 0, 0, 1500, true);
       assertEq(_rankedBattleContract.accumulatedPointsPerFighter(0, 0) == 1500, true); // Player A accumulated points is 1500 by winning the battle


       vm.prank(address(_GAME_SERVER_ADDRESS));
       _rankedBattleContract.updateBattleRecord(1, 0, 0, 1500, true);
       assertEq(_rankedBattleContract.accumulatedPointsPerFighter(1, 0) == 1500, true); // Player B accumulated points is 1500 by winning the battle

       // The favorable condition was met and the accumulated points for both players is 1500 (assuming stakingFactor as 1 and eloFactor as 1500)
       // Now the players can stake abysmally high amounts and still not lose any points, for simplicity we will stake 3999 NRN.

        vm.prank(playerA);
       _rankedBattleContract.stakeNRN(3_999 * 10 ** 18, 0);

       vm.prank(playerB);
       _rankedBattleContract.stakeNRN(3_999 * 10 ** 18, 1);

        vm.prank(address(_GAME_SERVER_ADDRESS));
        // Losing Scenario
       _rankedBattleContract.updateBattleRecord(0, 0, 2, 1500, true); // Player A lost
       assertEq(_rankedBattleContract.accumulatedPointsPerFighter(0, 0) == 0, true); // Player A accumulated points is 0 and
       assertEq(_rankedBattleContract.amountStaked(0) == 4_000 * 10 ** 18, true); // Player A staked amount is 4000 NRN, only minimal amount of points (1500) were lost

       vm.prank(address(_GAME_SERVER_ADDRESS));
       // Winning Scenario
       _rankedBattleContract.updateBattleRecord(1, 0, 0, 1500, true); // Player B won
       assertEq(_rankedBattleContract.accumulatedPointsPerFighter(1, 0) == 96000, true); // Player B accumulated points is now 96000
       assertEq(_rankedBattleContract.amountStaked(1) == 4_000 * 10 ** 18, true); // Player B staked amount is 4000 NRN

       // After this battle, Both players can withdraw their Stakes and gamify the system even more by creating new accounts or rejoining from old accounts
       // This process can be repeated in every round, putting the system at risk of being gamed.
      
   }
```
## Tools Used
Foundry


## Recommended Mitigation Steps


A possible solution would be to maintain a mapping of excessive points so that it can be used to slash NRNs during claims in future.

---