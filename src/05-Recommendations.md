
## Recommendations for Smart Contract Security in Solidity

<a name="avoid-external-calls"></a>

### Avoid external calls, when possible

External calls (including raw `call()`, `callcode()`, `delegatecall()`) can introduce several unexpected risks or errors. For calls to untrusted contracts, you may be executing malicious code in that contract _or_ any other contract that it depends upon. As such, it is strongly encouraged to minimize external calls. Over time, it is likely that a paradigm will develop that leads to safer external calls - but the risk currently is high.

If you must make an external call, ensure that external calls are the last call in a function - and that you've finalized your contract state before the call is made.

When possible, avoid external Contract calls (eg `ExternalContract.doSomething()`), including raw `call()`, `callcode()`, `delegatecall()`.

<a name="use-external-calls-safely"></a>

### Use external calls safely

*Raw calls* (`address.call()`, `address.callcode()`, `address.delegatecall()`) never throw an exception, but will return `false` if the call encounters an exception. On the other hand, *contract calls* (e.g., `ExternalContract.doSomething()`) will automatically propogate a throw (for example, `ExternalContract.doSomething()` will also `throw` if `doSomething()` throws)

Whether using *raw calls* or *contract calls*, assume that malicious code will execute if `ExternalContract` is untrusted.  Additionally, if you trust an `ExternalContract`, you also trust any contracts it calls.  If any malicious contract exists in the call chain, the malicious contract can attack you (see [Reentrant Attacks](https://github.com/ConsenSys/smart-contract-best-practices/blob/master/smart-contracts.md#reentrant-attacks)).

<a name="avoid-call-value"></a>

### Use `send()`, avoid `call.value()`

When sending Ether, use `someAddress.send()`, avoid `someAddress.call.value()()`.

As noted, external calls, such as `someAddress.call.value()()` are potentially dangerous because they always trigger code. `send()` also triggers code, specifically the [fallback function](https://github.com/ConsenSys/smart-contract-best-practices/blob/master/smart-contracts.md#keep-fallback-functions-simple), but `send()` is safe because it only has access to gas stipend of 2,300 gas. This is insufficient for the send recipient to trigger any state changes (the intention of the 2,300 gas stipend was to allow the recipient to log an event).

```
// bad
someAddress.call.value(100)(); // this is doubly dangerous, as it will forward all remaining gas and doesn't check for result
someAddress.call.value(100)(bytes4(sha3("deposit()"))); // if deposit throws an exception, the raw call() will only return false and transaction will NOT be reverted

// good
if(!someAddress.send(100)) {
    // Some failure code
}

ExternalContract(someAddress).deposit.value(100); // raw call() is avoided, so if deposit throws an exception, the whole transaction IS reverted

### Always test if `send()` and other raw calls have succeeded

`send()` and raw external calls can fail (e.g., when the call depth of 1024 is breached), so you should always test if it succeeded. If you don't test the result, it's recommended to note in a comment.

If you throw on a `send()` failure, be careful as you may create a [denial-of-service](https://github.com/ConsenSys/smart-contract-best-practices/blob/master/smart-contracts.md#dos-with-unexpected-throw) vulnerability.

```
// bad
someAddress.send(55);
someAddress.call.value(55)(); // this is doubly dangerous, as it will forward all remaining gas and doesn't check for result

if(!someAddress.call.value(55)(calldata)) { // checks the return value of call() but is not recommended since call() should be avoided
    // Some failure code
}

// good
if(!someAddress.send(55)) {
    // Some failure code
}

```

<a name="dos-with-unexpected-throw"></a>

### DoS with (Unexpected) Throw

Example 1: Here is an example where the routine throw on a failed `send()` can cause a denial-of-service.
In this auction, an attacker can [reject payments](https://solidity.readthedocs.io/en/latest/contracts.html#fallback-function) to it and will always be
the highest bidder. Any resource that is owned by the highest bidder, will permanently be owned by the attacker.
This is an example where it should be the responsibility of the recipient to accept payment.

((code snippet))

Example 2: Let’s assume one wants to iterate through an array to pay users accordingly. In some circumstances, one wants to make sure that a contract call succeeding (like having paid the address). If not, one should throw. The issue in this scenario is that if one call fails, you are reverting the whole payout system, essentially forcing a deadlock. No one gets paid, because one address is forcing an error.

```
address[] private refundAddresses;
mapping (address => uint) public refunds;

// bad
function refundAll() public {
    for(uint x; x < refundAddresses.length; x++) { // arbitrary length iteration based on how many addresses participated
        if(refundAddresses[x].send(refunds[refundAddresses[x]])) {
            throw; // doubly bad, now a single failure on send will hold up all funds
        }
    }
}

// good
mapping (address => uint) private refunds;

function getRefund() public {
    if(refunds[msg.sender] > 0) {
        uint amountToSend = refunds[msg.sender];
        refunds[msg.sender] = 0;

        if(!msg.sender.send(refunds[msg.sender])) {
            refunds[msg.sender] = amountToSend;
        }
    }
}
```

The recommended pattern is that each user should withdraw their payout themselves.

<a name="favor-pull-over-push-payments"></a>

### Favor *pull* payments over *push* payments

As noted, payments can fail for multiple reasons - including if the call stack exceeds 1024 or a failure on externally. To prevent payments failing, create a balance in an action that can be separately withdrawn by the external party using another action. This requires two actions on external party's part - a request to withdraw and a withdrawal - but reduces a number of potential errors.

In the future, with lower block times and having to interact across shards, this asynchronous payment might become a more important pattern for reasons other than security.

```
// bad
contract auction {
    address highestBidder;
    uint highestBid;

    function bid() {
        if (msg.value < highestBid) throw;

        if (highestBidder != 0) {
            if (!highestBidder.send(highestBid)) { // if this call consistently fails, no one else can bid
                throw;
            }
        }

       highestBidder = msg.sender;
       highestBid = msg.value;
    }
}

// good
contract auction {
    address highestBidder;
    uint highestBid;
    mapping(address => uint) refunds;

    function bid() {
        if (msg.value < highestBid) throw;

        if (highestBidder != 0) {
            refunds[highestBidder] += highestBid; // record the refund that this user can claim
        }

        highestBidder = msg.sender;
        highestBid = msg.value;
    }

    function withdrawRefund() {
        uint refund = refunds[msg.sender];
        refunds[msg.sender] = 0;
        if (!msg.sender.send(refund)) {
            refunds[msg.sender] = refund;
        }
    }
}
```

Source: [Smart Contract Security](https://blog.ethereum.org/2016/06/10/smart-contract-security/)

<a name="keep-fallback-functions-simple"></a>

### Keep fallback functions simple

Fallback functions are the default functions called when a contract is sent a message with no arguments, and only has access to 2,300 gas when called from a `.send()` call. As such, the most you should do in most fallback functions is call an event. Use a proper function if a computation or more gas is required.

```
// bad
function () { balances[msg.sender] += msg.value; }

// good
function() { throw; }
function() { LogSomeEvent(); }
function deposit() { balances[msg.sender] += msg.value; }
```

<a name="beware-rounding-with-integer-division"></a>

### Beware rounding with integer division

All integer divison rounds down to the nearest integer - use a multiplier to keep track, or use the future fixed point data types.

```
// bad
uint x = 5 / 2; // Result is 2, all integer divison rounds DOWN to the nearest integer

// good
uint multiplier = 10;
uint x = (5 * multiplier) / 2;

// in the near future, Solidity will have fixed point data types like fixed, ufixed
```

Source:

<a name="beware-division-by-zero"></a>

### Beware division by zero

Currently, Solidity [returns zero](https://github.com/ethereum/solidity/issues/670) and does not
`throw` an exception when a number is divided by zero.

<a name="differentiate-functions-events"></a>

### Differentiate functions and events

Favor capitalization and a prefix in front of events (we suggest *Log*), to prevent the risk of confusion between functions and events. For functions, always start with a lowercase letter, except for the constructor.

```
// bad
event transferHappened() {}
function Transfer() {}

// good
event LogTransfer() {}
function transfer() {}
```

Source: [Deconstructing the DAO Attack: A Brief Code Tour](http://vessenes.com/deconstructing-thedao-attack-a-brief-code-tour/) (Peter Vessenes)

<a name="mark-visibility"></a>

### Explicitly mark visibility in functions and state variables

Explicitly label the visibility of functions and state variables. Functions can be specified as being `external`, `public`, `internal` or `private`. For state variables, `external` is not possible.

```
// not great
uint x; // the default is private for state variables, but it should be made explicit
function transfer() { // the default is public

}

// good
uint private y;
function transfer() public {

}

function internalAction() internal {

}
```
<a name="mark-untrusted-contracts"></a>

### Mark untrusted contracts

Mark which contracts are untrusted. For example, some abstract contracts are implementable by 3rd parties. Use a prefix, or at minimum a comment, to highlight untrusted contracts.

```
// bad
Bank.withdraw(100); // Unclear whether trusted or untrusted

// good
ExternalBank.withdraw(100); // untrusted external call
Bank.withdraw(100); // external but trusted bank contract maintained by XYZ Corp
```
