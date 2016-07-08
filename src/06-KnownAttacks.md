
## Known Attacks

### Call depth attack

(it is sometimes also referred as the call stack attack)

Even if it is known that the likelihood of failure in a sub-execution is possible, this can be forced to happen through a call depth attack. There’s a limit to how deep the call stack can become in one transaction (limit of 1024). Thus an attacker can build up a chain of calls and then call a contract, forcing subsequent calls to fail even if enough gas is available. It has to be a call, within a call, within a call, etc.

For example, looking at the auction code from previously:

```
// DO NOT USE. THIS IS VULNERABLE.
contract auction {
    address highestBidder;
    uint highestBid;
    mapping(address => uint) refunds;

    function bid() {
	if (msg.value < highestBid) throw;
	if (highestBidder != 0)
	    refunds[highestBidder] += highestBid;
	highestBidder = msg.sender;
	highestBid = msg.value;
    }

    function withdrawRefund() {
	uint refund = refunds[msg.sender];
	refunds[msg.sender] = 0;
	msg.sender.send(refund); // vulnerable line.
	refunds[msg.sender] = refund;
    }
}
```

The send() can fail if the call depth is too large, causing ether to not be sent. However it would be marked as if it did send. As previously shown, the external call should be checked for errors. This example, the state would just revert to the previous state.

```
contract auction {
    address highestBidder;
    uint highestBid;
    mapping(address => uint) refunds;

    function bid() {
        if (msg.value < highestBid) throw;
        if (highestBidder != 0)
	    refunds[highestBidder] += highestBid;

        highestBidder = msg.sender;
        highestBid = msg.value;
    }

    function withdrawRefund() {
	uint refund = refunds[msg.sender];
	refunds[msg.sender] = 0;
	if (!msg.sender.send(refund))
	   refunds[msg.sender] = refund;
    }
}
```

Thus:

All raw external calls should be examined and handled carefully for errors.  In most cases, the return values should be checked and handled carefully.  We recommend explicit comments in the code when such a return value is deliberately not checked.

As you can see, the call depth attack can be a malicious attack on a contract for the purpose of failing a subsequent call. Thus even if you know what code will be executed, it could still be forced to fail.

### Reentrant Attacks

A key benefit of Ethereum is the ability for one contract to call another - but this also can introduce risks, especially when you don't know exactly what the external call will do. Reentrant attacks, which allow a function to be called in a contract when a function in that contract is running, are one example of ths. This can occur when the same function s called again, or when another function that shares state with the previously called function is called.

(The DAO hack combined both these attacks)

#### Recursive Reentrant Attack

This attack occurs when the state has not been properly set before the external call in a function. The attacker can reenter and call the function again, even though the programmer never intended for this to happen. For example, you can withdraw money and recursively call the withdraw again, as the balance may only be updated at the end of the `withdraw` function. The attacker can recursively call until the gas limit has been reached or before the call depth is reached.

Example:

```
mapping (address => uint) private userBalances;

function getBalance(address user) constant returns(uint) {
    return userBalances[user];
}

function addToBalance() public {
    userBalances[msg.sender] += msg.value;
}

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    if (!(msg.sender.call.value(amountToWithdraw)())) { throw; } // the ether is sent without zeroing out the user's balance
    userBalances[msg.sender] = 0;
}
```

In the DAO hack, this occurred when the fallback function was called by `address.call.value()()`, but can occur for any external call.

To protect against recursive reentry, the function needs to set the state such that if the function is called again in the same transaction, it won’t continue to execute.

Source: [Race to Empty is the Real Deal](http://vessenes.com/more-ethereum-attacks-race-to-empty-is-the-real-deal/) (Peter Vessenes)

#### Unexpected State Manipulation Reentrant Attack

This attack occurs when a function expects a certain state, but another contract function alters this state, while the original function is still running. A malicious party starts the first function, then calls the second function before the first function completes.

In [The DAO](https://github.com/slockit/DAO), the splitting function zeroed out token balances. However, before that was complete, the attacker reentered and transfered out his/her balance to another address, meaning the split function zeroed out an account that already had zero funds. The attacker could then call the splitting function in multiple transactions, as long as s/he kept transferring the tokens around before the final split.

```
TODO: Add snippet
```

If external functions (and the functions they call) share the same state, an attacker can cause substantial damage. Functions that do not share any state, are safe from this attack (even if state changes happen after the external call). This is not generally a safe pattern, so it is still recommended to make state changes before any external calls, even if no functions share state in your contract.

To mitigate this attack, you should:

- Always check the result of a contract call, even when using `send()` (don't ever assume things went well)
- Only make external calls at the end of functions, once state has been set

You can alternatively ensure that functions do not modify the same state variables:

```
contract StateManipulationReentrantSafe {
    uint fState;
    uint gState;

    f()  // only changes fState
    g() // only changes gState
}
```

Also, you can use a [mutex](https://en.wikipedia.org/wiki/Mutual_exclusion) (often used in concurrent programming) where you lock certain variables:

```
// Note: This is a rudimentary example, and mutexes are particularly useful where there is substantial logic and/or shared state
mapping (address => uint) private balances;
bool private lockBalances;

function deposit() public returns (bool) {
    if (!lockBalances) {
	lockBalances = true;
	balances[msg.sender] += msg.value;
	lockBalances = false;
        return true;
    }
    throw;
}

function withdraw(uint amount) public returns (bool) {
    if (!lockBalances && amount > 0 && balances[msg.sender] >= amount) {
        lockBalances = true;
        balances[msg.sender] -= amount;

        if (!msg.sender.send(amount)) {
            throw;
        }
        lockBalances = false;
        return true;
    }

    throw;
}
```

Mutexes have their own disadvantages with the potential for deadlocks and reduced throughput - so choose the approach that works best for your use case and text extensively.

<a name="timestamp-dependence"></a>

### DoS with Block Gas Limit

All Ethereum transactions must consume an amount of gas lower than the block gas limit (BGL).  An attacker can cause a denial-of-service against the contract, if the attacker can manipulate the gas used by the contract to provide the service.

Example: Manipulating the amount of elements in an array can increase gas costs substantially, forcing a DoS with the BGL. Taking the previous example, of wanting to pay out some stakeholders iteratively, it might seem fine, assuming that the amount of stakeholders won’t increase too much. The attacker would buy up, say 10000 tokens, and then split all 10000 tokens amongst 10000 addresses, causing the amount of iterations to increase, potentially exceeding the BGL.  Note that a contract cannot rely on gas refunds to protect against this DoS, because gas refunds are only provided at the end.

To mitigate around this, a pull vs push model comes in handy. For example:

((code snippet))

An alternative approach is to have a payout loop that can be split across multiple transactions, like so:

```
struct Payee {
    address addr;
    uint256 value;
}
Payee payees[];
uint256 nextPayeeIndex;

function payOut() {
    uint256 i = nextPayeeIndex;
    while (i < payees.length && msg.gas > 200000) {
	payees[i].addr.send(payees[i].value);
	i++;
    }
    nextPayeeIndex = i;
}
```

### Timestamp Dependence Bug

In Solidity, one has access to the timestamp of the block. If it is used as an important part of the contract, the developer needs to know that it can be manipulated by the miner.

((code snippet))

A way around this could be to use block numbers and estimate time that has passed based on the average block time. However, this is NOT future proof as block times might change in the future (such as the current planned 4 second block times in Casper). So, consider this when using block numbers as as timekeeping mechanism (how long your code will be around).

### Transaction-Ordering Dependence (TOD)

Since a transaction is in the mempool for a short while, one can know what actions will occur, before it is properly recorded (included in a block). This can be troublesome for things like decentralized markets, where a transaction to buy some tokens can be seen, and a market order implemented before the other transaction gets included. Protecting against is difficult, as it would come down to the specific contract itself. For example, in markets, it would be better to implement batch auctions (this also protects against high frequency trading concerns). Another potential way to use a pre-commit scheme (“I’m going to submit the details later”).


### Leech attack

If your contract is an oracle, it may want protection from leeches that will use your contract’s data for free. If not encrypted, the data would always be readable, but one can restrict the usage of this information in other smart contracts.

Part of the solution is to carefully review the visibilities of all function and state variable.
