
It is possible to forcibly send Ether to a contract without triggering its fallback function. This
is an important consideration when placing important logic in the fallback function or making
calculations based on a contract's balance. Take the following example:

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

Contract logic seems to disallow payments to the contract and therefore disallow "something bad"
from happening. However, a few methods exist for forcibly sending ether to the contract and
therefore making its balance greater than zero.

The `selfdestruct` contract method allows a user to specify a beneficiary to send any excess ether.
`selfdestruct`
[does not trigger a contract's fallback function](https://solidity.readthedocs.io/en/develop/security-considerations.html#sending-and-receiving-ether).

!!! Warning
    It is also possible to [precompute](https://github.com/Arachnid/uscc/tree/master/submissions-2017/ricmoo) a contract's address and send Ether to that address before deploying the contract.

    See [SWC-132](https://swcregistry.io/docs/SWC-132)
