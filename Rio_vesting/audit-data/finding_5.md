Front-Running Vulnerability in and `VestingEscrow.sol:revokeAll()` Function: Exploiting block.timestamp for Unintended Token Claims.

## Summary

There is a potential vulnerability related to the use of block.timestamp in the `VestingEscrow.sol:revokeAll()` function, which could be exploited by a front-running attack performed by the onlyRecipient address.

## Vulnerability Detail

The `VestingEscrow.sol:revokeAll()` function sets disabledAt using block.timestamp before transferring the revoked tokens to the owner. As block.timestamp is publicly readable, a front-running attack is possible.
A malicious recipient (onlyRecipient) could front-run the `VestingEscrow.sol:revokeAll()` transaction, quickly execute a transaction to claim the tokens before the `VestingEscrow.sol:revokeAll()` function is processed, and potentially receive more tokens than intended.

suppose startTime = t1 , endTime = t5 ,cliffLenght = 1 hours , totalLocked = 500
formula to calculate unvested tokens = [return Math.min(_totalLocked * (time - _startTime) / (endTime() - _startTime), _totalLocked)]
at t3 = min((500*(3-1)/5-1),500)= 200 at t4= min((500*(4-1)/5-1),500) = 375
At time t3 total unlocked tokens = 200 the malaicious recipient is miner then can hold the owner tranaction upto t4 until more tokens get released if owner wanted it to execute it on t3 where only 200 tokens were released but now as time passed more token will be available and he can recipient can claims tokens upto t4 and stops onwer to receive tokens.
result: end up claim more 125 tokens.

## Impact

The use of block.timestamp in revokeUnvested allows a recipient to exploit the time-sensitive operation, potentially claiming more tokens than intended.
Front-running could lead to an imbalance in token transfers, impacting the fairness and intended distribution of unvested tokens.

## Code Snippet

```javascript
 function claim(address beneficiary, uint256 amount) external onlyRecipient returns (uint256) {
        uint256 claimable = Math.min(unclaimed(), amount);
        totalClaimed += claimable;

        token().safeTransfer(beneficiary, claimable);
        emit Claim(beneficiary, claimable);

        return claimable;
    }
```

```javascript
   function revokeAll() external onlyOwner {
        if (!isFullyRevokable) revert NOT_FULLY_REVOKABLE();
        if (isFullyRevoked) revert ALREADY_FULLY_REVOKED();

        uint256 revokable = locked() + unclaimed();
        if (revokable == 0) revert NOTHING_TO_REVOKE();

        isFullyRevoked = true;
        disabledAt = uint40(block.timestamp);

        token().safeTransfer(_owner(), revokable);
        emit VestingFullyRevoked(msg.sender, revokable);
    }
```

## Tool used

Manual Review

## Recommendation

Consider using a secure source of time, such as block.number, to reduce the risk of front-running.
If possible, reorder the operations and find otherway which execude block.timeStamp to minimize the impact of front-running or consider alternative mechanisms to prevent front-running attacks.
