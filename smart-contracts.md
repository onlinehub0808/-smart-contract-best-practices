# Smart Contract Best Practices

This document outlines the best practices for smart contract development around security: making sure one gets rid of most of the errors and also strategies for mitigating problems if they do occur.

## Recommendations for Smart Contract Security in Solidity

### Proper Handling of External Calls

External calls can introduce several unexpected errors, even if one calls to a contract that one knows aren’t malicious (one you set up yourself). One can force what seems like a safe call to fail by manipulating the depth of the call stack & can have unexpected behaviour occur when a malicious contract calls back into the contract.

When calling unknown contracts you are potentially executing code that can be entirely malicious. Thus one should assume that in the default case, they are malicious. If you can avoid doing external calls, it should be avoided. If not, it should be the last action one does in a function, AFTER any state changes have been done. Examples of best practices will be given in the following sections.

### Safely using external calls

It should be noted that external calls are potentially dangerous because they can always trigger code, even when using send(). send() can trigger the fallback function, for example. send() is usually more safe as it only has access to gas stipend of 2300 gas, which is insufficient for the send recipient to trigger any state changes (the intention of the 2300 gas stipend was to allow the recipient to register a log). Any external call is potentially vulnerable: especially call(), callcode() & delegatecall().

There is a slight difference in Solidity as to what happens with Contract calls (ie ExternalContract.doSomething()) vs raw calls (address.send() or address.call()).  A raw call never throws an exception: it returns false if the call encounters an exception. Contract calls will automatically propagate a throw (vs a raw call). For example ExternalContract.doSomething() will also throw if doSomething() throws.

The most important point is the same for both, whether using ExternalContract.doSomething() or address.call(), if ExternalContract is untrusted, assume that malicious code will execute.  Note that if you trust an ExternalContract, you also trust any external contracts it calls, and that those call, are all non-malicious.  If any malicious contract exists in the call chain, the malicious contract can attack you (next section on Reentrant Attacks).

The return value of all raw external calls should be checked to see if it failed or not.

function doSomething() {
	if(!address.call.value(100000)()) { throw; }
}

NOTE: beware of this pattern for potentially deadlocking a contract. See that section.

Explicit comments should be made as to why the return value isn’t checked, if that is desired. Checking this is primarily due to the call depth attack.

### Call depth attack.
(it is sometimes also referred as the call stack attack)

Even if it is known that the likelihood of failure in a sub-execution is possible, this can be forced to happen through a call depth attack. There’s a limit to how deep the call stack can become in one transaction (limit of 1024). Thus an attacker can build up a chain of calls and then call a contract, forcing subsequent calls to fail even if enough gas is available. It has to be a call, within a call, within a call, etc.

For example:

((add code snippet of call depth attack))

Thus:

All raw external calls should be examined and handled carefully for errors.  In most cases, the return values should be checked and handled carefully.  We recommend explicit comments in the code when such a return value is deliberately not checked.

As you can see, the call depth attack can be a malicious attack on a contract for the purpose of failing a subsequent call. Thus even if you know what code will be executed, it could still be forced to fail.

### Deadlocking Through Forcing Unexpected Throws

Let’s assume one wants to iterate through an array to pay users accordingly. In some circumstances, one wants to make sure that a contract call succeeding (like having paid the address). If not, one should throw. The issue in this scenario is that if one call fails, you are reverting the whole payout system, essentially forcing a deadlock. No one gets paid, because one address is forcing an error.

((code snippet)) ((insert from https://blog.ethereum.org/2016/06/19/thinking-smart-contract-security/))

The recommended pattern is that each user should withdraw their payout themselves.

### Reentrant Attacks

Allowing contracts to call other contracts is one of the amazing benefits of Ethereum. It allows for emergent behaviour to occur across code. It is a powerful tool to easily attach additional functionality to existing smart contract ecosystems, but could open up malicious attacks. One of these are reentrant attacks: allowing a contract that is called to reenter the contract.

The more complicated issues with external calls (Contract OR raw calls) comes in when one does not know what the other contract might do when it is triggered. This can cause reentrant attacks, which can be separated into a recursive & unexpected state manipulation attack. These can be combined, which is what happened in the DAO hack.

### Recursive Reentrant Attack

This attack occurs when the state has not been properly set before the external call occurs in a function. The attacker can reenter and call the function again, being to recursively call a piece of code, whilst it was expected that they could only do it once. They can only recursively call until the gas limit has been reached or before the call depth is reached.

This is particularly problematic in the case where it is not known that something like address.call.value()() to send ether, can trigger a fallback function. However, this is not just limited to a fallback function. It can happen with any external (Contract or raw).

((code snippet))

To protect against recursive reentry, the function needs to set the state such that if the function is called again in the same transaction, it wouldn’t continue to execute.

### Unexpected State Manipulation Reentrant Attack

Unexpected state manipulation reentrant attack can occur when another function G in the contract shares the state of the current calling function F.

So, let’s imagine an attacker starts off by calling function F. It has an external call in it, which the attacker then uses to call function G. Since F & G share state, calling G, can manipulate the behaviour in F. By the time F is finished, the attacker might have manipulated state by calling G without function F being aware of it. This is one of the attacks that was done on the DAO. When splitting function was called, right at the end (after the external call), the token balances were zeroed out. However, before that was done, the attacker reentered and transfer out his balance to another address, basically then letting the split function zero out an account that already has zero funds in it. This allowed the attacker to again call the splitting function in multiple transactions, as long as he kept transferring the tokens around before the final split.

((code snippet))

To protect against this, the recommendation is again to make sure that any state changes that are to be applied needs to happen BEFORE the attacker can come in and manipulate the state.

If there are more than 2 functions that share the state, it can be seen that an attacker can potentially wreak massive havoc.  Whether an attacker is actually able to gain financially from an attack, is a separate question from an attacker rendering a contract unusable by abusing its state.

Functions that do not share any state, are safe from this attack (even if state changes happen after the external call). It is not generally a safe pattern, so it is still recommended to do state changes before doing any external calls even if no functions share states in your contract.

A non-example? ((insert here))
```
contract ReentrantSafe {
  uint fState;
  uint gState;

  f()  // only changes fState
  g() // only changes gState
}
```

In general, to protect against reentry attacks:

We recommend that external calls (of any sort) should be made at the end of function calls. If it can’t be done, extreme care needs to be taken to make sure that the functions do not share state that can be manipulated by the attacker before proceeding onwards.
In general, for both reentry attacks, the easiest fix is to make sure that you only do external calls as the last action in a function.

If one has an expectation of what type of computations will be done in an external call, to limit the gas appropriately (and not forwarding all the gas by default). This however limits the potential for any emergent useful use cases, so use wisely.

To summarise the protection against external call attacks:
- Always make sure to check the result of the call, even when using send(), or even Contract calls. Under no assumption should it be expected that everything went exactly as planned.
- Move external calls to the end of functions.
If this can’t be done, make extremely sure that state manipulation across functions won’t be possible.


## Design Patterns to avoid external calls:

The more ideal scenario is to try and avoid external calls where possible. This is particularly apparent when ether needs to be sent around, across contracts. It’s a potential cascade of calls.

### Push vs Pull & Asynchrony

A recommendation around this is to move to a pull vs push system. For example, let’s say a DAO needs to be paid out. The funds are sent to that address. Instead of in the same transaction, sending & splitting the funds when it reaches the DAO, the DAO simply logs that it received funds. Now, each participant who was supposed to receive must go and withdraw their funds from the DAO when they want to. So, now the call stack is reduced and attack space is reduced.

Generally, although asynchrony sometimes requires multiple transactions, it is a safer pattern in general, because you reduce the potential attack space. In the future of Ethereum, with lower block times & potentially having to interact across shards, asynchrony might become a more needed pattern besides just for security concerns.

### Gas Manipulation Attacks

Manipulating the amount of elements in an array can increase gas costs substantially, forcing a lock up to a certain gas limit. Taking the previous example, of wanting to pay out some stakeholders iteratively, it might seem fine, assuming that the amount of stakeholders won’t inflate too much. The attacker would buy up, say 10000 tokens, and then split all 10000 tokens amongst 10000 addresses, causing the amount of iterations to increase, potentially forcing a lock.

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

### Rounding & Integer Division Error

Currently in Solidity, when dividing integers (uint) it rounds down to the nearest integer. If not properly checked it can eventually lead to potential leakage over time.

Fixed point types are currently being implemented in Solidity (ie fixed, ufixed, etc). Until it becomes available, the best bet is make sure that the rounding down is taken into account, or that one increases the granularity of the division moving the decimal places.

### Function & Event Labels

We recommend that event names are not too similar to their function counterparts as other authors have suggested, to avoid confusion with functions that have a very similar name. In some smart contracts, they are simply differentiated with capitalising one, over the other. Perhaps a standard such as “Log” in front events would help mitigate this potential issue.

### Visibility

Explicitly label the visibility of functions and state variables. Functions can be specified as being external, public, internal or private. For state variables, external is not possible.

### Leech attack

If your contract is an oracle, it may want protection from leeches that will use your contract’s data for free. If not encrypted, the data would always be readable, but one can restrict the usage of this information in other smart contracts.

Part of the solution is to carefully review the visibilities of all function and state variable.

### Fallback functions

If one is using a fallback function to deal with ether, one has to keep in mind that send() does not forward any gas. It only has access to a stipend of 2300 gas, which is enough to usually just be able to emit an event that a contract has received some ether.  It is recommended that fallback functions only spend up to 2300 gas, to not break send() behavior. It is recommended that a proper function be used if computation consuming more than 2300 gas is desired.

### Naming of untrusted contracts

We recommend a coding convention that makes it more visible which contracts are untrusted.  For example, some abstract contracts are implementable by 3rd parties.  Having a prefix or some other convention, would make it clearer to identify untrusted contracts, from the rest of the system’s trusted contracts. At minimum, some comments to indicate this.

## Mitigation Strategies

In order to mitigate potential besides the potential errors the occur, we give recommendations what to do during development, deployment & on-chain.

### In Development:

### Keep It Simple, Stupid!
Contracts constructed using small, auditable components should be more debuggable than monolithic contracts. And if there has ever been an inappropriate place for inscrutable optimization hacks and space-saving syntax, it is in smart contracts. Legibility is auditability.

### Comprehensive Test Suite
A comprehensive test suite helps everyone, from you, to other developers, to users, to auditors, to new developers that want to contribute to your project.

### Editors detecting vulnerabilities

Work is being done to put warnings into certain editors. Browser Solidity is getting these features soon.

Oyente is an upcoming tool that analyses one’s code and finds some of these common vulnerabilities such transaction order dependence and not checking for exceptions.

### Functional Languages

Solidity is a procedural language.

Functional languages gives certain guarantees over procedural. The reasons why it gives better guarantees is because:

Essentially, the guarantees are provided by a richer type system. Ensuring that a program/function is properly typed proves that this formula has a certain structure.

The type system of procedural languages like Solidity are less fine-grained than those of functional languages.

With Solidity, you can ensure that a function that returns a uint256 will not return an address, for example. (notice that there is still no guarantee that the function wouldn’t throw, whereas in functional languages this is usually guaranteed).

With a more sophisticated type system, rich enough to express behavior such as ‘fungible resources’, (in short, a resource that cannot be copied, see resource on linear types below), one can prove more advanced properties of functions.

In a pure functional language like Haskell the functions used are stateless. This means that they act like mathematical functions in that they are completely determined by their return value given specific input values. From this it becomes possible to prove mathematically that your program exhibit certain behaviour.

FOR MORE READING:
Greg Merediths linear types can change the blochchain https://plus.google.com/u/0/events/cmqejp6d43n5cqkdl3iu0582f4k

Curry-Howard correspondence (types as propositions, programs as proofs)
Linear logic

### Deployment:

Before moving the mainnet, it can be prudent to first deploy the code to the testnet. Here, the code can be tested. In order to incentivize attackers, one can offer bug bounties for finding exploits in the testnet code.

When deploying on the main chain, one might want to the use the optimizer. This reduces the bytecode required and can sometimes be useful to fit large contracts into

### On-Chain:

There should be the assumption that even if you used all the tools at hand, bugs could still slip in. Thus, one should be prepared for this, by using various on-chain mechanisms to either revert or replace faulty code.

### Permissioned Guard

In order to be able to change code, once it is deployed, someone needs to have permission to do so. The simplest version is that you have a multi-signature control of code. It is a tradeoff since this opens up the system to potential manipulation and one has to trust the multi-signature stakeholders to not be malicious. One can extend this to as broad a set of stakeholders as one wants. For example, a set of token holders can all vote to upgrade a contract if some quorum is met.

A simple switch can also employed that allows the contracts to lock themselves, instead of having to worry about a fix. Ie, lock first, then upgrade.

### Circuit Breakers

Upgrading functionality is useful, especially in terms of fixing potential bugs. However, attacks can occur before the participants have any time of fixing it.

Circuit breakers are automated stop-gaps that stops any contract code automatically from being executed if certain conditions are met.

Under certain circumstances it can force the whole contract to be locked down if unexpected behaviour starts to occur. If not resolved, it will remain in stasis. Circuit breakers are not just useful for things like protecting ether. Any rate limit can be picked up. A generic can be done by sending requests through a proxy and logging function signatures and ether transfer and then locking requests if anything out of the ordinary is happening.

### Speed Bumps/Rate Limiting

One of the best lessons from the DAO is that slowing down processes arbitrarily may allow for time to respond to them when they behave unexpectedly. There are obvious user experience tradeoffs here, and speed bumps may be best implemented inside of circuit breaker logic so that they are only triggered in exceptional circumstances.

### Assert Guards

Akin to watching for unknown activities, an assert guard performs like a circuit breaker, but instead focuses on scenarios where an attacker can force a set of tests to fail. This however, does mean that the tests have to be written in Solidity as well, and assumes that tests are bug free as well. If an assert failure is triggers, the developers are allowed back in to upgrade the code, and only in those scenarios.

### Noted Security Blog Posts

#### Writing Safer Contracts

- [How to Write Safe Smart Contracts](https://chriseth.github.io/notes/talks/safe_solidity): Blog post from Devcon 1 from the creator or Solidity
- [Making Smart Contracts Smarter](http://www.comp.nus.edu.sg/~loiluu/papers/oyente.pdf) (Loi Luu, Duc-Hiep Chu, Prateek Saxena, Hrishi Olickel, Aquinas Hobor)
- [We need fault-tolerant smart contracts](https://medium.com/@peterborah/we-need-fault-tolerant-smart-contracts-ec1b56596dbc) (Peter Borah)
- [We need Escape Hatches](http://hackingdistributed.com/2016/06/22/smart-contract-escape-hatches/) (Hacking Distributed)
- [Thinking about Smart Contract Security](https://blog.ethereum.org/2016/06/19/thinking-smart-contract-security/) (Vitalik Buterin)
- [Assert Guards: Towards Automated Code Bounties & Safe Smart Contract Coding on Ethereum](https://medium.com/@ConsenSys/assert-guards-towards-automated-code-bounties-safe-smart-contract-coding-on-ethereum-8e74364b795c) (Simon de la Rouviere)
- [Safer Smart Contracts through type-driven development](http://publications.lib.chalmers.se/records/fulltext/234939/234939.pdf) (Jack Pettersson and Robert Edström)
- [In Bits We Trust?](https://medium.com/@coriacetic/in-bits-we-trust-4e464b418f0b) (David Xiao)

#### Common Contract Errors

- [Smart Contract Security](https://blog.ethereum.org/2016/06/10/smart-contract-security/) (Christian Reitwiessner)
- [More Ethereum Attacks: Race-to-empty is the Real Deal](http://vessenes.com/more-ethereum-attacks-race-to-empty-is-the-real-deal/)
- [Devcon1 and Ethereum contract Security](http://martin.swende.se/blog/Devcon1-and-contract-security.html)
- [Potential Attacks against Rouleth contract](https://github.com/Bunjin/Rouleth/blob/master/Security.md)
- [Ethereum Griefing Wallets: Send w/Throw Is Dangerous](http://vessenes.com/ethereum-griefing-wallets-send-w-throw-considered-harmful/)

#### DAO-related Security Posts

- [Deja Vu DAO Smart Contracts Audit Results](https://blog.slock.it/deja-vu-dao-smart-contracts-audit-results-d26bc088e32e#.x9frbu72d) (Stephen Tual)
- [DAO Call for Moratorium](http://hackingdistributed.com/2016/05/27/dao-call-for-moratorium/) (Dino Mark, Vlad Zamfir, and Emin Gün Sirer)
- [Analysis of the DAO Exploit](http://hackingdistributed.com/2016/06/18/analysis-of-the-dao-exploit/) (Phil Daian/Hacking Distributed)
- [Deconstructing the DAO Attack](http://vessenes.com/deconstructing-thedao-attack-a-brief-code-tour/) (Peter Vessenes)

#### Other

[Least Authority Security Audit](https://github.com/LeastAuthority/ethereum-analyses)

### License

Licensed under [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International](https://creativecommons.org/licenses/by-nc-sa/4.0/)
