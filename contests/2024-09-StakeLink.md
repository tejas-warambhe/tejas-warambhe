# Stake.Link
Stake.link contest || 30 Sep 2024 to 17 Oct 2024 on [Codehawks](https://codehawks.cyfrin.io/c/2024-09-stakelink/s/380)


## My Findings Summary

|ID|Title|Severity|
|--|-----|:------:|
|[M&#8209;01](#m-01-donating-tokens-as-soon-as-stakingpool-contract-is-deployed-bricks-the-entire-minting-system)|Donating tokens as soon as `StakingPool` contract is deployed bricks the entire minting system.|MEDIUM|

---

## [M-01]: Donating tokens as soon as `StakingPool` contract is deployed bricks the entire minting system
## Summary
Donating tokens as soon as the `StakingPool` contract is deployed will disrupt the minting system.


## Vulnerability Details

The [`StakingPool::donateToken`](https://github.com/Cyfrin/2024-09-stakelink/blob/f5824f9ad67058b24a2c08494e51ddd7efdbb90b/contracts/core/StakingPool.sol#L433) function allows anyone to donate tokens to the staking pool, but if a malicous actor donates tokens as soon as the `StakingPool` contract is deployed it can deny genuine users from staking tokens via `StakingPool::deposit` and also disrupt the working of `_updateStrategyRewards`.

- This happens because as soon as the contract is deployed  and `donateTokens` is called, the `totalStaked` which was `0` will increase.

```solidity
    function donateTokens(uint256 _amount) external {
        token.safeTransferFrom(msg.sender, address(this), _amount);
@>      totalStaked += _amount;
        emit DonateTokens(msg.sender, _amount);
    }
```

- Now, if a genuine user tries to deposit tokens via `StakingPool::deposit`, which would eventaully call the `StakingRewardsPool::_mint`
```solidity
    ...
    _mint(_account, _amount);
    ...
```
- This function will further calculates shares via `getSharesByStake` which will always return `0` as `totalShares` was never increased.

```solidity
    // Mint function called by the StakingPool:deposit function
    function _mint(address _recipient, uint256 _amount) internal override {
        uint256 sharesToMint = getSharesByStake(_amount); // returns 0
        _mintShares(_recipient, sharesToMint);

        emit Transfer(address(0), _recipient, _amount);
    }

    // We get a return as 0 because totalShares is 0
    function getSharesByStake(uint256 _amount) public view returns (uint256) {
        uint256 totalStaked = _totalStaked();
        if (totalStaked == 0) {
            return _amount;
        } else {
@>          return (_amount * totalShares) / totalStaked; // Will return 0
        }
    }
```
- Hence, the `_mintShares` function will be passed on a amount as `0`. This would lead to a revert as we try to remove `DEAD_SHARES` from the `_amount`.

```solidity
    function _mintShares(address _recipient, uint256 _amount) internal {
        require(_recipient != address(0), "Mint to the zero address");

        if (totalShares == 0) {
            shares[address(0)] = DEAD_SHARES;
            totalShares = DEAD_SHARES;
@>          _amount -= DEAD_SHARES;  // Error: VM Exception while processing transaction: reverted with panic code 0x11 (Arithmetic operation overflowed outside of an unchecked block)
        }

        totalShares += _amount;
        shares[_recipient] += _amount;
    }
```

- Similarly, the `StakingPool:_updateStrategyRewards` will also get disrupted as `sharesToMint` will resolve to `0` as totalShares are `0`.
```solidity
@>  uint256 sharesToMint = (totalFeeAmounts * totalShares) / (totalStaked - totalFeeAmounts); // Resolves to 0 
    _mintShares(address(this), sharesToMint);
```

## Proof of Concept
The below test was added in [`staking-pool.test.ts`](https://github.com/Cyfrin/2024-09-stakelink/blob/main/test/core/staking-pool.test.ts) file.
Firstly ensuring that we remove the deposit from `deployFixture` function of the test suite.
```javascript
- @>    await stakingPool.deposit(accounts[0], 1000, ['0x', '0x']) // comment / remove this line

```
Test Case:

```javascript
  it('Subsequent deposits should revert when donateTokens is used as soon as the contract is deployed', async () => {
    const { accounts, stakingPool, stake } = await loadFixture(deployFixture)

    await stakingPool.donateTokens(toEther(500))
    // expect revert
    await expect(stake(1, 1000)).to.be.reverted;
    // Error: VM Exception while processing transaction: reverted with panic code 0x11 (Arithmetic operation overflowed outside of an unchecked block)
  })
```

## Impact
Disrupts the entire minting system and denies users from depositing tokens.

## Tools Used
Manual review
Hardhat

## Recommendations
Add a check in the `donateTokens` function to check if the quantity of totalShares is greater than `0`.

```solidity
    function donateTokens(uint256 _amount) external {
        require(totalShares > 0, "Donating too early");
        token.safeTransferFrom(msg.sender, address(this), _amount);
        totalStaked += _amount;
        emit DonateTokens(msg.sender, _amount);
    }
```
