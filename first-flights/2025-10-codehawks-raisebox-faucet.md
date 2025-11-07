
# Raisebox Faucet

## Executive Summary

RaiseBox Faucet is a token drip faucet that drips 1000 test tokens to users every 3 days. It also drips 0.005 sepolia eth to first time users. The faucet tokens will be useful for testing the testnet of a future protocol that would only allow interactions using this tokens.

More details [here.](https://codehawks.cyfrin.io/c/2025-10-raisebox-faucet)

## Issues Found

| Severity | Number of issues found |
| -------- | ---------------------- |
| High     | 3                      |
| Medium   | 2                      |
| Low      | 1                      |


# [H-01] Returning users reset the ETH daily cap allowing unlimited ETH drips

## Description

* The protocol is designed to drip ETH to new users up to a predefined daily cap.

* However, when a returning user makes a claim, the `RaiseBoxFaucet::claimFaucetTokens` function resets `RaiseBoxFaucet::dailyDrips`. This unintentionally allows unlimited ETH drips for new users within the same day, effectively bypassing the daily limit.

```solidity
        } else {
@>          dailyDrips = 0;
        }
```

## Risk

**Likelihood**:

* The issue occurs each time a returning user successfully claims new Faucet tokens.

**Impact**

* All ETH held by the faucet can be drained in a single day. Malicious actors can exploit this by repeatedly triggering the reset with returning accounts and then using multiple new accounts to claim ETH beyond the intended cap.

## Proof of Concept

Add the following test to `RaiseBoxFaucet.t.sol` to reproduce the issue:

```solidity
    function test_audit_returningUsersResetTheEthDailyCapAllowingUnlimitedEthDrips()
        public
    {
        RaiseBoxFaucet testRaiseBox = new RaiseBoxFaucet(
            "raiseboxtoken",
            "RB",
            1000 * 10 ** 18, // faucetDrip
            0.5 ether, // sepEthDrip
            1 ether // dailySepEthCap
        );
        vm.deal(address(testRaiseBox), 2.5 ether);

        vm.prank(user1);
        testRaiseBox.claimFaucetTokens();

        advanceBlockTime(block.timestamp + 3 days);

        // Make sure we have more ETH balance than the ETH cap
        assertGt(address(testRaiseBox).balance, testRaiseBox.dailySepEthCap());
        // user1 is a returning user
        address[5] memory users = [user2, user3, user1, user4, user5];
        for (uint256 i = 0; i < users.length; i++) {
            vm.prank(users[i]);
            testRaiseBox.claimFaucetTokens();
        }

        assertEq(address(testRaiseBox).balance, 0);
    }
```

## Recommended Mitigation

Remove the unnecessary reset of `RaiseBoxFaucet::dailyDrips` from the `RaiseBoxFaucet::claimFaucetTokens` function to ensure that the daily cap is enforced consistently for all users.

```diff
-       } else {
-           dailyDrips = 0;
-       }
```

# [H-02] Claims blocked indefinitely when daily Faucet claim limit reached

## Description

* When the `RaiseBoxFaucet::dailyClaimLimit` is reached, the `RaiseBoxFaucet::dailyClaimCount` should reset after one day has passed, allowing users to claim again.

* Currently, the reset of `dailyClaimCount` is performed only when the `dailyClaimLimit` has not been reached and 24 hours have passed since the last update of `lastFaucetDripDay`. This logic prevents the count from ever resetting once the limit is reached, effectively blocking further claims indefinitely.

```Solidity
function claimFaucetTokens() public {

...

@>  if (dailyClaimCount >= dailyClaimLimit) {
          revert RaiseBoxFaucet_DailyClaimLimitReached();
    }

...

@>  if (block.timestamp > lastFaucetDripDay + 1 days) {
            lastFaucetDripDay = block.timestamp;
            dailyClaimCount = 0;
     }

```

## Risk

**Likelihood**:

* The issue occurs the first time the `dailyClaimLimit` is reached.

**Impact**:

* The protocol fails to reset the `dailyClaimCount`, permanently halting all future claims. This makes the faucet non-operational and prevents users from claiming tokens as intended.

## Proof of Concept

Add the following test to `RaiseBoxFaucet.t.sol` to reproduce the issue:

```solidity
function test_audit_claimsBlockedIndefinitelyWhenDailyFaucetClaimLimitReached()
        public
    {
        vm.prank(owner);
        // Current limit is 100, reduce it to 1
        raiseBoxFaucet.adjustDailyClaimLimit(99, false);

        vm.prank(user1);
        raiseBoxFaucet.claimFaucetTokens();

        // Expected revert
        vm.prank(user2);
        vm.expectRevert(
            RaiseBoxFaucet.RaiseBoxFaucet_DailyClaimLimitReached.selector
        );
        raiseBoxFaucet.claimFaucetTokens();

        // After 3 days
        advanceBlockTime(block.timestamp + 3 days);

        // Not expected revert - the Faucet box is stacked
        vm.prank(user3);
        vm.expectRevert(
            RaiseBoxFaucet.RaiseBoxFaucet_DailyClaimLimitReached.selector
        );
        raiseBoxFaucet.claimFaucetTokens();
    }
```

## Recommended Mitigation

* Under `RaiseBoxFaucet::claimFaucetTokens` move the reset functionality to occur before checking whether the limit has been reached. It is good idea to reset Faucet and ETH all together in order to be consistent. This ensures the reset happens correctly after the 24-hour period, regardless of the previous day’s claim activity.

```diff
function claimFaucetTokens() public {
        // Checks
        faucetClaimer = msg.sender;

+       // Use same variable lastDripDay for both - ETH and Faucet token to simplify and make sure they are in sync
+       if (block.timestamp > lastDripDay + 1 days) {
+          lastDripDay = block.timestamp;
+          dailyClaimCount = 0; // Faucet token
+          dailyDrips = 0; // ETH
+       }


        if (!hasClaimedEth[faucetClaimer] && !sepEthDripsPaused) {
-           uint256 currentDay = block.timestamp / 24 hours;

-           if (currentDay > lastDripDay) {
-               lastDripDay = currentDay;
-               dailyDrips = 0;
-               // dailyClaimCount = 0;
-           }



-       if (block.timestamp > lastFaucetDripDay + 1 days) {
-           lastFaucetDripDay = block.timestamp;
-           dailyClaimCount = 0;
-       }

        // Effects
```

# [H-03] Reentrancy when ETH is claimed allows double Faucet token claim

## Description

* New users should be permitted to claim one Faucet token drip and one ETH drip on their first claim, provided that neither limit has been reached.

* However, there is a reentrancy vulnerability during the ETH transfer process. When ETH is sent, the external call allows the user to re-enter the claim logic and perform an additional claim, effectively receiving double the faucet token drip.

```solidity
@>     (bool success, ) = faucetClaimer.call{value: sepEthAmountToDrip}("");
```

## Risk

**Likelihood**:

* The issue can occur for every new user who receives an ETH drip and is a contract.

**Impact**:

* First-time users can exploit the reentrancy to double claim faucet tokens, receiving more tokens than intended. This leads to unfair token distribution and potential depletion of the faucet balance.

## Proof of Concept

Add the following test and helper contract to `RaiseBoxFaucet.t.sol` to reproduce the issue:

```solidity
    function test_audit_reentrancyWhenEthIsClaimedAllowsDoubleFaucetTokenClaim()
        public
    {
        Claimer claimer = new Claimer(raiseBoxFaucet);
        claimer.claim();
        assertEq(
            raiseBoxFaucet.getBalance(address(claimer)),
            raiseBoxFaucet.faucetDrip() * 2
        );
    }

contract Claimer {
    RaiseBoxFaucet private s_raiseBoxFaucet;

    constructor(RaiseBoxFaucet _raiseBoxFaucet) {
        s_raiseBoxFaucet = _raiseBoxFaucet;
    }

    function claim() external {
        s_raiseBoxFaucet.claimFaucetTokens();
    }

    receive() external payable {
        s_raiseBoxFaucet.claimFaucetTokens();
    }
}
```

## Recommended Mitigation

Move the external ETH transfer (the whole IF block below) to the end of the `RaiseBoxFaucet.sol::claimFaucetTokens` function, after all internal state changes are applied. This ensures that reentrancy cannot affect the token claim logic.

```diff
-      // still checks
-      if (!hasClaimedEth[faucetClaimer] && !sepEthDripsPaused) {
-        ...
-      }

        // Effects

        lastClaimTime[faucetClaimer] = block.timestamp;
        dailyClaimCount++;

+       if (!hasClaimedEth[faucetClaimer] && !sepEthDripsPaused) {
+       ...
+       }

        // Interactions
        _transfer(address(this), faucetClaimer, faucetDrip);
```





















# [M-01] Burn function always transfer all tokens to the owner and block claims

## Description

* Under normal conditions, when the protocol owner wants to burn faucet tokens, the function should transfer the specified amount to the owner and then burn exactly that amount.

* In `RaiseBoxFaucet.sol::burnFaucetTokens`, instead of transferring the requested `amountToBurn`, the entire Faucet Token balance is transferred to the owner. The function then burns only the `amountToBurn`, leaving the remaining tokens in the owner’s balance.

```Solidity
    function burnFaucetTokens(uint256 amountToBurn) public onlyOwner {
        require(
            amountToBurn <= balanceOf(address(this)),
            "Faucet Token Balance: Insufficient"
        );

        // transfer faucet balance to owner first before burning
        // ensures owner has a balance before _burn (owner only function) can be called successfully
@>      _transfer(address(this), msg.sender, balanceOf(address(this)));

        _burn(msg.sender, amountToBurn);
    }
```

## Risk

**Likelihood**:

* The issue occurs each time the owner invokes the `RaiseBoxFaucet.sol::burnFaucetTokens` function.

**Impact**:

* The protocol will lose its entire Faucet Token balance, making it unable to provide tokens for users to claim, thus making the system non-operational.

* This vulnerability also violates a core invariant: the owner should not be able to claim faucet tokens. However, due to this flaw, the owner effectively gains control over the Faucet Token supply by receiving the entire balance.

## Proof of Concept

Add the following test to `RaiseBoxFaucet.t.sol` to reproduce the issue:

```Solidity
    function test_audit_burnFunctionAlwaysTransferAllTokensToTheOwnerAndBlockClaims()
        public
    {
        assertEq(raiseBoxFaucet.getFaucetTotalSupply(), INITIAL_SUPPLY_MINTED);

        vm.prank(owner);
        raiseBoxFaucet.burnFaucetTokens(0);

        assertEq(raiseBoxFaucet.getFaucetTotalSupply(), 0);
        assertEq(raiseBoxFaucet.getBalance(owner), INITIAL_SUPPLY_MINTED);
    }
```

## Recommended Mitigation

When performing the transfer, use the provided `amountToBurn` parameter instead of the full balance.

```diff
-  _transfer(address(this), msg.sender, balanceOf(address(this)));
+  _transfer(address(this), msg.sender, amountToBurn);
```



# [M-02] Faucet and ETH drips resets are not synced causing user friction

## Description

* The protocol defines daily limits for both Faucet token and ETH drips, which should reset every 24 hours.

* However, the `RaiseBoxFaucet::dailyClaimCount` is reset exactly after 24 hours have passed, while the ETH drip counter `RaiseBoxFaucet::dailyDrips` is reset based on a strict “day change” condition.

* This desynchronization causes inconsistent reset timing between the two.

```Solidity
   uint256 currentDay = block.timestamp / 24 hours;

@> if (currentDay > lastDripDay) {
             lastDripDay = currentDay;
             dailyDrips = 0;
            // dailyClaimCount = 0;
    }

@>  if (block.timestamp > lastFaucetDripDay + 1 days) {
            lastFaucetDripDay = block.timestamp;
            dailyClaimCount = 0;
    }
```

## Risk

**Likelihood**:

* The issue can occur on any `RaiseBoxFaucet::claimFaucetTokens` call where only one of the limits is reset due to the timing mismatch.

**Impact**

* Users may experience friction when one of the limits (Faucet or ETH) is reset while the other remains locked.

* As a result, new users may be unable to perform the intended “happy path” action of claiming both Faucet tokens and ETH in a single call, reducing usability and creating confusion.

## Proof of Concept

Add the following test to `RaiseBoxFaucet.t.sol` to reproduce the issue:

```solidity
  function test_audit_faucetAndEthDripsResetsAreNotSynced() public {
        // Start at 3 days and 12 hours not to hit the CLAIM_COOLDOWN period
        vm.warp(84 hours);

        vm.prank(user1);
        raiseBoxFaucet.claimFaucetTokens();

        // Move to the next day
        vm.warp(block.timestamp + 12 hours);

        vm.prank(user2);
        raiseBoxFaucet.claimFaucetTokens();

        uint256 dailyFaucetClaimCount = raiseBoxFaucet.dailyClaimCount();
        uint256 dailyEthDripsCount = raiseBoxFaucet.dailyDrips() /
            raiseBoxFaucet.sepEthAmountToDrip();

        // ETH count has been reset, but faucet count not
        assertGt(dailyFaucetClaimCount, dailyEthDripsCount);
    }
```

## Recommended Mitigation

* Unify the reset logic for both Faucet and ETH drips to use the same condition.

* A recommended approach is to perform the reset at the beginning of claimFaucetTokens, based on whether 24 hours have passed since the last reset.

* Using a 24-hour interval is preferable to an “exact day change,” as it ensures consistency and aligns with the `RaiseBoxFaucet::CLAIM_COOLDOWN` check.

```diff

    function claimFaucetTokens() public {
        // Checks
        faucetClaimer = msg.sender;

+       // Use same variable lastDripDay for both - ETH and Faucet token to simplify and make sure they are in sync
+       if (block.timestamp > lastDripDay + 1 days) {
+          lastDripDay = block.timestamp;
+          dailyClaimCount = 0; // Faucet token
+          dailyDrips = 0; // ETH
+      }


        if (!hasClaimedEth[faucetClaimer] && !sepEthDripsPaused) {
-            uint256 currentDay = block.timestamp / 24 hours;

-           if (currentDay > lastDripDay) {
-               lastDripDay = currentDay;
-               dailyDrips = 0;
-               // dailyClaimCount = 0;
-           }



-       if (block.timestamp > lastFaucetDripDay + 1 days) {
-           lastFaucetDripDay = block.timestamp;
-           dailyClaimCount = 0;
-       }

        // Effects
```

# [L-01] Claimers cannot claim when Faucet balance is just enough

## Description

* When there is sufficient Faucet Token balance for at least one claim, users should be able to claim and reduce the faucet balance to zero.

* In the `RaiseBoxFaucet.sol::claimFaucetTokens` function, the balance check uses a less-than-or-equal comparison (<=) against `faucetDrip`. As a result, when the remaining balance is exactly equal to the `faucetDrip` amount, users are unable to claim - even though there is enough balance for one final claim.

```Solidity
 @>     if (balanceOf(address(this)) <= faucetDrip) {
            revert RaiseBoxFaucet_InsufficientContractBalance();
        }
```

## Risk

**Likelihood**:

* The issue occurs each time the Faucet Token balance is exactly equal to the `faucetDrip` value.

**Impact**:

* Users are prevented from claiming tokens despite there being enough balance for one more claim. This causes the faucet to become partially unusable and leaves residual tokens locked in the contract.

## Proof of Concept

Add the following test to `RaiseBoxFaucet.t.sol` to reproduce the issue:

```Solidity
    function test_audit_claimersCannotClaimWhenFaucetBalanceIsJustEnough()
        public
    {
        vm.startPrank(owner);
        raiseBoxFaucet.burnFaucetTokens(INITIAL_SUPPLY_MINTED);
        raiseBoxFaucet.mintFaucetTokens(
            address(raiseBoxFaucet),
            raiseBoxFaucet.faucetDrip()
        );
        vm.stopPrank();

        // Make sure we have enough balance for 1 more Faucet drip
        assertEq(
            raiseBoxFaucet.getFaucetTotalSupply(),
            raiseBoxFaucet.faucetDrip()
        );

        vm.prank(user1);
        vm.expectRevert(
            RaiseBoxFaucet.RaiseBoxFaucet_InsufficientContractBalance.selector
        );
        raiseBoxFaucet.claimFaucetTokens();
    }
```

## Recommended Mitigation

Use a strict less-than comparison (<) instead of less-than-or-equal (<=) when checking the faucet balance.

```diff
-       if (balanceOf(address(this)) <= faucetDrip) {
+       if (balanceOf(address(this)) < faucetDrip) {
            revert RaiseBoxFaucet_InsufficientContractBalance();
        }
```









