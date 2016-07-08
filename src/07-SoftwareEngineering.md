
## Software Engineering Techniques

Designing your contract for unknown, often unknowable, failure scenarios is a key aspect of defensive programming, which aims to reduce the risk from newly discovered bugs. We list potential techniques you can use to mitigate various unknown failure scenarios - and many of these can be used together.

Be thoughtful about what techniques you incorporate, as certain techniques require more Solidity code, leading to a greater risk of bugs.

### Permissioned Guard (changing code once deployed)

Code will need to be changed if errors are discovered or if improvements need to be made. The simplest technique is to have a registry contract that holds the address of the latest contract. A more seamless approach for contract users is to have a contract that forwards calls and data onto the latest version of the contract.

Whatever the technique, it's important to have modularization and good separation between components (data, logic) - so that code changes do not break functionality, orphan data, or require substantial costs to port.

It's also critical to have a secure way for parties to decide to upgrade the code - and code changes may be approved by a single trusted party, a group of members, or a vote of the full set of stakeholders. You may often combine a permissioned guard with a contract pause or lock, ensuring that activity does not occur while changes are being made - or after a contract is upgraded.

**Example 1: Use a registry contract to store latest version of a contract**

In this example, the calls aren't forwarded, so users should fetch the current address each time before interacting with it.

```
contract SomeRegister {
    address backendContract;
    address[] previousBackends;
    address owner;

    function SomeRegister() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        if (msg.sender != owner) {
            throw;
        }
        _
    }

    function changeBackend(address newBackend) public
    onlyOwner()
    returns (bool)
    {
        if(newBackend != backendContract) {
            previousBackends.push(backendContract);
            backendContract = newBackend;
            return true;
        }

        return false;
    }
}
```

**Example 2: Use a `DELEGATECALL` to forward data and calls**

```
contract Relay {
    address public currentVersion;
    address public owner;

    modifier onlyOwner() {
        if (msg.sender != owner) {
            throw;
        }
        _
    }

    function Relay(address initAddr) {
        currentVersion = initAddr;
        owner = msg.sender; // this owner may be another contract with multisig, not a single contract owner
    }

    function changeContract(address newVersion) public
    onlyOwner()
    {
        currentVersion = newVersion;
    }

    function() {
        if(!currentVersion.delegatecall(msg.data)) throw;
    }
}
```

Source: [Stack Overflow](http://ethereum.stackexchange.com/questions/2404/upgradeable-contracts)

### Circuit Breakers (Pause contract functionality)

Circuit breakers stop execution if certain conditions are met, and can be useful when new errors are discovered. For example, most actions may be suspended in a contract if a bug is discovered, and the only action now active is a withdrawal. They can come at the cost of injecting some level of trust, though smart design can minimize the trust required.

A circuit breaker can also be combined with assert guards, automatically pausing the contract if certain assertions fail (e.g., ether to token peg changes).

Example:

```
bool private stopped = false;
address private owner;

function toggleContractActive() public
isAdmin() {
    // You can add an additional modifier that restricts stopping a contract to be based on another action, such as a vote of users
    stopped = !stopped;
}

modifier isAdmin() {
    if(msg.sender != owner) {
        throw;
    }
    _
}

modifier stopInEmergency { if (!stopped) _ }
modifier onlyInEmergency { if (stopped) _ }

function deposit() public
stopInEmergency() {
    // some code
}

function withdraw() public
onlyInEmergency() {
    // some code
}
```

Source: [We Need Fault Tolerant Smart Contracts](https://medium.com/@peterborah/we-need-fault-tolerant-smart-contracts-ec1b56596dbc#.ju7t49u82) (Peter Borah)

### Speed Bumps(Delay contract actions)

Speed bumps slow down actions, so that if malicious actions occur, there is time to recover. For example, [The DAO](https://github.com/slockit/DAO/) required 27 days between a successful request to split the DAO and the ability to do so. This ensured the funds were kept within the contract, increasing the likelihood of recovery (other fundamental flaws made this functionality useless without a fork in Ethereum). Speed bumps can be combined with other techniques (like circuit breakers or root access) for maximal effectiveness.

Example:

```
struct RequestedWithdrawal {
    uint amount;
    uint time;
}

mapping (address => uint) private balances;
mapping (address => RequestedWithdrawal) private requestedWithdrawals;
uint constant withdrawalWaitPeriod = 28 days; // 4 weeks

function requestWithdrawal() public {
    if (balances[msg.sender] > 0) {
	uint amountToWithdraw = balances[msg.sender];
	balances[msg.sender] = 0; // for simplicity, we withdraw everything;
	// presumably, the deposit function prevents new deposits when withdrawals are in progress

	requestedWithdrawals[msg.sender] = RequestedWithdrawal({
	    amount: amountToWithdraw,
	    time: now
	  });
    }
}

function withdraw() public {
    if(requestedWithdrawals[msg.sender].amount > 0 && now > requestedWithdrawals[msg.sender].time + withdrawalWaitPeriod) {
        uint amountToWithdraw = requestedWithdrawals[msg.sender].amount;
        requestedWithdrawals[msg.sender].amount = 0;

        if(!msg.sender.send(amountToWithdraw)) {
            throw;
        }
    }
}
```

### Rate Limiting

Rate limiting halts or requires approval for substantial changes. For example, a depositor may only be allowed to withdraw a certain amount or percentage of total deposits over a certain time period (e.g., max 100 ether over 1 day) - additional withdrawals in that time period may fail or require approval of an administrator. Or the rate limit could be at the contract level, with only a certain amount of tokens issued by the contract over a time period.

[Example](https://gist.github.com/PeterBorah/110c331dca7d23236f80e69c83a9d58c#file-circuitbreaker-sol)

Source: [We Need Fault Tolerant Smart Contracts](https://medium.com/@peterborah/we-need-fault-tolerant-smart-contracts-ec1b56596dbc)

### Assert Guards

An assert guard triggers when an assertion fails - such as an invariant property changing. For example, the token to ether issuance ratio, in a token issuance contract, may be fixed. You can verify that this is the case at all times with an assertion. Assert guards should often be combined with other techniques, such as pausing the contract and allowing upgrades.

Assert guards can also be combined with automated bug bounties that payout if the ratio changes in a test contract.

The following example reverts transactions if the ratio of ether to total number of tokens changes:

```
contract TokenWithInvariants {
    mapping(address => uint) public balanceOf;
    uint public totalSupply;

    modifier checkInvariants {
        _
        if (this.balance < totalSupply) throw;
    }

    function deposit(uint amount) public checkInvariants {
        // intentionally vulnerable
        balanceOf[msg.sender] += amount;
        totalSupply += amount;
    }

    function transfer(address to, uint value) public checkInvariants {
        if (balanceOf[msg.sender] >= value) {
            balanceOf[to] += value;
            balanceOf[msg.sender] -= value;
        }
    }

    function withdraw() public checkInvariants {
        // intentionally vulnerable
        uint balance = balanceOf[msg.sender];
        if (msg.sender.call.value(balance)()) {
            totalSupply -= balance;
            balanceOf[msg.sender] = 0;
        }
    }
}
```

Source: [We Need Fault Tolerant Smart Contracts](https://medium.com/@peterborah/we-need-fault-tolerant-smart-contracts-ec1b56596dbc#.ju7t49u82) (Peter Borah)

### Contract Rollout

Contracts should have a substantial and prolonged testing period - before substantial money is put at risk.

At minimum, you should:

- Have a full test suite with 100% test coverage (or close to it)
- Deploy on your own testnet
- Deploy on the public testnet with substantial testing and bug bounties
- Exhaustive testing should allow various players to interact with the contract at volume
- Deploy on the mainnet in beta, with limits to the amount at risk

##### Automatic Deprecation

During testing, you can force an automatic deprecation by preventing any actions, after a certain time period. For example, an alpha contract may work for several weeks and then automatically shut down all actions, except for the final withdrawal.

```
modifier isActive() {
    if (now > SOME_BLOCK_NUMBER) {
        throw;
    }
    _
}

function deposit() public
isActive() {
    // some code
}

function withdraw() public {
    // some code
}

```
##### Restrict amount of Ether per user/contract

In the early stages, you can restrict the amount of Ether for any user (or for the entire contract) - reducing the risk.
