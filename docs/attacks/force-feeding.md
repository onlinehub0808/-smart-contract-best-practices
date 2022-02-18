When a Solidity smart contract receives Ether, either its fallback function or `receive` will be invoked by default.
If neither of the two functions is defined, an exception is thrown, and the transaction reverts.

However, these precautions can be evaded.
Ether can be forcibly sent to a smart contract without triggering its functions.
This effect must be considered when business logic is placed into the fallback or `receive` functions.
Calculations based on the contract's balance must consider that additional funds might have been received.

In general, we strongly advise against using the contract's balance as a guard.

Consider the following example:

```sol
contract Vulnerable {
    function () payable {
        revert();
    }

    function somethingBad() {
        require(this.balance > 0);
        // Do something bad
    }
}
```

Contract logic seems to disallow payments to the contract and disallow "something bad" from happening.
However, a few methods exist for forcibly sending Ether to the contract and making its balance greater than zero.

The `selfdestruct` contract method allows a user to specify a beneficiary to send any excess ether.
`selfdestruct` [does not trigger a contract's fallback function](https://solidity.readthedocs.io/en/develop/security-considerations.html#sending-and-receiving-ether).
Additionally, depending on the balance required and the attacker's capabilities, they can start mining and set the target address as beneficiary.

!!! Warning
    It is also possible to [precompute](https://github.com/Arachnid/uscc/tree/master/submissions-2017/ricmoo) a contract's address and send Ether to that address before deploying the contract.

    See [SWC-132](https://swcregistry.io/docs/SWC-132)
