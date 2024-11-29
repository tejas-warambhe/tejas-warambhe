# Swan Dria
Swan Dria contest || 25 Oct 2024 to 1 Nov 2024 on [Codehawks](https://codehawks.cyfrin.io/c/2024-10-swan-dria)


## My Findings Summary

|ID|Title|Severity|
|--|-----|:------:|
|[H&#8209;01](#h-01-statisticsstddev-can-revert-due-to-improper-data-type-in-statisticsvariance)|`Statistics::stddev` can revert due to improper data type in `Statistics::variance`|HIGH|
|[M&#8209;01](#m-01-no-whitelisting-mechanism-allows-malicious-validators-and-generators-to-dos-the-protocol)|No whitelisting mechanism allows malicious validators and generators to DoS the protocol.|MEDIUM|
|[M&#8209;02](#m-02-llmoracleregistryvalidate-lacks-checks-sanity-checks-on-scores)|`LLMOracleRegistry::validate` lacks checks sanity checks on `scores`|MEDIUM|
|[M&#8209;03](#m-03-buyers-rounds-can-be-dosed-due-to-improper-maxassetcount-check)|Buyer's rounds can be DoSed due to improper `maxAssetCount` check|MEDIUM|
|[L&#8209;01](#l-01-assertvalidnonce-incorrectly-validates-the-given-nonce)|`assertValidNonce` incorrectly validates the given nonce.|LOW|

---

## [H-01]: `Statistics::stddev` can revert due to improper data type in `Statistics::variance`
## Summary
[`Statistics::stddev`](https://github.com/Cyfrin/2024-10-swan-dria/blob/c8686b199daadcef3161980022e12b66a5304f8e/contracts/libraries/Statistics.sol#L31) can revert due to incorrect data type usage in the [`Statistics::variance`](https://github.com/Cyfrin/2024-10-swan-dria/blob/c8686b199daadcef3161980022e12b66a5304f8e/contracts/libraries/Statistics.sol#L18) function.

## Vulnerability Details
The [`Statistics::variance`](https://github.com/Cyfrin/2024-10-swan-dria/blob/c8686b199daadcef3161980022e12b66a5304f8e/contracts/libraries/Statistics.sol#L18) function is used to calculate variance.

The issue is the way we calculate the `diff` which is `uint` (unsigned integer), this can cause underflow in a case where `data[i]` < `mean`.
```solidity
    function variance(uint256[] memory data) internal pure returns (uint256 ans, uint256 mean) {
        mean = avg(data);
        uint256 sum = 0;
        for (uint256 i = 0; i < data.length; i++) {
            uint256 diff = data[i] - mean;        <@ // Leads to underflow when data[i] < mean
            sum += diff * diff;
        }
        ans = sum / data.length;
    }
```

## Impact
The [`Statistics::stddev`](https://github.com/Cyfrin/2024-10-swan-dria/blob/c8686b199daadcef3161980022e12b66a5304f8e/contracts/libraries/Statistics.sol#L31) function will revert, which in turn reverts a crucial call ([`finalizeValidation`](https://github.com/Cyfrin/2024-10-swan-dria/blob/c8686b199daadcef3161980022e12b66a5304f8e/contracts/llm/LLMOracleCoordinator.sol#L323)).

## Tools Used
Manual Review.

## Recommendations
It is recommended to use signed integer instead of unsigned.

---
## [M-01]: No whitelisting mechanism allows malicious validators and generators to DoS the protocol.
## Summary

A malicious actor can DoS the entire protocol by using [`LLMOracleCoordinator::respond`](https://github.com/Cyfrin/2024-10-swan-dria/blob/c8686b199daadcef3161980022e12b66a5304f8e/contracts/llm/LLMOracleCoordinator.sol#L207) and [`LLMOracleCoordinator::validate`](https://github.com/Cyfrin/2024-10-swan-dria/blob/c8686b199daadcef3161980022e12b66a5304f8e/contracts/llm/LLMOracleCoordinator.sol#L260).

## Vulnerability Details

Lack of whitelisting mechanism in `LLMOracleRegistry` allows anyone to become a Validator or Generator via [`LLMOracleRegistry::register`](https://github.com/Cyfrin/2024-10-swan-dria/blob/c8686b199daadcef3161980022e12b66a5304f8e/contracts/llm/LLMOracleRegistry.sol#L94) function which is public in nature:
```solidity
    function register(LLMOracleKind kind) public {
```
This allow malicious actors to simple register a lot of malicious validators and generators which can be simply used to DoS (Denial of Service) the entire work-flow of the protocol by calling the [`LLMOracleCoordinator::respond`](https://github.com/Cyfrin/2024-10-swan-dria/blob/c8686b199daadcef3161980022e12b66a5304f8e/contracts/llm/LLMOracleCoordinator.sol#L207) and [`LLMOracleCoordinator::validate`](https://github.com/Cyfrin/2024-10-swan-dria/blob/c8686b199daadcef3161980022e12b66a5304f8e/contracts/llm/LLMOracleCoordinator.sol#L260) functions respectively because these functions have checks in place that only allow a certain number of calls for a particular `taskId`.

## Impact

This opens the possibility of DoS for the entire protocol.

## Tools Used

Manual Review

## Recommendations

Introduce whitelisting in `LLMOracleRegistry` in order to make it centralized / trusted.

---
## [M-02]: `LLMOracleRegistry::validate` lacks checks sanity checks on `scores`
## Summary

A malicious validator can pass in incorrect `score` as it is not being sanitized.

## Vulnerability Details

The protocol allows anyone to become a validator via [`LLMOracleRegistry::register`](https://github.com/Cyfrin/2024-10-swan-dria/blob/c8686b199daadcef3161980022e12b66a5304f8e/contracts/llm/LLMOracleRegistry.sol#L94) function, it must be noted that there is no whitelisting mechanism or a way to remove oracles by the admins.

Malicious actor can register themselves as validator and fairly perform the Proof-of-Work nonce but pass an incorrect score in [`LLMOracleCoordinator::validate`](https://github.com/Cyfrin/2024-10-swan-dria/blob/c8686b199daadcef3161980022e12b66a5304f8e/contracts/llm/LLMOracleCoordinator.sol#L260) function as it lacks enough checks.

```solidity
    function validate(uint256 taskId, uint256 nonce, uint256[] calldata scores, bytes calldata metadata){
        // ...
        // check nonce (proof-of-work)
        assertValidNonce(taskId, task, nonce);  <@ // Does not sanitize the scores array in any way.
        // ...
    }
```

Such actors can actually keep on listening for others calling [`LLMOracleCoordinator::validate`](https://github.com/Cyfrin/2024-10-swan-dria/blob/c8686b199daadcef3161980022e12b66a5304f8e/contracts/llm/LLMOracleCoordinator.sol#L260) function and just pass a mean score. This allows them to actually not do any work

## Impact

Malicious validators would pass incorrect scores affecting the outcome

## Tools Used

Manual Review

## Recommendations

Introduce whitelisting in `LLMOracleRegistry`.

---
## [M-03]: Buyer's rounds can be DoSed due to improper `maxAssetCount` check
## Summary
A Buyer's rounds can be DoSed (Denial-of-service) due to a strict `maxAssetCount` check against total assets listed.

## Vulnerability Details
A Seller can list assets to the buyer using the [`Swan::list`](https://github.com/Cyfrin/2024-10-swan-dria/blob/c8686b199daadcef3161980022e12b66a5304f8e/contracts/swan/Swan.sol#L157) and [`Swan::relist`](https://github.com/Cyfrin/2024-10-swan-dria/blob/c8686b199daadcef3161980022e12b66a5304f8e/contracts/swan/Swan.sol#L197) functions.

The issue here is that it implements a check against all the total listings of that particular round against `maxAssetCount`.
```solidity
// Swan::list

if (getCurrentMarketParameters().maxAssetCount == assetsPerBuyerRound[_buyer][round].length) {  <@ // Checks against all assets listed to this buyer
    revert AssetLimitExceeded(getCurrentMarketParameters().maxAssetCount);
}
```
```solidity
// Swan::relist

    uint256 count = assetsPerBuyerRound[_buyer][round].length;
    if (count >= getCurrentMarketParameters().maxAssetCount) {
        revert AssetLimitExceeded(count);
    }
```

A malicious actor can DoS these functions by listing assets with dust or `0` value ensuring no genuine seller gets to list to that particular buyer.

## Proof of Concept

Replace the function in the `test/Swan.test.ts` file at [`L167`](https://github.com/Cyfrin/2024-10-swan-dria/blob/c8686b199daadcef3161980022e12b66a5304f8e/test/Swan.test.ts#L167).

```typescript
    it("should list 5 bogus assets for the first round", async function () {
      await listAssets(
        swan,
        buyerAgent,
        [
          [seller, parseEther("0")],
          [seller, parseEther("0")],
          [seller, parseEther("0")],
          [seller, parseEther("0")],
          [seller, parseEther("0")],
        ],
        "useless",
        "useless",
        ethers.encodeBytes32String("useless description"),
        0n
      );

      // genuine seller trying to list an asset would fail
      await expect(swan.connect(seller).list(NAME, SYMBOL, DESC, PRICE1, await buyerAgent.getAddress()))
        .to.be.revertedWithCustomError(swan, "AssetLimitExceeded")
        .withArgs(MARKET_PARAMETERS.maxAssetCount);
    });
```


## Impact
Leads to Denial of service for the buyer agent.

## Tools Used
Manual Review + Hardhat

## Recommendations
Consider implementing a `seller => buyer => round` mapping to avoid checking against total listings.

---
## [L-01]: `assertValidNonce` incorrectly validates the given nonce
## Summary
The validation function [`LLMOracleCoordinator::assertValidNonce`](https://github.com/Cyfrin/2024-10-swan-dria/blob/c8686b199daadcef3161980022e12b66a5304f8e/contracts/llm/LLMOracleCoordinator.sol#L313) does not correctly follow the assertion mentioned in the code documentation which allows a case where `SHA3(taskId, input, requester, responder, nonce) == difficulty`.


## Vulnerability Details
The function [`LLMOracleCoordinator::assertValidNonce`](https://github.com/Cyfrin/2024-10-swan-dria/blob/c8686b199daadcef3161980022e12b66a5304f8e/contracts/llm/LLMOracleCoordinator.sol#L313) is used to validate the proof of work nonce.
```solidity
    function assertValidNonce(uint256 taskId, TaskRequest storage task, uint256 nonce) internal view {
        bytes memory message = abi.encodePacked(taskId, task.input, task.requester, msg.sender, nonce);
        if (uint256(keccak256(message)) > type(uint256).max >> uint256(task.parameters.difficulty)) {    <@ // Reverts when SHA3(taskId, input, requester, responder, nonce) > difficulty
            revert InvalidNonce(taskId, nonce);
        }
    }
```
This is incorrect because the documentation inside the `LLMOracleTask` contract states the following invariant in 2 different instances at [`L60`](https://github.com/Cyfrin/2024-10-swan-dria/blob/c8686b199daadcef3161980022e12b66a5304f8e/contracts/llm/LLMOracleTask.sol#L60) and [`L74`](https://github.com/Cyfrin/2024-10-swan-dria/blob/c8686b199daadcef3161980022e12b66a5304f8e/contracts/llm/LLMOracleTask.sol#L74) respectively:

```solidity
    /// @dev Proof-of-Work nonce for SHA3(taskId, input, requester, responder, nonce) < difficulty.
    uint256 nonce;
```

## Impact
This allows a nonce where `SHA3(taskId, input, requester, responder, nonce) == difficulty` which is unintended.
## Tools Used
Manual Review

## Recommendations
It is recommended to use `>=` instead in `assertValidNonce`:
```solidity
    function assertValidNonce(uint256 taskId, TaskRequest storage task, uint256 nonce) internal view {
        bytes memory message = abi.encodePacked(taskId, task.input, task.requester, msg.sender, nonce);
        if (uint256(keccak256(message)) >= type(uint256).max >> uint256(task.parameters.difficulty)) {    <@    // Replaced ">" with ">="
            revert InvalidNonce(taskId, nonce);
        }
    }
```
---

